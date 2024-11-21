---
title: SpringAOP细探
abbrlink: a46a5ca9
date: 2024-11-19 10:18:26
updated: 2024-11-20 14:27:05
tags:
  - Spring
  - Spring AOP
  - 切面
  - 动态代理
categories: Spring
---

## 一些概念
### 为什么要引入 AOP？
Java OOP 存在哪些局限性
- 静态化语言：类结构一旦定义，不容易被修改
- 侵入性扩展：通过继承或组合组织新的类结构
通过 AOP 我们可以把一些非业务逻辑的代码（比如安全检查、监控等代码）从业务中抽取出来，以非入侵的方式与原方法进行协同。
这样可以使得原方法更专注于业务逻辑，代码接口会更加清晰，便于维护。
<!-- more -->
### AOP 的使用场景
日志场景：
- 诊断上下文，如：log4j 或 logback 中的MDC
- 辅助信息，如：方法执行时间
统计场景：
- 方法调用次数
- 执行异常次数
- 数据抽样
- 数值累加
安防场景：
- 熔断，如：Netflix Hystrix
- 限流和降级：如：Alibaba Sentinel
- 认证和授权，如：Spring Security
- 监控，如：JMX

性能场景：
- 缓存，如 Spring Cache
- 超时控制

### AOP 中几个比较重要的概念
- **AspectJ**：切面，只是一个概念，没有具体的接口或类与之对应，是 Join point，Advice 和 Pointcut 的一个统称。
- **Join point**：连接点，指程序执行过程中的一个点，例如方法调用、异常处理等。在 Spring AOP 中，仅支持方法级别的连接点。
- **Advice**：通知，即我们定义的一个切面中的横切逻辑，有“around”，“before”和“after”三种类型。在很多的 AOP 实现框架中，Advice 通常作为一个拦截器，也可以包含许多个拦截器作为一条链路围绕着 Join point 进行处理。 
- **Pointcut**：切点，用于匹配连接点，一个 AspectJ 中包含哪些 Join point 需要由 Pointcut 进行筛选。
- **Introduction**：引介，让一个切面可以声明被通知的对象实现任何他们没有真正实现的额外的接口。例如可以让一个代理对象代理两个目标类。
- **Weaving**：织入，在有了连接点、切点、通知以及切面，如何将它们应用到程序中呢？没错，就是织入，在切点的引导下，将通知逻辑插入到目标方法上，使得我们的通知逻辑在方法调用时得以执行。
- **AOP proxy**：AOP 代理，指在 AOP 实现框架中实现切面协议的对象。在 Spring AOP 中有两种代理，分别是 JDK 动态代理和 CGLIB 动态代理。
- **Target object**：目标对象，就是被代理的对象。

### 有哪几种 AOP 框架
主流 AOP 框架：
- AspectJ：完整的 AOP 实现框架
- Spring AOP：非完整的 AOP 实现框架

Spring AOP 是基于 JDK 动态代理和 Cglib 提升实现的，**两种代理方式都属于运行时的一个方式**，所以它没有编译时的一个处理，那么因此 Spring 是通过 Java 代码实现的。
**AspectJ 自己有一个编译器**，在编译时期可以修改 .class 文件，在运行时也会进行处理。
Spring AOP 有别于其他大多数 AOP 实现框架，目的不是提供最完整的 AOP 实现（尽管 Spring AOP 相当强大）；
相反，其目的是在 AOP 实现和 Spring IoC 之间提供紧密的集成，以提供企业级核心特性。
Spring 将 Spring AOP 和 IoC 与 AspectJ 无缝集成，以实现 AOP 的所有功能都可以在一个 Spring 应用中。
这种集成不会影响 Spring AOP API 或 AOP Alliance API，保持向后兼容。

### 什么是 AOP 代理
代理模式是一种结构性设计模式，通过代理类为其他对象提供一种代理以控制对这个对象的访问。
AOP 代理是 AOP 框架中 AOP 的实现，主要分为静态代理和动态代理，如下：
- **静态代理**：代理类需要实现被代理类所实现的接口，同时持有被代理类的引用，新增处理逻辑，进行拦截处理，不过方法还是由被代理类的引用所执行。**静态代理通常需要由开发人员在编译阶段就定义好，不易于维护**。
	- 常用 OOP 继承和组合相结合
	- AspectJ，在编辑阶段会织入 Java 字节码，且在运行期间会进行增强。
- **动态代理**：不会修改字节码，而是**在 JVM 内存中根据目标对象新生成一个 Class 对象**，这个对象包含了被代理对象的全部方法，并且在其中进行了增强。
	- JDK 动态代理
	- 字节码提升，例如 CGLIB


### JDK 动态代理
**基于接口代理，通过反射机制生成一个实现代理接口的类，在调用具体方法时会调用 InvocationHandler 来处理。**

