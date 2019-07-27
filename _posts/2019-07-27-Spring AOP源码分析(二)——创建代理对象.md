---
layout:     post
title:      Spring AOP源码分析(二)——创建代理对象
subtitle:   AbstractAutoProxyCreator/ProxyFactory/JdkDynamicAopProxy/CglibAopProxy
date:       2019-07-27
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring AOP源码分析(二)——创建代理对象

上一篇文章[Spring AOP源码分析(一)——筛选合适的通知器](https://xuanjian1992.top/2019/07/27/Spring-AOP%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(%E4%B8%80)-%E7%AD%9B%E9%80%89%E5%90%88%E9%80%82%E7%9A%84%E9%80%9A%E7%9F%A5%E5%99%A8/)，分析了Spring是如何为目标bean筛选合适的通知器的。现在通知器选好了，接下来就要通过动态代理的方式将通知器(Advisor)所持有的通知(Advice)织入到bean的某些方法前后(即**创建代理对象**)。与筛选合适的通知器相比，创建代理对象的过程则要简单不少。下面先来了解一下相关的背景知识，再来分析创建代理对象的过程。

### 一、背景知识

##### 1. proxy-target-class

在Spring AOP配置中，proxy-target-class属性可影响Spring生成的代理对象的类型(代理由CGLib生成或JDK Proxy生成)。以XML配置为例，可进行如下配置：

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>

<aop:config proxy-target-class="true">
    <aop:aspect id="xxx" ref="xxxx">
        <!-- 省略 -->
    </aop:aspect>
</aop:config>
```

如上，默认情况下proxy-target-class属性为false。当目标bean实现了接口时，Spring会基于JDK动态代理为目标 bean创建代理对象。若未实现任何接口，Spring则会通过CGLib创建代理。而当proxy-target-class属性设为true时，则会强制Spring通过CGLib的方式创建代理对象，即使目标bean实现了接口。

关于proxy-target-class属性的用途介绍完毕，下面来看看两种不同创建动态代理的方式。

##### 2. 动态代理

##### (1) 基于JDK的动态代理

基于JDK的动态代理主要是通过JDK提供的代理创建类Proxy为目标对象创建代理，下面来看一下Proxy中创建代理的方法声明。如下：

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, 	
                                      InvocationHandler h)
```

介绍一下上面的参数列表：

- loader——类加载器；
- interfaces——目标类所实现的接口列表；
- h——用于封装代理的处理逻辑。

JDK动态代理对目标类是有一定要求的，即要求目标类必须实现了接口，JDK动态代理只能为实现了接口的目标类生成代理对象。至于InvocationHandler，是一个接口类型，定义了一个invoke方法。使用者需要实现该方法，并在其中封装代理逻辑。

关于JDK动态代理的介绍，就先说到这。下面来演示一下JDK 动态代理的使用方式，如下：

```java
public interface UserService {

    void save(User user);

    void update(User user);
}

public class UserServiceImpl implements UserService {

    @Override
    public void save(User user) {
        System.out.println("save user info");
    }

    @Override
    public void update(User user) {
        System.out.println("update user info");
    }
}
```

代理创建者定义：

```java
public interface ProxyCreator {

    Object getProxy();
}

public class JdkProxyCreator implements ProxyCreator, InvocationHandler {

    private Object target;

    public JdkProxyCreator(Object target) {
        assert target != null;
        Class<?>[] interfaces = target.getClass().getInterfaces();
        if (interfaces.length == 0) {
            throw new IllegalArgumentException("target class don`t implement any interface");
        }
        this.target = target;
    }

    @Override
    public Object getProxy() {
        Class<?> clazz = target.getClass();
        // 生成代理对象
        return Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println(System.currentTimeMillis() + " - " + method.getName() + " method start");
        // 调用目标方法
        Object retVal = method.invoke(target, args);
        System.out.println(System.currentTimeMillis() + " - " + method.getName() + " method over");

        return retVal;
    }
}
```

如上，invoke方法中的代理逻辑主要用于记录目标方法的调用时间和结束时间。下面写测试代码简单验证一下：

```java
public class JdkProxyCreatorTest {

