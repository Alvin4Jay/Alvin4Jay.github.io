---
layout:     post
title:      Spring AOP源码分析(一)——筛选合适的通知器
subtitle:   AbstractAutoProxyCreator
date:       2019-07-27
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring AOP源码分析(一)——筛选合适的通知器

本文接上一篇文章[Spring-AOP-使用介绍-从前世到今生](https://xuanjian1992.top/2019/07/22/Spring-AOP-%E4%BD%BF%E7%94%A8%E4%BB%8B%E7%BB%8D-%E4%BB%8E%E5%89%8D%E4%B8%96%E5%88%B0%E4%BB%8A%E7%94%9F/)，开始分析Spring AOP相关的代码实现。主要分为筛选合适的通知器、创建代理和拦截器链的执行过程三个方面。下面先给出一个示例，然后根据示例引出源码分析。

代理示例(xml和注解两种配置)：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-2.png)

最终的beanDefinitionMap Debug结果：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-1.png)

从Debug结果可以看到，蓝色方框对应的是id为userService、logArgsAspect、logResultAspect和logResultAspectAnno的普通Bean定义；绿色方框是xml配置中定义的两个切点，即logResultPointcut和internalPointcut；黄色方框对应的是两个通知，即`<aop:before .../>`和`<aop:after-returning …/>`。这里需要注意，注解@Aspect的切面Bean`logResultAspectAnno`**并不是在ApplicationContext解析xml配置文件的时候就转换成了多个Advisor**，而是在目标对象bean单例预初始化、给目标对象Bean查找合适的Advisor的时候做了处理，并被构建为一个或多个 Advisor(依赖于**AnnotationAwareAspectJAutoProxyCreator**)。

### 一、AOP xml配置的解析

对于如上的AOP xml配置，在Spring Framework中是由AopNamespaceHandler来处理的。

```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {

   // Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
   // '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
   // and '{@code scoped-proxy}' tags.
   @Override
   public void init() {
      // 注解parser
      // In 2.0 XSD as well as in 2.1 XSD.
      registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
      registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
      registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

      // Only in 2.0 XSD: moved to context namespace as of 2.1
      registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
   }

}
```

可以看出，在解析xml配置阶段，`<aop:config .../>`元素是由ConfigBeanDefinitionParser类处理的，它把`<aop:pointcut .../>`解析成AspectJExpressionPointcut、把`<aop:before ../>`和`<aop:after-returning …/>`解析成AspectJPointcutAdvisor，而`<aop:aspectj-autoproxy .../>`是由AspectJAutoProxyBeanDefinitionParser类处理的。AspectJAutoProxyBeanDefinitionParser类做了如下的事情：

```java
class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {

   @Override
   @Nullable
   public BeanDefinition parse(Element element, ParserContext parserContext) {
      // 注册AspectJAutoProxyCreator(AnnotationAwareAspectJAutoProxyCreator)
      AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
      extendBeanDefinition(element, parserContext);
      return null;
   }
}

public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
		ParserContext parserContext, Element sourceElement) {
	// 注册或者更新AspectJAutoProxyCreator
	BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			parserContext.getRegistry(), parserContext.extractSource(sourceElement));
	// 设置相关属性(proxy-target-class、expose-proxy)
	useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
	registerComponentIfNecessary(beanDefinition, parserContext);
}

public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
		BeanDefinitionRegistry registry, @Nullable Object source) {
	// 注册或者更新AspectJAutoProxyCreator
	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

private static BeanDefinition registerOrEscalateApcAsRequired(
		Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	// 判断internalAutoProxyCreator是否存在
	if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
		BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
		if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
			int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName()); // 计算优先级
			int requiredPriority = findPriorityForClass(cls);
			if (currentPriority < requiredPriority) { // 如果新优先级比原来的大，则更新类名
				apcDefinition.setBeanClassName(cls.getName());
			}
		}
		return null;
	}

	// 注册BeanDefinition
	RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
	beanDefinition.setSource(source);
	// 设置最高的顺序
	beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
	beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
	return beanDefinition;
}
```

可以看出AspectJAutoProxyBeanDefinitionParser主要就是注册AnnotationAwareAspectJAutoProxyCreator的BeanDefinition到BeanFactory。

### 二、为目标对象筛选合适的通知器

##### 1.AOP入口分析

