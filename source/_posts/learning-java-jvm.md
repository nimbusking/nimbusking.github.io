---
title: 深入理解Java虚拟机（第三版）
abbrlink: d7ba81a7
date: 2020-08-26 20:48:41
tags:
    - JVM
    - 周志明
categories: Java系列
---

## 书籍实录
### 第一章：走进JAVA
#### Java发展史
宏观上了解Java整个发展体系：
- 1995年5月23日，Oak语言改名为Java，并且在SunWorld大会上正式发布Java1.0版本，并且提出“Write Once, Run Anyware”的口号。
- 1996年1月23日，JDK1.0发布，代表技术包括：Java虚拟机、Applet、AWT等。
<!-- more -->
- 1997年2月19日，Sun公司发布JDK1.1，技术代表有：JAR文件格式、JDBC、JavaBeans、RMI等。语法特性有一定增强：如内部类（Inner Class）和反射（Reflaction）
- 1998年12月4日，JDK1.2发布（工程代号：Playground），Sun拆分Java技术体系为三个方向：分片是面向桌面应用开发的J2SE、面向企业开发的J2EE、面向手机等移动终端开发的J2ME。技术代表如：EJB、Java Plug-in、Java IDL、Swing等，并且在这个版本中Java虚拟机第一次内置了JIT（Just In Time）及时编译器。
- 2000年5月8日，JDK1.3发布（工程代号：Kestrel【美洲红隼】），代表技术：Java类库，使用CORBAIIOP来实现RMI的通信协议。
- 2002年2月13日，JDK1.4发布（工程代号：Merlin【灰背隼】）技术特性例如：正则表达式、异常链、NIO、日志类、XML解析器和XSLT转换器等等。
- 2004年9月30日，JDK5发布（工程代号：Tiger【老虎】）语法特性加强：自动装箱、泛型、动态注解、枚举、可变长参数、循环遍历（foreach循环）等等。虚拟机上，改进了Java内存模型（Java Memory Model，JMM）、提供了java.util.concurrent并发包等。
- 2006年12月11日，JDK6发布（工程代号：Mustang【野马】）改进的包括：提供初步的动态语言支持（通过内置的Mozilla JavaScript Rhino引擎实现）、提供编译期注解处理器和微型HTTP服务器API，等等。Java虚拟机内部做了大量改进，包括锁与同步、垃圾收集、类加载等方面。
- 2011年7月28日，JDK7正式版发布（工程代号Dopphin【海豚】）改进的包括：提供新的G1收集器、加强对非Java语言的调用支持、可并行的类加载架构等。
- 2014年3月18日，JDK8发布，主要功能特性包括：
    + JEP 126: 对Lambda表达式的支持，这让Java语言拥有了流畅的函数式表达能力
    + JEP 104: 内置Nashorn JavaScript引擎的支持
    + JEP 150: 新的时间、日期API
    + JEP 122: 彻底移除HotSpot的永久代
    + .....
- 2017年9月21日，JDK9艰难面世：Jigsaw模块化功能、增强若干工具（JS Shell、JLink、JHSDB等），整顿了HotSpot各个模块各自为战的日志系统，支持HTTP2客户端API等91个JEP
- 2018年3月20日，JDK10发布：主要目标，内部重构：统一源仓库、统一垃圾收集器接口、统一即时编译器接口（JVMCI在JDK9已经有了，这里引入新的Graal即时编译器）
- 2018年9月25日，JDK11发布：包含17个JEP，其中有ZGC这样的革命性垃圾收集器出现，也有把JDK10中的类型推断加入Lambda语法这种可见性的改善
- 2019年3月20日，JDK12发布：包含9个JEP：Switch表达式、Java微测试套件（JMH）等新功能，最引人注目的特性无疑是加入了由ReaHat
- 领导开发的Shenadnoah垃圾收集器。

#### Java虚拟机家族
- 虚拟机始祖：Sun Classiz/Exact VM
- 武林盟主：HotSpot VM
    HotSpot虚拟机的热点代码探测能力可以通过执行计数器找出最具有编译价值的代码，然后通知及时编译器以方法为单位进行编译。
- 小家碧玉：Mobile/Embedded VM
- 天下第二：BEA JRockit/IBM J9 VM
- 软硬合璧：BEA Liquid VM/Azul VM
    Azul VM实例都可以管理至少数十个CPU和数百GB的内存的硬件资源，并提供巨大内存范围内停顿时间可控的垃圾收集器（即业内赫赫有名的PGC和C4收集器）
    Zing虚拟机是一个从HotSpot某个旧版代码分支基础上独立出来重新开发的高性能Java虚拟机。要求低延迟、快速预热等场景中，Zing VM都要比HotSpot表现的更好。Zing的PGC、C4收集器可以轻易支持TB级别的Java堆内存，而且保证暂停时间仍然可以维持在不超过10毫秒的范围内，HotSpot要等到JDK11和JDK12的ZGC及Shenandoah收集器才达到了相同的目标，而且目前的效果仍然远不如C4。
- 挑战者：Apache Harmony/Google Android Dalvik VM
- 没有成功，但并非失败：Microsoft JVM及其它

### 第二章 Java内存区域与内存溢出异常
#### 运行时数据区域
Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域有各自的用途，以及创建和销毁的时间，有的区域随着需积极进程的启动而一直存在，有些区域则是以来用户线程的启用和结束而建立和销毁。分为：方法区（Method Area），虚拟机栈（VM Stack），本地方法栈（Native Method Stack），堆（Heap），程序计数器（Program Counter Register）

##### 程序计数器
一块较小的内存空间，它可以看做是当前线程所执行的字节码的行号指示器。

##### Java虚拟机栈
线程私有，生命周期与线程相同。
虚拟机栈描述的是Java方法执行的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等等。
每一个方法调用直至执行完毕的过程，就对应着一个栈帧在虚拟机占中从入栈到出站的过程。

## 实战相关