---
title: 深入学习Java并发
abbrlink: '181e5700'
date: 2024-10-25 16:35:56
tags:
    - 并发编程
    - 并发原理
updated: 2024-10-28 20:33:29
categories: Java系列
top: true
---

## 理论相关
### 并发三大特性
并发编程BUG的源头，都归纳为三个问题：**可见性、原子性与有序性**问题

<!-- more -->
### JMM相关
Java虚拟机规范中定义了Java内存模型（Java Memory Model，JMM），用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的并发效果。
JMM规范了Java虚拟机与计算机内存是如何协同工作的：**规定了一个线程如何和何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步的访问共享变量。**
JMM描述的是一种抽象的概念，一组规则，通过这组规则控制程序中各个变量在共享数据区域和私有数据区域的访问方式。
**JMM是围绕原子性、有序性、可见性展开的。**

![JMM内存模型示意图](181e5700/JMM内存模型示意图.png)
#### 关联两个疑问
1. 本地内存中的变量数据什么时候刷新到主内存中？答：不会立马刷新，而是有一定时间。
2. 本地内存中的变量数据什么时候会失效？答：有缓存淘汰时间，淘汰之后会立马read主内存中新的值。
例子中，使用Thread.yield()也可以保证可见，原因：这个方法释放了当前线程的CPU时间片，即存在上下文切换过程，再次获得时间片的时候，会加载上下文，因此会重新读主内存中的值。

### 可见性问题
**当一个线程修改了共享变量的值，其他线程能够看到修改的值。**
Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方法来实现可见性的。
如何保证内存可见性：
+ 通过 volatile 关键字保证可见性。
+ 通过 内存屏障保证可见性。
	+ UnsafeFactory.getUnsafe().storeFence()，与volatile底层是同一个实现，所以可以。
+ 通过 synchronized 关键字保证可见性。能够的原因：底层还是靠storeFence内存屏障实现的。
	+ System.out.println(); // 这个里面是有syn同步块的
+ 通过 Lock保证可见性。
	+ LockSupport.unpark(Thread.currentThread()); // 底层还是内存屏障
+ 通过 final 关键字保证可见性
	+ 比如包装器类，例如Integer，比较特殊，这个也可以。

#### Java中如何保障
从底层本质归纳起来两种，一种是在JVM层面调用storeFence()内存屏障；另一种就是实现上下文切换。
volatile关键字可以保证可见性原因：
- JVM内存屏障，storeLoad来实现；底层汇编：lock前缀; addl

### 缓存一致性协议-MESI
通过缓存中数据（Cache Line）施加4个状态，来达到缓存一致性目的：M-修改；E-独占；S-共享；I-失效。
当失效的时候，高速缓存会立即加载主内存；
注：TODO  可以搜一下这个实现的状态机
![MESI协议状态流转问题](181e5700/MESI协议状态流转问题.png)

**存在两个问题：**
1. 如果存在跨缓存行的时候，一致性协议有问题
2. 早期处理器是没有实现缓存一致性协议（*不同处理器同时向总线发起总线事务， 通过总线仲裁实现，代价非常大，了解*）

### 关于伪共享
伪共享的本质原因：因为缓存行（Cache Line），linux下默认64字节。当程序的不同变量，在同一个缓存行的时候，不同线程处理对应变量的时候，会造成相互干扰（参考MESI），导致频繁的失效要重新读取，性能严重下降。
![伪共享内存交互示意图](181e5700/伪共享内存交互示意图.png)
上图对应的程序代码片段
```java
class Pointer {
	volatie long x;
	volatie long y;
}
// 起两个线程，分别对x、y进行累加动作
```
#### 避免伪共享
1. **手动填充缓存行**：通过在变量之间插入填充变量（padding），可以将不同的变量分配到不同的缓存行中，从而避免伪共享问题。例如，在多线程环境下访问两个变量a和b，如果它们位于同一个缓存行中，可以在它们之间插入一个long类型的变量c，从而让a和b分别被存储到不同的缓存行中，避免了缓存行的竞争。

2. 使用JDK8提供的```@sun.misc.Contended```注解：在JDK8中，@sun.misc.Contended注解可以用来避免伪共享问题。这个注解只能用于类和属性，并且需要手动启用JVM的-XX:-RestrictContended参数。使用该注解后，JVM会自动为相关变量添加填充字节，确保它们不会位于同一个缓存行中。

