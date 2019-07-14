---
layout:     post
title:      Spring IoC容器源码分析
subtitle:   IoC/AbstractApplicationContext.refresh()
date:       2019-07-14
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring IoC容器源码分析

本文以**5.1.4.RELEASE**版本为源码版本，详细分析Spring Framework中第一个重要的概念——IoC以及IoC容器(另一个是AOP)。本文所述内容是基于xml的配置，这样相对简单，利于理解源码。基于Java的配置和组件扫描、自动配置的内容，后续在分析。

Spring IoC容器的创建过程实际分为2步，第一步是创建Bean容器(BeanFactory)，第二步是初始化非懒加载的单例Bean。

[TOC]

## 一、Spring IoC容器使用实例

先看下最基本的启动并使用Spring容器的例子:

POJO：

```java
// 用户模型
public class User {

   @Autowired
   private Address address;

    // id
    private int id;

    // 用户名
    private String name;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

   public Address getAddress() {
      return address;
   }

   public void setAddress(Address address) {
      this.address = address;
   }

   @Override
   public String toString() {
      return "User{" +
            "address=" + address +
            ", id=" + id +
            ", name='" + name + '\'' +
            '}';
   }

}

public class Address {

	private String id;
	private String mail;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getMail() {
		return mail;
	}

	public void setMail(String mail) {
		this.mail = mail;
	}
}
```

Bean配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd">

  <bean id="user" class="instantiate.defaultconstruct.User" >
		<property name="name" value="xuan"/>
		<property name="id" value="1"/>
	</bean>

	<bean id="address" class="instantiate.defaultconstruct.Address"/>

  <context:component-scan base-package="instantiate.defaultconstruct"/>

</beans>
```

测试类：

```java
public class UserTest {
   public static void main(String[] args) {
      ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("defaultconstruct/user.xml");
      // context.addBeanFactoryPostProcessor();
      // context.getBeanFactory().addBeanPostProcessor();
      User user = (User) context.getBean("user");
      System.out.println(user.toString());
   }
}

// 输出：
User{address=instantiate.defaultconstruct.Address@57d5872c, id=1, name='xuan'}
```

以上代码利用xml配置文件启动了一个Spring容器。`ApplicationContext context = new ClassPathXmlApplicationContext(…)`是在ClassPath中寻找xml配置文件，根据xml文件的内容来构建 ApplicationContext。下面先看下ApplicatioContext的继承体系：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/ApplicationContext%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB.png)



可以看到，ClassPathXmlApplicationContext兜兜转转了好久才到ApplicationContext接口，同样的，也可以使用绿颜色的**FileSystemXmlApplicationContext**和**AnnotationConfigApplicationContext**这两个类来引导IoC容器。

- **FileSystemXmlApplicationContext**的构造函数需要一个xml配置文件在系统中的路径，其他和 ClassPathXmlApplicationContext基本上一样。
- **AnnotationConfigApplicationContext**是基于注解来使用的，它不需要配置文件，采用Java配置类和各种注解来配置，是比较简单的方式。

本文旨在理解整个IoC容器的构建流程，所以使用ClassPathXmlApplicationContext进行分析。上面给出的例子比较简单，但足够说明本文的主题——启动IoC容器(ApplicationContext)的过程(IoC的核心)。ApplicationContext 启动过程中，会负责创建Bean实例，往各个Bean实例中注入依赖等。

## 二、BeanFactory简介

BeanFactory作为Bean容器，负责实例化和管理各个bean实例。前面说的ApplicationContext其实就是一个 BeanFactory，因为它继承自BeanFactory。下面看下和BeanFactory接口相关的类的主要继承体系：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/BeanFactory%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB.png)

ApplicationContext的继承体系前面已经介绍，下面根据BeanFactory的继承体系，着重介绍几点注意点。

- ApplicationContext继承ListableBeanFactory，这个Listable的意思是，通过这个接口，可以获取多个 Bean，看源码会发现，**最顶层BeanFactory接口的方法都是获取单个Bean的**。

- ApplicationContext继承HierarchicalBeanFactory，Hierarchical单词本身已经能说明问题了，也就是说可以在应用中起多个BeanFactory，然后可以将各个BeanFactory设置为父子关系。
- AutowireCapableBeanFactory，它是用来自动装配Bean用的，但是仔细看上图，ApplicationContext并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果看到ApplicationContext接口定义中的最后一个方法getAutowireCapableBeanFactory()就知道了。
- ConfigurableListableBeanFactory也是一个特殊的接口，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext没有。这点之后会用到。

## 三、IoC容器启动过程分析

首先从ClassPathXmlApplicationContext的构造方法开始。

```java
// ClassPathXmlApplicationContext类

public ClassPathXmlApplicationContext() {
}

// 如果已经有ApplicationContext并需要配置成父子关系，那么调用这个构造方法
public ClassPathXmlApplicationContext(ApplicationContext parent) {
	super(parent);
}

// 传入xml配置文件路径
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
	this(new String[] {configLocation}, true, null);
}

public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {

	super(parent);
  // 解析配置文件路径，替换占位符，并设置到String[] configLocations(配置文件数组)
	setConfigLocations(configLocations); 
	if (refresh) {
		refresh(); // 刷新上下文(核心方法)
	}
}
```

接下来是refresh()方法，这里简单说下为什么是refresh()，而不是init()这种名字的方法。因为 ApplicationContext建立起来以后，其实是可以通过调用refresh()这个方法重建的，refresh()会将原来的 ApplicationContext销毁，然后再重新执行一次初始化操作。**refresh()方法是IoC容器启动的核心方法，所有的逻辑都在这里实现。**

```java
// AbstractApplicationContext类
// 刷新上下文 
public void refresh() throws BeansException, IllegalStateException {
   // 获取监视器锁，防止refresh过程中再次启动或销毁容器
   synchronized (this.startupShutdownMonitor) {
      // 1.Prepare this context for refreshing. 准备工作，记录下容器的启动时间、标记“已启动”状态等
      prepareRefresh();

      // 2.Tell the subclass to refresh the internal bean factory. 刷新BeanFactory，加载BeanDefinition(重点)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 3.Prepare the bean factory for use in this context.
      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      prepareBeanFactory(beanFactory);

      try {
         // 4.Allows post-processing of the bean factory in context subclasses.
         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);

         // 5.Invoke factory processors registered as beans in the context.
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 回调方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // 6.Register bean processors that intercept bean creation.
         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。这里仅仅是注册，之后会看到回调这两方法的时机
         registerBeanPostProcessors(beanFactory);

         // 7.Initialize message source for this context. 初始化当前 ApplicationContext 的 MessageSource Bean(国际化)
         initMessageSource();

         // 8.Initialize event multicaster for this context. 初始化当前 ApplicationContext 的事件广播器
         initApplicationEventMulticaster();

         // 9.Initialize other special beans in specific context subclasses. 典型的模板方法(钩子方法)
         onRefresh(); // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）

         // 10.Check for listener beans and register them.
         // 注册应用事件监听器，监听器需要实现 ApplicationListener 接口
         registerListeners();

         // 11.Instantiate all remaining (non-lazy-init) singletons.
         // 初始化所有非懒加载的 singleton beans
         finishBeanFactoryInitialization(beanFactory);

         // 12.Last step: publish corresponding event. 最后，广播事件，ApplicationContext 初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex); // 重置active标记

         // Propagate exception to caller. 把异常往外抛
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches(); // 清空公共的缓存
      }
   }
}
```

可以看出，refresh()方法核心流程一共分为12步，下面一步步分析。

### 1.创建Bean容器前的准备工作

```java
// AbstractApplicationContext类
protected void prepareRefresh() {
   this.startupDate = System.currentTimeMillis(); // 记录容器启动时间
   // 将active属性设置为true，closed属性设置为false，它们都是AtomicBoolean类型
   this.closed.set(false);
   this.active.set(true);

   if (logger.isDebugEnabled()) {
      if (logger.isTraceEnabled()) {
         logger.trace("Refreshing " + this);
      }
      else {
         logger.debug("Refreshing " + getDisplayName());
      }
   }

   // Initialize any placeholder property sources in the context environment
   // (子类实现)初始化属性源，比如web环境: ServletContext ServletConfig
   initPropertySources();

   // Validate that all properties marked as required are resolvable
   // see ConfigurablePropertyResolver#setRequiredProperties
   // 验证所有设置为Required的属性必须可解析为具体的值
   getEnvironment().validateRequiredProperties();

   // 允许在multicaster可用时发布early事件
   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### 2.创建Bean容器，加载并注册BeanDefinitions到Bean容器

第二步是obtainFreshBeanFactory()。这个方法是全文最重要的部分之一，这里将会初始化BeanFactory、加载BeanDefinition、注册BeanDefinition等等。这步结束后，Bean并没有完成初始化。这里指的是Bean实例并未在这一步生成。

```java
// AbstractApplicationContext类
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 关闭现有BeanFactory，创建新的BeanFactory，加载并注册BeanDefinition
   refreshBeanFactory();
   return getBeanFactory();
}


```

```java
// AbstractRefreshableApplicationContext类
// 关闭现有BeanFactory，创建新的BeanFactory，加载并注册BeanDefinition
protected final void refreshBeanFactory() throws BeansException {
   // 如果ApplicationContext中已经加载过BeanFactory了，销毁所有单例Bean，关闭BeanFactory。
   // 注意，应用中BeanFactory本来就是可以多个的，这里可不是说应用全局是否有BeanFactory，而是当前
   // ApplicationContext是否有BeanFactory
   if (hasBeanFactory()) {
      destroyBeans(); // 销毁所有单例bean
      closeBeanFactory(); // 关闭BeanFactory
   }
   try {
      DefaultListableBeanFactory beanFactory = createBeanFactory(); // 创建DefaultListableBeanFactory实例并设置序列化id
      beanFactory.setSerializationId(getId());
      customizeBeanFactory(beanFactory); // 设置是否允许BeanDefinition覆盖和循环引用
      loadBeanDefinitions(beanFactory); // 加载BeanDefinition
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}

// 返回刚刚创建的 BeanFactory
public final ConfigurableListableBeanFactory getBeanFactory() {
	synchronized (this.beanFactoryMonitor) {
		if (this.beanFactory == null) {
			throw new IllegalStateException("BeanFactory not initialized or already closed - " +
					"call 'refresh' before accessing beans via the ApplicationContext");
		}
		return this.beanFactory;
	}
}
```

> 注意：
>
> ApplicationContext继承自BeanFactory，但是它不应该被理解为BeanFactory的实现类，而是说其内部持有一个实例化的BeanFactory(DefaultListableBeanFactory)。以后所有的BeanFactory相关的操作其实是委托给这个实例来处理的。

为什么选择实例化**DefaultListableBeanFactory** ？前面说了有个很重要的接口 ConfigurableListableBeanFactory，它实现了BeanFactory下面一层的所有三个接口。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/BeanFactory%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB.png)



