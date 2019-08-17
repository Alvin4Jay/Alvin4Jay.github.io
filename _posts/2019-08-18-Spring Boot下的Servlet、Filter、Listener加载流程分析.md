---
layout:     post
title:      Spring Boot下的Servlet、Filter、Listener加载流程分析
subtitle:   ServletContainerInitializer/ServletContextInitializer
date:       2019-08-18
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring MVC
---

# Spring Boot下的Servlet、Filter、Listener加载流程分析

本文将详细分析Spring Boot中Servlet、Filter、Listener的加载流程，包括DispatcherServlet的注册。读者阅读之前，最好先阅读Spring MVC和Servlet规范相关的内容，参考[Spring MVC](https://xuanjian1992.top/tags/#Spring%20MVC)。

### 一、Servlet、Filter、Listener的注册方式

下面先介绍Spring Boot中Servlet、Filter、Listener三大组件的注册方式，然后再详细分析这些注册方式的原理。

##### 1.Servlet 3.0注解+@ServletComponentScan

Spring Boot兼容Servlet 3.0一系列以 @Web* 开头的注解：@WebServlet，@WebFilter，@WebListener。

```java
@WebServlet("/hello")
public class HelloServlet  extends HttpServlet {}

@WebFilter("/hello/*")
public class HelloFilter implements Filter {}
```

然后在启动类中添加@ServletComponentScan注解，以扫描Servlet 3.0 @Web* 注解的类。

```java
@SpringBootApplication
@ServletComponentScan
public class SpringBootServletStartApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootServletStartApplication.class, args);
    }
}
```

##### 2.RegistrationBean

```java
@Bean
public ServletRegistrationBean servletRegistrationBean() {
    ServletRegistrationBean registrationBean = new ServletRegistrationBean();
    registrationBean.addUrlMappings("/hello");
    registrationBean.setServlet(new HelloServlet());
    return registrationBean;
}

@Bean
public FilterRegistrationBean filterRegistrationBean() {
    FilterRegistrationBean registrationBean = new FilterRegistrationBean();
    registrationBean.setFilter(new HelloFilter());
    registrationBean.addUrlPatterns("/hello/*");
    return registrationBean;
}
```

ServletRegistrationBean和FilterRegistrationBean都继承自RegistrationBean，RegistrationBean是Spring Boot中广泛应用的一个注册类，负责把Servlet、Filter、Listener给容器化，使他们被Spring托管，并且完成自身对Web容器的注册。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-19.jpg)

从图中可以看出RegistrationBean的地位，它的几个实现类作用分别是：帮助容器注册Filter、Servlet、Listener。另外RegistrationBean实现了ServletContextInitializer接口，这个接口将会是下面分析的核心接口，先了解一下，RegistrationBean是它的抽象实现。

### 二、Servlet、Filter、Listener的加载流程

前提：分析的Spring Boot版本是1.5.3.RELEASE，典型的Spring Boot Web项目，使用Tomcat嵌入式容器。

在Spring Boot应用启动过程中，由于Spring Boot自动配置机制的存在，对于Web模块，以下自动配置类将会生效(按照被容器处理的顺序排序)：EmbeddedServletContainerAutoConfiguration、DispatcherServletAutoConfiguration和WebMvcAutoConfiguration。

- EmbeddedServletContainerAutoConfiguration用于注册一个TomcatEmbeddedServletContainerFactory Bean:

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication
@Import(BeanPostProcessorsRegistrar.class)
public class EmbeddedServletContainerAutoConfiguration {

   @Configuration
   @ConditionalOnClass({ Servlet.class, Tomcat.class })
   @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
   public static class EmbeddedTomcat {

