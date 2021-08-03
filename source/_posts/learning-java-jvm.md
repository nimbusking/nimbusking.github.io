---
title: 深入理解Java虚拟机（第三版）
abbrlink: d7ba81a7
date: 2020-08-26 20:48:41
tags:
    - JVM
    - 周志明
categories: Java系列
---

# 背景
JVM作为Java底层的核心，一直以来是我的短板，其底层等相关不甚了解。借此机会，通过这种方式学习，记录，分析&分享。
书中涉及代码调试案例，收录在[learningjvm](https://github.com/nimbusking/learningjvm)项目中，各看官有需要可以自行查阅。

# 书籍实录
## 第一章：走进JAVA
### Java发展史
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

### Java虚拟机家族
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

## 第二章 Java内存区域与内存溢出异常
### 运行时数据区域
Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域有各自的用途，以及创建和销毁的时间，有的区域随着需积极进程的启动而一直存在，有些区域则是以来用户线程的启用和结束而建立和销毁。分为：方法区（Method Area），虚拟机栈（VM Stack），本地方法栈（Native Method Stack），堆（Heap），程序计数器（Program Counter Register）

#### 程序计数器
一块较小的内存空间，它可以看做是当前线程所执行的字节码的行号指示器。

#### Java虚拟机栈
线程私有，生命周期与线程相同。
虚拟机栈描述的是Java **方法执行**的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等等。
每一个方法调用直至执行完毕的过程，就对应着一个栈帧在虚拟机占中从入栈到出站的过程。

“栈”通常就是指这里讲的虚拟机栈，或者更多情况下只是指虚拟机栈中的局部变量表部分。

**局部变量表**存放了编译期可知的各种Java虚拟机基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。
这些数据类型在局部变量表中的存储空间以 **局部变量槽（slot）**来表示，其中64位长度的long和double类型的数据会占用两个变量槽，其余的数据类型只占用一个。

在《Java虚拟机规范》中，对栈区域规定了两类异常情况：
- 如果线程请求的栈深度大雨虚拟机所允许的深度，将抛出 **StackOverflowError**异常
- 如果Java虚拟机栈容量可以动态扩展（注：HotSpot虚拟机是不可以动态扩展的，只要线程申请成功了就不会有OOM，但是如果申请时就失败，仍然会出现OOM异常），当栈扩展时无法申请到足够的内存会抛出 **OutOfMemoryError**异常。

#### 本地方法栈
Native Method Stack与虚拟机栈类似，一个是位虚拟机执行Java方法（字节码）服务；一个是为虚拟机用到的本地（native）方法服务。

#### Java堆
Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域唯一的目的就是存放对象实例，Java世界里“几乎”所有对象实例都在这里分配内存。

Java堆是垃圾收集器管理的内存区域。从回收内存角度看，由于现代垃圾收集器大部分都是基于分代收集理论设计的，所以Java堆中经常会出现“新生代”，“老年代”，“永久代”，“Eden空间”，“From Survivor空间”，“To Survivor空间”等名词。

无论从什么角度，无论如何划分，都不会改变Java堆中存储内容堆共性，无论是哪个区域，存储的都只能是对象的实例，将Java堆细分的目的 **只是为了更好地回收内存，或者更快地分配内存**。

Java堆即可以被实现成固定大小，也可以是扩展的，不过当前主流的Java虚拟机都是按照可扩展来实现的（通过参数-Xmx和-Xms设定）。
**如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，Java虚拟机将会抛出OutOfMemoryError异常**

#### 方法区
与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的 **类型信息**、**常量**、**静态变量**、即时编译器编译后的代码缓存数据等。
方法区与“永久代”本来是两个概念，**仅仅因为**当时的HotSpot虚拟机设计团队选择把收集器的分代设计扩展至方法区，或者说使用“永久代”实现方法区而已，这样使得HotSpot的垃圾收集器能像管理Java堆一样管理这部分内存，省区专门为方法区编写内存管理代码的工作。

- JDK6时HotSpot开发团队就有放弃永久代，逐步改为采用本地内存（Native Memory）来实现方法区的计划里
- JDK7时，已经把原本放在永久代的字符串常量池、静态变量移出
- JDK8时，完全放弃永久代概念，该用与JRockit、J9一样在本地内存中实现的元空间（Metaspace）来代替，把JDK7中剩余的主要是类型信息，全部移动到元空间中。

**如果方法区无法满足新的内存分配需求时，将抛出OutOfMemoryError异常**

#### 运行常量池
Runtime Constant Pool是方法区的一部分。Class文件中除了有**类的版本**、**字段**、**方法**、**接口**等描述信息外，还有一项信息是**常量池表**。主要用于：**存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中**。

Java语言并不要求常量一定只有编译期才能产生，也就是说，并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可以将新的常量放入池中，这种特性被开发人员利用的较多的是String类的 **intern()**方法。

**当常量池无法再申请到内存时会抛出OutOfMemoryError（本来就是方法区一部分，自然收到方法区内存限制）**

#### 直接内存
Direct Memory并不是虚拟机运行时数据区的一部分，也不是《Java虚拟机》中定义的内存区域。**这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现**

##### 背景
在JDK1.4中新加入了NIO（New Input/Output）类。引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，**它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据**。
##### 问题
本机直接内存分配不受到Java堆大小限制，但是，既然时内存，肯定还是会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制。
**一般服务器管理员配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息**，但是会常常忽略直接内存，使得各个内存区域总和大于物理内存限制，**从而导致动态扩展时出现OutOfMemoryError异常**

### HotSpot虚拟机对象探秘
注：书中探讨的是常用的HotSpot虚拟机的Java堆中的 **对象分配、布局和访问的全过程**。
#### 对象的创建（过程）
- 检查过程
当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到另一个类的**符号引用**，并且检查这个符号引用代表的类是否加载、解析和初始化过。如果没有，那么必须先执行相应的**类加载过程**。
- 分配过程
类加载检查通过之后，接下来虚拟机将为新生对象分配内存。对象所需内存的大小在类加载完成之后便可完全确定，对象分配空间的任务实际上便等同于吧一块确定大小的内存块从Java堆中划分出来。
当使用Serial、ParNew等带压缩整理过程的收集器时，系统采用的分配算法是**指针碰撞**，既简单又高效。而当使用CMS这种基于清除（Sweep）算法的收集器时，理论上就只能采用较为复杂的**空闲列表**来分配内存。

**分配过程中的操作原子性**
    - 采用CAS配上失败重试的机制
    - 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在java堆中预先分配一小块内存，成为本地线程分配缓冲（Thread Local Allocation Buffer, TLAB），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓冲区时才需要同步锁定。**虚拟机是否需要使用TLAB，可以通过-XX：+/-UseTLAB参数来设定**
- 初始化过程
内存分配完成之hi，虚拟机必须将分配到内存空间（但不包括对象头）都初始化成零值。
    + 对象设置
        对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。
    + 构造函数初始化
        new指令之后会接着执行<init>()方法，按照程序员的意愿对对象进行初始化。

#### 对象的内存布局
在HotSpot虚拟机中，对象在堆内存中的存储布局可以划分为三个部分：**对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）**。
##### 对象头信息
- 一类：用于存储对象自身运行时数据，如：哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等
- 另一类：类型指针，即对象指向它的类型元数据的指针，Java虚拟机通过这个指针来确定该对象时哪个类型的实例。

##### 实例数据信息
程序代码中定义的各种类型字段内容，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。*这部分的存储顺序会收到虚拟机分配策略参数（-XX:FieldsAllocationStyle参数）和字段在Java源码中的定义顺序的影响*
##### 对齐填充
不是必然存在，也没有特别的含义，仅仅起着占位符的作用。任何对象大小都必须是8字节的整数倍。如果对象实例数据部分没有对齐的话，就需要通过对齐填充来补全。

#### 对象的访问定位
Java程序会通过栈上的reference数据来操作对上的具体对象，但由于reference类型在《Java虚拟机规范》里面只规定了 **它是一个指向对象的引用**，并没有定义这个引用应该通过什么方式去定位、访问到堆中对象的具体位置。主流的访问方式有：**句柄** 和 **直接指针**。
- 句柄访问：Java堆中可能会划分出一块内存来作为 *句柄池*，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息
- 直接指针访问：reference中存储的直接就是对象的地址

![通过句柄访问内存](d7ba81a7/memory_hander_detect.png)
![通过直接指针访问内存](d7ba81a7/direct_memory_detect.png)

#### 实战：OutOfMemoryError异常
**注：** 书中介绍了很多实战经验，这里仅记录一些经验相关事项，具体实例代码参考书中详细说明
本节实战的主要目的有两个：
- 通过代码验证《Java虚拟机规范》中描述的各个运行时区域存储的内容
- 遇到实际的内存溢出异常时，能够根据异常的提示信息迅速得知时哪个区域的内存溢出，知道怎样的代码可能会导致这些区域内存溢出，以及出现这些异常后该如何处理。

##### Java堆溢出
**根本原因：** Java堆用于存储对象实例，只要不断创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，那么随着对象数量增加，总容量触及最大堆的容量限制后就会产生内存溢出异常。
**常规处理方法：**
- 先通过内存映像分析工具（如Eclipse Memory Analyzer）对Dump出来的堆转存储快照进行分析，首先确认内存中导致的OOM的对象是否时必要的，也就是到底时出现了 **内存泄露（Memory Leak）** 还是 **内存溢出（Memory Overflow）**
- 如果是内存泄漏，可以通过工具查看泄漏对象到GC Roots的引用链，*找到泄漏对象是通过怎样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收*，根据泄漏对象的类型信息以及它到GC Roots引用链的信息，一般可以比较准确的定位到这些对象的创建位置，进而找出产生内存泄漏的代码的具体位置。
- 如果不是内存泄漏，就是对象必须要存活的话，就要分析Java虚拟机堆参数（-Xmx与-Xms）设置，与机器内存相比，是否还是有上调的空间。*再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等等，尽量减少程序运行期间的内存消耗。*

##### 虚拟机和本地方法栈溢出
《Java虚拟机规范》中描述了两种情况:
- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常
- 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出OutOfMemoryError异常
通常默认参数，对于正常的调用（包括不能做尾递归优化的递归调用），默认深度（1000~2000）完全够用。如果建立过多线程导致的内存溢出，在不能减少线程数量或者更换64位虚拟机的情况下，**就只能通过减少最大堆和减少栈容量** 来换取更多的线程

##### 方法区和运行时常量池溢出
方法区溢出也是一种常见的内存溢出异常，一个类如果要被垃圾收集器回收，要达成的条件比较苛刻。
在JDK8以后，永久代便完全退出了历史舞台，元空间作为其替代者登场，HotSpot提供了一些参数作为原空间的防御措施，主要包括：
- -XX: MaxMetaspaceSize: 设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小
- -XX: MetaspaceSize：指定元空间初始空间大小，以字节为单位，达到该值就会触发垃圾收集器进行类型卸载，同时收集器会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过XX: MaxMetaspaceSize（如果设置了的话）情况下，适当提高该值
- -XX: MinMetaspaceFreeRatio：作用是在垃圾收集器之后控制最小的元空间剩余容量的百分比，可减少因为元空间不足导致的垃圾收集的频率。类似的还有 ```-XX: MinMetaspaceFreeRatio``` 控制最大的元空间剩余容量的百分比。

##### 本机直接内存溢出
**注：** 至少本人目前从来没有直接接触到在Java程序层面涉及到的直接内存分配的场景，所以这节不作记录。

## 第三章 垃圾收集器与内存分配策略
### 对象已死
#### 关于引用计数算法
可以通过一个循环引用的Java程序实例就可以从侧面判定出，Java虚拟机并不是通过引用计数算法来判断对象是否存活的。
#### 可达性分析算法
当前主流的商用程序语言的内存管理子系统，都是通过 **可达性分析(Reachability Analysis)** 算法来判定对象是否存活的。
**基本思想：** 通过一系列称为“GC Roots”的根对象作为起始点集，从这些点开始，根据引用向下搜索，搜索过程所走的路径称之为“引用链”（Reference Chain），如果某个对象到GC Roots间没有任何引用链（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连，或者说（图论）从GC Roots到这个对象不可达时，则证明此对象时不可能再引用的。

Java技术体系中，可固定作为GC Roots对象包括以下几种类型：
- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量。
- 在方法区中 **类静态属性** 引用的对象，譬如Java类的引用类型静态变量
- 在本地方法栈中JNI（即通常所说的Native方法）引用的对象。
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如NullPointException、OutOfMemoryError）等，还有系统类加载器。
- 所有被同步锁（synchronized关键字）持有的对象。
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。

除了这些固定GC Roots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其它对象“临时性”地加入，共同构成完整的GC Roots集合。譬如后文将会提到的 **分代收集** 和 **局部回收（Partial GC）**

#### 再谈引用
在JDK1.2之后，Java对引用的概念进行了扩充，将引用分为 **强引用（Strongly Re-ference）**、**软引用（Soft Reference）**、**弱引用（Weak Reference）**、和 **虚引用（Phantom Reference）** 这4种，这4种引用强度依次逐渐减弱。

- 强引用是最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Object obj=new Object()”这种引用关系。无论任何情况下，**只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。**
- 软引用是用来描述一些还有用，但非必须的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK 1.2版之后提供了SoftReference类来实现软引用。
- 弱引用也是用来描述那些非必须对象，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK 1.2版之后提供了WeakReference类来实现弱引用。
- 虚引用也称为“幽灵引用”或者“幻影引用”，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。 **为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知。** 在JDK 1.2版之后提供了PhantomReference类来实现虚引用。

#### 生存还是死亡
要真正宣告一个对象死亡，至少要经历两次标记的过程：
- 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那他将会被第一次标记
- 随后进行一次筛选，筛选的条件是次对象是是否必要执行finalize()方法。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，那么虚拟机将这两种情况都视为“没有必要执行”。

#### 回收方法区
《Java虚拟机规范》中提到过可以不要求虚拟机在方法区中实现垃圾收集器。
方法区的垃圾收集主要回收两个部分： **废弃的常量和不再使用的类型**
- 回收废弃常量与回收Java堆中的对象非常相似，如果没有任何字符串对象引用常量池中的常量，且虚拟机中也没有其它地方引用这个字面量。如果此时发生内存回收，而且垃圾收集器判定必要的话，则这个常量会被回收。
- 判定一个类型是否属于“不再被使用的类”的条件就比较苛刻，需要同时满足下面三个条件：
    + 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例
    + 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的
    + 该类对应的java.lang.Class对象没有被任何地方引用，无法再任何地方通过反射访问该类。


### 垃圾收集算法
#### 分代收集理论
两个分代假说：
- 弱分代假说（Weak Generational Hypothesis ）：绝大多数对象都是朝生夕灭的。
- 强分代假说（String Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。
这两个分代假说共同奠定了多款常用的垃圾收集器的一致的设计原则：收集器应该将Java堆划分出不同的区域，然后将回收对象一句其年龄（年龄即对象熬过垃圾收集的过程的次数）分配到不同的区域之中存储。
在Java堆划分出不同的区域之后，垃圾收集器才可以每次只回收其中某一个或者某些部分的区域——因而才有了 **“Minor GC”“Major GC”“Full GC”** 这样的回收类型的划分；也才能够针对不同的区域安排与里面存储对象存亡特征相匹配的垃圾收集算法——因而发展出了 **“标记-复制算法”“标记-清除算法”“标记-整理算法”** 等针对性的垃圾收集算法。

通常意义下，直接可以按照不同内存区域中的对象，直接回收，但是往往存在一个明显的困难： **对象不是孤立的，对象之间会存在跨代引用**，为了解决这个问题，就需要对分代收集理论添加第三条经验法则：
- 跨代引用假说（Intergenrational Reference Hypothesis）：跨代引用相对于同代引用来说仅占极少数。

统一定义：
- 部分收集（Partial GC）：指 **目标不是完整收集整个Java堆的垃圾收集**，其中又分为
    + 新生代收集（Minor GC/Young GC）：**指目标只是新生代的垃圾收集。**
    + 老年代收集（Major GC/Old GC）：**指目标只是老年代的垃圾收集。** 目前只有CMS收集器会有单独收集老年代的行为。另外请注意“Major GC”这个说法现在有点混淆，在不同资料上常有不同所指，读者需按上下文区分到底是指老年代的收集还是整堆收集。
    + 混合收集（Mixed GC）：指目标是 **收集整个新生代以及部分老年代的垃圾收集**。目前只有G1收集器会有这种行为
- 整堆收集（Full GC）：**收集整个Java堆和方法区的垃圾收集**

#### 标记-清除算法
算法分为 **“标记”和“清除”** 两个阶段：首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回收所有未被标记的对象。
该算法主要的两个问题：
- 第一个是执行效率不稳定，如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须 *进行大量标记和清除的动作*，导致标记和清除两个过程的执行效率都随对象数量增长而降低。
- 第二个是 **内存空间的碎片化问题**，标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

#### 标记-复制算法
“半区复制”（Semispace Copying）的垃圾收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。

##### 该算法改进-Appel式回收
称为“Appel式回收”。HotSpot虚拟机的Serial、ParNew等新生代收集器均采用了这种策略来设计新生代的内存布局。
Appel式回收的具体做法是：**把新生代分为一块较大的Eden空间和两块较小的Survivor空间**，每次分配内存只使用Eden和其中一块Survivor。发生垃圾搜集时，将Eden和Survivor中仍然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1，也即每次新生代中可用内存空间为整个新生代容量的90%（Eden的80%加上一个Survivor的10%），只有一个Survivor空间，即10%的新生代是会被“浪费”的。

**特例：** Appel式回收还有一个充当罕见情况的“逃生门”的安全设计，**当Survivor空间不足以容纳一次Minor GC之后存活的对象时，就需要依赖其他内存区域（实际上大多就是老年代）进行分配担保（Handle Promotion）。**

#### 标记-整理算法
1974年Edward Lueders提出了另外一种有针对性的“标记-整理”（Mark-Compact）算法，其中的标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，**而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存**，“标记-整理”算法的示意图如图所示：
![“标记-整理”算法示意图](d7ba81a7/mark_compact_demostration.png)

### HotSpot的算法细节实现
**注：** 这一节非常琐碎，对算法实现细节做了很多阐述，核心就是怎么高效的收集？与之带来的问题基本上是一环套一环的，可以暂时先了解，如遇到特定场景、特定问题之后，再回过头来啃一啃这里面的实现。暂时，也就记录一些名词，不去理解具体的实现细节。
#### 根节点枚举
迄今为止，所有收集器在根节点枚举这一步骤时都是必须 **暂停用户线程** 的，因此毫无疑问根节点枚举与之前提及的整理内存碎片一样会面临相似的“Stop The World”的困扰。具体在HotSpot中的做法是：
使用一组称为 **OopMap的数据结构** 来达到这个目的。一旦类加载动作完成的时候，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译（见第11章）过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用。这样收集器在扫描时就可以直接得知这些信息了，并不需要真正一个不漏地从方法区等GC Roots开始查找。
#### 安全点
前面提到的OopMap存在一个问题： *可能导致引用关系变化，或者说导致OopMap内容变化的指令非常多，如果为每一条指令都生成对应的OopMap，那将会需要大量的额外存储空间，这样垃圾收集伴随而来的空间成本就会变得无法忍受的高昂。*

实际上HotSpot也的确没有为每条指令都生成OopMap，前面已经提到，只是在“特定的位置”记录了这些信息，**这些位置被称为安全点（Safepoint）。**
安全点位置的选取基本上是以“是否具有让程序长时间执行的特征”为标准进行选定的，因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这样的原因而长时间执行，**“长时间执行”的最明显特征** 就是 **指令序列的复用**，例如方法调用、循环跳转、异常跳转等都属于指令序列复用，所以只有具有这些功能的指令才会产生安全点。

对于安全点，另外一个需要考虑的问题是，如何在垃圾收集发生时让所有线程（这里其实不包括执行JNI调用的线程）都跑到最近的安全点，然后停顿下来。这里有两种方案可供选择：**抢先式中断（Preemptive Suspension）和主动式中断（Voluntary Suspension）**

#### 安全区域
安全点机制保证了程序执行时，在不太长的时间内就会遇到可进入垃圾收集过程的安全点。但是，程序“不执行”的时候呢？
所谓的程序不执行就是没有分配处理器时间，典型的场景便是用户线程处于Sleep状态或者Blocked状态，这时候 *线程无法响应虚拟机的中断请求，不能再走到安全的地方去中断挂起自己，虚拟机也显然不可能持续等待线程重新被激活分配处理器时间。* 对于这种情况，就必须引入 **安全区域（Safe Region）** 来解决。

**安全区域是指能够确保在某一段代码片段之中，引用关系不会发生变化，因此，在这个区域中任意地方开始垃圾收集都是安全的。我们也可以把安全区域看作被扩展拉伸了的安全点。**

#### 记忆集与卡表
##### 诞生的背景
垃圾收集器在**新生代**中建立了名为记忆集（Remembered Set）的数据结构，用以避免把整个老年代加进GC Roots扫描范围。

记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。

### 写屏障
在HotSpot虚拟机里是通过写屏障（Write Barrier）技术维护卡表状态的。

### 并发的可达性分析
想解决或者降低用户线程的停顿，就要先搞清楚为什么必须在一个能保障一致性的快照上才能进行对象图的遍历？
我们引入**三色标记（Tri-color M arking）**作为工具来辅助推导，把遍历对象图过程中遇到的对象，按照“是否访问过”这个条件标记成以下三种颜色：
- **白色：** 表示对象**尚未**被垃圾收集器访问过。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。
- **黑色：** 表示对象**已经**被垃圾收集器访问过，且这个对象的**所有引用**都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。黑色对象不可能直接（不经过灰色对象）指向某个白色对象。
- **灰色：** 表示对象已经被垃圾收集器访问过，但这个对象上**至少存在一个引用**还没有被扫描过。

### 经典垃圾收集器
说明：使用“经典”二字是为了与几款目前仍处于实验状态，但执行效果上有革命性改进的高性能低延迟收集器区分开来，这些经典的收集器尽管已经算不上是最先进的技术，但它们曾在实践中千锤百炼，足够成熟，基本上可认为是现在到未来两、三年内，能够在商用生产环境上放心使用的全部垃圾收集器了。
![HotSpot虚拟机的垃圾收集器](d7ba81a7/hotspot_gc.png)
#### Serial收集器
这个收集器是一个单线程工作的收集器，但它的“单线程”的意义并不仅仅是说明它只会使用一个处理器或一条收集线程去完成垃圾收集工作，更重要的是强调在它进行垃圾收集时，必须暂停其他所有工作线程，直到它收集结束。
![Serial/Serial Old收集器运行示意图](d7ba81a7/Serial_SerialOld.png)
**Serial收集器对于运行在客户端模式下的虚拟机来说是一个很好的选择**

#### ParNew收集器
ParNew收集器实质上是Serial收集器的多线程并行版本，除了**同时使用多条线程进行垃圾收集**之外，其余的行为包括Serial收集器完全一致。
![ParNew/Serial Old收集器运行示意图](d7ba81a7/ParNew_SerialOld.png)
不少运行在服务端模式下的HotSpot虚拟机，尤其是JDK 7之前的遗留系统中首选的新生代收集器，其中有一个与功能、性能无关但其实很重要的原因是：*除了Serial收集器外，目前只有它能与CMS收集器配合工作。*
可以理解为：ParNew合并入CMS，成为它专门处理新生代的组成部分。ParNew可以说是HotSpot虚拟机中第一款退出历史舞台的垃圾收集器。

ParNew收集器是激活CMS后（使用-XX：+UseConcMarkSweepGC选项）的默认新生代收集器，也可以使用-XX：+/-UseParNewGC选项来强制指定或者禁用它。

##### 并发与并行
- **并行（Parallel）：** 并行描述的是多条垃圾收集器线程之间的关系，说明同一时间有多条这样的线程在协同工作，通常默认此时用户线程是处于等待状态。
- **并发（Concurrent）：** 并发描述的是垃圾收集器线程与用户线程之间的关系，说明同一时间垃圾收集器线程与用户线程都在运行。由于用户线程并未被冻结，所以程序仍然能响应服务请求

#### Parallel Scavenge收集器
Parallel Scavenge收集器也是一款新生代收集器，它同样是**基于标记-复制算法实现**的收集器，也是能够并行收集的多线程收集器

Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，*CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间*，而Parallel Scavenge收集器的目标则是**达到一个可控制的吞吐量（Throughput）**。所谓**吞吐量**就是*处理器用于运行用户代码的时间与处理器总消耗时间的比值*

Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的-XX：MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX：GCTimeRatio参数。

#### Serial Old收集器
Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。这个收集器的主要意义也是供客户端模式下的HotSpot虚拟机使用。
![Serial Old收集器运行示意图](d7ba81a7/SerialOld.png)

#### Parallel Old收集器
Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。
由于老年代Serial Old收集器在服务端应用性能上的“拖累”，使用Parallel Scavenge收集器也未必能在整体上获得吞吐量最大化的效果。同样，由于单线程的老年代收集中无法充分利用服务器多处理器的并行处理能力，*在老年代内存空间很大而且硬件规格比较高级的运行环境中*，**这种组合的总吞吐量甚至不一定比ParNew加CMS的组合来得优秀。**
![ParallelOld收集器运行示意图](d7ba81a7/ParallelOld.png)

**在注重吞吐量或者处理器资源较为稀缺的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器这个组合。
**

#### CMS收集器
CMS（Concurrent Mark Sweep）收集器是一种以**获取最短回收停顿时间**为目标的收集器。基于标记-清除算法实现。
基本分为四个步骤：
- 1）初始标记（CMS initial mark）**（Stop The World）**：初始标记仅仅只是标记一下GCRoots能直接关联到的对象，速度很快；
- 2）并发标记（CMS concurrent mark）：并发标记阶段就是从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行；
- 3）重新标记（CMS remark）**（Stop The World）**：而重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录（详见3.4.6节中关于增量更新的讲解），这个阶段的停顿时间通常会比初始标记阶段稍长一些，但也远比并发标记阶段的时间短。
- 4）并发清除（CMS concurrent sweep）：清理删除掉标记阶段判断的已经死亡的对象，由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发的。
![Concurrent Mark Sweep收集器运行示意图](d7ba81a7/Concurrent_Mark_Sweep.png)

CMS至少有以下三个明显的缺点：
- CMS收集器对处理器资源非常敏感
- 由于CMS收集器无法处理“浮动垃圾”（Floating Garbage），有可能出现“Con-current Mode Failure”失败进而导致另一次完全“Stop The World”的Full GC的产生。
- CMS是一款基于“标记-清除”算法实现的收集器，就可能想到这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，往往会出现老年代还有很多剩余空间，但就是无法找到足够大的连续空间来分配当前对象，而不得不提前触发一次Full GC的情况。可以适当调高参数**-XX：CMSInitiatingOccupancyFraction**的值来提高CMS的触发百分比，降低内存回收频率，获取更好的性能。
CMS收集器提供了一个-XX：+UseCMSCompactAtFullCollection开关参数（默认是开启的，此参数从**JDK 9开始废弃**），用于在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程，由于这个内存整理必须移动存活对象，（在Shenandoah和ZGC出现前）是无法并发的。这样空间碎片问题是解决了，但停顿时间又会变长，因此虚拟机设计者们还提供了另外一个参数-XX：CMSFullGCsBeforeCompaction（此参数从JDK 9开始废弃），这个参数的作用是要求CM S收集器在执行过若干次（数量由参数值决定）不整理空间的Full GC之后，下一次进入Full GC前会先进行碎片整理（默认值为0，表示每次进入Full GC时都进行碎片整理）。

#### Garbage First收集器
面向堆内存任何部分来组成回收集（Collection Set，一般简称CSet）进行回收，衡量标准不再是它属于哪个分代，而是哪块内存中存放的垃圾数量最多，回收收益最大，这就是G1收集器的Mixed GC模式。

G1不再坚持固定大小以及固定数量的分代区域划分，而是把连续的Java堆划分为**多个大小相等的独立区域（Region）**，每一个Region都可以根据需要，扮演新生代的Eden空间、Survivor空间，或者老年代空间。

Region中还有一类特殊的Humongous区域，专门用来存储大对象。G1认为只要大小超过了一个Region容量一半的对象即可判定为大对象。每个Region的大小可以通过参数-XX：G1HeapRegionSize设定，取值范围为1M B～32M B，且应为2的N次幂。

G1收集器之所以能建立可预测的停顿时间模型：是因为它将Region作为单次回收的最小单元，即每次收集到的内存空间都是Region大小的整数倍，这样可以有计划地避免在整个Java堆中进行全区域的垃圾收集。

#### 低延迟垃圾收集器
衡量垃圾收集器的三项最重要的指标是：内存占用（Footprint）、吞吐量（Throughput）和延迟（Latency），三者共同构成了一个“不可能三角”。
##### Shenandoah收集器
##### ZGC收集器
ZGC和Shenandoah的目标是高度相似的，都希望在尽可能对吞吐量影响不太大的前提下，实现在任意堆内存大小下都可以把垃圾收集的停顿时间限制在十毫秒以内的低延迟。

### 选择合适的垃圾收集器
#### Epsilon收集器
从JDK 10开始，为了隔离垃圾收集器与Java虚拟机解释、编译、监控等子系统的关系，RedHat提出了垃圾收集器的统一接口，即JEP 304提案，Epsilon是这个接口的有效性验证和参考实现，同时也用于需要剥离垃圾收集器影响的性能测试和压力测试。

**背景：**
传统Java有着内存占用较大，在容器中启动时间长，即时编译需要缓慢优化等特点，这对大型应用来说并不是什么太大的问题，但对短时间、小规模的服务形式就有诸多不适。为了应对新的技术潮流，最近几个版本的JDK逐渐加入了提前编译、面向应用的类数据共享等支持。Epsilon也是有着类似的目标，如果读者的应用只要**运行数分钟甚至数秒**，只要Java虚拟机能正确分配内存，在堆耗尽之前就会退出，那显然运行负载极小、没有任何回收行为的Epsilon便是很恰当的选择。

#### 收集器的权衡
我们应该如何选择一款适合自己应用的收集器呢？这个问题的答案主要受以下三个因素影响：
- 应用程序的主要关注点是什么？
    + 如果是数据分析、科学计算类的任务，目标是能尽快算出结果，那吞吐量就是主要关注点；
    +  如果是SLA应用，那停顿时间直接影响服务质量，严重的甚至会导致事务超时，这样延迟就是主要关注点；
    +  而如果是客户端应用或者嵌入式应用，那垃圾收集的内存占用则是不可忽视的。
- 运行应用的基础设施如何？譬如硬件规格，要涉及的系统架构是x86-32/64、SPARC还是ARM /Aarch64；处理器的数量多少，分配内存的大小；选择的操作系统是Linux、Solaris还是Windows等。
- 使用JDK的发行商是什么？版本号是多少？是ZingJDK/Zulu、OracleJDK、Open-JDK、OpenJ9抑或是其他公司的发行版？该JDK对应了《Java虚拟机规范》的哪个版本？

#### 虚拟机及垃圾收集器日志
**阅读分析虚拟机和垃圾收集器的日志是处理Java虚拟机内存问题必备的基础技能**，垃圾收集器日志是一系列人为设定的规则，多少有点随开发者编码时的心情而定，没有任何的“业界标准”可言，换句话说，每个收集器的日志格式都可能不一样。

在JDK 9以前，HotSpot并没有提供统一的日志处理框架，虚拟机各个功能模块的日志开关分布在不同的参数上，**日志级别、循环日志大小、输出格式、重定向等** 设置在不同功能上都要单独解决。

直到JDK 9，这种混乱不堪的局面才终于消失，HotSpot所有功能的日志都收归到了“-Xlog”参数上

几个例子：
- 查看GC基本信息，在JDK 9之前使用-XX：+PrintGC，JDK 9后使用-Xlog：gc
- 查看GC详细信息，在JDK 9之前使用-XX：+PrintGCDetails，在JDK 9之后使用-X-log：gc*
- 查看GC前后的堆、方法区可用容量变化，在JDK 9之前使用-XX：+PrintHeapAtGC，JDK 9之后使用-Xlog：gc+heap=debug
- 查看GC过程中用户线程并发时间以及停顿的时间，在JDK 9之前使用-XX：+Print-GCApplicationConcurrentTime以及-XX：+PrintGCApplicationStoppedTime，JDK 9之后使用-Xlog：safepoint：
- 查看收集器Ergonomics机制（自动设置堆空间各分代区域大小、收集目标等内容，从Parallel收集器开始支持）自动调节的相关信息。在JDK 9之前使用-XX：+PrintAdaptive-SizePolicy，JDK 9之后使用-Xlog：gc+ergo*=trace
- 查看熬过收集后剩余对象的年龄分布信息，在JDK 9前使用-XX：+PrintTenuring-Distribution，JDK 9之后使用-Xlog：gc+age=trace

#### 垃圾收集器参数总结
![垃圾收集相关的常用参数1](d7ba81a7/gc_params1.png)
![垃圾收集相关的常用参数2](d7ba81a7/gc_params2.png)

### 实战：内存分配与回收策略
Java技术体系的自动内存管理，最根本的目标是自动化地解决两个问题：**自动给对象分配内存以及自动回收分配给对象的内存**。
#### 对象优先分配再Eden内存
**大多数情况下，对象再新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC**。
HotSpot虚拟机提供了 ```-XX: +PrintGCDetails``` 这个收集器参数日志，告诉虚拟机在发生垃圾收集行为时打印内存回收日志，并且在进程退出的时候出书当前的内存各个区域分配的情况。
```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
*/
public static void testAllocation() {
byte[] allocation1, allocation2, allocation3, allocation4;
allocation1 = new byte[2 * _1MB];
allocation2 = new byte[2 * _1MB];
allocation3 = new byte[2 * _1MB];
allocation4 = new byte[4 * _1MB]; // 出现一次Minor GC
}
```

#### 大对象直接进入老年代
HotSpot虚拟机提供了 ```-XX:PretenureSizeThreshold``` 参数，指定大于该设置值的对象直接在老年代分配，这样做的目的就是避免在Eden区以及两个Survivor区之间来回复制，产生大量的内存复制操作。
**注意**
-XX：PretenureSizeThreshold参数只对Serial和ParNew两款新生代收集器有效，HotSpot的其他新生代收集器，如Parallel Scavenge并不支持这个参数。如果必须使用此参数进行调优，可考虑ParNew加CMS的收集器组合。

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
* -XX:PretenureSizeThreshold=3145728
*/
public static void testPretenureSizeThreshold() {
byte[] allocation;
allocation = new byte[4 * _1MB]; //直接分配在老年代中
}
```

#### 长期存活的对象将进入老年代
HotSpot虚拟机中多数收集器都采用了**分代收集来管理堆内存**，那内存回收时就必须能决策哪些存活对象应当放在新生代，哪些存活对象放在老年代中。
为做到这点，虚拟机给每个对象定义了一个**对象年龄（Age）计数器**，存储在对象头中（详见第2章）。对象通常在Eden区里诞生，如果经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，该对象会被移动到Survivor空间中，并且将其对象年龄设为1岁。对象在Survivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15），就会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数 ```-XX：MaxTenuringThreshold``` 设置。

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:Survivor-
Ratio=8 -XX:MaxTenuringThreshold=1
* -XX:+PrintTenuringDistribution
*/
@SuppressWarnings("unused")
public static void testTenuringThreshold() {
    byte[] allocation1, allocation2, allocation3;
    allocation1 = new byte[_1MB / 4]; // 什么时候进入老年代决定于XX:MaxTenuringThreshold设置
    allocation2 = new byte[4 * _1MB];
    allocation3 = new byte[4 * _1MB];
    allocation3 = null;
    allocation3 = new byte[4 * _1MB];
}
```


#### 动态对象年龄判定
为了能更好地适应不同程序的内存状况，HotSpot虚拟机**并不是永远**要求对象的年龄必须达到-XX：MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到-XX：MaxTenuringThreshold中要求的年龄。

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
* -XX:+PrintTenuringDistribution
*/
@SuppressWarnings("unused")
public static void testTenuringThreshold2() {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[_1MB / 4]; // allocation1+allocation2大于survivo空间一半
    allocation2 = new byte[_1MB / 4];
    allocation3 = new byte[4 * _1MB];
    allocation4 = new byte[4 * _1MB];
    allocation4 = null;
    allocation4 = new byte[4 * _1MB];
}
```

#### 空间分配担保
在发生Minor GC之前，虚拟机必须先检查老年代最大可用的连续空间**是否大于**新生代所有对象总空间，
- 如果这个条件成立，那这一次Minor GC可以确保是安全的。
- 如果不成立，则虚拟机会先查看-XX：HandlePromotionFailure参数的设置值是否允许担保失败（Handle Promotion Failure）；
    + 如果允许，那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，
    + 如果大于，将尝试进行一次Minor GC，尽管这次Minor GC是有风险的；
    + 如果小于，或者-XX：HandlePromotionFailure设置不允许冒险，那这时就要改为进行一次Full GC。

**注：** 说白了，就是在MinorGC前，需要判定老年代要不要腾出空间来存储Survivor中的内存对象，如果老年代自己都不够，那就来一次Full GC腾出空间来与Survivor空间置换之后，再进行MinorGC。

## 第四章 虚拟机性能监控、故障处理工具
**给一个系统定位问题的时候，知识、经验是关键基础，数据是依据，工具是运用知识处理数据的手段。**
这里说的数据包括但不限于异常堆栈、虚拟机运行日志、垃圾收集器日志、线程快照（threaddump/javacore文件）、堆转储快照（heapdump/hprof文件）等。

### 基础故障处理工具
- 商业授权工具：主要是JMC（Java M ission Control）及它要使用到的JFR（Java Flight Recorder），JMC这个原本来自于JRockit的运维监控套件从JDK 7 Update 40开始就被集成到OracleJDK中，JDK 11之前都无须独立下载，但是在商业环境中使用它则是要付费的。
- 正式支持工具：这一类工具属于被长期支持的工具，不同平台、不同版本的JDK之间，这类工具可能会略有差异，但是不会出现某一个工具突然消失的情况。
- 实验性工具：这一类工具在它们的使用说明中被声明为“没有技术支持，并且是实验性质的”（Unsupported and Experimental）产品，日后可能会转正，也可能会在某个JDK版本中无声无息地消失。但事实上它们通常都非常稳定而且功能强大，也能在处理应用程序性能问题、定位故障时发挥很大的作用。

#### jps: 虚拟机进程状况工具
除了名字像UNIX的ps命令之外，它的功能也和ps命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual M achine Identifier）。

**作用：** 如果同时启动了多个虚拟机进程，无法根据进程名称定位时，那就必须依赖jps命令显示主类的功能才能区分了。
![jps工具主要选项](d7ba81a7/jps_params.png)

#### jstat：虚拟机统计信息监视工具
jstat（JVM Statistics M onitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的**类加载、内存、垃圾收集、即时编译等运行时数据**，在没有GUI图形界面、只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的常用工具。

![jstat工具主要选项](d7ba81a7/jstat_params.png)

#### jinfo：Java配置信息工具
jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了。

#### jmap：Java内存映像工具
jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文件）。如果不使用jmap命令，要想获取Java堆转储快照也还有一些比较“暴力”的手段：譬如在第2章中用过的-XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在内存溢出异常出现之后自动生成堆转储快照文件，通过-XX：+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break]键让虚拟机生成堆转储快照文件，又或者在Linux系统下通过**Kill -3**命令发送进程退出信号“恐吓”一下虚拟机，也能顺利拿到堆转储快照。

![jmap工具主要选项](d7ba81a7/jmap_params.png)

#### jhat：虚拟机堆转储快照分析工具
JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/Web服务器，生成堆转储快照的分析结果后，可以在浏览器中查看。*不过实事求是地说，在实际工作中，除非手上真的没有别的工具可用，否则多数人是不会直接使用jhat命令来分析堆转储快照文件的。*

#### jstack：Java堆栈跟踪工具
jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因。

![jstack工具主要选项](d7ba81a7/jstack_params.png)

### 可视化故障处理工具
注：本小结介绍了几种集中可视化分析内存的方案，各有特色，值得仔细研究，这是理论与实践相结合的最佳实践。

#### JHSDB：基于服务代理的调试工具
JDK中提供了JCMD和JHSDB两个集成式的多功能工具箱，它们不仅整合了前面介绍的所有基础工具所能提供的专项功能，而且由于有着“后发优势”，能够做的往往比之前的老工具们更好、更强大。

**注：** 本小节实践我实在jdk9.0.4版本下完成的，尝试在jdk8下实验，确实如作者所述在sa-jdi.jar包中存在HSDB工具类，但是尝试直接运行了，但是出现了找不到windbg.dll动态链接库的错误。因此，直接更换为jdk9运行。**故而以下文字记录，基于本书介绍的同时，涉及测试的结果，均是个人实验记录所得，可能跟原书中表现有些许差异。**

**介绍**
JHSDB是一款基于服务性代理（Serviceability Agent，SA）实现的进程外调试工具。
服务性代理是HotSpot虚拟机中一组用于映射Java虚拟机运行信息的、主要基于Java语言（含少量JNI代码）实现的API集合。服务性代理以HotSpot内部的数据结构为参照物进行设计，把这些C++的数据抽象出Java模型对象，相当于HotSpot的C++代码的一个镜像。通过服务性代理的API，可以在一个独立的Java虚拟机的进程里分析其他HotSpot虚拟机的内部数据，或者从HotSpot虚拟机进程内存中dump出来的转储快照里还原出它的运行状态细节。

测试代码(跟原书中略有差别，主要体现在变量命名的差异)，这片代码主要用于调试，三个对象分布在内存哪里？
```java
/**
* staticObj、instanceObj、localObj存放在哪里？
*/
public class JHSDBTest {

    static class Test {
        static ObjectHolder staticObj = new ObjectHolder();
        ObjectHolder instanceObj = new ObjectHolder();
        void foo() {
            ObjectHolder localObj = new ObjectHolder();
            System.out.println("done"); // 调试运行的时候，在这里设置一个断点，用于通过jps获取jvm进程id
        }
    }

    private static class ObjectHolder {}

    public static void main(String[] args) {
        Test test = new JHSDBTest.Test();
        test.foo();
    }

}
```


**答案：** 根据前面的理论学习，不难分析出：**staticObj随着Test的类型信息存放在方法区，instanceObj随着Test的对象实例存放在Java堆，localObject则是存放在foo()方法栈帧的局部变量表中。**
但是实际是这样的吗？
由于JHSDB本身对压缩指针的支持存在很多缺陷，建议用64位系统的读者在实验时禁用压缩指针，另外为了后续操作时可以加快在内存中搜索对象的速度，也建议读者限制一下Java堆的大小。本例中，作者建议采用的运行参数如下：
```java
-Xmx10m -XX:+UseSerialGC -XX:-UseCompressedOops
```

程序执行后，通过jps查到测试程序进程ID，从下面片段中得知，15986就是我们测试程序所要的进程id
```cmd
C:\Users\kemi>jps -l
15252 org.jetbrains.idea.maven.server.RemoteMavenServer36
15444
13868 jdk.jcmd/sun.tools.jps.Jps
16172 D:\Program
3628 org.jetbrains.jps.cmdline.Launcher
8588 cc.nimbusk.jhsdb.JHSDBTest
```

紧接着，通过jhsdb命令，来运行可视化界面：
```java
jhsdb hsdb --pid 8588
```

运行之后，界面如下图所示
![JHSDB的界面](d7ba81a7/hsdb_running_window.png)
从运行效果来看其实生成的就是一个Swing的小程序，这点我们从jdk8的sa-jdi.jar包中的目录结构也能看出来
![sa-jdi.jar包目录结构](d7ba81a7/sa-jdi_jar_infrastructure.png)

点击菜单：**Tools->Heap Parameters**，就可以看到堆内存分配布局，由于作者的运行参数中 **指定了使用的是Serial收集器**，图中我们看到了典型的Serial的分代内存布局，Heap Parameters窗口中清楚列出了新生代的Eden、S1、S2和老年代的容量（单位为字节）以及它们的虚拟内存地址起止范围。
![Serial收集器的堆布局](d7ba81a7/jhsdb_heap_parameters.png)
实际如下所示：
```
Heap Parameters:
Gen 0:   eden [0x0000026f4d200000,0x0000026f4d2d57c8,0x0000026f4d4b0000) space capacity = 2818048, 31.029989553052324 used
  from [0x0000026f4d500000,0x0000026f4d550000,0x0000026f4d550000) space capacity = 327680, 100.0 used
  to   [0x0000026f4d4b0000,0x0000026f4d4b0000,0x0000026f4d500000) space capacity = 327680, 0.0 usedInvocations: 1

Gen 1:   old  [0x0000026f4d550000,0x0000026f4d670de0,0x0000026f4dc00000) space capacity = 7012352, 16.873083382009344 usedInvocations: 0

```


其中：**0x0000026f4d200000** 就是起始虚拟内存地址（eden新生代起始），**0x0000026f4d500000** 就是结束虚拟地址（To Survivor结束）

再通过菜单Windows->Console窗口，打开命令行，使用scanoops命令在Java堆的新生代（从Eden起始地址到To Survivor结束地址）范围内查找ObjectHolder的实例
```java
scanoops 0x0000026f4d200000 0x0000026f4d500000 cc.nimbusk.jhsdb.JHSDBTest$ObjectHolder
```

得到结果：
```java

hsdb> scanoops 0x0000026f4d200000 0x0000026f4d500000 cc.nimbusk.jhsdb.JHSDBTest$ObjectHolder
No such type.
hsdb> scanoops 0x0000026f4d200000 0x0000026f4d500000 cc.nimbusk.jhsdb.JHSDBTest$ObjectHolder
0x0000026f4d2cb4b0 cc/nimbusk/jhsdb/JHSDBTest$ObjectHolder
0x0000026f4d2cb4d8 cc/nimbusk/jhsdb/JHSDBTest$ObjectHolder
0x0000026f4d2cb4e8 cc/nimbusk/jhsdb/JHSDBTest$ObjectHolder
Error: sun.jvm.hotspot.debugger.DebuggerException: Windbg Error: ReadVirtual failed!
hsdb> 
```

**注：** 这里跟书中作者有点区别，原书作者实例中，通过jps运行之后，得到的进程ID是没有包路径的，但是我自己实验的时候，发现是有的。这个在后面的scanoops命令使用的时候有用。如果不带包路径的话，在内存中是找不到的，会提示你 **No such type**

我们从结果分析一下，确实在内存中找到三个ObjectHolder对象，我们再看看第一行地址分布：
这三个对象的地址前缀都是：**0x0000026f4d2**，而我们通过前面的Heap Parameters拿到所有分代内存分布发现，只有Eden区域地址（eden [0x0000026f4d200000,0x0000026f4d2d57c8,0x0000026f4d4b0000)），**这也就顺带验证了：一般情况下新对象在Eden中创建的分配规则**。

再使用：**Tools->Inspector** 功能确认一下这三个地址中存放的对象：
![Insepector实例数据](d7ba81a7/jhsdb_tools_inspector.png)

Inspector为我们展示了对象头和指向对象元数据的指针，里面包括了 **Java类型的名字、继承关系、实现接口关系，字段信息、方法信息、运行时常量池的指针、内嵌的虚方法表（vtable）以及接口方法表（itable）等。** 由于我们的确没有在ObjectHolder上定义过任何字段，所以图中并没有看到任何实例字段数据，读者在做实验时不妨定义一些不同数据类型的字段，观察它们在HotSpot虚拟机里面是如何存储的。

**注：** 实际上，目前在实验的时候，发现点不开_metadata._klass: InstanceKlass for cc/nimbusk/jhsdb/JHSDBTest$ObjectHolder，看不到里面具体的内容，后台报了一个数组越界的错误。尝试将JHSDBTest类包路径去除，重新运行发现又报了一个空指针错误。。。嗯，看来这个swing问题不少，暂时没去研究了

接下来要根据堆中对象实例地址找出引用它们的指针，原本JHSDB的Tools菜单中有ComputeReverse Ptrs来完成这个功能，但是Swing本身可能会报空指针异常，这里直接通过命令来实现，revptrs命令后跟上第一个对象的地址。

```java
// 这里内存地址跟前文中看到的不一样，是因为，中途切换环境，我重新运行了。实际实验的时候，同样的方法，按照实际分配的地址即可。这里不再赘述
revptrs 0x00000119d44ca868
```

得到结果：
```java
hsdb> revptrs 0x00000119d44ca868
null
Oop for java/lang/Class @ 0x00000119d44c9488

```

在通过Inspector查找这个地址：0x00000119d44c9488，得到如下图所示：
![方法区实例内存地址](d7ba81a7/jhsdb_class_object.png)
果然找到了一个引用该对象的地方，是在一个 ```java.lang.Class``` 的实例里，并且给出了这个实例的地址，通过Inspector查看该对象实例，可以清楚看到这确实是一个 ```java.lang.Class``` 类型的对象实例，里面有一个名为 **staticObj** 的实例字段。（在JDK 7以前，即还没有开始“去永久代”行动时，这些静态变量是存放在永久代上的，JDK 7起把静态变量、字符常量这些从永久代移除出去。）

从《Java虚拟机规范》所定义的概念模型来看，所有Class相关的信息都应该存放在方法区之中，但方法区该如何实现，《Java虚拟机规范》并未做出规定，这就成了一件允许不同虚拟机自己灵活把握的事情。**JDK 7及其以后版本的HotSpot虚拟机选择把静态变量与类型在Java语言一端的映射Class对象存放在一起，存储于Java堆之中，从我们的实验中也明确验证了这一点。**

接着查找第二个对象实例：
```java
hsdb> revptrs 0x00000119d44ca890
null
Oop for JHSDBTest$Test @ 0x00000119d44ca878
```

这次找到一个类型为JHSDBTest$Test的对象实例，在Inspector中该对象实例显示如图所示：
![方法区实例内存地址](d7ba81a7/jhsdb_instance_object.png)
这个结果完全符合我们的预期，第二个ObjectHolder的指针是在Java堆中JHSDBTest$Test对象的instanceObj字段上。

我们查找第三个对象的时候，发现：
```java
hsdb> revptrs 0x00000119d44ca8a0
null
null
hsdb> 
```

**看来revptrs命令并不支持查找栈上的指针引用，不过没有关系，得益于我们测试代码足够简洁，人工也可以来完成这件事情。** 在Java Thread窗口选中main线程后点击Stack Memory按钮查看该线程的栈内存，如图下图所示：
![stack memory按钮](d7ba81a7/jhsdb_statck_memory.png)
打开栈内存之后：
![main线程的栈内存](d7ba81a7/jhsdb_statck_memory_view.png)

观察一个唯一的栈上的分配的内存，如下图所示：
![栈内存上的分配情况](d7ba81a7/jhsdb_statck_memory_collection.png)
这个线程只有两个方法栈帧，尽管没有查找功能，**但通过肉眼观察在地址0x0000000cb5aff570上的值正好就是0x00000119d44ca8a0**，而且JHSDB在旁边已经自动生成注释，说明这里确实是引用了一个来自新生代的JHSDBTest$ObjectHolder对象。
至此，本次实验中三个对象均已找到，并成功追溯到引用它们的地方，也就实践验证了开篇中提出的这些对象的引用是存储在什么地方的问题。

JHSDB提供了非常强大且灵活的命令和功能，本节的例子只是其中一个很小的应用，读者在实际开发、学习时，可以用它来调试虚拟机进程或者dump出来的内存转储快照，以积累更多的实际经验。
关于其它的相关描述，查看fx大神的描述[rednaxelafx](https://www.iteye.com/blog/rednaxelafx-1847971)

#### JConsole:Java监视与管理控制台
JConsole（Java Monitoring and Management Console）是一款基于JMX（Java Manage-mentExtensions）的可视化监视、管理工具。它的主要功能是通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整。
JMX是一种开放性的技术，不仅可以用在虚拟机本身的管理上，还可以运行于虚拟机之上的软件中，典型的如中间件大多也基于JMX来实现管理与监控。虚拟机对JMX MBean的访问也是完全开放的，可以使用代码调用API、支持JMX协议的管理控制台，或者其他符合JMX规范的软件进行访问。

##### 启动JConsole
![JConsole连接页面](d7ba81a7/jconsole_login.jpg)
通过JDK/bin目录下的jconsole.exe启动JConsole后，会自动搜索出本机运行的所有虚拟机进程，而不需要用户自己使用jps来查询，如图4-10所示。双击选择其中一个进程便可进入主界面开始监控。JMX支持跨服务器的管理，也可以使用下面的“远程进程”功能来连接远程服务器，对远程虚拟机进行监控。如下图所示：
![JConsole主界面](d7ba81a7/jconsole_main_page.jpg)

##### 内存监控
“内存”页签的作用相当于可视化的jstat命令，用于监视被收集器管理的虚拟机内存（被收集器直接管理的Java堆和被间接管理的方法区）的变化趋势。

测试代码：
```java
/**
 * 内存占位符对象，一个OOMObject大约占64KB<br>
 * 这段代码的作用是以64KB/50ms的速度向Java堆中填充数据，一共填充1000次，使用JConsole的“内存”页签进行监视，观察曲线和柱状指示图的变化。
 *
 * -Xms100m -Xmx100m -XX:+UseSerialGC
 *
 * @author nimbus.k 2021-08-03 22:22
 * @version 1.0
 */
public class OOMObjectTest {

    static class OOMObject {
        public byte[] placeholder = new byte[64 * 1024];
    }

    public static void fillHeap(int num) throws InterruptedException {
        List<OOMObject> list = new ArrayList<OOMObject>();
        for (int i = 0; i < num; i++) {
            // 稍作延时，令监视曲线的变化更加明显
            Thread.sleep(50);
            list.add(new OOMObject());
        }
        System.gc();
    }

    public static void main(String[] args) throws Exception {
        fillHeap(1000);
    }

}

```

**解读：**
程序运行后，在“内存”页签中可以看到内存池Eden区的运行趋势呈现折线状，如图4-12所示。监视范围扩大至整个堆后，会发现曲线是一直平滑向上增长的。从柱状图可以看到，在1000次循环执行结束，运行了System.gc()后，虽然整个新生代Eden和Survivor区都基本被清空了，但是代表老年代的柱状图仍然保持峰值状态，说明被填充进堆中的数据在System.gc()方法执行之后仍然存活。

这里作者给我们提出了两个问题：
- 虚拟机启动参数只限制了Java堆为100M B，但没有明确使用-Xmn参数指定新生代大小，读者能否从监控图中估算出新生代的容量？
- 为何执行了System.gc()之后，图4-12中代表老年代的柱状图仍然显示峰值状态，代码需要如何调整才能让System.gc()回收掉填充到堆中的对象？

![Eden区内存变化状况](d7ba81a7/jconsole_memory_monitoring.jpg)

问题1答案：图4-12显示Eden空间为27328KB，因为没有设置-XX：SurvivorRadio参数，所以Eden与Survivor空间比例的默认值为8∶1，因此整个新生代空间大约为27328KB×125%=34160KB。
问题2答案：执行System.gc()之后，空间未能回收是因为List<OOMObject>list对象仍然存活，fillHeap()方法仍然没有退出，因此list对象在System.gc()执行时仍然处于作用域之内 **(准确地说，只有虚拟机使用解释器执行的时候，“在作用域之内”才能保证它不会被回收，因为这里的回收还涉及局部变量表变量槽的复用、即时编译器介入时机等问题，具体读者可参考第8章)** 。如果把System.gc()移动到fillHeap()方法外调用就可以回收掉全部内存。

**注：** 问题一答案容易理解，实际就是这么分配的，如果想要验证可以通过调整SurvivorRadio参数来调整；问题2，我们来通过稍微修改一下上述OOMObjectTest代码片段，来看看是不是如作者所说：
代码片段如下，只将main方法中相关代码做了调整，其余不变
```java
public static void main(String[] args) throws Exception {
        for (int i = 10; i < 200;) {
            fillHeap(1000);
            if (i % 50 == 0) {
                // 每被50整除时，再手工触发一次GC
                System.gc();
            }
            i = i + 10;
        }

    }
```

此时运行监控结过，如下图所示：
![Eden区内存完全回收场景](d7ba81a7/jconsole_memory_monitoring2.jpg)
从图中可以看出，确实完全回收了（留有少部分，毕竟还有其它区域代码的实例需要占用空间）

##### 线程监控
如果说JConsole的“内存”页签相当于可视化的jstat命令的话，那“线程”页签的功能就相当于可视化的jstack命令了，遇到线程停顿的时候可以使用这个页签的功能进行分析。前面讲解jstack命令时提到 **线程长时间停顿的主要原因有等待外部资源（数据库连接、网络资源、设备资源等）、死循环、锁等待等。**

示例代码：
```java
/**
 * 线程死循环演示
 *
 * @author nimbus.k 2021-08-03 22:52
 * @version 1.0
 */
public class ThreadMonitor {

    /**
     * 线程死循环演示
     */
    public static void createBusyThread() {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) // 第41行
                    ;
            }
        }, "testBusyThread");
        thread.start();
    }

    /**
     * 线程锁等待演示
     */
    public static void createLockThread(final Object lock) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (lock) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }, "testLockThread");
        thread.start();
    }
    public static void main(String[] args) throws Exception {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        br.readLine();
        createBusyThread();
        br.readLine();
        Object obj = new Object();
        createLockThread(obj);
    }

}
```

程序运行后，首先在“线程”页签中选择main线程，如图4-13所示。堆栈追踪显示BufferedReader的readBytes()方法正在等待System.in的键盘输入，这时候线程为Runnable状态，Runnable状态的线程仍会被分配运行时间，但readBytes()方法检查到流没有更新就会立刻归还执行令牌给操作系统，这种等待只消耗很小的处理器资源。
![main线程](d7ba81a7/jconsole_thread_main.jpg)
接着监控testBusyThread线程，又如下图所示。testBusyThread线程一直在执行空循环，从堆栈追踪中看到一直在MonitoringTest.java代码的41行停留，41行的代码为while(true)。这时候线程为Runnable状态，而且没有归还线程执行令牌的动作，所以会在空循环耗尽操作系统分配给它的执行时间，直到线程切换为止，这种等待会消耗大量的处理器资源。
![testBusyThread线程](d7ba81a7/jconsole_thread_testBusyThread.jpg)
testLockThread线程在等待lock对象的notify()或notifyAll()方法的出现，**线程这时候处于WAITING状态，在重新唤醒前不会被分配执行时间。** 如下图所示：
![testLockThread线程](d7ba81a7/jconsole_memory_testLockThread.jpg)
testLockThread线程正处于正常的活锁等待中，只要lock对象的notify()或notifyAll()方法被调用，这个线程便能激活继续执行

###### 死锁代码样例
代码示例：
```java
/**
 * 一则死锁代码样例
 *
 * @author nimbus.k 2021-08-03 23:09
 * @version 1.0
 */
