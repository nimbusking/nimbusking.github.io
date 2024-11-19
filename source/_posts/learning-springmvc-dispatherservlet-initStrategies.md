---
title: DispatcherServlet九大组件介绍  
abbrlink: b18da5cf
date: 2024-11-18 16:31:34
updated: 2024-11-18 16:31:34
tags:
  - SpringMVC
  - DispatcherServlet
categories: Spring
---

# HandlerMapping组件
HandlerMapping 组件，请求的处理器匹配器，**负责为请求找到合适的 HandlerExecutionChain 处理器执行链，包含处理器（handler）和拦截器们（interceptors）**
- `handler` 处理器是 Object 类型，可以将其理解成 HandlerMethod 对象（例如我们使用最多的 @RequestMapping 注解所标注的方法会解析成该对象），包含了方法的所有信息，通过该对象能够执行该方法
- `HandlerInterceptor` 拦截器对处理请求进行增强处理，可用于在执行方法前、成功执行方法后、处理完成后进行一些逻辑处理
<!-- more -->
## HandlerMapping 组件
### 回顾一下doDispatch中的调用
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    // ... 省略相关代码
    // Determine handler for the current request.
    // <3> 获得请求对应的 HandlerExecutionChain 对象（HandlerMethod 和 HandlerInterceptor 拦截器们）
    mappedHandler = getHandler(processedRequest);
    if (mappedHandler == null) { // <3.1> 如果获取不到，则根据配置抛出异常或返回 404 错误
        noHandlerFound(processedRequest, response);
        return;
    }
    // ... 省略相关代码
}

@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历 handlerMappings 组件们
        for (HandlerMapping mapping : this.handlerMappings) {
            // 通过 HandlerMapping 组件获取到 HandlerExecutionChain 对象
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                // 不为空则直接返回
                return handler;
            }
        }
    }
    return null;
}
```
通过遍历 HandlerMapping 组件们，根据请求获取到对应 HandlerExecutionChain 处理器执行链。
**注意**，这里是通过一个一个的 HandlerMapping 组件去进行处理，如果找到对应 HandlerExecutionChain 对象则直接返回，不会继续下去，**所以初始化的 HandlerMapping 组件是有一定的先后顺序的，默认是BeanNameUrlHandlerMapping -> RequestMappingHandlerMapping**

### HandlerMapping 接口
org.springframework.web.servlet.HandlerMapping 接口，请求的处理器匹配器，负责为请求找到合适的 HandlerExecutionChain 处理器执行链，包含处理器（handler）和拦截器们（interceptors），代码如下：
```java
public interface HandlerMapping {

  String BEST_MATCHING_HANDLER_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingHandler";

  String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";

  String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";

  String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";

  String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";

  String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";

  String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";

  /**
   * 获得请求对应的处理器和拦截器们
   */
  @Nullable
  HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}

```
先来看一下继承图
![HandlerMapping继承图](b18da5cf/HandlerMapping.jpg)
其中：
- 蓝色框 AbstractHandlerMapping 抽象类，实现了 **“为请求找到合适的 HandlerExecutionChain 处理器执行链”** 对应的的骨架逻辑，而暴露 `getHandlerInternal(HttpServletRequest request)` 抽象方法，交由子类实现
- AbstractHandlerMapping 的子类，分成两派，分别是
  - 橙色框 AbstractUrlHandlerMapping 系，基于 URL 进行匹配。例如基于xml配置的。当然，目前这种方式已经基本不用了，被 @RequestMapping 等注解的方式所取代。不过，Spring MVC 内置的一些路径匹配，还是使用这种方式。
  - 红色框 AbstractHandlerMethodMapping 系，**基于 Method 进行匹配**。例如，我们所熟知的 `@RequestMapping` 等注解的方式。
- 绿色框的 `MatchableHandlerMapping` 接口，定义了 **“判断请求和指定 pattern 路径是否匹配”** 的方法。

#### 初始化过程
在 DispatcherServlet 的 **initHandlerMappings(ApplicationContext context)** 方法，会在 onRefresh 方法被调用，初始化 HandlerMapping 组件，方法如下：
```java
private void initHandlerMappings(ApplicationContext context) {
    // 置空 handlerMappings
    this.handlerMappings = null;

    // <1> 如果开启探测功能，则扫描已注册的 HandlerMapping 的 Bean 们，添加到 handlerMappings 中
    if (this.detectAllHandlerMappings) {
        // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
        // 扫描已注册的 HandlerMapping 的 Bean 们
        Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context,
                HandlerMapping.class, true, false);
        // 添加到 handlerMappings 中，并进行排序
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // We keep HandlerMappings in sorted order.
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    }
    // <2> 如果关闭探测功能，则获得 Bean 名称为 "handlerMapping" 对应的 Bean ，将其添加至 handlerMappings
    else {
        try {
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default HandlerMapping later.
        }
    }
    // Ensure we have at least one HandlerMapping, by registering
    // a default HandlerMapping if no other mappings are found.
    /**
     * <3> 如果未获得到，则获得默认配置的 HandlerMapping 类
     * {@link org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping}
     * {@link org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping}
     */
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}