      @Bean
      public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
         return new TomcatEmbeddedServletContainerFactory(); // 注册Bean
      }

   }
}  
```

- DispatcherServletAutoConfiguration内嵌两个配置类DispatcherServletConfiguration和DispatcherServletRegistrationConfiguration，前者用于配置DispatcherServlet Bean，后者用于注册DispatcherServlet。

```java
// DispatcherServletConfiguration
@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public DispatcherServlet dispatcherServlet() {
   DispatcherServlet dispatcherServlet = new DispatcherServlet();
   dispatcherServlet.setDispatchOptionsRequest(
         this.webMvcProperties.isDispatchOptionsRequest());
   dispatcherServlet.setDispatchTraceRequest(
         this.webMvcProperties.isDispatchTraceRequest());
   dispatcherServlet.setThrowExceptionIfNoHandlerFound(
         this.webMvcProperties.isThrowExceptionIfNoHandlerFound());
   return dispatcherServlet;
}
// DispatcherServletRegistrationConfiguration
@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public ServletRegistrationBean dispatcherServletRegistration(
		DispatcherServlet dispatcherServlet) {
	ServletRegistrationBean registration = new ServletRegistrationBean(
			dispatcherServlet, this.serverProperties.getServletMapping());
	registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
	registration.setLoadOnStartup(
			this.webMvcProperties.getServlet().getLoadOnStartup()); // -1	
	if (this.multipartConfig != null) {
		registration.setMultipartConfig(this.multipartConfig);
	}
	return registration;
}
```

可以看到**DispatcherServlet的注册**是通过**ServletRegistrationBean**的方式完成的。

- WebMvcAutoConfiguration内嵌EnableWebMvcConfiguration配置类，用于注册HandlerMapping、HandleAdapter等Web组件。

```java
@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
   RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter();
   adapter.setIgnoreDefaultModelOnRedirect(this.mvcProperties == null ? true
         : this.mvcProperties.isIgnoreDefaultModelOnRedirect());
   return adapter;
}
@Bean
@Primary
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
	// Must be @Primary for MvcUriComponentsBuilder to work
	return super.requestMappingHandlerMapping();
}
```

Spring Boot应用启动过程中使用的是AnnotationConfigEmbeddedWebApplicationContext，继承自EmbeddedWebApplicationContext。在AnnotationConfigEmbeddedWebApplicationContext刷新过程中，会调用EmbeddedWebApplicationContext.onRefresh方法:

```java
protected void onRefresh() {
   super.onRefresh();
   try {
      createEmbeddedServletContainer();
   }
   catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start embedded container",
            ex);
   }
}
private void createEmbeddedServletContainer() {
	EmbeddedServletContainer localContainer = this.embeddedServletContainer;
	ServletContext localServletContext = getServletContext();
	if (localContainer == null && localServletContext == null) {
		EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
		this.embeddedServletContainer = containerFactory
				.getEmbeddedServletContainer(getSelfInitializer()); // 重点
	}
	else if (localServletContext != null) {
		try {
			getSelfInitializer().onStartup(localServletContext);
		}
		catch (ServletException ex) {
			throw new ApplicationContextException("Cannot initialize servlet context",
					ex);
		}
	}
	initPropertySources();
}
```

在createEmbeddedServletContainer方法中调用TomcatEmbeddedServletContainerFactory.getEmbeddedServletContainer方法创建EmbeddedServletContainer。在getEmbeddedServletContainer调用时传入了**getSelfInitializer()**的返回值。getSelfInitializer()方法如下：

```java
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
   return new ServletContextInitializer() {
      @Override
      public void onStartup(ServletContext servletContext) throws ServletException {
         selfInitialize(servletContext);
      }
   };
}
```

getSelfInitializer()方法返回了一个**匿名的ServletContextInitializer实现**。该ServletContextInitializer实现是注册Servlet、Filter、Listener的基础。

TomcatEmbeddedServletContainerFactory.getEmbeddedServletContainer方法创建EmbeddedServletContainer的过程如下：

```java
public EmbeddedServletContainer getEmbeddedServletContainer(
      ServletContextInitializer... initializers) {
   // 前面代码省略
   prepareContext(tomcat.getHost(), initializers); // 1
   return getTomcatEmbeddedServletContainer(tomcat); // 2
}
```

在getEmbeddedServletContainer方法中先看1处的代码实现：

```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
   // 前面省略
  
   // 合并ServletContextInitializer
   ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
   configureContext(context, initializersToUse); 
	 // ...
}
```

```java
protected void configureContext(Context context,
      ServletContextInitializer[] initializers) {
   TomcatStarter starter = new TomcatStarter(initializers); // 创建TomcatStarter
   context.addServletContainerInitializer(starter, NO_CLASSES); // 添加到Context
   // ...
}
```

在prepareContext中调用了configureContext方法，在configureContext方法中创建了TomcatStarter实例，并在构造器中传入了上面匿名的ServletContextInitializer实现，并最终添加到了Tomcat Context，因此Web容器启动过程中会回调其onStartup方法。下面看TomcatStarter的实现(实现了ServletContainerInitializer接口)：

```java
class TomcatStarter implements ServletContainerInitializer {

   private final ServletContextInitializer[] initializers;

   TomcatStarter(ServletContextInitializer[] initializers) {
      this.initializers = initializers;
   }