对于如上的xml配置，在ClassPathXmlApplicationContext读取该配置之后，就存在了以上的BeanDefinitionMap。在初始化非懒加载的单例时(实例化、初始化)，对于id为userService的目标对象Bean，Spring AOP可能会为其生成代理对象。

从[Spring-IoC容器源码分析](https://xuanjian1992.top/2019/07/14/Spring-IoC%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)这篇文章中可以知道，在单例Bean初始化的过程中，会经过很多生命周期接口的回调处理，如BeanPostprocessor。而Bean代理对象的生成也依赖于BeanPostprocessor处理器的处理(**AOP和IOC的整合点**)，即上文说到的AnnotationAwareAspectJAutoProxyCreator。先看下AnnotationAwareAspectJAutoProxyCreator的类继承图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AOP-3.png)

可见，AnnotationAwareAspectJAutoProxyCreator实现了**BeanPostProcessor**、InstantiationAwareBeanPostProcessor和SmartInstantiationAwareBeanPostProcessor接口，并继承自AbstractAutoProxyCreator抽象类。**而AOP处理的核心逻辑主要在AbstractAutoProxyCreator类里，因此AnnotationAwareAspectJAutoProxyCreator(BeanPostProcessor)是AOP的入口**。本文先分析AOP代理对象生成的第一步，即为目标对象筛选合适通知器。

如下是AnnotationAwareAspectJAutoProxyCreator的父类AbstractAutoProxyCreator的相关方法，这些都是跟目标对象Bean预初始化过程中有关的生命周期回调接口，与AOP代理息息相关。

```java
public Object getEarlyBeanReference(Object bean, String beanName) { 
   // 解决循环依赖问题
   // ... 
}

public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
   // ...
}

public boolean postProcessAfterInstantiation(Object bean, String beanName) {
   return true;
}

public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
   return pvs;
}

public Object postProcessBeforeInitialization(Object bean, String beanName) {
   return bean;
}

public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   // ...
}
```

在目标对象(单例)初始化过程中，以上方法首先执行的是postProcessBeforeInstantiation方法，该方法在createBean方法中执行，而在doCreateBean调用之前：

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
   Object cacheKey = getCacheKey(beanClass, beanName);
       // beanName为空，或者当前方法中未创建对应的代理
   if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
      // cacheKey已经判断过，直接返回null
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
      // 对于实现了Advice，Pointcut，Advisor，AopInfrastructureBean接口的bean或者是切面的bean，不需要代理
      // shouldSkip方法判断是否跳过代理
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }

   // Create proxy here if we have a custom TargetSource.
   // Suppresses unnecessary default instantiation of the target bean:
   // The TargetSource will handle target instances in a custom fashion.
   // 有自定义的TargetSource, 则创建代理
   TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
   if (targetSource != null) {
      if (StringUtils.hasLength(beanName)) {
         this.targetSourcedBeans.add(beanName);
      }
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy; // 这里代理返回后，bean后续的初始化步骤就不执行了，直接使用该代理
   }

   return null;
}
```

postProcessBeforeInstantiation方法的逻辑是：如果存在自定义的TargetSource，则根据自定义的TargetSource创建代理对象并返回，并且这里返回代理后，bean后续的初始化步骤就不执行了。

若postProcessBeforeInstantiation方法返回null，则继续目标对象bean的初始化。并且会在doCreateBean中执行以下代码：

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

这行代码与解决bean之间的循环依赖有关，见[Spring IoC中的循环依赖问题及解决办法](https://xuanjian1992.top/2019/07/20/Spring-IoC%E4%B8%AD%E7%9A%84%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96%E5%8F%8A%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/)。

```java
public Object getEarlyBeanReference(Object bean, String beanName) { // 解决循环依赖问题
   Object cacheKey = getCacheKey(bean.getClass(), beanName);
   if (!this.earlyProxyReferences.contains(cacheKey)) {
      // 将cacheKey添加到earlyProxyReferences中，标记已创建 early 的代理对象
      this.earlyProxyReferences.add(cacheKey); 
   }
   return wrapIfNecessary(bean, beanName, cacheKey); // 创建 early 的代理对象
}
```

AbstractAutoProxyCreator类中的getEarlyBeanReference方法主要用于创建原始目标对象的代理对象(早期引用)，用于解决循环依赖问题。

接下来依次执行以下方法，无特殊逻辑：

```java
public boolean postProcessAfterInstantiation(Object bean, String beanName) {
   return true;
}

