---
title: Spring事务细探
abbrlink: 18702fe9
date: 2024-11-19 10:20:17
updated: 2024-11-21 15:54:54
tags:
  - Spring Transation
  - 事务
categories: Spring
---

## Spring 事务详解
Spring 事务里面分为“物理事务”和“逻辑事务”：

<!-- more -->

- 所谓的“物理事务”是：**指 JDBC 的事务**，上一次事务和本次事务之间是没有其他事务的，在执行一条命令（默认行为自动提交）都会产生一个事务，如果把 autocommit 设置为 false，需要主动 commit 才完成一个事务。
- 所谓的“逻辑事务”是：**Spring 对 JDBC 的一个抽象**，例如 Spring 默认的事务传播行为是 REQUIRED，当执行 @Transactional 注解标注的方法时，如果此时正处于一个事务（物理事务）中，那么加入到这个事务中，你可以理解为创建了一个“逻辑事务”，进行提交的时候不会执行 Connection 的 commit 方法，而是在外面的“物理事务”中进行 commit 时一并完成本次事务。
 
### Spring 事务的传播级别
- **REQUIRED**：默认传播级别，如果正处于一个事务中，则加入；否则，创建一个事务
- **SUPPORTS**：如果正处于一个事务中，则加入；否则，不使用事务
- **MANDATORY**：如果当前正处于一个事务中，则加入；否则，抛出异常
- **REQUIRES_NEW**：无论如何都会创建一个新的事务，如果正处于一个事务中，会先挂起，然后创建
- **NOT_SUPPORTED**：不使用事务，如果正处于一个事务中，则挂起，不使用事务
- **NEVER**：不使用事务，如果正处于一个事务中，则抛出异常
- **NESTED**：嵌套事务，如果正处于一个事务中，则创建一个事务嵌套在其中（MySQL 采用 SAVEPOINT 保护点实现的）；否则，创建一个事务

### Spring 事务的使用示例
#### SpringMVC下
引入 Spring 事务相关依赖后，在 Spring MVC 中有两种（XML 配置和注解）驱动 Spring 事务的方式，如下面所示：
##### 方式一
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <!-- 定义一个数据源 -->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">...</bean>
    <!-- 事务管理器 -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 指定数据源 -->
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <!-- 事务模块驱动，指定使用上面定义事务管理器，默认值为 transactionManager -->
    <tx:annotation-driven transaction-manager="txManager"/>
</beans>

```
##### 方式二
需要在 Spring 能扫描的一个 Bean 上**添加一个 @EnableTransactionManagement 注解**，然后添加一个 TransactionManagementConfigurer 实现类，如下：
```java

@Configuration
@EnableTransactionManagement
public class TransactionManagerConfig implements TransactionManagementConfigurer {

    @Resource
    DataSource dataSource;

    @Override
    public TransactionManager annotationDrivenTransactionManager() {
        // 返回一个事务管理器，设置数据源
        return new DataSourceTransactionManager(dataSource);
    }
}

```
此时你可以使用 @Transactional **注解标注在方法（或者类）上面，使得方法的执行处于一个事务中**，如下：
```java
@Transactional(propagation = Propagation.REQUIRED, rollbackFor = {Exception.class})
public void update(){
    try {
        // 数据库操作
    } catch (Exeception e){
        // 将事务状态设置为回滚
        TransactionInterceptor.currentTransactionStatus().setRollbackOnly();
    }
}

```

#### SpringBoot下
在 Spring Boot 中我们使用 @Transactional 注解的时候好像不需要 @EnableTransactionManagement 注解驱动 Spring 事务模块，这是为什么？
和 Spring AOP 的 @EnableAspectJAutoProxy 注解类似，**会有一个 TransactionAutoConfiguration 事务自动配置类**，我们一起来看看：
```java
@Configuration
@ConditionalOnClass(PlatformTransactionManager.class)
@AutoConfigureAfter({ JtaAutoConfiguration.class, HibernateJpaAutoConfiguration.class,
    DataSourceTransactionManagerAutoConfiguration.class,
    Neo4jDataAutoConfiguration.class })
@EnableConfigurationProperties(TransactionProperties.class)
public class TransactionAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public TransactionManagerCustomizers platformTransactionManagerCustomizers(
      ObjectProvider<List<PlatformTransactionManagerCustomizer<?>>> customizers) {
    return new TransactionManagerCustomizers(customizers.getIfAvailable());
  }

  @Configuration
  @ConditionalOnSingleCandidate(PlatformTransactionManager.class)
  public static class TransactionTemplateConfiguration {

    private final PlatformTransactionManager transactionManager;

    public TransactionTemplateConfiguration(
        PlatformTransactionManager transactionManager) {
      this.transactionManager = transactionManager;
    }

    @Bean
    @ConditionalOnMissingBean
    public TransactionTemplate transactionTemplate() {
      return new TransactionTemplate(this.transactionManager);
    }
  }

  @Configuration
  @ConditionalOnBean(PlatformTransactionManager.class)
  @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
  public static class EnableTransactionManagementConfiguration {
    @Configuration
    @EnableTransactionManagement(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
    public static class JdkDynamicAutoProxyConfiguration { }

    @Configuration
    @EnableTransactionManagement(proxyTargetClass = true)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
    public static class CglibAutoProxyConfiguration { }
  }
}

