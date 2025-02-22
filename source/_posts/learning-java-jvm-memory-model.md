---
title: JVM内存模型概要
abbrlink: 6377921b
date: 2024-10-14 23:29:12
tags:
    - JVM内存模型
updated: 2024-12-28 20:30:21
categories: Java系列
---


JVM 从软件层屏蔽不同操作系统在底层硬件与指令的区别
<!-- more -->

## 内存模型
**注**：这个图，看看即可，里面有很多箭头，不结合视频课程看，是根本不知道怎么回事的。【待补充完善】
![内存模型示意图](6377921b/内存模型示意图.png)

STW：暂停用户线程，使GC效率变高 


### 程序计数器
一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。**通过该指示器，来完成诸如：分支、循环、跳转、异常处理、线程恢复等基础功能**。

### Java虚拟机栈
生命周期与线程相同。虚拟机栈描述的是Java方法执行的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个**栈帧（stack frame）** 用于存储：**局部变量表、操作数栈、动态链接、方法出口等信息**。
每个方法调用直至执行完毕的过程，就对应一个栈帧在虚拟机栈中从入栈到出栈（FIFO）的过程

**涉及的异常**：StackOverflow、OutOfMemoryError（HotSpot虚拟机中不存在，旧的Classic虚拟机存在，所以了解即可）

#### 局部变量表
**存放数据类型**：存放了编译期可知的各种Java虚拟机基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用类型（reference类型，引用指针）和returnAddress类型（一条字节码指令地址）
**数据类型存放标准**：这些数据类型，在局部变量表中的存储空间已局部变量槽（Slot）来表示，其中64位长度的long和double会占用两个变量槽，其余的数据类型只占用一个。
**内存分配过程**：局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量是完全确定的。

### 本地方法栈
与虚拟机栈作用类似，只不过这里存放的是虚拟机使用的本地（Native）方法服务

### Java堆
Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此**内存区域的唯一目的就是存放对象实例，Java世界里“几乎”所有的对象实例都在这里分配内存**。（*书中提到的*：由于即时编译技术的进步，尤其是逃逸分析技术的日渐强大，栈上分配、标量替换优化手段，这些已经导致了一些微妙的变化，导致Java对象实例都分配在堆上已经不是绝对的了）
无论从什么角度，无论如何划分，都不会改变Java堆中存储内容的共性，无论是哪个区域，存储都只能是对象的实例，**将Java堆细分的目的**：**只是为了更好的回收内存，或者更快地分配内存。**
**涉及异常：OutOfMemoryError**

### 方法区
**用于存储已经被虚拟机加载的：类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。**
**历史沿袭：** JDK7之前，这部分区域还被称之为永久代，在JDK7的时候已经把字符串常量池和静态变量移出。到了JDK8，完全废弃了永久代概念，改用元空间（Metasapce）来实现。
**涉及异常**：OutOfMemoryError

#### 运行时常量池
运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池表（Constant Pool Table），用于存放编译器生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。


### 直接内存
直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是《java虚拟机规范》中定义的内存区域，但是这部分内存也频繁被使用，而且也有可能导致OutOfMemoryError异常。
**注：这部分跟NIO直接相关**
在JDK1.4中引入的NIO（New Input/Ouput）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用来进行操作。这样能在一部分场景中显出提高性能，因为避免了在Java堆和Native堆中来回复制数据。


## 对象创建流程
整体流程
![对象创建整体流程](6377921b/对象创建整体流程.png)

大体上可以分为5个步骤：
1. 类加载检查：
2. 分配内存
3. 初始化
4. 设置对象头
5. 执行init方法

### 1. 类加载检查
当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。（**注：这个过程比较复杂**）
参考另外一篇文章：[虚拟机加载机制](https://nimbusk.cc/post/7410cfeb)
### 2. 分配内存
在类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需内存的大小在类 加载完成后便可完全确定，为对象分配空间的任务等同于把 一块确定大小的内存从Java堆中划分出来。
#### 划分内存的方法
+ “指针碰撞”（Bump the Pointer）(默认用指针碰撞)：如果Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离。
+ “空闲列表”（Free List）：如果Java堆中的内存并不是规整的，已使用的内存和空 闲的内存相互交错，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记 录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录。
#### 解决内存分配并发问题的方法
+ CAS（compare and swap）：虚拟机采用CAS配上失败重试的方式保证更新操作的原子性来对分配内存空间的动作进行同步处理。
+ 本地线程分配缓冲（Thread Local Allocation Buffer,TLAB）：把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存。通过XX:+/UseTLAB参数来设定虚拟机是否使用TLAB(JVM会默认开启XX:+UseTLAB)，XX:TLABSize 指定TLAB大小。

### 3. 初始化
内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头）， 如果使用TLAB，这一工作过程也可以提前至TLAB分配时进行。这一步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

### 4. 设置对象头
初始化零值之后，虚拟机要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头Object Header之中。
在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域：**对象头（Header）、 实例数据（Instance Data）和对齐填充（Padding）。**
![对象头整体示意图](6377921b/对象头整体示意图.png)

HotSpot虚拟机的对象头包括两部分信息，第一部分用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时 间戳等。对象头的另外一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
**数组对象与普通对象的内存结构区别在于，数组对象头里面多了一个数组长度**
![数组类型对象头](6377921b/数组类型对象头.png)
32位对象头示意图：
![32位对象头示意图](6377921b/32位对象头示意图.png)
64位对象头示意图：
![64位对象头示意图](6377921b/64位对象头示意图.png)

### 5. 执行init方法
执行init方法，即对象按照程序员的意愿进行初始化。对应到语言层面上讲，就是为属性赋值（注意，这与上面的赋零值不同，这是由程序员赋的值），和执行构造方法。
