---
title: 细磕Spring点滴
abbrlink: f5eb228d
date: 2024-11-07 08:33:54
updated: 2024-11-17 18:53:12
tags:
  - Spring全家桶
  - Java
categories: Spring
top: true
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

### 项目编译环境
- 系统：window 11
- Gradle：5.6.4
- IDE：IntelliJ IDEA 2024.3

## Spring IOC

### 核心知识点


---


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
看一下里面的具体的调用时序：
{% plantuml %}
participant Actor
Actor -> AbstractApplicationContext : finishBeanFactoryInitialization
note over AbstractApplicationContext:完成此上下文的 bean 工厂的初始化，初始化所有剩余的单例 bean

alt beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) && beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)
AbstractApplicationContext -> BeanFactory : getBean
note over BeanFactory:如果底层 BeanFactory 容器包含 ConversionService 类型转换器，则初始化并设置到底层 BeanFactory 容器的属性中
activate BeanFactory
BeanFactory --> AbstractApplicationContext
deactivate BeanFactory
end

alt !beanFactory.hasEmbeddedValueResolver()
AbstractApplicationContext -> ConfigurableBeanFactory : addEmbeddedValueResolver
note over ConfigurableBeanFactory:如果底层 BeanFactory 容器没有设置 StringValueResolver 解析器，则添加一个 PropertySourcesPropertyResolver 解析器
activate ConfigurableBeanFactory
ConfigurableBeanFactory -> AbstractApplicationContext : strVal ->
activate AbstractApplicationContext
AbstractApplicationContext -> PropertyResolver : resolvePlaceholders
activate PropertyResolver
PropertyResolver --> AbstractApplicationContext
deactivate PropertyResolver
AbstractApplicationContext --> ConfigurableBeanFactory
deactivate AbstractApplicationContext
ConfigurableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableBeanFactory
end

note over AbstractApplicationContext:提前初始化 LoadTimeWeaverAware 类型的 Bean，跟AOP 相关
loop weaverAwareNames
AbstractApplicationContext -> AbstractApplicationContext : getBean
activate AbstractApplicationContext
AbstractApplicationContext -> AbstractApplicationContext : assertBeanFactoryActive
activate AbstractApplicationContext
alt !this.active.get()
alt this.closed.get()
else
note right of AbstractApplicationContext : Empty
end
note right of AbstractApplicationContext : Empty
end
AbstractApplicationContext --> AbstractApplicationContext
deactivate AbstractApplicationContext
AbstractApplicationContext -> BeanFactory : getBean
activate BeanFactory
BeanFactory --> AbstractApplicationContext
deactivate BeanFactory
AbstractApplicationContext --> AbstractApplicationContext
deactivate AbstractApplicationContext
end

AbstractApplicationContext -> ConfigurableListableBeanFactory : freezeConfiguration 冻结底层 BeanFactory 容器所有的 BeanDefinition，目的是不希望再去修改 BeanDefinition
activate ConfigurableListableBeanFactory
ConfigurableListableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableListableBeanFactory
AbstractApplicationContext -[#red]> ConfigurableListableBeanFactory : preInstantiateSingletons 【重点】初始化所有还未初始化的 Bean（不是抽象、单例模式、不是懒加载方式），依赖查找
activate ConfigurableListableBeanFactory
ConfigurableListableBeanFactory --> AbstractApplicationContext
deactivate ConfigurableListableBeanFactory
return
{% endplantuml %}

**关于这个里面创建Bean相关的重要逻辑，在文章后面的单独小节作介绍**

##### finishRefresh()
刷新上下文的最后一步工作，会发布 ContextRefreshedEvent 上下文完成刷新事件

### Bean加载之getBean初始化
这个过程相对比较复杂一点，需要处理的东西有点多：
主要涉及的是关于```org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean```方法的执行步骤
核心代码:
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // <1> 获取 `beanName`
    // 因为入参 `name` 可能是别名，也可能是 FactoryBean 类型 Bean 的名称（`&` 开头，需要裁切）
    // 所以需要获取真实的 beanName
    final String beanName = transformedBeanName(name);
    Object bean;

    // <2> 先从缓存（仅缓存单例 Bean ）中获取 Bean 对象，这里缓存指的是 3 个 Map缓存
    // 缓存中也可能是正在初始化的 Bean，可以避免**循环依赖注入**引起的问题
    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    // <3> 若从缓存中获取到对应的 Bean，且 `args` 参数为空
    if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
        if (isSingletonCurrentlyInCreation(beanName)) {
          logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
              "' that is not fully initialized yet - a consequence of a circular reference");
        }
        else {
          logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
        }
      }
      // <3.1> 获取 Bean 的目标对象，`scopedInstance` 非 FactoryBean 类型直接返回
      // 否则，调用 FactoryBean#getObject() 获取目标对象
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    // 缓存中没有对应的 Bean，则开启 Bean 的加载
    else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      // <4> 如果**单例模式**下的 Bean 正在创建，这里又开始创建，表明存在循环依赖，则直接抛出异常
      if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      BeanFactory parentBeanFactory = getParentBeanFactory();
      // <5> 如果从当前容器中没有找到对应的 BeanDefinition，则从父容器中加载（如果存在父容器）
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        // Not found -> check parent.
        // <5.1> 获取 `beanName`，因为可能是别名，则进行处理
        // 和第 `1` 步不同，不需要对 `&` 进行处理，因为进入父容器重新依赖查找
        String nameToLookup = originalBeanName(name);
        // <5.2> 若为 AbstractBeanFactory 类型，委托父容器的 doGetBean 方法进行处理
        // 否则，就是非 Spring IoC 容器，根据参数调用相应的 `getBean(...)`方法
        if (parentBeanFactory instanceof AbstractBeanFactory) {
          return ((AbstractBeanFactory) parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
        }
        else if (args != null) {
          return (T) parentBeanFactory.getBean(nameToLookup, args);
        }
        else if (requiredType != null) {
          return parentBeanFactory.getBean(nameToLookup, requiredType);
        }
        else {
          return (T) parentBeanFactory.getBean(nameToLookup);
        }
      }

      // <6> 如果不是仅仅做类型检查，则表示需要创建 Bean，将 `beanName` 标记为已创建过
      // 在后面的**循环依赖检查**中会使用到
      if (!typeCheckOnly) {
        markBeanAsCreated(beanName);
      }

      try {
        // <7> 从容器中获取 `beanName` 对应的的 RootBeanDefinition（合并后）
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        // 检查是否为抽象类
        checkMergedBeanDefinition(mbd, beanName, args);

        // Guarantee initialization of beans that the current bean depends on.
        // <8> 获取当前正在创建的 Bean 所依赖对象集合（`depends-on` 配置的依赖）
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
          for (String dep : dependsOn) {
            // <8.1> 检测是否存在循环依赖，存在则抛出异常
            if (isDependent(beanName, dep)) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
            }
            // <8.2> 将 `beanName` 与 `dep` 之间依赖的关系进行缓存
            registerDependentBean(dep, beanName);
            try {
              // <8.3> 先创建好依赖的 Bean（重新调用 `getBean(...)` 方法）
              getBean(dep);
            }
            catch (NoSuchBeanDefinitionException ex) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
            }
          }
        }

        // Create bean instance.
        // <9> 开始创建 Bean，不同模式创建方式不同
        if (mbd.isSingleton()) { // <9.1> 单例模式
          /*
           * <9.1.1> 创建 Bean，成功创建则进行缓存，并移除缓存的早期对象
           * 创建过程实际调用的下面这个 `createBean(...)` 方法
           */
          sharedInstance = getSingleton(beanName,
              // ObjectFactory 实现类
              () -> {
                try {
                  // **【核心】** 创建 Bean，区分单例和原型模式的Bean，其中有一段处理代理对象判断的逻辑
                  return createBean(beanName, mbd, args);
                } catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  // 如果创建过程出现异常，则显式地从缓存中删除当前 Bean 相关信息
                  // 在单例模式下为了解决循环依赖，创建过程会缓存早期对象，这里需要进行删除
                  destroySingleton(beanName);
                  throw ex;
                }
          });
          // <9.1.2> 获取 Bean 的目标对象，`scopedInstance` 非 FactoryBean 类型直接返回
          // 否则，调用 FactoryBean#getObject() 获取目标对象
          bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        // <9.2> 原型模式
        else if (mbd.isPrototype()) {
          // It's a prototype -> create a new instance.
          Object prototypeInstance = null;
          try {
            // <9.2.1> 将 `beanName` 标记为**非单例模式**正在创建
            beforePrototypeCreation(beanName);
            // <9.2.2> **【核心】** 创建 Bean
            prototypeInstance = createBean(beanName, mbd, args);
          }
          finally {
            // <9.2.3> 将 `beanName` 标记为不在创建中，照应第 `9.2.1` 步
            afterPrototypeCreation(beanName);
          }
          // <9.2.4> 获取 Bean 的目标对象，`scopedInstance` 非 FactoryBean 类型直接返回
          // 否则，调用 FactoryBean#getObject() 获取目标对象
          bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }
        // <9.3> 其他模式
        else {
          // <9.3.1> 获取该模式的 Scope 对象 `scope`，不存在则抛出异常
          String scopeName = mbd.getScope();
          final Scope scope = this.scopes.get(scopeName);
          if (scope == null) {
            throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
          }
          try {
            // <9.3.2> 从 `scope` 中获取 `beanName` 对应的对象（看你的具体实现），不存在则执行**原型模式**的四个步骤进行创建
            Object scopedInstance = scope.get(beanName, () -> {
              // 将 `beanName` 标记为**非单例模式**式正在创建
              beforePrototypeCreation(beanName);
              try {
                // **【核心】** 创建 Bean
                return createBean(beanName, mbd, args);
              }
              finally {
                // 将 `beanName` 标记为不在创建中，照应上一步
                afterPrototypeCreation(beanName);
              }
            });
            // 获取 Bean 的目标对象，`scopedInstance` 非 FactoryBean 类型直接返回
            // 否则，调用 FactoryBean#getObject() 获取目标对象
            bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
          }
          catch (IllegalStateException ex) {
            throw new BeanCreationException(beanName,
                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                ex);
          }
        }
      }
      catch (BeansException ex) {
        cleanupAfterBeanCreationFailure(beanName);
        throw ex;
      }
    }

    // Check if required type matches the type of the actual bean instance.
    // <10> 如果入参 `requiredType` 不为空，并且 Bean 不是该类型，则需要进行类型转换
    if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
        // <10.1> 通过类型转换机制，将 Bean 转换成 `requiredType` 类型
        // 底层通过反射转换的，Spring作了一层抽象封装，交付给 ```org.springframework.core.convert.converter.GenericConverter``` 其子实现类相关转换
        T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
        // <10.2> 转换后的 Bean 为空则抛出异常
        if (convertedBean == null) {
          throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
        // <10.3> 返回类型转换后的 Bean 对象
        return convertedBean;
      }
      catch (TypeMismatchException ex) {
        if (logger.isTraceEnabled()) {
          logger.trace("Failed to convert bean '" + name + "' to required type '" +
              ClassUtils.getQualifiedName(requiredType) + "'", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
    }
    // <11> 返回获取到的 Bean
    return (T) bean;
  }

```

#### 三级缓存处理之getSingleton方法
相关代码：
```java
// 处理循环依赖的getSingleton
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 这里的三个 Map 可以解决**循环依赖注入**，在前面调用的后面会看到，AbstractAutowireCapableBeanFactory 的 doCreateBean(...) 方法中可以看到
    // <1> **【一级 Map】**从单例缓存 `singletonObjects` 中获取 beanName 对应的 Bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // <2> 如果**一级 Map**中不存在，且当前 beanName 正在创建
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      // <2.1> 对 `singletonObjects` 加锁
      synchronized (this.singletonObjects) {
        // <2.2> **【二级 Map】**从 `earlySingletonObjects` 集合中获取，里面会保存从 **三级 Map** 获取到的正在初始化的 Bean
        singletonObject = this.earlySingletonObjects.get(beanName);
        // <2.3> 如果**二级 Map** 中不存在，且允许提前创建
        if (singletonObject == null && allowEarlyReference) {
          // <2.3.1> **【三级 Map】**从 `singletonFactories` 中获取对应的 ObjectFactory 实现类
          ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
          // 如果从**三级 Map** 中存在对应的对象，则进行下面的处理
          if (singletonFactory != null) {
            // <2.3.2> 调用 ObjectFactory#getOject() 方法，获取目标 Bean 对象（早期半成品）
            singletonObject = singletonFactory.getObject();
            // <2.3.3> 将目标对象放入**二级 Map**
            this.earlySingletonObjects.put(beanName, singletonObject);
            // <2.3.4> 从**三级 Map**移除 beanName
            this.singletonFactories.remove(beanName);
          }
        }
      }
    }
    // <3> 返回从缓存中获取的对象
    return singletonObject;
  }

