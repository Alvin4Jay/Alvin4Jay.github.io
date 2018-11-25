---
layout:     post
title:      Dubbo中的IoC与AOP实现解析
subtitle:   Dubbo SPI扩展点属性注入和Wrapper包装
date:       2018-11-25
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo中的IoC与AOP实现解析

### IoC解析

`Dubbo IoC`的过程实现在`com.alibaba.dubbo.common.extension.ExtensionLoader#injectExtension`方法中，该方法使用在下面的几处地方：

```java
1. createAdaptiveExtension() --> injectExtension((T) getAdaptiveExtensionClass().newInstance()) // 对已实例化的自适应扩展类(适配类)实例进行属性注入
2. createExtension(String name) 
    // 对已实例化的扩展点实现类实例进行属性注入
    --> injectExtension(instance) 
    // 对已实例化的包装类进行属性注入
    --> injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance)) 
```

#### `injectExtension(T instance)`方法解析

```java
// 对instance实例进行属性的注入
private T injectExtension(T instance) {
    try {
        // type = ExtensionFactory.class的情况下，objectFactory = null
        if (objectFactory != null) {
            // Class.getMethods()获取Class代表的类或接口的public方法，包括从父类或父接口继承的public方法
            for (Method method : instance.getClass().getMethods()) {
                // public setter 方法，只有一个参数
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0]; // 参数的类型
                    try {
                        // 参数名
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        // 使用objectFactory获取对应参数名和参数类型的参数实例
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                           	// setter 方法，反射注入
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                     + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

上述的`injectExtension()`方法，完成了对`instance`对象的属性的注入，通过`setter`方法进行反射调用实现。主要逻辑如下：

- 得到`setter`方法，获取`setter`方法参数的类型和名称
- 使用`objectFactory`获取对应参数名和参数类型的参数实例
- `setter` 方法，反射注入

#### `objectFactory.getExtension(pt, property)`

在`injectExtension(T instance)  IoC`注入的时候，`Object object = objectFactory.getExtension(pt, property);`这行代码完成了获取参数实例的工作。其中，`objectFactory`为`AdaptiveExtensionFactory`实例，其属性`factories = [SpringExtensionFactory intance, SpiExtensionFactory instance]`。

**AdaptiveExtensionFactory**类

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
        // 遍历ExtensionFactory实例，获取对应type和name的实例
        // 先SpiExtensionFactory / 后SpringExtensionFactory	
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

**SpiExtensionFactory**类

```java
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // Class type对应的必须是接口，并且需要注解了@SPI注解
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            // 判断该扩展点接口是否有扩展点实现类
            if (loader.getSupportedExtensions().size() > 0) {
                // 获取type对应接口的自适应类(适配类)实例。如果有@Adaptive注解的类,则返回该类的实例,否则返回一个动态生成类的实例(如Protocol$Adpative的实例)
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```

可知，扩展点实例`instance`可以被自动注入其他的扩展点自适应类实例。

**SpringExtensionFactory**类

```java
public class SpringExtensionFactory implements ExtensionFactory {

    /** 全局的Spring应用上下文集合 */
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
	// ServiceBean初始化的时候调用(在setApplicationContext()方法)，更新contexts集合
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
        // 遍历ApplicationContext
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) { // 如果context包含名称为name的bean
                // 获取名称为name的bean。如果是懒加载或原型的bean,此时会实例化名称为name的bean
                Object bean = context.getBean(name);
                // 判断bean是否是type对应的类的实例或者是其子类的实例，等于instanceof
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }

}
```

### AOP解析

以下面的代码执行过程为例，分析`Dubbo AOP`的执行流程。

```java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);// 1
Protocol dubboProtocol = loader.getExtension("dubbo");// 2
```

#### `ExtensionLoader`实例化

在第1行代码执行完之后，`ExtensionLoader`实例有如下的实例属性:

```java
- type: interface com.alibaba.dubbo.rpc.Protocol
- ExtensionFactory objectFactory = AdaptiveExtensionFactory(自适应类(适配类))
	- factories = [SpringExtensionFactory instance, SpiExtensionFactory instance]
```

#### `getExtension(String name)`方法解析

这个方法是扩展点具体实现类实例的获取函数，调用路径如下:

```java
getExtension(String name) 
	createExtension(String name)
		getExtensionClasses().get(name) // 根据name获取扩展点实现类Class对象
			loadExtensionClasses()
				loadFile(Map<String, Class<?>> extensionClasses, String dir)
		injectExtension(T instance) // IoC注入
		包装类包装 // AOP
```

```java
// 根据name获取扩展点实现的实例
// 先从cachedInstances缓存中查看是否已创建了实例，如果有，直接返回；无则创建后，放入缓存，然后返回。
/*
getExtension() --> createExtension() --> injectExtension()
*/
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    // 查看缓存
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 无实例缓存，则创建
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