   @Override
   public void onStartup(Set<Class<?>> classes, ServletContext servletContext)
         throws ServletException {
      try {
         for (ServletContextInitializer initializer : this.initializers) {
            initializer.onStartup(servletContext);
         }
      }
      catch (Exception ex) {
				// ...
   }

}
```

可以看到TomcatStarter所做的事情是在它的onStartup方法被回调时，依次调用传入的ServletContextInitializer的onStartup方法。那TomcatStarter的onStartup方法是什么时候被回调的呢？回到getEmbeddedServletContainer方法处：

```java
public EmbeddedServletContainer getEmbeddedServletContainer(
      ServletContextInitializer... initializers) {
   // 前面代码省略
   prepareContext(tomcat.getHost(), initializers); // 1
   return getTomcatEmbeddedServletContainer(tomcat); // 2
}
```

下面分析getEmbeddedServletContainer方法中代码2处的逻辑：

```java
protected TomcatEmbeddedServletContainer getTomcatEmbeddedServletContainer(
      Tomcat tomcat) {
   return new TomcatEmbeddedServletContainer(tomcat, getPort() >= 0);
}
public TomcatEmbeddedServletContainer(Tomcat tomcat, boolean autoStart) {
	Assert.notNull(tomcat, "Tomcat Server must not be null");
	this.tomcat = tomcat;
	this.autoStart = autoStart;
	initialize(); // 启动Tomcat Web容器
}
private void initialize() throws EmbeddedServletContainerException {
	TomcatEmbeddedServletContainer.logger
			.info("Tomcat initialized with port(s): " + getPortsDescription(false));
	synchronized (this.monitor) {
		try {
			addInstanceIdToEngineName();
			try {
				// ...
				// Start the server to trigger initialization listeners
				this.tomcat.start(); // 启动Tomcat Server
				// ...
			}
			catch (Exception ex) {
				containerCounter.decrementAndGet();
				throw ex;
			}
		}
		catch (Exception ex) {
			throw new EmbeddedServletContainerException(
					"Unable to start embedded Tomcat", ex);
		}
	}
}
```

可以看到TomcatEmbeddedServletContainer在初始化时调用了initialize()方法，并启动了Tomcat Server，来回调触发ServletContainerInitializer的onStartup方法。于是TomcatStarter中传入的ServletContextInitializer的onStartup方法得以依次回调(**Tomcat线程，非主线程**)。下面就来看一下上文中getSelfInitializer()返回的ServletContextInitializer的onStartup方法的逻辑。

```java
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
   return new ServletContextInitializer() {
      @Override
      public void onStartup(ServletContext servletContext) throws ServletException {
         selfInitialize(servletContext);
      }
   };
}
private void selfInitialize(ServletContext servletContext) throws ServletException {
	prepareEmbeddedWebApplicationContext(servletContext);
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(
			beanFactory);
	WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,
			getServletContext());
	existingScopes.restore();
	WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,
			getServletContext());
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
		beans.onStartup(servletContext); // 重点
	}
}
```

getSelfInitializer方法调用了selfInitialize方法，selfInitialize方法首先调用prepareEmbeddedWebApplicationContext方法，用于向ServletContext保存ROOT应用上下文的引用，即当前的AnnotationConfigEmbeddedWebApplicationContext实例。然后注册了web应用的Scope和环境Bean。下面的for循环是重点：

```java
for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
   beans.onStartup(servletContext);
}
```

Servlet、Filter、Listener的注册就发生在这里。首先看下getServletContextInitializerBeans()获取ServletContextInitializer Beans的逻辑：

```java
// Returns {@link ServletContextInitializer}s that should be used with the embedded
// Servlet context. By default this method will first attempt to find
// {@link ServletContextInitializer}, {@link Servlet}, {@link Filter} and certain
// {@link EventListener} beans.
// @return the servlet initializer beans
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
   return new ServletContextInitializerBeans(getBeanFactory());
}
public ServletContextInitializerBeans(ListableBeanFactory beanFactory) {
	this.initializers = new LinkedMultiValueMap<Class<?>, ServletContextInitializer>();
	addServletContextInitializerBeans(beanFactory); // 从容器获取ServletContextInitializer Beans
	addAdaptableBeans(beanFactory);
	List<ServletContextInitializer> sortedInitializers = new ArrayList<ServletContextInitializer>();
	for (Map.Entry<?, List<ServletContextInitializer>> entry : this.initializers
			.entrySet()) {
		AnnotationAwareOrderComparator.sort(entry.getValue()); // 排序
		sortedInitializers.addAll(entry.getValue());
	}
	this.sortedList = Collections.unmodifiableList(sortedInitializers);
}
private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
	for (Entry<String, ServletContextInitializer> initializerBean : getOrderedBeansOfType(
			beanFactory, ServletContextInitializer.class)) {
		addServletContextInitializerBean(initializerBean.getKey(),
				initializerBean.getValue(), beanFactory);
	}
}
private void addServletContextInitializerBean(String beanName,
		ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
	if (initializer instanceof ServletRegistrationBean) { // Servlet注册
		Servlet source = ((ServletRegistrationBean) initializer).getServlet();
		addServletContextInitializerBean(Servlet.class, beanName, initializer,
				beanFactory, source);
	}
	else if (initializer instanceof FilterRegistrationBean) { // Filter注册
		Filter source = ((FilterRegistrationBean) initializer).getFilter();
		addServletContextInitializerBean(Filter.class, beanName, initializer,
				beanFactory, source);
	}
	else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
		String source = ((DelegatingFilterProxyRegistrationBean) initializer)
				.getTargetBeanName();
		addServletContextInitializerBean(Filter.class, beanName, initializer,
				beanFactory, source);
	}
	else if (initializer instanceof ServletListenerRegistrationBean) { // Listener注册
		EventListener source = ((ServletListenerRegistrationBean<?>) initializer)
				.getListener();
		addServletContextInitializerBean(EventListener.class, beanName, initializer,
				beanFactory, source);
	}
	else {
		addServletContextInitializerBean(ServletContextInitializer.class, beanName,
				initializer, beanFactory, initializer);
	}
}
```

getServletContextInitializerBeans()方法主要从Spring容器中获取ServletContextInitializer Bean并排序。因此，第二部分开头介绍的**DispatcherServlet ServletRegistrationBean**也是在此时从Spring容器中获取的。getServletContextInitializerBeans()方法得到的为ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean等实例，均实现了ServletContextInitializer接口。因此，在上面的for循环中完成了Servlet、Filter、Listener的注册。

```java
for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
   beans.onStartup(servletContext); // Servlet、Filter、Listener注册
}
```

可以看到，Spring Boot中只要依赖于ServletContextInitializer接口，就可以实现Servlet、Filter、Listener的注册。

##### 1.Servlet 3.0注解+@ServletComponentScan注册方式的解释

```java
@Import(ServletComponentScanRegistrar.class)
public @interface ServletComponentScan {}
```

当@ServletComponentScan注解配置在@Configuration类上时，导入了ServletComponentScanRegistrar类。该类的左右是向Spring容器中注册ServletComponentRegisteringPostProcessor BeanDefinition。ServletComponentRegisteringPostProcessor是一个BeanFactoryPostProcessor，主要用于扫描类路径下注解了Servlet 3.0 @Web*注解的类。下面看下ServletComponentRegisteringPostProcessor的postProcessBeanFactory方法的逻辑：

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
      throws BeansException {
   if (isRunningInEmbeddedContainer()) {
      ClassPathScanningCandidateComponentProvider componentProvider = createComponentProvider(); // 组件扫描
      for (String packageToScan : this.packagesToScan) {
         scanPackage(componentProvider, packageToScan);
      }
   }
}
private void scanPackage(
		ClassPathScanningCandidateComponentProvider componentProvider,
		String packageToScan) {
	for (BeanDefinition candidate : componentProvider
			.findCandidateComponents(packageToScan)) {
		if (candidate instanceof ScannedGenericBeanDefinition) {
			for (ServletComponentHandler handler : HANDLERS) {
				handler.handle(((ScannedGenericBeanDefinition) candidate),
						(BeanDefinitionRegistry) this.applicationContext);
			}
		}
	}
}
```

