---
layout:     post
title:      Spring AOP源码分析(三)——拦截器链的执行过程分析
subtitle:   JdkDynamicAopProxy/ReflectiveMethodInvocation
date:       2019-07-27
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring AOP源码分析(三)——拦截器链的执行过程分析

前面两篇文章分别介绍了Spring AOP是如何为目标bean筛选合适的通知器，以及如何创建代理对象的过程。现在得到了bean的代理对象，接下来就是代理对象的方法调用及通知的执行。通知可能在目标方法前执行，也可能在目标方法后执行。具体的执行时机，取决于用户的配置。当目标方法被多个通知匹配到时，Spring通过引入拦截器链来保证每个通知的正常执行。本文将会通过源码分析Spring AOP是如何支持expose-proxy配置的，以及通知与拦截器之间的关系，拦截器链的执行过程等。

### 一、背景知识

关于expose-proxy配置，先来说说它有什么用，然后再来说说怎么用。Spring引入expose-proxy特性是为了解决**目标方法调用同对象中其他方法时，其他方法的切面逻辑无法执行的问题**。下面通过示例来演示一下它的用法，先来看看expose-proxy的配置：

```java
<bean id="hello" class="xyz.coolblog.aop.Hello"/>
<bean id="aopCode" class="xyz.coolblog.aop.AopCode"/>

<aop:aspectj-autoproxy expose-proxy="true" />

<aop:config expose-proxy="true">
    <aop:aspect id="myaspect" ref="aopCode">
        <aop:pointcut id="helloPointcut" expression="execution(* xyz.coolblog.aop.*.hello*(..))" />
        <aop:before method="before" pointcut-ref="helloPointcut" />
    </aop:aspect>
</aop:config>
```

如上，expose-proxy可配置在 `<aop:config/>` 和 `<aop:aspectj-autoproxy />` 标签上。在使用expose-proxy时，需要对内部调用进行改造，比如：

```java
public class Hello implements IHello {

    @Override
    public void hello() {
        System.out.println("hello");
        this.hello("world");
    }

    @Override
    public void hello(String hello) {
        System.out.println("hello " +  hello);
    }
}
```

hello()方法调用了同类中的另一个方法hello(String)，**此时hello(String)上的切面逻辑就无法执行**。这里需要对hello()方法进行改造，强制它调用代理对象中的hello(String)。改造结果如下：

```java
public class Hello implements IHello {

    @Override
    public void hello() {
        System.out.println("hello");
        ((IHello) AopContext.currentProxy()).hello("world");
    }

    @Override
    public void hello(String hello) {
        System.out.println("hello " +  hello);
    }
}
```

如上，AopContext.currentProxy()用于获取当前的代理对象。当expose-proxy被配置为true时，该代理对象会被放入ThreadLocal(AopContext)中。

### 二、拦截器链的执行过程分析

下面分析的源码来自JdkDynamicAopProxy，至于CglibAopProxy中的源码，有机会再分析。

##### 1. JDK动态代理的方法调用

