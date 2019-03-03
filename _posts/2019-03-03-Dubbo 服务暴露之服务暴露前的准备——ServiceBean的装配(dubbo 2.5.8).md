---
layout:     post
title:      Dubbo 服务暴露之服务暴露前的准备——ServiceBean的装配(基于dubbo 2.5.8)
subtitle:   ServiceBean/ServiceConfig
date:       2019-03-03
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo 服务暴露之服务暴露前的准备——ServiceBean的装配(基于dubbo 2.5.8)

`Dubbo`的服务暴露流程以`Dubbo`源码中提供的`dubbo-demo-provider`为例来分析，以下是`dubbo-demo-provider`的xml配置:

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
   http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="demo-provider"/>
	
    <!-- 使用zookeeper注册中心，并使用zkClient客户端，zk根节点为dubbo_test -->
    <dubbo:registry protocol="zookeeper" address="localhost:2181" client="zkclient" group="dubbo_test"/>

    <!-- 用dubbo协议在20881端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20881"/>

    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl2"/>

    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```

服务暴露是由`com.alibaba.dubbo.config.spring.ServiceBean`这个类的实例来实现的，这个类的实例是Spring通过解析`<dubbo:service />`标签创建的单例Bean，每一个`<dubbo:service />`都会创建一个`ServiceBean`实例。先看一下`ServiceBean`类的继承类图以及`Spring`中`Bean`的生命周期流程图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/ServiceBean%E7%BB%A7%E6%89%BF%E7%B1%BB%E5%9B%BE.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

以下是`ServiceBean`类的源码:

```java
/** ServiceBean 类型参数T--暴露服务接口 */
public class ServiceBean<T> extends ServiceConfig<T> implements ApplicationContextAware, 			BeanNameAware, ApplicationListener<ContextRefreshedEvent>,
           InitializingBean, DisposableBean {

    private static final long serialVersionUID = 213195494150089726L;

    private static final Logger logger = LoggerFactory.getLogger(ServiceBean.class);

    /** Spring应用上下文 */
    private static transient ApplicationContext SPRING_CONTEXT;

    /** Spring应用上下文 */
    private transient ApplicationContext applicationContext;

    /** 组件名称 */
    private transient String beanName;

    /** 支持应用监听器标识 */
    private transient boolean supportedApplicationListener;

    public ServiceBean() {
        super();
    }

    public ServiceBean(Service service) {
        super(service);
    }

    /** 返回Spring容器应用上下文。 */
    public static ApplicationContext getSpringContext() {
        return SPRING_CONTEXT;
    }

    /** ApplicationContextAware接口的方法，在InitializingBean.afterPropertiesSet()之前调用 */
    // 流程：
 	// 1 将applicationContext设置到SpringExtensionFactory中,用于后续从SpringExtensionFactory中获取Bean
	// 2 获取方法addApplicationListener(ApplicationListener<?> listener),之后将当前类实例（因为当前类监听了ContextRefreshedEvent事件）加入spring的监听器列表           
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        // 添加应用上下文到SpringExtensionFactory，用于后续SPI IoC注入。
        SpringExtensionFactory.addApplicationContext(applicationContext);
        if (applicationContext != null) {
            SPRING_CONTEXT = applicationContext;
            try {
                // 兼容Spring2.0.1
                Method method = applicationContext.getClass().getMethod("addApplicationListener", ApplicationListener.class);
                // 添加自己为监听器
                method.invoke(applicationContext, this);
                supportedApplicationListener = true;
            } catch (Throwable t) {
                if (applicationContext instanceof AbstractApplicationContext) {
                    try {
                        // 兼容Spring2.0.1
                        Method method = AbstractApplicationContext.class.getDeclaredMethod("addListener", ApplicationListener.class);
                        if (!method.isAccessible()) {
                            method.setAccessible(true);
                        }
                        method.invoke(applicationContext, this);
                        supportedApplicationListener = true;
                    } catch (Throwable ignored) {
                    }
                }
            }
        }
    }

    /** BeanNameAware接口的方法，在ApplicationContextAware.setApplicationContext(ApplicationContext)之前调用 */
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    /** ApplicationListener接口的方法，监听Spring上下文刷新完成事件。 */
    // delay没有设置或者是-1 && 服务没有暴露 && 服务没有取消暴露，则进行服务暴露          
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // -1表示延迟到Spring容器初始化完成时暴露服务
        if (isDelay() && !isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            export();
        }
    }

    // 判断是否延迟暴露服务
    private boolean isDelay() {
        Integer delay = getDelay();
        ProviderConfig provider = getProvider();
        if (delay == null && provider != null) {
            delay = provider.getDelay();
        }
        return supportedApplicationListener && (delay == null || delay == -1);
    }

    /** InitializingBean接口的方法 */
	// 流程:
	// 1 检查ServiceBean的ProviderConfig provider，如果为空，从applicationContext获取	ProviderConfig类型的bean（这里查找的过程其实就是看有没有配置<dubbo:provider />），如果获取到了，进行设置
	// 2 后续会参照1分别进行: 
	// -- ApplicationConfig application
	// -- ModuleConfig module
	// -- List<RegistryConfig> registries
	// -- MonitorConfig monitor
	// -- List<ProtocolConfig> protocols
	// -- String path：服务名称
	// 3 判断延迟的时间是否大于0，如果是，执行export()，进行服务暴露，如果不是，结束（这种情况下服务暴露，会发生在发布上下文刷新完成事件的时候）
    @Override
    @SuppressWarnings({"unchecked", "deprecation"})
    public void afterPropertiesSet() throws Exception {
        // 当前ServiceBean没有提供者配置 <dubbo:provider .../>
        if (getProvider() == null) {
            // 服务提供者缺省值配置
            Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
            // 存在ProviderConfig
            if (providerConfigMap != null && providerConfigMap.size() > 0) {
                // 获取服务提供者 协议配置
                Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
                // 不存在ProtocolConfig且存在的ProviderConfig个数大于1
                if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                        && providerConfigMap.size() > 1) { // 兼容旧版本(忽略这条if逻辑)
                    List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                    for (ProviderConfig config : providerConfigMap.values()) {
                        // 配置了default参数，且为true
                        if (config.isDefault() != null && config.isDefault()) {
                            providerConfigs.add(config);
                        }
                    }
                    if (providerConfigs.size() > 0) {
                        // 设置协议
                        setProviders(providerConfigs);
                    }
                } else { // 直接看这里!!
                    // 设置提供者配置
                    ProviderConfig providerConfig = null;
                    // 存在ProviderConfig，1.存在ProtocolConfig 或2.不存在ProtocolConfig，但存在ProviderConfig个数为1
                    for (ProviderConfig config : providerConfigMap.values()) {
                        if (config.isDefault() == null || config.isDefault()) {
                            if (providerConfig != null) {
                                // 提供者配置不能重复
                                throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                            }
                            providerConfig = config;
                        }
                    }
                    if (providerConfig != null) {
                        setProvider(providerConfig);
                    }
                }
            }
        }
        // 设置ApplicationConfig
        if (getApplication() == null
                && (getProvider() == null || getProvider().getApplication() == null)) {
            // 应用信息配置
            Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null :
                    BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
            if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
                ApplicationConfig applicationConfig = null;
                for (ApplicationConfig config : applicationConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault()) {
                        if (applicationConfig != null) {
                            // 默认的ApplicationConfig只能有一个
                            throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                        }
                        applicationConfig = config;
                    }
                }
                if (applicationConfig != null) {
                    setApplication(applicationConfig);
                }
            }
        }
        // 设置ModuleConfig
        if (getModule() == null
                && (getProvider() == null || getProvider().getModule() == null)) {
            Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null :
                    BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
            if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
                ModuleConfig moduleConfig = null;
                for (ModuleConfig config : moduleConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault()) {
                        // 默认的ModuleConfig只能有一个
                        if (moduleConfig != null) {
                            throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                        }
                        moduleConfig = config;
                    }
                }
                if (moduleConfig != null) {
                    setModule(moduleConfig);
                }
            }
        }
        // 设置注册中心配置RegistryConfigs
        if ((getRegistries() == null || getRegistries().size() == 0)
                && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().size() == 0)
                && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().size() == 0)) {
            // 注册中心配置
            Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null :
                    BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
            if (registryConfigMap != null && registryConfigMap.size() > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                for (RegistryConfig config : registryConfigMap.values()) {
                    // default未配置或default配置为true
                    if (config.isDefault() == null || config.isDefault()) {
                        registryConfigs.add(config);
                    }
                }
                if (registryConfigs.size() > 0) {
                    super.setRegistries(registryConfigs);
                }
            }
        }
        // 设置ProtocolConfigs
        if ((getProtocols() == null || getProtocols().size() == 0)
                && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().size() == 0)) {
            // 服务提供者协议配置
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null :
                    BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
                List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
                for (ProtocolConfig config : protocolConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault()) {
                        protocolConfigs.add(config);
                    }
                }
                if (protocolConfigs.size() > 0) {
                    super.setProtocols(protocolConfigs);
                }
            }
        }
        // 设置服务名称path
        if (getPath() == null || getPath().length() == 0) {
            if (beanName != null && beanName.length() > 0
                    // 接口类型
                    && getInterface() != null && getInterface().length() > 0
                    && beanName.startsWith(getInterface())) {
                setPath(beanName);
            }
        }
        // 设置的延迟时间大于0
        if (!isDelay()) {
            // 在BeanFactory设置了所有提供的bean属性之后调用它
            // Bean所有属性都设置完成后才暴露服务
            export();
        }
    }

    /** DisposableBean接口的方法 */
    @Override
    public void destroy() throws Exception {
        // 取消暴露服务
        unexport();
    }

}
```

这里最重要的两个方法是`afterPropertiesSet()`和`onApplicationEvent(ContextRefreshedEvent event)`。

### 一、检查、设置属性与是否暴露服务

当所有的Bean的属性被设置好之后，执行`afterPropertiesSet()`。该方法的流程：

##### 1.检查并设置属性

检查`ServiceBean`的某个属性（这里的属性包含如下6个）是否为空，如果为空，从`applicationContext`获取相应类型的bean，如果获取到了，则进行相应的设置。

- `ProviderConfig provider`：其实就是看有没有配置`<dubbo:provider>`
- `ApplicationConfig application`：其实就是看有没有配置`<dubbo:application>`
- `ModuleConfig module`：其实就是看有没有配置`<dubbo:module>`
- `List<RegistryConfig> registries`：其实就是看有没有配置`<dubbo:registry>`
- `MonitorConfig monitor`：其实就是看有没有配置`<dubbo:monitor>`
- `List<ProtocolConfig> protocols`：其实就是看有没有配置`<dubbo:protocol>`
- `String path`：服务名称

##### 2.是否暴露服务

之后判断延迟的时间`delay`是否大于0，如果是，执行`export()`，进行服务暴露，如果不是，结束（这种情况下服务暴露会发生在容器发布上下文刷新完成事件的时候）。在这里，我们并没有指定`delay`，所以`delay=null`，服务暴露会发生在容器发布上下文刷新完成事件的时候。

***

当`afterPropertiesSet()`结束之后，来看一下此时的`ServiceBean`实例，实例的私有属性如下(没有值的暂时不说):

```java
id = com.alibaba.dubbo.demo.DemoService
applicationContext = ClassPathXmlApplicationContext实例
beanName = com.alibaba.dubbo.demo.DemoService
interfaceName = com.alibaba.dubbo.demo.DemoService
supportedApplicationListener = true
ref = DemoServiceImpl2实例
path = com.alibaba.dubbo.demo.DemoService

