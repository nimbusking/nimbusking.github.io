---
title: Linux下Nexus私服搭建
abbrlink: f3b827b0
date: 2021-01-13 23:02:30
tags:
    - Nexus私服
    - CentOS 7
updated: 2025-04-12 12:30:00
categories: 操作系统
---

## 背景
本篇搭建设计到的相关资源如下：
- 操作系统：CentOS 7.9
- JDK: 1.8.0_211
- Nexus私服版本：nexus-3.29.2-02

### 更新日志
- 2025-04-12：补充升级迁移旧版本至最新版本：nexus-3.79.1-04步骤详情

### Nexus介绍
Nexus 是Maven仓库管理器，如果我们使用Maven，我们可以从Maven中央仓库下载所需要的构件（artifact），但这通常没有公司这么干，一般都是在本地架设一个Maven仓库服务器，在代理远程仓库的同时维护本地仓库，以节省带宽和时间，Nexus就可以满足这样的需要。此外，它还提供了强大的仓库管理功能，构件搜索功能，它基于REST，友好的UI是一个extjs的REST客户端，它占用较少的内存，基于简单文件系统而非数据库。这些优点使其日趋成为最流行的Maven仓库管理器。
Nexus不是Maven的核心概念，它仅仅是一种衍生出来的特殊的Maven仓库。对于Maven来说，仓库只有两种：本地仓库和远程仓库。

### Nexus私服特点
搭建私服后：本地仓库没有，再去私服下载，私服没有，再去中央仓库下载
- 减少网络带宽流量
- 加速Maven构建
- 部署第三方构件
- 提高稳定性、增强控制
- 降低中央仓库的负载

<!-- more -->

## 准备
## 搭建过程
### 部署
```shell
[root@microservice ~]# mkdir /usr/local/nexus
[root@microservice ~]# tar zxf nexus-3.29.2-02-unix.tar.gz -C /usr/local/nexus/
# 启动nexus必须使用nexus用户，不可以使用权限过高的用户，比如root，否则会启动失败
# 不要使用useradd，如果一定要使用，sudo useradd -m -d /home/username username 设置好你的家目录
[root@microservice ~]# adduser nexus
[root@microservice ~]# chown -R nexus:nexus /usr/local/nexus/
[root@microservice ~]# ls /usr/local/nexus/
nexus-3.29.2-02     # 这是应用目录
sonatype-work         # 这是工作目录，存放镜像仓库
# 运行内存和工作目录nexus-3.29.2-02/bin/nexus.vmoptions （修改对应字段即可）
# 运行监听地址和端口nexus-3.29.2-02/etc/nexus-default.properties
[root@microservice ~]# ln -s /usr/local/nexus/nexus-3.29.2-02/bin/nexus /usr/local/bin/
# 创建命令软连接
# 切换至nexus用户，并启动nexus服务，如果使用root用户，会因为权限过高而启动失败
[root@microservice ~]# su nexus
[nexus@microservice nexus]$ nexus start 
Starting nexus
[nexus@microservice ~]# netstat -anput | grep 8081
tcp        0      0 0.0.0.0:8081            0.0.0.0:*               LISTEN      4687/java 
```

启动nexus之后，在浏览器访问http://ip:8081端口，ip就是你服务器的ip，访问之后，单击页面右上角的Sign in
初次登录会提示一个获取admin用户的默认密码路径，如下图所示：
![admin初始密码](f3b827b0/03c5869d4ef5c4b3ee329ea106999903.png)

按指定的路径，cat即可，获取到初始化admin用户密码之后就可以登录了，登录后会提示重置admin用户密码。
在Configure Anonymous Access时，按需选择，如果允许匿名访问私服，则可以允许，反之选择disable。

### 创建角色
### 创建用户

