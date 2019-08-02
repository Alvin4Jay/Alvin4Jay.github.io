---
layout:     post
title:      ConfigurationClassPostProcessor源码分析
subtitle:   ConfigurationClassPostProcessor/ConfigurationClassParser/ConfigurationClassBeanDefinitionReader/ConfigurationClassEnhancer
date:       2019-08-02
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
---

# ConfigurationClassPostProcessor源码分析

 ConfigurationClassPostProcessor的类继承图如下:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/ConfigurationClassPostProcessor-1.jpg)

可见ConfigurationClassPostProcessor实现了BeanDefinitionRegistryPostProcessor、BeanFactoryPostProcessor和PriorityOrdered这三个接口。

ConfigurationClassPostProcessor是在Spring应用上下文启动的时候注册到应用上下文的。当在xml配置文件中配置了`<context:annotation-config/>`和`<context:component-scan/>`元素，于是在解析xml文件的时候就会向BeanFactory注册ConfigurationClassPostProcessor BeanDefinition。此外，使用AnnotationConfigApplicationContext时，其内部使用的AnnotatedBeanDefinitionReader初始化时也会注册ConfigurationClassPostProcessor BeanDefinition。以上几种情况，注册ConfigurationClassPostProcessor时，最终调用的方法是AnnotationConfigUtils.registerAnnotationConfigProcessors(BeanDefinitionRegistry registry)。

由于ConfigurationClassPostProcessor是BeanFactoryPostProcessor实现，因此该后置处理器的回调方法会在BeanDefinition加载、注册到BeanFactory之后调用，主要用于处理@Configuration配置类。例如下面的配置类：

```java
@Configuration
@PropertySource("classpath:props.properties") // 导入属性源
@ComponentScan(basePackages = {"configuration.annotation"}) // 测试组件扫描
@Import(TeacherImportSelector.class) // 导入ImportSelector实现
public class ClassA extends ClassB implements ClassC { // 配置类 继承超类、实现接口
	
   // 内部类
   @Import(ImportBeanDefinitionRegistrarTest.class) // 导入ImportBeanDefinitionRegistrar实现
   public class ClassE extends ClassF {

      @Bean
      public Date date() { // @Bean方法
         return new Date();
      }

   }

   @Bean
   public Student student() {
      return new Student();
   }

   @Bean
   public String aa() {
      student();
      return "aa";
   }

}
// 配置类
@Configuration
class ClassB {

   @Bean
   public String helloWorld() {
      return "hello, world";
   }

}
// 测试 接口@Bean方法
interface ClassC {

   @Bean
   default int defaultMethod() {
      return 0;
   }

}
// 测试 类@Bean方法
class ClassF {

   @Bean
   public String sayHi() {
      return "hi";
   }

}
// 测试ImportSelector实现
class TeacherImportSelector implements ImportSelector {
   @Override
   public String[] selectImports(AnnotationMetadata importingClassMetadata) {
      return new String[]{Teacher.class.getName(), Student.class.getName()};
   }
}
// 测试ImportBeanDefinitionRegistrar实现
class ImportBeanDefinitionRegistrarTest implements ImportBeanDefinitionRegistrar {
   @Override
   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      RootBeanDefinition beanDefinition = new RootBeanDefinition(Driver.class);
      BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);
   }
}
```

引导类：

```java
public class AllConfigTest {
   public static void main(String[] args) {
      AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
			// 配置类注册
      context.register(ClassA.class);
			// 刷新容器
      context.refresh();
			// 获取Bean实例
      context.getBean("date");
      context.getBean("helloWorld");
      context.getBean("sayHi");
      context.getBean(Driver.class);
      System.out.println(context.getBean("user"));
      context.getBean("student"); 
      context.getBean("configuration.annotation.Teacher");
   }
}
```

对于以上的配置类，当它注册到BeanFactory时会被ConfigurationClassPostProcessor处理。下面开始详细分析ConfigurationClassPostProcessor的处理逻辑。

### 一、配置类处理入口postProcessBeanDefinitionRegistry

```java
// 根据配置类的内容，注册BeanDefinitions到BeanRegistry
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
   int registryId = System.identityHashCode(registry);
   if (this.registriesPostProcessed.contains(registryId)) { // 判断是否已经处理过
      throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
   }
   if (this.factoriesPostProcessed.contains(registryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
   }
   this.registriesPostProcessed.add(registryId);

   processConfigBeanDefinitions(registry); // 处理入口
}
```

配置类的处理入口在postProcessBeanDefinitionRegistry方法，具体在processConfigBeanDefinitions方法。

```java
// 根据已注册到BeanFactory的@Configuration配置类BeanDefinition，构建配置类模型集合，并根据配置类模型
// 注册BeanDefinition，如@Bean注解方法、@ImportResource注解处理等。
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
   List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
   String[] candidateNames = registry.getBeanDefinitionNames(); // 取出容器中已经存在的所有bean定义的名字

   for (String beanName : candidateNames) {
      BeanDefinition beanDef = registry.getBeanDefinition(beanName);
      // 判断是否已经处理过
      if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
            ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
         if (logger.isDebugEnabled()) {
            logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
         }
      }
      // 判断每个bean定义是否是配置类(被@Configuration，@Component，@ComponentScan，@Import，
      // @ImportResource标记的),如果是则会给这个bean definition增加一个属性，避免重复解析
      else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
         configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
      }
   }

   // Return immediately if no @Configuration classes were found
   if (configCandidates.isEmpty()) { // 配置类列表为空，直接返回
      return;
   }

   // Sort by previously determined @Order value, if applicable 排序
   configCandidates.sort((bd1, bd2) -> {
      int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition()); // 获取order，默认LOWEST_PRECEDENCE
      int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
      return Integer.compare(i1, i2);
   });

   // Detect any custom bean name generation strategy supplied through the enclosing application context
   // 设置自定义的BeanNameGenerator
   SingletonBeanRegistry sbr = null;
   if (registry instanceof SingletonBeanRegistry) {
      sbr = (SingletonBeanRegistry) registry;
      if (!this.localBeanNameGeneratorSet) {
         BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
         if (generator != null) {
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
         }
      }
   }

   if (this.environment == null) { // 设置环境
      this.environment = new StandardEnvironment();
   }

   // Parse each @Configuration class 解析每个@Configuration注解的类，
   // 用于将@Configuration注解的类转换成ConfigurationClass，供ConfigurationClassBeanDefinitionReader注册BeanDefinition。
   ConfigurationClassParser parser = new ConfigurationClassParser(
         this.metadataReaderFactory, this.problemReporter, this.environment,
         this.resourceLoader, this.componentScanBeanNameGenerator, registry);

   Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
   Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
   do {
      parser.parse(candidates); // 配置类解析入口
      parser.validate(); // 校验

      // 得到配置类模型集合
      Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
      configClasses.removeAll(alreadyParsed); // 去除已解析的

      // Read the model and create bean definitions based on its content 读取配置类模型
      if (this.reader == null) {
         this.reader = new ConfigurationClassBeanDefinitionReader(
               registry, this.sourceExtractor, this.resourceLoader, this.environment,
               this.importBeanNameGenerator, parser.getImportRegistry());
      }
      this.reader.loadBeanDefinitions(configClasses); // 根据configClasses加载BeanDefinitions并注册
      alreadyParsed.addAll(configClasses); // configClasses添加到已处理的集合

      candidates.clear();
      if (registry.getBeanDefinitionCount() > candidateNames.length) { // 有新的BeanDefinition新增
         String[] newCandidateNames = registry.getBeanDefinitionNames(); // 新的bean names集合
         Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames)); // 旧的bean names集合
         Set<String> alreadyParsedClasses = new HashSet<>(); // 已经处理的类
         for (ConfigurationClass configurationClass : alreadyParsed) {
            alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
         }
         for (String candidateName : newCandidateNames) {
            if (!oldCandidateNames.contains(candidateName)) { // 新注册、新增的类
               BeanDefinition bd = registry.getBeanDefinition(candidateName);
               // 检查是否是配置类，且没有处理过
               if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                     !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                  candidates.add(new BeanDefinitionHolder(bd, candidateName)); // 如果成立，添加到候选中
               }
            }
         }
         candidateNames = newCandidateNames;
      }
   }
   while (!candidates.isEmpty()); // 如果检查得到的新增的配置类集合不为空，继续处理

   // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
   if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
      sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry()); // 注册ImportStack
   }

   if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
      // Clear cache in externally provided MetadataReaderFactory; this is a no-op
      // for a shared cache since it'll be cleared by the ApplicationContext.
      ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache(); // 清理缓存
   }
}
```

