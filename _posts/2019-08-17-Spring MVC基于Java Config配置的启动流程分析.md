---
layout:     post
title:      Spring MVC基于Java Config配置的启动流程分析
subtitle:   ServletContainerInitializer/WebApplicationInitializer
date:       2019-08-17
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring MVC
---

# Spring MVC基于Java Config配置的启动流程分析

本文将分析Spring MVC基于Java Config配置的启动流程，由于基于Java Config配置的核心启动流程与基于XML配置的启动流程类似，所以阅读本文前请先阅读[Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)、[Spring MVC DispatcherServlet处理用户请求的流程分析](https://xuanjian1992.top/2019/08/14/Spring-MVC-DispatcherServlet%E5%A4%84%E7%90%86%E7%94%A8%E6%88%B7%E8%AF%B7%E6%B1%82%E7%9A%84%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)、[Spring MVC之Servlet2.x与Servlet3.x的区别](https://xuanjian1992.top/2019/08/16/Spring-MVC%E4%B9%8BServlet2.x%E4%B8%8EServlet3.x%E7%9A%84%E5%8C%BA%E5%88%AB/)这三篇文章。下面首先给出Java Config配置的Spring MVC Web应用的例子。

### 一、Spring MVC Java Config配置示例

##### 1. 定义AbstractAnnotationConfigDispatcherServletInitializer实现

```java
// 基于Java Config的Spring MVC初始化器(WebApplicationInitializer实现)
public class MySpringMvcInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class}; // build ROOT WebApplicationContext
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class}; // build DispatcherServlet WebApplicationContext
    }

    @Override
    protected String[] getServletMappings() { // DispatcherServlet Mapping
        return new String[]{"/"};
    }
}
```

##### 2. WebConfig(DispatcherServlet WebApplicationContext)

```java
// DispatcherServlet对应的上下文的Java Config配置
@Configuration
@ComponentScan("com.jay.anno.web")
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Bean
    public ViewResolver viewResolver() { // 自定义ViewResolver
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("/WEB-INF/views");
        viewResolver.setSuffix(".jsp");
        viewResolver.setExposeContextBeansAsAttributes(true);
        return viewResolver;
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable(); // 静态资源使用DefaultServlet处理
    }
}
// 示例Controller
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String sayHello() {
        return "hello";
    }
}
```

##### 3. RootConfig(ROOT WebApplicationContext)

```java
// ROOT应用上下文Java Config配置
@Configuration
@ComponentScan(basePackages = "com.jay.anno.root",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)})
public class RootConfig {

}
```

以上三个类构成了Spring MVC基于Java Config的核心配置。这种配置下，无须再配置web.xml文件。Web容器(如Tomcat，需支持Servlet3.0，依赖于Servlet3.0的特性)能自动发现MySpringMvcInitializer类(WebApplicationInitializer实现)并进行相关初始化流程。

### 二、启动流程分析

从[Spring MVC之Servlet2.x与Servlet3.x的区别](https://xuanjian1992.top/2019/08/16/Spring-MVC%E4%B9%8BServlet2.x%E4%B8%8EServlet3.x%E7%9A%84%E5%8C%BA%E5%88%AB/)这篇文章可知，**ServletContainerInitializer**接口(**Servlet 3.0**)主要由框架(如Spring MVC)实现，在应用程序启动时由Web容器调用其onStartup()方法，负责框架的初始化动作，如初始化控制器Servlet(对应Spring MVC，DispatcherServlet)、连接数据库等。而Web容器对于ServletContainerInitializer实现的查找，是通过Java SPI机制实现的。首先看下Spring MVC对ServletContainerInitializer接口的实现——SpringServletContainerInitializer的配置：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-16.jpg)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-17.jpg)

因此，Web应用在依赖了Spring MVC相关依赖后，并部署到Tomcat等Web容器(**Servlet 3.0**)并启动时，会由容器调用SpringServletContainerInitializer.onStartup()方法。SpringServletContainerInitializer类如下：

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
   @Override
   public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext) throws ServletException {
     // 省略...
   }
}
```

基于Servlet 3.0的特性，SpringServletContainerInitializer实现了ServletContainerInitializer接口，同时注解了@HandlesTypes(WebApplicationInitializer.class)，表示该SpringServletContainerInitializer只关心类路径下WebApplicationInitializer接口相关的实现类。因此Web容器在回调SpringServletContainerInitializer.onStartup方法时会将类路径下匹配的WebApplicationInitializer接口相关实现类传入onStartup方法中。此时再来看第一部分示例中的MySpringMvcInitializer类，这个类实现了WebApplicationInitializer接口，因此在应用启动时会将该类传入SpringServletContainerInitializer.onStartup方法。下面分析下SpringServletContainerInitializer的实现：

```java
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
            if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
                  WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
               try {
                  initializers.add((WebApplicationInitializer) // 实例化WebApplicationInitializer
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
      AnnotationAwareOrderComparator.sort(initializers); // 排序
      for (WebApplicationInitializer initializer : initializers) {
         initializer.onStartup(servletContext); // 回调WebApplicationInitializer.onStartup方法
      }
   }

}
```

SpringServletContainerInitializer.onStartup()方法的主要逻辑是根据传入的WebApplicationInitializer实现，在过滤掉接口、抽象类与非WebApplicationInitializer实现之后，实例化所有符合要求的WebApplicationInitializer实现。然后对WebApplicationInitializer实现进行排序，接着依次调用WebApplicationInitializer.onStartup方法。

##### 1.注册ContextLoaderListener并初始化ROOT Web应用上下文

根据上面的分析，应用启动过程中，最终会调用到MySpringMvcInitializer类的onStartup方法。先看下MySpringMvcInitializer类的类继承图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-18.jpg?x-oss-process=style/markdown-pic)

因此，在调用MySpringMvcInitializer.onStartup方法时，会调用到父类AbstractDispatcherServletInitializer.onStratup方法：

```java
// AbstractDispatcherServletInitializer类
public void onStartup(ServletContext servletContext) throws ServletException {
   super.onStartup(servletContext);
   registerDispatcherServlet(servletContext);
}
```

AbstractDispatcherServletInitializer.onStratup方法首先调用父类AbstractContextLoaderInitializer类的onStartup方法。

```java
// AbstractContextLoaderInitializer类
public void onStartup(ServletContext servletContext) throws ServletException {
   registerContextLoaderListener(servletContext);
}
```

然后AbstractContextLoaderInitializer.onStartup方法调用其registerContextLoaderListener方法向ServletContext添加监听器ContextLoaderListener。

```java
// AbstractContextLoaderInitializer类
protected void c(ServletContext servletContext) {
   WebApplicationContext rootAppContext = createRootApplicationContext();
   if (rootAppContext != null) {
      ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
      listener.setContextInitializers(getRootApplicationContextInitializers());
      servletContext.addListener(listener);
   }
   else {
      logger.debug("No ContextLoaderListener registered, as " +
            "createRootApplicationContext() did not return an application context");
   }
}
```

registerContextLoaderListener方法中首先调用createRootApplicationContext方法创建ROOT WebApplicationContext：

```java
// AbstractAnnotationConfigDispatcherServletInitializer类
protected WebApplicationContext createRootApplicationContext() {
   Class<?>[] configClasses = getRootConfigClasses();
   if (!ObjectUtils.isEmpty(configClasses)) {
      AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
      context.register(configClasses);
      return context;
   }
   else {
      return null;
   }
}
// MySpringMvcInitializer类
protected Class<?>[] getRootConfigClasses() {
    return new Class[]{RootConfig.class}; // ROOT WebApplicationContext
}
```

createRootApplicationContext方法由子类AbstractAnnotationConfigDispatcherServletInitializer实现，AbstractAnnotationConfigDispatcherServletInitializer.createRootApplicationContext方法最终获取到RootConfig配置类，并创建AnnotationConfigWebApplicationContext实例。

AbstractContextLoaderInitializer.registerContextLoaderListener方法中，在创建了ROOT WebApplicationContext之后，接着实例化ContextLoaderListener，并添加到ServletContext中。由于ContextLoaderListener实现了ServletContextListener接口，因此应用启动过程中，容器会回调ContextLoaderListener.contextInitialized方法，进行ROOT WebApplicationContext的初始化。该过程与Spring MVC XML配置下的ROOT WebApplicationContext初始化流程类似，可参考[Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)。

##### 2.注册DispatcherServlet并初始化Web应用上下文

回到AbstractDispatcherServletInitializer.onStartup方法：

```java
// AbstractDispatcherServletInitializer类
public void onStartup(ServletContext servletContext) throws ServletException {
   super.onStartup(servletContext);
   registerDispatcherServlet(servletContext);
}
```

在调用父类AbstractContextLoaderInitializer类的onStartup方法向ServletContext添加ContextLoaderListener实例、创建并初始化ROOT WebApplicationContext之后，会调用registerDispatcherServlet方法向ServletContext添加DispatcherServlet实例。

```java
// AbstractDispatcherServletInitializer类
protected void registerDispatcherServlet(ServletContext servletContext) {
   String servletName = getServletName();
   Assert.hasLength(servletName, "getServletName() must not return null or empty");

   WebApplicationContext servletAppContext = createServletApplicationContext();
   Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

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

   Filter[] filters = getServletFilters();
   if (!ObjectUtils.isEmpty(filters)) {
      for (Filter filter : filters) {
         registerServletFilter(servletContext, filter);
      }
   }

   customizeRegistration(registration);
}
```

registerDispatcherServlet方法首先调用createServletApplicationContext方法创建DispatcherServlet对应的WebApplicationContext，该方法由子类AbstractAnnotationConfigDispatcherServletInitializer实现：

```java
// AbstractAnnotationConfigDispatcherServletInitializer
protected WebApplicationContext createServletApplicationContext() {
   AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext(); // 实例化web应用上下文
   Class<?>[] configClasses = getServletConfigClasses();
   if (!ObjectUtils.isEmpty(configClasses)) {
      context.register(configClasses);
   }
   return context;
}
// MySpringMvcInitializer类
protected Class<?>[] getServletConfigClasses() {
    return new Class[]{WebConfig.class}; // DispatcherServlet WebApplicationContext
}
```

registerDispatcherServlet方法然后创建DispatcherServlet实例并注册到ServletContext：

```java
FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());
ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);

