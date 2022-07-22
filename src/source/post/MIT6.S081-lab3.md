---
title: MIT6.s081 Lab3: page tables 笔记
date: 2022-07-20 17:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 3: page tables

In this lab you will explore page tables and modify them to simplify the functions that copy data from user space to kernel space.

Lab地址：[https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html](https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html)

## Print a page table (easy)

> Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this assignment if you pass the pte printout test of make grade.

虚拟地址 (virtual address)，物理地址 (physical address) 对应关系图

![关系图](../static/images/mit6.s081-lab3-1.png)

RISC-V 的逻辑地址寻址采用的是三级页表，9 bit 一级索引找到二级页表，9 bit 二级索引找到三级页表，9 bit 三级索引找到内存页，最低 12 bit 为页内偏移（即一个页 4096 Bytes）。

PPN (Physical Page Number)：物理页号。

仅有叶子结点的页可以读写。<!-- more -->

```
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

本函数需要模拟如上的 CPU 查询页表的过程，对三级页表进行遍历，然后按照一定格式输出

1、kernel/defs.h 中添加函数声明

```c
// kernel/defs.h
......
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
int             vmprint(pagetable_t pagetable); // 添加函数声明
```

2、按照实验要求，在 exec 返回之前打印一下页表

```c
// exec.c

int
exec(char *path, char **argv)
{
  // ......

  vmprint(p->pagetable); // 按照实验要求，在 exec 返回之前打印一下页表。
  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;

}
```

3、在 kernel/vm.c 中定义 vmprint，类似于提示中的 freewalk 程序

```c
void
freeprint(pagetable_t pagetable, uint64 dep) {
// there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if(pte & PTE_V) { // 合法的 PTE
      uint64 child = PTE2PA(pte);
      for (int j = 0; j < dep; ++j) {
          if (j != 0) printf(" ");
          printf("..");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      // 非叶子结点才递归，只有叶子结点才能读写操作
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0) freeprint((pagetable_t)child, dep + 1); // leaf
    }
  }
}

void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  freeprint(pagetable, 1);
}
```

然后运行

```shell
$ ./grade-lab-pgtbl pte printout
make: 'kernel/kernel' is up to date.
== Test pte printout == pte printout: OK (1.5s) 
```

## A kernel page table per process (hard)

> Your first job is to modify the kernel so that every process uses its own copy of the kernel page table when executing in the kernel. Modify struct proc to maintain a kernel page table for each process, and modify the scheduler to switch kernel page tables when switching processes. For this step, each per-process kernel page table should be identical to the existing global kernel page table. You pass this part of the lab if usertests runs correctly.

xv6中用户页表是每个进程一个的，但是默认内核页表是共享的，内核页表结构如下图：

![内核页表结构](../static/images/mit6.s081-lab3-2.png)

每个进程，都有对应自己的 kernel stack，左边的 kstack 即为

本实验即将 kernel pagetable 变为每个进程都拥有的。

1、proc struct 中添加 kernel_pagetable 字段

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

  pagetable_t kernel_pagetable;// kernel pagetable
};
```

2、修改 kvminit()

```c
// vm.c
void
kvminit()
{
  /*kernel_pagetable = (pagetable_t) kalloc();
  memset(kernel_pagetable, 0, PGSIZE);

  // uart registers
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);*/
  // 全部整理到一个新的函数，以供每个进程调用
  kernel_pagetable = kvm_new_pagetable();
}
void
kvm_map_pagetable(pagetable_t pagetable) {
  kvmmap_p(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  kvmmap_p(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  kvmmap_p(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  kvmmap_p(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  kvmmap_p(pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  kvmmap_p(pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  kvmmap_p(pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
pagetable_t
kvm_new_pagetable() {
  pagetable_t pagetable = (pagetable_t) kalloc();
  memset(pagetable, 0, PGSIZE);

  kvm_map_pagetable(pagetable);

  return pagetable;
}
void
kvmmap_p(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{ // 新增一个 kvmmap_p 映射函数
  if(mappages(pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap_p");
}
uint64
kvmpa(pagetable_t pagetable, uint64 va) // 添加了一个参数 pagetable 的翻译函数
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
```

记得 defs.h 中也要添加相应新函数声明。

kvmpa() 参数修改后，virtio_disk.c 里的调用也要修改

```c
// virtio_disk.c
void
virtio_disk_rw(struct buf *b, int write)
{
  ...
  disk.desc[idx[0]].addr = (uint64) kvmpa(myproc()->kernel_pagetable, (uint64) &buf0);
  ...
}
```