上面processConfigBeanDefinitions方法中判断BeanFactory中的BeanDefinition是否是配置类的逻辑如下：

```java
ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)
```

```java
// 判断指定的BeanDefinition是否是配置类
public static boolean checkConfigurationClassCandidate(
      BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

   String className = beanDef.getBeanClassName();
   if (className == null || beanDef.getFactoryMethodName() != null) {
      return false;
   }

   AnnotationMetadata metadata;
   if (beanDef instanceof AnnotatedBeanDefinition &&
         className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
      // Can reuse the pre-parsed metadata from the given BeanDefinition... AnnotationMetadata已解析
      metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
   }
   else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
      // Check already loaded Class if present... 如果类已加载，使用StandardAnnotationMetadata
      // since we possibly can't even load the class file for this Class.
      Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
      metadata = new StandardAnnotationMetadata(beanClass, true);
   }
   else {
      try {
         MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
         metadata = metadataReader.getAnnotationMetadata(); // ASM实现
      }
      catch (IOException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Could not find class file for introspecting configuration annotations: " +
                  className, ex);
         }
         return false;
      }
   }

   if (isFullConfigurationCandidate(metadata)) { // 是否完全模式，注解@Configuration
      beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
   }
   else if (isLiteConfigurationCandidate(metadata)) { // 轻量模式
      beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
   }
   else {
      return false;
   }

   // It's a full or lite configuration candidate... Let's determine the order value, if any.
   Integer order = getOrder(metadata); // 获取order
   if (order != null) {
      beanDef.setAttribute(ORDER_ATTRIBUTE, order); // 设置order属性
   }

   return true;
}
```

可见配置类分为完全模式和轻量模式两种：

- 判断是否是完全模式的配置类(CGLIB增强)

  ```java
  public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
     return metadata.isAnnotated(Configuration.class.getName());
  }
  ```

  类上注解了@Configuration注解的类就是完全模式的配置类。

- 判断是否是轻量模式的配置类(无CGLIB增强)

  ```java
  public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
     // Do not consider an interface or an annotation...
     if (metadata.isInterface()) {
        return false;
     }
  
     // Any of the typical annotations found? @Component @ComponentScan @Import @ImportResource
     for (String indicator : candidateIndicators) {
        if (metadata.isAnnotated(indicator)) {
           return true;
        }
     }
  
     // Finally, let's look for @Bean methods... 查找是否有@Bean注解的方法
     try {
        return metadata.hasAnnotatedMethods(Bean.class.getName());
     }
     catch (Throwable ex) {
        if (logger.isDebugEnabled()) {
           logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
        }
        return false;
     }
  }
  ```

  可见配置类注解了@Component、 @ComponentScan 、@Import、@ImportResource或者存在@Bean注解的方法，则是轻量模式的配置类。

然后走到下面这句代码：

```java
parser.parse(candidates); // 解析入口
```

到这里将会由ConfigurationClassParser这个类执行真正的配置类解析动作。

```java
// ConfigurationClassParser类
public void parse(Set<BeanDefinitionHolder> configCandidates) {
   for (BeanDefinitionHolder holder : configCandidates) { // 遍历每个BeanDefinition并解析
      BeanDefinition bd = holder.getBeanDefinition(); // 获取BeanDefinition
      try {
         // 解析
         if (bd instanceof AnnotatedBeanDefinition) {
            parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
         }
         else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
            parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
         }
         else {
            parse(bd.getBeanClassName(), holder.getBeanName());
         }
      }
      catch (BeanDefinitionStoreException ex) {
         throw ex;
      }
      catch (Throwable ex) {
         throw new BeanDefinitionStoreException(
               "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
      }
   }

   this.deferredImportSelectorHandler.process(); // 处理DeferredImportSelector
}

protected final void parse(@Nullable String className, String beanName) throws IOException {
	Assert.notNull(className, "No bean class name for configuration class bean definition");
	MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className); // ASM
	processConfigurationClass(new ConfigurationClass(reader, beanName));
}

protected final void parse(Class<?> clazz, String beanName) throws IOException {
	processConfigurationClass(new ConfigurationClass(clazz, beanName));
}

protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
	processConfigurationClass(new ConfigurationClass(metadata, beanName));
}

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
	// 根据@Conditional注解，判断是否跳过处理
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	if (existingClass != null) {
		if (configClass.isImported()) {
			if (existingClass.isImported()) {
				existingClass.mergeImportedBy(configClass);
			}
			// Otherwise ignore new imported config class; existing non-imported class overrides it.
			return;
		}
		else {
			// Explicit bean definition found, probably replacing an import.
			// Let's remove the old one and go with the new one.
			this.configurationClasses.remove(configClass);
			this.knownSuperclasses.values().removeIf(configClass::equals);
		}
	}

	// Recursively process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass); // 获取SourceClass
	do {
		sourceClass = doProcessConfigurationClass(configClass, sourceClass); // sourceClass可能是configClass本身或其超类
	}
	while (sourceClass != null);

	this.configurationClasses.put(configClass, configClass);
}
```

`ConfigurationClassParser.doProcessConfigurationClass`方法会对配置类及其父类、父接口、内部类上的@Component、@PropertySource、@ComponentScan、@Import、@ImportResource、@Bean方法等进行处理。下面按照顺序依次分析这些内容。

