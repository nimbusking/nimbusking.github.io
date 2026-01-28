---
title: Ubuntu相关系列
mathjax: false
abbrlink: 1458180
date: 2019-05-16 02:20:17
tags:
    - Ubuntu
    - Linux
updated: 2025-12-30 02:20:17
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

### 安装jdk并一键切换版本
首次运行下面命令，会显示下面的内容
```shell
kemi@datacenter:~$ java -version
Command 'java' not found, but can be installed with:
sudo apt install openjdk-17-jre-headless  # version 17.0.17+10-1~24.04, or
sudo apt install openjdk-21-jre-headless  # version 21.0.9+10-1~24.04
sudo apt install default-jre              # version 2:1.17-75
sudo apt install openjdk-11-jre-headless  # version 11.0.29+7-1ubuntu1~24.04
sudo apt install openjdk-25-jre-headless  # version 25.0.1+8-1~24.04
sudo apt install openjdk-8-jre-headless   # version 8u472-ga-1~24.04
sudo apt install openjdk-19-jre-headless  # version 19.0.2+7-4
sudo apt install openjdk-20-jre-headless  # version 20.0.2+9-1
sudo apt install openjdk-22-jre-headless  # version 22~22ea-1
```

提示你找不到java命令，需要安装。这里就是ubuntu仓库中各个版本的代号，按需安装即可。例如我默认安装的是jdk8，即：
```shell
sudo apt install openjdk-8-jre-headless
# 再次执行
kemi@datacenter:~$ java -version
openjdk version "1.8.0_472"
OpenJDK Runtime Environment (build 1.8.0_472-8u472-ga-1~24.04-b08)
OpenJDK 64-Bit Server VM (build 25.472-b08, mixed mode)
```

#### 安装多个版本jdk
通过如下命令，可以查看当前安装的jdk版本
```shell
kemi@middileware:~$ sudo dpkg -l | grep 'jdk\|jre'
ii  openjdk-11-jre-headless:amd64        11.0.29+7-1ubuntu1~24.04                amd64        OpenJDK Java runtime, using Hotspot JIT (headless)
ii  openjdk-8-jre-headless:amd64         8u472-ga-1~24.04                        amd64        OpenJDK Java runtime, using Hotspot JIT (headless)
```

通过下述命令，可以切换不同版本的jdk
```shell
kemi@middileware:~$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      manual mode
* 2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 
```

**如上所示：我选择了2，则前面会有一个星号，代码当前默认的版本，再次通过java -version可以确认**

#### 配置JAVA_HOME
直接
```shell
sudo vim /etc/environment
# 在下面添加一行即可，具体路径就是上面选择默认版本是第二列中间的
JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java"
```

配置好之后，记得刷新
```shell
source /etc/environment
echo $JAVA_HOME
```

#### 卸载jdk
- 找到JDK包：`sudo dpkg -l | grep 'jdk\|jre'`
- 使用以下命令卸载这些包，将包名替换为你找到的实际包名：
    ```shell
    sudo apt purge default-jdk default-jdk-headless default-jre default-jre-headless openjdk-21-jdk openjdk-21-jdk-headless openjdk-21-jre openjdk-21-jre-headless
    ```
- 卸载完成后，使用以下命令清除剩余的依赖项：
    ```shell
    sudo apt autoremove --purge
    ```
- 从/etc/environment文件中删除包含JAVA_HOME变量的行，并保存文件。

### 碰到通过useradd之后的用户问题
执行su命令之后，使用的是dash而不是bash，如下所示：
```shell
# 结尾是sh
kemi@microservice:~$ grep nexus /etc/passwd
nexus:x:1001:1001::/home/nexus:/bin/sh
# 通过如下命令修改
sudo usermod -s /bin/bash nexus
kemi@microservice:~$ grep nexus /etc/passwd
nexus:x:1001:1001::/home/nexus:/bin/bash

```

