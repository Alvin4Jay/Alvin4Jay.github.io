---
layout:     post
title:      Dubbo Compiler接口分析
subtitle:   Compiler/SPI
date:       2019-01-13
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo Compiler接口分析

`com.alibaba.dubbo.common.compiler.Compiler`接口是编译器SPI扩展接口，作为 Java 代码编译器，用于动态生成字节码，加速调用。接口描述如下:

```java
// 默认扩展实现为com.alibaba.dubbo.common.compiler.support.JavassistCompiler
@SPI("javassist")
public interface Compiler {
	
    /**
     * Compile java source code.
     *
     * @param code        Java source code
     * @param classLoader the class loader for the ExtensionLoader
     * @return Compiled class
     */
    Class<?> compile(String code, ClassLoader classLoader);

}
```

`Compiler`接口有三个实现，一个是`com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler`，即适配器类；其他两个是`Compiler`接口具体实现`com.alibaba.dubbo.common.compiler.support.JavassistCompiler`与`com.alibaba.dubbo.common.compiler.support.JdkCompiler`。

下面从如下的代码开始，具体解析`Compiler`接口的作用、实现与源码。

```java
ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol adaptiveExtension = loader.getAdaptiveExtension();
```

前面的文章已经解析了第一句代码的执行逻辑，下面从第二句代码的执行逻辑来详细分析`Compiler`接口。从`getAdaptiveExtension()`方法的代码调用路径可得到如下的层次结构，

```java
getAdaptiveExtension()
--createAdaptiveExtension()
----injectExtension(getAdaptiveExtensionClass())
------getAdaptiveExtensionClass()
--------getExtensionClasses() // 从spi配置文件中查找实现类上具有@Adaptive注解的类
----------loadExtensionClasses()
------------loadFile(Map<String, Class<?>> extensionClasses, String dir)
--------createAdaptiveExtensionClass() // 如果从spi配置文件中没有找到实现类上具有@Adaptive注解的类，则动态创建类
```

若在SPI接口实现类中没有`@Adaptive`注解的类，将会调用`createAdaptiveExtensionClass()`动态创建适配类。

```java
// 创建适配类的Class对象
private Class<?> createAdaptiveExtensionClass() {
    // 创建适配类的源码
    String code = createAdaptiveExtensionClassCode();
    // 获取类加载器
    ClassLoader classLoader = findClassLoader();
    // 获取Compiler的适配类
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译源码
    return compiler.compile(code, classLoader);
}
```

### 一、创建适配类的源码

创建适配类源码的过程在`createAdaptiveExtensionClassCode()`方法中。

