---
title: MIT6.s081 Lab4: traps 笔记
date: 2022-07-25 22:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 4: traps

This lab explores how system calls are implemented using traps. You will first do a warm-up exercises with stacks and then you will implement an example of user-level trap handling.

中断（traps）一般分为三类：系统调用/自陷（system call）、故障/异常（exception）和硬件中断（interupt），所谓中断，就是指的让CPU停下手边的事情，转而处理特殊事件的机制。

Lab地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/traps.html](https://pdos.csail.mit.edu/6.S081/2020/labs/traps.html)

一份资料：[https://pdos.csail.mit.edu/6.828/2020/lec/l-riscv-slides.pdf](https://pdos.csail.mit.edu/6.828/2020/lec/l-riscv-slides.pdf)

## RISC-V assembly (easy)

> Read the code in call.asm for the functions g, f, and main. The instruction manual for RISC-V is on the reference page. Here are some questions that you should answer (store the answers in a file answers-traps.txt)

<!-- more -->

```
Q: 哪些寄存器存储了函数调用的参数？举个例子，main 调用 printf 的时候，13 被存在了哪个寄存器中？
A: a0-a7; a2;

Q: main 中调用函数 f 对应的汇编代码在哪？对 g 的调用呢？ (提示：编译器有可能会内链(inline)一些函数)
A: 没有这样的代码。 g(x) 被内链到 f(x) 中，然后 f(x) 又被进一步内链到 main() 中

Q: printf 函数所在的地址是？
A: 0x0000000000000628, main 中使用 pc 相对寻址来计算得到这个地址。

Q: 在 main 中 jalr 跳转到 printf 之后，ra 的值是什么？
A: 0x0000000000000038, jalr 指令的下一条汇编指令的地址。

Q: 运行下面的代码
    unsigned int i = 0x00646c72;
    printf("H%x Wo%s", 57616, &i);      
输出是什么？
如果 RISC-V 是大端序的，要实现同样的效果，需要将 i 设置为什么？需要将 57616 修改为别的值吗？
A: "He110 World"; 0x726c6400; 不需要，57616 的十六进制是 110，无论端序（十六进制和内存中的表示不是同个概念）

Q: 在下面的代码中，'y=' 之后会答应什么？ (note: 答案不是一个具体的值) 为什么?
    printf("x=%d y=%d", 3);
A: 输出的是一个受调用前的代码影响的“随机”的值。因为 printf 尝试读的参数数量比提供的参数数量多。
第二个参数 `3` 通过 a1 传递，而第三个参数对应的寄存器 a2 在调用前不会被设置为任何具体的值，而是会
包含调用发生前的任何已经在里面的值。
```

## Backtrace (moderate)

> Implement a backtrace() function in kernel/printf.c. Insert a call to this function in sys_sleep, and then run bttest, which calls sys_sleep. Your output should be as follows:
> backtrace:
> 0x0000000080002cda
> 0x0000000080002bb6
> 0x0000000080002898

添加 backtrace 功能，打印出调用栈，用于调试。

记得在 defs.h 中添加声明

在 riscv.h 中添加获取当前 fp（frame pointer）寄存器的方法：

```c
// riscv.h
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x));
  return x;
}
```

****fp 指向当前栈帧的开始地址，sp 指向当前栈帧的结束地址。**** （栈从高地址往低地址生长，所以 fp 虽然是帧开始地址，但是地址比 sp 高）  
栈帧中从高到低第一个 8 字节 `fp-8` 是 return address，也就是当前调用层应该返回到的地址。  
栈帧中从高到低第二个 8 字节 `fp-16` 是 previous address，指向上一层栈帧的 fp 开始地址。  
剩下的为保存的寄存器、局部变量等。一个栈帧的大小不固定，但是至少 16 字节。  
在 xv6 中，使用一个页来存储栈，如果 fp 已经到达栈页的上界，则说明已经到达栈底。

![寄存器表](../static/images/mit6.s081-lab4-1.png)

实现 backtrace 函数：

```c
// printf.c
void backtrace() {
  uint64 fp = r_fp();
  while(fp != PGROUNDUP(fp)) { // 如果已经到达栈底
    uint64 ra = *(uint64*)(fp - 8); // return address
    printf("%p\n", ra);
    fp = *(uint64*)(fp - 16); // previous fp
  }
}
```

在 sys_sleep 的开头调用一次 backtrace()

```c
// sysproc.c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  backtrace(); // print stack backtrace.

  if(argint(0, &n) < 0)
    return -1;

  // ......

  return 0;
}
```

运行。 

## Alarm (hard)

> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes alarmtest and usertests.

### test0

实现一个定时器

1、Makefile 中添加 alarmtest 指令

```
// Makefile
UPROGS=\
    $U/_alarmtest\
```

2、user/user.h 中放入系统调用声明

```c
// user/user.h
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
```

3、user/usys.pl、kernel/syscall.h、kernel/syscall.c 完成系统调用

```perl
// user/usys.pl
entry("sigalarm");
entry("sigreturn");
```

```c
// kernel/syscall.c
extern uint64 sys_sigalarm(void);
extern uint64 sys_sigreturn(void);
static uint64 (*syscalls[])(void) = {
[SYS_sigalarm]   sys_sigalarm,
[SYS_sigreturn]   sys_sigreturn,
};
```

```c
// kernel/syscall.h
#define SYS_sigalarm  22
#define SYS_sigreturn  23
```

4、proc 中新增字段以及初始化

```c
// proc.c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  int interval_left; // HERE 剩余 ticks
  void (*handler)(); // HERE
};
allocproc(void)
{
  p->interval_left = -1;
  p->handler = 0;
}
```

5、实现系统调用
一定要用 argint、argaddr 拿参数，此时已经转为内核页表，不能直接拿
注意这个 `(void(*)())(handler)` 的转化方法

```c
int sys_sigalarm(void) {
  int ticks;
  if(argint(0, &ticks) < 0) return -1;
  uint64 handler;
  if(argaddr(0, &handler) < 0) return -1; // 注意一定要用 arg... 拿
  struct proc *p = myproc();
  p->interval_left = ticks;
  p->handler = (void(*)())(handler);
  return 0;
}
int sys_sigreturn(void) {
  return 0;
}
```

6、在 trap.c 中触发

我们主要是通过利用中断过程中的 userret 来返回用户态，也就是说需要将保存在 trapframe 的 epc 修改为 handler，从而实现返回用户态时控制流的转变

```c
void
usertrap(void)
{
// give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    // printf("interval_left: %d", p->interval_left);
    if (p->interval_left > 0) p->interval_left--;
    else if (p->interval_left == 0) {
        p->trapframe->epc = (uint64)p->handler; // 触发
        p->interval_left = -1;
    }
    yield();
  }
}
```

执行命令

```shell
$ alarmtest
test0 start
........................alarm!
test0 passed
test0 passed
usertrap(): unexpected scause 0x000000000000000d pid=4
            sepc=0x0000000000000166 stval=0x0000000000003018
```

### test1/2

要处理内存回收问题，以及触发事件结束后回到原来的地址运行，还要防止在 handler 中途闹钟到期再次调用 handler，导致 alarm_trapframe 被覆盖

1、在 proc 结构体的定义中，增加 alarm 相关字段：

```c
// proc.h
struct proc {
  int interval_left;
  int interval;
  int goingoff;
  void (*handler)();
  struct trapframe *origin_trapframe;
};
```

2、修改系统调用

```c
// sysproc.c
int sys_sigalarm(void) {
  int ticks;
  if(argint(0, &ticks) < 0) return -1;
  uint64 handler;
  if(argaddr(0, &handler) < 0) return -1;
  sigalarm(ticks, (void(*)())(handler));
  return 0;
}
int sys_sigreturn(void) {
  return sigreturn();
}
```

3、trap.c 中修改函数

```c
int sigalarm(int ticks, void (*handler)()) {
  struct proc *p = myproc();
  p->interval_left = ticks;
  p->handler = handler;
  p->interval = ticks;
  return 0;
}
int sigreturn(void) {
  struct proc *p = myproc();
  *(p->trapframe) = *(p->origin_trapframe); // 恢复到之前的 trapframe
  p->goingoff = 0; // 事件执行完毕
  return 0;
}
```

4、在 trap.c 中触发

```
// trap.c
void
usertrap(void)
{
  int which_dev = 0;

  ...

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    // printf("interval_left: %d", p->interval_left);
    if (p->interval != 0) { // 有定时器
        p->interval_left--;
        if (p->interval_left <= 0) {
            if (!p->goingoff) { // 无事件未结束
                p->interval_left = p->interval;
                *(p->origin_trapframe) = *(p->trapframe); // 储存原来的 trapframe 方便还原
                p->trapframe->epc = (uint64)p->handler; // 触发
                p->goingoff = 1; // 当前事件未结束
            }
        }
    }
    yield();
  }

  usertrapret();
}
```

5、proc.c

```c
// proc.c
static struct proc*
allocproc(void)
{
  ...
  p->interval_left = 0;
  p->handler = 0;
  p->interval = 0;
  p->goingoff = 0;

  if((p->origin_trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  return p;
}
static void
freeproc(struct proc *p)
{
  ...
  p->interval_left = 0;
  p->handler = 0;
  p->interval = 0;
  if(p->origin_trapframe) kfree((void*)p->origin_trapframe);
  p->origin_trapframe = 0;
  p->goingoff = 0;
}
```

然后运行 `alarmtest`
这里代码 test1 切回来似乎还是有寄存器有问题，使得i, j地址出错，待查

# **从用户空间产生的中断全流程**

ecall（指令，硬件完成）->uservec（汇编）->usertrap（C）->(syscall, devintr, ...)->usertrapret（C）->userret(汇编)->sret(指令，硬件完成)。

### ecall指令

涉及到的寄存器有stvec、sepc、scause、sstatus

执行如下操作

1. 如果陷阱是设备中断，并且状态**SIE**位被清空，则不执行以下任何操作。
2. 清除**SIE**以**禁用中断**。
3. User mode -> supervisor mode
4. Let sepc = pc
5. Let pc = stvec
6. Jump to pc
7. 设置`scause`以反映产生陷阱的原因。
8. 将当前模式（用户或管理）保存在状态的**SPP**位中。

### uservec函数

1. 保存现场（32个通用寄存器）
2. 把内核的page table、内核的stack、当前执行该进程的CPU号装载到寄存器里
3. 跳转到usertrap继续执行

### usertrap函数

1. 分情况，执行系统调用/中断/异常的处理逻辑
2. 修改了stvec的值，还可能会修改sepc的值

### usertrapret函数

1. 填入了trapframe的内容，这样下一次从用户空间转换到内核空间时可以用到这些数据。
2. 恢复stvec、sepc的值（supervisor mode register）

### userret函数

1. 恢复现场
2. 把用户空间的page table、用户空间的stack装载到寄存器里
3. 执行sret指令

### sret指令

1. 程序会切换回user mode
2. SEPC寄存器的数值会被拷贝到PC寄存器（程序计数器）
3. 重新打开中断

# gdb 调试方法

在一个窗口执行

```shell
$ make CPUS=1 qemu-gdb
```

另一个窗口执行

```shell
$ gdb-multiarch kernel/kernel
# (gdb) 进入gdb后执行
set confirm off
set architecture riscv:rv64
set riscv use-compressed-breakpoints yes
target remote localhost:26000
```

gdb 常用

```python
b # 打断点 (e.g.     b main | b *0x30)
c # continue

layout split # view src-code & asm-code

ni # 单步执行汇编(不进函数)
si # 单步执行汇编(有函数则进入函数)
n # 单步执行源码
s # 单步执行源码

p # print
p $a0 # 打印a0寄存器的值
p/x 1536 # 以16进制的格式打印1536
i r a0 # info registers a0
x/i 0x630 # 查看0x630地址处的指令
x/g 0x80000000 # 查看0x80000000地址处的值（g表示值的长度有64位）
```