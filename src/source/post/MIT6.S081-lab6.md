---
title: MIT6.s081 Lab6: Copy-on-Write Fork for xv6 笔记
date: 2022-07-30 16:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 6: Copy-on-Write Fork for xv6

实现 fork 懒复制机制，在进程 fork 后，不立刻复制内存页，而是将虚拟地址指向与父进程相同的物理地址。在父子任意一方尝试对内存页进行修改时，才对内存页进行复制。 物理内存页必须保证在所有引用都消失后才能被释放，这里需要有引用计数机制。

本实验实现部分有点像智能指针的引用计数。

1、首先修改 uvmcopy()，在复制父进程的内存到子进程的时候，不立刻复制数据，而是建立指向原物理页的映射，并将父子两端的页表项都设置为不可写。

这时候如果尝试修改懒复制的页，会出现 page fault 被 usertrap() 捕获。<!-- more -->

```c
// vm.c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  //char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    *pte = (*pte & ~PTE_W) | PTE_COW; // fixed: 使用这个方法，要记得修改父页面的位
    flags = PTE_FLAGS(*pte);
    //if((mem = kalloc()) == 0)
    //  goto err;
    //memmove(mem, (char*)pa, PGSIZE);
    //printf("pte: %p\n", *pte | flags);
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      //kfree(mem);
      goto err;
    }
    krefpage((void*)pa); // 该物理地址计数加1
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

2、上面用到了 PTE_COW 标志位，用于标示一个映射对应的物理页是否是懒复制页。这里 PTE_COW 需要在 riscv.h 中定义：

```c
// kernel/riscv.h
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_COW (1L << 9) // 是否为懒复制页，使用页表项 flags 中保留的第 8 位表示
// （页表项 flags 中，第 8、9、10 位均为保留给操作系统使用的位，可以用作任意自定义用途）
```

3、在 usertrap() 中添加对 page fault 的检测，并在当前访问的地址符合懒复制页条件时，对懒复制页进行实复制操作

```c
// trap.c
if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if ((r_scause() == 13 || r_scause() == 15) && uvmcheckcowpage(r_stval())) { 
    // page fault, 13、15 并且是因为 cow 的原因缺页
    if(uvmcowcopy(r_stval()) == -1) { // 分配一块新的内存地址完成复制
      p->killed = 1;
    }
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

4、同时 copyout() 由于是软件访问页表，不会触发缺页异常，所以需要手动添加同样的监测代码，检测接收的页是否是一个懒复制页，若是，执行实复制操作：

```c
// vm.c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  if(uvmcheckcowpage(dstva)) uvmcowcopy(dstva);

  ...
}
```

5、实现懒复制页的检测（uvmcheckcowpage()）与实复制（uvmcowcopy()）操作：

```c
// vm.c
int
uvmcheckcowpage(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();

  return va < p->sz // 在进程内存范围内
    && ((pte = walk(p->pagetable, va, 0)) != 0)
    && (*pte & PTE_V) // 页表项存在
    && (*pte & PTE_COW); // 页是一个懒复制页
}
int
uvmcowcopy(uint64 va) {
  pte_t *pte;
  struct proc *p = myproc();

  if((pte = walk(p->pagetable, va, 0)) == 0)
    panic("uvmcowcopy: walk");

  uint64 pa = PTE2PA(*pte); // 用 PTE2PA 而不是 walkaddr! walkaddr 有可能取到 0 地址
  uint64 new = (uint64)kcopy_n_deref((void*)pa); // 分配一块新的内存地址

   if (new == 0) return -1;

   uint64 flags = (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW; // 消除标记
   uvmunmap(p->pagetable, PGROUNDDOWN(va), 1, 0);  // 注意第三个参数是页数，并且不要清除实际内存页
   if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, new, flags) != 0){ // 注意第三个参数是字节数
     panic("uvmcowcopy: mappages");
   }
   return 0;
}
```

6、物理页生命周期以及引用计数

一个物理页的生命周期内，现在需要支持以下操作：

- kalloc(): 分配物理页，将其引用计数置为 1
- krefpage(): 创建物理页的一个新引用，引用计数加 1
- kcopy_n_deref(): 将物理页的一个引用实复制到一个新物理页上（引用计数为 1），返回得到的副本页；并将本物理页的引用计数减 1
- kfree(): 释放物理页的一个引用，引用计数减 1；如果计数变为 0，则释放回收物理页

定义一些宏，以及引用计数数组
修改原来的一些函数，添加两个函数

```c
// kalloc.c
#define PA2PGREF_ID(p) (((p)-KERNBASE)/PGSIZE)
#define PGREF_MAX_ENTRIES PA2PGREF_ID(PHYSTOP)

struct spinlock pgreflock; // 用于 pageref 数组的锁，防止竞态条件引起内存泄漏
int pageref[PGREF_MAX_ENTRIES]; // 从 KERNBASE 开始到 PHYSTOP 之间的每个物理页的引用计数

#define PA2PGREF(p) pageref[PA2PGREF_ID((uint64)(p))]

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&pgreflock, "pgref"); // 添加初始化
  freerange(end, (void*)PHYSTOP);
}
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&pgreflock); // 拿锁
  if (--PA2PGREF(pa) <= 0) { // 没有引用实例，可以 free
      // Fill with junk to catch dangling refs.
      memset(pa, 1, PGSIZE);

      r = (struct run*)pa;

      acquire(&kmem.lock);
      r->next = kmem.freelist;
      kmem.freelist = r;
      release(&kmem.lock);
  }
  release(&pgreflock);
}
void *
kalloc(void)
{ // 此处无需拿锁
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    PA2PGREF(r) = 1; // 添加引用计数
  }

  return (void*)r;
}

// 增加物理地址 pa 的引用计数
void
krefpage(void *pa) {
  acquire(&pgreflock);
  PA2PGREF(pa)++;
  release(&pgreflock);
}
// 将物理页的一个引用实复制到一个新物理页上（引用计数为 1），返回得到的副本页；并将本物理页的引用计数减 1
void*
kcopy_n_deref(void *pa) {
  acquire(&pgreflock);

  if (PA2PGREF(pa) <= 1) { // 仅有一个引用，直接返回本实例
    release(&pgreflock);
    return pa;
  }

  uint64 newpa = (uint64)kalloc(); // 分配新的物理页
  if(newpa == 0) {
    release(&pgreflock);
    return 0;
  }
  memmove((void*)newpa, (void*)pa, PGSIZE);

  PA2PGREF(pa)--; // 将本物理页引用计数减 1

  release(&pgreflock);
  return (void*)newpa;
}
```

执行测试

```shell
$ make grade
(9.3s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 
$ make qemu-gdb
(154.6s) 
    (Old xv6.out.usertests failure log removed)
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
```