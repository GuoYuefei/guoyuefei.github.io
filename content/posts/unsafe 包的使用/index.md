---
title: "Go语言unsafe包详解"
date: 2020-11-15
draft: false
tags: ["golang", "unsafe", "指针", "内存安全"]
categories: [“编程语言"]
keywords: ["golang", "unsafe包", "指针操作", "内存访问", "类型转换", "底层编程", "Go语言不安全编程"]
description: "深入解析Go语言unsafe包的使用方法、应用场景和潜在风险"
---

# unsafe 包的使用

## go指针的限制

1. go的指针不能进行数学运算
2. 不同类型的指针不能相互转换
3. 不同类型的指针不能使用 == 或 != 比较
4. 不同类型的指针变量不能相互赋值

## unsafe 包提供了 2 点重要的能力

1. 任何类型的指针和 unsafe.Pointer 可以相互转换。
2. uintptr 类型和 unsafe.Pointer 可以相互转换。

> ps.  uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。

## unsafe 包可以做什么坏事

**1.绕过私有成员的限制**

可以通过unsafe包绕过私有属性限制对私有属性读写。

```go
package A

type A struct {
	name string
	age int
    mark bool
}


```



```go
package main

import (
    "A"
	"fmt"
    "unsafe"
)

func main() {
    a := A.A{}
    fmt.Println(a)
    
    name := (*string)(unsafe.Pointer(&p))
    age := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Sizeof(string(""))))
    mark := (*bool)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Sizeof(string("")) + unsafe.Sizeof(0)))
    
    *name = "A"
    *age = 20
    *mark = false
    
    // output {A 20 false}
    fmt.Println(a)
}
```



**2.通过伪造快速对私有属性更改** 

这边以内置类型slice为例，可以参考slice的源码。

>  ps. *与本主题无关的提示*：  make得到的slice是实体，make得到的map是指针。

```go
package main

import (
	"fmt"
	"unsafe"
)

// 也可以使用小技巧，构造一个和 slice 一样的结构体来解析或操作切片
type slice struct {
	arrptr unsafe.Pointer
	l int
	c int
}

func main() {
	a := []int{1,2}
	a = append(a, 3)
	length := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a)) + unsafe.Sizeof(unsafe.Pointer(&a))))

	ca := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a)) + unsafe.Sizeof(unsafe.Pointer(&a)) + unsafe.Sizeof(int(0))))

	fmt.Println(*length, *ca)

	*length = *length + 1				// + 1024 没问题， 只要在保护的内存段中就行
	//a = append(a, 4)
	fmt.Println(a, *length, *ca)

	s := *(*slice)(unsafe.Pointer(&a))

	fmt.Println(s)
	fmt.Printf("[%d, %d, %d, %d]\n", *(*int)(s.arrptr), *(*int)(unsafe.Pointer(uintptr(s.arrptr) + unsafe.Sizeof(int(0)))),
		*(*int)(unsafe.Pointer(uintptr(s.arrptr) + 2*unsafe.Sizeof(int(0)))),
		*(*int)(unsafe.Pointer(uintptr(s.arrptr) + 3*unsafe.Sizeof(int(0)))),
		)

}
```