```java
// ConfigurationClassParser
// 构建一个完整的ConfigurationClass模型，不断读取找到的SourceClass
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
      throws IOException {
	 // 处理注解@Component
   if (configClass.getMetadata().isAnnotated(Component.class.getName())) { 
      // Recursively process any member (nested) classes first 先递归处理任何成员内部(嵌套)类
      processMemberClasses(configClass, sourceClass);
   }

   // Process any @PropertySource annotations 处理配置类上的@PropertySource
   // 获取所有@PropertySource注解的属性
   for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), PropertySources.class,
         org.springframework.context.annotation.PropertySource.class)) {
      if (this.environment instanceof ConfigurableEnvironment) {
         processPropertySource(propertySource);
      }
      else {
         logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
               "]. Reason: Environment must implement ConfigurableEnvironment");
      }
   }

   // Process any @ComponentScan annotations 处理配置类上的@ComponentScan注解
   Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
         sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
   if (!componentScans.isEmpty() &&
         // 检查是否跳过扫描
         !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
      for (AnnotationAttributes componentScan : componentScans) {
         // The config class is annotated with @ComponentScan -> perform the scan immediately 扫描BeanDefinition
         Set<BeanDefinitionHolder> scannedBeanDefinitions =
               this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
         // Check the set of scanned definitions for any further config classes and parse recursively if needed
         for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
            BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
            if (bdCand == null) {
               bdCand = holder.getBeanDefinition();
            }
            // 判断指定的BeanDefinition是否是配置类，如果是，则递归解析扫描出来的该BeanDefinition
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
               parse(bdCand.getBeanClassName(), holder.getBeanName());
            }
         }
      }
   }

   // Process any @Import annotations 处理配置类上的@Import注解
   processImports(configClass, sourceClass, getImports(sourceClass), true);

   // Process any @ImportResource annotations 处理配置类上的@ImportResource注解
   AnnotationAttributes importResource =
         AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
   if (importResource != null) {
      String[] resources = importResource.getStringArray("locations");
      Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
      for (String resource : resources) {
         String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
         // 添加到configClass，记录资源与BeanDefinitionReader的对应关系
         configClass.addImportedResource(resolvedResource, readerClass);
      }
   }

   // Process individual @Bean methods 处理配置类中的@Bean注解方法
   Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
   for (MethodMetadata methodMetadata : beanMethods) {
      // 记录BeanMethod到configClass
      configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
   }

   // Process default methods on interfaces 处理sourceClass配置类实现的接口上的默认方法
   processInterfaces(configClass, sourceClass);

   // Process superclass, if any  处理超类
   if (sourceClass.getMetadata().hasSuperClass()) {
      String superclass = sourceClass.getMetadata().getSuperClassName();
      if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
         this.knownSuperclasses.put(superclass, configClass);
         // Superclass found, return its annotation metadata and recurse 递归处理超类
         return sourceClass.getSuperClass();
      }
   }

   // No superclass -> processing is complete 没有超类，处理完成
   return null;
}
```

### 二、配置类@Component注解处理

```java
// doProcessConfigurationClass方法局部
if (configClass.getMetadata().isAnnotated(Component.class.getName())) { // 处理注解@Component
   // Recursively process any member (nested) classes first 先递归处理任何成员内部(嵌套)类
   processMemberClasses(configClass, sourceClass);
}
```

对于注解了@Component的配置类，先递归处理其成员内部(嵌套)类。

```java
// Register member (nested) classes that happen to be configuration classes themselves.
// 处理configClass的内部配置类
private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
   // 获取sourceClass的内部成员类或者接口(不包含继承的)
   Collection<SourceClass> memberClasses = sourceClass.getMemberClasses();
   if (!memberClasses.isEmpty()) {
      List<SourceClass> candidates = new ArrayList<>(memberClasses.size());
      for (SourceClass memberClass : memberClasses) {
         // 如果memberClass是配置类(Full/Lite)，且memberClass类名与configClass不相等，则添加到候选列表
         if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
               !memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
            candidates.add(memberClass);
         }
      }
      OrderComparator.sort(candidates); // 排序
      for (SourceClass candidate : candidates) {
         // 循环导入抛出异常
         if (this.importStack.contains(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
         }
         else {
            this.importStack.push(configClass);
            try {
               // 将SourceClass转换为ConfigurationClass继续处理
               processConfigurationClass(candidate.asConfigClass(configClass));
            }
            finally {
               this.importStack.pop();
            }
         }
      }
   }
}
```

对于成员内部配置类，处理逻辑是递归调用processConfigurationClass方法，进行内部配置类的解析，比如例子中的`ClassE`。

### 三、配置类@PropertySources注解处理

```java
// doProcessConfigurationClass方法局部
// Process any @PropertySource annotations 处理配置类上的@PropertySource
// 获取所有@PropertySource注解的属性
for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), PropertySources.class,
      org.springframework.context.annotation.PropertySource.class)) {
   if (this.environment instanceof ConfigurableEnvironment) {
      processPropertySource(propertySource); // 处理
   }
   else {
      logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
            "]. Reason: Environment must implement ConfigurableEnvironment");
   }
}
```

首先获取所有@PropertySource注解的属性AnnotationAttributes数组，然后根据AnnotationAttributes依次处理每个@PropertySource注解。

```java
// 处理@PropertySource注解
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
   String name = propertySource.getString("name"); // 属性源名
   if (!StringUtils.hasLength(name)) {
      name = null;
   }
   String encoding = propertySource.getString("encoding"); // 编码
   if (!StringUtils.hasLength(encoding)) {
      encoding = null;
   }
   String[] locations = propertySource.getStringArray("value"); // 属性源文件位置
   Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
   // 资源找不到，是否忽略
   boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");

   // PropertySource工厂
   Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
   PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
         DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));

   for (String location : locations) {
      try {
         String resolvedLocation = this.environment.resolveRequiredPlaceholders(location); // 解析占位符
         Resource resource = this.resourceLoader.getResource(resolvedLocation); // 获取资源
         // 创建并添加PropertySource
         addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
      }
      catch (IllegalArgumentException | FileNotFoundException | UnknownHostException ex) {
         // Placeholders not resolvable or resource not found when trying to open it
         if (ignoreResourceNotFound) {
            if (logger.isInfoEnabled()) {
               logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
            }
         }
         else {
            throw ex;
         }
      }
   }
}

// 添加属性源
private void addPropertySource(PropertySource<?> propertySource) {
   String name = propertySource.getName();
   MutablePropertySources propertySources = ((ConfigurableEnvironment) this.environment).getPropertySources();

   if (this.propertySourceNames.contains(name)) { // 已存在对应名称的属性源
      // We've already added a version, we need to extend it
      PropertySource<?> existing = propertySources.get(name);
      if (existing != null) {
         PropertySource<?> newSource = (propertySource instanceof ResourcePropertySource ?
               ((ResourcePropertySource) propertySource).withResourceName() : propertySource);
         if (existing instanceof CompositePropertySource) { // 如果是合成的属性源
            ((CompositePropertySource) existing).addFirstPropertySource(newSource); // propertySource放到第一位
         }
         else {
            if (existing instanceof ResourcePropertySource) {
               existing = ((ResourcePropertySource) existing).withResourceName();
            }
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(newSource);
            composite.addPropertySource(existing);
            propertySources.replace(name, composite); // 替换已存在的属性源
         }
         return;
      }
   }

   if (this.propertySourceNames.isEmpty()) {
      propertySources.addLast(propertySource);
   }
   else {
      // 第一个处理的属性源
      String firstProcessed = this.propertySourceNames.get(this.propertySourceNames.size() - 1);
      propertySources.addBefore(firstProcessed, propertySource); // propertySource放在最前面
   }
   this.propertySourceNames.add(name);
}
```

### 四、配置类@ComponentScan注解处理

