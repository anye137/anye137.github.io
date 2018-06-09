---
title: java web Filter（过滤器）下
tags: [java,java web,Filter]
date: 2018-02-08 22:43:03
categories: 
  - java
  - java web
permalink: java-web-filter-2
keywords: filter,java web,过滤器,登录验证,内容替换
---

# Filter 常用例子

## 3. 登录验证 Filter

我们在浏览网站的时候，有时候是没有登录的，或者因为登录很久后自动掉出来。可能有一些请求需要<font color="red">判断用户是否处于登录状态</font> ，以及该用户的权限。Java Web 程序一般使用 **session** 或者 **cookie** 来记录用户是否登录。**登录验证 Filter** 是将 request 提交给 servlet 之前，对 session 或者 cookie 进行校验。如果没有相应的登录信息，或者权限不够，则进行相应的处理，比如跳转到登录界面。

下面的例子中，我们仅判断用户是否处于登录状态。

<!--more-->

PrivilegeFilter.java

```java
package com.anye137.filter;
import java.io.IOException;
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.*;

@WebFilter(filterName="privilegeFilter",urlPatterns="/jsp/afterLogin.jsp")
public class PrivilegeFilter implements Filter{
    
    public void init(FilterConfig config) throws ServletException{}
    public void destroy(){}
    
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException{
        
        HttpServletRequest req = (HttpServletRequest)request;
        HttpServletResponse res = (HttpServletResponse)response;
        HttpSession session = req.getSession();
        
        String name = (String) session.getAttribute("userName");
        if(name==null || name.trim().equals(""))
            res.sendRedirect(req.getContextPath()+"/jsp/login.jsp");
        else
            chain.doFilter(req, response);
    }

 }
```

登录界面 `login.jsp` 

```jsp
<%@ page language="java" contentType="text/html; charset=utf-8"
    pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
        <title></title>
    </head>
  
    <body>
        <form action="${pageContext.request.contextPath}/loginServlet" method="post">
            姓名：<input type="text"  name="name"/><br>
            <input type="submit" value="登录"/><br>
        </form>
    </body>
</html>
```

处理登录的 servlet `LoginServlet.java`

```java
package com.anye137.test;
import java.io.IOException;  
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;  

@WebServlet("/loginServlet")
public class LoginServlet extends HttpServlet {  
  
    public void doGet(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        
        HttpSession session = request.getSession();
        session.setAttribute("userName", request.getParameter("userName"));
        response.sendRedirect("jsp/afterLogin.jsp");
    }  
  
    public void doPost(HttpServletRequest request, HttpServletResponse response)  
            throws ServletException, IOException {  
        doGet(request, response);  
    }  
  
}  
```

登录后的界面`afterLogin.jsp`

```jsp
<%@ page language="java" contentType="text/html; charset=utf-8"
    pageEncoding="utf-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
       <title></title>
    </head>
    <body>
        用户名： <%= session.getAttribute("name") %>
        您已成功登陆
    </body>
</html>
```

运行效果：通过`login.jsp` 页面可以登录进入 `afterLogin.jsp` 页面，如果没有登录直接输入`afterLogin.jsp` 的网址，则会跳转到登录界面。

注意，上面代码中并没有设置 request 和 response 编码，因为我在上一篇文章中已经写好了 `CharacterEncodingFilter` 。当然，如果不想用字符编码 Filter，我们也可以在 servlet 中简单设置下编码。

<br>

## 4.内容替换 Filter

有时候需要对网站的内容进行控制，**防止输出非法内容或者敏感内容**。常规的解决办法是在保存进数据库之前对非法敏感内容进行替环，或者在 servlet 里输出到客户端时进行替环。这两种解决方案都有很大的局限性，例如每个 servlet 都要进行替换、工作量大，与业务耦合比较严重等。

使用<font color="red">内容替换 Filter</font> 则方便得多，其工作原理是：**在 servlet 内将内容输出到 response 时，response 将内容缓存起来，在 Filter 中进行替换，然后再输出到客户端浏览器。由于默认的 response 并不能严格地缓存输出内容，因此需要自定义一个具备缓存功能的 response**。

OutputReplaceFilter.java