public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
   return pvs;
}

public Object postProcessBeforeInitialization(Object bean, String beanName) {
   return bean;
}
```

最重要的是postProcessAfterInitialization方法，一般Spring AOP会在这里为目标对象创建代理对象，即Bean调用初始化方法(init-method)之后。

```java
// 目标对象bean调用初始化方法之后的处理
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      // 不是提早暴露的Bean引用，没涉及循环依赖问题，即没创建早期的代理对象，则在这里创建可能的代理对象
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
// 创建代理对象或者返回原实例
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	// 有自定义的TargetSource且已经创建代理，则不需要再代理
	if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	// 不需要代理，直接返回
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	// 对于实现了Advice，Pointcut，Advisor，AopInfrastructureBean接口(基础设施类)的bean或者是
  // 切面的bean，不需要代理，shouldSkip方法判断是否跳过代理
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		// 将 <cacheKey, FALSE> 键值对放入缓存中，供上面的 if 分支使用
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// Create proxy if we have advice.
	// 1.找出合适的通知器
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) { // 需要代理
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 2.创建代理
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		// 返回代理对象，此时 IOC 容器输入 bean，得到 proxy。此时，
    // beanName 对应的 bean 是代理对象，而非原始的 bean
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
  // specificInterceptors = null，直接返回 bean
	return bean;
}
```

以上就是**Spring AOP主要的创建代理对象的方法**，过程比较简单：

1. 如果不需要创建代理，则直接返回；
2. 若bean是AOP基础设施类型，则直接返回；
3. 为bean查找合适的通知器；
4. 如果通知器数组不为空，则为bean生成代理对象，并返回该对象；
5. 若数组为空，则返回原始 bean。

本篇及后续的文章会对步骤3、4进行分析，下面先分析3的内容。

##### 2.筛选合适的通知器

向目标 bean 中织入通知之前，先要为 bean 筛选出合适的通知器（通知器持有通知）。如何筛选呢？方式由很多，比如可以通过正则表达式匹配方法名，当然更多的时候用的是 AspectJ 表达式进行匹配。那下面就来看一下使用 AspectJ 表达式筛选通知器的过程，如下：

```java
protected Object[] getAdvicesAndAdvisorsForBean(
      Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
	 // 查找合适的通知器
   List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
   if (advisors.isEmpty()) {
      return DO_NOT_PROXY;
   }
   return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	List<Advisor> candidateAdvisors = findCandidateAdvisors(); // 查找所有的通知器
	// 筛选可应用在 beanClass 上的 Advisor，通过 ClassFilter 和 MethodMatcher
 	// 对目标类和方法进行匹配
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	extendAdvisors(eligibleAdvisors); // 扩展操作
	if (!eligibleAdvisors.isEmpty()) {
		eligibleAdvisors = sortAdvisors(eligibleAdvisors); // 排序
	}
	return eligibleAdvisors;
}
```

如上，Spring 先查询出所有的通知器，然后再调用 findAdvisorsThatCanApply 对通知器进行筛选。在下面几节中，将分别对 findCandidateAdvisors 和 findAdvisorsThatCanApply 两个方法进行分析。

###### 2.1 查找通知器

前面示例显示了AOP xml配置和注解配置的区别，即xml配置的AOP会在Bean容器解析xml之后直接解析为AspectJPointcutAdvisor和AspectJExpressionPointcut BeanDefinition，而注解@Aspect的切面在这个时候还是普通的BeanDefinition，它会在目标bean查找通知器的过程中被解析为一个或多个Advisor。查找通知器的逻辑如下：

```java
// AnnotationAwareAspectJAutoProxyCreator类
protected List<Advisor> findCandidateAdvisors() {
   // 调用父类方法从容器中查找所有的通知器
   // Add all the Spring advisors found according to superclass rules.
   List<Advisor> advisors = super.findCandidateAdvisors(); // 找出实现了Advisor接口的类
   // 解析 @Aspect 注解，并构建通知器
   // Build Advisors for all AspectJ aspects in the bean factory.
   if (this.aspectJAdvisorsBuilder != null) {
      advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors()); // buildAspectJAdvisors()重点
   }
   return advisors;
}
```

AnnotationAwareAspectJAutoProxyCreator 覆写了父类AbstractAdvisorAutoProxyCreator的方法 findCandidateAdvisors，并增加了一步操作，即解析 @Aspect 注解，并构建成通知器。下面先来分析一下父类中的 findCandidateAdvisors 方法的逻辑，然后再来分析 buildAspectJAdvisors 方法的逻辑。

####### (1) findCandidateAdvisors方法分析

先来看一下 AbstractAdvisorAutoProxyCreator 中 findCandidateAdvisors 方法的定义，如下：

```java
// AbstractAdvisorAutoProxyCreator类
protected List<Advisor> findCandidateAdvisors() {
   Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
   return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

从上面的源码中可以看出，AbstractAdvisorAutoProxyCreator中的findCandidateAdvisor方法借助于 BeanFactoryAdvisorRetrievalHelper来查找实现了Advisor接口的Bean，即从bean容器中将Advisor类型的bean查找出来。如下：

```java
public List<Advisor> findAdvisorBeans() {
   // Determine list of advisor bean names, if not cached already.
   String[] advisorNames = this.cachedAdvisorBeanNames; // cachedAdvisorBeanNames 是 advisor 名称的缓存
   // 如果 cachedAdvisorBeanNames 为空，这里到容器中查找，
   // 并设置缓存，后续直接使用缓存即可
   if (advisorNames == null) {
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the auto-proxy creator apply to them!
      // 从容器中查找 Advisor 类型 bean 的名称 
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      this.cachedAdvisorBeanNames = advisorNames; // 设置缓存
   }
   if (advisorNames.length == 0) {
      return new ArrayList<>();
   }

   List<Advisor> advisors = new ArrayList<>();
   for (String name : advisorNames) {  // 遍历 advisorNames
      if (isEligibleBean(name)) {
         if (this.beanFactory.isCurrentlyInCreation(name)) { // 忽略正在创建中的 advisor bean
            if (logger.isTraceEnabled()) {
               logger.trace("Skipping currently created advisor '" + name + "'");
            }
         }
         else {
            try {
               // 调用 getBean 方法从容器中获取名称为 name 的 bean，
               // 并将 bean 添加到 advisors 中
               advisors.add(this.beanFactory.getBean(name, Advisor.class));
            }
            catch (BeanCreationException ex) {
               Throwable rootCause = ex.getMostSpecificCause();
               if (rootCause instanceof BeanCurrentlyInCreationException) {
                  BeanCreationException bce = (BeanCreationException) rootCause;
                  String bceBeanName = bce.getBeanName();
                  if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                     if (logger.isTraceEnabled()) {
                        logger.trace("Skipping advisor '" + name +
                              "' with dependency on currently created bean: " + ex.getMessage());
                     }
                     // Ignore: indicates a reference back to the bean we're trying to advise.
                     // We want to find advisors other than the currently created bean itself.
                     continue;
                  }
               }
               throw ex;
            }
         }
      }
   }
   return advisors;
}
```

以上就是从容器中查找 Advisor 类型的 bean 所有的逻辑。主要做了两件事情：

1. 从容器中查找所有类型为Advisor的bean对应的名称(如示例中的`<aop:before ../>`解析出来的**AspectJPointcutAdvisor**)；
2. 遍历 advisorNames，并从容器中获取对应的 bean。

看完上面的分析，继续来分析一下 @Aspect 注解的解析过程。

####### (2) buildAspectJAdvisors方法分析

与上一节的内容相比，解析 @Aspect 注解的过程还是比较复杂的，需要一些耐心去看。下面开始分析 buildAspectJAdvisors方法的源码，如下：

```java
public List<Advisor> buildAspectJAdvisors() {
   List<String> aspectNames = this.aspectBeanNames;

   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
            // 从容器中获取所有 bean 的名称
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) { // 遍历 beanNames
               if (!isEligibleBean(beanName)) {
                  continue;
               }
               // We must be careful not to instantiate beans eagerly as in this case they
               // would be cached by the Spring container but would not have been weaved.
               // 根据 beanName 获取 bean 的类型
               Class<?> beanType = this.beanFactory.getType(beanName);
               if (beanType == null) {
                  continue;
               }
               // 检测 beanType 是否包含 Aspect 注解
               if (this.advisorFactory.isAspect(beanType)) { 
                  aspectNames.add(beanName);
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  // SINGLETON
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) { 
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                     // 获取通知器
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                     if (this.beanFactory.isSingleton(beanName)) {
                        this.advisorsCache.put(beanName, classAdvisors);
                     }
                     else {
                        this.aspectFactoryCache.put(beanName, factory);
                     }
                     advisors.addAll(classAdvisors);
                  }
                  else {
                     // Per target or per this.
                     if (this.beanFactory.isSingleton(beanName)) {
                        throw new IllegalArgumentException("Bean with name '" + beanName +
                              "' is a singleton, but aspect instantiation instantiate.model is not singleton");
                     }
                     MetadataAwareAspectInstanceFactory factory =
                           new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                     this.aspectFactoryCache.put(beanName, factory);
                     advisors.addAll(this.advisorFactory.getAdvisors(factory));
                  }
               }
            }
            this.aspectBeanNames = aspectNames;
            return advisors;
         }
      }
   }

   if (aspectNames.isEmpty()) {
      return Collections.emptyList();
   }
   List<Advisor> advisors = new ArrayList<>();
   for (String aspectName : aspectNames) {
      List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
      if (cachedAdvisors != null) {
         advisors.addAll(cachedAdvisors);
      }
      else {
         MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
         advisors.addAll(this.advisorFactory.getAdvisors(factory));
      }
   }
   return advisors;
}
```

上面就是 buildAspectJAdvisors 的代码，看起来比较长，只需关注重点的方法调用即可。在进行后续的分析前，这里先对 buildAspectJAdvisors 方法的执行流程做个总结。如下：

1. 获取容器中所有bean的名称(beanName)；
2. 遍历上一步获取到的bean名称数组，并获取当前beanName对应的bean类型(beanType)；
3. 根据beanType判断当前bean是否是一个Aspect注解的类，若不是则不做任何处理；
4. 调用advisorFactory.getAdvisors获取通知器(**InstantiationModelAwarePointcutAdvisorImpl实例**)。

下面来重点分析`advisorFactory.getAdvisors(factory)`这个调用，如下：

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   // 获取 aspectClass 和 aspectName
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
   validate(aspectClass); // 校验切面类

   // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
   // so that it will only instantiate once. 只会实例化切面对象一次
   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new ArrayList<>();
   // getAdvisorMethods 用于返回不包含 @Pointcut 注解的方法
   for (Method method : getAdvisorMethods(aspectClass)) { 
      // 为每个方法分别调用 getAdvisor 方法，返回InstantiationModelAwarePointcutAdvisorImpl实例
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   // If it's a per target aspect, emit the dummy instantiating aspect.
   if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
      advisors.add(0, instantiationAdvisor);
   }

   // Find introduction fields.
   for (Field field : aspectClass.getDeclaredFields()) {
      Advisor advisor = getDeclareParentsAdvisor(field); // @DeclareParents注解
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   return advisors;
}

public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
		int declarationOrderInAspect, String aspectName) {

	validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

	// 获取AspectJ表达式切点AspectJExpressionPointcut
	AspectJExpressionPointcut expressionPointcut = getPointcut(
			candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
	if (expressionPointcut == null) {
		return null;
	}

	// 创建advisor(包含了创建Advise的逻辑) InstantiationModelAwarePointcutAdvisorImpl实例
	return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
			this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

如上，getAdvisor方法包含两个主要步骤，一个是获取AspectJ表达式切点，另一个是创建Advisor实现类。在第二个步骤中，包含一个隐藏步骤——创建Advice。下面将按顺序依次分析这两个步骤，先看获取AspectJ表达式切点的过程，如下：

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
   // 获取方法上的AspectJ相关注解，包括@Before，@After等，包装成AspectJAnnotation
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }
   // 创建一个AspectJExpressionPointcut对象
   AspectJExpressionPointcut ajexp =
         new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
   ajexp.setExpression(aspectJAnnotation.getPointcutExpression()); // 设置切点表达式
   if (this.beanFactory != null) {
      ajexp.setBeanFactory(this.beanFactory);
   }
   return ajexp;
}

