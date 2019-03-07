---
layout:     post
title:      Dubbo 服务暴露之服务远程暴露——创建Exporter与启动Netty服务端
subtitle:   Exporter/Netty Server
date:       2019-03-03
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo 服务暴露之服务远程暴露——创建Exporter与启动Netty服务端

服务远程暴露的总体步骤为

- 将`ref`封装为`Invoker`
- 将`Invoker`转换为`Exporter`
- 启动`Netty`服务端 
- 注册服务到`Zookeeper`
- 订阅与通知机制
- 返回新的`Exporter`实例

服务远程暴露的代码为

```java
// 如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露本地服务)
if (!Constants.SCOPE_LOCAL.equalsIgnoreCase(scope)) {
    if (logger.isInfoEnabled()) {
        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
    }
    if (registryURLs != null && registryURLs.size() > 0) {
        for (URL registryURL : registryURLs) {
            url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
            URL monitorUrl = loadMonitor(registryURL);
            if (monitorUrl != null) {
                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
            }
            if (logger.isInfoEnabled()) {
                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
            }
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
            DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

            Exporter<?> exporter = protocol.export(wrapperInvoker);
            exporters.add(exporter);
        }
    } else {
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

        Exporter<?> exporter = protocol.export(wrapperInvoker);
        exporters.add(exporter);
    }
}
```

首先将服务实现类实例`ref`封装为`Invoker`，之后将`Invoker`转换为`Exporter`，最后将`Exporter`放入缓存`List<Exporter<?>> exporters`中。

## 一、将服务实现类实例ref封装为Invoker

```java
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
```

### 1.为registryURL拼接export=providerUrl参数

一开始的`registryURL`为

```
registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&pid=78143&registry=zookeeper&timestamp=1550368210890
```

`registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString())`这句代码为`registryURL`添加了`export`参数并编码(这里给出没有编码的`url`)：

```java
export=dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=78143&side=provider&timestamp=1550368210931
```

此时的`registryURL`为

```java
registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&export=dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=78143&side=provider&timestamp=1550368210931&group=dubbo_test&pid=78143&registry=zookeeper&timestamp=1550368210890
```

### 2.ProxyFactory$Adaptive.getInvoker(DemoServiceImpl2实例, Class`<DemoService>`, registryURL)

```java
public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws com.alibaba.dubbo.rpc.RpcException {
    if (arg2 == null)
        throw new IllegalArgumentException("url == null");
    com.alibaba.dubbo.common.URL url = arg2;
    String extName = url.getParameter("proxy", "javassist");// 结果是javassist
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
    com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
    return extension.getInvoker(arg0, arg1, arg2); // 
}
```

这里本来是调用`JavassistProxyFactory`的`getInvoker`方法，但是`JavassistProxyFactory`被`StubProxyFactoryWrapper AOP`，因此`ProxyFactory$Adaptive.getInvoker`的调用首先调用的是`StubProxyFactoryWrapper.getInvoker`方法。

### 3.StubProxyFactoryWrapper.getInvoker(DemoServiceImpl2实例, Class`<DemoService>`, registryURL)

```java
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
    // 这个类先执行getInvoker，后执行JavassistProxyFactory.getInvoker() -- 包装
    return proxyFactory.getInvoker(proxy, type, url);
}
```

### 4.JavassistProxyFactory.getInvoker(DemoServiceImpl2实例, Class`<DemoService>`, registryURL)

```java
// 将具体服务实现类实例转为Invoker
// @param proxy 对外提供服务的实现类实例
// @param type 服务接口
// @param url URL
// @param <T> 服务类型
// @return Invoker
@Override
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper类不能正确处理带$的类名
    // 包装类
    final Wrapper wrapper = Wrapper
            .getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

首先是创建`Wrapper`类及其实例：`Wrapper.getWrapper(Class<DemoServiceImpl2>)`。该类实例记录了`DemoServiceImpl2`的属性名称，方法名称等信息。关键代码如下(完整代码见文章[Dubbo 服务暴露之服务本地暴露]())：

```java
public class Wrapper1 extends Wrapper {

    public static String[] pns; // property name array
    public static java.util.Map pts = new HashMap();// <property name, property types>
    public static String[] mns;// all method name array.
    public static String[] dmns;// declared method name array.
    public static Class[] mts0; // 方法参数类型数组

