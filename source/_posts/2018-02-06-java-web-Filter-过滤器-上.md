---
title: java web Filter（过滤器）上
tags: [java,java web,Filter,防盗链,字符编码]
date: 2018-02-06 16:34:51
categories: 
  - java
  - java web
permalink: java-web-filter-1
keywords: filter,java web,过滤器,防盗链,字符编码,
---

# Filter 概述

<font color="red">Filter（过滤器）</font>用于在 servlet 之外对 request 或者 response 进行拦截、修改，甚至可以拒绝、重定向或者转发 request。Filter 提出了 <font color="red">FilterChain</font> 的概念，一个 FilterChain 包括多个 Filter。客户端请求 request 在抵达 servlet 之前会经过 FilterChain 里的所有 Filter，服务器响应 response 在从 servlet 抵达客户端浏览器之前也会经过 FilterChain 里的所有Filter。Filter处理过程如图所示：

<!-- more-->

<img src="/images/2018/filter.jpg" width = "300" height = "200" alt="filter" align=center />

<br>

# Filter 接口

一个 Filter 必须实现 `javax.servlet.Filter` 接口。Filter 接口有**三个方法**，源代码如下：

```java
//Filter.java
package javax.servlet;
import java.io.IOException;
public interface Filter{
    /**
     * web程序启动时调用此方法，用于初始化该Filter
     * @param config 可以从该参数中获取初始化参数以及ServletContext信息等
     * @throws ServletException
     */
    public void init(FilterConfig config) throws ServletException;
    
    /**
     * 客户端请求服务器时会经过
     * @param request 请求 
     * @param response 响应
     * @param chain
     * @throws ServletException
     * @throws IOException
     */
    public void doFilter(ServletRequest request,ServletResponse response,
            FilterChain chain) throws ServletException, IOException;
  
    /**
     * web程序关闭时调用此方法，用于销毁一些资源
     */
    public void destroy();
}
```

其中，`init()` 与`destroy()` 方法只会分别在 web 程序加载和卸载的时候调用。而`doFilter()` 方法每次有客户端请求时都会被调用一次。

一个简单的`doFilter()` 方法例子

```java
public void doFilter(ServletRequest request, ServletResponse response,
                     FilterChain chain) throws IOException, ServletException{
    System.out.println("request 被处理之前... ");
    //一定要执行这句
    chain.doFilter(request, response);
    System.out.println("request 被处理之后，response 抵达客户端之前... ");
}
```

其中`chain.doFilter(request, response)` <font color="red">将 request 递交给 FilterChain 中的下一个 Filter，如果所有的 Filter 都执行完了则交给 servlet 处理</font>。

<br>

# Filter 配置

## 在 web.xml 中配置

一个例子

```xml
<filter>
    <filter-name>filterName</filter-name>
    <filter-class>com.anye137.SomeFilter</filter-class>
    <init-param>
        <param-name>paramName</param-name>
        <param-value>paramValue</param-value>
    </init-param>
</filter>

<filter-mapping>
    <filter-name>filterName</filter-name>
    <url-pattern>*.jsp</url-pattern>
    <url-pattern>*.do</url-pattern>
    <servlet-name>someServlet</servlet-name>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
</filter-mapping>
```

一个 Filter 需要配置`<filter>` 和`<filter-mapping>` 标签

`<filter>` 标签需要配置`<filter-name>` 和`<filter-class>` 以及0或多个`<init-param>` （初始化参数）

`<filter-mapping>` 配置 Filter 的拦截规则
* `<url-pattern>` 配置 URL 映射，可以配置多个。在上面的例子中，如果一个请求访问的 URL 结尾是 .do 或者是 .jsp，就可以匹配此 Filter。通过使用`<url-pattern>` ，我们不仅可以拦截 servlet 的请求，还可以拦截其他资源，例如图片、css 文件等
* `<servlet-name>` 配置 servlet 名称映射，即对应的 servlet 会被Filter 拦截，可以配置多个。有时候，许多 URL 会映射到同一个 servlet，这时候使用`<servlet-name>` 相较于`<url-pattern>` 比较方便
* `<dispatcher>` 配置派发请求的方式，可以配置多个。如果没有配置任何dispatcher，则默认为 REQUEST。`<dispatcher>` 取值有：
    1. **REQUEST**：直接请求，例如直接访问 URL。
    2. **FORWARD**：当调用`RequestDispatcher` 的`forward` 方法或者使用`<jsp: forward>` 标签时，将触发这些请求
    3. **INCLUDE**：当调用`RequestDispatcher` 的`include` 方法或者使用`<jsp: include>` 标签时，将触发这些请求（记住它与`<%@ include %>` 是不同的）
    4. **ERROR**：访问处理 http 错误（如404 Not Found等）的错误页面的请求
    5. **ASYNC**：（这个我也不太懂，不多说。。。）