需要借助 JDK 的 `java.lang.reflect.Proxy` 来创建代理对象，调用 `Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) `方法创建一个代理对象，方法的三个入参分别是：
- **ClassLoader loader**：用于加载代理对象的 Class 类加载器
- **Class<?>[] interfaces**：代理对象需要实现的接口
- **InvocationHandler h**：代理对象的处理器
新生成的代理对象的 Class 对象会继承 Proxy，且实现所有的入参 interfaces 中的接口，在实现的方法中实际是调用入参 InvocationHandler 的 invoke(..) 方法。

### CGLIB 动态代理
**在运行时，非编译时，来创建一个新的 Class 对象，这种方式称之为字节码提升。**
在 Spring 内部有两个字节码提升的框架，ASM（过于底层，直接操作字节码）和 CGLIB（相对于前者更加简便）。
CGLIB 动态代理则是基于类代理（字节码提升），通过 ASM（Java 字节码的操作和分析框架）将被代理类的 class 文件加载进来，修改其字节码生成一个子类。
需要借助于 CGLIB 的 `org.springframework.cglib.proxy.Enhancer` 类来创建代理对象，设置以下几个属性：
- Class<?> superClass：被代理的类
- Callback callback：回调接口

### JDK 动态代理和 CGLIB 动态代理的差异
两者都是在 JVM 运行时期新创建一个 Class 对象，实例化一个代理对象，对目标类（或接口）进行代理。
- JDK 动态代理只能基于接口进行代理，生成的代理类实现了这些接口；
- 而 CGLIB 动态代理则是基于类进行代理的，生成的代理类继承目标类，但是不能代理被 final 修饰的类，也不能重写 final 或者 private 修饰的方法。
- CGLIB 动态代理比 JDK 动态代理复杂许多，性能也相对比较差。

### Spring AOP 和 AspectJ 有什么关联？
Spring AOP 和 AspectJ 都是 AOP 的实现框架，AspectJ 是 AOP 的完整实现，Spring AOP 则是部分实现。
AspectJ 有一个很好的编程模型，包含了注解的方式，也包含了特殊语法。Spring 认为 AspectJ 的实现在 AOP 体系里面是完整的，不需要在做自己的一些实现。
Spring AOP 整合 AspectJ 注解与 Spring IoC 容器，比 AspectJ 的使用更加简单，也支持 API 和 XML 的方式进行使用。
**不过 Spring AOP 仅支持方法级别的 Pointcut 拦截。**

### Spring AOP 中有哪些 Advice 类型
- **Around Advice**：围绕型通知器，需要主动去触发目标方法的执行，这样可以在触发的前后进行相关相关逻辑处理
- **Before Advice**：前置通知器，在目标方法执行前会被调用
- **After Advice**：后置通知器
	- **AfterReturning**：在目标方法执行后被调用（方法执行过程中出现异常不会被调用）
	- **After**：在目标方法执行后被调用（执行过程出现异常也会被调用）
	- **AfterThrowing**：执行过程中抛出异常后会被调用（如果异常类型匹配）

**执行顺序（Spring 5.2.7 之前的版本）**：Around “前处理” > Before > 方法执行 > **Around “后处理” > After > AfterReturning|AfterThrowing**

**执行顺序（Spring 5.2.7 开始）**：Around “前处理” > Before > 方法执行 > **AfterReturning|AfterThrowing > After > Around “后处理”**

### Spring AOP 中 Advisor 接口是什么
Advisor 是 Advice 的一个容器接口，与 Advice 是一对一的关系，它的子接口 PointcutAdvisor 是 Pointcut 和 Advice 的容器接口，将 Pointcut 过滤 Joinpoint 的能力和 Advice 进行整合，这样一来就将两者进行关联起来了。
Pointcut 提供 ClassFilter 和 MethedMatcher，分别支持筛选类和方法，通过 PointcutAdvisor 和 Advice 进行整合，可以说是形成了一个“切面”。

### Spring AOP 自动代理的实现
Spring IoC 中 Bean 的加载过程，在整个过程中，Bean 的实例化前和初始化后等生命周期阶段都提供了扩展点，会调用相应的 BeanPostProcessor 处理器对 Bean 进行处理。
当我们**开启了 AspectJ 自动代理**（例如通过 @EnableAspectJAutoProxy 注解），**则会往 IoC 容器中注册一个 AbstractAutoProxyCreator 自动代理对象**，该对象实现了几种 BeanPostProcessor，例如在每个 Bean 初始化后会被调用，解析出当前 Spring 上下文中所有的 Advisor（会缓存），如果这个 Bean 需要进行代理，则会通过 JDK 动态代理或者 CGLIB 动态代理创建一个代理对象并返回，所以得到的这个 Bean 实际上是一个代理对象。
这样一来，开发人员只需要配置好 AspectJ 相关信息，Spring 则会进行自动代理，和 Spring IoC 完美地整合在一起。

