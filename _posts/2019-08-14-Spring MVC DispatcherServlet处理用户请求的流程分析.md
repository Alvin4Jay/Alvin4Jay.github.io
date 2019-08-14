---
layout:     post
title:      Spring MVC DispatcherServlet处理用户请求的流程分析
subtitle:   DispatcherServlet/HandlerMapping/HandlerAdapter/ViewResolver
date:       2019-08-14
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring
    - Spring MVC
---

# Spring MVC DispatcherServlet处理用户请求的流程分析

在前一篇文章[Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)中，详细探讨了`Spring MVC`在`Web容器`中部署后的启动过程，以及相关源码分析，同时也讨论了`DispatcherServlet类`的初始化创建过程。本文主要讲解`DispatcherServlet类`获取用户请求到响应的全过程，并针对相关源码进行分析。

首先，站在`Spring MVC`的四大组件:`DispatcherServlet`、`HandlerMapping`、`HandlerAdapter`以及`ViewResolver`的角度来看一下`Spring MVC`对用户请求的处理过程，有如下流程图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-10.jpg)

具体处理过程如下:

- ① 用户请求发送至`DispatcherServlet类`进行处理。
- ② `DispatcherServlet类`遍历所有配置的`HandlerMapping类`，请求查找`Handler`。`HandlerMapping类`根据`request请求`的`URL`等信息查找能够进行处理的`Handler`，以及相关拦截器`HandlerInterceptor`并构造`HandlerExecutionChain`。然后`HandlerMapping类`将构造的`HandlerExecutionChain类`的对象返回给前端控制器`DispatcherServlet类`。
- ③ 前端控制器拿着上一步的`Handler`遍历所有配置的`HandlerAdapter类`，请求执行`Handler`。
- ④ `HandlerAdapter类`执行相关`Handler`并获取`ModelAndView类`的对象，然后将该`ModelAndView类`的对象返回给前端控制器。
- ⑤ `DispatcherServlet类`遍历所有配置的`ViewResolver类`，请求进行视图解析。`ViewResolver类`解析视图后得到`View`对象并返回给前端控制器。
- ⑥ `DispatcherServlet类`进行视图`View`的渲染，填充`Model`。
- ⑦ `DispatcherServlet类`向用户返回响应。

通过流程图和上面的讲解不难发现，整个`Spring MVC`对于用户请求的响应和处理都是以`DispatcherServlet类`为核心，其他三大组件均与前端控制器进行交互，三大组件之间没有交互并且互相解耦，因此，三大组件可以替换不同的实现而互相没有任何影响，提高了整个架构的稳定性并且降低了耦合度。接下来会按照上述的响应过程逐一进行讲解。