```

**只要存在 PlatformTransactionManager 这个 Class 对象**就会将这个 Bean 注册到 IoC 容器中，里面涉及到一些 @Conditional 注解，这里就不一一解释了。
可以看到其中会有 @EnableTransactionManagement 注解，是不是和在 Spring MVC 中以注解驱动 Spring 事务的方式一样，但是好像没有 PlatformTransactionManager 事务管理器。
别急，我们看到这个自动配置类上面会**有 @AutoConfigureAfter({DataSourceTransactionManagerAutoConfiguration.class}) 注解**，表示会先加载 `DataSourceTransactionManagerAutoConfiguration` 这个自动配置类，我们一起来看看：
```java
@Configuration
@ConditionalOnClass({ JdbcTemplate.class, PlatformTransactionManager.class })
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceTransactionManagerAutoConfiguration {

  @Configuration
  @ConditionalOnSingleCandidate(DataSource.class)
  static class DataSourceTransactionManagerConfiguration {

    private final DataSource dataSource;

    private final TransactionManagerCustomizers transactionManagerCustomizers;

    DataSourceTransactionManagerConfiguration(DataSource dataSource,
        ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
      this.dataSource = dataSource;
      this.transactionManagerCustomizers = transactionManagerCustomizers
          .getIfAvailable();
    }

    @Bean
    @ConditionalOnMissingBean(PlatformTransactionManager.class)
    public DataSourceTransactionManager transactionManager(
        DataSourceProperties properties) {
      DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(
          this.dataSource);
      if (this.transactionManagerCustomizers != null) {
        this.transactionManagerCustomizers.customize(transactionManager);
      }
      return transactionManager;
    }
  }
}
```
可以看到会注入一个 DataSourceTransactionManager 事务管理器，关联这个当前 DataSource 数据源对象
通过上面的使用示例我们可以注意到 @EnableTransactionManagement 注解可以驱动整个 Spring 事务模块

## 核心API
先简单介绍一下涉及到的一些主要的 API，对 Spring 事务的源码有一个印象，如下：
- Spring 事务 @Enable 模块驱动 - @EnableTransactionManagement
- Spring 事务注解 - @Transactional
- Spring 事务事件监听器 - @TransactionalEventListener
- Spring 事务定义 - TransactionDefinition
- Spring 事务状态 - TransactionStatus
- Spring 平台事务管理器 - PlatformTransactionManager
- Spring 事务代理配置 - ProxyTransactionManagementConfiguration
- Spring 事务 PointAdvisor 实现 - BeanFactoryTransactionAttributeSourceAdvisor
- Spring 事务 MethodInterceptor 实现 - TransactionInterceptor
- Spring 事务属性源 - TransactionAttributeSource

简单的介绍Spring事务：
1. 需要通过 `@EnableTransactionManagement` 注解驱动整个 Spring 事务模块
2. 可以通过 `@Transactional` 注解定义在某个类或者方法上面定义一个事务（传播性、隔离性等），开启事务
3. **ProxyTransactionManagementConfiguration 代理配置类用来定义一个 BeanFactoryTransactionAttributeSourceAdvisor 切面，是一个用于 Spring 事务的 AOP 切面**
4. Spring 事务底层就是通过 Spring AOP 实现的，可以在上面看到有一个 PointcutAdvisor 切面，关联的 Pointcut 内部有一个 TransactionAttributeSource 对象，会借助于 TransactionAnnotationParser 解析器解析 @Transactional 注解，将这个事务定义的一些属性封装成一个 TransactionDefinition 事务定义对象
5. Spring AOP 拦截处理在 TransactionInterceptor 事务拦截器中，先借助 PlatformTransactionManager 平台事务管理器创建 TransactionStatus 事务对象，里面包含了 Transaction 事务，将 autocommit 自动提交关闭，方法的执行也就处于一个事务中
6. **事务的相关属性会保存在许多 ThreadLocal 中**，例如 DataSource、Connection 和 SqlSession 等属性，交由一个 TransactionSynchronizationManager 事务同步管理器进行管理，**所以说 Spring 事务仅支持在一个线程中完成**

### @EnableTransactionManagement 注解驱动
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

  /**
   * 默认优先使用 JDK 动态代理
   */
  boolean proxyTargetClass() default false;

    /**
     * 默认使用 Spring AOP 代理模式
     */
  AdviceMode mode() default AdviceMode.PROXY;

  int order() default Ordered.LOWEST_PRECEDENCE;
}

// org.springframework.transaction.annotation.TransactionManagementConfigurationSelector
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

  @Override
  protected String[] selectImports(AdviceMode adviceMode) {
    switch (adviceMode) {
      // 默认使用代理模式
      case PROXY:
        return new String[]{
            // 注册一个 InfrastructureAdvisorAutoProxyCreator 对象，目的是创建代理对象
            AutoProxyRegistrar.class.getName(),
            // 【关键】注册一个 Spring 事务代理配置类
            ProxyTransactionManagementConfiguration.class.getName()};
      // 选择 AspectJ 模式
      case ASPECTJ:
        return new String[]{determineTransactionAspectClass()};
      default:
        return null;
    }
  }

  private String determineTransactionAspectClass() {
    return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
        TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
        TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
  }
}

```

可以看到默认情况下会注册两个 Bean
- `AutoProxyRegistrar`：注册一个 InfrastructureAdvisorAutoProxyCreator 对象，目的是创建代理对象，在讲解 Spring AOP 的时候讲述过，这里不再赘述
- `ProxyTransactionManagementConfiguration`：一个 Spring 务代理配置类

#### ProxyTransactionManagementConfiguration
ProxyTransactionManagementConfiguration，Spring 事务代理配置类，定义好一个 AOP 切面
```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

  @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
    // <1> 创建 PointcutAdvisor 对象，作为 @Transactional 注解的一个切面
    BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
    // <2> 【Pointcut】设置 AnnotationTransactionAttributeSource，被关联在 Pointcut 中
    // 借助于 TransactionAnnotationParser 解析器解析 @Transactional 注解
    advisor.setTransactionAttributeSource(transactionAttributeSource());
    // <3> 【Advice】设置 Advice 为 TransactionInterceptor 事务拦截器
    advisor.setAdvice(transactionInterceptor());
    if (this.enableTx != null) {
      advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
    }
    return advisor;
  }

  @Bean
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource();
  }

  @Bean
  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  public TransactionInterceptor transactionInterceptor() {
    // 创建 TransactionInterceptor 事务拦截器（MethodInterceptor 对象）
    TransactionInterceptor interceptor = new TransactionInterceptor();
    // 设置这个 AnnotationTransactionAttributeSource 对象，@Bean 注解标注的方法返回的都是同一个对象
    interceptor.setTransactionAttributeSource(transactionAttributeSource());
    if (this.txManager != null) {
      // 设置默认的事务管理器
      interceptor.setTransactionManager(this.txManager);
    }
    return interceptor;
  }
}

```