可以看到ConfigurableListableBeanFactory只有一个实现类DefaultListableBeanFactory，而且实现类 DefaultListableBeanFactory 还通过实现右边的AbstractAutowireCapableBeanFactory通吃了右路。所以结论就是，最底下这个DefaultListableBeanFactory基本上是最牛的BeanFactory了，这也是为什么这边会使用这个类来实例化的原因。

> 如果想要在程序运行的时候动态往Spring IOC容器注册新的bean，就会使用到这个类。那怎么在运行时获得这个实例呢？
>
> 之前我们说过ApplicationContext接口能获取到AutowireCapableBeanFactory，然后它向下转型就能得到DefaultListableBeanFactory了。

在继续往下之前，需要先了解BeanDefinition。**我们说BeanFactory是Bean容器，那么Bean又是什么？**

这里的BeanDefinition就是所说的Spring的Bean，我们自己定义的各个Bean其实会转换成一个个BeanDefinition存在于Spring的BeanFactory中。

> BeanDefinition中保存了Bean的信息，比如Bean指向的是哪个类、是否是单例的、是否懒加载、这个Bean 依赖了哪些Bean等等。

#### (a) BeanDefinition接口定义

下面看下BeanDefinition接口的定义：

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

   // 可以看到，默认只提供singleton和prototype两种
   // 还有request,session,globalSession,application,websocket这几种，它们属于基于web的扩展。
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

   // 比较不重要，直接跳过
   int ROLE_APPLICATION = 0;
   int ROLE_SUPPORT = 1;
   int ROLE_INFRASTRUCTURE = 2;

   // 设置父BeanDefinition，这里涉及到BeanDefinition继承，不是Java继承。
   void setParentName(String parentName);

   // 获取父BeanDefinition的名称
   String getParentName();

   // 设置Bean的类名称，将来是要通过反射来生成实例的
   void setBeanClassName(String beanClassName);

   // 获取Bean的类名称
   String getBeanClassName();


   // 设置bean的scope
   void setScope(String scope);

   String getScope();

   // 设置是否懒加载
   void setLazyInit(boolean lazyInit);

   boolean isLazyInit();

   // 设置该Bean依赖的所有Bean，注意，这里的依赖不是指属性依赖(如@Autowire标记的)，
   // 是depends-on=""属性设置的值。
   void setDependsOn(String... dependsOn);

   // 返回该Bean的所有依赖
   String[] getDependsOn();

   // 设置该Bean是否可以注入到其他Bean中，只对根据类型注入有效，
   // 如果根据名称注入，即使这边设置了 false，也是可以的
   void setAutowireCandidate(boolean autowireCandidate);

   // 该Bean是否可以注入到其他Bean中
   boolean isAutowireCandidate();

   // 主要的。同一接口的多个实现，如果不指定名字的话，Spring会优先选择设置primary为true的bean
   void setPrimary(boolean primary);

   // 是否是primary的
   boolean isPrimary();

   // 如果该Bean采用工厂方法生成，指定工厂bean的bean name。
   void setFactoryBeanName(String factoryBeanName);
   // 获取工厂bean的bean name
   String getFactoryBeanName();
   // 指定工厂方法名称
   void setFactoryMethodName(String factoryMethodName);
   // 获取工厂方法名称
   String getFactoryMethodName();

   // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();

   // Bean中的属性值，后面给bean注入属性值的时候会用到
   MutablePropertyValues getPropertyValues();

   // 是否singleton
   boolean isSingleton();

   // 是否prototype
   boolean isPrototype();

   // 如果这个Bean是被设置为abstract，那么不能实例化，
   // 常用于作为父bean用于继承。
   boolean isAbstract();

   int getRole();
   String getDescription();
   String getResourceDescription();
   BeanDefinition getOriginatingBeanDefinition();
}
```

有了BeanDefinition的概念以后，再往下看refreshBeanFactory()方法中的剩余部分：

```java
customizeBeanFactory(beanFactory);
loadBeanDefinitions(beanFactory);
```

#### (b) customizeBeanFactory

```java
// AbstractRefreshableApplicationContext类
// 自定义BeanFactory的属性
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
   // 设置是否允许BeanDefinition覆盖
   if (this.allowBeanDefinitionOverriding != null) {
      beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   // 设置是否允许bean之间的循环引用
   if (this.allowCircularReferences != null) {
      beanFactory.setAllowCircularReferences(this.allowCircularReferences);
   }
}
```

- BeanDefinition的覆盖：在配置文件中定义bean时使用了相同的id或name，默认情况下，allowBeanDefinitionOverriding属性为null，如果在同一配置文件中重复了，会抛出异常，但是如果不是同一配置文件中，会发生覆盖。
- bean的循环引用：如A 依赖 B，而 B 依赖 A。或 A 依赖 B，B 依赖 C，而 C 依赖 A。默认情况下，Spring允许循环依赖。

#### (c) 加载BeanDefinition

接下来是重要的loadBeanDefinitions(beanFactory)方法，这个方法将根据配置，加载各个BeanDefinition，然后注册到BeanFactory中。读取配置的操作在XmlBeanDefinitionReader中，其负责加载xml配置、解析。

```java
// AbstractXmlApplicationContext类
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // Create a new XmlBeanDefinitionReader for the given BeanFactory. // 创建XmlBeanDefinitionReader
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment. // 配置XmlBeanDefinitionReader
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
   loadBeanDefinitions(beanDefinitionReader); // 加载BeanDefinition
}
```

现在还在这个类中，接下来用刚刚初始化的XmlBeanDefinitionReader来开始加载xml配置。

```java
// AbstractXmlApplicationContext类
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
   Resource[] configResources = getConfigResources();
   if (configResources != null) {
      // Resource[]
      reader.loadBeanDefinitions(configResources);
   }
   String[] configLocations = getConfigLocations();
   if (configLocations != null) {
      // String[] 这里继续
      reader.loadBeanDefinitions(configLocations);
   }
}
// AbstractBeanDefinitionReader类
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int count = 0;
	for (String location : locations) {
		count += loadBeanDefinitions(location); // 分别加载、解析各个配置文件
	}
	return count;
}
// AbstractBeanDefinitionReader类
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}
// AbstractBeanDefinitionReader类
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
	ResourceLoader resourceLoader = getResourceLoader(); // 资源加载器
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException(
				"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
	}
	// resourceLoader是ApplicationContext实例，下面条件成立
	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location); // 解析location
			int count = loadBeanDefinitions(resources); // 加载BeanDefinitions
			if (actualResources != null) {
				Collections.addAll(actualResources, resources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
			}
			return count;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		Resource resource = resourceLoader.getResource(location);
		int count = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
		}
		return count;
	}
}
// XmlBeanDefinitionReader类
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
	Assert.notNull(resources, "Resource array must not be null");
	int count = 0;
	for (Resource resource : resources) {
    // 注意这里是个for循环，也就是每个文件是一个Resource
		count += loadBeanDefinitions(resource); // 这里继续
	}
  // 最后返回count，表示总共加载了多少BeanDefinition
	return count;
}
// XmlBeanDefinitionReader类
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(new EncodedResource(resource));
}
// XmlBeanDefinitionReader类
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isTraceEnabled()) {
		logger.trace("Loading XML bean definitions from " + encodedResource);
	}
	// ThreadLocal resourcesCurrentlyBeingLoaded保存当前正在加载的资源文件
	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {
		throw new BeanDefinitionStoreException(
				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	}
	try {
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 核心部分
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			inputStream.close();
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(
				"IOException parsing XML document from " + encodedResource.getResource(), ex);
	}
	finally {
		currentResources.remove(encodedResource);
		if (currentResources.isEmpty()) {
			this.resourcesCurrentlyBeingLoaded.remove();
		}
	}
}
```

```java
// XmlBeanDefinitionReader类
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {

   try {
      // 将xml文件转为Document对象
      Document doc = doLoadDocument(inputSource, resource);
      // 从Document读取BeanDefinition并注册到BeanDefinitionRegistry
      int count = registerBeanDefinitions(doc, resource);
      if (logger.isDebugEnabled()) {
         logger.debug("Loaded " + count + " bean definitions from " + resource);
      }
      return count;
   }
   catch (BeanDefinitionStoreException ex) {
      throw ex;
   }
   catch (SAXParseException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
   }
   catch (SAXException ex) {
      throw new XmlBeanDefinitionStoreException(resource.getDescription(),
            "XML document from " + resource + " is invalid", ex);
   }
   catch (ParserConfigurationException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Parser configuration exception parsing XML from " + resource, ex);
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "IOException parsing XML document from " + resource, ex);
   }
   catch (Throwable ex) {
      throw new BeanDefinitionStoreException(resource.getDescription(),
            "Unexpected exception parsing XML document from " + resource, ex);
   }
}
// XmlBeanDefinitionReader类
// 返回值：返回从当前配置文件加载了多少数量的BeanDefinition
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	// DefaultBeanDefinitionDocumentReader实例
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	// 当前的BeanDefinition数
	int countBefore = getRegistry().getBeanDefinitionCount();
	// 从Document读取BeanDefinition并注册到BeanDefinitionRegistry
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore; // 返回解析的个数
}
// DefaultBeanDefinitionDocumentReader类
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
	// 从xml根节点开始解析文件
	doRegisterBeanDefinitions(doc.getDocumentElement());
}

