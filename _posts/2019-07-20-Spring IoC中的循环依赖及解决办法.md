---
layout:     post
title:      Spring IoC中的循环依赖问题及解决办法
subtitle:   
date:       2019-07-20
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring IoC中的循环依赖问题及解决办法

本文分析 Spring 是如何解决循环依赖问题的。在本篇文章中，首先介绍一下什么是循环依赖。然后，进入源码分析阶段。为了更好的说明 Spring 解决循环依赖的办法，文章将会从获取 bean 的方法`getBean(String)`开始，把整个调用过程梳理一遍。梳理完后，再来详细分析源码。

### 一、背景知识

##### 1.什么是循环依赖

所谓的循环依赖是指，A 依赖 B，B 又依赖 A，它们之间形成了循环依赖。或者是 A 依赖 B，B 依赖 C，C 又依赖 A。它们之间的依赖关系如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%961.png)

这里以两个类直接相互依赖为例，他们的实现代码可能如下：

```java
public class BeanB {
    private BeanA beanA;
    // 省略 getter/setter
}

public class BeanA {
    private BeanB beanB;
}
```

配置信息如下：

```java
<bean id="beanA" class="xyz.coolblog.BeanA">
    <property name="beanB" ref="beanB"/>
</bean>
<bean id="beanB" class="xyz.coolblog.BeanB">
    <property name="beanA" ref="beanA"/>
</bean>
```

IoC 容器在读到上面的配置时，会按照顺序，先去实例化 beanA。然后发现 beanA 依赖于 beanB，接在又去实例化 beanB。实例化 beanB 时，发现 beanB 又依赖于 beanA。如果容器不处理循环依赖的话，容器会无限执行上面的流程，直到内存溢出，程序崩溃。当然，Spring 是不会让这种情况发生的。在容器再次发现 beanB 依赖于 beanA 时，容器会获取 beanA 对象的一个早期的引用(**early reference**)，并把这个早期引用注入到 beanB 中，让 beanB 先完成实例化。beanB 完成实例化，beanA 就可以获取到 beanB 的引用，beanA 随之完成实例化。这里大家可能不知道“**早期引用**”是什么意思，请先别着急，它会在下面的内容中进行说明。

##### 2.一些缓存的介绍

在进行源码分析前，先来看一组缓存的定义。如下：

```java
// DefaultSingletonBeanRegistry类
/** Cache of singleton objects: bean name to bean instance. 完全初始化好的单例，可以直接拿来用 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name to ObjectFactory. */
/** 存放 单例bean 对象工厂，生成early singleton object, 用于解决循环依赖 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name to bean instance. */
/** 存放原始的单例 bean 对象（尚未填充属性），用于解决循环依赖 */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

根据缓存变量上面的注释，大家应该能大致了解它们的用途。这里简单说明一下：

| 缓存                  | 用途                                                         |
| :-------------------- | :----------------------------------------------------------- |
| singletonObjects      | 用于存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用 |
| earlySingletonObjects | 存放原始的 bean 对象（尚未填充属性），用于解决循环依赖       |
| singletonFactories    | 存放 bean 对象工厂，生成early singleton object，用于解决循环依赖 |

上面提到了”**早期引用**“，所谓的”早期引用“是指向**原始对象**的引用。所谓的**原始对象是指刚创建好的对象，但还未填充属性**。举例说明如下，这里先定义一个对象 Room：

```java
/** Room 包含了一些电器 */
public class Room {
    private String television;
    private String airConditioner;
    private String refrigerator;
    private String washer;
    // 省略 getter/setter
}
```

配置如下：

```java
<bean id="room" class="xyz.coolblog.demo.Room">
    <property name="television" value="Xiaomi"/>
    <property name="airConditioner" value="Gree"/>
    <property name="refrigerator" value="Haier"/>
    <property name="washer" value="Siemens"/>
