---
layout:     post
title:      Spring MVC启动流程分析(xml配置)
subtitle:   ContextLoaderListener/DispatcherServlet
date:       2019-08-13
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
    - Spring MVC
---

# Spring MVC启动流程分析(xml配置)

熟悉`SpringMVC`的启动过程，有助于理解相关文件配置的原理，深入理解`SpringMVC`的设计原理和执行过程。

### 一、Web应用部署初始化过程 (Web Application Deployement)

参考`Oracle`官方文档[Java Servlet Specification](https://link.jianshu.com/?t=http://download.oracle.com/otn-pub/jcp/servlet-3.0-fr-eval-oth-JSpec/servlet-3_0-final-spec.pdf)，可知Web应用部署的相关步骤如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-01.jpeg)

当一个Web应用部署到容器内时(比如Tomcat)，在Web应用开始响应并执行用户请求前，以下步骤会被依次执行:

- 部署描述文件中(比如Tomcat的web.xml)由`<listener>`元素标记的事件监听器会被创建和初始化；

- 对于所有事件监听器，如果实现了`ServletContextListener`接口，将会执行其实现的`contextInitialized()`方法；

- 部署描述文件中由`<filter>`元素标记的过滤器会被创建和初始化，并调用其`init()`方法；

- 部署描述文件中由`<servlet>`元素标记的servlet会根据`<load-on-startup>`的权值按顺序创建和初始化，并调用其`init()`方法。

通过上述官方文档的描述，可绘制如下`Web应用部署初始化`流程执行图。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-02.jpeg)

可以发现，在`Tomcat`下`web应用`的初始化流程是，先初始化`listener`，接着初始化`filter`，最后初始化`servlet`，当清楚认识到`Web应用`部署到容器后的初始化过程后，就可以进一步深入探讨`Spring MVC`的启动过程。

### 二、Spring MVC启动过程