### 设置自动启动
通过systemctl来进行管理，在/etc/systemd/system目录下见一个名为*nexus.service*的文件，文件内容如下
```shell
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking

# nexus运行的用户和组
User=nexus
Group=nexus

# 运行环境，nexus运行的依赖项，JAVA_HOME的运行环境
Environment=INSTALL4J_JAVA_HOME=/etc/java/jdk1.8.0_211
ExecStart=/usr/local/bin/nexus start
ExecStop=/usr/local/bin/nexus stop 
Restart=on-failure
# 通过systemd方式来启动服务的，需要加上文件句柄描述，不然会读取默认的systemd配置，导致全局的ulimit设置不生效
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

设置服务自启动
```shell
## 加载新的unit配置文件
sudo systemctl daemon-reload
## 设置允许服务自启动
sudo systemctl enable nexus.service
## 服务启动
sudo systemctl start nexus.service
## 服务停止
sudo systemctl stop nexus.service
## 服务启动状态
sudo systemctl status nexus.service
```

**如果在start过程中遇到问题，systemctl status nexus.service命令查看运行结果，如果还是有问题，通过journalctl -xe查看**

### 设置代理仓库
阿里云仓库的URL：https://maven.aliyun.com/nexus/content/groups/public/

## 注意事项
###  file descriptor警告
具体体现在/support/status状态页面的File Descriptors项目可能会显示 *Recommended file descriptor limit is 65536 but count is 4096.* 警告。
根据提示其实也能猜的出来，就是建议当前运行的私服建议文件句柄到65536。
这里在官网有具体的解释：https://help.sonatype.com/repomanager3/installation/system-requirements#SystemRequirements-AdequateFileHandleLimits
解决办法就是为当前nexus私服运行的用户，如：nexus，指定文件句柄：
```shell
# 使用root权限在/etc/security/limits.conf文件末尾添加：
nexus - nofile 65536
```

修改之后，重新进入运行用户，如：nexus，并重启nexus私服

### 项目中配置好私服访问地址后报未授权错误（Not authorized）
具体报错如下：
Could not transfer artifact org.apache.curator:curator-recipes:pom:2.11.0 from/to nexus
(http://192.168.198.128:8081/repository/maven-public/): Not authorized

这个原因是因为，在你本地maven配置文件（setting.xml）中mirrors标签只配置了私服的镜像，而在servers标签中配置对应的用户名和密码，要么找不到，要么没有配置
检查如下：
```xml
<mirror>
    <id>nexus</id>
    <name>Nexus private</name>
    <mirrorOf>*</mirrorOf>
    <url>http://192.168.198.128:8081/repository/maven-public/</url>
</mirror>
```

```xml
<server>
  <id>nexus</id>
  <username>admin</privateKey>
  <password>admin123</passphrase>