</bean>
```

先看一下完全实例化好后的 bean 长什么样的。如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%962.png)

从调试信息中可以看得出，Room 的每个成员变量都被赋上值了。然后再来看一下“原始的 bean 对象”长的是什么样的，如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%963.png)

结果比较明显了，所有字段都是 null。这里的 bean 和上面的 bean 指向的是同一个对象`Room@1567`，但现在这个对象所有字段都是 null，这种对象就是原始的对象。形象点说，上面的 bean 对象是一个装修好的房子，可以拎包入住了。而这里的 bean 对象还是个毛坯房，还要装修一下（填充属性）才行。

##### 3.回顾获取/创建 bean 的过程

本节来了解从 Spring IoC 容器中获取/创建 bean 实例的流程(简化版)，这对后续的源码分析会有比较大的帮助。先看图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%964.png)

先来简单介绍一下这张图，这张图是一个简化后的流程图。开始流程图中只有一条执行路径，在条件 sharedInstance != null 这里出现了岔路，形成了蓝色和红色两条路径。在上图中，读取/添加缓存的方法用☆标注了出来。至于虚线的箭头，和虚线框里的路径，这个下面会说到。

按照上面的图，可知整个流程的执行顺序是：流程从 getBean 方法开始，getBean 是个空壳方法，所有逻辑都在 doGetBean 方法中。doGetBean 首先会调用 getSingleton(beanName) 方法获取 sharedInstance，**sharedInstance 可能是完全实例化好的 bean，也可能是一个原始的 bean，当然也有可能是 null**。如果不为 null，则走蓝色的那条路径。再经 getObjectForBeanInstance 这一步处理后，蓝色的这条执行路径就结束了。

再来看一下红色的那条执行路径，也就是 sharedInstance = null 的情况。在第一次获取某个 bean 的时候，缓存中是没有记录的，所以这个时候要走创建逻辑。上图中的 getSingleton(beanName,
()  -> {…}) 方法会创建一个 bean 实例，上图虚线路径指的是 getSingleton 方法内部调用的两个方法，其逻辑如下：

```java
// DefaultSingletonBeanRegistry类
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    // 省略部分代码
    singletonObject = singletonFactory.getObject();
    // ...
    addSingleton(beanName, singletonObject);
}
```

如上所示，getSingleton 会在内部先调用 getObject 方法创建 singletonObject，然后再调用 addSingleton 将 singletonObject 放入缓存中。getObject 在内部调用了 createBean 方法，createBean 方法基本上也属于空壳方法，更多的逻辑是写在 doCreateBean 方法中的。doCreateBean 方法中的逻辑很多，其首先调用了 createBeanInstance 方法创建了一个原始的 bean 对象，随后调用 addSingletonFactory 方法向缓存中添加单例 bean 对象工厂，从该对象工厂可以获取原始对象的引用，也就是所谓的“**早期引用**”。再之后，继续调用 populateBean 方法向原始 bean 对象中填充属性，并解析依赖。getObject 执行完成后，会返回**完全实例化好的 bean**。紧接着再调用 addSingleton 把完全实例化好的 bean 对象放入缓存中。到这里，红色执行路径差不多也就要结束了。

这里没有把 getObject、addSingleton 方法和 getSingleton(String, ObjectFactory) 并列画在红色的路径里，目的是想简化一下方法的调用栈（都画进来有点复杂）。可以进一步简化上面的调用流程，比如下面：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%965.png?x-oss-process=style/markdown-pic)

这个流程看起来简单多了，命中缓存走蓝色路径，未命中走红色的创建路径。

### 二、源码分析

经过前面的铺垫，现在进入源码分析部分。下面按照方法的调用顺序，依次来看一下循环依赖相关的代码。如下：

```java
// AbstractBeanFactory类
protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            throws BeansException {

    // ...... 
    
    // 从缓存中获取 bean 实例
    Object sharedInstance = getSingleton(beanName);

    // ......
}

// DefaultSingletonBeanRegistry类
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

// DefaultSingletonBeanRegistry类
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从 singletonObjects 获取实例，singletonObjects 中的实例都是准备好的 bean 实例，可以直接使用
    Object singletonObject = this.singletonObjects.get(beanName);
    // 判断 beanName 对应的 bean 是否正在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 中获取提前曝光的 bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 获取相应的 bean 工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 提前曝光 bean 实例（raw bean），用于解决循环依赖
                    singletonObject = singletonFactory.getObject();
                    
                    // 将 singletonObject 放入缓存中，并将 singletonFactory 从缓存中移除
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

