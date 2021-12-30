---
title: Linux C++编程工具
date: 2021-12-27 21:38
categories:
- Linux
tags:
- Linux
---

# G++ 编译 C++

## 编译为可执行文件

将 `fly.cpp` 编译为可执行文件

```
$ g++ fly.cpp
```

编译源代码文件生成对象文件 (object file)，链接对象文件和 libstdc++ 库中的函数得到可执行程序。

## 编译可指定文件名的可执行文件

```
$ g++ fly.cpp -o fly
```

通过 -o 选项指定可执行程序的文件名

## 源文件生成对象文件

将编译源码文件 `fly.cpp` 并生成对象文件 `fly.o`：

```
$ g++ -c fly.cpp
```

选项 -c 用来告诉编译器编译源代码但不要执行链接，输出结果为对象文件。文件默认名与源码文件名相同，只是将其后缀变为 .o。

命令 g++ 也能识别 .o 文件并将其作为输入文件传递给链接器。

```
$ g++ -c fly.cpp
$ g++ fly.o -o fly
```

## 其他

编译选项：

- `-O2`: 开启 O2 优化

- `-std=c++11`: 开启 C++11

# gdb

## 启动

编译（注意加 -g）

```
$ g++ -g fly.cpp -o fly
```

调试

```
$ g++ fly
```

## 交互

### 运行

- `run` 或 `r`: 运行程序到断点 (可用来重新开始运行程序)
- `next` 或 `n`: 下一步 (不会进入到函数体)
- `step` 或 `s`: 下一步 (进入到函数体)
- `continue` 或 `c`: 继续执行到下一个断点
- `quit` 或 `q`: 退出 gdb
- `finish` 或 `fin`: 结束当前运行的程序
- `until`: 运行程序直到退出循环体

### 断点

- `break n` 或 `b n`: 在第 n 行下断点
- `break func` 或 `b func`: 在 func() 入口下断点
- `info b`: 查看当前断点设置情况
- `delete n` 或 `d n`: 删除第 n 个断点
- `clear n`: 删除第 n 行的断点

### 查看

- `print {expression}` 或 `p {expression}`: 输出一个表达式，比如 `p a` 输出变量a的值
- `display {expression}`：调试每次输出表达式的值
- `bt`:  显示堆栈列表
- `layout src`: 分割窗口，可以一边查看代码，一边测试
- Ctrl + L: 上面窗口花了，刷新

注意交互模式下，回车重复上一条指令，可用于方便单步调试

