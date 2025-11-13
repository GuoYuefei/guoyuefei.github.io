---
title: "从执行上下文角度理解JavaScript中的this指向"
date: 2019-03-13
draft: false
tags: ["javascript", "学习趣事", "编程技巧"]
categories: ["编程语言"]
keywords: ["javascript", "this", "执行上下文", "作用域链", "浏览器", "nodejs"]
description: "深入理解JavaScript中this关键字的指向机制"
---

# 从执行上下文角度理解JavaScript中的this指向

先来看一段代码，请尝试写出各个函数调用后的console输出：

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

在揭晓答案之前，我们先来理解this的基本概念：

## this关键字的本质

1. **this是关键字**，不是变量，不是属性名，JavaScript语法不允许给this赋值
2. **关键字this没有作用域限制**，嵌套的函数不会从调用它的函数中继承this
3. **如果嵌套函数作为方法调用**，其this指向调用它的对象
4. **如果嵌套函数作为函数调用**，其this值是window（非严格模式），或undefined（严格模式下）

## 执行上下文与this的关系

现在让我们通过详细的代码注释来分析每个this的指向：

```javascript
// 理解作用域链、执行上下文、JavaScript执行过程与this指向的关系
var fobj = {
    name: "G",
    showcc: function() {
        // 作为fobj的方法调用，this指向fobj
        console.log(this.name) // "G"
    },
    
    obj: {
        id: 39,
        // 这里的this是在全局执行上下文中，指向window（浏览器）或global（Node.js）
        // 注意：对象字面量不会创建新的执行上下文
        c: this,                
        
        showcc: () => {
            // 箭头函数继承父级执行上下文的this
            // 父级obj在全局上下文中定义，所以this指向window/global
            console.log(this.name) // undefined（浏览器）或undefined（Node.js）
            console.log(this) // window（浏览器）或{}（Node.js）
        },
        
        showiderr: () => {
            // 同上，箭头函数继承全局上下文的this
            console.log(this.id) // undefined
        },
        
        showid: function() {
            // 普通函数，this由调用方式决定
            console.log(this.c) // 当被obj调用时，this.c指向window/global
            console.log(this.id) // 当被obj调用时，输出39
        }
    },
    
    fn: function() {
        // 箭头函数继承fn执行上下文的this
        // 当fn被fobj调用时，this指向fobj
        return () => console.log(this.name) // "G"
    }
}

// 执行结果分析
fobj.showcc()                // "G" - 作为fobj的方法调用
fobj.obj.showcc()            // undefined 和 window/{} - 箭头函数继承全局this
fobj.obj.showiderr()         // undefined - 箭头函数继承全局this
fobj.obj.showid()            // window/{} 和 39 - 作为obj的方法调用
fobj.fn()()                  // "G" - 箭头函数继承fn的this（指向fobj）
var f = fobj.obj.showid      
f()                          // undefined 和 undefined - 作为独立函数调用
```

## 实际运行结果验证

**Chrome浏览器下运行结果：**
![Chrome下运行](https://github.com/GuoYuefei/storage/blob/master/js/images/1903/chrometest.png?raw=true)

**Node.js环境下运行结果：**
![Node.js下运行](https://github.com/GuoYuefei/storage/blob/master/js/images/1903/nodetest.png?raw=true)

## 关键知识点总结

1. **执行上下文类型**：
   - 全局执行上下文：this指向window（浏览器）或global（Node.js）
   - 函数执行上下文：this取决于调用方式

2. **函数调用方式决定this**：
   - 方法调用：`obj.method()` - this指向obj
   - 函数调用：`function()` - this指向window/global（非严格模式）
   - 构造函数调用：`new Constructor()` - this指向新创建的对象
   - apply/call调用：`func.apply(obj)` - this指向指定的obj

3. **箭头函数的特殊性**：
   - 箭头函数没有自己的this，继承父级执行上下文的this
   - 箭头函数的this在定义时确定，不会因调用方式改变

4. **对象字面量的特殊性**：
   - 对象字面量本身不创建新的执行上下文
   - 在对象字面量中直接使用this，指向的是外层执行上下文的this

## 学习心得

通过研究对象中对象的属性使用this时输出window的现象，可以让我们深刻理解了JavaScript的执行上下文机制。对象字面量中的this指向的是定义时的执行上下文，而不是对象本身。

这种理解对于编写可靠的JavaScript代码至关重要，特别是在使用框架和库时，能够准确预测和控制this的指向行为。
