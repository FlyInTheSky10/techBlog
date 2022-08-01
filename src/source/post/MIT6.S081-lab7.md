---
title: MIT6.s081 Lab7: Lab: Multithreading 笔记
date: 2022-07-31 14:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 7: Lab: Multithreading

This lab will familiarize you with multithreading. You will implement switching between threads in a user-level threads package, use multiple threads to speed up a program, and implement a barrier.

## Uthread: switching between threads (moderate)

> Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan. When you're done, make grade should say that your solution passes the uthread test.

实现一个用户态线程库，补全 uthread.c，完成用户态线程功能的实现。

这里的线程相比现代操作系统中的线程而言，更接近一些语言中的协程（coroutine）。原因是这里的线程是完全用户态实现的，多个线程也只能运行在一个 CPU 上，并且没有时钟中断来强制执行调度，需要线程函数本身在合适的时候主动 yield 释放 CPU。这样实现起来的线程并不对线程函数透明，所以比起操作系统的线程而言更接近 coroutine。<!-- more -->

1、从 proc.h 中搬一下 context 结构体，用于保存 ra、sp 以及 callee-saved registers

```c
// uthread.c
struct context { // 新结构体
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context ctx; // 新结构体
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(struct context* old, struct context* new); // 新声明
```

2、实现 thread_switch (从 kernel/swtch.S 中搬过来)

```
// uthread_switch.S
    .text

    /*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

    .globl thread_switch
thread_switch:
    /* YOUR CODE HERE */

    sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)

    ret    /* return to ra */
```

3、在 thread_schedule 中调用 thread_switch 进行上下文切换

```c
// uthread.c
void 
thread_schedule(void)
{
  ...

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch(&t->ctx, &next_thread->ctx); // 切换，并且保存寄存器
  } else
    next_thread = 0;
}
```

4、补齐 thread_create
因为栈的生长是从高地址到低地址，所以sp指向栈最高地址

```c
// uthread.c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.ra = (uint64)func; // 返回地址
  t->ctx.sp = (uint64)&t->stack + (STACK_SIZE - 1); // 栈指针，指向栈最高地址
}
```

## Using threads (moderate)

分析并解决一个哈希表操作的例子内，由于 race-condition 导致的数据丢失的问题。

对每一个桶加锁即可，get与get之间没有影响，不同桶put与put之间也没有影响

```c
pthread_mutex_t locks[NBUCKET];

int
main() {
  for (int i = 0; i < NBUCKET; ++i) pthread_mutex_init(&locks[i], NULL);
  return 0;
}

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&locks[i]);
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&locks[i]);
  }
}
```

## Barrier(moderate)

利用 pthread 提供的条件变量方法，实现同步屏障机制。

线程进入同步屏障 barrier 时，将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数。
如果未达到，则进入睡眠，等待其他线程。
如果已经达到，则唤醒所有在 barrier 中等待的线程，所有线程继续执行；屏障轮数 + 1；

「将已进入屏障的线程数量增加 1，然后再判断是否已经达到总线程数」这一步并不是原子操作，并且这一步和后面的两种情况中的操作「睡眠」和「唤醒」之间也不是原子的，如果在这里发生 race-condition，则会导致出现 「lost wake-up 问题」。

解决方法是，「屏障的线程数量增加 1；判断是否已经达到总线程数；进入睡眠」这三步必须原子。所以使用一个互斥锁 barrier_mutex 来保护这一部分代码。pthread_cond_wait 会在进入睡眠的时候原子性的释放 barrier_mutex，从而允许后续线程进入 barrier，防止死锁。

```c
static void 
barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex);
  if (++bstate.nthread < nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

# 进程切换上下文流程

![进程切换上下文流程1](../static/images/mit6.s081-lab7-1.png)

进程切换是先从当前用户进程转到当前内核进程，然后再转到当前内核中的调度进程，然后再转到目标内核进程，然后再转到目标用户进程。

具体到 xv6 中，进程切换上下文的具体流程 (proc_1 切换到 proc_2)：
![进程切换上下文流程2](../static/images/mit6.s081-lab7-2.png)

### swtch函数

swtch(ctx1, ctx2);

保存**callee寄存器**的内容到ctx1中；

恢复所有的ctx2的内容到**callee寄存器**里。

两个重要的寄存器ra、sp。

- ra：ra寄存器保存了swtch函数的调用点。

- sp：栈指针，指向函数调用栈的顶部（低地址）。