```java
// 创建适配类的源码
private String createAdaptiveExtensionClassCode() {
        StringBuilder codeBuidler = new StringBuilder();
        Method[] methods = type.getMethods(); // 接口中的方法
        boolean hasAdaptiveAnnotation = false; // 这些方法中是否有@Adaptive注解
        for (Method m : methods) {
            if (m.isAnnotationPresent(Adaptive.class)) {
                hasAdaptiveAnnotation = true;
                break;
            }
        }
        // 没有@Adaptiv注解的方法，则不需要生成Adaptive类
        if (!hasAdaptiveAnnotation)
            throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");
		// 构造代码
        codeBuidler.append("package " + type.getPackage().getName() 
        codeBuidler.append("\nimport " + ExtensionLoader.class.getName() + ";");
        codeBuidler.append("\npublic class " + type.getSimpleName() + "$Adaptive" + " implements " + type.getCanonicalName() + " {");
		
        for (Method method : methods) {
            Class<?> rt = method.getReturnType(); // 方法返回值类型
            Class<?>[] pts = method.getParameterTypes(); // 方法参数类型
            Class<?>[] ets = method.getExceptionTypes(); // 方法异常类型
			// 获取注解对象
            Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
            StringBuilder code = new StringBuilder(512);
            if (adaptiveAnnotation == null) {
                code.append("throw new UnsupportedOperationException(\"method ")
                        .append(method.toString()).append(" of interface ")
                        .append(type.getName()).append(" is not adaptive method!\");");
            } else {
                int urlTypeIndex = -1; // 方法中类型为URL的参数索引
                for (int i = 0; i < pts.length; ++i) {
                    if (pts[i].equals(URL.class)) {
                        urlTypeIndex = i;
                        break;
                    }
                }
                // 有类型为URL的参数
                if (urlTypeIndex != -1) {
                    // 空指针检查
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                            urlTypeIndex);
                    code.append(s);

                    s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
                    code.append(s);
                }
                // 方法没有URL类型的参数
                else {
                    String attribMethod = null; // 方法参数中的属性为URL类型，得到获取该属性的方法名

                    // 找到参数的URL属性及获取URL属性的方法名
                    LBL_PTS:
                    for (int i = 0; i < pts.length; ++i) {
                        Method[] ms = pts[i].getMethods();
                        for (Method m : ms) {
                            String name = m.getName();
                            if ((name.startsWith("get") || name.length() > 3)
                                    && Modifier.isPublic(m.getModifiers())
                                    && !Modifier.isStatic(m.getModifiers())
                                    && m.getParameterTypes().length == 0
                                    && m.getReturnType() == URL.class) {
                                urlTypeIndex = i; 
                                attribMethod = name; // 方法名
                                break LBL_PTS;
                            }
                        }
                    }
                    if (attribMethod == null) {
                        throw new IllegalStateException("fail to create adative class for interface " + type.getName()
                                + ": not found url parameter or url attribute in parameters of method " + method.getName());
                    }

                    // 空指针检查
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                            urlTypeIndex, pts[urlTypeIndex].getName());
                    code.append(s);
                    s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                            urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
                    code.append(s);

                    s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
                    code.append(s);
                }
				// @Adaptive注解value中的key
                String[] value = adaptiveAnnotation.value();
                // 没有设置key，则使用 扩展点接口名的点分隔 作为key
                if (value.length == 0) {
                    char[] charArray = type.getSimpleName().toCharArray();
                    StringBuilder sb = new StringBuilder(128);
                    for (int i = 0; i < charArray.length; i++) {
                        if (Character.isUpperCase(charArray[i])) {
                            if (i != 0) {
                                sb.append(".");
                            }
                            sb.append(Character.toLowerCase(charArray[i]));
                        } else {
                            sb.append(charArray[i]);
                        }
                    }
                    value = new String[]{sb.toString()};
                }

                boolean hasInvocation = false; // 是否含有Invocation类型的参数
                for (int i = 0; i < pts.length; ++i) {
                    if ("com.alibaba.dubbo.rpc.Invocation".equals(pts[i].getName())) {
                        // 空指针检查
                        String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                        code.append(s);
                        s = String.format("\nString methodName = arg%d.getMethodName();", i);
                        code.append(s);
                        hasInvocation = true;
                        break;
                    }
                }

                String defaultExtName = cachedDefaultName; // 默认扩展名
                String getNameCode = null;
                // 根据注解value配置的key，从URL的Key名，获取对应的Value作为要Adapt成的Extension
                // 名。如果URL这些Key都没有Value，使用默认扩展defaultExtName（在接口的SPI中设
                // 定的值）
                for (int i = value.length - 1; i >= 0; --i) {
                    if (i == value.length - 1) { // 从最后的参数开始遍历
                        if (null != defaultExtName) {
                            if (!"protocol".equals(value[i]))
                                if (hasInvocation)
                                    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                                else
                                    getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                        } else {
                            if (!"protocol".equals(value[i]))
                                if (hasInvocation)
                                    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                                else
                                    getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                            else
                                getNameCode = "url.getProtocol()";
                        }
                    } else {
                        if (!"protocol".equals(value[i]))
                            if (hasInvocation)
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                        else
                            getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
                    }
                }
                code.append("\nString extName = ").append(getNameCode).append(";");
                // 检查 extName == null?
                String s = String.format("\nif(extName == null) " +
                                "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                        type.getName(), Arrays.toString(value));
                code.append(s);

                s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                        type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
                code.append(s);

                // 返回语句
                if (!rt.equals(void.class)) {
                    code.append("\nreturn ");
                }

                s = String.format("extension.%s(", method.getName());
                code.append(s);
                for (int i = 0; i < pts.length; i++) {
                    if (i != 0)
                        code.append(", ");
                    code.append("arg").append(i);
                }
                code.append(");");
            }
			// 方法签名
            codeBuidler.append("\npublic " + rt.getCanonicalName() + " " + method.getName() + "(");
            for (int i = 0; i < pts.length; i++) {
                if (i > 0) {
                    codeBuidler.append(", ");
                }
                codeBuidler.append(pts[i].getCanonicalName());
                codeBuidler.append(" ");
                codeBuidler.append("arg" + i);
            }
            codeBuidler.append(")");
            if (ets.length > 0) { // 异常
                codeBuidler.append(" throws ");
                for (int i = 0; i < ets.length; i++) {
                    if (i > 0) {
                        codeBuidler.append(", ");
                    }
                    codeBuidler.append(ets[i].getCanonicalName());
                }
            }
            codeBuidler.append(" {");
            codeBuidler.append(code.toString());
            codeBuidler.append("\n}");
        }
        codeBuidler.append("\n}");
        if (logger.isDebugEnabled()) {
            logger.debug(codeBuidler.toString());
        }
        return codeBuidler.toString();
    }
```