可以看到会注册三个 Bean：
1. BeanFactoryTransactionAttributeSourceAdvisor 切面，这个 PointcutAdvisor 对象**关联的 Pointcut 切点用于筛选 @Transactional 注解的方法（标注在类上也可以**），在关联的 Advice 中会进行事务的拦截处理
2. Advice 通知，就是一个 TransactionInterceptor 方法拦截器，关联着一个 AnnotationTransactionAttributeSource 对象
3. AnnotationTransactionAttributeSource 事务属性资源对象，**被 Pointcut 和 Advice 关联，用于解析 @Transactional 注解，在它的构造方法中会添加一个 SpringTransactionAnnotationParser 事务注解解析器，用于解析 @Transactional 注解**，如下：
  ```java
  // AnnotationTransactionAttributeSource.java
  public AnnotationTransactionAttributeSource() {
      this(true);
  }

  public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
      this.publicMethodsOnly = publicMethodsOnly;
      if (jta12Present || ejb3Present) {
          this.annotationParsers = new LinkedHashSet<>(4);
          this.annotationParsers.add(new SpringTransactionAnnotationParser());
          if (jta12Present) {
              this.annotationParsers.add(new JtaTransactionAnnotationParser());
          }
          if (ejb3Present) {
              this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
          }
      }
      else {
          this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
      }
  }
  ```

#### PointcutAdvisor 事务切面
org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor，Spring 事务切面，如下：
```java
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

  /**
   * 事务属性源对象，用于解析 @Transactional 注解
   */
  @Nullable
  private TransactionAttributeSource transactionAttributeSource;

  /**
   * Pointcut 对象，用于判断 JoinPoint 是否匹配
   */
  private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
    @Override
    @Nullable
    protected TransactionAttributeSource getTransactionAttributeSource() {
      return transactionAttributeSource;
    }
  };
    
  public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
    this.transactionAttributeSource = transactionAttributeSource;
  }

  public void setClassFilter(ClassFilter classFilter) {
    this.pointcut.setClassFilter(classFilter);
  }

  @Override
  public Pointcut getPointcut() {
    return this.pointcut;
  }
}

```
**这里设置的 TransactionAttributeSource 就是上面的 AnnotationTransactionAttributeSource 对象**，关联的 Pointcut 切点就是一个 TransactionAttributeSourcePointcut 对象
**也就是说通过 Pointcut 事务切点筛选出来的 Bean 会创建一个代理对象，方法的拦截处理则交由 Advice 完成**

#### Pointcut 事务切点
org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut，Spring 事务的 AOP 切点，如下：
```java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

  @Override
  public boolean matches(Method method, Class<?> targetClass) {
    // <1> 目标类是 Spring 内部的事务相关类，则跳过，不需要创建代理对象
    if (TransactionalProxy.class.isAssignableFrom(targetClass) ||
        PlatformTransactionManager.class.isAssignableFrom(targetClass) ||
        PersistenceExceptionTranslator.class.isAssignableFrom(targetClass)) {
      return false;
    }
    // <2 获取 AnnotationTransactionAttributeSource 对象
    TransactionAttributeSource tas = getTransactionAttributeSource();
    // <3> 解析该方法相应的 @Transactional 注解，并将元信息封装成一个 TransactionAttribute 对象
    // 且缓存至 AnnotationTransactionAttributeSource 对象中
    // <4> 如果有对应的 TransactionAttribute 对象，则表示匹配，需要进行事务的拦截处理
    return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
  }
}
```
判断这个方法是否需要被 TransactionInterceptor 事务拦截器进行拦截的过程如下：
1. 目标类是 Spring 内部的事务相关类，则跳过，不需要创建代理对象
2. 获取 `AnnotationTransactionAttributeSource` 对象
3. 解析该方法相应的 `@Transactional` 注解，并将元信息封装成一个 `TransactionAttribute` 对象，且缓存至`AnnotationTransactionAttributeSource` 对象中
4. 如果有对应的 TransactionAttribute 对象，则表示匹配，需要进行事务的拦截处理

第 3 步解析 **@Transactional 注解通过 AnnotationTransactionAttributeSource#getTransactionAttribute(..) 方法完成的**，我们一起来看看这个解析过程
#### AnnotationTransactionAttributeSource#getTransactionAttribute
```java
// AbstractFallbackTransactionAttributeSource.java
@Override
@Nullable
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
    // <1> java.lang.Object 内定义的方法跳过
    if (method.getDeclaringClass() == Object.class) {
        return null;
    }
    // First, see if we have a cached value.
    // <2> 获取缓存 Key，MethodClassKey 对象，关联 Method 和 Class 对象
    Object cacheKey = getCacheKey(method, targetClass);
    // <3> 尝试从缓存中获取该方法对应的 TransactionAttribute 对象
    TransactionAttribute cached = this.attributeCache.get(cacheKey);
    if (cached != null) {
        // Value will either be canonical value indicating there is no transaction attribute,
        // or an actual transaction attribute.
        // <3.1> 缓存中缓存的是一个空的 TransactionAttribute 对象
        // 表示没有相应的 @Transactional 注解，返回 `null`
        if (cached == NULL_TRANSACTION_ATTRIBUTE) {
            return null;
        }
        // <3.2> 返回缓存的 TransactionAttribute 对象
        else {
            return cached;
        }
    }
    // <4> 开始解析方法对应的 @Transactional 注解
    else {
        // We need to work it out.
        // <4.1> 解析该方法或者类上面的 @Transactional 注解，封装成 RuleBasedTransactionAttribute 对象
        // 优先从方法上面解析该注解，其次从类上解析该注解，没有的话返回的是 `null`
        TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
        // Put it in the cache.
        // <4.2> 如果是 `null`，则缓存一个空的 TransactionAttribute 对象
        if (txAttr == null) {
            this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
        }
        // <4.3> 否则，将该 TransactionAttribute 对象缓存
        else {
            String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
            if (txAttr instanceof DefaultTransactionAttribute) {
                // 设置方法的描述符（类名.方法名）
                ((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
            }
            this.attributeCache.put(cacheKey, txAttr);
        }
        // <4.4> 返回这个 TransactionAttribute 对象
        return txAttr;
    }
}

```
解析的过程如下：
1. Object 内定义的方法跳过
2. 获取缓存 Key，MethodClassKey 对象，关联 Method 和 Class 对象
3. 尝试从缓存中获取该方法对应的 TransactionAttribute 对象，如果有的话
    1. 如果缓存中缓存的是一个“空”的 TransactionAttribute 对象，表示没有相应的 @Transactional 注解，返回 null
    2. 否则，返回缓存的 TransactionAttribute 对象
4. 否则，开始解析方法对应的 @Transactional 注解
    1. 解析该方法或者类上面的 @Transactional 注解，封装成 RuleBasedTransactionAttribute 对象，优先从方法上面解析该注解，其次从类上解析该注解，没有的话返回的是 null
    2. 如果是 null，则缓存一个“空”的 TransactionAttribute 对象
    3. 否则，将该 TransactionAttribute 对象缓存
    4. 返回这个 TransactionAttribute 对象