// ASPECTJ_ANNOTATION_CLASSES中的注解是大家熟悉的
private static final Class<?>[] ASPECTJ_ANNOTATION_CLASSES = new Class<?>[] {
			Pointcut.class, Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class};

protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
	for (Class<?> clazz : ASPECTJ_ANNOTATION_CLASSES) {
		// 查找注解
		AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) clazz);
		if (foundAnnotation != null) {
			return foundAnnotation;
		}
	}
	return null;
}
```

获取切点的过程并不复杂，不过需要注意的是，目前获取到的切点可能还只是个半成品，需要再次处理一下才行。比如下面的代码：

```java
@Aspect
public class AnnotationAopCode {

    @Pointcut("execution(* xyz.coolblog.aop.*.world*(..))")
    public void pointcut() {}

    @Before("pointcut()")
    public void before() {
        System.out.println("AnnotationAopCode`s before");
    }
}
```

@Before 注解中的表达式是`pointcut()`，也就是说 ajexp 设置的表达式只是一个中间值，不是最终值，即`execution(* xyz.coolblog.aop.*.world*(..))`。所以后续还需要将 ajexp 中的表达式进行转换，关于这个转换的过程，不再分析。

说完切点的获取过程，下面再来看看Advisor实现类的创建过程。如下：

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
      Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   this.declaredPointcut = declaredPointcut; // 切点
   this.declaringClass = aspectJAdviceMethod.getDeclaringClass(); // 声明类
   this.methodName = aspectJAdviceMethod.getName(); // advice方法
   this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
   this.aspectJAdviceMethod = aspectJAdviceMethod; // advice方法
   this.aspectJAdvisorFactory = aspectJAdvisorFactory;
   this.aspectInstanceFactory = aspectInstanceFactory;
   this.declarationOrder = declarationOrder;
   this.aspectName = aspectName; // 切面名

   // aspectInstanceFactory.getAspectMetadata(): BeanFactoryAspectInstanceFactory
   // 切面是否懒初始化
   if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) { 
      // Static part of the pointcut is a lazy type.
      Pointcut preInstantiationPointcut = Pointcuts.union( // 切点并集
            aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

      // Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
      // If it's not a dynamic pointcut, it may be optimized out
      // by the Spring AOP infrastructure after the first evaluation.
      this.pointcut = new PerTargetInstantiationModelPointcut(
            this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
      this.lazy = true;
   }
   else {
      // A singleton aspect. 单例的切面
      this.pointcut = this.declaredPointcut;
      this.lazy = false;
      // 按照Advice方法上的注解解析Advice
      this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
   }
}
```