```
这里需要先解释一下三个Map的含义：
- singletonObjects（一级 Map），里面保存了所有已经初始化好的单例 Bean，也就是会保存 Spring IoC 容器中所有单例的 Spring Bean。【这个一定不能少，不然每次获取都得重复创建多费劲】；
- earlySingletonObjects（二级 Map），里面会保存从 三级 Map 获取到的正在初始化的 Bean，即早期初始化的Bean
- singletonFactories（三级 Map），里面保存了正在初始化的 Bean 对应的 ObjectFactory 实现类，调用其 getObject() 方法返回正在初始化的 Bean 对象（仅实例化还没完全初始化好）【同样，从这个属性名字就能看出，这其实是一个工厂的集合】。

一句话概况处理循环依赖的步骤就是：
例如两个 Bean 出现循环依赖，A 依赖 B，B 依赖 A；
- 当我们去依赖查找 A，在实例化后初始化前会先生成一个 ObjectFactory 对象（可获取当前正在初始化 A）保存在上面的 singletonFactories 中，初始化的过程需注入 B；
- 接下来去查找 B，初始 B 的时候又要去注入 A，又去查找 A ，由于可以通过 singletonFactories 直接拿到正在初始化的 A，那么就可以完成 B 的初始化
- 最后也完成 A 的初始化，这样就避免出现循环依赖。

这个过程，Spring处理的还是较为复杂的，如果你看不懂，参考下面这个精简版的示例来辅助理解一下：
```java
@Slf4j
public class CycleProblemSuccess {

    private final static Map<String, Object> singletonMap = new HashMap<String, Object>();

    public static void main(String[] args) throws InstantiationException, IllegalAccessException {
        log.info("Test1 after init:[{}]", getBean(Test1.class).getTest2());
        log.info("Test2 after init:[{}]", getBean(Test2.class).getTest1());
    }


    private static <T> T getBean(Class<T> clazz) throws InstantiationException, IllegalAccessException {
        String beanName = clazz.getSimpleName().toLowerCase();

        if (singletonMap.containsKey(beanName)) {
            return (T) singletonMap.get(beanName);
        }

        // 初始化并未实例化对象，但是未对属性赋值
        Object object = clazz.newInstance();
        singletonMap.put(beanName, object);

        // 通过反射来属性填充
        Field[] fields = object.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            Class<?> filedClazz = field.getType();
            String fieldName = filedClazz.getSimpleName().toLowerCase();
            // 给成员变量赋值 如果 singletonMap 中有半成品就获取缓存内容，否则再通过调用getBean方法递归的创建
            field.set(object, singletonMap.containsKey(fieldName) ? singletonMap.get(fieldName) : getBean(filedClazz));
        }
        return (T) object;
    }

}


@Getter
@Setter
class Test1 {
    private Test2 test2;
}

@Getter
@Setter
class Test2 {
    private Test1 test1;
}
```

#### 实例化创建bean之doCreateBean
前文中，创建bean的过程中还有一块核心的代码，就是Bean的实例化阶段：
先看代码：
```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

    if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    // <1> 获取 `mbd` 对应的 Class 对象，确保当前 Bean 能够被创建出来
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    // 如果这里获取到了 Class 对象，但是 `mbd` 中没有 Class 对象的相关信息，表示这个 Class 对象是动态解析出来的
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      // 复制一份 `mbd`，并设置 Class 对象，因为动态解析出来的 Class 对象不被共享
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
      // <2> 对所有的 MethodOverride 进行验证和准备工作（确保存在对应的方法，并设置为不能重复加载）
      mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
          beanName, "Validation of method overrides failed", ex);
    }

    try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      /**
       * <3> 在实例化前进行相关处理，会先调用所有 {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation}
       * 注意，如果这里返回对象不是 `null` 的话，不会继续往下执行原本初始化操作，直接返回，也就是说这个方法返回的是最终实例对象
       * 可以通过这种方式提前返回一个代理对象，例如 AOP 的实现，或者 RPC 远程调用的实现（因为本地类没有远程能力，可以通过这种方式进行拦截）
       */
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
        return bean;
      }
    }
    catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
          "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
      // <4> 创建 Bean 对象 `beanInstance`，如果上一步没有返回代理对象，就只能走常规的路线进行 Bean 的创建了
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
        logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      // <5> 将 `beanInstance` 返回
      return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanCreationException(
          mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
  }


// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

    // Instantiate the bean.
    /**
     * <1> Bean 的实例化阶段，会将 Bean 的实例对象封装成 {@link BeanWrapperImpl} 包装对象
     * BeanWrapperImpl 承担的角色：
     * 1. Bean 实例的包装
     * 2. {@link org.springframework.beans.PropertyAccessor} 属性编辑器
     * 3. {@link org.springframework.beans.PropertyEditorRegistry} 属性编辑器注册表
     * 4. {@link org.springframework.core.convert.ConversionService} 类型转换器（Spring 3+，替换了之前的 TypeConverter）
     */
    BeanWrapper instanceWrapper = null;
    // <1.1> 如果是单例模式，则先尝试从 `factoryBeanInstanceCache` 缓存中获取实例对象，并从缓存中移除
    if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // <1.2> 使用合适的实例化策略来创建 Bean 的实例：工厂方法、构造函数自动注入、简单初始化
    // 主要是将 BeanDefinition 转换为 BeanWrapper 对象
    if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // <1.3> 获取包装的实例对象 `bean`
    final Object bean = instanceWrapper.getWrappedInstance();
    // <1.4> 获取包装的实例对象的类型 `beanType`
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    // <2> 对 RootBeanDefinition（合并后）进行加工处理
    synchronized (mbd.postProcessingLock) { // 加锁，线程安全
      // <2.1> 如果该 RootBeanDefinition 没有处理过，则进行下面的处理
      if (!mbd.postProcessed) {
        try {
          /**
           * <2.2> 对 RootBeanDefinition（合并后）进行加工处理
           * 调用所有 {@link MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition}
           * 【重要】例如有下面两个后置处理器：
           * 1. AutowiredAnnotationBeanPostProcessor 会先解析出 @Autowired 和 @Value 注解标注的属性的注入元信息，后续进行依赖注入；
           * 2. CommonAnnotationBeanPostProcessor 会先解析出 @Resource 注解标注的属性的注入元信息，后续进行依赖注入，
           * 它也会找到 @PostConstruct 和 @PreDestroy 注解标注的方法，并构建一个 LifecycleMetadata 对象，用于后续生命周期中的初始化和销毁
           *
           * {@link org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor }
           * {@link org.springframework.context.annotation.CommonAnnotationBeanPostProcessor }
           */
          applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        }
        catch (Throwable ex) {
          throw new BeanCreationException(mbd.getResourceDescription(), beanName,
              "Post-processing of merged bean definition failed", ex);
        }
        // <2.3> 设置该 RootBeanDefinition 被处理过，避免重复处理
        mbd.postProcessed = true;
      }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // <3> 提前暴露这个 `bean`，如果可以的话，目的是解决单例模式 Bean 的循环依赖注入
    // <3.1> 判断是否可以提前暴露
    boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
        && this.allowCircularReferences // 允许循环依赖，默认为 true
        && isSingletonCurrentlyInCreation(beanName)); // 当前单例 bean 正在被创建，在前面已经标记过
    if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
            "' to allow for resolving potential circular references");
      }
      /**
       * <3.2>
       * 创建一个 ObjectFactory 实现类，用于返回当前正在被创建的 `bean`，提前暴露，保存在 `singletonFactories` （**三级 Map**）缓存中
       * 可以回到前面的 {@link AbstractBeanFactory#doGetBean#getSingleton(String)} 方法
       * 加载 Bean 的过程会先从缓存中获取单例 Bean，可以避免单例模式 Bean 循环依赖注入的问题
       * 在getEarlyBeanReference方法中支持了一个初始化的后置处理器，如对特定的类，使用特定的实现创建类代理。
       * {@link org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor }
       */
      addSingletonFactory(beanName,
          // ObjectFactory 实现类
          //
          () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    // 开始初始化 `bean`
    Object exposedObject = bean;
    try {
      // <4> 对 `bean` 进行属性填充，注入对应的属性值
      populateBean(beanName, mbd, instanceWrapper);
      // <5> 初始化这个 `exposedObject`，调用其初始化方法
      exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
        throw (BeanCreationException) ex;
      }
      else {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
    }

    // <6> 循环依赖注入的检查
    if (earlySingletonExposure) {
      // <6.1> 获取当前正在创建的 `beanName` 被依赖注入的早期引用
      // 注意，这里有一个入参是 `false`，不会调用上面第 `3` 步的 ObjectFactory 实现类
      // 也就是说当前 `bean` 如果出现循环依赖注入，这里才能获取到提前暴露的引用
      Object earlySingletonReference = getSingleton(beanName, false);
      // <6.2> 如果出现了循环依赖注入，则进行接下来的检查工作
      if (earlySingletonReference != null) {
        // <6.2.1> 如果 `exposedObject` 没有在初始化阶段中被改变，也就是没有被增强
        // 则使用提前暴露的那个引用
        if (exposedObject == bean) {
          exposedObject = earlySingletonReference;
        }
        // <6.2.2> 否则，`exposedObject` 已经不是被别的 Bean 依赖注入的那个 Bean
        else if (!this.allowRawInjectionDespiteWrapping // 是否允许注入未加工的 Bean，默认为 false，这里取非就为 true
            && hasDependentBean(beanName)) { // 存在依赖 `beanName` 的 Bean（通过 `depends-on` 配置）
          // 获取依赖当前 `beanName` 的 Bean 们的名称（通过 `depends-on` 配置）
          String[] dependentBeans = getDependentBeans(beanName);
          Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
          // 接下来进行判断，如果依赖 `beanName` 的 Bean 已经创建
          // 说明当前 `beanName` 被注入了，而这里最终的 `bean` 被包装过，不是之前被注入的
          // 则抛出异常
          for (String dependentBean : dependentBeans) {
            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
              actualDependentBeans.add(dependentBean);
            }
          }
          if (!actualDependentBeans.isEmpty()) {
            throw new BeanCurrentlyInCreationException(beanName,
                "Bean with name '" + beanName + "' has been injected into other beans [" +
                StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                "] in its raw version as part of a circular reference, but has eventually been " +
                "wrapped. This means that said other beans do not use the final version of the " +
                "bean. This is often the result of over-eager type matching - consider using " +
                "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
          }
        }
      }
    }

    // Register bean as disposable.
    try {
      /**
       * <7> 为当前 `bean` 注册 DisposableBeanAdapter（如果需要的话），用于 Bean 生命周期中的销毁阶段
       * 可以看到 {@link DefaultSingletonBeanRegistry#destroySingletons()} 方法
       */
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }
    // <8> 返回创建好的 `exposedObject` 对象
    return exposedObject;
  }