`createAdaptiveExtensionClassCode()`方法中会判断如果一个类中没有`@Adaptive`注解的方法，则直接抛出`IllegalStateException`异常；否则，会为有`@Adaptive`注解的方法构造代码，而没有`@Adaptive`注解的方法直接抛出`UnsupportedOperationException`异常。

对于`Protocol`接口，生成的适配类`Protocol$Adaptive`如下(生成的源码格式)

```java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
public int	aa = a
public void destroy() {throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
}
public int getDefaultPort() {throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
}
public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
if (arg1 == null) throw new IllegalArgumentException("url == null");
com.alibaba.dubbo.common.URL url = arg1;
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1);
}
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.export(arg0);
}
}	
```

### 二、获取Compiler的适配类

```java
com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
```

先看`com.alibaba.dubbo.common.compiler.Compiler`接口：

```java
@SPI("javassist")
public interface Compiler {
    Class<?> compile(String code, ClassLoader classLoader);
}
```

默认扩展实现为`javassist`，因此默认实现类是`META-INF/dubbo/internal/com.alibaba.dubbo.common.compiler.Compiler`文件中的key为`javassit`的实现类。文件内容如下：

```java
adaptive=com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler
jdk=com.alibaba.dubbo.common.compiler.support.JdkCompiler
javassist=com.alibaba.dubbo.common.compiler.support.JavassistCompiler
```

根据之前对`ExtensionFactory`的`getAdaptiveExtension()`的讲解，最终获取到的`Compiler`的`AdaptiveExtension`是`com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler`。

根据源码，首先是获取`ExtensionLoader<com.alibaba.dubbo.common.compiler.Compiler> loader`，最终的`loader`包含如下属性：

- `Class<?> type = interface com.alibaba.dubbo.common.compiler.Compiler`
- `ExtensionFactory objectFactory = AdaptiveExtensionFactory（适配类）`
  - `factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]`

之后是`loader.getAdaptiveExtension()`。在该方法中，首先会调用`createAdaptiveExtension()`创建实例，之后放入缓存，然后返回。

```java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extenstion " + type + ", cause: " + e.getMessage(), e);
    }
}

private Class<?> getAdaptiveExtensionClass() {
    // 获取ExtensionClasses和适配类
    // 其中适配类cachedAdaptiveClass如果不存在,则需要使用createAdaptiveExtensionClass()进行创建
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

在`createAdaptiveExtension()`中首先会调用`getAdaptiveExtensionClass()`获取`ExtensionClasses`和修饰类，之后将修饰类返回。根据`META-INF/dubbo/internal/com.alibaba.dubbo.common.compiler.Compiler`文件的内容，最后返回

- `ExtensionClasses`
  - `"jdk" -> "class com.alibaba.dubbo.common.compiler.support.JdkCompiler"`
  - `"javassist" -> "class com.alibaba.dubbo.common.compiler.support.JavassistCompiler"`
- `cachedAdaptiveClass=class com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler`

之后调用`AdaptiveCompiler`的无参构造器创建`AdaptiveCompiler`对象实例，然后执行`injectExtension(T instance)`（这里没起作用）为`AdaptiveCompiler`对象实例注入相应的属性（`AdaptiveCompiler`必须提供相应的`setter`方法），最后返回`AdaptiveCompiler`对象实例。

### 三、编译代码并加载为Class<?>对象

创建好`AdaptiveCompiler`对象实例之后，然后执行下面的方法。

```java
Class<?> compile(String code, ClassLoader classLoader)
```

`AdaptiveCompiler`的源码如下：

```java
@Adaptive
public class AdaptiveCompiler implements Compiler {
    private static volatile String DEFAULT_COMPILER;// 默认的Compiler实现名字

