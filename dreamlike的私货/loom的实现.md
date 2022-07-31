##  **虚拟线程网络I/O底层实现**

> 本文是对[Networking i/o with virtual threads - under the hood – Inside.java](http://link.zhihu.com/?target=https%3A//inside.java/2021/05/10/networking-io-with-virtual-threads/)的翻译

## 使用虚拟线程进行网络 IO

Project Loom 主要目标是在 Java 平台上提供一种**易于使用、高吞吐量的轻量级并发性和新的编程模型**的 JVM 特性和API。这带来了许多有趣和令人兴奋的前景，其中之一是**简化网络交互的代码的同时兼顾性能**。现在的服务器能够处理打开的**socket连接的数量**，远远**超过它们能够支持的线程数量**，这既带来了机遇，也带来了挑战。

但是不幸的是，编写与网络交互的**可伸缩代码**是很困难的。我们一般使用同步 API 的方式进行编码，但是在超过一定阈值之后，同步代码就迎来了瓶颈，很难进行伸缩。因为这样的API在执行 I/O 操作时会阻塞，而 I/O 操作又会将线程绑定起来，直到操作就绪，例如尝试从套接字读取数据但是当前并没有数据要读取的时候。目前的线程，在 Java 平台中是一个**昂贵的资源**，以至于无法等待 I/O 操作的完成再去释放。为了解决这个限制，我们通常使用异步 I/O 或 Ractor 框架，因为它们可以构造出在 I/O 操作中不用绑定线程的代码，而是在 I/O 操作完成或准备就绪时使用**回调或事件**通知线程进行处理。

使用异步和非阻塞 API 比使用同步 API 更具有挑战性，部分原因是用这些 API 写出来的代码是比较反人类的。同步API在很大程度上更容易使用；代码更易于编写、更容易阅读和更易于调试，调试的时候堆栈里面的信息大部分是有用的。但是如前所述，使用同步 API 的代码不能像异步代码那样伸缩扩展，因此我们必须做一个艰难的选择：选择更简单的同步代码，并接受它不会扩展；或者选择更可伸缩的异步代码，并处理所有的复杂性。两个都不是个好选择！Project Loom 主要就是要让同步代码也能灵活伸缩扩展。

在这篇文章里面，我们将了解在调用虚拟线程时，Java平台的网络api在底层是如何工作的。细节在很大程度上是实现的产物，我们并不需要知道什么时候在上面编写代码，但是了解在底层是如何工作的仍然是很有意思的事情，而且可能可以帮助回答一些问题，因为如果没有答案，可能会导致再次不得不做出艰难的选择。

## 虚拟线程（**纤程**）

在进一步研究之前，我们需要了解一下ProjectLoom中的新线程--Virtual threads。

虚拟线程是用户态线程，被 JVM 管理，而不是操作系统。虚拟线程占用的系统资源很少，一个 JVM 可以容纳百万量级的虚拟线程。特别适合于经常执行阻塞时间比较长，经常等待 IO 的任务。

**平台线程**（即目前 Java 平台的线程），是和操作系统内核线程一一对应的。平台线程通常拥有一个非常大的栈，以及其他的一些系统维护的资源。**虚拟线程**则使用一小组用作载体线程的平台线程。在虚拟线程中执行的代码通常不会知道底层承载的线程。锁和 I/O 操作是将承载线程**从一个虚拟线程重新调度到另一个虚拟线程的调度点**。虚拟线程可能会 parked（例如`LockSupport.park()`），从而使其无法调度。一个已 parked 的虚拟线程可能被取消（例如`LockSupport.unpark(Thread)`），这样重新启用了它的调度。

## 网络 API

Java 平台中主要有两种网络 API：

1. 异步 - `AsynchronousServerSocketChannel`、`AsynchronousSocketChannel`
2. 同步 - `java.net.Socket`、`java.net.ServerSocket`、`java.net.DatagramSocket`、`java.nio.channels.SocketChannel`、`java.nio.channels.ServerSocketChannel`、`java.nio.channels.DatagramChannel`

第一类异步 API，创建启动在之后某个时间完成的 I/O 操作，可能在启动 I/O 操作的线程之外的线程上完成。根据定义，这些 API 不会导致阻塞的系统调用，因此在**虚拟线程中运行时不需要特殊处理**

第二类同步 API，从它们在虚拟线程中运行时的行为角度来看，它们更有趣。在这些 API 中，NIO channel 相关的可以配置成为非阻塞模式。这种 channel 通常使用 I/O 事件通知机制实现，例如注册到 Selector 上监听事件。类似于异步网络 API，在虚拟线程中执行不需要额外处理，因为 I/O 操作不自己调用阻塞的系统调用，这个调用留给了 Selector。最后，我们来看看将 channel 配置成为阻塞模式以及 `java.net` 相关 API 的情况（我们这里称这种 API 为同步阻塞 API）。同步 API 的语义要求 I/O 操作一旦启动，在调用线程中完成或失败，然后将控制权返回给调用方。但是，如果 I/O 操作“尚未准备好”怎么办呢？例如，目前没有数据可以读取。

## 同步阻塞 API

在虚拟线程中运行的 Java 同步网络 API 会**将底层原生 Socket 切换到非阻塞模式**。当 Java 代码启用一个 I/O 请求并且这个请求没有立即完成（原生 socket 返回 EAGAIN - 代表"未就绪"/"会阻塞"）的时候，这个底层 socket 会被注册到一个 JVM 内部事件通知机制(一个轮询器——Poller)，并且虚拟线程会被 parked。当底层 I/O 操作就绪的时候（有相关事件会到达 Poller），虚拟线程会 unparked 并且底层的 Socket 操作会被重试底层的socket操作。

让我们更近距离看看这个例子，这个`retrieveURLs`方法将下载并且返回多个url对应的响应

接下来编写代码：

```text
//Java 16 中的 Record 对象，可以理解为有包含两个 final 属性（url 和 response）的类
record URLData (URL url, byte[] response) { }
List<URLData> retrieveURLs(URL... urls) throws Exception {
  try (var executor = Executors.newVirtualThreadExecutor()) {
    var tasks = Arrays.stream(urls)
            .map(url -> (Callable<URLData>)() -> getURL(url))
            .toList();
    return executor.submit(tasks)
            .filter(Future::isCompletedNormally)
            .map(Future::join)
            .toList();
  }
}
```

`retrieveURLs`方法创造了一个任务的列表（为每个URL）然后把他们投递到线程池中，之后等待结果。线程池为每个任务开启一个新的虚拟线程，他们会调用`getURL`.为简单起见，只返回成功完成的任务。

`getURL`方法编写成使用同步`URLConnection` API来获得响应。

```text
URLData getURL(URL url) throws IOException {
  try (InputStream in = url.openStream()) {
    return new URLData(url, in.readAllBytes());
  }
}
```

`readAllBytes`方法是一个读取所有响应字节的批量同步读取操作。在外壳之下，`readAllBytes`最终在*java.net.socket*输入流的`read`方法中达到最底层。

如果我们运行一个小程序，使用`retrieveURLs`下载一个HTTP URL，而HTTP服务器没有提供完整的响应，我们可以检查线程的状态如下:

```text
$ java Main & echo $!
89215
$ jcmd 89215 JavaThread.dump threads.txt
Created /Users/chegar/threads.txt
```

我们查看`threads.txt`这个文件，其中我们关心的线程信息是：

```text
$ cat threads.txt
...
"<unnamed>" #15 virtual
  java.base/java.lang.Continuation.yield(Continuation.java:402)
  java.base/java.lang.VirtualThread.yieldContinuation(VirtualThread.java:367)
  java.base/java.lang.VirtualThread.park(VirtualThread.java:534)
  java.base/java.lang.System$2.parkVirtualThread(System.java:2370)
  java.base/jdk.internal.misc.VirtualThreads.park(VirtualThreads.java:60)
  java.base/sun.nio.ch.NioSocketImpl.park(NioSocketImpl.java:184)
  java.base/sun.nio.ch.NioSocketImpl.park(NioSocketImpl.java:212)
  java.base/sun.nio.ch.NioSocketImpl.implRead(NioSocketImpl.java:320)
  java.base/sun.nio.ch.NioSocketImpl.read(NioSocketImpl.java:356)
  java.base/sun.nio.ch.NioSocketImpl$1.read(NioSocketImpl.java:807)
  java.base/java.net.Socket$SocketInputStream.read(Socket.java:988)
  java.base/java.io.BufferedInputStream.fill(BufferedInputStream.java:255)
  java.base/java.io.BufferedInputStream.read1(BufferedInputStream.java:310)
  java.base/java.io.BufferedInputStream.lockedRead(BufferedInputStream.java:382)
  java.base/java.io.BufferedInputStream.read(BufferedInputStream.java:361)
  java.base/sun.net.www.MeteredStream.read(MeteredStream.java:141)
  java.base/java.io.FilterInputStream.read(FilterInputStream.java:132)
  java.base/sun.net.www.protocol.http.HttpURLConnection$HttpInputStream.read(HttpURLConnection.java:3648)
  java.base/java.io.InputStream.readNBytes(InputStream.java:409)
  java.base/java.io.InputStream.readAllBytes(InputStream.java:346)
  Main.getURL(Main.java:24)
  Main.lambda$retrieveURLs$0(Main.java:13)
  java.base/java.util.concurrent.FutureTask.run(FutureTask.java:268)
  java.base/java.util.concurrent.ThreadExecutor$TaskRunner.run(ThreadExecutor.java:385)
  java.base/java.lang.VirtualThread.run(VirtualThread.java:295)
  java.base/java.lang.VirtualThread$VThreadContinuation.lambda$new$0(VirtualThread.java:172)
  java.base/java.lang.Continuation.enter0(Continuation.java:372)
  java.base/java.lang.Continuation.enter(Continuation.java:365)
```

从下往上看堆栈帧;首先，我们看到许多与虚拟线程设置相关的帧(“continuation”是虚拟线程内部使用的虚拟机的机制)，它们对应于executor服务创建的新线程。其次，我们看到一些帧对应于调用 `retrieveURLs`'和'`getURL`的测试程序。第三，我们看到对应于HTTP协议处理程序的帧以及socket输入流实现的`read`方法。最后，在堆栈中跟踪这些帧，我们可以看到虚拟线程已经*暂停*，这是我们所期望的，因为服务器没有发送完整的响应，所以没有足够的数据来读取socket。但是，如果当数据到达socket上时，如何*启动*虚拟线程?

仔细看看*threads.txt*中的其他系统线程，我们可以看到:

```text
"Read-Poller" #16
  java.base@17-internal/sun.nio.ch.KQueue.poll(Native Method)
  java.base@17-internal/sun.nio.ch.KQueuePoller.poll(KQueuePoller.java:65)
  java.base@17-internal/sun.nio.ch.Poller.poll(Poller.java:195)
  java.base@17-internal/sun.nio.ch.Poller.lambda$startPollerThread$0(Poller.java:65)
  java.base@17-internal/sun.nio.ch.Poller$$Lambda$14/0x00000008010579c0.run(Unknown Source)
  java.base@17-internal/java.lang.Thread.run(Thread.java:1522)
  java.base@17-internal/jdk.internal.misc.InnocuousThread.run(InnocuousThread.java:161)
```

这个线程是jvm范围的读轮询器。它的核心是执行一个基本的事件循环，监视所有在虚拟线程中调用时没有立即准备好的同步网络操作：*read*， *connect*和*accept*。当I/O操作准备好时，将通知轮询器，并随后*启动后*适当的*暂停的*虚拟线程。对于*write*操作，有一个等效的*写-轮询器*。

上面的堆栈跟踪是在macOS上运行测试程序时捕获的，这就是为什么我们会看到与macOS上的轮询器实现相关的堆栈帧，即[kqueue](http://link.zhihu.com/?target=https%3A//www.freebsd.org/cgi/man.cgi%3Fquery%3Dkqueue%26sektion%3D2)。在Linux上轮询器使用[epoll](http://link.zhihu.com/?target=https%3A//man7.org/linux/man-pages/man7/epoll.7.html)，在Windows上是[wepoll](http://link.zhihu.com/?target=https%3A//github.com/piscisaureus/wepoll)(它在Winsock的辅助功能驱动程序上提供了类似epoll的API)。

轮询器维护一个*文件描述符*到虚拟线程的映射。当向轮询器注册文件描述符时，将向该文件描述符的映射添加一个条目，并将注册线程作为其值。当被事件唤醒时，轮询器的事件循环将使用事件的文件描述符来查找相应的虚拟线程并将其解除暂停状态。

### **扩展**

如果你仔细观察，你会发现上面的行为与当前使用NIO channel和selector的可扩展代码并没有太大的不同——它们可以在许多服务器端框架和库中找到。虚拟线程的不同之处在于向开发人员公开的编程模型。前者暴露了一个更复杂的模型,用户代码必须实现事件循环和维护应用程序逻辑,而后者暴露了一种更简单和更简单的编程模模型——Java平台来处理任务的调度和维护跨I / O边界的上下文。

用于调度虚拟线程的默认调度器是fork-join work-stealing调度器，它非常适合这项工作。用于监视就绪I/O操作的原生事件通知机制是操作系统提供的一种同样现代和高效的机制。虚拟线程构建在Java VM中的continuation支持之上。因此，同步的Java网络api可以支持的规模应该与更复杂的异步和非阻塞代码构造的规模相当。

### **结论**

同步Java网络api已经由[JEP 353](http://link.zhihu.com/?target=https%3A//openjdk.java.net/jeps/353)和[JEP 373](http://link.zhihu.com/?target=https%3A//openjdk.java.net/jeps/373)重新实现，为Project Loom做准备。在虚拟线程中运行时，如果I/O操作没有立即完成，将导致虚拟线程被暂停。当I/O就绪时，虚拟线程将被启动。该实现使用了来自Java VM和Core库的几个特性，提供了一个可扩展的、高效的替代方案，与当前的异步和非阻塞代码构造相比，它更有优势。



请尝试[Early Access](http://link.zhihu.com/?target=https%3A//jdk.java.net/loom/)loom的构建版本，我们很乐意听到你的体验，你可以发送到[loom-dev](https://zhuanlan.zhihu.com/邮箱:loom-dev@openjdk.java.net)邮件列表。