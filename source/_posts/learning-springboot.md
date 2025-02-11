---
title: SpringBoot细磕
abbrlink: f52b8ac8
date: 2024-11-18 16:02:34
updated: 2024-11-22 09:29:08
tags:
  - Spring Boot
categories: Spring
---

## 前言
项目注释代码：https://github.com/nimbusking/spring-boot/tree/2.3.x

## 与传统Web架构比较
1. **简化配置**
   - **优势**：Spring Boot 采用“约定优于配置”的理念，提供了大量的默认配置，开发者无需手动配置大量的 XML 或注解。
   - **传统架构**：传统的 Spring MVC 或 Java EE 需要手动配置大量的 XML 文件（如 `web.xml`、`dispatcher-servlet.xml` 等），开发效率较低。
2. **内嵌服务器**
   - **优势**：Spring Boot 内置了 Tomcat、Jetty 或 Undertow 等 Web 服务器，无需额外部署 WAR 文件到外部服务器，直接运行 JAR 文件即可启动应用。
   - **传统架构**：传统 Web 应用需要将 WAR 文件部署到外部的 Tomcat、WebLogic 等服务器，部署流程复杂。
3. **快速启动和开发**
   - **优势**：Spring Boot 提供了丰富的 Starter 依赖，只需引入相关依赖即可快速集成常用功能（如数据库、缓存、安全等），极大提高了开发效率。
   - **传统架构**：传统架构需要手动引入和配置各种依赖，开发周期较长。
4. **自动化依赖管理**
   - **优势**：Spring Boot 通过 `spring-boot-starter-*` 依赖简化了依赖管理，避免了版本冲突问题。
   - **传统架构**：传统架构需要手动管理依赖版本，容易出现依赖冲突。
5. **独立运行**
   - **优势**：Spring Boot 应用可以打包为可执行的 JAR 文件，独立运行，无需依赖外部服务器。
   - **传统架构**：传统 Web 应用需要依赖外部服务器（如 Tomcat）运行。
6. **强大的开发工具支持**
   - **优势**：Spring Boot 提供了丰富的开发工具，如：
     - Spring Boot DevTools：支持热部署，开发时无需重启应用。
     - Actuator：提供应用监控和管理功能。
   - **传统架构**：传统架构缺乏这些工具支持，开发和调试效率较低。
7. **微服务友好**
   - **优势**：Spring Boot 是构建微服务的理想选择，与 Spring Cloud 无缝集成，支持服务发现、负载均衡、配置中心等微服务特性。
   - **传统架构**：传统架构更适合单体应用，微服务支持较弱。
8. **更好的测试支持**
   - **优势**：Spring Boot 提供了完善的测试支持，如 `@SpringBootTest` 注解，可以轻松编写单元测试和集成测试。
   - **传统架构**：传统架构的测试配置复杂，测试效率较低。
9. **生态系统丰富**
   - **优势**：Spring Boot 基于 Spring 生态系统，支持大量的第三方库和工具，社区活跃，文档丰富。
   - **传统架构**：传统架构的生态系统相对较弱，社区支持有限。
10. **云原生支持**
   - **优势**：Spring Boot 天生支持云原生开发，与 Docker、Kubernetes 等云原生技术无缝集成。
   - **传统架构**：传统架构需要额外适配云原生环境。

### 总结对比

| **特性**               | **Spring Boot**                          | **传统 Web 架构**                  |
|------------------------|------------------------------------------|------------------------------------|
| **配置**               | 自动配置，简化开发                       | 手动配置，复杂且繁琐               |
| **服务器**             | 内嵌服务器，独立运行                     | 依赖外部服务器（如 Tomcat）        |
| **依赖管理**           | 自动化依赖管理，避免冲突                 | 手动管理依赖，易冲突               |
| **开发效率**           | 快速启动和开发，工具支持完善             | 开发周期长，工具支持有限           |
| **部署方式**           | 可执行 JAR，独立运行                     | 需要部署 WAR 文件到外部服务器      |
| **微服务支持**         | 与 Spring Cloud 无缝集成                 | 微服务支持较弱                     |
| **测试支持**           | 完善的测试支持                           | 测试配置复杂                       |
| **云原生支持**         | 天生支持云原生                           | 需要额外适配                       |

### 适用场景
- **Spring Boot**：适合快速开发、微服务架构、云原生应用等场景。
- **传统 Web 架构**：适合传统单体应用或对现有系统进行维护的场景。

下面就开始看看SpringBoot的启动

## 从怎么运行Jar包开始
Spring Boot 提供了 Maven 插件 **spring-boot-maven-plugin**，可以很方便的将我们的 Spring Boot 项目打成 jar 包或者 war 包。
考虑到部署的便利性，我们绝大多数（99.99%）的场景下，都会选择打成 jar 包，这样一来，我们就无需将项目部署于 Tomcat、Jetty 等 Servlet 容器中。
<!-- more -->
通过 Spring Boot 插件生成的 jar 包是如何运行，并启动 Spring Boot 应用的呢？
先来看一个SpringBoot打包出来的jar包，我们来看看内部结构
![ABootJarFile](f52b8ac8/ABootJarFile.jpg)
其中：
1. **BOOT-INF 目录**：里面保存了我们自己 Spring Boot 项目编译后的所有文件，其中 classes 目录下面就是编译后的 .class 文件，包括项目中的配置文件等，lib 目录下就是我们引入的第三方依赖
2. **META-INF 目录**：通过 **MANIFEST.MF** 文件提供 jar 包的元数据，声明 jar 的启动类等信息。每个 Java jar 包应该是都有这个文件的，参考 Oracle 官方对于 jar 的说明，里面有一个 **Main-Class 配置用于指定启动类**
3. **org.springframework.boot.loader 目录**：也就是 Spring Boot 的 spring-boot-loader 工具模块，它就是 java -jar xxx.jar 启动 Spring Boot 项目的秘密所在，上面的 Main-Class 指定的就是该工具模块中的一个类

### MANIFEST.MF
```properties
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: spring-boot-demo
Implementation-Version: 0.0.1-SNAPSHOT
Spring-Boot-Layers-Index: BOOT-INF/layers.idx # 文件索引，给Docker镜像用的，先不用管
Start-Class: cn.netbuffer.springboot.demo.SpringBootDemoApplication # 你的 Spring Boot 项目中的启动类
Spring-Boot-Classes: BOOT-INF/classes/ # 你的 Spring Boot 项目编译后的 .class 文件所在目录
Spring-Boot-Lib: BOOT-INF/lib/ # 你的 Spring Boot 项目所引入的第三方依赖所在目录
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.4.5
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher # spring-boot-loader 中的启动类

```

先解释一个问题，**为什么不直接将我们的 Application 启动类设置为 Main-Class 启动呢？**
有俩原因：
1. 通过 Spring Boot Maven Plugin 插件打包后的 jar 包，我们的 .class 文件在 BOOT-INF/classes/ 目录下，在 **Java 默认的 jar 包加载规则下找不到我们的 Application 启动类，也就需要通过 JarLauncher 启动加载。**
2. **Java 规定可执行器的 jar 包禁止嵌套其它 jar 包**，在 BOOT-INF/lib 目录下有我们 Spring Boot 应用依赖的所有第三方 jar 包，因此spring-boot-loader 项目自定义实现了 ClassLoader 实现类 `LaunchedURLClassLoader`，**支持加载 BOOT-INF/classes 目录下的 .class 文件，以及 BOOT-INF/lib 目录下的 jar 包**。