### Spring @EnableAspectJAutoProxy 的原理
使用了 `@EnableAspectJAutoProxy` 注解则会开启 Spring AOP 自动代理。
该注解上面有一个 `@Import(AspectJAutoProxyRegistrar.class)` 注解，`AspectJAutoProxyRegistrar` 实现了 `ImportBeanDefinitionRegistrar` 这个接口，在实现的方法中会注册一个 `AnnotationAwareAspectJAutoProxyCreator` 自动代理对象（如果没有注册的话），且将其优先级设置为最高，同时解析 `@EnableAspectJAutoProxy` 注解的配置并进行设置。
这个自动代理对象是一个 `BeanPostProcessor` 处理器，在 Spring 加载一个 Bean 的过程中，如果它需要被代理，那么会创建一个代理对象（JDK 动态代理或者 CGLIB 动态代理）。
除了注解的方式，也可以通过 `<aop:aspectj-autoproxy />` 标签开启 Spring AOP 自动代理，原理和注解相同，同样是注册一个自动代理对象。

### Spring Configuration Class CGLIB 提升与 AOP 类代理关系
在 Spring 底层 IoC 容器初始化后，会通过 `BeanDefinitionRegistryPostProcessor` 对其进行后置处理，其中会有一个 `ConfigurationClassPostProcessor` 处理器会对 `@Configuration` 标注的 `BeanDefinition` 进行处理，进行 CGLIB 提升，这样一来对于后续的 Spring AOP 工作就非常简单了，因为这个 Bean 天然就是一个 CGLIB 代理。
在 Spring 5.2 开始 @Configuration 注解中**新增了一个 proxyBeanMethods 属性（默认为 true）**，支持显示的配置是否进行 CGLIB 提升，毕竟进行 CGLIB 提升在启动过程会有一定的性能损耗，且创建的代理对象会占有一定的内存，通过该配置进行关闭，可以减少不必要的麻烦，对 Java 云原生有一定的提升。

### Sping AOP 应用到哪些设计模式
- 创建型模式：抽象工厂模式、工厂方法模式、构建器模式、单例模式、原型模式
- 结构型模式：适配器模式、组合模式、装饰器模式、享元模式、代理模式
- 行为型模式：模板方法模式、责任链模式、观察者模式、策略模式、命令模式、状态模式

### Spring AOP 在 Spring Framework 内部有哪些应用
- Spring 事件
- Spring 事务
- Spring 数据
- Spring 缓存抽象
- Spring 本地调度
- Spring 整合
- Spring 远程调用
基本涵盖了Spring众多核心分支

## Spring AOP 概述
先来看看下面这个AOP API相关总览：
![Spring_AOP_API总览](a46a5ca9/Spring_AOP_API.jpg)
导图分享链接：https://www.processon.com/view/link/673c6b265b61580d25932440?cid=673c39e7ede085743dbf0e5f

上面这张图片列出了 Spring AOP 涉及到的大部分 API，接下来依次简单看一下主要功能：
- **Joinpoint**：*连接点*，也就**是我们需要执行的目标方法**
- **Pointcut**：*切点*，提供 ClassFilter 类过滤器和 MethodMatcher 方法匹配器支持对类和方法进行筛选。
- **Advice**：*通知器*，一般都是 MethodInterceptor 方法拦截器，不是的话会通过 AdvisorAdapter 适配器转换成对应的 MethodInterceptor 方法拦截器。
- **Advisor**：*Advice 容器接口*，与 Advice 是一对一的关系；它的子接口 PointcutAdvisor 相当于一种 Pointcut 和 Advice 的容器，将 Pointcut 过滤 Joinpoint 的能力和 Advice 进行整合，这样一来就将两者关联起来了。
- **Advisor 适配器**：AdvisorAdapter 是 Advisor 的适配器，当筛选出能够应用于方法的所有 Advisor 后，需要获取对应的 Advice；
	- 如果不是 MethodInterceptor 类型，则需要通过 AdvisorAdapter 适配器转换成对应的 MethodInterceptor 方法拦截器。
- **AOP 代理对象**：AOP 代理对象，由 JDK 动态代理或者 CGLIB 动态代理创建的代理对象；
	- 选择哪种动态代理是通过 AopProxyFactory 代理工厂根据目标类来决定的。
- **AOP 代理配置**：AdvisedSupport 配置管理器中保存了对应代理对象的配置信息，例如满足条件的 Advisor 对象、TargetSource 目标类来源；
	- 在创建代理对象的过程，AdvisedSupport 扮演着一个非常重要的角色。
- **AOP 代理对象创建**：AOP 代理对象的创建分为手动模式和自动模式；不管哪种模式都和 AdvisedSupport 配置管理器有关；-
	- 手动模式就是通过 Spring AOP 提供的 API 进行创建；
	- 自动模式则是和 Spring IoC 进行整合，在 Bean 的加载过程中如果需要进行代理，则创建对应的代理对象；