registration.setLoadOnStartup(1); // 应用启动时初始化
registration.addMapping(getServletMappings());  // 设置mapping
registration.setAsyncSupported(isAsyncSupported()); // 异步设置
```

**由于这里的DispatcherServlet配置为容器启动时直接初始化，因此该DispatcherServlet会经历XML配置的DispatcherServlet相同的初始化流程**，可参考[Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)。

registerDispatcherServlet方法接着向ServletContext注册Filter:

```java
Filter[] filters = getServletFilters()
if (!ObjectUtils.isEmpty(filters)) {
   for (Filter filter : filters) {
      registerServletFilter(servletContext, filter);
   }
}

protected FilterRegistration.Dynamic registerServletFilter(ServletContext servletContext, Filter filter) {
	String filterName = Conventions.getVariableName(filter);
	Dynamic registration = servletContext.addFilter(filterName, filter);

	if (registration == null) {
		int counter = 0;
		while (registration == null) {
			if (counter == 100) {
				throw new IllegalStateException("Failed to register filter with name '" + filterName + "'. " +
						"Check if there is another filter registered under the same name.");
			}
			registration = servletContext.addFilter(filterName + "#" + counter, filter);
			counter++;
		}
	}

	registration.setAsyncSupported(isAsyncSupported());
	registration.addMappingForServletNames(getDispatcherTypes(), false, getServletName());
	return registration;
}

