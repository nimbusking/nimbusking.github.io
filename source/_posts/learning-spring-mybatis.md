---
title: Spring Mybatis细磕
abbrlink: 35e49e17
date: 2024-11-23 18:12:55
updated: 2024-12-26 13:26:49
tags:
  - Spring
  - MyBatis
  - 动态代理
categories: MyBatis
---

## 前言
项目fork注释地址：https://github.com/nimbusking/mybatis-3
【备注】：
- MyBatis的源码注释非常emmm简单，如仔细研究的话，需要花点功夫。
- 项目注释里面，如看过的部分源码，会格式化代码（原始代码缩进2空格，看起来不是很舒服，原谅强迫症~）

<!-- more -->

## 一些概念
### MyBatis的实现逻辑
1. 在 MyBatis 的初始化过程中，会生成一个 `Configuration` 全局配置对象，里面包含了所有初始化过程中生成对象
2. 根据 `Configuration` 创建一个 `SqlSessionFactory` 对象，用于创建 `SqlSession` “会话”
3. 通过 `SqlSession` 可以获取到 `Mapper` 接口对应的**动态代理对象**，去执行数据库的相关操作
4. 动态代理对象执行数据库的操作，由 `SqlSession` 执行相应的方法，在他的内部调用 `Executor` 执行器去执行数据库的相关操作
5. 在 `Executor` 执行器中，会进行相应的处理，将数据库执行结果返回

### MyBatis的缓存实现逻辑
MyBatis 提供了一级缓存和二级缓存
- 一级缓存
  在 Executor 执行器（SimpleExecutor）中有一个 Cache 对象中，默认**就是一个 HashMap 存储缓存数据**，执行数据库查询操作前，**如果在一级缓存中有对应的缓存数据，则直接返回，不会去访问数据库**
  默认的缓存区域为SESSION，表示开启一级缓存，可以设置为STATEMENT，执行完查询后会清空一级缓存，所有的数据库更新操作也会清空一级缓存
  - **缺陷：**在多个 SqlSession 会话时，**可能导致数据的不一致性**，某一个 SqlSession 更新了数据而其他 SqlSession 无法获取到更新后的数据，出现数据不一致性，这种情况是不允许出现了，**所以我们通常选择“关闭”一级缓存**
  ![MyBatis一级缓存示意图](35e49e17/MyBatis一级缓存示意图.jpg)
- 二级缓存
  在 Executor 执行器（CachingExecutor）中有一个 **TransactionalCacheManager** 对象中，可以在**一定程度上解决**的一级缓存中多个 SqlSession 会话可能会导致数据不一致的问题，就是**将一个 XML 映射文件中定义的缓存对象放在全局对象中，对于同一个 Mapper 接口都是使用这个 Cache 对象，不管哪个 SqlSession 都是使用该 Cache 对象**
  执行数据库查询操作前，如果在二级缓存中有对应的缓存数据，则直接返回，没有的话则去一级缓存中获取，如果有对应的缓存数据，则直接返回，不会去访问数据库
  默认全局开启，需要在每个 XML 映射文件中定义
  - 缺陷：对于不同的 XML 映射文件，如果某个的 XML 映射文件修改了相应的数据，其他的 XML 映射文件获取到的缓存数据就可能不是最新的，也出现了脏读的问题，当然你可以所有的 XML 映射文件都通过`<cache-ref />`来使用同一个 Cache 对象，不过这样太局限了，且缓存的数据仅仅是保存在了本地内存中，**对于当前高并发的环境下是无法满足要求的，所以我们通常不使用MyBatis的缓存**
  ![MyBatis二级缓存示意图](35e49e17/MyBatis二级缓存示意图.jpg)

### #{} 和 ${} 的区别
两者在 MyBatis 中都可以作为 SQL 的参数占位符，在处理方式上不同
- **#{}** ：在解析 SQL 的时候会将其 **替换成 ? 占位符**，然后通过 JDBC 的 `PreparedStatement` 对象添加参数值，这里会进行预编译处理，可以有效的防止 SQL 注入，提高系统的安全性
- **${}** ：在 MyBatis 中带有该占位符的 SQL 片段会被解析成动态 SQL 语句，根据入参直接替换掉这个值，然后执行数据库相关操作，**存在 SQL注入 的安全性问题**

### MyBatis中自定义标签的执行原理
MyBatis 提供了以下几种动态 SQL 的标签：`<if />、<choose />、<when />、<otherwise />、<trim />、<where />、<set />、<foreach />、<bind />`
大体的流程是：
*在 MyBatis 的初始化过程中的解析 SQL 过程中，会将定义的一个 SQL 解析成一个个的 SqlNode 对象，当需要执行数据库查询前，需要根据入参对这些 SqlNode 对象进行解析，使用OGNL表达式计算出结果，然后根据结果拼接对应的 SQL 片段，以此完成动态 SQL 的功能。*

### 简述Mapper接口的工作原理
在 MyBatis 的初始化过程中，每个一个 XML 映射文件中的`<select />、<insert />、<update />、<delete />`标签，**会被解析成一个 MappedStatement 对像**，对应的 id 就是 XML 映射文件配置的 namespace+'.'+statementId，这个 id 跟 Mapper 接口中的方法进行关联
延伸的一个问题：**同一个 Mapper 接口中为什么不能定义重载方法？**
  因为 Mapper 接口中的方法是通过 **接口名称+'.'+方法名** 去找到对应的 MappedStatement 对象，如果方法名相同，则对应的 MappedStatement 对象就是同一个，就存在问题了，所以同一个 Mapper 接口不能定义重载的方法

**每个 Mapper 接口都会创建一个动态代理对象（JDK 动态代理）**，代理类会拦截接口的方法，找到对应的 MappedStatement 对象，然后执行数据库相关操作
执行流程如下：
![Mapper执行流程](35e49e17/Mapper执行流程.jpg)

### 在Spring中Mapper接口是如何被注入的？
通过 SqlSession 的 `getMapper(Class<T> type)` 方法，可以获取到 Mapper 接口的动态代理对象，那么在 Spring 中是如何将 Mapper 接口注入到其他 Spring Bean 中的呢？

在 MyBatis 的 MyBatis-Spring 集成 Spring 子项目中，通过实现 Spring 的 BeanDefinitionRegistryPostProcessor 接口，实现它的 postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 方法，也就是**在 Spring 完成 BeanDefinition 的初始化工作后，会将 Mapper 接口也解析成 BeanDefinition 对象注册到 registry 注册表中**，并且会修改其 beanClass 为 MapperFactoryBean 类型，还添加了一个入参为 Mapper 接口的 Class 对象的名称

这样 Mapper 接口会对应一个 MapperFactoryBean 对象，由于这个对象实现了 FactoryBean 接口，实现它的 getObject() 方法，该方法会通过 SqlSession 的 `getMapper(Class<T> type) `方法，返回该 Mapper 接口的动态代理对象，所以**在 Spring Bean 中注入的 Mapper 接口时，调用其 getObeject() 方法，拿到的是 Mapper 接口的动态代理对象**