```
- 如果“开启”探测功能，则扫描已注册的 HandlerMapping 的 Bean 们，添加到 handlerMappings 中，**默认开启**
- 如果“关闭”探测功能，则获得 Bean 名称为 "handlerMapping" 对应的 Bean ，将其添加至 handlerMappings
- 如果未获得到，则获得默认配置的 HandlerMapping 类，调用 **getDefaultStrategies()** 方法，就是从 **DispatcherServlet.properties** 文件中读取 HandlerMapping 的默认实现类，如下：
  ```properties
  org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
  org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
  ```
  可以看出，默认配置的是：BeanNameUrlHandlerMapping 和 RequestMappingHandlerMapping 对象（初始化顺序在这里体现的）

#### AbstractHandlerMapping
org.springframework.web.servlet.handler.AbstractHandlerMapping，实现 HandlerMapping、Ordered、BeanNameAware 接口，继承 WebApplicationObjectSupport 抽象类
该类是 HandlerMapping 接口的抽象基类，实现了“为请求找到合适的 HandlerExecutionChain 处理器执行链”对应的的骨架逻辑，而暴露 getHandlerInternal(HttpServletRequest request) 

#### MatchableHandlerMapping
org.springframework.web.servlet.handler.MatchableHandlerMapping，定义了“判断请求和指定 pattern 路径是否匹配”的方法。代码如下：
```java
public interface MatchableHandlerMapping extends HandlerMapping {

  /**
   * 判断请求和指定 pattern 路径是否匹配
   */
  @Nullable
  RequestMatchResult match(HttpServletRequest request, String pattern);
}
```

目前实现 MatchableHandlerMapping 接口的类，有 RequestMappingHandlerMapping 类和 AbstractUrlHandlerMapping 抽象类，在后续都会进行分析

### HandlerInterceptor拦截器
org.springframework.web.servlet.HandlerInterceptor，处理器拦截器接口，代码如下：
```java
public interface HandlerInterceptor {
  /**
   * 前置处理，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行之前
   */
  default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
      throws Exception {
    return true;
  }

  /**
   * 后置处理，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行成功之后
   */
  default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
      @Nullable ModelAndView modelAndView) throws Exception {
  }

  /**
   * 完成处理，在 {@link HandlerAdapter#handle(HttpServletRequest, HttpServletResponse, Object)} 执行之后（无论成功还是失败）
   * 条件：执行 {@link #preHandle(HttpServletRequest, HttpServletResponse, Object)} 成功的拦截器才会执行该方法
   */
  default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
      @Nullable Exception ex) throws Exception {
  }
}

```
#### HandlerExecutionChain
org.springframework.web.servlet.HandlerExecutionChain，处理器执行链，也**就是通过 HandlerMapping 组件为请求找到的处理对象**，包含处理器（handler）和拦截器们（interceptors）
```java
public class HandlerExecutionChain {
  /**
   * 处理器
   */
  private final Object handler;

  /**
   * 拦截器数组
   */
  @Nullable
  private HandlerInterceptor[] interceptors;

  /**
   * 拦截器数组。
   *
   * 在实际使用时，会调用 {@link #getInterceptors()} 方法，初始化到 {@link #interceptors} 中
   */
  @Nullable
  private List<HandlerInterceptor> interceptorList;

  /**
   * 已成功执行 {@link HandlerInterceptor#preHandle(HttpServletRequest, HttpServletResponse, Object)} 的位置
   *
   * 在 {@link #applyPostHandle} 和 {@link #triggerAfterCompletion} 方法中需要用到，用于倒序执行拦截器的方法
   */
  private int interceptorIndex = -1;

  public HandlerExecutionChain(Object handler) {
    this(handler, (HandlerInterceptor[]) null);
  }

