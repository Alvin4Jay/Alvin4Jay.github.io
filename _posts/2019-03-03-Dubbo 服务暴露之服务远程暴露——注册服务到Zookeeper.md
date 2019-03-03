---
layout:     post
title:      Dubbo 服务暴露之服务远程暴露——注册服务到Zookeeper
subtitle:   Register Providers to Zookeeper
date:       2019-03-03
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo 服务暴露之服务远程暴露——注册服务到Zookeeper

服务远程暴露的总体步骤为

- 将`ref`封装为`Invoker`
- 将`Invoker`转换为`Exporter`
- 启动`Netty`服务端
- 注册服务到`Zookeeper`
- 订阅与通知机制
- 返回新的`Exporter`实例

前篇文章中已经分析了前三步的过程，下面分析第四步——注册服务到`Zookeeper`的过程。总体代码如下`RegistryProtocol.export(final Invoker<T> originInvoker)`:

```java
// 获取注册中心url: zookeeper://....
URL registryUrl = getRegistryUrl(originInvoker); // ①

// 连接Zookeeper注册中心，获取注册中心实例(ZookeeperRegistry)
final Registry registry = getRegistry(originInvoker); // ②
// 注册到注册中心的提供者url dubbo://....
final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker); // ③

// 是否注册服务
boolean register = registeredProviderUrl.getParameter("register", true);
// 缓存提供者、注册中心等信息
ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl,  registeredProviderUrl);

if (register) {
    // 注册服务提供者
    register(registryUrl, registeredProviderUrl); // ④
    ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
}
```

说明:

- ①: 获取注册中心`url`；
- ②: 连接`Zookeeper`注册中心，获取注册中心实例(`ZookeeperRegistry`)；
- ③: 获取注册到注册中心的提供者`url`；
- ④: 注册服务到`Zookeeper`。

### 一、获取注册中心url

```java
URL registryUrl = getRegistryUrl(originInvoker); // 获取注册中心url: zookeeper://....

// 获取注册中心url
// @param originInvoker DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
// @return 获取注册中心url
private URL getRegistryUrl(Invoker<?> originInvoker) {
    // registry://...
    URL registryUrl = originInvoker.getUrl();
    if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
        // protocol: zookeeper
        String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
        // 设置协议为zookeeper，移除registry参数
        registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
    }
    // zookeeper://...
    return registryUrl;
}
```

### 二、创建ZookeeperRegistry实例

#####1.RegistryProtocol.getRegistry(final Invoker<?> originInvoker)

```java
// 连接Zookeeper注册中心，获取注册中心实例(ZookeeperRegistry)
final Registry registry = getRegistry(originInvoker);

// 根据invoker url获取registry实例(ZookeeperRegistry)
// @param originInvoker 原始的服务提供者执行体 DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
// @return ZookeeperRegistry实例
private Registry getRegistry(final Invoker<?> originInvoker) {
    // 注册中心url: zookeeper://...
    URL registryUrl = getRegistryUrl(originInvoker);
    // ZookeeperRegistryFactory.getRegistry(), ZookeeperRegistryFactory继承于AbstractRegistryFactory
    return registryFactory.getRegistry(registryUrl);
}

// getRegistryUrl(Invoker<?> originInvoker)方法同上
```

首先对`originInvoker`中的`url`(注册中心`url`)进行处理：

- 将协议换成`zookeeper`
- 去掉`registry=zookeeper`的参数

看一下`originInvoker`的`url`(注册中心`url`)：（解码后的）

> **registry**://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&**export=dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=68041&side=provider&timestamp=1550913627603**&group=dubbo_test&pid=68041&**registry=zookeeper**&timestamp=1550913627590

说明：

- 第一个加粗部分代表协议：`registry`
- 第二个加粗部分是`export`参数
- 第三个加粗部分是`registry=zookeeper`

经过处理之后的`registryUrl`为：

> **zookeeper**://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&**export=dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=172.16.132.166&bind.port=20881&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=68041&side=provider&timestamp=1550913627603**&group=dubbo_test&pid=68041&timestamp=1550913627590	

之后使用注册中心工厂`RegistryFactory`来创建注册中心。

#####2.RegistryFactory$Adaptive.getRegistry(com.alibaba.dubbo.common.URL registryUrl)

```java
public class RegistryFactory$Adaptive implements com.alibaba.dubbo.registry.RegistryFactory {
    public com.alibaba.dubbo.registry.Registry getRegistry(com.alibaba.dubbo.common.URL arg0) {
        if (arg0 == null)
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg0;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );//zookeeper
        if(extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.registry.RegistryFactory extension = (com.alibaba.dubbo.registry.RegistryFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.registry.RegistryFactory.class).getExtension(extName);
        return extension.getRegistry(arg0);
    }
}
```