```

### 一些杂项问题
#### BeanFactory与FactoryBean
BeanFactory是spring顶层接口，使用了简单工厂模式，负责生产Bean实例
FactoryBean是对普通Bean对象增强的一个接口，实例化这种对象的时候，是会调用到实现的重新方法getObject()方法中初始化对象的。
#### 构造器注入和Setter注入
**构造器注入**：通过构造器的参数注入相关依赖对象
**Setter注入**：通过 Setter 方法注入依赖对象，也可以理解为字段注入
对于两种注入方式的看法（所以也可以说不推荐进行属性注入）：
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

#### Spring 中几种初始化方法的执行顺序
有以下几种初始化方式：
- **Aware 接口**：实现了 Spring 提供的相关 XxxAware 接口，例如 BeanNameAware、ApplicationContextAware，其 setXxx 方法会被回调，可以注入相关对象
- **@PostConstruct 注解**：该注解是 JSR-250 的标准注解，Spring 会调用该注解标注的方法
- **InitializingBean 接口**：实现了该接口，Spring 会调用其 afterPropertiesSet() 方法
- **自定义初始化方法**：通过 init-method 指定的方法会被调用

在 Spring 初始 Bean 的过程中上面的初始化方式的执行顺序如下：
1. Aware 接口的回调
2. JSR-250 @PostConstruct 标注的方法的调用
3. InitializingBean#afterPropertiesSet 方法的回调
4. init-method 初始化方法的调用

#### 通过 @Bean 注解定义在方法上面注入一个 Spring Bean，每次调用该方法所属的 Bean 的这个方法，得到的是同一个对象吗
不是的，@Configuration Class 在得到 CGLIB 提升后，会设置一个拦截器专门对 @Bean 方法进行拦截处理，通过依赖查找的方式从 IoC 容器中获取 Bean 对象，如果是单例 Bean，那么每次都是返回同一个对象，所以当主动调用这个方法时获取到的都是同一个 User 对象。

---

## SpringMVC
### 前瞻相关补充
#### 简单介绍一下 Spring MVC 框架
在早期 Java Web 的开发中，统一把显示层、控制层、显示层的操作全部交给 JSP 或者 Java Bean 来进行处理，存在一定的弊端，例如：JSP 和 Java Bean 之间严重耦合、开发效率低等弊端。
Spring MVC 是 Spring 体系中的一员，提供 **“模型-视图-控制器”（Model-View-Controller）架构** 和随时可用的组件，用于开发灵活且松散耦合的 Web 应用程序。
MVC 模式有助于分离应用程序的不同方面，如输入逻辑，业务逻辑和 UI 逻辑，同时在所有这些元素之间提供松散耦合。
#### Spring MVC 有什么优点
- 使用真的非常方便，无论是添加 HTTP 请求方法映射的方法，还是不同数据格式的响应。
- 提供拦截器机制，可以方便的对请求进行拦截处理。
- 提供异常机制，可以方便的对异常做统一处理。
- 可以任意使用各种视图技术，而不仅仅局限于 JSP ，例如 Freemarker、Thymeleaf 等等。

#### 描述一下 Spring MVC 的工作流程
Spring MVC 也是基于 Servlet 来处理请求的，主要通过 DispatcherServlet 这个 Servlet 来处理请求，处理过程需要通过九大组件来完成，先看到下面这个流程图：
![SpringMVC工作流程](f5eb228d/SpringMVC工作流程.jpg)
大体的步骤如下：
1. 用户的浏览器发送一个请求，这个请求经过互联网到达了我们的服务器。Servlet 容器（如常见的Tomcat）首先接待了这个请求，并将该请求委托给 `DispatcherServlet` 进行处理。
2. DispatcherServlet 将该请求传给了处理器映射组件 `HandlerMapping`，并获取到适合该请求的 `HandlerExecutionChain` 拦截器和处理器对象。
3. 在获取到处理器后，DispatcherServlet 还不能直接调用处理器的逻辑，需要进行对处理器进行适配。处理器适配成功后，DispatcherServlet 通过处理器适配器 `HandlerAdapter` 调用处理器的逻辑，并获取返回值 `ModelAndView` 对象。
4. 之后，DispatcherServlet 需要根据 ModelAndView 解析视图。解析视图的工作由 `ViewResolver` 完成，若能解析成功，ViewResolver 会返回相应的 `View` 视图对象。
5. 在获取到具体的 `View` 对象后，最后一步要做的事情就是由 `View` 渲染视图，并将渲染结果返回给用户。

总结就是：客户端发起请求后，最终会交由 DispatcherServlet 来处理，它会通过你的 URI 找到对应的方法，从请求中解析参数，然后通过反射机制调用该方法，将方法的执行结果设置到响应中，如果存在对应的 View 对象，则进行页面渲染，实际上就是将请求转发到指定的 URL

#### Spring MVC 的核心组件
九大组件（按使用顺序排序的）：

| 组件      |  说明           |
| ------------- |:-------------:|
| DispatcherServlet      | Spring MVC 的核心组件，是请求的入口，负责协调各个组件工作 |
| MultipartResolver      | 内容类型( Content-Type )为 multipart/* 的请求的解析器，例如解析处理文件上传的请求，便于获取参数信息以及上传的文件 |
| HandlerMapping      | 请求的处理器匹配器，负责为请求找到合适的 HandlerExecutionChain 处理器执行链，包含处理器（handler）和拦截器们（interceptors） |
| HandlerAdapter      | 处理器的适配器。因为处理器 handler 的类型是 Object 类型，需要有一个调用者来实现 handler 是怎么被执行。Spring 中的处理器的实现多变，比如用户处理器可以实现 Controller 接口、HttpRequestHandler 接口，也可以用 @RequestMapping 注解将方法作为一个处理器等，这就导致 Spring MVC 无法直接执行这个处理器。**所以这里需要一个处理器适配器，由它去执行处理器** |
| HandlerExceptionResolver      | 处理器异常解析器，将处理器（ handler ）执行时发生的异常，解析( 转换 )成对应的 ModelAndView 结果 |
| RequestToViewNameTranslator      | 视图名称转换器，用于解析出请求的默认视图名 |
| LocaleResolver      | 本地化（国际化）解析器，提供国际化支持 |
| ThemeResolver      | 主题解析器，提供可设置应用整体样式风格的支持 |
| ViewResolver      | 视图解析器，根据视图名和国际化，获得最终的视图 View 对象 |
| FlashMapManager      | FlashMap 管理器，负责重定向时，保存参数至临时存储（默认 Session） |

#### 一些关于SpringMVC的注解
##### @Controller 注解
`@Controller` 注解标记一个类为 Spring Web MVC 控制器 Controller。Spring MVC 会将扫描到该注解的类，然后扫描这个类下面带有 @RequestMapping 注解的方法，根据注解信息，**为这个方法生成一个对应的处理器对象**，在上面的 HandlerMapping 和 HandlerAdapter组件中讲到过。
当然，除了添加 @Controller 注解这种方式以外，你还可以实现 Spring MVC 提供的 Controller 或者 HttpRequestHandler 接口，对应的实现类也会被作为一个处理器对象。
##### @RequestMapping 注解
@RequestMapping 注解，配置处理器的 HTTP 请求方法，URI等信息，这样才能将请求和方法进行映射。这个注解可以作用于类上面，也可以作用于方法上面，在类上面一般是配置这个控制器的 URI 前缀。方法上的一般具体指定的路径。
##### @RestController 和 @Controller 的区别
@RestController 注解，在 @Controller 基础上，增加了 @ResponseBody 注解，更加适合目前前后端分离的架构下，**提供 Restful API** ，返回例如 JSON 数据格式。当然，返回什么样的数据格式，根据客户端的 ACCEPT 请求头来决定。
##### @RequestMapping 和 @GetMapping 的区别
- @RequestMapping：可注解在类和方法上；@GetMapping 仅可注册在方法上
- @RequestMapping：可进行 GET、POST、PUT、DELETE 等请求方法；
- @GetMapping 是 @RequestMapping 的 GET 请求方法的特例，目的是为了提高清晰度。

##### @RequestParam 和 @PathVariable的区别
两个注解都用于方法参数，**获取参数值的方式不同**：@RequestParam 注解的参数从请求携带的参数中获取，而 @PathVariable 注解从请求的 URI 中获取
#### 返回 JSON 格式使用什么注解
可以使用 @ResponseBody 注解，或者使用包含 @ResponseBody 注解的 @RestController 注解。
当然，还是需要配合相应的支持 JSON 格式化的 `HttpMessageConverter` 实现类。例如，Spring MVC 默认使用 `MappingJackson2HttpMessageConverter`
#### WebApplicationContext
WebApplicationContext 是实现 ApplicationContext 接口的子类，**专门为 WEB 应用准备的**
- 它允许从相对于 Web 根目录的路径中加载配置文件，完成初始化 Spring MVC 组件的工作。
- 从 WebApplicationContext 中，可以获取 ServletContext 引用，整个 Web 应用上下文对象将作为属性放置在 ServletContext 中，以便 Web 应用环境可以访问 Spring 上下文。

#### Spring MVC 拦截器
Spring MVC 拦截器有三个增强处理的地方：
- **前置处理**：在执行方法前执行，全部成功执行才会往下执行方法
- **后置处理**：在成功执行方法后执行，倒序
- **已完成处理**：不管方法是否成功执行都会执行，不过只会执行前置处理成功的拦截器，倒序
可以通过拦截器进行权限检验，参数校验，记录日志等操作
#### Spring MVC 的拦截器和 Filter 过滤器的区别
- **功能相同**：拦截器和 Filter 都能实现相应的功能，谁也不比谁强
- **容器不同**：拦截器构建在 Spring MVC 体系中；Filter 构建在 Servlet 容器之上
- **使用便利性不同**：拦截器提供了三个方法，分别在不同的时机执行；过滤器仅提供一个方法，当然也能实现拦截器的执行时机的效果，就是麻烦一些
一般拓展性好的框架，都会提供相应的拦截器或过滤器机制，方便的开发人员做一些拓展

### web.xml的生命之旅
#### Servlet3.0之前
回忆一下那时候，在Java Configuration技术出现之前，还是通过xml文件的方式配置，正如下面一个示例的web.xml配置开始：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <servlet>
        <servlet-name>HelloWorldServlet</servlet-name>
        <servlet-class>cc.nimbusk.webapp.servlet.HelloWorldServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>HelloWorldServlet</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>HelloWorldFilter</filter-name>
        <filter-class>cc.nimbusk.webapp.filter.HelloWorldFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>HelloWorldFilter</filter-name>
        <url-pattern>/hello</url-pattern>
    </filter-mapping>

</web-app>

```
在项目里面，我们通过手动继承`HttpServlet`类来实现Servlet功能，与之对应的可能还有过滤器的实现：
```java
public class HelloWorldServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.setContentType("text/plain");
        PrintWriter writer = resp.getWriter();
        writer.println("Hello World");
    }
}

public class HelloWorldFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("触发 Hello World 过滤器...");
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```
就像这样，配置Tomcat运行相关之后，启动通过浏览器指定端口：http://xx:8080/hello 就可以访问到后台返回的“Hello World”了。