```

经过漫长的链路，一个配置文件终于转换为一颗DOM树了，注意，这里指的是其中一个配置文件，不是所有的，可以看到上面有个for循环的。下面从根节点开始解析：

```java
// DefaultBeanDefinitionDocumentReader类
protected void doRegisterBeanDefinitions(Element root) {
	// Any nested <beans> elements will cause recursion in this method. In
	// order to propagate and preserve <beans> default-* attributes correctly,
	// keep track of the current (parent) delegate, which may be null. Create
	// the new (child) delegate with a reference to the parent for fallback purposes,
	// then ultimately reset this.delegate back to its original (parent) reference.
	// this behavior emulates a stack of delegates without actually necessitating one.
	// 看名字就知道，BeanDefinitionParserDelegate 必定是一个重要的类，它负责解析Bean定义，
	// 这里为什么要定义一个parent? 看到后面就知道了，是递归问题，
	// 因为<beans/>内部是可以定义<beans/>的，所以这个方法的root其实不一定就是xml的根节点，也可以是嵌套
  // 在里面的<beans/>节点.
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);

	// 这块说的是根节点<beans ... profile="dev" />中的profile是否是当前环境需要的，
	// 如果当前环境配置的profile不包含此profile，那就直接return 了，不对此<beans/>解析
	if (this.delegate.isDefaultNamespace(root)) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			// We cannot use Profiles.of(...) since profile expressions are not supported
			// in XML config. See SPR-12458 for details.
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return; // 不满足，直接返回
			}
		}
	}

	preProcessXml(root); // 钩子
	parseBeanDefinitions(root, this.delegate);
	postProcessXml(root); // 钩子

	this.delegate = parent;
}
```

preProcessXml(root)和postProcessXml(root)是给子类用的钩子方法，鉴于没有被使用到，直接跳过。

接下来，看核心解析方法parseBeanDefinitions(root, this.delegate) :

```java
// DefaultBeanDefinitionDocumentReader类
// default namespace涉及到的就四个标签<import />、<alias />、<bean /> 和 <beans />，其他的属于 custom的
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
					// 解析default namespace下面的几个元素
          // <import />、<alias />、<bean /> 和 <beans />
					parseDefaultElement(ele, delegate);
				}
				else {
					// 解析default namespace下面的几个元素
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}
```

从上面的代码可以看到，对于每个配置来说，会分别进入到parseDefaultElement(ele, delegate)和 delegate.parseCustomElement(ele)这两个分支。

parseDefaultElement(ele, delegate) 代表解析的节点是 `<import />`、`<alias />`、`<bean />`、`<beans />` 这几个。下面分析处理default标签的方法：

```java
// DefaultBeanDefinitionDocumentReader类
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      // 处理<import/>标签
      importBeanDefinitionResource(ele);
   }
   else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      // 处理<alias/>标签定义
      // <alias name="fromName" alias="toName"/>
      processAliasRegistration(ele);
   }
   else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      // 处理<bean/>标签定义，这是重点
      processBeanDefinition(ele, delegate);
   }
   else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // recurse
      // 如果碰到的是嵌套的<beans/>标签，需要递归
      doRegisterBeanDefinitions(ele);
   }
}
```

下面分析`<bean>`标签的解析过程。

##### processBeanDefinition解析`<bean>`标签

```java
// DefaultBeanDefinitionDocumentReader类
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
   // 将<bean/>节点中的信息提取出来，然后封装到一个BeanDefinitionHolder中
   BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
   if (bdHolder != null) {
      // 如果有自定义属性的话，进行相应的解析，先忽略
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
         // Register the final decorated instance.注册BeanDefinition到BeanFactory
         BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to register bean definition with name '" +
               bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event. 发送注册事件
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   }
}
```

继续往下看怎么解析之前，先看下` <bean/> `标签中可以定义哪些属性：

| Property                 | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| class                    | 全限定类名                                                   |
| name                     | 可指定 id、name(用逗号、分号、空格分隔)                      |
| scope                    | 作用域                                                       |
| constructor arguments    | 指定构造参数                                                 |
| properties               | 设置属性的值                                                 |
| autowiring mode          | no(默认值)、byName、byType、 constructor                     |
| lazy-initialization mode | 是否懒加载(如果被非懒加载的bean依赖了，那么其实也就不能懒加载了) |
| initialization method    | bean属性设置完成后，会调用这个方法                           |
| destruction method       | bean销毁后的回调方法                                         |

简单地说，就是像下面这样子：

```xml
<bean id="exampleBean" name="name1, name2, name3" class="com.spring.ExampleBean"
      scope="singleton" lazy-init="true" init-method="init" destroy-method="cleanup">

    <!-- 可以用下面三种形式指定构造参数 -->
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg index="0" value="7500000"/>

    <!-- property 的几种情况 -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>
```

当然，除了上面举例出来的这些，还有`factory-bean`、`factory-method`、`<lockup-method />`、`<replaced-method />`、`<meta />`、`<qualifier />` 这几个。有了以上这些知识以后，再继续往里看怎么解析bean元素，是怎么转换到BeanDefinitionHolder的。

```java
// BeanDefinitionParserDelegate类
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
   return parseBeanDefinitionElement(ele, null);
}
// BeanDefinitionParserDelegate类
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
	String id = ele.getAttribute(ID_ATTRIBUTE); // id属性
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE); // name属性

	// 将name属性的定义按照“逗号、分号、空格”切分，形成一个别名列表数组
	List<String> aliases = new ArrayList<>();
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

	String beanName = id;
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		// 如果没有指定id, 那么用别名列表的第一个名字作为beanName
		beanName = aliases.remove(0);
		if (logger.isTraceEnabled()) {
			logger.trace("No XML 'id' specified - using '" + beanName +
					"' as bean name and " + aliases + " as aliases");
		}
	}

	if (containingBean == null) { // 不是内部类的bean
		// 检查beanName aliases是否已被使用
		checkNameUniqueness(beanName, aliases, ele);
	}

	// 根据<bean ...>...</bean>中的配置创建BeanDefinition，然后把配置中的信息都设置到实例中
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	// 到这里，整个<bean />标签就算解析结束了，一个BeanDefinition就形成了。
	if (beanDefinition != null) {
		// 如果都没有设置id和name，那么此时的beanName就会为null，进入下面这块代码产生
		if (!StringUtils.hasText(beanName)) { // 如果还没有beanName
			try {
				if (containingBean != null) { // 如果ele是内部类的实例，containingBean外部类实例
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
					// 如果不定义id和name，那么beanName、beanClassName如下：
					//   1. beanName 为：com.spring.example.MessageServiceImpl#0
					//   2. beanClassName 为：com.spring.example.MessageServiceImpl
					beanName = this.readerContext.generateBeanName(beanDefinition);
					// Register an alias for the plain bean class name, if still possible,
					// if the generator returned the class name plus a suffix.
					// This is expected for Spring 1.2/2.0 backwards compatibility.
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName); // 把beanClassName设置为Bean的别名
					}
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Neither XML 'id' nor 'name' specified - " +
							"using generated bean name [" + beanName + "]");
				}
			}
			catch (Exception ex) {
				error(ex.getMessage(), ele);
				return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
    // 构造BeanDefinitionHolder返回
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray); 
	}

	return null;
}
```

然后，再看看怎么根据配置创建BeanDefinition实例的：

```java
// BeanDefinitionParserDelegate类
public AbstractBeanDefinition parseBeanDefinitionElement(
      Element ele, String beanName, @Nullable BeanDefinition containingBean) {

   this.parseState.push(new BeanEntry(beanName));

   String className = null;
   if (ele.hasAttribute(CLASS_ATTRIBUTE)) { // class属性
      className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
   }
   String parent = null;
   if (ele.hasAttribute(PARENT_ATTRIBUTE)) { // parent属性
      parent = ele.getAttribute(PARENT_ATTRIBUTE);
   }

   try {
      // 创建GenericBeanDefinition实例
      AbstractBeanDefinition bd = createBeanDefinition(className, parent); 
      // 设置BeanDefinition的一堆属性，这些属性定义在AbstractBeanDefinition中
      parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
      bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

      // 下面的一堆是解析 <bean>......</bean> 内部的子元素，
      // 解析出来以后的信息都放到 bd 的属性中

      // 解析 <meta />
      parseMetaElements(ele, bd);
      // 解析 <lookup-method />
      parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
      // 解析 <replaced-method />
      parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
      // 解析 <constructor-arg />
      parseConstructorArgElements(ele, bd);
      // 解析 <property />
      parsePropertyElements(ele, bd);
      // 解析 <qualifier />
      parseQualifierElements(ele, bd);

      bd.setResource(this.readerContext.getResource());
      bd.setSource(extractSource(ele));

      return bd;
   }
   catch (ClassNotFoundException ex) {
      error("Bean class [" + className + "] not found", ele, ex);
   }
   catch (NoClassDefFoundError err) {
      error("Class that bean class [" + className + "] depends on not found", ele, err);
   }
   catch (Throwable ex) {
      error("Unexpected failure during bean definition parsing", ele, ex);
   }
   finally {
      this.parseState.pop();
   }

   return null;
}
// BeanDefinitionParserDelegate类
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
		throws ClassNotFoundException {
	// 创建GenericBeanDefinition实例
	return BeanDefinitionReaderUtils.createBeanDefinition(
			parentName, className, this.readerContext.getBeanClassLoader());
}
// BeanDefinitionReaderUtils类
public static AbstractBeanDefinition createBeanDefinition(
		@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

	GenericBeanDefinition bd = new GenericBeanDefinition();
	bd.setParentName(parentName); // 设置parent name
	if (className != null) {
		if (classLoader != null) {
			// 设置Class对象
			bd.setBeanClass(ClassUtils.forName(className, classLoader));
		}
		else {
			// 设置BeanClassName
			bd.setBeanClassName(className);
		}
	}
	return bd;
}
```

到这里，已经完成了根据 `<bean/>` 配置创建了一个BeanDefinitionHolde实例的过程。注意，是一个。

回到解析`<bean />`的入口方法:

```java
// DefaultBeanDefinitionDocumentReader类
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
   // 将<bean/>节点中的信息提取出来，然后封装到一个BeanDefinitionHolder中
   BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
   if (bdHolder != null) {
      // 如果有自定义属性的话，进行相应的解析，先忽略
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
         // Register the final decorated instance.注册BeanDefinition到BeanFactory
         BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
         getReaderContext().error("Failed to register bean definition with name '" +
               bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event. 发送注册事件
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
   }
}
```

这里已经根据一个`<bean />`标签产生了一个BeanDefinitionHolder的实例，这个实例里面也就是一个 BeanDefinition的实例和它的beanName、aliases这三个信息。

```java
public class BeanDefinitionHolder implements BeanMetadataElement {

   private final BeanDefinition beanDefinition; // BeanDefinition实例

   private final String beanName; // bean名称

   @Nullable
   private final String[] aliases; // 别名
	
