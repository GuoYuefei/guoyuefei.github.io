---
title: "Interface Type Values and Interface Value Issues, Including nil Problems"
date: 2020-07-29
draft: false
tags: ["golang", "interfaces", "type-system"]
categories: ["Tech"]
keywords: ["golang", "interface types", "interface values", "nil issues", "Go interfaces", "type assertion", "empty interface", "interface comparison"]
description: "In-depth analysis of interface type values and interface value issues in Golang, particularly the special behavior and pitfalls of nil values in interfaces, covering type assertion, empty interface handling and other core concepts"
---

# Interface Type Values and Interface Value Issues, Including nil Value Problems

## Introduction

Let's start with an incorrect program:

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

When `debug=true`, there's no issue.

But when `debug=false`, a null pointer panic occurs at runtime: `out.Write([]byte("done!\n"))`

The reason is that the `out` interface has a dynamic type of `*bytes.Buffer`, but its interface value is nil. ---> Cause: The parameter wasn't assigned, but has the zero value of this type.

Whereas a true nil has both dynamic type and value as nil, so `nil != out` evaluates to `true`.

-----------

## iface and eface

Let's examine from the source code perspective:

Regular interfaces are defined by the iface type, while empty interfaces are eface.

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

The above is the definition of iface. `iface` maintains two pointers internally: `tab` points to an `itab` entity, representing the interface type and the entity type assigned to this interface. `data` points to the specific value of the interface, generally a pointer to heap memory.

Let's examine the `itab` struct more closely: The `_type` field describes the entity type, including memory alignment, size, etc.; The `inter` field describes the interface type. The `fun` field stores method addresses of the specific data type corresponding to interface methods, enabling dynamic dispatch for interface method calls. This table is generally updated during each interface assignment conversion, or cached itab is used directly.

Only methods related to the entity type and interface are listed here; other methods of the entity type won't appear here. If you've learned C++, this can be compared to the concept of virtual functions.

You might wonder why the `fun` array has size 1 - what if the interface defines multiple methods? Actually, this stores the function pointer of the first method. If there are more methods, they continue to be stored in the memory space after it. From an assembly perspective, these function pointers can be obtained by incrementing the address, which has no impact. By the way, these methods are arranged in dictionary order of function names.

Now look at the `interfacetype` type, which describes the interface type:

```go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
```

It wraps the `_type` type, where `_type` is actually a struct describing various data types in Go. Note that it also contains an `mhdr` field representing the function list defined by the interface, and `pkgpath` records the package name where the interface is defined.

Now look at the source code of `eface`:

```go
type eface struct {
  _type *_type
  data  unsafe.Pointer
}
```

Compared to `iface`, `eface` is simpler. It only maintains one `_type` field, representing the specific entity type carried by the empty interface. `data` describes the specific value.

## Dynamic Type and Dynamic Value of Interfaces

From the source code, we can see: `iface` contains two fields: `tab` is the interface table pointer, pointing to type information; `data` is the data pointer, pointing to specific data. They are called `dynamic type` and `dynamic value` respectively. Interface values include both `dynamic type` and `dynamic value`.

【Extension 1】Comparing interface types with `nil` (Cause of the error in the introduction program)

The zero value of an interface value means both `dynamic type` and `dynamic value` are `nil`. Only when both parts are `nil` will the interface value be considered `interface value == nil`.

Example:

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

Output:

```go
true
c: <nil>, <nil>
true
false
c: *main.Gopher, <nil>
```

【Extension 2】Let's look at another example and its output:

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

Function output:

```go
<nil>
false
```

Here we first define a `MyError` struct that implements the `Error` function, thus implementing the `error` interface. The `Process` function returns an `error` interface, which implicitly involves type conversion. So although its value is `nil`, its actual type is `*MyError`, and when compared with `nil`, the result is `false`. **Therefore, when returning no error, you should directly return nil instead of declaring first and then returning.**

【Extension 3】How to print the dynamic type and value of an interface?

Look at the code directly:

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

The code directly defines an `iface` struct using two pointers to describe `itab` and `data`, then forcibly interprets the memory content of a, b, c as our custom `iface`. Finally, we can print the addresses of dynamic types and dynamic values.

Output:

```go
{0 0} {17426912 0} {17426912 842350714568}
5
```

The addresses of a's dynamic type and dynamic value are both 0, meaning nil; b's dynamic type is the same as c's dynamic type, both `*int`; finally, c's dynamic value is 5.

The method of custom iface and forced conversion mentioned above is discussed in the 'unsafe' section of the 'standard library' in the reference material [《Go Questions》](https://www.bookstack.cn/read/qcrao-Go-Questions/interface-iface%20%E5%92%8C%20eface%20%E7%9A%84%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88.md). It's a good book worth reading repeatedly.

# References

[《Go 语言问题集》](https://www.bookstack.cn/read/qcrao-Go-Questions/interface-iface%20%E5%92%8C%20eface%20%E7%9A%84%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88.md)

[《Go语言圣经中文版（简体）》](https://www.bookstack.cn/read/gopl-zh-simple/ch7-ch7-05.md)