    public static void setDefaultCompiler(String compiler) { // setter
        DEFAULT_COMPILER = compiler;
    }

    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);// 获取名字为name的实现类的实例,在获取的过程中会完成IoC和AOP
        } else {
            compiler = loader.getDefaultExtension();// 获取默认的JavassitCompiler,调用getExtension(cachedDefaultName)
        }
        return compiler.compile(code, classLoader);// 根据获取到的实现类compiler实例,来执行真正的动态生成类的代码
    }
}
```

这里执行的是`compiler = loader.getDefaultExtension()`，即调用`getExtension(cachedDefaultName)`生成一个`JavassistCompiler`的实例。之后就是执行`JavassistCompiler`的`compile(String code, ClassLoader classLoader)`方法。

由于`JavassistCompiler`继承自`AbstractCompiler`，且`compile(String code, ClassLoader classLoader)`定义在`AbstractCompiler`，因此首先看`AbstractCompiler`。

```java
import com.alibaba.dubbo.common.compiler.Compiler;
import com.alibaba.dubbo.common.utils.ClassHelper;

import java.util.regex.Matcher;
import java.util.regex.Pattern;


// Abstract compiler. (SPI, Prototype, ThreadSafe)

// (1)Javassist文档及实例:
// 1.http://www.javassist.org/tutorial/tutorial.html
// 2.http://www.cnblogs.com/java-zhao/p/7617733.html
// (2)Java正则表达式例子:
// 1.https://my.oschina.net/CasparLi/blog/361859
// 2.https://winter8.iteye.com/blog/1463244
public abstract class AbstractCompiler implements Compiler {
	// package匹配正则表达式
    private static final Pattern PACKAGE_PATTERN = Pattern.compile("package\\s+([$_a-zA-Z][$_a-zA-Z0-9\\.]*);");
	// class匹配正则表达式
    private static final Pattern CLASS_PATTERN = Pattern.compile("class\\s+([$_a-zA-Z][$_a-zA-Z0-9]*)\\s+");

    /**
     * 1 根据正则表达式从code中获取包名和类名，组成全类名
     * 2 根据全类名使用Class.forName创建Class<?>，如果该类在jvm中存在，则成功，否则抛出ClassNotFoundException，执行doCompile方法。
     */
    @Override
    public Class<?> compile(String code, ClassLoader classLoader) {
        code = code.trim();
        Matcher matcher = PACKAGE_PATTERN.matcher(code); // 找到包名package
        String pkg;
        if (matcher.find()) { 
            pkg = matcher.group(1);
        } else {
            pkg = "";
        }
        matcher = CLASS_PATTERN.matcher(code); // 找到类名class
        String cls;
        if (matcher.find()) {
            cls = matcher.group(1);
        } else {
            throw new IllegalArgumentException("No such class name in " + code);
        }
        // 获取全限定的类名
        String className = pkg != null && pkg.length() > 0 ? pkg + "." + cls : cls;
        try {
            // 尝试直接加载该类
            return Class.forName(className, true, ClassHelper.getCallerClassLoader(getClass())); 
        } catch (ClassNotFoundException e) {
            // 类未找到
            if (!code.endsWith("}")) {
                throw new IllegalStateException("The java code not endsWith \"}\", code: \n" + code + "\n");
            }
            try {
                return doCompile(className, code); // Javaassist直接生成
            } catch (RuntimeException t) {
                throw t;
            } catch (Throwable t) {
                throw new IllegalStateException("Failed to compile class, cause: " + t.getMessage() + ", class: "
                        + className + ", code: \n" + code + "\n, stack: " + ClassUtils.toString(t));
            }
        }
    }

    protected abstract Class<?> doCompile(String name, String source) throws Throwable;

}
```

`compile(String code, ClassLoader classLoader)`方法调用了`JavassistCompiler的Class<?> doCompile(String name, String source)`方法，在这个方法中，使用正则表达式对传入的源码解析成属性方法等，并使用`javassist`的API创建`Class<?>`。

```java
public class JavassistCompiler extends AbstractCompiler {
	// import正则匹配
    private static final Pattern IMPORT_PATTERN = Pattern.compile("import\\s+([\\w\\.\\*]+);\n");
	// extends语句正则匹配
    private static final Pattern EXTENDS_PATTERN = Pattern.compile("\\s+extends\\s+([\\w\\.]+)[^\\{]*\\{\n");
	// implements语句正则匹配
    private static final Pattern IMPLEMENTS_PATTERN = Pattern.compile("\\s+implements\\s+([\\w\\.]+)\\s*\\{\n");
	// 方法正则匹配
    private static final Pattern METHODS_PATTERN = Pattern.compile("\n(private|public|protected)\\s+");
	// Field正则匹配
    private static final Pattern FIELD_PATTERN = Pattern.compile("[^\n]+=[^\n]+;");