- **AdvisorChainFactory**：Advisor 链路工厂
	- AdvisedSupport 配置管理器中保存了代理对象的所有 Advisor 对象，当拦截某个方法时，需要通过 AdvisorChainFactory 筛选出能够应用于该方法的 Advisor 们；
	- 另外还需要借助 AdvisorAdapter 适配器获取 Advisor 对应的 MethodInterceptor 方法拦截器，将这些 MethodInterceptor 有序地形成一条链路并返回。
- **IntroductionInfo**：引介接口，支持代理对象实现目标类没有真正实现的额外的接口；
	- 在 Advisor 的子接口 IntroductionAdvisor 中会继承这个 IntroductionInfo 接口，通过 @DeclareParents 注解定义的字段会被解析成 IntroductionAdvisor 对象。
- **AOP 代理目标对象来源**：目标类来源，和代理对象进行关联，用于获取被代理代理的目标对象；
	- 在代理对象中最终的方法执行都需要先通过 TargetSource 获取对应的目标对象，然后执行目标方法。

## Spring AOP 自动代理
在正式看AOP功能之前，我们先来回顾一下Bean的加载过程，整个过程中会调用相应的 BeanPostProcessor 对正在创建 Bean 进行处理，例如：
1. 在 Bean 的**实例化前**，会调用 `InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation(..)` 方法进行处理
2. 在 Bean **出现循环依赖的情况下**，会调用 `SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(..)` 方法对提前暴露的 Bean 进行处理
3. 在 Bean **初始化后**，会调用 `BeanPostProcessor#postProcessAfterInitialization(..)` 方法对初始化好的 Bean 进行处理

**Spring AOP 则是通过上面三个切入点进行创建代理对象，实现自动代理。**
- 在 Spring AOP 中**主要是**通过第 3 种 BeanPostProcessor 创建代理对象，**在 Bean 初始化后，也就是一个“成熟态”，然后再尝试是否创建一个代理对象；**
- 第 2 种方式是为了解决 Bean 循环依赖的问题，虽然 Bean 仅实例化还未初始化，**但是出现了循环依赖，不得不在此时创建一个代理对象；**
- 第 1 种方式是在 Bean **还没有实例化的时候就提前创建一个代理对象**（创建了则不会继续后续的 Bean 的创建过程），*例如 RPC 远程调用的实现，因为本地类没有远程能力，可以通过这种方式进行拦截*

**Spring AOP 自动代理的实现主要由 AbstractAutoProxyCreator 完成**，它实现了 `BeanPostProcessor、SmartInstantiationAwareBeanPostProcessor` 和 `InstantiationAwareBeanPostProcessor` 三个接口，那么这个类就是 Spring AOP 的入口，在这里将 Advice 织入我们的 Bean 中，创建代理对象。

### 如何激活 AOP 模块？
如何开启 Spring 的 AOP 模块，首先我们需要引入 spring-aop 和 aspectjweaver 两个模块，然后通过下面的方式开启 AOP：
- 添加 **`@EnableAspectJAutoProxy`** 注解
- 添加 **`<aop:aspectj-autoproxy />`** XML 配置

**备注**：在 Spring Boot 中使用 AOP 可以不需要上面两种配置，因为**在 Spring Boot 中当你引入上面两个模块后，默认开启**，可以看到下面这个配置类：
```java
package org.springframework.boot.autoconfigure.aop;
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class, AnnotatedElement.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = false)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
	public static class JdkDynamicAutoProxyConfiguration {}

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = true)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
	public static class CglibAutoProxyConfiguration {}
}

```
只要存在 EnableAspectJAutoProxy、Aspect、Advice、AnnotatedElement 四个 Class 对象，且 spring.aop.auto 配置为 true（没有配置则为 true），那么就会加载 AopAutoConfiguration 当前这个 Bean，而内部又使用了 @EnableAspectJAutoProxy 注解，那么表示开启 AOP。
### 类图
![AbstractAutoProxyCreator](a46a5ca9/AbstractAutoProxyCreator.jpg)
- 【重点】 `org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator`：AOP 自动代理的抽象类，完成主要的逻辑实现，提供一些骨架方法交由子类完成
- `org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator`：仅支持指定 `List<String> beanNames` 完成自动代理，需要指定 interceptorNames 拦截器
- 【重点】 `org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator`：支持从当前 Spring 上下文获取所有 Advisor 对象，存在能应用与 Bean 的 Advisor 则创建代理对象
- `org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator`：仅支持获取 Spring 内部的 Advisor 对象（**BeanDefinition 的角色为 ROLE_INFRASTRUCTURE**）
- `org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator`：支持配置前缀，只能获取名称已该前缀开头的 Advisor 对象
- 【重点】 `org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator`：支持按照 AspectJ 的方式对 Advisor 进行排序
- 【重点】 `org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`：支持从带有 @AspectJ 注解 Bean 中解析 Advisor 对象

我们主要关注上面【重点】的几个对象，因为 Sping AOP 推荐使用 AspectJ 里面的注解进行 AOP 的配置，你牢牢记住AbstractAutoProxyCreator这个自动代理类。

