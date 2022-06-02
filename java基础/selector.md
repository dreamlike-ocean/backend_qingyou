# selector

> 注意：由于jdk对于底层实现的迭代，为了方便描述，本文源码会参考
>
> [AdoptOpenJDK/openjdk-jdk11: Mirror of the jdk/jdk11 Mercurial forest at OpenJDK (github.com)](https://github.com/AdoptOpenJDK/openjdk-jdk11)
>
> 若无特殊说明均默认使用上面这个

在我们之前的socket一节讲到了我们去read一个inputstream是阻塞的，那么有没有办法变成非阻塞的而且还最好能直接告诉我那些可读呢？

是可以的，这就是我们本文的重点——java.nio的非阻塞IO

非阻塞 IO 的核心在于使用一个 Selector 来管理多个通道，可以是 SocketChannel，也可以是 ServerSocketChannel，将各个通道注册到 Selector 上，指定监听的事件。

之后可以只用一个线程来轮询这个 Selector，看看上面是否有通道是准备好的，当通道准备好可读或可写，然后才去开始真正的读写，这样速度就很快了。我们就完全没有必要给每个通道都起一个线程。

### java层

#### 实例

还是老样子 写一个echo服务器的demo

```java
Selector selector = Selector.open();
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        //让selector监听它的连接到来的事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        serverSocketChannel.bind(new InetSocketAddress(4399));
        while (true){
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey selectionKey = iterator.next();
                if (selectionKey.isConnectable()){
                    ((SocketChannel) selectionKey.channel()).finishConnect();
                    //留空 这里的意义后面讲
                    continue;
                }
                if (selectionKey.isWritable()) {
                   //留空 这里的意义后面讲
                    continue;
                }
                if (selectionKey.isReadable()) {
                
                    SocketChannel channel = (SocketChannel) selectionKey.channel();
                    ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
                    int c = channel.read(buffer);
                    //若读到-1就是连接断开，这里就不做这么仔细了
                    if (c != -1){
                        buffer.flip();
                        channel.write(buffer);
                    }else {
                        //会取消与其关联的全部selector监听事件
                        channel.close();
                    }
                    continue;
                }
                if (selectionKey.isAcceptable()){
                    SocketChannel accept = ((ServerSocketChannel) selectionKey.channel()).accept();
                    //很重要
                    accept.configureBlocking(false);
                    //让selector监听它的数据到来的事件
                    accept.register(selector, SelectionKey.OP_READ);
                }
                iterator.remove();
            }
        }
```

你看完这些代码就可以弄明白到底怎么用了，好，让我们看点细节上面的东西

#### 事件

##### OP_READ

这个就很好理解了，让selector监听这个socket，若其可以被读取，或者读到末尾（read返回0），或者连接断开（read返回-1），或者读取出现了错误都会触发这个事件，实际上我们大部分情况下只关心可被读取和断开两个情况，其他都是很少发生的边缘条件

##### OP_ACCEPT

若对应的ServerSocket的完成握手队列不为空就会触发这个事件

##### OP_WRITE

在之前的socket章节 我们提到了socket buffer若写满了，就会阻塞当前的线程直到可写为止，既然我们现在已经设置为非阻塞了，那我们直接写会阻塞吗？

不会，注意write的返回值，其代表了当前写入了多少数据，若为0则意味着socket buffer满了，这个时候再写实际上不会阻塞但也写不进去，这种时候就需要我们挂载这个事件监听了，等他来通知我们可写，而无需空转不停重试

实际上还有一种情况需要，[shutdown(3) - Linux man page (die.net)](https://linux.die.net/man/3/shutdown)

##### OP_CONNECT

实际上这个用于客户端多，首先，在`non-blocking`模式下调用`socketChannel.connect(new InetSocketAddress("127.0.0.1",8080));`连接远程主机，如果连接能立即建立就像本地连接一样，该方法会立即返回`true`，否则该方法会立即返回`false`,然后系统底层进行三次握手建立连接。连接有两种结果，一种是成功连接，第二种是异常，但是`connect`方法已经返回，无法通过该方法的返回值或者是异常来通知用户程序建立连接的情况，所以由`OP_CONNECT`事件和`finishConnect`方法来通知用户程序。不管系统底层三次连接是否成功，`selector`都会被唤醒继而触发`OP_CONNECT`事件，如果握手成功，并且该连接未被其他线程关闭，`finishConnect`会返回`true`，然后就可以顺利的进行`channle`读写。如果网络故障，或者远程主机故障，握手不成功，用户程序可以通过`finishConnect`方法获得底层的异常通知，进而处理异常。

#### 一些小细节

##### 传输文件

在[FileChannel](FileChannel.md)提到了一个transferTo方法 给两个channel之间拷贝数据，实际上它也可以用于在网络中传输大块的数据，比如说传输文件

```java
public void sendfile(FileChannel fileChannel,SocketChannel socketChannel) throws IOException {
        fileChannel.transferTo(0, fileChannel.size(), socketChannel);
}
```

当我们需要在网络中传输一个文件时，为了避免CPU拷贝就可以使用这个方法（在之后的nio网络编程章节会进行补充）

其内部实现依赖于操作系统对zero copy技术的支持。在unix操作系统和各种linux的版本中，这种功能最终是通过sendfile()系统调用实现。

我来给大家证明一下，通过代码追踪可得到最后会触发到FileChannelImpl的transferTo0方法上面，直接去看对应的c实现：

![1654179042207](assets/1654179042207.png)

其实就是调用[sendfile64](https://linux.die.net/man/2/sendfile64) 

在内核为2.4或者以上版本的linux系统上，socket缓冲区描述符将被用来满足这个需求。这个方式不仅减少了内核用户态间的切换，而且也省去了那次需要cpu参与的复制过程。 从用户角度来看依旧是调用transferTo()方法，但是其本质发生了变化：

1. 调用transferTo方法后数据被DMA从文件复制到了内核的一个缓冲区中。
2. 数据不再被复制到socket关联的缓冲区中了，仅仅是将一个描述符（包含了数据的位置和长度等信息）追加到socket关联的缓冲区中。DMA直接将内核中的缓冲区中的数据传输给协议引擎，消除了仅剩的一次需要cpu周期的数据复制。
3. ![1650621923358](../java%E5%9F%BA%E7%A1%80/assets/1650621923358.png)

#### 

##### wakeup

实际上你会发现`selector::select`会导致线程阻塞起来，若想立刻唤醒对应线程该怎么办？

很简单调用`selector.wakeup`即可，或者直接中断对应阻塞的线程即可。

前者原理非常简单，其实你实例化一个selector这个上面实际上预先注册了一个pipe的读端的读事件，wakeup只是向写端写了一个数据，这样就selector就捕获到了读端可读这个事件，自然也就停止阻塞返回了，每次select都会过滤掉这个读端事件，所以我们看不到

后者原理就是调用select时给Thread里面blocker赋值，相当于注册了一个调用interrupt()方法时的回调，这个回调就是调用wakeup

![1654177555794](assets/1654177555794.png)

然后我们看一下精简版本的wakeup

```java
public Selector wakeup() {
    //省去不必要用于同步的代码
    IOUtil.write1(fd1, (byte)0);      
    return this;
}
```



##### socket attachment

如果我们想给socket绑定一个对象的话，比如说绑定一个上次没写完的ByteBuffer，类似于epoll api中的epoll_data中的void *ptr

```java
selectionKey.attach(new Object());
Object o = selectionKey.attachment();
```

看起来和epoll_data用法差不多，但是并不是通过这个实现的，思考一下就知道了由于GC挪动对象，所以肯定不能用堆外指针指向一个对象。

这个attach实际上具有volatile语义，所以跨线程的情况下能保证attachment()调用的可见性和因果性，这个attachment是和selectionkey绑定的，也就是说找到了key就可以找到这个attachment

我们来看看linux下面是怎么实现的这个功能

![1654177407712](assets/1654177407712.png)

就是建立了一个fd->selectionKey的映射，epoll会告知我们触发的是哪个fd，然我们就可以找到对应的selectionKey，进而找到对应的attachment了，很巧妙。

##### 线程安全

Selector 对于多个并发线程来说是安全的，听起来很不错对吧？

实际上不注意很容易导致死锁,比如说一个线程无限期等待select返回，另外一个线程去register，这就会导致死锁。因为它们会争抢同一个publishkey对象作为monitor（这个结论实际上来自于jdk8的源码）

从jdk11来看，不存在这个问题，但是我们不能保证运行我们代码的环境不是8，虽然现在有向更高迁移的趋势。

正确的思路应该是：

用个生产者消费者模型，把要 register 的 channel 放到队列中，selector线程在每次 select 前先 register 队列中的 channel 即可，若selector线程在阻塞就再加一步wakeup即可，实际上netty和jdk11也是这样实现的。

#### 各个平台的selector实现

受限于我看的源码水平，我只了解linux和Windows的实现

Linux：

Epoll，而且是水平触发模式——即若socket接收缓冲区（RCVBUF）不为空那么就会一直触发，这个没法更改是写死在c代码里面的，而且由于还是写死的问题，一次select出来的**就绪**事件最多是1024个。其效率是O(m) m为就绪的fd数目。

讲到这里，我们再来提一提epoll另一个模式——边缘触发，即缓冲区满才触发，若我们一次性没读完，那么下次事件就得等到缓冲区满才触发了。

Windows：

17之前是基于[select](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-select)的，一次性只能监听1024个fd，而且其效率是O(n)，n为监听的fd数量，因为上限只有1024个，所以超过的部分会多开线程来监听，每1024fd开一个线程监听，每次的select方法调用都是统计一下各个线程汇报上来的消息。

17开始是基于[wepoll](https://github.com/piscisaureus/wepoll)的，其底层是IOCP——一种AIO实现，具体讨论请看[[JDK-8266369\]](https://bugs.openjdk.java.net/browse/JDK-8266369)性能相较于过去好很多