scanPackage方法中的HANDLERS变量包含WebServletHandler、WebFilterHandler、WebListenerHandler实例，分别处理@WebServlet、@WebFilter、@WebListener注解，将对应注解的类创建成ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean实例，注册到Spring容器。因此，应用启动过程中能够注册将Servlet、Filter、Listener注册到Tomcat Server Web容器中并生效，由于ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean都实现了ServletContextInitializer接口，因此其核心还是ServletContextInitializer接口。

##### 2.RegistrationBean的解释

根据上面的分析，由于ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean都实现了ServletContextInitializer接口，因此直接定义ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean的Spring Bean配置，也能将Servlet、Filter、Listener注册到Tomcat Server Web容器中并生效。

##### 3.第三种注册方式

根据上面的分析，其实可以直接定义ServletContextInitializer接口的Spring Bean，也能将Servlet、Filter、Listener注册到Tomcat Server Web容器中并生效。如下所示：

```java
@Component
public class MyServletContextInitializer implements ServletContextInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {

        System.out.println("创建 HelloServlet...");
        ServletRegistration.Dynamic servletRegistration =
                servletContext.addServlet(HelloServlet.class.getSimpleName(), HelloServlet.class);
        servletRegistration.addMapping("/hello");

        System.out.println("创建 HelloFilter...");
        FilterRegistration.Dynamic filterRegistration =
                servletContext.addFilter(HelloFilter.class.getSimpleName(), HelloFilter.class);

        EnumSet<DispatcherType> set = EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD);
        filterRegistration.addMappingForUrlPatterns(set, true, "/hello/*");

    }
}
```

