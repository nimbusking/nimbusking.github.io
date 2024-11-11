---
title: 细磕Spring点滴
abbrlink: f5eb228d
date: 2024-11-07 08:33:54
updated: 2024-11-11 21:36:25
tags:
  - Spring全家桶
  - Java
categories: Spring
---

## Spring概述
一句话的就是：Java体系的里面的另一座高山，不是在于底层多么的硬核，而是在于这个框架体系非常庞大，以至于想要一下次吃透很难。
<!-- more -->

### 前置说明
- GitHub上fork了一个5.1.15版本的Spring框架源码（可以参考我的：https://github.com/nimbusking/spring-framework）
- 需要具备一点Gradle的知识：Spring是基于Gradle构建
- Spring调试模块均在项目里面的：spring-nimbusk-为开头的子模块中
- 文章提供的类继承图是通过Idea导出的plantuml图，为了方便更好直观的嵌入我的博客里面，我直接拿来用了。原因是：直接截图没法灵活的修改继承树中的内容，比如加注释、关联注释什么的
  很遗憾，plantuml目前不支持从底部到顶部的那种常规的继承图，**而是倒过来的，**即最顶层的实现不是在底部，而是在顶部，继承树从上往下看。看的时候注意区分一下。

## Spring IOC

### Spring上下文生命周期

### SpringBean生命周期
大体分为5个大的步骤，如下图所示：
![SpringBean生命周期](f5eb228d/SpringBean生命周期.jpg)

#### BeanDefinition定义

BeanDefinition 是 Spring Framework 中定义 Bean 的配置元信息接口，主要包含一下信息：
- Bean 的类名
- Bean 行为配置类，如作用域、自动绑定模式、生命周期回调等
- 其他 Bean 引用，又可称作合作者或者依赖
- 配置设置，比如 Bean 属性
看一下相关的类图：
{% plantuml %}
!theme plain
top to bottom direction
skinparam linetype ortho

class AbstractBeanDefinition
interface AnnotatedBeanDefinition << interface >>
class AnnotatedGenericBeanDefinition
interface AttributeAccessor << interface >>
interface BeanDefinition << interface >>
interface BeanMetadataElement << interface >>
class ChildBeanDefinition
class GenericBeanDefinition
class RootBeanDefinition
class ScannedGenericBeanDefinition

