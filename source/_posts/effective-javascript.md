---
title: effective-javascript
mathjax: false
tags: Effective JavaScript
updated: 2024-10-28 20:30:21categories: JavaScript系列
abbrlink: ad5d9134
date: 2018-11-11 17:11:52
---

#### 前言
- 本文默认JS即为JavaScript语言
- “注：”为本人理解内容

#### 第2条：理解JavaScript的浮点数
- 不管是整数还是浮点数，JavaScript都将它们简单归类为数字。事实上，所有的数字都是**双精度浮点数**
- 位运算符将数字视为32位的有符号整数
- 当心浮点运算中的精度陷阱

```javascript
(0.1 + 0.2) + 0.3; // 0.6000000000000001
0.1 + (0.2 + 0.3); // 0.6
// 一个有效的解决方式是将浮点数转化为整数运算，例如上例中可以转换为
(10 + 20) + 30; // 60
10 + (20 + 30); // 60
```
#### 第3条 当心隐式强制转换
- 特别注意字符强制转换的问题：结果为null的变量在算术运算中不会导致失败，而是被隐式转换为0
- 测试某个值是否是NaN，在JS中唯一一个不等于其自身的值，因此，可以随时通过检查一个值是否等于其自身的方式来测试该值是否是NaN

```javascript
var a = NaN;
a !== a; // true
var b = "foo";
b != b; // false
```
#### 第5条：避免对混合类型使用==运算符
一个好的替代方法是使用严格相等运算符
![==运算符的强制转换规则](ad5d9134/table1.1.jpg)
<!--more-->
#### 分号插入局限性
分号插入第一条规则：分号仅在|标记之前、一个或多个换行之后和程序输入的结尾被插入。
```javascript
// 合法
function square(x) {
    var n = +x
    return n * n
}
function area(r) { r = +r; return Math.PI * r * r}
function add1(x) { return x + 1}
// 不合法
function area(r) { r = +r return Math.PI * r * r }
```
分号插入第二条规则：分号仅在随后的输入标记不能解析时插入，也可以理解为是一种错误矫正机制。
```javascript
a = b
(f());
// 能正确解析为一条单独的语句，等价于：a = b(f());
a = b
f();
// 则会解析为两条独立的语句
```
注：完全没必要用这个特性，知道有这么个玩意儿就行，但是在实际开发过程中，不要有这种特性的东西，不容易理解，还容易出错。

#### 第8条：尽量少用全局对象
#### 第9条：避免使用with语句
#### 熟练掌握闭包
闭包第一个基本事实：JS允许你引用在当前函数意外定义的变量
```javascript
function makeSandwich() {
    var magicIngredient = "peanut butter";
    function make(filling) {
        return magicIngredient + " and " + filling;
    }
    return make("jelly");
}
makeSandwich(); // "reanut butter and jelly"
```
第二个事实：即使外部函数返回，当前函数仍然可以引用在外部函数所定义的变量。
```javascript
function sandwichMaker() {
    var magicIngredient = "peanut butter";
    function make(filling) {
        return magicIngredient + " and " + filling;
    }
    return make;
}
var f = sandwichMaker();
f("jelly");

// 改进版本
function sandwichMaker(magicIngredient) {
    function make(filling) {
        return magicIngredient + " and " + filling;
    }
    return make;
}
var hamAnd = sandwichMaker("ham");
hamAnd("cheese");
// 提供直接匿名返回的方式
function sandwichMaker(magicIngredient) {
    return function(filling) {
        return magicIngredient + " and " + filling;
    };
}
```

第三个事实：闭包可以更新外部变量的值。
```javascript
function box() {
    var val = undefined;
    return {
        set: function(newVal) { val = newVal; },
        get: function() { return val; },
        type: function() { return typeof val; }
    };
}
var b = box();
b.type(); // "undefined"
b.set(98.6);
b.get(); // 98.6
b.type(); // "number"
```

#### 第12条：理解变量声明提升
JS不支持块级作用域，即变量定义的作用域不是离其最近的封闭语句或代码块中，而是包含它们的函数。当然try...catch语句是一个例外。
```javascript
function test() {
    var x = "var", result = [];
    result.push(x);
    try {
        throw "exception";
    } catch (x) {
        x = "catch";
    }
    result.push(x);
    return result;
}
```