```java
// doProcessConfigurationClass方法局部
// Process any @ComponentScan annotations 处理配置类上的@ComponentScan注解
Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
if (!componentScans.isEmpty() &&
      // 检查是否跳过扫描
      !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
   for (AnnotationAttributes componentScan : componentScans) {
      // The config class is annotated with @ComponentScan -> perform the scan immediately 扫描BeanDefinition
      Set<BeanDefinitionHolder> scannedBeanDefinitions =
            this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
      // Check the set of scanned definitions for any further config classes and parse recursively if needed
      for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
         BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
         if (bdCand == null) {
            bdCand = holder.getBeanDefinition();
         }
         // 判断指定的BeanDefinition是否是配置类，如果是，则递归解析扫描出来的该BeanDefinition
         if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
            parse(bdCand.getBeanClassName(), holder.getBeanName());
         }
      }
   }
}
```

对于@ComponentScan注解，首先依次获取这些注解的AnnotationAttributes集合，然后针对每个@ComponentScan注解，使用ComponentScanAnnotationParser进行BeanDefinition扫描与注册。然后判断得到的每个BeanDefinition是否是配置类，如果是，则递归解析这些配置类。

### 五、配置类@Import注解处理

```java
// doProcessConfigurationClass方法局部
// Process any @Import annotations 处理配置类上的@Import注解
processImports(configClass, sourceClass, getImports(sourceClass), true);
```

首先调用getImports(sourceClass)方法获取导入类：

```java
// 获取所有由@Import注解导入的类，考虑元注解
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
   Set<SourceClass> imports = new LinkedHashSet<>();
   Set<SourceClass> visited = new LinkedHashSet<>();
   collectImports(sourceClass, imports, visited); // 收集
   return imports;
}

private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
      throws IOException {

   if (visited.add(sourceClass)) {
      // 获取所有注解对应的SourceClass并遍历
      for (SourceClass annotation : sourceClass.getAnnotations()) {
         String annName = annotation.getMetadata().getClassName(); // 注解类名
         if (!annName.startsWith("java") && !annName.equals(Import.class.getName())) { // 排除@Import注解
            collectImports(annotation, imports, visited);
         }
      }
      // 添加导入类
      imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
   }
}
```

然后调用processImports方法处理这些导入类：

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
      Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

   if (importCandidates.isEmpty()) { // 为空，直接返回
      return;
   }
   // 判断configClass是否构成链式导入
   if (checkForCircularImports && isChainedImportOnStack(configClass)) {
      this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
   }
   else {
      this.importStack.push(configClass);
      try {
         for (SourceClass candidate : importCandidates) {
            if (candidate.isAssignable(ImportSelector.class)) { // ImportSelector实现类直接处理
               Class<?> candidateClass = candidate.loadClass();
               ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
               ParserStrategyUtils.invokeAwareMethods( // 调用xxxAware回调方法
                     selector, this.environment, this.resourceLoader, this.registry);
							// 处理DeferredImportSelector
              if (selector instanceof DeferredImportSelector) { 
                  this.deferredImportSelectorHandler.handle(
                        configClass, (DeferredImportSelector) selector);
               }
               else {
                  // currentSourceClass.getMetadata(): 当前配置类的注解元数据
                  String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata()); // 调用ImportSelector.selectImports方法
                  Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                  // 递归处理
                  processImports(configClass, currentSourceClass, importSourceClasses, false);
               }
            }
           	// ImportBeanDefinitionRegistrar延迟处理
            else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) { 
               Class<?> candidateClass = candidate.loadClass();
               ImportBeanDefinitionRegistrar registrar =
                     BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
               ParserStrategyUtils.invokeAwareMethods( // 调用xxxAware回调方法
                     registrar, this.environment, this.resourceLoader, this.registry);
               // 添加到configClass，构建ConfigurationClass模型，记录ImportBeanDefinitionRegistrar与SourceClass的关系
               configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
            }
            else {
               // 如果以上都不成立，当做@Configuration处理
               // 注册导入关系
               this.importStack.registerImport(
                     currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
               processConfigurationClass(candidate.asConfigClass(configClass)); // 递归处理
            }
         }
      }
      catch (BeanDefinitionStoreException ex) {
         throw ex;
      }
      catch (Throwable ex) {
         throw new BeanDefinitionStoreException(
               "Failed to process import candidates for configuration class [" +
               configClass.getMetadata().getClassName() + "]", ex);
      }
      finally {
         this.importStack.pop();
      }
   }
}
```

对于导入类的处理，如果导入类实现了ImportSelector接口且不是DeferredImportSelector实现，则直接调用ImportSelector.selectImport方法得到importClassNames，然后进行递归处理；如果导入类实现了ImportBeanDefinitionRegistrar，则会实例化ImportBeanDefinitionRegistrar实现并添加到ConfigurationClass模型中，会在后续进行处理；其他情况下，会将导入类当做@Configuration配置类处理，进行递归调用。

### 六、配置类@ImportResource注解处理

```java
// doProcessConfigurationClass方法局部
// Process any @ImportResource annotations 处理配置类上的@ImportResource注解
AnnotationAttributes importResource =
      AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
if (importResource != null) {
   String[] resources = importResource.getStringArray("locations");
   Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
   for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      // 添加到configClass，记录资源与BeanDefinitionReader的对应关系
      configClass.addImportedResource(resolvedResource, readerClass);
   }
}
```

对于@ImportResource注解，先获取其AnnotationAttributes。然后获取其表示的资源，然后将这些资源添加到ConfigurationClass模型中，会在后续处理。

### 七、配置类@Bean方法处理

```java
// doProcessConfigurationClass方法局部
// Process individual @Bean methods 处理配置类中的@Bean注解方法
Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
for (MethodMetadata methodMetadata : beanMethods) {
   // 记录BeanMethod到configClass
   configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
}
```

首先获取@Bean方法元数据，然后添加到ConfigurationClass模型中，会在后续处理。

获取@Bean方法元数据过程如下：

```java
// @Bean注解方法处理
private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
   AnnotationMetadata original = sourceClass.getMetadata();
   // 获取@Bean方法元数据
   Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
   if (beanMethods.size() > 1 && original instanceof StandardAnnotationMetadata) {
      try {
         // 确定@Bean注解的方法声明顺序
         AnnotationMetadata asm =
               this.metadataReaderFactory.getMetadataReader(original.getClassName()).getAnnotationMetadata();
         Set<MethodMetadata> asmMethods = asm.getAnnotatedMethods(Bean.class.getName());
         if (asmMethods.size() >= beanMethods.size()) {
            Set<MethodMetadata> selectedMethods = new LinkedHashSet<>(asmMethods.size());
            for (MethodMetadata asmMethod : asmMethods) {
               for (MethodMetadata beanMethod : beanMethods) {
                  if (beanMethod.getMethodName().equals(asmMethod.getMethodName())) {
                     selectedMethods.add(beanMethod);
                     break;
                  }
               }
            }
            if (selectedMethods.size() == beanMethods.size()) {
               // All reflection-detected methods found in ASM method set -> proceed
               beanMethods = selectedMethods;
            }
         }
      }
      catch (IOException ex) {
         logger.debug("Failed to read class file via ASM for determining @Bean method order", ex);
         // No worries, let's continue with the reflection metadata we started with...
      }
   }
   return beanMethods;
}
```

### 八、处理配置类所实现接口的默认方法

```java
// doProcessConfigurationClass方法局部
// Process default methods on interfaces 处理sourceClass配置类实现的接口上的默认方法
processInterfaces(configClass, sourceClass);
```

```java
// Register default methods on interfaces implemented by the configuration class. 
// 处理该配置类实现的接口上的默认方法
private void processInterfaces(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
   for (SourceClass ifc : sourceClass.getInterfaces()) { // 获取实现的接口
     // 获取@Bean注解方法
      Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(ifc);
      for (MethodMetadata methodMetadata : beanMethods) {
         if (!methodMetadata.isAbstract()) { // 排除抽象的方法，获取默认方法
            // A default method or other concrete method on a Java 8+ interface...
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
         }
      }
      processInterfaces(configClass, ifc);
   }
}
```

对于配置类实现的接口，首先获取这些接口的@Bean注解方法，然后选出默认方法，添加到ConfigurationClass模型中，会在后续进行处理(注册BeanDefinition)。

### 九、配置类超类(父类)处理

```java
// doProcessConfigurationClass方法局部
// Process superclass, if any  处理父类
if (sourceClass.getMetadata().hasSuperClass()) {
   String superclass = sourceClass.getMetadata().getSuperClassName();
   if (superclass != null && !superclass.startsWith("java") &&
         !this.knownSuperclasses.containsKey(superclass)) {
      this.knownSuperclasses.put(superclass, configClass);
      // Superclass found, return its annotation metadata and recurse 递归处理父类
      return sourceClass.getSuperClass(); // 返回父类，继续处理
   }
}

