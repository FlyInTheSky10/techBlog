---
title: MIT6.s081 Lab2: system calls 笔记
date: 2022-07-18 22:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 2: system calls

In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.

## System call tracing (moderate)

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new trace system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls trace(1 << SYS_fork), where SYS_fork is a syscall number from kernel/syscall.h. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The trace system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

即添加一个系统调用  trace，为每个进程设定一个位 mask，mask 中的位用于要为哪些系统调用输出调试信息。<!-- more -->

### 添加系统调用

1、在 syscall.h 中加入新 system call 的序号

```c
// kernel/syscall.h
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
#define SYS_kill    6
#define SYS_exec    7
#define SYS_fstat   8
#define SYS_chdir   9
#define SYS_dup    10
#define SYS_getpid 11
#define SYS_sbrk   12
#define SYS_sleep  13
#define SYS_uptime 14
#define SYS_open   15
#define SYS_write  16
#define SYS_mknod  17
#define SYS_unlink 18
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
#define SYS_trace  22 // trace
```

2、用 extern 全局声明新的内核调用函数，并且在 syscalls 映射表中，加入从前面定义的编号到系统调用函数指针的映射

```c
// kernel/syscall.c 
extern uint64 sys_chdir(void);
extern uint64 sys_close(void);
extern uint64 sys_dup(void);
extern uint64 sys_exec(void);
extern uint64 sys_exit(void);
extern uint64 sys_fork(void);
extern uint64 sys_fstat(void);
extern uint64 sys_getpid(void);
extern uint64 sys_kill(void);
extern uint64 sys_link(void);
extern uit64 sys_mkdir(void);
extern uint64 sys_mknod(void);
extern uint64 sys_open(void);
extern uint64 sys_pipe(void);
extern uint64 sys_read(void);
extern uint64 sys_sbrk(void);
extern uint64 sys_sleep(void);
extern uint64 sys_unlink(void);
extern uint64 sys_wait(void);
extern uint64 sys_write(void);
extern uint64 sys_uptime(void);
extern uint64 sys_trace(void);   // trace

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_trace]   sys_trace,  // trace
};
```

3、在 usys.pl 中，加入用户态到内核态的跳板函数。

```perl
# user/usys.pl

entry("fork");
entry("exit");
entry("wait");
entry("pipe");
entry("read");
entry("write");
entry("close");
entry("kill");
entry("exec");
entry("open");
entry("mknod");
entry("unlink");
entry("fstat");
entry("link");
entry("mkdir");
entry("chdir");
entry("dup");
entry("getpid");
entry("sbrk");
entry("sleep");
entry("uptime");
entry("trace"); # trace
```

这个脚本在运行后会生成 usys.S 汇编文件，里面定义了每个 system call 的用户态跳板函数

```asm
trace:		# 定用户态跳板函数
li a7, SYS_trace	# 将系统调用 id 存入 a7 寄存器
ecall				# ecall，调用 system call ，跳到内核态的统一系统调用处理函数 syscall()  (syscall.c)
ret

```

系统调用 id 存入 a7 寄存器后，会在之后的 syscall() 里被拿出来判断当前执行哪个系统调用。

4、在用户态的头文件加入定义，使得用户态程序可以找到这个跳板入口函数。

```c
// user/user.h
// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
int kill(int);
int exec(char*, char**);
int open(const char*, int);
int mknod(const char*, short, short);
int unlink(const char*);
int fstat(int fd, struct stat*);
int link(const char*, const char*);
int mkdir(const char*);
int chdir(const char*);
int dup(int);
int getpid(void);
char* sbrk(int);
int sleep(int);
int uptime(void);
int trace(int); // trace
```

5、首先在内核中合适的位置（取决于要实现的功能属于什么模块，理论上随便放都可以，只是主要起归类作用），实现我们的内核调用（在这里是 trace 调用）：

```c
// kernel/sysproc.c
uint64
sys_trace(void)
{
    int mask;

    if(argint(0, &mask) < 0)
	    return -1;

    myproc()->mask = mask;
    return 0;
}
```

内核与用户进程的页表以及寄存器都不通 (指针也不能直接互通访问)，所以参数无法直接通过 C 语言参数的形式传过来，而是需要使用 argaddr、argint、argstr 等系列函数，从进程的 trapframe 中读取用户进程寄存器中的参数。

6、lab 代码

proc 结构体添加一个字段 (每个进程都有一个自己的 proc 结构体)

```c
// kernel/proc.h
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstte state;        // Process state
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
  uint64 mask;        // Mask for syscall tracing (新添加的用于标识追踪哪些 system call 的 mask)
};
```

在 proc.c 中，创建新进程的时候，为新添加的 mask 附上默认值 0（否则初始状态下可能会有垃圾数据）。

```c
// kernel/proc.c
static struct proc*
allocproc(void)
{
  ......

  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  p->mask = 0; // (newly added) 为 mask 设置一个 0 的默认值

  return p;
}
```

修改 fork 函数，使得子进程可以继承父进程的 mask：

```c
// kernel/proc.c
int
fork(void)
{
  ......

  safestrcpy(np->name, p->name, sizeof(p->name));

  np->syscall_trace = p->syscall_trace; // HERE!!! 子进程继承父进程的 syscall_trace

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}


```

所有的系统调用到达内核态后，都会进入到 syscall() 这个函数进行处理