上面的源码中，doGetBean 所调用的方法 getSingleton(String) 是一个空壳方法，其主要逻辑在 getSingleton(String, boolean) 中。该方法逻辑比较简单，首先从 singletonObjects 缓存中获取 bean 实例。若未命中，再去 earlySingletonObjects 缓存中获取原始 bean 实例。如果仍未命中，则从 singletonFactory 缓存中获取 ObjectFactory 对象，然后再调用 getObject 方法获取原始 bean 实例的引用，也就是早期引用。获取成功后，将该实例放入 earlySingletonObjects 缓存中，并将 ObjectFactory 对象从 singletonFactories 移除。看完这个方法，再来看看 getSingleton(String, ObjectFactory) 方法，这个方法也是在 doGetBean 中被调用的。这次把 doGetBean 的代码多贴一点出来，如下：

```java
// AbstractBeanFactory类
protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {

    // ...... 
    Object bean;

    // 从缓存中获取 bean 实例
    Object sharedInstance = getSingleton(beanName);

    // 这里先忽略 args == null 这个条件
    if (sharedInstance != null && args == null) {
        // 进行后续的处理
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // ......

        // mbd.isSingleton() 用于判断 bean 是否是单例
        if (mbd.isSingleton()) {
            // 创建单例 bean 实例
            sharedInstance = getSingleton(beanName, () -> {
              try {
                // 创建单例 bean 实例，createBean 返回的 bean 是完全实例化好的
                return createBean(beanName, mbd, args);
              } catch (BeansException ex) {
                destroySingleton(beanName);
                throw ex;
              }
            });
            // 进行后续的处理
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }

        // ......
    }

    // ......

    // 返回 bean
    return (T) bean;
}
```

这里的代码逻辑和在第一部分`回顾获取/创建 bean 的过程` 一节的最后贴的主流程图已经很接近了，对照那张图和代码中的注释，大家应该可以理解 doGetBean 方法了。继续往下看getSingleton(String beanName, ObjectFactory<?> singletonFactory)：

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
        // 调用 getObject 方法创建 bean 实例
				// 此时获得的singletonObject已完全初始化
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
				// 如果是新的单例，则添加到缓存(此处singletonObject已完全初始化，属性已填充)
        // 添加 bean 到 singletonObjects 缓存中，并从其他集合中将 bean 相关记录移除
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
// DefaultSingletonBeanRegistry类
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // 将 <beanName, singletonObject> 映射存入 singletonObjects 中
        this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

        // 从其他缓存中移除 beanName 相关映射
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```

上面的代码中包含两步操作，第一步操作是调用 getObject 创建 bean 实例，第二步是调用 addSingleton 方法将创建好的 bean 放入缓存中。代码逻辑并不复杂，相信大家都能看懂。那么接下来继续往下看，这次分析的是 doCreateBean 中的一些逻辑。如下：

```java
// AbstractAutowireCapableBeanFactory类
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    BeanWrapper instanceWrapper = null;

    // ......

    // ☆ 创建 bean 对象，并将 bean 对象包裹在 BeanWrapper 对象中
    instanceWrapper = createBeanInstance(beanName, mbd, args);
    
    // 从 BeanWrapper 对象中获取 bean 对象，这里的 bean 指向的是一个原始的对象
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);

    // earlySingletonExposure 用于表示是否”提前暴露“原始对象的引用，用于解决循环依赖。
    // 对于单例 bean，该变量一般为 true。
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // ☆ 添加 bean 对象工厂到 singletonFactories 缓存中
        addSingletonFactory(beanName, () -> {
          // 获取原始对象的早期引用，在 getEarlyBeanReference 方法中，会执行 AOP 
          // 相关逻辑。若 bean 未被 AOP 拦截，getEarlyBeanReference 原样返回 
          // bean，所以大家可以把 
          //      return getEarlyBeanReference(beanName, mbd, bean) 
          // 等价于：
          //      return bean;
          return getEarlyBeanReference(beanName, mbd, bean);
        });
    }

    Object exposedObject = bean;

    // ......
    
    // ☆ 填充属性，解析依赖
    populateBean(beanName, mbd, instanceWrapper);

    // ......

    // 返回 bean 实例
    return exposedObject;
}