// No superclass -> processing is complete 没有父类，处理完成
return null;
```

对于父类，doProcessConfigurationClass方法会返回父类给上层processConfigurationClass方法，继续处理父类中的配置。

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
   // ...
   // Recursively process the configuration class and its superclass hierarchy.
   SourceClass sourceClass = asSourceClass(configClass); // 获取SourceClass
   do {
      // sourceClass可能是configClass本身或父类
      sourceClass = doProcessConfigurationClass(configClass, sourceClass); 
   }
   while (sourceClass != null);

   this.configurationClasses.put(configClass, configClass);
}
```

至此，doProcessConfigurationClass方法对配置类进行解析的流程分析结束。

### 十、读取ConfigurationClass模型并注册BeanDefinitions

然后回到ConfigurationClassPostProcessor.processConfigBeanDefinitions方法的以下代码处：

```java
// ConfigurationClassPostProcessor.processConfigBeanDefinitions方法局部
do {
   parser.parse(candidates); // 解析入口
   parser.validate(); // 校验

   // 得到配置类模型集合
   Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
   configClasses.removeAll(alreadyParsed); // 去除已解析的

   // Read the model and create bean definitions based on its content 读取配置类模型
   if (this.reader == null) {
      this.reader = new ConfigurationClassBeanDefinitionReader(
            registry, this.sourceExtractor, this.resourceLoader, this.environment,
            this.importBeanNameGenerator, parser.getImportRegistry());
   }
   this.reader.loadBeanDefinitions(configClasses); // 根据configClasses加载并注册BeanDefinitions
   alreadyParsed.addAll(configClasses); // configClasses添加到已处理的集合

  	// ...省略部分代码
  
}
while (!candidates.isEmpty()); // 如果检查得到的新增的配置类集合不为空，继续处理
```

在解析完配置类之后，会得到配置类模型集合`Set<ConfigurationClass> configClasses`。接着使用ConfigurationClassBeanDefinitionReader读取配置类模型，根据配置类模型加载并注册BeanDefinitions。

```java
 this.reader.loadBeanDefinitions(configClasses); // 根据configClasses加载并注册BeanDefinitions
```

```java
// 读取配置类模型，注册相关的BeanDefinitions
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
   TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
   for (ConfigurationClass configClass : configurationModel) {
      loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
   }
}

// 注册配置类本身的BeanDefinition、@Bean注解方法处理、@ImportResource注解处理、
// ImportBeanDefinitionRegistrar实现处理
private void loadBeanDefinitionsForConfigurationClass(
      ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

   if (trackedConditionEvaluator.shouldSkip(configClass)) { // 是否跳过，如果跳过，移除相关的BeanDefinition
      String beanName = configClass.getBeanName();
      if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
         this.registry.removeBeanDefinition(beanName); // 移除BeanDefinition
      }
      this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
      return;
   }

   if (configClass.isImported()) { // 如果配置类是被导入的，则注册配置类本身的BeanDefinition
      registerBeanDefinitionForImportedConfigurationClass(configClass); 
   }
   for (BeanMethod beanMethod : configClass.getBeanMethods()) {
      loadBeanDefinitionsForBeanMethod(beanMethod); // 注册@Bean注解方法的BeanDefinition
   }

   // @ImportResource注解处理，读取BeanDefinitions
   loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
   // ImportBeanDefinitionRegistrar实现处理
   loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

根据loadBeanDefinitionsForConfigurationClass方法的逻辑，每个配置类模型的处理主要分为以下4步。

##### 1. 注册配置类本身的BeanDefinition

```java
if (configClass.isImported()) { // 如果配置类是被导入的，则注册配置类本身的BeanDefinition
   registerBeanDefinitionForImportedConfigurationClass(configClass); 
}
```

```java
// 注册配置类本身的BeanDefinition
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
   AnnotationMetadata metadata = configClass.getMetadata();
   AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

   ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
   configBeanDef.setScope(scopeMetadata.getScopeName()); // 设置scope
   String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
   AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata); // 处理公共注解

   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
   // 创建代理
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   // 注册BeanDefinition
   this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
   configClass.setBeanName(configBeanName);

   if (logger.isTraceEnabled()) {
      logger.trace("Registered bean definition for imported class '" + configBeanName + "'");
   }
}
```

##### 2. 注册@Bean注解方法的BeanDefinition

```java
for (BeanMethod beanMethod : configClass.getBeanMethods()) {
   loadBeanDefinitionsForBeanMethod(beanMethod); // 注册@Bean注解方法的BeanDefinition
}
```

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
   ConfigurationClass configClass = beanMethod.getConfigurationClass();
   MethodMetadata metadata = beanMethod.getMetadata();
   String methodName = metadata.getMethodName();

   // Do we need to mark the bean as skipped by its condition? 根据条件判断是否跳过处理
   if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
      configClass.skippedBeanMethods.add(methodName); // 记录并返回
      return;
   }
   // 如果跳过的记录里已包含该方法，直接跳过
   if (configClass.skippedBeanMethods.contains(methodName)) { 
      return;
   }
		// 获取@Bean注解属性
   AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class); 
   Assert.state(bean != null, "No @Bean annotation attributes");

   // Consider name and any aliases
   List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
   // 如果配置了name属性，则以name属性值的数组，第一个元素为beanName；否则以方法名为beanName
   String beanName = (!names.isEmpty() ? names.remove(0) : methodName); // 确定beanName

   // Register aliases even when overridden
   for (String alias : names) {
      this.registry.registerAlias(beanName, alias); // 注册别名
   }

   // Has this effectively been overridden before (e.g. via XML)?
   if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
      if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
         throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
               beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
               "' clashes with bean name for containing configuration class; please make those names unique!");
      }
      return;
   }

   // 创建方法BeanDefinition
   ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
   beanDef.setResource(configClass.getResource());
   beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

   if (metadata.isStatic()) {
      // static @Bean method 静态@Bean方法，则以配置类为beanClass，以当前方法为静态工厂方法处理
      beanDef.setBeanClassName(configClass.getMetadata().getClassName());
      beanDef.setFactoryMethodName(methodName);
   }
   else {
      // instance @Bean method 实例@Bean方法，则以配置类对应的Bean为实例工厂Bean，以当前方法为实例工厂方法处理
      beanDef.setFactoryBeanName(configClass.getBeanName());
      beanDef.setUniqueFactoryMethodName(methodName);
   }
   beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR); // 设置装配模式为构造器装配
   beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
         SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

   AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata); // 处理公共注解

   // 以下是@Bean注解相关属性处理
   Autowire autowire = bean.getEnum("autowire");
   if (autowire.isAutowire()) {
      beanDef.setAutowireMode(autowire.value());
   }

   boolean autowireCandidate = bean.getBoolean("autowireCandidate");
   if (!autowireCandidate) {
      beanDef.setAutowireCandidate(false);
   }

   String initMethodName = bean.getString("initMethod");
   if (StringUtils.hasText(initMethodName)) {
      beanDef.setInitMethodName(initMethodName); // 初始化方法
   }

   String destroyMethodName = bean.getString("destroyMethod");
   beanDef.setDestroyMethodName(destroyMethodName); // 销毁方法

   // Consider scoping 考虑作用域
   ScopedProxyMode proxyMode = ScopedProxyMode.NO;
   AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
   if (attributes != null) {
      beanDef.setScope(attributes.getString("value"));
      proxyMode = attributes.getEnum("proxyMode");
      if (proxyMode == ScopedProxyMode.DEFAULT) {
         proxyMode = ScopedProxyMode.NO;
      }
   }

   // Replace the original bean definition with the target one, if necessary
   BeanDefinition beanDefToRegister = beanDef;
   if (proxyMode != ScopedProxyMode.NO) {
      // 创建代理
      BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
            new BeanDefinitionHolder(beanDef, beanName), this.registry,
            proxyMode == ScopedProxyMode.TARGET_CLASS);
      beanDefToRegister = new ConfigurationClassBeanDefinition(
            (RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
   }

   if (logger.isTraceEnabled()) {
      logger.trace(String.format("Registering bean definition for @Bean method %s.%s()",
            configClass.getMetadata().getClassName(), beanName));
   }
   this.registry.registerBeanDefinition(beanName, beanDefToRegister); // 注册
}
```

