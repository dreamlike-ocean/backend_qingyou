# State of Loom: Part 2

**Ron Pressler, May 2020**

**Contents**

- [Channels](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#channels)
- [结构化并发](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#structured-concurrency)
- [范围变量](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#scope-variables)
- [处理器本地变量](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#processor-locals)
- [对于中断和取消的更多工作](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#more-on-interruption-and-cancellation)
- [强制抢占](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part2.html#forced-preemption)

[*State of Loom: Part 1*](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part1.html)介绍了虚拟线程以及JDK如何去做适配的。随着线程数量的增加和寿命的缩短，管理线程和在线程之间分配工作负载的新方法也属于Project Loom的权限范围：在线程之间传递数据的新的、灵活的机制（如**channels**）可能是可取的；聚集大量线程可以受益于一种称为结构化并发的组织和监督线程的方法；最后，我们正在探索一种比线程局部变量更方便、更高效的上下文数据构造，我们暂时称之为**范围变量(scope vaiables)**。

和往常一样，如果您能向[loom-dev](https://mail.openjdk.java.net/pipermail/loom-dev/) 邮件列表反馈您使用loom的体验，我们将不胜感激。

### Channels

当涉及到通信由线程计算的单个结果时,[`java.util.concurrent.Future`](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Future.html) 已经足够用了，[`CompletableFuture`](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/CompletableFuture.html)类提供了一种很好的方式来连接同步和异步世界：使用`thenXXX`方法实现异步，使用`get`实现同步。当涉及到多个结果的通信时， [`Flow`](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/Flow.html) 类为异步代码提供了一个很好的解决方案，在设计出更多的同步解决方案之前， [`BlockingQueue`](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/BlockingQueue.html)可以用于在线程（尤其是 [`LinkedTransferQueue`](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/LinkedTransferQueue.html)）之间传递多个值。

然而,`BlockingQueue`不能简洁地表示 “数据流的结束”或者是一个错误（译者：前者指当前传递数据结束，后者是指源头发送了一个错误告诉下游源头出问题了），而且发送方和接收方也不能简洁地分开，这导致了在此之上附加过滤器（filter），映射器（mapper）和其他组合子或者表示分发信息非常笨拙。如果可以拥有一个适合的channel类型会非常有利于我们——同步代码中与`Flow`对应的类型——Doug Lea(译者：这位是J.U.C的创造者之一)正在致力于它的设计。[Pattern matching](https://cr.openjdk.java.net/~briangoetz/amber/datum.html)可以让channel和message配合的很好

在一个原型中，channel类型被称为`Carrier`，以区别于NIO中的channel，虽然之后API可能会发生变换，但是就目前来讲，它可能看起来是这样的：

```java
Carrier<String> carrier = new Carrier<>();

Thread producer = Thread.startVirtualThread(() -> {
  Carrier.Sink<String> sink = carrier.sink();
  sink.send("message1");
  sink.send("message2");
  sink.closeExceptionally(new InternalError());
});

Thread consumer = Thread.startVirtualThread(() -> {
  try (Carrier.Source<String> source = carrier.source()) {
    while (true) {
      String message = source.receive();
      System.out.println(message);
    }
  } catch (IllegalStateException e) {
    System.out.println("consumer: " + e + " cause: " + e.getCause());
  }
});

producer.join();
consumer.join();
```

### 结构化并发

由于现在线程变得廉价且数量众多，他们可以从协作中收益良多。我们发现了一个非常吸引人的方法可以帮我们轻松实现这种——*结构化并发*(structured concurrency)

结构化并发将线程生命周期压缩为代码块。与结构化编程将顺序执行的控制流限制在定义良好的代码块中的方式类似，结构化并发对并发控制流也有同样的作用。它的基本原理是：在某个代码单元中创建的线程必须在我们退出该代码单元时全部终止；如果执行拆分到某个作用域内的多个线程，则必须在退出该作用域之前进行join。特殊情况下，无限生成线程的方法不应返回。

为了让这个方法理解起来更加清晰，让我们看一些代码。在我们当前的原型里面，我们表示一个结构化并发的作用域，通过让[`java.util.concurrent.ExecutorService`](https://download.java.net/java/early_access/loom/docs/api/java.base/java/util/concurrent/ExecutorService.html) 实现`AutoCloseable`接口来限制子线程的生命周期，`close`会关闭这个线程池并等待全部任务的结束。这保证了提交到线程池的任务在我们退出try-with-resources（TWR）块的时候任务都已经终止，这样就做到了将生命周期限制在代码结构中

```java
ThreadFactory vtf = Thread.builder().virtual().factory();
try (ExecutorService e = Executors.newUnboundedExecutor(vtf)) {
   e.submit(task1);
   e.submit(task2);
} // 阻塞并等待
```

在我们退出TWR块之前，当前的线程将会被阻塞，等待全部的任务及其线程结束。一旦离开这个作用域，我们就可以保证全部的任务都已经结束

因此，我们的代码直接就可以用于那些非常有组织的表示，类似于一些[process calculi](https://en.wikipedia.org/wiki/Process_calculus)。如果`;`代表顺序组合而`|`代表并行组合，我们的代码可以这样描述:`a;(b|((c|d);e));f`，其中的连接点——就是我们等待子线程完成的点——使并行操作之后和下一个顺序操作之前的右括号，比如`(x|y|z);w`

除了代码变得清晰之外，这种结构还带来了一种强大的不变量：每一个线程（在结构化并发上下文中的）都有一些“父”线程被阻塞等待其终止，因此要关心其何时终止或者怎样终止。它在错误传播和取消操作方面有一些强大的优势

#### 结构化中断

An unstructured thread is detached from any context or clear responsibility. Since a structured thread clearly performs some work for its parent, when the parent is cancelled the child should also be cancelled.

一个非结构化的线程与任何上下文以及清晰的职责无关。由于结构化线程显然会为其父线程执行一些工作，因此当父线程被取消时，子线程也应该被取消

因此，如果父线程被中断，我们将中断传播到子线程。我们还可以给所有任务一个deadline，该deadline将中断那些在任务到期时尚未终止的子任务（以及当前线程）：

```java
try (var e = Executors.newUnboundedExecutor(myThreadFactory)
                      .withDeadline(Instant.now().plusSeconds(30))) {
   e.submit(task1);
   e.submit(task2);
}
```

#### 结构化错误

一个非结构化的线程可能会出现异常然后在无人注意的情况下死去。一个结构化的线程的错误将被其父线程观测到，然后这个错误将会被放到上下文中，举个例子，通过将子线程异常的堆栈加入到父线程的堆栈中。

但是异常的传播存在一些挑战。假设子线程引发的异常会自动传播到其父线程，从而取消（中断）其所有其他子线程。在某些情况下，这很可能是一种可取的行为，但这是否应该成为默认行为还不太清楚。所以目前，我们正在实验更明显的错误和结果处理

我们可以使用新的[`ExecutorService.submitTasks`](https://download.java.net/java/early_access/loom/docs/api/java.base/java/util/concurrent/ExecutorService.html#submitTasks(java.util.Collection)) 和[`CompletableFuture.stream`](https://download.java.net/java/early_access/loom/docs/api/java.base/java/util/concurrent/CompletableFuture.html#stream(java.util.Collection))，在每个任务完成的时候流式处理各个结果，处理成功与否（这也是通往`CompletableFuture`异步世界的桥梁），以等待第一个成功完成的任务，然后取消全部的其他任务

```java
try (var e = Executors.newUnboundedVirtualThreadExecutor()) {
  List<CompletableFuture<String>> tasks = e.submitTasks(List.of(
    () -> "a",
    () -> { throw new IOException("too lazy for work"); },
    () -> "b",
  ));
                                                        
  try {
    String first = CompletableFuture.stream(tasks)
      .filter(Predicate.not(CompletableFuture::isCompletedExceptionally))
      .map(CompletableFuture::join)
      .findFirst()
      .orElse(null);
    System.out.println("one result: " + first);
  } catch (ExecutionException ee) {
    System.out.println("¯\\_(ツ)_/¯");
  } finally {
    tasks.forEach(cf -> cf.cancel(true));
  }
}
```

一些常见的模式可以通过一些辅助函数来实现，比如说`ExecutorService`的`invokeAll` 或 `invokeAny`，这个例子与上面那个例子做了相同的事情

```java
try (var e = Executors.newUnboundedVirtualThreadExecutor()) {
  String first = e.invokeAny(List.of(
    () -> "a", 
    () -> { throw new IOException("too lazy for work"); },
    () -> "b"
  ));
  System.out.println("one result: " + first);
} catch (ExecutionException ee) {
  System.out.println("¯\\_(ツ)_/¯");
}
```

这些API还在EA版本（译者：早期预览版）,但随着我们努力使线程管理更加友好，这方面可能会有很多变化。

#### 结构化的可维护性和可观察性

结构化并发不只是帮助我们组织代码，它还可以在分析和调试中提供一些有意义的上下文。一百万个线程的线程转储可能没有用处，但如果这些线程可以按照结构化并发范围层次结构排列在树中，那么它们就更有意义了；类似地，JFR可以通过SC（译者：Structured Scope结构化并发作用域）作用域对线程及其执行的操作进行分组，允许放大或缩小配置文件。但这个功能作不太可能在第一次预览中提供。

### 范围变量

我们有时需要以对中间层透明的方式将一些上下文从调用者传递给可传递的被调用者（译者：比如ThreadLocal传参）。举个例子，假设有这么一个调用链`foo`→`bar`→`baz`,`foo`和`baz`使应用层代码，但`bar`是第三方库代码反之亦然。`foo`想要在无需在`bar`参与的情况下与`baz`共享数据。如今，这通常是通过ThreadLocals实现的，在这里我们称之为TL，这种方式有很大的缺点。

首先，它们是非结构化的，与我们上面使用的类似：一旦设置了TL值，它就会在线程的整个生命周期内生效，或者直到它被设置为其他值为止。事实上，我们通常会看到一种使用模式，试图借用TL结构（不幸的是，这种没有任何性能优势）：

```java
var oldValue = myTL.get();
myTL.set(newValue);
try {
  ...
} finally {
  myTL.set(oldValue);
}
```

如果没有这种强制结构，当一个线程在多个任务之间共享时，一个任务的TL值可能会泄漏到另一个任务中。虚拟线程通过足够轻量级而不需要共享来解决这个问题。然而，这种非结构化还意味着TL实现必须依赖于弱引用，以允许GC清除不再使用的TL，这使得它们这种实现的运行速度显著降低。

另一个问题是继承。例如，那些使用分布式跟踪（如OpenTracing）的人可能希望从父线程继承跟踪“跨度”(span)。这可以通过`InheritableThreadLocal`（iTL）实现。创建线程时必须复制线程中的iTL映射，因为（i）TLs是可变的，因此无法共享。这会造成内存使用效率和运行速度损失。另外，因为现在的线程是非结构化的，所以当一个子线程访问其继承的跨度时，它的父线程可能已经关闭了它。

TL继承的问题只会因为虚拟线程而加剧，因为虚拟线程鼓励创建许多小线程，一些线程代表其父线程执行小任务，比如单个HTTP请求，从而增加了TL继承的需要，以及复杂的占用空间和速度成本。 

如果TL在设置后是不可变的，继承则是非常高效的，但是考虑到一个可能设置TL值的方法，它可能会抛出一个非法状态的异常，这种情况取决于调用方是否对相同的TL设置了值，这种情况严重影响了代码的可组合性

为了解决这种问题，我们正在探索一种更好的结构，它具有更好的性能，内存占用和而且是结构化的，正确的。我们暂时称之为*范围变量*(scope variables，SV)。类似于TL，SV引入了一些隐式上下文，但是不同于TL，它们是在代码块的范围内构造和生效的，而不是在线程的整个生命周期内。SV也是不可变的，尽管它们的值可以被嵌套的作用域遮蔽（shadow）。

下面是一个在当前EA原型中使用`java.lang.Scoped` API的例子

```java
static final Scoped<String> sv = Scoped.forType(String.class);

void foo() {
    try (var __ = sv.bind("A")) {
    bar();
    baz();
    bar();
  }
}

void bar() {
  System.out.println(sv.get());
}

void baz() {
  try (var __ = sv.bind("B")) {
    bar();
  }
}
```

`baz`并没有修改`sv`绑定的值而是在嵌套的作用域内绑定了新的值，它遮蔽了原来绑定的值，所以`foo`会打印

```java
A
B
A
```

因为SV绑定的生命周期是明确定义的，所以我们不需要依赖GC进行清理，因此我们不需要弱引用来降低速度。

那继承呢？因为SV是不可变的，而且结构化并发也给了我们一个限制性语法的线程生存期，所以SV继承就像手套一样适合结构化并发：

```java
try (var context = Foo.openContext()) { // some temporary context that can be closed
  try (var __ = contextSV.bind(context);
       var executor = Executors.newUnboundedExecutor(myThreadFactory)) {
    executor.submit(() -> { ... });
    executor.submit(() -> { ... });
  }
}
```

提交的任务会自动继承`contextSV`的`context`值，并且由于无界线程池的作用域被`context`的生命周期所包括，因此任务可以确定它们从`contextSV`获取到的上下文并没有被关闭

其他类型的结构化构造（即计算仅限于语法元素的构造）也可以提供自动SV继承。例如：

```java
try (var __ = sv.bind(value)) {
    Set.of("a", "b", "c").stream().parallel()
       .forEach(s -> System.out.println(s + ": " + sv.get()));
}
```

因为流的`forEach`操作也完全局限于SV的绑定范围，所以可以继承`value`，即使`forEach`可能在不同线程的不同流元素上执行其主体操作。

范围变量仍处于设计阶段的早期阶段，并且与我们可能引入的更一般的更改有关，以尝试使用资源（请参阅[这里](https://bugs.openjdk.java.net/browse/JDK-8243098)了解一些想法）。即使我们决定继续使用SVs，他们也可能会错过第一次预览和GA。

### 处理器本地变量

线程局部变量的另一个用途不是将数据与线程上下文关联，而是“条带化”一些写操作繁重、可变的数据结构，以避免争用（例如，LongAdder，它不使用ThreadLocal类，但依赖于类似的思想）。当线程的数量不比内核的数量大多少时，这是有意义的，但对于可能有数百万个线程的情况，这纯粹是开销。我们正在探索一种具有类似CAS语义的“处理器本地”构造，它甚至比在适当的操作系统支持下的无竞争CAS还要快，比如说[Linux’s restartable sequences](https://www.efficios.com/blog/2019/02/08/linux-restartable-sequences/).

（译者：条带化即为`stripe`,将一块数据分成不同部分写的时候不同部分可以并行，读的时候再串起来，类似于早版本的ConcurrentHashMap的segment实现）

### 对于中断和取消的更多工作

线程支持一种协作式的中断机制，这种机制由方法`interrupt()`, `interrupted()`, `isInterrupted()`, 和`InterruptedException`组成。这是一个相当复杂的机制：一些线程调用另一个线程的`interrupt`，设置目标线程的中断状态。目标线程轮询其中断状态，可能会从阻塞方法中抛出InterruptedException，但也会`清除`该状态,

这有两个原因：第一个，线程可能是被池化的共享资源，当前任务可能会被中断，但调度器可能希望重用线程来运行其他任务，因此必须重置中断状态。对于虚拟线程来说，这是不必要的，因为它们足够轻量级，不能用于不同的任务。但还有另一个原因：线程可能会观察到它被中断，并且作为其清理过程的一部分，若其希望之后调用阻塞方法，如果状态没有被清除，如果状态未被清除，阻塞方法将立即抛出`InterruptedException`.虽然这个机制确实解决了一个实际需求，但它很容易出错，我们想重新讨论一下。我们已经尝试了一些原型，但目前还没有任何具体的建议。

### 强制抢占

尽管我们已经阐述了目前[调度](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part1.html#scheduling)是什么样子的，但在某些特殊情况下，强制抢占占用CPU的线程可能是有用的。例如，代表多个客户端应用程序执行复杂数据查询的批处理服务可能会接收客户端任务，并在各自的虚拟线程中运行它们。如果这样的任务占用了太多的CPU，服务可能希望强制抢占它，并在服务负载较轻时再次调度它。为此，我们计划让虚拟机支持一种操作，该操作试图在任何安全点强制抢先执行。该功能将如何向调度器公开是待定的，并且很可能不会出现在第一次预览中。