`DispatcherServlet类`本质上依旧是一个`Servlet`并且其父类实现了`Servlet接口`，因为`Servlet`执行`service()`方法对用户请求进行响应，根据前一篇文章[Spring MVC启动流程分析(xml配置)](https://xuanjian1992.top/2019/08/13/Spring-MVC%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90(xml%E9%85%8D%E7%BD%AE)/)的分析，可以得到如下的调用逻辑图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/Spring%20Framework/Spring%20MVC-11.jpeg)

从上图的源码调用逻辑可以看出，`HttpServlet抽象类`实现了`Servlet接口`的`service(ServletRequest, ServletResponse)`的方法，因此，用户请求的第一执行方法为该方法，该方法紧接着直接调用了`service(HttpServletRequest, HttpServletResponse)`方法，其子类`FrameworkServlet抽象类`重写了该方法，因为多态的特性最终是调用了`FrameworkServlet抽象类`的`service(HttpServletRequest, HttpServletResponse)`方法，`FrameworkServlet抽象类`同样也重写了`doHead()`、`doPost()`、`doPut()`、`doDelete()`、`doOptions()`、`doTrace()`方法，`service(ServletRequest, ServletResponse)`方法根据请求类型的不同分别调用上述方法，上述六个方法都调用了`processRequest()`方法，而该方法最终调用了`DispatcherServlet类`的`doService()`方法。通过层层分析，找到了最终要调用的处理用户请求的方法`doService()`。

下面先看下`processRequest()`方法的逻辑：

```java
// FrameworkServlet类
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   long startTime = System.currentTimeMillis();
   Throwable failureCause = null;

   LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
   LocaleContext localeContext = buildLocaleContext(request); // 构建LocaleContext

   RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
   // 构建请求属性ServletRequestAttributes
   ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

   // 将LocaleContext和ServletRequestAttributes绑定到当前线程
   initContextHolders(request, localeContext, requestAttributes);

   try {
      doService(request, response); // DispatcherServlet实现，处理请求
   }
   catch (ServletException | IOException ex) {
      failureCause = ex;
      throw ex;
   }
   catch (Throwable ex) {
      failureCause = ex;
      throw new NestedServletException("Request processing failed", ex);
   }

   finally {
      // 重置线程的LocaleContext和RequestAttributes
      resetContextHolders(request, previousLocaleContext, previousAttributes);
      if (requestAttributes != null) {
         requestAttributes.requestCompleted();
      }
      logResult(request, response, failureCause, asyncManager);
      publishRequestHandledEvent(request, response, startTime, failureCause); // 发布事件
   }
}
```

`processRequest()`方法首先构建`LocaleContext`和`ServletRequestAttributes`对象，并将这些对象绑定到当前线程，然后调用` doService(request, response)`方法处理用户请求。接下来看`DispatcherServlet.doService(request, response)`方法的代码：

```java
// DispatcherServlet类
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
   logRequest(request); // 日志记录请求

   // 保存属性快照
   Map<String, Object> attributesSnapshot = null;
   if (WebUtils.isIncludeRequest(request)) {
      attributesSnapshot = new HashMap<>();
      Enumeration<?> attrNames = request.getAttributeNames();
      while (attrNames.hasMoreElements()) {
         String attrName = (String) attrNames.nextElement();
         if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
            attributesSnapshot.put(attrName, request.getAttribute(attrName));
         }
      }
   }

   // 将DispatcherServlet的web应用上下文(子IoC容器)、LocaleResolver等放入请求中，供handler、view对象使用
   request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
   request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
   request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
   request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

   if (this.flashMapManager != null) {
      FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
      if (inputFlashMap != null) {
         request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
      }
      request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
      request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
   }

   try {
      doDispatch(request, response); // 真正进行用户请求的处理
   }
   finally {
      if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
         if (attributesSnapshot != null) { // 还原属性
            restoreAttributesAfterInclude(request, attributesSnapshot);
         }
      }
   }
}
```

`doService()`方法主要进行一些参数的设置，并将部分参数放入`request`请求中，真正执行用户请求并作出响应的方法则为`doDispatch()`方法，查看`doDispatch()`方法的源码如下:

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request; // 用户请求
   HandlerExecutionChain mappedHandler = null; // handler执行链
   boolean multipartRequestParsed = false; // 判断是否解析了文件类型的数据

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null; // 视图与模型
      Exception dispatchException = null;

      try {
         // 解析MultipartHttpServletRequest请求，检查是否包含文件等类型的数据
         processedRequest = checkMultipart(request); 
         multipartRequestParsed = (processedRequest != request);

         // 根据请求，从HandlerMapping获取HandlerExecutionChain
         mappedHandler = getHandler(processedRequest); 
         if (mappedHandler == null) {
            // 如果HandlerExecutionChain为null，则没有能够进行处理的Handler，返回404
            noHandlerFound(processedRequest, response);
            return;
         }

         // 根据查找到的Handler，获取能够进行处理的HandlerAdapter
         // 获取RequestMappingHandlerAdapter
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler()); 

         // 判断自上次请求后是否有修改，没有修改直接返回响应
         String method = request.getMethod();
         boolean isGet = "GET".equals(method);
         if (isGet || "HEAD".equals(method)) {
            // -1
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler()); 
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return; // get请求，且未修改，直接返回
            }
         }

         // 按顺序依次执行HandlerInterceptor的preHandle方法
         // 如果任一HandlerInterceptor的preHandle方法没有通过，则不继续进行处理
         if (!mappedHandler.applyPreHandle(processedRequest, response)) { // 1.应用HandlerInterceptor的preHandle方法
            return; // 返回false，则后续handler的调用不在执行
         }

         // Actually invoke the handler. 2.调用handler(通过HandlerAdapter执行查找到的handler)
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) { // false
            return;
         }

         applyDefaultViewName(processedRequest, mv); // 应用默认视图名
         // 3.逆序执行HandlerInterceptor的postHandle方法
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      // 视图渲染，如果有异常，则渲染异常页面
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      // 如果有异常，按逆序执行所有HandlerInterceptor的afterCompletion方法
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      // 如果有异常，按逆序执行所有HandlerInterceptor的afterCompletion方法
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         if (mappedHandler != null) {
            // 逆序执行所有HandlerInterceptor的afterCompletion方法
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // 如果请求包含文件类型的数据，则进行相关清理工作
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}
```

根据上述源码并结合文章开始讲解的`DispatcherServlet类`结合三大组件对用户请求的处理过程，不难理解相关处理流程。

### 一、查找HandlerExecutionChain

`doDispatch()`方法通过调用`getHandler()`方法并传入`reuqest`，通过`HandlerMapping`查找`HandlerExecutionChain`，查看其源码如下:

```java
// DispatcherServlet类
// 根据请求，从HandlerMapping获取HandlerExecutionChain
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   if (this.handlerMappings != null) {
      for (HandlerMapping mapping : this.handlerMappings) { // 遍历HandlerMapping
         HandlerExecutionChain handler = mapping.getHandler(request); // 根据请求获取处理器执行链
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}
```

`getHandler()`方法然后调用`RequestMappingHandlerMapping.getHandler(HttpServletRequest)`方法，具体实现在`RequestMappingHandlerMapping`的父类`AbstractHandlerMapping`中：

```java
// AbstractHandlerMapping类
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   Object handler = getHandlerInternal(request); // 根据请求查找HandlerMethod
   if (handler == null) {
      handler = getDefaultHandler();
   }
   if (handler == null) {
      return null;
   }
   // Bean name or resolved handler?
   if (handler instanceof String) {
      String handlerName = (String) handler;
      handler = obtainApplicationContext().getBean(handlerName);
   }
   // 构建处理器执行链：Handler + HandlerInterceptors
   HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

   if (logger.isTraceEnabled()) {
      logger.trace("Mapped to " + handler);
   }
   else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
      logger.debug("Mapped to " + executionChain.getHandler());
   }

   if (CorsUtils.isCorsRequest(request)) {
      CorsConfiguration globalConfig = this.corsConfigurationSource.getCorsConfiguration(request);
      CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
      CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   }

   return executionChain;
}
```

上面的`getHandler(HttpServletRequest)`方法中，首先根据请求查找`HandlerMethod`(Web容器启动时通过扫描Bean，已经缓存了请求信息对应的`HandlerMethod`)，然后根据找到的`HandlerMethod`和请求`HttpServletRequest`构建`HandlerExecutionChain`处理器执行链。

于是，只要`RequestMappingHandlerMapping.getHandler(HttpServletRequest)`方法找到`HandlerExecutionChain`执行链，`DispatcherServlet`的`getHandler()`便会返回。

如果没有找到对应的`HandlerExecutionChain`对象，则会执行`noHandlerFound()`方法，继续查看其源码如下:

```java
// 如果HandlerExecutionChain为null，则没有能够进行处理的Handler，返回404
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
   if (pageNotFoundLogger.isWarnEnabled()) {
      pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
   }
   if (this.throwExceptionIfNoHandlerFound) { // false
      throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
            new ServletServerHttpRequest(request).getHeaders());
   }
   else {
      response.sendError(HttpServletResponse.SC_NOT_FOUND); // 返回404
   }
}
```

如果HandlerExecutionChain为null，则没有能够对请求进行处理的Handler，返回404。

### 二、查找HandlerAdapter

继续查看`doDispatch()`方法的源码，如果找到了`HandlerExecutionChain`，接下来会调用`getHandlerAdapter()`方法来查找能够对`Handler`进行处理的`HandlerAdapter`，查看其源码如下:

```java
// handler: HandlerMethod对象
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException { 
   if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) {
         if (adapter.supports(handler)) { // RequestMappingHandlerAdapter
            return adapter;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

与`HandlerMapping`类似，查找能够处理具体`Handler`的`HandlerAdapter`时也会遍历所有配置了的`HandlerAdapter`，`HandlerAdapter`是一个接口，包含一个`support()`方法，该方法根据`Handler`是否实现某个特定的接口来判断该`HandlerAdapter`是否能够处理这个具体的`Handler`，这里使用适配器模式，通过这样的方式就可以支持不同类型的`HandlerAdapter`。如果没有查找到能够处理`Handler`的`HandlerAdapter`则会抛出异常。

### 三、执行HandlerExecutionChain

查找到了对应的`HandlerAdapter`后(如`RequestMappingHandlerAdapter`)，就会调用`HandlerExecutionChain`的`applyPreHandle()`方法来执行配置的所有`HandlerInteceptor`的`preHandle()`方法，查看其源码如下:

```java
// 应用HandlerInterceptor的preHandle方法
if (!mappedHandler.applyPreHandle(processedRequest, response)) { 
   return; // 返回false，则后续handler的调用不在执行
}

boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
		for (int i = 0; i < interceptors.length; i++) {
			HandlerInterceptor interceptor = interceptors[i];
			if (!interceptor.preHandle(request, response, this.handler)) {
				triggerAfterCompletion(request, response, null);
				return false; // preHandle返回false，则剩余的HandlerInterceptor和handler不执行
			}
			this.interceptorIndex = i;
		}
	}
	return true;
}
```

`HandlerExecutionChain`的`applyPreHandle()`方法会按照顺序依次调用`HandlerInterceptor`的`preHandle()`方法，但当任一`HandlerInterceptor`的`preHandle()`方法返回了`false`就不再继续执行其他`HandlerInterceptor`的`preHandle()`方法，而是直接跳转执行`triggerAfterCompletion()`方法，查看该方法源码如下:

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
      throws Exception {

   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = this.interceptorIndex; i >= 0; i--) { // 反序执行afterCompletion方法
         HandlerInterceptor interceptor = interceptors[i];
         try {
            interceptor.afterCompletion(request, response, this.handler, ex);
         }
         catch (Throwable ex2) {
            logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
         }
      }
   }
}
```

这里遍历的下标为`interceptorIndex`，该变量在前一个方法`applyPreHandle()`方法中赋值，如果`preHandle()`方法返回`true`该变量加一，因此该方法会逆序执行所有`preHandle()`方法返回了`true`的`HandlerInterceptor`的`afterCompletion()`方法。

继续阅读`doDispatch()`方法的源码，如果所有拦截器的`preHandle()`方法都返回了`true`，则接下来前端控制器会请求执行上文获取的`Handler`，并获取到`ModelAndView类`的对象。

```java
// 调用handler(通过HandlerAdapter执行查找到的handler)
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

