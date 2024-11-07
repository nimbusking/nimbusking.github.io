---
title: CentOS7基础操作相关
abbrlink: ca10a347
date: 2019-11-25 01:28:43
tags:
    - CentOS7
    - Linux
updated: 2024-10-28 20:30:21
categories: 操作系统
---

### 基本命令
#### 系统更新
```shell
yum -y update
```

#### 安装网络工具
```shell
yum install net-tools
```
#### 找不到yum-config-manager
直接安装：yum-utils即可

### 环境相关
#### 配置Java8
1. 从Oracle官网下载相应版本的64位JDK压缩包，通常是**tar.gz**结尾
2. 通过SFTP工具上传至CentOS后台指定目录下，如：**\usr\java**
3. 解压目录

```shell
tar -zxvf jdk-8u211-linux-x64.tar.gz 
```

<!--more-->

4. 配置环境变量：
```shell
# 示例解压在/usr/java/jdk1.8.0_211目录下
vi /etc/profile
# 结尾添加
export JAVA_HOME=/usr/java/jdk1.8.0_211
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
# 保存退出后
source /etc/profile
```

#### 修改IP地址
直接修改**ifcfg-ens\*\***相关文件即可，表示通配，实际值需要在实际环境中观察。
```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens192
```

如同下图所示，将**BOOTPROTO**改为static(静态路由)，在结尾添加指定的静态IP、子网掩码、默认网关以及DNS信息。
![设置IP地址](ca10a347/config_ipv4.jpg)

#### 修改更新源
```shell
# 备份原有系统源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
# 拉取阿里云repo
## 如果发现wget找不到，那是没有安装wget，此时repo已经重名吧了，再次通过yum安装失败的，只需要反过来在将backup文件重命名为原来的repo即可
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 运行yum makecache生成缓存
yum makecache
# 更新
yum -y update
```