##### 3. @ImportResource注解处理

```java
// @ImportResource注解处理，读取BeanDefinitions
loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
```

```java
private void loadBeanDefinitionsFromImportedResources(
      Map<String, Class<? extends BeanDefinitionReader>> importedResources) {
   // BeanDefinitionReader实例缓存
   Map<Class<?>, BeanDefinitionReader> readerInstanceCache = new HashMap<>();

   importedResources.forEach((resource, readerClass) -> {
      // Default reader selection necessary? 选择默认的BeanDefinitionReader
      if (BeanDefinitionReader.class == readerClass) {
         if (StringUtils.endsWithIgnoreCase(resource, ".groovy")) {
            // When clearly asking for Groovy, that's what they'll get...
            readerClass = GroovyBeanDefinitionReader.class; // groovy
         }
         else {
            // Primarily ".xml" files but for any other extension as well
            readerClass = XmlBeanDefinitionReader.class; // xmlx
         }
      }

      BeanDefinitionReader reader = readerInstanceCache.get(readerClass);
      if (reader == null) {
         try {
            // Instantiate the specified BeanDefinitionReader 实例化BeanDefinitionReader
            reader = readerClass.getConstructor(BeanDefinitionRegistry.class).newInstance(this.registry);
            // Delegate the current ResourceLoader to it if possible
            if (reader instanceof AbstractBeanDefinitionReader) {
               AbstractBeanDefinitionReader abdr = ((AbstractBeanDefinitionReader) reader);
               abdr.setResourceLoader(this.resourceLoader);
               abdr.setEnvironment(this.environment);
            }
            readerInstanceCache.put(readerClass, reader);
         }
         catch (Throwable ex) {
            throw new IllegalStateException(
                  "Could not instantiate BeanDefinitionReader class [" + readerClass.getName() + "]");
         }
      }

      // TODO SPR-6310: qualify relative path locations as done in AbstractContextLoader.modifyLocations
      reader.loadBeanDefinitions(resource); // 加载BeanDefinitions
   });
}
```

对于@ImportResource导入的资源，使用XmlBeanDefinitionReader、GroovyBeanDefinitionReader等加载、注册BeanDefinitions。

##### 4. ImportBeanDefinitionRegistrar实现处理

```java
// ImportBeanDefinitionRegistrar实现处理
loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
```

```java
// ImportBeanDefinitionRegistrar注册BeanDefinitions
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
   registrars.forEach((registrar, metadata) ->
         registrar.registerBeanDefinitions(metadata, this.registry));
}
```

对于ImportBeanDefinitionRegistrar实现，调用其registerBeanDefinitions方法执行其注册BeanDefinition逻辑。

### 十一、注册ImportStack实例

```java
// ConfigurationClassPostProcessor.processConfigBeanDefinitions方法局部
// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
   // 注册ImportStack实例
   sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry()); 
}
```

在ConfigurationClassPostProcessor.processConfigBeanDefinitions方法的最后，会注册ImportRegistry实现，即ImportStack实例到BeanFactory，以支持实现了ImportAware接口的配置类。

至此，以ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry为入口的配置类处理流程分析完毕。

### 十二、@Configuration配置类CGLIB增强

在执行完ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry方法之后，会继续执行ConfigurationClassPostProcessor.postProcessBeanFactory方法，如下所示，进行@Configuration配置类的CGLIB增强。

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   int factoryId = System.identityHashCode(beanFactory);
   if (this.factoriesPostProcessed.contains(factoryId)) {
      throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + beanFactory);
   }
   this.factoriesPostProcessed.add(factoryId);
   if (!this.registriesPostProcessed.contains(factoryId)) {
      // BeanDefinitionRegistryPostProcessor hook apparently not supported...
      // Simply call processConfigurationClasses lazily at this point then.
      processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
   }

   // @Configuration配置类CGLIB增强
   enhanceConfigurationClasses(beanFactory);
   // 此BeanPostProcessor会将实现了ImportAware接口的类注入 其导入配置类的注解元数据
   beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

