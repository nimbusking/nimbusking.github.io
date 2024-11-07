---
title: 细磕Spring点滴
abbrlink: f5eb228d
date: 2024-11-07 08:33:54
updated: 2024-11-07 08:33:54
tags:
  - Spring全家桶
  - Java
categories: Spring
---

## Spring概述
<!-- more -->

## Spring IOC

### SpringBean生命周期
大体分为5个大的步骤，如下图所示：
![SpringBean生命周期](SpringBean生命周期.jpg)

#### Bean加载过程
{% plantuml %}
    !theme plain
top to bottom direction
skinparam linetype ortho

class AbstractAutowireCapableBeanFactory
class AbstractBeanFactory
interface AliasRegistry << interface >>
interface AutowireCapableBeanFactory << interface >>
interface BeanDefinitionRegistry << interface >>
interface BeanFactory << interface >>
interface ConfigurableBeanFactory << interface >>
interface ConfigurableListableBeanFactory << interface >>
class DefaultListableBeanFactory
class DefaultSingletonBeanRegistry
class FactoryBeanRegistrySupport
interface HierarchicalBeanFactory << interface >>
interface ListableBeanFactory << interface >>
interface Serializable << interface >>
class SimpleAliasRegistry
interface SingletonBeanRegistry << interface >>
annotation SuppressWarnings << annotation >>

