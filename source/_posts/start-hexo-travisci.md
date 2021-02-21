---
title: Hexo与TravisCI持续集成
abbrlink: e62993b1
date: 2021-02-21 22:18:07
tags:
    - 站点
    - Hexo
    - TravisCI
categories: 站点
---

## 背景
Hexo个人博客满打满算也算是建站有2~3年了，原先自己的写作的方式是：
- 有多平台协作的软件：例如为知笔记、iOS自带的备忘录、OneNote等等，这些多平台协作的软件，供自己日常写稿子所需。
- 一个平台编译发布：家中的Windows台式机
- 整个Hexo工程目录丢在OneDrive中同步

这样的模式，存在几个问题：
<!-- more -->
- 协作协作并不方便，其实根本没有做到随时随地编写发布到博客网站的可能
- 一个平台编译：多平台编译其实可以，但是存在问题的是，也是之前碰到过的一种情况。
我在自己办公windows的备机上也搭建了Hexo环境，也能编译。也登录了同步OneDrive账号，同步了Hexo工程目录。也能编译，也能发布。但是由于某天，不知道什么鬼原因，两台电脑同步出了问题，导致OneDrive自己给所有文件通通重命名了，所有文件名称中加入了计算机名称，这个让我抓狂的。
后来就是全部删了，重新hexo init，重新搞了一遍。
- 跨平台可能未必行：在windows编译的，跑到Mac或者Linux环境下编译，能不能搞定，这个还没试过。不过也能可想而知，会出奇怪的问题。

### 为啥我选择了TravisCI
也正是因为上述主要的三个问题，导致了，必须得更新同步方式。原先想，自己在家中的服务器上，搭建编译环境，也不是不可以，说白了也就是写一个Shell脚本嘛。不过后来放弃了，**最主要的一个问题是：文章怎么同步？** 我换电脑，写的东西怎么同步到环境里？更别说，如果要用手机呢？
后来，在百度和Google找来找去，发现很多提到的一个持续集成的工具TravisCI。
在英文版的Hexo官网中，提到了很多现在一键部署的方式，有兴趣可以点击下方传送门：
https://hexo.io/docs/one-command-deployment

## 前置须知
### 涉及到的相关组件版本
我的Hexo用的最新的。
我的Hexo主题也是用的最新的，当前版本是：8.2.1
Node JS版本是：v12.20.2
### 关于TravisCI
一句话解释Travis CI干嘛的，说白了就是一个在线编译环境，如果你玩过Jekins，这玩意儿就很好懂了。具体细节，真的建议，看官方文档就好。
https://docs.travis-ci.com/
挑一个你了解的开发语言，如Java、Python什么的，去看看示例就好了。
### 关于我的站点结构
- 文章托管自然是GitHub，仓库的名字就是：nimbusking.github.io。PS：别折腾自定义的仓库名字了，你要是改了，后面又要折腾不少配置，尤其是绑定了自定义域名的。
- 绑定了一个自己的主站域名，就是你所见到的。
- 开启了HTTPS，我交给了[Netlify](https://www.netlify.com/)托管，Netlify证书是通过Let’s Encrypt自签的。严格意义上说，现在我的文章都是部署在Netlify的.
- 域名DNS绑到了DNSPOD
- 无CDN（未来也不会加）
- Hexo主题用的是[NexT](https://github.com/next-theme/hexo-theme-next)，版本8.2.1，新版本仓库已经迁移了，集成到npm了，而且主要使用了Numjucks
以上就是主要的一些情况的补充说明

### 关于Hexo与TravisCI集成
折腾了，差不多1天吧，才搞通了。中间在一个问题上纠结了很久，把遇到的主要情况，先行说明一下。
网上百度和Google中搜了很多，很多都试了，都是不行，无法编译通过。后来不经意在，Hexo韩文官网下藏了一篇关于TravisCI集成的文章（很奇怪，为毛英文主站没有，还好Google收录了，让我搜到了）。
文章地址：https://hexo.io/ko/docs/github-pages.html
怕日后没了，我离线了，在附件中可以查看。
其实最主要的问题就一个：**让TravisCI拉取的仓库中放哪些文件？**这点很多别人的文章中，并没有明确说明，导致我自己也是绕了好大一个弯。
这点在官网的文章里，其实有详细说明的：
{% note primary%}
Push the files of your Hexo folder to the repository. The public/ folder is not (and should not be) uploaded by default, make sure the .gitignore file contains public/ line. The folder structure should be roughly similar to this repo, without the .gitmodules file
{% endnote %}



## 准备工作

## 集成步骤

## 一点不一样的修改

## 附件
- ![Hexo官网中关于TravisCI集成的相关说明](e62993b1/hexo github pages.7z)