### JarLauncher
![JarLauncher](f52b8ac8/JarLauncher.jpg)
上面的 WarLauncher 是针对 war 包的启动类，和 JarLauncher 差不多，感兴趣的可以看一看，这里我们直接来看到 JarLauncher 这个类
```java
public class JarLauncher extends ExecutableArchiveLauncher {

  static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

  static final String BOOT_INF_LIB = "BOOT-INF/lib/";

  public JarLauncher() {
  }

  protected JarLauncher(Archive archive) {
    super(archive);
  }

  @Override
  protected boolean isNestedArchive(Archive.Entry entry) {
    // 只接受 `BOOT-INF/classes/` 目录
    if (entry.isDirectory()) {
      return entry.getName().equals(BOOT_INF_CLASSES);
    }
    // 只接受 `BOOT-INF/lib/` 目录下的 jar 包
    return entry.getName().startsWith(BOOT_INF_LIB);
  }

  /**
   * 这里是 java -jar 启动 SpringBoot 打包后的 jar 包的入口
   * 可查看 jar 包中的 META-INF/MANIFEST.MF 文件（该文件用于对 Java 应用进行配置）
   * 参考 Oracle 官方对于 jar 的说明（https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html）
   * 该文件其中会有一个配置项：Main-Class: org.springframework.boot.loader.JarLauncher
   * 这个配置表示会调用 JarLauncher#main(String[]) 方法，也就当前方法
   */
  public static void main(String[] args) throws Exception {
    // <1> 创建当前类的实例对象，会创建一个 Archive 对象（当前应用），可用于解析 jar 包（当前应用）中所有的信息
    // <2> 调用其 launch(String[]) 方法
    new JarLauncher().launch(args);
  }
}
```
可以看到它有个 main(String[]) 方法，前面说到的 META-INF/MANIFEST.MF **文件中的 Main-Class 配置就是指向了这个类**，也就会调用这里的 main 方法，会做下面两件事：
1. 创建一个 JarLauncher 实例对象，在 ExecutableArchiveLauncher 父类中会做以下事情：
    ```java
    // org.springframework.boot.loader.ExecutableArchiveLauncher
    public abstract class ExecutableArchiveLauncher extends Launcher {
      public ExecutableArchiveLauncher() {
        try {
          // 为当前应用创建一个 Archive 对象，可用于解析 jar 包（当前应用）中所有的信息
          this.archive = createArchive();
          // 创建一个提供 JAR 排序信息的类路径索引文件，子类实现的
          this.classPathIndex = getClassPathIndex(this.archive);
        }
        catch (Exception ex) {
          throw new IllegalStateException(ex);
        }
      }
    }
    // org.springframework.boot.loader.Launcher#createArchive
      protected final Archive createArchive() throws Exception {
      // 获取 jar 包（当前应用）所在的绝对路径
      ProtectionDomain protectionDomain = getClass().getProtectionDomain();
      CodeSource codeSource = protectionDomain.getCodeSource();
      URI location = (codeSource != null) ? codeSource.getLocation().toURI() : null;
      String path = (location != null) ? location.getSchemeSpecificPart() : null;
      if (path == null) {
        throw new IllegalStateException("Unable to determine code source archive");
      }
      // 当前 jar 包的文件句柄
      File root = new File(path);
      if (!root.exists()) {
        throw new IllegalStateException("Unable to determine code source archive from " + root);
      }
      // 为当前 jar 包创建一个 JarFileArchive（根条目），需要通过它解析出 jar 包中的所有信息
      // 如果是文件夹的话则创建 ExplodedArchive（根条目）
      return (root.isDirectory() ? new ExplodedArchive(root) : new JarFileArchive(root));
    }
    ```
2. 调用 JarLauncher#launch(String[]) 方法，也就是调用父类 Launcher 的这个方法

### Launcher类
Spring Boot 应用的启动器
```java
public abstract class Launcher {

  private static final String JAR_MODE_LAUNCHER = "org.springframework.boot.loader.jarmode.JarModeLauncher";

  /**
   * Launch the application. This method is the initial entry point that should be
   * called by a subclass {@code public static void main(String[] args)} method.
   * @param args the incoming arguments
   * @throws Exception if the application fails to launch
   */
  protected void launch(String[] args) throws Exception {
    // <1> 先判断包是否解压，然后 注册 URL（jar）协议的处理器
    if (!isExploded()) {
      JarFile.registerUrlProtocolHandler();
    }
    // <2> 先从 `archive`（当前 jar 包应用）解析出所有的 JarFileArchive
    // <3> 创建 Spring Boot 自定义的 ClassLoader 类加载器，可加载当前 jar 中所有的类
    ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
    // jarmode docker镜像相关的
    String jarMode = System.getProperty("jarmode");
    String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
    // <4> 获取当前应用的启动类（你自己写的那个 main 方法）
    // <5> 执行你的那个 main 方法
    launch(args, launchClass, classLoader);
  }
}
```

会做下面几件事情：
1. **调用 JarFile#registerUrlProtocolHandler() 方法**：注册 URL（jar）协议的处理器，主要是使用自定义的 URLStreamHandler 处理器处理 jar 包
2. **调用 getClassPathArchivesIterator() 方法**：先从 archive（当前 jar 包应用）解析出所有的 JarFileArchive，这个 archive 就是在上面创建 JarLauncher 实例对象过程中创建的
3. **调用 `createClassLoader(Iterator<Archive>)` 方法**：创建 Spring Boot 自定义的 ClassLoader 类加载器，可加载当前 jar 包中所有的类，包括依赖的第三方包
4. **调用 getMainClass() 方法**：获取当前应用的启动类（你自己写的那个 main 方法所在的 Class 类对象）
5. **调用 launch(...) 方法**：执行你的项目中那个启动类的 main 方法（反射）

**可以理解为会创建一个自定义的 ClassLoader 类加载器，主要可加载 BOOT-INF/classes 目录下的类，以及 BOOT-INF/lib 目录下的 jar 包中的类，然后调用你 Spring Boot 应用的启动类的 main 方法**

接下来看看里面几个重要的方法
#### registerUrlProtocolHandler 方法
```java
public static void registerUrlProtocolHandler() {
    // <1> 获取系统变量中的 `java.protocol.handler.pkgs` 配置的 URLStreamHandler 路径
    Handler.captureJarContextUrl();
    String handlers = System.getProperty(PROTOCOL_HANDLER, "");
    // <2> 将 Spring Boot 自定义的 URL 协议处理器路径（`org.springframework.boot.loader`）添加至系统变量中
    // JVM 启动时会获取 `java.protocol.handler.pkgs` 属性，多个用 `|` 分隔，以他们作为包名前缀，然后使用 `包名前缀.协议名.Handler` 作为该协议的实现
    // 那么这里就会将 `org.springframework.boot.loader.jar.Handler` 作为 jar 包协议的实现
    System.setProperty(PROTOCOL_HANDLER,
        ("".equals(handlers) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
    // <3> 重置已缓存的 URLStreamHandler 处理器们，避免重复创建
    resetCachedUrlHandlers();
  }
```

#### getClassPathArchivesIterator 方法

#### createClassLoader 方法
**创建 Spring Boot 自定义的 ClassLoader 类加载器，可加载当前 jar 中所有的类**

### LaunchedURLClassLoader类
org.springframework.boot.loader.LaunchedURLClassLoader 是 spring-boot-loader 中自定义的类加载器，**实现对 jar 包中 BOOT-INF/classes 目录下的类和 BOOT-INF/lib 下第三方 jar 包中的类的加载。**