#### Servlet 3.0
关于Servlet 3.0技术特性，这里就不提了，毕竟上古的东西。要知道两点，在Servlet3.0中引入了对注解的支持同时，对`ServletContext`的管理也得到了增强，具体就是ServletContext 为动态配置 Servlet 增加了如下方法：
- ServletRegistration.Dynamic addServlet(String servletName,Class<? extends Servlet> servletClass)
- ServletRegistration.Dynamic addServlet(String servletName, Servlet servlet)
- ServletRegistration.Dynamic addServlet(String servletName, String className)
- T createServlet(Class clazz)
- ServletRegistration getServletRegistration(String servletName)
- Map<string,? extends servletregistration> getServletRegistrations()

其中前三个方法的作用是相同的，只是参数类型不同而已；
通过 createServlet() 方法创建的 Servlet，通常需要做一些自定义的配置，然后**使用 addServlet() 方法来将其动态注册为一个可以用于服务的 Servlet**。
两个 getServletRegistration() 方法主要用于动态为 Servlet 增加映射信息，这等价于在 web.xml( 抑或 web-fragment.xml) 中使用 标签为存在的 Servlet 增加映射信息。

以上 ServletContext 新增的方法要么是在 `ServletContextListener` 的 contexInitialized 方法中调用，要么是在 `ServletContainerInitializer` 的 `onStartup()` 方法中调用。
**ServletContainerInitializer 也是 Servlet 3.0 新增的一个接口**，容器在启动时使用 JAR 服务 API(JAR Service API) 来发现 ServletContainerInitializer 的实现类，并且容器将 WEB-INF/lib 目录下 JAR 包中的类都交给该类的 onStartup() 方法处理，我们通常需要在该实现类上使用 @HandlesTypes 注解来指定希望被处理的类，过滤掉不希望给 onStartup() 处理的类。

有了上面这个动态添加的机制，我们就可以像下面这样来动态添加一个Servlet服务了：
```java
public class CustomServletContainerInitializer implements ServletContainerInitializer {

    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        System.out.println("创建 Hello World Servlet...");
        javax.servlet.ServletRegistration.Dynamic servlet = ctx.addServlet(
            HelloWorldServlet.class.getSimpleName(), HelloWorldServlet.class);
        servlet.addMapping("/hello");

        System.out.println("创建 Hello World Filter...");
        javax.servlet.FilterRegistration.Dynamic filter = ctx.addFilter(HelloWorldFilter.class.getSimpleName(), HelloWorldFilter.class);
        EnumSet<DispatcherType> dispatcherTypes = EnumSet.allOf(DispatcherType.class);
        dispatcherTypes.add(DispatcherType.REQUEST);
        dispatcherTypes.add(DispatcherType.FORWARD);
        filter.addMappingForUrlPatterns(dispatcherTypes, true, "/hello");
    }
}

```

这里还有个问题，正如前文所说，可以通过`ServletContainerInitializer`来扫描并添加我们实现的代码，但是怎么让容器发现我们的实现类呢？
这里就需要借助SPI的机制来辅助我们的web容器来发现我们的实现类，具体就是，通过在项目 ClassPath 路径下创建 `META-INF/services/javax.servlet.ServletContainerInitializer` 文件来做到的，内容如下：
```java
cc.nimbusk.webapp.CustomServletContainerInitializer
```

这样一来就可以转起来了，通过ServletContainerInitializer和SPI机制就可以和web.xml说再见了。

#### Servlet 3.0与Spring整合
说到这里，我们已经可以构建一个动态拓展的Servlet服务了，但是我们回想使用Spring（具体指SpringMVC）这么多年来，似乎已经忘记为啥，不知不觉从以前的web.xml的项目就不用这个xml文件了！
这里，我们很有必要回顾一下，为什么整合到spring之后，不用配置这个文件了。
通常，我们使用springmvc构建javaweb程序的时候，肯定需要依赖一个spring子工程，spring-web，我们看spring-web子工程源码的时候，可以发现，在其classpath下有这么一个文件：
```java
META-INF/services/javax.servlet.ServletContainerInitializer
```

这个文件里面有个一行配置：
```java
org.springframework.web.SpringServletContainerInitializer
```

如下图所示：
![SpringServletContainerInitializer](f5eb228d/spring-web对应的SpringServletContainerInitializer.jpg)
我们来看看，这个类的实现
```java
// 删除了类和方法注释，其实这两段注释，非常详细的解释了这个类以及这个onStartUp方法具体是要干嘛的。
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

  @Override
  public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
      throws ServletException {

    List<WebApplicationInitializer> initializers = new LinkedList<>();

    if (webAppInitializerClasses != null) {
      for (Class<?> waiClass : webAppInitializerClasses) {
        // Be defensive: Some servlet containers provide us with invalid classes,
        // no matter what @HandlesTypes says...
        // <1> 过滤不满足Spring加载的类：不是接口、不是抽象类，是WebApplicationInitializer实现子类
        // NOTE: isAssignableFrom是从class源码级别判断，而不是从实例对象判断两者的继承关系的，是一个native方法，这个与instance of有着本质区别
        if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
            WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
          try {
            initializers.add((WebApplicationInitializer)
                ReflectionUtils.accessibleConstructor(waiClass).newInstance());
          }
          catch (Throwable ex) {
            throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
          }
        }
      }
    }

    if (initializers.isEmpty()) {
      servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
      return;
    }

    servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
    AnnotationAwareOrderComparator.sort(initializers);
    for (WebApplicationInitializer initializer : initializers) {
      // <2> 循环调用onStartup方法
      initializer.onStartup(servletContext);
    }
  }

}
```
这里有几个点需要解释一下：
1. @HandlesTypes注解是Servlet3.0中引入的一个新注解，这个注解用来标记表示当前ServletContainerInitializer的实现类，能处理的类型。
2. <1> 示我们由于 Servlet 厂商实现的差异，onStartup 方法会加载我们本不想处理的 Class 对象，所以进行了特判。
3. Spring 与我们上述提供的 Demo 不同，并没有在 SpringServletContainerInitializer 中直接对 Servlet 和 Filter 进行注册，而是委托给了一个陌生的类 `WebApplicationInitializer` ，这个类便是 Spring 用来初始化 Web 环境的委托者类。

##### WebApplicationInitializer
先来看看这个类的继承图：
![WebApplicationInitializer](f5eb228d/WebApplicationInitializer.png)
这里面出现了一个身影：`AbstractDispatcherServletInitializer`，这个类如果不熟悉，那大名鼎鼎的`DispatcherServlet`你一定不陌生。
**而`AbstractDispatcherServletInitializer` 便是无 `web.xml` 前提下，创建 `DispatcherServlet` 的关键**，相关代码如下：
```java
// org.springframework.web.servlet.support.AbstractDispatcherServletInitializer
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {

  /**
   * The default servlet name. Can be customized by overriding {@link #getServletName}.
   */
  public static final String DEFAULT_SERVLET_NAME = "dispatcher";


  @Override
  public void onStartup(ServletContext servletContext) throws ServletException {
    // 调用父类启动的逻辑，创建WebApplicationContext相关
    // 创建AnnotationConfigWebApplicationContext后，创建ContextLoaderListener实例，该实例持有AnnotationConfigWebApplicationContext，该实例主要用于获取加载spring IOC配置信息相关
    super.onStartup(servletContext);
    // 注册 DispatcherServlet
    registerDispatcherServlet(servletContext);
  }

  protected void registerDispatcherServlet(ServletContext servletContext) {
    // 获得 Servlet 名
    String servletName = getServletName();
    Assert.hasLength(servletName, "getServletName() must not return null or empty");

    // <1> 创建 WebApplicationContext 对象
    WebApplicationContext servletAppContext = createServletApplicationContext();
    Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

    // <2> 创建 DispatchServlet 对象，FrameworkServlet 是其父类
    FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
    Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
    dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

    ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
    if (registration == null) {
      throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
          "Check if there is another servlet registered under the same name.");
    }

    registration.setLoadOnStartup(1);
    registration.addMapping(getServletMappings());
    registration.setAsyncSupported(isAsyncSupported());

    // <3> 注册过滤器
    Filter[] filters = getServletFilters();
    if (!ObjectUtils.isEmpty(filters)) {
      for (Filter filter : filters) {
        registerServletFilter(servletContext, filter);
      }
    }
    // <4> 执行可选的自定义的注册信息
    customizeRegistration(registration);
  }
}
```
至此传统的SpringMVC如何跟我们过去的web.xml进行解耦的过程，大致弄清楚了。
但是，创建之旅还需要提前介绍一下SpringBoot是如何加载Sevlet的，毕竟目前而言，很少有项目使用原生的SpringMVC来构建java-web程序的了。

#### SpringBoot与Servlet创建
先铺垫一下，前文中的传统加载方式，在springboot中的加载流程基本上没有关系，这是一个全新的加载流程。
##### 两种加载方式
1. Servlet3.0 注解 +@ServletComponentScan
SpringBoot 依旧兼容 Servlet 3.0 一系列以 @Web* 开头的注解：@WebServlet，@WebFilter，@WebListener
例如：
```java
@WebServlet("/hello")
public class HelloWorldServlet extends HttpServlet{}

@WebFilter("/hello/*")
public class HelloWorldFilter implements Filter {}
```

在启动类上面添加 `@ServletComponentScan` 注解去扫描到这些注解
```java
@SpringBootApplication
@ServletComponentScan
public class SpringBootServletApplication {
   public static void main(String[] args) {
      SpringApplication.run(SpringBootServletApplication.class, args);
   }
}
```

这种方式相对来说比较简介直观，其中 `org.springframework.boot.web.servlet.@ServletComponentScan` 注解通过 `@Import(ServletComponentScanRegistrar.class)` 方式，它会将扫描到的 `@WebServlet、@WebFilter、@WebListener` 的注解对应的类，最终封装成 `FilterRegistrationBean、ServletRegistrationBean、ServletListenerRegistrationBean` 对象，注册到 Spring 容器中。
也就是说，和注册方式二：RegistrationBean统一了