注意，**这里解析出来的 TransactionAttribute 会进行缓存，后续在 TransactionInterceptor（Advice）中无需解析，直接取缓存即可**

#### 小节
在这个 PointcutAdvisor 切面关联着一个 Pointcut 切点，为 TransactionAttributeSourcePointcut 对象，内部有一个 AnnotationTransactionAttributeSource 事务属性资源对象。
**在这个切点判断某个方法是否需要进行事务处理时**，通过内部的 AnnotationTransactionAttributeSource 对象解析 @Transactional 注解（没有的话表示不匹配），解析过程**需要借助于 SpringTransactionAnnotationParser 解析器解析 @Transactional 注解，将这个事务定义的一些属性封装成一个 RuleBasedTransactionAttribute 事务定义对象**（实现了 TransactionDefinition 接口），并缓存

### TransactionInterceptor 事务拦截处理
通过 Pointcut 事务切点筛选出来的 Bean，会创建一个代理对象，Bean 内部肯定定义了 @Transactional 注解，如果是**类上定义的 @Transactional 注解，每个方法都需要进行事务处理**。
**代理对象的事务拦截处理在 TransactionInterceptor 拦截器中**，实现了 MethodInterceptor 方法拦截器，也就是实现了 Object invoke(MethodInvocation invocation) 这个方法
![TransactionInterceptor集成图](18702fe9/TransactionInterceptor.jpg)

关键代码：
```java
// TransactionInterceptor.java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    // 目标类
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // 在事务中执行方法调用器
    return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}

```
调用的invokeWithinTransaction如下：
```java
// TransactionAspectSupport.java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    TransactionAttributeSource tas = getTransactionAttributeSource();
    // <1> 获取 `@Transactional` 注解对应的 TransactionAttribute 对象（如果在 AnnotationTransactionAttributeSource 解析过则取缓存）
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    // <2> 获取 PlatformTransactionManager 事务管理器（可以指定，没有指定则获取默认的）
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    // <3> 获取方法的唯一标识，默认都是 `类名.方法名`
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // <4> 如果已有 `@Transactional` 注解对应的 TransactionAttribute 对象，或者是一个非回调偏向的事务管理器（默认不是）
    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        // <4.1> 创建 TransactionInfo 事务信息对象，绑定在 ThreadLocal 中
        // 包含一个 DefaultTransactionStatus 事务状态对象
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // <4.2> 继续执行方法调用器
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // <4.3> 如果捕获到异常，则在这里完成事务，进行回滚或者提交
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // <4.4> `finally` 语句块，释放 ThreadLocal 中的 TransactionInfo 对象，设置为上一个事务信息对象（没有的话为空）
            cleanupTransactionInfo(txInfo);
        }
        // <4.5> 正常情况，到这里完成事务
        commitTransactionAfterReturning(txInfo);
        // <4.6> 返回执行结果
        return retVal;
    }
    // <5> 否则，就是支持回调的事务管理器，编程式事务（回调偏向），暂时忽略
    else {
        // .....
    }
}

```
整个过程有点复杂，我们一步一步来看：
1. 获取 @Transactional 注解对应的 TransactionAttribute 对象（如果在 AnnotationTransactionAttributeSource 解析过则取缓存），在 Pointcut 事务切点中已经分析过
2. 获取 `PlatformTransactionManager` 事务管理器，需要指定，在 Spring Boot 中默认为 `DataSourceTransactionManager`
3. 获取方法的唯一标识，默认都是 类名.方法名
4. 如果已有 `@Transactional` 注解对应的 `TransactionAttribute` 对象，或者不是一个回调偏向的事务管理器（默认不是）
    1. 调用 `createTransactionIfNecessary`(..) 方法，创建 `TransactionInfo` 事务信息对象（包含一个 `DefaultTransactionStatus` 事务状态对象），绑定在 ThreadLocal 中
    2. 继续执行方法调用器（执行方法）
    3. 如果捕获到异常，则在这里完成事务，进行回滚或者提交，调用 `completeTransactionAfterThrowing`(..) 方法
    4. finally 语句块，释放 `ThreadLocal` 中的 `TransactionInfo` 对象，设置为上一个事务信息对象（没有的话为空）
    5. 正常情况，到这里完成事务，调用 `commitTransactionAfterReturning`(..) 方法
    6. 返回执行结果
5. 否则，就是支持回调的事务管理器，编程式事务（回调偏向），暂时忽略

我们继续来剖析一下上面的第4步的执行细节：
1. 为本地方法的执行创建一个事务，过程比较复杂，可以先理解为需要把 Connection 连接的 autocommit 关闭，然后根据 `@Transactional `注解的属性进行相关设置，例如根据事务的传播级别判断是否需要创建一个新的事务
2. 事务准备好了，那么继续执行方法调用器，也就是方法的执行
3. 捕获到异常，进行回滚，或者提交（异常类型不匹配）
4. 正常情况，走到这里就完成事务，调用 Connection 的 commit() 方法完成本次事务（不是一定会执行，因为可能是“嵌套事务”或者“逻辑事务”等情况）

### Spring创建事务过程
#### 创建事务
createTransactionIfNecessary(..) 方法，创建一个事务（如果有必要的话），如下：
```java
// TransactionAspectSupport.java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {
    // <1> 没有设置事务名称，则封装成一个 DelegatingTransactionAttribute 委托对象，支持返回一个事务名称（类名.方法名）
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }
    // <2> 获取一个 TransactionStatus 对象
    TransactionStatus status = null;
    if (txAttr != null) {
        // <2.1> 如果存在事务管理器
        if (tm != null) {
            // 从事务管理器中获取一个 TransactionStatus 事务状态对象（对事务的封装），该对象包含以下信息：
            // TransactionDefinition 事务定义、DataSourceTransactionObject 数据源事务对象（包括 DataSource 和 Connection）、
            // 是否是一个新的事务、是否是一个新的事务同步器、被挂起的事务资源对象
            status = tm.getTransaction(txAttr);
        }
        // <2.2> 否则，跳过
        else { }
    }
    // <3> 创建一个 TransactionInfo 事务信息对象，并绑定到 ThreadLocal 中
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```
过程如下：
1. 如果没有设置事务名称，则封装成一个 DelegatingTransactionAttribute 委托对象，支持返回一个事务名称（类名.方法名）
2. 获取一个 TransactionStatus 对象（对事务的封装）
  1. 如果存在事务管理器，Spring Boot 中默认为 DataSourceTransactionManager，则通过事务管理器根据 @Transactional 注解获取一个 TransactionStatus 事务状态对象，该对象是对事务的封装，包含了以下信息：
    - TransactionDefinition 事务定义
    - DataSourceTransactionObject 数据源事务对象（包括 DataSource 和 Connection）
    - 是否是一个新的事务
    - 是否是一个新的事务同步器
    - 被挂起的事务资源对象（如果有）
  2. 否则，跳过