一个 web 程序可以配置多个 Filter，<font color="red">如何比较两个Filter 执行顺序</font>：   
(1) 首先，URL 映射优先级高于 servlet 名称映射，不管其在 web.xml 中配置的顺序如何      
(2) 其次，如果同为 URL 映射或者同为 servlet 名称映射，则比较`<servlet-mapping>` 在 web.xml 中出现的顺序

## 使用注解配置

Filter 也可以使用注解配置，下面配置跟上面是等价的

```java
import javax.servlet.annotation.WebFilter;
@WebFilter(
    filterName = "myFilter",
    urlPattern = {"*.do", "*.jsp"},
    servletName = {"someServlet"},
    dispatcherTypes = {DispatcherType.REQUEST, DispatcherType.FORWARD}
    initParams = {
        @WebInitParam(name = "paramName", value = "paramValue")
    }
)
public class SomeFilter implements Filter{
    //......
}
```

使用注解配置的主要<font color="red">缺点</font>是，**不能对 FilterChain 上的 Filter 进行排序**。

<br>

# Filter 常用例子

我们可以通过在`doFilter()` 方法内编写代码，达到以下目的：
(1) 通过控制是否调用 `chain.doFilter()` ，来决定是否访问目标资源。例如：防盗链 Filter、权限验证 Filter。
(2) 调用 `chain.doFilter()` 之前，做某些处理。例如：字符编码 Filter。
(3) 调用 `chain.dpFilter()` 之后，做某些处理。例如：内容替换 Filter，GZIP压缩 Filter
（当然，我们也可以综合使用上面三种方式）

下面介绍下一些例子。

## 1. 防盗链 Filter

实现效果：如果其他的网站引用本网站的图片等资源，将会显示一个错误图片，只有本站内的网页引用时，图片才会正常显示。

```java
package com.anye137.filter;
import java.io.IOException;
import javax.servlet.*;
import javax.servlet.http.*;

public class AntiStealingLinkFilter implements Filter {    
    
    public AntiStealingLinkFilter() {}	
    public void init(FilterConfig fConfig) throws ServletException {}
    public void destroy() {}

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {	
        HttpServletRequest req=(HttpServletRequest)request;
        
        //请求来源网址
        String referer=req.getHeader("referer");
      
        //如果请求不是来自本站
        if(referer==null||!referer.contains(req.getServerName())){
            req.getRequestDispatcher("/error.jpg").forward(req, response);
        }
        else{  //如果来自本站
            chain.doFilter(req, response);
        }
        
    }
}

```

`doFilter()` 方法中，**首先要将 `request` 转为 `HttpServletRequest` 类的对象 req**。只有这样，才能使用一些针对 http 协议的方法，例如 `String getMethod()` 、`String getHeader(String name)` 、`HttpSession getSession()` 等。

在 web.xml 中配置如下：

```xml
<filter>
    <filter-name>antiStealingLinkFilter</filter-name>
    <filter-class>com.anye137.filter.AntiStealingLinkFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>antiStealingLinkFilter</filter-name>
    <url-pattern>/images/*</url-pattern>
</filter-mapping>
```

在此配置下，从别的网站请求 **WebContent/images** 文件夹下的资源，或者**直接输入资源 URL** 就只会显示我们之前在 Filter 类中设置好的 error.jpg。本站访问，则正常显示。

<br>

## 2.字符编码 Filter

字符编码 Filter 是最常用的 Filter 之一，常用来解决 Tomcat 等服务器里 request，response 乱码的问题。本例中我们使用 Filter 来解决**全站中文乱码**问题。

<font color="red">对 response 的设置：</font>
```java
response.setCharacterEncoding("utf-8");
response.setContentType("text/html;charset=utf-8");
```
其中，`setCharacterEncoding` 用于解决`response.getWriter()` 输出字符流的乱码问题，其作用是将 response 中的数据解码后发向浏览器。`setContentType` 用于设置浏览器用我们指定的方式解码，然后呈现出来。

<font color="red">对 request 的设置：</font>
(1) 若是 **POST** 方式，直接设置
```java
request.setCharacterEncoding("utf-8"); `
```
(2) 若是 **GET** 方式，
```java
paramValue = request.getParameter(paramName)
paramValue = new String(paramValue.getBytes("ISO8859-1"),"utf-8");
```

如果对每个 Servlet 都编写上面的代码，则会显得冗赘。使用 Filter 只需一次编写，就能解决全站编码问题。同时，为了实现对 POST 和 GET 方式的分开处理，我们后面实现了**自定义的 Request 类：EncodingRequest.java**

Filter 代码： CharacterEncodingFilter.java

```java
package com.anye137.filter;
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class CharacterEncodingFilter implements Filter {
	
    //是否启用该Filter
    private boolean enabled; 
	
    @Override
    public void init(FilterConfig config) throws ServletException {
        // 初始化时从 web.xml中加载参数		
        enabled = config.getInitParameter("enabled").trim().equalsIgnoreCase("true");
    }
	
    @Override
    public void destroy() {}

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        //如果启用了Filter
        if(enabled){
             //设置response编码	
             response.setCharacterEncoding("utf-8");
             response.setContentType("text/html;charset=utf-8");
             //设置request编码
             HttpServletRequest req=(HttpServletRequest)request;	  
             EncodingRequest er=new EncodingRequest(req);
		
             chain.doFilter(er, response);
        }
        else
            chain.doFilter(request, response);		
    }
    
    //内部类
    class EncodingRequest extends HttpServletRequestWrapper{	    	        
        //代码
    }

}

