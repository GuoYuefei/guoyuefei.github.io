---
title: "接口类型值和接口值问题，包括nil值问题"
date: 2020-07-29
draft: false
tags: ["golang", "接口", "类型系统"]
categories: ["技术"]
keywords: ["golang", "接口类型", "接口值", "nil值问题", "Go接口", "类型断言", "空接口", "接口比较"]
description: "深入解析Golang中接口类型值和接口值的相关问题，特别是nil值在接口中的特殊行为和陷阱，涵盖类型断言、空接口处理等核心概念"
---

# 接口类型值和接口值问题，包括nil值问题

## 引言

先来看一个错误的程序

```go
const debug = true
func main() {
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    f(buf) // NOTE: subtly incorrect!
    if debug {
        // ...use buf...
    }
}
// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    // ...do something...
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
```

当 <code>debug=true</code> ，开启状态是没有问题的。

但当 <code>debug=false</code> ，运行时就会出现空指针问题，<code>out.Write([]byte("done!\n"))</code>

原因在于out接口其动态类型为 <code>*bytes.Buffer</code>, 但是其接口值为nil。 ---> 原因是： 实参未被赋值，而是这个类型的空值

而nil的类型的动态类型和值都是nil，所以，<code>nil != out</code> 是为 <code>true</code> 的。

-----------

## iface and eface

可以从源码层面看下

普通接口配型有iface类型定义， 空接口为eface

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    hash   uint32 // copy of _type.hash. Used for type switches.
    bad    bool   // type does not implement interface
    inhash bool   // has this itab been added to hash?
    unused [2]byte
    fun    [1]uintptr // variable sized
}
```

以上为iface的定义， `iface` 内部维护两个指针，`tab` 指向一个 `itab` 实体， 它表示接口的类型以及赋给这个接口的实体类型。`data` 则指向接口具体的值，一般而言是一个指向堆内存的指针。

再来仔细看一下 `itab` 结构体：`_type` 字段描述了实体的类型，包括内存对齐方式，大小等；`inter` 字段则描述了接口的类型。`fun` 字段放置和接口方法对应的具体数据类型的方法地址，实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时会更新此表，或者直接拿缓存的 itab。

这里只会列出实体类型和接口相关的方法，实体类型的其他方法并不会出现在这里。如果你学过 C++ 的话，这里可以类比虚函数的概念。

另外，你可能会觉得奇怪，为什么 `fun` 数组的大小为 1，要是接口定义了多个方法可怎么办？实际上，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间里继续存储。从汇编角度来看，通过增加地址就能获取到这些函数指针，没什么影响。顺便提一句，这些方法是按照函数名称的字典序进行排列的。

再看一下 `interfacetype` 类型，它描述的是接口的类型：

```go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
```

可以看到，它包装了 `_type` 类型，`_type` 实际上是描述 Go 语言中各种数据类型的结构体。我们注意到，这里还包含一个 `mhdr` 字段，表示接口所定义的函数列表， `pkgpath` 记录定义了接口的包名。

接着来看一下 `eface` 的源码：

```go
type eface struct {
  _type *_type
  data  unsafe.Pointer
}
```

相比 `iface`，`eface` 就比较简单了。只维护了一个 `_type` 字段，表示空接口所承载的具体的实体类型。`data` 描述了具体的值。

## 接口的动态类型和动态值

从源码里可以看到：`iface`包含两个字段：`tab` 是接口表指针，指向类型信息；`data` 是数据指针，则指向具体的数据。它们分别被称为`动态类型`和`动态值`。而接口值包括`动态类型`和`动态值`。

【引申1】接口类型和 `nil` 作比较 （引言中的程序错误的原因）

接口值的零值是指`动态类型`和`动态值`都为 `nil`。当仅且当这两部分的值都为 `nil` 的情况下，这个接口值就才会被认为 `接口值 == nil`。

例子：

```go
package main
import "fmt"
type Coder interface {
    code()
}
type Gopher struct {
    name string
}
func (g Gopher) code() {
    fmt.Printf("%s is coding\n", g.name)
}
func main() {
    var c Coder
    fmt.Println(c == nil)
    fmt.Printf("c: %T, %v\n", c, c)
    var g *Gopher
    fmt.Println(g == nil)
    c = g
    fmt.Println(c == nil)
    fmt.Printf("c: %T, %v\n", c, c)
}
```

输出为： 

```go
true
c: <nil>, <nil>
true
false
c: *main.Gopher, <nil>
```

【引申2】来看一个例子，看一下它的输出：

```go
package main
import "fmt"
type MyError struct {}
func (i MyError) Error() string {
    return "MyError"
}
func main() {
    err := Process()
    fmt.Println(err)
    fmt.Println(err == nil)
}
func Process() error {
    var err *MyError = nil
    return err
}
```

函数运行结果：

```go
<nil>
false
```

这里先定义了一个 `MyError` 结构体，实现了 `Error` 函数，也就实现了 `error` 接口。`Process` 函数返回了一个 `error` 接口，这块隐含了类型转换。所以，虽然它的值是 `nil`，其实它的类型是 `*MyError`，最后和 `nil` 比较的时候，结果为 `false`。 **所以返回错误要返回无错误时，应该直接返回nil，而非用先声明再返回的方式**

【引申3】如何打印出接口的动态类型和值？

直接看代码：

```go
package main
import (
    "unsafe"
    "fmt"
)
type iface struct {
    itab, data uintptr
}
func main() {
    var a interface{} = nil
    var b interface{} = (*int)(nil)
    x := 5
    var c interface{} = (*int)(&x)
    ia := *(*iface)(unsafe.Pointer(&a))
    ib := *(*iface)(unsafe.Pointer(&b))
    ic := *(*iface)(unsafe.Pointer(&c))
    fmt.Println(ia, ib, ic)
    fmt.Println(*(*int)(unsafe.Pointer(ic.data)))
}
```

代码里直接定义了一个 `iface` 结构体，用两个指针来描述 `itab` 和 `data`，之后将 a, b, c 在内存中的内容强制解释成我们自定义的 `iface`。最后就可以打印出动态类型和动态值的地址。

Output: 

```go
{0 0} {17426912 0} {17426912 842350714568}
5
```

a 的动态类型和动态值的地址均为 0，也就是 nil；b 的动态类型和 c 的动态类型一致，都是 `*int`；最后，c 的动态值为 5。

以上自定义inface强转的方法， 在以下参考资料的[《Go 语言问题集》](https://www.bookstack.cn/read/qcrao-Go-Questions/interface-iface%20%E5%92%8C%20eface%20%E7%9A%84%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88.md)的‘标准库’的‘unsafe’部分有讲。是本好书，值得反复看。



# 参考

[《Go 语言问题集》](https://www.bookstack.cn/read/qcrao-Go-Questions/interface-iface%20%E5%92%8C%20eface%20%E7%9A%84%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88.md)

[《Go语言圣经中文版（简体）》](https://www.bookstack.cn/read/gopl-zh-simple/ch7-ch7-05.md)