    @Test
    public void getProxy() throws Exception {
        ProxyCreator proxyCreator = new JdkProxyCreator(new UserServiceImpl());
        UserService userService = (UserService) proxyCreator.getProxy();
        
        System.out.println("proxy type = " + userService.getClass());
        System.out.println();
        userService.save(null);
        System.out.println();
        userService.update(null);
    }
}
```

测试结果如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-4.png)

从测试结果中可以看出，代理逻辑得到正常执行。另外，注意一下userService指向对象的类型，并非是 xyz.coolblog.proxy.UserServiceImpl，而是**com.sun.proxy.$Proxy4(Proxy子类)**。

##### (2) 基于CGLib的动态代理

当要为未实现接口的类生成代理时，就无法使用JDK动态代理。那么此类的目标对象生成代理时就需要使用CGLib。在CGLib中，代理逻辑是封装在MethodInterceptor实现类中的，代理对象则是通过Enhancer类的create方法进行创建。下面演示一下CGLib创建代理对象的过程：

被代理类：

```java
public class Tank59 {
    void run() {
        System.out.println("极速前行中....");
    }

    void shoot() {
        System.out.println("轰...轰...轰...轰...");
    }
}
```

CGLib代理创建者：

```java
public class CglibProxyCreator implements ProxyCreator {

    private Object target;

    private MethodInterceptor methodInterceptor; // 拦截器

    public CglibProxyCreator(Object target, MethodInterceptor methodInterceptor) {
        assert (target != null && methodInterceptor != null);
        this.target = target;
        this.methodInterceptor = methodInterceptor;
    }

    @Override
    public Object getProxy() {
        Enhancer enhancer = new Enhancer();
        // 设置代理类的父类
        enhancer.setSuperclass(target.getClass());
        // 设置代理逻辑
        enhancer.setCallback(methodInterceptor);
        // 创建代理对象
        return enhancer.create();
    }
}
```

方法拦截器MethodInterceptor：

```java
public class TankRemanufacture implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        if (method.getName().equals("run")) {
            System.out.println("正在重造59坦克...");
            System.out.println("重造成功，已获取 59改 之 超音速飞行版");
            System.out.print("已起飞，正在突破音障。");

            methodProxy.invokeSuper(o, objects);

            System.out.println("已击落黑鸟 SR-71，正在返航...");
            return null;
        }

        return methodProxy.invokeSuper(o, objects);
    }
}
```

测试代码如下：

```java
public class CglibProxyCreatorTest {

    @Test
    public void getProxy() throws Exception {
        ProxyCreator proxyCreator = new CglibProxyCreator(new Tank59(), new TankRemanufacture());
        Tank59 tank59 = (Tank59) proxyCreator.getProxy();
        
        System.out.println("proxy class = " + tank59.getClass() + "\n");
        tank59.run();
        System.out.println();
        System.out.print("射击测试：");
        tank59.shoot();
    }
}
```

测试结果如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-5.png)

如上，“极速前行中…” 和 “轰…轰…轰…轰…” 这两行字符串是目标对象中的方法打印出来的，其他的则是由代理逻辑打印的。由此可知，代理逻辑已生效。

### 二、代理对象创建过程分析

Spring AOP在为目标bean创建代理对象前，需要先创建AopProxy对象，然后再调用该对象的getProxy方法创建实际的代理类。先来看看AopProxy这个接口的定义，如下：

```java
public interface AopProxy {

    /** 创建代理对象 */
    Object getProxy();
    
    Object getProxy(ClassLoader classLoader);
}
```

在Spring中，有两个类实现AopProxy，如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-6.png)

Spring AOP在为目标bean创建代理的过程中，要根据bean是否实现接口，以及一些其他配置来决定使用AopProxy何种实现类为目标bean创建代理对象。下面来看一下代理创建的过程，如下：

```java
// AbstractAutoProxyCreator类，创建代理对象
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      // 暴露被代理类
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this); // 复制ProxyConfig配置
   // 判断是否代理 类 或者 接口
   // 默认配置下，或用户显式配置 proxy-target-class = "false" 时，
   // 这里的 proxyFactory.isProxyTargetClass() 也为 false
   if (!proxyFactory.isProxyTargetClass()) { // 如果配置的proxyTargetClass为false，进入下面的判断
      // 检查BeanDefinition preserveTargetClass属性，来判断是否代理类
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 检查接口来判断是否代理接口或者类
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }
   // 构建最终的Advisor数组
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   proxyFactory.addAdvisors(advisors); // 添加advisor
   proxyFactory.setTargetSource(targetSource); // 设置TargetSource
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }
   // 创建代理
   return proxyFactory.getProxy(getProxyClassLoader());
}