上面是InstantiationModelAwarePointcutAdvisorImpl的构造方法，不过无需太关心这个方法中的一些初始化逻辑。把目光移到构造方法的最后一行代码，即instantiateAdvice(this.declaredPointcut)，这个方法用于创建通知 Advice。通知器Advisor是通知Advice的持有者，所以在Advisor实现类的构造方法中创建通知也是合适的。下面就来看看构建通知的过程是怎样的，如下：

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
   Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
         this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
   return (advice != null ? advice : EMPTY_ADVICE);
}

public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
		MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

	Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass(); // 切面类
	validate(candidateAspectClass); // 验证切面类

	// 获取Advice注解
	AspectJAnnotation<?> aspectJAnnotation =
			AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
	if (aspectJAnnotation == null) {
		return null;
	}

	// If we get here, we know we have an AspectJ method.
	// Check that it's an AspectJ-annotated class
	if (!isAspect(candidateAspectClass)) {
		throw new AopConfigException("Advice must be declared inside an aspect type: " +
				"Offending method '" + candidateAdviceMethod + "' in class [" +
				candidateAspectClass.getName() + "]");
	}

	if (logger.isDebugEnabled()) {
		logger.debug("Found AspectJ method: " + candidateAdviceMethod); // debug
	}

	AbstractAspectJAdvice springAdvice;

	// 按照注解类型生成相应的Advice实现类
	switch (aspectJAnnotation.getAnnotationType()) {
		case AtPointcut: // @Pointcut -> null
			if (logger.isDebugEnabled()) {
				logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
			}
			return null;
		case AtAround:  // @Around -> AspectJAroundAdvice
			springAdvice = new AspectJAroundAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
		case AtBefore: // @Before -> AspectJMethodBeforeAdvice
			springAdvice = new AspectJMethodBeforeAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
		case AtAfter: // @After -> AspectJAfterAdvice
			springAdvice = new AspectJAfterAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
		case AtAfterReturning: // @AfterReturning -> AspectJAfterReturningAdvice
			springAdvice = new AspectJAfterReturningAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
			if (StringUtils.hasText(afterReturningAnnotation.returning())) {
        // 设置返回值参数名
				springAdvice.setReturningName(afterReturningAnnotation.returning()); 
			}
			break;
		case AtAfterThrowing: // @AfterThrowing -> AspectJAfterThrowingAdvice
			springAdvice = new AspectJAfterThrowingAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
			if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
				springAdvice.setThrowingName(afterThrowingAnnotation.throwing()); // 设置异常名
			}
			break;
		default:
			throw new UnsupportedOperationException(
					"Unsupported advice type on method: " + candidateAdviceMethod);
	}

	// Now to configure the advice... 配置advice
	springAdvice.setAspectName(aspectName);
	springAdvice.setDeclarationOrder(declarationOrder);
	// 获取方法的参数列表名称，比如方法 int sum(int numX, int numY),
	// getParameterNames(sum) 得到 argNames = [numX, numY]
	String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
	if (argNames != null) {
		springAdvice.setArgumentNamesFromStringArray(argNames); // 设置参数名
	}
	springAdvice.calculateArgumentBindings();

	return springAdvice;
}
```

上面的代码逻辑不是很复杂，主要的逻辑就是根据注解类型生成与之对应的通知对象。下面来总结一下获取通知器（getAdvisors）整个过程的逻辑，如下：

1. 从目标bean中获取不包含Pointcut注解的方法列表；
2. 遍历上一步获取的方法列表，并调用getAdvisor获取当前方法对应的Advisor；
   1. 创建AspectJExpressionPointcut对象，并从方法的注解中获取表达式，最后设置到切点对象中；
   2. 创建Advisor实现类对象InstantiationModelAwarePointcutAdvisorImpl；
      1. 调用instantiateAdvice方法构建通知；
         1. 调用getAdvice方法，并根据注解类型创建相应的通知。

如上所示，上面的步骤做了一定的简化。总的来说，获取通知器的过程还是比较复杂的。下面来分析一下 AspectJMethodBeforeAdvice，也就是@Before注解对应的通知实现类。

####### (3) AspectJMethodBeforeAdvice分析

```java
public class AspectJMethodBeforeAdvice extends AbstractAspectJAdvice implements MethodBeforeAdvice, Serializable {

