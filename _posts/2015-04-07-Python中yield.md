---
layout: post
categories: [Python]
description: none
keywords: Python
---
# Python中yield

## 
以前只是粗略的知道yield可以用来为一个函数返回值塞数据，比如下面的例子：
```
def addlist(alist):
    for i in alist:
        yield i + 1
```
取出alist的每一项，然后把i + 1塞进去。然后通过调用取出每一项：
```
alist = [1, 2, 3, 4]
for x in addlist(alist):
    print(x)
```

## 包含yield的函数
假如你看到某个函数包含了yield，这意味着这个函数已经是一个Generator，它的执行会和其他普通的函数有很多不同。比如下面的简单的函数：






























