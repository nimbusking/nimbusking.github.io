---
title: 工具Tips
mathjax: false
tags: 工具使用
updated: 2024-10-28 20:30:21categories: 工具
abbrlink: a4b1030a
date: 2018-04-15 10:37:03
---

### 前言
本系列罗列日常工作学习中遇到的工具问题以及相应的解决方案，写下来供后续自己参考的同时也作分享一波！
### 编程相关
#### Sublime Text 3
##### win10下解决中文输入法光标不跟随问题
国内一大老Fork并修改之后的IMESupport[主页地址](http://zcodes.net/2017/02/12/sublime_text_3_imesupport.html)，在win10下使用完美解决。看下图所示：
![使用效果](a4b1030a/ChineseTypeWritingInSublimeText_CursorFollowing.jpg)
<!--more-->
可以直接从GitHub上下载下来，把插件路径解压到：**%AppData%\Sublime Text 3\Packages**
下即可，下图是我本地安装的：
![本地安装包](a4b1030a/SublimeTextInstalledPackages.jpg)
##### 安装Markdown Editing插件之后文本编辑区域过窄问题
跟该插件的默认设置有关，解决方案：
打开路径：Preferences->Package Settings->Markdown Editing->Markdown GFM Settings --User，在配置下添加关于**wrap_width**的属性，我修改成了130，如下图所示：
![MarkdownEditing用户配置](a4b1030a/MarkdownEditingUserConfiguration.jpg)
设置前效果：
![设置前](a4b1030a/BeforeConfiguring.jpg)
设置后效果：
![设置后](a4b1030a/AfterConfiguring.jpg)
**效果明显！**

##### Markdown Editing插件配置
推荐几个自己常用的配置：
```json
"color_scheme": "Packages/Boxy Theme/schemes/Boxy Monokai.tmTheme", // 修改风格的主题,我这里是sublime的boxy主题自带的,默认有这几种主题
    // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor.tmTheme",
    // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Dark.tmTheme",
    // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Yellow.tmTheme",
"highlight_line": true, // 高亮正在编辑的行
"line_numbers": true,   // 显示行号
"tab_size": 4,          // tab宽度
"translate_tabs_to_spaces": true,   // tab转换为空格
"trim_trailing_white_space_on_save": true,  // 保存时去掉行尾空格
"word_wrap": true,      // 自动换行
"wrap_width": "auto",    // 换行的宽度,默认80会造成左侧大量留白
"mde.keep_centered": true,  // 可以保持你正在编辑的行始终处于屏幕的中间
```

##### 打开文件新打开标签页，而不是不替换当前标签页
路径：Preferences->Settings
~~在默认设置页中设置即可~~： ```"open_files_in_new_window": true```
如果发现默认设置页面中就是上述设置，那么在右侧用户设置 ```"preview_on_click":false```
此时唯一区别就是打开文件的时候需要双击
##### 卸载插件
Ctrl+Shift+P，输入Remove，选择Remove Package，之后选择相应的插件，直接回车即可

##### 快速插入代码片段
打开代码片段（Snippet）: Tools -> Develop -> New Snippet
例如，我想要再markdown中快速生成插入shell代码的片段，那么可以安装下图中的方式编写，很简单：
这里面的语法很简单：
**<![CDATA[写入你想要快速插入的代码片段]]>** 这个XML转义票签里面就是写入你想要快速插入的代码片段
**tabTrigger**就是你需要再什么单词后按tab键自动插入上述你编写的代码片段
![配置Sublime Snippet](a4b1030a/ConfigSublimeSnippet.jpg)
Ctrl + S 保存，目录默认即可，文件名随意，文件后缀名必须是：**.sublime-snippet**，效果就如下图所示：
![演示效果](a4b1030a/shellAfterPressTabButton.gif)

##### 快速查找重复行
Ctrl + F 打开查找控制面板，输入：
``` javascript
^(.+)$[\r\n](^\1$[\r\n]{0,1})+
```
点击Find即可。

##### 设置行间距相关
在个人用户设置里面加上下面的两行json配置即可，段前段后增加间距，设置2个人感觉刚刚好：
```json
   "line_padding_bottom": 2,
   "line_padding_top": 2
```

### 办公相关

### Windows相关
#### WinDbg（windows 蓝屏分析工具）
近期周末在玩一款游戏的时候，老是会遇到蓝屏的情况，不清楚什么原因，可见的过程中对这个游戏送一串？？？？
##### 准备工作
windows蓝屏之后，通常都会在C:\Windows\Minidump该目录下面转存储相应的dump文件，如下图所示：
![windows蓝屏dump文件](a4b1030a/windowsbluescreendumpfiles.jpg)
此时需要用到微软官方dump文件分析工具，官方下载地址，如下：
[Windows SKD Tool Kits](https://developer.microsoft.com/en-US/windows/downloads/windows-10-sdk)
点击页面中的“DOWNLOAD THE INSTALLER”
下载一个约1.29MB大小的安装文件
双击运行之后，前面的安装路径自行填写，其中在“Select the Features you want to download”页面，如下图所示：
![Select the Features you want to download](a4b1030a/selectthefutures.jpg)
当中，特别注意“Debuging Tools for Windows”，这个便是我们想要的工具
**当然，如果你想最简单的方式，直接默认全选，一路next下去，也是可以的**
等待安装完成
*PS：可能要试具体网络环境，我这边很快，不到5分钟全部安装完成*
##### 分析dump文件之前的配置工作
在开始中找到WinDbg(x64)，运行，运行之后是这样的：
![windbg](a4b1030a/windbg.jpg)
点击：File->Symbol File Path，如下图所示：
![symbolspath](a4b1030a/symbolspath.jpg)
在Symbol Path会话框里输入如下路径：
``` shell
SRV*G:\Symbol*http://msdl.microsoft.com/download/symbols
```
其中G:\Symbol是各自symbols文件下载到本地的路径，可以自行配置

**如果不配置上述symbol配置会怎样？**
会提示你，如下一行错误：
```
Your debugger is not using the correct symbols 
```
是无法正常分析dump文件的

##### 开始分析dump文件
点击：File->Open Crash Dump，找到对应的dump文件
打开之后，开始由于上述配置symbols路径，会自动从msdl下载分析时遇到的symbols，如下图所示：
![开始下载symbols文件](a4b1030a/startdownloadsymbols.jpg)
**注意：此时左下角状态栏会显示debuggee not connected字样，这是正常现象，表示debug分析还在准备中，如果你注意最外层WinDbg左下角的状态栏，是可以看到相应的下载Symbols进程的，一个'/'符号在转圈，注意观察**
如果网络正常，你会看到在你配置下载的symbols路径下面会多了很多文件夹，如下图所示：
![已下载好的符号文件](a4b1030a/downloadedsymbols.jpg)
**但是，如果你长时间发现，只有一个文件夹，而且点进去，如果发现一个名为：download.error文件，而且这个文件不是0kb，那么你就要考虑可能是你的网络有问题，不能正常从msdl下载。一种可能DNS解析msdl就是有问题；另一种可能被ban了？没考证，我的解决方案是直接把这个域名msdl.microsoft.com加到黑名单里，走科学上网，发现很快就能正常下载symbols文件了。**下载好的内容如下图所示：
![一个成功下载的symbol文件](a4b1030a/onesymboldownloadsuccessfully.jpg)

稍等片刻，等变成如下图所示的页面，表面可进入到准备分析阶段：
![准备分析](a4b1030a/readytoanalys.jpg)
**此时注意，左下角状态已变**

点击上图中的**!analyze -v**链接，或者在下方状态栏的命令行中直接输入“!analyze -v”，两者等价。
等待分析完成之后，提供一则我的dump分析结果
```
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KMODE_EXCEPTION_NOT_HANDLED (1e)
This is a very common bugcheck.  Usually the exception address pinpoints
the driver/function that caused the problem.  Always note this address
as well as the link date of the driver/image that contains this address.
Arguments:
Arg1: ffffffffc0000005, The exception code that was not handled
Arg2: fffff8036f6d2316, The address that the exception occurred at
Arg3: 0000000000000000, Parameter 0 of the exception
Arg4: 0000000000000014, Parameter 1 of the exception

Debugging Details:
------------------


KEY_VALUES_STRING: 1


TIMELINE_ANALYSIS: 1


DUMP_CLASS: 1

DUMP_QUALIFIER: 400

BUILD_VERSION_STRING:  17134.1.amd64fre.rs4_release.180410-1804

SYSTEM_MANUFACTURER:  System manufacturer

SYSTEM_PRODUCT_NAME:  System Product Name

SYSTEM_SKU:  SKU

SYSTEM_VERSION:  System Version

BIOS_VENDOR:  American Megatrends Inc.

BIOS_VERSION:  1009

BIOS_DATE:  07/23/2017

BASEBOARD_MANUFACTURER:  ASUSTeK COMPUTER INC.

BASEBOARD_PRODUCT:  PRIME Z270-A

BASEBOARD_VERSION:  Rev 1.xx

DUMP_TYPE:  2

BUGCHECK_P1: ffffffffc0000005

BUGCHECK_P2: fffff8036f6d2316

BUGCHECK_P3: 0

BUGCHECK_P4: 14

READ_ADDRESS: fffff8036f661388: Unable to get MiVisibleState
Unable to get NonPagedPoolStart
Unable to get NonPagedPoolEnd
Unable to get PagedPoolStart
Unable to get PagedPoolEnd
 0000000000000014 

EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - <Unable to get error code text>

FAULTING_IP: 
nt!CmpCreateKeyBody+166
fffff803`6f6d2316 0fb74514        movzx   eax,word ptr [rbp+14h]

EXCEPTION_PARAMETER2:  0000000000000014

BUGCHECK_STR:  0x1E_c0000005_R

CPU_COUNT: 8

CPU_MHZ: 1068

CPU_VENDOR:  GenuineIntel

CPU_FAMILY: 6

CPU_MODEL: 9e

CPU_STEPPING: 9

CPU_MICROCODE: 6,9e,9,0 (F,M,S,R)  SIG: 70'00000000 (cache) 70'00000000 (init)

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXPNP: 1 (!blackboxpnp)


CUSTOMER_CRASH_COUNT:  1

DEFAULT_BUCKET_ID:  WIN8_DRIVER_FAULT

PROCESS_NAME:  services.exe

CURRENT_IRQL:  0

ANALYSIS_SESSION_HOST:  KEMI-PC

ANALYSIS_SESSION_TIME:  08-18-2018 12:38:32.0201

ANALYSIS_VERSION: 10.0.17134.12 amd64fre

EXCEPTION_RECORD:  ffffd5896efe0b10 -- (.exr 0xffffd5896efe0b10)
ExceptionAddress: ffffd5896ef03b50
   ExceptionCode: 01000000
  ExceptionFlags: 00000002
NumberParameters: 1896624160
   Parameter[0]: 0000000000000000
   Parameter[1]: 0000000000000000
   Parameter[2]: ffffd5896f0037b0
   Parameter[3]: ffffd5896ec217e0
   Parameter[4]: ffffd5896e780ba0
   Parameter[5]: 0000000000000000
   Parameter[6]: 00000000000a000a
   Parameter[7]: ffffd5896efe0f80
   Parameter[8]: 0000000000180018
   Parameter[9]: ffffd5896efe0f8a
   Parameter[10]: ffffd5896f9a8bd0
   Parameter[11]: ffffd5896e780c70
   Parameter[12]: 0000000000000000
   Parameter[13]: ffffd5896eeff8b0
   Parameter[14]: 0000000000000000

LAST_CONTROL_TRANSFER:  from fffff8036f2e25dd to fffff8036f3bcca0

STACK_TEXT:  
ffff8689`0c47e3c8 fffff803`6f2e25dd : 00000000`0000001e ffffffff`c0000005 fffff803`6f6d2316 00000000`00000000 : nt!KeBugCheckEx
ffff8689`0c47e3d0 fffff803`6f3cd942 : ffffd589`6efe0b10 ffffd589`7a2e8730 00000000`00000000 ffffd589`6b5e4938 : nt!KiDispatchException+0x58d
ffff8689`0c47ea80 fffff803`6f3ca4bf : 00000000`00000000 00000000`00000040 ffffd589`69bc6420 ffffc10a`55a3e398 : nt!KiExceptionDispatch+0xc2
ffff8689`0c47ec60 fffff803`6f6d2316 : 00000000`00000010 ffffd589`77284e40 00000000`00000030 ffff8689`0c47ee50 : nt!KiPageFault+0x3ff
ffff8689`0c47edf8 ffff8689`0c47f090 : 00000000`00000001 ffff8689`0c47f840 00000000`00000000 00000000`00000000 : nt!CmpCreateKeyBody+0x166
ffff8689`0c47eea8 00000000`00000001 : ffff8689`0c47f840 00000000`00000000 00000000`00000000 ffff8689`0c47ef80 : 0xffff8689`0c47f090
ffff8689`0c47eeb0 ffff8689`0c47f840 : 00000000`00000000 00000000`00000000 ffff8689`0c47ef80 ffff8689`0c47ef19 : 0x1
ffff8689`0c47eeb8 00000000`00000000 : 00000000`00000000 ffff8689`0c47ef80 ffff8689`0c47ef19 ffff8689`0c47efd0 : 0xffff8689`0c47f840


THREAD_SHA1_HASH_MOD_FUNC:  cc68484f0ad7b2ba7711fc62c7e22b20f74cd551

THREAD_SHA1_HASH_MOD_FUNC_OFFSET:  09da40a0301b7e372ce92d3ab9bddb9d5a57ce52

THREAD_SHA1_HASH_MOD:  f08ac56120cad14894587db086f77ce277bfae84

FOLLOWUP_IP: 
nt!CmpCreateKeyBody+166
fffff803`6f6d2316 0fb74514        movzx   eax,word ptr [rbp+14h]

FAULT_INSTR_CODE:  1445b70f

SYMBOL_STACK_INDEX:  4

SYMBOL_NAME:  nt!CmpCreateKeyBody+166

FOLLOWUP_NAME:  MachineOwner

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

DEBUG_FLR_IMAGE_TIMESTAMP:  5b63c7b5

IMAGE_VERSION:  10.0.17134.228

STACK_COMMAND:  .thread ; .cxr ; kb

BUCKET_ID_FUNC_OFFSET:  166

FAILURE_BUCKET_ID:  0x1E_c0000005_R_nt!CmpCreateKeyBody

BUCKET_ID:  0x1E_c0000005_R_nt!CmpCreateKeyBody

PRIMARY_PROBLEM_CLASS:  0x1E_c0000005_R_nt!CmpCreateKeyBody

TARGET_TIME:  2018-08-18T03:01:10.000Z

OSBUILD:  17134

OSSERVICEPACK:  228

SERVICEPACK_NUMBER: 0

OS_REVISION: 0

SUITE_MASK:  272

PRODUCT_TYPE:  1

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

OSEDITION:  Windows 10 WinNt TerminalServer SingleUserTS

OS_LOCALE:  

USER_LCID:  0

OSBUILD_TIMESTAMP:  2018-08-03 11:10:45

BUILDDATESTAMP_STR:  180410-1804

BUILDLAB_STR:  rs4_release

BUILDOSVER_STR:  10.0.17134.1.amd64fre.rs4_release.180410-1804

ANALYSIS_SESSION_ELAPSED_TIME:  ff90

ANALYSIS_SOURCE:  KM

FAILURE_ID_HASH_STRING:  km:0x1e_c0000005_r_nt!cmpcreatekeybody

FAILURE_ID_HASH:  {c4b33015-e33f-270e-d1a4-e4a9a8d630df}

Followup:     MachineOwner
---------
```
##### 分析结论
从上述结果中，注意观察下述字眼：
```
DEFAULT_BUCKET_ID:  WIN8_DRIVER_FAULT

PROCESS_NAME:  services.exe
```
错误原因是**WIN8_DRIVER_FAULT**，错误进程是**services.exe**;在另一个dump文件中，错误原因一样，错误进程是**dwm.exe**，俩程序都是windows自身程序。结合错误原因，大致猜测：**可能是显卡驱动程序问题**。
##### 处理观察
打开Geforce Experence，更新N卡驱动，之后继续观察还有没有蓝屏发生
#### 又一次蓝屏分析
昨晚惯例周五晚上打游戏的时间（Gaming Night），玩着玩着，就蓝屏了。
蓝屏页面没有拍，报的是：SYSTEM_SERVICE_EXCEPTION的错误。
重启之后，WinDbg打开分析一下，发现是OneDrive的锅。
![SystemServiceException](a4b1030a/SystemServiceException.png)
很有可能是最近重新做系统之后，OneDrive一直在更新状态导致的，具体原因不知。
##### 解决过程
卸载OneDrive，官网重新下载安装包。重新安装。**安装好之后，重新指定新的路径为OD的同步路径**
##### 问题分析
怀疑是重装系统之后，直接用的老的OD的同步路径，导致OD一直是同步文件的状态。最终在某次偶然的机会，崩溃蓝屏了。。。