    // 调用方法
    // @param o  服务实现类实例
    // @param n  方法名称
    // @param p  参数类型
    // @param v  参数值
    // @return 方法调用结果
    // @throws java.lang.reflect.InvocationTargetException
    public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException {
        com.alibaba.dubbo.demo.provider.DemoServiceImpl2 w;
        try {
            w = ((com.alibaba.dubbo.demo.provider.DemoServiceImpl2) o);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        try {
            if ("sayHello".equals(n) && p.length == 1) {
                return ($w) w.sayHello((java.lang.String) v[0]);
            }
        } catch (Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchMethodException("Not found method \"" + n + "\" in class com.alibaba.dubbo.demo.provider.DemoServiceImpl2.");
    }
}
```

创建完`DemoServiceImpl2`的`Wrapper`类实例之后（实际上该实例在服务本地暴露的时候已经缓存，这里只是从缓存中拿出来而已），再创建一个`AbstractProxyInvoker`实例。

```java
public abstract class AbstractProxyInvoker<T> implements Invoker<T> {
    // 服务接口的实现类实例
    private final T proxy;
    // 服务接口Class
    private final Class<T> type;
    // 注册中心url 或者 提供者url
    private final URL url;

    public AbstractProxyInvoker(T proxy, Class<T> type, URL url) {
        if (proxy == null) {
            throw new IllegalArgumentException("proxy == null");
        }
        if (type == null) {
            throw new IllegalArgumentException("interface == null");
        }
        if (!type.isInstance(proxy)) {
            throw new IllegalArgumentException(proxy.getClass().getName() + " not implement interface " + type);
        }
        this.proxy = proxy;
        this.type = type;
        this.url = url;
    }

    @Override
    public Class<T> getInterface() {
        return type;
    }

    @Override
    public URL getUrl() {
        return url;
    }

    @Override
    public boolean isAvailable() {
        return true;
    }

    @Override
    public void destroy() {
    }

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(),
                    invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method "
                    + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    // 调用结果
   	// @param proxy 对外提供服务的实现类实例
    // @param methodName 方法名
    // @param parameterTypes 参数类型数组
    // @param arguments 具体参数数组
    // @return 调用结果
    // @throws Throwable
    protected abstract Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable;

    @Override
    public String toString() {
        return getInterface() + " -> " + (getUrl() == null ? " " : getUrl().toString());
    }
}
```

最后创建完成的`AbstractProxyInvoker`实例属性如下：

- proxy：`DemoServiceImpl2`实例
- type：`Class<com.alibaba.dubbo.demo.DemoService>`
- url：`registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&export=dubbo%3A%2F%2F172.16.132.166%3A20881%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D172.16.132.166%26bind.port%3D20881%26cellinvokemode%3Dsharing%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D78143%26side%3Dprovider%26timestamp%3D1550368210931&group=dubbo_test&pid=78143&registry=zookeeper&timestamp=1550368210890`

**这样就将`ref`实现类实例转换成了`Invoker`实例，之后在调用该`invoker.invoke(Invocation invocation)`的时候，会调用`invoker.doInvoke(T proxy, String methodName,Class<?>[] parameterTypes, Object[] arguments)`的时候，就会调用相应的实现类`proxy`的`Wrapper`类的`invokeMethod(proxy, methodName, parameterTypes, arguments)`，该方法又会调用真实的实现类的`methodName`方法**。上面已给出`AbstractProxyInvoker.invoke(Invocation invocation)`方法的代码，即

```java
@Override
public Result invoke(Invocation invocation) throws RpcException {
    try {
        return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
    } catch (InvocationTargetException e) {
        return new RpcResult(e.getTargetException());
    } catch (Throwable e) {
        throw new RpcException("Failed to invoke remote proxy method "
                               + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

这里的`proxy`就是上边赋好值的`proxy：DemoServiceImpl2`实例。而消费者调用的方法信息会封装在`Invocation`对象中，该对象在服务引用时介绍。

## 二、将Invoker转换为Exporter

```java
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
Exporter<?> exporter = protocol.export(wrapperInvoker);
```

首先将`AbstractProxyInvoker`实例包装成`DelegateProviderMetaDataInvoker`实例，再导出该实例。

### 1.Protocol$Adaptive.export(DelegateProviderMetaDataInvoker实例)

```java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
    if (arg0 == null)
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
    if (arg0.getUrl() == null)
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());//registry
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}
```

这里由于AOP的原因，首先调用了`ProtocolListenerWrapper`的`export(Invoker<T> invoker)`，如下：

```java
// invoker---DelegateProviderMetaDataInvoker实例(包装AbstractProxyInvoker实例)
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 注册中心协议
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        // 协议为registry，通过ProtocolFilterWrapper导出服务  
        return protocol.export(invoker);
    }
    // 忽略
    return new ListenerExporterWrapper<T>(protocol.export(invoker), Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                    .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
}
```

由于协议是“registry”，所以不做任何处理，继续调用`ProtocolFilterWrapper`的`export(Invoker<T> invoker)`，如下:

```java
// invoker---DelegateProviderMetaDataInvoker实例(包装AbstractProxyInvoker实例)
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        // 协议为registry，通过RegistryProtocol注册并暴露服务
        return protocol.export(invoker);
    }
    // 忽略
    return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY,
            Constants.PROVIDER));
}
```

同理，由于协议是“registry”，所以不做任何处理，继续调用`RegistryProtocol.export(final Invoker<T> originInvoker)`，如下：

```java
// 暴露远程服务，Invoker转为Exporter
// @param originInvoker DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
// @param <T> 服务接口
// @return Exporter
// @throws RpcException
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // 利用DubboProtocol启动Netty Server，打开本地监听端口，将Invoker转为Exporter
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    // 获取注册中心url: zookeeper://....
    URL registryUrl = getRegistryUrl(originInvoker);

    // 连接注册中心zk，获取注册中心实例(ZookeeperRegistry)
    final Registry registry = getRegistry(originInvoker);
    // 注册到注册中心的提供者url dubbo://....
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    // 是否注册
    boolean register = registeredProviderUrl.getParameter("register", true);
    // 缓存提供者、注册中心等信息
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

    if (register) {
        // 注册服务提供者
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // 订阅服务提供者的override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    // 提供者动态覆盖配置数据变更的监听器
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 进行订阅
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    // 保证每次export都返回一个新的exporter实例
    return new Exporter<T>() {
        @Override
        public Invoker<T> getInvoker() {
            // DelegateProviderMetaDataInvoker实例
            return exporter.getInvoker();
        }

        @Override
        public void unexport() {
            try {
                // 取消暴露远程服务 移除exporter，invoker destroy
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                // 注销服务提供者(zk取消注册)
                registry.unregister(registeredProviderUrl);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                // 移除服务提供者的override数据监听器，取消订阅override数据
                overrideListeners.remove(overrideSubscribeUrl);
                registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    };
}
```

该方法完成了服务远程暴露的全部流程。

- 将`Invoker`转换为`Exporter`
- 启动`Netty`服务端
- 注册服务到`Zookeeper`
- 订阅与通知
- 返回新的`Exporter`实例

### 2.将Invoker转换为Exporter并启动Netty服务

```java
final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
```

`doLocalExport(final Invoker<T> originInvoker)`方法如下:

```java
// 利用DubboProtocol启动Netty服务端，打开监听端口，并生成ExporterChangeableWrapper实例
 流程：
 1 从originInvoker的URL中的Map<String, String> parameters中获取key为export的providerUrl，该url将是服务注册在zk上的节点
 2 从 Map<String, ExporterChangeableWrapper<?>> bounds 缓存中获取key为上述providerUrl的exporter，如果有，直接返回；如果没有，创建并返回
// @param originInvoker DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
// @param <T> 服务接口
// @return ExporterChangeableWrapper实例
@SuppressWarnings("unchecked")
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
    // key(providerUrl): dubbo://...
    String key = getCacheKey(originInvoker); // 根据originInvoker获取providerUrl
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                // getProviderUrl(originInvoker): dubbo://xxx
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker)); // 存储originInvoker和providerUrl
                // protocol.export(invokerDelegete)这行代码启动Netty Server，监听端口 
                // ProtocolListenerWrapper-->ProtocolFilterWrapper-->DubboProtocol
                // protocol.export(invokerDelegete)返回ListenerExporterWrapper实例
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                // providerUrl --> ExporterChangeableWrapper实例
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```

#### 2.1 从originInvoker中获取providerUrl

该方法首先调用`getCacheKey(final Invoker<?> originInvoker)`获取`providerUrl`，这里的`originInvoker`就是上述创建出来的`DelegateProviderMetaDataInvoker`实例(包装了`AbstractProxyInvoker`实例)，注意它的`url`是`registry`协议的，该`url`的`export`参数的值就是要获取的`providerUrl`。获取`providerUrl`的源码如下：

```java
// 根据originInvoker，获取bounds 缓存的 key
// @param originInvoker 原始的服务提供者执行体 DelegateProviderMetaDataInvoker类型
// @return bounds 缓存的 key
private String getCacheKey(final Invoker<?> originInvoker) {
    // providerUrl: dubbo://....
    URL providerUrl = getProviderUrl(originInvoker);
    // 移除dynamic、enabled为key的参数
    return providerUrl.removeParameters("dynamic", "enabled").toFullString();
}

// 通过 invoker 的 url，获取 providerUrl
// @param originInvoker 原始的服务提供者执行体 DelegateProviderMetaDataInvoker类型
// @return 服务提供者配置URL
private URL getProviderUrl(final Invoker<?> originInvoker) {
    // originInvoker.getUrl(): registry://....?export=dubbo://...
    // export=dubbo://xxx
    String export = originInvoker.getUrl().getParameterAndDecoded(Constants.EXPORT_KEY);
    if (export == null || export.length() == 0) {
        throw new IllegalArgumentException("The registry export url is null! registry: " + originInvoker.getUrl());
    }

    return URL.valueOf(export);
}
```

之后一系列的操作，就是获取该`providerUrl key`对应的`exporter`，之后放入缓存`Map<String, ExporterChangeableWrapper<?>> bounds`中，所以一个`providerUrl`只会对应一个`exporter`。

#### 2.2 创建InvokerDelegete

```java
final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
```

`InvokerDelegete`类是`RegistryProtocol`的一个静态内部类，该类实例是`originInvoker`实例的一个委托，该类实例存储了`originInvoker`实例；其父类`InvokerWrapper`还会存储`providerUrl`，`InvokerWrapper`会调用`originInvoker`的`invoke`方法，也会销毁`invoker`。可以管理`invoker`的生命周期。

`InvokerDelegete`类代码:

```java
public static class InvokerDelegete<T> extends InvokerWrapper<T> {

    // 底层的调用执行体 DelegateProviderMetaDataInvoker类型
    private final Invoker<T> invoker;

    // @param invoker DelegateProviderMetaDataInvoker类型
    // @param url     提供者url: dubbo://xxx
    public InvokerDelegete(Invoker<T> invoker, URL url) {
        super(invoker, url);
        this.invoker = invoker;
    }

    public Invoker<T> getInvoker() {
        if (invoker instanceof InvokerDelegete) {
            return ((InvokerDelegete<T>) invoker).getInvoker();
        } else {
            return invoker;
        }
    }
}
```

`InvokerWrapper`类代码

```java
public class InvokerWrapper<T> implements Invoker<T> {

    // 代表一个可执行体，可向它发起 invoke 调用
    private final Invoker<T> invoker; // originInvoker

    // 服务提供者配置
    private final URL url; // providerUrl

    public InvokerWrapper(Invoker<T> invoker, URL url) {
        this.invoker = invoker;
        this.url = url;
    }

    @Override
    public Class<T> getInterface() {
        return invoker.getInterface();
    }

    @Override
    public URL getUrl() {
        return url;
    }

    @Override
    public boolean isAvailable() {
        return invoker.isAvailable();
    }

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    @Override
    public void destroy() {
        invoker.destroy();
    }

}
```

**这样一个`InvokerDelegete`实例就创建好了，属性如下：**

- invoker：`originInvoker(DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例))`
- InvokerWrapper.invoker：`originInvoker(DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例))`
- url：`providerUrl(dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=79559&side=provider&timestamp=1550379785219)`

#### 2.3 使用DubboProtocol将InvokerDelegete转换为Exporter

```java
exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
```

##### 2.3.1 Protocol$Adaptive.export(com.alibaba.dubbo.rpc.Invoker InvokerDelegete实例)

```java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
    if (arg0 == null)
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
    if (arg0.getUrl() == null)
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());//dubbo
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}
```

之后调用`ProtocolListenerWrapper`的`ProtocolListenerWrapper.export(Invoker<T> InvokerDelegete)`，之后调用`ProtocolFilterWrapper.export(Invoker<T> InvokerDelegete)`：首先对`InvokerDelegete`对象进行8个`Filter`的递归包装，之后使用`DubboProtocol`对包装后的`InvokerDelegete`对象进行`export`。

`ProtocolFilterWrapper`中`Filter`包装的代码如下(`buildInvokerChain(final Invoker<T> invoker, String key, String group)`):

```java
// 构建服务调用拦截链。
 1.根据key从url中获取相应的filter的values，再根据这个values和group去获取类上带有@Active注解的filter集合
 2.之后将这些filter对传入的invoker进行递归包装成invoker（就是一个链表）
// @param invoker InvokerDelegete实例
// @param key service.filter
// @param group provider
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker; // 最后调用的invoker，原invoker最后执行
    // 拦截器列表
    // 根据order从小到大排序
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class)
            .getActivateExtension(invoker.getUrl(), key, group);
    if (filters.size() > 0) {
        // 构建调用链，从filter(0)开始执行到filter(size()-1)，最后到原invoker
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last; // 下一个被调用的invoker

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
```

这里列出一个`Filter`的源码：

```java
@Activate(group = Constants.PROVIDER, order = -110000)
public class EchoFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Object[] arguments = invocation.getArguments();
        if (invocation.getMethodName().equals(Constants.$ECHO)
                && arguments != null && arguments.length == 1) {
            return new RpcResult(arguments[0]);
        }
        return invoker.invoke(invocation);
    }

}
```

可以看到，该`Filter`会调用传入的`next invoker`的`invoke`方法。

最后给出被递归包装后的对象：（命名为**InvokerDelegete的Filter对象**）

```java
EchoFilter
-->ClassLoaderFilter
   -->GenericFilter
      -->ContextFilter
         -->TraceFilter
            -->TimeoutFilter
               -->MonitorFilter
                  -->ExceptionFilter
                     -->InvokerDelegete对象
```

##### 2.3.2 DubboProtocol.export(Invoker`<T>` InvokerDelegete的Filter对象)

```java
// Invoker 导出为 Exporter
 1.根据invoker及url获取将要暴露的远程服务的key(serviceGroup/serviceName:serviceVersion:port)，即com.alibaba.dubbo.demo.DemoService:20881。注意：本地暴露的key是com.alibaba.dubbo.demo.DemoService
 2.打开ExchangeServer
// @param invoker 服务的执行体 Filter包裹后的Invoker--InvokerDelegete
// @param <T> 服务接口类型
// @return Exporter
// @throws RpcException
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 提供者url dubbo://...
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url); // e.g. com.alibaba.dubbo.demo.DemoService:20881
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispaching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                                                      "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    // 启动Netty Server
    openServer(url);

    return exporter;
}
```

首先从“**InvokerDelegete的Filter对象**”中的`url`获取`key`，即获取`serviceGroup/serviceName:serviceVersion:port`这样形式的一个`key`，最后获取到的是`com.alibaba.dubbo.demo.DemoService:20881`。之后创建`DubboExporter`。

###### 2.3.2.1 new DubboExporter`<T>`(InvokerDelegete的Filter对象, "com.alibaba.dubbo.demo.DemoService:20881", exporterMap)

```java
public class DubboExporter<T> extends AbstractExporter<T> {
    // 服务关键字([group/]interface[:version]:port)，如com.alibaba.dubbo.demo.DemoService:20881
    private final String key;

    // 服务关键字([group/]interface[:version]:port) ---> DubboExporter实例
    // { "com.alibaba.dubbo.demo.DemoService:20881" -> 当前的DubboExporter实例 }
    private final Map<String, Exporter<?>> exporterMap;

    public DubboExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
    }

    @Override
    public void unexport() {
        super.unexport();
        exporterMap.remove(key);
    }

}
```

注意这里的`exporterMap`是引用传递。

`DubboExporter`的父类:

```java
public abstract class AbstractExporter<T> implements Exporter<T> {

    protected final Logger logger = LoggerFactory.getLogger(getClass());

    // 服务提供者执行体
    private final Invoker<T> invoker;

    // 服务已取消暴露标识
    private volatile boolean unexported = false;

    public AbstractExporter(Invoker<T> invoker) {
        if (invoker == null)
            throw new IllegalStateException("service invoker == null");
        if (invoker.getInterface() == null)
            throw new IllegalStateException("service type == null");
        if (invoker.getUrl() == null)
            throw new IllegalStateException("service url == null");
        this.invoker = invoker;
    }

    @Override
    public Invoker<T> getInvoker() {
        return invoker;
    }

    // 取消暴露服务提供者
    @Override
    public void unexport() {
        if (unexported) {
            return;
        }
        unexported = true;
        getInvoker().destroy();
    }

    @Override
    public String toString() {
        return getInvoker().toString();
    }

}
```

这里把一个“`InvokerDelegete的Filter对象`”赋给了`AbstractExporter`的`Invoker`引用，也就是说从`exporter`中可以获取到`invoker`。最后在`DubboProtocol.export(Invoker<T> invoker)`中执行：`exporterMap.put(key, exporter); `这样就将`{ "com.alibaba.dubbo.demo.DemoService:20881" -> 当前的DubboExporter实例 }`存储起来了。

来看一下现在的`DubboExporter`实例：

- key：`com.alibaba.dubbo.demo.DemoService:20881`
- invoker：“`InvokerDelegete的Filter对象`”
- exporterMap：`{ "com.alibaba.dubbo.demo.DemoService:20881" -> 当前的DubboExporter实例 }`

###### 2.3.2.2 开启ExchangeServer

```java
// 从缓存Map<String, ExchangeServer> serverMap中根据"host:port"获取ExchangeServer，如果没有，创建ExchangeServer，之后放入缓存。
// @param url 提供者url dubbo://.....
private void openServer(URL url) {
    // find server. host:port
    String key = url.getAddress();
    // 是否是Server端
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            // 如果没缓存，直接创建server
            serverMap.put(key, createServer(url));
        } else {
            // server支持reset,配合override功能使用
            server.reset(url);
        }
    }
}
```

首先从`provderUrl`中获取`host:port`作为`key`，之后从缓存`serverMap`中获取`ExchangeServer`，如果没有，创建`ExchangeServer`，最后以如下方式放入缓存：`Map<String, ExchangeServer> serverMap：{ "172.16.132.166:20881"<->ExchangeServer实例 }`。

**(1)创建ExchangeServer：createServer(URL providerUrl)**

```java
// 创建ExchangeServer
// @param url 提供者url dubbo://...
private ExchangeServer createServer(URL url) {
    // 默认开启server关闭时发送readonly事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // 默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    // 协议的服务端实现类型 netty
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
    // 检查服务端Transporter扩展是否存在
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);
    }

    // 编解码器
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        // 返回HeaderExchangerServer
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

首先是在原本`providerUrl`上添加参数：`channel.readonly.sent=true&heartbeat=60000&codec=dubbo`。(其中的`heartbeat`参数会在`HeaderExchangeServer`启动心跳定时器时使用)

之后使用`Exchangers.bind("添加参数后的providerUrl", requestHandler)`创建`ExchangeServer`。首先来看一下`DubboProtocol.requestHandler`实例。这个实例极其重要，后续经过层层包装后，会成为最终`Netty`的服务端逻辑处理器。

```java
/** 请求处理器 */
private final ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
    // 响应请求
    // @param channel ExchangeChannel
    // @param message Invocation
    @Override
    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv); // InvokerDelegete
            //如果是callback 需要处理高版本调用低版本的问题
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName() + " not found in callback service interface ,invoke will be ignored. please update the api interface. url is:" + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            return invoker.invoke(inv);
        }
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            reply((ExchangeChannel) channel, message);
        } else {
            super.received(channel, message);
        }
    }

