---
title: "Impact of Instruction Optimization and Memory Reordering on Concurrent Programs"
date: 2020-06-30
draft: false
tags: ["golang", "concurrency", "performance-optimization"]
categories: ["Tech"]
keywords: ["golang", "instruction optimization", "memory reordering", "concurrent programs", "memory model", "compiler optimization", "concurrency safety", "reordering"]
description: "In-depth analysis of how instruction optimization and memory reordering affect Golang concurrent programs, exploring compiler optimizations, memory models and ensuring program correctness in optimized environments"
---

# Impact of Instruction Optimization and Memory Reordering on Concurrent Programs

## Problem Introduction

Let's start with some fascinating code that deeply attracted me at first sight:

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

According to the original author, this code wouldn't finish running but would fall into an infinite loop. Of course, the author ran this code in Go 1.9.x, and later versions of Go have been optimized in this regard.

The reason for the infinite loop, as explained by the author:

> The above example starts as many goroutines as the number of CPU cores, each executing an infinite loop.
>
> After creating these goroutines, the main function executes a `time.Sleep(time.Second)` statement. When the Go scheduler sees this statement, it's delighted - it's a perfect opportunity for scheduling. So the main goroutine is scheduled away. The previously created `threads` goroutines perfectly "fill every slot," occupying all M and P.
>
> Inside these goroutines, there are no calls to things like `channel` or `time.sleep` that would trigger the scheduler to work. Trouble arises, and these infinite loops just keep executing.

But in reality, this situation doesn't occur because Go programs still have the g0 goroutine and sysmon background monitoring thread, preventing the above problem. I haven't run this code in Go 1.9.x environment, but interested friends can try it.

Why show this code?

The reason is that this can output `x = 0`. Why is x still 0 after 1 second? Unbelievable.

## Cause Explanation

The main reason this occurs is due to the presence of various levels of cache and store buffer.

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

x will be read from Memory to the store buffer, and then core1 keeps writing to x so that x never gets flushed to L3 Cache or memory. At this point, the core where the main goroutine is located is unaware of core1's modification of x - it keeps reading x as 0 from memory.

## Problem Fix

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

This way, all x++ goroutines will close after the main goroutine issues the close command and flush to memory. This understanding makes sense, but Go seems to do some good things with goroutines containing channel operations, performing some synchronization operations.

Here we start the same number of goroutines as cores. When changing the number of goroutines to 1, the value of x is larger than in the multi-goroutine case!? On my computer, the difference is about an order of magnitude.

The speculated reason is that with single goroutine x++, there's no need to consider write synchronization - just operate on x in the store buffer until it's flushed to memory or L3 Cache before the main goroutine accesses x. With multi-goroutine x++, synchronization issues need to be considered, leading to lower efficiency.

The efficiency difference is quite significant, so we should pay attention to this aspect in future programming, especially when there's frequent reading and writing of a single variable.

> There are other interesting problems that can arise, some of which are difficult to explain. We'll continue exploring when we have the opportunity...