## SpringApplication启动类启动过程
Spring Boot 项目的启动类通常有下面三种方式
```java
// 方式一
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
// 方式二
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(DemoApplication.class).run(args);
    }
}
// 方式三
@SpringBootApplication
public class DemoApplication extends SpringBootServletInitializer {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(DemoApplication.class);
    }
}
```
其中：**方式一和方式二本质上都是通过调用 SpringApplication#run(..) 方法来启动应用**，不同的是方式二通过构建器模式，先构建一个 SpringApplication 实例对象，然后调用其 run(..) 方法启动应用，这种方式可以对 SpringApplication 进行配置，更加的灵活。
**方式三**，它和方式一差不多，不同的是它继承了 SpringBootServletInitializer 这个类，**作用就是当你的 Spring Boot 项目打成 war 包时能够放入外部的 Tomcat 容器中运行**，如果是 war 包，那上面的 main(...) 方法自然是不需要的，当然，configure(..) 方法也可以不要

### 启动流程概括
1. **启动应用**  
   - 执行`main`方法，调用`SpringApplication.run()`。

2. **初始化SpringApplication**  
   - 设置应用类型（Servlet、Reactive等）。
   - 加载`ApplicationContextInitializer`和`ApplicationListener`。
   - 读取`spring.factories`中的配置。

3. **准备环境**  
   - 创建并配置`Environment`（加载配置文件如`application.properties`）。

4. **创建应用上下文**  
   - 根据应用类型创建`ApplicationContext`（如`AnnotationConfigServletWebServerApplicationContext`，实际持有的是ServletWebServerApplicationContext对象）。

5. **刷新应用上下文**  
   调用`refresh()`方法，执行以下步骤：
   - 准备上下文。
   - 加载Bean定义。
   - 调用BeanFactoryPostProcessor。
   - 注册BeanPostProcessor。
   - 初始化MessageSource。
   - 初始化事件广播器。
   - 注册监听器。
   - 实例化单例Bean。
   - 完成上下文刷新。

6. **执行Runners**  
   - 执行`CommandLineRunner`和`ApplicationRunner`。

7. **启动完成**  
   - 应用启动完成，开始处理请求。

### SpringApplication类
Spring 应用启动器。正如其代码上所添加的注释，它来提供启动 Spring 应用的功能。
#### 相关属性
```java
public class SpringApplication {

  /**
   * Spring 应用上下文（非 Web 场景）
   * The class name of application context that will be used by default for non-web environments.
   */
  public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
      + "annotation.AnnotationConfigApplicationContext";

  /**
   * Spring 应用上下文（Web 场景）
   * The class name of application context that will be used by default for web environments.
   */
  public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
      + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

  /**
   * Spring 应用上下文（Reactive 场景）
   * The class name of application context that will be used by default for reactive web environments.
   */
  public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
      + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

  /**
   * 主 Bean（通常为我们的启动类，优先注册）
   */
  private Set<Class<?>> primarySources;
  /**
   * 来源 Bean（优先注册）
   */
  private Set<String> sources = new LinkedHashSet<>();
  /**
   * 启动类
   */
  private Class<?> mainApplicationClass;
  /**
   * Banner 打印模式
   */
  private Banner.Mode bannerMode = Banner.Mode.CONSOLE;
  /**
   * 是否打印应用启动耗时日志
   */
  private boolean logStartupInfo = true;
  /**
   * 是否接收命令行中的参数
   */
  private boolean addCommandLineProperties = true;
  /**
   * 是否设置 ConversionService 类型转换器
   */
  private boolean addConversionService = true;
  /**
   * Banner 对象（用于输出横幅）
   */
  private Banner banner;
  /**
   * 资源加载对象
   */
  private ResourceLoader resourceLoader;
  /**
   * Bean 名称生成器
   */
  private BeanNameGenerator beanNameGenerator;
  /**
   * Spring 应用的环境对象
   */
  private ConfigurableEnvironment environment;
  /**
   * Spring 应用上下文的 Class 对象
   */
  private Class<? extends ConfigurableApplicationContext> applicationContextClass;
  /**
   * Web 应用的类型（Servlet、Reactive）
   */
  private WebApplicationType webApplicationType;
  /**
   * 是否注册钩子函数，用于 JVM 关闭时关闭 Spring 应用上下文
   */
  private boolean registerShutdownHook = true;
  /**
   * 保存 ApplicationContextInitializer 对象（主要是对 Spring 应用上下文做一些初始化工作）
   */
  private List<ApplicationContextInitializer<?>> initializers;
  /**
   * 保存 ApplicationListener 监听器（支持在整个 Spring Boot 的多个时间点进行扩展）
   */
  private List<ApplicationListener<?>> listeners;
  /**
   * 默认的配置项
   */
  private Map<String, Object> defaultProperties;
  /**
   * 额外的 profile
   */
  private Set<String> additionalProfiles = new HashSet<>();
  /**
   * 是否允许覆盖 BeanDefinition
   */
  private boolean allowBeanDefinitionOverriding;
  /**
   * 是否为自定义的 Environment 对象
   */
  private boolean isCustomEnvironment = false;
  /**
   * 是否支持延迟初始化（需要通过 {@link LazyInitializationExcludeFilter} 过滤）
   */
  private boolean lazyInitialization = false;
}

```

#### 构造器
```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // <1> 设置资源加载器
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // <2> 设置主要的 Class 类对象，通常是我们的启动类
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // <3> 通过 `classpath` 判断是否存在相应的 Class 类对象，来决定当前 Web 应用的类型（REACTIVE、SERVLET、NONE），默认为 **SERVLET**
    // 不同的类型后续创建的 Environment 类型不同
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    /**
     * <4> 初始化所有 `ApplicationContextInitializer` 类型的对象，并保存至 `initializers` 集合中
     * 通过类加载器从 `META-INF/spring.factories` 文件中获取 ApplicationContextInitializer 类型的类名称，并进行实例化
     */
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    /**
     * <5> 初始化所有 `ApplicationListener` 类型的对象，并保存至 `listeners` 集合中
     * 通过类加载器从 `META-INF/spring.factories` 文件中获取 ApplicationListener 类型的类名称，并进行实例化
     * 例如有一个 {@link ConfigFileApplicationListener}
     */
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // <6> 获取当前被调用的 `main` 方法所属的 Class 类对象，并设置（主要用于打印日志）
    this.mainApplicationClass = deduceMainApplicationClass();
}
```
实例化的过程中做了不少事情，如下：
1. 设置资源加载器，默认为 null，可以通过 SpringApplicationBuilder 设置
2. 设置 primarySources 为主要的 Class 类对象，通常是我们的启动类
3. 通过 classpath 判断是否存在相应的 Class 类对象，来决定当前 Web 应用的类型（REACTIVE、SERVLET、NONE），默认为 SERVLET，不同的类型后续创建的 Environment 类型不同
    ```java
    public enum WebApplicationType {

      NONE, SERVLET, REACTIVE;

      private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
          "org.springframework.web.context.ConfigurableWebApplicationContext" };

      private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

      private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

      private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

      private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";

      private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

      static WebApplicationType deduceFromClasspath() {
        if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
          return WebApplicationType.REACTIVE;
        }
        for (String className : SERVLET_INDICATOR_CLASSES) {
          if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
          }
        }
        return WebApplicationType.SERVLET;
      }
    }
    ```
4. 初始化所有 ApplicationContextInitializer 类型的对象，并保存至 initializers 集合中
5. 初始化所有 ApplicationListener 类型的对象，并保存至 listeners 集合中，例如 ConfigFileApplicationListener 和 LoggingApplicationListener
6. 获取当前被调用的 main 方法所属的 Class 类对象，并设置（主要用于打印日志）
  ```java
  private Class<?> deduceMainApplicationClass() {
      try {
          // 获取当前的调用栈
          StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
          // 对调用栈进行遍历，找到 `main` 方法所在的 Class 类对象并返回
          for (StackTraceElement stackTraceElement : stackTrace) {
              if ("main".equals(stackTraceElement.getMethodName())) {
                  return Class.forName(stackTraceElement.getClassName());
              }
          }
      } catch (ClassNotFoundException ex) {
          // Swallow and continue
      }
      return null;
  }
  ```