   public AspectJMethodBeforeAdvice(
         Method aspectJBeforeAdviceMethod, AspectJExpressionPointcut pointcut, AspectInstanceFactory aif) {

      super(aspectJBeforeAdviceMethod, pointcut, aif);
   }


   @Override
   public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
      // 调用通知方法
      invokeAdviceMethod(getJoinPointMatch(), null, null);
   }

   @Override
   public boolean isBeforeAdvice() {
      return true;
   }

   @Override
   public boolean isAfterAdvice() {
      return false;
   }

}

protected Object invokeAdviceMethod(
		@Nullable JoinPointMatch jpMatch, @Nullable Object returnValue, @Nullable Throwable ex)
		throws Throwable {
	// 调用通知方法，并向其传递参数
	return invokeAdviceMethodWithGivenArgs(argBinding(getJoinPoint(), jpMatch, returnValue, ex));
}

protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
	Object[] actualArgs = args;
	if (this.aspectJAdviceMethod.getParameterCount() == 0) {
		actualArgs = null;
	}
	try {
		ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
		// TODO AopUtils.invokeJoinpointUsingReflection
		// 通过反射调用通知方法
		return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
	}
	catch (IllegalArgumentException ex) {
		throw new AopInvocationException("Mismatch on arguments to advice method [" +
				this.aspectJAdviceMethod + "]; pointcut expression [" +
				this.pointcut.getPointcutExpression() + "]", ex);
	}
	catch (InvocationTargetException ex) {
		throw ex.getTargetException();
	}
}
```

如上，AspectJMethodBeforeAdvice 的源码比较简单，这里仅关注before方法。这个方法调用了父类中的 invokeAdviceMethod，然后 invokeAdviceMethod 在调用 invokeAdviceMethodWithGivenArgs，最后在 invokeAdviceMethodWithGivenArgs 通过反射执行通知方法。

###### 2.2 筛选合适的通知器

查找出所有的通知器，整个流程还没算完，接下来还要对这些通知器进行筛选。适合应用在当前目标bean上的通知器留下，不适合的就过滤掉。那下面我就来分析一下通知器筛选的过程，如下：

```java
protected List<Advisor> findAdvisorsThatCanApply(
      List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

   ProxyCreationContext.setCurrentProxiedBeanName(beanName);
   try {
      // 筛选合适的通知器
      return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
   }
   finally {
      ProxyCreationContext.setCurrentProxiedBeanName(null);
   }
}

