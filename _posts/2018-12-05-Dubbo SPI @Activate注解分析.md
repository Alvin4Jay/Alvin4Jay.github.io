---
layout:     post
title:      Dubbo SPI @Activate注解分析
subtitle:   根据条件激活对应的SPI扩展点
date:       2018-12-05
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo SPI @Activate注解分析

`Dubbo @Activate`注解机制是对`Dubbo SPI`机制的扩展，该注解用在`SPI`接口实现的定义上，表明这些`SPI`扩展接口的实现类被激活或者不被激活的条件，比如`Filter`接口有很多实现，如`AccessLogFilter/GenericImplFilter`等，`Dubbo` 框架在`RPC`调用过程中可指定具体的条件，与这些`SPI`接口实现类`@Activate`注解上要求的条件进行匹配，成功则这些`SPI`实现类激活，否则不被激活，从而在`RPC`调用中起到作用。

## @Activate注解

`@Activate`注解可以注解在类和方法上，该注解定义如下:

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * Group过滤条件。
     * <br />
     * 包含{@link ExtensionLoader#getActivateExtension}的group参数给的值，则返回扩展。
     * <br />
     * 如没有Group设置，则不过滤。
     */
    String[] group() default {};

    /**
     * Key过滤条件。包含{@link ExtensionLoader#getActivateExtension}的URL的参数Key中有，则返回扩展。
     * <p/>
     * 示例：<br/>
     * 注解的值 <code>@Activate("cache,validatioin")</code>，
     * 则{@link ExtensionLoader#getActivateExtension}的URL的参数有<code>cache</code>Key，或是<code>validatioin</code>则返回扩展。
     * <br/>
     * 如没有设置，则不过滤。
     */
    String[] value() default {};

    /**
     * 排序信息，可以不提供。
     */
    String[] before() default {};

    /**
     * 排序信息，可以不提供。
     */
    String[] after() default {};

    /**
     * 排序信息，可以不提供。
     */
    int order() default 0;
}
```

该注解包含两个匹配条件`value`和`group`，两者都是字符串数组，用来指定`SPI`实现类激活的条件；`before/after/order`表示排序顺序，在这些`SPI`实现类获取并进行排序时起作用。以`com.alibaba.dubbo.rpc.Filter`举例:

```java
// AccessLogFilter，指定如果Filter使用方属于Constants.PROVIDER(provider)，并且URL中包含有效的
// Constants.ACCESS_LOG_KEY(accesslog)参数，就激活这个AccessLogFilter过滤器，并且指定了排序顺序
// -8000.
@Activate(group = Constants.PROVIDER, value = Constants.ACCESS_LOG_KEY, order = -8000)
public class AccessLogFilter implements Filter {
    
}

// GenericImplFilter，指定如果Filter使用方属于Constants.CONSUMER(consumer)，并且URL中包含有效的
// Constants.GENERIC_KEY(generic)参数，就激活这个GenericImplFilter过滤器，并且指定了排序顺序
// 20000.
@Activate(group = Constants.CONSUMER, value = Constants.GENERIC_KEY, order = 20000)
public class GenericImplFilter implements Filter {
    
}
```

## Dubbo SPI中激活扩展的实现解析

上述表明了`@Activate`注解可以配置`SPI`实现类被激活使用的条件，下面分析`Dubbo SPI`如何指定条件，找到匹配的`SPI`扩展实现，实现过程在`ExtensionLoader`中。

```java
// 如下的4个方法可用于获取激活的扩展点，前三个方法最终调用的是第4个方法
getActivateExtension(URL url, String key)
getActivateExtension(URL url, String[] values)
getActivateExtension(URL url, String key, String group)
getActivateExtension(URL url, String[] values, String group)
```

```java
/**
 * 根据 url和url参数key 返回激活的扩展点列表
 * @param url url
 * @param key url参数key，用于获取扩展定实现类的名字
 * @return 激活的扩展点
 * @see #getActivateExtension(com.alibaba.dubbo.common.URL, String, String)
 */
public List<T> getActivateExtension(URL url, String key) {
    return getActivateExtension(url, key, null);
}

/**
 * 根据 url和指定的扩展定实现类的名字数组 返回激活的扩展点列表
 * @param url url
 * @param 扩展定实现类的名字
 * @return 激活的扩展点
 * @see #getActivateExtension(com.alibaba.dubbo.common.URL, String[], String)
 */
public List<T> getActivateExtension(URL url, String[] values) {
    return getActivateExtension(url, values, null);
}

/**
 * 根据 url和url参数key以及group 返回激活的扩展点列表
 * @param url   url
 * @param key   url参数key，用于获取扩展定实现类的名字
 * @param group group
 * @return 激活的扩展点
 * @see #getActivateExtension(com.alibaba.dubbo.common.URL, String[], String)
 */
public List<T> getActivateExtension(URL url, String key, String group) {
    // 根据key，从url获取扩展点实现类的名字数组
    String value = url.getParameter(key);
    return getActivateExtension(url, value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), group);
}