  public HandlerExecutionChain(Object handler, @Nullable HandlerInterceptor... interceptors) {
    if (handler instanceof HandlerExecutionChain) {
      HandlerExecutionChain originalChain = (HandlerExecutionChain) handler;
      this.handler = originalChain.getHandler();
      this.interceptorList = new ArrayList<>();
      // 将原始的 HandlerExecutionChain 的 interceptors 复制到 this.interceptorList 中
      CollectionUtils.mergeArrayIntoCollection(originalChain.getInterceptors(), this.interceptorList);
      // 将入参的 interceptors 合并到 this.interceptorList 中
      CollectionUtils.mergeArrayIntoCollection(interceptors, this.interceptorList);
    } else {
      this.handler = handler;
      this.interceptors = interceptors;
    }
  }
}
```

- handler：请求对应的处理器对象，可以先理解为 HandlerMethod 对象（例如我们常用的 @RequestMapping 注解对应的方法会解析成该对象），也就是我们的某个 Method 的所有信息，可以被执行
- interceptors：请求匹配的拦截器数组
- interceptorList：请求匹配的拦截器集合，这个属性似乎跟上面一个数组属性是重复的，这里笔者只能猜测可能要实现的逻辑：interceptorList是内部持有的，并没有直接暴露给外部；而interceptors通过getInterceptors方法可以暴露给外部。通过interceptorList向数组array转换的时候，提供了一些便捷？这部分代码是上古的代码，从03年就存在了🤪
- interceptorIndex：记录已成功执行前置处理的拦截器位置，因为已完成处理只会执行前置处理成功的拦截器，且倒序执行

#### HandlerInterceptor 的实现类
看一下继承图：
![HandlerInterceptor继承图](b18da5cf/HandlerInterceptor.png)
这里面提几个重要的类：
##### MappedInterceptor
org.springframework.web.servlet.handler.MappedInterceptor，实现 HandlerInterceptor 接口，支持地址匹配的 HandlerInterceptor 实现类
每一个 **`<mvc:interceptor />`** 标签，将被解析成一个 MappedInterceptor 类型的 Bean 拦截器对象
这里遇到了第一个**mvc:**打头的标签，结合拦截器的示例，我们先看看一般怎么使用的：
```xml
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**" />
        <mvc:exclude-mapping path="/error/**" />
        <bean class="cc.nimbusk.javaweb.interceptor.JwtInterceptor" />
    </mvc:interceptor>
</mvc:interceptors>

```
- 每一个 `<mvc:interceptor />` 标签，将被解析成一个 MappedInterceptor 类型的 Bean 拦截器对象
- 然后 MappedInterceptor 类型的拦截器在 AbstractHandlerMapping 的 `initApplicationContext() -> detectMappedInterceptors` 会被扫描到
  ```java
    protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
      // 扫描已注册的 MappedInterceptor 的 Bean 们，添加到 mappedInterceptors 中
      // MappedInterceptor 会根据请求路径做匹配，是否进行拦截
      mappedInterceptors.addAll(BeanFactoryUtils
              .beansOfTypeIncludingAncestors(obtainApplicationContext(), MappedInterceptor.class, true, false)
              .values());
  }
  ```
  **也就是说在初始化 HandlerMapping 组件的时候会扫描到我们自定义的拦截器，并添加到属性中**。

#### 如何配置interceptor
**这里有个问题：`<mvc:interceptor />` 标签如何被解析成MappedInterceptor对象的？**
可以打开spring-webmvc子工程目录下的spring-webmvc/src/main/resources/META-INF/spring.handlers，内容如下：
```properties
http\://www.springframework.org/schema/mvc=org.springframework.web.servlet.config.MvcNamespaceHandler
```
**指定了 NamespaceHandler 为 MvcNamespaceHandler 对象，也就是说`<mvc />`标签会被该对象进行解析**，如下：
```java
public class MvcNamespaceHandler extends NamespaceHandlerSupport {
  @Override
  public void init() {
    registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
    registerBeanDefinitionParser("default-servlet-handler", new DefaultServletHandlerBeanDefinitionParser());
    registerBeanDefinitionParser("interceptors", new InterceptorsBeanDefinitionParser());
    registerBeanDefinitionParser("resources", new ResourcesBeanDefinitionParser());
    registerBeanDefinitionParser("view-controller", new ViewControllerBeanDefinitionParser());
    registerBeanDefinitionParser("redirect-view-controller", new ViewControllerBeanDefinitionParser());
    registerBeanDefinitionParser("status-controller", new ViewControllerBeanDefinitionParser());
    registerBeanDefinitionParser("view-resolvers", new ViewResolversBeanDefinitionParser());
    registerBeanDefinitionParser("tiles-configurer", new TilesConfigurerBeanDefinitionParser());
    registerBeanDefinitionParser("freemarker-configurer", new FreeMarkerConfigurerBeanDefinitionParser());
    registerBeanDefinitionParser("groovy-configurer", new GroovyMarkupConfigurerBeanDefinitionParser());
    registerBeanDefinitionParser("script-template-configurer", new ScriptTemplateConfigurerBeanDefinitionParser());
    registerBeanDefinitionParser("cors", new CorsBeanDefinitionParser());
  }
}
```
其中`<mvc:interceptor />`标签则会被 InterceptorsBeanDefinitionParser 对象进行解析，如下：
```java
class InterceptorsBeanDefinitionParser implements BeanDefinitionParser {
  @Override
  @Nullable
  public BeanDefinition parse(Element element, ParserContext context) {
    context.pushContainingComponent(
        new CompositeComponentDefinition(element.getTagName(), context.extractSource(element)));

    RuntimeBeanReference pathMatcherRef = null;
    if (element.hasAttribute("path-matcher")) {
      pathMatcherRef = new RuntimeBeanReference(element.getAttribute("path-matcher"));
    }

    List<Element> interceptors = DomUtils.getChildElementsByTagName(element, "bean", "ref", "interceptor");
    for (Element interceptor : interceptors) {
      // 将 <mvc:interceptor /> 标签解析 MappedInterceptor 对象
      RootBeanDefinition mappedInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
      mappedInterceptorDef.setSource(context.extractSource(interceptor));
      mappedInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

      ManagedList<String> includePatterns = null;
      ManagedList<String> excludePatterns = null;
      Object interceptorBean;
      if ("interceptor".equals(interceptor.getLocalName())) {
        includePatterns = getIncludePatterns(interceptor, "mapping");
        excludePatterns = getIncludePatterns(interceptor, "exclude-mapping");
        Element beanElem = DomUtils.getChildElementsByTagName(interceptor, "bean", "ref").get(0);
        interceptorBean = context.getDelegate().parsePropertySubElement(beanElem, null);
      }
      else {
        interceptorBean = context.getDelegate().parsePropertySubElement(interceptor, null);
      }
      mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, includePatterns);
      mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, excludePatterns);
      mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(2, interceptorBean);