对于JDK Proxy生成的动态代理，其方法调用时，执行逻辑封装在InvocationHandler接口实现类的invoke方法中。JdkDynamicAopProxy实现了InvocationHandler接口，下面就来分析一下JdkDynamicAopProxy的invoke方法。如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation; // 方法调用
   Object oldProxy = null; // 旧的代理对象
   boolean setProxyContext = false;

   TargetSource targetSource = this.advised.targetSource;
   Object target = null; // 目标对象

   try {
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         // AOP不代理equals方法，直接使用JdkDynamicAopProxy实例的equals方法
         return equals(args[0]);
      }
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         // AOP不代理hashCode方法，直接使用JdkDynamicAopProxy实例的hashCode方法
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) { // getDecoratedClass方法
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // 方法的声明类是Advised接口或者TargetClassAware接口，以this.advised为实例调用该方法
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }

      Object retVal; // 方法返回值

      // 如果expose-proxy属性为true，则暴露代理对象
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy); // 向AopContext中设置代理对象
         setProxyContext = true;
      }

      // Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      target = targetSource.getTarget(); // 目标对象
      Class<?> targetClass = (target != null ? target.getClass() : null);

      // Get the interception chain for this method. 获取适合当前方法的拦截器
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      // 如果拦截器链为空，则直接执行目标方法
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         // 适配方法参数
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse); // 反射调用
      }
      else {
         // We need to create a method invocation... 创建一个方法调用，并将拦截器链传入其中
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain. 执行拦截器链
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      Class<?> returnType = method.getReturnType();  // 获取方法返回值类型
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         // 如果方法返回值为this，即 return this; 则将代理对象proxy赋值给retVal
         retVal = proxy;
      }
      // 如果返回值类型为基础类型，比如int，long等，当返回值为null，抛出异常
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
   }
   finally {
      if (target != null && !targetSource.isStatic()) {
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy. // 还原老的代理
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

上面invoke方法的执行流程如下：

1. 检测expose-proxy配置是否为true，若为true，则暴露代理对象；
2. 获取适合当前方法的拦截器；
3. 如果拦截器链为空，则直接通过反射执行目标方法；
4. 若拦截器链不为空，则创建ReflectiveMethodInvocation 对象；
5. 调用ReflectiveMethodInvocation对象的proceed()方法启动拦截器链；
6. 处理返回值，并返回该值。

在以上步骤中，重点关注第2步和第5步中的逻辑。

##### 2.获取调用方法的拦截器

拦截器是指用于对目标方法的调用进行拦截的一种工具。下面给出前置通知拦截器的代码：

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

   /** 方法前置通知 */
   private final MethodBeforeAdvice advice;

   public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
      Assert.notNull(advice, "Advice must not be null");
      this.advice = advice;
   }

   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
      // 执行前置通知逻辑
      this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
      // 通过MethodInvocation调用下一个拦截器，若所有拦截器均执行完，则调用目标方法
      return mi.proceed();
   }
}
```

如上，前置通知的逻辑在目标方法执行前被执行。这里先简单介绍一下拦截器的作用，关于拦截器更多的描述将在后面分析。下面先来看看如何获取拦截器：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
   MethodCacheKey cacheKey = new MethodCacheKey(method);
   List<Object> cached = this.methodCache.get(cacheKey); // 从缓存中获取
   if (cached == null) {  // 缓存未命中，则进行下一步处理
      // 获取所有的拦截器
      cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, method, targetClass);
      this.methodCache.put(cacheKey, cached); // 存入缓存
   }
   return cached;
}

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
		Advised config, Method method, @Nullable Class<?> targetClass) {

	// This is somewhat tricky... We have to process introductions first,
	// but we need to preserve order in the ultimate list.
	// registry为DefaultAdvisorAdapterRegistry类型
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	Advisor[] advisors = config.getAdvisors();
	List<Object> interceptorList = new ArrayList<>(advisors.length);
	Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	Boolean hasIntroductions = null;

	for (Advisor advisor : advisors) { // 遍历通知器列表
		if (advisor instanceof PointcutAdvisor) {
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			// 调用ClassFilter对bean类型进行匹配，无法匹配则说明当前通知器不适合应用在当前bean上。已经预过滤的直接进入下面的逻辑
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				boolean match;
				if (mm instanceof IntroductionAwareMethodMatcher) {
					if (hasIntroductions == null) {
						hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
					}
					match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
				}
				else {
					// 通过方法匹配器对目标方法进行匹配
					match = mm.matches(method, actualClass);
				}
				if (match) {
					// 将advisor中的advice转成相应的拦截器
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					if (mm.isRuntime()) { // 运行时需要调用3个参数的match方法进行匹配
						// Creating a new object instance in the getInterceptors() method
						// isn't a problem as we normally cache created chains.
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			// IntroductionAdvisor类型的通知器，仅需进行类级别的匹配即可
			if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}

	return interceptorList;
}

public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
	List<MethodInterceptor> interceptors = new ArrayList<>(3);
	Advice advice = advisor.getAdvice();
	// 若advice是MethodInterceptor类型的，直接添加到interceptors中即可。
    // 比如AspectJAfterAdvice就实现了MethodInterceptor接口
	if (advice instanceof MethodInterceptor) {
		interceptors.add((MethodInterceptor) advice);
	}
	// 对于AspectJMethodBeforeAdvice等类型的通知，由于没有实现MethodInterceptor
    // 接口，所以这里需要通过适配器进行转换
	for (AdvisorAdapter adapter : this.adapters) {
		if (adapter.supportsAdvice(advice)) {
			interceptors.add(adapter.getInterceptor(advisor));
		}
	}
	if (interceptors.isEmpty()) {
		throw new UnknownAdviceTypeException(advisor.getAdvice());
	}
	return interceptors.toArray(new MethodInterceptor[0]);
}
```

以上就是获取拦截器的过程。这里简单总结一下以上代码的执行过程：

