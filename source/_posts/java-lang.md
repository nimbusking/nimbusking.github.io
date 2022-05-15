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

##### AutoCloseable

##### CharSequence

##### Cloneable

##### Comparable<T>

##### Iterable<T>

##### Readable

##### Runnable

##### Thread.UncaughtExceptionHandler

#### 类(Class)相关

##### Boolean
##### Byte
##### Character
##### Character.Subset
##### Character.UnicodeBlock
##### Class<T>
##### ClassLoader
##### ClassValue<T>
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