---
title: Java系列
mathjax: false
tags:
  - Core Java
  - JVM
  - Concurrent
updated: 2025-02-14 09:23:46
categories: Java系列
comments: false
abbrlink: c2b92334
date: 2018-03-14 08:16:06
---
Java系列目录：
- Core Java
    + 集合类
- JVM
- Java多线程
- JavaFX


## 相关知识点


以下是关于 **Java 核心** 的高频面试题总结，涵盖 **Java 基础**、**集合类**、**多线程**、**JVM** 等核心知识点，帮助快速复习关键概念：

---

### **一、Java 基础**
1. **`==` 和 `equals()` 的区别？**  
   - `==`：比较对象内存地址是否相同（基本类型比较值）。  
   - `equals()`：默认比较地址，可重写用于逻辑相等性（如 `String` 重写比较内容）。  

2. **String、StringBuffer、StringBuilder 的区别？**  
   - **String**：不可变，线程安全，频繁修改性能差。  
   - **StringBuffer**：可变，线程安全（方法加 `synchronized`），性能较低。  
   - **StringBuilder**：可变，非线程安全，性能高（推荐单线程场景）。  

3. **final 关键字的作用？**  
   - 修饰类：不可继承。  
   - 修饰方法：不可重写。  
   - 修饰变量：常量（基本类型值不可变，引用类型地址不可变）。  

4. **接口 vs 抽象类？**  
   - **接口**：多继承，默认方法（Java 8+），无构造方法，定义行为规范。  
   - **抽象类**：单继承，可以有构造方法、成员变量，定义模板和部分实现。  

5. **重载（Overload）和重写（Override）的区别？**  
   - **重载**：同一类中，方法名相同、参数列表不同（返回值无关）。  
   - **重写**：子类继承父类，方法名、参数、返回值相同，访问权限≥父类，不抛更宽泛异常。  

6. **Java 异常体系？Error 和 Exception 的区别？**  
   - **Throwable** 是根类，子类为 `Error`（JVM 严重错误，如 `OutOfMemoryError`）和 `Exception`（程序可处理异常）。  
   - `Exception` 分为 `Checked Exception`（编译时检查，如 `IOException`）和 `Unchecked Exception`（运行时异常，如 `NullPointerException`）。  

---

### **二、集合框架（Collections）**
7. **ArrayList 和 LinkedList 的区别？**  
   - **ArrayList**：基于动态数组，随机访问快（O(1)），增删慢（需移动元素）。  
   - **LinkedList**：基于双向链表，增删快（O(1)），随机访问慢（O(n)）。  

8. **HashMap 的工作原理？**  
   - 基于数组+链表/红黑树（JDK 8+），通过 `key` 的 `hashCode()` 计算桶位置。  
   - 解决哈希冲突：链表法（同一桶中形成链表，长度≥8时转红黑树）。  
   - 扩容：默认负载因子 0.75，容量翻倍，重新哈希。  

9. **ConcurrentHashMap 如何保证线程安全？**  
   - JDK 7：分段锁（Segment），每个段独立加锁。  
   - JDK 8：CAS + synchronized 锁单个桶（链表头/红黑树根节点）。  

10. **HashSet 的实现原理？**  
    - 内部基于 `HashMap` 实现，元素作为 `key` 存储，值用固定 `PRESENT` 对象填充。  

11. **Iterator 的 Fail-Fast 和 Fail-Safe 机制？**  
    - **Fail-Fast**：直接遍历集合时修改结构会抛 `ConcurrentModificationException`（如 ArrayList）。  
    - **Fail-Safe**：遍历集合的副本，允许修改原集合（如 `CopyOnWriteArrayList`）。  

---

### **三、多线程与并发**
12. **线程的创建方式？**  
    - 继承 `Thread` 类，重写 `run()`。  
    - 实现 `Runnable` 接口，传递给 `Thread`。  
    - 实现 `Callable` 接口，配合 `FutureTask`（可获取返回值）。  

13. **synchronized 和 Lock 的区别？**  
    - **synchronized**：JVM 级别锁，自动释放锁，不可中断等待。  
    - **Lock**：API 级别锁（如 `ReentrantLock`），需手动释放，支持超时、中断、公平锁。  

14. **volatile 关键字的作用？**  
    - 保证变量可见性（直接读写主内存），禁止指令重排序，但不保证原子性。  

15. **线程池的核心参数？**  
    - `corePoolSize`：核心线程数。  
    - `maximumPoolSize`：最大线程数。  
    - `keepAliveTime`：非核心线程空闲存活时间。  
    - `workQueue`：任务队列（如 `ArrayBlockingQueue`）。  
    - `ThreadFactory`：线程创建工厂。  
    - `RejectedExecutionHandler`：拒绝策略（如 AbortPolicy、CallerRunsPolicy）。  

---

### **四、JVM 核心**
16. **JVM 内存模型（运行时数据区）？**  
    - **堆（Heap）**：对象实例和数组，线程共享。  
    - **栈（Stack）**：每个线程私有，存放局部变量、方法调用栈帧。  
    - **方法区（Method Area）**：类信息、常量、静态变量（JDK 8+ 由元空间实现）。  
    - **程序计数器**：当前线程执行指令的地址。  

17. **垃圾回收算法有哪些？**  
    - **标记-清除**：标记无用对象并清除，产生内存碎片。  
    - **复制算法**：将存活对象复制到另一块内存（适用于新生代）。  
    - **标记-整理**：标记后整理存活对象到一端（适用于老年代）。  
    - **分代收集**：堆分为新生代（复制算法）和老年代（标记-清除/整理）。  

18. **类加载过程？**  
    - **加载**：读取类文件到内存。  
    - **验证**：确保字节码符合规范。  
    - **准备**：为静态变量分配内存并赋默认值。  
    - **解析**：将符号引用转为直接引用。  
    - **初始化**：执行静态代码块和赋值。  

---

### **五、其他核心问题**
19. **Java 反射的作用及缺点？**  
    - **作用**：运行时动态获取类信息、创建对象、调用方法。  
    - **缺点**：性能较低，破坏封装性，绕过泛型检查。  

20. **泛型中 `<? extends T>` 和 `<? super T>` 的区别？**  
    - **`<? extends T>`**：上界通配符，只能读取为 `T` 或其子类。  
    - **`<? super T>`**：下界通配符，只能写入 `T` 或其父类。  

21. **深拷贝 vs 浅拷贝？**  
    - **浅拷贝**：复制对象引用，不复制引用对象本身（如 `Object.clone()`）。  
    - **深拷贝**：递归复制所有引用对象（需自行实现）。  

---

### **六、代码示例与陷阱**
22. **String 不可变性的代码验证：**  
    ```java
    String s1 = "abc";
    String s2 = s1.toUpperCase(); // 返回新对象 "ABC"
    System.out.println(s1 == s2); // false
    ```

23. **HashMap 的线程不安全示例：**  
    多线程同时扩容可能导致链表成环，触发死循环或数据丢失。

24. **正确实现 equals 和 hashCode：**  
    ```java
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
    ```

---

### **七、扩展知识（Java 8+）**
25. **Lambda 表达式与函数式接口**  
    - 函数式接口（如 `Runnable`、`Comparator`）可使用 Lambda 简化代码。  

26. **Stream API 的核心操作**  
    - `filter()`, `map()`, `reduce()`, `collect()` 等链式处理集合数据。  

27. **Optional 类的作用**  
    - 避免 `NullPointerException`，明确处理可能为空的值。  

---

