---
layout:     post
title:      Spring MVC HttpMessageConverter转换请求和响应数据的过程分析
subtitle:   HttpMessageConverter
date:       2019-08-14
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
    - Spring MVC
---

# Spring MVC HttpMessageConverter转换请求和响应数据的过程分析

转自[HttpMessageConverter是这样转换数据的](https://mp.weixin.qq.com/s?__biz=Mzg3NjIxMjA1Ng==&mid=2247483668&idx=1&sn=69e83dded8f92539084a095be50440f8&chksm=cf34fb23f8437235b5bd57d42c33247f11d35eba581925c5cb120496fc2982eeb187440c1e28&mpshare=1&scene=1&srcid=081458cbnAFyhxRL6XohInI8&sharer_sharetime=1565741846092&sharer_shareid=1fdfc09d408b1e8635267ab8cac2b4d5#rd)

Java Web 人员经常要设计 RESTful API，通过 json 数据进行交互。那么前端传入的 json 数据如何被解析成 Java 对象作为 API入参，API 返回结果又如何将 Java 对象解析成 json 格式数据返回给前端？其实在整个数据流转过程中，`HttpMessageConverter` 起到了重要作用。

### 一、HttpMessageConverter简介

`org.springframework.http.converter.HttpMessageConverter` 是一个策略接口，接口说明如下：

>  Strategy interface that specifies a converter that can convert from and to HTTP requests and responses. 简单说就是 HTTP request (请求)和response (响应)的转换器。
> 该接口只有5个方法，就是获取支持的 MediaType（application/json之类），接收到请求时判断是否能读（canRead），能读则读（read）；返回结果时判断是否能写（canWrite），能写则写（write）。

```java
boolean canRead(Class<?> clazz, MediaType mediaType);
boolean canWrite(Class<?> clazz, MediaType mediaType);
List<MediaType> getSupportedMediaTypes();
T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;
void write(T t, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException;
```

##### 1. 缺省配置

我们写 Demo 没有配置任何 MessageConverter，但是数据前后传递依旧好用，是因为 SpringMVC 启动时(使用Java Config配置，使用`@EnableWebMVc`注解)会自动配置一些HttpMessageConverter，在 `WebMvcConfigurationSupport` 类中添加了缺省 MessageConverter：

```java
protected final List<HttpMessageConverter<?>> getMessageConverters() {
   if (this.messageConverters == null) {
      this.messageConverters = new ArrayList<>();
      configureMessageConverters(this.messageConverters); // 用户自定义HttpMessageConverters
      if (this.messageConverters.isEmpty()) {
         // 如果用户自定义的HttpMessageConverters为空，则添加默认HttpMessageConverters
         addDefaultHttpMessageConverters(this.messageConverters);
      }
      extendMessageConverters(this.messageConverters);
   }
   return this.messageConverters;
}
// 添加默认HttpMessageConverters
protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
	StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
	stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

	messageConverters.add(new ByteArrayHttpMessageConverter());
	messageConverters.add(stringHttpMessageConverter);
	messageConverters.add(new ResourceHttpMessageConverter());
	messageConverters.add(new ResourceRegionHttpMessageConverter());
	messageConverters.add(new SourceHttpMessageConverter<>());
	messageConverters.add(new AllEncompassingFormHttpMessageConverter());

	if (romePresent) {
		messageConverters.add(new AtomFeedHttpMessageConverter());
		messageConverters.add(new RssChannelHttpMessageConverter());
	}

	if (jackson2XmlPresent) {
		Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
		if (this.applicationContext != null) {
			builder.applicationContext(this.applicationContext);
		}
		messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
	}
	else if (jaxb2Present) {
		messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
	}

	if (jackson2Present) {
		Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
		if (this.applicationContext != null) {
			builder.applicationContext(this.applicationContext);
		}
    // 对象到json数据的转化器
		messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
	}
	else if (gsonPresent) {
		messageConverters.add(new GsonHttpMessageConverter());
	}
	else if (jsonbPresent) {
		messageConverters.add(new JsonbHttpMessageConverter());
	}

	if (jackson2SmilePresent) {
		Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.smile();
		if (this.applicationContext != null) {
			builder.applicationContext(this.applicationContext);
		}
		messageConverters.add(new MappingJackson2SmileHttpMessageConverter(builder.build()));
	}
	if (jackson2CborPresent) {
		Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.cbor();
		if (this.applicationContext != null) {
			builder.applicationContext(this.applicationContext);
		}
		messageConverters.add(new MappingJackson2CborHttpMessageConverter(builder.build()));
	}
}
```

我们看到很熟悉的 `MappingJackson2HttpMessageConverter`，如果我们引入 jackson 相关包，Spring 就会为我们添加该 HttpMessageConverter，但是我们通常在搭建框架的时候还是会手动添加配置 `MappingJackson2HttpMessageConverter`，为什么？

> 因为当我们配置了自己的 HttpMessageConverter， SpringMVC 启动过程就不会调用 `addDefaultHttpMessageConverters` 方法。可以看上面代码块的第一个方法——如果用户自定义的HttpMessageConverters不为空，则`addDefaultHttpMessageConverters`不会调用，这样做也是为了定制化我们自己的 HttpMessageConverter。

##### 2. 类关系图

在此处仅列出 `MappingJackson2HttpMessageConverter` 和 `StringHttpMessageConverter` 两个转换器，我们发现， 前者实现了 `GenericHttpMessageConverter` 接口, 而后者却没有，留有这个**关键**印象，这是数据流转过程分析的关键逻辑判断。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-12.jpg)

