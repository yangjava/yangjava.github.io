---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# Linux源码shell

## 命令说明
ls等命令不是系统内核固有的，这些命令都是通过coreutils软件包来实现的，当然是在调用shell函数的基础上啦。

## 官方介绍
来自：http://www.gnu.org/software/coreutils/

Coreutils - GNU core utilities
Introduction to Coreutils

The GNU Core Utilities are the basic file, shell and text manipulation utilities of the GNU operating system.
These are the core utilities which are expected to exist on every operating system.

简单来说就是一个软件工具包，里面有各种命令：如ls，mv，cat，touch，mkdir等等的源码实现，这个包依赖系统的shell

## 源码下载
1、通过网页等下载

ftp://alpha.gnu.org/gnu/coreutils/

https://ftp.gnu.org/gnu/coreutils/

2、命令行方式：
```
sudo apt-get source coreutils
```

## 查看源码