public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
	if (candidateAdvisors.isEmpty()) {
		return candidateAdvisors;
	}
	List<Advisor> eligibleAdvisors = new ArrayList<>();
	for (Advisor candidate : candidateAdvisors) {
		// 筛选IntroductionAdvisor类型的通知器
		if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
			eligibleAdvisors.add(candidate);
		}
	}
	boolean hasIntroductions = !eligibleAdvisors.isEmpty();
	for (Advisor candidate : candidateAdvisors) {
		if (candidate instanceof IntroductionAdvisor) {
			// already processed
			continue;
		}
		// 筛选普通类型的通知器
		if (canApply(candidate, clazz, hasIntroductions)) {
			eligibleAdvisors.add(candidate);
		}
	}
	return eligibleAdvisors;
}

public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
	if (advisor instanceof IntroductionAdvisor) {
		// 从通知器中获取类型过滤器ClassFilter，并调用matches方法进行匹配。
		// 以ClassFilter接口的实现类AspectJExpressionPointcut为例，该类的
		// 匹配工作由AspectJ表达式相关的机制负责
		return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
	}
	else if (advisor instanceof PointcutAdvisor) {
		// 对于普通类型的通知器，这里继续调用重载方法进行筛选
		PointcutAdvisor pca = (PointcutAdvisor) advisor;
		return canApply(pca.getPointcut(), targetClass, hasIntroductions);
	}
	else {
		// It doesn't have a pointcut so we assume it applies.
		return true;
	}
}

