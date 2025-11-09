---
title: "Golang Pitfall Avoidance Guide"
date: 2020-04-27
draft: false
tags: ["golang"]
categories: ["Tech"]
description: "Go language pitfalls to avoid"

# Golang Pitfall Avoidance Guide

----
<h2>Table of Contents</h2>

- [1. Can Return Pointers to Local Variables](#1-can-return-pointers-to-local-variables)
- [2. Two Allocation Primitives Provided by Go - Built-in Functions new and make](#2-two-allocation-primitives-provided-by-go---built-in-functions-new-and-make)
- [3. Composite Literals](#3-composite-literals)
- [4. Are Arrays, Slices, Maps, and Channels Value Types or Reference Types in Basic Types](#4-are-arrays-slices-maps-and-channels-value-types-or-reference-types-in-basic-types)
- [5. Initialization Function init](#5-initialization-function-init)
- [6. About Pointers and Values](#6-about-pointers-and-values)
- [7. Built-in Functions Related to Slices, Maps, and Channels](#7-built-in-functions-related-to-slices-maps-and-channels)
- [8. The Story of append](#8-the-story-of-append)
- [9. Two Ways to Traverse Strings](#9-two-ways-to-traverse-strings)

----
## 1. Can Return Pointers to Local Variables

As one of the few languages that includes pointers, it differs from C. In C, functions cannot return pointers to local variables because local variables are released from the stack when the function ends. However, Golang can return pointers to local variables.

```
#include <iostream>
using namespace std;
int* get_some() {
	int a = 1;
	return &a;
}

int main() {
	cout << "a = " << *get_some() << endl;
	return 0;
}
```
*This is clearly incorrect code in C/C++ - after `a` leaves the stack, it's gone. **The following error occurs:**
```
$ g++ t.cpp
> t.cpp: In function 'int* get_some()':
> t.cpp:4:6: warning: address of local variable 'a' > returned [-Wreturn-local-addr]
>   int a = 1;
      ^
```

Go language test code:
```
package main
import "fmt"
func GetSome() *int {
	a := 1;
	return &a;
}

func main() {
	fmt.Printf("a = %d", *GetSome())
}
```
*Basically the same code, but with the following result*
```
> $ go run t.go  
> a = 1   
```

Clearly, it's not that the Go compiler can't identify this issue, but rather it optimizes for this case. Reference from Go FAQ:
> How do I know whether a variable is allocated on the heap or the stack?
>
> From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
>
> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
>
> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

This means we don't need to worry about returned pointers being dangling pointers. I understand this to mean that normally local variables in functions are stored on the stack, but if a local variable is too large, the compiler might choose to store it on the heap, which makes more sense. Another case is when the compiler cannot prove that the variable is not referenced after the function ends, it will allocate the variable on the garbage-collected heap. Summary: **The compiler will analyze and decide whether to allocate local variables on the stack or heap.**

Hmm... Using this feature, we can use the following approach to achieve concurrent operations:
```
func SomeFun() <-chan int {
    out := make(chan int)
    go func() {
        // Do something secret...
    }()
    return out
}
```

## 2. Two Allocation Primitives Provided by Go - Built-in Functions new and make

Go provides two allocation primitives - the built-in functions new and make. They do different things.

1. new does not **initialize** memory, but **zeros** the memory. That is, new(T) allocates zeroed memory space for a new item of type T and returns its address, which is *T. That is, it returns a pointer pointing to the zero value space of type T.

2. make function signature make(T, args). It is only used for creating slices, maps, and channel types. make directly returns a value of type T rather than a **pointer**, and of course this value is initialized. Taking slices as an example:
```
make([]int, 10, 100)
```
Allocates a slice structure of int type with capacity 100 and length 10.
```
new([]int)
```
This returns a pointer to a newly allocated, zeroed slice structure, i.e., a pointer pointing to a nil slice value.

The following example illustrates the difference between new and make:
```
var p *[]int = new([]int)       // Allocates slice structure; *p = nil; basically useless
var v []int = make([]int, 100)  // Slice v now references a new array with 100 int elements

// No need to be so complicated
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic usage
v := make([]int, 100)
```
**Remember**, make only applies to maps, slices, and channels and does not return pointers. To get an explicit pointer, use new to allocate memory.

## 3. Composite Literals

In the os standard package, there is the following code. This function is equivalent to constructors in other languages:
```
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```
This code appears too verbose and can be simplified using composite literals:
```
func NewFile(fd int, name string) *File {
	if fd < 0 {
		return nil
	}
	f := File{fd, name, nil, 0}
	return &f
}
```
From the above code, we can see that the composite literal `File{fd, name, nil, 0}` returns a value reference rather than a **pointer**, so the address operator is needed when returning.

## 4. Are Arrays, Slices, Strings, Maps, and Channels Value Types or Reference Types

This is something worth noting because when you pass an array to a function, can it change the value of the external array? This tests whether the array type is a value type or reference type. If it's a reference type, it's like passing a pointer in C language - you can change the value of the external parameter inside the function. But if it's a value type, only a copy is passed during the function call, so you cannot modify the external value inside the function. Strings are value types, and their type maintains an unsafe pointer and length.

**Arrays in Go are value types**, which is very different from other languages. Take C/C++ as an example:
```
#include <iostream>
using namespace std;
const int NUM = 5;
int a[NUM] = {5,4,3,2,1};

void change_a(int arr[],int n) {
        for(int i = 0; i < n; i++){
                arr[i]--;
        }
}

int main() {
        change_a(a,NUM);
        for(int i = 0; i < NUM; i++) {
                cout << "a[" << i << "] = " << a[i] << endl;
        }
}
```
**Result:**
```
Administrator@PC-201809211459 MINGW64 ~/Desktop
$ g++ v.cpp -o v.exe

Administrator@PC-201809211459 MINGW64 ~/Desktop
$ ./v.exe
a[0] = 4
a[1] = 3
a[2] = 2
a[3] = 1
a[4] = 0

```
Clearly, changing the formal parameter array inside the function caused the global variable a array to change.

Here's the Golang code:
```
package main

import "fmt"

var a [5]int = [5]int{5,4,3,2,1}

func changeA(arr [5]int) {
        for i := 0; i < 5; i++ {
                arr[i]--
        }
}

func main() {
        changeA(a)
        for i,v := range a {
                fmt.Println("a[",i,"] = ",v)
        }
}
```
**Result:**
```
Administrator@PC-201809211459 MINGW64 ~/Desktop
$ go run v.go
a[ 0 ] =  5
a[ 1 ] =  4
a[ 2 ] =  3
a[ 3 ] =  2
a[ 4 ] =  1
```
This shows that arrays in Go are value types. In fact, we rarely use arrays to pass parameters because in Go, if you use arrays to pass parameters, you need to hardcode the array size in the function's parameter list, which is not required in C/C++.

But in Go, you can use slices to pass parameters because **slices are reference types**. Same example as above:
```
package main

import "fmt"

func main() {
        a := []int{5,4,3,2,1}
        func(arr []int) {
                for i := 0; i < len(arr); i++ {
                        arr[i]--
                }
        }(a)
        for i,v := range a {
                fmt.Println("a[",i,"] = ",v)
        }
}
```
**Result:**
```
Administrator@PC-201809211459 MINGW64 ~/Desktop
$ go run vv.go
a[ 0 ] =  4
a[ 1 ] =  3
a[ 2 ] =  2
a[ 3 ] =  1
a[ 4 ] =  0
```
This shows that **Golang arrays are value types, but slices are reference types.**

Remember: `[]T{}`, map, and chan are the three reference types among the basic system types, and all three can use the make built-in function. Section 7 will summarize these built-in functions.

## 5. Initialization Function init

This function is quite magical. I didn't fully understand it when reading the official documentation. A passage from the official documentation (Chinese mirror site):

> Finally, each source file can define its own parameterless init function to set up some necessary state. (Actually, each file can have multiple init functions.) Its end means the end of initialization: init will only be called after all variable declarations in the package have been evaluated through their initializers, and those init functions will only be evaluated after all imported packages have been initialized.

> Besides initialization that cannot be represented as declarations, init functions are often used to check or correct program state before the program actually starts executing.

After experimenting, I roughly concluded that when importing a package, the init functions of all files in that package will be executed, in the order determined by the file system sorting.

**vv.go**, the file containing the main function:
```
package main

import(
	"fmt"
	"./some"
	_ "./another"
)

func init() {
	fmt.Println("hello")
}

func main() {
	a := []int{5,4,3,2,1}
	some.ChangeA(a)
	for i,v := range a {
		fmt.Println("a[",i,"] = ",v)
    }
	var c int = 10
	fmt.Println(c)
}
```

Package some has three files:
some0.go
```
package some
import "fmt"
func init() {
	fmt.Println("some0")
}
```
some1.go
```
package some
import (
	"fmt"
)
var a int
func init(){
	a = 10
	fmt.Println("package some init done!",a)
}
func ChangeA(arr []int) {
	for i := 0; i < len(arr); i++ {
		arr[i]--
	}
}
```
some2.go
```
package some
import "fmt"
func init() {
	fmt.Println("some2")
}
```

Package another has one file:
another.go
```
package another
import "fmt"
func init() {
	fmt.Println("package another init done!")
}

```
To save space, some blank lines for aesthetics are omitted.

Final running result:
```
some0
package some init done! 10
some2
package another init done!
hello
a[ 0 ] =  4
a[ 1 ] =  3
a[ 2 ] =  2
a[ 3 ] =  1
a[ 4 ] =  0
10
```

If we make a small modification to the another.go file:
```
package another
import "fmt"
import _ "../some"
func init() {
	fmt.Println("package another init done!")
}
```
The result remains unchanged, showing that the init function is only executed once, not every time its package is imported.

## 6. About Pointers and Values

This is also something I find confusing, so I've organized and experimented with some parts here. I expect there will be more issues regarding this in the future.

First, note the **important points**:

1. Methods bound to type pointer *T can change the value of that type, but methods only bound to type T cannot change the value of that type. Example code:
```
package main

import "fmt"

type Si int

func (s *Si)Plus1(a Si) {
	*s += a
}

func (s Si)Plus2(a Si) {
	s += a
}

func main() {
	var s Si = 10
	s.Plus1(1)
	fmt.Println("after Plus1: s = ",s)
	s.Plus2(1)
	fmt.Println("after Plus2: s = ",s)
}
```
Running result:
```
after Plus1: s =  11
after Plus2: s =  11
```
We can see that Plus2 didn't work as expected.

Go is a language that's easy to understand at a glance. In other languages, member functions are magically encapsulated in a class, but Go is different. The parameter in the parentheses before the function name specifies which type this function belongs to, and explicitly passes the value of that type to the function. In other words, the parentheses before the function name can be seen as the formal parameter list.

2. When assigning a type to an interface, the address should be taken
```
package main

import "fmt"

type Si int

type Plus interface {
	Plus1(a Si)
	Plus2(a Si)
}

func (s *Si)Plus1(a Si) {
	*s += a
}

func (s Si)Plus2(a Si) {
	s += a
}

func main() {
    var s Si = 10
	var ss Plus = &s
	ss.Plus1(1)
	fmt.Println("after Plus1: s = ",s)
}
```
The above code is valid and runs correctly.

But if we change `var ss Plus = &s` to `var ss Plus = s`, a compilation error occurs:
```
# command-line-arguments
cmd\tt.go:26:6: cannot use s (type Si) as type Plus in assignment:
	Si does not implement Plus (Plus1 method has pointer receiver)
```
This compilation error message is interesting. The first half reminds us that we haven't implemented the Plus interface, but we might think that the Si type clearly implements the Plus interface. What's going on? Actually, looking at the text in parentheses combined with the non-error program above, we understand that the compiler means that *Si implements the Plus interface but Si does not.

Why does this happen? It's because there's a function in the interface bound to a pointer `func (s *Si)Plus1(a Si)`, and the Si type doesn't implement this function, so it doesn't implement the Plus interface. But why does `func (s Si)Plus1(a Si)` bound to Si but *Si also implements the Plus interface? That's because the Go compiler can automatically generate `func (s *Si)Plus1(a Si)` based on the `func (s Si)Plus1(s Si)` function, so *Si implements all functions.

Of course, this automatic generation process cannot work in reverse because pointers have greater permissions - `func (s *Si)Plus1(a Si)` might change the value of s, while `func (s Si)Plus1(a Si)` cannot do this, so the compiler won't automatically generate it either.

Based on the above analysis, let's experiment with the following example:
```
package main

import "fmt"

type Si int

type Plus interface {
	Plus1(a Si)
	Plus2(a Si)
}

func (s Si)Plus1(a Si) {
	s += a
}

func (s Si)Plus2(a Si) {
	s += a
}

func main() {
	var s Si = 10
	var ss Plus = s
	ss.Plus1(1)
	fmt.Println("after Plus1: s = ",s)
}

Running result:
after Plus1: s =  10
```
Why it's 10 has been explained in the first point. Compilation passes, second point analysis is reasonable!

3. Based on the above examples, I discovered a strange but not surprising phenomenon. Both *T and T can directly call functions. And when assigning to an interface, the interface declaration part doesn't need to be declared as a pointer and cannot be declared as a pointer. After the interface is assigned, you cannot take its content, although some interfaces appear to be pointers.

No examples here.

Considering the above three points, I think it's necessary to develop several habits:
1. **When assigning to an interface, it's best to use type pointers for assignment**
2. **When writing member functions, follow the principle of least privilege, paying attention to the difference between *T and T**
3. **When calling member functions, both pointers and the type itself can be called directly**
4. **When calling member functions through interfaces, just call them directly**

## 7. Built-in Functions Related to Slices, Maps, and Channels

This part is also somewhat messy. The built-in functions related to these three basic types can be roughly divided into three categories: creation, deletion, and operations.

1. Creation: map, slice, and channel creation generally use the make function for memory allocation.
2. Deletion: delete is mainly used to remove instances from maps. Well, let's put channel's close in this category too.
3. Operations: len and cap can be used for different types. len can be used for the length of strings, slices, and arrays. cap generally returns the allocated space size of slices. copy is used to copy slices. append is used to append to slices.

ps: new is used for memory allocation of various types, not just the above ones.

<h6>*Specific usage as follows*:</h6>

1. make  
    1.1. channel: Here we only use the common type int as an example:
```go
ch1 := make(chan int)        // Channel without buffer
ch2 := make(chan int, 1024)     // Channel with buffer
```
    1.2. slice: Function signatures for slice: make([]type,len) and make([]type,len,cap)
```go
slice1 := make([]int, 10)		// slice1 has 10 elements with zero initial values
slice2 := make([]int, 10, 100)	// slice2 has 10 elements with zero initial values, and initial capacity of 100
```
    1.3. map: Signature make(map[keyType]valueType)
```go
mp := make(map[string]int)
mp["aaa"] = 3
```
2. append  
    2.1. slice: append(slice []Type, elems ...Type) []Type  
    The append function is mainly used to add elements to the end of a slice. As a special case, you can add a string to a byte slice `[]byte("hello")`. It returns an updated slice. If you want to use it, you need a variable to receive this updated value. Examples:
```go
slice1 = append(slice1,2,3,4)
slice2 = append(slice2,slice1...)
slice3 := append([]byte("hello"),"world"...)
```
3. copy  
    3.1. slice: copy(dst, src []Type) int
    This function requires caution. If slice1 and slice2 have lengths of 5 and 3 respectively... Let's use code to illustrate:
```go
// len(slice1) is 5
// len(slice2) is 3
// i==3, only copies the first three elements of slice1 to slice2
i := copy(slice2,slice1)
// i==3, only copies the first three elements of slice2 to slice1
i = copy(slice1,slice2)
```
4. len, cap  
    4.1. len is used to get the length of slices and maps, the number of unread elements in channels. cap is used to get the capacity of slices and the buffer capacity of channels. Signatures: len(v Type) int, cap(v Type) int
```go
len(ch1)
len(slice1)
len(mp)
cap(slice1)
cap(ch1)
```
len is not only used for these three data types, but also for strings, arrays, and pointers to arrays. Source code comments as follows:
> // The len built-in function returns the length of v, according to its type:  
> //	Array: the number of elements in v.  
> //	Pointer to array: the number of elements in *v (even if v is nil).  
> //	Slice, or map: the number of elements in v; if v is nil, len(v) is zero.  
> //	String: the number of bytes in v.  
> //	Channel: the number of elements queued (unread) in the channel buffer;  
> //	if v is nil, len(v) is zero.  

cap can also be used with arrays and pointers to arrays, but its return content is the same as the len function. Source code comments as follows:
> // The cap built-in function returns the capacity of v, according to its type:  
> //	Array: the number of elements in v (same as len(v)).  
> //	Pointer to array: the number of elements in *v (same as len(v)).  
> //	Slice: the maximum length the slice can reach when resliced;  
> //	if v is nil, cap(v) is zero.  
> //	Channel: the channel buffer capacity, in units of elements;  
> //	if v is nil, cap(v) is zero.  

5. delete  
    5.1. delete is only used to remove elements from maps. Signature: delete(m map[Type]Type1, key Type)
```go
delete(mp,"aaa")
```
6. close  
    6.1. Channels can receive and send data, and can also be closed. When a channel is closed, sending data to the channel will cause a panic. But when a channel is closed, we can still retrieve data from it. If there is still data that hasn't been retrieved, we can still get that data. When all buffered data has been retrieved, we can still retrieve data from the channel, but the data will be zero values. Signature: close(c chan<- Type)
```
close(ch)
```

## 8. The Story of append

append returns a slice and requires a variable to receive this slice.

1. append will continue to use the original slice's underlying array when the capacity is **sufficient**. In this case, the returned slice is not the original slice, but the underlying array used is the original slice's array.
2. append will replace the underlying array when the original slice's underlying array capacity is **insufficient**. In this case, the returned slice is not the original slice, and the underlying array is not the original slice's array.

```go
package main

import "fmt"

func main() {
	x := []int{0,1,2}
	x = append(x, 3)
	
    a := append(x, 4)
    b := append(x, 5)
	
	// output x is [0 1 2 3]
	fmt.Println("x is", x)
    // output a is [0 1 2 3 5]
	fmt.Println("a is", a)
    // output b is [0 1 2 3 5]
	fmt.Println("b is", b)
	
}
```

If you want to get a slice independent of the original slice, it's best to use the copy function. Note that the elements here are all value types. If they are reference types, the copied elements will still be associated with the original slice.

```go
package main

import "fmt"

func main() {
	x := []int{0,1,2}
	x = append(x, 3)
	
	a := make([]int, len(x)+1)
	b := make([]int, len(x)+1)
	
	copy(a, append(x, 4))
	copy(b, append(x, 5))			// If not using copy, a[4]==5, b[4]==5. Using the same memory
	
	// output x is [0 1 2 3]
	fmt.Println("x is", x)
    // output a is [0 1 2 3 4]
	fmt.Println("a is", a)
    // output b is [0 1 2 3 5]
	fmt.Println("b is", b)
	
}
```

## 9. Two Ways to Traverse Strings

In Golang, there are two direct ways to traverse strings:
- 1. If using `for range` traversal, the element value obtained is of rune type with UTF encoding
- 2. If using the traditional `for i := ...` approach, you get byte type

Because of this difference, the lengths traversed by these two methods are also different, although there's no difference when traversing UTF and ASCII identical characters, such as pure English.

To be continued...

---
2020-07-16 Update
If I continue writing, it might become too long, so I'll write separately, like [Pitfalls of defer](/en/posts/defer的一些坑/).

---
2020-07-29 Update

```go
const c int = 111
var a int = 11
ap := &a
fmt.Println(ap, &ap)
// fmt.Println(&c)			// Not allowed to get constant pointer
// fmt.Println(&(&a)) 	 	// Not allowed, &a itself has no reference
```
Reference: [Interface Type Values and Interface Value Issues, Including nil Value Problems](/en/posts/接口类型值和接口值问题包括nil值问题/)