```java
// 配置类CGLIB增强，使其拥有调用@Bean方法总是返回一个对象的能力(单例)
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
   Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
   for (String beanName : beanFactory.getBeanDefinitionNames()) {
      BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
      // 只增强@Configuration注解的类
      if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) { // @Configuration注解的Full完全模式的配置类
         // Spring提供的所有BeanDefinition的实现类都继承了AbstractBeanDefinition
         if (!(beanDef instanceof AbstractBeanDefinition)) {
            throw new BeanDefinitionStoreException("Cannot enhance @Configuration bean definition '" +
                  beanName + "' since it is not stored in an AbstractBeanDefinition subclass");
         }
         // 如果已经存在了一个单例实例，那么不会增强它，造成这个现象的原因是非静态的@Bean方法的返回类型
         // 是BeanDefinitionRegistryPostProcessor
         else if (logger.isInfoEnabled() && beanFactory.containsSingleton(beanName)) {
            logger.info("Cannot enhance @Configuration bean definition '" + beanName +
                  "' since its singleton instance has been created too early. The typical cause " +
                  "is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor " +
                  "return type: Consider declaring such methods as 'static'.");
         }
         configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
      }
   }
   if (configBeanDefs.isEmpty()) {
      // nothing to enhance -> return immediately
      return;
   }

   // 使用ConfigurationClassEnhancer增强配置类，使其拥有调用@Bean方法总是返回一个对象的能力
   ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
   for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
      AbstractBeanDefinition beanDef = entry.getValue();
      // If a @Configuration class gets proxied, always proxy the target class
      beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
      try {
         // Set enhanced subclass of the user-specified bean class
         Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
         if (configClass != null) {
            Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
            if (configClass != enhancedClass) {
               if (logger.isTraceEnabled()) {
                  logger.trace(String.format("Replacing bean definition '%s' existing class '%s' with " +
                        "enhanced class '%s'", entry.getKey(), configClass.getName(), enhancedClass.getName()));
               }
               beanDef.setBeanClass(enhancedClass); // 代理类替换原始的类
            }
         }
      }
      catch (Throwable ex) {
         throw new IllegalStateException("Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
      }
   }
}
```

enhanceConfigurationClasses方法中使用了ConfigurationClassEnhancer对配置类进行CGLIB增强处理。ConfigurationClassEnhancer.enhance()方法会使用CGLIB返回配置类的一个代理类，这个代理类拦截了@Bean方法，重复调用这个方法总是返回同一个对象(默认行为，除非指定@Sope)，使其拥有单例的能力。

```java
// 增强
public Class<?> enhance(Class<?> configClass, @Nullable ClassLoader classLoader) {
   if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
      if (logger.isDebugEnabled()) {
         logger.debug(String.format("Ignoring request to enhance %s as it has " +
               "already been enhanced. This usually indicates that more than one " +
               "ConfigurationClassPostProcessor has been registered (e.g. via " +
               "<context:annotation-config>). This is harmless, but you may " +
               "want check your configuration and remove one CCPP if possible",
               configClass.getName()));
      }
     	// 已经增强则直接返回
      return configClass;
   }
   Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader)); // 创建代理类
   if (logger.isTraceEnabled()) {
      logger.trace(String.format("Successfully enhanced %s; enhanced class name is: %s",
            configClass.getName(), enhancedClass.getName()));
   }
   return enhancedClass;
}

private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
	Enhancer enhancer = new Enhancer();
	enhancer.setSuperclass(configSuperClass);
	enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
	enhancer.setUseFactory(false);
	enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
	enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
	enhancer.setCallbackFilter(CALLBACK_FILTER);
	enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
	return enhancer;
}

private Class<?> createClass(Enhancer enhancer) {
	Class<?> subclass = enhancer.createClass();
	Enhancer.registerStaticCallbacks(subclass, CALLBACKS); // 注册拦截器实例
	return subclass;
}

public interface EnhancedConfiguration extends BeanFactoryAware {}
```

enhancer指定了实现接口为**EnhancedConfiguration**，此接口继承了BeanFactoryAware接口，所以就拥有了setBeanFactory()方法，因此Spring容器会将持有的BeanFactory通过此方法传递给代理对象。enhancer又配置了一个**BeanFactoryAwareGeneratorStrategy**，它会给代理类添加一个名为`$$beanFactory`并且类型为BeanFactory的字段。加上拦截器**BeanFactoryAwareMethodInterceptor**，结合这三者，BeanFactoryAwareMethodInterceptor起作用的时候就会将`$$beanFactory`赋值为当前容器，而后代理对象就拥有了Spring容器。 

```java
private static final String BEAN_FACTORY_FIELD = "$$beanFactory";

private static class BeanFactoryAwareGeneratorStrategy extends DefaultGeneratorStrategy {
	protected ClassGenerator transform(ClassGenerator cg) throws Exception {
		ClassEmitterTransformer transformer = new ClassEmitterTransformer() {
			@Override
			public void end_class() {
				// 在代理类添加 $$beanFactory Field
				declare_field(Constants.ACC_PUBLIC, BEAN_FACTORY_FIELD, Type.getType(BeanFactory.class), null);
				super.end_class();
			}
		};
		return new TransformingClassGenerator(cg, transformer);
	}
}
```

enhancer设置了一组拦截器(CALLBACKS)，每当执行代理类的一个方法都会经过拦截器，至于选择哪个拦截器是由`ConditionalCallbackFilter CALLBACK_FILTER`决定的。如下所示，最终由拦截器的isMatch()方法决定是哪个拦截器匹配生效，CALLBACKS是三个Callback实例，分别起到不同的作用。

```java
// 拦截器(方法回调)
private static final Callback[] CALLBACKS = new Callback[] {
		new BeanMethodInterceptor(),
		new BeanFactoryAwareMethodInterceptor(),
		NoOp.INSTANCE
};

// Filter: 方法拦截时选择以上的拦截器
private static final ConditionalCallbackFilter CALLBACK_FILTER = new ConditionalCallbackFilter(CALLBACKS);


private static class ConditionalCallbackFilter implements CallbackFilter {
   // 回调拦截器
   private final Callback[] callbacks;
   // 回调拦截器类型
   private final Class<?>[] callbackTypes;

   public ConditionalCallbackFilter(Callback[] callbacks) {
      this.callbacks = callbacks;
      this.callbackTypes = new Class<?>[callbacks.length];
      for (int i = 0; i < callbacks.length; i++) {
         this.callbackTypes[i] = callbacks[i].getClass();
      }
   }

   // 选择回调拦截器
   @Override
   public int accept(Method method) {
      for (int i = 0; i < this.callbacks.length; i++) {
         Callback callback = this.callbacks[i];
         // 如果callback不是ConditionalCallback，或者是ConditionalCallback，且匹配方法，则返回当前的callback进行拦截处理
         if (!(callback instanceof ConditionalCallback) || ((ConditionalCallback) callback).isMatch(method)) {
            return i;
         }
      }
      throw new IllegalStateException("No callback available for method " + method.getName());
   }

   public Class<?>[] getCallbackTypes() {
      return this.callbackTypes;
   }
}
```

 当Spring容器调用代理对象的setBeanFactory()方法时，拦截器BeanFactoryAwareMethodInterceptor匹配该方法调用，执行拦截后的操作。