// DefaultSingletonBeanRegistry类
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            // 将 singletonFactory 添加到 singletonFactories 缓存中
            this.singletonFactories.put(beanName, singletonFactory);

            // 从其他缓存中移除相关记录，即使没有
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```

上面的代码简化了不少，不过看起来仍有点复杂。好在上面代码的主线逻辑比较简单，由三个方法组成。如下：

```java
1. 创建原始 bean 实例 → createBeanInstance(beanName, mbd, args)
2. 添加 原始bean 对象工厂 到 singletonFactories 缓存中 
        → addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))
3. 填充属性，解析依赖 → populateBean(beanName, mbd, instanceWrapper)
```

到这里，本节涉及到的源码就分析完了。可是看完源码后，似乎仍然不知道这些源码是如何解决循环依赖问题的。难道本篇文章就到这里了吗？答案是否。下面来解答这个问题，这里还是以 BeanA 和 BeanB 两个类相互依赖为例。在上面的方法调用中，有几个关键的地方，下面一一列举出来：

##### 1. 创建原始 bean 对象

```java
// AbstractAutowireCapableBeanFactory类
instanceWrapper = createBeanInstance(beanName, mbd, args);
final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
```

假设 beanA 先被创建，创建后的原始对象为 `BeanA@1234`，上面代码中的 bean 变量指向就是这个对象。

##### 2. 暴露早期引用

```java
// DefaultSingletonBeanRegistry类
addSingletonFactory(beanName, () -> {
  return getEarlyBeanReference(beanName, mbd, bean);
});
```

beanA 指向的原始对象创建好后，就开始把指向原始对象的引用通过 ObjectFactory 暴露出去。getEarlyBeanReference 方法的第三个参数 bean 指向的正是 createBeanInstance 方法创建出来的原始 bean 对象 BeanA@1234。

##### 3. 解析依赖

```java
// AbstractAutowireCapableBeanFactory类
populateBean(beanName, mbd, instanceWrapper);
```

populateBean 用于向 beanA 这个原始对象中填充属性，当它检测到 beanA 依赖于 beanB 时，会首先去实例化 beanB。beanB 在此方法处也会解析自己的依赖，当它检测到 beanA 这个依赖，于是调用 BeanFactry.getBean(“beanA”) 这个方法，从容器中获取 beanA。

##### 4. 获取早期引用

```java
// DefaultSingletonBeanRegistry类
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // ☆ 从缓存中获取早期引用
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // ☆ 从 SingletonFactory 中获取早期引用
                    singletonObject = singletonFactory.getObject();
                    
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

接着上面的步骤讲，populateBean 调用 BeanFactry.getBean(“beanA”) 以获取 beanB 的依赖。getBean(“beanA”) 会先调用 getSingleton(“beanA”)，尝试从缓存中获取 beanA。此时由于 beanA 还没完全实例化好，于是 this.singletonObjects.get(“beanA”) 返回 null。接着 this.earlySingletonObjects.get(“beanA”) 也返回空，因为 beanA 早期引用还没放入到这个缓存中。最后调用 singletonFactory.getObject() 返回 singletonObject，此时 singletonObject != null。singletonObject 指向 BeanA@1234，也就是 createBeanInstance 创建的原始对象。此时 beanB 获取到了这个原始对象的引用，beanB 就能顺利完成实例化。beanB 完成实例化后，beanA 就能获取到 beanB 所指向的实例，beanA 随之也完成了实例化工作。由于 beanB.beanA 和 beanA 指向的是同一个对象 BeanA@1234，所以 beanB 中的 beanA 此时也处于可用状态了。

以上的过程对应下面的流程图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%966.png)

至此，循环依赖的处理流程分析结束。

### 参考文章

- [Spring IOC 容器源码分析 - 循环依赖的解决办法](http://www.tianxiaobo.com/2018/06/08/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96%E7%9A%84%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/)
- [Spring源码-IOC容器(六)-bean的循环依赖](https://my.oschina.net/u/2377110/blog/979226)

