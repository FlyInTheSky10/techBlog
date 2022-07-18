---
title: MIT6.s081 Lab1: Unix utilities 笔记
date: 2022-07-18 17:38
categories:
- 项目
tags:
- 操作系统
- Mit6s081
---

# Lab 1: Unix utilities

This lab will familiarize you with xv6 and its system calls.

## Boot xv6 (easy)

配置实验环境，配置的方法见 [https://zhuanlan.zhihu.com/p/343655412](https://zhuanlan.zhihu.com/p/343655412)

要使用 Ubuntu 20.04 系统

## sleep (easy)

> Implement the UNIX program sleep for xv6; your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c.

练手题，熟悉一下指令的添加

```c
// sleep.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h" // 必须以这个顺序 include，由于三个头文件有依赖关系

int main(int argc, char **argv) {
    if(argc < 2) {
        printf("usage: sleep <ticks>\n");
    }
    sleep(atoi(argv[1]));
    exit(0);
}
```

<!-- more -->

## pingpong (easy)

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "pid: received ping", where pid is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "pid: received pong", and exit. Your solution should be in the file user/pingpong.c.

学习使用 `pipe()` 和 `fork()` 以及 `wait()` 子进程的方法

这里应该使用两个 `pipe` 的，这里只用一个不太符合规范

```c
// pingpong.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int p[2];
int main(int argc, char **argv) {
 pipe(p);
 int pid = fork();
 if (pid > 0) { // parent
  char buf[100], a = 0;
  write(p[1], &a, sizeof a);
  close(p[1]);
  int x;
  wait(&x);
  int n = read(p[0], buf, sizeof buf);
  if (n > 0) {
   printf("%d: received pong\n", getpid());
  }
  close(p[0]);
 } else if (pid == 0) { // child
  char buf[100], a = 0;
  int n = read(p[0], buf, sizeof buf);
  close(p[0]);
  if (n > 0) {
   printf("%d: received ping\n", getpid());
  }
  write(p[1], &a, sizeof a);
  close(p[1]);
 }
 exit(0);
}
```

## primes ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.

使用多进程和管道的素数筛，见[this page](http://swtch.com/~rsc/thread/)或者[this page]([(101条消息) go并发实现素数筛_kelianlee的博客-CSDN博客](https://blog.csdn.net/leekerian/article/details/114832843))

简单来说有点像流水线，第一个把所有数字都给右边2，然后2再把所有不是2的倍数传给右边3,3又把所有不是5的倍数传给右边....依次类推

然后使用 `pipe` 来传数据，`wait`来等待子程序结束

```c
// primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
void filter(int in, int prime) {
 printf("prime %d\n", prime);
 int x, len;
 while ((len = read(in, &x, sizeof x)) > 0) {
  if (x == 1) break;
  if (x % prime != 0) {
   int p[2];
   pipe(p);
   int pid = fork();
   if (pid == 0) {
    close(p[1]);
    filter(p[0], x);
   }
   close(p[0]);
   while ((len = read(in, &x, sizeof x)) > 0) {
    if (x % prime != 0) write(p[1], &x, sizeof x);
    if (x == 1) break;
   }
   close(in);
   wait(0);
   exit(0);
  }
 }
 close(in);
 wait(0);
 exit(0);
}
int main(int argc, char **argv) {
 int p[2];
 pipe(p);
 int pid = fork();
 if (pid == 0) {
  close(p[1]);
  filter(p[0], 2);
 }
 close(p[0]);
 for (int i = 3; i <= 35; ++i) {
  write(p[1], &i, sizeof i);
 }
 int gg = 1;
 write(p[1], &gg, sizeof gg);
 close(p[1]);
 wait(0);
 exit(0);
}
```

## find ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.

可以直接抄 `ls.c`，修改得到 `find.c`

```c
// find.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* getFileName(char *path) {
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;
  return p;
}

void find(char *path, char *filename)
{
  // printf("hhhhhh: %s\n", path);
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:
    // printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    // printf("hh: %s %s %d\n", fmtname(path), filename, strcmp(fmtname(path), filename));
    if (strcmp(getFileName(path), filename) == 0) {
     printf("%s\n", path);
    }
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      // printf("gg: %s\n", buf);
      if ((strcmp(getFileName(buf), ".") && strcmp(getFileName(buf), ".."))) {
       find(buf, filename);
      }
      // printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}

int main(int argc, char *argv[])
{
  find(argv[1], argv[2]);
  exit(0);
}
```



## xargs ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))

> Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file user/xargs.c.

大概就是个字符串处理题

```c
// xargs.c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/param.h"
#include "kernel/stat.h"
 
static void xargs(int argc, char *argv[]){
    char *cmd = argv[1];
    char *params[MAXARG], buf[512];
    int i;
    /* for Child  execcmd */
    for (i = 1; i < argc; i++) {
        /* 去除argv[]中的xargs */
        params[i - 1] = argv[i];
    }
    params[argc] = 0;
    /* for Child  execcmd */
    
        for(;;) {
  i = 0;
  for(;;) {
  /*  从命令行读取一条指令  */
      int n = read(0, &buf[i], 1);
      if (n == 0 || buf[i] == '\n') break;
      i++;
  }
     
  if (i == 0)
      break;
  buf[i] = 0;
  params[argc - 1] = buf;
  //printf("dgsdg: %s\n", buf);
  /*将命令行输入拼接到指令末尾，之后使用子进程exec执行cmd params 直到再也无法从命令行读取输入*/
  if (fork() == 0) {
      exec(cmd, params);
      exit(0);
  } 
  else {
      wait((int *)0);
  }
        }
}
int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(2, "Usage: %s <command>\n",argv[0]);
        exit(1);
    }
    if (argc + 1 > MAXARG) {
        fprintf(2, "Too many arguments\n");
        exit(1);
    }
    xargs(argc, argv);
    exit(0);
}
```