上述代码的作用与前两种注册方式原理一致，不过使用上还是前两种注册方式比较常用。

### 三、DispatcherServlet的细节(重要)

在Spring MVC中，DispatcherServlet默认情况下会在容器启动时直接初始化，调用其init方法。而在Spring Boot中，DispatcherServlet在其注册到ServletContext时初始化，默认情况下会在第一次HTTP请求到来时调用其init方法(**loadOnStartup = -1**)。init方法调用时的不同点如下：

```java
// FrameworkServlet
protected WebApplicationContext initWebApplicationContext() {
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   WebApplicationContext wac = null;

   if (this.webApplicationContext != null) {
      wac = this.webApplicationContext;
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         if (!cwac.isActive()) {
            if (cwac.getParent() == null) {
               cwac.setParent(rootContext);
            }
            configureAndRefreshWebApplicationContext(cwac);
         }
      }
   }
   if (wac == null) {
      wac = findWebApplicationContext();
   }
   if (wac == null) {
      wac = createWebApplicationContext(rootContext);
   }

   if (!this.refreshEventReceived) {
      onRefresh(wac);
   }

   if (this.publishContext) {
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
      if (this.logger.isDebugEnabled()) {
         this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
               "' as ServletContext attribute with name [" + attrName + "]");
      }
   }

   return wac;
}
```

在Spring Boot中，DispatcherServlet.init方法调用时应用已启动完成，DispatcherServlet实例已由Spring容器初始化(自动配置机制)并设置ApplicationContext，即AnnotationConfigEmbeddedWebApplicationContext。

```java
// FrameworkServlet
public void setApplicationContext(ApplicationContext applicationContext) {
	if (this.webApplicationContext == null && applicationContext instanceof WebApplicationContext) {
		this.webApplicationContext = (WebApplicationContext) applicationContext;
		this.webApplicationContextInjected = true;
	}
}
```

因此在DispatcherServlet.init方法调用时，接着调用父类FrameworkServlet的initWebApplicationContext()方法，其`this.webApplicationContext != null(wac != null)`。而Spring MVC中，initWebApplicationContext()调用时`this.webApplicationContext == null`，因此**Spring MVC会创建两个Web应用上下文；而Spring Boot中只创建了一个Web应用上下文**，即AnnotationConfigEmbeddedWebApplicationContext。

### 四、总结

根据前面的介绍，在Spring Boot中，通过定义一个TomcatStarter(ServletContainerInitializer实现，Servlet 3.0 API)，将Tomcat Web容器与Spring容器(AnnotationConfigEmbeddedWebApplicationContext)挂钩。Spring Boot提供ServletContextInitializer接口，开发者间接或者直接定义ServletContextInitializer接口的实现BeanDefinition，由Spring容器管理，然后由TomcatStarter触发这些ServletContextInitializer的onStartup方法，实现Servlet、Filter、Listener到ServletContext的注册。

下面是总结的Spring MVC与Spring Boot用到的Servlet API与自定义API的区别：

```java
Spring MVC:
	ServletContextListener  Servlet API --> 用户
	ServletContainerInitializer Servlet API --> 框架: SpringServletContainerInitializer(SPI)
		WebApplicationInitializer Spring API --> 用户

Spring Boot:
	ServletContainerInitializer Servlet API --> TomcatStarter
		ServletContextInitializer Spring Boot API -> 用户
```

### 参考文章

- [Spring揭秘--寻找遗失的web.xml](https://www.cnkirito.moe/servlet-explore/)