### Mybatis有哪些Executor执行器
- **SimpleExecutor（默认）**：每执行一次数据库的操作，就创建一个 Statement 对象，用完立刻关闭 Statement 对象。
- **ReuseExecutor：**执行数据库的操作，以 SQL 作为 key 查找缓存的 Statement 对象，存在就使用，不存在就创建；用完后，不关闭 Statement 对象，而是放置于缓存` Map<String, Statement>` 内，供下一次使用，就是重复使用 Statement 对象
- **BatchExecutor：**执行数据库更新操作（没有查询操作，因为 JDBC 批处理不支持查询操作），**将所有 SQL 都添加到批处理中（通过 addBatch 方法），等待统一执行（使用 executeBatch 方法）**。它缓存了多个 Statement 对象，每个 Statement 对象都是调用 addBatch 方法完毕后，等待一次执行 executeBatch 批处理。实际上，整个过程与 JDBC 批处理是相同。
- **CachingExecutor：**在上述的三个执行器之上，增加二级缓存的功能

### MyBatis的延迟加载原理
MyBatis 获取到查询结果需要进行结果集映射，也就是将 JDBC 返回的结果集转换成我们需要的 Java 对象
在映射的过程中，如果存在嵌套子查询且需要延迟加载，则会为该返回结果的实例对象创建一个动态代理对象（Javassist），**也就是说我们拿到的返回结果实际上是一个动态代理对象**
这个动态代理对象中包含一个 ResultLoaderMap 对象，**保存了需要延迟加载的属性和嵌套子查询的映射关系**
当你调用了需要延迟加载的属性的 getter 方法时，**会执行嵌套子查询，将结果设置到该对象中**，然后将该延迟加载的属性从 ResultLoaderMap 中移除
- 如果你调用了该对象equals、clone、hashCode、toString某个方法（默认），会触发所有的延迟加载，然后全部移除
- 如果你调用了延迟加载的属性的 setter 方法，也会将将延迟加载的属性从 ResultLoaderMap 中移除

### MyBatis的插件运行原理
MyBatis 的插件机制仅针对 `ParameterHandler、ResultSetHandler、StatementHandler、Executor` 这 4 种接口中的方法进行增强处理
Mybatis 使用 JDK 的动态代理，为需要拦截的接口生成代理对象以实现接口方法拦截功能，每当执行这 4 种接口对象的方法时，就会进入拦截方法，具体就是 InvocationHandler 的 #invoke(...)方法。当然，只会拦截那些你指定需要拦截的方法
#### 为什么仅支持这 4 种接口呢？
因为 MyBatis 仅在创建上述接口对象后，将所有的插件应用在该对象上面，也就是为该对象创建一个动态代理对象，然后返回
#### 如何编写一个 MyBatis 插件？
让插件类实现 `org.apache.ibatis.plugin.Interceptor` 接口，还需要通过注解标注该插件的拦截点，然后在 MyBatis 的配置文件中添加的插件，例如：
```java
@Intercepts({
  @Signature(
    type = Executor.class,
    method = "query",
    args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
  )
})
public class ExamplePlugin implements Interceptor {

  // Executor的查询方法：
  // public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    Object[] args = invocation.getArgs();
    RowBounds rowBounds = (RowBounds) args[2];
    if (rowBounds == RowBounds.DEFAULT) { // 无需分页
      return invocation.proceed();
    }
    /*
     * 将query方法的 RowBounds 入参设置为空对象
     * 也就是关闭 MyBatis 内部实现的分页（逻辑分页，在拿到查询结果后再进行分页的，而不是物理分页）
     */
    args[2] = RowBounds.DEFAULT;

    MappedStatement mappedStatement = (MappedStatement) args[0];
    BoundSql boundSql = mappedStatement.getBoundSql(args[1]);

    // 获取 SQL 语句，拼接 limit 语句
    String sql = boundSql.getSql();
    String limit = String.format("LIMIT %d,%d", rowBounds.getOffset(), rowBounds.getLimit());
    sql = sql + " " + limit;

    // 创建一个 StaticSqlSource 对象
    SqlSource sqlSource = new StaticSqlSource(mappedStatement.getConfiguration(), sql, boundSql.getParameterMappings());

    // 通过反射获取并设置 MappedStatement 的 sqlSource 字段
    Field field = MappedStatement.class.getDeclaredField("sqlSource");
    field.setAccessible(true);
    field.set(mappedStatement, sqlSource);

    // 执行被拦截方法
    return invocation.proceed();
  }

  @Override
  public Object plugin(Object target) {
    // default impl
    return Plugin.wrap(target, this);
  }

  @Override
  public void setProperties(Properties properties) {
    // default nop
  }
}

```

### Mybatis是如何进行分页的
Mybatis 使用 **RowBounds** 对象进行分页，**它是针对 ResultSet 结果集执行的内存分页，而非数据库分页**
所以，实际场景下，不适合直接使用 MyBatis 原有的 RowBounds 对象进行分页。而是使用如下两种方案：
- 在 SQL 内直接书写带有数据库分页的参数来完成数据库分页功能
- 也可以使用分页插件来完成数据库分页。

这两者都是基于数据库分页，差别在于**前者**是工程师**手动编写分页条件**，**后者**是插件**自动添加分页条件**。
其中，**分页插件的基本原理是使用 Mybatis 提供的插件接口，实现自定义分页插件。**在插件的拦截方法内，拦截待执行的 SQL ，然后重写 SQL ，根据 dialect 方言，添加对应的物理分页语句和物理分页参数。

### Mybatis如何处理include标签的
例如：如果 A 标签通过 include 引用了 B 标签的内容，请问，B 标签能否定义在 A 标签的后面，还是说必须定义在A标签的前面
虽然 Mybatis 解析 XML 映射文件是按照顺序解析的。但是，被引用的 B 标签依然可以定义在同一个 XML 映射文件的任何地方，Mybatis 都可以正确识别。**也就是说，无需按照顺序，进行定义。**
原理是，Mybatis 解析 A 标签，发现 A 标签引用了 B 标签，但是 B 标签尚未解析到，尚不存在，此时，**Mybatis 会将 A 标签标记为未解析状态**。然后，继续解析余下的标签，包含 B 标签，待所有标签解析完毕，Mybatis 会重新解析那些被标记为未解析的标签，此时再解析A标签时，B 标签已经存在，A 标签也就可以正常解析完成了（PS：跟IOC容器处理循环依赖很相似）。

### MyBatis与Hibernate有哪些不同
按照用户的需求在有限的资源环境下只要能做出维护性、扩展性良好的软件架构都是好架构，所以框架只有适合才是最好。简单总结如下：
- Hibernate 属于全自动 ORM 映射工具，使用 Hibernate 查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取。
- Mybatis 属于半自动 ORM 映射工具，在查询关联对象或关联集合对象时，需要手动编写 SQL 来完成。

### Mybatis比IBatis比较大的几个改进是什么
1. 有接口绑定，包括注解绑定 SQL 和 XML 绑定 SQL 。
2. 动态 SQL 由原来的节点配置变成 OGNL 表达式。
3. 在一对一或一对多的时候，引进了 `<association />` 标签，在一对多的时候，引入了 `<collection />`标签，不过都是在 `<resultMap /> `里面配置