    // 解析source code，生成Class<?>
    @Override
    public Class<?> doCompile(String name, String source) throws Throwable {
        // 获取simpleName
        int i = name.lastIndexOf('.');
        String className = i < 0 ? name : name.substring(i + 1);
        ClassPool pool = new ClassPool(true);
        // 添加类路径
        pool.appendClassPath(new LoaderClassPath(ClassHelper.getCallerClassLoader(getClass()))); 
        Matcher matcher = IMPORT_PATTERN.matcher(source); // 匹配import
        List<String> importPackages = new ArrayList<String>();
        Map<String, String> fullNames = new HashMap<String, String>();
        while (matcher.find()) {
            String pkg = matcher.group(1);
            if (pkg.endsWith(".*")) { // import com.aa.bb.*; 这种
                String pkgName = pkg.substring(0, pkg.length() - 2);
                pool.importPackage(pkgName); //导入包名
                importPackages.add(pkgName); // 保存包名
            } else {
                int pi = pkg.lastIndexOf('.'); 
                if (pi > 0) {
                    String pkgName = pkg.substring(0, pi);// 获取包名
                    pool.importPackage(pkgName);
                    importPackages.add(pkgName);
                    fullNames.put(pkg.substring(pi + 1), pkg); // <类名，全限定类名>
                }
            }
        }
        String[] packages = importPackages.toArray(new String[0]); // 包名数组
        matcher = EXTENDS_PATTERN.matcher(source); // extends匹配
        CtClass cls;
        if (matcher.find()) {
            String extend = matcher.group(1).trim();
            String extendClass; // 全限定类名
            if (extend.contains(".")) {
                extendClass = extend;
            } else if (fullNames.containsKey(extend)) {
                extendClass = fullNames.get(extend);
            } else {
                extendClass = ClassUtils.forName(packages, extend).getName();
            }
            cls = pool.makeClass(name, pool.get(extendClass)); // 创建CtClass
        } else {
            cls = pool.makeClass(name);
        }
        matcher = IMPLEMENTS_PATTERN.matcher(source); // 匹配implements
        if (matcher.find()) {
            String[] ifaces = matcher.group(1).trim().split("\\,");
            for (String iface : ifaces) {
                iface = iface.trim();
                String ifaceClass;
                if (iface.contains(".")) {
                    ifaceClass = iface;
                } else if (fullNames.containsKey(iface)) {
                    ifaceClass = fullNames.get(iface);
                } else {
                    ifaceClass = ClassUtils.forName(packages, iface).getName();
                }
                cls.addInterface(pool.get(ifaceClass)); // 添加接口
            }
        }
       	// 类的body，包含方法和Field
        String body = source.substring(source.indexOf("{") + 1, source.length() - 1);
        String[] methods = METHODS_PATTERN.split(body); // 找到方法，包含构造器
        for (String method : methods) {
            method = method.trim();
            if (method.length() > 0) {
                if (method.startsWith(className)) {
                    // 构造器
                    cls.addConstructor(CtNewConstructor.make("public " + method, cls));
                } else if (FIELD_PATTERN.matcher(method).matches()) { // 全匹配返回true，判断为Field
                    cls.addField(CtField.make("private " + method, cls));
                } else {
                    // 普通方法
                    cls.addMethod(CtNewMethod.make("public " + method, cls));
                }
            }
        }
        // 加载为Class
        return cls.toClass(ClassHelper.getCallerClassLoader(getClass()), JavassistCompiler.class.getProtectionDomain());
    }

}
```

最后，该`Protocol adaptiveExtension = loader.getAdaptiveExtension();`代码返回的`adaptiveExtension = Protocol$Adaptive`实例。

### 四、总结

- `ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`最终返回的是：`AdaptiveExtensionFactory`实例，其属性`factories = [SpringExtensionFactory实例, SpiExtensionFactory实例]`
- 不管是获取哪一个SPI接口（除了`ExtensionFactory`接口）的`ExtensionLoader`，最终一定会有一个`objectFactory`=上述的`AdaptiveExtensionFactory`实例
- `getAdaptiveExtension()`：作用就是获取一个适配类或动态代理类的实例, 如果有`@Adaptive`注解的类,则直接返回该类的实例,否则返回一个动态代理类的实例(例如`Protocol$Adaptive`的实例)，之后完成属性注入（`dubbo-IoC`），最后返回实例。
- `getExtension(String key)`：作用就是从`extensionClasses`（即指定SPI接口的没有`@Adaptive`的实现类）获取指定key的`extensionClass`，并且实例化，之后完成属性注入（`dubbo-IoC`），再之后完成`dubbo-AOP`，最后返回实例。

