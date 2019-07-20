---
layout:     post
title:      `<context:component-scan/>`配置的解析与`@Autowired`注解的属性与方法的值注入
subtitle:   AutowiredAnnotationBeanPostProcessor
date:       2019-07-19
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# `<context:component-scan/>`配置的解析与`@Autowired`注解的属性与方法的值注入

前面的文章[Spring IoC容器源码分析](https://xuanjian1992.top/2019/07/14/Spring-IoC%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)从大致的流程上，分析了Spring IoC容器的启动与Bean初始化流程。下面在此基础之上，首先分析Spring对xml配置中`<context:component-scan/>`元素的解析过程，然后分析`@Autowired`注解的属性与方法的值注入过程。

举例如下：
POJO：

```java
// 用户模型
public class User {

   @Autowired // @Autowired注解的依赖注入
   private Address address;

    // id
    private int id;

    // 用户名
    private String name;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

   public Address getAddress() {
      return address;
   }

   public void setAddress(Address address) {
      this.address = address;
   }

   @Override
   public String toString() {
      return "User{" +
            "address=" + address +
            ", id=" + id +
            ", name='" + name + '\'' +
            '}';
   }

}

public class Address {

	private String id;
	private String mail;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public String getMail() {
		return mail;
	}

	public void setMail(String mail) {
		this.mail = mail;
	}
}
```

Bean配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd">

  <bean id="user" class="instantiate.defaultconstruct.User" >
		<property name="name" value="xuan"/>
		<property name="id" value="1"/>
	</bean>

	<bean id="address" class="instantiate.defaultconstruct.Address"/>
	<!--<context:component-scan />解析-->
  <context:component-scan base-package="instantiate.defaultconstruct"/>

</beans>
```

测试类：

```java
public class UserTest {
   public static void main(String[] args) {
      ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("defaultconstruct/user.xml");
      User user = (User) context.getBean("user");
      System.out.println(user.toString());
   }
}

// 输出：
User{address=instantiate.defaultconstruct.Address@57d5872c, id=1, name='xuan'}
```


### 一、`<context:component-scan/>`元素的解析

##### 1. 获取ContextNameSpaceHandler

xml元素的解析过程由DefaultBeanDefinitionDocumentReader.parseBeanDefinitions方法完成，同样`<context:component-scan/>`的解析也是在这个方法进行的。

```java
// DefaultBeanDefinitionDocumentReader类
// default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />，其他的属于 custom 的
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
   if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
               // 解析 default namespace 下面的几个元素
               parseDefaultElement(ele, delegate);
            }
            else {
               // 解析custom namespace下面的几个元素 如<context:component-scan base-
               // package="xxx.yyy.zzz"/>
               // 对于component-scan：注册扫描到的@Component注解的类的BeanDefinition和
               // @Autowired、@PreDestroy等注解的后置处理器的BeanDefinition
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

由于`<context:component-scan/>`元素不属于默认命名空间的元素，所以下面执行`delegate.parseCustomElement(ele)`。

```java
// BeanDefinitionParserDelegate类
public BeanDefinition parseCustomElement(Element ele) {
   return parseCustomElement(ele, null);
}
// BeanDefinitionParserDelegate类
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) { 
   // containingBd: 外部类的BeanDefinition，相对于内部类
   // 如http://www.springframework.org/schema/context
   String namespaceUri = getNamespaceURI(ele); 
   if (namespaceUri == null) {
      return null;
   }
   // this.readerContext.getNamespaceHandlerResolver(): DefaultNamespaceHandlerResolver
   // handler: ContextNameSpaceHandler
   NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri); 
   if (handler == null) {
      error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
      return null;
   }
   return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

根据上面的流程，首先分析`this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri)`的逻辑:

```java
// DefaultNamespaceHandlerResolver类
// 根据namespaceUri获取NamespaceHandler
public NamespaceHandler resolve(String namespaceUri) {
   Map<String, Object> handlerMappings = getHandlerMappings(); // uri--NamespaceHandler映射
   Object handlerOrClassName = handlerMappings.get(namespaceUri);
   if (handlerOrClassName == null) {
      return null;
   }
   else if (handlerOrClassName instanceof NamespaceHandler) {
      return (NamespaceHandler) handlerOrClassName;
   }
   else {
      String className = (String) handlerOrClassName; // 类名
      try {
         Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
         // 检查是否实现了NamespaceHandler接口
         if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
            throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                  "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
         }
         // 实例化NameSpaceHandler
         NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
         namespaceHandler.init(); // 调用init方法，注册BeanDefinitionParser
         handlerMappings.put(namespaceUri, namespaceHandler); // 缓存
         return namespaceHandler;
      }
      catch (ClassNotFoundException ex) {
         throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
               "] for namespace [" + namespaceUri + "]", ex);
      }
      catch (LinkageError err) {
         throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
               className + "] for namespace [" + namespaceUri + "]", err);
      }
   }
}
// DefaultNamespaceHandlerResolver类
// 懒加载NamespaceHandler映射(从文件读取)<namespaceUri, NameSpaceHandler>
private Map<String, Object> getHandlerMappings() {
	Map<String, Object> handlerMappings = this.handlerMappings;
	if (handlerMappings == null) {
		synchronized (this) {
			handlerMappings = this.handlerMappings;
			if (handlerMappings == null) { // double check
				if (logger.isTraceEnabled()) {
					logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
				}
				try {
					Properties mappings = // 从类路径META-INF/spring.handlers中加载handler
							PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
					if (logger.isTraceEnabled()) {
						logger.trace("Loaded NamespaceHandler mappings: " + mappings);
					}
					handlerMappings = new ConcurrentHashMap<>(mappings.size());
					CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings); // 缓存
					this.handlerMappings = handlerMappings;
				}
				catch (IOException ex) {
					throw new IllegalStateException(
							"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
				}
			}
		}
	}
	return handlerMappings;
}
```

可以看出，resolve(String namespaceUri)方法首先加载类路径下的META-INF/spring.handlers文件，这些文件里面保存着namespaceUri与NameSpaceHandler的映射关系。此外，如果获取到的NameSpaceHandler还未实例化，则首先实例化，然后调用其init方法。init方法逻辑如下：

```java
// ContextNamespaceHandler类
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

   @Override
   public void init() {
     	// 注册context命名空间下不同元素的解析器
      registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
      registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
      registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
      registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
      registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
      registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
   }

}
// NamespaceHandlerSupport类
protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
	this.parsers.put(elementName, parser);
}
```

##### 2. `<context:component-scan/>`元素解析

上面得到了ContextNameSpaceHandler处理器，然后执行BeanDefinitionParserDelegate.parseCustomElement方法里的`handler.parse(ele, new ParserContext(this.readerContext, this, containingBd))`逻辑。

```java
// NamespaceHandlerSupport类
public BeanDefinition parse(Element element, ParserContext parserContext) {
   // 如ComponentScanBeanDefinitionParser
   BeanDefinitionParser parser = findParserForElement(element, parserContext);
   return (parser != null ? parser.parse(element, parserContext) : null);
}
// NamespaceHandlerSupport类
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
	String localName = parserContext.getDelegate().getLocalName(element); // 如component-scan
	// 如ComponentScanBeanDefinitionParser
  BeanDefinitionParser parser = this.parsers.get(localName); 
	if (parser == null) {
		parserContext.getReaderContext().fatal(
				"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
	}
	return parser;
}
```

ComponentScanBeanDefinitionParser.parse方法执行真正的`<context:component-scan/>`解析逻辑。

```java
// ComponentScanBeanDefinitionParser类
// <context:component-scan/>元素解析
public BeanDefinition parse(Element element, ParserContext parserContext) {
   String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE); // 扫描的包名
   // 解析扫描包路径中的占位符
   basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage); 
   // 解析传入的包路径为包路径数组
   String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
         ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS); 

   // Actually scan for bean definitions and register them.
   // 1.配置ClassPathBeanDefinitionScanner
   ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
   // 2.扫描组件，注册BeanDefinition
   Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
   // 3.注册注解后置处理器
   registerComponents(parserContext.getReaderContext(), beanDefinitions, element); 

   return null;
}
```

解析过程可分为以下三步。

###### (1) 配置ClassPathBeanDefinitionScanner

```java
// ComponentScanBeanDefinitionParser类
// 配置ClassPathBeanDefinitionScanner
protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
   boolean useDefaultFilters = true;
   // 是否设置了use-default-filters属性
   if (element.hasAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE)) { 
      useDefaultFilters = Boolean.valueOf(element.getAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE));
   }

   // Delegate bean definition registration to scanner class.
   // 创建ClassPathBeanDefinitionScanner
   ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
	 // 设置默认值   
 	scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults()); 
   // 设置装配候选者pattern。e.g. "*Service", "data*", "*Service*", "data*Service", 逗号分隔："*Service,*Dao".
   scanner.setAutowireCandidatePatterns(parserContext.getDelegate().getAutowireCandidatePatterns());

   if (element.hasAttribute(RESOURCE_PATTERN_ATTRIBUTE)) {
      // 设置资源模式 **/*.class
      scanner.setResourcePattern(element.getAttribute(RESOURCE_PATTERN_ATTRIBUTE)); 
   }

   try {
      parseBeanNameGenerator(element, scanner); // 设置BeanNameGenerator
   }
   catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }

   try {
      parseScope(element, scanner); // 设置ScopeMetadataResolver
   }
   catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }

   parseTypeFilters(element, scanner, parserContext); // 设置TypeFilter

   return scanner;
}
```

createScanner方法:

```java
// ComponentScanBeanDefinitionParser类
// 创建ClassPathBeanDefinitionScanner
protected ClassPathBeanDefinitionScanner createScanner(XmlReaderContext readerContext, boolean useDefaultFilters) {
   return new ClassPathBeanDefinitionScanner(readerContext.getRegistry(), useDefaultFilters,
         readerContext.getEnvironment(), readerContext.getResourceLoader());
}
// ClassPathBeanDefinitionScanner类 初始化
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters, Environment environment, @Nullable ResourceLoader resourceLoader) {

	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	this.registry = registry;

	if (useDefaultFilters) {
		registerDefaultFilters(); // 注册默认的Filter
	}
	setEnvironment(environment);
	setResourceLoader(resourceLoader);
}
// ClassPathBeanDefinitionScanner类
// 注册默认的Filter
protected void registerDefaultFilters() {
	// 添加AnnotationTypeFilter  @Component @ManagedBean @Named
	this.includeFilters.add(new AnnotationTypeFilter(Component.class));
	ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
	try {
		this.includeFilters.add(new AnnotationTypeFilter(
				((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
		logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
	}
	catch (ClassNotFoundException ex) {
		// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
	}
	try {
		this.includeFilters.add(new AnnotationTypeFilter(
				((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
		logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
	}
	catch (ClassNotFoundException ex) {
		// JSR-330 API not available - simply skip.
	}
}
```

parseBeanNameGenerator方法:

```java
// ComponentScanBeanDefinitionParser类
protected void parseBeanNameGenerator(Element element, ClassPathBeanDefinitionScanner scanner) {
   if (element.hasAttribute(NAME_GENERATOR_ATTRIBUTE)) {
      // 实例化BeanNameGenerator
      BeanNameGenerator beanNameGenerator = (BeanNameGenerator) instantiateUserDefinedStrategy(
            element.getAttribute(NAME_GENERATOR_ATTRIBUTE), BeanNameGenerator.class,
            scanner.getResourceLoader().getClassLoader());
      scanner.setBeanNameGenerator(beanNameGenerator);
   }
}
```

parseScope方法:

```java
// ComponentScanBeanDefinitionParser类
protected void parseScope(Element element, ClassPathBeanDefinitionScanner scanner) {
   // Register ScopeMetadataResolver if class name provided.
   if (element.hasAttribute(SCOPE_RESOLVER_ATTRIBUTE)) {
      if (element.hasAttribute(SCOPED_PROXY_ATTRIBUTE)) {
         throw new IllegalArgumentException(
               "Cannot define both 'scope-resolver' and 'scoped-proxy' on <component-scan> tag");
      }
      // 实例化ScopeMetadataResolver
      ScopeMetadataResolver scopeMetadataResolver = (ScopeMetadataResolver) instantiateUserDefinedStrategy(
            element.getAttribute(SCOPE_RESOLVER_ATTRIBUTE), ScopeMetadataResolver.class,
            scanner.getResourceLoader().getClassLoader()); 
      scanner.setScopeMetadataResolver(scopeMetadataResolver);
   }

   if (element.hasAttribute(SCOPED_PROXY_ATTRIBUTE)) { // scoped-proxy
      String mode = element.getAttribute(SCOPED_PROXY_ATTRIBUTE); // 设置代理模式
      if ("targetClass".equals(mode)) {
         scanner.setScopedProxyMode(ScopedProxyMode.TARGET_CLASS);
      }
      else if ("interfaces".equals(mode)) {
         scanner.setScopedProxyMode(ScopedProxyMode.INTERFACES);
      }
      else if ("no".equals(mode)) {
         scanner.setScopedProxyMode(ScopedProxyMode.NO);
      }
      else {
         throw new IllegalArgumentException("scoped-proxy only supports 'no', 'interfaces' and 'targetClass'");
      }
   }
}
```

parseTypeFilters:

```java
// ComponentScanBeanDefinitionParser类
protected void parseTypeFilters(Element element, ClassPathBeanDefinitionScanner scanner, ParserContext parserContext) {
   // Parse exclude and include filter elements. 
   // 解析exclude与include TypeFilter，设置到ClassPathBeanDefinitionScanner
   ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
   NodeList nodeList = element.getChildNodes();
   for (int i = 0; i < nodeList.getLength(); i++) {
      Node node = nodeList.item(i);
      if (node.getNodeType() == Node.ELEMENT_NODE) {
         String localName = parserContext.getDelegate().getLocalName(node);
         try {
            if (INCLUDE_FILTER_ELEMENT.equals(localName)) { // <context:include-filter />
               TypeFilter typeFilter = createTypeFilter((Element) node, classLoader, parserContext);
               scanner.addIncludeFilter(typeFilter);
            }
            else if (EXCLUDE_FILTER_ELEMENT.equals(localName)) { // <context:exclude-filter />
               TypeFilter typeFilter = createTypeFilter((Element) node, classLoader, parserContext);
               scanner.addExcludeFilter(typeFilter);
            }
         }
         catch (ClassNotFoundException ex) {
            parserContext.getReaderContext().warning(
                  "Ignoring non-present type filter class: " + ex, parserContext.extractSource(element));
         }
         catch (Exception ex) {
            parserContext.getReaderContext().error(
                  ex.getMessage(), parserContext.extractSource(element), ex.getCause());
         }
      }
   }
}
```

###### (2) 扫描组件，注册BeanDefinition

扫描指定的包，并注册BeanDefinition到BeanDefinitionRegistry。

```java
// ClassPathBeanDefinitionScanner类
// 扫描指定的包，并注册BeanDefinition到BeanDefinitionRegistry。该方法不会注册@Autowire 
// @PreDestroy等注解的后置处理器
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      // 找出所有候选的组件
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage); 
      for (BeanDefinition candidate : candidates) {
         // 解析scope
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         // 生成bean name
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            // 解析公共的注解
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) { // 检查对应beanName的BeanDefinition是否已存在
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            // 生成scope代理
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            registerBeanDefinition(definitionHolder, this.registry); // 注册BeanDefinition
         }
      }
   }
   return beanDefinitions;
}
```

findCandidateComponents方法:

```java
// ClassPathScanningCandidateComponentProvider类
// 找出所有候选的组件
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
   // Spring 5.0新特性(加快BeanDefinition扫描)，后续再分析
   if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
      return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
   }
   else {
      // 主要在这里
      return scanCandidateComponents(basePackage); // 扫描候选组件
   }
}
// ClassPathScanningCandidateComponentProvider类
// 找出所有候选的组件
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
	Set<BeanDefinition> candidates = new LinkedHashSet<>();
	try {
		// 扫描路径
		String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
				resolveBasePackage(basePackage) + '/' + this.resourcePattern;
		Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
		boolean traceEnabled = logger.isTraceEnabled();
		boolean debugEnabled = logger.isDebugEnabled();
		for (Resource resource : resources) {
			if (traceEnabled) {
				logger.trace("Scanning " + resource);
			}
			if (resource.isReadable()) { // 资源是否可读
				try {
					// getMetadataReaderFactory(): CachingMetadataReaderFactory
					// metadataReade: SimpleMetadataReader实例
					MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
          // 根据匹配filter和condition的结果，判断是否是候选组件
					if (isCandidateComponent(metadataReader)) { 
						ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
						sbd.setResource(resource); // 设置BeanDefinition资源来源
						sbd.setSource(resource);
						if (isCandidateComponent(sbd)) { // 判断ScannedGenericBeanDefinition是否符合要求
							if (debugEnabled) {
								logger.debug("Identified candidate component class: " + resource);
							}
							candidates.add(sbd);
						}
						else {
							if (debugEnabled) {
								logger.debug("Ignored because not a concrete top-level class: " + resource);
							}
						}
					}
					else {
						if (traceEnabled) {
							logger.trace("Ignored because not matching any filter: " + resource);
						}
					}
				}
				catch (Throwable ex) {
					throw new BeanDefinitionStoreException(
							"Failed to read candidate component class: " + resource, ex);
				}
			}
			else {
				if (traceEnabled) {
					logger.trace("Ignored because not readable: " + resource);
				}
			}
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
	}
	return candidates;
}
// ClassPathScanningCandidateComponentProvider类
// 判断是否是候选组件
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
	for (TypeFilter tf : this.excludeFilters) {
		// metadataReader：SimpleMetadataReader，getMetadataReaderFactory()：CachingMetadataReaderFactory
		if (tf.match(metadataReader, getMetadataReaderFactory())) { // 调用AnnotationTypeFilter的match方法
			return false; // 只要有一个匹配，就返回false
		}
	}
  // includeFilters包括 @Component @ManagedBean @Named (重点)
  // 检查类上是否存在这些注解，或者考虑元注解
	for (TypeFilter tf : this.includeFilters) {
		if (tf.match(metadataReader, getMetadataReaderFactory())) { // includeFilter匹配后，检查条件匹配
			return isConditionMatch(metadataReader); // 只要条件匹配，返回true
		}
	}
	return false;
}
// ClassPathScanningCandidateComponentProvider类
// 判断ScannedGenericBeanDefinition是否符合要求
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
	AnnotationMetadata metadata = beanDefinition.getMetadata();
	// 必须是独立的，可以是具体的或者是抽象的并且有@Lookup注解的方法
	return (metadata.isIndependent() && (metadata.isConcrete() ||
			(metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```

###### (3) 注册`@Autowired`等注解的后置处理器

```java
// ComponentScanBeanDefinitionParser类
// 注册@Autowired等注解的后置处理器
protected void registerComponents(
      XmlReaderContext readerContext, Set<BeanDefinitionHolder> beanDefinitions, Element element) {

   Object source = readerContext.extractSource(element);
   CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), source);

   for (BeanDefinitionHolder beanDefHolder : beanDefinitions) {
      compositeDef.addNestedComponent(new BeanComponentDefinition(beanDefHolder));
   }

   // Register annotation config processors, if necessary. 
   // 注册注解配置后置处理器，如@Autowired注解的后置处理器
   boolean annotationConfig = true;
   if (element.hasAttribute(ANNOTATION_CONFIG_ATTRIBUTE)) {
      annotationConfig = Boolean.valueOf(element.getAttribute(ANNOTATION_CONFIG_ATTRIBUTE));
   }
   if (annotationConfig) {
      // 注册注解配置的后置处理器，返回注册后的后置处理器BeanDefinitionHolder
      Set<BeanDefinitionHolder> processorDefinitions =
            AnnotationConfigUtils.registerAnnotationConfigProcessors(readerContext.getRegistry(), source);
      for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
         compositeDef.addNestedComponent(new BeanComponentDefinition(processorDefinition));
      }
   }

   readerContext.fireComponentRegistered(compositeDef); // 发送注册事件
}
```

AnnotationConfigUtils.registerAnnotationConfigProcessors方法:

```java
// AnnotationConfigUtils类
// 注册注解配置的后置处理器，返回注册后的后置处理器BeanDefinitionHolder
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, @Nullable Object source) {

   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         // 比较器设置
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE); 
      }
      // beanFactory.getAutowireCandidateResolver()：SimpleAutowireCandidateResolver实例
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) { 
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      // 注册ConfigurationClassPostProcessor的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      // 注册AutowiredAnnotationBeanPostProcessor的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      // 注册CommonAnnotationBeanPostProcessor的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      // 注册PersistenceAnnotationBeanPostProcessor的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      // 注册EventListenerMethodProcessor的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      // 注册 DefaultEventListenerFactory 的BeanDefinition
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

综上，在`<context:component-scan/>`解析过程中主要做了两件事，一是根据扫描的包路径，扫描出注解了@Component、@ManagedBean等注解的类并注册其BeanDefinition到BeanDefinitionRegistry；二是注册@Autowired、@PostConstruct、@PreDestroy等注解的后置处理器到BeanDefinitionRegistry。

### 二、`@Autowired`注解的属性与方法的值注入过程

上面分析了`<context:component-scan/>`元素(`<context:annotation-config/>`类似)解析过程中会向BeanDefinitionRegistry注册@Autowired、@PostConstruct、@PreDestroy等注解的后置处理器，如AutowiredAnnotationBeanPostProcessor、ConfigurationClassPostProcessor、CommonAnnotationBeanPostProcessor等。而下面将要分析的`@Autowired`注解的属性与方法的值注入过程，则与AutowiredAnnotationBeanPostProcessor有关。

首先看下AutowiredAnnotationBeanPostProcessor的类继承结构：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/AutowiredAnnotationBeanPostProcessor%E7%B4%AF%E7%BB%A7%E6%89%BF%E7%BB%93%E6%9E%84.png)