### AbstractAutoProxyCreator
AbstractAutoProxyCreator：AOP 自动代理的抽象类，完成主要的逻辑实现，提供一些骨架方法交由子类完成
#### 相关属性
```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

	/** 空对象，表示不需要进行代理 */
	@Nullable
	protected static final Object[] DO_NOT_PROXY = null;

	/**
	 * 空的数组，表示需要进行代理，但是没有解析出 Advice
	 * 查看 {@link BeanNameAutoProxyCreator#getAdvicesAndAdvisorsForBean} 就知道其用途
	 */
	protected static final Object[] PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS = new Object[0];

	/** DefaultAdvisorAdapterRegistry 单例，Advisor适配器注册中心 */
	private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();

	/** 是否冻结代理对象 */
	private boolean freezeProxy = false;

	/** 公共的拦截器对象 */
	private String[] interceptorNames = new String[0];

	/** 是否将 `interceptorNames` 拦截器放在最前面 */
	private boolean applyCommonInterceptorsFirst = true;

    /** 自定义的 TargetSource 创建器 */
	@Nullable
	private TargetSourceCreator[] customTargetSourceCreators;

	@Nullable
	private BeanFactory beanFactory;

	/**
	 * 保存自定义 {@link TargetSource} 对象的 Bean 的名称
	 */
	private final Set<String> targetSourcedBeans = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/**
	 * 保存提前创建代理对象的 Bean
	 * key：cacheKey（Bean 的名称或者 Class 对象）
	 * value：Bean 对象
	 *
	 * Spring AOP 的设计之初是让 Bean 在完全创建好后才完成 AOP 代理，如果出现了循环依赖，则需要提前（实例化后还未初始化）创建代理对象
	 * 那么需要先保存提前创建代理对象的 Bean，这样在后面可以防止再次创建代理对象
	 */
	private final Map<Object, Object> earlyProxyReferences = new ConcurrentHashMap<>(16);

	/**
	 * 保存代理对象的 Class 对象
	 * key：cacheKey（Bean 的名称或者 Class 对象）
	 * value：代理对象的 Class 对象（目标类的子类）
	 *
	 */
	private final Map<Object, Class<?>> proxyTypes = new ConcurrentHashMap<>(16);

	/**
	 * 保存是否需要创建代理对象的信息
	 * key：cacheKey（Bean 的名称或者 Class 对象）
	 * value：是否需要创建代理对象，false 表示不需要创建代理对象，true 表示已创建代理对象
	 */
	private final Map<Object, Boolean> advisedBeans = new ConcurrentHashMap<>(256);
}

```

#### getEarlyBeanReference 方法
getEarlyBeanReference(Object bean, String beanName)，用于处理早期暴露的对象，如果有必要的话会创建一个代理对象，该方法接口定义在 SmartInstantiationAwareBeanPostProcessor 中，实现如下：
```java
/**
 * 该方法对早期对象（提前暴露的对象，已实例化还未初始化）进行处理
 * 参考 {@link org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference }
 *
 * @param bean     the raw bean instance
 * @param beanName the name of the bean
 * @return 早期对象（可能是一个代理对象）
 */
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    // <1> 获取这个 Bean 的缓存 Key，默认为 Bean 的名称，没有则取其对应的 Class 对象
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    /*
     * <2> 将当前 Bean 保存至 earlyProxyReferences 集合（早期的代理应用对象）
     * 也就是说当这个 Bean 出现循环依赖了，在实例化后就创建了代理对象（如果有必要）
     */
    this.earlyProxyReferences.put(cacheKey, bean);
    // <3> 为这个 Bean 创建代理对象（如果有必要的话）
    return wrapIfNecessary(bean, beanName, cacheKey);
}

```

对于 getEarlyBeanReference(..) 方法在哪被调用，可能你已经忘记了，这里来回顾一下：
```java
// 在 AbstractAutowireCapableBeanFactory#doCreateBean 创建 Bean 的过程中，实例化后会执行下面步骤
/**
 * <3.2>
 * 创建一个 ObjectFactory 实现类，用于返回当前正在被创建的 `bean`，提前暴露，保存在 `singletonFactories` （**三级 Map**）缓存中
 *
 * 可以回到前面的 {@link AbstractBeanFactory#doGetBean#getSingleton(String)} 方法
 * 加载 Bean 的过程会先从缓存中获取单例 Bean，可以避免单例模式 Bean 循环依赖注入的问题
 */
addSingletonFactory(beanName,
        // ObjectFactory 实现类
        () -> getEarlyBeanReference(beanName, mbd, bean));

// 获取早期暴露的对象时候的处理，会调用 SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(..) 方法
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() // RootBeanDefinition 不是用户定义的（由 Spring 解析出来的）
            && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}

```
上面这个过程是为了处理循环依赖的问题，在 Bean 实例化后就提前暴露这个对象，如果真的出现了循环依赖，如果这个 Bean 需要进行代理，那么就不得不提前为它创建一个代理对象，虽然这个 Bean 还未初始化，不是一个“成熟态”。