```java
private static class BeanFactoryAwareMethodInterceptor implements MethodInterceptor, ConditionalCallback {

   public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
     	// 找到Field
      Field field = ReflectionUtils.findField(obj.getClass(), BEAN_FACTORY_FIELD); 
      Assert.state(field != null, "Unable to find generated BeanFactory field");
      field.set(obj, args[0]); // 设置value，注入BeanFactory实例

      if (BeanFactoryAware.class.isAssignableFrom(ClassUtils.getUserClass(obj.getClass().getSuperclass()))) {
         // 实际被代理的类如果也实现了BeanFactoryAware，则也注入BeanFactory实例
         return proxy.invokeSuper(obj, args); 
      }
      return null;
   }

  // 如果调用的方法是setBeanFactory，则使用本拦截器拦截，注入BeanFactory实例
   public boolean isMatch(Method candidateMethod) { 
      return isSetBeanFactory(candidateMethod);
   }

   public static boolean isSetBeanFactory(Method candidateMethod) {
      return (candidateMethod.getName().equals("setBeanFactory") && // 方法签名检查、声明类检查
            candidateMethod.getParameterCount() == 1 &&
            BeanFactory.class == candidateMethod.getParameterTypes()[0] &&
            BeanFactoryAware.class.isAssignableFrom(candidateMethod.getDeclaringClass()));
   }
}
```

BeanFactoryAwareMethodInterceptor.isMatch方法匹配后，执行intercept方法。此方法就是如上所说，将BeanFactory保存到代理对象的`$$beanFactory`属性，供代理对象使用，因为后面**BeanMethodInterceptor**拦截器生效的时候会用到。

如果调用目标对象的方法不是setBeanFactory方法并且是@Bean方法，则使用BeanMethodInterceptor加强(代理)此方法。值得注意的是，如果请求的bean是FactoryBean，则创建一个子类代理来拦截对getObject()的调用并返回任何缓存的bean实例。作用域代理FactoryBean是一种特殊情况，不应该进一步代理。

```java
private static class BeanMethodInterceptor implements MethodInterceptor, ConditionalCallback {

	public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs, MethodProxy cglibMethodProxy) throws Throwable {

		// 根据BEAN_FACTORY_FIELD字段，从代理类实例获取BeanFactory实例
    // enhancedConfigInstance 代理类实例
		ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance); 
    // 确定 Bean name
		String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod); 

		if (BeanAnnotationHelper.isScopedProxy(beanMethod)) { // 是否作用域代理
			// 获取作用域代理的目标bean的bean name  scopedTarget.beanName
			String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
			if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
				beanName = scopedBeanName;
			}
		}

		if (factoryContainsBean(beanFactory, BeanFactory.FACTORY_BEAN_PREFIX + beanName) &&
				factoryContainsBean(beanFactory, beanName)) {
      // 获取FactoryBean实例
			Object factoryBean = beanFactory.getBean(BeanFactory.FACTORY_BEAN_PREFIX + beanName); 
			if (factoryBean instanceof ScopedProxyFactoryBean) {
				// 对于factory bean，如果显式的设置了@Scope属性proxyMode为代理模式，则不需要进一步代理
			}
			else {
				// 生成FactoryBean代理
				return enhanceFactoryBean(factoryBean, beanMethod.getReturnType(), beanFactory, beanName);
			}
		}

		// Spring通过工厂方法创建时会保存创建对象的工厂方法，所以说第一次调用创建对象方法不会被拦截
		if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
			// 直接使用工厂方法创建对象(会在Spring容器缓存该单例对象)
			return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
		}

		// 如果不是第一次调用此方法，从容器中直接获取达到单例目的
		return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
	}

	private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
			ConfigurableBeanFactory beanFactory, String beanName) {

		boolean alreadyInCreation = beanFactory.isCurrentlyInCreation(beanName);
		try {
			if (alreadyInCreation) {
				beanFactory.setCurrentlyInCreation(beanName, false);
			}
			boolean useArgs = !ObjectUtils.isEmpty(beanMethodArgs);
			if (useArgs && beanFactory.isSingleton(beanName)) {
				for (Object arg : beanMethodArgs) {
					if (arg == null) {
						useArgs = false;
						break;
					}
				}
			}
			// 获取实例
			Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
					beanFactory.getBean(beanName));
			if (!ClassUtils.isAssignableValue(beanMethod.getReturnType(), beanInstance)) {
				if (beanInstance.equals(null)) {
					if (logger.isDebugEnabled()) {
						logger.debug(String.format("@Bean method %s.%s called as bean reference " +
								"for type [%s] returned null bean; resolving to null value.",
								beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
								beanMethod.getReturnType().getName()));
					}
					beanInstance = null;
				}
				else {
					String msg = String.format("@Bean method %s.%s called as bean reference " +
							"for type [%s] but overridden by non-compatible bean instance of type [%s].",
							beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
							beanMethod.getReturnType().getName(), beanInstance.getClass().getName());
					try {
						BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(beanName);
						msg += " Overriding bean of same name declared in: " + beanDefinition.getResourceDescription();
					}
					catch (NoSuchBeanDefinitionException ex) {
					}
					throw new IllegalStateException(msg);
				}
			}
			// 获取目前正在调用的工厂方法(可能是其他@Bean方法调用的获取当前beanInstance的动作)
			Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
			if (currentlyInvoked != null) {
				String outerBeanName = BeanAnnotationHelper.determineBeanNameFor(currentlyInvoked);
				beanFactory.registerDependentBean(beanName, outerBeanName); // 注册依赖关系
			}
			return beanInstance;
		}
		finally {
			if (alreadyInCreation) {
				beanFactory.setCurrentlyInCreation(beanName, true);
			}
		}
	}

	public boolean isMatch(Method candidateMethod) {
		return (candidateMethod.getDeclaringClass() != Object.class &&
				!BeanFactoryAwareMethodInterceptor.isSetBeanFactory(candidateMethod) &&
				BeanAnnotationHelper.isBeanAnnotated(candidateMethod));
	}
}
```

如果当前调用的工厂方法就是该@Bean方法(@Bean方法定义的Bean一般是单例，会在容器初始化时预初始化这些Bean，使用的是工厂方法初始化的方式，这是第一次调用@Bean方法创建单例)，intercept方法会进入如下的分支，直接使用工厂方法创建对象(会在Spring容器缓存该单例对象)。后续再次请求该@Bean方法对应的单例对象时，会直接从容器单例缓存中返回。

```java
// 判断method是否是当前正在调用的工厂方法
private boolean isCurrentlyInvokedFactoryMethod(Method method) { 
  // 当前正在调用的工厂方法
	Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod(); 
	return (currentlyInvoked != null && method.getName().equals(currentlyInvoked.getName()) &&
			Arrays.equals(method.getParameterTypes(), currentlyInvoked.getParameterTypes()));
}
```

```java
// Spring通过工厂方法创建对象时会保存创建对象的工厂方法(ThreadLocal)
if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
	// 直接使用工厂方法创建对象(会在Spring容器缓存该单例对象)
	return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
}


```

如果当前调用的工厂方法不是该@Bean方法(一般在容器预初始化阶段)，可能是其他@Bean方法间接依赖该@Bean方法的对象，由其他@Bean方法间接调用该@Bean方法，因此直接从容器中获取，达到单例目的。

```java
return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);

// resolveBeanReference方法局部
// 获取实例
Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
		beanFactory.getBean(beanName));
```

至此，@Configuration配置类CGLIB增强逻辑分析结束。

### 参考文章

- [【Spring源码分析】19-ConfigurationClassPostProcessor](https://blog.csdn.net/shenchaohao12321/article/details/85120597)
- [Spring启动过程分析.番外(ConfigurationClassPostProcessor)](https://blog.wangqi.love/articles/Spring/Spring%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90.%E7%95%AA%E5%A4%96(ConfigurationClassPostProcessor).html)