---

## 其它补充
### MyBatis 工作原理总结
- 1. **核心组件**：
   - **SqlSessionFactory**：全局单例，通过`mybatis-config.xml`和Mapper文件构建，生成`SqlSession`。
   - **SqlSession**：代表一次数据库会话，线程不安全，需确保每次请求独立创建。
   - **Executor**：执行器，负责SQL调度（如`SimpleExecutor`、`ReuseExecutor`、`BatchExecutor`）。
   - **MapperProxy**：动态代理对象，将Mapper接口方法映射到XML或注解的SQL语句。
   - **StatementHandler**：处理JDBC Statement（预编译、参数设置）。
   - **ParameterHandler**：将Java对象转换为SQL参数。
   - **ResultSetHandler**：将ResultSet结果集转换为Java对象。
- 2. **核心组件与初始化过程**
   - **配置文件加载**：MyBatis 通过 `mybatis-config.xml` 加载全局配置（如数据源、类型处理器、插件等），并通过 Mapper XML 文件或注解定义 SQL 映射。
   - **SqlSessionFactory 构建**：使用 `SqlSessionFactoryBuilder` 解析配置文件，生成 `SqlSessionFactory`，用于创建 `SqlSession`。
   - **SqlSession 生命周期**：每个线程通过 `SqlSessionFactory` 创建 `SqlSession` 实例，用于执行 SQL 操作。`SqlSession` 是非线程安全的，通常在一次请求或事务中使用后关闭。
- 3. **动态代理与 SQL 执行流程**
   - **Mapper 接口代理**：MyBatis 通过 JDK 动态代理为 Mapper 接口生成代理对象，将接口方法调用转换为对应的 SQL 语句执行。
   - **执行器（Executor）**：`SqlSession` 内部通过 `Executor` 执行 SQL，处理缓存（一级缓存）、参数绑定、结果映射等逻辑。
   - **结果映射**：通过 `ResultMap` 将 JDBC `ResultSet` 转换为 Java 对象，支持自动映射或自定义映射规则。
- 4. **核心流程步骤**
   1. **加载配置**：解析`mybatis-config.xml`和Mapper文件，构建`Configuration`对象。
   2. **创建SqlSessionFactory**：通过`SqlSessionFactoryBuilder`生成工厂实例。
   3. **创建SqlSession**：通过工厂实例开启会话，内部生成`Executor`。
   4. **获取Mapper代理**：通过`SqlSession.getMapper()`生成Mapper接口的代理对象。
   5. **执行SQL**：代理对象调用方法，根据方法名关联SQL，通过`Executor`执行。
   6. **结果映射**：通过`ResultSetHandler`将结果转换为POJO或集合。

### MyBatis 与 Spring 集成原理
- 1. **Spring 管理 MyBatis 核心组件**
   - **SqlSessionFactoryBean**：Spring 通过 `SqlSessionFactoryBean` 替代原生 `SqlSessionFactoryBuilder`，将 `SqlSessionFactory` 作为 Bean 注册到 IoC 容器，整合数据源和 MyBatis 配置。
   - **数据源注入**：由 Spring 管理数据源（如 HikariCP、Druid），通过依赖注入到 `SqlSessionFactoryBean`。
- 2. **Mapper 接口的自动扫描与代理**
   - **MapperScannerConfigurer 或 @MapperScan**：自动扫描指定包下的 Mapper 接口，生成代理 Bean 并注册到 Spring 容器，开发者可直接通过 `@Autowired` 注入使用。
   - **代理逻辑**：调用 Mapper 方法时，代理对象通过 `SqlSessionTemplate` 执行 SQL，确保线程安全。
- 3. **SqlSessionTemplate 的作用**
   - **替代原生 SqlSession**：`SqlSessionTemplate` 是线程安全的，内部通过 `SqlSessionHolder` 结合 Spring 事务管理，确保同一事务内共享同一个 `SqlSession`。
   - **事务同步**：与 Spring 事务管理器协同，在事务提交或回滚时自动关闭 `SqlSession`。
- 4. **事务管理整合**
   - **声明式事务**：通过 `@Transactional` 注解，由 Spring 的 `DataSourceTransactionManager` 管理事务。
   - **Connection 绑定**：事务开启时，Spring 将数据库连接绑定到当前线程，MyBatis 操作复用该连接，保证事务原子性。



### 常见问题扩展
- **Q：如何防止MyBatis的N+1查询问题？**  
  - 【注】 N+1 查询问题：当查询主对象（如订单）时，触发 1 次主查询（获取 N 条数据），随后对每条主数据执行关联子查询（如查询每个订单的商品列表），导致总执行 1 + N 次 SQL，性能急剧下降。
  **A**：使用联合查询（JOIN）或批量加载（`<collection fetchType="eager">`，默认就是急加载）【一般使用联合查询就能解决了】
- **Q：MyBatis如何支持多数据源？**  
  **A**：配置多个`SqlSessionFactory`，结合Spring的`@Primary`和`@Qualifier`注解区分。
- **Q：XML和注解配置的优劣？**  
  **A**：XML适合复杂SQL，注解适合简单场景，但可读性差。


---

## 使用手册
### 一个示例的mybatis-config.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- plugins在配置文件中的位置必须符合要求，否则会报错，顺序如下: properties?, settings?, typeAliases?,
        typeHandlers?, objectFactory?,objectWrapperFactory?, plugins?, environments?,
        databaseIdProvider?, mappers? -->
    <!-- 配置mybatis的缓存，延迟加载等等一系列属性 -->
    <settings>
        <!-- 全局映射器启用缓存 -->
        <setting name="cacheEnabled" value="false" />
        <!-- 对于未知的SQL查询，允许返回不同的结果集以达到通用的效果 -->
        <setting name="multipleResultSetsEnabled" value="true" />
        <!-- 是否开启自动驼峰命名规则映射，数据库的A_COLUMN映射为Java中的aColumn -->
        <setting name="mapUnderscoreToCamelCase" value="true" />
        <!-- MyBatis利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询 -->
        <setting name="localCacheScope" value="STATEMENT" />
    </settings>
    <!-- 指定路径下的实体类支持别名(默认实体类的名称,首字母小写), @Alias注解可设置别名 -->
    <typeAliases>
        <package name="cc.nimbusk.mybatis.simple.model" />
    </typeAliases>
    <!-- 配置当前环境信息 -->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="" value=""/>
            </transactionManager>
            <!-- 配置数据源 -->
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/nimbusk?useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false"/>
                <property name="username" value="user"/>
                <property name="password" value="password"/>
            </dataSource>
        </environment>
    </environments>
  <!-- 指定Mapper接口的路径 -->
    <mappers>
        <package name="cc.nimbusk.mybatis.simple.mapper"/>
    </mappers>
</configuration>

