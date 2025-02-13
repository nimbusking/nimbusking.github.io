---
title: 设计模式
abbrlink: a0df4ec2
date: 2019-08-25 12:07:10
tags:
    - 软件工程
    - 设计模式
updated: 2025-02-13 21:42:52
categories: 设计模式
---

## 背景
废话少说，面试常问的直接端上来

## 一些知识点

### 一、单例模式
1. 实现方式与线程安全
饿汉式：类加载时初始化实例，线程安全但可能浪费资源
```java
public class Singleton {
   private static final Singleton INSTANCE = new Singleton();
   private Singleton() {}
   public static Singleton getInstance() { return INSTANCE; }
}
```

- **双重检查锁**：延迟加载且线程安全，需用`volatile`防止指令重排序 [1]()[3]()[6]()  
  ```java 
     public class Singleton {
         private volatile static Singleton instance;
         private Singleton() {}
         public static Singleton getInstance() {
             if (instance == null) {
                 synchronized (Singleton.class)  {
                     if (instance == null) instance = new Singleton();
                 }
             }
             return instance;
         }
     }
     ```
枚举实现：Java5+推荐方式，天然线程安全且防反射攻击

2. 应用场景
频繁创建/销毁的对象（如线程池、数据库连接池）
需要全局唯一访问点的配置管理类

---

## 二、工厂模式
### 1. 分类与核心思想
简单工厂：通过参数动态创建对象，但违反开闭原则
public class CoffeeFactory {
   public static Coffee createCoffee(String type) {
       if ("Latte".equals(type)) return new Latte();
       else if ("Americano".equals(type)) return new Americano();
       throw new IllegalArgumentException("Invalid type");
   }
}
- **工厂方法**：通过子类决定实例化对象，符合开闭原则 [3]()[8]()  
- **抽象工厂**：创建产品族，如JDBC连接工厂生产Statement/ResultSet [4]()[8]()

### 2. JDK应用示例
- `Boolean.valueOf()` 使用静态工厂方法 [5]()[8]()  
- Spring的`BeanFactory`通过配置文件创建Bean [4]()[8]()

---

## 三、代理模式 
### 1. 类型与实现 
- **静态代理**：手动编写代理类，如AOP切面 [1]()[4]()  
- **动态代理**：
 - **JDK动态代理**：基于接口，通过`InvocationHandler`实现（如Motan RPC） [1]()[4]()  
 - **CGLIB代理**：基于继承，无需接口（如Spring AOP） [3]()[6]()

### 2. 应用场景
- 权限控制、日志记录、RPC调用封装 [1]()[4]()

---

## 四、观察者模式 
### 1. 结构与实现 
- **Subject**（被观察者）：维护观察者列表，通知状态变化 [2]()[6]()  
- **Observer**（观察者）：定义更新接口，如Swing事件监听 [5]()[6]()  
- **应用示例**：gRPC流式处理、Spring事件机制 [1]()[4]()

### 2. 优缺点
- **优点**：解耦观察者与被观察者，支持广播通信 [6]()  
- **缺点**：通知顺序不可控，可能引发循环依赖 [6]()

---

## 五、适配器模式 
### 1. 类型与场景 
- **类适配器**：继承被适配类（如SLF4J适配Log4j） [1]()[3]()  
- **对象适配器**：组合被适配对象（如Spring MVC的`HandlerAdapter`） [4]()[6]()

### 2. 典型应用
- JDBC驱动通过适配不同数据库接口 [6]()  
- Spring AOP的`AdvisorAdapter`统一拦截器调用链 [3]()

---

## 六、装饰模式 
### 1. 特点与实现 
- **动态扩展对象功能**：通过组合替代继承 [2]()[6]()  
- **JDK示例**：Java IO的`BufferedReader`装饰`FileReader` [5]()[6]()  
    ```java 
     Reader reader = new BufferedReader(new FileReader("file.txt")); 
    ```
### 七、模板方法模式
1. 核心思想
定义算法骨架：子类重写特定步骤（如JdbcTemplate）
```java
public abstract class AbstractTemplate {
   public void process() {
       step1();
       step2(); // 子类实现 
       step3();
   }
   private void step1() { /*...*/ }
   protected abstract void step2();
   private void step3() { /*...*/ }
}
```

---

## 八、高频进阶问题
1. **设计模式分类**：  
  - 创建型（5种）：单例、工厂、建造者等 
  - 结构型（7种）：代理、适配器、装饰者等 
  - 行为型（11种）：观察者、策略、模板方法等 

2. **设计原则**：  
  - 开闭原则、里氏替换原则、依赖倒置原则等 

3. **框架中的应用**：  
  - Spring的单例Bean、AOP代理 
  - MyBatis的Executor模板方法

---