// 创建实例
private T createExtension(String name) {
    // 从cachedClasses缓存中获取实现类Class对象
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        // 这个时候还没有Class，说明真的没有，直接抛出异常
        throw findException(name);
    }
    try {
        // 查看EXTENSION_INSTANCES缓存是否已经创建了该实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 无，则实例化
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance); // IoC注入
        Set<Class<?>> wrapperClasses = cachedWrapperClasses; // Wrapper包装 
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                                        type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

在`createExtension(String name)`方法调用过程中，会进行`Wrapper`包装，同时根据`META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol`中的配置可知，`com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper/com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper`这两个类是实现了`Protocol`接口的**包装类**，分别具有一个以`Protocol`作为参数的构造器且类上无`@Adaptive`注解，放置在`ExtensionLoader`的私有属性`cachedWrapperClasses`中。

这时的`ExtensionLoader`实例有如下的实例属性:

```java
- type: interface com.alibaba.dubbo.rpc.Protocol
- ExtensionFactory objectFactory = AdaptiveExtensionFactory(自适应类(适配类))
	- factories = [SpringExtensionFactory instance, SpiExtensionFactory instance]
- cachedWrapperClasses = [class ProtocolListenerWrapper, class ProtocolFilterWrapper]
```

在`createExtension(String name)`方法中的`AOP`包装部分代码如下所示:

```java
// cachedWrapperClasses = [class ProtocolListenerWrapper, class ProtocolFilterWrapper]
Set<Class<?>> wrapperClasses = cachedWrapperClasses; // Wrapper包装 
if (wrapperClasses != null && wrapperClasses.size() > 0) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
```

**处理逻辑**如下：

- 获取`ProtocolListenerWrapper`的单参构造器，以`DubboProtocol`实例为构造器参数创建`ProtocolListenerWrapper`实例，并完成对`ProtocolListenerWrapper`实例的属性注入。此时的`instance=ProtocolListenerWrapper`实例，不再是之前的`DubboProtocol`实例。
- 使用`ProtocolFilterWrapper`以同样的方式对`ProtocolListenerWrapper`实例进行包装。

**Wrapper包装后的关系**如下:

```java
instance = ProtocolFilterWrapper实例 {
    protocol = ProtocolListenerWrapper实例 {
        protocol = DubboProtocol实例
    }  
}    
```

`ProtocolListenerWrapper`和`ProtocolFilterWrapper`的源码如下，源码在**服务暴露和引用**的时候再分析：

```java
// ProtocolListenerWrapper
public class ProtocolListenerWrapper implements Protocol {
    static {
        try {
            Class serverClass = Protocol.class.getClassLoader().loadClass("com.alibaba.dubbo.qos.server.Server");
            Method serverGetInstanceMethod = serverClass.getMethod("getInstance");
            Object serverInstance = serverGetInstanceMethod.invoke(null);
            Method startMethod = serverClass.getMethod("start");
            startMethod.invoke(serverInstance);
        }catch (Throwable throwable){

        }

    }

    private final Protocol protocol;

    public ProtocolListenerWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }

    public int getDefaultPort() {
        return protocol.getDefaultPort();
    }

    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        return new ListenerExporterWrapper<T>(protocol.export(invoker),
                Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                        .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
    }

    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            return protocol.refer(type, url);
        }
        return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
                Collections.unmodifiableList(
                        ExtensionLoader.getExtensionLoader(InvokerListener.class)
                                .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
    }

    public void destroy() {
        protocol.destroy();
    }

}

// ProtocolFilterWrapper
public class ProtocolFilterWrapper implements Protocol {

    // 底层的协议
    private final Protocol protocol;

    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }

    // 构建调用执行体链
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        // 拦截器列表
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class)
                .getActivateExtension(invoker.getUrl(), key, group);
        if (filters.size() > 0) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {
                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        // 进行调用拦截
                        return filter.invoke(next, invocation);
                    }

                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }

    @Override
    public int getDefaultPort() {
        return protocol.getDefaultPort();
    }

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker); // 服务提供者注册并暴露服务
        }
        return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY,
                Constants.PROVIDER)); // 提供者暴露服务的拦截器
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            return protocol.refer(type, url); // 服务消费者注册并引用服务
        }
        return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY,
                Constants.CONSUMER); // 消费者引用服务的拦截器
    }

    @Override
    public void destroy() {
        protocol.destroy();
    }

}
```

最后`createExtension(String name)`方法返回的实例是`ProtocolFilterWrapper`对象，即

```java
Protocol dubboProtocol = loader.getExtension("dubbo");
```

这行代码执行后，返回的`Protocol`实例是`ProtocolFilterWrapper`对象，即包装之后的对象。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/1.png)

以上便是`Dubbo IOC、AOP`的整个过程。