</server>
```

**解决方案：检查你的配置，是否有确实两个配置中的一个或者是否两个配置中的id是否一致。如果没有配置则配置一下，如果不匹配，则将俩id改成一致**
本质就是，通过mirror中的id去寻找server中相同id的用户名和密码，完了去访问私服。
如果配错或者漏配了，自然当前配置的mirror找不到用户名和密码，自然不能访问。（前提默认你私服的配置不允许匿名访问仓库）


## 升级迁移过程
开始之前有一些说明：
- 在开始之前，请仔细看一下下面的升级流程后再进行升级操作。
- 官网关于升级的注意介绍：https://help.sonatype.com/en/migrating-to-a-new-database.html
- 中间版本3.70，我是在win上完成的，方便操作而已，你也可以选择linux版本，本质一样。
    - 还有个原因，数据库迁移工具，官方建议是要有16G的RAM，我懒得整我exsi服务器了，我win电脑64G，方便整。
- 官网建议是使用外部数据库存储，使用的是PostgreSQL。这个看你实际情况来，我是自己使用，自然H2足够；
    - 新版本使用H2数据库存储，总共支持10W个组件，每天20W笔请求；感觉一般团队其实也够用了
- 旧版本，记得全量备份，全量备份，全量备份！


### 升级流程
{% mermaid graph TD %}
    A[开始] --> B[先升级nexus 3.70版本]
    B --> C[登录3.70控制台备份数据]
    C --> D[使用nexus-db-migrator-3.70.3-01.jar迁移数据到h2数据库]
    D --> E[升级nexus版本到nexus-3.79.1-04]
    E --> F[配置文件配置2个配置；复制h2数据库文件]
    F --> G[启动新服务]
    G --> H[结束]
{% endmermaid %}

### 1. 升级3.7.0版本
[官方下载3.7.0的win版本](https://sonatype-download.global.ssl.fastly.net/repository/downloads-prod-group/3/nexus-3.70.3-01-java11-win64.zip)
解压到你本地目录，如下图所示：
![win本地目录](f3b827b0/MyCapture_2025-04-12_11-47-24.jpg)

- 此时，将之前老版本的nexus的【sonatype-work】目录，全部下载并覆盖3.70版本的同目录。
- 打开命令行，CD到nexus-3.70.3-01\bin目录下，直接运行：``` nexus.exe /run ```，第一次运行稍等，差不多看到下图且控制台没有明显报错，即代表启动成功
    ![启动成功](f3b827b0/MyCapture_2025-04-12_11-51-34.jpg)

### 2. 登录3.70版本控制台
- 浏览器打开：http://127.0.0.1:8081/
- 输入之前老版本的nexus的admin用户和密码（因为是整个目录覆盖的，这些账户密码不变）
- 打开路径：设置（上方齿轮按钮） -> System -> Tasks -> Create task
- 任务选择：**Admin-Export databases for backup**，其余的直接参考下图设置即可
    ![配置导出任务](f3b827b0/MyCapture_2025-04-12_11-55-05.jpg)
- 创建后，手工点击：Run按钮，等待Status Running变为Status Waiting及Last result Ok的出现。
- 命令行停止nexus程序：ctrl + C中断即可。

### 3. 迁移h2数据库
下载迁移工具：[官方下载jar工具](https://sonatype-download.global.ssl.fastly.net/repository/downloads-prod-group/nxrm3-migrator/nexus-db-migrator-3.70.3-01.jar)
**注意：**这里必须使用3.70版本的工具，不要使用高版本的，因为高版本的没有提供从老数据库（OrientDB）迁移到H2的选项了，只有从postgres向h2互相迁移的指令，即：h2_to_postgres和postgres_to_h2
具体的可以看官网描述：https://help.sonatype.com/en/legacy-database-migration.html

- 经过上一步操作，在目录：```sonatype-work\nexus3\exportFile ``` 下，你会看到一些bak的备份文件
- 此时把上面下载的：nexus-db-migrator-3.70.3-01.jar，放到这个目录下
- 命令行CD到这个目录，运行下面命令：
    ```java
    java -Xmx16G -Xms16G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M -jar nexus-db-migrator-3.70.3-01.jar --migration_type=h2
    ```
- 提示 Do you want to continue [y/n]?是，输入y，然后等到数据库迁移完成。
- 迁移成功如下图所示，在这个目录下会生成一个```nexus.mv.db```文件，这个就是迁移后的H2数据库文件，后面要用到：
    ![迁移成功](f3b827b0/MyCapture_2025-04-12_12-10-04.jpg)

#### 运行BC加密错误
如果你运行上面工具报错了，提示下面内容 ```JCE cannot authenticate the provider BC```：
```log
Caused by: java.lang.SecurityException: JCE cannot authenticate the provider BC
        at javax.crypto.JceSecurity.getInstance(JceSecurity.java:118)
        at javax.crypto.SecretKeyFactory.getInstance(SecretKeyFactory.java:244)
        at org.sonatype.nexus.crypto.internal.CryptoHelperImpl.createSecretKeyFactory(CryptoHelperImpl.java:254)
        at org.sonatype.nexus.crypto.internal.PbeCipherFactoryImpl$PbeCipherImpl.<init>(PbeCipherFactoryImpl.java:80)
        at org.sonatype.nexus.crypto.internal.PbeCipherFactoryImpl.create(PbeCipherFactoryImpl.java:60)
        at com.sonatype.nexus.db.migrator.config.CipherConfig.pbeCipher(CipherConfig.java:47)
        at com.sonatype.nexus.db.migrator.config.CipherConfig$$EnhancerBySpringCGLIB$$87ca059f.CGLIB$pbeCipher$0(<generated>)
        at com.sonatype.nexus.db.migrator.config.CipherConfig$$EnhancerBySpringCGLIB$$87ca059f$$FastClassBySpringCGLIB$$9fd9e8f7.invoke(<generated>)
        at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:244)
        at org.springframework.context.annotation.ConfigurationClassEnhancer$BeanMethodInterceptor.intercept(ConfigurationClassEnhancer.java:331)
        at com.sonatype.nexus.db.migrator.config.CipherConfig$$EnhancerBySpringCGLIB$$87ca059f.pbeCipher(<generated>)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154)
        ... 71 common frames omitted