#### postProcessBeforeInstantiation 方法
`postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`，Bean 的创建过程中实例化前置处理，也就是允许你在创建 Bean 之前进行处理。如果该方法返回的不为 null，后续 Bean 加载过程不会继续，也就是说这个方法可用于获取一个 Bean 对象。
**通常这里用于创建 AOP 代理对象，或者 RPC 远程调用的实现（因为本地类没有远程能力，可以通过这种方式进行拦截）。**
```java
/**
 * 在加载 Bean 的过程中，Bean 实例化的前置处理
 * 如果返回的不是 null 则不会进行后续的加载过程，也就是说这个方法用于获取一个 Bean 对象
 * 通常这里用于创建 AOP 代理对象，返回的对象不为 null，则会继续调用下面的 {@link this#postProcessAfterInitialization} 方法进行初始化后置处理
 * 参考 {@link org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation}
 *
 * @param beanClass the class of the bean to be instantiated
 * @param beanName  the name of the bean
 * @return 代理对象或者空对象
 */
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    // <1> 获取这个 Bean 的缓存 Key，默认为 Bean 的名称，没有则取其对应的 Class 对象
    Object cacheKey = getCacheKey(beanClass, beanName);

    // <2> 如果没有 beanName 或者没有自定义生成 TargetSource
    if (!StringUtils.hasLength(beanName) // 没有 beanName
            || !this.targetSourcedBeans.contains(beanName)) // 没有自定义生成 TargetSource
    {
        /*
         * <2.1> 已创建代理对象（或不需要创建），则直接返回 null，进行后续的 Bean 加载过程
         */
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        /*
         * <2.2>不需要创建代理对象，则直接返回 null，进行后续的 Bean 加载过程
         */
        if (isInfrastructureClass(beanClass) // 如果是 Spring 内部的 Bean（Advice、Pointcut、Advisor 或者 AopInfrastructureBean 标记接口）
                || shouldSkip(beanClass, beanName)) // 应该跳过
        {
            // 将这个 Bean 不需要创建代理对象的结果保存起来
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }
    /*
     * <3> 通过自定义 TargetSourceCreator 创建自定义 TargetSource 对象
     * 默认没有 TargetSourceCreator，所以这里通常都是返回 null
     */
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    /*
     * <4> 如果 TargetSource 不为空，表示需要创建代理对象
     */
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            // <4.1> 将当前 beanName 保存至集合，表示这个 Bean 已自定义生成 TargetSource 对象
            this.targetSourcedBeans.add(beanName);
        }
        // <4.2> 获取能够应用到当前 Bean 的所有 Advisor（已根据 @Order 排序）
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        // <4.3> 创建代理对象，JDK 动态代理或者 CGLIB 动态代理
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        // <4.4> 将代理对象的 Class 对象（目标类的子类）保存
        this.proxyTypes.put(cacheKey, proxy.getClass());
        // <4.5> 返回代理对象
        return proxy;
    }
    // <5> 否则，直接返回 null，进行后续的 Bean 加载过程
    return null;
}

```

这里先解释一下 TargetSource，**这个对象表示目标类的来源，用于获取代理对象的目标对象**，上面如果存在 TargetSourceCreator，表示可以创建自定义的 TargetSource，也就需要进行 AOP 代理。
默认情况下是没有 TargetSourceCreator 的，具体使用场景目前还没有接触过。

上面的 4.2 和 4.3 两个方法非常复杂，放在后面进行分析

同样对于 postProcessBeforeInstantiation(..) 方法在哪被调用，可能你已经忘记了，这里来回顾一下：
```java
// 在 AbstractAutowireCapableBeanFactory#createBean 创建 Bean 的过程中，开始前会执行下面步骤
/**
 * <3> 在实例化前进行相关处理，会先调用所有 {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation}
 * 注意，如果这里返回对象不是 `null` 的话，不会继续往下执行原本初始化操作，直接返回，也就是说这个方法返回的是最终实例对象
 * 可以通过这种方式提前返回一个代理对象，例如 AOP 的实现，或者 RPC 远程调用的实现（因为本地类没有远程能力，可以通过这种方式进行拦截）
 */
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
if (bean != null) {
    return bean;
}

protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        // 如果 RootBeanDefinition 不是用户定义的（由 Spring 解析出来的），并且存在 InstantiationAwareBeanPostProcessor 处理器
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                 // 实例化前置处理
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 后置处理
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}

```
上面过程是提供一种扩展点，可以让你在 Bean 创建之前进行相关处理，例如进行 AOP 代理，或者 RPC 远程调用的实现（因为本地类没有远程能力，可以通过这种方式进行拦截）。