在这里拿 a7 寄存器中的系统调用 id

返回值在 a0 寄存器中

```c
// kernel/syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7; // 拿 a7 寄存器中的系统调用 id
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) { // 如果系统调用编号有效
    p->trapframe->a0 = syscalls[num](); // 通过系统调用编号，获取系统调用处理函数的指针，调用并将返回值存到用户进程的 a0 寄存器中
	// 如果当前进程设置了对该编号系统调用的 trace，则打出 pid、系统调用名称和返回值。
    if((p->mask >> num) & 1) {
      printf("%d: syscall %s -> %d\n",p->pid, syscall_names[num], p->trapframe->a0); // syscall_names[num]: 从 syscall 编号到 syscall 名的映射表
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1; // 返回值在 a0 寄存器
  }
}

const char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};


```

然后就可以运行 trace 系统调用了。

### 系统调用全流程

实现用户态和内核态的良好隔离。

```
user/user.h: 用户态程序调用跳板函数 trace()
user/usys.S: 跳板函数 trace() 使用 CPU 提供的 ecall 指令，调用到内核态
kernel/syscall.c 到达内核态统一系统调用处理函数 syscall()，所有系统调用都会跳到这里来处理。
kernel/syscall.c syscall() 根据跳板传进来的系统调用编号，查询 syscalls[] 表，找到对应的内核函数并调用。
kernel/sysproc.c 到达 sys_trace() 函数，执行具体内核操作
```

## Sysinfo (moderate)

> In this assignment you will add a system call, sysinfo, that collects information about the running system. The system call takes one argument: a pointer to a struct sysinfo (see kernel/sysinfo.h). The kernel should fill out the fields of this struct: the freemem field should be set to the number of bytes of free memory, and the nproc field should be set to the number of processes whose state is not UNUSED. We provide a test program sysinfotest; you pass this assignment if it prints "sysinfotest: OK".

添加一个系统调用，返回空闲的内存、以及已创建的进程数量。

### 获取空闲内存

在内核的头文件中声明计算空闲内存的函数。

```c
// kernel/defs.h
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
uint64 			count_free_mem(void); // here
```

在 kalloc.c 中添加计算空闲内存的函数

```c
// kernel/kalloc.c
uint64
count_free_mem(void) // added for counting free memory in bytes (lab2)
{
  acquire(&kmem.lock); // 必须先锁内存管理结构，防止竞态条件出现
  
  // 统计空闲页数，乘上页大小 PGSIZE 就是空闲的内存字节数
  uint64 mem_bytes = 0;
  struct run *r = kmem.freelist;
  while(r){
    mem_bytes += PGSIZE;
    r = r->next;
  }

  release(&kmem.lock);

  return mem_bytes;
}

```

xv6 中空闲内存页的记录方式是，将空虚内存页**本身**直接用作链表节点，形成一个空闲页链表，每次需要分配，就把链表根部对应的页分配出去。每次需要回收，就把这个页作为新的根节点，把原来的 freelist 链表接到后面。注意这里是**直接使用空闲页本身**作为链表节点，所以不需要使用额外空间来存储空闲页链表

### 获取运行的进程数

在内核的头文件中添加函数声明

```c
// kernel/defs.h
......
void            sleep(void*, struct spinlock*);
void            userinit(void);
int             wait(uint64);
void            wakeup(void*);
void            yield(void);
int             either_copyout(int user_dst, uint64 dst, void *src, uint64 len);
int             either_copyin(void *dst, int user_src, uint64 src, uint64 len);
void            procdump(void);
uint64			count_process(void); // here

```

在 proc.c 中实现该函数

```c
uint64
count_process(void) { // added function for counting used process slots (lab2)
  uint64 cnt = 0;
  for(struct proc *p = proc; p < &proc[NPROC]; p++) {
    // acquire(&p->lock);
    // 不需要锁进程 proc 结构，因为我们只需要读取进程列表，不需要写
    if(p->state != UNUSED) { // 不是 UNUSED 的进程位，就是已经分配的
        cnt++;
    }
  }
  return cnt;
}


```

### 实现 sysinfo 系统调用

添加系统调用的流程与实验 1 类似

```c
// sysproc.c
uint64
sys_sysinfo(void)
{
  // 从用户态读入一个指针，作为存放 sysinfo 结构的缓冲区
  uint64 addr;
  if(argaddr(0, &addr) < 0)
    return -1;
  
  struct sysinfo sinfo;
  sinfo.freemem = count_free_mem(); // kalloc.c
  sinfo.nproc = count_process(); // proc.c
  
  // 使用 copyout，结合当前进程的页表，获得进程传进来的指针（逻辑地址）对应的物理地址
  // 然后将 &sinfo 中的数据复制到该指针所指位置，供用户进程使用。
  // 内核与用户进程页表不通，只能用这个方法
  if(copyout(myproc()->pagetable, addr, (char *)&sinfo, sizeof(sinfo)) < 0)
    return -1;
  return 0;
}


```

在 user.h 提供用户态入口：

```c
// user.h
char* sbrk(int);
int sleep(int);
int uptime(void);
int trace(int);
struct sysinfo; // 这里要声明一下 sysinfo 结构，供用户态使用。
int sysinfo(struct sysinfo *);


```

然后就可以调用 sysinfotest 了