/**
 * 获取激活的扩展(最终的激活匹配逻辑在这里)
 * @param url    url
 * @param values 扩展定实现类的名字，用户指定的扩展点实例类
 * @param group  用户指定的组
 * @return 激活的扩展点
 * @see com.alibaba.dubbo.common.extension.Activate
 */
public List<T> getActivateExtension(URL url, String[] values, String group) {
    List<T> exts = new ArrayList<T>(); // 最终返回的激活的扩展点列表
    List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values); // 根据参数key从url获取到的扩展点实现类的名字
    // 名字中不包含“-default”
    if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
        // 加载所有的扩展点实现类
        getExtensionClasses();
        // 在getExtensionClasses()调用过程中，遇到注解了@Activate注解的SPI实现类，会把
        // 实现类的name和@Activate实例以Key-Value的形式存入cachedActivates，便于下面匹配
        // 激活的扩展时使用
        /* loadFile(Map<String, Class<?>> extensionClasses, String dir)中
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                cachedActivates.put(names[0], activate);
            }
        */
        for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
            String name = entry.getKey(); // 扩展点实现类的名字
            Activate activate = entry.getValue();  // 扩展点实现类上@Activate实例
            if (isMatchGroup(group, activate.group())) { // 先根据group匹配
                // group匹配后获取扩展点实例
                T ext = getExtension(name);
                // 1. 根据参数key从url获取到的扩展点实现类的名字中不包含当前遍历的实现类名字
                // 2. 不排除当前遍历的实现类名字(-name)
                // 3. activate实例的value 在url有对应参数key和有效的值
                if (!names.contains(name)
                    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                    && isActive(activate, url)) {
                    exts.add(ext);
                }
            }
        }
        // 根据Activate注解的before、after、order信息，对Activate实例排序
        Collections.sort(exts, ActivateComparator.COMPARATOR);
    }
    List<T> usrs = new ArrayList<T>(); // usrs暂存扩展点实例类
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        // name不是以-开始，或者names不包含-name(没有排除name)
        if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
            && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
            if (Constants.DEFAULT_KEY.equals(name)) {
                // 遇到default，就把用户指定的扩展点实现类中default之前的实现类放到exts前面
                if (usrs.size() > 0) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                // 否则先暂存到usrs
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    // 把用户指定的扩展点实现类中default之后的实现类放到exts后面
    if (usrs.size() > 0) {
        exts.addAll(usrs);
    }
    return exts; // 返回符合条件的激活扩展
}

/**
 * group是用户指定的组，groups是扩展点实现类@Activate注解上设置的组
 */
private boolean isMatchGroup(String group, String[] groups) {
    // 如果用户获取激活的扩展点时未指定group，直接返回true，表示该条件匹配
    if (group == null || group.length() == 0) {
        return true;
    }
    // group不为空时，需要看是否group在groups的范围内，是则返回true；否则返回false
    if (groups != null && groups.length > 0) {
        for (String g : groups) {
            if (group.equals(g)) {
                return true;
            }
        }
    }
    return false;
}

/**
 * activate是扩展点实现类对应的@Activate实例
 */
private boolean isActive(Activate activate, URL url) {
    String[] keys = activate.value(); // activate配置的value，也就是对应于url上的参数key
    // 扩展点实现类对url上的参数key无要求，返回true
    if (keys == null || keys.length == 0) {
        return true;
    }
    // 有要求
    for (String key : keys) {
        // 从url中获取参数，进行遍历，如果有一个参数同key一致，或者是以.key的方式结尾，
        // 并且对应的值是有效的，则返回true
        for (Map.Entry<String, String> entry : url.getParameters().entrySet()) {
            String k = entry.getKey();
            String v = entry.getValue();
            if ((k.equals(key) || k.endsWith("." + key))
                && ConfigUtils.isNotEmpty(v)) {
                return true;
            }
        }
    }
    return false;
}
```

根据上面的源码分析，可以得到如下的结论：

- 根据`ExtensionLoader.getActivateExtension`中的`group`和搜索到该SPI接口类型的扩展点实现类进行比较，如果`group`能匹配到，进行下一步`value`的匹配。
- `@Activate`中的`value`是参数是第二层过滤参数（第一层是通过`group`），在`group`校验通过的前提下，如果url中的参数（`k`）与值（`v`）中的参数名同`@Activate`中的`value`值一致或者包含，那么才会被选中。相当于加入了`value`后，条件更为苛刻点，需要`url`中有此参数并且参数必须存在有效值。
- `@Activate`的`order`参数对于同一个类型的多个扩展来说，`order`值越小，优先级越高。

## 应用场景

主要用在`filter`上，有的`filter`需要在`provider`边执行，有的需要在`consumer`边执行，根据`url`中的参数指定和`group`(`provider`还是`consumer`)，运行时决定哪些`filter`需要被引入执行。