### 设置常见时区
```shell
# 中国标准时间
sudo timedatectl set-timezone Asia/Shanghai

# 美国东部时间
sudo timedatectl set-timezone America/New_York

# 欧洲伦敦时间
sudo timedatectl set-timezone Europe/London

# 日本东京时间
sudo timedatectl set-timezone Asia/Tokyo

# 协调世界时（UTC）
sudo timedatectl set-timezone UTC

# 打开NTP同步
sudo timedatectl set-ntp true
```

### 为自定义添加的用户添加sudo权限

```shell
# 方法A：添加到 sudo 组（推荐）
usermod -aG sudo nexus
```

### 固定IP
直接修改netplan

```shell
vim /etc/netplan/50-cloud-init.yaml
```

修改成如下形式，ip按照你的实际来

```yaml
network:
  version: 2
  ethernets:
    ens34:
      dhcp4: false
      addresses: [your_ip/24]
      routes:
        - to: default
          via: 你的网关地址，一般就是你的路由器ip
      nameservers:
          addresses: [你的DNS地址，默认就是路由器网关ip即可]
          search: []
```

修改之后：```netplan apply``` 生效
在通过如下命令检查配置是否生效：
```shell
ip addr show
ip route show
ping www.baidu.com
```

### 安装mysql
在线安装
#### 确认ubuntu系统版本
```shell
kemi@datacenter:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.3 LTS
Release:        24.04
Codename:       noble
```

#### 更新国内镜像源
不然 sudo apt update 可能会失败。

- 备份原来的镜像文件
    ```shell
    sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/ubuntu.sources.backup
    ```
- 编辑 /etc/apt/sources.list.d/ubuntu.sources 文件
    ```shell
    sudo vim /etc/apt/sources.list.d/ubuntu.sources
    ```
- 替换镜像源后的内容
    ```shell
    Types: deb
    URIs: http://mirrors.aliyun.com/ubuntu/
    Suites: noble noble-updates noble-backports
    Components: main restricted universe multiverse
    Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

    Types: deb
    URIs: http://mirrors.aliyun.com/ubuntu/
    Suites: noble-security
    Components: main restricted universe multiverse
    Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

    ```
#### 更新软件包
```shell
# 更新本地的“软件包索引缓存”，获取镜像源地址中最新的软件版本
sudo apt update

# 更新升级 apt update 得到的新版本，更新安装这些新版本到我们的服务器上
sudo apt upgrade -y

```

#### 手动添加官方 MySQL APT 源
官网地址：
https://dev.mysql.com/doc/refman/8.4/en/linux-installation-apt-repo.html

- 下载最新的apt配置包：
    文件版本号直接在下面的网址中查看即可：
    https://dev.mysql.com/downloads/repo/apt/
    ```shell
    wget https://dev.mysql.com/get/mysql-apt-config_0.8.36-1_all.deb
    ```
- 安装 MySQL APT 配置包
    ```shell
    sudo dpkg -i mysql-apt-config_0.8.36-1_all.deb
    ```
    可以确认下面的安装内容，如果需要选择特定版本：
    ![apt配置页面](1458180/config_mysql_apt_config.jpg)
    一般默认即可：
    ![选择指定版本](1458180/select_target_version_of_mysql.jpg)

- 按上下键把光标移动到 OK，按回车保存退出。
- 更新 APT 缓存
    ```shell
    sudo apt update
    ```

- 重新查询本地可以用的MySQL版本
    ```shell
    apt-cache madison mysql-server
    kemi@datacenter:~$ apt-cache madison mysql-server
    mysql-server | 8.4.8-1ubuntu24.04 | http://repo.mysql.com/apt/ubuntu noble/mysql-8.4-lts amd64 Packages
    mysql-server | 8.0.44-0ubuntu0.24.04.2 | http://mirrors.aliyun.com/ubuntu noble-updates/main amd64 Packages
    mysql-server | 8.0.44-0ubuntu0.24.04.1 | http://mirrors.aliyun.com/ubuntu noble-security/main amd64 Packages
    mysql-server | 8.0.36-2ubuntu3 | http://mirrors.aliyun.com/ubuntu noble/main amd64 Packages
    mysql-community | 8.4.8-1ubuntu24.04 | http://repo.mysql.com/apt/ubuntu noble/mysql-8.4-lts Sources
    ```

