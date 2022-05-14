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
<!-- more -->
### 准备工作
如果你没有从事过开发相关工作/没有学习过计算机编程、网络相关知识，下面提到的，建议还是花点时间了解一下，不必了解那么深。
- nodejs/npm相关语法
- hexo及hexo相关语法，这个参考官网文档相关说明即可：https://hexo.io/docs/
- git仓库/github
- 域名/域名CDN记录
- CDN加速
- HTTPS

### 我的目前的方案
- markdown源文章、相关配置文件，托管在 **Github**
- Hexo自动编译&部署相关，托管在 **TravisCI**
- 站点、站点https证书等部署相关，托管在 **Netlify**
#### 这个方案目前的优点
- 编译环境不再依赖你的本地环境，你只需要关注你的文章。即便是换了电脑，你只需要把你的文章，从github上clone下来即可。
- 站点不需要你关系细节，什么HTTPS、什么证书等等
#### 我的Hexo站点使用到的插件
```nodejs
npm install hexo-theme-next --save
npm install hexo-deployer-git --save
npm install hexo-generator-feed
npm install hexo-wordcount --save
npm install hexo-abbrlink --save
npm install lozad --save
npm uninstall hexo-generator-index --save
npm install hexo-generator-index-pin-top --save
npm install hexo-algolia --save
```

### 开始建站
关于hexo语法，插件什么的，这里不再赘述，瞅瞅官网什么，再本地init一个测试blog即可，语法不是很复杂。常用的，就那么几个。
#### github托管源码
先上我的仓库的图：
![我的博客源码目录结构](178a5861/github.png)

最主要的就是：source目录下的文章
其余有几个配置文件，分别说明一下：
- .gitignore，git提交的忽略配置文件
- .travis.yml, travis的部署脚本
- config.next.yml next主题的配置文件
- config.yml 博客的主体配置文件

##### travis部署脚本
在这里贴一下，细节看我另外一篇文章

```yml
language: node_js
node_js:
  - 12

cache: npm
branches:
  only:
    - master # build master branch only
# 设置缓存文件
cache:
  directories:
    - node_modules

install:
  - npm install
  - npm install hexo-theme-next --save
  - npm install hexo-deployer-git --save
  # - npm install hexo-generator-searchdb --save
  - npm install hexo-generator-feed
  - npm install hexo-wordcount --save
  - npm install hexo-abbrlink --save
  - npm install lozad --save
  - npm uninstall hexo-generator-index --save
  - npm install hexo-generator-index-pin-top --save
  - npm install hexo-algolia --save

before_script:
  # 替换同目录下的_config.yml文件中github_token字符串为travis后台刚才配置的变量，注>意此处sed命令用了双引号。单引号无效！
  - sed -i "s/algolia_applicationID/${ALGOLIA_APPLICATIONID}/g" ./_config.yml
  - sed -i "s/algolia_apiKey/${ALGOLIA_APIKEY}/g" ./_config.yml
  - sed -i "s/algolia_indexName/${ALGOLIA_INDEXNAME}/g" ./_config.yml
  - sed -i "s/gitalk_client_id/${GITALK_CLIENT_ID}/g" ./_config.next.yml
  - sed -i "s/gitalk_client_secret/${GITALK_CLIENT_SECRET}/g" ./_config.next.yml
  # - pwd
  # clone自定义配置
  - git clone -b customer_config https://github.com/nimbusking/nimbusking.github.io.git temp
  - cp -f ./temp/post-meta.njk ./node_modules/hexo-theme-next/layout/_partials/post
  - rm -rf ./temp

script:
  # - hexo clean
  - hexo generate
  - hexo algolia
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  on:
    branch: master
  local-dir: public
```
#### TravisCI
细节不再这里说明，可以看我另一篇文章中的详细说明：[Hexo与TravisCI持续集成](https://nimbusk.cc/post/e62993b1.html)
#### 准备
原理基本上基于放弃使用github.io托管博客静态页面，寻找第三方支持全站启用https的站点。
参考[掘金网](https://juejin.im/entry/5a8bd9f25188257a5911cfc6)对比比较内容
#### Netlify
官网：http://www.netflix.com
最大的特点：支持同步Github仓库数据、免费启用Let's Encrypt SSL证书

##### 效果

### 其它