      if (pathMatcherRef != null) {
        mappedInterceptorDef.getPropertyValues().add("pathMatcher", pathMatcherRef);
      }

      String beanName = context.getReaderContext().registerWithGeneratedName(mappedInterceptorDef);
      context.registerComponent(new BeanComponentDefinition(mappedInterceptorDef, beanName));
    }

    context.popAndRegisterContainingComponent();
    return null;
  }

  private ManagedList<String> getIncludePatterns(Element interceptor, String elementName) {
    List<Element> paths = DomUtils.getChildElementsByTagName(interceptor, elementName);
    ManagedList<String> patterns = new ManagedList<>(paths.size());
    for (Element path : paths) {
      patterns.add(path.getAttribute("path"));
    }
    return patterns;
  }
}
```
逻辑不复杂，会将 `<mvc:interceptor />` 标签解析 BeanDefinition 对象，beanClass 为 MappedInterceptor，解析出来的属性也会添加至其中，也就会给初始化成 MappedInterceptor 类型的 Spring Bean 到 Spring 上下文中

#### JavaConfig
在 SpringBoot 2.0+ 项目中，添加拦截器的方式可以如下：
```java
@Component
public class JwtInterceptor implements HandlerInterceptor {
    /**
     * 前置处理
     *
     * @param handler  拦截的目标，处理器
     * @return 该请求是否继续往下执行
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // JWT 校验
        // 验证通过，返回 true，否则返回false
        return true;
    }
    /** 后置处理 */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) 
        throws Exception {
    }
    /** 已完成处理 */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) 
        throws Exception {
    }
}

@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        List<String> excludePath = new ArrayList<>();
        // 将拦截器添加至 InterceptorRegistry
    registry.addInterceptor(jwtInterceptor()).addPathPatterns("/**").excludePathPatterns(excludePath);
    }

    @Bean
    public JwtInterceptor jwtInterceptor() {
        return new JwtInterceptor();
    }

}

```

### AbstractHandlerMethodMapping
先来回顾一下在 DispatcherServlet 中处理请求的过程中通过 HandlerMapping 组件，获取到 HandlerExecutionChain 处理器执行链的方法，**是通过AbstractHandlerMapping 的 getHandler 方法来获取的**，如下：
```java
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // <1> 获得处理器（HandlerMethod 或者 HandlerExecutionChain），该方法是抽象方法，由子类实现
    Object handler = getHandlerInternal(request);
    // <2> 获得不到，则使用默认处理器
    // <3> 还是获得不到，则返回 null
    // <4> 如果找到的处理器是 String 类型，则从 Spring 容器中找到对应的 Bean 作为处理器
    // <5> 创建 HandlerExecutionChain 对象（包含处理器和拦截器）
    // ... 省略相关代码
    return executionChain;
}

```
在 AbstractHandlerMapping 获取 HandlerExecutionChain 处理器执行链的方法中，**需要先调用 getHandlerInternal(HttpServletRequest request) 抽象方法，获取请求对应的处理器，该方法由子类去实现。**

## HandlerMapping 组件（二）之 HandlerInterceptor 拦截器

## HandlerMapping 组件（三）之 AbstractHandlerMethodMapping

## HandlerMapping 组件（四）之 AbstractUrlHandlerMapping