上面的第 4 和 5 步都是通过类加载器**从 META-INF/spring.factories 文件中分别获取 ApplicationContextInitializer 和 ApplicationListener 类型的类名称**，然后进行实例化。
这个两种类型的对象都是对 Spring 的一种拓展，像很多框架整合 Spring Boot 都可以通过自定义的 ApplicationContextInitializer 对 ApplicationContext 进行一些初始化，通过 ApplicationListener 在 Spring 应用启动的不同阶段来织入一些功能。

#### SpringApplication#run 方法
直接看run方法
```java
public ConfigurableApplicationContext run(String... args) {
    // <1> 创建 StopWatch 对象并启动，主要用于统计当前方法执行过程的耗时
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // <2> 设置 `java.awt.headless` 属性，和 AWT 相关，暂时忽略
    configureHeadlessProperty();
    // <3> 初始化所有 `SpringApplicationRunListener` 类型的对象，并全部封装到 `SpringApplicationRunListeners` 对象中
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // <4> 启动所有的 `SpringApplicationRunListener` 监听器
    // 例如 `EventPublishingRunListener` 会广播 ApplicationEvent 应用正在启动的事件，
    // 它里面封装了所有的 `ApplicationListener` 对象，那么此时就可以通过它们做一些初始化工作，进行拓展
    listeners.starting();
    try {
        // <5> 创建一个应用参数对象，将 `main(String[] args)` 方法的 `args` 参数封装起来，便于后续使用
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // <6> 准备好当前应用 Environment 环境，这里会加载出所有的配置信息，包括 `application.yaml` 和外部的属性配置
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

        configureIgnoreBeanInfo(environment);
        // <7> 打印 banner 内容
        Banner printedBanner = printBanner(environment);
        // <8> 对 `context` （Spring 上下文）进行实例化
        // 例如 Servlet（默认）会创建一个 AnnotationConfigServletWebServerApplicationContext 实例对象
        context = createApplicationContext();
        // <9> 获取异常报告器，通过类加载器从 `META-INF/spring.factories` 文件中获取 SpringBootExceptionReporter 类型的类名称，并进行实例化
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // <10> 对 Spring 应用上下文做一些初始化工作，例如执行 ApplicationContextInitializer#initialize(..) 方法
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        /**
         * <11> 刷新 Spring 应用上下文，在这里会完成所有 Spring Bean 的初始化，同时会初始化好 Servlet 容器，例如 Tomcat
         * 这一步涉及到 Spring IoC 的所有内容，参考[死磕Spring之IoC篇 - Spring 应用上下文 ApplicationContext](https://www.cnblogs.com/lifullmoon/p/14453083.html)
         * 在 {@link ServletWebServerApplicationContext#onRefresh()} 方法中会创建一个 Servlet 容器（默认为 Tomcat），也就是当前 Spring Boot 应用所运行的 Web 环境
         */
        refreshContext(context);
        // <12> 完成刷新 Spring 应用上下文的后置操作，空实现，扩展点
        afterRefresh(context, applicationArguments);
        // <13> 停止 StopWatch，统计整个 Spring Boot 应用的启动耗时，并打印
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // <14> 对所有的 SpringApplicationRunListener 监听器进行广播，发布 ApplicationStartedEvent 应用已启动事件
        listeners.started(context);
        // <15> 回调 IoC 容器中所有 ApplicationRunner 和 CommandLineRunner 类型的启动器
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 处理异常，同时会将异常发送至上面第 `9` 步获取到的异常报告器
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // <16> 对所有的 SpringApplicationRunListener 监听器进行广播，发布 ApplicationReadyEvent 应用已就绪事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        // 处理异常，同时会将异常发送至上面第 `9` 步获取到的异常报告器
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}

```

#### refreshContext 方法
refreshContext(ConfigurableApplicationContext) 方法，刷新 Spring 应用上下文，在这里会完成所有 Spring Bean 的初始化，同时会初始化好 Servlet 容器，例如 Tomcat
```java
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        try {
            // 为当前 Spring 应用上下文注册一个钩子函数
            // 在 JVM 关闭时先关闭 Spring 应用上下文
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
    // 调用 AbstractApplicationContext#refresh() 方法，刷新 Spring 上下文
    refresh(context);
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}

```
前文第8步 createApplicationContext 方法 方法中讲到，我们默认情况下是 SERVLET 应用类型，也就是创建一个 AnnotationConfigServletWebServerApplicationContext 对象，在其父类 ServletWebServerApplicationContext 中重写了 onRefresh() 方法，会创建一个 Servlet 容器（默认为 Tomcat），也就是当前 Spring Boot 应用所运行的 Web 环境。
**这部分就跟我们的SpringIOC关联起来了，下面一小节就来介绍是怎么关联的。**

## 内嵌Tomcat容器实现
### 先来回顾一下refresh方法
```java
/**
 * 刷新上下文，在哪会被调用？
 * 在 **Spring MVC** 中，{@link org.springframework.web.context.ContextLoader#initWebApplicationContext} 方法初始化上下文时，会调用该方法
 */
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

            // <6> 执行 BeanFactoryPostProcessor 处理器，包含 BeanDefinitionRegistryPostProcessor 处理器
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

            // <12> 设置 ConversionService 类型转换器，**初始化**所有还未初始化的 Bean（不是抽象、单例模式、不是懒加载方式）
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
这里面跟springboot关联的是第10步和第13步：
在该方法的第 10 步可以看到会调用 onRefresh() 方法再进行一些初始化工作，**这个方法交由子类进行扩展，那么在 Spring Boot 中的 ServletWebServerApplicationContext 重写了该方法，会创建一个 Servlet 容器（默认为 Tomcat）**，也就是当前 Spring Boot 应用所运行的 Web 环境。
第 13 步会调用 **onRefresh() 方法，ServletWebServerApplicationContext 重写了该方法**，启动 WebServer，对 Servlet 进行加载并初始化

### 小结
Spring Boot 的 **Spring 应用上下文（ServletWebServerApplicationContext）在 refresh() 刷新阶段进行了扩展，分别在 onRefresh() 和 finishRefresh() 两个地方**，分别做了以下事情：
1. **创建一个 WebServer 服务对象**，例如 TomcatWebServer 对象，对 Tomcat 的封装，用于控制 Tomcat 服务器
    1. **先创建一个 org.apache.catalina.startup.Tomcat 对象** tomcat，使用临时目录作为基础目录（tomcat.端口号），退出时删除，同时会设置端口、编码、最小空闲线程和最大线程数
    2. **为 tomcat 创建一个 TomcatEmbeddedContext 上下文对象**，会添加一个 TomcatStarter（实现 javax.servlet.ServletContainerInitializer 接口）到这个上下文对象中
    3. 将 **tomcat 封装到 TomcatWebServer 对象中，实例化过程会启动 tomcat**，启动后会触发 javax.servlet.ServletContainerInitializer 实现类的回调，也就会触发 TomcatStarter 的回调，在其内部会调用 Spring Boot 自己的 ServletContextInitializer 初始器，例如 ServletWebServerApplicationContext#selfInitialize(ServletContext) 匿名方法
    4. 在这个匿名方法中**会找到所有的 RegistrationBean，执行他们的 onStartup 方法**，将其关联的 Servlet、Filter 和 EventListener 添加至 Servlet 上下文中，**包括 Spring MVC 的 DispatcherServlet 对象**
2. **启动上一步创建的 TomcatWebServer 对象**，上面仅启动 Tomcat 容器，Servlet 添加到了 ServletContext 上下文中，这里会将这些 Servlet 进行加载并初始化

## 外部Tomcat容器实现
TODO 针对WAR包的处理

## 剖析@SpringBootApplication注解
SpringBootApplication 注解在 Spring Boot 的 spring-boot-autoconfigre 子模块下，当我们**引入 spring-boot-starter 模块后会自动引入该子模块**
该注解是一个组合注解，如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited // 表明该注解定义在某个类上时，其子类会继承该注解
@SpringBootConfiguration // 继承 `@Configuration` 注解
@EnableAutoConfiguration // 开启自动配置功能
// 扫描指定路径下的 Bean
@ComponentScan( excludeFilters = {
          // 默认没有 TypeExcludeFilter
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
          // 排除掉自动配置类
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
  /**
   * 需要自动配置的 Class 类
   */
  @AliasFor(annotation = EnableAutoConfiguration.class)
  Class<?>[] exclude() default {};

  /**
   * 需要自动配置的类名称
   */
  @AliasFor(annotation = EnableAutoConfiguration.class)
  String[] excludeName() default {};

  /**
   * 需要扫描的路径
   */
  @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
  String[] scanBasePackages() default {};

  /**
   * 需要扫描的 Class 类
   */
  @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
  Class<?>[] scanBasePackageClasses() default {};

  /**
   * 被标记的 Bean 是否进行 CGLIB 提升
   */
  @AliasFor(annotation = Configuration.class)
  boolean proxyBeanMethods() default true;
}
```

