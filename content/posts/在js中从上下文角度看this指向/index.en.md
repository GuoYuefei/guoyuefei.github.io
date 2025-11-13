---
title: "Understanding JavaScript 'this' from Execution Context Perspective"
date: 2019-03-13
draft: false
tags: ["javascript", "learning", "programming-techniques"]
categories: ["Programming Languages"]
keywords: ["javascript", "this", "execution-context", "scope-chain", "browser", "nodejs"]
description: "Deep understanding of the 'this' keyword mechanism in JavaScript"
---

# Understanding JavaScript 'this' from Execution Context Perspective

Let's start with some code - try to predict what each function call will output to the console:

```javascript
var fobj = {
    name: "G",
    showcc: function() {
        console.log(this.name)
    },
    obj: {
        id: 39,
        c: this,                
        showcc: () => {
            console.log(this.name)
            console.log(this)
        },
        showiderr: () => {
            console.log(this.id)
        },
        showid: function() {
            console.log(this.c)                
            console.log(this.id)
        }
    },
    
    fn: function() {
        return () => console.log(this.name)
    }
}

fobj.showcc()
fobj.obj.showcc()
fobj.obj.showiderr()
fobj.obj.showid()            
fobj.fn()()            
var f = fobj.obj.showid
f()            
```

Before revealing the answers, let's understand the fundamental concepts of `this`:

## The Nature of 'this' Keyword

1. **'this' is a keyword** - not a variable, not a property name, JavaScript syntax doesn't allow assigning values to 'this'
2. **'this' keyword has no scope restrictions** - nested functions don't inherit 'this' from their calling functions
3. **If a nested function is called as a method** - its 'this' points to the calling object
4. **If a nested function is called as a function** - its 'this' value is window (non-strict mode) or undefined (strict mode)

## Execution Context and 'this' Relationship

Now let's analyze each 'this' reference through detailed code comments:

```javascript
// Understanding scope chain, execution context, JavaScript execution process and 'this'指向
var fobj = {
    name: "G",
    showcc: function() {
        // Called as fobj's method, 'this' points to fobj
        console.log(this.name) // "G"
    },
    
    obj: {
        id: 39,
        // This 'this' is in global execution context, pointing to window (browser) or global (Node.js)
        // Note: object literals don't create new execution contexts
        c: this,                
        
        showcc: () => {
            // Arrow functions inherit 'this' from parent execution context
            // Parent obj is defined in global context, so 'this' points to window/global
            console.log(this.name) // undefined (browser) or undefined (Node.js)
            console.log(this) // window (browser) or {} (Node.js)
        },
        
        showiderr: () => {
            // Same as above, arrow function inherits global context's 'this'
            console.log(this.id) // undefined
        },
        
        showid: function() {
            // Regular function, 'this' determined by calling method
            console.log(this.c) // When called by obj, this.c points to window/global
            console.log(this.id) // When called by obj, outputs 39
        }
    },
    
    fn: function() {
        // Arrow function inherits 'this' from fn's execution context
        // When fn is called by fobj, 'this' points to fobj
        return () => console.log(this.name) // "G"
    }
}

// Execution result analysis
fobj.showcc()                // "G" - called as fobj's method
fobj.obj.showcc()            // undefined and window/{} - arrow function inherits global 'this'
fobj.obj.showiderr()         // undefined - arrow function inherits global 'this'
fobj.obj.showid()            // window/{} and 39 - called as obj's method
fobj.fn()()                  // "G" - arrow function inherits fn's 'this' (pointing to fobj)
var f = fobj.obj.showid      
f()                          // undefined and undefined - called as standalone function
```

## Actual Execution Results Verification

**Chrome Browser Execution Results:**
![Chrome Execution](https://github.com/GuoYuefei/storage/blob/master/js/images/1903/chrometest.png?raw=true)

**Node.js Environment Execution Results:**
![Node.js Execution](https://github.com/GuoYuefei/storage/blob/master/js/images/1903/nodetest.png?raw=true)

## Key Knowledge Points Summary

1. **Execution Context Types**:
   - Global Execution Context: 'this' points to window (browser) or global (Node.js)
   - Function Execution Context: 'this' depends on calling method

2. **Function Calling Methods Determine 'this'**:
   - Method Call: `obj.method()` - 'this' points to obj
   - Function Call: `function()` - 'this' points to window/global (non-strict mode)
   - Constructor Call: `new Constructor()` - 'this' points to newly created object
   - apply/call Call: `func.apply(obj)` - 'this' points to specified obj

3. **Arrow Function Particularities**:
   - Arrow functions don't have their own 'this', they inherit parent execution context's 'this'
   - Arrow function's 'this' is determined at definition time, won't change due to calling method

4. **Object Literal Particularities**:
   - Object literals themselves don't create new execution contexts
   - Using 'this' directly in object literals points to outer execution context's 'this'

## Learning Journey

By studying the phenomenon where using 'this' in properties of nested objects outputs window, I gained a deep understanding of JavaScript's execution context mechanism. The 'this' in object literals points to the execution context at definition time, not to the object itself.

This understanding is crucial for writing reliable JavaScript code, especially when using frameworks and libraries, enabling accurate prediction and control of 'this'指向 behavior.