```

### 配置动态数据源
基于 Spring 提供的 `AbstractRoutingDataSource`，自己实现数据源的切换，下面就如何配置动态数据源提供一个简单的实现
例如:
```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {

  @Nullable
  private Object defaultTargetDataSource;

  @Nullable
  private Map<Object, DataSource> resolvedDataSources;

  @Nullable
  private DataSource resolvedDefaultDataSource;

  @Override
  public Connection getConnection() throws SQLException {
    return determineTargetDataSource().getConnection();
  }

  @Override
  public Connection getConnection(String username, String password) throws SQLException {
    return determineTargetDataSource().getConnection(username, password);
  }

  protected DataSource determineTargetDataSource() {
    Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        // 确定当前要使用的数据源
    Object lookupKey = determineCurrentLookupKey();
    DataSource dataSource = this.resolvedDataSources.get(lookupKey);
    if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
      dataSource = this.resolvedDefaultDataSource;
    }
    if (dataSource == null) {
      throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
    }
    return dataSource;
  }

  /**
   * Determine the current lookup key. This will typically be implemented to check a thread-bound transaction context.
   * <p>
   * Allows for arbitrary keys. The returned key needs to match the stored lookup key type, as resolved by the
   * {@link #resolveSpecifiedLookupKey} method.
   */
  @Nullable
  protected abstract Object determineCurrentLookupKey();
  
  // 省略相关代码...
}

```

## 整体架构
项目源码结构一览
![mybatis源码结构](35e49e17/mybatis源码结构.jpg)
MyBatis 的整体架构分为三层：基础支持层、核心处理层和接口层
![MyBatis架构图](35e49e17/MyBatis架构图.jpg)
每个模块分层介绍如下：
### 接口层
接口层，**核心为 SqlSession 接口**，该接口定义了 MyBatis 暴露给应用程序调用的 API，也就是上层应用与 MyBatis 交互的桥梁。*接口层在接收到调用请求时，会调用核心处理层的相应模块来完成具体的数据库操作*
### 核心处理层
核心处理层，实现了 MyBatis 的核心处理流程，包括 MyBatis 的初始化以及完成一次数据库操作的涉及的全部流程
### 基础支持层
- **解析器模块**：解析文件，处理占位符
- **反射模块**：对 Java 原生的反射进行良好的封装，进行了一系列的优化，提供更加简洁易用的 API 方便使用
- **异常模块**：定义 MyBatis 自己的 Exception
- **数据源模块**：提供相应的 DataSource 数据源实现，支持与第三方数据源的继承
- **事务模块**：对数据库中的事务进行了抽象，提供事务接口的简单实现
- **缓存模块**：提供一级缓存和二级缓存的支持
- **类型模块**：提供别名机制，JDBC Type 和 Java Type 之间的转换
- **IO模块**：提供资源加载功能
- **日志模块**：提供日志输出，支持集成第三方日志框架
- **注解模块**：提供相关注解
- **Binding模块**：提供 Mapper 接口与 XML 映射文件进行关联的支持

## 基础支持层
### 解析器模块
**主要包路径**：`org.apache.ibatis.parsing`
**主要功能**：初始化时解析**mybatis-config.xml**配置文件、为处理动态SQL语句中占位符提供支持
这里面有三个重要的类，来分别看一下：
- **org.apache.ibatis.parsing.XPathParser**：基于Java XPath 解析器，用于解析MyBatis的mybatis-config.xml和**Mapper.xml等XML配置文件
- **org.apache.ibatis.parsing.GenericTokenParser**：通用的Token解析器
- **org.apache.ibatis.parsing.PropertyParser**：动态属性解析器

#### XPathParser类
代码不复杂，就不贴了，咱们就来看看总结的东西
这个类里面定义了几个属性：

| 类型        | 属性名           | 说明  |
| ------------- |:-------------:|:-------------:|
|  Document      |document | XML文件被解析后生成对应的`org.w3c.dom.Document`对象 |
|  boolean      |validation | 是否校验XML文件，一般情况下为true |
|  EntityResolver      |entityResolver | `org.xml.sax.EntityResolver`对象，XML实体解析器，一般通过自定义的`org.apache.ibatis.builder.xml.XMLMapperEntityResolver`从本地获取DTD文件解析 |
|  Properties      |variables | 变量Properties对象，**用来替换需要动态配置的属性值**，例如我们在MyBatis的配置文件中使用变量将用户名密码放在另外一个配置文件中，那么这个配置会被解析到Properties对象用，用于替换XML文件中的动态值 |
|  XPath      |xpath | javax.xml.xpath.XPath 对象，用于**查询XML**中的节点和元素 |

构造函数有很多，**基本都相似**，内部都是调用commonConstructor方法设置相关属性和createDocument方法为该XML文件创建一个Document对象

提供了一系列的`eval*`方法，用于获取Document对象中的元素或者节点：
- **eval相关元素的方法**：**根据表达式获取我们常用类型的元素的值**，其中会基于variables调用PropertyParser的parse方法替换掉其中的动态值（如果存在），**这就是MyBatis如何替换掉XML中的动态值实现的方式**
- **eval相关节点的方法**：根据表达式获取到org.w3c.dom.Node节点对象，将其封装成自己定义的XNode对象，方便主要为了动态值的替换

#### PropertyParser类
`org.apache.ibatis.parsing.PropertyParser`：动态属性解析器
这里面有一个parse方法
```java
  public static String parse(String string, Properties variables) {
    // <2.1> 创建 VariableTokenHandler 对象
    VariableTokenHandler handler = new VariableTokenHandler(variables);
    // <2.2> 创建 GenericTokenParser 对象
    GenericTokenParser parser = new GenericTokenParser("${", "}", handler);
    // <2.3> 执行解析
    return parser.parse(string);
  }