public Object getProxy(@Nullable ClassLoader classLoader) {
	// 先创建 AopProxy 实现类对象(如JdkDynamicAopProxy)，然后再调用 getProxy 为目标 bean 创建代理对象
	return createAopProxy().getProxy(classLoader);
}
```

##### (1) 配置proxyTargetClass属性

在创建代理对象时，会先判断是否代理接口或者代理类。

```java
if (!proxyFactory.isProxyTargetClass()) { // 如果配置的proxyTargetClass为false，进入下面的判断
  // 检查BeanDefinition preserveTargetClass属性，来判断是否代理类
  if (shouldProxyTargetClass(beanClass, beanName)) {
     proxyFactory.setProxyTargetClass(true);
  }
  else {
     // 检查接口来判断是否代理接口或者类
     evaluateProxyInterfaces(beanClass, proxyFactory);
  }
}
```

shouldProxyTargetClass方法检查BeanDefinition的preserveTargetClass属性，来判断是否代理类：

```java
protected boolean shouldProxyTargetClass(Class<?> beanClass, @Nullable String beanName) {
   return (this.beanFactory instanceof ConfigurableListableBeanFactory &&
         // 检查preserveTargetClass属性
         AutoProxyUtils.shouldProxyTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName));
}
// AutoProxyUtils类
public static boolean shouldProxyTargetClass(
		ConfigurableListableBeanFactory beanFactory, @Nullable String beanName) {

	if (beanName != null && beanFactory.containsBeanDefinition(beanName)) {
		BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
    // 检查preserveTargetClass属性是否为true
		return Boolean.TRUE.equals(bd.getAttribute(PRESERVE_TARGET_CLASS_ATTRIBUTE));
	}
	return false;
}
```

而evaluateProxyInterfaces方法检查接口来判断是否代理接口或者类：

```java
// 评估代理接口
protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
   // 获取beanClass及其父类实现的所有接口
   Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
   boolean hasReasonableProxyInterface = false;
   for (Class<?> ifc : targetInterfaces) {
      // 判断是否存在合理的代理接口(去除容器回调接口、内部语言接口)
      if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
            ifc.getMethods().length > 0) {
         hasReasonableProxyInterface = true;
         break;
      }
   }
   if (hasReasonableProxyInterface) {
      // Must allow for introductions; can't just set interfaces to the target's interfaces only.
      for (Class<?> ifc : targetInterfaces) {
         proxyFactory.addInterface(ifc); // 如果存在合理的代理接口，则添加代理接口
      }
   }
   else {
      proxyFactory.setProxyTargetClass(true); // 代理类
   }
}
```

##### (2) 构建最终的Advisor数组

构建最终的Advisor数组，将MethodInterceptor、Advice适配为Advisor。

```java
protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
   // Handle prototypes correctly...
   Advisor[] commonInterceptors = resolveInterceptorNames(); // 解析通过setInterceptorNames设置的拦截器

   List<Object> allInterceptors = new ArrayList<>();
   if (specificInterceptors != null) {
      allInterceptors.addAll(Arrays.asList(specificInterceptors));
      if (commonInterceptors.length > 0) {
         // 先应用commonInterceptors中的拦截器 与否
         if (this.applyCommonInterceptorsFirst) {
            allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
         }
         else {
            allInterceptors.addAll(Arrays.asList(commonInterceptors));
         }
      }
   }
   if (logger.isTraceEnabled()) {
      int nrOfCommonInterceptors = commonInterceptors.length;
      int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
      logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
            " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
   }

   Advisor[] advisors = new Advisor[allInterceptors.size()];
   for (int i = 0; i < allInterceptors.size(); i++) {
      // 如果有必要，将MethodInterceptor、Advice，包装为Advisor
      advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
   }
   return advisors;
}
// 解析通过setInterceptorNames设置的拦截器
private Advisor[] resolveInterceptorNames() {
	BeanFactory bf = this.beanFactory;
	ConfigurableBeanFactory cbf = (bf instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) bf : null);
	List<Advisor> advisors = new ArrayList<>();
	for (String beanName : this.interceptorNames) {
		if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
			Assert.state(bf != null, "BeanFactory required for resolving interceptor names");
			Object next = bf.getBean(beanName);
			// 对于interceptorNames中的拦截器(MethodInterceptor、Advice)，包装为Advisor
			advisors.add(this.advisorAdapterRegistry.wrap(next));
		}
	}
	return advisors.toArray(new Advisor[0]);
}
```

##### (3) 创建代理对象

getProxy有两个方法调用，一个是调用createAopProxy创建AopProxy实现类对象，然后再调用AopProxy实现类对象中的getProxy方法创建代理对象。这里先来看一下创建AopProxy实现类对象的过程，如下：

```java
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate(); // 设置标记
   }
   return getAopProxyFactory().createAopProxy(this);
}