AbstractBeanDefinition          -[#008200,dashed]-^  AttributeAccessor              
AbstractBeanDefinition          -[#008200,dashed]-^  BeanDefinition                 
AbstractBeanDefinition          -[#008200,dashed]-^  BeanMetadataElement            
AnnotatedBeanDefinition         -[#008200,plain]-^  BeanDefinition                 
AnnotatedGenericBeanDefinition  -[#008200,dashed]-^  AnnotatedBeanDefinition        
AnnotatedGenericBeanDefinition  -[#000082,plain]-^  GenericBeanDefinition          
BeanDefinition                  -[#008200,plain]-^  AttributeAccessor              
BeanDefinition                  -[#008200,plain]-^  BeanMetadataElement            
ChildBeanDefinition             -[#000082,plain]-^  AbstractBeanDefinition         
GenericBeanDefinition           -[#000082,plain]-^  AbstractBeanDefinition         
RootBeanDefinition              -[#000082,plain]-^  AbstractBeanDefinition         
ScannedGenericBeanDefinition    -[#008200,dashed]-^  AnnotatedBeanDefinition        
ScannedGenericBeanDefinition    -[#000082,plain]-^  GenericBeanDefinition          
{% endplantuml %}
这里面总结来看，按照不同的使用场景可以划分为：
- 基于XML定义的bean：GenericBeanDefinition
- @Component 以及派生注解定义 Bean：ScannedGenericBeanDefinition
- 借助于 @Import 导入 Bean：AnnotatedGenericBeanDefinition
- @Bean 定义的方法：ConfigurationClassBeanDefinition
- 在 Spring BeanFactory 初始化 Bean 的前阶段，会根据 BeanDefinition 生成一个合并后的 RootBeanDefinition 对象

### 基于XML的Bean的解析
一图总结：
![XML方式加载BeanDefinition流程](f5eb228d/XML方式加载.jpg)
#### component:scan标签
通过ContextNamespaceHandler注册了很多BeanDefinitionParser类
```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

  @Override
  public void init() {
    registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
    registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
    registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
    registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
    registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
    registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
  }
}
```

### 基于注解的Bean的解析
与XML方式基本类似，大致总结就是：
1. 创建AnnotationConfigApplicationContext，来执行解析配置，这里面有俩分支，这俩基本类似，只是起始的步骤不大一样而已：
    - 通过ClassPathBeanDefinitionScanner 会去扫描到包路径下所有的 .class 文件。
    - 通过AnnotatedBeanDefinitionReader去读指定class及其派生的类信息，从给定的组件类中派生 bean 定义并自动刷新上下文。
2. 通过 ASM（Java 字节码操作和分析框架）获取 .class 对应类的所有元信息
3. 根据元信息判断是否符合条件（带有 @Component 注解或其派生注解），符合条件则根据这个类的元信息生成一个 BeanDefinition 进行注册

### Bean加载过程（IOC核心）
以AnnotationConfigApplicationContext为示例，下面将梳理一下处理过程
#### this();
初始化准备工作，这里面主要分为三个步骤：
1. 初始化核心Bean工厂：this.beanFactory = new DefaultListableBeanFactory();
2. 注册内置的后置处理器：this.reader = new AnnotatedBeanDefinitionReader(this);
3. 注册包扫描器：this.scanner = new ClassPathBeanDefinitionScanner(this);
具体步骤如下图所示：
![AnnotationConfigApplicationContext初始化准备](f5eb228d/AnnotationConfigApplicationContext初始化准备.jpg)

##### 关于DefaultListableBeanFactory
IOC容器的核心的Bean工厂实现
{% plantuml %}
!theme plain
top to bottom direction
skinparam linetype ortho

class AbstractApplicationContext
class AbstractAutowireCapableBeanFactory
class AbstractBeanFactory
interface AliasRegistry << interface >>
interface ApplicationContext << interface >>
interface AutowireCapableBeanFactory << interface >>
interface BeanDefinitionRegistry << interface >>
interface BeanFactory << interface >>
interface ConfigurableBeanFactory << interface >>
interface ConfigurableListableBeanFactory << interface >>
note left:这个接口下文要看
class DefaultListableBeanFactory
class DefaultSingletonBeanRegistry
class FactoryBeanRegistrySupport
interface HierarchicalBeanFactory << interface >>
interface ListableBeanFactory << interface >>
class SimpleAliasRegistry
interface SingletonBeanRegistry << interface >>

AbstractApplicationContext          -[#008200,dashed]-^  ApplicationContext                 
AbstractAutowireCapableBeanFactory  -[#000082,plain]-^  AbstractBeanFactory                
AbstractAutowireCapableBeanFactory  -[#008200,dashed]-^  AutowireCapableBeanFactory         
AbstractBeanFactory                 -[#008200,dashed]-^  ConfigurableBeanFactory            
AbstractBeanFactory                 -[#000082,plain]-^  FactoryBeanRegistrySupport         
ApplicationContext                  -[#008200,plain]-^  HierarchicalBeanFactory            
ApplicationContext                  -[#008200,plain]-^  ListableBeanFactory                
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
DefaultSingletonBeanRegistry        -[#000082,plain]-^  SimpleAliasRegistry                
DefaultSingletonBeanRegistry        -[#008200,dashed]-^  SingletonBeanRegistry              
FactoryBeanRegistrySupport          -[#000082,plain]-^  DefaultSingletonBeanRegistry       
HierarchicalBeanFactory             -[#008200,plain]-^  BeanFactory                        
ListableBeanFactory                 -[#008200,plain]-^  BeanFactory                        
SimpleAliasRegistry                 -[#008200,dashed]-^  AliasRegistry                     
{% endplantuml %}

##### 关于BeanFactoryPostProcessor
**这里面特别需要提一个，就是上小节中第2步中的后置处理器，IOC框架初始化的时候大量使用：**
```java
/***
 * 
允许对应用程序上下文的 bean 定义进行自定义修改，从而调整上下文的基础 bean 工厂的 bean 属性值。
应用程序上下文可以在其 bean 定义中自动检测 BeanFactoryPostProcessor bean，并在创建任何其他 bean 之前应用它们。
对于针对系统管理员的自定义配置文件非常有用，这些文件会覆盖在应用程序上下文中配置的 Bean 属性。
有关满足此类配置需求的开箱即用的解决方案，请参见PropertyResourceConfigurer及其具体实现。
BeanFactoryPostProcessor可以与 bean 定义交互并修改 bean，但不能与 bean 实例进行交互。这样做可能会导致 bean 过早实例化，违反容器并导致意外的副作用。如果需要 bean 实例交互，请考虑改为实现 BeanPostProcessor 。
 */
@FunctionalInterface
public interface BeanFactoryPostProcessor {

  /**
   * 在标准初始化后修改应用程序上下文的内部 Bean 工厂。所有 bean 定义都已加载，但尚未实例化任何 bean。这允许覆盖或添加属性，甚至允许预先初始化 bean。
   */
  void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

#### register(componentClasses);
主要外层有个循环，调用到DefaultListableBeanFactory#registerBeanDefinition方法里注册BeanDefinition
#### refresh();
核心的是调用refresh方法，这个方法脉络非常清楚，但调用链非常深，让我们一个个按重点看看。
PS：**下文中，重要的核心方法，会配上相关的时序图。**
```java
@Override
  public void refresh() throws BeansException, IllegalStateException {
    // <1> 来个锁，不然 refresh() 还没结束，你又来个启动或销毁容器的操作，那不就乱套了嘛
    synchronized (this.startupShutdownMonitor) {
      
      // <2> 刷新上下文环境的准备工作，记录下容器的启动时间、标记'已启动'状态、对上下文环境属性进行校验
      prepareRefresh();

      // <3> 创建并初始化一个 BeanFactory 对象 `beanFactory`，会加载出对应的 BeanDefinition 元信息们
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // <4> 为 `beanFactory` 进行一些准备工作，例如添加几个 BeanPostProcessor，手动注册几个特殊的 Bean
      prepareBeanFactory(beanFactory);

      try {
        // <5> 对 `beanFactory` 在进行一些后期的加工，交由子类进行扩展
        postProcessBeanFactory(beanFactory);

        // <6> 执行 BeanFactoryPostProcessor 处理器，包含初始化 BeanDefinitionRegistryPostProcessor 处理器
        invokeBeanFactoryPostProcessors(beanFactory);

        // <7> 对 BeanPostProcessor 处理器进行初始化，并添加至 BeanFactory 中
        registerBeanPostProcessors(beanFactory);

        // <8> 设置上下文的 MessageSource 对象
        initMessageSource();

        // <9> 设置上下文的 ApplicationEventMulticaster 对象，上下文事件广播器
        initApplicationEventMulticaster();

        // <10> 刷新上下文时再进行一些初始化工作，交由子类进行扩展
        onRefresh();

        // <11> 将所有 ApplicationListener 监听器添加至 `applicationEventMulticaster` 事件广播器，如果已有事件则进行广播
        registerListeners();

        // <12> 设置 ConversionService 类型转换器，**初始化**所有还未初始化的 Bean（不是抽象、不是懒加载方式，是单例模式的）
        finishBeanFactoryInitialization(beanFactory);

        // <13> 刷新上下文的最后一步工作，会发布 ContextRefreshedEvent 上下文完成刷新事件
        finishRefresh();
      }
      // <14> 如果上面过程出现 BeansException 异常
      catch (BeansException ex) {
        if (logger.isWarnEnabled()) {
          logger.warn("Exception encountered during context initialization - " +
              "cancelling refresh attempt: " + ex);
        }

        // <14.1> “销毁” 已注册的单例 Bean
        destroyBeans();

        // <14.2> 设置上下文的 `active` 状态为 `false`
        cancelRefresh(ex);

        // <14.3> 抛出异常
        throw ex;
      }
      // <15> `finally` 代码块
      finally {
        // Reset common introspection caches in Spring's core, since we
        // might not ever need metadata for singleton beans anymore...
        // 清除相关缓存，例如通过反射机制缓存的 Method 和 Field 对象，缓存的注解元数据，缓存的泛型类型对象，缓存的类加载器
        resetCommonCaches();
      }
    }
  }
```
##### prepareRefresh()
刷新上下文环境的准备工作，记录下容器的启动时间、标记'已启动'状态、对上下文环境属性进行校验
过程比较简单，就不具体贴时序图了
##### obtainFreshBeanFactory()
这个过程，主要就是创建：ConfigurableListableBeanFactory：一个BeanFactory的子类实现配置接口由大多数可列出的 bean 工厂实现。除了 ConfigurableBeanFactory之外，**它还提供了分析和修改 bean 定义以及预实例化单例的工具**。
**实际上是创建的DefaultListableBeanFactory，上文有说明。**

{% plantuml %}
!theme plain
top to bottom direction
skinparam linetype ortho

interface AutowireCapableBeanFactory << interface >>
interface BeanFactory << interface >>
interface ConfigurableBeanFactory << interface >>
interface ConfigurableListableBeanFactory << interface >>
interface HierarchicalBeanFactory << interface >>
interface ListableBeanFactory << interface >>
interface SingletonBeanRegistry << interface >>

AutowireCapableBeanFactory       -[#008200,plain]-^  BeanFactory                     
ConfigurableBeanFactory          -[#008200,plain]-^  HierarchicalBeanFactory         
ConfigurableBeanFactory          -[#008200,plain]-^  SingletonBeanRegistry           
ConfigurableListableBeanFactory  -[#008200,plain]-^  AutowireCapableBeanFactory      
ConfigurableListableBeanFactory  -[#008200,plain]-^  ConfigurableBeanFactory         
ConfigurableListableBeanFactory  -[#008200,plain]-^  ListableBeanFactory             
HierarchicalBeanFactory          -[#008200,plain]-^  BeanFactory                     
ListableBeanFactory              -[#008200,plain]-^  BeanFactory     
{% endplantuml %}

##### postProcessBeanFactory(beanFactory)
做一些准备工作，为 `beanFactory` 进行一些准备工作，例如添加几个 BeanPostProcessor，手动注册几个特殊的 Bean，没什么特别需要标注的地方：
{% plantuml %}
participant Actor
note over AbstractApplicationContext:这个过程主要是向ConfigurableListableBeanFactory\n做一些初始化及Bean工厂中注册BeanPostProcessor
Actor -> AbstractApplicationContext : prepareBeanFactory

activate AbstractApplicationContext
AbstractApplicationContext -> AbstractApplicationContext:setBeanClassLoader设置ClassLoder

create StandardBeanExpressionResolver
AbstractApplicationContext -> StandardBeanExpressionResolver : new
note right:设置 BeanExpressionResolver 表达式语言处理器
activate StandardBeanExpressionResolver
StandardBeanExpressionResolver --> AbstractApplicationContext
deactivate StandardBeanExpressionResolver

create ResourceEditorRegistrar
AbstractApplicationContext -> ResourceEditorRegistrar : new 添加一个默认的 PropertyEditorRegistrar 属性编辑器
activate ResourceEditorRegistrar
ResourceEditorRegistrar --> AbstractApplicationContext
deactivate ResourceEditorRegistrar
AbstractApplicationContext -> ConfigurableBeanFactory : addPropertyEditorRegistrar
activate ConfigurableBeanFactory
ConfigurableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableBeanFactory

create ApplicationContextAwareProcessor
AbstractApplicationContext -> ApplicationContextAwareProcessor : new 添加一个 BeanPostProcessor 处理器，ApplicationContextAwareProcessor，初始化 Bean 的前置处理
note left:这个 BeanPostProcessor 其实是对几种 Aware 接口的处理，调用其 setXxx 方法
activate ApplicationContextAwareProcessor
ApplicationContextAwareProcessor --> AbstractApplicationContext
deactivate ApplicationContextAwareProcessor
AbstractApplicationContext -> ConfigurableBeanFactory : addBeanPostProcessor
activate ConfigurableBeanFactory
ConfigurableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableBeanFactory

create ApplicationListenerDetector
AbstractApplicationContext -> ApplicationListenerDetector : new 添加一个 BeanPostProcessor 处理器，ApplicationListenerDetector，用于装饰监听器
activate ApplicationListenerDetector
ApplicationListenerDetector --> AbstractApplicationContext
deactivate ApplicationListenerDetector
AbstractApplicationContext -> ConfigurableBeanFactory : addBeanPostProcessor
activate ConfigurableBeanFactory
ConfigurableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableBeanFactory
alt 存在
AbstractApplicationContext -> BeanFactory : containsBean
activate BeanFactory
BeanFactory --> AbstractApplicationContext
deactivate BeanFactory
create LoadTimeWeaverAwareProcessor
AbstractApplicationContext -> LoadTimeWeaverAwareProcessor : new 增加对 AspectJ 的支持，AOP 相关的PostProcessor
activate LoadTimeWeaverAwareProcessor
LoadTimeWeaverAwareProcessor --> AbstractApplicationContext
deactivate LoadTimeWeaverAwareProcessor
AbstractApplicationContext -> ConfigurableBeanFactory : addBeanPostProcessor
end
== 下面几个就是：注册几个 ApplicationContext 上下文默认的 Bean 对象 ==
activate ConfigurableBeanFactory
ConfigurableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableBeanFactory
AbstractApplicationContext -> HierarchicalBeanFactory : containsLocalBean
activate HierarchicalBeanFactory
HierarchicalBeanFactory --> AbstractApplicationContext
deactivate HierarchicalBeanFactory
AbstractApplicationContext -> SingletonBeanRegistry : registerSingleton 注册的：ConfigurableEnvironment
activate SingletonBeanRegistry
SingletonBeanRegistry --> AbstractApplicationContext
deactivate SingletonBeanRegistry
AbstractApplicationContext -> HierarchicalBeanFactory : containsLocalBean
activate HierarchicalBeanFactory
HierarchicalBeanFactory --> AbstractApplicationContext
deactivate HierarchicalBeanFactory
AbstractApplicationContext -> SingletonBeanRegistry : registerSingleton 注册的一个系统Properties属性的Map
activate SingletonBeanRegistry
SingletonBeanRegistry --> AbstractApplicationContext
deactivate SingletonBeanRegistry
AbstractApplicationContext -> HierarchicalBeanFactory : containsLocalBean
activate HierarchicalBeanFactory
HierarchicalBeanFactory --> AbstractApplicationContext
deactivate HierarchicalBeanFactory
AbstractApplicationContext -> SingletonBeanRegistry : registerSingleton 注册一个系统环境变量的Map
activate SingletonBeanRegistry
SingletonBeanRegistry --> AbstractApplicationContext
deactivate SingletonBeanRegistry
return
{% endplantuml %}

##### invokeBeanFactoryPostProcessors(beanFactory)
执行 BeanFactoryPostProcessor 处理器，包含初始化 BeanDefinitionRegistryPostProcessor 处理器。
这里面用到一个委托Delegate设计模式类：PostProcessorRegistrationDelegate，这个类用于 AbstractApplicationContext 的后处理器处理，这里面有俩核心的public方法：
![PostProcessorRegistrationDelegate结构](f5eb228d/structure_of_PostProcessorRegistrationDelegate.jpg)
先看invokeBeanFactoryPostProcessors方法，另外一个在下面介绍：
- 方法签名：```public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)```
具体执行步骤如下：
先进行前置判断：beanFactory instanceof BeanDefinitionRegistry
1. (true) 执行当前 Spring 应用上下文和底层 BeanFactory 容器中的 BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor 们的处理
    1. 先遍历当前 Spring 应用上下文中的 `beanFactoryPostProcessors`，如果是 BeanDefinitionRegistryPostProcessor 类型则进行处理
    2. 获取底层 BeanFactory 容器中所有 BeanDefinitionRegistryPostProcessor 类型的 Bean 们，**遍历处理：PriorityOrdered类型**
    3. 获取底层 BeanFactory 容器中所有 BeanDefinitionRegistryPostProcessor 类型的 Bean 们，**遍历处理：PriorityOrdered类型**
    4. 处理上面还没处理的BeanDefinitionRegistryPostProcessor
    5. 调用接下来执行它们的 postProcessBeanFactory(beanFactory) 方法
2. (false) **执行当前 Spring 应用上下文中的 BeanFactoryPostProcessor 处理器的 postProcessBeanFactory(beanFactory) 方法**，下面的执行就是执行postProcessBeanFactory方法。
3. 获取底层 BeanFactory 容器中所有 BeanFactoryPostProcessor 类型的 Beans
4. 循环判断，在实现 PriorityOrdered 的 BeanFactoryPostProcessor 之间分开用单独List存：这个步骤为了后续处理一些排序的BeanFactoryPostProcessor
    1. 处理PriorityOrdered 类型的 BeanFactoryPostProcessor 对象，缓存并注册初始化
    2. 处理Ordered 类型的 BeanFactoryPostProcessor 对象，缓存并注册初始化
    3. 处理nonOrdered 的 BeanFactoryPostProcessor 对象， 缓存并注册初始化
5. 清除一些元数据缓存

##### registerBeanPostProcessors(beanFactory)
跟上面是调用的同一Delegate对象，主要处理：对 BeanPostProcessor 处理器进行初始化，并添加至 BeanFactory 中
**这个里面初始化就是调用BeanFacory的getBean方法**。
这个getBean处理了常见的循环依赖问题，后文中单独作一小节描述。

##### initMessageSource()
设置上下文的 MessageSource 对象，常见的类如国际化相关的。

##### initApplicationEventMulticaster()
设置上下文的 ApplicationEventMulticaster 对象，上下文事件广播器

##### onRefresh()
刷新上下文时再进行一些初始化工作，交由子类进行扩展

##### registerListeners()
将所有 ApplicationListener 监听器添加至 `applicationEventMulticaster` 事件广播器，如果已有事件则进行广播

##### finishBeanFactoryInitialization(beanFactory)
置 ConversionService 类型转换器，**初始化**所有还未初始化的 Bean（不是抽象、不是懒加载方式，是单例模式的）。
**这里面基本上就是初始化我们自己定义的相关的Bean对象了**

##### finishRefresh()
刷新上下文的最后一步工作，会发布 ContextRefreshedEvent 上下文完成刷新事件

### Bean加载之getBean初始化
这个过程相对比较复杂一点，需要处理的东西有点多：
TODO

### 一些杂项问题
#### BeanFactory与FactoryBean
BeanFactory是spring顶层接口，使用了简单工厂模式，负责生产Bean实例
FactoryBean是对普通Bean对象增强的一个接口，实例化这种对象的时候，是会调用到实现的重新方法getObject()方法中初始化对象的。
#### 构造器注入和Setter注入
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