### 基础服务相关
#### MySQL安装
##### 添加MySQL YUM存储库
将MySQL Yum存储库添加到系统的存储库列表中。这是一次性操作，可以通过安装MySQL提供的RPM来执行。跟着下面步骤：
1. 到MySQL官网下载指定的MySQL Yum存储库，[MySQL Yum存储库下载地址](https://dev.mysql.com/downloads/repo/yum/)
下载你指定的需要的版本，Linux 8对应CentOS 8，Linux 7对应CentOS 7
**注：**官网下载需要登录一个已有的oracle官网帐号
![MySQL Yum存储库](ca10a347/download_mysql_yum_resouces__or_centos7.jpg)
2. 选择并下载适用于你所需要的平台发行包。将下载的rpm包上传到你的centos服务器中
3. cd到rpm包所在目录，使用以下命令来安装下载的发行包，替换*platform-and-version-specific-package-name*为下再的rpm包的名称：
```shell
sudo yum localinstall platform-and-version-specific-package-name.rpm
```

##### 选择需要安装的发布版本
如果你需要直接安装最新版本的MySQL，可以直接略过本步骤，直接通过yum一次性安装。相反，如果你要安装指定版本，则需要手工配置安装版本。
通过命令，来查看当前安装库中mysql的版本：
```shell
yum repolist all | grep mysql
```

可以看到如下图所示的内容：
![yum repolist all](ca10a347/after_config_mysql_yum_install.jpg)
这是修改后的效果，修改之前**mysql80-community/x86_64**这项状态是**enable**的。
通过如下命令，修改需要安装的版本信息：
注：如果提示yum-config-manager不存在，看上文小结中，直接安装**yum-utils**即可
```shell
# 关闭MySQL 8
sudo yum-config-manager --disable mysql80-community
# 关闭MySQL 5.7，如果之前看到的是已经关闭的状态，这步则不用关闭
sudo yum-config-manager --disable mysql57-community
# 启用MySQL 5.6
sudo yum-config-manager --enable mysql56-community
```

##### 安装MySQL
通过以下命令安装MySQL:
```shell
sudo yum install mysql-community-server
```

在这个过程中，这将安装MySQL server（mysql-community-server）的包以及运行服务器所需组件的包，包括client（mysql-community-client）的包，客户端和服务器的常见错误消息和字符集（mysql-community-common）以及共享客户端库（mysql-community-libs） 。

##### 启动MySQL服务器
通过以下命令启动MySQL服务器：
```shell
service mysqld start
```

与之相对应的，停止(stop)，重启(restart)，直接修改后面的参数即可
##### 配置MySQL相关参数
###### 本地连接MySQL数据库
通过以下命令连接MySQL
```shell
mysql -u root -p
```

出现**Enter password**，输入密码。由于刚安装，没有设置密码，直接回车 Enter 进入
![连接MySQL](ca10a347/ConnectToMySQLThroughCommand.jpg)
输入命令**show databases;**查看默认安装的数据库
![查看安装的数据库Schemal](ca10a347/ShowDatabases.jpg)
###### 设置root密码
依次通过以下命令修改root用户名密码：
```sql
mysql>use mysql;
mysql>update user set password=password('your password') where user='root';
mysql>flush privileges;
```

**your password**为你需要设置的密码
输入**quit**命令退出当前登录，用新的密码重新连接 mysql。
###### 设置远程登录
mysql默认只能本机登录，通过以下命令，设置允许远程登录：
```sql
mysql>GRANT ALL PRIVILEGES ON *.* TO 'your username'@'%' IDENTIFIED BY 'your password' WITH GRANT OPTION;
```

**your username**和**your password**改成 mysql 数据库的用户和密码

###### 高版本5.7+更改root密码
在开启mysql服务之后，简单密码切记仅供本地自我学习使用
```shell
//获取临时密码
grep "password" /var/log/mysqld.log

//进入数据库
mysql -u root -p

//修改密码验证规则
mysql> set global validate_password_policy=0;
mysql> set global validate_password_length=1;

//设置简单密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```

到此，在 CentOS 7上安装 MySQL 5.6 完成，CentOS 6 也是类似操作。
用一个客户端工具连接一下试试，

猜测一下，可能你会发现通过工具远程并无法连接 :)
原因是：CentOS中默认防火墙策略是禁止所有远程访问的，可以通过如下命令来开放端口：
```shell
# 查看当前防火墙策略
firewall-cmd --list-all
# 开放3306端口
firewall-cmd --permanent --add-port=3306/tcp
# 重启防火墙
service firewalld restart
## 或者
firewall-cmd --reload
# 查看防火墙是否开放3306端口，返回yes就OK了，当然也可以再次通过list-all来查看
firewall-cmd --query-port=3306/tcp
```

再次回到工具中，测试连接，连接成功 :)
![工具远程测试连接](ca10a347/ConnetMySQLSuccesfully.jpg)
至此就真的结束啦....

#### Redis单机安装
安装版本是redis 5.0.7稳定版本
##### 准备
###### 下载redis包
直接运行wget http://download.redis.io/releases/redis-5.0.7.tar.gz即可
```shell
[root@localhost ~]# wget http://download.redis.io/releases/redis-5.0.7.tar.gz
--2020-01-08 20:46:22--  http://download.redis.io/releases/redis-5.0.7.tar.gz
Resolving download.redis.io (download.redis.io)... 109.74.203.151
Connecting to download.redis.io (download.redis.io)|109.74.203.151|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1984203 (1.9M) [application/x-gzip]
Saving to: ‘redis-5.0.7.tar.gz’

100%[========================================================================================================================================>] 1,984,203    355KB/s   in 5.5s   

2020-01-08 20:46:29 (355 KB/s) - ‘redis-5.0.7.tar.gz’ saved [1984203/1984203]
```

###### 解压
```shell
tar -zxvf redis-5.0.7.tar.gz
```

##### 编译
###### 安装GCC
```shell
# 没啥遇到确认的直接y
yum install gcc
```

###### 编译安装
```shell
[root@localhost redis-5.0.7]# make MALLOC=libc
[root@localhost redis-5.0.7]# cd src && make install
```