```

在`CharacterEncodingFilter` 类中，先获取初始化参数`enabled` 判断是否开启该 Filter，如果开启，则先设置 response 编码，然后设置 request 编码。我们使用自定义的`EncodingRequest` 内部类来实现对`request` 字符编码的设置（**注意区分 GET 和 POST 处理方式的不同**）。

内部类`EncodingRequest` 代码如下：

```java
public class EncodingRequest extends HttpServletRequestWrapper{
    private HttpServletRequest req;
    public EncodingRequest(HttpServletRequest request){
        super(request);
        this.req = request;
        if(req.getMethod().equalsIgnoreCase("POST")){
            try {
                req.setCharacterEncoding("utf-8");
            } catch (UnsupportedEncodingException e) {					
                e.printStackTrace();
            }
        }
    }
    @Override
    public String getParameter(String name){
        String value = null;	        
        if(req!=null){
            value = req.getParameter(name);
            if(req.getMethod().equalsIgnoreCase("GET")){
                value = req.getParameter(name);
                if(value!=null){
                    try{
                        value = new String(value.getBytes("ISO-8859-1"),"utf-8");
                    }catch(UnsupportedEncodingException e){
                        e.printStackTrace();
                    }
                }
            }   
        }           
        return value;	        	       
    }
}
```

Filter 在 web.xml 中配置如下：

```xml
<filter>
    <filter-name>characterEncodingFilter</filter-name>  
    <filter-class>com.anye137.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>enabled</param-name>
        <param-value>true</param-value>
    </init-param> 	
</filter>
  
<filter-mapping>
    <filter-name>characterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

测试页面 `testEncoding.jsp` :

```jsp
<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
    <head>
        <title></title> 
    </head>
  
    <body>
        <h1>post提交</h1>
        <form action="${pageContext.request.contextPath }/testEncoding" method="post">
    	    姓名：<input type="text"  name="name"/><br>
    	    <input type="submit" value="提交"/><br>
        </form>
      
        <h1>get提交</h1>
        <form action="${pageContext.request.contextPath }/testEncoding" method="get">
            姓名：<input type="text"  name="name"/><br>
    	    <input type="submit" value="提交"/><br>
        </form>
    </body>
</html>

```

处理提交参数的 `testEncoding.java` :

```java
package com.anye137.test;
import java.io.IOException;  
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;  

@WebServlet("/testEncoding")
public class TestEncoding extends HttpServlet {  
  
    public void doGet(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {
      //输出信息，测试是否乱码
        response.getWriter().write("Hello, 测试乱码！输入姓名为： "+request.getParameter("name"));                 
    }  
  
    public void doPost(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        doGet(request, response);  
    }  
  
}  
```

输入`testEncoding.jsp` 所在网址，填写信息提交后，将由 `TestEncoding` servlet 来处理提交的信息。该 servlet 会输出一条信息，以此判断是否乱码。至此，我们已经完成了 CharacterEncodingFilter 的编写及测试。

**根据从网上查阅的资料，对 POST 方式和 GET 方式的编码处理是不一样的，我上面的代码也是这样。然而，我在测试的时候发现，把 GET 方式的编码处理方法应用于 POST 方式，也是行得通的，这样代码更简洁点。所以，为什么网上的资料是分开处理的呢？处理字符编码的代码的原理又是啥？这个以后有时间研究下**。

<br>

# 总结

<font color="red">**Filter（过滤器）是一个很有弹性的机制，功能很强大，而且与servlet、jsp等没有任何的耦合，可自由拆卸。
如果配置了多个 Filter，则执行会有先有后，彼此之间还可能会相互影响，要注意正确配置 Filter 的顺序**</font>。

下篇博客将继续写 Filter 使用案例。

**本文章完整代码见 [<font color="blue">Github</font>](https://github.com/anye137/Java-Web-Learning/tree/master/FilterAndListener)**。

<br>

#  参考资料

(1) 《Java Web 王者归来》 
(2) 《Java Web 高级编程》
(3)  [servlet Filter ](http://blog.csdn.net/nokiaisacat/article/details/51326914) from CSDN
(4) [Servlet 中文乱码问题及解决方案剖析](http://blog.csdn.net/xiazdong/article/details/7217022) from CSDN
(5) [解决全站字符乱码问题](https://github.com/codingXiaxw/filter)  from GitHub