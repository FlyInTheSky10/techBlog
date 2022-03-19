---
title: Muduo 网络库分析
date: 2022-03-19 21:38
categories:
- Linux
tags:
- Linux
- 计算机网络
- 网络编程
- 技术
---

# 简介

Muduo 是一个 Linux 平台上的 C++ 网络库，由 Chen Shuo 开发，使用 Reactor 模式，多线程+IO复用的一个 TCP 网络库。

使用事件循环的方式来调用事件，代码支持了 C++11，对于学习 C++ 以及 Linux 网络编程非常有用。

Github地址：https://github.com/chenshuo/muduo <!-- more -->

# 架构和概念

Muduo 使用 Reactor 模式。

**One loop per thread， 即一个 EventLoop 唯一对应一个 thread。**

![Reactor模型](../static/images/muduo-reactor.png)

Reactor 模式，就是一个循环，判断监听的**事件 (Event)**是否触发，触发则调用**回调 (Callback)**。

这里的事件包括 Socket 上的读写事件、定时器事件、自定义事件等等，通常用 **fd (File Description)** 来标识事件。

## 核心类

Muduo 有这么几个核心类：

- EventLoop

用于事件循环，`loop()` 函数在其其中

- Poller

用于在 `loop()` 时调用 poll/epoll 来获取当前被触发的事件

- TimerQueue

负责计时器的管理类，Muduo 将计时器化为事件，使其能像 IO 处理一样被处理。

- Channel

仅属于一个 EventLoop，仅处理一个 fd，不被 Poller 所拥有。

Channel 为事件的处理单位，一般对事件的处理都使用其对应的 Channel 处理。Channel 不拥有事件的 fd。

- TcpConnection

一次 TCP 连接对应一个 TcpConnection，储存了 Channel

- Acceptor

在 TcpServer 里，用来接受新的连接。拥有服务端 Socket 对应事件 Channel

- TcpServer

最上层的类，提供直接开始的接口，包含 Acceptor 和所有已连接的 TcpConnection

- EventLoopThreadPool

多线程池类，采用 round-robin 算法分配每个连接该由哪个 Reactor 线程接管。

## 基本架构

### Reactor 处理连接

用户使用 TcpServer 类，TcpServer 类拥有一个服务端 Socket 的 EventLoop，这个 EventLoop 只用于 Acceptor 进行 Socket 连接事件监听。客户端连接该 Socket，Acceptor 回调采用 round-robin 算法分配该 Connection 到线程池中某个建立完毕的 Reactor 线程 (EventLoop)。Connection 在自己所属的线程 (EventLoop) 中注册可读事件 (通过 Channel，回调) 来使得事件循环 (EventLoop、Poller) 能够发现这些触发的事件。当事件循环触发事件，则会在当前线程 (EventLoop) 调用回调 (Callback)。若使用多线程处理事件，则在调用回调处将处理事件随机分配到一个已经建立完毕的事件处理线程池内处理。将定时器事件转化为 IO 事件，则可以将其与其他 IO 事件一并处理。

有一些必须要在主 Reactor 线程中进行的方法 (如注册定时器事件)，可以使用 EventLoop 中定义的 `RunInLoop()`或者`queueInLoop()`来避免竞态，使其不需要加锁使用。

### 连接的断开

连接的断开由于涉及到对象生命周期的影响，比较容易出错且复杂。

Channel 收到从 EventLoop 的回调，开始断开连接。Channel 使用 TcpConnection 注册的 Close Callback (TcpConnection::handleClose) 来注销 TcpConnection。

TcpConnection::handleClose 调用 TcpServer 注册的 Close Callback (TcpServer::removeConnection) 来使 TcpServer 注销当前 TcpConnection。

TcpServer::removeConnection 将 TcpConnection::connectDestroyed 放到 EventLoop 主线程上进行，并且使用 std::bind 延长 TcpConnection 生命周期到其销毁。

TcpConnection::connectDestroyed 调用 EventLoop::removeChannel 删除对应的 Channel，EventLoop::removeChannel 调用 Poller::removeChannel

TcpServer 析构时对每个存在的 TcpConnection 调用 TcpConnection::connectDestroyed

![Reactor模型](../static/images/muduo-close-connection.png)



# 一些设计

## runInLoop 的实现

runInLoop：
```c++
void EventLoop::runInLoop(const Functor& cb)
{
  if (isInLoopThread())
  {
    cb();
  }
  else
  {
    queueInLoop(cb);
  }
}
```

queueInLoop：
```c++
void EventLoop::queueInLoop(const Functor& cb)
{
  {
  MutexLockGuard lock(mutex_); // pendingFunctors_ 暴露在外，需要加锁
  pendingFunctors_.push_back(cb);
  }

  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}
```
这里有几个问题：
- 为什么要唤醒 EventLoop？
- `wakeup` 是怎么实现的?
- `pendingFunctors_`是如何被消费的?

## 为什么要唤醒 EventLoop

我们首先调用了`pendingFunctors_.push_back(cb);`, 将该函数放在`pendingFunctors_`中。EventLoop 的每一轮循环最后会调用 `doPendingFunctors` 依次执行这些函数。

而 EventLoop 的唤醒是通过 `epoll_wait` 实现的，如果此时该 EventLoop 中迟迟没有事件触发，那么 `epoll_wait` 一直就会阻塞。 这样会导致，`pendingFunctors_`迟迟不能被执行了。

所以对 EventLoop 的唤醒是必要的。

## wakeup 是怎么实现的

muduo 这里采用了对 eventfd 的读写来实现对 EventLoop 的唤醒。

在 EventLoop 建立之后，就创建一个 eventfd，并将其可读事件注册到 EventLoop 中。

`wakeup()` 的过程本质上是对这个 `eventfd` 进行写操作，以触发该 `eventfd` 的可读事件。这样就起到了唤醒 `EventLoop` 的作用。

```c++
void EventLoop::wakeup()
{
  uint64_t one = 1;
  sockets::write(wakeupFd_, &one, sizeof one);
}
```
很多库为了兼容 macos，往往使用 pipe 来实现这个功能。muduo 采用了 `eventfd`，性能更好些

## doPendingFunctors的实现

```c++
void EventLoop::doPendingFunctors()
{
  std::vector<Functor> functors;

  callingPendingFunctors_ = true;

  {
  MutexLockGuard lock(mutex_);
  functors.swap(pendingFunctors_);
  }

  for (size_t i = 0; i < functors.size(); ++i)
  {
    functors[i]();
  }
  callingPendingFunctors_ = false;
}
```

从代码可以看出，如果`callingPendingFunctors_`为 false，则说明此时尚未开始执行`doPendingFunctors`函数。 这个有什么作用呢，我们需要结合下 queueInLoop 中，对是否执行`wakeup()`的判断

```c++
if (!isInLoopThread() || callingPendingFunctors_)
{
  wakeup();
}
```

这里还需要结合下 EventLoop 循环的实现，其中`doPendingFunctors()`是**每轮循环的最后一步处理**。 如果调用 queueInLoop 和 EventLoop 在同一个线程，且`callingPendingFunctors_`为false时，则说明：**此时尚未执行到doPendingFunctors()。** 那么此时即使不用`wakeup()`，也可以在之后照旧执行`doPendingFunctors()`了。

这么做的好处非常明显，可以减少对 eventfd 的 IO 读写。

# 基础部分

## Buffer

## Thread

## Mutex、Condition