2. RegistrationBean
例如：
```java
@Configuration
public class WebConfig {
    @Bean
    public ServletRegistrationBean<HelloWorldServlet> helloWorldServlet() {
        ServletRegistrationBean<HelloWorldServlet> servlet = new ServletRegistrationBean<>();
        servlet.addUrlMappings("/hello");
        servlet.setServlet(new HelloWorldServlet());
        return servlet;
    }

    @Bean
    public FilterRegistrationBean<HelloWorldFilter> helloWorldFilter() {
        FilterRegistrationBean<HelloWorldFilter> filter = new FilterRegistrationBean<>();
        filter.addUrlPatterns("/hello/*");
        filter.setFilter(new HelloWorldFilter());
        return filter;
    }
}
```

`ServletRegistrationBean` 和 `FilterRegistrationBean` 都继成 RegistrationBean，它是 SpringBoot 中广泛应用的一个注册类，负责把 Servlet，Filter，Listener 给容器化，使它们被 Spring 托管，并且完成自身对 Web 容器的注册。

##### SpringBoot加载 Servlet 的流程
当使用内嵌的 Tomcat 时，你在 `SpringServletContainerInitializer` 上面打断点，会发现根本不会进入该类的内部，因为 SpringBoot 完全走了另一套初始化流程，而是进入了 `org.springframework.boot.web.embedded.tomcat.TomcatStarter` 这个类
SpringBoot在取舍原始`java -jar`模式运行的基础上，构建了一个新的加载器：`ServletContextInitializer`，和 Servlet**Container**Initializer 不一样：
- 前者 ServletContextInitializer 是 *org.springframework.boot.web.servlet.ServletContextInitializer*
- 后者 ServletContainerInitializer 是 *javax.servlet.ServletContainerInitializer*，前文提到的 RegistrationBean 就实现了 ServletContextInitializer 接口

TomcatStarter 中 org.springframework.boot.context.embedded.ServletContextInitializer[] initializers 属性，是 SpringBoot 初始化 Servlet，Filter，Listener 的关键，代码如下：
```java
class TomcatStarter implements ServletContainerInitializer {

  private static final Log logger = LogFactory.getLog(TomcatStarter.class);

  private final ServletContextInitializer[] initializers;

  private volatile Exception startUpException;

  TomcatStarter(ServletContextInitializer[] initializers) {
    this.initializers = initializers;
  }

  @Override
  public void onStartup(Set<Class<?>> classes, ServletContext servletContext) throws ServletException {
    try {
      for (ServletContextInitializer initializer : this.initializers) {
        initializer.onStartup(servletContext);
      }
    }
    catch (Exception ex) {
      this.startUpException = ex;
      // Prevent Tomcat from logging and re-throwing when we know we can
      // deal with it in the main thread, but log for information here.
      if (logger.isErrorEnabled()) {
        logger.error("Error starting Tomcat context. Exception: "
            + ex.getClass().getName() + ". Message: " + ex.getMessage());
      }
    }
  }

  public Exception getStartUpException() {
    return this.startUpException;
  }

}

```

这里面加载的时候，内嵌容器是通过它在内嵌容器中的实现类：`org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext `，来加载 Filter、Servlet 和 Listener 这部分的代码，总结来看，这部分流程就是：
- ServletWebServerApplicationContext 的 onRefresh() 方法触发配置了一个匿名的 ServletContextInitializer
- 这个匿名的 ServletContextInitializer 的 onStartup 方法会去容器中搜索到了所有的 RegisterBean 并按照顺序加载到 ServletContext 中
- 这个匿名的 ServletContextInitializer 最终传递给 TomcatStarter，由 TomcatStarter 的 onStartup 方法去触发 ServletContextInitializer 的 onStartup 方法，最终完成装配

至此前面大致铺垫了相关知识之后，我们开始正式剖析SpringMVC内部的核心要点：
### WebApplicationContext容器初始化
先来看一段经典的xml配置的spring-mvc.xml文件：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
    <display-name>Archetype Created Web Application</display-name>

    <!-- 【1】 Spring 配置 -->
    <!-- 在容器（Tomcat、Jetty）启动时会被 ContextLoaderListener 监听到，
         从而调用其 contextInitialized() 方法，初始化 Root WebApplicationContext 容器 -->
    <!-- 声明 Spring Web 容器监听器 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    
    <!-- Spring 和 MyBatis 的配置文件 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mybatis.xml</param-value>
    </context-param>

    <!-- 【2】 Spring MVC 配置 -->
    <!-- 1.SpringMVC 配置 前置控制器（SpringMVC 的入口）
         DispatcherServlet 是一个 Servlet，所以可以配置多个 DispatcherServlet -->
    <servlet>
        <!-- 在 DispatcherServlet 的初始化过程中，框架会在 web 应用 的 WEB-INF 文件夹下，
             寻找名为 [servlet-name]-servlet.xml 的配置文件，生成文件中定义的 Bean. -->
        <servlet-name>SpringMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- 配置需要加载的配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <!-- 程序运行时从 web.xml 开始，加载顺序为：context-param -> Listener -> Filter -> Structs -> Servlet
             设置 web.xml 文件启动时加载的顺序(1 代表容器启动时首先初始化该 Servlet，让这个 Servlet 随 Servlet 容器一起启动)
             load-on-startup 是指这个 Servlet 是在当前 web 应用被加载的时候就被创建，而不是第一次被请求的时候被创建  -->
        <load-on-startup>1</load-on-startup>
        <async-supported>true</async-supported>
    </servlet>
    <servlet-mapping>
        <!-- 这个 Servlet 的名字是 SpringMVC，可以有多个 DispatcherServlet，是通过名字来区分的
             每一个 DispatcherServlet 有自己的 WebApplicationContext 上下文对象，同时保存在 ServletContext 中和 Request 对象中
             ApplicationContext（Spring 容器）是 Spring 的核心
             Context 我们通常解释为上下文环境，Spring 把 Bean 放在这个容器中，在需要的时候，可以 getBean 方法取出-->
        <servlet-name>SpringMVC</servlet-name>
        <!-- Servlet 拦截匹配规则，可选配置：*.do、*.action、*.html、/、/xxx/* ，不允许:/* -->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
其中：
- 【1】 处，配置了 `org.springframework.web.context.ContextLoaderListener` 对象，它实现了 Servlet 的 `javax.servlet.ServletContextListener` 接口，能够监听 `ServletContext` 对象的生命周期，也就是监听 Web 应用的生命周期，当 Servlet 容器启动或者销毁时，会触发相应的 ServletContextEvent 事件，ContextLoaderListener 监听到启动事件，则会初始化一个Root Spring WebApplicationContext 容器，监听到销毁事件，则会销毁该容器
- 【2】 处，配置了 `org.springframework.web.servlet.DispatcherServlet` 对象，它继承了 javax.servlet.http.HttpServlet 抽象类，也就是一个 Servlet。Spring MVC 的核心类，处理请求，会初始化一个属于它的 Spring WebApplicationContext 容器，并且这个容器是以 【1】 处的 Root 容器作为父容器

#### Root WebApplicationContext 容器
在前文中的web.xml中可以看到，Root WebApplicationContext 容器的初始化，通过 `ContextLoaderListener` 来实现。在 Servlet 容器启动时，例如 Tomcat、Jetty 启动后，**则会被 ContextLoaderListener 监听到，从而调用 contextInitialized(ServletContextEvent event) 方法，初始化 Root WebApplicationContext 容器**。
##### ContextLoaderListener监听器
我们先看看该监听器的继承图：
![ContextLoaderListener](f5eb228d/ContextLoaderListener.jpg)
org.springframework.web.context.ContextLoaderListener 类，实现 javax.servlet.ServletContextListener 接口，继承 ContextLoader 类，实现 Servlet 容器启动和关闭时，分别初始化和销毁 WebApplicationContext 容器，代码如下：

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {

  public ContextLoaderListener() {
  }

  public ContextLoaderListener(WebApplicationContext context) {
    // 调用父类ContextLoader初始化
    super(context);
  }


  @Override
  public void contextInitialized(ServletContextEvent event) {
        // <1> 初始化 Root WebApplicationContext
    initWebApplicationContext(event.getServletContext());
  }

  @Override
  public void contextDestroyed(ServletContextEvent event) {
        // <2> 销毁 Root WebApplicationContext
    closeWebApplicationContext(event.getServletContext());
    ContextCleanupListener.cleanupAttributes(event.getServletContext());
  }

}
```

我们接着顺着上面代码往里面看看，上述代码的super(context)，这个在类ContextLoader中：
```java
// org.springframework.web.context.ContextLoader
public class ContextLoader {

  /**
   * Name of the class path resource (relative to the ContextLoader class)
   * that defines ContextLoader's default strategy names.
   */
  private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";

    /**
     * 默认的配置 Properties 对象
     */
  private static final Properties defaultStrategies;

  static {
    // Load default strategy implementations from properties file.
    // This is currently strictly internal and not meant to be customized by application developers.
    try {
      ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
      defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
      throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
    }
  }
}

```
这里有一段静态代码块，加载了一个 `ContextLoader.properties` 的文件，我们可以打开文件看看：
```properties
## 注意下面的原英文注释
# Default WebApplicationContext implementation class for ContextLoader.
# Used as fallback when no explicit context implementation has been specified as context-param.
# Not meant to be customized by application developers.

org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext

```
这个配置意味着，如果我们没有在 `<context-param />` 标签中指定 WebApplicationContext，则默认使用 `XmlWebApplicationContext` 类，**我们在使用 Spring 的过程中一般情况下不会主动指定**。

ContextLoader剩下的其余构造方法
```java
public class ContextLoader {
    
  public static final String CONFIG_LOCATION_PARAM = "contextConfigLocation";
    
  private static final Map<ClassLoader, WebApplicationContext> currentContextPerThread = new ConcurrentHashMap<>(1);

  @Nullable
  private static volatile WebApplicationContext currentContext;

  @Nullable
  private WebApplicationContext context;


  public ContextLoader() {
  }

  public ContextLoader(WebApplicationContext context) {
    this.context = context;
  }
    
    // ... 省略其他相关配置属性
}