可见AutowiredAnnotationBeanPostProcessor实现了InstantiationAwareBeanPostProcessor和MergedBeanDefinitionPostProcessor接口。

在创建Bean的过程中，@Autowired等注解的处理时机在Bean实例化之后，属性填充之前。下面先看下AbstractAutowireCapableBeanFactory.doCreateBean的逻辑。

```java
// AbstractAutowireCapableBeanFactory类
// 实际的Bean创建过程
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean. 实例化Bean
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      // 1.说明不是 FactoryBean，实例化Bean(默认构造器实例化/工厂方法实例化/构造器注入实例化)
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance(); // bean实例
   Class<?> beanType = instanceWrapper.getWrappedClass(); // bean类型
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   // 涉及接口MergedBeanDefinitionPostProcessor(使用很少)
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references 解决循环依赖，及早缓存bean引用
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      // 添加单例对象的工厂
      // 此处bean还未填充属性
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance. 初始化
   Object exposedObject = bean; // 创建的实例
   try {
      // 2.填充Bean属性
      populateBean(beanName, mbd, instanceWrapper);
      // 3.初始化Bean，应用各种PostProcessor、init method
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }
   // TODO 待看 循环引用 ??
   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable. 注册销毁逻辑，比如Bean实现了DisposableBean接口，指定了destroy method等
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }
   // 返回最终的实例(可能是个代理)
   return exposedObject;
}
```

