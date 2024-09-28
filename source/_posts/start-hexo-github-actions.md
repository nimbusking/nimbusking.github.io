---
title: Hexo与GitHub Actions持续集成
abbrlink: 9bb5c5d6
date: 2024-09-28 15:11:16
tags:
    - 站点
    - Hexo
    - Nodejs
    - GitHub Actions
categories: 站点
---

## 背景
以前本博客是借助Travis来进行CI的，配置NodeJs及Hexo环境自动编译等。有兴趣，可以看看，该文章：https://nimbik.cc/post/e62993b1.html
（也是惭愧，有1年有余没有跟新博客了，导致Travis收费都不知道，今天开始更新的时候，发现远端编译失败了。。。。提示要收费）
遂，果断迁移之GitHub Actions。

<!-- more -->

## 开始之前
找到本篇文章之前，假设看官您已经对CI相关工作原理已经熟悉了，且对下面的一些使用场景比较熟悉了（原谅我懒，有问题请leave a comment）
- GitHub的ssh配置过程
- Hexo的编译过程
- 一些简单的Linux shell

## 准备过程
准备一组新的github的deploy公私钥，通过ssh-keygen生成即可，后面需要用到。

## GitHub Actions配置过程
在配置actions脚本的时候，需要先配置两处东西
### 仓库Settings界面
点开你的仓库，点开Settings
![仓库Settings](9bb5c5d6/Github Repository Setttings.png)

注意圈出来的两个地方，我们需要先设置一下
1. Deploy keys
	这个地方，将刚才*准备过程*中新生成的公钥（id_rsa.pub文件中），不漏的copy上，完了点击右上角的“Add depoly key”之后，在下图中填上，标题随便，Key就是刚才你粘贴的那个公钥内容。
	![AddDeployKeys](9bb5c5d6/AddDeployKeys.png)
2. Secrets and variables
	这个里面主要设置一些变量相关的内容，步骤不多赘述。这里主要是配置一些，在hexo配置文件中使用到的一些变量相关的值，特别是一些插件的client_id，授权密钥相关的敏感信息都可以放在这个里面配置。在后面配置脚本的时候，可以看到在哪里用到。
	![secrets and variables](9bb5c5d6/secrets and variables.png)

	其中有一个变量，DEPLOY_KEY是上面生成的私钥（id_rsa文件中），完整复制进去即可。

### 脚本编写
这部分不用过多赘述，直接看我的脚本注释里面即可。
有两种方式创建脚本，一种通过仓库页面选项卡的上方“Actions选项卡”进行创建；另一种是你直接创建相关的yml文件，路径放在你博客源码仓库页面的：.github/workflows/xxx.yml即可。文件名自定义。

下面是我的脚本内容（包含了若干自己博客所需的hexo插件，你按需更改相关命令即可）：
```yaml
name: hexo_blog_audo_deploy # actions名称，自定义即可
on:
  push:
    branches:
      - master # 监听分支，注意你的分支名称，特别是github新建的仓库，是main，不是master
  release:
    types:
      - published

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: 检查分支
      uses: actions/checkout@v2
      with:
        ref: master # 同上

    - name: 安装 Node
      uses: actions/setup-node@v1
      with:
        node-version: "12.x" # 按需选择你的hexo运行的编译的nodejs版本

    - name: 安装 Hexo
      run: |
        export TZ='Asia/Shanghai'
        npm install hexo-cli -g --save

    - name: 缓存 Hexo
      id: cache
      uses: actions/cache@v1
      env:
        cache-name: cache-node-modules
      with:
        path: node_modules
        key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
            ${{ runner.os }}-build-${{ env.cache-name }}-
            ${{ runner.os }}-build-
            ${{ runner.os }}-

    - name: 安装依赖
      if: steps.cache.outputs.cache-hit != 'true'
      run: |
        npm install hexo-theme-next --save
        npm install hexo-deployer-git --save
        npm install hexo-generator-searchdb --save
        npm install hexo-generator-feed
        npm install hexo-wordcount --save
        npm install hexo-abbrlink --save
        npm install lozad --save
        npm uninstall hexo-generator-index --save
        npm install hexo-generator-index-pin-top --save
        npm install --save

    - name: 处理相关变量
      run: |
      	# 先将主题配置文件中的关于gitalk的敏感信息替换成实际在上述settings中配置的，调用方式就是 **secrets.你刚才配置的变量名称**（我使用的next主题，按你实际使用的主题配置更改）
        sed -i 's/GITALK_CLIENT_ID/${{ secrets.GITALK_CLIENT_ID }}/g' ./_config.next.yml
        sed -i 's/GITALK_CLIENT_SECRET/${{ secrets.GITALK_CLIENT_SECRET }}/g' ./_config.next.yml
        # 处理自定义配置（这里主要是处理一个插件里面的内容，我需要自定义一些玩意儿，所以我这么处理了，你没有这个需求大可删了下面三行）
        git clone -b customer_config https://github.com/nimbusking/nimbusking.github.io.git temp
        cp -f ./temp/post-meta.njk ./node_modules/hexo-theme-next/layout/_partials/post
        rm -rf ./temp
        
    - name: 配置环境
      env:
        DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }} # 这里获取仓库提交公钥，后面用到
      run: |
        mkdir -p ~/.ssh/
          echo "$DEPLOY_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.email "useremail" 
          git config --global user.name "username"
          ssh-keygen -y -f ~/.ssh/id_rsa

    - name: 部署 Nimbusk博客
      run: |
        hexo clean
        hexo d
```

没写那么细致，一个简单的actions脚本就完成了，不是特别复杂。更高级的actions语法，需要你参考actions具体教程。