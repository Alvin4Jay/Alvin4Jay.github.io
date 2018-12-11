---
layout:     post
title:      Spring xml配置:` <context:annotation-config`与`<context:component-scan>`比较
subtitle:   
date:       2018-12-11
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

#Spring xml配置:` <context:annotation-config`与`<context:component-scan>`比较

### 两者区别

`<context:annotation-config>` 与 `<context:component-scan>`之间的区别如下:

- `<context:annotation-config>`的作用是激活在应用上下文中已经注册的`bean`中对应的注解。该配置不关心`bean`是以`xml`配置还是使用 `<context:component-scan>`等方式被注册到应用上下文中的。

- `<context:component-scan>`的作用是将`bean`注册到应用上下文中，并扫描`bean`中的注解、激活它们。因此，`<context:component-scan>`具有`<context:annotation-config>`的功能之外，还具有包扫描、`bean`注册功能。

### 举例说明

下面的示例会创建3个`bean`， `BeanA` 依赖于 `BeanB`和`BeanC`，并且会用不同的配置方式配置它们，根据输出说明`<context:annotation-config>` 与 `<context:component-scan>`之间的区别。

##### Bean类与测试类

```java
// BeanA
@SuppressWarnings("unused")
@Component
public class BeanA {
    private BeanB beanB;
    private BeanC beanC;

    public BeanA() {
        System.out.println("Creating bean BeanA");
    }

    @Autowired
    public void setBeanB(BeanB beanB) {
        System.out.println("Setting bean reference to BeanB");
        this.beanB = beanB;
    }

    @Autowired
    public void setBeanC(BeanC beanC) {
        System.out.println("Setting bean reference to BeanC");
        this.beanC = beanC;
    }
}

// BeanB
@Component
public class BeanB {
    public BeanB() {
        System.out.println("Creating bean BeanB");
    }
}

// BeanC
@Component
public class BeanC {
    public BeanC() {
        System.out.println("Creating bean BeanC");
    }
}

// 测试类
public class BeanDemo {
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
    }
}
```

下面开始配置`beans.xml`文件。

##### (a)只定义bean tags

```xml
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanA" id="beanA"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanB" id="beanB"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanC" id="beanC"/>

// 输出
Creating bean BeanA
Creating bean BeanB
Creating bean BeanC
```

这个例子中三个`bean`都被创建了，并且`BeanB`和`BeanC`没有被注入到`BeanA`中，因为没有在`BeanA`的定义中使用`property ref`属性。

#####(b)定义 bean tags和property ref属性

```xml
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanA" id="beanA">
    <property name="beanB" ref="beanB"/>
    <property name="beanC" ref="beanC"/>
</bean>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanB" id="beanB"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanC" id="beanC"/>

// 输出
Creating bean BeanA
Creating bean BeanB
Creating bean BeanC
Setting bean reference to BeanB
Setting bean reference to BeanC
```

这个例子中三个`bean`都被创建了，并且`BeanB`和`BeanC`被注入到了`BeanA`中。

##### (c)只使用<context:annotation-config />

```xml
<context:annotation-config/>
// 无输出
```

`<context:annotation-config />` 只会发现和激活已经注册到应用上下文中的`bean`中的注解，在这里我们未找到任何注册的`bean`。

##### (d)使用<context:annotation-config />和bean定义

```xml
<context:annotation-config/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanA" id="beanA"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanB" id="beanB"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanC" id="beanC"/>

// 输出
Creating bean BeanA
Creating bean BeanB
Setting bean reference to BeanB
Creating bean BeanC
Setting bean reference to BeanC
```

上面的例子使用`<bean/>`元素注册`bean`到应用上下文，因此`<context:annotation-config/>`能发现注册的`bean`，并激活`@Autowired`注解，注入依赖到`BeanA`。

##### (e)只使用<context:component-scan />

```xml
<context:component-scan base-package="com.wacai.middleware.annotationconfiguration.config"/>

// 输出
Creating bean BeanA
Creating bean BeanB
Setting bean reference to BeanB
Creating bean BeanC
Setting bean reference to BeanC
```

`<context:component-scan />`扫描包并注册带有`@Component`注解的`bean`到应用上下文，并激活`bean`上的注解，如`@Autowired`。

#####(f)使用<context:annotation-config />和<context:component-scan />

```xml
<context:annotation-config/>
<context:component-scan base-package="com.wacai.middleware.annotationconfiguration.config"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanA" id="beanA"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanB" id="beanB"/>
<bean class="com.wacai.middleware.annotationconfiguration.config.BeanC" id="beanC"/>

// 输出
Creating bean BeanA
Creating bean BeanB
Setting bean reference to BeanB
Creating bean BeanC
Setting bean reference to BeanC
```

从这个例子可以看出，尽管配置了多种注册方式，但`bean`定义只会注册一次，并且只会被处理一次。

以上便是`<context:annotation-config>` 与 `<context:component-scan>`之间区别的说明。

###参考

1. [Spring MVC: <context:annotation-config> vs <context:component-scan>](https://howtodoinjava.com/spring-mvc/spring-mvc-difference-between-contextannotation-config-vs-contextcomponent-scan/)
2. [Spring配置文件详解：<context:annotation-config/>和<context:component-scan base-package=""/>和<mvc:annotation-driven />](https://www.cnblogs.com/lcngu/p/5080702.html)

