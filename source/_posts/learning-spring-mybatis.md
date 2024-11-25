---
title: Spring Mybatis细磕
abbrlink: 35e49e17
date: 2024-11-23 18:12:55
updated: 2024-11-25 10:24:27
tags:
  - Spring
  - MyBatis
  - 动态代理
categories: MyBatis
---

## 前言
项目fork注释地址：https://github.com/nimbusking/mybatis-3
【备注】：MyBatis的源码注释非常上，如仔细研究的话，需要花点功夫。

## 一些概念
### MyBatis的实现逻辑
<!-- more -->
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
- **#{}**：在解析 SQL 的时候会将其**替换成 ? 占位符**，然后通过 JDBC 的 `PreparedStatement` 对象添加参数值，这里会进行预编译处理，可以有效的防止 SQL 注入，提高系统的安全性
- **${}**：在 MyBatis 中带有该占位符的 SQL 片段会被解析成动态 SQL 语句，根据入参直接替换掉这个值，然后执行数据库相关操作，**存在 SQL注入 的安全性问题**

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

在 MyBatis 的 MyBatis-Spring 集成 Spring 子项目中，通过实现 Spring 的 BeanDefinitionRegistryPostProcessor 接口，实现它的 postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 方法，也就是在 Spring 完成 BeanDefinition 的初始化工作后，**会将 Mapper 接口也解析成 BeanDefinition 对象注册到 registry 注册表中**，并且会修改其 beanClass 为 MapperFactoryBean 类型，还添加了一个入参为 Mapper 接口的 Class 对象的名称

这样 Mapper 接口会对应一个 MapperFactoryBean 对象，由于这个对象实现了 FactoryBean 接口，实现它的 getObject() 方法，该方法会通过 SqlSession 的 `getMapper(Class<T> type) `方法，返回该 Mapper 接口的动态代理对象，所以在 Spring Bean 中注入的 Mapper 接口时，调用其 getObeject() 方法，拿到的是 Mapper 接口的动态代理对象

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


## MyBatis-SQL执行过程

## SqlSession会话与SQL执行入口

## 插件机制

## MyBatis-Spring整合相关源码

## Spring-Boot-Starter相关

## 其它细节