    @Override
    public void connected(Channel channel) throws RemotingException {
        invoke(channel, Constants.ON_CONNECT_KEY);
    }

    @Override
    public void disconnected(Channel channel) throws RemotingException {
        if (logger.isInfoEnabled()) {
            logger.info("disconnected from " + channel.getRemoteAddress() + ", url: " + channel.getUrl());
        }
        invoke(channel, Constants.ON_DISCONNECT_KEY);
    }

    private void invoke(Channel channel, String methodKey) {
       	// 创建Invocation
        Invocation invocation = createInvocation(channel.getUrl(), methodKey);
        if (invocation != null) {
            try {
                received(channel, invocation);
            } catch (Throwable t) {
                logger.warn("Failed to invoke event method " + invocation.getMethodName() + "(), cause: " + t.getMessage(), t);
            }
        }
    }

    // 创建Invocation
    private Invocation createInvocation(URL url, String methodKey) {
        String method = url.getParameter(methodKey);
        if (method == null || method.length() == 0) {
            return null;
        }
        RpcInvocation invocation = new RpcInvocation(method, new Class<?>[0], new Object[0]);
        invocation.setAttachment(Constants.PATH_KEY, url.getPath());
        invocation.setAttachment(Constants.GROUP_KEY, url.getParameter(Constants.GROUP_KEY));
        invocation.setAttachment(Constants.INTERFACE_KEY, url.getParameter(Constants.INTERFACE_KEY));
        invocation.setAttachment(Constants.VERSION_KEY, url.getParameter(Constants.VERSION_KEY));
        if (url.getParameter(Constants.STUB_EVENT_KEY, false)) {
            invocation.setAttachment(Constants.STUB_EVENT_KEY, Boolean.TRUE.toString());
        }
        return invocation;
    }
};
```

从上可以看出在该`handler`中，定义了**与客户端连接成功／断开连接／接受到客户端消息／响应消息，以及创造Invocation的方法**。其中的`getInvoker(Channel channel, Invocation inv)`方法简码如下：

```java
String serviceKey = serviceKey(port, path,inv.getAttachments().get(Constants.VERSION_KEY), inv.getAttachments().get(Constants.GROUP_KEY));
DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
return exporter.getInvoker();
```

这就是刚刚放置到`exporterMap`中的`DubboExporter`，而其中的`invoker`就是**“Filter的invokerdelegete对象”**。

**(2)使用Exchangers.bind(providerUrl, ExchangeHandlerAdapter对象)创建ExchangeServer**

```java
// 返回HeaderExchangeServer
// @param url 提供者url
// @param handler 请求消息数据交换处理器  ExchangeHandlerAdapter对象
// @return HeaderExchangeServer
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // getExchanger(url) --> HeaderExchanger
    return getExchanger(url).bind(url, handler);
}
public static Exchanger getExchanger(URL url) {
    String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
    return getExchanger(type);
}