###  二、数据流转分析

数据的请求和响应都要经过 `DispatcherServlet` 类的 `doDispatch(HttpServletRequest request, HttpServletResponse response)` 方法的处理。

##### 1. 请求过程解析

看 doDispatch 方法中的关键代码：

```java
// 这里的 Adapter 实际上是 RequestMappingHandlerAdapter
HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler()); 
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
}
// 执行handler
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());            mappedHandler.applyPostHandle(processedRequest, response, mv);
```

这里将进入 ha.handle 方法后的调用栈粘贴在此处：

```java
readWithMessageConverters:192, AbstractMessageConverterMethodArgumentResolver (org.springframework.web.servlet.mvc.method.annotation)
readWithMessageConverters:150, RequestResponseBodyMethodProcessor (org.springframework.web.servlet.mvc.method.annotation)
resolveArgument:128, RequestResponseBodyMethodProcessor (org.springframework.web.servlet.mvc.method.annotation)
resolveArgument:121, HandlerMethodArgumentResolverComposite (org.springframework.web.method.support)
getMethodArgumentValues:158, InvocableHandlerMethod (org.springframework.web.method.support)
invokeForRequest:128, InvocableHandlerMethod (org.springframework.web.method.support)
 // 下面的调用栈重点关注，处理请求和返回值的分叉口就在这里
invokeAndHandle:97, ServletInvocableHandlerMethod (org.springframework.web.servlet.mvc.method.annotation)
invokeHandlerMethod:849, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
handleInternal:760, RequestMappingHandlerAdapter (org.springframework.web.servlet.mvc.method.annotation)
handle:85, AbstractHandlerMethodAdapter (org.springframework.web.servlet.mvc.method)
doDispatch:967, DispatcherServlet (org.springframework.web.servlet)
```

这里重点说明调用栈最顶层 `readWithMessageConverters` 方法中内容：

```java
// 遍历 messageConverters
for (HttpMessageConverter<?> converter : this.messageConverters) {
    Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
    // 上文类关系图处要重点记住的地方，主要判断 MappingJackson2HttpMessageConverter 是否是 GenericHttpMessageConverter 类型
    if (converter instanceof GenericHttpMessageConverter) {
        GenericHttpMessageConverter<?> genericConverter = (GenericHttpMessageConverter<?>) converter;
        if (genericConverter.canRead(targetType, contextClass, contentType)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
            }
            if (inputMessage.getBody() != null) {
                inputMessage = getAdvice().beforeBodyRead(inputMessage, parameter, targetType, converterType);
                body = genericConverter.read(targetType, contextClass, inputMessage);
                body = getAdvice().afterBodyRead(body, inputMessage, parameter, targetType, converterType);
            }
            else {
                body = getAdvice().handleEmptyBody(null, inputMessage, parameter, targetType, converterType);
            }
            break;
        }
    }
    else if (targetClass != null) {
        if (converter.canRead(targetClass, contentType)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
            }
            if (inputMessage.getBody() != null) {
                inputMessage = getAdvice().beforeBodyRead(inputMessage, parameter, targetType, converterType);
                body = ((HttpMessageConverter<T>) converter).read(targetClass, inputMessage);
                body = getAdvice().afterBodyRead(body, inputMessage, parameter, targetType, converterType);
            }
            else {
                body = getAdvice().handleEmptyBody(null, inputMessage, parameter, targetType, converterType);
            }
            break;
        }
    }
}
```

然后就判断是否canRead，能读就read，最终走到下面代码处将输入的内容反序列化出来：

