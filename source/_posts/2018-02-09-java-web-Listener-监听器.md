---
title: java web Listener（监听器）
tags: [java,java web,Listener]
date: 2018-02-09 20:28:26
categories:
  - java
  - java web
permalink: java-web-Listener
keywords: java web,Listener,监听器,单态登录
---

# Listener 概述

<font color="red">Listener（监听器）</font>用于**监听 Java Web 程序中的事件**，例如创建、修改、删除 session、request、context 等，**并触发相应的事件**。利用 Listener 能够用很少的代码实现比较好的效果。

<br>

# Listener 分类

## 分类

Servlet 2.5 规范中共有<font color="red">8种 Listener</font>，这8种 Listener 可以分为**三类**：
**(1) 监听 session、context、request 等的创建与销毁：HttpSessionListener、ServletContextListener、ServletRequestListener。**
<!--more-->
+ <font color="red">HttpSessionListener</font>:
  + `void sessionCreated(HttpSessionEvent e)`
  + `void sessionDestroyed(HttpSessionEvent e)`
+ <font color="red">ServletContextListener</font>:
  + `void contextInitialized(ServletContextEvent e)` 
  + `void contextDestroy(ServletContextEvent e)`
+ <font color="red">ServletRequestListener</font>:
  + `void requestInitialized(ServletRequestEvent e)`
  + `void requestDestroyed(ServletRequestEvent e)`  

**(2) 监听 session、context、request 的属性变化：HttpSessionAttributeListener、ServletContextAttributeListener、ServletRequestAttributeListener。**
+ <font color="red">ServletContextAttributeListener</font>:
```java
    void attributeAdded(ServletContextAttributeEvent e)
    void attributeRemoved(ServletContextAttributeEvent e)
    void attributeReplaced(ServletContextAttributeEvent e) 
```

    其他两个 Listener 里的方法跟这个类似，不多说。

**(3) 监听 session 内的对象：HttpSessionBindingListener、HTTPSessionActivationListener**
<1><font color="red">HttpSessionBindingListener</font>：当对象被放到 session 里时执行`valueBound(HttpSessionBindingEvent event)` 方法。当对象被从 session 里移除时执行 `valueUnbound(HttpSessionBindingEvent event)` 方法。**对象必须实现该 Listener 接口**。
<2><font color="red">HttpSessionActivationListener</font>：服务器关闭时，会将 session 里的内容保存到硬盘上，这个过程叫做序列化（钝化）。服务器重新启动时，会将 session 从硬盘重新加载。当 session 里的对象被**序列化（钝化）**时会执行`sessionWillPassivate(HttpSessionEvent event)` 方法，当对象被**重新加载（反序列化）**时执行`sessionDidActivate(HttpSessionEvent event)` 。**对象必须实现该 Listener 接口和 Serializable 接口**。

**注意：这两种 Listener 不需要在 web.xml 中配置（当然也不需要注解配置）。**。



## Listener 简单例子1

监听session、context、request的创建与销毁：

```java
package com.anye137.listener;

import javax.servlet.*;
import javax.servlet.http.*;
import javax.servlet.annotation.WebListener;;

@WebListener
public class ListenerTest implements HttpSessionListener,
        ServletContextListener, ServletRequestListener{
    
    @Override
    public void contextInitialized(ServletContextEvent e) {
        //加载servlet上下文时被调用
        ServletContext context = e.getServletContext();
        System.out.println("启动context："+context.getContextPath());
    } 
    @Override
    public void contextDestroyed(ServletContextEvent e) {
        ServletContext context = e.getServletContext();
        System.out.println("关闭context："+context.getContextPath());     
    }

    @Override
    public void sessionCreated(HttpSessionEvent e) {
        //创建session时被调用
        HttpSession session = e.getSession();       
        System.out.println("创建session，id为："+session.getId());       
    }
    @Override
    public void sessionDestroyed(HttpSessionEvent e) {
        //销毁session前被调用
        HttpSession session = e.getSession();
        System.out.println("销毁session，id为："+session.getId());      
    }
    
    @Override
    public void requestInitialized(ServletRequestEvent e) {
        //创建requests时调用
        HttpServletRequest request = (HttpServletRequest)e.getServletRequest();
        String uri = request.getRequestURI();       
        uri = uri+"?"+request.getQueryString();
        request.setAttribute("dateCreated", System.currentTimeMillis());
        System.out.println("IP "+request.getRemoteAddr()+" 请求 "+uri);      
    }
    @Override
    public void requestDestroyed(ServletRequestEvent e) {
        //销毁request时被调用
        HttpServletRequest request = (HttpServletRequest)e.getServletRequest();
        long time = System.currentTimeMillis()-(Long)request.getAttribute("dateCreated");    
        System.out.println("IP "+request.getRemoteAddr()+"请求处理结束，用时"+time+"毫秒");        
    }

}

```

这个例子中的 Listener 是用注解配置的，如果用 web.xml 配置，需要添加一下代码：

```xml
<listener>
    <listener-class>com.anye137.listener.ListenerTest</listener-class>
</listener>
```

## Listener 简单例子2
监听 session 属性变化（监听context、request与监听session类似）

