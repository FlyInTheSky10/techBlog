---
title: Linux 网络编程入门
date: 2022-02-13 16:15
categories:
- Linux
tags:
- Linux
- 网络编程
- 计算机网络
- 技术
---

# 项目简介

## 概况

项目：Linux 多线程聊天室服务器
项目地址：[https://github.com/FlyInThesky10/PraticeLinuxServer](https://github.com/FlyInThesky10/PraticeLinuxServer)

## 设计思路

程序使用Reactor模型，并使用多线程提高并发度。为避免线程频繁创建和销毁带来的开销，使用线程池，在程序的开始创建固定数量的线程。使用epoll作为IO多路复用的实现方式。

![Reactor模型](../static/images/reactor.png)

- Treadpool: 多线程采用线程池 (Treadpool)
- Epoll: 采用 ET 模式，并且使用 EPOLLONESHOT 来避免一个 socket 被同时多个线程处理
- Socket: 独立封装，将 ClientSocket 存在 Data 类中进行储存信息
- 使用shared_ptr管理指针，防止内存泄漏

流程：

- (主线程) 通过命令行启动服务器，进入 main.cpp，创建一个 Server 实例
- (主线程) Server 实例中先初始化 ServerSocket 并初始化
- (主线程) 然后创建 Treadpool，初始调用 `wait` 都为睡眠状态
- (主线程) 同时也初始化 Epoll，注册好服务器的 fd 事件，准备监听连接服务器事件
- (主线程) 进行 `epoll_wait`，判断是否存在事件，若有事件则将事件插入 Treadpool 的请求队列 (上锁)，然后 `notify` 子线程
- (子线程) 接到 `notify` 被唤醒，将队列中的事件进行处理 (上锁)

<!-- more -->

# 具体分析

## Socket

ServerSocket，ClientSocket 表在主线程

### SO_REUSEADDR

允许服务器bind一个地址，即使这个地址当前已经存在已建立的连接

```C++
void setReusePort(int fd) {
    int opt = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (const void *)&opt, sizeof(opt));
}
```

### socket, bind, listen, accept, close 与 TCP 三次握手

### 非阻塞 connection

需要 `while(true)` 直到 `(errno == EWOULDBLOCK) || (errno == EAGAIN)` 才不进行 accept 操作

### 设置非阻塞文件描述符

对 fd 设置非阻塞

```C++
int setnonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
```

### recv 与 send



## Epoll

采用 ET 模式，EPOLLONESHOT，存在于主线程

### 原理

使用 `epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &event)` 注册 client fd 的读就绪事件，**主线程**调用 `epoll_wait` 来查看当前是否有新的读就绪事件，有就丢给工作线程处理事件。**所有 client fd 注册为非阻塞**

### ET 模式

与 LT 模式不同，如果有事件 Epoll 只会通知一次，要 `while(true)` 直到 `(errno == EWOULDBLOCK) || (errno == EAGAIN)` 才停止处理 Epoll 事件

### EPOLLONESHOT 保证同一时间一个线程操作一个 socket

注册事件的时候加上 EPOLLONESHOT，但是**每次轮询完后要重新注册一下事件**

## 锁、条件变量

### 锁

```c++
// MutexLock.h
#pragma once

#include <pthread.h>
#include "noncopyable.h"

class MutexLock : public noncopyable {

public:
    MutexLock() {
        pthread_mutex_init(&mutex_, NULL);
    }
    ~MutexLock() {
        pthread_mutex_destroy(&mutex_);
    }
    void lock() {
        pthread_mutex_lock(&mutex_);
    }
    void unlock() {
        pthread_mutex_unlock(&mutex_);
    }
    pthread_mutex_t* getMutex() {
        return &mutex_;
    }
private:
    pthread_mutex_t mutex_;
};

class MutexLockGuard : public noncopyable {

public:
    explicit MutexLockGuard(MutexLock &mutex) : mutex_(mutex) {
        mutex_.lock();
    }
    ~MutexLockGuard() {
        mutex_.unlock();
    }
private:
    MutexLock &mutex_;
};
```

MutexLockGuard 可借助构造函数、析构函数在特定作用域自动上锁、解锁

### 条件变量

```c++
// condition.h
#pragma once

#include <pthread.h>
#include "MutexLock.h"
#include "noncopyable.h"

class Condition : public noncopyable {
public:
    explicit Condition(MutexLock &mutex) : mutex_(mutex) {
        pthread_cond_init(&cond_, NULL);
    }
    ~Condition() {
        pthread_cond_destroy(&cond_);
    }
    void wait() {
        pthread_cond_wait(&cond_, mutex_.getMutex());
    }
    void notify() {
        pthread_cond_signal(&cond_);
    }
    void notifyAll() {
        pthread_cond_broadcast(&cond_);
    }
private:
    MutexLock &mutex_;
    pthread_cond_t cond_;
};
```

需要配合一个互斥锁

wait 前需要取得互斥锁，然后 wait 后相当于先将锁释放，该线程休眠，直到别的线程使用 `notify()` 来唤醒该进程

## Thread

### 上锁问题

涉及到对 Thread 临界区都要上锁。如请求队列的添加和拿取。

### 条件变量用法

在请求队列为空时，将锁让出 (`wait`)，该线程睡眠，等待唤醒处理事件

### 相关 api

#### pthread_join

等待线程退出，通常用于 shutdown

#### pthread_create

调用的方式只能是 `static` 的，可以使用对象指针来调用对象方法

```c++
// in ThreadPool
pthread_create(&threads[i], NULL, worker, this)

// static 方法，想要调用对象的方法 run()
void* ThreadPool::worker(void *args) {
    ThreadPool *pool = static_cast<ThreadPool*>(args);
    if (pool == nullptr) return NULL;

    prctl(PR_SET_NAME,"FlyThread");
    pool->run();
    return NULL;
}
```

## C++

### shared_ptr 智能指针管理内存

当所有引用都销毁后，自动释放指针所指内存

- `shared_ptr.use_count()`： 用于调试查看当前存在几个引用

- `shared_ptr.get()`：用于获取内部c指针的地址，不能滥用

## noncopyable 类

```c++
// noncopyable.h
#pragma once

class noncopyable
{
public:
    noncopyable(const noncopyable&) = delete;
    noncopyable& operator=(const noncopyable&) = delete;
protected:
    noncopyable() = default;
    ~noncopyable() = default;
};
```

## cmake

### CmakeList.txt

### 使用

```
cmake . && make
```

### bind 参数绑定

如果是类成员函数，需要第一个参数绑定自身 `this`

### function 函数模板

主要用于方便存函数
