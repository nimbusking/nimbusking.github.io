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

## 先从运行Jar包开始
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



## Condition接口扩展

## 配置加载

## 日期组件

## @ConfigurationProperties注解实现