```java
package com.anye137.filter;
import java.io.*;
import java.util.Properties;
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.annotation.WebInitParam;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletResponseWrapper;

@WebFilter(
    filterName="outputReplaceFilter", 
    urlPatterns="/*",
    initParams={@WebInitParam(name="file",value="/WEB-INF/sensitive.properties")}  
)

public class OutputReplaceFilter implements Filter {
    
    //非法词、敏感词配置在 properties文件中
    private Properties pp = new Properties();
    
    public OutputReplaceFilter() {}
      
    public void init(FilterConfig config) throws ServletException {
        //properties文件名
        String file = config.getInitParameter("file");
        //文件位置
        String realPath = config.getServletContext().getRealPath(file);
        
        try{
            //如果properties文件是ISO-8859-1编码，可以直接用这句：pp.load(new FileInputStream(realPath));
            //如果文件是utf-8编码，且包含中文，需用下面两句
            BufferedInputStream in = new BufferedInputStream(new FileInputStream(realPath));    
            pp.load(new InputStreamReader(in, "utf-8"));                       
        }catch(IOException e){}
    }
	public void destroy() {}

	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
		System.out.println("replace");
	    HttpServletResponse res = (HttpServletResponse)response;
	    //使用自定义response
	    OutputReplaceResponse orr = new OutputReplaceResponse(res);
	    
	    //doFilter，使用自定义 response orr
	    chain.doFilter(request, orr);
	    
	    //得到response输出内容
	    String output = orr.getCharArrayWriter().toString();
	    
	    //遍历所有敏感词
	    for(Object obj: pp.keySet()){
	        String key = (String)obj;
	        //替换敏感词
	        output = output.replace(key, pp.getProperty(key));
	    }
	    
	    //通过原来的response的getWriter()方法输出
	    PrintWriter out = response.getWriter();
	    //真正输出到客户端
	    out.write(output);		
	}
	
    //自定义response类
	class OutputReplaceResponse extends HttpServletResponseWrapper{                //字符数组Writer 
	}
	
}

```

自定义的`response` 类`OutputReplaceResponse`

```java
class OutputReplaceResponse extends HttpServletResponseWrapper{
        
	    //字符数组Writer
	    private CharArrayWriter caw=new CharArrayWriter();
        public OutputReplaceResponse(HttpServletResponse response) {
            super(response);          
        }
        @Override
        public PrintWriter getWriter() throws IOException{
            //返回字符数组Writer，缓存内容
            return new PrintWriter(caw);
        }
        
        public CharArrayWriter getCharArrayWriter(){
            return caw;
        }	    
	}
```

本例中，自定义的 response 只是一个**“伪装”**的 response。servlet会通过它输出内容到客户端，但是<font color="red">它内部只是将内容缓存起来了，并没有真正输出到客户端。最终输出到客户端还是通过原来的 response 完成的</font>。

词库配置 `sensitive.properties`

```properties
# 自动更正
Chna = China
# 自动替换
赌博 = **
色情 = **
情色 = **
```

我把 properties 文件设置为 utf-8 编码的 （eclipse中右键文件名，点Properties 设置就行了），在上面的 Filter 中是这样加载的：

```java
BufferedInputStream in = new BufferedInputStream(new FileInputStream(realPath));    
pp.load(new InputStreamReader(in, "utf-8"));
```

使用 **utf-8** 编码的话，在 properties 文件中直接输入中文会正常显示。
如果把文件编码设置为 **ISO-8859-1**，这时在编辑器输入中文，只能显示出中文对应的 Unicode 编码，但是对后续的操作其实影响不大。加载此文件只需一行代码：

```java
pp.load(new FileInputStream(realPath));
```

<br>
测试页面`testReplace.jsp`

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
        <title></title>
    </head>
    <body>
        Chna <br/>
        <br/>
        色情 <br/>
        赌博 <br/>
        情色 <br/>
    </body>
</html>
```

在浏览器中输入测试页面网址，可以看到替换后的效果。

上面 Filter 是在`init()` 方法中加载 properties 文件的，Filter `inti()` 方法只在 web 程序第一次运行时调用一次。所以如果词库更新的话，需要**重启** web 程序。

Filter 还有许多常用例子，例如**缓存Filter**、**GZIP压缩Filter**等，这里就不一一详述了。

<br>

本文详细代码见 [<font color="blue">**Github**</font>](https://github.com/anye137/Java-Web-Learning/tree/master/FilterAndListener)



<br>

# 参考资料

(1) 《JavaWeb 王者归来》
(2) [Java读写.properties文件实例，解决中文乱码问题](http://blog.csdn.net/qq_27093465/article/details/70765870) from CSDN