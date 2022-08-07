---
title: MIT6.s081 Lab9: Lab: file system 笔记
date: 2022-08-06 17:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 9: file system

In this lab you will add large files and symbolic links to the xv6 file system.

## Large files (moderate)

修改 bmap() 函数使得 xv6 文件系统支持的最大文件 blocks 数变为 65803。

xv6文件系统结构
![xv6文件系统结构](../static/images/mit6.s081-lab9-2.png)

inode 结构
![inode结构](../static/images/mit6.s081-lab9-1.png)

- type字段：表明inode是文件还是目录

- nlink字段，也就是link计数器，用来跟踪究竟有多少文件名指向了当前的inode

- size字段，表明了文件数据有多少个字节

- direct block number：这12个block编号指向了构成文件的前12个block。

- indirect block number：它对应了磁盘上一个block，这个block包含了256个block number，这256个block number包含了文件的数据。

原始 xv6 操作系统文件最大的长度是 (256 + 12) * 1024 字节 <!-- more -->

这个实验要求我们将一个direct block number改成doubly-indirect block，即多级，类似于页表那里的多级页表，使得文件最大长度可以增长 256 * 256 个 blocks。

那么就上代码：

首先修改宏

```c
// fs.h
#define ROOTINO  1   // root i-number
#define BSIZE 1024  // block size
#define PSIZE 256 // 页大小，也可以用 NINDIRECT
#define NDIRECT 11 // 变成 11

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // 变为 +2
};
```

```c
// file.h
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+2]; // 记得这里也要修改为 +2
};
```

```c
// fs.c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp, *bp_fir;

  if(bn < NDIRECT){ // 前 11 个 direct block
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT) { // 第 12 块的 singly-indirect block
    // Load singly-indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  // 第 13 块，doubly-indirect block

  bn -= NINDIRECT;

  uint fir = bn / PSIZE; // 第一层索引
  uint sec = bn % PSIZE; // 第二层索引

  if((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);

  // first
  bp_fir = bread(ip->dev, addr);
  a = (uint*)bp_fir->data;
  if((addr = a[fir]) == 0){
    a[fir] = addr = balloc(ip->dev);
    log_write(bp_fir);
  }

  // second
  bp = bread(ip->dev, addr);
  a = (uint*)bp->data;
  if((addr = a[sec]) == 0){
    a[sec] = addr = balloc(ip->dev);
    log_write(bp);
  }
  brelse(bp);
  brelse(bp_fir);
  return addr;
}
void
itrunc(struct inode *ip) // 要能正确释放空间
{
  int i, j, k;
  struct buf *bp, *bp_sec;
  uint *a, *b;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if (ip->addrs[NDIRECT + 1]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){ // 直接套一层就好
      if(a[j]) {
        bp_sec = bread(ip->dev, a[j]);
        b = (uint*)bp_sec->data;
        for(k = 0; k < NINDIRECT; k++) {
          if (b[k]) 
            bfree(ip->dev, b[k]);
        }
        brelse(bp_sec);
        bfree(ip->dev, a[j]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

## Symbolic links ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))

目的：写一个符号链接系统调用：`symlink(char *target, char *path)`可以参考 `ln -s` 的功能。

1、创建一个系统调用 symlink

2、实现 symlink 系统调用，用于创建符号链接。 符号链接与普通的文件一样，需要占用 inode 块。这里使用 inode 中的第一个 direct-mapped 块（1024字节）来存储符号链接指向的文件。

```c
// sysfile.c

uint64
sys_symlink(void)
{
  struct inode *ip;
  char target[MAXPATH], path[MAXPATH];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }

  // use the first data block to store target path.
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }

  iunlockput(ip);

  end_op();
  return 0;
}
```

3、在 fcntl.h 中补齐 O_NOFOLLOW 的定义

```c
// fcntl.h
#define O_RDONLY   0x000
#define O_WRONLY   0x001
#define O_RDWR     0x002
#define O_CREATE   0x200
#define O_TRUNC    0x400
#define O_NOFOLLOW 0x800


```

4、在 stat.h 中补齐 T_SYMLINK 的定义

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#define T_SYMLINK 4
```

5、修改 sys_open，使其在遇到符号链接的时候，可以递归跟随符号链接，直到跟随到非符号链接的 inode 为止。

```c

```