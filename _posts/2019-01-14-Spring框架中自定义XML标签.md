---
layout:     post
title:      Spring框架中自定义XML标签
subtitle:   NamespaceHandler/BeanDefinitionParser
date:       2019-01-14
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# Spring框架中自定义XML标签

转自[李懿名-spring自定义xml标签](https://blog.csdn.net/lilongjiu/article/details/76695310)



要实现自定义的`xml`配置，需要有两个默认`spring`配置文件来支持。一个是`spring.schemas`,一个是`spring.handlers`，前者是为了验证你自定义的`xml`配置文件是否符合你的格式要求，后者是告诉`spring`该如何来解析你自定义的配置文件。

自定义标签涉及的核心接口为：

- `NamespaceHandler`
- `BeanDefinitionParser`
  实际使用的时候，**一般**分别继承类：

- `NamespaceHandlerSupport：init`
- `AbstractSingleBeanDefinitionParser：parse`

以`spring`事务标签为例（前提：要了解`bean`注册过程）：

### 一、配置文件

- `spring.handlers `

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-1.png)

- `spring.schemas `

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-2.png)

### 二、调用逻辑

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-3.png)

```java
org.springframework.context.support.AbstractApplicationContext#refresh 
org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory 
org.springframework.context.support.AbstractApplicationContext#refreshBeanFactory 
org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory 
org.springframework.context.support.AbstractRefreshableApplicationContext#loadBeanDefinitions org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)
```

- 在第3步中 **`registerBeanDefinitions`** 调用 **`createReaderContext`**

```java
// class XmlBeanDefinitionReader
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	int countBefore = getRegistry().getBeanDefinitionCount();
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

- 第4步 **`createReaderContext`** 中调用`getNamespaceHandlerResolver`方法获取： **`NamespaceHandlerResolver`**

```java
// class XmlBeanDefinitionReader
public XmlReaderContext createReaderContext(Resource resource) {
    return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
    this.sourceExtractor, this, getNamespaceHandlerResolver());
    }

public NamespaceHandlerResolver getNamespaceHandlerResolver() {
    if (this.namespaceHandlerResolver == null) {
    this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
    }
    return this.namespaceHandlerResolver;
}
```

`new `一个 **`DefaultNamespaceHandlerResolver`**

```java
 // class XmlBeanDefinitionReader
 protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
 	return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
 }
```

`DEFAULT_HANDLER_MAPPINGS_LOCATION: META-INF/spring.handlers`

```java
// DefaultNamespaceHandlerResolver
public DefaultNamespaceHandlerResolver(ClassLoader classLoader) {
    this(classLoader, DEFAULT_HANDLER_MAPPINGS_LOCATION);
}

public DefaultNamespaceHandlerResolver(ClassLoader classLoader, String handlerMappingsLocation) {
    Assert.notNull(handlerMappingsLocation, "Handler mappings location must not be null");
    this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
    this.handlerMappingsLocation = handlerMappingsLocation;
}
```

`DefaultNamespaceHandlerResolver`的属性：`private volatile Map handlerMappings`保存的是 
`namespace URI` 与 `NamespaceHandler`的映射

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-4.png)

- 第5步调用`DefaultBeanDefinitionDocumentReader`（父类`BeanDefinitionDocumentReader`）的 **`registerBeanDefinitions`** 方法

```java
// BeanDefinitionDocumentReader
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}

protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }

    preProcessXml(root);
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);

    this.delegate = parent;
}
```

**`delegate.parseCustomElement(ele);`** 这一行代码是解析自定义标签

```java
// class DefaultBeanDefinitionDocumentReader
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

1. 根据元素标签获取：`NamespaceHandler`，并调用其 **`init`** 方法
2. 调用 `NamespaceHandler` 的 **`parse`** 方法，在 `parse`方法里解析标签，然后返回`BeanDefinition`，后续会被`spring`注册

```java
 // class BeanDefinitionParserDelegate
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(ele);
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

### 三、定制点

- **定制 `init` 方法**：一般在`init`方法中注册`BeanDefinitionParser`, 存储在其父类`NamespaceHandlerSupport`的属性：`parsers` 中

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-5.png)

- **定制 `parse` 方法**：一般在`parse`方法中返回`BeanDefinition`

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/spring-xml-6.png)

