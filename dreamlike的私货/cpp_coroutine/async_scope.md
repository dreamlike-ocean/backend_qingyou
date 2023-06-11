# Async Scope

之前的章节只是讲解了协程中的各种概念

本章我们来实现一个async scope，即一个可以在其中使用co_await和co_return的函数

最后我们要在这个基础上实现一个单元素的回调转协程



## Promise实现

这里我们直接把整个Scope建模为一个Task

先看看整体的样式

```cpp
Task<int> simple_task2() {
  std::cout << __func__ << "\n";
  co_return 2;
}
Task<int> simple_task1() {
  std::cout << __func__ << "\n";
  co_return 1;
}

Task<int> simple_task() {
  // result2 == 2

  auto result2 = co_await simple_task1();
  std::cout << __func__ << " co_await task1 \n";

  // result3 == 3
  auto result3 = co_await simple_task2();
  std::cout << __func__ << " co_await task2 \n";

  co_return 1 + result2 + result3;
}

```

这里可以推断出两个要做的事情

* co_await 对应的await_transform允许接受Task作为参数并返回一个Awaiter
* co_return 对应的return_value允许接受T来作为整个任务的结果值

先考虑一个单独异步任务，即我们最常见的那种Promise/Future模型，promise负责支持complete触发回调，future负责挂载回调，那么我们就可以这样考虑：

promise::complete 的实现就是 return_value 的实现

```cpp
  void return_value(ResultType value) {
      done = true;
      res = Result<ResultType>(std::move(value));
      for (auto &callback : completion_callbacks) {
        callback(res);
      }
    }

```

那么下一步的设计就该考虑completion_callbacks放在哪里了

首先按照习惯性设计外部的Future（这里是C++的协程概念中的future，就是promise外面那个）是会持有当前的coroutine_handle的通过这个很容易拿到promise，但是promise是不知道future的，所以我们直接把completion_callbacks放在promise内部即可。

> 这里跟co_await实现倒是没什么关系。。。只是给一个外部可以给task挂载回调的机制罢了

```cpp
 Task &finally(std::function<void()> &&func) {
    handle.promise().on_completed([func](auto result) { func(); });
    return *this;
  }
```

接下来就是暂存结果值和实现三大件了

都很简单

```cpp
struct promise_type {
    auto initial_suspend() { return std::suspend_never{}; }
    auto final_suspend() noexcept { return std::suspend_always{}; }
    auto unhandled_exception() {
      done = true;
      res = Result<ResultType>(std::current_exception());
      for (auto &callback : completion_callbacks) {
        callback(res);
      }
    }
    // 定制co_return行为 当调用这个方法时意味着此时
    void return_value(ResultType value) {
      done = true;
      res = Result<ResultType>(std::move(value));
      for (auto &callback : completion_callbacks) {
        callback(res);
      }
    }

    Task<ResultType> get_return_object() {
      return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
    }

    template <typename _ResultType>
    TaskAwaiter<_ResultType> await_transform(Task<_ResultType> &&task) {
      return TaskAwaiter<_ResultType>(std::move(task));
    }


    void on_completed(std::function<void(Result<ResultType>)> &&func) {
      if (done) {
        // result 已经有值
        auto value = res;
        // 解锁之后再调用 func
        func(value);
      } else {
        // 否则添加回调函数，等待调用
        completion_callbacks.push_back(func);
      }
    }

    auto get_result() { return res.get_or_throw(); }

  public:
    Result<ResultType> res;
    bool done = false;
    std::list<std::function<void(Result<ResultType>)>> completion_callbacks;
  };
```

## Awaitable

剩下的就是awaitable实现，这个主要用于co_await这个操作符的返回值，此时我们需要根据其右侧的状态来决定是否挂起以及返回值了，就是等待子任务·

我们再看一眼这个函数的签名，注意Task是个C++的Future哦

```CPP
template <typename _ResultType>
    TaskAwaiter<_ResultType> await_transform(Task<_ResultType> &&task) {
      return TaskAwaiter<_ResultType>(std::move(task));
    }
```

直接看代码吧 很简单

