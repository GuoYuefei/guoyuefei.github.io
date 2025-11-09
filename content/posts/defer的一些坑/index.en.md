---
title: "Pitfalls of defer in Go"
date: 2020-07-16
draft: false
tags: ["golang"]
categories: ["Tech"]
description: "Avoiding pitfalls with defer in Go language"
---

# Pitfalls of defer

## Key Points

**Key Point 1**: defer executes before return.

**Key Point 2**: return itself is not an atomic operation.

## Examples of Pitfalls

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
	return 0 is not atomic - assignment first, then return
	defer executes between assignment and return, which is what official docs mean by "defer before return"
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
	No named parameters
	keng4 is actually similar to keng2 - keng4 has no named return, but the return value address space still exists
	Execution steps:
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

There are four pitfalls here. For any of them, just analyze defer and return using the key points mentioned above.

For example, keng1 function:

```go
// output 1
func keng1() (ret int) {
	defer func() {
		ret++
	}()
	return 0
}
```

Can be translated as:

```go
ret = 0
ret++
return			// This return returns ret, so output is 1
```

For keng2 function:

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

Can be translated as:

```go
t := 5
// Start of return breakdown with defer content
ret = t
t = t + 5
return 			// This return returns ret, so output is 5
```

Analyze keng3 function yourself, let's jump to keng4 function. Why is it similar to keng2?

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

No named return value is used, but we can assume it exists (the address space is actually allocated), so let's temporarily use result to represent this return area:

```go
ret := 0
// you know
result = ret
ret++
return 				// returns result, so output is 0
```

From the translation perspective, this is essentially the same as keng2.

----
2020-07-29 Update

Each time a defer statement executes, the function is "pushed onto the stack" and function parameters are copied; when the outer function (not a code block, like a for loop) exits, defer functions execute in reverse order of definition; if the defer-executed function is nil, it will cause panic when the function is ultimately called.

## Reference
[Golang之轻松化解defer的温柔陷阱](https://mp.weixin.qq.com/s?__biz=MjM5MDUwNTQwMQ==&mid=2257483686&idx=1&sn=9be2edd3e5cb8202dd34fbd5309c7a50&scene=21#wechat_redirect)