```java
package com.anye137.listener;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSession;
import javax.servlet.http.HttpSessionAttributeListener;
import javax.servlet.http.HttpSessionBindingEvent;

@WebListener
public class SessionAttributeListenerTest implements HttpSessionAttributeListener{

    public void attributeAdded(HttpSessionBindingEvent e)  { 
        //添加属性时被调用
        HttpSession session = e.getSession();
        String name = e.getName();
        System.out.println("新建session属性："+name+"，值为："+e.getValue());
    }
  
    public void attributeRemoved(HttpSessionBindingEvent e)  { 
        //删除属性前被调用
        HttpSession session = e.getSession();
        String name = e.getName();
        System.out.println("删除session属性："+name+"，值为"+e.getValue());
    }
    
    public void attributeReplaced(HttpSessionBindingEvent e)  { 
        //修改属性时被调用
        HttpSession session = e.getSession();
        String name = e.getName();
        Object oldValue = e.getValue();
        System.out.println("修改session属性："+name+"，原值："+oldValue+",新值："+session.getAttribute(name));
    }	
  
}
```

<br>

# Listener 使用案例

## 1.单态登录

单态登录，就是一个账号只能在一台机器上登录，如果在其他机器上登录了，则原来的登录自动失效。这样可以防止多台机器同时使用同一个账号。

LoginSessionListener.java

```java
package com.anye137.listener;
import java.util.HashMap;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.*;

@WebListener
public class LoginSessionListener implements HttpSessionAttributeListener {
    
    //储存目前登录的用户,key为userName,value为session
    HashMap<String, HttpSession> map = new HashMap<>();
    
    @Override
    //添加session属性时被调用
    public void attributeAdded(HttpSessionBindingEvent e){        
        //新添加的属性
        String attrName = e.getName();
        
        if(attrName.equals("userName")){
            String attrValue = (String)e.getValue();
            //若map中存在该用户
            if(map.get(attrValue)!=null){
                //将以前的登录失效
                HttpSession session = map.get(attrValue);               
                session.removeAttribute("userName");                     
            }
            //新用户session存进map 
            map.put(attrValue, e.getSession());
            System.out.println("账号:"+attrValue+",登录");
        }      
    }
    
    @Override
    //删除属性前被调用
    public void attributeRemoved(HttpSessionBindingEvent e){
        //被删除的属性
        String attrName = e.getName();       
        if(attrName.equals("userName")){
            String attrValue = (String) e.getValue();
            map.remove(attrValue);
            System.out.println("账号："+attrValue+",注销");
        }
    }
    
    @Override
    //修改属性时被调用
    public void attributeReplaced(HttpSessionBindingEvent e){
        String attrName = e.getName();
        //没有注销的情况下，用另一个账号登录
        if(attrName.equals("userName")){
            String oldValue = (String) e.getValue();
            String newValue = (String) e.getSession().getAttribute("userName");
            //注意要判断新值旧值是否相等
            if(!oldValue.equals(newValue)){
                //检查新登录的账号是否在别的机器上登录过
                if(map.get(newValue)!=null){                
                    HttpSession session = map.get(oldValue);
                    session.removeAttribute("userName");               
                    map.remove(oldValue);
                }              
                map.put(newValue, e.getSession());
                System.out.println("原账号："+oldValue+"。被新账号："+newValue+"替代");
            }
            else{
                System.out.println("账号："+oldValue+",重新登录");
            }
        }
    }
    
}
```

登录界面和处理登录的 servlet 跟前面我写过的文章里，登录验证 Filter 里的代码一样，登录后的界面则增加了每隔5s自动刷新以及推出登录的代码

登录界面`login.jsp`

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
            姓名：<input type="text"  name="userName"/><br>
            <input type="submit" value="登录"/><br>
        </form>
    </body>
</html>
```

处理登录的servlet `loginServlet.java`

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
<%
    if(request!=null){
        String action = request.getParameter("action");
        if("logout".equals(action)){
            out.println("jjj");
            session.removeAttribute("userName");
            response.sendRedirect(request.getContextPath()+"/jsp/login.jsp");
        }
    }
%>
<html>
    <head>
	    <title></title>
        <script type="text/javascript">
             <!-- 每5秒刷新一次 -->
             setTimeout("location=location; ", 5000);
        </script>
    </head>
    <body>
         用户名： <%= session.getAttribute("userName") %>
         您已成功登陆
         <br>
         <a href="${pageContext.request.requestURI}?action=logout">退出</a>
    </body>
</html>
```

本例中，Listener 用 map 储存已登录的用户名及对应 session，并**监听 session 中 userName 属性的变化，作出相应的处理来实现单态登录**。

在实际应用中，我们也可以使用类 User 来储存用户更详细的信息，比如 ip 地址，登录时间等，这时 session 存储的属性值应该是 User 类对象。

<br>

<font size="5">**本文章完整代码见 [<font color="blue">Github</font>](https://github.com/anye137/Java-Web-Learning/tree/master/FilterAndListener)**</font>。

<br>



# 参考资料

(1) 《JavaWeb王者归来》