```
有过BC加解密的开发经验的，你可能马上就觉察到了，BC加密包问题。
官方建议的是，直接用OpenJDK(整合了BC包)，我win的jdk是oracle jdk 1.8，默认自然没有BC包的。

简单的方案是：直接下载BC包，修改java.security，具体做法如下：
- 下载BC包：https://repo1.maven.org/maven2/org/bouncycastle/bcprov-jdk18on/1.80/bcprov-jdk18on-1.80.jar
- 放到：```jdk-1.8\jre\lib\ext```路径下面
- 修改：java.security，在指定位置增加一行如下，11序号类推（如果你以前配过其它的，编号+1即可）：
    ```properties
    security.provider.11=org.bouncycastle.jce.provider.BouncyCastleProvider
    ```
- 重新运行上面jar命令即可

### 4. 升级线上nexus服务
- 去官网下载linux版本：[nexus-3.79.1-04-linux-x86_64.tar.gz](https://sonatype-download.global.ssl.fastly.net/repository/downloads-prod-group/3/nexus-3.79.1-04-linux-x86_64.tar.gz)
- 解压到你服务的某个目录下
- 将第3步，你本地3.70版本的【sonatype-work】，全部覆盖3.79版本的目录
- 先删除```sonatype-work\nexus3\db```目录下所有内容
- 再将 ```sonatype-work\nexus3\exportFile\nexus.mv.db```复制到：```sonatype-work\nexus3\db```目录下，即这个目录下只有这个db文件即可
- 配置```sonatype-work\nexus3\etc\nexus.properties```文件，增加俩东西
    - 结尾增加2行
        ```properties
        nexus.datastore.enabled=true
        nexus.secrets.file=/home/nexus/sonatype-work/secrets/file.json
        ```
    - 关于第二行说明：因为高版本的nexus是需要我们配置自定义外置的密钥的，属于一个增强安全的配置。如果不配置就会看到下面一个告警，配置细节看下一小结详细介绍：
        ![Default Secret告警](f3b827b0/MyCapture_2025-04-12_11-05-56.jpg)
- 配置好之后，就可以直接nexus start启动了
    - 前文中涉及到的自启动服务，已经相关软链，对应修改即可。

#### 自定义加密配置文件
运行如下命令，路径与上面配置的保持一致即可，当然你也可以选择其它路径：
```shell
mkdir -p /home/nexus/sonatype-work/secrets && vim /home/nexus/sonatype-work/secrets/file.json
```

键入下面内容：
active与id的值保持一致
```json
{
  "active": "nexus-private-key",
  "keys": [
    {
      "id": "nexus-private-key",
      "key": "你自己随机生成的密码"
    }
  ]
}

```

- 强制请求刷新，让配置生效。这部需要做，不然你配的这个不会生效的。
    启动后，登录后台，F12随意找一笔请求，如下图所示：
    ![随意一笔请求](f3b827b0/MyCapture_2025-04-12_11-18-29.jpg)
    拿到：NX-ANTI-CSRF-TOKEN
- 再你的服务后台curl一笔请求, NX-ANTI-CSRF-TOKEN换成你拿到的，secretKeyId就是上面配置的file.json文件中配置的：
    ```shell
    curl -X 'PUT' \
      'http://192.168.51.221:8081/service/rest/v1/secrets/encryption/re-encrypt' \
      -H 'accept: application/json' \
      -H 'Content-Type: application/json' \
      -H 'NX-ANTI-CSRF-TOKEN: 0.4569008429' \
      -H 'X-Nexus-UI: true' \
      -d '{
      "secretKeyId": "nexus-private-key"
    }'
    ```
- 重启服务
- 再次登录后台admin，此时应该就会弹框，如下图所示：
    ![输入密码授权](f3b827b0/MyCapture_2025-04-12_11-17-20.jpg)
- 输入密码授权之后，就可以了。此时后台密码就重新加密了


### 5. 启动新服务
启动好之后，登录后就如下：
![新版本nexus服务首页](f3b827b0/MyCapture_2025-04-12_11-02-31.jpg)

至此升级完成。

## 引用
- [部署maven及Nexus私服](https://blog.51cto.com/14227204/2492248)
- [Maven私服Nexus安装与使用](https://www.jianshu.com/p/88fbca59b963)