###### 设置环境变量等
任意通过redis-service直接启动
```shell
mkdir -p /usr/local/redis/bin
cp src/redis-server /usr/local/redis/bin
cp src/redis-cli /usr/local/redis/bin
cp src/redis-sentinel /usr/local/redis/bin
cp src/redis-trib.rb /usr/local/redis/bin
cp src/redis-check-aof /usr/local/redis/bin
cp src/redis-check-rdb /usr/local/redis/bin
cp src/redis-benchmark /usr/local/redis/bin
echo 'export PATH=$PATH:/usr/local/redis/bin' >> .bash_profile
. .bash_profile
```

##### 启动
###### 直接启动
cd到src目录后，直接./redis-server
```shell
[root@localhost src]# ./redis-server 
12725:C 08 Jan 2020 20:53:33.796 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
12725:C 08 Jan 2020 20:53:33.796 # Redis version=5.0.7, bits=64, commit=00000000, modified=0, pid=12725, just started
12725:C 08 Jan 2020 20:53:33.796 # Warning: no config file specified, using the default config. In order to specify a config file use ./redis-server /path/to/redis.conf
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.7 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 12725
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

12725:M 08 Jan 2020 20:53:33.798 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
12725:M 08 Jan 2020 20:53:33.798 # Server initialized
12725:M 08 Jan 2020 20:53:33.798 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
12725:M 08 Jan 2020 20:53:33.798 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
12725:M 08 Jan 2020 20:53:33.798 * Ready to accept connections
```

不方便，原因是这个窗口需要一直运行，ctrl + c直接退出
###### 后台进程方式运行
如果你需要外部客户端直接访问的话，修改下面三行配置：
/# bind 127.0.0.1
protected-mode no
/# 后台运行
daemonize yes
直接运行redis-server
```shell
redis-server redis.conf
```

#### redis伪集群安装
顾名思义，伪集群安装，实际集群应该分布在不同的redis服务器上的，这里个人使用，就在单服务器上实现了
##### 准备配置文件
配置文件目录随意，这里我就在redis编译好的路径下配置了
单个配置修改如下：
```properties
# 监听端口
port 7001
# 监听 ip
bind 0.0.0.0
# 指定文件存放路径 （ .rdb .aof nodes-xxxx.conf 这样的文件都会在此路径下）
dir /opt/redis-7001
# 启动集群模式 
cluster-enabled yes
# 集群节点配置文件
cluster-config-file nodes-7001.conf
# 后台启动
daemonize yes
# 集群节点超时时间​
cluster-node-timeout 5000
# 指定持久化方式，开启 AOF 模式
appendonly yes
# 非保护模式
protected-mode no
```

其余的文件直接copy上述配置好的7001配置文件就好了。创建6个配置文件（redis集群最低6个节点）
```shell
mkdir /etc/redis-cluster
cp redis.conf /etc/redis-cluster/redis-7001.conf
cp redis.conf /etc/redis-cluster/redis-7002.conf
cp redis.conf /etc/redis-cluster/redis-7003.conf
cp redis.conf /etc/redis-cluster/redis-7004.conf
cp redis.conf /etc/redis-cluster/redis-7005.conf 
cp redis.conf /etc/redis-cluster/redis-7006.conf
mkdir /opt/redis-7001
mkdir /opt/redis-7002
mkdir /opt/redis-7003
mkdir /opt/redis-7004
mkdir /opt/redis-7005
mkdir /opt/redis-7006
```

copy之后，可以直接通过sed命令替换配置文件中的相关端口信息，如下：
```shell
# 以此类推，分别修改，当然你也可以写到一个shell脚本里，一次性全部执行。
# 确认修改是否完成，grep '7001' nodes-7001.conf，你应该看到有3行
sed -i 's/7001/7002/' nodes-7002.conf
```