```
其中：
- 在前文的 web.xml 文件中可以看到定义的 `contextConfigLocation` 参数为 spring-mybatis.xml 配置文件路径
- **currentContextPerThread**：用于保存当前 ClassLoader 类加载器与 WebApplicationContext 对象的映射关系
- **currentContext**：如果当前线程的类加载器就是 ContextLoader 类所在的类加载器，则该属性用于保存 WebApplicationContext 对象
- **context**：WebApplicationContext 实例对象

initWebApplicationContext(ServletContext servletContext) 方法，初始化 WebApplicationContext 对象，代码如下：
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 该方法通过ContextLoaderListener.contextInitialized开始调用，这是servlet容器定义的初始化入口
    // <1> 若已经存在 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE 对应的 WebApplicationContext 对象，则抛出 IllegalStateException 异常。
      // 例如，在 web.xml 中存在多个 ContextLoader
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
      throw new IllegalStateException(
          "Cannot initialize context because there is already a root application context present - " +
          "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }

    // <2> 打印日志
    servletContext.log("Initializing Spring root WebApplicationContext");
    Log logger = LogFactory.getLog(ContextLoader.class);
    if (logger.isInfoEnabled()) {
      logger.info("Root WebApplicationContext: initialization started");
    }
    // 记录开始时间
    long startTime = System.currentTimeMillis();

    try {
      // Store context in local instance variable, to guarantee that
      // it is available on ServletContext shutdown.
      if (this.context == null) {
        // <3> 初始化 context ，即创建 context 对象
        this.context = createWebApplicationContext(servletContext);
      }
      // <4> 如果是 ConfigurableWebApplicationContext 的子类，如果未刷新，则进行配置和刷新
      if (this.context instanceof ConfigurableWebApplicationContext) {
        ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
        if (!cwac.isActive()) { // <4.1> 未刷新（激活）
          // The context has not yet been refreshed -> provide services such as
          // setting the parent context, setting the application context id, etc
          if (cwac.getParent() == null) { // <4.2> 无父容器，则进行加载和设置。
            // The context instance was injected without an explicit parent ->
            // determine parent for root web application context, if any.
            ApplicationContext parent = loadParentContext(servletContext);
            cwac.setParent(parent);
          }
          // <4.3> 【核心】 配置 context 对象，并进行刷新
          configureAndRefreshWebApplicationContext(cwac, servletContext);
        }
      }
      // <5> 记录在 servletContext 中
      servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

      // <6> 记录到 currentContext 或 currentContextPerThread 中
      ClassLoader ccl = Thread.currentThread().getContextClassLoader();
      if (ccl == ContextLoader.class.getClassLoader()) {
        currentContext = this.context;
      }
      else if (ccl != null) {
        currentContextPerThread.put(ccl, this.context);
      }

      if (logger.isInfoEnabled()) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
      }

      // <7> 返回 context
      return this.context;
    }
    catch (RuntimeException | Error ex) {
      logger.error("Context initialization failed", ex);
      servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
      throw ex;
    }
  }
```
配合上文中的中文注释，其中有一块需要额外说明：
- 第3步，如果context为空，则调用**createWebApplicationContext(ServletContext sc)**方法，初始化一个 Root WebApplicationContext 对象，方法如下：
```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // <1> 获得 context 的类（默认情况是从 ContextLoader.properties 配置文件读取的，为 XmlWebApplicationContext）
    Class<?> contextClass = determineContextClass(sc);
    // <2> 判断 context 的类，是否符合 ConfigurableWebApplicationContext 的类型
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    // <3> 创建 context 的类的对象
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}

```

- 第4步，如果是 `ConfigurableWebApplicationContext` 的子类，并且未刷新，则进行配置和刷新：
  - 如果未刷新（激活），默认情况下，是符合这个条件的，所以会往下执行
  - 如果无父容器，则进行加载和设置。默认情况下，loadParentContext(ServletContext servletContext) 方法返回一个空对象，也就是没有父容器了
  - 调用**configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc)**方法，配置context对象，并进行刷新

**configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc)** 方法，配置 ConfigurableWebApplicationContext 对象，并进行刷新，方法如下：
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    // <1> 如果 wac 使用了默认编号，则重新设置 id 属性
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
      // The application context id is still set to its original default value
      // -> assign a more useful id based on available information
      // 情况一，使用 contextId 属性
      String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
      if (idParam != null) {
        wac.setId(idParam);
      }
      else { // 情况二，自动生成
        // Generate default id...
        wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
            ObjectUtils.getDisplayString(sc.getContextPath()));
      }
    }

    // <2>设置 context 的 ServletContext 属性
    wac.setServletContext(sc);
    // <3> 设置 context 的配置文件地址
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
      wac.setConfigLocation(configLocationParam);
    }

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    // 立马调用initPropertySources为了确保所有的属性初始化
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
      ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }

    // <4> 对 context 进行定制化处理
    customizeContext(sc, wac);
    // <5> 【实际对象是XmlWebApplicationContext，改refresh方法定义在AbstractApplicationContext中】
    // 刷新 context ，执行初始化
    wac.refresh();
  }
```
其中：
1. 如果 wac 使用了默认编号，则重新设置 id 属性。默认情况下，我们不会对 wac 设置编号，所以会执行进去。而实际上，id 的生成规则，也分成使用 contextId 在 <context-param /> 标签中由用户配置，和自动生成两种情况。😈 默认情况下，会走第二种情况。
2. 设置 wac 的 ServletContext 属性
3. 【关键】设置 context 的配置文件地址。例如我们在前文中的 web.xml 中所看到的
```xml
<!-- Spring 和 MyBatis 的配置文件 -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring-mybatis.xml</param-value>
</context-param>
```

4. 对 wac 进行定制化处理，暂时忽略
5. 【关键】触发 wac 的刷新事件，执行初始化。此处，就会进行一些的 Spring 容器的初始化工作，涉及到 Spring IOC 相关内容。这个过程就是调用我们在IOC篇章中，看的最多的：org.springframework.context.support.AbstractApplicationContext#refresh方法。**至此，SpringMVC和SpringIOC相关打通了。**

### Servlet WebApplicationContext 容器
咱们接着顺着web.xml的内容，往下看看：
在web.xml中，我们已经看到，除了会初始化一个 Root WebApplicationContext 容器外，**还会往 Servlet 容器的 ServletContext 上下文中注入一个 DispatcherServlet 对象，初始化该对象的过程也会初始化一个 Servlet WebApplicationContext 容器**。
先看看DispatcherServlet继承图：
![DispatchSevlet](f5eb228d/DispatchSevlet.jpg)
可以看到 DispatcherServlet 是一个 Servlet 对象，在注入至 Servlet 容器会**调用其 init 方法**，完成一些初始化工作
- HttpServletBean ，负责将 ServletConfig 设置到当前 Servlet 对象中，它的 Java doc：
```java
/**
 * Simple extension of {@link javax.servlet.http.HttpServlet} which treats
 * its config parameters ({@code init-param} entries within the
 * {@code servlet} tag in {@code web.xml}) as bean properties.
 */

```

- FrameworkServlet ，负责**初始化 Spring Servlet WebApplicationContext 容器，同时该类覆写了 doGet、doPost 等方法**，并将所有类型的请求委托给 doService 方法去处理，doService 是一个抽象方法，需要子类实现，它的 Java doc：
```java
/**
 * Base servlet for Spring's web framework. Provides integration with
 * a Spring application context, in a JavaBean-based overall solution.
 */

```

- DispatcherServlet ，负责初始化 Spring MVC 的各个组件，以及处理客户端的请求，协调各个组件工作，它的 Java doc：
```java
/**
 * Central dispatcher for HTTP request handlers/controllers, e.g. for web UI controllers
 * or HTTP-based remote service exporters. Dispatches to registered handlers for processing
 * a web request, providing convenient mapping and exception handling facilities.
 */

```

每个组件都各司其职，下面分别来看看：
#### HttpServletBean
org.springframework.web.servlet.HttpServletBean 抽象类，实现 EnvironmentCapable、EnvironmentAware 接口，继承 HttpServlet 抽象类，**负责将 ServletConfig 集成到 Spring 中**
```java
public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {

  @Nullable
  private ConfigurableEnvironment environment;

  /**
   * 必须配置的属性的集合，在 {@link ServletConfigPropertyValues} 中，会校验是否有对应的属性
   * 默认为空
   */
  private final Set<String> requiredProperties = new HashSet<>(4);

  protected final void addRequiredProperty(String property) {
    this.requiredProperties.add(property);
  }

    /**
     * 实现了 EnvironmentAware 接口，自动注入 Environment 对象
     */
  @Override
  public void setEnvironment(Environment environment) {
    Assert.isInstanceOf(ConfigurableEnvironment.class, environment, "ConfigurableEnvironment required");
    this.environment = (ConfigurableEnvironment) environment;
  }

    /**
     * 实现了 EnvironmentAware 接口，返回 Environment 对象
     */
  @Override
  public ConfigurableEnvironment getEnvironment() {
    if (this.environment == null) {
            // 如果 Environment 为空，则创建 StandardServletEnvironment 对象
      this.environment = createEnvironment();
    }
    return this.environment;
  }

  /**
   * Create and return a new {@link StandardServletEnvironment}.
   */
  protected ConfigurableEnvironment createEnvironment() {
    return new StandardServletEnvironment();
  }
}
```

我们重点看一下这个类中的init方法：
```java
@Override
public final void init() throws ServletException {
    // Set bean properties from init parameters.
    // <1> 解析 <init-param /> 标签，封装到 PropertyValues pvs 中
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            // <2.1> 将当前的这个 Servlet 对象，转化成一个 BeanWrapper 对象。从而能够以 Spring 的方式来将 pvs 注入到该 BeanWrapper 对象中
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            // <2.2> 注册自定义属性编辑器，一旦碰到 Resource 类型的属性，将会使用 ResourceEditor 进行解析
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            // <2.3> 空实现，留给子类覆盖，目前没有子类实现
            initBeanWrapper(bw);
            // <2.4> 以 Spring 的方式来将 pvs 注入到该 BeanWrapper 对象中
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }
    // Let subclasses do whatever initialization they like.
    // 交由子类去实现，查看 FrameworkServlet#initServletBean() 方法
    initServletBean();
}
```

其中：
1. 解析 Servlet 配置的 `<init-param />` 标签，封装成 PropertyValues的pvs对象。其中，ServletConfigPropertyValues 是 HttpServletBean 的私有静态类，继承 MutablePropertyValues 类。
2. 如果存在`<init-param />`初始化参数：
    1. 将当前的这个 Servlet 对象，转化成一个 BeanWrapper 对象。从而能够以 Spring 的方式来将 pvs 注入到该 BeanWrapper 对象中。**简单来说，BeanWrapper 是 Spring 提供的一个用来操作 Java Bean 属性的工具，使用它可以直接修改一个对象的属性**。
    2. 注册自定义属性编辑器，一旦碰到 Resource 类型的属性，将会使用 ResourceEditor 进行解析
    3. 调用initBeanWrapper(BeanWrapper bw)方法，可初始化当前这个 Servlet 对象，空实现，留给子类覆盖，目前好像还没有子类实现
    4. 遍历 pvs 中的属性值，注入到该 BeanWrapper 对象中，也就是设置到当前 Servlet 对象中，*例如 FrameworkServlet 中的 contextConfigLocation 属性则会设置为上面的 classpath:spring-mvc.xml 值了。*
3. 【关键】调用initServletBean()方法，空实现，交由子类去实现，完成自定义初始化逻辑，顺着下来，我们看看 FrameworkServlet#initServletBean() 方法

#### FrameworkServlet
org.springframework.web.servlet.FrameworkServlet 抽象类，实现 ApplicationContextAware 接口，继承 HttpServletBean 抽象类，负责初始化 Spring Servlet WebApplicationContext 容器
##### 构造方法
```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    // ... 省略部分属性
    
    /** Default context class for FrameworkServlet. */
  public static final Class<?> DEFAULT_CONTEXT_CLASS = XmlWebApplicationContext.class;

  /** WebApplicationContext implementation class to create. */
  private Class<?> contextClass = DEFAULT_CONTEXT_CLASS;

  /** Explicit context config location. 配置文件的地址 */
  @Nullable
  private String contextConfigLocation;

  /** Should we publish the context as a ServletContext attribute?. */
  private boolean publishContext = true;

  /** Should we publish a ServletRequestHandledEvent at the end of each request?. */
  private boolean publishEvents = true;

  /** WebApplicationContext for this servlet. */
  @Nullable
  private WebApplicationContext webApplicationContext;

  /** 标记是否是通过 {@link #setApplicationContext} 注入的 WebApplicationContext */
  private boolean webApplicationContextInjected = false;

  /** 标记已经是否接收到 ContextRefreshedEvent 事件，即 {@link #onApplicationEvent(ContextRefreshedEvent)} */
  private volatile boolean refreshEventReceived = false;

  /** Monitor for synchronized onRefresh execution. */
  private final Object onRefreshMonitor = new Object();

  public FrameworkServlet() {
  }

  public FrameworkServlet(WebApplicationContext webApplicationContext) {
    this.webApplicationContext = webApplicationContext;
  }
    
    @Override
  public void setApplicationContext(ApplicationContext applicationContext) {
    if (this.webApplicationContext == null && applicationContext instanceof WebApplicationContext) {
      this.webApplicationContext = (WebApplicationContext) applicationContext;
      this.webApplicationContextInjected = true;
    }
  }
}