### @SpringBootConfiguration 注解
SpringBootConfiguration 注解，Spring Boot 自定义注解
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

    /**
   * 被标记的 Bean 是否进行 CGLIB 提升
   */
  @AliasFor(annotation = Configuration.class)
  boolean proxyBeanMethods() default true;
}

```
该注解很简单，上面标注了 @Configuration 元注解，所以作用相同，**同样是将一个类标注为配置类，能够作为一个 Bean 被 Spring IoC 容器管理**

### @ComponentScan 注解
ComponentScan 注解，Spring 注解，扫描指定路径下的标有 @Component 注解的类，解析成 Bean 被 Spring IoC 容器管理
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {

    /**
     * 指定的扫描的路径
     */
  @AliasFor("basePackages")
  String[] value() default {};

    /**
     * 指定的扫描的路径
     */
  @AliasFor("value")
  String[] basePackages() default {};

    /**
     * 指定的扫描的 Class 对象
     */
  Class<?>[] basePackageClasses() default {};

  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

  Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;

  ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;

  String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;

  boolean useDefaultFilters() default true;

    /**
     * 扫描时的包含过滤器
     */
  Filter[] includeFilters() default {};
  /**
     * 扫描时的排除过滤器
     */
  Filter[] excludeFilters() default {};

  boolean lazyInit() default false;
}
```
该注解通常需要和 @Configuration 注解一起使用，因为需要 **先被当做一个配置类，然后解析到上面有 @ComponentScan** 注解后则处理该注解，通过 `ClassPathBeanDefinitionScanner` 扫描器去扫描指定路径下标注了 @Component 注解的类，将他们解析成 BeanDefinition（Bean 的前身），后续则会生成对应的 Bean 被 Spring IoC 容器管理

当然，如果该注解没有通过 basePackages 指定路径，**Spring 会选在以该注解标注的类所在的包作为基础路径**，然后扫描包下面的这些类

### @EnableAutoConfiguration 注解
EnableAutoConfiguration 注解，Spring Boot 自定义注解，**用于驱动 Spring Boot 自动配置模块，这个也是SpringBoot能够实现自动装配的核心注解**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage // 注册一个 Bean 保存当前注解标注的类所在包路径
@Import(AutoConfigurationImportSelector.class) // Spring Boot 自动配置的实现
public @interface EnableAutoConfiguration {
  /**
   * 可通过这个配置关闭 Spring Boot 的自动配置功能
   */
  String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

  /**
   * 指定需要排除的自动配置类的 Class 对象
   */
  Class<?>[] exclude() default {};

  /**
   * 指定需要排除的自动配置类的名称
   */
  String[] excludeName() default {};
}

```
**对于 Spring 中的模块驱动注解的实现都是通过 @Import 注解来实现的**
模块驱动注解通常需要结合 @Configuration 注解一起使用，因为需要先被当做一个配置类，然后解析到上面有 @Import 注解后则进行处理，对于 @Import 注解的值有三种情况：
1. 该 Class 对象实现了 `ImportSelector` 接口，调用它的 `selectImports(..)` 方法获取需要被处理的 Class 对象的名称，也就是可以将它们作为一个 Bean 被 Spring IoC 管理
    - *该 Class 对象实现了 DeferredImportSelector 接口，和上者的执行时机不同，在所有配置类处理完后再执行，且支持 @Order 排序*
2. 该 Class 对象实现了 `ImportBeanDefinitionRegistrar` 接口，会调用它的 `registerBeanDefinitions(..)` 方法，自定义地往 `BeanDefinitionRegistry` 注册中心注册 BeanDefinition（Bean 的前身）
3. 该 Class 对象是一个 @Configuration 配置类，会将这个类作为一个 Bean 被 Spring IoC 管理

这里的 @EnableAutoConfiguration 自动配置模块驱动注解，通过 @Import 导入 **AutoConfigurationImportSelector** 这个类（实现了 DeferredImportSelector 接口）来驱动 Spring Boot 的自动配置模块，下面会进行分析

### @AutoConfigurationPackage 注解
@EnableAutoConfiguration 注解上面还有一个 @AutoConfigurationPackage 元注解，它的作用就是**注册一个 Bean，保存了当前注解标注的类所在包路径**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
/**
 * 将当前注解所标注的类所在包名封装成一个 {@link org.springframework.boot.autoconfigure.AutoConfigurationPackages.BasePackages} 进行注册
 * 例如 JPA 模块的会使用到这个对象（JPA entity scanner）
 */
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage { }

```
同样这里使用了 @Import 注解来实现的，对应的是一个 AutoConfigurationPackages.Registrar 内部类，如下：
```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 注册一个 BasePackages 类的 BeanDefinition，角色为内部角色，名称为 `org.springframework.boot.autoconfigure.AutoConfigurationPackages`
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        // 将注解元信息封装成 PackageImport 对象，对注解所在的包名进行封装
        return Collections.singleton(new PackageImport(metadata));
    }
}

```

### AutoConfigurationImportSelector类
AutoConfigurationImportSelector，实现了 DeferredImportSelector 接口，是 @EnableAutoConfiguration 注解驱动自动配置模块的核心类
#### selectImports 方法
selectImports(AnnotationMetadata) 方法，返回需要注入的 Bean 的类名称
```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // <1> 如果通过 `spring.boot.enableautoconfiguration` 配置关闭了自动配置功能
    if (!isEnabled(annotationMetadata)) {
        // 返回一个空数组
        return NO_IMPORTS;
    }
    /**
     * <2> 解析 `META-INF/spring-autoconfigure-metadata.properties` 文件，生成一个 AutoConfigurationMetadata 自动配置类元数据对象
     *
     * 说明：引入 `spring-boot-autoconfigure-processor` 工具模块依赖后，其中会通过 Java SPI 机制引入 {@link AutoConfigureAnnotationProcessor} 注解处理器在编译阶段进行相关处理
     * 其中 `spring-boot-autoconfigure` 模块会引入该工具模块（不具有传递性），那么 Spring Boot 在编译 `spring-boot-autoconfigure` 这个 `jar` 包的时候，
     * 在编译阶段会扫描到带有 `@ConditionalOnClass` 等注解的 `.class` 文件，也就是自动配置类，将自动配置类的信息保存至 `META-INF/spring-autoconfigure-metadata.properties` 文件中
     * 例如保存类 `自动配置类类名.注解简称` => `注解中的值(逗号分隔)` 和 `自动配置类类名` => `空字符串`
     *
     * 当然，你自己写的 Spring Boot Starter 中的自动配置模块也可以引入这个 Spring Boot 提供的插件
     */
    AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
    // <3> 从所有的 `META-INF/spring.factories` 文件中找到 `@EnableAutoConfiguration` 注解对应的类（需要自动配置的类）
    // 会进行过滤处理，然后封装在一个对象中
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata, annotationMetadata);
    // <4> 返回所有需要自动配置的类
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

```

