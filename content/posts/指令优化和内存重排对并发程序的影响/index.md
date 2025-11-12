---
title: "指令优化和内存重排对并发程序的影响"
date: 2020-06-30
draft: false
tags: ["golang", "并发编程", "性能优化", "内存安全"]
categories: ["编程语言"]
keywords: ["golang", "指令优化", "内存重排", "并发程序", "内存模型", "编译器优化", "并发安全", "重排序"]
description: "深入分析指令优化和内存重排对Golang并发程序的影响，探讨编译器优化、内存模型以及如何保证并发程序在优化环境下的正确性"
---

# 指令优化和内存重排对并发程序的影响

## 引出问题

先来上码，代码比较神奇，第一眼见他就深深的吸引了我。

```go
package main

import (
  	"fmt"
  	"time"
  	"runtime"
)

func main() {
    var x int
    threads := runtime.GOMAXPROCS(0)
    for i := 0; i < threads; i++ {
        go func() {
            for { x++ }
        }()
    }
    time.Sleep(time.Second)
    fmt.Println("x =", x)
}
```

据作者说，这段代码不会运行结束，而是陷入死循环。当然该作者运行这段代码的时候还是go1.9.x版本，后期go已经得到这方面的优化了。

会陷入死循环的原因，作者的解释是：

> 上面的例子会启动和机器的 CPU 核心数相等的 goroutine，每个 goroutine 都会执行一个无限循环。
>
> 创建完这些 goroutines 后，main 函数里执行一条 `time.Sleep(time.Second)` 语句。Go scheduler 看到这条语句后，简直高兴坏了，要来活了。这是调度的好时机啊，于是主 goroutine 被调度走。先前创建的 `threads` 个 goroutines，刚好“一个萝卜一个坑”，把 M 和 P 都占满了。
>
> 在这些 goroutine 内部，又没有调用一些诸如 `channel`，`time.sleep` 这些会引发调度器工作的事情。麻烦了，只能任由这些无限循环执行下去了。

但事实上并不会出现这种情况，原因是go程序还存在g0协程和sysmon后台监控线程，不至于出现以上的问题。本人没在go1.9.x环境运行过这段代码，有兴趣的朋友可以试下。

为什么上这份代码？

原因是，这个会输出<code>x = 0</code>的情况。为啥1s过去了，x还是0？难以置信。

## 原因解答

其发生的主要原因是存在各级缓存和store buffer。

```
+---------------+             +--------------+
|	  Core1		|             |	   Core2     |
|			    |             |				 |
|_______________|             |______________|	
| x |   |   |   |	          |  |   |   |   |
+---------------+             +--------------+
|   L1 Cache	|             |	  L1 Cache	 |
+---------------+             +--------------+
|   L2 Cache	|             |	  L2 Cache	 |
+---------------+-------------+--------------+
|                   L3 Cache                 |
+--------------------------------------------+
                      |
==================== bus ==================== >
                      |
+---------------------------------------------+
|                                             |
|                    Memory                   |
|                                             |
+---------------------------------------------+
```



x 会在从Memory读取到store buffer，然后core1在一直写x以至于x一直没有能够刷新到L3 Cache或者内存。此时，主协程所在的核心对core1修改x这一行为是无感知的，他从内存中取出的x一直为0.



## 问题修复

```go
package main

import (
	"fmt"
  "time"
  "runtime"
)

func main() {
  var x int
  var done chan bool = make(chan bool)
  threads := runtime.GOMAXPROCS(0)
  for i := 0; i < threads; i++ {
    go func() {
      for {
        select {
        case <- done: return
        default: x++
        }
      }
    }()
  }
  time.Sleep(time.Second)
  close(done)
  fmt.Println("x =", x)
}
```

这是所有的x++协程就会在主协程发出关闭命令后关闭，刷新到内存。其实理解这样没毛病，但是go貌似在含通道操作的协程上做了些好事，做了些同步操作。

这边启动了和核心相同的协程，当将协程数量改至1时，x的值会比多协程的情况下大！？本人电脑尝试出来是相差一个数量级。

猜想原因是单协程x++时，可以不用考虑写同步，只要在store buffer下操作x，直到主协程访问x之前刷新进内存或L3 Cache。而多协程x++时会需要考虑同步问题，导致效率低下。

效率相差比较大，以后编程时可以注意这方面，特别是有频繁的对一个变量读写时。

> 还有引起其他有趣的问题，有些还难以解释，有机会继续探索。。


