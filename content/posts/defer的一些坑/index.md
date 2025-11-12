---
title: "defer的一些坑"
date: 2020-07-16
draft: false
tags: ["golang", "编程技巧"]
categories: ["编程语言"]
keywords: ["golang", "defer", "defer陷阱", "return执行顺序", "延迟执行", "命名返回值", "闭包捕获", "defer原理"]
description: "深入解析Go语言defer语句的常见陷阱，详细分析return与defer的执行顺序、命名返回值的影响、闭包捕获机制等核心问题，提供避坑指南"
---

# defer的一些坑

## 核心点

**核心1**： defer是在return之前执行的。

**核心2**： return本身不是一条原子操作。

## 坑的例子

```go
package main

import "fmt"

func main() {
	fmt.Println(keng1())
	fmt.Println(keng2())
	fmt.Println(keng3())
	fmt.Println(keng4())
}

/**
	return 0, 不是原子操作，先赋值后返回
	其中赋值与返回之间执行defer内容， 这就是官方资料中说的所谓的在return之前defer
 */
// output 1
func keng1() (ret int) {
	defer func() {
		ret++
	}()
	return 0
}

// output 5
func keng2() (ret int) {
	t := 5
	defer func() {
		t = t + 5
	}()
	return t
}

// output 1
func keng3() (r int) {
	defer func(r int) {
		r += 5
	}(r)
	return 1
}

/**
	不带命名参数
	keng4的情况其实是和keng2一样的， keng4无返回命名，但是返回值的地址空间还是有的
	执行步骤如下
	ret := 0
	result = ret
	defer func() { ret++ }()
	return result
 */
// output 0
func keng4() int {
	ret := 0
	defer func() {
		ret++
	}()
	return ret
}
```

这边有四个坑，无论是哪种坑，只要把defer和return拿出来，做上面核心所说的分析就行了。

如keng1函数

```go
// output 1
func keng1() (ret int) {
	defer func() {
		ret++
	}()
	return 0
}
```

其实可以翻译成

```go
ret = 0
ret++
return			// 这部分return的是ret， so output is 1
```

如keng2函数

```go
// output 5
func keng2() (ret int) {
	t := 5
	defer func() {
		t = t + 5
	}()
	return t
}
```

其实可以翻译成

```go
t := 5
// 以下开始是return拆分，并参入了defer的内容
ret = t
t = t + 5
return 			// 这部分return的是ret， so output is 5
```

keng3函数自行分析，直接跳到keng4函数，为什么说它和keng2是一样的呢？

```go
// output 0
func keng4() int {
	ret := 0
	defer func() {
		ret++
	}()
	return ret
}
```

没有用命名的返回值，但是我们可以认为存在（事实上地址空间还是开辟着的），所以暂且用result表示这块返回区域

```go
ret := 0
// you know
result = ret
ret++
return 				// return 的是result， so output is 0
```

从翻译内容来看，就是keng2的内容。

----
2020-07-29 补充

每次defer语句执行的时候，会把函数“压栈”，函数参数会被拷贝下来；当外层函数（非代码块，如一个for循环）退出时，defer函数按照定义的逆序执行；如果defer执行的函数为nil, 那么会在最终调用函数的产生panic.

## 参考
[Golang之轻松化解defer的温柔陷阱](https://mp.weixin.qq.com/s?__biz=MjM5MDUwNTQwMQ==&mid=2257483686&idx=1&sn=9be2edd3e5cb8202dd34fbd5309c7a50&scene=21#wechat_redirect)