然后在 allocproc 中调用

```c
// proc.c
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // kernel pagetable，为内核页表开辟空间并添加一些映射
  p->kernel_pagetable = kvm_new_pagetable();

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

3、修改 procinit，为每个进程都添加一个 kernel stack

由于 xv6 支持多核/多进程调度，同一时间可能会有多个进程处于内核态，所以需要对所有处于内核态的进程创建其独立的内核态内的栈，也就是内核栈，供给其内核态代码执行过程。

xv6 在启动过程中，会在 procinit() 中为所有可能的 64 个进程位都预分配好内核栈 kstack，具体为在高地址空间里，每个进程使用一个页作为 kstack，并且两个不同 kstack 中间隔着一个无映射的 guard page 用于检测栈溢出错误。如上图。

可以将所有进程的内核栈 map 到其**各自内核页表内的固定位置**（不同页表内的同一逻辑地址，指向不同物理内存）。

```c
// proc.c
void
procinit(void)
{
  struct proc *p;

  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      /*char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;*/
      // 移动到分配进程内核页表的地方
  }
  kvminithart();
}
```

```c
// proc.c
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // kernel pagetable
  p->kernel_pagetable = kvm_new_pagetable();

  // 移动于此
  char *pa = kalloc();
  if(pa == 0) panic("kalloc");
  uint64 va = KSTACK((int)0); // 固定位置，填0就好，KSTACK 宏在 memlayout.h 中
  kvmmap_p(p->kernel_pagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

4、修改 scheduler() ，在进程切换的时候切换内核页表。(修改 SATP 寄存器)

```c
// proc.c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // 修改 SATP，修改为要切换的进程，注意代码位置
        w_satp(MAKE_SATP(p->kernel_pagetable));
        sfence_vma(); 

        swtch(&c->context, &p->context);

        // 切回全局内核页表
        kvminithart();

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

5、在 freeproc() 中添加释放内核页表的方法

```c
// proc.c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;

  // 新增内容
  // 先找到内核栈的物理地址，然后kfree
  void *kstack_pa = (void *)kvmpa(p->kernel_pagetable, p->kstack);
  kfree(kstack_pa);

  p->kstack = 0;

  // 递归 free 页表 (除了叶子)
  kvm_free_kernel_pagetable(p->kernel_pagetable);
  p->kernel_pagetable = 0;
  p->state = UNUSED;
}
```

这里使用 freewalk() 类似方法实现 kvm_free_kernel_pagetable，可以不 free 掉叶子页，因为叶子都是一些内核核心页，清除了会系统崩溃

```c
// vm.c
void
kvm_free_kernel_pagetable(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      kvm_free_kernel_pagetable((pagetable_t)child);
      pagetable[i] = 0;
    }
  }
  kfree((void*)pagetable);
}
```

然后运行

```shell
$ ./grade-lab-pgtbl usertests
make: 'kernel/kernel' is up to date.
== Test usertests == (128.6s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
```

## Simplify copyin/copyinstr (hard)

> Replace the body of copyin in kernel/vm.c with a call to copyin_new (defined in kernel/vmcopyin.c); do the same for copyinstr and copyinstr_new. Add mappings for user addresses to each process's kernel page table so that copyin_new and copyinstr_new work. You pass this assignment if usertests runs correctly and all the make grade tests pass.

原来在内核态的 copyin，copyout 函数不能直接解引用用户态指针，现在将他们换成 copyin_new 和 copyinstr_new，需要修改 xv6 内核代码来实现在内核态直接解引用用户态指针。

这样做相比原来 copyin 的实现的优势是，原来的 copyin 是通过软件模拟访问页表的过程获取物理地址的，而在内核页表内维护映射副本的话，可以利用 CPU 的硬件寻址功能进行寻址，效率更高并且可以受快表加速。

基本思路是，内核态维护一个映射表对应于用户态，我们需要在每一处内核对用户页表进行修改的时候，将同样的修改也同步应用在进程的内核页表上，使得两个页表的程序段（0 到 PLIC 段）地址空间的映射同步。

在 PLIC 之前还有一个 CLINT（核心本地中断器）的映射，该映射会与我们要 map 的程序内存冲突。查阅 xv6 book 的 Chapter 5 以及 start.c 可以知道 CLINT 仅在内核启动的时候需要使用到，而用户进程在内核态中的操作并不需要使用到该映射。

那么我们只需要在全局内核页表中映射 CLINT 即可，其他的不需要映射，防止 remap。

1、修改 vm.c 中映射 CLINT 相关

```c
// vm.c
void
kvminit()
{
  kernel_pagetable = kvm_new_pagetable();
  kvmmap_p(kernel_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W); // 新增
}
void
kvm_map_pagetable(pagetable_t pagetable) {
  kvmmap_p(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  kvmmap_p(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  // kvmmap_p(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W); 每个进程就不用都映射 CLINT 了
  kvmmap_p(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  kvmmap_p(pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  kvmmap_p(pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  kvmmap_p(pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
```

2、在 vm.c 中添加相关函数 (defs.h 中需要添加声明)

```c
// vm.c

// 将 src 页表的一部分页映射关系拷贝到 dst 页表中。
// 只拷贝页表项，不拷贝实际的物理页内存。
// 成功返回0，失败返回 -1
int
kvmcopymappings(pagetable_t src, pagetable_t dst, uint64 start, uint64 sz) {
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = PGROUNDUP(start); i < start + sz; i += PGSIZE) {
    if((pte = walk(src, i, 0)) == 0)
      panic("kvmcopymappings: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("kvmcopymappings: page not present");
    pa = PTE2PA(*pte);
    // 不是用户页表，设置权限
    // 必须设置该权限，RISC-V 中内核是无法直接访问用户页的。
    flags = PTE_FLAGS(*pte) & ~PTE_U; 
    if(mappages(dst, i, PGSIZE, pa, flags) != 0){
      goto err;
    }
  }

  return 0;

  err:
  uvmunmap(dst, 0, i / PGSIZE, 0); // 注意是 0，不释放实际内存页
  return -1;
}

// 与 uvmdealloc 功能类似，将程序内存从 oldsz 缩减到 newsz。但区别在于不释放实际内存
// 用于内核页表内程序内存映射与用户页表程序内存映射之间的同步
uint64
kuvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if(newsz >= oldsz)
    return oldsz;

  if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)){
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 0); // 注意是 0，不释放实际内存页
  }

  return newsz;
}
```

3、在 exec 中加入检查，防止程序内存超过 PLIC

```c
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if (sz1 >= PLIC) { // 限制
      goto bad;
    }
    sz = sz1;
}
```

4、同步映射
后面的步骤就是在每个修改到进程用户页表的位置，都将相应的修改同步到进程内核页表中。一共要修改：fork()、exec()、growproc()、userinit()

对于 init 进程，由于不像其他进程，init 不是 fork 得来的，所以需要在 userinit 中也添加同步映射的代码。

```c
// proc.c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  ...

  // Copy user memory from parent to child.
  // 新增，同步映射
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0 ||
     kvmcopymappings(np->pagetable, np->kernel_pagetable, 0, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  ...
}
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  // 此处开始都有修改

  sz = p->sz;
  if(n > 0){
    uint64 newsz;
    if((newsz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    if(kvmcopymappings(p->pagetable, p->kernel_pagetable, sz, n) != 0) {
      uvmdealloc(p->pagetable, newsz, sz); 
      return -1;
    } // 同步
    sz = newsz;
  } else if(n < 0){
    uvmdealloc(p->pagetable, sz, sz + n);
    sz = kuvmdealloc(p->kernel_pagetable, sz, sz + n); // 同步
  }
  p->sz = sz;
  return 0;
}
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;

  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;
  kvmcopymappings(p->pagetable, p->kernel_pagetable, 0, p->sz); // 同步

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```

```c
// exec.c
int
exec(char *path, char **argv)
{

  ...

  // Save program name for debugging.
  ...

  uvmunmap(p->kernel_pagetable, 0, PGROUNDUP(oldsz)/PGSIZE, 0);
  kvmcopymappings(pagetable, p->kernel_pagetable, 0, sz);

  // Commit to the user image.
  ...
}
```

5、替换 copyin、copyinstr 实现

```c
// vm.c

// 声明新函数原型
int copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);
int copyinstr_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);

// 将 copyin、copyinstr 改为转发到新函数
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

然后运行 `make grade`

```shell
== Test pte printout == 
$ make qemu-gdb
pte printout: OK (2.9s) 
== Test answers-pgtbl.txt == answers-pgtbl.txt: FAIL 
    Cannot read answers-pgtbl.txt
== Test count copyin == 
$ make qemu-gdb
count copyin: OK (0.7s) 
== Test usertests == 
$ make qemu-gdb
(115.7s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: FAIL 
    Cannot read time.txt
Score: 60/66
```