3. 创建一个 TransactionInfo 事务信息对象，并绑定到 ThreadLocal 中，如下：
  ```java
  // TransactionAspectSupport.java
  protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
          @Nullable TransactionAttribute txAttr, String joinpointIdentification,
          @Nullable TransactionStatus status) {

      // <1> 创建一个 TransactionInfo 对象
      TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
      // <2> 如果有 @Transactional 注解的元信息
      if (txAttr != null) {
          // 设置 DefaultTransactionStatus 事务状态对象
          txInfo.newTransactionStatus(status);
      }
      else { }

      // We always bind the TransactionInfo to the thread, even if we didn't create
      // a new transaction here. This guarantees that the TransactionInfo stack
      // will be managed correctly even if no transaction was created by this aspect.
      // <3> 将当前 TransactionInfo 对象保存至 ThreadLocal 中
      txInfo.bindToThread();
      // <4> 返回这个 TransactionInfo 对象
      return txInfo;
  }
  ```
  **可以看到，即使没有创建事务，也会创建一个 TransactionInfo 对象，并绑定到 ThreadLocal 中**

#### PlatformTransactionManager管理事务
这个借口里面就是定义了三个方法
```java
public interface PlatformTransactionManager {

  TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
      throws TransactionException;
      
  void commit(TransactionStatus status) throws TransactionException;


  void rollback(TransactionStatus status) throws TransactionException;
}

```
**三个方法分别对应创建事务，提交，回滚三个操作，关于 Spring 事务也就这三个核心步骤了**
我们一个个看看
##### getTransaction方法
```java
// AbstractPlatformTransactionManager.java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
    // <1> 先从当前事务管理器中获取 DataSource 对象，然后尝试以它作为一个 Key 从一个 ThreadLocal 的 Map 中获取对应的 ConnectionHolder 连接对象
    // 会包装成一个 DataSourceTransactionObject 对象返回
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    // <2> 如果没有 @Transactional 注解对应的元信息，则创建一个默认的 TransactionDefinition 对象
    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    // <3> 如果上面 `transaction` 数据源事务对象已有 Connection 连接，且正处于一个事务中，表示当前线程已经在一个事务中了
    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        // <3.1> 根据 Spring 事务传播级别进行不同的处理，同时创建一个 DefaultTransactionStatus 事务状态对象，包含以下信息：
        // TransactionDefinition 事务定义、DataSourceTransactionObject 数据源事务对象、
        // 是否需要新创建一个事务、是否需要一个新的事务同步器、被挂起的事务资源对象
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // <4> 否则，当前线程没有事务

    // 超时不能小于默认值
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // <4.1> 如果是 **MANDATORY** 事务传播级别（当前线程已经在一个事务中，则加入该事务，否则抛出异常），因为当前线程没有事务，此时抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    // <4.2> 否则，如果事务传播级别为 **REQUIRED|REQUIRES_NEW|NESTED**
    else if (
        // 如果当前线程已经在一个事务中，则加入该事务，否则新建一个事务（默认）
        definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
        // 无论如何都会创建一个新的事务，如果当前线程已经在一个事务中，则挂起当前事务，创建一个新的事务
        definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
        // 执行一个嵌套事务
        definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) 
    {
        // <4.2.1> 创建一个“空”的挂起资源对象
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            // 是否需要新的事务同步器，默认为 true
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            // <4.2.2> 创建一个 DefaultTransactionStatus 事务状态对象，设置相关属性
            // 这里 `newTransaction` 参数为 `true`，表示是一个新的事务
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // <4.2.3> 【关键】执行 begin 操作，如果没有 Connection 数据库连接，则通过 DataSource 创建一个新的连接
            // 设置 Connection 的隔离级别、是否只读，并执行 Connection#setAutoCommit(false) 方法，不自动提交
            // 同时将 DataSource（数据源对象）和 ConnectionHolder（数据库连接持有者）保存至 ThreadLocal 中
            doBegin(transaction, definition);
            // <4.2.4> 借助 TransactionSynchronizationManager 事务同步管理器设置相关 ThreadLocal 变量
            prepareSynchronization(status, definition);
            // <4.2.5> 返回上面创建的 DefaultTransactionStatus 事务状态对象
            return status;
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    // <4.3> 否则，创建一个“空”的事务状态对象
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + definition);
        }
        // 是否需要新的事务同步器，默认为 true
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        // 创建一个 DefaultTransactionStatus 事务状态对象，设置相关属性，这里也是一个新的事务
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}

```

###### doBegin 方法
```java
// AbstractPlatformTransactionManager.java
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        // <1> 如果没有 Connection 数据库连接，或者连接处于事务同步状态
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            // <1.1> 通过 DataSource 创建一个 Connection 数据库连接
            Connection newCon = obtainDataSource().getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            // <1.2> 重置 ConnectionHolder 连接持有者，封装刚创建的数据库连接
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        // <2> 设置ConnectionHolder 连接持有者处于事务同步中
        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        // <3> 获取 Connection 数据库连接
        con = txObject.getConnectionHolder().getConnection();

        // <4> 设置 Connection 是否只读和事务隔离性
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        // <5> 保存之前的事务隔离级别（被挂起的事务）
        txObject.setPreviousIsolationLevel(previousIsolationLevel);

        // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
        // so we don't want to do it unnecessarily (for example if we've explicitly
        // configured the connection pool to set it already).
        // <6> 将自动提交关闭
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        // <7> 如果需要强制设置只读（默认不需要），且连接本身是只读的，则这里提前设置事务的只读性
        prepareTransactionalConnection(con, definition);
        // <8> 当前 Connection 数据库连接正处于一个事务中
        txObject.getConnectionHolder().setTransactionActive(true);

        // <9> 设置超时时间
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        // <10> 如果是新的事务，则将 DataSource（数据源对象）和 ConnectionHolder（数据库连接持有者）保存至 ThreadLocal 中
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
        }
    }
    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}

```