public static Exchanger getExchanger(String type) {
    return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
}
```

`getExchanger(URL url)`返回一个`HeaderExchanger`实例。所以`ExchangeServer`的创建交由`HeaderExchanger`来实现。

**(3)HeaderExchanger.bind(providerUrl, ExchangeHandlerAdapter对象)** 

```java
// 绑定数据交换服务器。对handler进行两次包装：首先将ExchangeHandlerAdapter赋给HeaderExchangeHandler中的ExchangeHandler handler属性；然后将创建出来的HeaderExchangeHandler赋给DecodeHandler的父类AbstractChannelHandlerDelegate的ChannelHandler handler属性
// @param url 提供者url
// @param handler 请求处理器ExchangeHandler requestHandler = new ExchangeHandlerAdapter(){}
// @return HeaderExchangeServer
@Override
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    // Transporters.bind() --> NettyServer实例
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

说明：

- 这里首先对传入的`ExchangeHandlerAdapter`实例进行了两次包装，最终得到`DecodeHandler`实例；
- 之后，使用`Transporters.bind(providerUrl, DecodeHandler对象)`创建了一个`NettyServer`；
- 最后使用`HeaderExchangeServer`包装了上边的`NettyServer`，并启动了心跳定时器。
  - `HeaderExchangeServer`实例也是最终返回的`ExchangeServer`实例，将最终被存储在`Map<String, ExchangeServer> serverMap：{ "172.16.132.166:20881"<->HeaderExchangeServer实例 }`