##### 1.准备装配元数据InjectionMetadata

在createBeanInstance实例化Bean之后，属性填充之前，会调用applyMergedBeanDefinitionPostProcessors方法：

```java
// AbstractAutowireCapableBeanFactory类
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof MergedBeanDefinitionPostProcessor) {
         // AutowiredAnnotationBeanPostProcessor实现了InstantiationAwareBeanPostProcessor接口
         MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
         bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
      }
   }
}
```

由于AutowiredAnnotationBeanPostProcessor实现了InstantiationAwareBeanPostProcessor接口，因此会调用AutowiredAnnotationBeanPostProcessor的postProcessMergedBeanDefinition方法。

```java
// AutowiredAnnotationBeanPostProcessor类
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   // 找出装配的元数据
   InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null); 
   // 注册InjectionMetadata的数据到BeanDefinition的externallyManagedConfigMembers
   metadata.checkConfigMembers(beanDefinition); 
}
```

postProcessMergedBeanDefinition中调用了findAutowiringMetadata方法找出装配的元数据，这是关键的一步。

```java
// AutowiredAnnotationBeanPostProcessor类
// 找出装配的元数据
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
   // Fall back to class name as cache key, for backwards compatibility with custom callers.
   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
   // Quick check on the concurrent map first, with minimal locking.
   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey); // 先检查缓存
   if (InjectionMetadata.needsRefresh(metadata, clazz)) { // InjectionMetadata需要刷新
      synchronized (this.injectionMetadataCache) {
         metadata = this.injectionMetadataCache.get(cacheKey);
         if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            if (metadata != null) {
               metadata.clear(pvs); // 清除processed属性
            }
            metadata = buildAutowiringMetadata(clazz); // 根据clazz构建InjectionMetadata
            this.injectionMetadataCache.put(cacheKey, metadata);
         }
      }
   }
   return metadata;
}
// AutowiredAnnotationBeanPostProcessor类
// 构建装配元数据
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
   List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
   Class<?> targetClass = clazz;

   do {
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
      // 在所有声明的field上执行回调
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
         // 回调逻辑
         // 获取注解(@Autowaired @Value @Inject)属性
         AnnotationAttributes ann = findAutowiredAnnotation(field); 
         if (ann != null) {
            if (Modifier.isStatic(field.getModifiers())) { // static属性不支持
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static fields: " + field);
               }
               return;
            }
            boolean required = determineRequiredStatus(ann); // 是否必须
           	// 添加AutowiredFieldElement
            currElements.add(new AutowiredFieldElement(field, required));
         }
      });

      // 在所有声明的方法上执行回调
      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         // 桥接方法见：https://blog.csdn.net/mhmyqn/article/details/47342577
         // 找出被桥接的原始方法
         Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
         // 检查参数类型和返回值类型是否相同
         if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) { 
            return;
         }
         AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod); // 获取注解属性
         if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
            if (Modifier.isStatic(method.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static methods: " + method);
               }
               return;
            }
            if (method.getParameterCount() == 0) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation should only be used on methods with parameters: " +
                        method);
               }
            }
            boolean required = determineRequiredStatus(ann); // 是否必须
            // 找到方法对应的属性描述符
            PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz); 
            // 添加AutowiredMethodElement
            currElements.add(new AutowiredMethodElement(method, required, pd));
         }
      });

      elements.addAll(0, currElements);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);

   return new InjectionMetadata(clazz, elements);
}
// AutowiredAnnotationBeanPostProcessor类
// 在ao上找出指定注解，并返回其属性
private AnnotationAttributes findAutowiredAnnotation(AccessibleObject ao) {
	if (ao.getAnnotations().length > 0) {  // autowiring annotations have to be local
    // @Autowaired @Value @Inject
		for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) { 
			AnnotationAttributes attributes = AnnotatedElementUtils.getMergedAnnotationAttributes(ao, type);
			if (attributes != null) {
				return attributes;
			}
		}
	}
	return null;
}
// AutowiredAnnotationBeanPostProcessor类
// 确定是否是必须的依赖
protected boolean determineRequiredStatus(AnnotationAttributes ann) {
	// 不包含required属性，或者包含该属性，并且值为true，则该依赖是必须的
	return (!ann.containsKey(this.requiredParameterName) ||
			this.requiredParameterValue == ann.getBoolean(this.requiredParameterName));
}
```

