---
title: "golang中关于math.NaN()的比较问题"
date: 2020-08-03
draft: false
tags: ["golang", "浮点数", "数学运算"]
categories: ["编程语言"]
keywords: ["golang", "math.NaN", "NaN比较", "浮点数比较", "Go语言NaN", "IEEE754", "浮点数陷阱", "数学运算"]
description: "深入解析Golang中math.NaN()的比较问题，探讨IEEE754浮点数标准在Go语言中的实现，以及如何正确处理NaN值的比较和判断"
---

# 关于math.NaN() 的比较问题

## 缘起

缘起于二刷map底层时看到的一个例子

```go
func main() {
    m := make(map[float64]int)
    m[1.4] = 1
    m[2.4] = 2
    m[math.NaN()] = 3
    m[math.NaN()] = 3
    for k, v := range m {
        fmt.Printf("[%v, %d] ", k, v)
    }
    fmt.Printf("\nk: %v, v: %d\n", math.NaN(), m[math.NaN()])
    fmt.Printf("k: %v, v: %d\n", 2.400000000001, m[2.400000000001])
    fmt.Printf("k: %v, v: %d\n", 2.4000000000000000000000001, m[2.4000000000000000000000001])
    fmt.Println(math.NaN() == math.NaN())
}
```

Output :

```go
[2.4, 2] [NaN, 3] [NaN, 3] [1.4, 1] 
k: NaN, v: 0
k: 2.400000000001, v: 0
k: 2.4, v: 2
false
```

由此证明， NaN 是不等于 NaN 的，然后我就在那边纠结这个为啥不相等呢？ 我们都知道接口有动态类型和动态值相同的限制才算相等，但是关键注意 NaN 本质上将还是 float64 类型， 也就是说关于接口的比较法则是不适用于它。

于是我在那边查了半天资料，一无所获。（可能是我查资料的姿势不对吧）于是乎，决定自己动动手，找找看。

## 直接搜寻 Go 的 源码（无结论）

```go
// go version is 1.14.6
// runtime/alg.go 
func f64equal(p, q unsafe.Pointer) bool {
	return *(*float64)(p) == *(*float64)(q)
}
```

还是要解读下的，可能刚入门的gopher认为这个不就是直接比较嘛，其实这个函数只是为了统一接口，在这个文件下有很多和这个函数相同函数原型的函数，方便各变量调用相等操作。

可能我找的不够全面，反正从这边我无法找到关于 NaN 的特殊处理方法。

额，既然到了源码这部分，还是来认识下什么是 NaN 吧！

```go
// go version is 1.14.6
// math/bits.go

// NaN returns an IEEE 754 ``not-a-number'' value.
func NaN() float64 { return Float64frombits(uvnan) }
```

Float64frombits 函数顾名思义就是从比特位层面看int64是代表什么样的float64

关键uvnan是什么， 其实就是一个常数

```go
// go version is 1.14.6
// math/bits.go

const uvnan    = 0x7FF8000000000001
```

其实看源码的时候你能看到很多常数，有一个很有趣的常数 <code>uvinf    = 0x7FF0000000000000</code>,这个当符号位为正时，代表双精度的最大值，为负数时为最小值

根据 IEEE 754 标准呢，NaN 应该是指数部分全1， 小数部分非零， 但是只要小数部分有一个非0，就能区分出是 uvinf 还是 uvnan 了。go语言这边使用的是小数部分最低位为1其余小数部分为0的方式。

现在我们知道 NaN 到底是样什么东西了。

## 通过汇编找问题所在

```go
// main.go
package main

import "fmt"
//import "unsafe"
import "math"

func main() {
  a := math.NaN()

  b := xxxx(a)

  fmt.Println(b)
}

func xxxx(a float64) bool {
  r := a == a
  return r
}

```

以上是我的测试代码， 其实可以不用fmt， 通过-N方式关闭编译优化就行。 使用一个xxxx函数的原因是为了方便定位比较这块的汇编代码。

使用以下命令打印出汇编指令

```go
go tool compile -S main.go
```

然后找到这块内容，这是我们需要的