包装`ExchangeHandlerAdapter`，获取`DecodeHandler`实例的逻辑比较简单这里不再详述。

最终获取到的`DecodeHandler`实例的层级关系：

```java
DecodeHandler实例
	-->HeaderExchangeHandler实例
		-->ExchangeHandlerAdapter实例
```

下面分析使用`Transporters.bind(providerUrl, DecodeHandler对象)`创建一个`NettyServer`的流程。

**(4)Transporters.bind(providerUrl, DecodeHandler对象)**

```java
// @param url 提供者url
// @param handlers DecodeHandler实例
// @return NettyServer
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    // 返回NettyServer
    return getTransporter().bind(url, handler);
}

public static Transporter getTransporter() {
    return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
}
```

**(5)Transporter$Adaptive.bind(providerUrl, DecodeHandler对象)**

```java
public com.alibaba.dubbo.remoting.Server bind(com.alibaba.dubbo.common.URL arg0, com.alibaba.dubbo.remoting.ChannelHandler arg1) throws com.alibaba.dubbo.remoting.RemotingException {
    if (arg0 == null)
        throw new IllegalArgumentException("url == null");
    com.alibaba.dubbo.common.URL url = arg0;
    String extName = url.getParameter("server", url.getParameter("transporter", "netty"));//netty
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString() + ") use keys([server, transporter])");
    com.alibaba.dubbo.remoting.Transporter extension = (com.alibaba.dubbo.remoting.Transporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Transporter.class).getExtension(extName);
    return extension.bind(arg0, arg1);
}
```

最后`NettyServer`的创建由`NettyTransporter`来创建。

**(6)NettyTransporter.bind(providerUrl, DecodeHandler对象)**

```java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty";

    // @param url 提供者url
    // @param listener 事件监听器 DecodeHandler实例
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }

    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```

**(7)new NettyServer(providerUrl, DecodeHandler对象)**

```java
NettyServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
}
```

这里首先为`providerUrl`添加参数：`threadname=DubboServerHandler-172.16.132.166:20881`（`ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)`）；

之后，使用`ChannelHandlers.wrap(DecodeHandler对象, providerUrl)`对`DecodeHandler`对象进行了**三层包装**，最终得到`MultiMessageHandler`实例；

最后调用父类的构造器初始化`NettyServer`的各个属性，最后启动`Netty`。

**(8)ChannelHandlers.wrap(DecodeHandler对象, providerUrl)**

```java
// 这里又是层层包裹：
// MultiMessageHandler
// --HeartbeatHandler
//   --AllChannelHandler
//     --DecodeHandler
//       --HeaderExchangeHandler
//         --ExchangeHandlerAdapter
// @param handler DecodeHandler实例
// @param url 提供者url
protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
    return new MultiMessageHandler(new HeartbeatHandler(ExtensionLoader.getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
}
```

`ExtensionLoader.getExtensionLoader(Dispatcher.class).getAdaptiveExtension()`获取到一个`Dispatcher$Adaptive`适配类。

**(9)Dispatcher$Adaptive.dispatch(DecodeHandler对象, providerUrl)**

```java
public com.alibaba.dubbo.remoting.ChannelHandler dispatch(com.alibaba.dubbo.remoting.ChannelHandler arg0, com.alibaba.dubbo.common.URL arg1) {
    if (arg1 == null)
        throw new IllegalArgumentException("url == null");
    com.alibaba.dubbo.common.URL url = arg1;
    String extName = url.getParameter("dispatcher", url.getParameter("dispather", url.getParameter("channel.handler", "all")));//all
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.Dispatcher) name from url(" + url.toString() + ") use keys([dispatcher, dispather, channel.handler])");
    com.alibaba.dubbo.remoting.Dispatcher extension = (com.alibaba.dubbo.remoting.Dispatcher)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.Dispatcher.class).getExtension(extName);
    return extension.dispatch(arg0, arg1);
}
```

这里获取到`AllDispatcher`实例，**`Dispatcher`决定了`Dubbo`的线程模型**，指定了哪些线程做什么。讲到`Dubbo`通信的时候再详述。

**(10)AllDispatcher.dispatch(DecodeHandler对象, providerUrl)**

```java
// @param handler DecodeHandler实例
// @param url 提供者url
@Override
public ChannelHandler dispatch(ChannelHandler handler, URL url) {
    return new AllChannelHandler(handler, url);
}
```

**(11)new AllChannelHandler(DecodeHandler对象, providerUrl)**

```java
// @param handler DecodeHandler实例
// @param url 提供者url
AllChannelHandler(ChannelHandler handler, URL url) {
    super(handler, url);
}
```

下面看`AllChannelHandler`父类`WrappedChannelHandler`的构造器：

**(12)WrappedChannelHandler(DecodeHandler对象, providerUrl)**

```java
protected static final ExecutorService SHARED_EXECUTOR = Executors.newCachedThreadPool(
        new NamedThreadFactory("DubboSharedHandler", true));
protected final ExecutorService executor;
protected final ChannelHandler handler;
protected final URL url;

// @param handler DecodeHandler实例
// @param url 提供者url
public WrappedChannelHandler(ChannelHandler handler, URL url) {
    this.handler = handler;
    this.url = url;
    // FixedThreadPool.getExecutor()
    executor = (ExecutorService) ExtensionLoader.getExtensionLoader(ThreadPool.class)
            .getAdaptiveExtension().getExecutor(url);

    String componentKey = Constants.EXECUTOR_SERVICE_COMPONENT_KEY;
    if (Constants.CONSUMER_SIDE.equalsIgnoreCase(url.getParameter(Constants.SIDE_KEY))) {
        componentKey = Constants.CONSUMER_SIDE;
    }
    DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
    dataStore.put(componentKey, Integer.toString(url.getPort()), executor);
}
```

