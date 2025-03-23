---
title: 前端VUE框架学习
abbrlink: 128e050c
date: 2025-03-23 19:59:29
updated: 2025-03-23 19:59:29
tags:
  - 前端框架
  - vue
  - JavaScript
categories: 前端
---

## 前言
工作需要，系统的学习一下vue，之前工作过程中也只是了解，并没有系统学习过，这里学习记录一下。

<!-- more -->
### 引用
- 核心语法引用B站Up：[吴悠讲编程](https://www.bilibili.com/video/BV1oj411D7jk/?spm_id_from=333.337.search-card.all.click&vd_source=1568f4ca7dfe468df4258e571170b468)
	- 入门看的这个，说的条理可以的，剩下的可以仔细研读vue的官方文档

## 一些概念
- VUE是一个渐进式的JavaScript的框架
- VUE支持组件化开发

## VUE核心语法
PS：对照下文的示例代码结合起来看

- 1. 响应式与插值表达式
	- 插值表达式：
		- 数据读取
		- 简单的逻辑计算
	- methods封装
	- 响应式：
		![一个响应式示例](128e050c/一个响应式示例.jpg)
- 2. 计算属性（conputed）：计算属性具有缓存特性
- 3. 侦听器：监听数据变动，主要是用在除了更新页面数据之外，还要处理一些额外的动作的时候，用到这个功能。
- 4. 内容指令：简化了一些元素处理方式（封装了）
	- 内容指令
	- 渲染指令
		一个v-if的示例
		![一个vif的示例](128e050c/一个vif的示例.jpg)

		一个vshow示例
		![一个vshow示例](128e050c/一个vshow示例.jpg)
	- 属性指令
	- 事件指令
		一个事件绑定的示例
		![一个事件绑定的示例](128e050c/一个事件绑定的示例.jpg)
	- 表单指令：只能使用在表单元素，v-modle来实现双向绑定效果。实例中是通过一个input框（数据）与一个p标签（视图），当绑定同一个响应式数据inputValue的时候，这时候就可以通过修改input内容，来同时可以和视图**联动变化**
- 5. 修饰符：实现一些与指令相关的常用操作，目标简化代码与DOM操作

```javascript
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="box"></div>
    <div id="app">
        <!-- 1.2 插值表达式 -->
        <p>{{ title }}</p>
        <p>{{ content }}</p>
        <p>{{ 1 + 2 + 3 }}</p>
        <p>{{ output() }}</p>
        <p>{{ output() }}</p>
        <p>{{ output() }}</p>
        <p>{{ outputContent }}</p>
        <p>{{ outputContent }}</p>
        <p>{{ outputContent }}</p>
        <!-- 4.1 内容指令 -->
         <!-- 这个跟以前的textContent和innerHtml功能是一致的 -->
        <p v-text="htmlContent">123</p>
        <p v-html="htmlContent">123</p>
        <!-- 4.2 渲染指令 -->
        <p v-for="item in 5">这是通过for渲染指令渲染出来的5个p标签</p>
        <!-- 注意这里的(item, key, index)，是渲染指令里面特有的方式，说白了可以灵活遍历list和map的方式
            其中，item是值，key是键，index索引序号 
        -->
        <p v-for="(item, key, index) in arr">这是v-for遍历响应式arr数组的结果：{{ item }}</p>
        <!-- 通过一个bool值来控制一个元素是否创建或者销毁 -->
        <p v-if="true">标签内容true显示</p>
        <!-- 对应的就是直接不创建标签 -->
        <p v-if="false">标签内容false</p>
        <!-- 对应的就是display: none -->
        <p v-show="false">标签内容false</p>

        <!-- 4.3 属性指令 -->
        <p title="标题">这是属性</p>
        <!-- 这个属性绑定就灵活了 -->
        <p v-bind:title="title">这是属性绑定的</p>
        <p :title="title">这是属性绑定的另外一种写法</p>

        <!-- 4.4 事件指令 -->
        <button v-on:click="output">这是一个按钮</button>
        <button @click="output">这是事件简写按钮</button>

        <!-- 4.5 表单指令 只能使用在表单元素，v-modle来实现双向绑定效果-->
        <input type="text" v-model="inputValue">
        <p v-text="inputValue"></p>

        <!-- 5. 修饰符 实现指令相关的常用操作 -->
         <!-- trim就是将两边空格去除 -->
        <input type="text" v-model.trim="inputValue">
    </div>
    <script src="./vue.min.js"></script>
    <script>
        // let value = '这是内容'
        // document.getElementById('box').textContent = value
        // value = '系内容'
        // document.getElementById('box').textContent = value
        // 1. 响应式数据与插值表达式
        const vm = new Vue({
            el: '#app',
            data () {
                return {
                   title: '这是标题内容！',
                   content: '这是标题文本！',
                   htmlContent: '这是一个<span>span</span>标签',
                   arr: ['a', 'b', 'c', 'd'],
                   inputValue: '默认输入内容'
                }
            },
            // 1.3 函数的使用
            methods: {
                output () {
                    console.log('methods执行了')
                    return '读取标题为：' + this.title + ', 内容为：' + this.content 
                }
            },
            // 2. 计算属性
            computed: {
                outputContent () {
                    console.log('computed执行了')
                    return '读取标题为：' + this.title + ', 内容为：' + this.content 
                }
            },
            // 3. 侦听器
            watch: {
                title (newValue, oldValue) {
                    console.log('新值:' + newValue, '旧值:' + oldValue)
                }
            }
        })
    </script>
</body>
 
</html>
```

## 脚手架与组件开发