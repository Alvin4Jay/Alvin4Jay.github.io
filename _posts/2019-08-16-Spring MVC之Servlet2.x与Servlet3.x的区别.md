---
layout:     post
title:      Spring MVC之Servlet2.x与Servlet3.x的区别
subtitle:   ServletContextListener/ServletContainerInitializer
date:       2019-08-16
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Spring MVC
---

# Spring MVC之Servlet2.x与Servlet3.x的区别

### 一、基于Servlet 2.x的Web应用

##### 1.定义Servlet

```java
public class EchoServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doGet(request, response);
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        PrintWriter writer = response.getWriter();
        writer.println("Hello world!");
        writer.flush();
    }
}
```

##### 2.定义Filter

```java
public class EchoFilter implements Filter {
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("before...");
        chain.doFilter(request, response);
        System.out.println("after...");
    }

    public void destroy() {

    }
}
```

##### 3.部署描述文件web.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <servlet>
        <servlet-name>EchoServlet</servlet-name>
        <servlet-class>servlet2.EchoServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>EchoServlet</servlet-name>
        <url-pattern>/echo</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>EchoFilter</filter-name>
        <filter-class>servlet2.EchoFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>EchoFilter</filter-name>
        <url-pattern>/echo/*</url-pattern>
    </filter-mapping>

</web-app>
```

在Servlet2.x中，Servlet、Filter的定义必须通过部署描述文件**web.xml**进行描述。

### 二、Servlet3.x特性

##### 1.使用注解定义Servlet、Filter、Listener(无需定义web.xml)

Servlet:

```java
// Servlet
@WebServlet(name = "echoServlet", urlPatterns = "/echo")
public class EchoServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        PrintWriter writer = response.getWriter();
        writer.println("Hello world!+1");
        writer.flush();
    }
}

@WebServlet(urlPatterns = "/hello", loadOnStartup = 1,
        initParams = {
                @WebInitParam(name = "db", value = "jdbc://..."),
                @WebInitParam(name = "dbuser", value = "fred"),
                @WebInitParam(name = "dbpasswd", value = "fred")
        })
public class HelloServlet extends HttpServlet {

    @Override
    public void init(ServletConfig config) throws ServletException {
        String jdbc = config.getInitParameter("db"); // 获取Servlet初始化参数
        System.out.println("db: " + jdbc);
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().println("hello+2");
        resp.getWriter().flush();
    }

}
```

Filter:

```java
@WebFilter(urlPatterns = "/hello", dispatcherTypes = DispatcherType.REQUEST)
public class HelloServletFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // ...
        chain.doFilter(request, response);
        System.out.println("filtered...");
    }

    @Override
    public void destroy() {

    }
}
```

Listener:

```java
@WebListener("Auto configure Struts")
public class MyListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ServletContext context = sce.getServletContext();
        // register the servlet
        ServletRegistration.Dynamic registration =
                context.addServlet("echo", "servlet30.EchoServlet");
        // set init param
        registration.setInitParameter("com.jay.anno.config", "/WEB-INF/struts-com.jay.anno.config.xml");
        registration.setLoadOnStartup(2);
        // add mapping
        registration.addMapping("/echo");
        // Annotate with @MultipartConfig 处理multipart/form-data
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {

    }
}
```

##### 2.Multipart支持

```java
@WebServlet("/multi")
@MultipartConfig(location = "/tmp")
public class MultiPartServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Part part = req.getPart("myFile"); // 读取文件
        InputStream in = part.getInputStream();
        // read file...
        byte[] data = new byte[1024];
        int len;
        while((len=in.read(data)) != -1) {
            System.out.println(new String(data, 0, len));
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
```

##### 3.动态配置Servlet、Filter、Listener

###### (1) 使用ServletContextListener

ServletContextListener由应用程序开发人员提供实现，其contextInitialized方法在应用程序启动时调用。

```java
// 编程式配置
@WebListener("Auto configure Struts")
public class MyListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ServletContext context = sce.getServletContext();
        // register the servlet 注册Servlet
        ServletRegistration.Dynamic registration =
                context.addServlet("echo", "servlet30.EchoServlet");
        // set init param 设置初始化参数
        registration.setInitParameter("com.jay.anno.config", "/WEB-INF/struts-com.jay.anno.config.xml");
        registration.setLoadOnStartup(2);
        // add mapping 添加映射
        registration.addMapping("/echo");
        // Annotate with @MultipartConfig 处理multipart/form-data
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {

    }
}
```

以上代码通过定义ServletContextListener实现，在其contextInitialized回调的时候动态配置EchoServlet。

###### (2) 使用ServletContainerInitializer

ServletContainerInitializer主要由框架实现，在应用程序启动时调用其onStartUp()方法，负责框架的初始化动作，如初始化控制器Servlet(对应Spring MVC，DispatcherServlet)、连接数据库等。下面给出简单示例：

```java
@HandlesTypes(StrategyHandler.class) // 感兴趣的类型
public class MyServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
      	// 在这里动态配置Servlet、Filter、Listener
        System.out.println("scanned clazz:");
        c.forEach(clazz -> {
            System.out.println("\t" + clazz.getCanonicalName());
        });
    }
}
```

然后在META-INF/services目录下定义该实现，文件名为`javax.servlet.ServletContainerInitializer`，文件内容为`initializer.MyServletContainerInitializer`，Web容器通过SPI机制发现该实现。

输出：

```java
scanned clazz:
	servlet30.handler.impl.BizStrategyHandler
```

##### 4.异步Servlet

非常适用于以下情况的 web 应用程序：

- **长处理时间或伪长处理时间**(**长轮询**)
- 等待资源释放 — 如数据库连接
- 等待事件发生 — 如聊天消息
- 等待缓慢服务的响应 — 如 web 服务

```java
@WebServlet(urlPatterns = "/async", asyncSupported = true) // Servle声明支持异步
public class AsyncServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // Put the request in async mode 1.将请求置于异步模式，释放Tomcat请求线程
        AsyncContext asyncContext = req.startAsync();

        // Create an instance of handler
        SlowResourceHandler handler = new SlowResourceHandler(asyncContext);

        // Starts the process 2.启动工作线程处理该请求
        asyncContext.start(handler);

        // Request will not commit on return 对于请求的客户端，请求不会返回
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}

public class SlowResourceHandler implements Runnable {

    private AsyncContext asyncContext;

    public SlowResourceHandler(AsyncContext asyncContext) {
        this.asyncContext = asyncContext;
    }

    public void run() {
        try {
            int i = ThreadLocalRandom.current().nextInt(3) + 1;
            TimeUnit.SECONDS.sleep(i); // 模拟长时间的处理
            asyncContext.getResponse().getWriter().println("finished..." + i);
        } catch (Exception e) {
            //
        }
        // 3.只有工作线程处理完毕，调用AsyncContext.complete()方法，客户端请求才返回
        asyncContext.complete();
    }
}
```