```assembly
"".xxxx STEXT nosplit size=23 args=0x10 locals=0x0
	0x0000 00000 (const.go:15)	TEXT	"".xxxx(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (const.go:15)	PCDATA	$0, $-2
	0x0000 00000 (const.go:15)	PCDATA	$1, $-2
	0x0000 00000 (const.go:15)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (const.go:15)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (const.go:15)	FUNCDATA	$2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (const.go:16)	PCDATA	$0, $0
	0x0000 00000 (const.go:16)	PCDATA	$1, $0
	0x0000 00000 (const.go:16)	MOVSD	"".a+8(SP), X0
	0x0006 00006 (const.go:16)	UCOMISD	X0, X0
	0x000a 00010 (const.go:16)	SETEQ	CL
	0x000d 00013 (const.go:16)	SETPC	AL
	0x0010 00016 (const.go:16)	ANDL	AX, CX
	0x0012 00018 (const.go:17)	MOVB	CL, "".~r1+16(SP)
	0x0016 00022 (const.go:17)	RET
	0x0000 f2 0f 10 44 24 08 66 0f 2e c0 0f 94 c1 0f 9b c0  ...D$.f.........
	0x0010 21 c1 88 4c 24 10 c3                             !..L$..
```

<code>MOVSD	"".a+8(SP), X0 </code>这部分开始一直到 <code>RET</code>，都是我们所需要的。

```assembly
MOVSD	"".a+8(SP), X0 					; 将函数的第一个参数放入X0中 		双字长
UCOMISD	X0, X0							; 无序比较					就是这一步，等会慢慢讲
SETEQ	CL										; CL寄存器中存入ZF标志位
SETPC	AL										; AL寄存器中存入PF标志位	奇偶标志位		// 参照SETNP指令 后来发现其实不是取PF，而是取PF反后放入AL寄存器
ANDL	AX, CX									; 按位与运算
MOVB	CL, "".~r1+16(SP)				; 与运算结果放入返回值的地址上
RET														; 返回
```

比较的关键在<code>UCOMISD</code>指令的作用上。

无序比较操作符 uncomisd 的作用如下： 

> ### (V)UCOMISD (all versions)[ ¶](https://www.felixcloutier.com/x86/ucomisd#-v-ucomisd--all-versions-)
>
> ```
> RESULT← UnorderedCompare(DEST[63:0] <> SRC[63:0]) {
> (* Set EFLAGS *) CASE (RESULT) OF
>  UNORDERED: ZF,PF,CF←111;
>  GREATER_THAN: ZF,PF,CF←000;
>  LESS_THAN: ZF,PF,CF←001;
>  EQUAL: ZF,PF,CF←100;
> ESAC;
> OF, AF, SF←0; }
> ```

由此可见，当为无序时，也就是出现一个比较操作符是NaN时， ZF, PF, CF都为1， ZF AND PF == 1。 其他情况，ZF AND PF == 0， 这就做出了区分. 将按位与结果CL放入放回值地址，return函数。ps. 为嘛用0代表true， 感觉不对啊。 而且普通数比较相等还是不相等也没区分啊？？？所以上面一定有遗漏。

```go
package main

import "fmt"
import "unsafe"

func main() {
  a := true
  c := *(*byte)(unsafe.Pointer(&a))
  fmt.Printf("%d", c)
}
```

Ouput :

```go
1
```

事实证明true是在内存里为1的。

最后我把错误锁定在了对 setpc 指令的理解上。

SETPC ,也就是普通汇编中SETNP这个指令，是当PF寄存器为0时，取值才为1.

所以最后CL寄存器中的值应该为<code>ZF AND NOT PF</code>, 所以在无序比较中一旦有NaN存在，CL的值<code>1 and 0</code>就为0， 正常比较是不相等<code>0 and 1</code>c也是0， 只有相等时<code>1 and 1</code>才为1.

## 结语

NaN 是由 IEEE 754 标准定义的， 其比较是由底层汇编 UCOMISD 完成了。 该指令考虑了无序（存在NaN的情况）时的比较。



## 参考文献： 

https://www.felixcloutier.com/x86/ucomisd

https://quasilyte.dev/blog/post/go-asm-complementary-reference/			(GO 汇编和其他汇编的对照表)

https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf   （查找SETNP指令）