在找出装配的元数据后，会调用InjectionMetadata.checkConfigMembers方法。

```java
// InjectionMetadata类
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
   Set<InjectedElement> checkedElements = new LinkedHashSet<>(this.injectedElements.size());
   for (InjectedElement element : this.injectedElements) {
      Member member = element.getMember();
      if (!beanDefinition.isExternallyManagedConfigMember(member)) {
         beanDefinition.registerExternallyManagedConfigMember(member);
         checkedElements.add(element);
         if (logger.isTraceEnabled()) {
            logger.trace("Registered injected element on class [" + this.targetClass.getName() + "]: " + element);
         }
      }
   }
   this.checkedElements = checkedElements;
}
```

##### 2.依赖注入

然后进入到AbstractAutowireCapableBeanFactory.populateBean方法:

```java
// AbstractAutowireCapableBeanFactory类
// 填充属性
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase(阶段) for null instance.
         return;
      }
   }

  // 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改
  // bean 状态的机会。关于这段后置引用，官方的解释是：让用户可以自定义属性注入。比如用户实现一
  // 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过
  // postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。当然，如果无
  // 特殊需求，直接使用配置中的信息注入即可。另外，Spring 并不建议大家直接实现
  // InstantiationAwareBeanPostProcessor 接口，如果想实现这种类型的后置处理器，更建议
  // 通过继承 InstantiationAwareBeanPostProcessorAdapter 抽象类实现自定义后置处理器。
   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的BeanPostProcessor
           	// 的处理。一般返回true
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }

   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
   // 根据名称或类型找到所有属性值
   if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // Add property values based on autowire by name if applicable.
      // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
      if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // Add property values based on autowire by type if applicable.
      // 通过类型装配
      if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors(); // true
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

  // 这里又是一种后置处理，用于在 Spring 填充属性到 bean 对象前，对属性的值进行相应的处理(注解处理)，
  // 比如可以修改某些属性的值。这时注入到 bean 中的值就不是配置文件中的内容了，
  // 而是经过后置处理器修改后的内容
   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 对属性进行后置处理
            // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
            // 对采用 @Autowired、@Value 注解的依赖进行设值。如果使用@Resource注解，则是CommonAnnotationBeanPostProcessor(注意)
            // 这里实现了注解的处理和自动装配
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) { // pvsToUse==null，表示应用自定义的postProcessPropertyValues实现
               if (filteredPds == null) {
                  // 过滤不需要依赖检查的属性
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               // 对属性进行后置处理
               // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
               // 对采用 @Autowired、@Value 注解的依赖进行设值。如果使用@Resource注解，则是CommonAnnotationBeanPostProcessor(注意)
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }
   // 应用属性值到 bean 对象中
   if (pvs != null) {
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

由于AutowiredAnnotationBeanPostProcessor实现了InstantiationAwareBeanPostProcessor接口，所以在populateBean方法中首先会调用其postProcessAfterInstantiation方法，这里并没有特殊逻辑：

```java
// InstantiationAwareBeanPostProcessorAdapter类
public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
   return true; // 没有做什么事
}
```

然后在属性填充之前，即applyPropertyValues调用之前，会调用AutowiredAnnotationBeanPostProcessor的postProcessProperties方法对属性或方法上的注解(@Autowaired @Value @Inject)进行依赖注入。

```java
PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
```

```java
// AutowiredAnnotationBeanPostProcessor类
// 属性和方法的注解处理(@Autowaired @Value @Inject)，依赖注入
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
   // 委托给InjectionMetadata对象来完成属性和方法注入
   // bean对应的InjectionMetadata(前面在调用AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition方法时已缓存)
   InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs); 
   try {
      metadata.inject(bean, beanName, pvs); // 注入
   }
   catch (BeanCreationException ex) {
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
   }
   return pvs;
}
```

AutowiredAnnotationBeanPostProcessor委托给InjectionMetadata来完成@Autowaired @Value @Inject注解的属性和方法的依赖注入。

```java
// InjectionMetadata类
// @Autowaired @Value @Inject注解的属性或方法注入
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
   Collection<InjectedElement> checkedElements = this.checkedElements;
   Collection<InjectedElement> elementsToIterate =
         (checkedElements != null ? checkedElements : this.injectedElements);
   if (!elementsToIterate.isEmpty()) {
      for (InjectedElement element : elementsToIterate) {
         if (logger.isTraceEnabled()) {
            logger.trace("Processing injected element of bean '" + beanName + "': " + element);
         }
         // AutowiredFieldElement/AutowiredMethodElement注入
         element.inject(target, beanName, pvs);
      }
   }
}
```

###### (1) AutowiredFieldElement处理

```java
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {

   private final boolean required;

   private volatile boolean cached = false;

   @Nullable
   private volatile Object cachedFieldValue; // DependencyDescriptor

   public AutowiredFieldElement(Field field, boolean required) {
      super(field, null);
      this.required = required;
   }

   @Override
   protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
      Field field = (Field) this.member; // 待注入的属性
      Object value;
      if (this.cached) {
         // 如果已缓存，直接解析参数即可
         value = resolvedCachedArgument(beanName, this.cachedFieldValue); 
      }
      else {
         // 依赖描述符
         DependencyDescriptor desc = new DependencyDescriptor(field, this.required); 
         desc.setContainingClass(bean.getClass()); // 设置包含该desc的类
         Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
         Assert.state(beanFactory != null, "No BeanFactory available");
         TypeConverter typeConverter = beanFactory.getTypeConverter(); // 类型转换器
         try {
            // 依赖解析
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
         }
         catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
         }
         synchronized (this) {
            if (!this.cached) { // 未缓存
               if (value != null || this.required) {
                  this.cachedFieldValue = desc;
                  registerDependentBeans(beanName, autowiredBeanNames); // 注册依赖关系
                  if (autowiredBeanNames.size() == 1) {
                     String autowiredBeanName = autowiredBeanNames.iterator().next();
                     if (beanFactory.containsBean(autowiredBeanName) &&
                           beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                        this.cachedFieldValue = new ShortcutDependencyDescriptor(
                              desc, autowiredBeanName, field.getType());
                     }
                  }
               }
               else {
                  this.cachedFieldValue = null;
               }
               this.cached = true;
            }
         }
      }
      if (value != null) {
         ReflectionUtils.makeAccessible(field);
         // 设置Field值，注入
         field.set(bean, value);
      }
   }
}
// Resolve the specified cached method argument or field value. field值或方法参数解析
@Nullable
private Object resolvedCachedArgument(@Nullable String beanName, @Nullable Object cachedArgument) {
	if (cachedArgument instanceof DependencyDescriptor) {
		DependencyDescriptor descriptor = (DependencyDescriptor) cachedArgument;
		Assert.state(this.beanFactory != null, "No BeanFactory available");
		// 解析依赖
		return this.beanFactory.resolveDependency(descriptor, beanName, null, null);
	}
	else {
		return cachedArgument;
	}
}
```

###### (2) AutowiredMethodElement处理

```java
private class AutowiredMethodElement extends InjectionMetadata.InjectedElement {