// 返回DefaultAopProxyFactory实例
public AopProxyFactory getAopProxyFactory() {
	return this.aopProxyFactory;
}

// DefaultAopProxyFactory类
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	// 下面的三个条件简单分析一下：
	//   条件1：config.isOptimize() - 是否需要优化
	//   条件2：config.isProxyTargetClass() - 检测proxyTargetClass的值，前面的代码会设置这个值
	//   条件3：hasNoUserSuppliedProxyInterfaces(config) - 是否不存在用户定义的代理接口
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			// 创建JDK动态代理
			return new JdkDynamicAopProxy(config);
		}
		// 创建CGLib代理，ObjenesisCglibAopProxy继承自CglibAopProxy
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		// 创建JDK动态代理
		return new JdkDynamicAopProxy(config);
	}
}
```

如上，DefaultAopProxyFactory根据一些条件决定生成什么类型的AopProxy实现类对象。生成好AopProxy实现类对象后，下面就要为目标bean创建代理对象了。这里以JdkDynamicAopProxy为例，来看一下该类的getProxy方法的执行逻辑。

```java
public Object getProxy() {
	return getProxy(ClassUtils.getDefaultClassLoader());
}
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
   }
   // 确定完整的代理接口
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   // 确定所有代理接口中是否定义了equals和hashCode方法
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   // 调用 newProxyInstance 创建代理对象
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

 确定完整的代理接口：

```java
static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised, boolean decoratingProxy) {
   Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces(); // 获取代理接口数组
   if (specifiedInterfaces.length == 0) { // 如果代理接口数组为空，进入下面的判断
      // No user-specified interfaces: check whether target class is an interface.
      Class<?> targetClass = advised.getTargetClass(); // 获取目标类
      if (targetClass != null) {
         if (targetClass.isInterface()) { // 如果目标类是个接口，就设置代理接口为该接口
            advised.setInterfaces(targetClass);
         }
         else if (Proxy.isProxyClass(targetClass)) { // 如果已经是代理类
            advised.setInterfaces(targetClass.getInterfaces());// 则设置代理接口为代理类代理的接口
         }
         specifiedInterfaces = advised.getProxiedInterfaces(); // 获取更新后的代理接口数组
      }
   }
  // 判断是否添加SpringProxy、Advised、DecoratingProxy接口
   boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
   boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
   boolean addDecoratingProxy = (decoratingProxy && !advised.isInterfaceProxied(DecoratingProxy.class));
   int nonUserIfcCount = 0;
   if (addSpringProxy) {
      nonUserIfcCount++;
   }
   if (addAdvised) {
      nonUserIfcCount++;
   }
   if (addDecoratingProxy) {
      nonUserIfcCount++;
   }
   Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
   System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
   int index = specifiedInterfaces.length;
   // 添加SpringProxy、Advised、DecoratingProxy接口
   if (addSpringProxy) {
      proxiedInterfaces[index] = SpringProxy.class;
      index++;
   }
   if (addAdvised) {
      proxiedInterfaces[index] = Advised.class;
      index++;
   }
   if (addDecoratingProxy) {
      proxiedInterfaces[index] = DecoratingProxy.class;
   }
   return proxiedInterfaces;
}
```

确定所有代理接口中是否定义了equals和hashCode方法：

```java
private void findDefinedEqualsAndHashCodeMethods(Class<?>[] proxiedInterfaces) {
   for (Class<?> proxiedInterface : proxiedInterfaces) {
      Method[] methods = proxiedInterface.getDeclaredMethods();
      for (Method method : methods) {
         if (AopUtils.isEqualsMethod(method)) {
            this.equalsDefined = true;
         }
         if (AopUtils.isHashCodeMethod(method)) {
            this.hashCodeDefined = true;
         }
         if (this.equalsDefined && this.hashCodeDefined) {
            return;
         }
      }
   }
}
```

根据如上JdkDynamicAopProxy创建代理的代码，**JdkDynamicAopProxy最终调用Proxy.newProxyInstance方法来创建代理对象**。

至此，创建代理对象的整个过程分析完毕。

### 参考文章

- [Spring-AOP-源码分析-创建代理对象](http://www.tianxiaobo.com/2018/06/20/Spring-AOP-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%88%9B%E5%BB%BA%E4%BB%A3%E7%90%86%E5%AF%B9%E8%B1%A1/)
- [Spring AOP 源码解析](https://javadoop.com/post/spring-aop-source)

