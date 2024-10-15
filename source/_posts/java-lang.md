---
title: java-lang包学习
abbrlink: '56093694'
date: 2022-05-15 23:17:42
tags:
    - java.lang
    - rt.jar
    - jdk8
categories: Java源码系列
---


## 前言
废话少说，从头学Java之第一篇。
按日常使用优先级分级阅读，为了防止篇幅过于太长，其中值得详细探讨的，会单独独立po新文出来。

<!-- more -->

## lang包基本概览
doc api中是这样描述的：
> Provides classes that are fundamental to the design of the Java programming language.

对于Java语言来说，不可或缺的(be fundamental to)，可见这个包是多么的重要！
这个包下面有几大块内容：
- Object类：“万恶之源” 
- 基本类型的包装器
- 常用数学运算工具类：Math
- 字符串操作相关类：String等
- "系统操作"相关类：ClassLoader等
- 异常/错误相关定义：Exception、Error等

从上面的大概的几块内容来看，确实是日常Java语言中，无处不在的。

### lang包下一些类

#### 接口(Interface)相关

##### Appendable
一个附加字符串定义的接口，提供了三个操作方法，异常抛出的是IOException：
这些定义方法中，没有明确任何关于线程安全的定义，需要有各个子类实现类中具体实现
```java
Appendable append(CharSequence csq) throws IOException;
Appendable append(CharSequence csq, int start, int end) throws IOException;
Appendable append(char c) throws IOException;
```
###### 继承图谱
![hierachy_appendable](56093694/hierachy_appendable.jpg)


##### AutoCloseable
一个在JDK1.7中引入的特性，从源码文档中可以看出：一个在关闭之前可能持有资源（例如文件或套接字句柄）的对象。 AutoCloseable对象的close() 方法在退出资源规范标头中已为其声明对象的 try-with-resources 块时自动调用。
你要实现自动关闭，则需要配合try-with-resources语法定义，才会生效。

##### CharSequence
提供了一个可读的char值序列，注意一个点：这个接口没有细化 equals 和 hashCode 方法的一般契约。 因此，比较实现 CharSequence 的两个对象的结果通常是未定义的。 每个对象都可以由不同的类实现，并且不能保证每个类都能够测试其实例与其他类的实例是否相等。 **因此，将任意 CharSequence 实例用作集合中的元素或映射中的键是不合适的。**

在JDK1.8中引入了两个stream操作:
```java
default IntStream chars(); // zero-extending
default IntStream codePoints(); // Any surrogate pairs encountered
```

##### Cloneable
Object.clone时的标记接口：在未实现 Cloneable 接口的实例上调用 Object 的 clone 方法会导致抛出异常 CloneNotSupportedException。

关联子类众多，无处不在。

##### Comparable<T>
该接口对实现它的每个类的对象进行了总排序。这种排序称为类的自然排序，类的 compareTo 方法称为其自然比较方法。
实现此接口的对象列表（和数组）可以由 Collections.sort（和 Arrays.sort）自动排序。实现此接口的对象可以用作排序映射中的键或排序集中的元素，而无需指定比较器。


##### Iterable<T>
1.5 只提供关于迭代器定义实现。

##### Readable
1.5 提供的一个字符串读取的read方法的定义，将字符读取到字符缓冲区。其直接实现的子类有两个：
- 抽象类：Reader
- 抽象类：CharBuffer nio包下的字符缓冲操作类，细节在nio章节详述
可以见得，与其子类相关字符操作的，均会涉及该方法重写相关。

##### Runnable
1.0 开始，元组级别的接口定义，干嘛的不用多说，接触过java，谁还不知道这个interface呢？
在1.8里面，有点点不一样。类上加了一个注解： ```@FunctionalInterface``` 该注解在lang的子包: annoation中。
主要特性，在doc中也有详细阐述，我们需要知道的是：
- 从定义上，这是一个信息性注解类型(informative annotation type)，就是定义声明了一种在JLS中定义的功能性的接口实现。
- 从功能上，这种被赋予该注解的功能性的接口，其对应示例可以通过 **lambda 表达式、方法引用或构造函数引用来创建。** 这个特性，在1.8中引入lambda支持之后，日常开发过程中真是无处不再。
- 元组增强的，我们平时经常能碰到的就Runnable这个被增强过，另一个是在JUC中的Callable接口，还有一个通常跟JWT相关的玩意儿，就不说了；剩下的，跟Core Java相关的，都是围绕1.8引入的lambda, stream息息相关

##### Thread.UncaughtExceptionHandler
一个线程由于未捕获的异常而突然终止时调用的处理程序接口。
这也是一个功能性接口，是Thread内部的一个扩展interface，一般业务侧系统开发，不会直接实现重写这个，除非做底层中间件开发相关的。

#### 类(Class)相关
- 关于包装器类，需要通用知道的：
    + 每个包装器（除了Boolean，true,false太特殊，在JVM中有特定表示）都有一个SIZE，用二进制补码的方式表明这个包装器类允许的最大长度(byte)
    + 数值型的包装器，往往都有一个内联的Cache类，用于缓存，在与基本类型转换的时候要注意，**特殊场景的越界可能会发生**