3. **使用OpenMP的归约子句（reduction）**：在并行编程中使用OpenMP的归约子句（reduction），可以将多个线程的变量合并到一个共享变量中，从而减少缓存行的冲突。这种方法不同于数据填充进行边界对齐的方式，代码中不再将结果声明为数组而是声明为普通变量，使用reduction子句会使得每个线程都有一个变量，通过指定的运算符进行归约计算。
**注：这个只作为了解即可**，有这么个设计泛型，目前支持OpenMP的只有：C、C++和Fortran。有兴趣的可以去看看该模型的specification，[地址](https://www.openmp.org/wp-content/uploads/OpenMP-API-Specification-5-2.pdf)。在Java中，有前面两个功能即可。

## Java线程模型相关
### 有几个主题问题
1. CAS涉及到用户模式到内核模式的切换吗？
	1. CAS不会涉及用户模式到内核模式切换，因为CAS操作最终直接执行的是CPU指令，不涉及切换
2. 为什么说创建Java线程的本质上只有一种？Java线程和go语言的协程有什么区别？
3. 如何优雅的终止线程？
4. Java线程之间如何通信的，有那些方式？

### 线程和进程的区别
- **进程**：操作系统会以进程为单位，分配系统资源（CPU时间片、内存等资源），进程是资源分配的最小单位
- **线程**：线程，有时被称为轻量级进程(Lightweight Process，LWP），是操作系统调度（CPU调度）执行的最小单位
**两者具体的区别：**
- 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集
- 进程拥有共享的资源，如内存空间等，供其内部的线程共享
- 进程间通信较为复杂
	- 同一台计算机的进程通信称为 IPC（Inter-processcommunication）
	- 不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP
- 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量
- 线程更轻量，线程上下文切换成本一般上要比进程上下文切换低

### 进程之间的通信方式
1. **管道（pipe）及有名管道（named pipe）**：管道可用于具有亲缘关系的父子进程间的通信，有名管道除了具有管道所具有的功能外，它还允许无亲缘关系进程间的通信。
2. **信号（signal）**：信号是在软件层次上对中断机制的一种模拟，它是比较复杂的通信方式，用于通知进程有某事件发生，一个进程收到一个信号与处理器收到一个中断请求效果上可以说是一致的。
3. **消息队列（message queue）**：消息队列是消息的链接表，它克服了上两种通信方式中信号量有限的缺点，具有写权限得进程可以按照一定得规则向消息队列中添加新信息；对消息队列有读权限得进程则可以从消息队列中读取信息。
4. **共享内存（shared memory）**：可以说这是最有用的进程间通信方式。它使得多个进程可以访问同一块内存空间，不同进程可以及时看到对方进程中对共享内存中数据得更新。这种方式需要依靠某种同步操作，如互斥锁和信号量等。
5. **信号量（semaphore）**：主要作为进程之间及同一种进程的不同线程之间得同步和互斥手段。
6. **套接字（socket）**：这是一种更为一般得进程间通信机制，它可用于网络中不同机器之间的进程间通信，应用非常广泛。

### 线程的同步互斥
### 上下文切换
- 上下文切换只能在内核模式下发生。
- 上下文切换时多任务操作系统的一个基本特性
- 上下文切换通常是计算密集型的

### 内核模式与用户模式
#### Kernel Mode
在内核模式下，执行代码可以完全且不受限制地访问底层限制。它可以执行任何CPU指令和引用任何内存地址。内核模式通常为操
#### User Mode

#### 两者切换的场景
![用户态和内核态切换的场景](181e5700/用户态和内核态切换的场景.png)
- 系统调用：调用系统底层API
- 异常事件：当发生某些预先不可知的异常，就会切换到内核态，以执行相关的异常事件。
- 设备中断：在使用外围设备时，如外围设备收到用户请求，就会向CPU发送一个中断信号，此时，CPU就会暂停执行原本的下一条指令，转去处理中断事件。此时，如果原来在用户态，自然就会切换到内核态。

### 线程生命周期(操作系统层面)
一般的认为有5种：
![多数的5种状态图](181e5700/多数的5种状态图.png)
在Java层面划分了6种：**NEW, RUNNABLE, BLOCKED, WAITTING, TIMED_WATTING, TERMINATED**，相互交互的示意图如下：
![Java层面的6种](181e5700/Java层面的6种.png)

### 跟Java线程相关
#### 线程的创建方式
1. 直接new Thread类
2. 实现Runnable接口配合Thread
3. 使用有返回值的Callable
4. 使用Lambda表达式
**上述本质上，只有一种创建方式：new Thread(Runnable()).start()方式启动**
关于Thread类中start方法分析：
**注：只有通过start方法**，才能真正调用（通过JNI方式调用）操作系统底层创建线程，大体流程是：
	1. 使用new Thread()创建一个线程，然后调用start()方法进行java层面的线程启动
	2. 调用本地方法start0()，去调用JVM中的JVM_StartThread方法进行线程创建和启动
	3. 调用new JavaThread(&thread_entry, sz)进行线程的创建，并根据不同的操作系统平台调用对应的os::create_thread方法进行线程的创建
	4. 新创建的线程状态为Initialized，调用sync->wait()的方法进行等待，等待被唤醒才会继续执行thread->run()
	5. 调用Thread::start(native_thread)方法进行线程启动，此时将线程状态设置为RSUNNABLE，接着调用os::start_thread(thread)，根据不同的操作系统选择不同的线程启动方式
	6. 线程启动之后状态设置为RUNNABLE，并唤醒第4步中等待的线程，接着执行thread->run()的方法
	7. JavaThread::run()方法会回调第1步new Thread中复写的run方法

#### 协程的概念
协程，英文Coroutines, 是一种**基于线程之上，但又比线程更加轻量级的存在**，协程不是被操作系统内核所管理，而完全是由程序所控制（也就是在用户态执行），具有对内核来说不可见的特性。这样带来的好处就是性能得到了很大的提升，不会像线程切换那样消耗资源。

#### Java线程的调度机制
Java线程调度是抢占式调度的

#### 如何优雅的停止线程
为什么不要用Thread.stop方法，**因为stop方法会释放线程锁，导致并发问题**

##### 优雅的方式：利用线程的中断机制
Java没有提供一种安全、直接的方法来停止某个线程，而是提供了中断机制。
**中断机制是一种协作机制，也就是说通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理。被中断的线程拥有完全的自主权，它既可以选择立即停止，也可以选择一段时间后停止，也可以选择压根不停止。**

Java中提供了几个API来实现：
- interrupt()： 将线程的中断标志位设置为true，不会停止线程
- isInterrupted(): 判断当前线程的中断标志位是否为true，不会清除中断标志位
- Thread.interrupted()：判断当前线程的中断标志位是否为true，并清除中断标志位，重置为fasle
特别要注意，如果使用中断了，一定要注意中断标志位是否被清除，比如在调用sleep（sleep会清除中断标志）期间，一定要还原中断标志
#### 线程之间的通信
volatile
#### 等待唤醒机制
wait/notify ：这种方式需要依赖synchronized加锁才行，另外notify唤醒，没法指定某个线程唤醒
**park/unpark：这个方式就可以直接指定唤醒具体哪个线程**

#### 管道的输入输出流
Thread.join()


## 线程安全相关(原子与有序问题)
### CAS原子性
#### 什么是CAS
通常指的是这样一种原子操作：针对一个变量，首先比较它的内存值与某个期望值是否相同，如果相同，就给它赋一个新值。
**CAS 可以看作是它们合并后的整体——一个不可分割的原子操作，并且其原子性是直接在硬件层面得到保障的。**
伪代码如下：
```
if (value == expectedValue) {
	value = newValue;
}
```
**CAS操作天然能够保持内存可见性，硬件底层指令**
#### CAS应用
在 Java 中，CAS 操作是由 Unsafe 类提供支持的，该类定义了三种针对不同类型变量的 CAS 操作，
```java
// sun.misc.Unsafe
    public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```
#### CAS缺陷
- **自旋CAS长时间不成功，则会给CPU带来非常大的开销**
- **只能保证一个共享变量原子操作**
- **ABA问题：通过额外增加版本号逻辑来避免ABA问题**

#### LongAdder/DoubleAdder
解决AtomicLong等在高并发下**自旋造成的性能影响**
假设一个场景，4个线程，并发采用CAS处理一个变量，LongAdder处理流程如下图所示：
![LongAdder竞争示意图](181e5700/LongAdder竞争示意图.png)

其中：**LongAdder内部有一个base变量，一个Cell[]数组：**
- base变量：非竞态条件下，直接累加到该变量上
- Cell[]数组：竞态条件下，累加个各个线程自己的槽Cell[i]中

### synchronized
#### 临界区(Critical Section)
- 一个程序运行多个线程本身是没有问题的
- 问题出在多个线程访问共享资源
	- 多个线程读共享资源其实也没有问题
	- **在多个线程对共享资源读写操作时发生指令交错，就会出现问题**
一段代码块内如果存在对共享资源的多线程读写操作，称**这段代码块为临界区**，其共享资源为临界资源。

#### 竞态条件(Race Condition)
多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件
为了避免临界区的竞态条件发生，有多种手段可以达到目的：
- **阻塞式的解决方案：synchronized，Lock**
- **非阻塞式的解决方案：原子变量(如CAS)**
#### 底层原理
synchronized是JVM内置锁，基于Monitor机制实现，依赖底层操作系统的互斥原语Mutex（互斥量），它是一个重量级锁，性能较低。
当然，JVM内置锁在1.5之后版本做了重大的优化，如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（LightweightLocking）、偏向锁（Biased Locking）、自适应自旋（Adaptive Spinning）等技术来减少锁操作的开销，**内置锁的并发性能已经基本与Lock持平**。

Java虚拟机通过一个同步结构支持方法和方法中的**指令序列的同步：monitor**。
**同步方法**是通过方法中的access_flags中**设置ACC_SYNCHRONIZED标志**来实现；
**同步代码块**是通过**monitorenter和monitorexit来实现**。
**两个指令的执行是JVM通过调用操作系统的互斥原语mutex来实现**，被阻塞的线程会被挂起、等待重新调度，会导致 **“用户态和内核态”两个态之间来回切换，对性能有较大影响**。

**synchronized是天然的一个可重入锁。**

##### Monitor(管程/监视器)
Monitor，直译为“监视器”，而操作系统领域一般翻译为“管程”。
**管程是指管理共享变量以及对共享变量操作的过程，让它们支持并发。**
在Java 1.5之前，Java语言提供的唯一并发语言就是管程，Java 1.5之后提供的SDK并发包也是以管程为基础的。除了Java之外，C/C++、C#等高级语言也都是支持管程的。
**synchronized关键字和wait()、notify()、notifyAll()这三个方法是Java中实现管程技术的组成部分。**

##### MESA模型(Java核心锁实现原理)
在管程的发展史上，先后出现过三种不同的管程模型，分别是Hasen模型、Hoare模型和MESA模型。现在正在广泛使用的是MESA模型
![MEAS管程模型](181e5700/MEAS管程模型.png)
##### 精简MESA模型
Java 参考了 MESA 模型，语言内置的管程（synchronized）对 MESA 模型进行了**精简**。
MESA模型中，条件变量可以有多个，Java 语言内置的管程里只有一个条件变量。模型如下图所示：
![精简MESA模型](181e5700/精简MESA模型.png)
##### Monitor机制在Java中的实现
了解数据结构中有三个结构：
```c++
// ObjectMonitor::ObjectMonitor
ObjectMonitor() {
    _header       = NULL; //对象头 markOop
    _count        = 0;
    _waiters      = 0,
    _recursions   = 0; // 锁的重入次数
    _object       = NULL; //存储锁对象
    _owner        = NULL; // 标识拥有该monitor的线程（当前获取锁的线程）
    _WaitSet      = NULL; // 等待线程（调用wait）组成的双向循环链表，_WaitSet是第一个节点
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; //多线程竞争锁会先存到这个单向链表中 （FILO栈结构）
    FreeNext      = NULL ;
    _EntryList    = NULL ; //存放在进入或重新进入时被阻塞(blocked)的线程 (也是存竞争锁失败的线程)
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```
分别注意： **WaitSet、cxq、EntryList这三个结构**
这三个结构协作来对锁场景使用情况如下图流程所示，解释竞争顺序：
![Java中的Monitor机制执行流程](181e5700/Java中的Monitor机制执行流程.png)
解释起来：
在获取锁时，是将当前线程插入到cxq的头部，而释放锁时，默认策略（QMode=0）是：如果EntryList为空，则将cxq中的元素按原有顺序插入到EntryList，并唤醒第一个线程，也就是当EntryList为空时，是后来的线程先获取锁；**EntryList不为空，直接从_EntryList中唤醒线程。**
**synchronized的锁竞争顺序概括就是：**在竞争相同资源情况下，如果有wait线程，则该线程会被优先唤醒；反之，就是按照先进后出（FILO）的规则来。

##### 锁在对象内存布局
Hotspot虚拟机中，对象在内存中存储的布局可以分为三块区域：**对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）**。
- **对象头**：比如 hash码，对象所属的年代，对象锁，锁状态标志，偏向锁（线程）ID，偏向时间，数组长度（数组对象才有）等。
- **实例数据**：存放类的属性数据信息，包括父类的属性信息；
- **对齐填充**：由于虚拟机要求 对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。

概括对象在内存中占用字节数（64位，默认开启压缩指针）：
- 对象头：8字节
- 元数据指针：4字节
- 数组对象（如果有）：4字节
- 实例数据：按具体数据类型来分配
- 对齐填充位：按最后8的整数倍来，缺多少补多少。
![对象头内存占用分布示意图](181e5700/对象头内存占用分布.png)

##### 锁对象状态转换
![锁对象状态转换示意图](181e5700/锁对象状态转换示意图.png)
##### 锁的优化
- 偏向锁：某些场景不存在竞争，此时锁偏向某个线程，后续该线程进入同步块的时候，则不会产生加锁/释放锁的性能开销。例如StringBuffer。
- 轻量级锁：线程之间存在轻微的竞争，线程交替执行，临界区简单不复杂，可以快速执行，通过CAS获取锁，失败膨胀（升级）为重量级锁。轻量级锁没有自旋场景。
- 重量级锁：多线程竞争激烈的场景，膨胀期间会通过创建一个Monitor对象来处理竞争条件。
**这三个锁切换示意图，通过图例中的标注，可以看出各种锁是如何切换的**：
![优化锁切换示意图](181e5700/优化锁切换示意图.png)

这里面有几个点：
1. **偏向锁到无锁之间的偏向锁撤销**：例如调用hashcode方法就会产生，原因是：偏向锁对象头中无法存hashcode，而无锁可以。再次看看对象头的分布图：
**注意看无锁和偏向锁之间的差异**
![对象头内存分布](181e5700/对象头内存分布.png)
2. **为什么轻量级锁可以解锁成无锁状态**：原因是，轻量级锁在创建栈中的记录时，会将无锁的markword拷贝到栈数据中，因此解锁的时候之间还原即可。
3. 
###### 一段锁偏移的代码
```java
@Slf4j
public class LockEscalationDemo {

    public static void main(String[] args) throws InterruptedException {
        log.info(ClassLayout.parseInstance(new Object()).toPrintable());
        //HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式
        Thread.sleep(5000);
        Object obj = new Object();
        // 思考： 如果对象调用了hashCode,还会开启偏向锁模式吗
        //obj.hashCode();
        log.info(ClassLayout.parseInstance(obj).toPrintable());

        new Thread(new Runnable() {
            @Override
            public void run() {
                log.info(Thread.currentThread().getName() + "开始执行。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj) {
                    // 思考：偏向锁执行过程中，调用hashcode会发生什么？
                    //obj.hashCode();
//                    //obj.notify();
////                    try {
////                        obj.wait(50);
////                    } catch (InterruptedException e) {
////                        e.printStackTrace();
////                    }

                    log.info(Thread.currentThread().getName() + "获取锁执行中。。。\n"
                            + ClassLayout.parseInstance(obj).toPrintable());
                }
                log.info(Thread.currentThread().getName() + "释放锁。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
            }
        }, "thread1").start();

        //控制线程竞争时机
        Thread.sleep(1);

        new Thread(new Runnable() {
            @Override
            public void run() {
                log.info(Thread.currentThread().getName() + "开始执行。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj) {

                    log.info(Thread.currentThread().getName() + "获取锁执行中。。。\n"
                            + ClassLayout.parseInstance(obj).toPrintable());
                }
                log.info(Thread.currentThread().getName() + "释放锁。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
            }
        }, "thread2").start();


        new Thread(new Runnable() {
            @Override
            public void run() {
                log.info(Thread.currentThread().getName() + "开始执行。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj) {

                    log.info(Thread.currentThread().getName() + "获取锁执行中。。。\n"
                            + ClassLayout.parseInstance(obj).toPrintable());
                }
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info(Thread.currentThread().getName() + "释放锁。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
            }
        }, "thread3").start();

        Thread.sleep(5000);
        log.info(ClassLayout.parseInstance(obj).toPrintable());

    }
}
```
##### synchronized锁的高级优化
###### 偏向锁批量重偏向&批量撤销
从偏向锁的加锁解锁过程中可看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时，再将偏向锁撤销为无锁状态或升级为轻量级，会消耗一定的性能，**所以在多线程竞争频繁的情况下，偏向锁不仅不能提高性能，还会导致性能下降。于是，就有了批量重偏向与批量撤销的机制。**
应用场景：
- **批量重偏向（bulk rebias）机制是为了解决**：一个线程创建了大量对象并执行了初始的同步操作，后来另一个线程也来将这些对象作为锁对象进行操作，这样会导致大量的偏向锁撤销操作。
- **批量撤销（bulk revoke）机制是为了解决：**在明显多线程竞争剧烈的场景下使用偏向锁是不合适的。

概括：
1. **批量重偏向和批量撤销是针对类的优化，和对象无关。**
2. **偏向锁重偏向一次之后不可再次重偏向。**
3. **当某个类已经触发批量撤销机制后，JVM会默认当前类产生了严重的问题，剥夺了该类的新实例对象使用偏向锁的权利**

###### 自旋优化
重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。
- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。
- 在 Java 6 之后自旋是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，比较智能。
- Java 7 之后不能控制是否开启自旋功能

注意：**自旋的目的是为了减少线程挂起的次数，尽量避免直接挂起线程（挂起操作涉及系统调用，存在用户态和内核态切换，这才是重量级锁最大的开销）**

###### 锁粗化
假设一系列的连续操作都会**对同一个对象反复加锁及解锁，甚至加锁操作是出现在循环体中的**，即使没有出现线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。如果JVM检测到有一连串零碎的操作都是对同一对象的加锁，将会扩大加锁同步的范围（即锁粗化）到整个操作序列的外部。
```java
StringBuffer buffer = new StringBuffer();
public void append() {
	buffer.append("aaa").append(" bbb").append(" ccc");
}
```
上述代码每次调用 buffer.append 方法都需要加锁和解锁，如果JVM检测到有一连串的对同一个对象加锁和解锁的操作，**就会将其合并成一次范围更大的加锁和解锁操作**，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

###### 锁消除
锁消除即删除不必要的加锁操作。锁消除是Java虚拟机在JIT编译期间，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。
StringBuffer的append是个同步方法，但是append方法中的 StringBuffer 
```java
public class LockEliminationTest {

    StringBuffer buffer = new StringBuffer();
    /**
     * 锁粗化
     */
    public void append(){
        buffer.append("aaa").append(" bbb").append(" ccc");
    }

    /**
     * 锁消除
     * -XX:+EliminateLocks 开启锁消除(jdk8默认开启）
     * -XX:-EliminateLocks 关闭锁消除
     * @param str1
     * @param str2
     */
    public void append(String str1, String str2) {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append(str1).append(str2);
    }

    public static void main(String[] args) throws InterruptedException {
        LockEliminationTest demo = new LockEliminationTest();
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            demo.append("aaa", "bbb");
        }
        long end = System.currentTimeMillis();
        System.out.println("执行时间：" + (end - start) + " ms");
    }

}
```
属于一个局部变量，不可能从该方法中逃逸出去，因此其实这过程是线程安全的，可以将锁消除。
测试结果：开启锁消除：1903 ms；关闭锁消除：2833 ms
### AQS
#### 什么是AQS
java.util.concurrent包中的大多数同步器实现都是围绕着共同的基础行为，比如等待队列、条件队列、独占获取、共享获取等，而这些行为的抽象就是基于**AbstractQueuedSynchronizer（简称AQS）**实现的，AQS是一个抽象同步框架，可以用来实现一个依赖状态的同步器。
JDK中提供的大多数的同步器如Lock, Latch, Barrier等，都是基于AQS框架来实现的：
- 一般是通过一个内部类Sync继承 AQS
- 将同步器所有调用都映射到Sync对应的方法
##### AQS相关特点
5大特性：
- 阻塞等待队列
- 共享/独占
- 公平/非公平
- 可重入
- 允许中断

AQS内部维护的属性 ```volatile int state``` 解释：state表示资源的可用状态。
**访问该state有三种方式**
1. getState()
2. setState()
3. compareAndSetState()

AQS定义两种资源共享方式：
1. **Exclusive-独占**：只有一个线程能执行，如ReentrantLock
2. **Share-共享**：多个线程可以同时执行，如Semaphore/CountDownLatch

AQS定义两种队列（MESA实现）:
1. **同步等待队列**： 主要用于维护获取锁失败时入队的线程
2. **条件等待队列**： 调用**await()**的时候会释放锁，然后线程会加入到条件队列，调用**signal()**唤醒的时候会把条件队列中的线程节点移动到同步队列中，等待再次获得锁

AQS 定义了5个队列中节点状态（竞争失败后进行处理的内容，在队列节点Node中实现的）：
1. **初始化状态**：值为0，，表示当前节点在sync队列中，等待着获取锁。
2. **CANCELLED**：值为1，表示当前的线程被取消；
3. **SIGNAL**：值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；
4. **CONDITION**：值为-2，表示当前节点在等待condition，也就是在condition队列中；
5. **PROPAGATE**：值为-3，表示当前场景下后续的acquireShared能够得以执行；

不同的自定义同步器竞争共享资源的方式也不同。
自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。
**自定义同步器实现时主要实现以下几种方法：**
- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

##### 同步等待队列
AQS当中的同步等待队列也称CLH队列，CLH队列是Craig、Landin、Hagersten三人发明的一种基于双向链表数据结构的队列，是FIFO先进先出线程等待队列。
Java中的CLH队列是原CLH队列的一个变种，**线程由原自旋机制改为阻塞机制。**
AQS 依赖CLH同步队列来完成同步状态的管理：
- 当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程
- 当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。
- 通过signal或signalAll将条件队列中的节点转移到同步队列。（由条件队列转化为同步队列）
![AQS-同步等待队列](181e5700/AQS-同步等待队列.png)

##### 条件等待队列
AQS中条件队列是使用单向链表保存的，用nextWaiter来连接：
- 调用await方法阻塞线程；
- 当前线程存在于同步队列的头结点，调用await方法进行阻塞（从同步队列转化到条件队列）

#### Condition接口详解
![Condition接口](181e5700/structure_of_condition.png)
- 调用**Condition#await**方法会释放当前持有的锁，然后**阻塞当前线程**，同时向Condition队列尾部添加一个节点，所以调用Condition#await方法的时候必须持有锁。
- 调用**Condition#signal**方法会将Condition队列的首节点移动到阻塞队列尾部，然后唤醒因调用Condition#await方法而阻塞的线程(唤醒之后这个线程就可以去竞争锁了)，所以调用Condition#signal方法的时候必须持有锁，持有锁的线程唤醒被因调用Condition#await方法而阻塞的线程。

**Condition结合ReentrantLock，在阻塞队列中使用的非常多。**

#### ReentrantLock(独占锁)
ReentrantLock是一种基于AQS框架的应用实现，是JDK中的一种线程并发访问的同步手段，它的功能**类似于synchronized是一种互斥锁，可以保证线程安全。**
相对于 synchronized， ReentrantLock具备如下特点：
- 可中断
- 可以设置超时时间：一定时间内是否能够获取锁
- 可以设置为公平锁
- 支持多个条件变量
- 与 synchronized 一样，都支持可重入

##### synchronized和ReentrantLock的区别
- synchronized是JVM层次的锁实现，ReentrantLock是JDK层次的锁实现；
- synchronized的锁状态是无法在代码中直接判断的，但是ReentrantLock可以通过ReentrantLock#isLocked判断；
- synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的；
- synchronized是不可以被中断的，而ReentrantLock#lockInterruptibly方法是可以被中断的；
- 在发生异常时synchronized会自动释放锁，而ReentrantLock需要开发者在finally块中显示释放锁；
- ReentrantLock获取锁的形式有多种：如立即返回是否成功的tryLock(),以及等 待指定时长的获取，更加灵活；
- synchronized在特定的情况下对于已经在等待的线程是后来的线程先获得锁（cqx队列），而ReentrantLock对于已经在等待的线程是先来的线程先获得锁；

##### 使用场景
**使用场景特别要注意的**：一定要处理好异常情况，即释放锁的时候不能出现因为异常而导致释放锁失败的情况。
- **可中断**：线程协作时的一个典型用法，在A线程中调用lockInterruptibly()后，在B线程中调用interrupt()时会影响A线程中的后续执行逻辑，直接抛出异常。
- **超时失败**：tryLock中传入时间参数
- **公平锁**：默认是非公平的，创建的时候传入ture标志的时候创建的就是公平锁。默认非公平的原因是效率高。

##### 相关源码实现
1. ReentrantLock加锁解锁的逻辑
2. 公平和非公平，可重入锁的实现
3. 线程竞争锁失败入队阻塞逻辑和获取锁的线程释放锁唤醒阻塞线程竞争锁的逻辑实现（设计的精髓：并发场景下入队和出队操作）。

一段debug代码：
很简单，就是3个线程对一个total变量进行累加操作，中间通过ReentrantLock来进行独占
```java
@Slf4j
public class DebugReentrantLockDemo {

    private static int total = 0;
    private static final Lock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {

        for (int i = 0; i < 3; i++) {
            Thread thread = new Thread(() -> {
                // 加锁 下面一行设置断点
                // 在idea中将断点模式设置为thread（下同），方便对指定线程执行操作
                lock.lock();
                try {
                    // 临界区代码，下面一行设置断点
                    for (int j = 0; j < 10000; j++) {
                        total++;
                    }
                } finally {
                    // 解锁
                    lock.unlock();
                }
            });
            thread.start();
        }

        Thread.sleep(2000);
        log.info("summary result is:[{}]", total);
    }
}
```
其中在IDEA中设置Thread模式断点如下图所示（在断点上鼠标右键）：
![IDEAThread断点模式](181e5700/IDEAThread断点模式.png)
结合这个调试程序单独拆解里面的步骤：
3个线程的名字，假设分别叫Thread1、2、3，用于下文标记
###### 1. 初始化ReentrantLock对象
这里需要结合ReentrantLock的相关代码结构才好看后面的流程
首先关注一下ReentrantLock跟本程序调用相关的结构：
![ReentrantLock结构1](181e5700/ReentrantLock结构1.png)
这里面：
- Sync内置抽象类继承了AbstractQueuedSynchronizer，是ReentrantLock锁的同步控制基础（核心结构）。分为公平和非公平版本（内部类NonfairSync和FairSync）。使用 AQS 状态来表示锁定的保持次数。
- 看看AbstractQueuedSynchronizer：这里面定义了所有基于AQS实现的锁机制的公共方法类，跟这个debug程序相关的，需要知道：
	- AbstractQueuedSynchronizer静态初始化了几个内部变量，用于后续CAS同步锁，其中有个属性：**volatile int state用于标记同步状态的，默认初始化为0**。
	- 定义了一个静态内部类：```java.util.concurrent.locks.AbstractQueuedSynchronizer.Node```，这个类定义了等待队列节点，参考CLH，并扩展为双向指针链表（独占模式下是，共享模式退化成单链表结构）。具体直接可以看该类的文档注释。
	- Node结构中有几个重要的属性结构，为后续等待队列的结构变动，线程的等待/唤醒服务：
		- volatile int waitStatus：节点的等待状态，有5种值，在默认的独占模式（EXCLUSIVE）下，值用只用到了1种，即-1（SIGNAL指示后继线程需要取消停放），加上默认的0，只有2种。
		- volatile Node prev：双向链表前驱节点
		- volatile Node next：双向链表后继节点
		- volatile Thread thread：当前等待线程
	- AbstractQueuedSynchronizer父类：AbstractOwnableSynchronizer内持有一个对象```transient Thread exclusiveOwnerThread```，用于在独占模式下标记同步的当前线程。
	- 其它方法结构，后续时序图中碰到再单独解释
- NonfairSync内部类继承Sync并实现了lock方法，是非公平锁的同步对象
- ReentrantLock默认构造函数实现的是：NonfairSync

###### 2.1 Thread1调用ReentrantLock.lock()
对应代码：
```java
    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
```
时序图如下，不是特别复杂：
{% mermaid sequenceDiagram %}
actor User
User ->> NonfairSync : lock
activate NonfairSync
NonfairSync ->> AbstractQueuedSynchronizer : CAS对AQS抽象类的stateOffset操作compareAndSetState(0,1)
activate AbstractQueuedSynchronizer
AbstractQueuedSynchronizer -->> NonfairSync : #32; 
deactivate AbstractQueuedSynchronizer
alt 判断上述compareAndSetState(0, 1)结果
NonfairSync ->> Thread : 调用currentThread()设置exclusiveOwnerThread=currentThread
activate Thread
Thread -->> NonfairSync : #32; 
deactivate Thread
else 
NonfairSync ->> AbstractQueuedSynchronizer : 独占模式尝试获取锁acquire(1)
activate AbstractQueuedSynchronizer
AbstractQueuedSynchronizer -->> NonfairSync : #32; 
deactivate AbstractQueuedSynchronizer
end
deactivate NonfairSync
{% endmermaid %}

###### 2.2 Thread1获取到锁
此时Thread1的ReentrantLock.state=1
###### 3. Thread2调用lock方法
这时候调用compareAndSetState(0, 1)的CAS操作，肯定是返回失败的。走到调用acquire(1)方法
```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
	// 这一行调用了三个方法：tryAcquire(arg)-尝试获取锁
	// addWaiter(Node.EXCLUSIVE) - 首次调用构建双向队列
	// acquireQueued(Node) - 入队列
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        	// 调用复位中断标记位
            selfInterrupt();
    }
```
```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#addWaiter
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
        	// 首次调用为假，后续再次调用时本质就是尾插法
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 入队列
        enq(node);
        return node;
    }
```

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
            	// 首次调用先初始化表head节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
            	// 再次调用会调用尾插法入队列
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) { // 注意这里有个自旋操作，这个是后续阻塞队列种的中断线程重新获取锁的关键
            	// 本例子Thread2第一次进入这里时，获取的p是head节点
            	// 自旋后Thread2第二次进入，同样为false
                final Node p = node.predecessor(); // 获取当前节点的前驱节点
            	// Thread2第一次执行判断： p == head为真，tryAcquire(arg)再次获取锁为假
                if (p == head && tryAcquire(arg)) {
                	// 新唤起的线程获取锁成功之后，要清除当前队列的head节点，并更新当前node为head节点。
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted; 
                }
                // Thread2第一次执行下面判断条件，shouldParkAfterFailedAcquire返回false
                // Thread2第二次执行，shouldParkAfterFailedAcquire为true，开始执行parkAndCheckInterrupt()，挂起成功返回true
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
// java.util.concurrent.locks.AbstractQueuedSynchronizer#shouldParkAfterFailedAcquire
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // Thread2首次进入时：ws=0
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
        	// Thread2首次进入时：通过CAS将waitStatus=-1
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

// java.util.concurrent.locks.AbstractQueuedSynchronizer#parkAndCheckInterrupt
private final boolean parkAndCheckInterrupt() {
		// Thread2 进入这里执行后，Thread2线程状态WAIT，开始阻塞
		// park里面会有识别中断标志位，以便后续唤醒操作
        LockSupport.park(this);
        // 清除中断，为了后续自旋加锁逻辑
        // 后续需要恢复，在acquire方法里面的：selfInterrupt()
        return Thread.interrupted();
    }
```

```java
// java.util.concurrent.locks.ReentrantLock.Sync#nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            // 获取当前state状态
            int c = getState();
            if (c == 0) {
            	// 如果是0，表示可以获取锁
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
            	// 比较当前线程是不是锁持有线程，用来标记可重入次数
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
###### 4. Thread3调用lock方法
同第3步一样，上锁失败完了，入队调用park进入阻塞。
###### 5. Thread1调用unlock方法
直接调用relase方法
```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#release
public final boolean release(int arg) {
        if (tryRelease(arg)) { // tryRelease就是设置state=0和重置exclusiveOwnerThread=null
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
// java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor
private void unparkSuccessor(Node node) {
        
        int ws = node.waitStatus;
        if (ws < 0)
        	// 清除head节点中的等待信号
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
        	// 这里调用unpark唤醒线程，传入的参数是node节点中暂存的thread线程
            LockSupport.unpark(s.thread);
    }
```
###### 5.1. 唤醒Thread2线程
上面代码里面注释已经标注
###### 5.2. Thread2再次尝试获取锁
###### 5.3. Thread2加锁成功
###### 5.3. 调整内部链表结构

###### 总结-整体流程
调用流程大致如下：
*PS：整整画了我2小时 :(*
![完整调用流程](181e5700/ReentrantLock执行流程图.jpg)

#### Semaphore
**Semaphore，俗称信号量，它是操作系统中PV操作的原语在java的实现**，它也是基于AbstractQueuedSynchronizer实现的。
Semaphore的功能非常强大：
大小为1的信号量就类似于互斥锁，通过同时只能有一个线程获取信号量实现；
大小为n（n>0）的信号量可以实现限流的功能，它可以实现只能有n个线程同时获取信号量。
![Semaphore示意图](181e5700/Semaphore示意图.jpg)

##### 常用用法
```java
public void acquire() throws InterruptedException // 表示阻塞并获取许可
public boolean tryAcquire() // 在没有许可的情况下会立即返回 false，要获取许可的线程不会阻塞
public void release() // 释放许可
public int availablePermits() // 返回此信号量中当前可用的许可证数
public final int getQueueLength() // 返回正在等待获取许可证的线程数。
public final boolean hasQueuedThreads() // 是否有线程正在等待获取许可证
protected void reducePermits(int reduction) // 减少 reduction 个许可证
protected Collection<Thread> getQueuedThreads() // 返回所有等待获取许可证的线程集合
```
##### 应用场景
一个简单的限流示例
```java
public class SemaphoreTest2 {

    /**
     * 实现一个同时只能处理5个请求的限流器
     */
    private static Semaphore semaphore = new Semaphore(5);
    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    /**
     * 定义一个线程池
     */
    private static ThreadPoolExecutor executor = new ThreadPoolExecutor
            (10, 50, 60,
                    TimeUnit.SECONDS, new LinkedBlockingDeque<>(200));

    /**
     * 模拟执行方法
     */
    public static void exec() {
        try {
            //占用1个资源
            semaphore.acquire(1);
            //TODO  模拟业务执行
            System.out.println(Thread.currentThread().getName() + ":" + sdf.format(new Date()) + ":执行exec方法");
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            //释放一个资源
            semaphore.release(1);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (; ; ) {
            Thread.sleep(100);
            // 模拟请求以10个/s的速度
            executor.execute(() -> exec());
        }
    }
}
```
##### 源码分析
跟ReentrantLock处理队列的方式相同（都是在AQS抽象类里面实现的），区别在于：
```java
// java.util.concurrent.Semaphore#acquire(int)
public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
```

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireSharedInterruptibly
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
            // 
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0) // 先尝试获取锁，如果返回的是负数，就执行
            doAcquireSharedInterruptibly(arg);
    }

// java.util.concurrent.Semaphore.Sync#nonfairTryAcquireShared
final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
            	// 初始化Semaphore的时候，构造器传入的资源数将赋值给state，这里取出来
                int available = getState();
                int remaining = available - acquires; // 计算剩余资源数
                // 大于0，通过CAS交换；反之直接返回负数的结果
                if (remaining < 0 || compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
共享模式下跟独占模式的队列操作不一样的核心代码：
```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireSharedInterruptibly
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 这里通过addWaiter初始化的Node节点模式是Node.SHARED
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                // 获取资源失败之后，这里就开始准备阻塞和进行阻塞两个动作。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
释放资源，重新累加state的同时需要唤醒队列线程


#### CountDownLatch
CountDownLatch（闭锁）是一个**同步协助类，允许一个或多个线程等待，直到其他线程完成操作集**。
CountDownLatch使用给定的计数值（count）初始化。
await方法会阻塞直到当前的计数值（count）由于countDown方法的调用达到0，count为0之后所有等待的线程都会被释放，并且随后对await方法的调用都会立即返回。
**这是一个一次性现象：count不会被重置。**
如果你需要一个**重置count的版本**，那么请**考虑使用CyclicBarrier**。
![CountDownLatch示意图](181e5700/CountDownLatch示意图.jpg)

##### 应用场景
CountDownLatch一般用作多线程倒计时计数器，强制它们等待其他一组（CountDownLatch的初始化决定）任务执行完成
##### 实现原理
底层基于 AbstractQueuedSynchronizer 实现：
CountDownLatch 构造函数中**指定的count直接赋给AQS的state**；每次countDown()则都是release(1)减1，最后减到0时unpark阻塞线程；这一步是由最后一个执行countdown方法的线程执行的。
而调用await()方法时，当前线程就会判断state属性是否为0，如果为0，则继续往下执行，如果不为0，则使当前线程进入等待状态，直到某个线程将state属性置为0，其就会唤醒在await()方法中等待的线程。

##### CountDownLatch与Thread.join的区别
- CountDownLatch的作用就是**允许一个或多个线程等待**其他线程完成操作，看起来有点类似join() 方法，但其提供了比 join() 更加灵活的API。
- CountDownLatch**可以手动控制**在n个线程里调用n次countDown()方法使计数器进行减一操作，也可以在一个线程里调用n次执行减一操作。
- 而 join() 的实现原理是不停检查join线程是否存活，如果**join 线程存活则让当前线程永远等待**。所以两者之间相对来说还是CountDownLatch使用起来较为灵活。
##### CountDownLatch与CyclicBarrier的区别
- CountDownLatch的计数器只能使用一次，而**CyclicBarrier的计数器可以使用reset()方法重置**。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。
- CyclicBarrier还提供getNumberWaiting(可以获得CyclicBarrier阻塞的线程数量)、isBroken(用来知道阻塞的线程是否被中断)等方法。
- CountDownLatch会阻塞主线程，CyclicBarrier不会阻塞主线程，只会阻塞子线程。
- CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同:
	- CountDownLatch一般用于一个或多个线程，等待其他线程执行完任务后，再执行。
	- CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行。
- CyclicBarrier 还可以提供一个 barrierAction，**合并多线程计算结果**。
- **CyclicBarrier是通过ReentrantLock的"独占锁"和Conditon来实现一组线程的阻塞唤醒的，而CountDownLatch则是通过AQS的“共享锁”实现**

#### CyclicBarrie
字面意思回环栅栏，通过它可以实现**让一组线程等待至某个状态（屏障点）之后再全部同时执行**。叫做回环是因为当所有等待线程都被释放以后，**CyclicBarrier可以被重用**。
原始计数变量存在副本parties属性中，调用重置方法之后会还原该值。
![CyclicBarrie示意图](181e5700/CyclicBarrie示意图.jpg)

##### 应用场景
一个合并计算的示例
```java
/**
 * 栅栏与闭锁的关键区别在于，所有的线程必须同时到达栅栏位置，才能继续执行。
 */
public class CyclicBarrierTest2 {

    //保存每个学生的平均成绩
    private ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<String, Integer>();

    private ExecutorService threadPool = Executors.newFixedThreadPool(3);

    private CyclicBarrier cb = new CyclicBarrier(3, () -> {
        int result = 0;
        Set<String> set = map.keySet();
        for (String s : set) {
            result += map.get(s);
        }
        System.out.println("三人平均成绩为:" + (result / 3) + "分");
    });


    public void count() {
        for (int i = 0; i < 3; i++) {
            threadPool.execute(new Runnable() {

                @Override
                public void run() {
                    //获取学生平均成绩
                    int score = (int) (Math.random() * 40 + 60);
                    map.put(Thread.currentThread().getName(), score);
                    System.out.println(Thread.currentThread().getName()
                            + "同学的平均成绩为：" + score);
                    try {
                        //执行完运行await(),等待所有学生平均成绩都计算完毕
                        cb.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }

            });
        }
    }


    public static void main(String[] args) {
        CyclicBarrierTest2 cb = new CyclicBarrierTest2();
        cb.count();
    }
}
```
一段等待就绪示例：
```java
/**
 * 一个小的“人满发车”场景的demo
 */
@Slf4j
public class CyclicBarrierTest3 {

    public static void main(String[] args) throws InterruptedException {

        AtomicInteger counter = new AtomicInteger(); // 计数标识选手序号
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                5, 5, 1000, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100),
                (r) -> new Thread(r, "第" + counter.addAndGet(1) + "号 "),
                new ThreadPoolExecutor.AbortPolicy());

        int cbInitSize = 5;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(cbInitSize,
                () -> System.out.println("===========裁判：比赛开始==========="));

        for (int i = 0; i < cbInitSize * 4; i++) {
            threadPoolExecutor.submit(new Runner(cyclicBarrier));
        }
        threadPoolExecutor.shutdown();

    }

    static class Runner extends Thread {
        private CyclicBarrier cyclicBarrier;

        public Runner(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                int sleepMills = ThreadLocalRandom.current().nextInt(1000);
                Thread.sleep(sleepMills);
                System.out.println(Thread.currentThread().getName() + " 选手已就位, 准备共用时:" + sleepMills + " ms" + ", 准备完成序号:" + cyclicBarrier.getNumberWaiting());
                cyclicBarrier.await();

            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }

}
```
##### 相关源码
使用了Condition作为条件队列原语：调用Condition.await()进行阻塞，入队操作。 
这里面会有一个条件队列转同步队列后再唤醒的机制：
用ReentrantLock锁条件队列阻塞过程，在阻塞过程中，转同步队列后，条件队列释放锁后续同步队列唤醒、拿锁、执行（AQS的for循环自旋）
核心代码就下面这段
```java
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 注意这里的 ReentrantLock
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run(); // 条件满足后，执行run方法
                    ranAction = true;
                    nextGeneration(); // 进入下一个屏障区间，并调用singalAll()唤醒所有线程（有点复杂）
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await(); // 这个地方会进行阻塞，等后续唤醒后再执行
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
        	// 注意这里的unlock()
        	// 当最后一个条件队列计数完成，等待唤醒的时候，
        	// 当前所在的线程调用unlock，由于再次之前调用signalAll()方法的时候，已经完成了条件队列节点向同步队列节点传输转换的动作。
        	// 所以这时候调用unlock会唤醒后续节点的线程
        	// 同样的，后续被唤醒的线程也会走到这里，继续“传递”似的唤醒后续同步队列的节点，如此往复，直到同步队列节点全部被唤醒。
            lock.unlock();
        }
    }
// java.util.concurrent.CyclicBarrier#nextGeneration
 private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```
唤醒逻辑中实现了队列转换的逻辑：
```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer.ConditionObject#doSignalAll
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    // 这个地方可以琢磨一下，为什么不用while而是用do-while
    // 原因在于，如果是首次进入的是，第一次调用transferForSignal，只是做了条件队列节点向同步队列传输的动作
    // 与此同时会初始化同步队列的结构，但是并不会唤醒。
    // 此时尾循环的判断就起到作用，为真的时候，会再次进入循环调用transferForSignal入队。
    // 直到 (first != null) 为假跳出循环
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
// java.util.concurrent.locks.AbstractQueuedSynchronizer#transferForSignal
final boolean transferForSignal(Node node) {
		// 核心其实在于NODE节点设计的巧妙，通过不同的状态位，巧妙的在这里转换过去
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

#### ReentrantReadWriteLock(读写锁)
现实中有这样一种场景：**对共享资源有读和写的操作，且写操作没有读操作那么频繁（读多写少）**。
在**没有写操作**的时候，多个线程同时读一个资源没有任何问题，所以应该**允许多个线程同时读取**共享资源（读读可以并发）；
但是如果**一个线程想去写**这些共享资源，**就不应该允许其他线程对该资源进行读和写操作了**（*读写，写读，写写互斥*）。
在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

针对这种场景，JAVA的并发包提供了读写锁ReentrantReadWriteLock，它内部，维护了一对相关的锁，一个用于只读操作，称为读锁；一个用于写入操作，称为写锁: 
- 线程进入读锁的前提条件:
	- 没有其他线程的写锁
	- 没有写请求或者，有写请求但调用线程和持有锁的线程是同一个。
- 线程进入写锁的前提条件：
	- 没有其他线程的读锁
	- 没有其他线程的写锁

##### 读写锁的三个重要特性
- **公平选择性**：支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平。
- **可重入**：读锁和写锁都支持线程重入。以读写线程为例：读线程获取读锁后，能够再次获取读锁。写线程在获取写锁之后能够再次获取写锁，同时也可以获取读锁。
- **锁降级**：遵循获取写锁、再获取读锁最后释放写锁的次序，写锁能够降级成为读锁。
##### 相关原理
![读写锁类结构](181e5700/structure_of_ReentrantReadWriteLock.jpg)
相比其它锁，共有的两个公平非公平同步结构之外，还多了两个单独实现Lock接口的锁模块。
两个锁本质是通过state状态的高低位是否为0来判断是否有读锁/写锁，其中：写锁实现就是独占锁；读锁实现就是共享锁实现。
![ReentrantReadWrite锁工作示意图](181e5700/ReentrantReadWrite锁工作示意图.jpg)
采用“按位切割使用”的方式来维护这个变量，高16为表示读，低16为表示写：
- 写状态，等于 S & 0x0000FFFF（将高 16 位全部抹去）。 当写状态加1，等于S+1.
- 读状态，等于 S >>> 16 (无符号补 0 右移 16 位)。当读状态加1，等于S+（1<<16）,也就是S+0x00010000

根据状态的划分能得出一个推论：**S不等于0时，当写状态（S&0x0000FFFF）等于0时，则读状态（S>>>16）大于0，即读锁已被获取。**
![读写锁高低位](181e5700/读写锁高低位.jpg)


##### 读写锁使用
一个经典的使用场景就是缓存，读多写少。

##### 锁降级
**锁降级指的是写锁降级成为读锁。**
**如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。**
**降级的目的是为了内存可见性问题**。
锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。锁降级可以帮助我们拿到当前线程修改后的结果而不被其他线程所破坏，防止更新丢失。

## 并发编程相关
### 阻塞队列相关
BlockingQueue 继承了 Queue 接口，是队列的一种。Queue 和 BlockingQueue 都是在 Java5 中加入的。阻塞队列（BlockingQueue）是**一个在队列基础上又支持了两个附加操作的队列**，常用解耦。
两个附加操作:
- 支持阻塞的插入方法put: 队列满时，队列会阻塞插入元素的线程，直到队列不满。
- 支持阻塞的移除方法take: 队列空时，获取元素的线程会等待队列变为非空

#### BlockingQueue常用方法示例
当队列满了无法添加元素，或者是队列空了无法移除元素时：
1. 抛出异常：add、remove、element
2. 返回结果但不抛出异常：offer、poll、peek
3. 阻塞：put、take

| 方法        | 抛出异常           | 返回特定值  | 阻塞  |  阻塞特定时间  |
| ------------- |:-------------:|:-------:|:-----:|:-------:|
| 入队      | add(e) | offer(e) | put(e) | offer(e, time, unit) |
| 出队      | remove()      |   poll() | take() | poll(time, unit) |
| 获取队首元素      | element()      |   peek() | 不支持 | 不支持 |

#### 阻塞队列特性
阻塞队列区别于其他类型的队列的**最主要的特点就是“阻塞”**这两个字：阻塞功能使得生产者和消费者两端的能力得以平衡，当有任何一端速度过快时，阻塞队列便会把过快的速度给降下来。实现阻塞**最重要的两个方法是 take 方法和 put 方法。**
##### take方法
ake 方法的功能是获取并移除队列的头结点，通常在队列里有数据的时候是可以正常移除的。可是一旦执行 take 方法的时候，队列里无数据，则阻塞，直到队列里有数据。一旦队列里有数据了，就会立刻解除阻塞状态，并且取到数据。
![take方法示例](181e5700/take方法示例.jpg)

##### put方法
put 方法插入元素时，如果队列没有满，那就和普通的插入一样是正常的插入，但是如果队列已满，那么就无法继续插入，则阻塞，直到队列里有了空闲空间。如果后续队列有了空闲空间，比如消费者消费了一个元素，那么此时队列就会解除阻塞状态，并把需要添加的数据添加到队列中。
![put方法示例](181e5700/put方法示例.jpg)

#### 常见的阻塞队列
| 队列      |  描述           |
| ------------- |:-------------:|
| ArrayBlockingQueue      | 基于数组结构实现的一个有界阻塞队列 |
| LinkedBlockingQueue      | 基于链表结构实现的一个有界阻塞队列 |
| PriorityBlockingQueue      | 支持按优先级排序的无界阻塞队列 |
| DelayQueue      | 基于优先级队列（PriorityBlockingQueue）实现的无界阻塞队列 |
| SynchronousQueue      | 不存储元素的阻塞队列 |
| LinkedTransferQueue      | 基于链表结构实现的一个无界阻塞队列 |
| LinkedBlockingDeque      | 基于链表结构实现的一个双端阻塞队列 |

![阻塞队列一览](181e5700/阻塞队列一览.jpg)
https://www.processon.com/view/link/6724421b7f2523247300baf9?cid=6724363c61fdee7d75fa9b4f

## 设计模式相关

## JDK8下的JUC包相关