这里获取到的`extension`是`ZookeeperRegistryFactory`，之后使用`ZookeeperRegistryFactory`进行`Registry`实例的创建。首先看一下`ZookeeperRegistryFactory`的继承图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/ZookeeperRegistryFactory%E7%BB%A7%E6%89%BF%E5%9B%BE.png?x-oss-process=style/markdown-pic)

`getRegistry`方法在`ZookeeperRegistryFactory`的父类`AbstractRegistryFactory`中。

#####3.AbstractRegistryFactory.getRegistry(URL registryUrl)

```java
@Override
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    // 注册中心url字符串表示
    String key = url.toServiceString();
    // 锁定注册中心获取过程，保证注册中心单一实例
    LOCK.lock();
    try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        // 创建ZookeeperRegistry实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        // 释放锁
        LOCK.unlock();
    }
}
```

流程：

- 先处理`url`，之后获取`Registry`的`key`，然后根据该`key`从`Map<String, Registry> REGISTRIES`注册中心集合缓存中获取`Registry`，如果有，直接返回，如果没有，创建`Registry`，之后存入缓存，最后返回。

首先处理传入的`registryUrl`：

- 设置：`path=com.alibaba.dubbo.registry.RegistryService`
- 添加参数：`interface=com.alibaba.dubbo.registry.RegistryService`
- 去除`export、refer`参数

最终得到的`registryUrl`如下：

```java
zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&interface=com.alibaba.dubbo.registry.RegistryService&pid=69353&timestamp=1550921731982
```

之后根据上述的`registryUrl`创建`Registry`的`key`，该`{ key : Registry }`最终会被存储在`Map<String, Registry> REGISTRIES`注册中心缓存集合中(该属性是`ZookeeperRegistryFactory`父类`AbstractRegistryFactory`的一个属性)。

**根据registryUrl创建Registry的key：url.toServiceString()**

```java
public String toServiceString() {
    return buildString(true, false, true, true);
}

private String buildString(boolean appendUser, boolean appendParameter, boolean useIP, boolean useService, String... parameters) {
    // protocol://username:password@host:port/group/interface{path}:version
    StringBuilder buf = new StringBuilder();
    if (protocol != null && protocol.length() > 0) {
        buf.append(protocol);
        buf.append("://");
    }
    if (appendUser && username != null && username.length() > 0) {  
        buf.append(username);
        if (password != null && password.length() > 0) {
            buf.append(":");
            buf.append(password);
        }
        buf.append("@");
    }
    String host;
    if (useIP) {
        host = getIp();
    } else {
        host = getHost();
    }
    if (host != null && host.length() > 0) {
        buf.append(host);
        if (port > 0) {
            buf.append(":");
            buf.append(port);
        }
    }
    String path;
    if (useService) {
        path = getServiceKey();
    } else {
        path = getPath();
    }
    if (path != null && path.length() > 0) {
        buf.append("/");
        buf.append(path);
    }
    if (appendParameter) {
        buildParameters(buf, true, parameters);
    }
    return buf.toString();
}

public String getServiceKey() {
    // 先获取interface参数，如果没有的话，取path的值，这里都是 
    // com.alibaba.dubbo.registry.RegistryService
    String inf = getServiceInterface();
    if (inf == null) return null;
    StringBuilder buf = new StringBuilder();
    String group = getParameter(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        buf.append(group).append("/"); // group
    }
    buf.append(inf);
    String version = getParameter(Constants.VERSION_KEY);
    if (version != null && version.length() > 0) {
        buf.append(":").append(version); // version
    } 
    return buf.toString();
}
```
最终得到的是这样的形式：`protocol://username:password@host:port/group/interface{path}:version`。这里`key`为`zookeeper://127.0.0.1:2181/dubbo_test/com.alibaba.dubbo.registry.RegistryService`。

之后来到真正创建`Registry`的地方。

```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    // 连接zookeeper注册中心
    // @param url 注册中心url zookeeper://...
    // @return ZookeeperRegistry实例
    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

这里的`ZookeeperTransporter`对象是一个`com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter$Adaptive`对象，是在加载`ZookeeperRegistryFactory`扩展实例的时候，**通过AOP设置**进来的。

在创建`ZookeeperRegistry`之前来看一下它的继承图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/ZookeeperRegistry%E7%B1%BB%E7%BB%A7%E6%89%BF%E5%9B%BE.png?x-oss-process=style/markdown-pic)

**new ZookeeperRegistry(registryUrl, ZookeeperTransporter$Adaptive对象)**

```java
private final static int DEFAULT_ZOOKEEPER_PORT = 2181;
private final static String DEFAULT_ROOT = "dubbo";
// ZooKeeper的根节点
private final String root;
private final Set<String> anyServices = new ConcurrentHashSet<String>();
//<url订阅条件，<通知监听器，子节点变更监听>> 
private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();
// ZooKeeper客户端
private final ZookeeperClient zkClient;