// AbstractHandlerMethodAdapter类
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
	return handleInternal(request, response, (HandlerMethod) handler);
}

// RequestMappingHandlerAdapter类
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ModelAndView mav;
	checkRequest(request); // 检查请求方法是否支持、session检查

	// Execute invokeHandlerMethod in synchronized block if required.
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No HttpSession available -> no mutex necessary
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	}
	else {
		// No synchronization on session demanded at all...
		mav = invokeHandlerMethod(request, response, handlerMethod); // 执行HandlerMethod逻辑
	}

	if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
		if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) { // false
			applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
		}
		else {
			prepareResponse(response);
		}
	}

	return mav;
}
```

通过`RequestMapppingHandlerAdapter`执行`Handler`时，最终会调用到`invokeHandlerMethod()`方法，其源码如下：

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   try {
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod); // ServletRequestDataBinderFactory
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory); // ModelFactory
      // ServletInvocableHandlerMethod
      ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      if (this.argumentResolvers != null) {
         invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      }
      if (this.returnValueHandlers != null) {
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      }
      invocableMethod.setDataBinderFactory(binderFactory);
      invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      ModelAndViewContainer mavContainer = new ModelAndViewContainer();
      mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
      modelFactory.initModel(webRequest, mavContainer, invocableMethod);
      mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

      AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
      asyncWebRequest.setTimeout(this.asyncRequestTimeout);

      WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
      asyncManager.setTaskExecutor(this.taskExecutor);
      asyncManager.setAsyncWebRequest(asyncWebRequest);
      asyncManager.registerCallableInterceptors(this.callableInterceptors);
      asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

      if (asyncManager.hasConcurrentResult()) {
         Object result = asyncManager.getConcurrentResult();
         mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
         asyncManager.clearConcurrentResult();
         LogFormatUtils.traceDebug(logger, traceOn -> {
            String formatted = LogFormatUtils.formatValue(result, !traceOn);
            return "Resume with async result [" + formatted + "]";
         });
         invocableMethod = invocableMethod.wrapConcurrentResult(result);
      }

      invocableMethod.invokeAndHandle(webRequest, mavContainer); // 调用HandlerMethod，并返回响应，设置是否需要视图解析
      if (asyncManager.isConcurrentHandlingStarted()) {
         return null;
      }

      return getModelAndView(mavContainer, modelFactory, webRequest); // 获取ModelAndView
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

`invokeHandlerMethod()`方法会将`HandlerMethod`包装成`ServletInvocableHandlerMethod`对象，然后调用其`invokeAndHandle()`方法：

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs); // 调用HandlerMethod
   setResponseStatus(webRequest);

   if (returnValue == null) {
      if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
         mavContainer.setRequestHandled(true); // 请求已处理，不需要视图解析
         return;
      }
   }
   else if (StringUtils.hasText(getResponseStatusReason())) {
      mavContainer.setRequestHandled(true);
      return;
   }

   mavContainer.setRequestHandled(false);
   Assert.state(this.returnValueHandlers != null, "No return value handlers");
   try {
      // this.returnValueHandlers: HandlerMethodReturnValueHandlerComposite
      this.returnValueHandlers.handleReturnValue( // 处理返回值
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
   catch (Exception ex) {
      if (logger.isTraceEnabled()) {
         logger.trace(formatErrorForReturnValue(returnValue), ex);
      }
      throw ex;
   }
}
```

