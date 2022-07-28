---
title: MIT6.s081 Lab5: xv6 lazy page allocation 笔记
date: 2022-07-28 16:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 5: xv6 lazy page allocation

实现一个内存页懒分配机制，在调用 sbrk() 的时候，不立即分配内存，而是只作记录。在访问到这一部分内存的时候才进行实际的物理内存分配。

本次 lab 分为三个部分，但其实都是属于同一个实验的不同步骤，所以本文将三点集合到一起 <!-- more -->

1、修改 growproc，使得其不再预先分配内存
注意这里要处理 n 为负数的情况，直接调用原函数即可，注意判断缩减后的大小是否合法

```c
// proc.c
int
growproc(int n)
{
  struct proc *p = myproc();

  /*if(n > 0){
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }*/
  if (n < 0) { // 处理 n 为负数的情况
    if (p->sz + n < 0) return -1; // 缩减太大
    uvmdealloc(p->pagetable, p->sz, p->sz + n); // 缩减
  }
  p->sz += n;
  return 0;
}
```

2、修改 usertrap 用户态 trap 处理函数，为缺页异常添加检测，缺页异常（(r_scause() == 13 || r_scause() == 15)）

```c
// trap.c
void
usertrap(void)
{
  ...
  if(r_scause() == 8){
    ...
  } else if((which_dev = devintr()) != 0){
    ...
  } else if (r_scause() == 13 || r_scause() == 15) { // HERE
    uint64 va = r_stval(); // stval 寄存器储存当前需要分配的页的地址
    if (is_lazy_alloc(va)) { // 是因为懒分配导致的缺页异常
        if (lazy_alloc(va) < 0) { 
            p->killed = 1; // 分配内存不成功就杀掉进程
        }
    } else {
        p->killed = 1; // 不是因为懒分配导致的缺页异常，那么直接杀掉
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  ...
}
```

3、编写懒分配代码
注意
如果询问的虚拟地址大于sz，则不要分配地址
如果在 guard page，则不要分配地址 (对应实验最后一个提示)
guard page 在 stack page 下面，在 trapframe 中有 sp 寄存器指向 stack page 的地址，那么就可以通过PGROUNDDOWN(p->trapframe->sp)找到 stack page 最下面的地址，然后下面一个 PGSIZE 大小地址就是 guard page。（解决 usertests 中的 stacktest 失败的问题）

```c
// proc.c
int
is_lazy_alloc(int va) { // 是否因为懒分配导致的缺页异常
  struct proc *p = myproc();
  if (va >= p->sz) { // 如果询问的虚拟地址大于sz，则不是
      return 0;
  }
  if (va <= PGROUNDDOWN(p->trapframe->sp) && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE) { // 如果在 guard page，则不要分配地址 (对应实验最后一个提示)
      return 0;
  }
  return 1;
}

int
lazy_alloc(int va) {
    struct proc *p = myproc();
    //printf("page fault: %p\n", va);
    uint64 ka = (uint64) kalloc();
    if (ka == 0) {
        return -1;
    } else {
        memset((void*)ka, 0, PGSIZE);
        va = PGROUNDDOWN(va);
        if (mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_U|PTE_R) != 0) {
          kfree((void*)ka);
          return -1;
        }
    } // 分配内存
    return 0;
}
```

然后修改一些映射函数
uvmunmap 两处直接 continue，因为不存在的情况是存在的
walkaddr 分配内存
uvmcopy 一处直接 break，不存在地址之后的地址应该也是不存在的了，不需要遍历

```c
// vm.c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;//panic("uvmunmap: walk"); 直接 continue，因为不存在的情况是存在的
    if((*pte & PTE_V) == 0)
      continue;//panic("uvmunmap: not mapped"); 直接 continue，因为不存在的情况是存在的
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0) { // 这三个不能简单返回0，有可能缺页了
      if (is_lazy_alloc(va)) {
          if (lazy_alloc(va) < 0) { // 查询是否因为缺页造成，是就分配内存重新执行本函数
              return 0;
        }
        return walkaddr(pagetable, va);
      }
      return 0;
  }
  pa = PTE2PA(*pte);
  return pa;
}
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      break; //panic("uvmcopy: page not present"); 直接 break
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```