```
- **contextClass 属性**：创建的 WebApplicationContext 类型，默认为 XmlWebApplicationContext.class，在 Root WebApplicationContext 容器的创建过程中也是它
- **contextConfigLocation 属性**：配置文件的地址，例如：classpath:spring-mvc.xml
- **webApplicationContext 属性**：WebApplicationContext 对象，Servlet WebApplicationContext 容器，有四种创建方式
  - 通过上面的构造方法
  - 实现了 ApplicationContextAware 接口，通过 Spring 注入，也就是 setApplicationContext(ApplicationContext applicationContext) 方法
  - 通过 findWebApplicationContext() 方法，下文见
  - 通过 createWebApplicationContext(WebApplicationContext parent) 方法，下文见

##### initServletBean方法
initServletBean() 方法，重写父类的方法，在 HttpServletBean 的 init() 方法的最后一步会调用，进一步初始化当前 Servlet 对象，当前主要是初始化Servlet WebApplicationContext 容器。
流程不是特别复杂，就忽略相关代码的展示了。

##### initWebApplicationContext方法
initWebApplicationContext() 方法【核心】，初始化 Servlet WebApplicationContext 对象，方法如下
```java
protected WebApplicationContext initWebApplicationContext() {
    // <1> 获得根 WebApplicationContext 对象
    WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    // <2> 获得 WebApplicationContext wac 对象
    WebApplicationContext wac = null;

    // 第一种情况，如果构造方法已经传入 webApplicationContext 属性，则直接使用
    if (this.webApplicationContext != null) {
      // A context instance was injected at construction time -> use it
      wac = this.webApplicationContext;
      // 如果是 ConfigurableWebApplicationContext 类型，并且未激活，则进行初始化
      if (wac instanceof ConfigurableWebApplicationContext) {
        ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
        if (!cwac.isActive()) { // 未激活
          // The context has not yet been refreshed -> provide services such as
          // setting the parent context, setting the application context id, etc
          if (cwac.getParent() == null) {
            // The context instance was injected without an explicit parent -> set
            // the root application context (if any; may be null) as the parent
            cwac.setParent(rootContext);
          }
          // 配置和初始化 wac
          configureAndRefreshWebApplicationContext(cwac);
        }
      }
    }
    // 第二种情况，从 ServletContext 获取对应的 WebApplicationContext 对象
    if (wac == null) {
      // No context instance was injected at construction time -> see if one
      // has been registered in the servlet context. If one exists, it is assumed
      // that the parent context (if any) has already been set and that the
      // user has performed any initialization such as setting the context id
      wac = findWebApplicationContext();
    }
    // 第三种，创建一个 WebApplicationContext 对象
    if (wac == null) {
      // No context instance is defined for this servlet -> create a local one
      wac = createWebApplicationContext(rootContext);
    }

    // <3> 如果未触发刷新事件，则主动触发刷新事件
    // 注意这里，后文中还会提到
    if (!this.refreshEventReceived) {
      // Either the context is not a ConfigurableApplicationContext with refresh
      // support or the context injected at construction time had already been
      // refreshed -> trigger initial onRefresh manually here.
      synchronized (this.onRefreshMonitor) {
        onRefresh(wac);
      }
    }

    // <4> 将 context 设置到 ServletContext 中
    if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
    }

    return wac;
  }
```
其中：
1. 调用 WebApplicationContextUtils#getWebApplicationContext((ServletContext sc) 方法，从 ServletContext 中获得 Root WebApplicationContext 对象，可以回到ContextLoader#initWebApplicationContext方法中的第 5 步，你会觉得很熟悉
2. 获得 WebApplicationContext wac 对象，有三种情况
    1. 如果构造方法已经传入 webApplicationContext 属性，则直接引用给 wac，也就是上面构造方法中提到的第 1、2 种创建方式
      如果 wac 是 ConfigurableWebApplicationContext 类型，并且未刷新（未激活），则调用 configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) 方法，进行配置和刷新，下文见
      如果父容器为空，则设置为上面第 1 步获取到的 Root WebApplicationContext 对象。
    2. 调用 findWebApplicationContext()方法，从 ServletContext 获取对应的 WebApplicationContext 对象，也就是上面构造方法中提到的第 3 种创建方式
        ```java
        @Nullable
        protected WebApplicationContext findWebApplicationContext() {
            String attrName = getContextAttribute();
            // 需要配置了 contextAttribute 属性下，才会去查找，一般我们不会去配置
            if (attrName == null) {
                return null;
            }
            // 从 ServletContext 中，获得属性名对应的 WebApplicationContext 对象
            WebApplicationContext wac = WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
            // 如果不存在，则抛出 IllegalStateException 异常
            if (wac == null) {
                throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
            }
            return wac;
        }
        
        ```
        **知道有这种方式，一般不会这么做的。**
    3. 调用createWebApplicationContext(@Nullable WebApplicationContext parent)方法，创建一个 WebApplicationContext 对象
      ```java
              protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
            // <a> 获得 context 的类，XmlWebApplicationContext.class
            Class<?> contextClass = getContextClass();
            // 如果非 ConfigurableWebApplicationContext 类型，抛出 ApplicationContextException 异常
            if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
                throw new ApplicationContextException(
                        "Fatal initialization error in servlet with name '" + getServletName() +
                        "': custom WebApplicationContext class [" + contextClass.getName() +
                        "] is not of type ConfigurableWebApplicationContext");
            }
            // <b> 创建 context 类的对象
            ConfigurableWebApplicationContext wac = (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

            // <c> 设置 environment、parent、configLocation 属性
            wac.setEnvironment(getEnvironment());
            wac.setParent(parent);
            String configLocation = getContextConfigLocation();
            if (configLocation != null) {
                wac.setConfigLocation(configLocation);
            }
            // <d> 配置和初始化 wac
            configureAndRefreshWebApplicationContext(wac);

            return wac;
        }
      ```
3. 如果未触发刷新事件，则调用 onRefresh(ApplicationContext context) 方法，主动触发刷新事件，该方法为空实现，交由子类 DispatcherServlet 去实现
4. 将 context 设置到 ServletContext 中

##### configureAndRefreshWebApplicationContext方法
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    // <1> 如果 wac 使用了默认编号，则重新设置 id 属性
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // The application context id is still set to its original default value
        // -> assign a more useful id based on available information
        // 情况一，使用 contextId 属性
        if (this.contextId != null) {
            wac.setId(this.contextId);
        }
        // 情况二，自动生成
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
        }
    }

    // <2> 设置 wac 的 servletContext、servletConfig、namespace 属性
    wac.setServletContext(getServletContext());
    wac.setServletConfig(getServletConfig());
    wac.setNamespace(getNamespace());
    // <3> 添加监听器 SourceFilteringListener 到 wac 中
    wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    // <4>
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
    }

    // <5> 执行处理完 WebApplicationContext 后的逻辑。目前是个空方法，暂无任何实现
    postProcessWebApplicationContext(wac);
    // <6> 执行自定义初始化 context
    applyInitializers(wac);
    // <7> 刷新 wac ，从而初始化 wac
    wac.refresh();
}

```
这里面关键的就是：**触发 wac 的刷新事件，执行初始化。**此处，就会进行一些的 Spring 容器的初始化工作，涉及到 Spring IOC 相关内容。

##### onRefresh方法
onRefresh(ApplicationContext context) 方法，当 Servlet WebApplicationContext 刷新完成后，触发 Spring MVC 组件的初始化，方法如下
```java
/**
 * Template method which can be overridden to add servlet-specific refresh work.
 * Called after successful context refresh.
 * <p>This implementation is empty.
 * @param context the current WebApplicationContext
 * @see #refresh()
 */
protected void onRefresh(ApplicationContext context) {
    // For subclasses: do nothing by default.
}

```
这是一个空方法，具体实现交付给其子类DispatcherServlet中实现了：
```java
// DispatcherServlet.java
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
    // 初始化 MultipartResolver
    initMultipartResolver(context);
    // 初始化 LocaleResolver
    initLocaleResolver(context);
    // 初始化 ThemeResolver
    initThemeResolver(context);
    // 初始化 HandlerMappings
    initHandlerMappings(context);
    // 初始化 HandlerAdapters
    initHandlerAdapters(context);
    // 初始化 HandlerExceptionResolvers 
    initHandlerExceptionResolvers(context);
    // 初始化 RequestToViewNameTranslator
    initRequestToViewNameTranslator(context);
    // 初始化 ViewResolvers
    initViewResolvers(context);
    // 初始化 FlashMapManager
    initFlashMapManager(context);
}

```
初始化九个组件，这里只是先提一下，在后续的文档中会进行分析
**onRefresh方法的触发有两种方式**：
- 方式一：如果refreshEventReceived 为 false，也就是未接收到刷新事件（防止重复初始化相关组件），则在 initWebApplicationContext 方法中直接调用
- 方式二：通过在 configureAndRefreshWebApplicationContext 方法中，触发 wac 的刷新事件

方式二为什么可以刷新呢？是因为：
先看到 configureAndRefreshWebApplicationContext 方法的第 3 步，添加了一个 SourceFilteringListener 监听器，如下
```java
// <3> 添加监听器 SourceFilteringListener 到 wac 中
wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
```

监听到相关事件后，会委派给 ContextRefreshListener 进行处理，它是 FrameworkServlet 的私有内部类，来看看它又是怎么处理的，代码如下：
```java
private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        FrameworkServlet.this.onApplicationEvent(event);
    }
}

```
直接将该事件委派给了 FrameworkServlet 的 onApplicationEvent 方法，如下：
```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 标记 refreshEventReceived 为 true
    this.refreshEventReceived = true;
    synchronized (this.onRefreshMonitor) {
        // 处理事件中的 ApplicationContext 对象，空实现，子类 DispatcherServlet 会实现
        onRefresh(event.getApplicationContext());
    }
}
```
就这样，通过一个事件驱动的方式完成了刷新。

**至此我们就了解到了传统SpringMVC初始化相关的大部分内容，关于注解的内容，后面结合SpringBoot我们再一起来看。**