```
其中：parse方法：创建VariableTokenHandler对象和GenericTokenParser对象，然后调用GenericTokenParser的**parse方法替换其中的动态值**

#### GenericTokenParser类
本质根据开始字符串和结束字符串解析出里面的表达式（例如${name}->name），然后通过TokenHandler进行解析处理

### 反射模块
将一个Entity实体类或者Map集合转换成MetaObject对象，该对象通过反射机制封装了各种简便的方法，使更加方便安全地操作该对象，创建过程：
1. 通过Configuration全局配置对象的newMetaObject(Object object)方法创建，会传入DefaultObjectFactory、DefaultObjectWrapperFactory和DefaultReflectorFactory几个默认实现类
2. 内部调用MetaObject的forObject静态方法，通过它的构造方法创建一个实例对象
3. 在MetaObject的构造函数中，会根据Object对象的类型来创建ObjectWrapper对象
4. 如果是创建BeanWrapper，则在其构造函数中，会再调用MetaClass的forClass方法创建MetaClass对象，也就是通过其构造函数创建一个实例对象
5. 如果是MapWrapper，则直接复制给内部的`Map<String, Object> map`属性即可，其他集合对象类似
6. 在MetaClass的构造函数中，会通过调用DefaultReflectorFactory的findForClass方法创建Reflector对象
7. 在Reflector的构造函数中，通过反射机制解析该Class类，属性的set和get方法会被封装成MethodInvoker对象

### 数据源模块
MyBatis支持三种数据源配置，分别为`UNPOOLED、POOLED和JNDI`。内部提供了两种数据源实现，分别是`UnpooledDataSource`和`PooledDataSource`。
在三种数据源配置中，**`UNPOOLED` 和 `POOLED` 是常用的两种配置。**
至于 JNDI，MyBatis 提供这种数据源的目的是为了让其能够运行在 EJB 或应用服务器等容器中，这一点官方文档中有所说明。由于 JNDI 数据源在日常开发中使用甚少，因此，本篇文章不打算分析 JNDI 数据源相关实现。大家若有兴趣，可自行分析。

## MyBatis初始化
在MyBatis初始化过程中，**大致会有以下几个步骤**：
1. 创建Configuration全局配置对象，会往TypeAliasRegistry别名注册中心添加Mybatis需要用到的相关类，并设置默认的语言驱动类为XMLLanguageDriver
2. 加载mybatis-config.xml配置文件、Mapper接口中的注解信息和XML映射文件，解析后的配置信息会形成相应的对象并保存到Configuration全局配置对象中
3. 构建DefaultSqlSessionFactory对象，通过它可以创建DefaultSqlSession对象，MyBatis中SqlSession的默认实现类

### 加载mybatis-config.xml
初始化入口在**org.apache.ibatis.session.SqlSessionFactoryBuilder**构造器中，因为需要通过mybatis-config.xml配置文件构建一个SqlSessionFactory工厂，用于创建SqlSession会话
主要涉及到以下几个类：
- **org.apache.ibatis.session.SqlSessionFactoryBuilder**：用于构建SqlSessionFactory工厂
- **org.apache.ibatis.builder.xml.XMLConfigBuilder**：根据配置文件进行解析，开始Mapper接口与XML映射文件的初始化，生成Configuration全局配置对象
- **org.apache.ibatis.builder.xml.XMLMapperBuilder**：继承BaseBuilder抽象类，用于解析XML映射文件内的标签
- **org.apache.ibatis.session.Configuration**：MyBatis的全局配置对象，保存所有的配置与初始化过程所产生的对象。上面解析mybatis-config.xml后的配置属性会存到这个类相关属性中去。

解析的过程就不用过多贴出来了，总结就是：
**MyBatis会在SqlSessionFactoryBuilder构造器中根据mybatis-config.xml配置文件初始化Configuration全局配置对象，然后创建对应的DefaultSqlSessionFactory工厂**

### 加载Mapper接口与映射文件
我们先看看一个示例的mapper文件
![mybatis-mapper-xml-sample](35e49e17/mybatis-mapper-xml-sample.jpg)
这个过程里面主要涉及下面一些类：
- **org.apache.ibatis.builder.xml.XMLConfigBuilder**：根据配置文件进行解析，开始Mapper接口与XML映射文件的初始化，生成Configuration全局配置对象
- **org.apache.ibatis.binding.MapperRegistry**：Mapper接口注册中心，将Mapper接口与其动态代理对象工厂进行保存，这里我们解析到的Mapper接口需要往其进行注册
- **org.apache.ibatis.builder.annotation.MapperAnnotationBuilder**：解析Mapper接口，主要是解析接口上面注解，其中加载XML映射文件内部会调用XMLMapperBuilder类进行解析
- **org.apache.ibatis.builder.xml.XMLMapperBuilder**：解析XML映射文件
- **org.apache.ibatis.builder.xml.XMLStatementBuilder**：解析XML映射文件中的Statement配置（`<select /> <update /> <delete /> <insert />`标签）
- **org.apache.ibatis.builder.MapperBuilderAssistant**：Mapper构造器小助手，用于创建ResultMapping、ResultMap和MappedStatement对象
- **org.apache.ibatis.mapping.ResultMapping**：保存`<resultMap />`标签的子标签相关信息，也就是 Java Type 与 Jdbc Type 的映射信息
- **org.apache.ibatis.mapping.ResultMap**：保存了`<resultMap />`标签的配置信息以及子标签的所有信息
- **org.apache.ibatis.mapping.MappedStatement**：保存了解析`<select /> <update /> <delete /> <insert />`标签内的SQL语句所生成的所有信息

#### XMLConfigBuilder-解析入口
在org.apache.ibatis.builder.xml.XMLConfigBuilder中会解析mybatis-config.xml配置文件中的`<mapper />`标签，调用其`parse()->parseConfiguration(XNode root)->mappersElement(XNode parent)`方法，那么我们来看看这个方法，代码如下：
```java
private void mappersElement(XNode context) throws Exception {
        if (context == null) {
            return;
        }
        // <0> 遍历子节点
        for (XNode child : context.getChildren()) {
            // <1> 如果是 package 标签，则扫描该包
            if ("package".equals(child.getName())) {
                // 获得包名
                String mapperPackage = child.getStringAttribute("name");
                // 添加到 configuration 中
                configuration.addMappers(mapperPackage);
            } else { // 如果是 mapper 标签
                // 获得 resource、url、class 属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // <2> 使用相对于类路径的资源引用
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    // 获得 resource 的 InputStream 对象
                    try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
                        // 创建 XMLMapperBuilder 对象
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
                                configuration.getSqlFragments());
                        // 执行解析
                        mapperParser.parse();
                    }
                    // <3> 使用完全限定资源定位符（URL）
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    // 获得 url 的 InputStream 对象
                    try (InputStream inputStream = Resources.getUrlAsStream(url)) {
                        // 创建 XMLMapperBuilder 对象
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url,
                                configuration.getSqlFragments());
                        // 执行解析
                        mapperParser.parse();
                    }
                    // <4> 使用映射器接口实现类的完全限定类名
                } else if (resource == null && url == null && mapperClass != null) {
                    // 获得 Mapper 接口
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    // 添加到 configuration 中
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException(
                            "A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
```
上述的`configuration.addMappers(mapperPackage)`方法，底层是通过调用它的**MapperRegistry的addMappers(String packageName)方法**进行注册的。
看一下MapperRegistry中addMapper方法：
```java
// org.apache.ibatis.binding.MapperRegistry#addMapper
public <T> void addMapper(Class<T> type) {
        // <1> 判断，必须是接口。
        if (type.isInterface()) {
            // <2> 已经添加过，则抛出 BindingException 异常
            if (hasMapper(type)) {
                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                // <3> 将Mapper接口对应的代理工厂添加到 knownMappers 中
                knownMappers.put(type, new MapperProxyFactory<>(type));
                // It's important that the type is added before the parser is run
                // otherwise the binding may automatically be attempted by the
                // mapper parser. If the type is already known, it won't try.
                // <4> 解析 Mapper 的注解配置
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
                // 解析 Mapper 接口上面的注解和 Mapper 接口对应的 XML 文件
                parser.parse();
                // <5> 标记加载完成
                loadCompleted = true;
            } finally {
                // <6> 若加载未完成，从 knownMappers 中移除
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
```
注意上面第4步，通过new了一个`MapperAnnotationBuilder`对象，完了调用parse方法解析。而**这个对象就是解析该Mapper接口与对应XML映射文件的重要类**。
##### MapperAnnotationBuilder类
MapperAnnotationBuilder：解析Mapper接口，主要是解析接口上面注解，加载XML文件会调用XMLMapperBuilder类进行解析
```java
// org.apache.ibatis.builder.annotation.MapperAnnotationBuilder#parse
public void parse() {
        String resource = type.toString();
        if (!configuration.isResourceLoaded(resource)) {
            // 加载该接口对应的 XML 文件
            loadXmlResource();
            configuration.addLoadedResource(resource);
            assistant.setCurrentNamespace(type.getName());
            // 解析 Mapper 接口的 @CacheNamespace 注解，创建缓存
            parseCache();
            // 解析 Mapper 接口的 @CacheNamespaceRef 注解，引用其他命名空间
            parseCacheRef();
            for (Method method : type.getMethods()) {
                // 先判断前置条件：不是桥接方法（泛型擦除时虚拟机生成的）且不是default方法
                if (!canHaveStatement(method)) {
                    continue;
                }
                // 处理ResultMap
                if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
                        && method.getAnnotation(ResultMap.class) == null) {
                    parseResultMap(method);
                }
                try {
                    // 解析方法上面的注解
                    parseStatement(method);
                } catch (IncompleteElementException e) {
                    configuration.addIncompleteMethod(new MethodResolver(this, method));
                }
            }
        }
        configuration.parsePendingMethods(false);
    }

// org.apache.ibatis.builder.annotation.MapperAnnotationBuilder#loadXmlResource
private void loadXmlResource() {
        // Spring may not know the real resource name so we check a flag
        // to prevent loading again a resource twice
        // this flag is set at XMLMapperBuilder#bindMapperForNamespace
        // 检查一下类型放置有问题
        if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
            String xmlResource = type.getName().replace('.', '/') + ".xml";
            // #1347
            InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
            if (inputStream == null) {
                // Search XML mapper that is not in the module but in the classpath.
                try {
                    inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
                } catch (IOException e2) {
                    // ignore, resource is not required
                }
            }
            if (inputStream != null) {
                // 创建 XMLMapperBuilder 对象
                XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource,
                        configuration.getSqlFragments(), type.getName());
                // 解析该 XML 文件
                xmlParser.parse();
            }
        }
    }
```
上述loadXmlResource()方法中是通过，**XMLMapperBuilder类**来具体进行解析的，我们直接看看对应的parse方法
```java
public void parse() {
        // <1> 判断当前 Mapper 是否已经加载过
        if (!configuration.isResourceLoaded(resource)) {
            // <2> 解析 `<mapper />` 节点
            configurationElement(parser.evalNode("/mapper"));
            // <3> 标记该 Mapper 已经加载过
            configuration.addLoadedResource(resource);
            // <4> 绑定 Mapper
            bindMapperForNamespace();
        }
        // <5> 解析待定的 <resultMap /> 节点
        configuration.parsePendingResultMaps(false);
        // <6> 解析待定的 <cache-ref /> 节点
        configuration.parsePendingCacheRefs(false);
        // <7> 解析待定的 SQL 语句的节点
        configuration.parsePendingStatements(false);
    }
```
这里面的执行步骤是：
1. 根据Configuration全局配置判断当前XML映射文件是否已经加载过，例如resource为：xxx/xxx/xxx.xml
2. 解析 `<mapper />` 节点，也就是解析整个的XML映射文件，在下面的configurationElement方法中讲解
3. 标记该XML映射文件已经加载过，往Configuration全局配置添加该字段文件，例如添加：xxx/xxx/xxx.xml
4. 绑定 Mapper 到该命名空间，避免在`MapperAnnotationBuilder#loadXmlResource`方法中重复加载该XML映射文件
5. 解析待定的 `<resultMap />、<cache-ref />`节点以及 `Statement` 对象，因为我们配置的这些对象可能还依赖的其他对象，在解析的过程中这些依赖可能还没解析出来，导致这个对象解析失败，所以先保存在Configuration全局配置对象中，待整个XML映射文件解析完后，再遍历之前解析失败的对象进行初始化，这里就不做详细的讲述了，感兴趣的小伙伴可以看一下

我们接着上面，看几个特定的方法：
- 第2步的：configurationElement(parser.evalNode("/mapper"))
  ```java
  private void configurationElement(XNode context) {
        try {
            // <1> 获得 namespace 属性
            String namespace = context.getStringAttribute("namespace");
            if (namespace == null || namespace.isEmpty()) {
                throw new BuilderException("Mapper's namespace cannot be empty");
            }
            builderAssistant.setCurrentNamespace(namespace);
            // <2> 解析 <cache-ref /> 节点
            cacheRefElement(context.evalNode("cache-ref"));
            // <3> 解析 <cache /> 节点
            cacheElement(context.evalNode("cache"));
            // 已废弃！老式风格的参数映射。内联参数是首选,这个元素可能在将来被移除，这里不会记录。
            parameterMapElement(context.evalNodes("/mapper/parameterMap"));
            // <4> 解析 <resultMap /> 节点
            resultMapElements(context.evalNodes("/mapper/resultMap"));
            // <5> 解析 <sql /> 节点们
            sqlElement(context.evalNodes("/mapper/sql"));
            // <6> 解析 <select /> <insert /> <update /> <delete /> 节点
            buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
        } catch (Exception e) {
            throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
        }
    }
  ```
  这里面呢，就是处理mapper文件的具体步骤。对应的功能也非常清楚，我们就挑常见的继续看看。
  `resultMapElements(List<XNode> list)`方法用于解析`<resultMap />`节点，最后会调用`resultMapElement()`方法逐个解析生成`ResultMap`对象
  ![XMLMapperBuilder解析mapper文件时序流程](35e49e17/XMLMapperBuilder解析mapper文件时序流程.jpg)
  看看resultMapElements方法：
  ```java
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings,
                                       Class<?> enclosingType) {
        // 获取当前线程的上下文
        ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
        // <1> 获得 type 属性
        String type = resultMapNode.getStringAttribute("type", resultMapNode.getStringAttribute("ofType",
                resultMapNode.getStringAttribute("resultType", resultMapNode.getStringAttribute("javaType"))));
        // 获得 type 对应的类
        Class<?> typeClass = resolveClass(type);
        if (typeClass == null) {
            // 从 enclosingType Class 对象获取该 property 属性的 Class 对象
            typeClass = inheritEnclosingType(resultMapNode, enclosingType);
        }
        Discriminator discriminator = null;
        // 创建 ResultMapping 集合
        List<ResultMapping> resultMappings = new ArrayList<>(additionalResultMappings);
        // 添加父 ResultMap 的 ResultMapping 集合
        List<XNode> resultChildren = resultMapNode.getChildren();
        // <2> 遍历 <resultMap /> 的子节点
        for (XNode resultChild : resultChildren) {
            if ("constructor".equals(resultChild.getName())) { // <2.1> 处理 <constructor /> 节点
                processConstructorElement(resultChild, typeClass, resultMappings);
            } else if ("discriminator".equals(resultChild.getName())) { // <2.2> 处理 <discriminator /> 节点
                discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
            } else {
                // <2.3> 处理其它节点
                List<ResultFlag> flags = new ArrayList<>();
                if ("id".equals(resultChild.getName())) {
                    // 为添加该 ResultMapping 添加一个 Id 标志
                    flags.add(ResultFlag.ID);
                }
                // 生成对应的 ResultMapping 对象
                resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
            }
        }
        // 获得 id 属性，没有的话自动生成
        String id = resultMapNode.getStringAttribute("id", resultMapNode.getValueBasedIdentifier());
        // 获得 extends 属性
        String extend = resultMapNode.getStringAttribute("extends");
        // 获得 autoMapping 属性
        Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
        // <3> 创建 ResultMapResolver 对象，执行解析
        ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator,
                resultMappings, autoMapping);
        try {
            // 处理 ResultMap 并添加到 Configuration 全局配置中
            return resultMapResolver.resolve();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteResultMap(resultMapResolver);
            throw e;
        }
    }
  ```
  
- sqlElement方法
  `sqlElement(List<XNode> list)`方法用于解析所有的`<sql />`节点，内部调用
  ```java
  public class XMLMapperBuilder extends BaseBuilder {
      private void sqlElement(List<XNode> list, String requiredDatabaseId) {
      // <1> 遍历所有 <sql /> 节点
      for (XNode context : list) {
        // <2> 获得 databaseId 属性
        String databaseId = context.getStringAttribute("databaseId");
        // <3> 获得完整的 id 属性
        String id = context.getStringAttribute("id");
        // 设置为 `${namespace}.${id}` 格式
        id = builderAssistant.applyCurrentNamespace(id, false);
        // <4> 判断 databaseId 是否匹配
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
          // <5> 添加到 sqlFragments 中
          sqlFragments.put(id, context);
        }
      }
    }
  }
  ```

#### XMLStatementBuilder类-Statement配置
XMLStatementBuilder：解析XML映射文件中的Statement配置
也就是解析`<select /> <insert /> <update /> <delete /> `节点，**解析过程在parseStatementNode()方法中**
```java
public void parseStatementNode() {
        // 获得 id 属性，编号。
        String id = context.getStringAttribute("id");
        // 获得 databaseId ， 判断 databaseId 是否匹配
        String databaseId = context.getStringAttribute("databaseId");

        if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
            return;
        }
        // 获取当前节点名称
        String nodeName = context.getNode().getNodeName();
        // <1> 根据节点名称判断 SQL 类型
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        // 是否为 Select 语句
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        // <2> 是否清空缓存
        boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
        // <3> 是否使用缓存
        boolean useCache = context.getBooleanAttribute("useCache", isSelect);
        // <4> 是否按照查询结果集的顺序来处理数据
        boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

        // Include Fragments before parsing
        XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
        // <5> 将该节点的子节点 <include /> 转换成 <sql /> 节点
        includeParser.applyIncludes(context.getNode());

        // 获取参数类型名称
        String parameterType = context.getStringAttribute("parameterType");
        // <6> 参数类型名称转换成 Java Type
        Class<?> parameterTypeClass = resolveClass(parameterType);

        // <7> 获得 lang 对应的 LanguageDriver 对象
        String lang = context.getStringAttribute("lang");
        LanguageDriver langDriver = getLanguageDriver(lang);

        // Parse selectKey after includes and remove them.
        // <8> 将该节点的子节点 <selectKey /> 解析成 SelectKeyGenerator 生成器
        processSelectKeyNodes(id, parameterTypeClass, langDriver);

        // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
        KeyGenerator keyGenerator;
        String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
        /*
         * <9> 1. 如果上面存在 <selectKey /> 子节点，则获取上面对其解析后生成的 SelectKeyGenerator 2. 否则判断该节点是否配置了 useGeneratedKeys 属性为 true 并且是
         * 插入语句，则使用 Jdbc3KeyGenerator
         */
        if (configuration.hasKeyGenerator(keyStatementId)) {
            keyGenerator = configuration.getKeyGenerator(keyStatementId);
        } else {
            keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                    configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
                    ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        }

        // <10> 创建对应的 SqlSource 对象，保存了该节点下 SQL 相关信息
        SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
        // <11> 获得 Statement 类型，默认 PREPARED
        StatementType statementType = StatementType
                .valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
        Integer fetchSize = context.getIntAttribute("fetchSize");
        Integer timeout = context.getIntAttribute("timeout");
        String parameterMap = context.getStringAttribute("parameterMap");
        // <12> 获得返回结果类型名称
        String resultType = context.getStringAttribute("resultType");
        // 获取返回结果的 Java Type
        Class<?> resultTypeClass = resolveClass(resultType);
        // 获取 resultMap
        String resultMap = context.getStringAttribute("resultMap");
        if (resultTypeClass == null && resultMap == null) {
            resultTypeClass = MapperAnnotationBuilder.getMethodReturnType(builderAssistant.getCurrentNamespace(), id);
        }
        String resultSetType = context.getStringAttribute("resultSetType");
        ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
        if (resultSetTypeEnum == null) {
            resultSetTypeEnum = configuration.getDefaultResultSetType();
        }
        // 对应的 java 属性，结合 useGeneratedKeys 使用
        String keyProperty = context.getStringAttribute("keyProperty");
        // 对应的 column 列名，结合 useGeneratedKeys 使用
        String keyColumn = context.getStringAttribute("keyColumn");
        String resultSets = context.getStringAttribute("resultSets");
        boolean dirtySelect = context.getBooleanAttribute("affectData", Boolean.FALSE);
        // <13> 通过MapperBuilderAssistant构造器根据这些属性信息构建一个MappedStatement对象
        // PS: 一个有点奇葩的方法，21个参数
        builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType, fetchSize, timeout, parameterMap,
                parameterTypeClass, resultMap, resultTypeClass, resultSetTypeEnum, flushCache, useCache, resultOrdered,
                keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets, dirtySelect);
    }
```

### SQL初始化过程
在解析`<select /> <insert /> <update /> <delete />`节点的过程中，是如何解析SQL语句，如何实现动态SQL语句，最终会生成一个o`rg.apache.ibatis.mapping.SqlSource`对象的，对于这烦琐且易出错的过程，我们来看看MyBatis如何实现的？

整体解析过程：
- 在XMLLanguageDriver语言驱动类中，通过XMLScriptBuilder对该到节点的内容进行解析，创建相应的SqlSource资源对象
- 在其解析的过程会根据不同的NodeHandler节点处理器对MyBatis自定义的标签（`<if /> <foreach />`等）进行处理，生成相应的SqlNode对象，
- 最后将所有的SqlNode对象存放在MixedSqlNode中

其中解析的过程中会判断是否为动态的SQL语句，包含了MyBatis自定义的标签或者使用了${}都是动态的SQL语句，动态的SQL语句创建DynamicSqlSource对象，否则创建RawSqlSource对象
而通过SqlSource这个对象根据入参可以获取到对应的BoundSql对象，BoundSql对象中包含了数据库需要执行的SQL语句、ParameterMapping参数信息、入参对象和附加的参数（通过`<bind />`标签生成的，或者`<foreach />`标签中的集合的元素等等）

## MyBatis-SQL执行过程
MyBatis中SQL执行的整体过程如下图所示：
![MyBatis的SQL执行过程](35e49e17/MyBatis的SQL执行过程.jpg)
在 SqlSession 中，会将执行 SQL 的过程交由Executor执行器去执行，过程大致如下：
1. 通过DefaultSqlSessionFactory创建与数据库交互的 SqlSession “会话”，其内部会创建一个Executor执行器对象
2. 然后Executor执行器通过StatementHandler创建对应的java.sql.Statement对象，并通过ParameterHandler设置参数，然后执行数据库相关操作
3. 如果是数据库更新操作，则可能需要通过KeyGenerator先设置自增键，然后返回受影响的行数
4. 如果是数据库查询操作，则需要将数据库返回的ResultSet结果集对象包装成ResultSetWrapper，然后通过DefaultResultSetHandler对结果集进行映射，最后返回 Java 对象

## SqlSession会话与SQL执行入口

## 插件机制

## MyBatis-Spring整合相关源码

## Spring-Boot-Starter相关

## 常见的使用MyBatis遇到的错误
在使用 MyBatis 的过程中，开发者可能会遇到一些常见的错误。以下是这些错误的分类、原因分析以及解决方法：

### 1. **配置错误**
#### **1.1 数据源配置错误**
- **错误现象**：
  - 数据库连接失败。
  - 提示 `Cannot get connection` 或 `DataSource not configured`。
- **原因**：
  - 数据源配置不正确（如 URL、用户名、密码错误）。
  - 数据库驱动未正确加载。
- **解决方法**：
  - 检查数据源配置（如 `jdbc.url`、`jdbc.username`、`jdbc.password`）。
  - 确保数据库驱动已正确添加到项目中。

#### **1.2 MyBatis 配置文件错误**
- **错误现象**：
  - 启动时提示 `Invalid configuration` 或 `Cannot find configuration file`。
- **原因**：
  - `mybatis-config.xml` 文件路径错误或格式不正确。
  - XML 文件中有语法错误。
- **解决方法**：
  - 检查 `mybatis-config.xml` 文件路径是否正确。
  - 使用 XML 校验工具检查文件格式。

### 2. **SQL 映射错误**
#### **2.1 SQL 语句错误**
- **错误现象**：
  - 执行 SQL 时提示语法错误。
  - 查询结果不符合预期。
- **原因**：
  - SQL 语句编写错误（如缺少逗号、关键字拼写错误）。
  - 动态 SQL 拼接错误（如 `<if>` 标签使用不当）。
- **解决方法**：
  - 检查 SQL 语句的正确性。
  - 使用日志打印最终执行的 SQL 语句（开启 MyBatis 的 SQL 日志）。

#### **2.2 参数绑定错误**
- **错误现象**：
  - 提示 `Parameter not found` 或 `BindingException`。
- **原因**：
  - SQL 中的参数名与 Java 方法参数名不一致。
  - 参数类型不匹配。
- **解决方法**：
  - 确保 SQL 中的参数名与方法参数名一致。
  - 使用 `@Param` 注解显式指定参数名。

#### **2.3 结果映射错误**
- **错误现象**：
  - 查询结果无法映射到 Java 对象。
  - 提示 `ResultMap not found` 或 `Unknown column`。
- **原因**：
  - 数据库字段名与 Java 对象属性名不一致。
  - `resultMap` 配置错误。
- **解决方法**：
  - 检查 `resultMap` 配置，确保字段名与属性名一致。
  - 使用别名或 `@Results` 注解显式映射字段。

### 3. **事务管理错误**
#### **3.1 事务未生效**
- **错误现象**：
  - 数据库操作未回滚。
- **原因**：
  - 未配置 Spring 事务管理器。
  - 事务注解 `@Transactional` 未正确使用。
- **解决方法**：
  - 配置 `DataSourceTransactionManager`。
  - 确保 `@Transactional` 注解添加到 Service 层方法上。

#### **3.2 事务传播行为错误**
- **错误现象**：
  - 嵌套事务未按预期执行。
- **原因**：
  - 未正确配置事务传播行为（如 `REQUIRED`、`REQUIRES_NEW`）。
- **解决方法**：
  - 根据业务需求配置事务传播行为。

### 4. **动态 SQL 错误**
#### **4.1 动态 SQL 拼接错误**
- **错误现象**：
  - 动态 SQL 拼接结果不符合预期。
- **原因**：
  - `<if>`、`<choose>` 等标签使用错误。
  - 参数为空时未正确处理。
- **解决方法**：
  - 检查动态 SQL 标签的使用。
  - 使用 `OGNL` 表达式处理空值。

#### **4.2 SQL 注入风险**
- **错误现象**：
  - SQL 语句被恶意注入。
- **原因**：
  - 直接拼接用户输入的参数。
- **解决方法**：
  - 使用 `#{}` 代替 `${}` 进行参数绑定。
  - 避免直接拼接 SQL。

### 5. **缓存错误**
#### **5.1 缓存未生效**
- **错误现象**：
  - 查询结果未缓存。
- **原因**：
  - 未启用 MyBatis 二级缓存。
  - 缓存配置错误。
- **解决方法**：
  - 在 `mybatis-config.xml` 中启用二级缓存。
  - 在 Mapper 接口或 XML 文件中配置缓存。

#### **5.2 缓存脏数据**
- **错误现象**：
  - 查询结果与数据库不一致。
- **原因**：
  - 缓存未及时更新。
- **解决方法**：
  - 在更新操作后清除缓存。
  - 配置缓存的刷新策略。

### 6. **其他常见错误**
#### **6.1 类型转换错误**
- **错误现象**：
  - 提示 `TypeHandler not found` 或 `Cannot convert type`。
- **原因**：
  - 数据库字段类型与 Java 类型不匹配。
- **解决方法**：
  - 使用合适的类型处理器（`TypeHandler`）。
  - 在 SQL 中显式转换类型。

#### **6.2 懒加载错误**
- **错误现象**：
  - 提示 `LazyInitializationException`。
- **原因**：
  - 在 Session 关闭后尝试加载懒加载数据。
- **解决方法**：
  - 在 Session 关闭前加载数据。
  - 使用 `@Transactional` 注解确保 Session 未关闭。

#### **6.3 分页错误**
- **错误现象**：
  - 分页查询结果不正确。
- **原因**：
  - 分页参数传递错误。
  - 分页插件（如 PageHelper）配置错误。
- **解决方法**：
  - 检查分页参数是否正确传递。
  - 确保分页插件配置正确。

### 总结
在使用 MyBatis 时，常见的错误主要集中在配置、SQL 映射、事务管理、动态 SQL 和缓存等方面。通过仔细检查配置、日志和代码逻辑，可以快速定位并解决这些问题。以下是一些通用的排查建议：
1. 开启 MyBatis 的 SQL 日志，检查实际执行的 SQL。
2. 使用单元测试验证 Mapper 方法。
3. 确保 Spring 和 MyBatis 的版本兼容。
4. 参考官方文档和社区资源解决问题。