#### 小结
Spring 创建事务的过程主要分为两种情况，**当前线程不处于一个事务中和正处于一个事务中**，两种情况都需要根据事务的传播级别来做出不同的处理。
创建一个事务的核心就是调用 Connection#setAutocommit(false) 方法，将自动提交关闭，这样一来，就可以在一个事务中根据行为作出相应的操作，例如出现异常进行回滚，没有问题则进行提交
- 当前线程**不处于**一个事务中：
  - 如果是**MANDATORY**传播级别，则抛出异常
  - 否则，如果是 **REQUIRED | REQUIRES_NEW | NESTED** 传播级别，则“创建”一个事务，将数据库的 commit 设置为 false，此时会设置事务状态里面的 newTransaction 属性 true，表示是一个新的事务
  - 否则，创建一个“空”的事务状态对象，也就是不使用事务
- 当前线程**正处于**一个事务中：
  - 如果是**NEVER**传播级别，则抛出异常
  - 否则，如果是**NOT_SUPPORTED**传播级别，则将当前事务挂起，然后创建一个“空”的事务状态对象，也就是不使用事务
  - 否则，如果是R**EQUIRES_NEW** 传播级别，则将当前事务挂起，然后“创建”一个事务，将数据库的 commit 设置为 false，此时会设置事务状态里面的 newTransaction 属性 true，表示是一个新的事务；同时还保存了被挂起的事务相关资源，在本次事务结束后会唤醒它
  - 否则，如果是 **NESTED **传播级别，则沿用当前事务，就是设置事务状态里面的 newTransaction 属性 false，表示不是一个新的事务，不过会调用 Connection#setSavepoint(String) 方法创建一个 **SAVEPOINT** 保存点，相当于嵌套事务
  - 否则，就是 **SUPPORTS | REQUIRED** 传播级别，沿用当前事务，就是设置事务状态里面的 newTransaction 属性 false，表示不是一个新的事务

注意到 DefaultTransactionStatus 事务状态对象有一个 `newTransaction` 属性，通过它可以知道是否是一个新的事务，在后续的 commit 和 rollback 有着关键的作用

### Spring提交事务
在 TransactionInterceptor 事务拦截处理过程中，如果方法的执行过程没有抛出异常，那么此时我们是不是需要调用 Connection#commit() 方法，提交本次事务
```java
// TransactionAspectSupport.java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // 通过 DataSourceTransactionManager 提交当前事务
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}

```

#### commit 方法
```java
// AbstractPlatformTransactionManager.java
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    // <1> 如果事务已完成，此时又提交，则抛出异常
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    // <2> 事务明确标记为回滚
    if (defStatus.isLocalRollbackOnly()) {
        // 进行回滚过程（预料之中）
        processRollback(defStatus, false);
        return;
    }

    // <3> 判断全局回滚时是否需要提交（默认不需要），且当前事务为全局回滚
    // 例如 **REQUIRED** 传播级别，当已有一个事务时则加入其中，此时如果抛出异常，则会设置为全局回滚，那么当事务进行提交时，对于整个事务都需要回滚
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        // 进行回滚过程（预料之外）
        processRollback(defStatus, true);
        return;
    }

    // <4> 执行提交事务
    processCommit(defStatus);
}

```

#### processCommit 方法
```java
// AbstractPlatformTransactionManager.java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;
        try {
            boolean unexpectedRollback = false;
            // <1> 进行三个前置操作
            prepareForCommit(status); // <1.1> 在 Spring 中为空方法
            // <1.2> 调用 TransactionSynchronization#beforeCommit 方法
            // 例如在 Mybatis-Spring 中的 SqlSessionSynchronization 会调用其 SqlSession#commit() 方法，提交批量操作，刷新缓存
            triggerBeforeCommit(status);
            // <1.3> 调用 TransactionSynchronization#beforeCompletion 方法
            // 由 Spring 事务托管，不是真的关闭连接，从 ThreadLocal 中删除 DataSource 和 ConnectionHolder 的映射关系
            // 例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，
            // 且调用其 SqlSession#close() 方法
            triggerBeforeCompletion(status);
            // <2> 标记三个前置操作已完成
            beforeCompletionInvoked = true;

            // <3> 有保存点，即嵌套事务
            if (status.hasSavepoint()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                // <3.1> 释放保存点，等外层的事务进行提交
                status.releaseHeldSavepoint();
            }
            // <4> 否则，如果是一个新的事务
            else if (status.isNewTransaction()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                // <4.1> 提交事务，执行 Connection#commit() 方法
                doCommit(status);
            }
            // <5> 否则，在事务被标记为全局回滚的情况下是否提前失败（默认为 false）
            else if (isFailEarlyOnGlobalRollbackOnly()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
            }

            // Throw UnexpectedRollbackException if we have a global rollback-only
            // marker but still didn't get a corresponding exception from commit.
            // 如果全局标记为仅回滚，但是提交时没有得到异常，则这里抛出异常
            // 目的是需要回滚
            if (unexpectedRollback) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        catch (UnexpectedRollbackException ex) {
            // can only be caused by doCommit
            // 触发完成后事务同步，状态为回滚
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        }
        // 事务异常
        catch (TransactionException ex) {
            // can only be caused by doCommit
            // 提交失败回滚
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            }
            // 触发完成后回调，事务同步状态为未知
            else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }
        // 运行时异常或者其它异常
        catch (RuntimeException | Error ex) {
            // 如果上面三个前置步骤未完成，调用最后一个前置步骤，即调用 TransactionSynchronization#beforeCompletion 方法
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            // 提交异常回滚
            doRollbackOnCommitException(status, ex);
            throw ex;
        }

        // Trigger afterCommit callbacks, with an exception thrown there
        // propagated to callers but the transaction still considered as committed.
        try {
            // <6> 触发提交后的回调，调用 TransactionSynchronization#afterCommit 方法
            // JMS 会有相关操作，暂时忽略
            triggerAfterCommit(status);
        }
        finally {
            // <7> 触发完成后的回调，事务同步状态为已提交，调用 TransactionSynchronization#afterCompletion 方法
            // 例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，
            // 且调用其 SqlSession#close() 方法，解决可能出现的跨线程的情况
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        // <8> 在完成后清理，清理相关资源，“释放”连接，唤醒被挂起的资源
        cleanupAfterCompletion(status);
    }
}

```