private EnumSet<DispatcherType> getDispatcherTypes() {
	return (isAsyncSupported() ?
			EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD, DispatcherType.INCLUDE, DispatcherType.ASYNC) :
			EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD, DispatcherType.INCLUDE));
}
```

### 三、总结

至此，在应用启动过程中，向web容器ServletContext添加ContextLoaderListener、DispactherServlet、Filter的流程分析结束。可以看到，以上分析的内容基本与在web.xml中配置ContextLoaderListener、DispactherServlet、Filter的作用类似。只不过在Spring MVC Java Config下，利用了ServletContainerInitializer接口和WebApplicationInitializer接口完全去掉了web.xml和Spring XML配置，基于代码的方式配置Spring MVC，初始化ROOT和DispatcherServlet对应的WebApplicationContext。

### 参考文章

- [Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)
- [Spring MVC DispatcherServlet处理用户请求的流程分析](https://xuanjian1992.top/2019/08/14/Spring-MVC-DispatcherServlet%E5%A4%84%E7%90%86%E7%94%A8%E6%88%B7%E8%AF%B7%E6%B1%82%E7%9A%84%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)

- [Spring MVC之Servlet2.x与Servlet3.x的区别](https://xuanjian1992.top/2019/08/16/Spring-MVC%E4%B9%8BServlet2.x%E4%B8%8EServlet3.x%E7%9A%84%E5%8C%BA%E5%88%AB/)
- Spring In Action(第四版)
- [AbstractAnnotationConfigDispatcherServletInitializer 无效的解决方案](https://blog.csdn.net/l1161558158/article/details/85016464)