接下来以一个常见的`web.xml`配置开始对`Spring MVC`启动过程的分析，`web.xml`配置内容如下:

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
       version="3.1">

   <display-name>Spring MVC origin</display-name>

   <!-- Web 应用初始化启动时，依次初始化 Listener、Filter、Servlet -->

   <!-- ROOT WebApplicationContext -->
   <context-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/applicationContext.xml</param-value>
   </context-param>
   <listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
   </listener>

   <!-- 解决乱码问题 -->
   <filter>
      <filter-name>CharacterEncodingFilter</filter-name>
      <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      <init-param>
         <param-name>encoding</param-name>
         <param-value>UTF-8</param-value>
      </init-param>
   </filter>
   <filter-mapping>
      <filter-name>CharacterEncodingFilter</filter-name>
      <url-pattern>/*</url-pattern>
   </filter-mapping>

   <!-- DispatcherServlet's Child WebApplicationContext -->
   <servlet>
      <servlet-name>DispatcherServlet</servlet-name>
      <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     <!-- ServletConfig -->
      <init-param>
         <param-name>contextConfigLocation</param-name>
         <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
      </init-param>
      <load-on-startup>1</load-on-startup>
   </servlet>

   <servlet-mapping>
      <servlet-name>DispatcherServlet</servlet-name>
      <url-pattern>/</url-pattern>
   </servlet-mapping>

</web-app>
```

##### 1. Listener的初始化过程

首先定义了`<context-param>`标签，用于配置一个全局变量，`<context-param>`标签的内容读取后会被放进`ServletContext`中，做为Web应用的全局变量使用，接下来创建`listener`时会使用到这个全局变量，因此，Web应用在容器中部署后，进行初始化时会先读取这个全局变量，之后再进行上述讲解的初始化启动过程。

接着定义了一个`ContextLoaderListener类`的`listener`。查看`ContextLoaderListener`的类继承结构，如下图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-03.jpg?x-oss-process=style/markdown-pic)

`ContextLoaderListener`类继承了`ContextLoader`类并实现了`ServletContextListener`接口，首先看一下`ServletContextListener`接口源码:

```java
public interface ServletContextListener extends EventListener {
    public void contextInitialized(ServletContextEvent sce);

    public void contextDestroyed(ServletContextEvent sce);
}
```

该接口只有两个方法`contextInitialized`和`contextDestroyed`，当`Web应用`初始化或销毁时会分别调用这两个方法。

继续看`ContextLoaderListener`，该`listener`实现了`ServletContextListener`接口，因此在`Web应用`初始化时会调用该方法，该方法的具体实现如下：

```java
public void contextInitialized(ServletContextEvent event) {
   // 初始化ROOT web应用上下文
   initWebApplicationContext(event.getServletContext());
}
```

`ContextLoaderListener`的`contextInitialized()`方法直接调用了`initWebApplicationContext()`方法，这个方法是继承自`ContextLoader类`，通过函数名可以知道，该方法是用于初始化Web应用上下文，即`IoC容器`。继续查看`ContextLoader类`的`initWebApplicationContext()`方法的源码如下:

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
   // web.xml中只允许存在一个ContextLoader类或其子类的对象
   if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
      throw new IllegalStateException(
            "Cannot initialize context because there is already a root application context present - " +
            "check whether you have multiple ContextLoader* definitions in your web.xml!");
   }

   servletContext.log("Initializing Spring root WebApplicationContext");
   Log logger = LogFactory.getLog(ContextLoader.class);
   if (logger.isInfoEnabled()) {
      logger.info("Root WebApplicationContext: initialization started");
   }
   long startTime = System.currentTimeMillis();

   try {
      if (this.context == null) {
         // 不存在，则创建ROOT web应用上下文
         this.context = createWebApplicationContext(servletContext); 
      }
      if (this.context instanceof ConfigurableWebApplicationContext) { // true
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
         if (!cwac.isActive()) { // 是否已refresh，且未关闭 false
            if (cwac.getParent() == null) {
               // 加载ROOT web应用上下文的parent
               ApplicationContext parent = loadParentContext(servletContext); 
               cwac.setParent(parent);
            }
            // 配置并刷新整个根IoC容器，在这里会进行Bean的创建和初始化
            configureAndRefreshWebApplicationContext(cwac, servletContext);
         }
      }
      // 将ROOT Web应用上下文引用保存到ServletContext
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

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

      return this.context;
   }
   catch (RuntimeException | Error ex) {
      logger.error("Context initialization failed", ex);
      servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
      throw ex;
   }
}
```

`initWebApplicationContext()`方法如上注解讲述，主要目的就是创建`Root WebApplicationContext对象`，即`根IoC容器`，其中比较重要的就是，整个`Web应用`如果存在`根IoC容器`则有且只能有一个，`根IoC容器`作为全局变量存储在`ServletContext`中。将`根IoC容器`放入到`ServletContext`对象之前进行了`IoC容器`的配置和刷新操作，调用了`configureAndRefreshWebApplicationContext()`方法，该方法源码如下:

```java
// 配置并刷新上下文，进行Bean的创建和初始化
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
   if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
      String idParam = sc.getInitParameter(CONTEXT_ID_PARAM); // 自定义contextId
      if (idParam != null) {
         wac.setId(idParam);
      }
      else {
         // Generate default id... 默认的context id
         wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
               ObjectUtils.getDisplayString(sc.getContextPath()));
      }
   }

   wac.setServletContext(sc);
   // CONFIG_LOCATION_PARAM = "contextConfigLocation"，xml配置文件路径
   // 获取web.xml中<context-param>标签配置的全局变量，其中key为CONFIG_LOCATION_PARAM，
   // 也就是配置的相应Bean的xml文件名，并将其设置到WebApplicationContext中。
   String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
   if (configLocationParam != null) {
      wac.setConfigLocation(configLocationParam); // 设置配置文件路径
   }

   ConfigurableEnvironment env = wac.getEnvironment();
   if (env instanceof ConfigurableWebEnvironment) {
      // 替换占位符属性源为实际的属性源
      ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
   }

   // 自定义上下文，获取所有初始化器ApplicationContextInitializer，并回调initialize方法
   customizeContext(sc, wac);
   wac.refresh(); // 刷新上下文
}
```

比较重要的就是获取到了`web.xml`中的`<context-param>标签`配置的全局变量`contextConfigLocation`，然后调用`customizeContext(ServletContext, ConfigurableWebApplicationContext)`方法，并在最后调用了`refresh()`方法，下面先看一下`customizeContext(ServletContext, ConfigurableWebApplicationContext)`方法的源码：

```java
protected void customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac) {
   List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> initializerClasses =
         determineContextInitializerClasses(sc); // 获取应用上下文初始化器类列表

   for (Class<ApplicationContextInitializer<ConfigurableApplicationContext>> initializerClass : initializerClasses) {
      // 解析上下文类
      Class<?> initializerContextClass =
            GenericTypeResolver.resolveTypeArgument(initializerClass, ApplicationContextInitializer.class);
      if (initializerContextClass != null && !initializerContextClass.isInstance(wac)) {
         throw new ApplicationContextException(String.format(
               "Could not apply context initializer [%s] since its generic parameter [%s] " +
               "is not assignable from the type of application context used by this " +
               "context loader: [%s]", initializerClass.getName(), initializerContextClass.getName(),
               wac.getClass().getName()));
      }
      this.contextInitializers.add(BeanUtils.instantiateClass(initializerClass)); // 实例化初始化器
   }

   AnnotationAwareOrderComparator.sort(this.contextInitializers); // 排序
   for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
      initializer.initialize(wac); // 调用初始化器的initialize方法
   }
}
```

`customizeContext(ServletContext, ConfigurableWebApplicationContext)`方法主要实例化了`ApplicationContextInitializer`实例集合，并应用到Web应用上下文中。

```java
initializer.initialize(wac);
```

然后执行Web应用上下文的`refresh()`方法，该方法主要用于创建并初始化`contextConfigLocation`配置的`xml文件`中的`Bean`，因此，如果在配置`Bean`时出错，在`Web应用`启动时就会抛出异常，而不是等到运行时才抛出异常。

整个`ContextLoaderListener类`的启动过程到此就结束了，可以发现，创建`ContextLoaderListener`是比较核心的一个步骤，主要工作就是为了创建`根IoC容器`并使用特定的`key`将其放入到`ServletContext`对象中，供整个`Web应用`使用。由于在`ContextLoaderListener类`中构造的`根IoC容器`配置的`Bean`是全局共享的，因此，在`<context-param>`标识的`contextConfigLocation`的`xml配置文件`一般包括:`数据库DataSource`、`DAO层`、`Service层`、`事务`等相关`Bean`。

##### 2. Filter的初始化

在监听器`listener`初始化完成后，按照文章开始的讲解，接下来会进行`filter`的初始化操作，`filter`的创建和初始化中没有涉及`IoC容器`的相关操作，因此不是本文讲解的重点，本文举例的`filter`是一个用于编码用户请求和响应的过滤器，采用`utf-8`编码用于适配中文。

##### 3. Servlet的初始化

`Web应用`启动的最后一个步骤就是创建和初始化相关`Servlet`，在开发中常用的`Servlet`就是`DispatcherServlet类`前端控制器，前端控制器作为中央控制器是整个`Web应用`的核心，用于获取、分发用户请求并返回响应，借用网上一张关于`DispatcherServlet类`的类图，其类图如下所示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-04.jpeg?x-oss-process=style/markdown-pic)

通过类图可以看出`DispatcherServlet类`的间接父类实现了`Servlet接口`，因此其本质上依旧是一个`Servlet`。`DispatcherServlet类`的设计很巧妙，上层父类不同程度的实现了相关接口的部分方法，并留出了相关方法用于子类覆盖，将不变的部分统一实现，将变化的部分预留方法用于子类实现。

通过对上述类图中相关类的源码分析，可以绘制如下相关初始化方法调用逻辑:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-05.jpeg)

通过类图和相关初始化函数调用的逻辑来看，`DispatcherServlet类`的初始化过程将模板方法使用的淋漓尽致，其父类完成不同的统一的工作，并预留出相关方法用于子类覆盖去完成不同的可变工作。

`DispatcherServelt`类的本质是`Servlet`，通过文章开始的讲解可知，在`Web应用`部署到容器后进行`Servlet`初始化时会调用相关的`init(ServletConfig)`方法，因此，`DispatchServlet类`的初始化过程也由该方法开始。上述调用逻辑中比较重要的就是`FrameworkServlet抽象类`中的`initServletBean()`方法、`initWebApplicationContext()`方法以及`DispatcherServlet类`中的`onRefresh()`方法，接下来会逐一进行讲解。

首先查看一下`initServletBean()`的相关源码如下图所示:

```java
protected final void initServletBean() throws ServletException {
   getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
   if (logger.isInfoEnabled()) {
      logger.info("Initializing Servlet '" + getServletName() + "'");
   }
   long startTime = System.currentTimeMillis();

   try {
      // 初始化DispatcherServlet web应用上下文
      this.webApplicationContext = initWebApplicationContext(); 
      initFrameworkServlet(); // 空方法
   }
   catch (ServletException | RuntimeException ex) {
      logger.error("Context initialization failed", ex);
      throw ex;
   }

   if (logger.isDebugEnabled()) {
      String value = this.enableLoggingRequestDetails ?
            "shown which may lead to unsafe logging of potentially sensitive data" :
            "masked to prevent unsafe logging of potentially sensitive data";
      logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
            "': request parameters and headers will be " + value);
   }

   if (logger.isInfoEnabled()) {
      logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
   }
}
```

该方法重写了`FrameworkServlet抽象类`的父类`HttpServletBean抽象类`的`initServletBean()`方法，`HttpServletBean抽象类`在执行`init()`方法时会调用`initServletBean()`方法，由于多态的特性，最终会调用其子类`FrameworkServlet抽象类`的`initServletBean()`方法。该方法由`final`标识，子类就不可再次重写了。该方法中比较重要的就是`initWebApplicationContext()`方法的调用，该方法仍由`FrameworkServlet抽象类`实现，继续查看其源码如下所示:

```java
protected WebApplicationContext initWebApplicationContext() {
   // 获取由ContextLoaderListener创建的ROOT Web应用上下文
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) { // 如果通过构造器注入了上下文，则直接使用
      // A context instance was injected at construction time -> use it
      wac = this.webApplicationContext;
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         if (!cwac.isActive()) { // 如果未刷新，则刷新上下文
            if (cwac.getParent() == null) {
               cwac.setParent(rootContext); // 设置父上下文
            }
            // 配置并刷新Web上下文，初始化Bean
            configureAndRefreshWebApplicationContext(cwac);
         }
      }
   }
   if (wac == null) {
      wac = findWebApplicationContext(); // 从servlet context查找是否有初始化完成的web应用上下文
   }
   if (wac == null) {
      wac = createWebApplicationContext(rootContext); // 还是没有，则创建一个web应用上下文
   }

   // 这里refreshEventReceived为true，onApplicationEvent(ContextRefreshedEvent)
   // 在上下文初始化完成后已将refreshEventReceived置为true
   if (!this.refreshEventReceived) {
      synchronized (this.onRefreshMonitor) {
         onRefresh(wac);
      }
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
      // 将当前web应用上下文的引用设置到servlet context
      getServletContext().setAttribute(attrName, wac); 
   }

   return wac;
}
```

通过函数名不难发现，该方法的主要作用同样是创建一个`WebApplicationContext`对象，即`Ioc容器`，不过前文讲过每个`Web应用`最多只能存在一个`根IoC容器`，这里创建的则是特定`Servlet`拥有的`子IoC容器`。可能有些读者会有疑问，为什么需要多个`Ioc容器`，首先介绍一个`父子IoC容器`的访问特性，有兴趣的读者可以自行实验。

------

###### 父子IoC容器的访问特性

在学习`Spring`时，都是从读取`xml配置文件`来构造`IoC容器`，常用的类有`ClassPathXmlApplicationContext类`，该类存在一个初始化方法用于传入`xml文件`路径以及一个`父容器`，因此可以创建两个不同的`xml配置文件`并实现如下代码:

```java
// applicationContext1.xml文件中配置一个id为baseBean的Bean
ApplicationContext baseContext = new ClassPathXmlApplicationContext("applicationContext1.xml");

Object obj1 = baseContext.getBean("baseBean");

System.out.println("baseContext Get Bean " + obj1);

// applicationContext2.xml文件中配置一个id未subBean的Bean
ApplicationContext subContext = new ClassPathXmlApplicationContext(new String[]{"applicationContext2.xml"}, baseContext);

Object obj2 = subContext.getBean("baseBean");

System.out.println("subContext get baseContext Bean " + obj2);

Object obj3 = subContext.getBean("subBean");

System.out.println("subContext get subContext Bean " + obj3);

// 抛出NoSuchBeanDefinitionException异常
Object obj4 = baseContext.getBean("subBean");

System.out.println("baseContext get subContext Bean " + obj4);
```

首先创建`baseContext`没有为其设置`父容器`，接着可以成功获取`id`为`baseBean`的`Bean`，接着创建`subContext`并将`baseContext`设置为其`父容器`，`subContext`可以成功获取`baseBean`以及`subBean`，最后试图使用`baseContext`去获取`subContext`中定义的`subBean`，此时会抛出异常`NoSuchBeanDefinitionException`，由此可见，`父子容器`类似于类的继承关系，子类可以访问父类中的成员变量，而父类不可访问子类的成员变量，同样的，`子容器`可以访问`父容器`中定义的`Bean`，但`父容器`无法访问`子容器`定义的`Bean`。

通过上述实验就可以理解为何需要创建多个`Ioc容器`，`根IoC容器`做为全局共享的`IoC容器`放入`Web应用`需要共享的`Bean`，而`子IoC容器`根据需求的不同，放入不同的`Bean`，这样能够做到隔离，保证系统的安全性。

------

接下来继续讲解`DispatcherServlet类`的`子IoC容器`创建过程，如果当前`Servlet`存在一个`IoC容器`则为其设置`根IoC容器`作为其父类，并配置刷新该容器，用于构造其定义的`Bean`，这里的方法与前文讲述的`根IoC容器`类似。如果当前`Servlet`不存在一个`子IoC容器`就去查找一个，如果仍然没有查找到则调用`createWebApplicationContext()`方法去创建一个，查看该方法的源码如下图所示:

```java
protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
   return createWebApplicationContext((ApplicationContext) parent);
}

protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
	Class<?> contextClass = getContextClass(); // 获取配置的上下文类
	if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) { // 类型检查
		throw new ApplicationContextException(
				"Fatal initialization error in servlet with name '" + getServletName() +
				"': custom WebApplicationContext class [" + contextClass.getName() +
				"] is not of type ConfigurableWebApplicationContext");
	}
	ConfigurableWebApplicationContext wac =
			(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass); // 实例化

	// 设置环境、父上下文、配置文件路径
	wac.setEnvironment(getEnvironment());
	wac.setParent(parent);
	String configLocation = getContextConfigLocation();
	if (configLocation != null) {
		wac.setConfigLocation(configLocation);
	}
	configureAndRefreshWebApplicationContext(wac); // 配置并刷新Web应用上下文

	return wac;
}

// 配置并刷新Web应用上下文
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		if (this.contextId != null) {
			wac.setId(this.contextId); // 设置自定义的id
		}
		else {
			// Generate default id... 使用默认的id
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
		}
	}

	wac.setServletContext(getServletContext()); // 设置ServletContext、ServletConfig、Namespace
	wac.setServletConfig(getServletConfig());
	wac.setNamespace(getNamespace());
	// 添加事件监听器，只处理该web应用上下文发出的事件
	wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		// 初始化属性源ServletContext、ServletConfig，替换servlet占位符属性源
		((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
	}

	postProcessWebApplicationContext(wac); // 后处理web上下文
	applyInitializers(wac); // 应用初始化器到上下文
	wac.refresh(); // 刷新上下文
}
```

`createWebApplicationContext()`方法用于创建一个`子IoC容器`并将`根IoC容器`做为其父容器，接着进行配置和刷新操作用于构造相关的`Bean`。在`configureAndRefreshWebApplicationContext`方法中，将`ServletContext、ServletConfig`设置到了`子IoC容器`，并添加了事件监听器`new SourceFilteringListener(wac, new ContextRefreshListener())`，其只对于`wac(ConfigurableWebApplicationContext)`发布的`ContextRefreshedEvent`事件进行响应。`ContextRefreshListener`类源码如下：

```java
private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {

   @Override
   public void onApplicationEvent(ContextRefreshedEvent event) {
      // 调用FrameworkServlet.onApplicationEvent方法
      FrameworkServlet.this.onApplicationEvent(event);
   }
}

public void onApplicationEvent(ContextRefreshedEvent event) {
	this.refreshEventReceived = true;
	synchronized (this.onRefreshMonitor) {
		onRefresh(event.getApplicationContext()); // 回调
	}
}
```

根据`ContextRefreshListener`的源码可知，在应用上下文刷新完成后，会调用`FrameworkServlet.this.onApplicationEvent(event)`，也就是在这里回调了`onRefresh()`方法，该方法由子类`DispatcherServlet`实现。

在`configureAndRefreshWebApplicationContext`方法中，也调用了`applyInitializers(wac)`，用于将·应用上下文初始化器`ApplicationContextInitializer`应用到`子IoC容器`ConfigurableWebApplicationContext中，如下所示：

```java
protected void applyInitializers(ConfigurableApplicationContext wac) {
   // 全局初始化器类名
   String globalClassNames = getServletContext().getInitParameter(ContextLoader.GLOBAL_INITIALIZER_CLASSES_PARAM);
   if (globalClassNames != null) {
      for (String className : StringUtils.tokenizeToStringArray(globalClassNames, INIT_PARAM_DELIMITERS)) {
         this.contextInitializers.add(loadInitializer(className, wac)); // 实例化初始化器
      }
   }

   if (this.contextInitializerClasses != null) {
      for (String className : StringUtils.tokenizeToStringArray(this.contextInitializerClasses, INIT_PARAM_DELIMITERS)) {
         this.contextInitializers.add(loadInitializer(className, wac));
      }
   }

   AnnotationAwareOrderComparator.sort(this.contextInitializers);// 排序
   for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
      initializer.initialize(wac); // 回调initialize方法
   }
}
```

至此，`根IoC容器`以及相关`Servlet`的`子IoC容器`已经配置完成，`子容器`中管理的`Bean`一般只被该`Servlet`使用，因此，其中管理的`Bean`一般是“局部”的，如`Spring MVC`中需要的各种重要组件，包括`Controller`、`HandlerAdapter`、`HandlerMapping`、`ViewResolver`等。

相关关系如下图所示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-06.jpeg)

当`IoC子容器`刷新完成后会调用`onRefresh()`方法，该方法的调用与`initServletBean()`方法的调用相同，由父类调用但具体实现由子类覆盖，调用`onRefresh()`方法时将前文创建的`IoC子容器`作为参数传入，查看`DispatcherServletBean类`的`onRefresh()`方法源码如下:

```java
// context为DispatcherServlet创建的一个IoC子容器 WebApplicationContext
protected void onRefresh(ApplicationContext context) { 
   initStrategies(context);
}

protected void initStrategies(ApplicationContext context) { 
   initMultipartResolver(context); // 初始化文件上传解析器
   initLocaleResolver(context); // 初始化LocaleResolver(AcceptHeaderLocaleResolver)
   initThemeResolver(context); // 初始化ThemeResolver(FixedThemeResolver)
   initHandlerMappings(context); // 初始化BeanNameUrlHandlerMapping, RequestMappingHandlerMapping
   initHandlerAdapters(context); // 初始化HandlerAdapter
   initHandlerExceptionResolvers(context); // 初始化HandlerExceptionResolver
   initRequestToViewNameTranslator(context);
   initViewResolvers(context); // 初始化ViewResolver
   initFlashMapManager(context);
}
```

`onRefresh()`方法直接调用了`initStrategies()`方法，源码如上，通过函数名可以判断，该方法用于初始化创建`multipartResovler`来支持图片等文件的上传、本地化解析器、主题解析器、`HandlerMapping`处理器映射器、`HandlerAdapter`处理器适配器、异常解析器、视图解析器、flashMap管理器等，这些组件都是`SpringMVC`开发中的重要组件，相关组件的初始化创建过程均在此完成。本文第三部分将详细分析`HandlerMapping`、`HandlerAdapter`以及`ViewResolver`的初始化逻辑。

至此，`DispatcherServlet类`的创建和初始化过程也就结束了，整个`Web应用`部署到容器后的初始化启动过程的重要部分全部分析清楚了，通过前文的分析可以认识到层次化设计的优点，以及`IoC容器`的继承关系所表现的隔离性。分析源码能让我们更清楚的理解和认识到相关初始化逻辑以及配置文件的配置原理。

### 三、DispatcherServlet初始化过程中的细节

##### 1. HandlerMapping的初始化(RequestMappingHandlerMapping)

先看下`initHandlerMappings(context)`的实现：

```java
private void initHandlerMappings(ApplicationContext context) {
   this.handlerMappings = null;

   // 是否检测出所有的HandlerMapping Bean，还是只检测name为handlerMapping的HandlerMapping Bean
   if (this.detectAllHandlerMappings) {
      // 找出所有的HandlerMapping Bean，包括父BeanFactory中的定义
      Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerMappings = new ArrayList<>(matchingBeans.values());
         // We keep HandlerMappings in sorted order.
         AnnotationAwareOrderComparator.sort(this.handlerMappings); // 排序
      }
   }
   else {
      try {
         HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
         this.handlerMappings = Collections.singletonList(hm); // 一个
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerMapping later.
      }
   }

   // 默认是BeanNameUrlHandlerMapping, RequestMappingHandlerMapping(这里！！！)
   if (this.handlerMappings == null) {
      // 初始化BeanNameUrlHandlerMapping, RequestMappingHandlerMapping(初始化时扫描Controller，注册HandlerMethod)
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

在`initHandlerMappings(context)`方法中，默认情况下会初始化`BeanNameUrlHandlerMapping, RequestMappingHandlerMapping`这两个`HandlerMapping`。这里重点分析**RequestMappingHandlerMapping(基于@RequestMapping注解的HandlerMapping)**的初始化流程，`BeanNameUrlHandlerMapping`请读者自行查看源码。

首先看下`RequestMappingHandlerMapping`的类继承结构：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-07.jpg)

可见`RequestMappingHandlerMapping`继承了`ApplicationObjectSupport、WebApplicationObjectSupport、AbstractHandlerMapping、AbstractHandlerMethodMapping、RequestMappingInfoHandlerMapping`类，且实现了`ApplicationContextAware、ServletContextAware、InitializingBean`接口。

在`RequestMappingHandlerMapping`初始化时，首先会回调`ApplicationContextAware`接口的回调方法。

```java
// ApplicationObjectSupport类
public final void setApplicationContext(@Nullable ApplicationContext context) throws BeansException {
   if (context == null && !isContextRequired()) {
      // Reset internal context state.
      this.applicationContext = null;
      this.messageSourceAccessor = null;
   }
   else if (this.applicationContext == null) {
      // Initialize with passed-in context.
      if (!requiredContextClass().isInstance(context)) {
         throw new ApplicationContextException(
               "Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
      }
      this.applicationContext = context;
      this.messageSourceAccessor = new MessageSourceAccessor(context);
      initApplicationContext(context); // 初始化
   }
   else {
      // Ignore reinitialization if same context passed in.
      if (this.applicationContext != context) {
         throw new ApplicationContextException(
               "Cannot reinitialize with different application context: current one is [" +
               this.applicationContext + "], passed-in one is [" + context + "]");
      }
   }
}
```

上面代码重点看`initApplicationContext(context)`部分。

```java
// WebApplicationObjectSupport类
protected void initApplicationContext(ApplicationContext context) {
	 // ApplicationObjectSupport.initApplicationContext()
   super.initApplicationContext(context); 
   if (this.servletContext == null && context instanceof WebApplicationContext) {
     	// 设置ServletContext
      this.servletContext = ((WebApplicationContext) context).getServletContext();
      if (this.servletContext != null) {
         initServletContext(this.servletContext); // 空方法体
      }
   }
}
// ApplicationObjectSupport类
protected void initApplicationContext(ApplicationContext context) throws BeansException {
	initApplicationContext(); // --> AbstractHandlerMapping类
}
// AbstractHandlerMapping类
protected void initApplicationContext() throws BeansException {
	extendInterceptors(this.interceptors); // 添加自定义的拦截器(空方法)
	detectMappedInterceptors(this.adaptedInterceptors); // 从容器中获取所有MappedInterceptor Bean
	initInterceptors(); // 初始化拦截器，并适配拦截器
}
// AbstractHandlerMapping类
protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
	mappedInterceptors.addAll(
			BeanFactoryUtils.beansOfTypeIncludingAncestors(
					obtainApplicationContext(), MappedInterceptor.class, true, false).values());
}
// AbstractHandlerMapping类
protected void initInterceptors() {
	if (!this.interceptors.isEmpty()) {
		for (int i = 0; i < this.interceptors.size(); i++) {
			Object interceptor = this.interceptors.get(i);
			if (interceptor == null) {
				throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
			}
			this.adaptedInterceptors.add(adaptInterceptor(interceptor)); // 将拦截器适配成HandlerInterceptor
		}
	}
}
```

`initApplicationContext(context)`方法主要加载了拦截器`HandlerInterceptor`，并设置了`ServletContext`。

在`ApplicationContextAware`接口回调方法完成后，执行`ServletContextAware`的回调方法。

```java
// WebApplicationObjectSupport类
public final void setServletContext(ServletContext servletContext) {
   if (servletContext != this.servletContext) {
      this.servletContext = servletContext; // 设置ServletContext
      initServletContext(servletContext);
   }
}
```

然后执行`RequestMappingHandlerMapping`的`afterPropertiesSet`方法：

```java
public void afterPropertiesSet() {
   this.config = new RequestMappingInfo.BuilderConfiguration();
   this.config.setUrlPathHelper(getUrlPathHelper());
   this.config.setPathMatcher(getPathMatcher());
   this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
   this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
   this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
   this.config.setContentNegotiationManager(getContentNegotiationManager());

   super.afterPropertiesSet(); // 扫描bean，注册HandlerMethod
}
```

在`afterPropertiesSet`方法中会调用父类`AbstractHandlerMethodMapping`的`afterPropertiesSet`方法。

```java
// AbstractHandlerMethodMapping类
public void afterPropertiesSet() {
   initHandlerMethods();  // 扫描bean，注册HandlerMethod
}
```

在`AbstractHandlerMethodMapping.afterPropertiesSet()`方法中，调用了`initHandlerMethods()`方法，扫描bean并注册HandlerMethod。

```java
// AbstractHandlerMethodMapping类
protected void initHandlerMethods() {
   for (String beanName : getCandidateBeanNames()) { // 获取所有的bean names
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         processCandidateBean(beanName);
      }
   }
   handlerMethodsInitialized(getHandlerMethods()); // 日志打印
}
```

`initHandlerMethods()`方法调用`processCandidateBean()`方法来检测`HandlerMathod`。

```java
// 确定bean的类型，判断是否是Handler，检测HandlerMethod
protected void processCandidateBean(String beanName) {
   Class<?> beanType = null;
   try {
      beanType = obtainApplicationContext().getType(beanName); // 获取Bean类型
   }
   catch (Throwable ex) {
      // An unresolvable bean type, probably from a lazy bean - let's ignore it. (忽略)
      if (logger.isTraceEnabled()) {
         logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
      }
   }
   // 类上标注@Controller或@RequestMapping注解，即为Handler
   if (beanType != null && isHandler(beanType)) { 
      detectHandlerMethods(beanName); // 检测HandlerMathod并注册
   }
}
```

`processCandidateBean`方法检测bean的类型，若bean类上标注`@Controller或@RequestMapping`注解，则该bean为Handler。然后调用`detectHandlerMethods()`方法检测并注册HandlerMathod。

```java
protected void detectHandlerMethods(Object handler) {
   Class<?> handlerType = (handler instanceof String ?
         obtainApplicationContext().getType((String) handler) : handler.getClass()); // 获取handler type

   if (handlerType != null) {
      Class<?> userType = ClassUtils.getUserClass(handlerType); // 获取目标类
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
               try {
                  return getMappingForMethod(method, userType);
               }
               catch (Throwable ex) {
                  throw new IllegalStateException("Invalid mapping on handler class [" +
                        userType.getName() + "]: " + method, ex);
               }
            });
      if (logger.isTraceEnabled()) {
         logger.trace(formatMappings(userType, methods)); // 格式化输出RequestMappingInfo
      }
      methods.forEach((method, mapping) -> {
         Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
         registerHandlerMethod(handler, invocableMethod, mapping); // 注册HandlerMethod
      });
   }
}
```

`detectHandlerMethods()`方法首先使用`MethodIntrospector`查找类和方法上的`@RequestMapping`注解信息，并构建`RequestMappingInfo`对象。然后根据这些获取到的`RequestMappingInfo`对象和相应的方法，调用`registerHandlerMethod`方法注册`HandlerMethod`。

首先来看`getMappingForMethod()`方法，该方法用于根据`Method`构建`RequestMappingInfo`信息:

```java
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
   RequestMappingInfo info = createRequestMappingInfo(method); // 创建RequestMappingInfo
   if (info != null) {
      RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType); // 获取类上的RequestMappingInfo
      if (typeInfo != null) {
         info = typeInfo.combine(info); // 合并类级别和方法级别的RequestMappingInfo
      }
      String prefix = getPathPrefix(handlerType);
      if (prefix != null) {
         info = RequestMappingInfo.paths(prefix).build().combine(info);
      }
   }
   return info;
}
// 基于@RequestMapping注解，创建RequestMappingInfo对象
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
	// 获取@RequestMapping注解
	RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
	RequestCondition<?> condition = (element instanceof Class ?
			getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
	return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

`getMappingForMethod()`方法查找方法和类上的`@RequestMapping`注解，并创建`RequestMappingInfo`对象。

然后来看`registerHandlerMethod`方法的逻辑：

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
   this.mappingRegistry.register(mapping, handler, method); // 注册HandlerMethod
}

// 注册HandlerMethod
// @param mapping RequestMappingInfo
// @param handler handler beanName(String)
// @param method 处理器方法
public void register(T mapping, Object handler, Method method) {
	this.readWriteLock.writeLock().lock();
	try {
		HandlerMethod handlerMethod = createHandlerMethod(handler, method); // 创建HandlerMethod
		assertUniqueMethodMapping(handlerMethod, mapping); // 确定映射的唯一性
		this.mappingLookup.put(mapping, handlerMethod); // 保存

		List<String> directUrls = getDirectUrls(mapping); // 从RequestMappingInfo获取不需要match的url
		for (String url : directUrls) {
			this.urlLookup.add(url, mapping);
		}

		String name = null;
		if (getNamingStrategy() != null) {
			name = getNamingStrategy().getName(handlerMethod, mapping);
			addMappingName(name, handlerMethod);
		}

		CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
		if (corsConfig != null) {
			this.corsLookup.put(handlerMethod, corsConfig);
		}

		this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
	}
	finally {
		this.readWriteLock.writeLock().unlock();
	}
}
```

`registerHandlerMethod()`方法调用了`MappingRegistry`的`register`方法，该方法主要是创建了方法`method`对应的`HandlerMethod`，并注册到对应的缓存中。

至此，`RequestMappingHandlerMapping`的初始化流程分析结束，主要是扫描容器中的Bean，提取类上和方法上的`@RequestMapping`注解信息，根据这些注解信息创建`RequestMappingInfo`对象，并注册`HandlerMethod`到缓存中，以便于`DispatcherServlet`接收用户请求时匹配。

##### 2. HandlerAdapter的初始化(RequestMappingHandlerAdapter)

先看下`initHandlerAdapters(context)`的实现：

```java
private void initHandlerAdapters(ApplicationContext context) {
   this.handlerAdapters = null;

   // 是否检测出所有的HandlerAdapter Bean，还是只检测name为handlerAdapter的HandlerAdapter Bean
   if (this.detectAllHandlerAdapters) {
      // 找出所有的HandlerAdapter Bean，包括父BeanFactory中的定义
      Map<String, HandlerAdapter> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerAdapters = new ArrayList<>(matchingBeans.values());
         AnnotationAwareOrderComparator.sort(this.handlerAdapters); // 排序
      }
   }
   else {
      try {
         HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
         this.handlerAdapters = Collections.singletonList(ha); // 一个
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerAdapter later.
      }
   }

   // 没有自定义的HandlerAdapter，则使用默认的：HttpRequestHandlerAdapter、
   // SimpleControllerHandlerAdapter、RequestMappingHandlerAdapter（注意这里！！！）
   if (this.handlerAdapters == null) {
      this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

在`initHandlerAdapters(context)`方法中，默认情况下会初始化`HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter、RequestMappingHandlerAdapter`这三个`HandlerAdapter`。这里分析下**RequestMappingHandlerAdapter**的初始化流程，其余两个类请读者自行查看源码。

首先看下`RequestMappingHandlerAdapter`的类继承结构：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-08.jpg)

可见`RequestMappingHandlerAdapter`继承了`ApplicationObjectSupport、WebApplicationObjectSupport、WebContentGenerator、AbstractHandlerMethodAdapter`类，且实现了`ApplicationContextAware、ServletContextAware、InitializingBean`接口。

首先看`RequestMappingHandlerAdapter`的构造函数：

```java
public RequestMappingHandlerAdapter() {
   StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
   stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

   this.messageConverters = new ArrayList<>(4);
   this.messageConverters.add(new ByteArrayHttpMessageConverter());
   this.messageConverters.add(stringHttpMessageConverter);
   try {
      this.messageConverters.add(new SourceHttpMessageConverter<>());
   }
   catch (Error err) {
      // Ignore when no TransformerFactory implementation is available
   }
   this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
}
```

从构造函数可以看出，构造函数中向`messageConverters`变量添加了4个`HttpMessageConverter`：`ByteArrayHttpMessageConverter、StringHttpMessageConverter、SourceHttpMessageConverter、AllEncompassingFormHttpMessageConverter`。

`ApplicationContextAware、ServletContextAware`这两个接口的回调过程前面已经分析，这里不再介绍。

接着看`RequestMappingHandlerAdapter`的`afterPropertiesSet()`方法：

```java
public void afterPropertiesSet() {
   // Do this first, it may add ResponseBody advice beans
   initControllerAdviceCache(); // 初始化ControllerAdvice缓存

   if (this.argumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
      this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.initBinderArgumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
      this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.returnValueHandlers == null) {
      List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
      this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   }
}

// 初始化ControllerAdvice缓存
private void initControllerAdviceCache() {
	if (getApplicationContext() == null) {
		return;
	}

	// 找出所有注解了@ControllerAdvice的Bean
	List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
	AnnotationAwareOrderComparator.sort(adviceBeans); // 排序

	List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

	for (ControllerAdviceBean adviceBean : adviceBeans) { // 遍历
		Class<?> beanType = adviceBean.getBeanType();
		if (beanType == null) {
			throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
		}
		Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
		if (!attrMethods.isEmpty()) {
			this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
		}
		Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
		if (!binderMethods.isEmpty()) {
			this.initBinderAdviceCache.put(adviceBean, binderMethods);
		}
		if (RequestBodyAdvice.class.isAssignableFrom(beanType)) {
			requestResponseBodyAdviceBeans.add(adviceBean);
		}
		if (ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
			requestResponseBodyAdviceBeans.add(adviceBean);
		}
	}

	if (!requestResponseBodyAdviceBeans.isEmpty()) {
		this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
	}

	if (logger.isDebugEnabled()) {
		int modelSize = this.modelAttributeAdviceCache.size();
		int binderSize = this.initBinderAdviceCache.size();
		int reqCount = getBodyAdviceCount(RequestBodyAdvice.class);
		int resCount = getBodyAdviceCount(ResponseBodyAdvice.class);
		if (modelSize == 0 && binderSize == 0 && reqCount == 0 && resCount == 0) {
			logger.debug("ControllerAdvice beans: none");
		}
		else {
			logger.debug("ControllerAdvice beans: " + modelSize + " @ModelAttribute, " + binderSize +
					" @InitBinder, " + reqCount + " RequestBodyAdvice, " + resCount + " ResponseBodyAdvice");
		}
	}
}
```

`afterPropertiesSet()`方法中首先调用了`initControllerAdviceCache()`方法，初始化ControllerAdvice缓存。然后分别加载了`argumentResolvers、initBinderArgumentResolvers、returnValueHandlers`这些变量对应的实例。

至此，`RequestMappingHandlerAdapter`类的初始化简要分析到此结束。

##### 3. ViewResovler的初始化(InternalResourceViewResolver)

先看下`initViewResolvers(context)`的逻辑：

```java
private void initViewResolvers(ApplicationContext context) {
   this.viewResolvers = null;

   if (this.detectAllViewResolvers) {
      // Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
      Map<String, ViewResolver> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.viewResolvers = new ArrayList<>(matchingBeans.values());
         // We keep ViewResolvers in sorted order.
         AnnotationAwareOrderComparator.sort(this.viewResolvers);
      }
   }
   else {
      try {
         ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
         this.viewResolvers = Collections.singletonList(vr);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default ViewResolver later.
      }
   }

   // 默认使用InternalResourceViewResolver（注意这里！！！）
   if (this.viewResolvers == null) {
      this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No ViewResolvers declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

在`initViewResolvers(context)`方法中，默认情况下会初始化`InternalResourceViewResolver`。下面看下`InternalResourceViewResolver`的初始化流程。

首先看下`InternalResourceViewResolver`的类继承结构：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-09.jpg)

可见`InternalResourceViewResolver`继承了`ApplicationObjectSupport、WebApplicationObjectSupport、AbstractCachingViewResolver、UrlBasedViewResolver`类，且实现了`ApplicationContextAware、ServletContextAware`接口。

`InternalResourceViewResolver`初始化时，先调用其构造函数设置视图类：

```java
public InternalResourceViewResolver() {
   Class<?> viewClass = requiredViewClass(); // 获取视图类InternalResourceView
   if (InternalResourceView.class == viewClass && jstlPresent) {
      viewClass = JstlView.class;
   }
   setViewClass(viewClass); // 设置视图类
}

protected Class<?> requiredViewClass() {
	return InternalResourceView.class;
}
```

`ApplicationContextAware、ServletContextAware`这两个接口的回调过程前面已经分析，这里不再介绍。

`InternalResourceViewResolver`类的初始化简要分析到此结束。

### 四、总结

这里给出一个简洁的文字描述版`Spring MVC启动过程`:

Tomcat web容器启动时会去读取`web.xml`这样的`部署描述文件`，相关组件启动顺序为: `解析<context-param>` => `解析<listener>` => `解析<filter>` => `解析<servlet>`，具体初始化过程如下:

1、解析`<context-param>`里的键值对。

2、创建`ServletContext`，servlet上下文，用于全局共享。

3、将`<context-param>`的键值对放入`ServletContext`中，`Web应用`内全局共享。

4、读取`<listener>`标签创建监听器，一般会使用`ContextLoaderListener类`，如果使用了`ContextLoaderListener类`，`Spring`就会创建一个`WebApplicationContext类`的对象，`WebApplicationContext类`就是`IoC容器`，`ContextLoaderListener类`创建的`IoC容器`是`根IoC容器`，为全局性的，并将其放置在`ServletContext`中，作为应用内全局共享，键名为`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`。

​     这个全局的`根IoC容器`只能获取到在该容器中创建的`Bean`，不能访问到其他容器创建的`Bean`，也就是读取`web.xml`配置的`contextConfigLocation`参数的`xml文件`来创建对应的`Bean`。

5、`listener`创建完成后如果有`<filter>`则会去创建`filter`。

6、初始化创建`<servlet>`，一般使用`DispatchServlet类`。

7、`DispatchServlet`的父类`FrameworkServlet`会重写其父类的`initServletBean`方法，并调用`initWebApplicationContext()`以及`onRefresh()`方法。

8、`initWebApplicationContext()`方法会创建一个当前`servlet`的一个`IoC子容器`，如果存在上述的全局`WebApplicationContext`，则将其设置为`父容器`。

9、读取`<servlet>`标签的`<init-param>`配置的`xml文件`并加载相关`Bean`。

10、`onRefresh()`方法创建`Web应用`相关组件，如`HandlerMapping、HandlerAdapter、ViewResolver`等。

### 参考文章

- [SpringMVC 启动流程及相关源码分析](https://www.jianshu.com/p/dc64d02e49ac)
- [Spring MVC 原理探秘 - 容器的创建过程](http://www.tianxiaobo.com/2018/06/30/Spring-MVC-%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A7%98-%E5%AE%B9%E5%99%A8%E7%9A%84%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B/)