提交过程如下：
1. 进行三个前置操作
    1. 准备工作，在 Spring 中为空方法
    2. 调用 TransactionSynchronization#beforeCommit 方法，例如在 Mybatis-Spring 中的 SqlSessionSynchronization 会调用其 SqlSession#commit() 方法，提交批量操作，刷新缓存
    3. 调用 TransactionSynchronization#beforeCompletion 方法，由 Spring 事务托管，不是真的关闭连接，从 ThreadLocal 中删除 DataSource 和 ConnectionHolder 的映射关系；例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，且调用其 SqlSession#close() 方法
2. 标记三个前置操作已完成
3. 如果有保存点，即**嵌套事务**，则调用 Connection#releaseSavepoint(Savepoint) 方法释放保存点，等外层的事务进行提交
4. 否则，如果是一个新的事务，根据之前一直提到的 newTransaction 属性进行判断是否是一个新的事务
    1. 提交事务，执行 Connection#commit() 方法
    ```java
    // DataSourceTransactionManager.java
    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        Connection con = txObject.getConnectionHolder().getConnection();
        if (status.isDebug()) {
            logger.debug("Committing JDBC transaction on Connection [" + con + "]");
        }
        try {
            con.commit();
        }
        catch (SQLException ex) {
            throw new TransactionSystemException("Could not commit JDBC transaction", ex);
        }
    }

    ```
5. 否则，在事务被标记为全局回滚的情况下是否提前失败（默认为 false）
6. 触发提交后的回调，调用 TransactionSynchronization#afterCommit 方法，JMS 会有相关操作，暂时忽略
7. 触发完成后的回调，事务同步状态为已提交，调用 TransactionSynchronization#afterCompletion 方法，例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，且调用其 SqlSession#close() 方法，解决可能出现的跨线程的情况
8. 在完成后清理，清理相关资源，“释放”连接，唤醒被挂起的资源，如下：
  ```java
  // AbstractPlatformTransactionManager.java
  private void cleanupAfterCompletion(DefaultTransactionStatus status) {
      // <1> 设置为已完成
      status.setCompleted();
      // <2> 如果是一个新的事务同步器
      if (status.isNewSynchronization()) {
          // 清理事务管理器中的 ThreadLocal 相关资源，包括事务同步器、事务名称、只读属性、隔离级别、真实的事务激活状态
          TransactionSynchronizationManager.clear();
      }
      // <3> 如果是一个新的事务
      if (status.isNewTransaction()) {
          // 清理 Connection 资源，例如释放 Connection 连接，将其引用计数减一（不会真的关闭）
          // 如果这个 `con` 是中途创建的，和 ThreadLocal 中的不一致，则需要关闭
          doCleanupAfterCompletion(status.getTransaction());
      }
      // <4> 如果之前有被挂起的事务，则唤醒
      if (status.getSuspendedResources() != null) {
          Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
          // 唤醒被挂起的事务和资源，重新将 DataSource 和 ConnectionHolder 的映射绑定到 ThreadLocal 中
          // 将之前挂起的相关属性重新设置到 ThreadLocal 中
          resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
      }
  }
  ```

整个过程在提交事务的前后会进行相关处理，例如清理资源；对于嵌套事务，这里会释放保存点，等外层的事务进行提交；对于新的事务，这里会调用Connection#commit()方法提交事务；其他情况不会真的提交事务，在这里仅清理相关资源，唤醒被挂起的资源

### Spring回滚事务
在 TransactionInterceptor 事务拦截处理过程中，如果方法的执行过程抛出异常，那么此时我们是不是需要调用 Connection#roback() 方法，对本次事务进行回滚
```java
// TransactionAspectSupport.java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // 如果 @Transactional 配置了需要对那些异常进行回退，则需要判断抛出的异常是否匹配
        // 没有配置的话只处理 RuntimeException 或者 Error 两种异常
        // <1> 如果异常类型匹配成功，则进行回滚
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                // 回滚操作
                // 嵌套事务，回滚到保存点；否则，新事务，回滚；否则，加入到的一个事务，设置为需要回滚
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
        }
        // <2> 否则，不需要回滚，提交事务
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            try {
                // 提交操作
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
        }
    }
}

// AbstractPlatformTransactionManager.java
@Override
public final void rollback(TransactionStatus status) throws TransactionException {
   // <1> 如果事务已完成，此时又回滚，则抛出异常
   if (status.isCompleted()) {
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   // <2> 进行回滚（预料之中）
   processRollback(defStatus, false);
}

// AbstractPlatformTransactionManager.java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    try {
        boolean unexpectedRollback = unexpected;
        try {
            // <1> 调用 TransactionSynchronization#beforeCompletion 方法
            // 由 Spring 事务托管，不是真的关闭连接，从 ThreadLocal 中删除 DataSource 和 ConnectionHolder 的映射关系
            // 例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，
            // 且调用其 SqlSession#close() 方法
            triggerBeforeCompletion(status);

            // <2> 如果有 Savepoint 保存点
            if (status.hasSavepoint()) {
                // 回滚到保存点，调用 Connection#rollback(Savepoint) 方法
                status.rollbackToHeldSavepoint();
            }
            // <3> 否则，如果是新的事务，例如传播级别为 **REQUIRED_NEW** 则一定是一个新的事务
            else if (status.isNewTransaction()) {
                // 事务回滚，调用 Connection#rollback() 方法
                doRollback(status);
            }
            // <4> 否则，不是新的事务也没有保存点，那就是加入到一个已有的事务这种情况，例如 **REQUIRED** 传播级别，如果已存在一个事务，则加入其中
            else {
                // Participating in larger transaction
                if (status.hasTransaction()) {
                    // 如果已经标记为回滚，或当加入事务失败时全局回滚（默认 true）
                    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        // 设置当前 ConnectionHolder#rollbackOnly 为 true
                        // 在这个事务提交的时候进行回滚
                        doSetRollbackOnly(status);
                    }
                    else { }
                }
                else { }
                // Unexpected rollback only matters here if we're asked to fail early
                // 在事务被标记为全局回滚的情况下是否提前失败
                // 默认为 false，表示不希望抛出异常
                if (!isFailEarlyOnGlobalRollbackOnly()) {
                    // 那么设置 unexpectedRollback 为 false
                    unexpectedRollback = false;
                }
            }
        }
        // 运行时异常或者其它异常
        catch (RuntimeException | Error ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }

        // 触发完成后的回调，事务同步状态为已提交，调用 TransactionSynchronization#afterCompletion 方法
        // 例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，
        // 且调用其 SqlSession#close() 方法，解决可能出现的跨线程的情况
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

        // Raise UnexpectedRollbackException if we had a global rollback-only marker
        // 通过上面可以看到，通常情况这里不会抛出异常
        if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
    }
    finally {
        // 在完成后清理，清理相关资源，“释放”连接，唤醒被挂起的资源
        cleanupAfterCompletion(status);
    }
}


```