`invokeAndHandle()`方法调用处理器方法`HandlerMethod`，并处理返回值`returnValue`(比如将返回值直接写出到Response)。

```java
// this.returnValueHandlers: HandlerMethodReturnValueHandlerComposite
this.returnValueHandlers.handleReturnValue( // 处理返回值
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
```

处理返回值时的源码如下：

```java
// HandlerMethodReturnValueHandlerComposite类
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	 // 选择HandlerMethodReturnValueHandler
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType); 
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
   // handler: RequestResponseBodyMethodProcessor(处理)
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}


```

首先选择`HandlerMethodReturnValueHandler`对象，**这里假设**`MethodParameter returnType`对应的类或方法上带有`@ResponseBody`注解，则得到的`HandlerMethodReturnValueHandler`为`RequestResponseBodyMethodProcessor`对象。接着调用`RequestResponseBodyMethodProcessor`的`handleReturnValue()`方法。

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   mavContainer.setRequestHandled(true);// 不需要再使用ModelAndView进行视图渲染
   ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
   ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

   // Try even with null return value. ResponseBodyAdvice could get involved.
   writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

由于`@ResponseBody`注解的存在，可以向用户直接返回响应而不需要再使用`ModelAndView`进行视图渲染处理，因此执行`mavContainer.setRequestHandled(true)`代码，表示请求已被完全处理完毕、不再需要视图解析。接着调用`writeWithMessageConverters()`方法，使用`HttpMessageConverters`将响应内容写出到Response body，返回给请求用户。`writeWithMessageConverters()`方法中会先进行响应内容的处理，如`ResponseBodyAdvice.beforeBodyWrite()`以及`HttpMessageConverters`对内容进行转换，再写出到`Response`。