上面第 2 步调用的方法：
```java
final class AutoConfigurationMetadataLoader {

  protected static final String PATH = "META-INF/spring-autoconfigure-metadata.properties";

  private AutoConfigurationMetadataLoader() {
  }

  static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
    return loadMetadata(classLoader, PATH);
  }

  static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
    try {
      // <1> 获取所有 `META-INF/spring-autoconfigure-metadata.properties` 文件 URL
      Enumeration<URL> urls = (classLoader != null) ? classLoader.getResources(path)
          : ClassLoader.getSystemResources(path);
      Properties properties = new Properties();
      // <2> 加载这些文件并将他们的属性添加到 Properties 中
      while (urls.hasMoreElements()) {
        properties.putAll(PropertiesLoaderUtils.loadProperties(new UrlResource(urls.nextElement())));
      }
      // <3> 将这个 Properties 封装到 PropertiesAutoConfigurationMetadata 对象中并返回
      return loadMetadata(properties);
    }
    catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load @ConditionalOnClass location [" + path + "]", ex);
    }
  }

  static AutoConfigurationMetadata loadMetadata(Properties properties) {
    return new PropertiesAutoConfigurationMetadata(properties);
  }
}
```
这里补充解释一下spring-autoconfigure-metadata.properties作用：
Spring Boot 做了一个优化，通过自己提供的工具，**在编译阶段将自动配置类的一些注解信息保存在一个 properties**文件中，这样一来，在你启动应用的过程中，就可以直接读取该文件中的信息，提前过滤掉一些自动配置类，相比于每次都去解析它们所有的注解，性能提升不少

#### getAutoConfigurationEntry 方法
getAutoConfigurationEntry(AutoConfigurationMetadata, AnnotationMetadata) 方法，返回符合条件的自动配置类，如下：
```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
        AnnotationMetadata annotationMetadata) {
    // <1> 如果通过 `spring.boot.enableautoconfiguration` 配置关闭了自动配置功能
    if (!isEnabled(annotationMetadata)) {
        // 则返回一个“空”的对象
        return EMPTY_ENTRY;
    }
    // <2> 获取 `@EnableAutoConfiguration` 注解的配置信息
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // <3> 从所有的 `META-INF/spring.factories` 文件中找到 `@EnableAutoConfiguration` 注解对应的类（需要自动配置的类）
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // <4> 对所有的自动配置类进行去重
    configurations = removeDuplicates(configurations);
    // <5> 获取需要排除的自动配置类
    // 可通过 `@EnableAutoConfiguration` 注解的 `exclude` 和 `excludeName` 配置
    // 也可以通过 `spring.autoconfigure.exclude` 配置
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    // <6> 处理 `exclusions` 中特殊的类名称，保证能够排除它
    checkExcludedClasses(configurations, exclusions);
    // <7> 从 `configurations` 中将 `exclusions` 需要排除的自动配置类移除
    configurations.removeAll(exclusions);
    /**
     * <8> 从 `META-INF/spring.factories` 找到所有的 {@link AutoConfigurationImportFilter} 对 `configurations` 进行过滤处理
     * 例如 Spring Boot 中配置了 {@link org.springframework.boot.autoconfigure.condition.OnClassCondition}
     * 在这里提前过滤掉一些不满足条件的自动配置类，在 Spring 注入 Bean 的时候也会判断哦~
     */
    configurations = filter(configurations, autoConfigurationMetadata);
    /**
     * <9> 从 `META-INF/spring.factories` 找到所有的 {@link AutoConfigurationImportListener} 事件监听器
     * 触发每个监听器去处理 {@link AutoConfigurationImportEvent} 事件，该事件中包含了 `configurations` 和 `exclusions`
     * Spring Boot 中配置了一个 {@link org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener}
     * 目的就是将 `configurations` 和 `exclusions` 保存至 {@link AutoConfigurationImportEvent} 对象中，并注册到 IoC 容器中，名称为 `autoConfigurationReport`
     * 这样一来我们可以注入这个 Bean 获取到自动配置类信息
     */
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // <10> 将所有的自动配置类封装成一个 AutoConfigurationEntry 对象，并返回
    return new AutoConfigurationEntry(configurations, exclusions);
}

```
这个过程解释如下：
1. 如果通过 spring.boot.enableautoconfiguration 配置关闭了自动配置功能，则返回一个“空”的对象
2. 获取 @EnableAutoConfiguration 注解的配置信息
3. 从所有的 **META-INF/spring.factories** 文件中找到 **@EnableAutoConfiguration 注解对应的类**（需要自动配置的类），保存在 configurations 集合中
    ```java
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        // 从所有的 `META-INF/spring.factories` 文件中找到 `@EnableAutoConfiguration` 注解对应的类（需要自动配置的类）
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        // 如果为空则抛出异常
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
                + "are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
    ```
4. 对所有的自动配置类进行去重
    ```java
    protected final <T> List<T> removeDuplicates(List<T> list) {
        return new ArrayList<>(new LinkedHashSet<>(list));
    }
    ```
5. 获取需要排除的自动配置类，可通过 @EnableAutoConfiguration 注解的 exclude 和 excludeName 配置，也可以通过 spring.autoconfigure.exclude 配置
6. 处理 exclusions 中特殊的类名称，保证能够排除它
7. 从 configurations 中将 exclusions 需要排除的自动配置类移除
8. 调用 filter(..) 方法， 目的就是过滤掉一些不符合 Condition 条件的自动配置类，和在 1. selectImports 方法 小节中讲到的性能优化有关哦
9. 从 META-INF/spring.factories 找到所有的 AutoConfigurationImportListener 事件监听器，触发每个监听器去处理 AutoConfigurationImportEvent 事件，该事件中包含了 configurations 和 exclusions
    Spring Boot 中配置了一个监听器，**目的就是将 configurations 和 exclusions 保存至 AutoConfigurationImportEvent 对象中，并注册到 IoC 容器中**，名称为 autoConfigurationReport，这样一来我们可以注入这个 Bean 获取到自动配置类信息
10. 将所有的自动配置类封装成一个 AutoConfigurationEntry 对象，并返回



