---
title: Linux 指令
date: 2022-03-29 21:38
categories:
- Linux
tags:
- Linux
- 技术
---

这里是一些常用的 Linux 指令说明。

<!-- more -->

# 进程线程类

## ps

1、`ps -ef` : 显示系统进程信息

```shell
$ ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  1 14:23 ?        00:00:02 /sbin/init splash
root           2       0  0 14:23 ?        00:00:00 [kthreadd]
root           3       2  0 14:23 ?        00:00:00 [rcu_gp]
root           4       2  0 14:23 ?        00:00:00 [rcu_par_gp]
root           5       2  0 14:23 ?        00:00:00 [kworker/0:0-events]
root           6       2  0 14:23 ?        00:00:00 [kworker/0:0H-events_highpri
root           7       2  0 14:23 ?        00:00:00 [kworker/0:1-events]
root           8       2  0 14:23 ?        00:00:01 [kworker/u256:0-events_unbou
root           9       2  0 14:23 ?        00:00:00 [mm_percpu_wq]
root          10       2  0 14:23 ?        00:00:00 [rcu_tasks_rude_]
root          11       2  0 14:23 ?        00:00:00 [rcu_tasks_trace]
fly         4139    2057  0 14:27 ?        00:00:00 /usr/libexec/deja-dup/deja-d
fly         4157    2557  1 14:27 ?        00:00:00 /usr/bin/python3 /usr/bin/gn
fly         4158    4157  1 14:27 ?        00:00:00 /usr/bin/gnome-terminal.real
fly         4163    1955  7 14:27 ?        00:00:00 /usr/libexec/gnome-terminal-
fly         4181    4163  0 14:27 pts/0    00:00:00 bash
fly         4187    4181  0 14:27 pts/0    00:00:00 ps -ef
```

2、`ps -aux`：显示系统进程信息

```shell
$ ps -aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  1.1  0.3 166140 12368 ?        Ss   14:23   0:02 /sbin/init sp
root           2  0.0  0.0      0     0 ?        S    14:23   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   14:23   0:00 [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   14:23   0:00 [rcu_par_gp]
root           5  0.0  0.0      0     0 ?        I    14:23   0:00 [kworker/0:0-
root           6  0.0  0.0      0     0 ?        I<   14:23   0:00 [kworker/0:0H
root           7  0.0  0.0      0     0 ?        I    14:23   0:00 [kworker/0:1-
root           8  0.4  0.0      0     0 ?        I    14:23   0:01 [kworker/u256
root           9  0.0  0.0      0     0 ?        I<   14:23   0:00 [mm_percpu_wq
root          10  0.0  0.0      0     0 ?        S    14:23   0:00 [rcu_tasks_ru
root          11  0.0  0.0      0     0 ?        S    14:23   0:00 [rcu_tasks_tr
fly         4139  0.1  0.3 429904 12276 ?        Sl   14:27   0:00 /usr/libexec/
fly         4157  0.2  0.4  37468 19584 ?        S    14:27   0:00 /usr/bin/pyth
fly         4158  0.2  0.6 383988 27424 ?        Sl   14:27   0:00 /usr/bin/gnom
fly         4163  3.7  1.2 561840 50056 ?        Rsl  14:27   0:00 /usr/libexec/
fly         4181  0.0  0.1  14444  5704 pts/0    Ss   14:27   0:00 bash
fly         4188  0.0  0.1  15840  4012 pts/0    R+   14:28   0:00 ps -aux
```

`aux` 是 BSD 风格，`-ef`是 System V 风格。

`aux` 会截断 command 列，而 `-ef` 不会。

3、`ps j`：查看会话、进程组

```shell
$ ps j
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
   1898    2002    2002    2002 tty2        2002 Ssl+  1000   0:00 /usr/libexec/gdm-w
   2002    2005    2002    2002 tty2        2002 Sl+   1000   0:00 /usr/libexec/gnome
   4163    4181    4181    4181 pts/0       4280 Ss    1000   0:00 bash
   4181    4280    4280    4181 pts/0       4280 R+    1000   0:00 ps j
```

## kill

发送信号

1、`kill PID`：杀死 PID 进程

2、`kill -l`： 信号列表

3、`kill -n PID `：发送指定信号

## pstree

显示进程树。

```shell
$ pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager───2*[{NetworkManager}]
        ├─VGAuthService
        ├─dockerd───25*[{dockerd}]
        ├─fwupd───4*[{fwupd}]
        ├─gdm3─┬─gdm-session-wor─┬─gdm-wayland-ses─┬─gnome-session-b
        │      │                 │                 └─2*[{gdm-wayland-ses}]
        │      │                 └─2*[{gdm-session-wor}]
        │      └─2*[{gdm3}]
        ├─gnome-keyring-d───3*[{gnome-keyring-d}]
        ├─irqbalance───{irqbalance}
        ├─2*[kerneloops]
        ├─networkd-dispat
        ├─packagekitd───2*[{packagekitd}]
        ├─polkitd───2*[{polkitd}]
        ├─power-profiles-───2*[{power-profiles-}]
        ├─rsyslogd───3*[{rsyslogd}]
        ├─rtkit-daemon───2*[{rtkit-daemon}]
```

## top



# 其他

## grep

使用管道输入，可以查找文字所在行返回

例：`ps -ef | grep bash`

## more

使输出更加详细

例：`ps j | more`