   // ...
}
```

然后准备注册这个BeanDefinition到BeanFactory，最后把这个注册事件发送出去。

##### 注册BeanDefinition

```java
// BeanDefinitionReaderUtils类
public static void registerBeanDefinition(
      BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
      throws BeanDefinitionStoreException {

   // Register bean definition under primary name.
   String beanName = definitionHolder.getBeanName(); // bean name
   // 注册这个 Bean
   registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

   // Register aliases for bean name, if any.
   // 如果还有别名的话，也要根据别名全部注册一遍，不然根据别名就会找不到Bean了
   String[] aliases = definitionHolder.getAliases();
   if (aliases != null) {
      for (String alias : aliases) {
         // alias->beanName保存它们的别名信息，这个很简单，用一个map保存一下就可以了，
         // 获取的时候，会先将alias转换为beanName，然后再查找
         registry.registerAlias(beanName, alias);
      }
   }
}
```

注册BeanDefinition到BeanFactory(BeanDefinitionRegistry)：

```java
// DefaultListableBeanFactory类
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
      throws BeanDefinitionStoreException {

   Assert.hasText(beanName, "Bean name must not be empty");
   Assert.notNull(beanDefinition, "BeanDefinition must not be null");

   if (beanDefinition instanceof AbstractBeanDefinition) {
      try {
         // 1.验证BeanDefinition，如方法覆盖等
         ((AbstractBeanDefinition) beanDefinition).validate();
      }
      catch (BeanDefinitionValidationException ex) {
         throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
               "Validation of bean definition failed", ex);
      }
   }

   // 已存在的BeanDefinition
   BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
   // 2.1处理重复名称的Bean定义的情况
   if (existingDefinition != null) {
      if (!isAllowBeanDefinitionOverriding()) {
         // 如果不允许覆盖的话，抛异常
         throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
      }
      else if (existingDefinition.getRole() < beanDefinition.getRole()) {
         // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
         // 用框架定义的Bean definition覆盖用户自定义的 Bean definition
         if (logger.isInfoEnabled()) {
            logger.info("Overriding user-defined bean definition for bean '" + beanName +
                  "' with a framework-generated bean definition: replacing [" +
                  existingDefinition + "] with [" + beanDefinition + "]");
         }
      }
      else if (!beanDefinition.equals(existingDefinition)) {
         // 用新的Bean definition覆盖旧的 Bean definition
         if (logger.isDebugEnabled()) {
            logger.debug("Overriding bean definition for bean '" + beanName +
                  "' with a different definition: replacing [" + existingDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      else {
         // 用同等的Bean definition覆盖旧的Bean definition，这里指的是equals方法返回true的Bean definition
         if (logger.isTraceEnabled()) {
            logger.trace("Overriding bean definition for bean '" + beanName +
                  "' with an equivalent definition: replacing [" + existingDefinition +
                  "] with [" + beanDefinition + "]");
         }
      }
      this.beanDefinitionMap.put(beanName, beanDefinition); // 覆盖
   }
   // 2.2注册新的BeanDefinition
   else {
      // 判断是否已经有其他的Bean开始初始化了
      // 注意，"注册Bean"这个动作结束，Bean依然还没有初始化
      // 在Spring容器启动的最后，会预初始化所有的singleton beans
      if (hasBeanCreationStarted()) {
         // Cannot modify startup-time collection elements anymore (for stable iteration)
         synchronized (this.beanDefinitionMap) {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
            if (this.manualSingletonNames.contains(beanName)) {
               Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
               updatedSingletons.remove(beanName);
               this.manualSingletonNames = updatedSingletons;
            }
         }
      }
      else {
         // 最正常的应该是进到这个分支。
         // Still in startup registration phase

         // 将BeanDefinition放到这个map中，这个map保存了所有的BeanDefinition
         this.beanDefinitionMap.put(beanName, beanDefinition);
         // 这是个ArrayList，所以会按照bean配置的顺序保存每一个注册的Bean的名字
         this.beanDefinitionNames.add(beanName);
         // 这是个LinkedHashSet，代表的是手动注册的singleton bean，
         // 注意这里是remove方法，到这里的Bean当然不是手动注册的
         // 手动指的是通过调用以下方法注册的bean ：
         //     registerSingleton(String beanName, Object singletonObject)
         // 这不是重点，解释只是为了不让大家疑惑。Spring会在后面"手动"注册一些Bean，
         // 如"environment"、"systemProperties"等bean，我们自己也可以在运行时注册Bean到容器中
         this.manualSingletonNames.remove(beanName);
      }
      // 这个不重要，在预初始化的时候会用到，不必管它
      this.frozenBeanDefinitionNames = null;
   }

   if (existingDefinition != null || containsSingleton(beanName)) {
      resetBeanDefinition(beanName);
   }
}
```

总结一下，到这里已经初始化了Bean容器，`<bean/>` 配置也相应的转换为了一个个BeanDefinition，然后注册了各个BeanDefinition 到BeanFactory，并且发送了注册事件。

### 3.准备BeanFactory(prepareBeanFactory)

设置BeanFactory的类加载器，添加几个BeanPostProcessor，手动注册几个特殊的 bean等。

```java
// AbstractApplicationContext类
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // 1.Tell the internal bean factory to use the context's class loader etc.
   // 设置 BeanFactory 的类加载器，我们知道 BeanFactory 需要加载类，也就需要类加载器，
   // 这里设置为加载当前 ApplicationContext 类的类加载器
   beanFactory.setBeanClassLoader(getClassLoader());
   // 设置表达式解析器StandardBeanExpressionResolver(解析bean定义中的一些表达式)
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   // 添加属性编辑器注册器
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // 2.Configure the bean factory with context callbacks.
   // 添加一个 BeanPostProcessor，这个 processor 比较简单：
   // 实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，
   // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
   // 注意：它不仅仅回调 ApplicationContextAware，还会负责回调 EnvironmentAware、ResourceLoaderAware 等
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   // 取消
   // nvironmentAware,
   // eddedValueResolverAware,
   // esourceLoaderAware,
   // pplicationEventPublisherAware,
   // essageSourceAware,
   // pplicationContextAware这6个接口的自动注入。
   // 为ApplicatioinContextAwareProcessor把这6这个接口的实现工作做了
   //
   // 存在beanFactory的ignoredDependencyInterfaces集合中
   //
   //ApplicatioinContextAwareProcessor的作用在于为实现*Aware接口的bean调用该Aware接口定义的方法，并传入对应的参数。
   // 如实现EnvironmentAware接口的bean在该Processor内部会调用EnvironmentAware接口的setEnvironment方法，把Spring容器内部的ConfigurationEnvironment传递进去。
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // 3.BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
   // 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
   // 之前说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行。
   // ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
   // 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext。
   // 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean。
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // 4.Register early post-processor for detecting inner beans as ApplicationListeners.
   // 这个 BeanPostProcessor 也很简单，在 bean 实例化后，如果是 ApplicationListener 的子类，
   // 那么将其添加到 listener 列表中，可以理解成：注册 事件监听器
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 5.Detect a LoadTimeWeaver and prepare for weaving, if found.
   // 这里涉及到特殊的 bean，名为：loadTimeWeaver，这不是我们的重点，忽略它.
   // tips: ltw 是 AspectJ 的概念，指的是在运行期进行织入，这个和 Spring AOP 不一样.
   // 参考关于 AspectJ 的一篇文章 https://www.javadoop.com/post/aspectj
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 从下面几行代码我们可以知道，Spring 往往很 "智能" 就是因为它会帮我们默认注册一些有用的 bean，
   // 我们也可以选择覆盖

   // 6.Register default environment beans.
   // 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个(单例)
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   // 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个(单例)
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   // 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个(单例)
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

### 4.postProcessBeanFactory

```java
// AbstractApplicationContext类
// 这里是提供给子类的扩展点，到这里的时候，所有的Bean都加载、注册完成了，但是都还没有初始化。
// 具体的子类可以在这步的时候添加一些特殊的BeanPostProcessor的实现类或做点什么事
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

### 5.invokeBeanFactoryPostProcessors

接下去调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory) 回调方法。

```java
// AbstractApplicationContext类
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   // 实例化BeanFactoryPostProcessor beans，调用postProcessBeanFactory方法
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   // ConfigurationClassPostProcessor在处理配置类的过程中，发现配置类里配了LoadTimeWeaver bean定义
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory)的过程交由PostProcessorRegistrationDelegate处理：

```java
// PostProcessorRegistrationDelegate类
public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any. 先调用BeanDefinitionRegistryPostProcessor的回调方法
   Set<String> processedBeans = new HashSet<>(); // 已经处理过的bean

   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      // BeanDefinitionPostProcessor实例列表
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      // BeanDefinitionRegistryPostProcessor实例列表
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

      // 首先处理作为参数传入的beanFactoryPostProcessors
      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         // 如果postProcessor是BeanDefinitionRegistryPostProcessor，则调用postProcessBeanDefinitionRegistry
         // 对beanFactory进行处理。然后加入registryPostProcessors，否则加入regularPostProcessors
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            regularPostProcessors.add(postProcessor);
         }
      }

      // Do not initialize FactoryBeans here: We need to leave all regular beans (c)
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.

      // 当前调用阶段的BeanDefinitionRegistryPostProcessor列表
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
      // 首先调用实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessors
      // 获取实现了BeanDefinitionRegistryPostProcessor接口的bean名数组(不初始化FactoryBean)
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) { // PriorityOrdered实例
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName); // 添加到已处理的processedBeans集合
         }
      }
      // priorityOrderedPostProcessors中保存的是ConfigurationClassPostProcessor(在xml中配置<context:component-scan base-package="com.xxx.yyy"/>)
      sortPostProcessors(currentRegistryProcessors, beanFactory); // 排序
      registryProcessors.addAll(currentRegistryProcessors); // 添加到registryProcessors列表
      // 调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      // 然后调用实现了Ordered接口的BeanDefinitionRegistryPostProcessors
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         // 还没处理过，且是Ordered的实例
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName); // 添加到已处理的processedBeans集合
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory); // 排序
      registryProcessors.addAll(currentRegistryProcessors); // 添加到registryProcessors列表
      // 调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      // 调用其他BeanDefinitionRegistryPostProcessors
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) { // 还没处理过
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory); // 排序
         registryProcessors.addAll(currentRegistryProcessors);  // 添加到registryProcessors列表
         // 调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
      // 调用postProcessBeanFactory回调
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      // 调用传入的beanFactoryPostProcessors的回调方法
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans(不初始化FactoryBean)
   // uninitialized to let the bean factory post-processors apply to them!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   // 分别调用实现了PriorityOrdered，Ordered接口的BeanFactoryPostProcessors和其余的BeanFactoryPostProcessors
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above 已处理的忽略
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   // 调用实现了PriorityOrdered接口的BeanFactoryPostProcessors
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   // 调用实现了Ordered接口的BeanFactoryPostProcessors
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   // 调用其余BeanFactoryPostProcessors
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   beanFactory.clearMetadataCache();
}
// PostProcessorRegistrationDelegate类
// 排序后置处理器
private static void sortPostProcessors(List<?> postProcessors, ConfigurableListableBeanFactory beanFactory) {
	Comparator<Object> comparatorToUse = null;
	if (beanFactory instanceof DefaultListableBeanFactory) {
		comparatorToUse = ((DefaultListableBeanFactory) beanFactory).getDependencyComparator();
	}
	if (comparatorToUse == null) {
		comparatorToUse = OrderComparator.INSTANCE;
	}
	postProcessors.sort(comparatorToUse);
}
// PostProcessorRegistrationDelegate类
private static void invokeBeanDefinitionRegistryPostProcessors(
		Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

	for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanDefinitionRegistry(registry);
	}
}
// PostProcessorRegistrationDelegate类
private static void invokeBeanFactoryPostProcessors(
		Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

	for (BeanFactoryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanFactory(beanFactory);
	}
}
```

### 6.注册BeanPostProcessors

注册BeanPostProcessor的实现类到BeanFactory。

```java
// AbstractApplicationContext类
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	// 实例化BeanPostProcessor，并注册到BeanFactory
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
// PostProcessorRegistrationDelegate类
// 实例化BeanPostProcessor，并注册到BeanFactory
public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
	// 找到所有实现了BeanPostProcessor接口的bean名
	// internalAutowiredAnnotationProcessor, internalRequiredAnnotationProcessor, internalCommonAnnotationProcessor, internalAutoProxyCreator
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// Register BeanPostProcessorChecker that logs an info message when
	// a bean is created during BeanPostProcessor instantiation, i.e. when
	// a bean is not eligible for getting processed by all BeanPostProcessors.
	// BeanPostProcessorChecker用于检测在BeanPostProcessor初始化的时候，一个bean创建了，这个bean还不能
	// 被所有BeanPostProcessors处理，因此打印一条info消息
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length; // 总的BeanPostProcessor个数
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// Separate between BeanPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	// 将BeanPostProcessors分为实现了PriorityOrdered接口、实现了Ordered接口、其他，三种类型
	// priorityOrderedPostProcessors中有AutowiredAnnotationBeanPostProcessor, RequiredAnnotationBeanPostProcessor, CommonAnnotationBeanPostProcessor
	// orderedPostProcessorNames中有AnnotationAwareAspectJAutoProxyCreator
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class); // 实现PriorityOrdered接口
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) { // 实现Ordered接口
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName); // 其余BeanPostProcessor
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
	// 注册实现了PriorityOrdered接口的BeanPostProcessors，
	// 当前主要有
	// AutowiredAnnotationBeanPostProcessor
	// RequiredAnnotationBeanPostProcessor
	// CommonAnnotationBeanPostProcessor
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	// 注册到beanFactory中。添加到List<BeanPostProcessor> beanPostProcessors中。
	// 当前beanPostProcessors中有：
	// ApplicationContextAwareProcessor
	// ApplicationListenerDetector
	// ImportAwareBeanPostProcessor
	// BeanPostProcessorChecker
	// CommonAnnotationBeanPostProcessor
	// AutowiredAnnotationBeanPostProcessor
	// RequiredAnnotationBeanPostProcessor
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors); // 注册实现了PriorityOrdered接口的BeanPostProcessor

	// Next, register the BeanPostProcessors that implement Ordered.
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors); // 注册实现了Ordered接口的BeanPostProcessor

	// Now, register all regular BeanPostProcessors.
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	// 注册到beanFactory中。添加到List<BeanPostProcessor> beanPostProcessors中。
	// 当前beanPostProcessors中有：
	// ApplicationContextAwareProcessor
	// ApplicationListenerDetector
	// ImportAwareBeanPostProcessor
	// BeanPostProcessorChecker
	// CommonAnnotationBeanPostProcessor
	// AutowiredAnnotationBeanPostProcessor
	// RequiredAnnotationBeanPostProcessor
	// AnnotationAwareAspectJAutoProxyCreator
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors); // 注册其余BeanPostProcessor

	// Finally, re-register all internal BeanPostProcessors.
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors); // 注册MergedBeanDefinitionPostProcessor

	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
	// 添加ApplicationListenerDetector，用于检测ApplicationListeners
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	//
	 当前beanPostProcessors中有：
		ApplicationContextAwareProcessor
		ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
		PostProcessorRegistrationDelegate$BeanPostProcessorChecker
		AnnotationAwareAspectJAutoProxyCreator
		CommonAnnotationBeanPostProcessor
		AutowiredAnnotationBeanPostProcessor
		ApplicationListenerDetector
		RequiredAnnotationBeanPostProcessor
	 //
}
// PostProcessorRegistrationDelegate类
private static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

	for (BeanPostProcessor postProcessor : postProcessors) {
		beanFactory.addBeanPostProcessor(postProcessor);
	}
}
```

### 7.初始化MessageSource

```java
// AbstractApplicationContext类
// 初始化当前ApplicationContext的MessageSource Bean(国际化)
protected void initMessageSource() {
   // DefaultListableBeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   // 当前BeanFactory是否包含MessageSource实例，不查找父Context
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) { 
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
      // Make MessageSource aware of parent MessageSource. 如果父MessageSource不存在，设置父MessageSource
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
         if (hms.getParentMessageSource() == null) {
            // Only set parent context as parent MessageSource if no parent MessageSource
            // registered already.
            hms.setParentMessageSource(getInternalParentMessageSource());
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using MessageSource [" + this.messageSource + "]");
      }
   }
   // 当前BeanFactory不包含MessageSource实例
   else {
      // Use empty MessageSource to be able to accept getMessage calls.使用DelegatingMessageSource，代理给父MessageSource
      DelegatingMessageSource dms = new DelegatingMessageSource();
      dms.setParentMessageSource(getInternalParentMessageSource());
      this.messageSource = dms;
      // 注册bean
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
      }
   }
}
```

### 8.初始化ApplicationEventMulticaster

```java
// AbstractApplicationContext类
// 初始化当前ApplicationContext的事件广播器
protected void initApplicationEventMulticaster() {
   // DefaultListableBeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory(); 
   // 当前BeanFactory是否包含ApplicationEventMulticaster实例，不查找父Context
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isTraceEnabled()) {
         logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
      // 使用SimpleApplicationEventMulticaster
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      // 注册bean
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
               "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
      }
   }
}
```

### 9.模板方法onRefresh

具体的子类可以在这里初始化一些特殊的Bean(在初始化singleton beans之前)

```java
// AbstractApplicationContext类
protected void onRefresh() throws BeansException {
   // For subclasses: do nothing by default.
}
```

### 10.注册ApplicationListener

注册应用事件监听器，监听器需要实现ApplicationListener接口。

```java
// AbstractApplicationContext类
protected void registerListeners() {
   // Register statically specified listeners first. 先注册静态指定的监听器
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   // 获取实现了ApplicationListener接口的Bean的名称，包括BeanDefinition和FactoryBean本身(allowEagerInit:fasle，表示不会实例化FactoryBean)
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents; // 发布early event
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

### 11. 初始化所有非懒加载的单例Beans

到目前为止，BeanFactory已经创建完成，并且所有的实现了BeanFactoryPostProcessor接口的Bean都已经初始化并且其中的postProcessBeanFactory(factory)方法已经得到回调执行。而且Spring已经“手动”注册了一些特殊的 Bean，如'environment'、'systemProperties'等。剩下的就是初始化非懒加载的singleton beans了。

```java
// AbstractApplicationContext类
// 初始化所有非懒加载的单例bean
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // Initialize conversion service for this context. 初始化名字为conversionService的Bean
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            // 初始化conversion service
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }

   // Register a default embedded value resolver if no bean post-processor
   // (such as a PropertyPlaceholderConfigurer bean) registered any before:
   // at this point, primarily for resolution in annotation attribute values. 添加EmbeddedValueResolver
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
   // 初始化LoadTimeWeaverAware类型的 Bean
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   // Allow for caching all bean definition metadata, not expecting further changes. 	
   // BeanDefinition已经不能再改变了
   beanFactory.freezeConfiguration();

   // Instantiate all remaining (non-lazy-init) singletons. 实例化所有遗留的、非懒加载的单例Bean
   beanFactory.preInstantiateSingletons();
}
```

#### (a) preInstantiateSingletons

```java
// DefaultListableBeanFactory类
// 实例化所有遗留的、非懒加载的单例Bean
public void preInstantiateSingletons() throws BeansException {  
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }

   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
   // 拷贝一份beanDefinitionNames，防止在一些bean初始化时可能会注册新的bean definitions，导致beanDefinitionNames变更
   // this.beanDefinitionNames 保存了所有的 beanNames
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames); 

   // Trigger initialization of all non-lazy singleton beans... 实例化所有非懒加载的单例Bean
   for (String beanName : beanNames) {
      // 获取MergedLocalBeanDefinition，合并父子BeanDefinition
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName); 
      // 非抽象、单例、非懒加载的bean初始化。如果配置了'abstract = true'，那是不需要初始化的
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         if (isFactoryBean(beanName)) { // 是否是Factory Bean
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName); // 实例化FactoryBean本身
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               // 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  // 使用FactoryBean创建实例，初始化该FactoryBean对应的实例(getObject())
                  getBean(beanName);
               }
            }
         }
         else {
            // 不是FactoryBean，直接初始化
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans... 执行SmartInitializingSingleton接口回调(单例已全部初始化)
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

接下来，就进入到getBean(beanName)方法了，这个方法经常**用来从BeanFactory中获取一个Bean，而初始化的过程也封装在这个方法里**。

#### (b) getBean/doGetBean

如果name对应的bean已经初始化，直接返回；否则初始化之后(缓存之后)再返回。

```java
// AbstractBeanFactory类
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
// AbstractBeanFactory类
// 目前是在剖析初始化Bean的过程，但是getBean方法经常是用来从容器中获取Bean用的，注意切换思路，
// 已经初始化过了就从容器中直接返回，否则就先初始化再返回。
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
		@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	// name转换为正规的名称
	// 获取一个“正统的”beanName，处理两种情况，一个是前面说的FactoryBean(前面带‘&’)，
	// 一个是别名问题，因为这个方法是getBean，获取Bean用的，要是传一个别名进来，是完全可以的
	final String beanName = transformedBeanName(name);
	Object bean; // 返回值

	// Eagerly check singleton cache for manually registered singletons. 
  // 1.首先检查单例缓存(包含已创建的和手动注册的单例)
	// 缓存singletonObjects不包含FactoryBean.getObject()创建的单例，只有普通单例
	// (包含手动注册的和通过BeanDefinition生成的单例)和FactoryBean实例本身，
	// FactoryBean.getObject()创建的单例缓存在FactoryBeanRegistrySupport.factoryBeanObjectCache中
	Object sharedInstance = getSingleton(beanName); // 检查缓存，看是否已创建
	// 前面进来的时候都是getBean(beanName)，所以args传参其实是null的，但是如果args不为空的时候，
	// 意味着调用方不是希望获取Bean，而是创建Bean
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 返回普通Bean实例或FactoryBean实例或FactoryBean创建的Bean实例
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) { // 原型Bean不能循环依赖
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		// 2.若当前BeanFactory中不存在该beanName对应的BeanDefinition，则向父BeanFactory获取
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args. 返回父容器的查询结果
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}

		if (!typeCheckOnly) {
			// 标记beanName为已创建
			markBeanAsCreated(beanName);
		}

		// 稍稍总结一下：
		// 到这里的话，要准备创建Bean了，对于singleton的Bean来说，容器中还没创建过此Bean；
		// 对于prototype的Bean来说，本来就是要创建一个新的Bean。

		try {
			// 3.在当前BeanFactory中创建Bean实例
      // 获取merged BeanDefinition
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName); 
			checkMergedBeanDefinition(mbd, beanName, args); // 验证bean definition

			// Guarantee initialization of beans that the current bean depends on.
			// 3.1确保当前bean依赖(depend on)的bean先初始化
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					// depend on属性定义的依赖关系不能出现循环依赖
					if (isDependent(beanName, dep)) { // dep是否依赖beanName
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					// 注册Bean之间的依赖关系
					registerDependentBean(dep, beanName);
					try {
						// 先初始化依赖的bean
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.创建Bean实例
			// 3.2如果是单例，则创建单例实例
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						// 重点步骤，创建Bean单例实例
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						// Explicitly remove instance from singleton cache: It might have been put there
						// eagerly by the creation process, to allow for circular reference resolution.
						// Also remove any beans that received a temporary reference to the bean.
						destroySingleton(beanName);
						throw ex;
					}
				});
				// sharedInstance可能是普通Bean，也可能是FactoryBean实例本身。getObjectForBeanInstance()
        // 根据name/beanName返回普通Bean、FactoryBean实例创建的实例或FactoryBean实例本身。
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
			// 3.3创建原型Bean实例
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
          // 标记当前待创建的Bean到ThreadLocal变量prototypesCurrentlyInCreation
					beforePrototypeCreation(beanName); 
					prototypeInstance = createBean(beanName, mbd, args); // 创建原型实例
				}
				finally {
          // 从ThreadLocal变量prototypesCurrentlyInCreation中移除该bean标记
					afterPrototypeCreation(beanName); 
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				// 3.4特定作用域的Bean创建
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance. 4.Bean类型转换
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				// 类型不匹配
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

##### getSingleton(String beanName)

根据beanName，查找已经实例化的单例。

```java
// DefaultSingletonBeanRegistry类
// 查找已经实例化的单例
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// 缓存singletonObjects不包含FactoryBean.getObject()创建的单例，只有普通单例
	// (包含手动注册的和通过BeanDefinition生成的单例)和FactoryBean实例本身，
	// FactoryBean.getObject()创建的单例缓存在FactoryBeanRegistrySupport.factoryBeanObjectCache中
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}
```

#####  getObjectForBeanInstance

对于给定的beanInstance，根据name、beanName，返回普通Bean实例或FactoryBean实例本身或FactoryBean创建的Bean实例。

```java
// AbstractBeanFactory类
// 根据name、beanName，返回普通Bean实例或FactoryBean实例本身或FactoryBean创建的Bean实例
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

   // Don't let calling code try to dereference the factory if the bean isn't a factory.
   // 1.如果name以&开头，但beanInstance却不是FactoryBean，则认为有问题。
   if (BeanFactoryUtils.isFactoryDereference(name)) {
      if (beanInstance instanceof NullBean) {
         return beanInstance;
      }
      if (!(beanInstance instanceof FactoryBean)) {
         throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
      }
   }

   // Now we have the bean instance, which may be a normal bean or a FactoryBean.
   // If it's a FactoryBean, we use it to create a bean instance, unless the
   // caller actually wants a reference to the factory.
   // 2.如果上面的判断通过了，表明beanInstance可能是一个普通的bean，也可能是一个
   // FactoryBean。如果是一个普通的bean，这里直接返回beanInstance即可；如果是
   // FactoryBean且用户要去返回FactoryBean本身(name包含&前缀)，则直接返回beanInstance。
   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
   }

   // 下面是获取FactoryBean创建的Bean实例
   Object object = null;
   if (mbd == null) {
      // 3.如果mbd为空，则从缓存中加载bean。FactoryBean生成(getObject)的单例bean会被缓存
      // 在factoryBeanObjectCache集合中，不用每次都创建.
      object = getCachedObjectForFactoryBean(beanName);
   }
   if (object == null) {
      // Return bean instance from factory.
      // 经过前面的判断，到这里可以保证beanInstance是FactoryBean类型的，所以可以进行类型转换
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      // Caches object obtained from FactoryBean if it is a singleton.
      // 如果mbd为空，则判断是否存在名字为beanName的BeanDefinition
      if (mbd == null && containsBeanDefinition(beanName)) {
         // 获取合并的BeanDefinition
         mbd = getMergedLocalBeanDefinition(beanName);
      }
      boolean synthetic = (mbd != null && mbd.isSynthetic()); // 是否是合成的 // TODO ??
      // 4.调用FactoryBean.getObject()获取单例实例并缓存到FactoryBeanRegistrySupport.factoryBeanObjectCache中
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```

##### getSingleton(String beanName, ObjectFactory<?> singletonFactory)

根据beanName获取缓存的单例，如果不存在，则使用singletonFactory创建单例之后缓存该单例实例，再返回。

```java
// DefaultSingletonBeanRegistry类
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(beanName, "Bean name must not be null");
   synchronized (this.singletonObjects) {
      // 首先从缓存中获取单例实例
      Object singletonObject = this.singletonObjects.get(beanName);
      // singletonObject为空，则创建
      if (singletonObject == null) {
         // 是否当前正在执行destroySingletons()方法
         if (this.singletonsCurrentlyInDestruction) {
            throw new BeanCreationNotAllowedException(beanName,
                  "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                  "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
         }
         beforeSingletonCreation(beanName); // 标记当前待创建的单例bean为正在创建状态
         boolean newSingleton = false; // 是否为新的单例实例
         boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
         if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
         }
         try {
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
         }
         catch (IllegalStateException ex) {
            // Has the singleton object implicitly appeared in the meantime ->
            // if yes, proceed with it since the exception indicates that state.
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
               throw ex;
            }
         }
         catch (BeanCreationException ex) {
            if (recordSuppressedExceptions) {
               for (Exception suppressedException : this.suppressedExceptions) {
                  ex.addRelatedCause(suppressedException);
               }
            }
            throw ex;
         }
         finally {
            if (recordSuppressedExceptions) {
               this.suppressedExceptions = null;
            }
            afterSingletonCreation(beanName); // 标记当前待创建的单例bean为创建完成状态
         }
         if (newSingleton) {
            // 如果是新的单例，则添加到缓存
            addSingleton(beanName, singletonObject);
         }
      }
      return singletonObject;
   }
}
```

下面分析createBean方法。

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException;
```

