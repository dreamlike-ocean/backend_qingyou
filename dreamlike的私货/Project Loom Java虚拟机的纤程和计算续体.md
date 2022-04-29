# Project Loom: Java虚拟机的纤程和计算续体

译者：

几个重要名词的翻译

 continuation（计算续体）

scheduler（调度器）

delimited continuation（定界续体）

原文部分由于英文单词含有一定语义 故下文对应部分不翻译

[Loom Proposal.md (java.net)](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)

## 概述

Project Loom的目标是使编写、调试、配置和维护并发应用程序变得更容易，以满足当今的需求。Java从一开始就提供的线程是一种自然而方便的并发结构(撇开线程间通信的问题)，它正被不太方便的抽象所取代，因为通过系统内核线程实现的他们并不足以满足当下的需求，并且浪费了在云计算中特别有价值的计算资源。Project Loom将引入纤程作为由Java虚拟机管理的轻量级、高效的线程，让开发人员使用同样简单的抽象，但具有更好的性能和更低的占用空间。我们想让并发再次变得简单!纤程由两个部分组成——continuation（计算续体）和scheduler（调度器）。由于lava已经有了`ForkJoinPool`形式的优秀调度器，纤程将通过向IVM添加continuation来实现

## 动机

许多为Java虚拟机编写的应用程序都是并发的——也就是说，像服务器和数据库这样的程序需要服务于许多请求，并发由此产生并竞争计算资源。Project Loom旨在显著降低编写高效并发应用程序的难度，或者更准确地说，消除编写并发程序时对简单性和效率之间的权衡难度。

二十多年前，当Java首次发布时，它最重要的贡献之一就是方便地访问线程和同步原语。Java线程(直接使用或间接使用它，例如，通过Java servlet处理HTTP请求)为编写并发应用程序提供了一个相对简单的抽象。然而，目前编写满足当今需求的并发程序的主要困难之一是运行时提供的软件并发单元(线程)不能与业务领域的并发单元(无论是用户、事务还是单个操作)的规模相匹配。即使应用程序并发性是一个粗粒度的单位——比如一个会话,一个单一的套接字连接,服务器可以处理一百万个并发打开的套接字,然而Java运行时提供的则是使用操作系统的线程实现的Java线程,它不能有效地处理超过几千的线程数。这几个数量级的不匹配会产生很大的影响

程序员被迫选择是把领域并发单元直接建模为线程，这在单个服务器会损失极大的并发规模，还是使用其他结构在比线程(任务)更细粒度的级别上实现并发，并且通过编写不阻塞运行线程的异步代码来支持并发。

近年来，Java生态系统引入了许多异步api，从JDK中的异步NIO、异步servlet和许多异步第三方库。这些api被创造出来不是因为它们更容易编写和理解，甚至它们实际上更难;不是因为它们更容易调试或分析——甚至会更困难(它们甚至不会产生有意义的堆栈跟踪);并不是因为他们的代码结合比同步的api好——他们的结合不那么优雅;不是因为它们更适合语言中的其他部分，或者与现有代码集成得很好，而是因为并行性的软件单元——线程——的实现从内存和性能的角度来看是不够的。由于抽象的运行时性能问题，一个好的、自然的抽象被抛弃，而倾向于一个不那么自然的抽象，这是一个可悲的情况。

虽然使用内核线程作为Java线程的实现有一些优点——最显著的原因是所有本地代码都由内核线程支持，因此在线程中运行的Java代码可以调用本地api——但是上面提到的缺点太大了以至于不容忽视，结果要么是难以编写、维护代价昂贵的代码，要么是计算资源的严重浪费，这些代价在代码在云平台中运行时尤其昂贵。事实上，一些语言和语言运行时成功地提供了轻量级线程实现，最著名的是Erlang和Go，该特性非常有用和流行。

这个项目的主要目标是添加一个轻量级的线程结构，我们称之为纤程，由Java运行时管理，它将可选地与现有的重量级、os提供的线程实现一起使用。纤程在内存占用方面比内核线程轻得多，它们（译者：指纤程之间）之间的任务切换开销接近于零。在单个JVM实例中可以产生数百万个纤程，程序员们无需考虑过多就可以发出同步的阻塞调用，因为阻塞实际上是轻量的。除了使并发应用程序更简单或更具有可伸缩性之外，这将使库作者的工作更容易，因为不再需要为不同的简洁性/性能权衡而同时提供同步和异步api。简洁性将是没有代价的。

