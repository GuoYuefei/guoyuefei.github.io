---
title: "Comparison Issues with math.NaN() in Golang"
date: 2020-08-03
draft: false
tags: ["golang", "floating-point", "mathematics"]
categories: ["Programming-Languages"]
keywords: ["golang", "math.NaN", "NaN comparison", "floating point comparison", "Go NaN", "IEEE754", "floating point pitfalls", "mathematical operations"]
description: "In-depth analysis of comparison issues with math.NaN() in Golang, exploring IEEE754 floating-point standard implementation in Go and how to properly handle NaN value comparisons and checks"
---

# Comparison Issues with math.NaN()

## Origin

This originated from an example I encountered while reviewing the underlying implementation of maps:

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

Output:

```go
[2.4, 2] [NaN, 3] [NaN, 3] [1.4, 1] 
k: NaN, v: 0
k: 2.400000000001, v: 0
k: 2.4, v: 2
false
```

This demonstrates that NaN is not equal to NaN. I was puzzled about why they're not equal. While we know that interfaces require identical dynamic types and values to be considered equal, note that NaN is essentially still of type float64, meaning interface comparison rules don't apply here.

After searching for information without success (perhaps my search methods were inadequate), I decided to investigate myself.

## Direct Search in Go Source Code (Inconclusive)

```go
// go version is 1.14.6
// runtime/alg.go 
func f64equal(p, q unsafe.Pointer) bool {
	return *(*float64)(p) == *(*float64)(q)
}
```

Some interpretation: This function provides a unified interface for equality operations, as there are many functions with the same prototype in this file for different variable types to call equality operations.

I couldn't find special handling for NaN in my search. Since we're examining source code, let's understand what NaN actually is:

```go
// go version is 1.14.6
// math/bits.go

// NaN returns an IEEE 754 ``not-a-number'' value.
func NaN() float64 { return Float64frombits(uvnan) }
```

The Float64frombits function, as the name suggests, interprets int64 bits as a float64 representation.

The key is uvnan, which is actually a constant:

```go
// go version is 1.14.6
// math/bits.go

const uvnan = 0x7FF8000000000001
```

When examining the source code, you'll see many constants. One interesting constant is `uvinf = 0x7FF0000000000000`, which represents the maximum double-precision value when the sign bit is positive, and the minimum when negative.

According to the IEEE 754 standard, NaN should have all exponent bits set to 1 and a non-zero mantissa. As long as there's at least one non-zero bit in the mantissa, it distinguishes uvinf from uvnan. Go uses the approach where the least significant bit of the mantissa is 1 and the rest are 0.

Now we understand what NaN actually is.

## Finding Answers Through Assembly

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

This is my test code. The fmt package isn't strictly necessary - we can disable compiler optimization with the -N flag. The xxxx function is used to easily locate the comparison assembly code.

Use the following command to print assembly instructions:

```go
go tool compile -S main.go
```

Then find this section, which is what we need:

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

From `MOVSD "".a+8(SP), X0` to `RET` is what we need.

```assembly
MOVSD	"".a+8(SP), X0 					; Load function's first parameter into X0 (double word)
UCOMISD	X0, X0							; Unordered comparison - this is the key step
SETEQ	CL								; Store ZF flag in CL register
SETPC	AL								; Store PF flag in AL register (parity flag)
ANDL	AX, CX							; Bitwise AND operation
MOVB	CL, "".~r1+16(SP)				; Store AND result in return value address
RET										; Return
```

The key to comparison lies in the `UCOMISD` instruction's operation.

The uncomisd (unordered compare scalar double-precision floating-point) instruction works as follows:

> ### (V)UCOMISD (all versions)
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

Thus, when unordered (i.e., when a comparison operand is NaN), ZF, PF, and CF are all 1, so ZF AND PF == 1. In other cases, ZF AND PF == 0, creating the distinction. The bitwise AND result CL is stored in the return address, and the function returns.

However, using 0 to represent true seems incorrect. And normal number comparisons for equality or inequality aren't distinguished either??? So there must be something missing above.

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

Output:

```go
1
```

This proves that true is represented as 1 in memory.

Finally, I identified the error in my understanding of the setpc instruction.

SETPC, which corresponds to SETNP in regular assembly, sets the value to 1 only when the PF register is 0.

Therefore, the final value in the CL register should be `ZF AND NOT PF`. So when NaN exists in unordered comparison, CL's value `1 AND 0` becomes 0. For normal unequal comparisons, `0 AND 1` is also 0. Only when equal, `1 AND 1` becomes 1.

## Conclusion

NaN is defined by the IEEE 754 standard, and its comparison is handled by the underlying assembly UCOMISD instruction. This instruction accounts for unordered comparisons (when NaN is present).

## References:

https://www.felixcloutier.com/x86/ucomisd

https://quasilyte.dev/blog/post/go-asm-complementary-reference/ (GO assembly and other assembly cross-reference)

https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf (Look up SETNP instruction)