##### 启动集群
按单机启动方式启动所有节点，注意配置文件路径就是。
启动之后，ps查看redis信息，可以看到如下信息：
```shell
# 启动
redis-server /etc/redis-cluster/redis-7001.conf
redis-server /etc/redis-cluster/redis-7002.conf
redis-server /etc/redis-cluster/redis-7003.conf
redis-server /etc/redis-cluster/redis-7004.conf
redis-server /etc/redis-cluster/redis-7005.conf
redis-server /etc/redis-cluster/redis-7006.conf
# 启动之后
[root@localhost redis-5.0.7]# ps -ef | grep redis
root      7635     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7001 [cluster]
root      7642     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7002 [cluster]
root      7647     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7003 [cluster]
root      7652     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7004 [cluster]
root      7657     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7005 [cluster]
root      7662     1  0 10:34 ?        00:00:00 redis-server 192.168.31.49:7006 [cluster]
```

说明启动成功了
接下来直接通过redis-cli启动集群即可
```shell
redis-cli --cluster create 192.168.31.49:7001 192.168.31.49:7002 192.168.31.49:7003 192.168.31.49:7004 192.168.31.49:7005 192.168.31.49:7006 --cluster-replicas 1
```

启动完成之后，会看到如下信息：
```shell
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.31.49:7005 to 192.168.31.49:7001
Adding replica 192.168.31.49:7006 to 192.168.31.49:7002
Adding replica 192.168.31.49:7004 to 192.168.31.49:7003
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 4a14d7fad92c4bd5cca5a5a5b620fe63979f6345 192.168.31.49:7001
   slots:[0-5460] (5461 slots) master
M: f31ae790a628511807e7a00182844d67da8a68aa 192.168.31.49:7002
   slots:[5461-10922] (5462 slots) master
M: fbb2295a6c7c27cf031a392f7b4d80ea2a58fc86 192.168.31.49:7003
   slots:[10923-16383] (5461 slots) master
S: 469631e70b417d6f2f32d400cb78a2d948adc22f 192.168.31.49:7004
   replicates f31ae790a628511807e7a00182844d67da8a68aa
S: f6d19acd0462b08f5f3b044d114fff7a215f4a2f 192.168.31.49:7005
   replicates fbb2295a6c7c27cf031a392f7b4d80ea2a58fc86
S: efcf9038cf9251ab8b8e53fca31b604f2bb399e1 192.168.31.49:7006
   replicates 4a14d7fad92c4bd5cca5a5a5b620fe63979f6345
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 192.168.31.49:7001)
M: 4a14d7fad92c4bd5cca5a5a5b620fe63979f6345 192.168.31.49:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 469631e70b417d6f2f32d400cb78a2d948adc22f 192.168.31.49:7004
   slots: (0 slots) slave
   replicates f31ae790a628511807e7a00182844d67da8a68aa
S: f6d19acd0462b08f5f3b044d114fff7a215f4a2f 192.168.31.49:7005
   slots: (0 slots) slave
   replicates fbb2295a6c7c27cf031a392f7b4d80ea2a58fc86
M: f31ae790a628511807e7a00182844d67da8a68aa 192.168.31.49:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: efcf9038cf9251ab8b8e53fca31b604f2bb399e1 192.168.31.49:7006
   slots: (0 slots) slave
   replicates 4a14d7fad92c4bd5cca5a5a5b620fe63979f6345
M: fbb2295a6c7c27cf031a392f7b4d80ea2a58fc86 192.168.31.49:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

说明启动完成了，所有主从节点信息也显示出来了。

##### 查看集群信息
```shell
[root@microservice redis-cluster]# redis-cli -c -h 192.168.31.49 -p 7001
192.168.31.49:7001> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:457
cluster_stats_messages_pong_sent:458
cluster_stats_messages_sent:915
cluster_stats_messages_ping_received:453
cluster_stats_messages_pong_received:457
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:915
```

##### 查看节点信息


##### 遇到redis-cli集群加载失败
这种情况通常遇到上一次redis未正常退出导致的。
通过redis-cli --cluster create时，提示下述错误：
```shell
Node 192.168.31.49:7001 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0
```

此时，使用下述命令，直接清空异常redis
```shell
./redis-cli -h ip地址 -p 端口
>> flushdb
```
如果在清空之后仍然无效，尝试将当前启动的所有节点重新kill调，重新运行后再执行创建集群命令看看

#### zookeeper安装
##### 下载zk
```shell
wget https://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz
```

##### 配置文件
解压后，cd到conf目录下，copy sample配置文件为zoo.cfg

```shell
[root@localhost conf]# vim zoo.cfg 
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/etc/apache-zookeeper-3.6.2/data
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

