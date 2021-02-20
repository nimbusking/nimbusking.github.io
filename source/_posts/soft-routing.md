---
title: 新手从零软路由系列
abbrlink: c07c0078
date: 2019-11-23 19:56:13
tags:
    - 软路由
    - SoftRouting
    - RouterOS
    - OpenWRT
    - LEDE
    - ESXi 6.7
categories: 软路由
---

### 需求背景
9012年了，对于像我这种，平时没事会去折腾一些硬件。比如一些装机、数码之类的。关注了很多新鲜的东西，当然其中一个就是软路由。
在了解软路由之前，可以去了解一下软路由具体是干嘛的？完了再了解了解下文中具体配置相关的东西。
### 需要准备
- 一台软路由设备
    + 底层安装好ESXi或者LEDE
    + 至少拥有两个物理网口
    + 内存按个人实际分配使用，如果纯软路由场景2~4G足以；如果辅助需要安装其它操作系统，如Linux或者Windows乃至黑苹果等，等同划分需要扩展相应的内存
- Mikrotkit OS镜像
- OpenWRT镜像

<!-- more -->

### 背景知识
#### Mikrotkit OS
#### OpenWRT
##### 自编译OpenWRT
初始编辑可参考如下博客文中所述：[编译Lean的Openwrt固件全攻略](https://imgki.com/archives/openwrt-lean.html)
基本上没有什么问题，可以通过公有云编译，也可以通过虚拟机本地编译都可以。
自编译要注意下面几个问题：
1. 极端情况，编译失败可能是因为网络问题
2. 自编译，磁盘空间需要保留足够，目前初略看，至少得50G。曾经本地编译过一次，居然最后是因为磁盘空间不足导致最终编译失败了。
3. 如果本地编译出现失败，又无法确定是否是网络原因。采取反证法，到公有云上再编译，如果还是出现同样的失败错误。**注意了** 很有可能是你选择的**LUCI插件冲突**了，少选几个再重新试一下看看。

###### 怎么在ESXi中用
编译好的文件，默认配置会输出两个文件，如下所示：
- openwrt-x86-64-combined-squashfs.img
- openwrt-x86-64-combined-squashfs.vmdk

但是这俩文件，是无法直接在ESXi后台控制页面，通过添加虚拟机的方式使用的。这里以vmdk文件为例，通过ESXi后台自带的**vmkfstools**转换一下之后，就可以了。
1. 先将上述编译好的vmdk文件，通过“数据库存储浏览器”上传到指定目录下，如下图所示：
![upload_vmdk](c07c0078/upload_vmdk.jpg)

2. 打开ESXi的SSH：主机->操作->服务->启用安全的Shell
3. 通过SSH客户端工具，远程连接到ESXi后台
4. cd到第一步上传的目录
    - 这里注意一下：我的vmdk存放的数据仓库的名称为“datastore1”，则实际路径为：/vmfs/volumes/datastore1，以此类推
5. 键入如下命令开始转换

```shell
# 语法为：vmkfstools -i [原始文件] [目的文件] -d thin
vmkfstools -i openwrt-x86-64-combined-squashfs.vmdk openwrt_new-x86-64-combined-squashfs.vmdk -d thin
```

如下图所示，转换完成：
![转换完成](c07c0078/convert_done.jpg)

转换完成后的vmdk文件就可以直接正常通过ESXi创建虚拟机了。

### 安装过程
与正常安装Windows过程大体相同，将ESXi对应的ISO文件（这里我安装的是6.7U3b版本），用第三方工具写入U盘即可
这里推荐rufus，rufus官网下载最新的版本(http://rufus.ie/)，如下图所示：
![rufus](c07c0078/AutoCapture_2021-01-03_235940.jpg)
- 设备选择你的U盘，不要选错了。
- 点击选择，选择指定路径下的ESXi的ISO镜像文件
- 分区类型和目标系统类型保持默认即可，如图分别为MBR和BIOS或UEFI
- 剩下的直接点击开始即可

后续插入到目标物理机上，通过U盘启动即可。如果是裸机的话，直接插上去，开机就行了。
具体安装过程很简单，输入root密码的时候，正常输入即可。
安装完毕之后，会提示机器重启。

#### 安装异常问题
##### 提示Using 'simple offset' UEFI RTS napping policy
这种一般在较为新的主板上会遇到，尤其是强制使用安全认证模式的UEFI模式启动的模式。
我在我的Intel的**NUC10i7FNH**主机上遇到了这个情况，涉及安全启动运行的给关了。
一方面，我升级了这个主板的BIOS版本，具体到指定主板的厂商的官网支持都会有最新的BIOS固件，按照官网提示的步骤升级即可。
###### 插曲：NUC10i7FNH对应的最新的固件更新说明
**注意：**如果你是用F7 & U盘方式更新BIOS的，**不要格式化U盘为extFAT**格式，BIOS会找不到的，格式化成NTFS或者FAT32都行。
官网说明：https://www.intel.cn/content/www/cn/zh/support/articles/000033291/intel-nuc.html
如下：
- (Intel PTT)Security > Security Features: Intel Platform Trust Technology: Disabled
- Boot > Secure Boot > Secure Boot: Disabled

更多关于在NUC10i7FNH下建议的BIOS设置，可以参考如下相关链接：
[intel-nuc-recommended-bios-settings-for-vmware-esxi](https://www.virten.net/2020/03/intel-nuc-recommended-bios-settings-for-vmware-esxi/)

##### 安装时自检提示找不到网卡驱动
安装的时候提示“No Network Adapters”，正如这个提示所示，找不到网络驱动器，导致这个原因一般就是当前安装版本的ESXi要么你网卡较新，要么比较小众，官网没有相应的默认维护进ESXi驱动，此时提供两种解决方案：
- 在网上找别人封装好的驱动的ESXi的ISO镜像
- 自己通过VMWare的离线bundle包打驱动

在virten上的一篇文章介绍了在线模式的打包整合驱动的方法[esxi-on-10th-gen-intel-nuc-comet-lake-frost-canyon](https://www.virten.net/2020/03/esxi-on-10th-gen-intel-nuc-comet-lake-frost-canyon/)，不推荐，因为依赖网络环境（你懂的）中间失败了，你可能又会遇到新的问题，完了来回折腾，搞了一大圈最后发现是网络问题，费时费力。但是这里面介绍的安装步骤，还是值得参考的

这里针对离线方式说明一下，用的封装命令，在下面这个网站里面有详细说明：
https://www.v-front.de/p/esxi-customizer-ps.html#download
这里安装物理机硬件如下：
- Intel NUC10I7FNH
- 网卡：I219-V
- ESXi 6.7U3b
- 内存：威刚万紫千红 32GB * 2
- SSD：恺侠 RD10 500GB

###### 下载VMware-PowerCLI-6.5.0和ESXi-Customizer-PS
在这里贴一下两个文件的下载地址，其中PowerCLI我在百度云分流了：
*VMware-PowerCLI-6.5.0：*
MD5值：1B7A5378835C6158CFB63C3D10BB9E18
http://down.whsir.com/downloads/VMware-PowerCLI-6.5.0-4624819.exe
备份下载：链接: https://pan.baidu.com/s/1L-Yeq0-ohyFwQB9lFRwiGQ 提取码: tf38
*ESXi-Customizer-PS：*
http://vibsdepot.v-front.de/tools/ESXi-Customizer-PS-v2.6.0.ps1

###### 下载离线6.7的bundle包
[官网VMware vSphere Hypervisor 6.7](https://my.vmware.com/zh/web/vmware/evalcenter?p=free-esxi6)
需要登录，建议提前注册一个vmware账号，注册流程略。
这里下载最新的6.7U3b版本即可，如下图所示：
![ESXi670-201912001](c07c0078/AutoCapture_2021-01-04_000640.jpg)
MD5值：153EA9DE288D1CC2518E747F3806F929
备用下载：
链接: https://pan.baidu.com/s/1ScNhiCr0BqKTIhJJsbgHwQ 提取码: 7j9r

###### 下载好指定驱动
支持列表[vibsdepot.v-front](https://vibsdepot.v-front.de/wiki/index.php/List_of_currently_available_ESXi_packages)
比如我的NUC10i7FNH这个设备上网卡是I219-V，上面网站中没有列出，在官网有提供：
[ESXi670-NE1000](https://download3.vmware.com/software/vmw-tools/ESXi670-NE1000-32543355-offline_bundle-15486963.zip)

###### 开始bundle
- 在本地任一磁盘根目录新建文件夹，例如我的是在G盘，建了一个名为newesxi的文件夹，文件夹内容目录如下，把上述下载的东西放进去。
![目录结构](c07c0078/AutoCapture_2021-01-04_005334.jpg)
- 其中offline文件夹内，提取上述ESXi670-NE1000压缩包中的VIB文件
- 运行PowerCLI
CD到上述newesxi目录下
键入如下命令：
```shell
.\ESXi-Customizer-PS-v2.6.0.ps1 -izip .\ESXi670-201912001.zip -load net-e1000e -pkgDir .\offline -outDir .\out -nsc
```

**注意：**上述命令中的*net-e1000e*就是指定网卡型号，这个型号就在上述vibsdepot.v-front网站中，有列出。
同时，如果你要 **同时**制作其它网卡型号，在后面添加英文逗号","即可。
例如：
```shell
.\ESXi-Customizer-PS-v2.6.0.ps1 -izip .\ESXi670-201912001.zip -load net-e1000e,net51-r8169,net55-r8168,net-atl1e,net-r8101 -pkgDir .\offline -outDir .\out -nsc
```

稍等片刻之后，看到如下提示则表示bundle完成。
![打包过程](c07c0078/AutoCapture_2021-01-04_011047.jpg)

至此离线驱动镜像包就制作好了，接下来就可以正常写入U盘安装了。
这里分享自己打包好的多驱动集成ISO镜像包。
MD5值：2EC7F4417E8C233F4F36B52DF6D564CD
链接: https://pan.baidu.com/s/19aJ_FZqQLmsk5jqaNQ8EBA 提取码: yyj7
已集成驱动列表：
- net-e1000e(包含I219-V网卡)
- net51-r8169
- net55-r8168
- net-atl1e
- net-r8101

### 系统设置
#### ESXI后台设置
##### 设置网络适配器（Network Adapters）
##### IPv4设置
##### 替换ESXI登录HTTPS证书
通常在刚安装好esxi之后，默认后台登录控制台，会提示你当前访问的链接不安全的提示，就像下面那样：
![不安全的链接](c07c0078/AutoCapture_2021-01-30_211415.jpg)
这种问题本质就是，你当前通过https访问的地址并未签发相应的SSL证书导致的。
因此，找到一个免费的SSL证书，并替换esxi后台的默认ssl证书即可。
由于我本站博客域名国内的DNS解析加速选择的是dnspod，所以这里就以腾讯DNSpod申请免费证书为例，其它国内诸如阿里云、七牛云等等，都有提供免费申请SSL证书的渠道，不过目前来看，似乎腾讯的dnspod审批的比较快。
dnspod免费的ssl证书提供商是亚信，自己本地玩无所谓的。
###### dnspod申请免费SSL证书
登录到DNSPOD控制台：https://console.dnspod.cn/
搜索SSL证书，或直接访问URL：https://console.cloud.tencent.com/ssl 如下图所示：
点击“申请免费证书”
![购买证书第一页](c07c0078/AutoCapture_2021-01-30_212517.jpg)
默认，点击确认即可，选择其它的ssl证书提供商，会收费。
![提交资料](c07c0078/AutoCapture_2021-01-30_212701.jpg)
在提交资料页面：
- 算法选择：默认即可
- 证书绑定域名：填写你要申请绑定证书的域名地址，例如我的就是esxi.nimbusk.cc
- 申请邮箱：填写自己的邮箱就好
- 证书名备注：备注这个证书是绑定什么域名干嘛用的就好
- 私钥密码：自己本地玩，不需要填写

填完之后，下一步，选择验证方式：
![选择验证方式](c07c0078/AutoCapture_2021-01-30_213052.jpg)
选择第一个，第一个会自动在你当前匹配的域名下增加一条TXT记录
第三步验证的时候，dnspod会验证你的解析域名下是否包含相应的记录，进而会影响你最终审核结果。

在下一步之后，**验证域名步骤中，可能会显示当前顶级域名验证失败**，等收到验证通过消息（邮件/短信）之后，再次点击查看域名验证状态就好。

签发成功之后，直接下载。会下载一个压缩包，等待后续使用。
###### 替换exsi后台默认证书
将上述步骤下载下来的zip压缩包中的 **nginx**文件夹内的两个文件，一个crt一个key文件，均重命名为rui。
通过linux文件传输工具，将下载重命名后的文件覆盖上传的如下路径下（当然覆盖之前你也可以备份原来的俩文件）
```shell
/etc/vmware/ssl/rui.crt
/etc/vmware/ssl/rui.key
```

覆盖替换完成之后，在SSH后台键入：services.sh restart
重启服务

之后，在浏览器，通过HTTPS域名的方式访问esxi后台即可。

当然，你在windows下浏览器下直接这么访问，可能会提示你根本无法访问，那是通过这种方式，你本地esxi后台ip是没有跟你的域名证书绑定在一起的。最简单的解决方案就是，在你windows的host文件加上一条域名解析记录即可。

完事之后，再通过浏览器https访问就好了，如下图所示：
![https访问esxi后台](c07c0078/AutoCapture_2021-01-30_215000.jpg)

###### Let's Encrypt自签
在网上找到一段脚本，先贴在这里，主要是用的linux的certbot来自签的，**目前我还没有自己实践过，暂时还不知脚本的准确性如何**
```shell
#!/bin/bash
#
## -----------------------------=[ WARNING ]=-------------------------------- ##
#
# This script is now woefully out of date due to which accounts ESXi allows to
# ssh into the box as well as sticky folders/file flags.
# I've since ported the whole thing to python with a lot of bells and whistles
# and if i get around to making it public, i'll put a link here.
#
## -------------------------------=[ Info ]=--------------------------------- ##
#
# Generate letsencrypt cert on local server and scp to esxi target.
# Designed and tested on Ubuntu 16.04LTS.
# Assumes you have upnp control over local network. Tested with Ubiquiti USG.
#
# Dependencies:
# miniupnpc (sudo apt install miniupnpc)
# certbot (sudo apt install certbot)
#
## -=[ Author ]=------------------------------------------------------------- ##
#
# shr00mie
# 9.21.2018
# v0.5
#
## -=[ Use Case ]=----------------------------------------------------------- ##
#
# Allows for the generation of certificates on a separate host which can then
# be securely copied to target esxi host.
#
## -=[ Breakdown ]=---------------------------------------------------------- ##
#
# 1. Prompt for esxi target FQDN, reminder email, and esxi admin username
# 2. Check if ssh keys exist for target.
#     - If keys exist, continue.
#     - If keys don't exist:
#       - Silently generate 4096 RSA key, no passphrase, user@target as comment.
#       - Add key to ssh-agent
#       - Create target folder/file structure for scp automation
#       - Restart SSH service on target.
# 3. Enable port forwarding.
# 4. Generate 4096 bit letsencrypt cert
# 5. Backup existing cert with datetime suffix
# 6. Copy cert to target
# 7. Restart target services
# 8. Remove port forwarding
#
## -=[ To-Do ]=-------------------------------------------------------------- ##
#
# change: PermitRootLogin yes -> PermitRootLogin no
# add: ChallengeResponseAuthentication no
# add: PasswordAuthentication no
#
## -=[ Functions ]=---------------------------------------------------------- ##

# Usage: status "Status Text"
function status() {
  GREEN='\033[00;32m'
  RESTORE='\033[0m'
  echo -e "\n...${GREEN}$1${RESTORE}...\n"
}

# Usage: input "Prompt Text" "Variable Name"
function input() {
  GREEN='\033[00;32m'
  RESTORE='\033[0m'
  echo -en "\n...${GREEN}$1${RESTORE}: "
  read $2
  echo -e ""
}

function pressanykey(){
  GREEN='\033[00;32m'
  RESTORE='\033[0m'
  echo -en "\n...${GREEN}$1. Press any key to continue.${RESTORE}..."
  read -r -p "" -n 1
}

## ---------------------------=[ Script Start ]=----------------------------- ##

# Importing Variables
status "Importing Variables"

# Read ESXiHost
input "Enter the FQDN for the certificate/host in host.domain.tld format" "ESXiHost"

# Read Email
input "Enter the email for confirmation & renewal notifications" "Email"

# Read ESXiUser
input "Enter ESXi target admin username" "ESXiUser"

# Prompt user to confirm/enable SSH on ESXi target
pressanykey "Confirm/Enable SSH access on $ESXiHost."

# Check for existing ssh keys for esxi host
status "Checking for existing ssh keys for $ESXiHost"

if [[ -e ~/.ssh/$ESXiHost'_rsa' ]]
    then
    status "Keys for $ESXiHost exist. Continuing"
else
  status "Keys for $ESXiHost not found. Generating 4096 bit keys"
  # Generate 4096 bit key for user@target
    ssh-keygen -b 4096 -t rsa -f ~/.ssh/$ESXiHost'_rsa' -q -N "" -C "$ESXiUser@$HOSTNAME LetsEncrypt"
  status "Adding new key to ssh-agent"
  # Add key to agent
    eval `ssh-agent` && ssh-add ~/.ssh/$ESXiHost'_rsa'
  status "Configuring $ESXiHost for ssh access"
  # Store key as variable
  pubkey=`cat ~/.ssh/$ESXiHost'_rsa.pub'`
  # Create directory for authorized user, copy key to target, set permissions,
  # and restart ssh service on target.
  ssh $ESXiUser@$ESXiHost "mkdir -p /etc/ssh/keys-$ESXiUser &&
  echo $pubkey > /etc/ssh/keys-$ESXiUser/authorized_keys &&
  chmod 700 -R /etc/ssh/keys-$ESXiUser &&
  chmod 600 /etc/ssh/keys-$ESXiUser/authorized_keys &&
  chown -R $ESXiUser /etc/ssh/keys-$ESXiUser &&
  /etc/init.d/SSH restart"
fi

# Enable UPnP http(s) port forward for requesting device
status "Enabling http(s) port forwarding to client for letsencrypt verification"
upnpc -e "letsencrypt http" -r 80 tcp
upnpc -e "letsencrypt https" -r 443 tcp

# Acquire letsencrypt cert
status "Requesting 4096 bit certificate for $ESXiHost"
sudo certbot certonly --standalone --preferred-challenges tls-sni --agree-tos -m $Email -d $ESXiHost --rsa-key-size 4096

# Backup existing SSL components on ESXi target
status "Backing up existing certificates on $ESXiHost"
time=$(date +%Y.%m.%d_%H:%M:%S)
ssh $ESXiUser@$ESXiHost "cp /etc/vmware/ssl/castore.pem /etc/vmware/ssl/castore.pem.back.$time"
ssh $ESXiUser@$ESXiHost "cp /etc/vmware/ssl/rui.crt /etc/vmware/ssl/rui.crt.back.$time"
ssh $ESXiUser@$ESXiHost "cp /etc/vmware/ssl/rui.key /etc/vmware/ssl/rui.key.back.$time"

# Copy letsencrypt cert to ESXi target
status "Coping letsencrypt cert to $ESXiHost"
sudo scp /etc/letsencrypt/live/$ESXiHost/fullchain.pem $ESXiUser@$ESXiHost:/etc/vmware/ssl/castore.pem
sudo scp /etc/letsencrypt/live/$ESXiHost/cert.pem $ESXiUser@$ESXiHost:/etc/vmware/ssl/rui.crt
sudo scp /etc/letsencrypt/live/$ESXiHost/privkey.pem $ESXiUser@$ESXiHost:/etc/vmware/ssl/rui.key

# Restart services on ESXi target
status "Restarting services on $ESXiHost"
ssh $ESXiUser@$ESXiHost "services.sh restart"

# Disable UPnP http(s) port forward
status "Removing http(s) port forwarding"
upnpc -d 80 tcp
upnpc -d 443 tcp

# Prompt user to confirm/disable SSH on ESXi target
pressanykey "Remember to disable SSH service on $ESXiHost"
```

### 引用相关
- [Mikrotkit About us](https://www.mikrotik.com/aboutus)
- [ESXi6.7安装流程(2019.7重编版)](https://www.jianshu.com/p/f989606a0331)
- [esxi-on-10th-gen-intel-nuc-comet-lake-frost-canyon](https://www.virten.net/2020/03/esxi-on-10th-gen-intel-nuc-comet-lake-frost-canyon/)