```java
// 调用handler(通过HandlerAdapter执行查找到的handler)
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

在通过`HandlerAdapter`执行查找到的`Handler`并获取到`ModelAndView类`的对象之后，会执行`HandlerInterceptor`的`postHandle()`方法：

```java
// 逆序执行HandlerInterceptor的postHandle方法
mappedHandler.applyPostHandle(processedRequest, response, mv);

void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
		throws Exception {

	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
		for (int i = interceptors.length - 1; i >= 0; i--) { // 反序执行afterCompletion方法
			HandlerInterceptor interceptor = interceptors[i];
			interceptor.postHandle(request, response, this.handler, mv);
		}
	}
}
```

可以发现，`postHandle()`方法是按照逆序执行的。

### 四、视图渲染

执行完`postHandle()`方法后，`doDispatch()`方法调用了`processDispatchResult()`方法，其源码如下:

```java
// 视图渲染，如果有异常，则渲染异常页面
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
		@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
		@Nullable Exception exception) throws Exception {

	boolean errorView = false;

	// 判断HandlerMapping、HandlerAdapter处理时的异常是否为空
	if (exception != null) {
		// 上述两个组件处理时的异常不为空
		// 如果为ModelAndViewDefiningException异常，则获取一个异常视图
		if (exception instanceof ModelAndViewDefiningException) {
			logger.debug("ModelAndViewDefiningException encountered", exception);
			mv = ((ModelAndViewDefiningException) exception).getModelAndView();
		}
		else {
			// 如果不为ModelAndViewDefiningException异常，进行异常视图的获取
			Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
			mv = processHandlerException(request, response, handler, exception);
			errorView = (mv != null);
		}
	}

	// Did the handler return a view to render?
	// 判断mv是否为空，不管是正常的ModelAndView还是异常的ModelAndView，只要存在mv就进行视图渲染
	if (mv != null && !mv.wasCleared()) {
		render(mv, request, response); // 视图渲染
		if (errorView) {
			WebUtils.clearErrorRequestAttributes(request);
		}
	}
	else {  // 否则记录无视图
		if (logger.isTraceEnabled()) {
			logger.trace("No view rendering, null ModelAndView returned.");
		}
	}

	if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
		// Concurrent handling started during a forward
		return;
	}

	if (mappedHandler != null) { // 执行HandlerInterceptor的afterCompletion方法
		mappedHandler.triggerAfterCompletion(request, response, null);
	}
}
```

`processDispatchResult()`方法传入了一个异常类的对象`dispatchException`，阅读`doDispatch()`方法的源码可以看出，`Spring MVC`对整个`doDispatch()`方法用了嵌套的`try-catch`语句，内层的`try-catch`用于捕获`HandlerMapping`进行映射查找`HandlerExecutionChain`以及`HandlerAdapter`执行具体`Handler`时的处理异常，并将异常传入到上述`processDispatchResult()`方法中。

`processDispatchResult()`方法主要用于视图渲染，该视图可能是正常视图，也可能是针对产生的异常构造的异常视图。然后不管视图是正常视图还是异常视图，均调用`render()`方法来渲染，查看`render()`方法的具体源码如下:

```JAVA
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
   // Determine locale for request and apply it to the response.
   Locale locale =
         (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
   response.setLocale(locale); // 设置locale

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      // We need to resolve the view name. 使用ViewResolver解析视图
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request); // jsp: InternalResourceView视图
      if (view == null) { //如果视图View为空，抛出异常
         throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
               "' in servlet with name '" + getServletName() + "'");
      }
   }
   else {
      // No need to lookup: the ModelAndView object contains the actual View object.
      view = mv.getView();
      if (view == null) {
         throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
               "View object in servlet with name '" + getServletName() + "'");
      }
   }

   // Delegate to the View object for rendering.
   if (logger.isTraceEnabled()) {
      logger.trace("Rendering view [" + view + "] ");
   }
   try {
      if (mv.getStatus() != null) { // 设置Http响应状态码
         response.setStatus(mv.getStatus().value());
      }
      view.render(mv.getModelInternal(), request, response); // 调用视图View的render方法，通过Model来渲染视图
   }
   catch (Exception ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("Error rendering view [" + view + "]", ex);
      }
      throw ex;
   }
}
```

`render()`方法通过调用`resolveViewName()`方法根据视图名称解析对应的视图`View`，该方法源码如下:

```JAVA
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
      Locale locale, HttpServletRequest request) throws Exception {

   if (this.viewResolvers != null) {
      for (ViewResolver viewResolver : this.viewResolvers) {
         // InternalResourceViewResolver
         View view = viewResolver.resolveViewName(viewName, locale); 
         if (view != null) { // InternalResourceView
            return view;
         }
      }
   }
   return null;
}
```

`resolveViewName()`方法通过遍历配置的所有`ViewResolver类`根据视图名称来解析对应的视图`View`，如果找到则返回对应视图`View`，没有找到则返回`null`。默认情况下，得到InternalResourceView对象。

回到前一个`render()`方法，如果上述方法返回的视图为`null`则抛出异常，这个异常相信大多数人也见过，当开发时写错了返回的`View`视图名称时就会抛出该异常。接下来调用具体视图的`render()`方法来进行`Model`数据的渲染填充，最终构造成完整的视图。

到这里，`doDispatch()`的外层`try-catch`代码块的作用就明显了——为了捕获渲染视图时的异常。通过两层嵌套的`try-catch`，`Spring MVC`就能够捕获到三大组件`HandlerMapping、HandlerAdapter、ViewResolver`在处理用户请求时的异常，通过这样的方法能够很方便的实现统一的异常处理。

### 五、总结

通过前文的源码分析，我们能够清楚的认识到`Spring MVC`对用户请求的处理过程，进一步加深对`Spring MVC`的理解。

### 参考文章

- [SpringMVC DispatcherServlet执行流程及源码分析](https://www.jianshu.com/p/0f981efdfbbd)
- [Spring MVC 原理探秘 - 一个请求的旅行过程](http://www.tianxiaobo.com/2018/06/29/Spring-MVC-%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A7%98-%E4%B8%80%E4%B8%AA%E8%AF%B7%E6%B1%82%E7%9A%84%E6%97%85%E8%A1%8C%E8%BF%87%E7%A8%8B/)