#### postProcessAfterInitialization 方法
`postProcessAfterInitialization(@Nullable Object bean, String beanName)`，Bean 的初始化后置处理，在 Bean 初始化后，已经进入一个“成熟态”，那么此时就可以创建 AOP 代理对象了，如果有必要的话。
```java
/**
 * 在加载 Bean 的过程中，Bean 初始化的后置处理
 * 参考 {@link org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)}
 */
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    // <1> 如果 bean 不为空则进行接下来的处理
    if (bean != null) {
        // <1.1> 获取这个 Bean 的缓存 Key，默认为 Bean 的名称，没有则取其对应的 Class 对象
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        /*
         * <1.2> 移除 `earlyProxyReferences` 集合中保存的当前 Bean 对象（如果有的话）
         * 如果 earlyProxyReferences 集合中没有当前 Bean 对象，表示在前面没有创建代理对象
         */
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            // 这里尝试为这个 Bean 创建一个代理对象（如果有必要的话）
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    // <2> 直接返回 bean 对象
    return bean;
}

```

对于 postProcessAfterInitialization(..) 方法在哪被调用，可能你已经忘记了，这里来回顾一下
```java
// 在 AbstractAutowireCapableBeanFactory#doCreateBean#initializeBean 创建 Bean 的过程中，属性填充后会进行初始化，初始化后会执行下面的操作
/**
 * <4> **初始化**阶段的**后置处理**，执行所有 BeanPostProcessor 的 postProcessAfterInitialization 方法
 *
 * 在 {@link AbstractApplicationContext#prepareBeanFactory} 方法中会添加 {@link ApplicationListenerDetector} 处理器
 * 如果是单例 Bean 且为 ApplicationListener 类型，则添加到 Spring 应用上下文，和 Spring 事件相关
 */
if (mbd == null || !mbd.isSynthetic()) {
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException {
    Object result = existingBean;
    // 遍历所有 BeanPostProcessor
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        // 初始化的后置处理，返回 `current` 处理结果
        Object current = processor.postProcessAfterInitialization(result, beanName);
        // 处理结果为空，则直接返回 `result`
        if (current == null) {
            return result;
        }
        // 否则，`result` 复制 `current`
        result = current;
    }
    return result;
}

```
上面这个过程在 Bean 初始化后，提供一个扩展点允许对这个 Bean 进行后置处理，此时 Bean 进入一个 “成熟态”，在这里则可以进行 AOP 代理对象的创建

#### wrapIfNecessary 方法
`wrapIfNecessary(Object bean, String beanName, Object cacheKey)`，**该方法用于创建 AOP 代理对象，如果有必要的话**上面的 `getEarlyBeanReference(..)` 和 `postProcessAfterInitialization(..) `方法都会调用这个方法，如下：
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    /*
     * <1> 如果当前 Bean 已经创建过自定义 TargetSource 对象
     * 表示在上面的**实例化前置处理**中已经创建代理对象，那么直接返回这个对象
     */
    if (StringUtils.hasLength(beanName)
            && this.targetSourcedBeans.contains(beanName))
    {
        return bean;
    }
    // <2> `advisedBeans` 保存了这个 Bean 没有必要创建代理对象，则直接返回
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    /*
     * <3> 不需要创建代理对象，则直接返回当前 Bean
     */
    if (isInfrastructureClass(bean.getClass()) // 如果是 Spring 内部的 Bean（Advice、Pointcut、Advisor 或者 AopInfrastructureBean 标记接口）
            || shouldSkip(bean.getClass(), beanName)) // 应该跳过
    {
        // 将这个 Bean 不需要创建代理对象的结果保存起来
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    // <4> 获取能够应用到当前 Bean 的所有 Advisor（已根据 @Order 排序）
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    // <5> 如果有 Advisor，则进行下面的动态代理创建过程
    if (specificInterceptors != DO_NOT_PROXY) {
        // <5.1> 将这个 Bean 已创建代理对象的结果保存至 `advisedBeans`
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // <5.2> 创建代理对象，JDK 动态代理或者 CGLIB 动态代理
        // 这里传入的是 SingletonTargetSource 对象，可获取代理对象的目标对象（当前 Bean）
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        // <5.3> 将代理对象的 Class 对象（目标类的子类）保存
        this.proxyTypes.put(cacheKey, proxy.getClass());
        // <5.4> 返回代理对象
        return proxy;
    }

    // <6> 否则，将这个 Bean 不需要创建代理对象的结果保存起来
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    // <7> 返回这个 Bean 对象
    return bean;
}

```
#### 小结
至此，我们先总结一下：
**Spring AOP 自动代理的入口是 AbstractAutoProxyCreator 对象，其中自动代理的过程主要分为下面两步：**
1. 筛选出能够应用于当前 Bean 的 Advisor
2. 找到了合适 Advisor 则创建一个代理对象， JDK 动态代理或者 CGLIB 动态代理

### 筛选合适的通知器
接着上文来接着分析筛选合适的通知器的处理过程，包含 @AspectJ 等 AspectJ 注解的解析过程。**这里的“通知器”指的是 Advisor 对象。**
上文中wrapIfNecessary 方法中的第4步，调用 `getAdvicesAndAdvisorsForBean(..)` 方法，**获取能够应用到当前 Bean 的所有 Advisor（已根据 @Order 排序）**
而`getAdvicesAndAdvisorsForBean(..)`是一个抽象方法，具体实现交付于子类实现的
```java
	// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#getAdvicesAndAdvisorsForBean
	@Nullable
	protected abstract Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName,
			@Nullable TargetSource customTargetSource) throws BeansException;