第三个参数args数组代表创建实例需要的参数，这是给构造方法用的参数，或者是工厂Bean的参数，不过要注意，在初始化阶段，args是null。

这回要到一个新的类AbstractAutowireCapableBeanFactory了。主要是为了以下场景，采用@Autowired注解注入属性值：

```java
public class MessageServiceImpl implements MessageService {
    @Autowired
    private UserService userService;

    public String getMessage() {
        return userService.getMessage();
    }
}
```

```xml
<bean id="messageService" class="com.javadoop.example.MessageServiceImpl" />
```

以上这种属于混用了xml和注解两种方式的配置方式，Spring会处理这种情况。

#### (c) createBean

创建Bean实例，填充属性，应用后置处理器等。

```java
// AbstractAutowireCapableBeanFactory类
// 创建Bean实例，填充属性，应用后置处理器等
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'"); // beanName为正规BeanName
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and  确保类已解析、加载
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides. 准备方法覆盖(查找方法和替换方法)，检查方法是否存在、是否重载等
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // 使 InstantiationAwareBeanPostProcessor 在这一步有机会返回代理
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      // 创建Bean实例，实例化、填充属性、初始化(post-processors)
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

##### 创建Bean(doCreateBean)

真正创建Bean实例的逻辑。

```java
// AbstractAutowireCapableBeanFactory类
// Actually create the specified bean. Pre-creation processing has already happened
// at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
// <p>Differentiates between default bean instantiation, use of a
// factory method, and autowiring a constructor.
// @param beanName the name of the bean
// @param mbd the merged bean definition for the bean
// @param args explicit arguments to use for constructor or factory method invocation
// @return a new instance of the bean
// @throws BeanCreationException if the bean could not be created
// @see #instantiateBean 默认构造器创建Bean
// @see #instantiateUsingFactoryMethod 工厂方法创建Bean
// @see #autowireConstructor 构造器注入创建Bean
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean. 实例化Bean
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      // 1.说明不是FactoryBean，实例化Bean(默认构造器实例化/工厂方法实例化/构造器注入实例化)
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance(); // bean实例
   Class<?> beanType = instanceWrapper.getWrappedClass(); // bean类型
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   // 涉及接口MergedBeanDefinitionPostProcessor(使用很少)
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references 
   // 解决循环依赖，及早缓存bean引用
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      // 添加单例对象的工厂
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance. 初始化
   Object exposedObject = bean; // 创建的实例
   try {
      // 2.填充Bean属性
      populateBean(beanName, mbd, instanceWrapper);
      // 3.初始化Bean，应用各种PostProcessor、init method
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }
   // TODO 待看 循环引用 ??
   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable. 
   // 注册销毁逻辑，比如Bean实现了DisposableBean接口，指定了destroy method等
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }
   // 返回最终的实例(可能是个代理)
   return exposedObject;
}
```

根据以上createBean的逻辑，下面分三个细节做下分析。一个是创建Bean实例的createBeanInstance方法，一个是依赖注入的populateBean方法，还有就是回调方法initializeBean。

##### createBeanInstance创建Bean实例

创建Bean实例，使用工厂方法、构造器依赖注入或简单初始化(默认构造器初始化)。

```java
// AbstractAutowireCapableBeanFactory类
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point. 
   // 解析BeanClass，确保已经加载了此 class
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   // BeanClass访问权限检查
   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }
   // 1.通过Supplier获取实例(Spring 5.0新增)
   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }
   // 2.静态工厂实例化、实例工厂方法实例化 factoryMethodName != null
   if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   // 这里加速判断实例创建是构造器注入还是默认无参构造器初始化(反射)，对原型等多次创建有用。
   // 如果不是第一次创建，比如第二次创建 prototype bean。
   // 这种情况下，可以从第一次创建知道，采用无参构造函数，还是构造函数依赖注入 来完成实例化。
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   if (resolved) {
      if (autowireNecessary) {
         // 构造函数依赖注入
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         // 无参构造函数
         return instantiateBean(beanName, mbd);
      }
   }

   // Candidate constructors for autowiring? 3.构造器依赖注入
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // Preferred constructors for default construction? 3.构造器依赖注入
   ctors = mbd.getPreferredConstructors();
   if (ctors != null) {
      return autowireConstructor(beanName, mbd, ctors, null);
   }

   // No special handling: simply use no-arg constructor. 4.默认使用无参构造器实例化
   return instantiateBean(beanName, mbd);
}
```

下面分析下使用默认无参构造器实例化bean的逻辑：

```java
// AbstractAutowireCapableBeanFactory类
// 使用默认构造器实例化
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      final BeanFactory parent = this;
      if (System.getSecurityManager() != null) {
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               getInstantiationStrategy().instantiate(mbd, beanName, parent),
               getAccessControlContext());
      }
      else {
         // 使用实例化策略(CglibSubclassingInstantiationStrategy实例)实例化Bean
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
      }
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw); // 初始化BeanWrapper，设置ConversionService、注册属性编辑器
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}
```

这里关键的地方在于：

```java
beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
```

这里会进行实际的实例化过程：

```java
// SimpleInstantiationStrategy类
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
   // Don't override the class with CGLIB if no overrides. 没有查找方法和替换方法
   // 如果不存在方法覆写，那就使用Java反射进行实例化，否则使用CGLIB
   if (!bd.hasMethodOverrides()) {
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         if (constructorToUse == null) {
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  constructorToUse = AccessController.doPrivileged(
                        (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
               }
               else {
                  constructorToUse = clazz.getDeclaredConstructor();
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
      return BeanUtils.instantiateClass(constructorToUse); // 利用构造方法进行反射实例化
   }
   else {
      // Must generate CGLIB subclass. 生成代理类实例
      // 存在方法覆写，利用CGLIB来完成实例化，需要依赖于CGLIB生成子类。
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

到这里，bean实例化完成了。下面分析bean的属性注入。

##### Bean属性注入

该方法负责进行bean属性设值，处理依赖。

```java
// AbstractAutowireCapableBeanFactory类
// 填充实例的属性值
@SuppressWarnings("deprecation")  // for postProcessPropertyValues
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase(阶段) for null instance.
         return;
      }
   }

   // 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改
   // bean 状态的机会。关于这段后置引用，官方的解释是：让用户可以自定义属性注入。比如用户实现一
   // 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过
   // postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。当然，如果无
   // 特殊需求，直接使用配置中的信息注入即可。另外，Spring 并不建议大家直接实现
   // InstantiationAwareBeanPostProcessor 接口，如果想实现这种类型的后置处理器，更建议
   // 通过继承 InstantiationAwareBeanPostProcessorAdapter 抽象类实现自定义后置处理器。
   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               // 如果返回false，代表不需要进行后续的属性设值，也不需要再经过其他的BeanPostProcessor
               // 的处理
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }

   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
   // 根据名称或类型找到所有属性值
   if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // Add property values based on autowire by name if applicable.
      // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
      if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // Add property values based on autowire by type if applicable.
      // 通过类型装配
      if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

  // 这里又是一种后置处理，用于在Spring填充属性到bean对象前，对属性的值进行相应的处理，
  // 比如可以修改某些属性的值。这时注入到bean中的值就不是配置文件中的内容了，
  // 而是经过后置处理器修改后的内容
   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 对属性进行后置处理
            // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
            // 对采用 @Autowired、@Value 注解的依赖进行设值。如果使用@Resource注解，则是CommonAnnotationBeanPostProcessor(注意)
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               // 对属性进行后置处理
               // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
               // 对采用 @Autowired、@Value 注解的依赖进行设值。如果使用@Resource注解，则是CommonAnnotationBeanPostProcessor(注意)
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }
   // 应用属性值到 bean 对象中
   if (pvs != null) {
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

下面看下applyPropertyValues的细节：

```java
// AbstractAutowireCapableBeanFactory类
// 应用属性值，解析对其他bean的引用
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
   if (pvs.isEmpty()) { // 属性值为空，返回
      return;
   }

   if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
      ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
   }

   MutablePropertyValues mpvs = null;
   List<PropertyValue> original;

   if (pvs instanceof MutablePropertyValues) {
      mpvs = (MutablePropertyValues) pvs;
      // 如果属性列表pvs已被转换过，则直接设置属性值并返回即可
      if (mpvs.isConverted()) {
         // Shortcut: use the pre-converted values as-is.
         try {
            bw.setPropertyValues(mpvs); // 设置属性值
            return;
         }
         catch (BeansException ex) {
            throw new BeanCreationException(
                  mbd.getResourceDescription(), beanName, "Error setting property values", ex);
         }
      }
      original = mpvs.getPropertyValueList();
   }
   else {
      original = Arrays.asList(pvs.getPropertyValues());
   }

   // 类型转换器， BeanWrapper本身
   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }
   BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

   // Create a deep copy, resolving any references for values.
   List<PropertyValue> deepCopy = new ArrayList<>(original.size());
   boolean resolveNecessary = false;
   // 遍历属性列表
   for (PropertyValue pv : original) {
      if (pv.isConverted()) { // 如果属性值被转换过，则就不需要再次转换
         deepCopy.add(pv);
      }
      else {
         String propertyName = pv.getName();
         Object originalValue = pv.getValue();
          // 解析属性值。举例说明，先看下面的配置：
          //
          //  <bean id="macbook" class="MacBookPro">
          //       <property name="manufacturer" value="Apple"/>
          //       <property name="width" value="280"/>
          //       <property name="cpu" ref="cpu"/>
          //       <property name="interface">
          //           <list>
          //               <value>USB</value>
          //               <value>HDMI</value>
          //               <value>Thunderbolt</value>
          //           </list>
          //       </property>
          //   </bean>
          //
          // 上面是一款电脑的配置信息，每个 property 配置经过下面的方法解析后，返回如下结果：
          //   propertyName = "manufacturer", resolvedValue = "Apple"
          //   propertyName = "width", resolvedValue = "280"
          //   propertyName = "cpu", resolvedValue = "CPU@1234"  注：resolvedValue 是一个对象
          //   propertyName = "interface", resolvedValue = ["USB", "HDMI", "Thunderbolt"]
          //
          // 如上所示，resolveValueIfNecessary 会将 ref 解析为具体的对象，将 <list>
          // 标签转换为 List 对象等。对于 int 类型的配置，这里并未做转换，所以
          // width = "280"，还是字符串。除了解析上面几种类型，该方法还会解析 <set/>、
          // <map/>、<array/> 等集合配置
         // 解析后的值
         Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
         // 类型转换后的值
         Object convertedValue = resolvedValue;
         // 属性值是否可转换(true条件1.属性可写 2.非nested或者indexed属性)
         boolean convertible = bw.isWritableProperty(propertyName) &&
               !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
         // 对于一般的属性，convertible 通常为 true
         if (convertible) {
            // 对属性值的类型进行转换，比如将 String 类型的属性值 "123" 转为 Integer 类型的 123
            convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
         }
         // Possibly store converted value in merged bean definition,
         // in order to avoid re-conversion for every created bean instance. 避免重复转换
         // 如果 originalValue 是通过 autowireByType 或 autowireByName 解析而来，
         // 那么此处条件成立，即 (resolvedValue == originalValue) = true
         if (resolvedValue == originalValue) {
            if (convertible) {
               // 将 convertedValue 设置到 pv 中，后续再次创建同一个 bean 时，就无需再次进行转换了
               pv.setConvertedValue(convertedValue);
            }
            deepCopy.add(pv);
         }
         // 如果原始值 originalValue 是 TypedStringValue，且转换后的值
         // convertedValue 不是 Collection 或数组类型，则将转换后的值存入到 pv 中。
         else if (convertible && originalValue instanceof TypedStringValue &&
               !((TypedStringValue) originalValue).isDynamic() &&
               !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
            pv.setConvertedValue(convertedValue);
            deepCopy.add(pv);
         }
         else {
            resolveNecessary = true;
            deepCopy.add(new PropertyValue(pv, convertedValue));
         }
      }
   }
   if (mpvs != null && !resolveNecessary) {
      mpvs.setConverted(); // mpvs标记为已转换状态
   }

   // Set our (possibly massaged) deep copy.
   try {
      // 将所有的属性值设置到 bean 实例中
      bw.setPropertyValues(new MutablePropertyValues(deepCopy));
   }
   catch (BeansException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
   }
}
```

applyPropertyValues方法中使用BeanDefinitionValueResolver来解析属性值，例如将RuntimeBeanReference解析为实际的Bean实例引用。

```java
// BeanDefinitionValueResolver类
// 解析属性value，如RuntimeBeanReference、RuntimeBeanNameReference等。
@Nullable
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
   // We must check each value to see whether it requires a runtime reference
   // to another bean to be resolved.
   if (value instanceof RuntimeBeanReference) {
      RuntimeBeanReference ref = (RuntimeBeanReference) value;
      // 1.解析RuntimeBeanReference为bean实例
      return resolveReference(argName, ref);
   }
   else if (value instanceof RuntimeBeanNameReference) {
      String refName = ((RuntimeBeanNameReference) value).getBeanName();
      refName = String.valueOf(doEvaluate(refName)); // SpEL计算
      if (!this.beanFactory.containsBean(refName)) {
         throw new BeanDefinitionStoreException(
               "Invalid bean name '" + refName + "' in bean reference for " + argName);
      }
      // 2.返回BeanName
      return refName;
   }
   else if (value instanceof BeanDefinitionHolder) {
      // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
      BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
      return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition()); // 3.解析inner bean
   }
   else if (value instanceof BeanDefinition) {
      // Resolve plain BeanDefinition, without contained name: use dummy(虚假的) name.
      BeanDefinition bd = (BeanDefinition) value;
      String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
            ObjectUtils.getIdentityHexString(bd);
      return resolveInnerBean(argName, innerBeanName, bd); // 3.解析inner bean
   }
   else if (value instanceof ManagedArray) { // <array>标签
      // May need to resolve contained runtime references.
      ManagedArray array = (ManagedArray) value;
      Class<?> elementType = array.resolvedElementType; // 元素类型
      if (elementType == null) {
         String elementTypeName = array.getElementTypeName();
         if (StringUtils.hasText(elementTypeName)) {
            try {
               elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
               array.resolvedElementType = elementType;
            }
            catch (Throwable ex) {
               // Improve the message by showing the context.
               throw new BeanCreationException(
                     this.beanDefinition.getResourceDescription(), this.beanName,
                     "Error resolving array type for " + argName, ex);
            }
         }
         else {
            elementType = Object.class;
         }
      }
      return resolveManagedArray(argName, (List<?>) value, elementType);
   }
   else if (value instanceof ManagedList) { // <list>标签
      // May need to resolve contained runtime references.
      return resolveManagedList(argName, (List<?>) value);
   }
   else if (value instanceof ManagedSet) { // <set>标签
      // May need to resolve contained runtime references.
      return resolveManagedSet(argName, (Set<?>) value);
   }
   else if (value instanceof ManagedMap) { // <map>标签
      // May need to resolve contained runtime references.
      return resolveManagedMap(argName, (Map<?, ?>) value);
   }
   else if (value instanceof ManagedProperties) { // <props>标签
      Properties original = (Properties) value;
      Properties copy = new Properties();
      original.forEach((propKey, propValue) -> {
         if (propKey instanceof TypedStringValue) {
            propKey = evaluate((TypedStringValue) propKey);
         }
         if (propValue instanceof TypedStringValue) {
            propValue = evaluate((TypedStringValue) propValue);
         }
         if (propKey == null || propValue == null) {
            throw new BeanCreationException(
                  this.beanDefinition.getResourceDescription(), this.beanName,
                  "Error converting Properties key/value pair for " + argName + ": resolved to null");
         }
         copy.put(propKey, propValue);
      });
      return copy;
   }
   else if (value instanceof TypedStringValue) { // TypedStringValue
      // Convert value to target type here. 将value转为目标类型
      TypedStringValue typedStringValue = (TypedStringValue) value;
      Object valueObject = evaluate(typedStringValue); // 值
      try {
         Class<?> resolvedTargetType = resolveTargetType(typedStringValue); // 获取目标类型
         if (resolvedTargetType != null) {
            // 类型转换
            return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
         }
         else {
            return valueObject;
         }
      }
      catch (Throwable ex) {
         // Improve the message by showing the context.
         throw new BeanCreationException(
               this.beanDefinition.getResourceDescription(), this.beanName,
               "Error converting typed String value for " + argName, ex);
      }
   }
   else if (value instanceof NullBean) {
      return null;
   }
   else {
      return evaluate(value);
   }
}
```

##### 初始化Bean(initializeBean)

属性注入完成后，这一步就是处理各种回调了。

```java
// AbstractAutowireCapableBeanFactory类
// 初始化Bean，应用各种PostProcessor、init method
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      // 1.调用xxxAware接口回调方法
      // 如果bean实现了BeanNameAware、BeanClassLoaderAware或BeanFactoryAware接口，回调
      invokeAwareMethods(beanName, bean);
   }

   // 2.执行postProcessBeforeInitialization回调方法
   // BeanPostProcessor的postProcessBeforeInitialization 回调
   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 3.调用InitializingBean.afterPropertiesSet()方法和自定义的初始化方法init-method
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   // 4.执行postProcessAfterInitialization方法
   // BeanPostProcessor的postProcessAfterInitialization回调
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