public class DeadLockTest {

    /**
     * 线程死锁等待演示
     */
    static class SynAddRunalbe implements Runnable {
        int a, b;
        public SynAddRunalbe(int a, int b) {
            this.a = a;
            this.b = b;
        }
        @Override
        public void run() {
            synchronized (Integer.valueOf(a)) {
                synchronized (Integer.valueOf(b)) {
                    System.out.println(a + b);
                }
            }
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new Thread(new SynAddRunalbe(1, 2)).start();
            new Thread(new SynAddRunalbe(2, 1)).start();
        }
    }

}
```

**代码释意**
这段代码开了200个线程去分别计算1+2以及2+1的值，理论上for循环都是可省略的，两个线程也可能会导致死锁，不过那样概率太小，需要尝试运行很多次才能看到死锁的效果。如果运气不是特别差的话，上面带for循环的版本最多运行两三次就会遇到线程死锁，程序无法结束。
**根本原因**
造成死锁的根本原
因是Integer.valueOf()方法出于减少对象创建次数和节省内存的考虑，会对数值为-128～127之间的Integer对象进行缓存（这是《Java虚拟机规范》中明确要求缓存的默认值，实际值可以调整，具体取决于java.lang.Integer.Integer-Cache.high参数的设置。），如果valueOf()方法传入的参数在这个范围之内，就直接返回缓存中的对象。也就是说代码中尽管调用了200次Integer.valueOf()方法，但一共只返回了两个不同的Integer对象。*假如某个线程的两个synchronized块之间发生了一次线程切换，那就会出现线程A在等待被线程B持有的Integer.valueOf(1)，线程B又在等待被线程A持有的Integer.valueOf(2)，结果大家都跑不下去的情况。*

![testLockThread线程](d7ba81a7/jconsole_deadlock.jpg)
**很清晰地显示，线程Thread-3在等待一个被线程Thread-4持有的Integer对象，而点击线程Thread-4则显示它也在等待一个被线程Thread-3持有的Integer对象，这样两个线程就互相卡住，除非牺牲其中一个，否则死锁无法释放。**


#### VisualVM：多合-故障处理工具
VisualVM（All-in-One Java Troubleshooting Tool）是功能最强大的运行监视和故障处理程序之一，曾经在很长一段时间内是Oracle官方主力发展的虚拟机故障处理工具。Oracle曾在VisualVM的软件说明中写上了“All-in-One”的字样，预示着它除了 **常规的运行监视、故障处理外，还将提供其他方面的能力，譬如性能分析（Profiling）。**
相比其它收费第三方工具，VisualVM还有一个很大的优点：**不需要被监视的程序基于特殊Agent去运行，因此它的通用性很强，对应用程序实际性能的影响也较小，使得它可以直接应用在生产环境中。**

##### VisualVM兼容范围与插件安装
VisualVM基于NetBeans平台开发工具，所以一开始它就具备了通过插件扩展功能的能力，有了插件扩展支持，VisualVM可以做到：
- 显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。
- 监视应用程序的处理器、垃圾收集、堆、方法区以及线程的信息（jstat、jstack）。
- dump以及分析堆转储快照（jmap、jhat）。
- 方法级的程序运行性能分析，找出被调用最多、运行时间最长的方法。
- 离线程序快照：收集程序的运行时配置、线程dump、内存dump等信息建立一个快照，可以将快照发送开发者处进行Bug反馈。
- 其他插件带来的无限可能性。

**用笔者的话来说**
首次启动VisualVM后，读者先不必着急找应用程序进行监测，初始状态下的VisualVM并没有加载任何插件，虽然基本的监视、线程面板的功能主程序都以默认插件的形式提供，但是如果不在VisualVM上装任何扩展插件，就相当于放弃它最精华的功能，和没有安装任何应用软件的操作系统差不多。

独立安装的插件存储在VisualVM的根目录，譬如JDK 9之前自带的VisulalVM，插件安装后是放在JDK_HOME/lib/visualvm中的。
JDK 9之后VisulalVM形成了一个独立项目：[VisualVM主页](https://visualvm.github.io/?Java_VisualVM)

插件安装非特殊场景，无需手动安装，在有网络连接的环境下，点击“工具->插件菜单”，在页签的“可用插件”及“已安装”中列举了当前版本VisualVM可以使用的全部插件，选中插件后在右边窗口会显示这个插件的基本信息，如开发者、版本、功能描述等。
![testLockThread线程](d7ba81a7/visualvm_plugins_main_page.jpg)

##### 生成、浏览堆转储快照
在VisualVM中生成堆转储快照文件有两种方式，可以执行下列任一操作：
- 在“应用程序”窗口中右键单击应用程序节点，然后选择“堆Dump”。
- 在“应用程序”窗口中双击应用程序节点以打开应用程序标签，然后在“监视”标签中单击“堆Dump”。

生成堆转储快照文件之后，应用程序页签会在该堆的应用程序下增加一个以[heap-dump]开头的子节点，并且在主页签中打开该转储快照，如图4-20所示。如果需要把堆转储快照保存或发送出去，就应在heapdump节点上右键选择“另存为”菜单，否则当VisualVM关闭时，生成的堆转储快照文件会被当作临时文件自动清理掉。要打开一个由已经存在的堆转储快照文件，通过文件菜单中的“装入”功能，选择硬盘上的文件即可。
![浏览dump文件](d7ba81a7/visualvm_heapdump.jpg)

堆页签中的“摘要”面板可以看到应用程序dump时的运行时参数、System.getPro-perties()的内容、线程堆栈等信息；“类”面板则是以类为统计口径统计类的实例数量、容量信息；“实例”面板不能直接使用，因为VisualVM在此时还无法确定用户想查看哪个类的实例，所以需要通过“类”面板进入，在“类”中选择一个需要查看的类，然后双击即可在“实例”里面看到此类的其中500个实例的具体属性信息；“OQL控制台”面板则是运行OQL查询语句的，同jhat中介绍的OQL功能一样。如果读者想要了解具体OQL的语法和使用方法，可参见本书附录D的内容。

# 实战相关