首先创建了一个共享线程池：`SHARED_EXECUTOR`；之后为`handler／url／executor`赋值，其中`executor`是一个200个线程的`fixed`线程池（队列为0，即同步队列）。

```java
// 获取线程池
public Executor getExecutor(URL url) {
    String name = url.getParameter(Constants.THREAD_NAME_KEY, Constants.DEFAULT_THREAD_NAME);// 默认为dubbo，但是这里是DubboServerHandler-172.16.132.166:20881（就是之前设置到url上的threadname）
    int threads = url.getParameter(Constants.THREADS_KEY, Constants.DEFAULT_THREADS);//200
    int queues = url.getParameter(Constants.QUEUES_KEY, Constants.DEFAULT_QUEUES);//0
    return new ThreadPoolExecutor(threads, threads, 0, TimeUnit.MILLISECONDS,
                                  queues == 0 ? new SynchronousQueue<Runnable>() :
                                  (queues < 0 ? new LinkedBlockingQueue<Runnable>()
                                   : new LinkedBlockingQueue<Runnable>(queues)),
                                  new NamedThreadFactory(name, true), new AbortPolicyWithReport(name, url));
}
```

之后获取了一个数据存储器：`SimpleDataStore`，并将`{"java.util.concurrent.ExecutorService":{"20881": executo线程池}}`数据存储在`SimpleDataStore`的`ConcurrentMap<String, ConcurrentMap<String, Object>> data`数据结构中。也就是说：每一个端口，有一个线程池。

**注意：为什么SimpleDataSource可以做缓存来使用？**

```java
// ExtensionLoader方法
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
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
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

```

如上述代码所示，`SimpleDataStore`实例会存储在`cachedInstances`缓存中，下一次获取不会再创建，而是直接获取该缓存实例。

这样之后，一个`AllChannelHandler`实例就创建完成了，该实例属性如下：

- WrappedChannelHandler.url：`dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&channel.readonly.sent=true&codec=dubbo&dubbo=2.0.0&generic=false&heartbeat=60000&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=82894&side=provider&threadname=DubboServerHandler-172.16.132.166:20881&timestamp=1550395492993`
- WrappedChannelHandler.handler：`DecodeHandler对象`
- WrappedChannelHandler.executor：`FixedThreadPool`实例

当然还有一个类变量`WrappedChannelHandler.SHARED_EXECUTOR=CachedThreadPool`实例。

之后`AllChannelHandler`实例会被`HeartbeatHandler`进行包裹，之后`HeartbeatHandler`实例又会被`MultiMessageHandler`所包裹，最后得到的`MultiMessageHandler`实例的层级结构如下：

```java
MultiMessageHandler
-->handler: HeartbeatHandler
   -->handler: AllChannelHandler
         -->url: providerUrl
         -->executor: FixedExecutor
         -->handler: DecodeHandler
            -->handler: HeaderExchangeHandler
               -->handler: ExchangeHandlerAdapter
```

**(13)NettyServer属性的初始化**

`MultiMessageHandler`实例创建出来之后，`NettyServer`就开始调用其各个父类进行属性的初始化了。首先来看一下`NettyServer`的父类层级图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/NettyServer%E7%B1%BB%E7%BB%A7%E6%89%BF%E5%9B%BE.png?x-oss-process=style/markdown-pic)

`AbstractServer`类：

```java
protected static final String SERVER_THREAD_POOL_NAME = "DubboServerHandler";
private ExecutorService executor;
private InetSocketAddress localAddress; // 服务端地址
private InetSocketAddress bindAddress; // bind 地址
private int accepts;
private int idleTimeout = 600; //600 seconds

public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler); // MultiMessageHandler
    localAddress = getUrl().toInetSocketAddress(); //  /172.16.132.166:20881

    String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
    int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
    if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
        bindIp = NetUtils.ANYHOST; // 0.0.0.0
    }
    bindAddress = new InetSocketAddress(bindIp, bindPort); // /0.0.0.0:20881
    this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
    this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT); // 单位秒
    try {
        doOpen();
        if (logger.isInfoEnabled()) {
            // getClass(): Class<NettyServer>
            logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
        }
    } catch (Throwable t) {
        throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
    }
    //fixme replace this with better method
    DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
    // executor怎么获取的 --> ChannelHandlers
    executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
}
```

首先调用父类`AbstractEndpoint`初始化属性，之后启动服务。

```java
// 编解码器 DubboCountCodec
private Codec2 codec;
// 远程服务调用超时时间
private int timeout;
// 连接超时时间
private int connectTimeout;

public AbstractEndpoint(URL url, ChannelHandler handler) {
    super(url, handler);
    this.codec = getChannelCodec(url); // dubbo
    this.timeout = url.getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT); // RPC调用超时，默认1s
    this.connectTimeout = url.getPositiveParameter(Constants.CONNECT_TIMEOUT_KEY, Constants.DEFAULT_CONNECT_TIMEOUT); // 连接超时，默认3s
}
```

`AbstractPeer`类：

```java
private final ChannelHandler handler; // MultiMessageHandler实例

private volatile URL url; // 提供者url

// closing closed分别表示关闭流程中、完成关闭
private volatile boolean closing;

private volatile boolean closed;

public AbstractPeer(URL url, ChannelHandler handler) {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    this.url = url;
    this.handler = handler;
}
```

来看一下最后初始化好的`NettyServer`实例的属性：

- url：`providerUrl（dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&channel.readonly.sent=true&codec=dubbo&dubbo=2.0.0&generic=false&heartbeat=60000&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=82894&side=provider&timestamp=1550395492993）`
- handler：`MultiMessageHandler`实例
- codec：`DubboCountCodec`实例
- timeout：1000
- connectTimeout：3000
- idleTime：600*1000
- localAddress：172.16.132.166:20880
- bindAddress：0.0.0.0:20880
- accepts：0
- executor：null（此时的executor还没被赋值，要等`Netty`服务起来之后才会从缓存中获取之前存储在`SimpleDataStore`缓存中的那个200个线程数的`FixedThreadPool`实例）

**(14)启动Netty服务**

现在就要启动`Netty`服务了。

```java
// 启动netty服务，监听客户端连接
protected void doOpen() throws Throwable {
    // 设置logger factory
    NettyHelper.setNettyLoggerFactory();
    // boss worker线程池
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker,
            getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    bootstrap = new ServerBootstrap(channelFactory); // 线程模型、IO模型
    // getUrl()----提供者url dubbo://....
    // NettyHandler----ChannelHandler 处理入站、出站事件
    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    // <ip:port(消费者), channel(NettyChannel)>
    channels = nettyHandler.getChannels();
    // https://issues.jboss.org/browse/NETTY-365
    // https://issues.jboss.org/browse/NETTY-379
    // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
    // 设置pipeline创建工厂
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            // 构造器参数:<DubboCountCodec实例，提供者url，ChannelHandler(当前NettyServer实例)>
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            /*int idleTimeout = getIdleTimeout();
            if (idleTimeout > 10000) {
                pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
            }*/
            pipeline.addLast("decoder", adapter.getDecoder()); // 解码器
            pipeline.addLast("encoder", adapter.getEncoder()); // 编码器
            pipeline.addLast("handler", nettyHandler); // 服务端逻辑处理器，处理入站、出站事件
            return pipeline;
        }
    });
    // bind
    channel = bootstrap.bind(getBindAddress());
}
```