1. 从缓存中获取当前方法的拦截器链；
2. 若缓存未命中，则调用getInterceptorsAndDynamicInterceptionAdvice获取拦截器链；
3. 遍历通知器列表；
4. 对于PointcutAdvisor类型的通知器，这里要调用通知器所持有的切点(Pointcut)对类和方法进行匹配；对于IntroductionAdvisor，只需进行类级别的匹配即可。匹配成功说明应向当前方法织入通知逻辑。
5. 调用getInterceptors方法，将Advisor转换为MethodInterceptor；
6. 返回拦截器数组，并在随后存入缓存中。

这里需要说明一下，部分通知器是没有实现MethodInterceptor接口的，比如AspectJMethodBeforeAdvice。可以看一下前置通知适配器是如何将前置通知转为拦截器的，如下：

```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

   @Override
   public boolean supportsAdvice(Advice advice) {
      return (advice instanceof MethodBeforeAdvice);
   }

   @Override
   public MethodInterceptor getInterceptor(Advisor advisor) {
      MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
      // 创建MethodBeforeAdviceInterceptor拦截器
      return new MethodBeforeAdviceInterceptor(advice);
   }

}
```

如上所示，适配器的逻辑比较简单。现在已经获得了拦截器链，那接下来要做的事情就是启动拦截器。

##### 3. 启动拦截器链

###### (1) 执行拦截器链

首先介绍一下ReflectiveMethodInvocation。ReflectiveMethodInvocation贯穿于拦截器链执行的始终。该类的 proceed方法用于启动拦截器链的执行，下面就去看看这个方法的逻辑。

```java
private int currentInterceptorIndex = -1;

public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early. 
   // 拦截器链中的最后一个拦截器执行完后，即可执行目标方法
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint(); // 调用目标方法
   }

   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
      // 调用具有三个参数(3-args)的matches方法动态匹配目标方法，
      // 两个参数(2-args)的matches方法用于静态匹配
      if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
         return dm.interceptor.invoke(this); // 匹配后调用拦截器逻辑
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed(); // 如果匹配失败，则忽略当前的拦截器，调用下一个拦截器
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      // 调用拦截器逻辑，并传递当前ReflectiveMethodInvocation对象本身
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

如上，proceed方法根据currentInterceptorIndex来确定当前应执行哪个拦截器，并在调用拦截器的invoke方法时，将自己作为参数传给该方法。前面介绍了前置拦截器的代码，这里来看一下后置拦截器：

```java
public class AspectJAfterAdvice extends AbstractAspectJAdvice
      implements MethodInterceptor, AfterAdvice, Serializable {

   public AspectJAfterAdvice(
         Method aspectJBeforeAdviceMethod, AspectJExpressionPointcut pointcut, AspectInstanceFactory aif) {

      super(aspectJBeforeAdviceMethod, pointcut, aif);
   }


   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
      try {
         return mi.proceed(); // 调用proceed方法
      }
      finally {
         // 调用后置通知逻辑
         invokeAdviceMethod(getJoinPointMatch(), null, null);
      }
   }

   @Override
   public boolean isBeforeAdvice() {
      return false;
   }

   @Override
   public boolean isAfterAdvice() {
      return true;
   }

}
```

如上，由于后置通知需要在目标方法返回后执行，所以AspectJAfterAdvice先调用mi.proceed()执行下一个拦截器逻辑，等下一个拦截器返回后，再执行后置通知逻辑。这里举个例子，假设目标方法method在执行前，需要执行两个前置通知和一个后置通知。下面看一下由三个拦截器组成的拦截器链是如何执行的： 

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-7.png?x-oss-process=style/markdown-pic)

注：这里用advice.after()表示执行后置通知。

本部分的最后，介绍一个特殊的拦截器，即ExposeInvocationInterceptor。在[Spring AOP源码分析(一)——筛选合适的通知器](https://xuanjian1992.top/2019/07/27/Spring-AOP%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(%E4%B8%80)-%E7%AD%9B%E9%80%89%E5%90%88%E9%80%82%E7%9A%84%E9%80%9A%E7%9F%A5%E5%99%A8/)这篇文章中，在介绍extendAdvisors方法时，有一个点没有详细说明。这里再贴一下 extendAdvisors方法的代码，如下：

```java
protected void extendAdvisors(List<Advisor> candidateAdvisors) {
   AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(candidateAdvisors);
}

