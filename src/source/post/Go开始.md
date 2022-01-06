---
title: Go 基础及一些注意事项
date: 2021-12-31 14:38
categories:
- Go
tags:
- Go
---

# Go 基础


## 环境变量

### GOROOT

GOROOT 环境变量即 Go 的 SDK 安装目录

### GOPATH

Go 工作目录，`go get` 的东西将会下载于此
推荐使用 `go mod`

## go mod

设置代理：
```
$ go env -w GO111MODULE=on
$ go env -w GOPROXY=https://goproxy.cn,direct
```

设置完后显著提高下载速度，并且进入了 go mod 状态

然后将自己的代码**随意放在一个地方**，结构为`github.com\yourname\yourpath`
然后此时就能将本地包的依赖解决，之后在有 `go.mod` 的地方使用 `go get`或者使用IDEA的智能处理，即可解决所有的依赖问题

如果没有`go.mod`，使用下面代码在代码根目录初始化
```
go mod init 模块名
```
<!-- more -->

https://blog.csdn.net/xuelang532777032/article/details/108594416

https://www.cnblogs.com/terrycc/p/13548803.html

## Go 文件结构

Go 的文件结构即为
```
bin\
pkg\
src\
    github.com\
               yourpath\
    other.com\
               otherpath\
```

Go 包引用使用 github 路径。

# 一些注意点

- Go 的程序入口包一定是 main
- Go 导包，包函数、变量可以写在多个文件里
- Go 会在导包时运行包内的`init()`函数，在`main()`运行之前