application:
-- id = demo-provider
-- name = demo-provider

registries = [
    RegistryConfig:
    -- id = com.alibaba.dubbo.config.RegistryConfig
    -- protocol = zookeeper
    -- address = localhost:2181
    -- client = zkclient
    -- group = dubbo_test
]

protocols = [ 
    ProtocolConfig: 
    -- id = dubbo
    -- name = dubbo
    -- port = 20881    
]
```

实际上在创建`ServiceBean`实例的时候，也会初始化其父类`ServiceConfig`的静态属性：

```java
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```

其中`protocol`实例是：`Protocol$Adaptive`实例，`Protocol$Adaptive`类的代码在[Dubbo-Compiler接口分析](https://xuanjian1992.top/2019/01/13/Dubbo-Compiler%E6%8E%A5%E5%8F%A3%E5%88%86%E6%9E%90/)这篇文章中已经列出。

下边来看一下第二句代码的源码。首先，看一下`ProxyFactory`的定义：

```java
/** 动态代理扩展，将Invoker接口转换成业务接口。 */
@SPI("javassist")
public interface ProxyFactory {

    // 创建服务代理实例
    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    // 创建 invoker
    // @param <T> 泛型
    // @param proxy 对外提供服务的实现类实例
    // @param type 服务接口Class
    // @param url URL
    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```

`ExtensionLoader.getExtensionLoader(ProxyFactory.class)`的实现结果还是：

`ExtensionLoader<com.alibaba.dubbo.rpc.ProxyFactory> loader`，最终的`loader`包含如下属性：

- `Class<?> type = interface com.alibaba.dubbo.rpc.ProxyFactory`
- `ExtensionFactory objectFactory = AdaptiveExtensionFactory`（适配类）
  - `factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]`

之后，执行`getAdaptiveExtension()`方法。

来看一下: `META-INF/dubbo/internal/com.alibaba.dubbo.rpc.ProxyFactory`的内容

```java
stub=com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
jdk=com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory
javassist=com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory
```

从`ProxyFactory的@SPI("javassist")`，默认选用的实现是`com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory`。`com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper`是一个`wrapper`类，但是`wrapper`类只在`getExtension("xxx")`中会实现`aop`，而在`getAdaptiveExtension()`不会进行`aop`包裹。

这里的三个实现类没有一个类上带有`@Adaptive`注解，所以会动态创建类。动态生成的类`ProxyFactory$Adaptive`代码(已格式化)如下：

```java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ProxyFactory$Adaptive implements com.alibaba.dubbo.rpc.ProxyFactory {
    public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg2 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getInvoker(arg0, arg1, arg2);
    }
    
    public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) 
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if(extName == null) 
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0);
    }
}
```

所以`ServiceConfig`中的静态属性`proxyFactory`为`ProxyFactory$Adaptive`实例。

至此，一个`ServiceBean`实例就创建完成了。

### 二、 在上下文刷新完成时进行服务暴露

```java
/** ApplicationListener接口的方法，监听Spring上下文刷新完成事件。 */
// delay没有设置或者是-1 && 服务没有暴露 && 服务没有取消暴露，则进行服务暴露          
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    // -1表示延迟到Spring容器初始化完成时暴露服务
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}

// 判断是否延迟暴露服务
private boolean isDelay() {
    Integer delay = getDelay();
    ProviderConfig provider = getProvider();
    if (delay == null && provider != null) {
        delay = provider.getDelay();
    }
    return supportedApplicationListener && (delay == null || delay == -1);
}
```

一切准备好之后，就在这里开始进行服务暴露，调用`export()`方法。