public static boolean makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
	// Don't add advisors to an empty list; may indicate that proxying is just not required
  // 如果通知器列表是一个空列表，则啥都不做
	if (!advisors.isEmpty()) {
		boolean foundAspectJAdvice = false;
		// 下面的for循环用于检测advisors列表中是否存在
		// AspectJ类型的Advisor或Advice
		for (Advisor advisor : advisors) {
			// Be careful not to get the Advice without a guard, as
			// this might eagerly instantiate a non-singleton AspectJ aspect
			if (isAspectJAdvice(advisor)) {
				foundAspectJAdvice = true;
			}
		}
		// 向 advisors 列表的首部添加 DefaultPointcutAdvisor，
		if (foundAspectJAdvice && !advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
			advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
			return true;
		}
	}
	return false;
}

private static boolean isAspectJAdvice(Advisor advisor) {
	return (advisor instanceof InstantiationModelAwarePointcutAdvisor ||
			advisor.getAdvice() instanceof AbstractAspectJAdvice ||
			(advisor instanceof PointcutAdvisor &&
					((PointcutAdvisor) advisor).getPointcut() instanceof AspectJExpressionPointcut));
}
```

如上，extendAdvisors方法所调用的方法会向通知器列表首部添加ExposeInvocationInterceptor.ADVISOR。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-9.png)

现在来看下ExposeInvocationInterceptor的源码：

```java
public final class ExposeInvocationInterceptor implements MethodInterceptor, PriorityOrdered, Serializable {

   public static final ExposeInvocationInterceptor INSTANCE = new ExposeInvocationInterceptor();

   // 创建DefaultPointcutAdvisor对象
   public static final Advisor ADVISOR = new DefaultPointcutAdvisor(INSTANCE) {
      @Override
      public String toString() {
         return ExposeInvocationInterceptor.class.getName() +".ADVISOR";
      }
   };

   private static final ThreadLocal<MethodInvocation> invocation =
         new NamedThreadLocal<>("Current AOP method invocation");


   public static MethodInvocation currentInvocation() throws IllegalStateException {
      MethodInvocation mi = invocation.get();
      if (mi == null) {
         throw new IllegalStateException(
               "No MethodInvocation found: Check that an AOP invocation is in progress, and that the " +
               "ExposeInvocationInterceptor is upfront in the interceptor chain. Specifically, note that " +
               "advices with order HIGHEST_PRECEDENCE will execute before ExposeInvocationInterceptor!");
      }
      return mi;
   }


   // 私有构造方法
   private ExposeInvocationInterceptor() {
   }

   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
      MethodInvocation oldInvocation = invocation.get(); // 旧的方法调用
      invocation.set(mi); // 设置新方法调用到ThreadLocal
      try {
         return mi.proceed(); // 执行拦截器链
      }
      finally {
         invocation.set(oldInvocation); // 还原旧的方法调用
      }
   }

   @Override
   public int getOrder() {
      return PriorityOrdered.HIGHEST_PRECEDENCE + 1;
   }

   private Object readResolve() {
      return INSTANCE;
   }

}
```

如上，ExposeInvocationInterceptor.ADVISOR经过DefaultAdvisorAdapterRegistry.getInterceptors方法(前面已分析)处理后，即可得到ExposeInvocationInterceptor。ExposeInvocationInterceptor的作用是用于暴露 MethodInvocation对象到ThreadLocal中。如果其他地方需要使用当前的MethodInvocation对象，直接通过调用 currentInvocation方法取出即可。

拦截器链的执行过程分析结束，下面来看一下目标方法的执行过程。

###### (2) 执行目标方法

```java
protected Object invokeJoinpoint() throws Throwable {
	return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}

public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args)
      throws Throwable {

   // Use reflection to invoke the method.
   try {
      // 反射调用连接点方法
      ReflectionUtils.makeAccessible(method);
      return method.invoke(target, args);
   }
   catch (InvocationTargetException ex) {
      // Invoked method threw a checked exception.
      // We must rethrow it. The client won't see the interceptor.
      throw ex.getTargetException();
   }
   catch (IllegalArgumentException ex) {
      throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
            method + "] on target [" + target + "]", ex);
   }
   catch (IllegalAccessException ex) {
      throw new AopInvocationException("Could not access method [" + method + "]", ex);
   }
}
```

至此，代理对象的方法调用及拦截器链的执行过程分析结束。

### 参考文章

- [Spring-AOP-源码分析-拦截器链的执行过程](http://www.tianxiaobo.com/2018/06/22/Spring-AOP-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%8B%A6%E6%88%AA%E5%99%A8%E9%93%BE%E7%9A%84%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/)
- [Spring AOP 源码解析](https://javadoop.com/post/spring-aop-source)