## Metrics Providers
#
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true
```

##### 配置service开机启动
cd 到 /etc/init.d/目录下
新建名为zookeeper文件，文件内容如下，注意ZK_PATH和JAVA_HOME填写实际路径
```shell
[root@localhost init.d]# vim zookeeper
#!/bin/bash
#chkconfig:2345 20 90
#description:zookeeper
#processname:zookeeper
ZK_PATH=/etc/apache-zookeeper-3.6.2
export JAVA_HOME=/etc/java/jdk1.8.0_211
case $1 in
         start) sh  $ZK_PATH/bin/zkServer.sh start;;
         stop)  sh  $ZK_PATH/bin/zkServer.sh stop;;
         status) sh  $ZK_PATH/bin/zkServer.sh status;;
         restart) sh $ZK_PATH/bin/zkServer.sh restart;;
         *)  echo "require start|stop|status|restart"  ;;
esac
```

检查服务：
```shell
# 添加服务
chkconfig --add zookeeper
# 修改执行权限
chmod 755 zookeeper
# 先停止服务
service zookeeper stop
# 在启动服务
service zookeeper start
# 查询服务状态
service zookeeper status
```

#### kafka伪集群部署
##### 准备
通过官网下载kafka二进制客户端
```shell
wget https://downloads.apache.org/kafka/2.4.1/kafka_2.11-2.4.1.tgz
```
##### 配置集群
###### 配置KAFKA_HOME
```shell
vim /etc/profile
# 末尾添加，export PATH如果有JAVA_HOME，类似在后面添加冒号
export KAFKA_HOME=/usr/kafka/kafka_2.11-2.4.1
export PATH=$PATH:$KAFKA_HOME/bin
```

###### 配置KAFKA集群文件
直接cd到config目录下，复制三个**server.properties**文件
每个文件中的配置修改如下几项：
```shell
broker.id=1
## 填你实际机器IP
listeners=PLAINTEXT://192.168.**.***:9092
log.dirs=/usr/local/src/kafka_2.11-2.1.0/tmp/kafka-logs1
## 填你zk服务的地址
zookeeper.connect=192.168.**.***:2182
```

##### 后台启动
```shell
./bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server1.properties &
./bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server2.properties &
./bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server3.properties &
```

##### 测试
###### 创建一个topic
```shell
# 创建topic，包含一个分区，3个副本
./bin/kafka-topics.sh --create --zookeeper 192.168.31.49:2181 --replication-factor 3 --partitions 1 --topic test_topic
> WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
> Created topic test_topic.
# 查看topic
./bin/kafka-topics.sh --list --zookeeper 192.168.31.49:2181
> test_topic
```

###### 创建生产者&消费者
```shell
# 创建生产者
./bin/kafka-console-producer.sh --broker-list master:9092,master:9093,master:9094 --topic test_topic
# 创建消费者
./bin/kafka-console-consumer.sh --bootstrap-server master:9092,master:9093,master:9094 --from-beginning --topic test_topic
```

###### 通过指定consumer.properties文件创建topic
先在config文件夹下copy consumer.properties，假定命名为：cc.nimbusk.consumer.properties
```shell
# 通过kafka-console-consumer.sh指定分组下的topic
# bin/kafka-console-consumer.sh --new-consumer --bootstrap-server localhost:9092 --topic test_topic --from-beginning --consumer.config config/cc.nimbusk.consumer.properties --delete-consumer-offsets
```




### 引用参考
- [Redis 5.0.7 讲解，单机、集群模式搭建](https://www.cnblogs.com/leffss/p/11996580.html)