public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
	Assert.notNull(pc, "Pointcut must not be null");
	// 1.使用ClassFilter匹配class
	if (!pc.getClassFilter().matches(targetClass)) {
		return false;
	}

	MethodMatcher methodMatcher = pc.getMethodMatcher();
	if (methodMatcher == MethodMatcher.TRUE) {
		// No need to iterate the methods if we're matching any method anyway...
		return true;
	}

	IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
	if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
		introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
	}

	// 查找当前类及其父类（以及父类的父类等等）所实现的接口，由于接口中的方法是public，
	// 所以当前类可以继承其父类，和父类的父类中所有的接口方法
	Set<Class<?>> classes = new LinkedHashSet<>();
	if (!Proxy.isProxyClass(targetClass)) {
		classes.add(ClassUtils.getUserClass(targetClass));
	}
	classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

	for (Class<?> clazz : classes) {
    // 获取clazz的方法列表，包括从父类中继承的方法
		Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
		for (Method method : methods) {
			if (introductionAwareMethodMatcher != null ?
					introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
          // 2.使用methodMatcher匹配方法，匹配成功即可立即返回
					methodMatcher.matches(method, targetClass)) { 
				return true;
			}
		}
	}

	return false;
}
```

以上是筛选通知器的过程，筛选的工作主要由ClassFilter和MethodMatcher完成。关于ClassFilter和MethodMatcher，在 AOP 中，切点Pointcut是用来匹配连接点的，以AspectJExpressionPointcut类型的切点为例。该类型切点实现了ClassFilter和MethodMatcher接口，匹配的工作则是由AspectJ表达式相关的机制负责。除了使用AspectJ表达式进行匹配，Spring还提供了基于正则表达式的切点类，以及更简单的根据方法名进行匹配的切点类。

在完成通知器的查找和筛选过程后，还需要进行最后一步处理——对通知器列表进行拓展。

##### 3. 拓展筛选出的通知器列表

拓展方法extendAdvisors做的事情并不多，逻辑也比较简单如下：

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

由上面的源码可以看出，extendAdvisors方法调用了AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary方法。至于 makeAdvisorChainAspectJCapableIfNecessary 这个方法，该方法主要的目的是向**通知器列表首部**添加 DefaultPointcutAdvisor类型的通知器，也就是**ExposeInvocationInterceptor.ADVISOR**。至于添加此种类型通知器的原因，会在[后面文章]()里说明，这里不便展开。

### 三、总结

至此，AOP代理创建的第一步——筛选合适的通知器分析结束。

### 参考文章

- [Spring-AOP-源码分析-筛选合适的通知器](http://www.tianxiaobo.com/2018/06/20/Spring-AOP-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E7%AD%9B%E9%80%89%E5%90%88%E9%80%82%E7%9A%84%E9%80%9A%E7%9F%A5%E5%99%A8/)
- [Spring AOP 源码解析](https://javadoop.com/post/spring-aop-source)