调用xxxAware的回调方法：

```java
// AbstractAutowireCapableBeanFactory类
// 调用xxxAware的回调方法
private void invokeAwareMethods(final String beanName, final Object bean) {
   if (bean instanceof Aware) {
      if (bean instanceof BeanNameAware) {
         ((BeanNameAware) bean).setBeanName(beanName);
      }
      if (bean instanceof BeanClassLoaderAware) {
         ClassLoader bcl = getBeanClassLoader();
         if (bcl != null) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
         }
      }
      if (bean instanceof BeanFactoryAware) {
         ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
      }
   }
}
```

执行BeanPostProcessor的postProcessBeforeInitialization回调方法：

```java
// AbstractAutowireCapableBeanFactory类
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {
   // 应用BeanPostProcessor.postProcessBeforeInitialization方法
   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessBeforeInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

调用InitializingBean.afterPropertiesSet()方法和自定义的初始化方法init-method：

```java
// AbstractAutowireCapableBeanFactory类
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
      throws Throwable {

   // 1.调用InitializingBean.afterPropertiesSet()方法
   boolean isInitializingBean = (bean instanceof InitializingBean);
   if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
      if (logger.isTraceEnabled()) {
         logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
      }
      if (System.getSecurityManager() != null) {
         try {
            AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
               ((InitializingBean) bean).afterPropertiesSet();
               return null;
            }, getAccessControlContext());
         }
         catch (PrivilegedActionException pae) {
            throw pae.getException();
         }
      }
      else {
         ((InitializingBean) bean).afterPropertiesSet(); // 调用
      }
   }

   // 2.执行自定义的初始化方法
   if (mbd != null && bean.getClass() != NullBean.class) {
      String initMethodName = mbd.getInitMethodName();
      if (StringUtils.hasLength(initMethodName) &&
            !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
            !mbd.isExternallyManagedInitMethod(initMethodName)) {
         invokeCustomInitMethod(beanName, bean, mbd);
      }
   }
}
// AbstractAutowireCapableBeanFactory类
// 调用自定义的init method
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
		throws Throwable {

	String initMethodName = mbd.getInitMethodName();
	Assert.state(initMethodName != null, "No init method set");
	final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
			BeanUtils.findMethod(bean.getClass(), initMethodName) :
			ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

	if (initMethod == null) {
		if (mbd.isEnforceInitMethod()) {
			throw new BeanDefinitionValidationException("Could not find an init method named '" +
					initMethodName + "' on bean with name '" + beanName + "'");
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No default init method named '" + initMethodName +
						"' found on bean with name '" + beanName + "'");
			}
			// Ignore non-existent default lifecycle methods.
			return;
		}
	}

	if (logger.isTraceEnabled()) {
		logger.trace("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
	}

	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			ReflectionUtils.makeAccessible(initMethod);
			return null;
		});
		try {
			AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
				initMethod.invoke(bean), getAccessControlContext());
		}
		catch (PrivilegedActionException pae) {
			InvocationTargetException ex = (InvocationTargetException) pae.getException();
			throw ex.getTargetException();
		}
	}
	else {
		try {
			// 调用初始化方法
			ReflectionUtils.makeAccessible(initMethod);
			initMethod.invoke(bean);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}
```

执行BeanPostProcessor的postProcessAfterInitialization回调方法：

```java
// AbstractAutowireCapableBeanFactory类
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {
   // 应用BeanPostProcessor.postProcessAfterInitialization方法
   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessAfterInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

至此，**初始化所有非懒加载的单例Beans**流程中bean的整个初始化过程结束。

### 12.完成上下文的刷新

完成上下文的刷新，启动LifeCycle Beans，发布ContextRefreshedEvent事件。

```java
// AbstractApplicationContext类
protected void finishRefresh() {
   // Clear context-level resource caches (such as ASM metadata from scanning).
   clearResourceCaches(); // 清除Resource Loader的资源缓存

   // Initialize lifecycle processor for this context. 初始化LifecycleProcessor
   initLifecycleProcessor();

   // Propagate refresh to lifecycle processor first.
   getLifecycleProcessor().onRefresh(); // 调用DefaultLifecycleProcessor的onRefresh()方法，启动LifeCycle Beans

   // Publish the final event. 发布ContextRefreshedEvent事件
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   LiveBeansView.registerApplicationContext(this); // 注册MBean
}
```

至此，整个Spring IoC容器的启动过程分析完毕。



## 整理的资料

- [ApplicationContext和BeanFactory继承体系(UML图)](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/ApplicationContext%E5%92%8CBeanFactory%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB.png)
- [ApplicationContext.refresh()总体流程(思维导图)](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/ApplicationContext.refresh()%E6%80%BB%E4%BD%93%E6%B5%81%E7%A8%8B.xmind)



## 参考文献

- [Spring IOC 容器源码分析](https://javadoop.com/post/spring-ioc)
- [Spring启动过程分析1(overview)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%901(overview).html)
- [Spring启动过程分析2(prepareBeanFactory)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%902(prepareBeanFactory).html)
- [Spring启动过程分析3(invokeBeanFactoryPostProcessors)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%903(invokeBeanFactoryPostProcessors).html)
- [Spring启动过程分析4(registerBeanPostProcessors)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%904(registerBeanPostProcessors).html)
- [Spring启动过程分析5(finishBeanFactoryInitialization)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%905(finishBeanFactoryInitialization).html)
- [Spring启动过程分析6(finishRefresh)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%906(finishRefresh).html)
- [Spring IOC 容器源码分析系列文章导读(TODO待看)](http://www.tianxiaobo.com/2018/05/30/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/)
- [Spring源码-IOC容器(TODO 待看)](https://my.oschina.net/u/2377110/blog/902073)



## TODO

- 分析@Autowired注解的属性注入的过程 AutowiredAnnotationBeanPostProcessor
- `<context:component-scan base-package=""/>`的原理
- Bean循环依赖及解决办法
- ConfigurationClassPostProcessor分析(重点)