### AutoConfigureAnnotationProcessor
AutoConfigureAnnotationProcessor，Spring Boot 的 spring-boot-autoconfigure-processor 工具模块中的自动配置类的注解处理器，在编译阶段扫描自动配置类的注解元信息，并将他们保存至一个 properties 文件中
```java
@SupportedAnnotationTypes({ "org.springframework.boot.autoconfigure.condition.ConditionalOnClass",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnBean",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnSingleCandidate",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication",
    "org.springframework.boot.autoconfigure.AutoConfigureBefore",
    "org.springframework.boot.autoconfigure.AutoConfigureAfter",
    "org.springframework.boot.autoconfigure.AutoConfigureOrder" })
public class AutoConfigureAnnotationProcessor extends AbstractProcessor {
  /**
   * 生成的文件
   */
  protected static final String PROPERTIES_PATH = "META-INF/spring-autoconfigure-metadata.properties";
  /**
   * 保存指定注解的简称和注解全称之间的对应关系（不可修改）
   */
  private final Map<String, String> annotations;

  private final Map<String, ValueExtractor> valueExtractors;

  private final Properties properties = new Properties();

  public AutoConfigureAnnotationProcessor() {
    // <1> 创建一个 Map 集合
    Map<String, String> annotations = new LinkedHashMap<>();
    // <1.1> 将指定注解的简称和全称之间的对应关系保存至第 `1` 步创建的 Map 中
    addAnnotations(annotations);
    // <1.2> 将 `1.1` 的 Map 转换成不可修改的 UnmodifiableMap 集合，赋值给 `annotations`
    this.annotations = Collections.unmodifiableMap(annotations);
    // <2> 创建一个 Map 集合
    Map<String, ValueExtractor> valueExtractors = new LinkedHashMap<>();
    // <2.1> 将指定注解的简称和对应的 ValueExtractor 对象保存至第 `2` 步创建的 Map 中
    addValueExtractors(valueExtractors);
    // <2.2> 将 `2.1` 的 Map 转换成不可修改的 UnmodifiableMap 集合，赋值给 `valueExtractors`
    this.valueExtractors = Collections.unmodifiableMap(valueExtractors);
  }
}
```
其中：
AbstractProcessor 是 JDK 1.6 引入的一个抽象类，支持在编译阶段进行处理，在构造器中做了以下事情：
1. 创建一个 Map 集合
  1. 将指定注解的简称和全称之间的对应关系保存至第 1 步创建的 Map 中
    ```java
    protected void addAnnotations(Map<String, String> annotations) {
        annotations.put("ConditionalOnClass", "org.springframework.boot.autoconfigure.condition.ConditionalOnClass");
        annotations.put("ConditionalOnBean", "org.springframework.boot.autoconfigure.condition.ConditionalOnBean");
        annotations.put("ConditionalOnSingleCandidate", "org.springframework.boot.autoconfigure.condition.ConditionalOnSingleCandidate");
        annotations.put("ConditionalOnWebApplication", "org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication");
        annotations.put("AutoConfigureBefore", "org.springframework.boot.autoconfigure.AutoConfigureBefore");
        annotations.put("AutoConfigureAfter", "org.springframework.boot.autoconfigure.AutoConfigureAfter");
        annotations.put("AutoConfigureOrder", "org.springframework.boot.autoconfigure.AutoConfigureOrder");
    }
    ```
  2. 将 1.1 的 Map 转换成不可修改的 UnmodifiableMap 集合，赋值给 annotations
2. 创建一个 Map 集合
  1. 将指定注解的简称和对应的 ValueExtractor 对象保存至第 2 步创建的 Map 中
    ```java
    private void addValueExtractors(Map<String, ValueExtractor> attributes) {
        attributes.put("ConditionalOnClass", new OnClassConditionValueExtractor());
        attributes.put("ConditionalOnBean", new OnBeanConditionValueExtractor());
        attributes.put("ConditionalOnSingleCandidate", new OnBeanConditionValueExtractor());
        attributes.put("ConditionalOnWebApplication", ValueExtractor.allFrom("type"));
        attributes.put("AutoConfigureBefore", ValueExtractor.allFrom("value", "name"));
        attributes.put("AutoConfigureAfter", ValueExtractor.allFrom("value", "name"));
        attributes.put("AutoConfigureOrder", ValueExtractor.allFrom("value"));
    }
    ```
  2. 将 2.1 的 Map 转换成不可修改的 UnmodifiableMap 集合，赋值给 valueExtractors

#### process 方法
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    // <1> 遍历上面的几个 `@Conditional` 注解和几个定义自动配置类顺序的注解，依次进行处理
    for (Map.Entry<String, String> entry : this.annotations.entrySet()) {
        // <1.1> 对支持的注解进行处理，也就是找到所有标注了该注解的类，然后解析出该注解的值，保存至 Properties
        // 例如 `类名.注解简称` => `注解中的值(逗号分隔)` 和 `类名` => `空字符串`，将自动配置类的信息已经对应注解的信息都保存起来
        // 避免你每次启动 Spring Boot 应用都要去解析自动配置类上面的注解，是引入 `spring-boot-autoconfigure` 后可以从 `META-INF/spring-autoconfigure-metadata.properties` 文件中直接获取
        // 这么一想，Spring Boot 设计的太棒了，所以你自己写的 Spring Boot Starter 中的自动配置模块也可以引入这个 Spring Boot 提供的插件
        process(roundEnv, entry.getKey(), entry.getValue());
    }
    // <2> 如果处理完成
    if (roundEnv.processingOver()) {
        try {
            // <2.1> 将 Properties 写入 `META-INF/spring-autoconfigure-metadata.properties` 文件
            writeProperties();
        }
        catch (Exception ex) {
            throw new IllegalStateException("Failed to write metadata", ex);
        }
    }
    // <3> 返回 `false`
    return false;
}

```

#### process 重载方法
```java
private void process(RoundEnvironment roundEnv, String propertyKey, String annotationName) {
    // <1> 获取到这个注解名称对应的 Java 类型
    TypeElement annotationType = this.processingEnv.getElementUtils().getTypeElement(annotationName);
    if (annotationType != null) {
        // <2> 如果存在该注解，则从 RoundEnvironment 中获取标注了该注解的所有 Element 元素，进行遍历
        for (Element element : roundEnv.getElementsAnnotatedWith(annotationType)) {
            // <2.1> 获取这个 Element 元素 innermost 最深处的 Element
            Element enclosingElement = element.getEnclosingElement();
            // <2.2> 如果最深处的 Element 的类型是 PACKAGE 包，那么表示这个元素是一个类，则进行处理
            if (enclosingElement != null && enclosingElement.getKind() == ElementKind.PACKAGE) {
                // <2.2.1> 解析这个类上面 `annotationName` 注解的信息，并保存至 `properties` 中
                processElement(element, propertyKey, annotationName);
            }
        }
    }
}
```

#### processElement 方法
```java
private void processElement(Element element, String propertyKey, String annotationName) {
    try {
        // <1> 获取这个类的名称
        String qualifiedName = Elements.getQualifiedName(element);
        // <2> 获取这个类上面的 `annotationName` 类型的注解信息
        AnnotationMirror annotation = getAnnotation(element, annotationName);
        if (qualifiedName != null && annotation != null) {
            // <3> 获取这个注解中的值
            List<Object> values = getValues(propertyKey, annotation);
            // <4> 往 `properties` 中添加 `类名.注解简称` => `注解中的值(逗号分隔)`
            this.properties.put(qualifiedName + "." + propertyKey, toCommaDelimitedString(values));
            // <5> 往 `properties` 中添加 `类名` => `空字符串`
            this.properties.put(qualifiedName, "");
        }
    }
    catch (Exception ex) {
        throw new IllegalStateException("Error processing configuration meta-data on " + element, ex);
    }
}