```cpp
   //因为task是future所以我们有个handle指针再获取到promise再获取到done变量状态 
   bool await_ready() const noexcept {
      return task.handle.promise().done;
    }

//注意这个handle代指的是当前continuation，是调用方的句柄，当我resume时，等价于从co_await处向下执行 而task则是代指的子任务
    void await_suspend(std::coroutine_handle<> handle) noexcept {
      // 当 task 执行完之后调用 resume
      task.finally([handle]() { handle.resume(); });
    }

    // 协程恢复执行时，被等待的子 Task 已经执行完，调用 get_result 来获取结果
    R await_resume() noexcept { return task.get_result(); }

```

简单的一个async scope就完成了

## 回调转协程

这个就更简单啦

我们要实现的结构类似于js中的

```js
fn(args,(i) => {fun1(i)});

fun1(await fn(args)) 
```

### 状态储存

这个就有点像我们Vert.x的Promise/Future实现了

```cpp
template <typename T> struct AsyncResult {
  explicit AsyncResult() = default;

  // 当 Task 正常返回时用结果初始化 Result
  explicit AsyncResult(T &&value)
      : res(value),
        completion_callbacks(std::move(value.completion_callbacks)) {}

  // 这里先暂时一把大锁控制一下 好理解
  void complete(Result<T> &&async_value) {
    std::cout << __func__ << "AsyncResult 准备触发回调 \n";
    auto scope = std::lock_guard(queue_lock);
    done = true;
    res = async_value;
    for (auto fn : completion_callbacks) {
      fn(res);
    }
    completion_callbacks.clear();
  }

  void on_completed(std::function<void(Result<T>)> &&func) {
      auto scope = std::lock_guard(queue_lock);
    if (done) {
      // result 已经有值
      auto value = res;
      // 解锁之后再调用 func
      func(value);
    } else {
      // 否则添加回调函数，等待调用
      completion_callbacks.push_back(func);
    }
  }

  bool is_done() { return done; }

  Result<T> get_result() { return res; }

private:
  std::mutex queue_lock;
  std::list<std::function<void(Result<T>)>> completion_callbacks;
  Result<T> res;
  bool done = false;
};
```

### awaitable

既然要允许await那就得把之前实现的AsyncResult添加点awaitable实现

这里我们采用组合的方式来做

这里是一个多所有权的场景 直接用share_ptr把这个玩意让协程和异步任务共同持有

> 我思考了一下，这里因为存在一定的顺序性 raw ptr也是可行的
>
> 异步任务持有指针不释放
>
> 等待awaitable析构时自然释放即可

```cpp

template <typename T> struct AsyncResultAwaiter {
//省略构造参数
  bool await_ready() const noexcept { return res->is_done(); }

  void await_suspend(std::coroutine_handle<> handle) noexcept {
    // 当 task 执行完之后调用 resume
    res->on_completed([handle](auto a) {
      std::cout << __func__ << "AsyncResultAwaiter  回调 \n";
      std::cout << handle.address();
      handle.resume();
    });
  }

  Result<T> await_resume() noexcept {
    return res.get()->get_result();
  }

  std::shared_ptr<AsyncResult<T>> res;
  std::coroutine_handle<> handle;
};
```

这里就相当于kt的那种做法，直接拿到coroutine_handle来用

最后我们在之前实现的task的promise中添加对它的支持就行了

这里的参数名为future的参数就是与异步任务共享的那个回调，这里只是为了能拿到协程句柄丢到对应的异步任务回调中罢了

```cpp
template <typename _ResultType>
    AsyncResultAwaiter<_ResultType>
    await_transform(std::shared_ptr<AsyncResult<_ResultType>> future) {
      return AsyncResultAwaiter(
          future, std::coroutine_handle<promise_type>::from_promise(*this));
    }
```

用法

```cpp
Task<int> async_task() {
  // result2 == 2
  auto shared_ptr = std::make_shared<AsyncResult<int>>();
  auto thread = std::thread([shared_ptr]() {
    std::cout << "thread :in lambda \n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    shared_ptr->complete(Result(12));
  });
  thread.detach();
  std::cout << "before await count:" << shared_ptr.use_count() << "\n";
  auto res = co_await shared_ptr;
  std::cout << "res:" << res.get_or_throw() << "\n";
  co_return res.get_or_throw() + 3;
}

```

## 总结

这里面我只是实现了 而非打磨好了 其中还有一些内存所有权的问题还没有解决甚至存在内存泄漏

但是先学会再打磨

完整代码参考[codepieces/async_scope_and_future.cpp at main · dreamlike-ocean/codepieces · GitHub](https://github.com/dreamlike-ocean/codepieces/blob/main/async_scope_and_future.cpp)