   private final boolean required;

   private volatile boolean cached = false;

   @Nullable
   private volatile Object[] cachedMethodArguments;

   public AutowiredMethodElement(Method method, boolean required, @Nullable PropertyDescriptor pd) {
      super(method, pd);
      this.required = required;
   }

   @Override
   protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
      if (checkPropertySkipping(pvs)) { // 根据显式配置的属性，检查是否跳过方法的依赖注入
         return;
      }
      Method method = (Method) this.member;
      Object[] arguments;
      if (this.cached) { // 如果已缓存，直接解析方法参数的依赖，然后进行方法反射注入
         // Shortcut for avoiding synchronization...
         arguments = resolveCachedArguments(beanName);
      }
      else {
         Class<?>[] paramTypes = method.getParameterTypes(); // 方法参数类型数组
         arguments = new Object[paramTypes.length];
         DependencyDescriptor[] descriptors = new DependencyDescriptor[paramTypes.length];
         Set<String> autowiredBeans = new LinkedHashSet<>(paramTypes.length);
         Assert.state(beanFactory != null, "No BeanFactory available");
         TypeConverter typeConverter = beanFactory.getTypeConverter();
         for (int i = 0; i < arguments.length; i++) {
            MethodParameter methodParam = new MethodParameter(method, i);
            DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
            currDesc.setContainingClass(bean.getClass());
            descriptors[i] = currDesc;
            try {
               // 依赖Bean解析
               Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
               if (arg == null && !this.required) {
                  arguments = null;
                  break;
               }
               arguments[i] = arg;
            }
            catch (BeansException ex) {
               throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(methodParam), ex);
            }
         }
         synchronized (this) {
            if (!this.cached) { // 未缓存
               if (arguments != null) {
                  Object[] cachedMethodArguments = new Object[paramTypes.length];
                  System.arraycopy(descriptors, 0, cachedMethodArguments, 0, arguments.length);
                  registerDependentBeans(beanName, autowiredBeans);  // 注册依赖关系
                  if (autowiredBeans.size() == paramTypes.length) {
                     Iterator<String> it = autowiredBeans.iterator();
                     for (int i = 0; i < paramTypes.length; i++) {
                        String autowiredBeanName = it.next();
                        if (beanFactory.containsBean(autowiredBeanName) &&
                              beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
                           cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
                                 descriptors[i], autowiredBeanName, paramTypes[i]);
                        }
                     }
                  }
                  this.cachedMethodArguments = cachedMethodArguments;
               }
               else {
                  this.cachedMethodArguments = null;
               }
               this.cached = true;
            }
         }
      }
      if (arguments != null) {
         try {
            ReflectionUtils.makeAccessible(method);
            // 反射注入依赖
            method.invoke(bean, arguments);
         }
         catch (InvocationTargetException ex) {
            throw ex.getTargetException();
         }
      }
   }

   // 解析缓存参数
   @Nullable
   private Object[] resolveCachedArguments(@Nullable String beanName) {
      Object[] cachedMethodArguments = this.cachedMethodArguments;
      if (cachedMethodArguments == null) {
         return null;
      }
      Object[] arguments = new Object[cachedMethodArguments.length];
      for (int i = 0; i < arguments.length; i++) {
         // 依次解析依赖
         arguments[i] = resolvedCachedArgument(beanName, cachedMethodArguments[i]);
      }
      return arguments;
   }
}
```

AutowiredFieldElement和AutowiredMethodElement处理的流程比较简单，直接看代码注释即可。

### 三、总结

至此，`<context:component-scan/>`配置的解析与`@Autowired`注解的属性与方法的值注入过程分析到此结束。可见，必须配置`<context:component-scan/>`(`<context:annotation-config>`也可)才能处理BeanDefinition中的@Autowired @Value @Inject注解的方法和属性的依赖注入问题。

此外`<context:component-scan/>`配置处理的过程，也包括了@Component注解的类的BeanDefinition的扫描和注册到BeanFactory的过程。

### 参考文献

- [Spring IoC容器源码分析](https://xuanjian1992.top/2019/07/14/Spring-IoC%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
- [ClassPathBeanDefinitionScanner](https://docs.spring.io/spring/docs/5.0.6.RELEASE/javadoc-api/org/springframework/context/annotation/ClassPathBeanDefinitionScanner.html)
- [ClassPathScanningCandidateComponentProvider](https://docs.spring.io/spring/docs/5.0.6.RELEASE/javadoc-api/org/springframework/context/annotation/ClassPathScanningCandidateComponentProvider.html)
- [AutowiredAnnotationBeanPostProcessor](https://docs.spring.io/spring/docs/5.0.6.RELEASE/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)