回滚的过程如下：
1. 调用 TransactionSynchronization#beforeCompletion 方法，由 Spring 事务托管，不是真的关闭连接，从 ThreadLocal 中删除 DataSource 和 ConnectionHolder 的映射关系；例如在 Mybatis-Spring 中的 SqlSessionSynchronization 中，会从 ThreadLocal 中删除 SqlSessionFactory 和 SqlSessionHolder 的映射关系，且调用其 SqlSession#close() 方法
2. 如果有 Savepoint 保存点，也就是嵌套事务，则回滚到保存点，调用 Connection#rollback(Savepoint) 方法回滚到保存点，再调用Connection#releaseSavepoint(Savepoint) 方法释放该保存点
3. 否则，如果是新的事务，例如传播级别为 REQUIRED_NEW 则一定是一个新的事务，则对当前事务进行回滚，调用 Connection#rollback() 方法，如下：
  ```java
  // DataSourceTransactionManager.java
  @Override
  protected void doRollback(DefaultTransactionStatus status) {
      DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
      Connection con = txObject.getConnectionHolder().getConnection();
      try {
          con.rollback();
      }
      catch (SQLException ex) {
          throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
      }
  }

  ```
4. 否则，不是新的事务也没有保存点，那就是加入到一个已有的事务这种情况，例如 REQUIRED 传播级别，如果已存在一个事务，则加入其中
  1. 如果已经标记为回滚，或当加入事务失败时全局回滚（默认 true），那么设置 ConnectionHolder 的 rollbackOnly 为 true，也就是标记需要全局回滚，对应到前面“提交事务”的时候会判断是否标记为全局回滚，标记了则进行回滚，而不是提交
可以看到，对于默认的REQUIRED事务传播级别，如果已有一个事务（“物理事务”），则加入到当前事务中（相当于创建了一个“逻辑事务”），当这个“逻辑事务”出现异常时，整个事务（包括外面的“物理事务”）都需要回滚

## 总结
本文对 Spring 事务做了比较详细的讲解，我们通过 @EnableTransactionManagement 注解驱动整个 Spring 事务模块，此时会往 IoC 注入一个 PointcutAdvisor 事务切面，关联了一个 TransactionAttributeSourcePointcut（Pointcut）事务切点和一个 TransactionInterceptor（Advice）事务拦截器。

这个 TransactionAttributeSourcePointcut（Pointcut）事务切点，它里面关联了一个 AnnotationTransactionAttributeSource 事务属性资源对象，通过它解析这个方法（或者类）上面的 @Transactional 注解；
底层需要借助 SpringTransactionAnnotationParser 进行解析，解析出一个 TransactionAttribute 事务属性对象，并缓存；
没有解析出对应的 TransactionAttribute 对象也就不会被事务拦截器拦截，否则，需要为这个 Bean 创建一个代理对象。

这个 TransactionInterceptor（Advice）事务拦截器让方法的执行处于一个事务中（如果定义了 @Transactional 注解，且被 public 修饰符修饰）；
首先会创建一个事务（如果有必要），最核心就是将数据库的 commit 设置为 false，不自动提交，在方法执行完后进行提交（或者回滚）；

## 拓展
**Spirng 事务（Transactions）的底层实现总体上是这样的**：以 @EnableXxx 模块驱动注解驱动整个模块，同时会注入一个 PointcutAdvisor 切面，关联一个 Pointcut 和一个 Advice 通知；通过 Pointcut 筛选出符合条件的 Bean；然后在 Advice 中进行拦截处理，实现相应的功能。

Spring 缓存（Caching）的底层实现和 Spirng 事务（Transactions）整体上差不多，当你对 Spirng 事务（Transactions）的底层了解后，你会发现 **Spring 缓存（Caching）的实现基本是照搬过来的。**

**Spring 异步处理（Async）的底层实现和上面两者类似（原理差不多）**，不过它没有直接注入一个 PointcutAdvisor 切面，**而是注入了一个 `AbstractAdvisingBeanPostProcessor` 对象**，继承 ProxyProcessorSupport（AOP 代理配置类），且实现 BeanPostProcessor 接口；在这个对象里面会关联一个 AsyncAnnotationAdvisor 切面对象，然后通过实现 BeanPostProcessor 接口在 Spring Bean 的生命周期中的初始化后进行扩展，对于符合条件的 Bean 会通过 ProxyFactory 创建一个代理对象；AsyncAnnotationAdvisor 关联的 Advice 会对方法进行拦截处理，也就是将方法的执行放入一个 Executor 线程池中执行，会返回一个 Future 可用于获取执行结果。

## 几种会导致Spring事务失效的场景
在Spring中，事务失效可能由以下场景引起：
1. **非public方法**  
   Spring事务默认只对public方法生效，非public方法上的`@Transactional`注解会被忽略。
2. **自调用**  
   类内部方法调用另一个带有`@Transactional`注解的方法时，事务不会生效，因为代理无法拦截自调用。
3. **异常处理不当**  
   默认情况下，Spring只对未检查异常（RuntimeException）回滚。如果捕获了异常且未重新抛出，或抛出了检查异常，事务不会回滚。
4. **事务传播配置错误**  
   如果事务传播行为配置为`Propagation.NOT_SUPPORTED`或`Propagation.NEVER`，事务将不会生效。
5. **多线程调用**  
   新线程中的操作不在原事务管理范围内，导致事务失效。
6. **数据库引擎不支持事务**  
   如MySQL的MyISAM引擎不支持事务，即使使用`@Transactional`注解也不会生效。
7. **未启用事务管理**  
   如果未在配置中启用事务管理（如未配置`@EnableTransactionManagement`），事务注解不会生效。
8. **错误的代理模式**  
   如果使用基于类的代理（CGLIB）且类为final，或使用基于接口的代理（JDK动态代理）但方法未在接口中声明，事务可能失效。
9. **手动回滚未处理**  
   如果在代码中手动调用了`setRollbackOnly()`但未正确处理，事务可能不会按预期回滚。
10. **事务管理器配置错误**  
    如未正确配置事务管理器，或使用了错误的事务管理器，事务将无法生效。