```

### 小结
@EnableAutoConfiguration 注解的实现原理并不复杂，借助于 @Import 注解，从所有 META-INF/spring.factories 文件中找到 org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的值。
会将这些自动配置类作为一个 Bean 尝试注入到 Spring IoC 容器中，注入的时候 Spring 会通过 @Conditional 注解判断是否符合条件，因为并不是所有的自动配置类都满足条件。当然，Spring Boot 对 @Conditional 注解进行了扩展，例如 @ConditionalOnClass 可以指定必须存在哪些 Class 对象才注入这个 Bean。
Spring Boot 会借助 spring-boot-autoconfigure-processor 工具模块在编译阶段将自己的自动配置类的注解元信息保存至一个 properties 文件中，避免每次启动应用都要去解析这么多自动配置类上面的注解。同时会通过几个过滤器根据这个 properties 文件过滤掉一些自动配置类。

## Condition接口扩展
### Condition 演进史-先从Profile看起
在 Spring 3.1 的版本，为了满足不同环境注册不同的 Bean ，引入了 @Profile 注解。例如：
```java
@Configuration
public class DataSourceConfiguration {

    @Bean
    @Profile("DEV")
    public DataSource devDataSource() {
        // ... 单机 MySQL
    }

    @Bean
    @Profile("PROD")
    public DataSource prodDataSource() {
        // ... 集群 MySQL
    }
}

```
Spring 3.1.x 的 @Profile 注解如下：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Profile {

  /**
   * The set of profiles for which this component should be registered.
   */
  String[] value();
}

```
最开始 @Profile 注解并没有结合 @Conditional 注解一起使用，而是在后续版本才引入的

### Condition 的出现
在 Spring 4.0 的版本，出现了 Condition 功能，体现在 org.springframework.context.annotation.Condition 接口，如下：
```java
@FunctionalInterface
public interface Condition {

  /**
   * Determine if the condition matches.
   * @param context the condition context
   * @param metadata the metadata of the {@link org.springframework.core.type.AnnotationMetadata class}
   * or {@link org.springframework.core.type.MethodMetadata method} being checked
   * @return {@code true} if the condition matches and the component can be registered,
   * or {@code false} to veto the annotated component's registration
   */
  boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
函数式接口，**只有一个 matches(..) 方法，判断是否匹配**，从入参中可以知道，它是和注解配合一起实现 Condition 功能的，也就是 @Conditional 注解，如下：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

  /**
   * All {@link Condition} classes that must {@linkplain Condition#matches match}
   * in order for the component to be registered.
   */
  Class<? extends Condition>[] value();
}
```
随之 @Profile 注解也进行了修改，和 @Conditional 注解配合使用
Spring 5.1.x 的 @Profile 注解如下：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {

  /**
   * The set of profiles for which the annotated component should be registered.
   */
  String[] value();
}
```
这里指定的的 Condition 实现类是 ProfileCondition，如下：
```java
class ProfileCondition implements Condition {

  @Override
  public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
    if (attrs != null) {
      for (Object value : attrs.get("value")) {
        if (context.getEnvironment().acceptsProfiles(Profiles.of((String[]) value))) {
          return true;
        }
      }
      return false;
    }
    return true;
  }
}
```
逻辑很简单，**从当前 Spring 应用上下文的 Environment 中判断 @Profile 指定的环境是否被激活，被激活了表示匹配成功，则注入对应的 Bean，否则，不进行操作**
但是 Spring 本身提供的 Condition 实现类不多，只有一个 ProfileCondition 对象

### SpringBootCondition 的完善
Spring Boot 为了满足更加丰富的 Condition 场景，对 Spring 的 Condition 接口进行了扩展，提供更多的实现类，如下：
![SpringBootCondition](f52b8ac8/SpringBootCondition.jpg)
上面仅列出了部分 SpringBootCondition 的子类，同时这些子类与对应的注解配置一起使用
- @ConditionalOnClass：必须都存在指定的 Class 对象们
- @ConditionalOnMissingClass：指定的 Class 对象们必须都不存在
- @ConditionalOnBean：必须都存在指定的 Bean 们
- @ConditionalOnMissingBean：指定的 Bean 们必须都不存在
- @ConditionalOnSingleCandidate：必须存在指定的 Bean
- @ConditionalOnProperty：指定的属性是否有指定的值
- @ConditionalOnWebApplication：当前的 WEB 应用类型是否在指定的范围内（ANY、SERVLET、REACTIVE）
- @ConditionalOnNotWebApplication：不是 WEB 应用类型

上面列出了 Spring Boot 中常见的几种 @ConditionXxx 注解，**他们都配合 @Conditional 注解与对应的 Condition 实现类一起使用，提供了非常丰富的 Condition 场景**

### Condition 在哪生效？
- 通过 @Component 注解（及派生注解）标注的 Bean
- @Configuration 标注的配置类中的 @Bean 标注的方法 Bean

#### 普通 Bean
第一种情况是在 **Spring 扫描指定路径下的 .class 文件解析成对应的 BeanDefinition（Bean 的前身）时，会根据 @Conditional 注解判断是否符合条件**，如下：
```java
// ClassPathBeanDefinitionScanner.java
public int scan(String... basePackages) {
    // <1> 获取扫描前的 BeanDefinition 数量
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    // <2> 进行扫描，将过滤出来的所有的 .class 文件生成对应的 BeanDefinition 并注册
    doScan(basePackages);

    // Register annotation config processors, if necessary.
    // <3> 如果 `includeAnnotationConfig` 为 `true`（默认），则注册几个关于注解的 PostProcessor 处理器（关键）
    // 在其他地方也会注册，内部会进行判断，已注册的处理器不会再注册
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    // <4> 返回本次扫描注册的 BeanDefinition 数量
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
// ClassPathScanningCandidateComponentProvider.java
private boolean isConditionMatch(MetadataReader metadataReader) {
    if (this.conditionEvaluator == null) {
        this.conditionEvaluator =
                new ConditionEvaluator(getRegistry(), this.environment, this.resourcePatternResolver);
    }
    return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
}
```

#### 配置类
第二种情况是 Spring 会对 配置类进行处理，扫描到带有 @Bean 注解的方法，尝试解析成 BeanDefinition（Bean 的前身）时，会根据 @Conditional 注解判断是否符合条件，如下：
```java
// ConfigurationClassBeanDefinitionReader.java
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    // <1> 如果不符合 @Conditional 注解的条件，则跳过
    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    // <2> 如果当前 ConfigurationClass 是通过 @Import 注解被导入的
    if (configClass.isImported()) {
        // <2.1> 根据该 ConfigurationClass 对象生成一个 BeanDefinition 并注册
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    // <3> 遍历当前 ConfigurationClass 中所有的 @Bean 注解标注的方法
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        // <3.1> 根据该 BeanMethod 对象生成一个 BeanDefinition 并注册（注意这里有无 static 修饰会有不同的配置）
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    // <4> 对 @ImportResource 注解配置的资源进行处理，对里面的配置进行解析并注册 BeanDefinition
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    // <5> 通过 @Import 注解导入的 ImportBeanDefinitionRegistrar 实现类往 BeanDefinitionRegistry 注册 BeanDefinition
    // Mybatis 集成 Spring 就是基于这个实现的，可查看 Mybatis-Spring 项目中的 MapperScannerRegistrar 这个类
    // https://github.com/liu844869663/mybatis-spring/blob/master/src/main/java/org/mybatis/spring/annotation/MapperScannerRegistrar.java
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}

```
上面只是简单的提一下，可以看到会通过 TrackedConditionEvaluator 计算器进行计算，判断是否满足条件
这里提一下，**对于 @Bean 标注的方法，会得到 CGLIB 的提升，也就是返回的是一个代理对象**，设置一个拦截器专门对 @Bean 方法进行拦截处理，通过依赖查找的方式从 IoC 容器中获取 Bean 对象，如果是单例 Bean，那么每次都是返回同一个对象，所以当主动调用这个方法时获取到的都是同一个对象。



## 配置加载

## 日期组件

## @ConfigurationProperties注解实现