---
title: Java8函数式编程原理剖析
abbrlink: 4a0d265b
date: 2024-11-02 09:33:11
updated: 2024-11-12 15:17:18
tags:
  - Java函数式编程
  - Java Stream
categories: Java系列
---


## 关于Lambda表达式
本质上，在java中的lambda表达式就是对匿名函数的一种简化实现
<!-- more -->
底层，需要知道，Lambda表达式由虚拟机指令InvokeDynamic实现方法调用。而InvokeDynamic是JVM的一条字节码指令，其通过运行时动态解析出调用的方法，然后去执行该方法。
Java8中支持Lambda表达式的，通常会以```@FunctionalInterface```来进行修饰，用来检查是否符合函数接口规范。
### 示例代码
本文用到的所有示例代码，位于：[Github:CoreJavaSample](https://github.com/nimbusking/CoreJavaSample/tree/main/src/main/java/cc/nimbusk/stream)

## Stream的定义
- 在 Java8 之前，我们通常是通过 for 循环或者 Iterator 迭代来重新排序合并数据，又或者通过重新定义 Collections.sorts 的 Comparator 方法来实现，这两种方式对于大数据量系统来说，效率并不是很理想。
- Java8 中添加了一个新的接口类 Stream，他和我们之前接触的字节流概念不太一样，**Java8 集合中的 Stream 相当于高级版的 Iterator，他通过 Lambda 表达式对集合进行各种非常便利、高效的聚合操作（Aggregate Operation），或者大批量数据操作 (Bulk Data Operation)。**

## Stream的原理
可以认为，将要处理的元素看做一种流，流在管道中传输，并且可以在管道的节点上处理，包括过滤筛选、去重、排序、聚合等。元素流在管道中进过中间操作的处理，最后由最终操作得到前面处理的结果。
来看一个简单的例子，从代码来看，使用了JavaStream的代码，确实要简洁很多：
【备注】：先打个预防针，在实际项目中，要非常清楚你的逻辑以及对Stream的使用场景，切莫盲目为了“炫技”来使用Stream。后文中会提到一些不正当使用stream带来的后果。
```java
/**
 * 一个简单的Stream示例用法比较：
 * 过滤分组一所中学里身高在 160cm 以上的男女同学
 */
@Slf4j
public class AboutStreamSample {


    public static void main(String[] args) {
        List<Student> testList = GenerateTestData.genListData();
        traditionalLoop(testList);
        usingJavaStream(testList);
        usingJavaParallelStream(testList);
    }

    public static void traditionalLoop(List<Student> studentList) {
        log.info("Before traditional loop, studentList: {}", studentList);
        Map<String, List<Student>> studentMap = new HashMap<>();
        for (Student student : studentList) {
            if (student.getHigh() > 160) {
                if (!studentMap.containsKey(student.getSex())) {
                    List<Student> students = new ArrayList<>();
                    students.add(student);
                    studentMap.put(student.getSex(), students);
                } else {
                    studentMap.get(student.getSex()).add(student);
                }
            }
        }
        log.info("After traditional loop, studentMap: {}", studentMap);
    }

    public static void usingJavaStream(List<Student> studentList) {
        log.info("Before using Java stream, studentList: {}", studentList);
        Map<String, List<Student>> studentMap = studentList.stream()
                .filter(student -> student.getHigh() > 160)
                .collect(Collectors.groupingBy(Student::getSex));
        log.info("After using Java stream, studentMap: {}", studentMap);
    }

    public static void usingJavaParallelStream(List<Student> studentList) {
        log.info("Before using Java parallel stream, studentList: {}", studentList);
        Map<String, List<Student>> stuMap = studentList.parallelStream()
                .filter(student -> student.getHigh() > 160)
                .collect(Collectors.groupingBy(Student::getSex));
        log.info("After using Java parallel stream, studentMap: {}", stuMap);
    }

}
```

## Stream如何优化遍历
### Stream的操作分类
Stream 的操作分为两大类：**中间操作（Intermediate operations）和终结操作（Terminal operations）**。
我们通常还会将中间操作称为懒操作，也正是由这种懒操作结合终结操作、数据源构成的处理管道（Pipeline），实现了 Stream 的高效。
| 操作类型        | 详情           | 备注  |
| ------------- |:-------------:|:-------------:|
|  中间操作      |只对操作进行了记录，即只会返回一个流，不会进行计算操作| 可以分为无状态（Stateless）与有状态（Stateful）操作，前者是指元素的处理不受之前元素的影响，后者是指该操作只有拿到所有元素之后才能继续下去。 |
|  终结操作      |终结操作是实现了计算操作| 可以分为短路（Short-circuiting）与非短路（Unshort-circuiting）操作，前者是指遇到某些符合条件的元素就可以得到最终结果，后者是指必须处理完所有元素才能得到最终结果。 |

![Stream操作分类](4a0d265b/Stream操作分类.jpg)

### Stream相关源码实现
先来看看几个类继承图：
{% plantuml %}
!theme plain
top to bottom direction
skinparam linetype ortho

class AbstractPipeline<E_IN, E_OUT, S>
interface AutoCloseable << interface >>
interface BaseStream<T, S> << interface >>
class PipelineHelper<P_OUT>
class ReferencePipeline<P_IN, P_OUT>
interface Stream<T> << interface >>
class Head<E_IN, E_OUT>
class StatelessOp<E_IN, E_OUT>
class StatefulOp<E_IN, E_OUT>
interface Sink<T> << interface >>
interface Consumer<T> << interface >>

AbstractPipeline   -[#008200,dashed]-^  BaseStream
AbstractPipeline   -[#000082,plain]-^  PipelineHelper
BaseStream         -[#008200,plain]-^  AutoCloseable
ReferencePipeline  -[#000082,plain]-^  AbstractPipeline
ReferencePipeline  -[#008200,dashed]-^  Stream
Stream             -[#008200,plain]-^  BaseStream
Head  -[#000082,plain]-^  ReferencePipeline
StatelessOp  -[#000082,plain]-^  ReferencePipeline
StatefulOp  -[#000082,plain]-^  ReferencePipeline
Sink             -[#008200,plain]-^  Consumer
{% endplantuml %}
其中：
- BaseStream 和 Stream 为最顶端的接口类。
  - BaseStream 主要定义了流的基本接口方法，例如，spliterator、isParallel 等；
  - Stream 则定义了一些流的常用操作方法，例如，map、filter 等
- ReferencePipeline 是一个结构类，他通过定义内部类组装了各种操作流。他定义了 Head、StatelessOp、StatefulOp 三个内部类，实现了 BaseStream 与 Stream 的接口方法
- **Sink 接口是定义每个 Stream 操作之间关系的协议**，他包含 begin()、end()、cancellationRequested()、accept() 四个方法
- **ReferencePipeline 最终会将整个 Stream 流操作组装成一个调用链**，而这条调用链上的各个 Stream 操作的上下关系就是通过 Sink 接口协议来定义实现的

### Stream的操作叠加
管道结构通常是由 ReferencePipeline 类实现的，前面的类图Stream 包结构时，解释过 ReferencePipeline 包含了 Head、StatelessOp、StatefulOp 三种内部类。
这几个内部类分别包含的作用如下表所示：
| 内部类      |  作用           |
| ------------- |:-------------:|
| Head      |   主要用来定义数据源操作，在我们初次调用 names.stream() 方法时，会初次加载 Head 对象，此时为加载数据源操作 |
| StatelessOp      | 无状态中间操作 |
| StatefulOp      | 有状态中间操作 |
| Stage      | 在 JDK 中每次的中断操作会以使用阶段（Stage）命名 |
| AbstractPipeline      | 用于生成一个中间操作 Stage 链表 |
其中：中间操作之后，此时的 Stage 并没有执行，而是通过 AbstractPipeline 生成了一个中间操作 Stage 链表；
当我们调用终结操作时，会生成一个最终的 Stage，通过这个 Stage 触发之前的中间操作，从最后一个 Stage 开始，递归产生一个 Sink 链。

### Stream执行流程
一个简单的例子：
```java
public class ExecutionStep {

    public static void main(String[] args) {
        List<Student> students = GenerateTestData.genListData();
        log.info("Before test students list:{}", students);
        //
        students.stream()
                .filter(student -> student.getHigh() > 160)
                .mapToInt(Student::getAge)
                .max()
                .ifPresent(System.out::println);

    }

}

```
你以为的流程可能是：
- 首先遍历一次集合，得到所有学生的年龄；
- 然后遍历一次 filter 得到的筛选后的集合；
- 获取年龄集合；
- 最后再从年龄集合中找到最大的年龄；
- 如果存在就打印到控制台
但是实际的流程是：
1. students.stream() 方法将会调用集合类基础接口 Collection 的 Stream 方法；
2. Stream 方法就会调用 StreamSupport 类的 Stream 方法，方法中**初始化了一个 ReferencePipeline 的 Head 内部类**对象；
3. 再调用 filter 和 map 方法，**这两个方法都是无状态的中间操作**，所以执行 filter 和 map 操作时，并没有进行任何的操作，而是分别**创建了一个 Stage 来标识**用户的每一次操作；
4. new StatelessOp 将会调用父类 AbstractPipeline 的构造函数，这个构造函数将前后的 Stage 联系起来，**生成一个 Stage 链表**：
5. 当执行 max 方法时，会调用 ReferencePipeline 的 max 方法，此时由于 max 方法是终结操作，所以会创建一个 TerminalOp 操作，同时创建一个 ReducingSink，并且将操作封装在 Sink 类中。
6. Java8 中的 Spliterator 的 forEachRemaining 会迭代集合，每迭代一次，都会执行一次 filter 操作，如果 filter 操作通过，就会触发 map 操作，然后将结果放入到临时数组 object 中，再进行下一次的迭代。完成中间操作后，就会触发终结操作 max。

#### 看一看内部相关代码
1. students.stream()方法将会调用集合类基础接口 Collection 的 Stream 方法
关联源码如下：
```java
default Stream<E> stream() {
        // spliterator()方法后文介绍，这里知道这个方法
        return StreamSupport.stream(spliterator(), false);
    }
```
StreamSupport.stream方法：
```java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        // 这里内部调用到AbstractPipeline构造器函数进行属性赋值
        return new ReferencePipeline.Head<>(spliterator,
                                            StreamOpFlag.fromCharacteristics(spliterator),
                                            parallel);
    }
```



#### 关于Spliterator