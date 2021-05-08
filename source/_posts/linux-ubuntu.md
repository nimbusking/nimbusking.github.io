---
title: Ubuntu相关系列
mathjax: false
abbrlink: '145818e1'
date: 2019-05-16 02:20:17
tags:
    - Ubuntu
    - Linux
categories: 操作系统
---

## 背景
又是一个熬夜的夜晚，凌晨2点半，新晋小备机工作站基本搞定了。 Yeah!

## Ubuntu安装与初始化配置
### 安装
### 机械硬盘挂载
### 切换数据源为阿里云

```shell
#备份source.list
cp -f /etc/apt/sources.list /etc/apt/sources.list.bak
```

通过vi/vim修改sources.list
将原有的都注释了，把下面的数据源新增在文件末尾即可：
```shell
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

保存之后，记得：**sudo apt-get update**
更新数据源即可

### 首次登录设置root密码
- 打开终端，输入**sudo passwd root**，
在[sudo] password for wheat:后输入当前用户的密码

- 当前用户密码验证通过后
输入需要设置的root超级管理员账户密码Enter new UNIX password:
验证输入的密码Retype new UNIX password:

- 出现**passwd: password updated successfully**字样，表示超级管理员root账户密码设置成功
- 验证：输入su，后输入超级管理员账户的密码，验证通过则成功切换到root账户root

## 系列话题
### 在ubuntu下编译openjdk
编译方法来自于《深入理解JAVA虚拟机 第三版》，详细请参考该书中第一章，1.6小节的相关说明
## 相关命令
## 参考