#### 安装 MySQL Server
安装指定版本的mysql
```shell
sudo apt install -y mysql-server=8.4.8-1ubuntu24.04
```

如下图所示，设置好root密码即可安装：
![设置root密码](145818e1/set_root_password.jpg)

#### 卸载MySQL
- 先查询mysql所有已安装的包
    ```shell
    dpkg -l | grep mysql
    ```
    如下图所示：
    ![所有安装版本](1458180/all_setup_version_of_mysql.jpg)

- 使用 sudo apt-get remove 命令，卸载掉上面查询到的所有安装的包
    ```shell
    sudo apt-get remove --purge -y mysql-apt-config mysql-server mysql-client mysql-common mysql-community-server mysql-community-client mysql-community-client-core mysql-community-server-core mysql-community-client-plugins
    ```

- 删除配置文件和数据文件
    ```shell
    sudo rm -rf /etc/mysql /var/lib/mysql /var/log/mysql /var/log/mysql.*
    ```

- 清理无用依赖和缓存
    ```shell
    sudo apt-get -y autoremove
    sudo apt-get -y autoclean
    ```
- 查询是否还有mysql的包

#### 远程连接mysql
修改 mysqld.cnf 配置文件：
```shell
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

修改 bind-address 属性，属性不存在就添加在[mysqld]块中（如果只需要指定的 ip 访问，使用防火墙控制）
```shell
bind-address = 0.0.0.0
# 端口，如果想修改端口的话，就修改 port 的值
# port = 3306

# 最大连接数量，默认是151，超过这个数量会报 Too many connections 错误
# max_connections = 500    # 使用 SHOW PROCESSLIST 命令查看连接了多少个数量了

# 空闲连接最长 sleep 时间为 300秒(5分钟)，通过 SHOW PROCESSLIST 查看 sleep
# wait_timeout = 300    # 默认是28800秒(8小时)，wait_timeout 主要针对非交互客户端的连接，连接池的连接就是非交互客户端

# interactive_timeout = 300     # 默认是28800秒(8小时)，interactive_timeout 针对的是交互客户端，比如命令行连接的方式
```

重启服务后，通过如下命令连接mysql
```shell
sudo mysql -u root -p
# 登录指定的 ip 和 端口
# mysql -u root -p -h 127.0.0.1 -P 3306

```

创建指定用户并授权
```sql
# 把username换成你所需的用户名即可
CREATE USER 'username'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'username'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

#### 其它MySQL相关命令
查看 MySQL 服务状态：`sudo systemctl status mysql`
停止 MySQL 服务：`sudo systemctl stop mysql`
启动 MySQL 服务：`sudo systemctl start mysql`
重启 MySQL 服务：`sudo systemctl restart mysql`
查看 MySQL 的开机自启状态：`systemctl is-enabled mysql`  ，**enable：启用，disable：禁用**
启动 MySQL 开机自启动：`sudo systemctl enable mysql`
禁用 MySQL 开机自启动：`sudo systemctl disable mysql`

## 系列话题
### 在ubuntu下编译openjdk
编译方法来自于《深入理解JAVA虚拟机 第三版》，详细请参考该书中第一章，1.6小节的相关说明
#### 准备源码包
根据书中的说明，推荐直接通过网页下载，具体方法如下：
去链接**https://hg.openjdk.java.net/jdk/jdk12/** 访问，点击左侧的**Browse**后，在下方有个zip压缩包，直接点击下载即可。如下图所示：
![下载openjdk12源码压缩包](145818e1/2021-05-08_165324.png)
#### 系统需求

## 相关命令
## 参考