说明：

- `boss`线程数默认只有一个；
- `worker`线程数：`Runtime.getRuntime().availableProcessors() + 1`，为计算机核数+1；
- 服务端逻辑处理器为`NettyHandler`：
- 编码器为：`InternalEncoder`实例，内部使用`NettyServer`的`DubboCountCodec`实例来编码
- 解码器为：`InternalDecoder`实例，内部使用`NettyServer`的`DubboCountCodec`实例来解码

` NettyHandler`：

```java
@Sharable
public class NettyHandler extends SimpleChannelHandler {

    // NettyHandler初始化的时候，初始化channels变量
    private final Map<String, Channel> channels = new ConcurrentHashMap<String, Channel>(); // <ip:port(消费者), NettyChannel>
    // 提供者url
    private final URL url;
    // NettyServer实例
    private final ChannelHandler handler;

    public NettyHandler(URL url, ChannelHandler handler) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.url = url;
        this.handler = handler;
    }

    public Map<String, Channel> getChannels() {
        return channels;
    }

    @Override
    public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            if (channel != null) {
                // Channel连接的时候添加到channels map
                channels.put(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()), channel);
            }
            handler.connected(channel);
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }

    @Override
    public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            // Channel断开连接的时候从channels map移除
            channels.remove(NetUtils.toAddressString((InetSocketAddress) ctx.getChannel().getRemoteAddress()));
            handler.disconnected(channel);
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }

    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            handler.received(channel, e.getMessage());
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }

    @Override
    public void writeRequested(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        super.writeRequested(ctx, e);
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            handler.sent(channel, e.getMessage());
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            handler.caught(channel, e.getCause());
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }

}
```

说明：

属性

- handler：当前的`NettyServer`实例
- url：`providerUrl`
- channels：存放连接到来的`channel`

监听连接完成／连接断开／接收到消息／发送完消息／异常捕捉事件，之后使用`NettyServer`实例进行相应的处理，`NettyServer`又会调用`MultiMessageHandler`实例（该`handler`属性位于`NettyServer`的父类`AbstractPeer`中）进行处理。

再来看编码器和解码器：

**NettyCodecAdapter(DubboCountCodec实例, providerUrl, 当前的NettyServer实例)**

```java
final class NettyCodecAdapter {
    private final ChannelHandler encoder = new InternalEncoder();
    private final ChannelHandler decoder = new InternalDecoder();
    private final Codec2 codec;
    private final URL url;
    private final int bufferSize;
    private final com.alibaba.dubbo.remoting.ChannelHandler handler; // 当前的NettyServer实例

    public NettyCodecAdapter(Codec2 codec, URL url, com.alibaba.dubbo.remoting.ChannelHandler handler) {
        this.codec = codec;
        this.url = url;
        this.handler = handler;
        int b = url.getPositiveParameter(Constants.BUFFER_KEY, Constants.DEFAULT_BUFFER_SIZE);// 8*1024
        this.bufferSize = b >= Constants.MIN_BUFFER_SIZE && b <= Constants.MAX_BUFFER_SIZE ? b : Constants.DEFAULT_BUFFER_SIZE;// 8*1024
    }

    public ChannelHandler getEncoder() {
        return encoder;
    }

    public ChannelHandler getDecoder() {
        return decoder;
    }

    @Sharable
    private class InternalEncoder extends OneToOneEncoder {
        @Override
        protected Object encode(ChannelHandlerContext ctx, Channel ch, Object msg) throws Exception {
            ...
            codec.encode(channel, buffer, msg);
            ...
        }
    }

    private class InternalDecoder extends SimpleChannelUpstreamHandler {
        @Override
        public void messageReceived(ChannelHandlerContext ctx, MessageEvent event) throws Exception {
           ...
            msg = codec.decode(channel, message);
           ...
        }
        ...
    }
}
```

可以看到，`InternalEncoder`实例和`InternalDecoder`实例内部还是使用`NettyServer`的`DubboCountCodec`实例来编解码的。

到此为止，`NettyServer`就创建成功了。 之后，执行到：

**(15)new HeaderExchangeServer(Server NettyServer)**

```java
private final ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(1,
        new NamedThreadFactory("dubbo-remoting-server-heartbeat", true));
// 远程服务器 NettyServer实例
private final Server server;
// 心跳定时器
private ScheduledFuture<?> heartbeatTimer;
// 心跳间隔
private int heartbeat;
// 心跳超时，毫秒。缺省0，不会执行心跳。
private int heartbeatTimeout;

private final AtomicBoolean closed = new AtomicBoolean(false);

// server---NettyServer实例
public HeaderExchangeServer(Server server) {
    if (server == null) {
        throw new IllegalArgumentException("server == null");
    }
    this.server = server;
    // 60000 在createServer(URL providerUrl)中拼接了heartbeat参数
    this.heartbeat = server.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
    // 3*60000
    this.heartbeatTimeout = server.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
    if (heartbeatTimeout < heartbeat * 2) {
        throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
    }
    startHeartbeatTimer();
}
```

说明：

- 属性
  - scheduled：是一个有1个名字为`dubbo-remoting-server-heartbeat`的后台线程的定时线程池；
  - server：之前创建出来的`NettyServer`实例；
  - heartbeatTimer：心跳定时器；
  - heartbeat：心跳间隔，该参数会在`HeaderExchangeServer`的构造器中进行赋值，60000
  - heartbeatTimeout：心跳超时时间(客户端在心跳超时时，会进行channel重连)，180000
- 启动心跳定时器

**startHeatbeatTimer()**

```java
private void startHeartbeatTimer() {
    stopHeartbeatTimer();
    if (heartbeat > 0) {
        heartbeatTimer = scheduled.scheduleWithFixedDelay(
                new HeartBeatTask(new HeartBeatTask.ChannelProvider() {
                    @Override
                    public Collection<Channel> getChannels() {
                        return Collections.unmodifiableCollection(
                                HeaderExchangeServer.this.getChannels());
                    }
                }, heartbeat, heartbeatTimeout),
                heartbeat, heartbeat, TimeUnit.MILLISECONDS);
    }
}

private void stopHeartbeatTimer() {
    try {
        ScheduledFuture<?> timer = heartbeatTimer;
        if (timer != null && !timer.isCancelled()) {
            timer.cancel(true);
        }
    } catch (Throwable t) {
        logger.warn(t.getMessage(), t);
    } finally {
        heartbeatTimer = null;
    }
}
```

首先停掉之前的定时器，之后创建心跳检测任务`HeartBeatTask`，该任务在任务创建`heartbeat`毫秒（60s）后第一次执行，之后每隔`heartbeat`毫秒（60s）执行一次任务。来看一下`HeartBeatTask`:

`HeartBeatTask`