##### Boolean
基本类型boolean包装器类，提供了一些与String互转的操作方法。
关于转换和逻辑运算的方法，就不过多阐述。
源码实现中，有个小点值得提一下：
```java
public static int hashCode(boolean value) {
        return value ? 1231 : 1237;
    }
```
Boolean的hashCode返回了是俩数字，而不是其它的。理论，**任意（足够大）的素数即可**
为啥？这里要涉及到一个知识点，哈希碰撞：
细节参阅维基百科的介绍：[hash collision](https://en.wikipedia.org/wiki/Hash_collision)

##### Byte
基本类型byte包装器类，提两个操作：
- 从String转为byte方法：public static byte parseByte(String s, int radix)
- 对String进行decode成Byte包装器：public static Byte decode(String nm) throws NumberFormatException
这俩操作，都跟Integer包装器类息息相关，底层都调用了Integer包装器类中的相关方法进行运算，这里不过多展开。细节看Integer包装器的介绍。

##### Character
基本类型char的包装器，大量包含Unicode操作相关，平时直接遇到的场景可能不会很多。
值得一提的是，这里面有个of方法：
```java
static final CharacterData of(int ch) {
        if (ch >>> 8 == 0) {     // fast-path
            return CharacterDataLatin1.instance;
        } else {
            switch(ch >>> 16) {  //plane 00-16
            case(0):
                return CharacterData00.instance;
            case(1):
                return CharacterData01.instance;
            case(2):
                return CharacterData02.instance;
            case(14):
                return CharacterData0E.instance;
            case(15):   // Private Use
            case(16):   // Private Use
                return CharacterDataPrivateUse.instance;
            default:
                return CharacterDataUndefined.instance;
            }
        }
    }
```
性能可能不是最好的一种快速路由查找的方式，但是却是可能最常用的。 ```.instance``` 对实例的处理，私有化默认构造器的方式，都是值得我们平时编写工具类的时候所可以借鉴的。
关于Unicode以及对应BMP(Basic Multilingual Plane)参考下面的链接：
- Unicode: https://home.unicode.org/
- BMP: https://en.wikipedia.org/wiki/Plane_(Unicode)

##### Class<T> && ClassLoader && ClassValue<T>
这三个作为一个整体，单独研究研究。
细节参考下面的传送门：
{% post_link java-lang-Class %}

##### Compiler
##### Double
##### Enum<E extends Enum<E>>
##### Float
##### InheritableThreadLocal<T>
##### Integer
##### Long
##### Math
##### Number
##### Object
##### Package
##### Process
##### ProcessBuilder
##### ProcessBuilder.Redirect
##### Runtime
##### RuntimePermission
##### SecurityManager
##### Short
##### StackTraceElement
##### StrictMath
##### String
##### StringBuffer
##### StringBuilder
##### System
##### Thread
##### ThreadGroup
##### ThreadLocal<T>
##### Throwable
##### Void

#### 枚举(Enum)相关
##### Character.UnicodeScript
##### ProcessBuilder.Redirect.Type
##### Thread.State
#### 异常(Exception)相关
##### ArithmeticException
##### ArrayIndexOutOfBoundsException
##### ArrayStoreException
##### ClassCastException
##### ClassNotFoundException
##### CloneNotSupportedException
##### EnumConstantNotPresentException
##### Exception
##### IllegalAccessException
##### IllegalArgumentException
##### IllegalMonitorStateException
##### IllegalStateException
##### IllegalThreadStateException
##### IndexOutOfBoundsException
##### InstantiationException
##### InterruptedException
##### NegativeArraySizeException
##### NoSuchFieldException
##### NoSuchMethodException
##### NullPointerException
##### NumberFormatException
##### ReflectiveOperationException
##### RuntimeException
##### SecurityException
##### StringIndexOutOfBoundsException
##### TypeNotPresentException
##### UnsupportedOperationException
#### 错误(Error)相关
##### AbstractMethodError
##### AssertionError
##### BootstrapMethodError
##### ClassCircularityError
##### ClassFormatError
##### Error
##### ExceptionInInitializerError
##### IllegalAccessError
##### IncompatibleClassChangeError
##### InstantiationError
##### InternalError
##### LinkageError
##### NoClassDefFoundError
##### NoSuchFieldError
##### NoSuchMethodError
##### OutOfMemoryError
##### StackOverflowError
##### ThreadDeath
##### UnknownError
##### UnsatisfiedLinkError
##### UnsupportedClassVersionError
##### VerifyError
##### VirtualMachineError
#### 注解(Annotation)相关
##### Deprecated
##### FunctionalInterface
##### Override
##### SafeVarargs
##### SuppressWarnings

### java.lang.annotation

### java.lang.instrument

### java.lang.invoke

### java.lang.management

### java.lang.ref

### java.lang.reflect


## 一些经验

## 总结