### DispatcherServlet
再次回顾一下前文中的关于一个http请求流的执行流程：
https://nimbusk.cc/post/f5eb228d.html#default-10
Spring MVC 对各个组件的职责划分的比较清晰。
DispatcherServlet 负责协调，其他组件则各自做分内之事，互不干扰。
经过这样的职责划分，代码会便于维护。同时对于源码阅读者来说，也会很友好。可以降低理解源码的难度，使大家能够快速理清主逻辑。这一点值得我们学习。
#### 静态代码块
```java
private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";

private static final Properties defaultStrategies;

static {
    // Load default strategy implementations from properties file.
    // This is currently strictly internal and not meant to be customized by application developers.
    try {
        ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
    }
}
```
这部分会**从 `DispatcherServlet.properties` 文件中加载默认的组件实现类，**将相关配置加载到 defaultStrategies 中，文件如下：
```properties
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
  org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
  org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
  org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
  org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
  org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

#### 构造方法
```java
/** MultipartResolver used by this servlet. multipart 数据（文件）处理器 */
@Nullable
private MultipartResolver multipartResolver;

/** LocaleResolver used by this servlet. 语言处理器，提供国际化的支持 */
@Nullable
private LocaleResolver localeResolver;

/** ThemeResolver used by this servlet. 主题处理器，设置需要应用的整体样式 */
@Nullable
private ThemeResolver themeResolver;

/** List of HandlerMappings used by this servlet. 处理器匹配器，返回请求对应的处理器和拦截器们 */
@Nullable
private List<HandlerMapping> handlerMappings;

/** List of HandlerAdapters used by this servlet. 处理器适配器，用于执行处理器 */
@Nullable
private List<HandlerAdapter> handlerAdapters;

/** List of HandlerExceptionResolvers used by this servlet. 异常处理器，用于解析处理器发生的异常 */
@Nullable
private List<HandlerExceptionResolver> handlerExceptionResolvers;

/** RequestToViewNameTranslator used by this servlet. 视图名称转换器 */
@Nullable
private RequestToViewNameTranslator viewNameTranslator;

/** FlashMapManager used by this servlet. FlashMap 管理器，负责重定向时保存参数到临时存储（默认 Session）中 */
@Nullable
private FlashMapManager flashMapManager;

/** List of ViewResolvers used by this servlet. 视图解析器，根据视图名称和语言，获取 View 视图 */
@Nullable
private List<ViewResolver> viewResolvers;

public DispatcherServlet() {
    super();
    setDispatchOptionsRequest(true);
}

public DispatcherServlet(WebApplicationContext webApplicationContext) {
    super(webApplicationContext);
    setDispatchOptionsRequest(true);
}

```
值得注意的是，构造器里面默认初始化设置了一个true值：setDispatchOptionsRequest(true)
这个值呢，初始的时候默认是false的，主要作用：设置当前 Servlet 是否应将 HTTP OPTIONS 请求分派给 doService 该方法，即能否处理Options请求
**从 4.3版本 DispatcherServlet 开始，由于FrameworkServlet内置了对 OPTIONS 的支持，因此默认将此属性设置为 “true”。**

#### onRefresh方法
前文已经看过了，初始化每个组件的东西，在下一个小节中单独介绍

#### doService方法
doService(HttpServletRequest request, HttpServletResponse response)方法，DispatcherServlet 的处理请求的入口方法，代码如下：
```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // <1> 如果日志级别为 DEBUG，则打印请求日志
    logRequest(request);

    // Keep a snapshot of the request attributes in case of an include,
    // to be able to restore the original attributes after the include.
    // <2> 保存当前请求中相关属性的一个快照
    Map<String, Object> attributesSnapshot = null;
    if (WebUtils.isIncludeRequest(request)) {
      attributesSnapshot = new HashMap<>();
      Enumeration<?> attrNames = request.getAttributeNames();
      while (attrNames.hasMoreElements()) {
        String attrName = (String) attrNames.nextElement();
        // this.cleanupAfterInclude == true || 以org.springframework.web.servlet开头
        if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
          attributesSnapshot.put(attrName, request.getAttribute(attrName));
        }
      }
    }

    // Make framework objects available to handlers and view objects.
    // <3> 设置 Spring 框架中的常用对象到 request 属性中
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    // <4> FlashMap 的相关配置初始化，PS：处理重定向请求的时候会用到
    if (this.flashMapManager != null) {
      FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
      if (inputFlashMap != null) {
        request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
      }
      request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
      request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
    }

    try {
      // <5> 执行请求的分发
      doDispatch(request, response);
    }
    finally {
      // <6> 异步请求相关，通过：WebAsyncManager来管理的
      if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Restore the original attribute snapshot, in case of an include.
        if (attributesSnapshot != null) {
          restoreAttributesAfterInclude(request, attributesSnapshot);
        }
      }
    }
  }
```

其中：
1. 调用logRequest(HttpServletRequest request) 方法，如果日志级别为 DEBUG，则打印请求日志
2. 保存当前请求中相关属性的一个快照，作为异步处理的属性值，防止被修改，暂时忽略
3. 设置 Spring 框架中的常用对象到 request 属性中，例如 webApplicationContext、localeResolver、themeResolver
4. FlashMap 的相关配置：先简单的知道，这个是一个Session相关的缓存Map，在处理请求重定向的时候会用到。
5. 【重点】**调用 doDispatch(HttpServletRequest request, HttpServletResponse response) 方法，执行请求的分发**
6. 异步处理相关，暂时忽略

#### doDispatch方法 【核心】
开始之前，再次唠叨回顾一下前文中的SpringMVC请求分发图：
![SpringMVC工作流程](f5eb228d/SpringMVC工作流程.jpg)
接着我们来看一下这个方法的核心代码：
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    // <1> 获取异步管理器
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
        // <2> 检测请求是否为上传请求，如果是则通过 multipartResolver 将其封装成 MultipartHttpServletRequest 对象
        processedRequest = checkMultipart(request);
        multipartRequestParsed = (processedRequest != request);

        // Determine handler for the current request.
        // <3> 获得请求对应的 HandlerExecutionChain 对象（HandlerMethod 和 HandlerInterceptor 拦截器们）
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) { // <3.1> 如果获取不到，则根据配置抛出异常或返回 404 错误
          noHandlerFound(processedRequest, response);
          return;
        }

        // Determine handler adapter for the current request.
        // <4> 获得当前 handler 对应的 HandlerAdapter 对象
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // Process last-modified header, if supported by the handler.
        // <4.1> 处理有 Last-Modified 请求头的场景
        String method = request.getMethod();
        boolean isGet = "GET".equals(method);
        if (isGet || "HEAD".equals(method)) { // 这里对于HTTP的HEAD请求，SpringMVC处理和GET请求处理一致，没有单独起分支处理
          // 获取请求中服务器端最后被修改时间
          long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
          if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
            return;
          }
        }

        // <5> 前置处理 拦截器
        // 注意：该方法如果有一个拦截器的前置处理返回false，则开始倒序触发所有的拦截器的 已完成处理
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
          return;
        }

        // Actually invoke the handler.
        // <6> 真正的调用 handler 方法，也就是执行对应的方法，并返回视图
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
        
        // <7> 如果是异步
        if (asyncManager.isConcurrentHandlingStarted()) {
          return;
        }

        // <8> 无视图的情况下设置默认视图名称
        applyDefaultViewName(processedRequest, mv);
        // <9> 后置处理 拦截器
        mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
        dispatchException = ex; // <10> 记录异常
      }
      catch (Throwable err) {
        // As of 4.3, we're processing Errors thrown from handler methods as well,
        // making them available for @ExceptionHandler methods and other scenarios.
        dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      // <11> 处理正常和异常的请求调用结果
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
      // <12> 已完成处理 拦截器
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
      // <12> 已完成处理 拦截器
      triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", err));
    }
    finally {
      // <13.1> Async回调处理
      if (asyncManager.isConcurrentHandlingStarted()) {
        // Instead of postHandle and afterCompletion
        if (mappedHandler != null) {
          mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
        }
      }
      // <13.1> 如果是上传请求则清理资源，清理
      else {
        // Clean up any resources used by a multipart request.
        if (multipartRequestParsed) {
          cleanupMultipart(processedRequest);
        }
      }
    }
  }
```
我们来看一下每个步骤的功能：
1. 获得 WebAsyncManager 异步处理器，TODO 暂时忽略
2. 【文件】 调用 checkMultipart(HttpServletRequest request) 方法，检测请求是否为上传请求，**如果是则通过 multipartResolver 组件将其封装成 MultipartHttpServletRequest 对象，便于获取参数信息以及文件**
3. 【处理器匹配器】 调用 getHandler(HttpServletRequest request) 方法，通过 HandlerMapping 组件**获得请求对应的 HandlerExecutionChain 处理器执行链**，包含 HandlerMethod 处理器和 HandlerInterceptor 拦截器们。如果获取不到对应的执行链，则根据配置抛出异常或返回 404 错误。
4. 【处理器适配器】 调用 getHandlerAdapter(Object handler) 方法，获得当前处理器对应的 HandlerAdapter 适配器对象
5. 处理有 Last-Modified 请求头的场景，TODO 暂时忽略
6. 【拦截器】 调用 HandlerExecutionChain 执行链的 applyPreHandle(HttpServletRequest request, HttpServletResponse response) 方法，拦截器的前置处理；如果出现拦截器前置处理失败，则会**调用拦截器的已完成处理方法（倒序）**
7. 【重点】 调用 HandlerAdapter 适配器的 handle(HttpServletRequest request, HttpServletResponse response, Object handler) 方法，**真正的执行处理器，也就是执行对应的方法（例如我们定义的 Controller 中的方法），并返回视图**
8. 如果是异步，则直接 return，**注意**，还是会执行 finally 中的代码
9. 调用 applyDefaultViewName(HttpServletRequest request, ModelAndView mv) 方法，ModelAndView 不为空，但是没有视图，则设置默认视图名称，使用到了 viewNameTranslator 视图名称转换器组件
10. 【拦截器】 调用 HandlerExecutionChain 执行链的 applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) 方法，拦截器的后置处理（倒序）
11. 记录异常，**注意**，此处仅仅记录，不会抛出异常，而是统一交给 <11> 处理
12. 【处理执行结果】 调用 **processDispatchResult()** 方法，处理正常和异常的请求调用结果，包含页面渲染
13. 【拦截器】 如果上一步发生了异常，则调用triggerAfterCompletion()方法，即调用 HandlerInterceptor 执行链的triggerAfterCompletion()方法，拦截器已完成处理（倒序）
14. finally 代码块，异步处理，暂时忽略，如果是涉及到文件的请求，则清理相关资源

我们来总结一下：
![doDispatch执行流程](f5eb228d/doDispatch执行流程.jpg)

#### 九大组件
为了节省本文篇幅，这部分内容独立到单独文章中：
https://nimbusk.cc/post/b18da5cf.html

### SpringAOP
为了节省本文篇幅，这部分内容独立到单独文章中：
https://nimbusk.cc/post/a46a5ca9.html

### Spring事务
为了节省本文篇幅，这部分内容独立到单独文章中：
https://nimbusk.cc/post/18702fe9.html

---
## SpringBoot
为了节省本文篇幅，这部分内容独立到单独文章中：
https://nimbusk.cc/post/f52b8ac8.html

---
## SpringCloud
为了节省本文篇幅，这部分内容独立到单独文章中：
https://nimbusk.cc/post/3b599cdb.html

---
## Spring其它相关