```java
protected Object _readMapAndClose(JsonParser p0, JavaType valueType) throws IOException{
    try (JsonParser p = p0) {
        Object result;
        JsonToken t = _initForReading(p);
        if (t == JsonToken.VALUE_NULL) {
            // Ask JsonDeserializer what 'null value' to use:
            DeserializationContext ctxt = createDeserializationContext(p,
                    getDeserializationConfig());
            result = _findRootDeserializer(ctxt, valueType).getNullValue(ctxt);
        } else if (t == JsonToken.END_ARRAY || t == JsonToken.END_OBJECT) {
            result = null;
        } else {
            DeserializationConfig cfg = getDeserializationConfig();
            DeserializationContext ctxt = createDeserializationContext(p, cfg);
            JsonDeserializer<Object> deser = _findRootDeserializer(ctxt, valueType);
            if (cfg.useRootWrapping()) {
                result = _unwrapAndDeserialize(p, ctxt, cfg, valueType, deser);
            } else {
                result = deser.deserialize(p, ctxt);
            }
            ctxt.checkUnresolvedObjectId();
        }
        // Need to consume the token too
        p.clearCurrentToken();
        return result;
    }
}
```

到这里从请求中解析参数过程的分析就到此结束了，趁热打铁来看将响应结果返回给前端的过程。

##### 2. 返回过程分析

在上面调用栈请求和返回结果分叉口处同样处理返回值的内容：

```java
writeWithMessageConverters:224, AbstractMessageConverterMethodProcessor (org.springframework.web.servlet.mvc.method.annotation)
handleReturnValue:174, RequestResponseBodyMethodProcessor (org.springframework.web.servlet.mvc.method.annotation)
handleReturnValue:81, HandlerMethodReturnValueHandlerComposite (org.springframework.web.method.support)
// 分叉口
invokeAndHandle:113, ServletInvocableHandlerMethod (org.springframework.web.servlet.mvc.method.annotation)
```

重点关注调用栈顶层内容，是不是很熟悉的样子，完全一样的逻辑, 判断是否能写canWrite，能写则write：

```java
for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
    if (messageConverter instanceof GenericHttpMessageConverter) {
        if (((GenericHttpMessageConverter) messageConverter).canWrite(
                declaredType, valueType, selectedMediaType)) {
            outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                    (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                    inputMessage, outputMessage);
            if (outputValue != null) {
                addContentDispositionHeader(inputMessage, outputMessage);
                ((GenericHttpMessageConverter) messageConverter).write(
                        outputValue, declaredType, selectedMediaType, outputMessage);
                if (logger.isDebugEnabled()) {
                    logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                            "\" using [" + messageConverter + "]");
                }
            }
            return;
        }
    }
    else if (messageConverter.canWrite(valueType, selectedMediaType)) {
        outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                inputMessage, outputMessage);
        if (outputValue != null) {
            addContentDispositionHeader(inputMessage, outputMessage);
            ((HttpMessageConverter) messageConverter).write(outputValue, selectedMediaType, outputMessage);
            if (logger.isDebugEnabled()) {
                logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                        "\" using [" + messageConverter + "]");
            }
        }
        return;
    }
}
```

上面代码第5行，我们看到有这样代码：

```java
outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                    (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                    inputMessage, outputMessage);
```

其实，我们在设计 RESTful API 接口的时候通常会将返回的数据封装成统一格式，通常我们会实现 ResponseBodyAdvice 接口来处理所有 API 的返回值，在真正 write 之前将数据进行统一的封装：

```java
@RestControllerAdvice
public class CommonResultResponseAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
            ServerHttpResponse response) {
        if (body instanceof CommonResult) {
            return body;
        }
        return new CommonResult<Object>(body);
    }
}
```

至此，通过 HttpMessageConverter 转换请求和响应数据的流程就是这样。

### 三、细节分析

canRead 和 canWrite 的判断逻辑是什么呢？ 请看下图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-13.jpg)

客户端 Request Header 中设置好 Content-Type（传出的数据格式）和Accept（接收的数据格式），根据配置好的 MessageConverter 来判断是否 canRead 或 canWrite，然后决定 response.body 的 Content-Type 的第一要素是对应的request.headers.Accept 属性的值。如果服务端支持这个 Accept，那么应该按照这个 Accept 来确定返回response.body 对应的格式，同时把 response.headers.Content-Type 设置成自己支持的符合那个 Accept 的 MediaType。

### 四、总结与思考

站在上帝视角看，整个流程可以按照下图进行概括，请求报文先转换成 HttpInputMessage, 然后再通过 HttpMessageConverter 将其转换成 SpringMVC 的 java 对象，反之亦然。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-14.jpg)

将各种常用 HttpMessageConverter 支持的MediaType 和 JavaType 以及对应关系总结在此处：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-15.jpg)

### 参考文章

- [SpringMVC源码剖析（五)-消息转换器HttpMessageConverter](https://my.oschina.net/lichhao/blog/172562)

