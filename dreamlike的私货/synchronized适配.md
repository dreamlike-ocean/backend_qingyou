# VitrualThread对于synchronized适配

最新loom的dev mailist公布了一则消息，关于[VitrualThread对于synchronized适配](https://mail.openjdk.org/pipermail/loom-dev/2024-February/006433.html)

的，目前遇到抢objectMonitor失败后不再会阻塞载体线程，但是对于Object::wait和Object::<clinit>的阻塞问题还没有得到解决。

## jvm层适配

对于ObjectMonitor整体的逻辑，抛开避免进入重量级mutex的那些优化来讲，其实跟传统的mutex实现和jdk中常见的aqs并没有太多的区别，这里面我们只关注对于“挂起”与“恢复”的适配，类似于LockSupport::park和LockSupport::unpark的功能。

### ObjectMonitor::enter

这里的适配非常的容易寻找，直接去找LOOM_MONITOR_SUPPORT这个宏即可，这个宏起效果的前提在于`#if defined(AMD64) || defined (AARCH64)`，目前只支持这两个平台，从某种角度来看足够了。

```cpp
#ifdef LOOM_MONITOR_SUPPORT
    ContinuationEntry* ce = current->last_continuation();
    if (ce != nullptr && ce->is_virtual_thread() && current->is_on_monitorenter()) {
        //这里扮演了类似于continuation::yield的功能，让当前的continuation让出了计算资源
      int result = Continuation::try_preempt(current, ce->cont_oop(current));
      if (result == freeze_ok) {
          //在这里进行的park的适配
        bool acquired = HandlePreemptedVThread(current);
        DEBUG_ONLY(int state = java_lang_VirtualThread::state(current->vthread()));
        assert((acquired && current->preemption_cancelled() && state == java_lang_VirtualThread::RUNNING) ||
               (!acquired && !current->preemption_cancelled() && state == java_lang_VirtualThread::BLOCKING), "invariant");
        return true;
      }
    }
#endif
```

类似于aqs里面node的概念，在objectMonitor中对应物为ObjectWaiter，当有线程释放objectMonitor时会选择ObjectWaiter来唤醒对应的线程抢ObjectMonitor。

```cpp
   oop vthread = current->vthread();
  assert(java_lang_VirtualThread::state(vthread) == java_lang_VirtualThread::RUNNING, "wrong state for vthread");
  java_lang_VirtualThread::set_state(vthread, java_lang_VirtualThread::BLOCKING);

  ObjectWaiter* node = new ObjectWaiter(vthread);
  node->_prev   = (ObjectWaiter*) 0xBAD;
  node->TState  = ObjectWaiter::TS_CXQ;
```

这里能够看到ObjectWaiter的构造器函数传入的参数为当前的virtualthread实例，这个重载是这个功能带来的，我们可以仔细看下这个构造器函数和之前的构造器函数有什么区别，parkEvent你可以理解为一个condition的同步原语

```cpp
ObjectWaiter::ObjectWaiter(JavaThread* current) {
  _next     = nullptr;
  _prev     = nullptr;
  _notified = 0;
  _notifier_tid = 0;
  TState    = TS_RUN;
  _thread   = current;
// 这里是新旧两个的最大区别  对应的parkEvent不同，新的构造器对应的parkEvent是一个全局的parkEvent而非是对应线程的parkEvent
  _event    = _thread != nullptr ? _thread->_ParkEvent : ObjectMonitor::vthread_unparker_ParkEvent();
  _active   = false;
  assert(_event != nullptr, "invariant");
}
// 新的构造函数
ObjectWaiter::ObjectWaiter(oop vthread) : ObjectWaiter((JavaThread*)nullptr) {
  _vthread = OopHandle(JavaThread::thread_oop_storage(), vthread);
}
```

现在新的ObjectWaiter也入队了，continuation也yield了，整体的enter就完成了

### ObjectMonitor::exit

由于ObjectMonitor是可重入的，所以这里我们直接跳过递归部分，直接看完全释放ObjectMonitor后的操作，除此之外我再做一个已经有很多vthread在等待的假设，这样方便寻找unpark的适配

```cpp
//寻找到下一个需要被唤醒的节点
    _EntryList = w;
    ObjectWaiter* q = nullptr;
    ObjectWaiter* p;
    for (p = w; p != nullptr; p = p->_next) {
      guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
      p->TState = ObjectWaiter::TS_ENTER;
      p->_prev = q;
      q = p;
    }

    // In 1-0 mode we need: ST EntryList; MEMBAR #storestore; ST _owner = nullptr
    // The MEMBAR is satisfied by the release_store() operation in ExitEpilog().

    // See if we can abdicate to a spinner instead of waking a thread.
    // A primary goal of the implementation is to reduce the
    // context-switch rate.
    if (_succ != nullptr) continue;

    w = _EntryList;
    if (w != nullptr) {
      guarantee(w->TState == ObjectWaiter::TS_ENTER, "invariant");
        //正式的处理在这里
      ExitEpilog(current, w);
      return;
    }
  }
```

对于ExitEpilog正式唤醒对应的线程，要做两件事情，第一件是判断是否为vthread,第二件事情是获取到对应的ParkEvent

在过去的版本实现中，只需要ParkEvent来使用condition唤醒对应线程即可，但是这里为了适配vthread做了一些看起来并不对称的操作。这里的实现是将vthread入队，然后唤醒一个在vthread类初始化时开启的线程来执行submitRunContinuation方法，让对应调度器来重新执行Continuation（这部分后面能看到）

```cpp
void ObjectMonitor::ExitEpilog(JavaThread* current, ObjectWaiter* Wakee) {
  assert(owner_raw() == owner_for(current), "invariant");

  // Exit protocol:
  // 1. ST _succ = wakee
  // 2. membar #loadstore|#storestore;
  // 2. ST _owner = nullptr
  // 3. unpark(wakee)

  oop vthread = nullptr;
  //在vthread的场景下只需要塞入vthread 反之塞入_thread 这两个值二选一且不会同时出现
  if (Wakee->_thread != nullptr) {
    // Platform thread case
    _succ = Wakee->_thread;
  } else {
    assert(Wakee->vthread() != nullptr, "invariant");
    vthread = Wakee->vthread();
    _succ = (JavaThread*)java_lang_Thread::thread_id(vthread);
  }
  ParkEvent * Trigger = Wakee->_event;
  Wakee  = nullptr;
  release_clear_owner(current);
  OrderAccess::fence();
    
  if (vthread == nullptr) {
    // Platform thread case
    Trigger->unpark();
  } else if (java_lang_VirtualThread::set_onWaitingList(vthread, _vthread_cxq_head)) {
      //先入队再唤醒对应的线程
      //看这里这个unpark操作的是全局的那个ParkEvent
    Trigger->unpark();
  }
  // Maintain stats and report events to JVMTI
  OM_PERFDATA_OP(Parks, inc());
}
```

### java_lang_VirtualThread::set_onWaitingList

这里就有点jvm层和java层交界的味道了，简单来说这里就是一个介于jvm和java层之间的一个mpsc阻塞队列（BlockingQueue\<VirtualThread\>），jvm层面向里面塞入vthread对象，java层面的线程不断poll出来vthread来resume

```cpp
bool java_lang_VirtualThread::set_onWaitingList(oop vthread, OopHandle& list_head) {
    //这里是指的VirtualThread中onWaitingList字段来指示是否在队列中
    //0 为不在
  uint8_t* addr = vthread->field_addr<uint8_t>(_onWaitingList_offset);
  uint8_t value = Atomic::load(addr);
  assert(value == 0x00 || value == 0x01, "invariant");
  if (value == 0x00) {
    value = Atomic::cmpxchg(addr, (uint8_t)0x00, (uint8_t)0x01);
    if (value == 0x00) {
      for (;;) {
          //从调用方可得list_head来自于一个全局字段_vthread_cxq_head
        oop head = list_head.resolve();
          //以下两句为头插法入队
        java_lang_VirtualThread::set_next(vthread, head);
        if (list_head.cmpxchg(head, vthread) == head) return true;
      }
    }
  }
  return false; // already on waiting list
}

void java_lang_VirtualThread::set_next(oop vthread, oop next_vthread) {
  vthread->obj_field_put(_next_offset, next_vthread);
}
```

## java层面适配

在VirtualThread中我们能发现在类初始化的时候多了一些代码，而且多了一些字段。整体的逻辑就是通过`takeVirtualThreadListToUnblock`这个函数获取到vthread然后根据这个链依次唤醒对应的vthread

```java
  // has the value 1 when on the list of virtual threads waiting to be unblocked
    private volatile byte onWaitingList;

    // next virtual thread on the list of virtual threads waiting to be unblocked
    private volatile VirtualThread next;

private static void unblockVirtualThreads() {
    while (true) {
        VirtualThread vthread = takeVirtualThreadListToUnblock();
        while (vthread != null) {
            assert vthread.onWaitingList == 1;
            VirtualThread nextThread = vthread.next;

            // remove from list and unblock
            vthread.next = null;
            boolean changed = vthread.compareAndSetOnWaitingList((byte) 1, (byte) 0);
            assert changed;
            vthread.unblock();

            vthread = nextThread;
        }
    }
}

// takes the list of virtual threads that are waiting to be unblocked, waiting if
// necessary until a list becomes available
private static native VirtualThread takeVirtualThreadListToUnblock();

static {
    var unblocker = InnocuousThread.newThread("VirtualThread-unblocker",
            VirtualThread::unblockVirtualThreads);
    unblocker.setDaemon(true);
    unblocker.start();
}

 private void unblock() {
        assert !Thread.currentThread().isVirtual();
        unblocked = true;
        if (state() == BLOCKED && compareAndSetState(BLOCKED, UNBLOCKED)) {
            unblocked = false;
            submitRunContinuation();
        }
    }
```

那么具体就要去看看`takeVirtualThreadListToUnblock`这个native方法是如何实现的了

通过VirtualThread.c中的Java_java_lang_VirtualThread_registerNatives可知，其对应的真实函数为JVM_TakeVirtualThreadListToUnblock。

```cpp
JVM_ENTRY(jobject, JVM_TakeVirtualThreadListToUnblock(JNIEnv* env, jclass ignored))
    //这个就是之前提到的全局的那个vthread_unparker_ParkEvent，会在exit的时候将vthread入队之后再唤醒这个
  ParkEvent* parkEvent = ObjectMonitor::vthread_unparker_ParkEvent();
  assert(parkEvent != nullptr, "not initialized");
//这个也是exit时挂载的那个队头元素，就是我们之前提到的那个全局的“阻塞队列”
  OopHandle& list_head = ObjectMonitor::vthread_cxq_head();
  oop vthread_head = nullptr;
  while (true) {
    if (list_head.peek() != nullptr) {
      for (;;) {
        oop head = list_head.resolve();
          //相当于把整个链摘下来 并把队头返回给java层
        if (list_head.cmpxchg(head, nullptr) == head) {  
          return JNIHandles::make_local(THREAD, head);
        }
      }
    }
    ThreadBlockInVM tbivm(THREAD);
    parkEvent->park();
  }
JVM_END
```



这样整体的兼容就做完了