正如我们将看到的，线程不是一个单一结构，而是两个关注点的组合——scheduler和*continuation*。我们目前的意图是将这两个问题分开,并实现Java纤程最重要的这两个构建部分而且,尽管纤程是这个项目的主要动机,也添加continuation作为用户面对所面向的抽象,作为continuation也有其他用途(例如 [Python's generators](https://wiki.python.org/moin/Generators))。

## 目标和范围

纤程可以提供一个低级的原语，在此基础上可以实现有趣的编程范例，比如channel、actor和dataflow，但是尽管这些用途将被考虑在内，但是设计任何那些高级的结构并不是这个项目的目标，也不建议新的编程风格或纤程之间那些推荐的交换信息的模式(例如，共享内存vs.消息传递)。由于限制线程的内存访问是其他OpenJDK项目的主题，而且这个问题无论是重量级的还是轻量级的都适用于线程抽象的任何实现，因此这个项目可能会与其他项目发生交叉。

这个项目的目标是向Java平台添加一个轻量级的线程结构——纤程。下面将讨论这个结构可能采用的面向用户的形式。其目标是允许*大多数* Java代码(意思是Java类文件中的代码，不一定是用Java语言编写的)在纤程内不加修改地运行，或者只做最小的修改。允许从Java代码调用的本机代码在纤程中运行不是这个项目的要求，尽管在某些情况下这是可能的。这个项目的目标也不是确保每段代码在纤程中运行时都能获得性能上的好处;事实上，一些不太适合轻量级线程的代码在纤程中运行时可能会影响性能。

这个项目的目标是向Java平台添加一个公开的delimited continuation(或 *coroutine*)结构。然而，这个目标次于纤程(纤程需要continuation，稍后会解释，但这些continuation不一定要作为公共API公开)。

这个项目的目标是实验各种各样的纤程调度器，但进行任何严肃的研究调度器设计不是这个项目的意图，很大程度上是因为我们认为`ForkJoinPool`可以作为一个非常好的纤程调度器。

向JVM添加调用堆栈的操作能力无疑是必需的,也是这个项目添加一个更轻量级的结构,允许在某些点上进行堆栈展开,然后调用一个给定参数的方法(一般是一个泛用的有效尾调用)。我们将该特性称为*unwind-and-invoke*，或UAI。这个项目的目标不是向JVM添加一个自动的尾调用优化。

这个项目可能会涉及到Java平台的不同组件，其特性是这样划分的:

- 在JVM内部实现Continuation和UAI，并且暴露为简洁的Java API
- 纤程很有可能会在JDK的java库中实现，但是可能会需要JVM的帮助
- JDK库中使用会阻塞线程的本地代码（native code）会被适配为可以运行在纤程上的版本。这意味着需要更改`java.io`的类
- JDK库中是使用低层次线程同步（特别是`LockSupport`类），比如`java.util.concurrent`将被适配为可以支持纤程的版本，但是所要的工作量取决于纤程API，而且在任意情况下工作量应该是很小（因为纤程暴露出来的API和线程很相似）
- 调试器，分析器和其他服务性服务需要知道纤程的存在以提供良好的用户体验。这意味着JFR和JVMTI将需要适应纤程的更改，并且可能会添加相关的平台MBean
- 在这一点上，我们没有预见到对Java语言进行更改的需要。

这个项目还处于早期阶段，所以一切——包括它的范围——都可能发生变化

## 术语

由于内核线程和轻量级线程只是同一抽象的不同实现，因此必然会出现一些术语上的混淆。本文将采用以下约定，项目中的每一次通信（correspondence）都应遵循以下约定:

- 单词*thread*只指抽象(稍后将讨论)，而不是特定的实现，所以*thread*可以指抽象的任何实现，无论是由操作系统还是运行时完成。
- 当提到特定的实现时，术语*重量级线程*、*内核线程*和*OS线程*可以互换地用来表示操作系统内核提供的线程的实现。术语*轻量级线程*、*用户模式线程*和*纤程*可以互换地用来表示语言运行时(Java平台中的JVM和JDK库)提供的线程的实现。这些词*并不*指的是特定的Java类(至少在这些API设计不清楚的早期阶段)
- 大写的“Thread”和“Fiber”指的是特定的Java类，主要用于讨论API的设计而不是实现。

## 线程是什么

线程（thread）是按顺序执行的计算机指令序列。当我们正在处理一些操作,其不仅可能涉及计算而且还有可能存在IO操作,暂停,和线程同步——一般来说,指令导致的计算等待一些外部事件——一个线程,有能力*暂停*本身,和自动等待的事件发生时恢复。当一个线程等待时，它应该让出CPU核心，并允许另一个线程运行。

这些功能由两个不同的部分提供。一个*continuation*是一个顺序执行的指令序列，并且可能会暂停自身(更详细的处理将在后面的[continuations](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html#header-n135)小节中给出)。scheduler（调度器）将continuation分配给CPU内核，用一个准备运行的continuation替换一个暂停的continuation，并确保一个准备恢复的continuation最终将被分配给一个CPU内核。因此，线程需要两个结构:continuation和scheduler，尽管这两个不一定单独作为api公开。

同样，线程（thread）(至少在这个上下文中)是一种基本的抽象，并不意味着任何编程范式。特别是，它们仅指允许程序员编写可以运行和暂停的代码序列的抽象，而不是线程之间共享信息的任何机制，如共享内存或传递消息。

因为有两个独立的关注点，我们可以为每个选择不同的实现。目前，Java平台提供的线程结构是`Thread`类，它是由内核线程实现的;它依赖于OS来实现continuation和scheduler。

Java平台公开的continuation结构可以与现有的Java  scheduler(如`ForkJoinPool`、`ThreadPoolExecutor`或第三方的实现)相结合，或者与专门为此目的优化的scheduler相结合，以实现纤程。

还可以在运行时和操作系统之间拆分这两个线程构建块的实现。例如，在Google([video](https://www.youtube.com/watch?v=KXuZi9aeGTw)， [slides](http://www.linuxplumbersconf.org/2013/ocw/system/presentations/1653/original/LPC - User Threading.pdf))中对Linux内核所做的修改，允许用户态代码接管内核线程调度，因此基本上依赖于操作系统来实现continuation，同时也可以使用库来处理调度。这具有用户模式调度所提供的好处，同时仍然允许本地代码在这个线程实现上运行，但它仍然存在相对较高的内存占用和不能调整堆栈大小的缺点，而且目前还不可用。以另一种方式分割实现——操作系统负责调度，运行时提供continuation——似乎没有任何好处，因为它结合了两个世界的最坏情况。

但是为什么用户模式线程会比内核线程更好，为什么它们值得被称为“轻量级”?同样，可以方便地分别考虑continuation和scheduler这两个组件。

为了挂起（suspend）计算，需要一个continuation来存储整个调用堆栈上下文，或者简单地存储堆栈。为了支持本地语言（native language），存储堆栈的内存必须是连续的，并保持在相同的内存地址。虽然虚拟内存确实提供了一些灵活性，但这类内核continuation(即栈)的轻量级和灵活性仍然存在限制。理想情况下，我们希望堆栈根据使用情况增长和收缩。由于线程的语言运行时实现不需要支持任意本机代码，因此我们可以在如何存储continuation方面获得更大的灵活性，从而减少占用空间。

使用操作系统实现的线程的更大的问题是调度器。首先，操作系统调度器在内核模式下运行，因此每当线程阻塞和将控制权返回给调度器时，必须进行一次非廉价的用户/内核切换。另一方面，操作系统调度器被设计成通用的，可以调度许多不同类型的程序线程。但是运行视频编码器的线程与网络服务器请求的线程的行为是非常不同的，同样的调度算法对两者都不是最优的。服务器上处理事务的线程倾向于呈现特定的行为模式，这对通用操作系统调度器构成了挑战。例如，事务服务线程`A`对请求执行某些操作，然后将数据传递给另一个线程`B `进行进一步处理，这是一种常见的模式。这需要一些同步两个线程间的切换,可能涉及一个锁或一个消息队列,但模式是一样的:`A`对一些数据`x`进行一些操作,交给`B`,唤醒`B`然后`A`阻塞到需要处理从网络或者其他线程的请求。这种模式非常常见，我们可以假设`A`在解除`B`的阻塞后不久就会阻塞，所以将`B`和`A`安排在同一个核上将是有益的，因为`x`已经在核的cache中了;此外，向CPU核心本地队列添加`B`并不需要任何代价高昂的竞争同步。事实上，像`ForkJoinPool`这样的工作窃取调度器做出了这个精确的假设，它通过把任务添加到本地队列进行任务调度。然而，操作系统内核不能做出这样的假设。从内核的预测来看，线程`A`可能想要在唤醒`B `之后继续运行一段时间，所以它会把最近未阻塞`B `调度到不同的核心，这样就需要一些同步，并且一旦`B `访问`x`就会导致cache-fault(译者：类于cache缺失引发的一个信号)。

## 纤程

纤程就是我们所说的Java计划提供的用户态线程。本节将列出纤程的要求，并探讨一些设计问题和选项。它并不是详尽的，只是提供了一个设计空间的轮廓，并指出了一些涉及到的挑战。

就基本功能而言，纤程必须与其他线程(轻量级或重量级)并发地运行任意一段Java代码，并允许用户等待它们的停止，参与他们的协作（join them）。显然，必须有一些机制来挂起和恢复纤程，类似于` LockSupport `的 ` park ` /`unpark`。我们还希望获得纤程的堆栈跟踪，以便监控/调试以及它的状态(挂起/运行)等。简而言之，因为纤程是一个线程，它将拥有与重量级线程(由'`Thread `类表示)非常相似的API。关于Java内存模型，纤程的行为将完全像当前的 Thread 实现。虽然纤程将使用jvm管理的continuation来实现，但我们也可能希望它们与OS的continuation兼容，比如Google的用户态调度内核线程。

纤程还有一些独特的功能:我们想要一个纤程有一个可插拔的调度调度器(固定在纤程的结构,或者在它暂停的时候可以被更换,例如有需要调度器作为参数`unpark`方法),我们想让纤程是可串行化的(这在一个单独的章节讨论)。

一般来说，纤程API将与“线程”的API几乎相同，因为抽象是相同的，我们也希望将目前在内核线程中运行的代码可以通过少量修改或者不需要修改就能运行在纤程中。这立即提出了两种设计选择:

1. 将纤程表示为`Fiber`类，并将`Fiber`和`Thread`的通用API分解为一个通用超类型，暂时称为`Strand`。不能直接知道具体实现的线程( Thread-implementation-agnostic )的代码将针对`Strand`进行编程，如果代码在一个纤程中运行，`Strand.currentStrand`将返回一个纤程，而如果代码运行在一个纤程中`Strand.sleep`将挂起纤程
2. 为两种不同线程使用相同的`Thread`类——用户态和内核态——并在调用`start `之前，在构造函数或setter中选择一个作为动态属性集的实现。

一个单独的`Fiber `类可能允许我们更灵活地脱离'`Thread `，但也会带来一些挑战。因为一个用户态调度器没无法直接访问CPU核心,给纤程分配一个核心是由那些运行的内核线程所做的,所以每个纤程至少在被调度到一个CPU核心上的时候对应一个底层的内核线程,尽管底层内核线程的身份并不是固定的,如果调度器决定将相同的纤程调度到不同的工作内核线程，则可能会发生变化。如果调度器是用Java编写的——正如我们所希望的那样——每个纤程甚至都有一个底层的`Thread`实例。如果 `Fiber`类代表了纤程，在纤程中运行的代码则可以访问底层的`Thread`实例(如`Thread.currentThread`或 `Thread.sleep`)，这似乎是不可取的。

如果纤程由相同的`Thread`类表示，那么用户代码将无法访问纤程的底层内核线程，这似乎是合理的，但有许多含义。首先，它需要在JVM中做更多的工作，JVM大量使用`Thread`类，并且需要知道可能的纤程实现。另一方面，这会限制我们的设计灵活性。它在编写调度程序时也会产生一些循环，需要通过将他们分配给线程(内核线程)来*实现*线程(纤程)。这意味着我们需要公开纤程(由`Thread `表示)的continuation，以便调度器使用。

因为纤程由Java调度器所调度,所以它们不必是GC根，因为在任何给定的时间，纤程要么是可运行的，在这种情况下，其调度器持有对它的引用；要么是阻塞的，在这种情况下，阻塞它的对象持有对它的引用（例如锁或IO队列），这样就可以取消阻塞。

另一个相对重要的设计决策涉及线程局部变量。当前，线程本地数据由（`Inheritable`）`ThreadLocal`类表示。如何处理纤程中的线程本地数据？关键的是，`ThreadLocal`有两种截然不同的用法。一种是将数据与线程上下文相关联。纤程可能也需要这种能力。另一个是通过串行化减少并发数据结构中的竞争。使用`ThreadLocal`作为处理器本地（更准确地说，是CPU核心本地）结构的近似值。有了纤程，这两种不同的用途需要清楚地分开，因为现在一个线程本地可能超过数百万个线程（纤程）这根本不是处理器本地数据的良好的近似。将线程作为上下文而不是将线程作为处理器的近似值进行更显式处理的要求不仅限于实际的`ThreadLocal`类，还包括为了串行化而将`Thread`例映射到数据的任何类。如果纤程由`Thread`表示，则需要对这种串行化数据结构进行一些更改。在任何情况下，纤程的添加都需要添加一个显式API来访问处理器，无论是精确的还是近似的。

内核线程的一个重要特性是基于时间片的抢占（为了简洁起见，这里称之为强制抢占）。如果一个内核线程在没有阻塞IO或线程同步的情况下执行运算一段时间，那么它将在一段时间后被强制抢占。乍一看，这似乎是纤程的一个重要设计和实现问题，实际上，我们可能会决定支持这个特性；JVM safepoint特性应该让它变得简单——但是它不仅不重要，而且拥有这个特性根本没有什么区别（所以最好放弃这个特性）。原因如下：与内核线程不同，纤程的数量可能非常大（几十万甚至数百万）。如果*许多*纤程需要如此多的CPU时间，以至于它们需要*经常*被强制抢占，那么当线程数超过内核数几个数量级时，应用程序的资源将不足以进行调度，而且没有任何调度策略可以符合这种情况。如果*许多*纤程*不经常*需要运行长时间的计算，那么一个好的调度器将通过将纤程分配给可用的内核（即工作内核线程）来解决这个问题。如果一些纤程需要频繁地运行长时间的计算，那么最好在重量级线程中运行代码；虽然不同的线程实现提供了相同的抽象，但有时一种实现比另一种更好，而且我们的纤程并不一定在任何情况下都比内核线程更好。

然而，一个真正的实现功能的问题可能是如何协调纤程与会阻塞内核线程的JVM内部的代码。下面的示例是暗含阻塞的代码，比如将类从磁盘加载到使用者指定位置的功能，比如`synchronized`和`Object.wait`。由于纤程调度器将许多纤程多路复用到一小组工作内核线程上，因此阻塞内核线程可能会消耗调度器可用资源的很大一部分，这种情况应该被避免。

在一个极端情况下，每种情况都需要对纤程友好，即如果由纤程调用阻塞API，则只阻塞纤程而不是底层内核线程；另一方面，所有情况都可能继续阻塞底层内核线程。在这两者之间，我们可能会使一些API阻塞纤程，而让另一些API阻塞内核线程。有充分的理由相信，这些情况中的许多可以保持不变，即内核线程阻塞。例如，类加载只在启动期间频繁发生，在启动之后很少发生，并且如上所述，纤程调度器可以轻松地围绕这种阻塞进行调度。*synchronized*的许多用法只在极短的时间内保护内存访问和阻塞线程—如此之短以至于这个问题可以完全忽略。我们甚至可以决定保持*synchronized*不变，并鼓励那些用*synchronized*包围IO访问并以这种方式频繁阻塞的人，如果他们想在纤程中运行代码，就更改代码以使用`j.u.c`（这将是纤程友好的）。类似地，对`Object.wait`的使用，它在现代代码中并不常见（或者我们现在认为是这样），更多的则是使用了`j.u.c`的类似功能。

在任何情况下，阻塞其底层内核线程的纤程都会触发一些可以用JFR/mbean监视的系统事件。

虽然纤程鼓励使用普通、简单和自然的同步阻塞代码，但很容易将现有的异步api改编成纤程阻塞代码。假设库为某个长时间运行的操作`foo`公开了这个异步API，该操作返回一个`String`：

```java
interface AsyncFoo {
   public void asyncFoo(FooCompletion callback);
}
```

其中回调或完成处理器`FooCompletion`的定义如下:

```java
interface FooCompletion {
  void success(String result);
  void failure(FooException exception);
}
```

我们将提供一个异步到纤程阻塞结构，它可能看起来像这样：

```java
abstract class _AsyncToBlocking<T, E extends Throwable> {
    private _Fiber f;
    private T result;
    private E exception;
  
    protected void _complete(T result) {
        this.result = result;
        unpark f
    }
  
    protected void _fail(E exception) { 
        this.exception = exception;
        unpark f
    }
  
    public T run() throws E { 
        this.f = current fiber
        register();
        park
        if (exception != null)
           throw exception;
        return result;
    }
  
    public T run(_timeout) throws E, TimeoutException { ... }
  
    abstract void register();
}
```

然后，我们可以通过首先定义以下类来创建API的阻塞版本：

```java
abstract class AsyncFooToBlocking extends _AsyncToBlocking<String, FooException> 
     implements FooCompletion {
  @Override
  public void success(String result) {
    _complete(result);
  }
  @Override
  public void failure(FooException exception) {
    _fail(exception);
  }
}
```

然后我们使用它将异步API包装为同步版本:

```java
class SyncFoo {
    AsyncFoo foo = get instance;
  
    String syncFoo() throws FooException {
        new AsyncFooToBlocking() {
          @Override protected void register() { foo.asyncFoo(this); }
        }.run();
    }
}
```

我们可以为常见的异步类（如`CompletableFuture`）添加这种扩展

## Continuations

将continuation添加到Java平台的动机是为了实现纤程，但是continuation还有一些其他有趣的用途，因此将continuation作为公共API提供是本项目的第二个目标。然而，这些其他用途的作用预计远低于纤程的作用。事实上，continuation并不能在纤程上增加表现力（也就是说，可以在纤程上实现连续体）。

在本文档和ProjectLoom中的任何地方，*continuation*一词都表示*delimited continuation*（有时也称为*coroutine*[1](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html#dfref-footnote-1)）。在这里，我们将把*delimited continuation*看作可以挂起（自身）和恢复（由调用方恢复）的顺序代码。有些人可能更熟悉将continuation视为表示计算的“剩余”或“未来”的对象（通常是子子程序）的观点。两者描述的是同一件事：一个挂起的continuation，是一个对象，当恢复或“调用”时，它执行计算的剩余部分。

delimited continuation是一个带有入口点（如线程）的continuation子程序，我们称之为*entry point*（在Scheme中，这是*reset point*），它可以在某个点暂停或执行，我们称之为*suspension point*或*yield point*（在Scheme中，这是*shift*point）。当一个delimited continuation挂起时，控制权被传递到continuation的外部，当它被恢复时，控制权返回到最后一个*yield point*，执行上下文一直保存在*entry point*，有许多方法可以表示delimited continuation，但是对于Java程序员来说，下面的例子可以很好的解释这个概念

```java
foo() { // (2)
  ... 
  bar()
  ...
}

bar() {
  ...
  suspend // (3)
  ... // (5)
}

main() {
  c = continuation(foo) // (0)
  c.continue() // (1)
  c.continue() // (4)
}
```

在（0）创建了一个continuation，他的*entry point*是方法`foo`；然后它在（1）处被调用把控制权传递给（2）处的continuation的 entry point，然后它将执行到在子程序`bar`中的下一个挂起点（3），此时返回了（1）处的调用。当continuation在（4）被调用，控制权返回到（5）所在的挂起点

这里讨论的continuation是“stackful”，因为continuation可能会在堆栈的任何嵌套深度处阻塞（在我们的示例中，在函数`bar`内部，该函数由`foo`调用，`foo`是入口点）。相反，stackless continuation只能挂起在与入口点相同的子程序中。此外，这里讨论的continuation是不可重入的，这意味着任何对continuation的调用都可能更改“当前”挂起点。换句话说，continuation对象是有状态的。

实现continuations的主要技术任务——实际上是整个项目的任务——是为HotSpot添加捕获、存储和恢复调用堆栈的能力，而不是作为内核线程的一部分。JNI堆栈帧可能不受支持。

由于continuations是纤程的基础，如果continuation作为公共API公开，我们将需要支持嵌套的continuation，这意味着在continuation内部运行的代码不仅必须能够挂起continuation本身，而且还必须能够挂起封闭的continuation（例如，挂起封闭的纤程）。例如，continuation的一个常见用法是在生成器的实现中。生成器公开一个迭代器，并且每次生成迭代器时，在生成器中运行的代码都会为迭代器生成另一个值。因此，应该可以这样编写代码：

```java
new _Fiber(() -> {
  for (Object x : new _Generator(() -> {
      produce 1
      fiber sleep 100ms
      produce 2
      fiber sleep 100ms
      produce 3
  })) {
      System.out.println("Next: " + x);
  }
})
```

在参考文献中，允许这种行为的嵌套连续体有时被称为“delimited continuations with multiple named prompts”，但我们将其称为*作用域计算续体*。请参阅[该博客](http://blog.paralleluniverse.co/2015/08/07/scoped-continuations/)讨论限定范围连续体的理论表达能力（对那些感兴趣的人来说，continuation是一种“一般效果”，可以用来实现任何效果-例如赋值-即使是在没有其他副作用的纯语言中；这就是为什么在某种意义上，continuation是命令式编程的基本抽象）。

在continuation中运行的代码不应该引用continuation实例，并且作用域通常有一些固定的名称（因此挂起作用域`A`将挂起作用域`A`最内层的封闭continuation）。 但是，让出点（yield point）提供了一种机制，可以将信息从代码传递到continuation实例并返回。 当continuation挂起时，不会触发包围让出点`“try/finally`块（即，在continuation中运行的代码无法意识到到它正在挂起的过程中）。 

将continuation实现为独立的纤程结构（无论它们是否作为公共 API 公开）的原因之一是明确地将关注点分离。 因此，continuation不是线程安全的，并且它们的任何操作都不会创建跨线程的happens-before关系。纤程必须要实现一个职责，即确保将continuation从一个内核线程迁移到另一个内核线程的内存可见性

下面给出了可能的 API 的粗略概述。 Continuations 是一个非常低层次的原语，只会被库作者用来构建更高级别的结构（就像 `java.util.Stream` 实现利用了 `Spliterator`）。 预计使用continuation的类将拥有continuation类的私有实例，甚至更有可能是它的子类，并且continuation实例不会直接暴露给该结构的使用者。 

```java
class _Continuation {
    public _Continuation(_Scope scope, Runnable target) 
    public boolean run()
    public static _Continuation suspend(_Scope scope, Consumer<_Continuation> ccc)
    
    public ? getStackTrace()
}
```

`run` 方法在continuation终止时返回 `true`，如果它挂起则返回 false。 `suspend` 方法允许将信息从让出点传递到continuation（可以使用`ccc`这个回调把信息注入到给定的continuation实例中），并从continuation返回到挂起点（使用可以查询信息的返回值 ，其就是continuation本身）。 

为了演示在continuation方面实现纤程是多么容易，这里是表示纤程的 `_Fiber` 类的部分简单实现。 正如您将注意到的，大部分代码在维护纤程的状态，以确保它不会被同时调度多次： 

```java
class _Fiber {
    private final _Continuation cont;
    private final Executor scheduler;
    private volatile State state;
    private final Runnable task;

    private enum State { NEW, LEASED, RUNNABLE, PAUSED, DONE; }
  
    public _Fiber(Runnable target, Executor scheduler) {
        this.scheduler = scheduler;
        this.cont = new _Continuation(_FIBER_SCOPE, target);
      
        this.state = State.NEW;
        this.task = () -> {
              while (!cont.run()) {
                  if (park0())
                     return; // parking; otherwise, had lease -- continue
              }
              state = State.DONE;
        };
    }
  
    public void start() {
        if (!casState(State.NEW, State.RUNNABLE))
            throw new IllegalStateException();
        scheduler.execute(task);
    }
  
    public static void park() {
        _Continuation.suspend(_FIBER_SCOPE, null);
    }
  
    private boolean park0() {
        State st, nst;
        do {
            st = state;
            switch (st) {
              case LEASED:   nst = State.RUNNABLE; break;
              case RUNNABLE: nst = State.PAUSED;   break;
              default:       throw new IllegalStateException();
            }
        } while (!casState(st, nst));
        return nst == State.PAUSED;
    }
  
    public void unpark() {
        State st, nst;
        do {
            State st = state;
            switch (st) {
              case LEASED: 
              case RUNNABLE: nst = State.LEASED;   break;
              case PAUSED:   nst = State.RUNNABLE; break;
              default:       throw new IllegalStateException();
            }
        } while (!casState(st, nst));
        if (nst == State.RUNNABLE)
            scheduler.execute(task);
    }
  
    private boolean casState(State oldState, State newState) { ... }  
}
```

## 调度器

如上所述，像`ForkJoinPools`这样的工作窃取调度器特别适合调度经常使用IO进行通讯且阻塞 或经常与其他线程通信的线程。 然而，纤程将具有可插拔的调度器，并且用户将能够编写自己的调度器（调度程序的 SPI 可以像`Executor`一样简单）。 根据之前的经验，预计异步模式下的 `ForkJoinPool `可以作为大多数用途的优秀默认纤程调度器，但我们可能还想探索一两个更简单的设计，例如 pinned-scheduler， 总是将给定的纤程调度到特定的内核线程（假定该线程固定到处理器核心）。 

## Unwind-and-Invoke

译者：unwind-and-invoke见上文 此功能用于实现纤程的堆栈恢复

与continuation不同，展开的堆栈帧的内容不会被保留，并且任何对象都不需要实例化这个结构。

TBD

## 其余的挑战

虽然此目标的主要动机是使并发更容易/更具可扩展性， 除了Java 运行时实现的线程以及运行时对其具有更多控制权，还有其他好处。 例如，这样的线程可以在一台机器上暂停和序列化，然后在另一台机器上反序列化和恢复。 这在分布式系统中很有用，在这些系统中，代码可以从更靠近它访问的数据中受益，或者在提供 [function-as-a-service](https://en.wikipedia.org/wiki/Function_as_a_Service) 的云平台中 ，其中运行用户代码的机器实例可以在该代码等待某些外部事件时终止，然后在另一个实例上恢复，可能在不同的物理机器上，从而更好地利用可用资源并降低主机和客户端的成本。 一个纤程将拥有像`parkAndSerialize`和`deserializeAndUnpark`这样的方法。 

由于我们希望纤程是可序列化的，因此continuation也应该是可序列化的。 如果它们是可序列化的，我们不妨让它们可克隆，因为克隆continuation的能力实际上增加了表现力（因为它允许回到以前的暂停点）。 然而，让continuation可以被克隆且对此类用处来说足够好用是一个很困难的挑战，因为 Java 代码在堆栈外存储了大量信息，并且要有用，因此克隆需要以某种可定制的方式“深入”。

## 其他方法

对于并发性的简单性与性能问题的纤程的替代解决方案称为 async/await，并已被 C# 和 Node.js 采用，并且很可能被标准 JavaScript 采用。continuation和纤程在 async/await 中占主导地位，因为 async/await 很容易用continuation来实现（事实上，它可以用一种弱形式的delimited continuation来实现，称为无栈continuation，它不捕获整个调用堆栈，但保存仅单个子程序的本地上下文），反之亦然。

While implementing async/await is easier than full-blown continuations and fibers, that solution falls far too short of addressing the problem. While async/await makes code simpler and gives it the appearance of normal, sequential code, like asynchronous code it still requires significant changes to existing code, explicit support in libraries, and does not interoperate well with synchronous code. In other words, it does not solve what's known as the ["colored function" problem](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).

虽然实现 async/await 比成熟的 continuation 和 Fiber 更容易，但该解决方案远远不能解决问题。 虽然 async/await 使代码更简单，并赋予它正常、顺序代码的外观，就像异步代码一样，它仍然需要对现有代码进行重大更改、库中的显式支持，并且不能与同步代码很好地互操作。 换句话说，它没有解决所谓的 ["colored function" problem](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)

译者：即async/await的侵入性问题和传染性问题，以及不同类型代码不兼容问题

------

1 以后我们称它为 continuation 还是 coroutine 是待定的——虽然意思上有区别，但命名似乎没有完全标准化，continuation 似乎被用作更通用的术语。[↩ .[↩](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html#ref-footnote-1)