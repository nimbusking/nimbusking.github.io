---
title: Hexo建站
mathjax: false
abbrlink: 178a5861
date: 2018-08-12 02:40:53
tags:
    - 站点
    - Hexo
categories: 站点
---

### 背景
至于为什么要自建博客，想必你能找到这篇hexo教程，之前肯定检索过不少信息了 :) 就不过多赘述了。
### 准备工作
如果你没有从事过开发相关工作/没有学习过计算机编程、网络相关知识，下面提到的，建议还是花点时间了解一下，不必了解那么深。
- nodejs/npm相关语法
- hexo及hexo相关语法
- git仓库/github
- 域名/域名CDN记录
- CDN加速
- HTTPS

npm install hexo-cli -g --save
npm install hexo-deployer-git --save
### 开始建站
### 站点启用HTTPS
#### 准备
原理基本上基于放弃使用github.io托管博客静态页面，寻找第三方支持全站启用https的站点。
参考[掘金网](https://juejin.im/entry/5a8bd9f25188257a5911cfc6)对比比较内容
#### Netlify
官网：http://www.netflix.com
最大的特点：支持同步Github仓库数据、免费启用Let's Encrypt SSL证书
##### 绑定流程

##### 添加自定义域名
##### 启用全站HTTPS
##### 效果