```java
final class HeartBeatTask implements Runnable {

    private static final Logger logger = LoggerFactory.getLogger(HeartBeatTask.class);
    // 底层的Channels
    private ChannelProvider channelProvider;
    // 心跳间隔，默认60s
    private int heartbeat;
    // 心跳超时时间，默认180s
    private int heartbeatTimeout;

    HeartBeatTask(ChannelProvider provider, int heartbeat, int heartbeatTimeout) {
        this.channelProvider = provider;
        this.heartbeat = heartbeat;
        this.heartbeatTimeout = heartbeatTimeout;
    }

    @Override
    public void run() {
        try {
            long now = System.currentTimeMillis();
            for (Channel channel : channelProvider.getChannels()) { // ExchangeChannel，包装NettyChannel
                if (channel.isClosed()) {
                    continue;
                }
                try {
                    Long lastRead = (Long) channel.getAttribute(
                            HeaderExchangeHandler.KEY_READ_TIMESTAMP); // 读时间戳 "READ_TIMESTAMP"
                    Long lastWrite = (Long) channel.getAttribute(
                            HeaderExchangeHandler.KEY_WRITE_TIMESTAMP); // 写时间戳 "READ_TIMESTAMP"
                    // channel没有数据读写，因此发送心跳包
                    // 如果最后一次读和写在heartbeat时间（60s）内，则最后一次的读和写本身可以看作心跳；否则，需要程序发送心跳
                    if ((lastRead != null && now - lastRead > heartbeat)
                            || (lastWrite != null && now - lastWrite > heartbeat)) {
                        Request req = new Request();
                        req.setVersion("2.0.0");
                        req.setTwoWay(true);
                        req.setEvent(Request.HEARTBEAT_EVENT);
                        channel.send(req);
                        if (logger.isDebugEnabled()) {
                            logger.debug("Send heartbeat to remote channel " + channel.getRemoteAddress()
                                    + ", cause: The channel has no data-transmission exceeds a heartbeat period: " + heartbeat + "ms");
                        }
                    }
                    // 如果最后一次读的时间距离现在已经超过heartbeatTimeout了，我们认为channel已经断了（因为在这个过程中，送了三次心跳都没反应），此时channel进行重连(客户端)
                    if (lastRead != null && now - lastRead > heartbeatTimeout) {
                        logger.warn("Close channel " + channel
                                + ", because heartbeat read idle time out: " + heartbeatTimeout + "ms");
                        if (channel instanceof Client) {
                            try {
                                ((Client) channel).reconnect(); // 客户端channel，重连服务端
                            } catch (Exception ignored) {
                                // ignore
                            }
                        } else {
                            channel.close(); // 服务端channel，则直接将与客户端的连接channel关闭
                        }
                    }
                } catch (Throwable t) {
                    logger.warn("Exception when heartbeat to remote channel " + channel.getRemoteAddress(), t);
                }
            }
        } catch (Throwable t) {
            logger.warn("Unhandled exception when heartbeat, cause: " + t.getMessage(), t);
        }
    }

    interface ChannelProvider {
        Collection<Channel> getChannels();
    }

}
```

说明：

- 属性
  - channelProvider：在`startHeatbeatTimer()`中创建，并且获取了当前的`HeaderExchangeServer`的所有`channels`
  - heartbeat：60s
  - heartbeatTimeout：180s
- run()
  - 如果最后一次读和写的时间距离现在在`heartbeat`时间（60s）内，则最后一次的读和写本身可以看作心跳；否则，发送心跳；
  - 如果最后一次读的时间距离现在已经超过`heartbeatTimeout`了，认为`channel`已经断了（因为在这个过程中，发送了三次心跳都没反应），此时`channel`进行重连(客户端)。

到现在一个完整的`ExchangeServer`就创建完成了。之后将创建出来的`ExchangeServer`实例存放在`DubboProtocol`的`Map<String, ExchangeServer> serverMap`属性中：`{ "172.16.132.166:20880" : ExchangeServer实例 }`

最后，`DubboProtocol.export(Invoker<T> invoker)`将之前创建的`DubboExporter`实例返回。

#### 2.4 创建RegistryProtocol.ExporterChangeableWrapper来封装Exporter和originInvoker

```java
exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
```

```java
// exporter代理,建立返回的exporter与protocol export出的exporter的对应关系，在override时可以进行关系修改.
private class ExporterChangeableWrapper<T> implements Exporter<T> {
    // DelegateProviderMetaDataInvoker类型
    private final Invoker<T> originInvoker;
    // ListenerExporterWrapper实例
    private Exporter<T> exporter;

    ExporterChangeableWrapper(Exporter<T> exporter, Invoker<T> originInvoker) {
        this.exporter = exporter;
        this.originInvoker = originInvoker;
    }

    public Invoker<T> getOriginInvoker() {
        return originInvoker;
    }

    @Override
    public Invoker<T> getInvoker() {
        // ListenerExporterWrapper.getInvoker() --> DubboProtocol.getInvoker()，最终返回DelegateProviderMetaDataInvoker类型
        return exporter.getInvoker();
    }

    public void setExporter(Exporter<T> exporter) {
        this.exporter = exporter;
    }

    @Override
    public void unexport() {
        // dubbo://....
        String key = getCacheKey(this.originInvoker);
        bounds.remove(key);
        exporter.unexport();
    }
}
```

`ExporterChangeableWrapper`类是`RegistryProtocol`的私有内部类**。**

最后，将`<providerUrl, ExporterChangeableWrapper实例>`放入`RegistryProtocol`的属性`Map<String, ExporterChangeableWrapper<?>> bounds`中。

- key：`dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=84068&side=provider&timestamp=1550405355720`
- value：`RegistryProtocol$ExporterChangeableWrapper`实例
  - originInvoker: `DelegateProviderMetaDataInvoker`实例，属性如下：
    - invoker :  `AbstractProxyInvoker`实例，属性如下:
      - proxy: `DemoServiceImpl2`实例
      - type: `Class<com.alibaba.dubbo.demo.DemoService>`
      - url: `registry://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&export=dubbo%3A%2F%2F172.16.132.166%3A20881%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D172.16.132.166%26bind.port%3D20881%26cellinvokemode%3Dsharing%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D84068%26side%3Dprovider%26timestamp%3D1550405355720&group=dubbo_test&pid=84068&registry=zookeeper&timestamp=1550405355702`
    - metadata:  `ServiceBean`实例
  - exporter: `ListenerExporterWrapper`实例
    - exporter:  `DubboExporter`实例，属性如下：
      - key：`com.alibaba.dubbo.demo.DemoService:20881`
      - invoker: `"InvokerDelegete的Filter对象"`
      - exporterMap：`{ "com.alibaba.dubbo.demo.DemoService:20881" -> 当前的DubboExporter实例 }`

到此为止，`RegistryProtocol.export(final Invoker<T> originInvoker)`的第一行代码就分析完成了。

```java
// 利用DubboProtocol启动Netty Server，打开本地监听端口，将Invoker转为Exporter
final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
```

