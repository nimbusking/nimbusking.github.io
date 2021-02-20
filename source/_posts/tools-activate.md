---
title: 工具激活相关
mathjax: false
abbrlink: c0604aeb
date: 2018-05-05 12:24:03
tags: 工具激活
categories: 工具
---

### 前言
本系列内容均搬自互联网，仅供个人学习使用，勿用作商业用途。
若资金允许，请支持正版，谢谢！

### 开发类
#### IDEA
##### 服务器激活方式
目前可用（部分失效）激活地址，如果一个失效选择另外一个服务器，若都失效，尝试离线文件激活方式。
http://kadara.ru:1017
http://idea.imsxm.com/
http://www.aku.vn/idea
http://jetbrains.tech/
http://idea.imsxm.com/
http://idea.iteblog.com/key.php
<!--more-->
##### 离线破解文件激活方式
[大佬主页](http://idea.lanyus.com/)
步骤：
+ 安装最新旗舰版(Ultimate)的IDEA，先不要立马运行
+ 下载大佬主页的jar破解文件，例如：*JetbrainsCrack-2.7-release-str.jar*
+ 修改安装目录下的bin文件夹内的*idea.exe.vmoptions*和*idea64.exe.vmoptions*两个配置文件（俩均是文本文件），在文件最后添加一行：
```shell
-javaagent:[full path]/JetbrainsCrack-2.7-release-str.jar
```
注意：[full path]是你下载的jar文件，在你本地计算机下的绝对路径
+ 启动IDEA，如果显示Register界面，选择Activate Code选项卡，内容随便输一个字母即可，点击OK即可。如果没有显示Register页面，Help->Register，打开即可。如下图所示：
![激活页面](c0604aeb/idea_register_window.jpg)
+ 最后激活成功，搞定愉快码代码吧！
![激活结果](c0604aeb/idea_201801激活成功.jpg)

#### JRebel
牛X的真·热部署插件，非常好用！
[官网](https://zeroturnaround.com/software/jrebel/)，个人版，一年600+刀
以IDEA为例，其它平台类似：
+ IDEA下装好JRebel插件：Settings->Plugins下直接搜即可
+ 下载本地激活项目，移步[国内一大佬的开源激活项目](https://gitee.com/gsls200808/JrebelLicenseServerforJava)
+ 使用Maven命令编译，编译步骤大佬项目主页都写了，按照步骤执行即可。
+ 运行编译好的JAR
+ 打开Help->JRebel->Activation，如下图所示
![JRebel激活界面](c0604aeb/jrebel_activate_window.png)
    - 填写Connect to Server：```http://localhost:8081/UUID```，其中UUID在网上随便找一个UUID生成器网站即可。
        提供一个：[uuidgenerator](https://www.uuidgenerator.net/)
    - 填写第二行Email地址，随便填
+ 点击Activate按钮（第一次激活），后续再次激活则变成：Change License按钮，道理一样。
+ 等待激活成功，重启IDEA，打开Settings->JRebel，查看激活状态，绿色的VALID，激活成功
![JRebel激活成功](c0604aeb/jrebel_activated_successfully.jpg)
**点击Work Offline按钮**，一定要进入离线模式，防止在线激活验证导致激活失效。
图中显示本次离线周期到2018年11月，半年左右
+ OK结束