```

#### 筛选出合适的 Advisor 的流程
1. 解析出当前 IoC 容器所有 Advisor 对象
	1. 获取当前 IoC 容器所有 Advisor 类型的 Bean
	2. 解析当前 IoC 容器中所有带有 @AspectJ 注解的 Bean，具体如下：
		- `@Before|@After|@Around|@AfterReturning|@AfterThrowing `注解的方法解析出对应的 PointcutAdvisor 对象，
		- 带有 `@DeclareParents` 注解的字段解析出 `IntroductionAdvisor` 对象
		- `@Around` -> `AspectJAroundAdvice`，实现了 `MethodInterceptor`
		- `@Before` -> `AspectJMethodBeforeAdvice`
		- `@After` -> `AspectJAfterAdvice`，实现了 `MethodInterceptor`
		- `@AfterReturning` -> `AspectJAfterAdvice`
		- `@AfterThrowing` -> `AspectJAfterThrowingAdvice`，实现了 `MethodInterceptor`
2. 筛选出能够应用于这个 Bean 的 Advisor 们，主要**通过 ClassFilter 类过滤器和 MethodMatcher 方法匹配器进行匹配**
3. 对筛选出来的 Advisor 进行扩展，例如子类会往首部添加一个 PointcutAdvisor 对象
4. 对筛选出来的 Advisor 进行排序
	- 不同的 AspectJ 根据 @Order 排序
	- 同一个 AspectJ 中不同 Advisor 的排序，优先级是：
		**`AspectJAfterThrowingAdvice > AspectJAfterReturningAdvice > AspectJAfterAdvice > AspectJAroundAdvice > AspectJMethodBeforeAdvice`**

主要涉及下面几个类：
- **AbstractAdvisorAutoProxyCreator**：支持从当前 Spring 上下文获取所有 Advisor 对象
- **AnnotationAwareAspectJAutoProxyCreator**：支持从带有 @AspectJ 注解 Bean 中解析 Advisor 对象
- **BeanFactoryAspectJAdvisorsBuilder**：Advisor 构建器，用于解析出当前 BeanFactory 中所有带有 @AspectJ 注解的 Bean 中的 Advisor
- **ReflectiveAspectJAdvisorFactory**：Advisor 工厂，用于解析 @AspectJ 注解的 Bean 中的 Advisor

#### AbstractAdvisorAutoProxyCreator
AbstractAdvisorAutoProxyCreator：**支持从当前 Spring 上下文获取所有 Advisor 对象，存在能应用与 Bean 的 Advisor 则创建代理对象**

#### AnnotationAwareAspectJAutoProxyCreator
AnnotationAwareAspectJAutoProxyCreator：支持从带有 @AspectJ 注解 Bean 中解析 Advisor 对象

#### BeanFactoryAspectJAdvisorsBuilder
BeanFactoryAspectJAdvisorsBuilder，Advisor 构建器，用于解析出当前 BeanFactory 中所有带有 @AspectJ 注解的 Bean 中的 Advisor

#### ReflectiveAspectJAdvisorFactory
ReflectiveAspectJAdvisorFactory，Advisor 工厂，用于解析 @AspectJ 注解的 Bean 中的 Advisor

### 创建代理对象
#### 创建代理对象的流程
1. 创建一个 ProxyFactory 代理工厂对象，设置需要创建的代理类的配置信息，例如 Advisor 数组和 TargetSource 目标类来源
2. 借助 DefaultAopProxyFactory 选择 JdkDynamicAopProxy（JDK 动态代理）还是 ObjenesisCglibAopProxy（CGLIB 动态代理）
	- 当 proxy-target-class 为 false 时，优先使用 JDK 动态代理，如果目标类没有实现可代理的接口，那么还是使用 CGLIB 动态代理
	- 如果为 true，优先使用 CGLIB 动态代理，如果目标类本身是一个接口，那么还是使用 JDK 动态代理
3. 通过 JdkDynamicAopProxy 或者 ObjenesisCglibAopProxy 创建一个代理对象
	- JdkDynamicAopProxy 本身是一个 InvocationHandler 实现类，通过 JDK 的 Proxy.newProxyInstance(..) 创建代理对象
	- ObjenesisCglibAopProxy 借助 CGLIB 的 Enhancer 创建代理对象，会设置 Callback 数组和 CallbackFilter 筛选器（选择合适 Callback 处理对应的方法），整个过程相比于 JDK 动态代理更复杂点，主要的实现在 DynamicAdvisedInterceptor 方法拦截器中

## AOP 两种代理对象的拦截处理

## AOP 注解驱动与XML配置