// @param url 注册中心url zookeeper://....
// @param zookeeperTransporter 用于连接zk ZookeeperTransporter$Adaptive实例
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    // zk根节点
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT); // dubbo_test
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group; // /dubbo_test
    // 在这里连接zk，创建zk客户端，启动会话
    zkClient = zookeeperTransporter.connect(url);
    // 添加zk连接状态变化监听器，
    // 监听重新连接成功事件，重新连接成功后，之前已经完成注册和订阅的url
    // 要重新进行注册和订阅(因为临时节点可能已经没了)。
    zkClient.addStateListener(new StateListener() {
        @Override
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    // 恢复注册与订阅
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

**new FailbackRegistry(registryUrl)**

```java
	// 重试的定时任务执行器
    private final ScheduledExecutorService retryExecutor =  Executors.newScheduledThreadPool(1, new NamedThreadFactory("DubboRegistryFailedRetryTimer", true));
    // 失败重试定时器定时检查是否有请求失败，如有，无限次重试
    // retryFuture代表定时重试的结果
    private final ScheduledFuture<?> retryFuture;
    private final Set<URL> failedRegistered = new ConcurrentHashSet<URL>();
    private final Set<URL> failedUnregistered = new ConcurrentHashSet<URL>();
    private final ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
    private final ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
    private final ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified = new ConcurrentHashMap<URL, Map<NotifyListener, List<URL>>>();

public FailbackRegistry(URL url) {
    super(url);
    // 重试周期
    int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
    this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            // 检测并连接注册中心
            try {
                // 重试失败的动作，如注册、取消注册、订阅、取消订阅、通知。
                retry();
            } catch (Throwable t) { // 防御性容错
                logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}
```

**new AbstractRegistry(registryUrl)**

```java
// URL地址分隔符，用于文件缓存中，服务提供者URL分隔
private static final char URL_SEPARATOR = ' ';
// URL地址分隔正则表达式，用于解析文件缓存中服务提供者URL列表
private static final String URL_SPLIT = "\\s+";
// 本地磁盘缓存，其中特殊的key值.registies记录注册中心列表，其它均为notified服务提供者列表
private final Properties properties = new Properties();
// 文件缓存定时写入
private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));
//是否是同步保存文件
private final boolean syncSaveFile;
private final AtomicLong lastCacheChanged = new AtomicLong();
// 已经注册的url集合
private final Set<URL> registered = new ConcurrentHashSet<URL>();
// 已经订阅的url，对应监听器
private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
// 已经通知的数据<订阅的url，<category, 匹配的变更数据>>
private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();
// 注册中心配置url
private URL registryUrl;
// 本地磁盘缓存文件
private File file;

// @param url 注册中心地址 zookeeper://..
public AbstractRegistry(URL url) {
    setUrl(url);
    // Start file save timer 启动文件保存定时器
    syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
    // 文件名
    String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
    File file = null;
    if (ConfigUtils.isNotEmpty(filename)) {
        file = new File(filename);
        if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
            if (!file.getParentFile().mkdirs()) { // 创建父目录
                throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
            }
        }
    }
    this.file = file;
    // 加载本地缓存文件
    loadProperties();
    // url.getBackupUrls(): 注册中心url集合。
    // notify方法作用：通知变更数据
    notify(url.getBackupUrls());
}
```

先总结一下：父子三代分别做的事情：

- `AbstractRegistry`主要用来维护**缓存文件**。
- `FailbackRegistry`主要用来做**失败重试操作**（包括：注册失败／反注册失败／订阅失败／反订阅失败／通知失败的重试）；也提供了供`ZookeeperRegistry`使用的zk重连后的恢复工作的方法。
- `ZookeeperRegistry`**创建zk客户端，启动会话**；并且调用`FailbackRegistry`**实现zk重连后的恢复工作**。

先看`AbstractRegistry`:

- 设置属性`registryUrl=zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&interface=com.alibaba.dubbo.registry.RegistryService&pid=69353&timestamp=1550921731982`

- 创建文件`/Users/Jay/.dubbo/dubbo-registry-demo-provider-localhost:2181.cache`的文件夹`/Users/Jay/.dubbo`

- 设置属性`file：/Users/Jay/.dubbo/dubbo-registry-demo-provider-localhost:2181.cache`文件，该文件存储信息将是这样的：

  ```
  com.alibaba.dubbo.demo.DemoService=empty\://172.16.132.166\:20881/com.alibaba.dubbo.demo.DemoService?anyhost\=true&application\=demo-provider&category\=configurators&check\=false&dubbo\=2.0.0&generic\=false&interface\=com.alibaba.dubbo.demo.DemoService&methods\=sayHello&pid\=5259&side\=provider&timestamp\=1507294508053
  ```

  

- 如果`file`存在，将`file`中的内容写入`properties`属性；既然有读`file`，那么是什么时候写入`file`的呢？`AbstractRegistry`创建了一个含有一个名字为`DubboSaveRegistryCache`的后台线程的`FixedThreadPool`，只在`notify(URL url, NotifyListener listener, List<URL> urls)`方法中会被调用，此处由于`ConcurrentMap<URL, Set<NotifyListener>> subscribed`为空，所以`AbstractRegistry(URL url)`中的`notify(url.getBackupUrls())`不会执行，此处也不会创建文件。

- 最后是`notify(url.getBackupUrls())`。

再来看`FailbackRegistry`:

只做了一件事，利用一个含有一个名为`DubboRegistryFailedRetryTimer`的后台线程的`ScheduledThreadPool`创建并启动了定时任务，该任务5s后开始第一次执行`retry()`，之后每隔5s执行一次。来看一下`retry()`:

```java
// 重试失败的动作，如注册、反注册、订阅、取消反订阅、通知。
// 将所有注册失败的url（failedRegistered中的url）进行注册，之后从failedRegistered进行移除；
// 将所有反注册失败的url（failedUnregistered中的url）进行反注册，之后从failedUnregistered进行移除；
// 将所有订阅失败的url（failedSubscribed中的url）进行重新订阅，之后从failedSubscribed进行移除；
// 将所有反订阅失败的url（failedUnsubscribed中的url）进行反订阅，之后从failedUnsubscribed进行移除；
// 将所有通知失败的url（failedNotified中的url）进行通知，之后从failedNotified进行移除；
protected void retry() {
    // 注册失败的
    if (!failedRegistered.isEmpty()) {
        Set<URL> failed = new HashSet<URL>(failedRegistered);
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry register " + failed);
            }
            try {
                for (URL url : failed) {
                    try {
                        // 注册数据的实际调用方法
                        doRegister(url);
                        failedRegistered.remove(url);
                    } catch (Throwable t) { // 忽略所有异常，等待下次重试
                        logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                    }
                }
            } catch (Throwable t) { // 忽略所有异常，等待下次重试
                logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 取消注册失败的
    if (!failedUnregistered.isEmpty()) {
        Set<URL> failed = new HashSet<URL>(failedUnregistered);
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry unregister " + failed);
            }
            try {
                for (URL url : failed) {
                    try {
                        // 取消注册的实际调用函数
                        doUnregister(url);
                        failedUnregistered.remove(url);
                    } catch (Throwable t) { // 忽略所有异常，等待下次重试
                        logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                    }
                }
            } catch (Throwable t) { // 忽略所有异常，等待下次重试
                logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 订阅失败的
    if (!failedSubscribed.isEmpty()) {
        Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedSubscribed);
        for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry subscribe " + failed);
            }
            try {
                for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                    URL url = entry.getKey();
                    Set<NotifyListener> listeners = entry.getValue();
                    for (NotifyListener listener : listeners) {
                        try {
                            // listener需要一个一个订阅，每订阅一个，就将该listener从当前的url监听器列表中移除
                            doSubscribe(url, listener);
                            listeners.remove(listener);
                        } catch (Throwable t) { // 忽略所有异常，等待下次重试
                            logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) { // 忽略所有异常，等待下次重试
                logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 取消订阅失败的
    if (!failedUnsubscribed.isEmpty()) {
        Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedUnsubscribed);
        for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry unsubscribe " + failed);
            }
            try {
                for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                    URL url = entry.getKey();
                    Set<NotifyListener> listeners = entry.getValue();
                    for (NotifyListener listener : listeners) {
                        try {
                            // listener需要一个一个反订阅，每反订阅一个，就将该listener从当前的url监听器列表中移除
                            doUnsubscribe(url, listener);
                            listeners.remove(listener);
                        } catch (Throwable t) { // 忽略所有异常，等待下次重试
                            logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) { // 忽略所有异常，等待下次重试
                logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
    // 通知失败的
    if (!failedNotified.isEmpty()) {
        Map<URL, Map<NotifyListener, List<URL>>> failed = new HashMap<URL, Map<NotifyListener, List<URL>>>(failedNotified);
        for (Map.Entry<URL, Map<NotifyListener, List<URL>>> entry : new HashMap<URL, Map<NotifyListener, List<URL>>>(failed).entrySet()) {
            if (entry.getValue() == null || entry.getValue().size() == 0) {
                failed.remove(entry.getKey());
            }
        }
        if (failed.size() > 0) {
            if (logger.isInfoEnabled()) {
                logger.info("Retry notify " + failed);
            }
            try {
                for (Map<NotifyListener, List<URL>> values : failed.values()) {
                    for (Map.Entry<NotifyListener, List<URL>> entry : values.entrySet()) {
                        try {
                            NotifyListener listener = entry.getKey();
                            List<URL> urls = entry.getValue();
                            listener.notify(urls);
                            values.remove(listener);
                        } catch (Throwable t) { // 忽略所有异常，等待下次重试
                            logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                }
            } catch (Throwable t) { // 忽略所有异常，等待下次重试
                logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
            }
        }
    }
}
```

最后回到`ZookeeperRegistry`:

首先为属性设值`root=/dubbo_test`，之后创建zk客户端，启动会话，最后创建了一个`StateListener`监听器，监听zk重新连接成功事件，重新连接成功后，之前已经完成注册和订阅的`url`要重新进行注册和订阅(因为临时节点可能已经没了)。

下面看创建zk客户端，启动会话的代码，这是此处最核心的部分:

**ZookeeperTransporter$Adaptive.connect(com.alibaba.dubbo.common.URL registryUrl)**

```java
public com.alibaba.dubbo.remoting.zookeeper.ZookeeperClient connect(com.alibaba.dubbo.common.URL arg0) {
    if (arg0 == null)
        throw new IllegalArgumentException("url == null");
    com.alibaba.dubbo.common.URL url = arg0;
    String extName = url.getParameter("client", url.getParameter("transporter", "zkclient"));// zkclient
    if(extName == null)
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter) name from url(" + url.toString() + ") use keys([client, transporter])");
    com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter extension = (com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.remoting.zookeeper.ZookeeperTransporter.class).getExtension(extName);
    return extension.connect(arg0);
}
```

这里创建的`extension`是`ZkclientZookeeperTransporter`实例。

```java
public class ZkclientZookeeperTransporter implements ZookeeperTransporter {

    public final static String NAME = "zkclient";

    // 使用Zkclient连接至zookeeper，返回zkclient zk客户端
    // @param url 注册中心url zookeeper://....
    // @return ZkclientZookeeperClient实例
    @Override
    public ZookeeperClient connect(URL url) {
        return new ZkclientZookeeperClient(url);
    }

}
```

**new ZkclientZookeeperClient(registryUrl)**

```java
private final ZkClientWrapper client; // ZkClient的包装
private volatile KeeperState state = KeeperState.SyncConnected;  // zk连接状态

// 使用ZkClient连接至zookeeper
// @param url 注册中心url zookeeper://...
ZkclientZookeeperClient(URL url) {
    super(url);
    // 创建连接zk的任务ListenableFutureTask，返回ZkClientWrapper实例
    client = new ZkClientWrapper(url.getBackupAddress(), 30000);
    // 给创建的ListenableFutureTask添加监听器，任务完成即在使用ZkClient连接zookeeper之后，添加zk连接状态变更监听器，监听连接断开／连接成功／重新连接成功事件
    // （实际上这里只有重新连接成功事件会被处理，而处理器实际上就是ZookeeperRegistry构造器中的那个执行recover()的StateListener）
    client.addListener(new IZkStateListener() {
        @Override
        public void handleStateChanged(KeeperState state) throws Exception {
            ZkclientZookeeperClient.this.state = state;

            if (state == KeeperState.Disconnected) {
                // 回调
                stateChanged(StateListener.DISCONNECTED);
            } else if (state == KeeperState.SyncConnected) {
                stateChanged(StateListener.CONNECTED);
            }
        }

        @Override
        public void handleNewSession() throws Exception {
            stateChanged(StateListener.RECONNECTED);
        }
    });
    client.start(); // 启动连接zk的任务，返回ZkClient客户端实例
}

// zk连接状态变化时的回调
protected void stateChanged(int state) {
    for (StateListener sessionListener : getSessionListeners()) {
        // 此处的实现类，只有ZookeeperRegistry构造器中的那个StateListener
        sessionListener.stateChanged(state); 
    }
}
```

**父类`AbstractZookeeperClient`**

```java
// 注册中心url zookeeper://...
private final URL url;
// 状态监听器
private final Set<StateListener> stateListeners = new CopyOnWriteArraySet<StateListener>();
//<Path, <ChildListener, TargetChildListener>>
private final ConcurrentMap<String, ConcurrentMap<ChildListener, TargetChildListener>> childListeners = new ConcurrentHashMap<String, ConcurrentMap<ChildListener, TargetChildListener>>();
// 是否已关闭
private volatile boolean closed = false;

public AbstractZookeeperClient(URL url) {
    this.url = url; // 记录注册中心url
}
```

说明：

- 设置属性`url=registryUrl：zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&interface=com.alibaba.dubbo.registry.RegistryService&pid=69353&timestamp=1550921731982`
- 创建了一个`Set<StateListener> stateListeners`，`ZookeeperRegistry`构造器中的那个执行`recover()`的`StateListener`就将会放在这里。

**ZkClientWrapper类——ZkClient包装**

```java
// 构造器——new ZkClientWrapper(url.getBackupAddress(), 30000)
// 创建连接zk的任务ListenableFutureTask
// @param serverAddr zk集群地址
// @param timeout 连接超时设置
public ZkClientWrapper(final String serverAddr, long timeout) {
    this.timeout = timeout; // 连接超时设置
    // 创建连接zk的任务
    listenableFutureTask = ListenableFutureTask.create(new Callable<ZkClient>() {
        @Override
        public ZkClient call() throws Exception {
            // 连接zk
            return new ZkClient(serverAddr, Integer.MAX_VALUE);
        }
    });
}

/**
 * 给创建的ListenableFutureTask添加监听器，任务完成即在ZkClient连接zookeeper之后，添加zk连接状态监听器
 * @param listener zk连接状态监听器
 */
public void addListener(final IZkStateListener listener) {
    listenableFutureTask.addListener(new Runnable() {
        @Override
        public void run() {
            try {
                client = listenableFutureTask.get(); // 获取任务执行结果，即ZkClient实例
                client.subscribeStateChanges(listener); // 添加zk连接状态变化监听器
            } catch (InterruptedException e) {
                logger.warn(Thread.currentThread().getName() + " was interrupted unexpectedly, which may cause unpredictable exception!");
            } catch (ExecutionException e) {
                logger.error("Got an exception when trying to create zkclient instance, can not connect to zookeeper server, please check!", e);
            }
        }
    });
}

// 连接zk，返回ZkClient实例
public void start() {
    if (!started) {
        // 另起线程
        Thread connectThread = new Thread(listenableFutureTask);
        connectThread.setName("DubboZkclientConnector");
        connectThread.setDaemon(true);
        connectThread.start();
        try {
            // 获取执行结果
            client = listenableFutureTask.get(timeout, TimeUnit.MILLISECONDS);
        } catch (Throwable t) {
            // 超时，抛出超时异常
            logger.error("Timeout! zookeeper server can not be connected in : " + timeout + "ms!", t);
        }
        started = true;
    } else {
        logger.warn("Zkclient has already been started!");
    }
}
```

`ZkclientZookeeperClient`构造函数中首先调用父类`AbstractZookeeperClient`的构造器，记录注册中心`url`；然后创建连接zk的任务，返回`ZkClientWrapper`实例；之后给创建的连接zk的`ListenableFutureTask`任务添加监听器，任务完成即在使用`ZkClient`连接`zookeeper`之后，添加zk连接状态变更监听器（监听连接断开／连接成功／重新连接成功事件，实际上这里只有重新连接成功事件会被处理，而处理器实际上就是`ZookeeperRegistry`构造器中的那个执行`recover()`的`StateListener`）；最后启动连接zk的任务。

至此，一个完整的`ZookeeperRegistry`实例就创建完成了，下面看一下属性：

- ```markdown
  - ZookeeperClient  zkClient= ZkclientZookeeperClient实例
    - client：ZkClientWrapper实例
      - ZkClient实例
      - timeout=30000
    - String url：zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&interface=com.alibaba.dubbo.registry.RegistryService&pid=69353&timestamp=1550921731982
    - Set<StateListener> stateListeners：{ 监听了重连成功事件的执行recover()的StateListener }
  - String root="/dubbo_test"
  - URL registryUrl = zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&cellinvokemode=sharing&client=zkclient&dubbo=2.0.0&group=dubbo_test&interface=com.alibaba.dubbo.registry.RegistryService&pid=69353&timestamp=1550921731982
  - Set<URL> registered：0  // 已经注册的url集合，此处为空
  - ConcurrentMap<URL, Set<NotifyListener>> subscribed：0  // 已经订阅的数据<订阅的条件URL,  通知监听器Set<NotifyListener>>
  - ConcurrentMap<URL, Map<String, List<URL>>> notified：0 // 已经通知的<订阅条件URL, Map<String(category), List<URL>>数据>
  - Set<URL> failedRegistered：0 // 注册失败的url
  - Set<URL> failedUnregistered：0 // 反注册失败的url
  - ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed：0 // 订阅失败的url
  - ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed：0 // 反订阅失败的url
  - ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified：0 // 通知失败的url
  - ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners：0
  ```

还有一个定时线程：`DubboRegistryFailedRetryTimer`每隔5s执行一次`retry()`，进行失败重试。

最后，该`ZookeeperRegistry`实例会存储在`ZookeeperRegistry`的父类的`static`属性`Map<String, Registry> REGISTRIES`中：

```java
Map<String, Registry> REGISTRIES：{ "zookeeper://127.0.0.1:2181/dubbo_test/com.alibaba.dubbo.registry.RegistryService" : ZookeeperRegistry实例 }
```

### 三、获取注册到zk的服务提供者url

```java
// 获取注册到注册中心的提供者url dubbo://....
final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

// 返回注册到注册中心的提供者URL ，对 URL 参数进行一次过滤。
// @param originInvoker 原始的服务提供者执行体 DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
// @return 提供者url
private URL getRegisteredProviderUrl(final Invoker<?> originInvoker) {
    // dubbo://....
    URL providerUrl = getProviderUrl(originInvoker);
    // 注册中心中看到的提供者地址，过滤URL中不需要输出的参数(以点号开头的)，删除monitor、bing.ip、bind.port key
    return providerUrl.removeParameters(getFilteredKeys(providerUrl))
        .removeParameter(Constants.MONITOR_KEY)
        .removeParameter(Constants.BIND_IP_KEY)
        .removeParameter(Constants.BIND_PORT_KEY);
}

// 从invoker的URL中的Map<String, String> parameters中获取key为export的值providerUrl：
// @param originInvoker DelegateProviderMetaDataInvoker类型(包装AbstractProxyInvoker实例)
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

// 过滤URL中不需要输出的参数(以点号开头的)
private static String[] getFilteredKeys(URL url) {
    Map<String, String> params = url.getParameters();
    if (params != null && !params.isEmpty()) {
        List<String> filteredKeys = new ArrayList<String>();
        for (Map.Entry<String, String> entry : params.entrySet()) {
            if (entry != null && entry.getKey() != null && entry.getKey().startsWith(Constants.HIDE_KEY_PREFIX)) {
                filteredKeys.add(entry.getKey());
            }
        }
        return filteredKeys.toArray(new String[filteredKeys.size()]);
    } else {
        return new String[]{};
    }
}
```

最后得到的`registedProviderUrl`是：

```java
dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=69353&side=provider&timestamp=1550921732007
```

### 四、注册服务提供者到zookeeper

```java
if (register) {
    // 注册服务提供者
    register(registryUrl, registeredProviderUrl);
    ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
}

// 注册服务提供者。
// @see Registry.register(URL)
// @param registryUrl 注册中心配置 zookeeper://...
// @param registeredProviderUrl 服务提供者配置 dubbo://....
public void register(URL registryUrl, URL registeredProviderUrl) {
    // registryUrl protocol: zookeeper，registryFactory.getRegistry()实际调用的是ZookeeperRegistryFactory.getRegistry()
    // 而ZookeeperRegistryFactory继承于AbstractRegistryFactory
    // registry: ZookeeperRegistry实例，继承于FailbackRegistry。
    Registry registry = registryFactory.getRegistry(registryUrl);
    // 注册服务提供者
    registry.register(registeredProviderUrl);
}
```

这里的`registry`是`ZookeeperRegistry`。`registry.register(registeredProviderUrl)`方法在`ZookeeperRegistry`的父类`FailbackRegistry`中实现。

#####1.FailbackRegistry.register(registedProviderUrl)

```java
// 注册数据
// @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
@Override
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // 向服务器端发送注册请求
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;

        // 如果开启了启动时检测check=true，则直接抛出异常，不会加入到failedRegistered中
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        // 跳过失败重试
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
        } else {
            logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }

        // 将失败的注册请求记录到失败列表，定时重试
        failedRegistered.add(url);
    }
}
```

首先调用父类`AbstractRegistry`的`register(registedProviderUrl)`将当前的`registeredProviderUrl`放到`Set<URL> registered`属性中，如下：

```java
// 注册该数据
// @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
@Override
public void register(URL url) {
    if (url == null) {
        throw new IllegalArgumentException("register url == null");
    }
    if (logger.isInfoEnabled()) {
        logger.info("Register: " + url);
    }
    registered.add(url); // 添加数据
}
```

之后从`failedRegistered`和`failedUnregistered`两个`url`集合中删除该`url`。然后执行真正的服务注册（创建节点，`doRegister(url)`），如果在创建过程中抛出异常，如果`url`的协议不是`consumer`并且开启了`check=true`的属性并且当前存储的`URL registryUrl`也有`check=true`的话，那么直接抛出异常，不会将该`url`加入到`failedRegistered`集合；当然抛出的异常如果是`SkipFailbackWrapperException`，那么也会直接抛出异常，不会将该`url`加入到`failedRegistered`集合。否则，会将该`url`加入到`failedRegistered`集合，然后`DubboRegistryFailedRetryTimer`线程会每隔5s执行一次`doRegister(url)`。

下面看真正的注册逻辑`doRegister(url)`。

#####2.ZookeeperRegistry.doRegister(registedProviderUrl)

```java
// 创建服务配置URL叶子节点。
// @param url 注册信息 服务提供者url
@Override
protected void doRegister(URL url) {
    try {
        // toUrlPath(url) 返回url在zk上的路径 /dubbo_test/com.alibaba.dubbo.demo.DemoService/providers/dubbo...
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

首先是对入参`registedProviderUrl`进行处理:

```java
private String toUrlPath(URL url) {
    // /dubbo_test/com.alibaba.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F172.16.132.166%3A20881%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26cellinvokemode%3Dsharing%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D69353%26side%3Dprovider%26timestamp%3D1550921732007
    return toCategoryPath(url) + Constants.PATH_SEPARATOR + URL.encode(url.toFullString());
}

// /dubbo_test/com.alibaba.dubbo.demo.DemoService/providers
private String toCategoryPath(URL url) {
    return toServicePath(url) + Constants.PATH_SEPARATOR + url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
}

// /dubbo_test/com.alibaba.dubbo.demo.DemoService
private String toServicePath(URL url) {
    String name = url.getServiceInterface(); // com.alibaba.dubbo.demo.DemoService
    if (Constants.ANY_VALUE.equals(name)) {
        return toRootPath();
    }
    return toRootDir() + URL.encode(name);
}

// /dubbo_test/
private String toRootDir() {
    if (root.equals(Constants.PATH_SEPARATOR)) {
        return root;
    }
    return root + Constants.PATH_SEPARATOR;
}

// /dubbo_test
private String toRootPath() {
    return root;
}
```

这里就体现了上边的`ZookeeperRegistry`的`root`属性的作用。最终实际上得到的是：`/dubbo_test/interface/category/encode过的export`，该节点也将是创建在zk上的节点。

- `/dubbo_test`是根节点
- `/interface`是服务接口
- `/category`是`providers／consumers／routers／configurators`等

最终得到的`url`是：

- `/dubbo_test/com.alibaba.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F172.16.132.166%3A20881%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26cellinvokemode%3Dsharing%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D69353%26side%3Dprovider%26timestamp%3D1550921732007`
- 解码后：`/dubbo_test/com.alibaba.dubbo.demo.DemoService/providers/dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=69353&side=provider&timestamp=1550921732007`

最后执行`zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true))`来创建节点，该方法由`ZkclientZookeeperClient`的父类`AbstractZookeeperClient`来执行：

```java
@Override
public void create(String path, boolean ephemeral) {
    int i = path.lastIndexOf('/');
    if (i > 0) {
        String parentPath = path.substring(0, i);
        if (!checkExists(parentPath)) {
            create(parentPath, false); // 递归创建父节点
        }
    }
    if (ephemeral) {
        createEphemeral(path);
    } else {
        createPersistent(path);
    }
}
```

这里实际上是通过递归分别创建持久化的`/dubbo_test`，`/dubbo_test/com.alibaba.dubbo.demo.DemoService`以及`/dubbo_test/com.alibaba.dubbo.demo.DemoService/providers`节点；最后创建临时节点`/dubbo_test/com.alibaba.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F172.16.132.166%3A20881%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26cellinvokemode%3Dsharing%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D69353%26side%3Dprovider%26timestamp%3D1550921732007`。值得注意的是，`url.getParameter(Constants.DYNAMIC_KEY, true)`为`true`则最终创建的节点是临时节点，否则是持久化节点。

创建节点的操作是在`ZkclientZookeeperClient`中进行的。

```java
/// ZkclientZookeeperClient类方法
@Override
public void createPersistent(String path) {
    try {
        // client ZkClientWrapper，包装(ZkClient客户端)
        client.createPersistent(path);
    } catch (ZkNodeExistsException ignored) {
    }
}

@Override
public void createEphemeral(String path) {
    try {
        client.createEphemeral(path);
    } catch (ZkNodeExistsException ignored) {
    }
}

/// ZkClientWrapper类方法
// 创建临时节点
public void createEphemeral(String path) {
    Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
    // client ZkClient实例
    client.createEphemeral(path);
}

// 创建持久节点
public void createPersistent(String path) {
    Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
    client.createPersistent(path, true);
}
```

到此为止，服务提供者注册服务到zk成功。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/%E6%9C%8D%E5%8A%A1%E6%8F%90%E4%BE%9B%E8%80%85%E6%B3%A8%E5%86%8C%E5%88%B0zk%E7%9A%84%E7%BB%93%E6%9E%9C.png)

现在再来看看`ZookeeperRegistry`的属性变化。相较于注册前：

- `Set<URL> registered：[ dubbo://172.16.132.166:20881/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&cellinvokemode=sharing&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=69353&side=provider&timestamp=1550921732007 ]`