AbstractAutowireCapableBeanFactory  -[#000082,plain]-^  AbstractBeanFactory                
AbstractAutowireCapableBeanFactory  -[#008200,dashed]-^  AutowireCapableBeanFactory         
AbstractBeanFactory                 -[#008200,dashed]-^  ConfigurableBeanFactory            
AbstractBeanFactory                 -[#000082,plain]-^  FactoryBeanRegistrySupport         
AutowireCapableBeanFactory          -[#008200,plain]-^  BeanFactory                        
BeanDefinitionRegistry              -[#008200,plain]-^  AliasRegistry                      
ConfigurableBeanFactory             -[#008200,plain]-^  HierarchicalBeanFactory            
ConfigurableBeanFactory             -[#008200,plain]-^  SingletonBeanRegistry              
ConfigurableListableBeanFactory     -[#008200,plain]-^  AutowireCapableBeanFactory         
ConfigurableListableBeanFactory     -[#008200,plain]-^  ConfigurableBeanFactory            
ConfigurableListableBeanFactory     -[#008200,plain]-^  ListableBeanFactory                
DefaultListableBeanFactory          -[#000082,plain]-^  AbstractAutowireCapableBeanFactory 
DefaultListableBeanFactory          -[#008200,dashed]-^  BeanDefinitionRegistry             
DefaultListableBeanFactory          -[#008200,dashed]-^  ConfigurableListableBeanFactory    
DefaultListableBeanFactory          -[#008200,dashed]-^  Serializable                       
DefaultListableBeanFactory          -[#999900,dotted]-  SuppressWarnings                   
DefaultSingletonBeanRegistry        -[#000082,plain]-^  SimpleAliasRegistry                
DefaultSingletonBeanRegistry        -[#008200,dashed]-^  SingletonBeanRegistry              
FactoryBeanRegistrySupport          -[#000082,plain]-^  DefaultSingletonBeanRegistry       
HierarchicalBeanFactory             -[#008200,plain]-^  BeanFactory                        
ListableBeanFactory                 -[#008200,plain]-^  BeanFactory                        
SimpleAliasRegistry                 -[#008200,dashed]-^  AliasRegistry  
{% endplantuml %}


### 一些杂项问题
####  构造器注入和Setter注入
**构造器注入**：通过构造器的参数注入相关依赖对象
**Setter注入**：通过 Setter 方法注入依赖对象，也可以理解为字段注入
对于两种注入方式的看法：
- **构造器注入可以避免一些尴尬的问题**，比如说状态不确定性地被修改，在初始化该对象时才会注入依赖对象，一定程度上保证了 Bean 初始化后就是不变的对象，这样对于我们的程序和维护性都会带来更多的便利；
- **构造器注入不允许出现循环依赖**，因为它要求被注入的对象都是成熟态，保证能够实例化，而 **Setter 注入或字段注入没有这样的要求**；
- **构造器注入可以保证依赖的对象能够有序的被注入**，而 Setter注入或字段注入底层是通过反射机制进行注入，无法完全保证注入的顺序；
- **如果构造器注入出现比较多的依赖**导致代码不够优雅，我们应该**考虑自身代码的设计是否存在问题**，是否需要重构代码结构。

除了上面的注入方式外，Spring 还提供了接口回调注入，通过实现 Aware 接口（例如 BeanNameAware、ApplicationContextAware）可以注入相关对象，Spring 在初始化这类 Bean 时会调用其 setXxx 方法注入对象，例如注入 beanName、ApplicationContext等。

#### Spring内建的Bean作用域
| 来源      |  说明           |
| ------------- |:-------------:|
| singleton      | 默认 Spring Bean 作用域，一个 BeanFactory 有且仅有一个实例 |
| prototype      | 原型作用域，每次依赖查找和依赖注入生成新 Bean 对象 |
| request      | 将 Spring Bean 存储在 ServletRequest 上下文中 |
| session      | 将 Spring Bean 存储在 HttpSession 中 |
| application      | 将 Spring Bean 存储在 ServletContext 中 |

#### BeanPostProcessor与BeanFactoryPostProcessor的区别
- BeanPostProcessor 提供 Spring Bean **初始化前和初始化后的生命周期回调**，允许对关心的 Bean 进行扩展，甚至是替换，其相关子类也提供 Spring Bean 生命周期中其他阶段的回调。
- BeanFactoryPostProcessor 提供 Spring **BeanFactory（底层 IoC 容器）的生命周期的回调**，用于扩展 BeanFactory（实际为 ConfigurableListableBeanFactory），BeanFactoryPostProcessor 必须由 Spring ApplicationContext 执行，BeanFactory 无法与其直接交互。

#### 如何基于 Extensible XML authoring 扩展 Spring XML 元素
对Spring XML扩展
1. **编写 XML Schema 文件（XSD 文件）**：定义 XML 结构
2. **自定义 NamespaceHandler 实现**：定义命名空间的处理器
3. **自定义 BeanDefinitionParser 实现**：绑定命名空间下不同的 XML 元素与其对应的解析器
4. **注册 XML 扩展（META-INF/spring.handlers 文件）**：命名空间与命名空间处理器的映射
5. **编写 Spring Schema 资源映射文件（META-INF/spring.schemas 文件）**：XML Schema 文件通常定义为网络的形式，在无网的情况下无法访问，所以一般在本地的也有一个 XSD 文件，可通过编写 spring.schemas 文件，将网络形式的 XSD 文件与本地的 XSD 文件进行映射，这样会优先从本地获取对应的 XSD 文件
Mybatis 对 Spring 的集成项目中的 ```<mybatis:scan /> ```标签就是这样实现的，可以参考：NamespaceHandler、MapperScannerBeanDefinitionParser、XSD 等文件

#### 简述 Spring 事件机制原理
一个典型的观察者模式的应用
在Spring中实现，首先主要有以下几个角色：
- **Spring 事件** - org.springframework.context.ApplicationEvent，实现了 java.util.EventListener 接口
- **Spring 事件监听器** - org.springframework.context.ApplicationListener，实现了 java.util.EventObject 类
- **Spring 事件发布器** - org.springframework.context.ApplicationEventPublisher
- **Spring 事件广播器** - org.springframework.context.event.ApplicationEventMulticaster
Spring有4个内建的事件：
- ContextRefreshedEvent：Spring 应用上下文就绪事件
- ContextStartedEvent：Spring 应用上下文启动事件
- ContextStoppedEvent：Spring 应用上下文停止事件
- ContextClosedEvent：Spring 应用上下文关闭事件

Spring 应用上下文就是一个 ApplicationEventPublisher 事件发布器，其内部有一个 ApplicationEventMulticaster 事件广播器（被观察者），里面保存了所有的 ApplicationListener 事件监听器（观察者）。
Spring 应用上下文发布一个事件后会通过 ApplicationEventMulticaster 事件广播器进行广播，能够处理该事件类型的 ApplicationListener 事件监听器则进行处理。

#### @EventListener的工作原理
**@EventListener 用于标注在方法上面，该方法则可以用来处理 Spring 的相关事件。**
Spring 内部有一个处理器 EventListenerMethodProcessor，它实现了 SmartInitializingSingleton 接口，在所有的 Bean（不是抽象、单例模式、不是懒加载方式）初始化后，Spring 会再次遍历所有初始化好的单例 Bean 对象时会执行该处理器对该 Bean 进行处理。在 EventListenerMethodProcessor 中会对标注了 @EventListener 注解的方法进行解析，如果符合条件则生成一个 ApplicationListener 事件监听器并注册。

#### Spring 提供的注解
大致分为5类：
1. Spring 模式注解
| Spring 注解        | 场景说明           | 起始版本  |
| ------------- |:-------------:|:-------------:|
|  @Repository      |数据仓储模式注解 | 2.0 |
|  @Component      |通用组件模式注解 | 2.5 |
|  @Service      |服务模式注解 | 2.5 |
|  @Controller      |Web 控制器模式注解 | 2.5 |
|  @Configuration      |配置类模式注解 | 3.0 |

2. 装配注解
| Spring 注解        | 场景说明           | 起始版本  |
| ------------- |:-------------:|:-------------:|
|  @ImportResource     |替换 XML 元素 <import> | 2.5 |
|  @Import      |导入 Configuration 类 | 2.5 |
|  @ComponentScan      |扫描指定 package 下标注 Spring 模式注解的类 | 3.1 |

3. 依赖注入注解
| Spring 注解        | 场景说明           | 起始版本  |
| ------------- |:-------------:|:-------------:|
| @Autowired     |Bean 依赖注入，支持多中依赖查找方式 | 2.5 |
| @Qualifier     |细粒度的 @Autowired 依赖查找 | 2.5 |

4. @Enable 模块驱动
| Spring 注解        | 场景说明           | 起始版本  |
| ------------- |:-------------:|:-------------:|
| @EnableWebMvc    |启动整个 Web MVC 模块 | 3.1 |
| @EnableTransactionManagement    |启动整个事务管理模块 | 3.1 |
| @EnableCaching    |启动整个缓存模块 | 3.1 |
| @EnableAsync    |启动整个异步处理模块 | 3.1 |

@Enable 模块驱动是以 @Enable 为前缀的注解驱动编程模型。所谓“模块”是指具备相同领域的功能组件集合，组合所形成一个独立的单元。比如 Web MVC 模块、AspectJ 代理模块、Caching（缓存）模块、JMX（Java 管理扩展）模块、Async（异步处理）模块等。
**这类注解底层原理就是通过 @Import 注解导入相关类（Configuration Class、 ImportSelector 接口实现、ImportBeanDefinitionRegistrar 接口实现），来实现引入某个模块或功能。**

5. 条件注解
| Spring 注解        | 场景说明           | 起始版本  |
| ------------- |:-------------:|:-------------:|
| @Conditional    |条件限定，引入某个 Bean | 4.0 |
| @Profile    |从 Spring 4.0 开始，@Profile 基于 @Conditional 实现，限定 Bean 的 Spring 应用环境 | 4.0 |

## SpringMVC


## SpringBoot

## SpringCloud

## Spring其它相关