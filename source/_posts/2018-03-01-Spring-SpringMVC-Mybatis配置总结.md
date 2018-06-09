---
title: Spring+SpringMVC+MyBatis 配置总结
tags: [Spring, Spring MVC, MyBatis, java web]
date: 2018-03-01 22:44:07
categories:
  - java
  - java web
permalink: configure-ssm
keywords: Spring, Spring MVC, MyBatis, 配置
---

这篇文章主要讲 <font color='red' >MyBatis 的配置</font>， <font color='red' >Spring MVC 的配置</font>，<font color='red' >以及 SSM 整合配置</font>。此文章**仅**涉及常用的基本设置。

本文章是基于我个人的理解和知识水平，通过查阅资料所写下的笔记总结，主要是方便自己记住常用的基本配置。**更详细的知识点请参见文末的参考资料**。

<br>

<!--more -->

# MyBatis 配置

在工程的 src 文件夹下创建 MyBatis 配置文件`SqlMapConfig.xml`

一个简单例子如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//MyBatis.org//DTD Config 3.0//EN"
        "http://MyBatis.org/dtd/MyBatis-3-config.dtd">
<configuration>
    <!-- 和spring整合后 environments配置将废除-->
    <environments default="development">
        <environment id="development">
            <!-- 使用jdbc事务管理-->
            <transactionManager type="JDBC" />
            <!-- 数据库连接池,由MyBatis管理-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://localhost:3306/databaseName?characterEncoding=utf-8" />
                <property name="username" value="root" />
                <property name="password" value="admin" />
            </dataSource>
        </environment>
    </environments>

    <!-- 加载映射文件-->
    <mappers>
        <mapper resource="mapper/Student.xml"/>
    </mappers>
</configuration>
```

各常用便签配置顺序大致如下：

- configuration

  - properties

  - settings

  - typeAliases

  - environments

    - environment

      - transactionManager

      - dataSource

  - mappers

## properties 

properties 包含可外部化的，可替换的属性，可以在典型的Java属性文件实例中配置，或通过 properties 元素的子元素传入。例如：

```properties
<properties resource="org/MyBatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

然后这些属性可以在整个`SqlMapConfig.xml` 配置文件中使用。

**注意加载属性的顺序：先读取 properties 元素的子元素定义的属性。然后读取 properties 元素中 resource 加载的属性，它会覆盖已读取的同名属性。**

**（一般建议不要在 properties 元素体内定义属性，只将属性定义在 properties 文件中。）**

一个常见的例子是，<font color='red'>在 `db.properties` 中配置 dataSource 参数</font>：

```properties
jdbc.driver = com.mysql.jdbc.Driver
jdbc.url = jdbc:mysql://localhost:3306/databaseName?characterEncoding=utf-8
jdbc.username = root
jdbc.password = admin
```

同时改动 `SqlMapConfig.xml` 文件

```xml
<properties resource="db.properties">     
</properties>

<environments default="development">
    <environment id="development">
        <!-- 使用jdbc事务管理-->
        <transactionManager type="JDBC" />
        <!-- 数据库连接池,由MyBatis管理-->
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}" />
            <property name="url" value="${jdbc.url}" />
            <property name="username" value="${jdbc.username}" />
            <property name="password" value="${jdbc.password}" />
        </dataSource>
    </environment>
</environments>
```

通过这种方式配置，可以**方便对数据库参数的统一管理**

## settings

settings 可以配置 MyBatis 在运行时的行为方式，比如：**开启延迟加载**（默认没有开启），**开启二级缓存**（默认没有开启）等。

```xml
<settings>
    <!-- 打开延迟加载 的开关 -->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 将积极加载改为消极加载即按需要加载 -->
    <setting name="aggressiveLazyLoading" value="false"/>
  
    <!-- 开启二级缓存 -->
    <setting name="cacheEnabled" value="true"/>
</settings>
```

**注意：开启二级缓存除了在 settings 中设置，还需要在别的地方设置，这里不多说**

## typeAliases

typeAliases，指的是**类型别名** 。

我们在`mapper.xml` 中给 resultType 或者 parameterType 中指定映射 java 类型时，需要输入类型的全路径。例如：

```xml
<select id="findAuthorById" parameterType="int" resultType="domain.blog.Author">
    SELECT * FROM author WHERE id=#{value}
</select>
```

我们可以使用类型别名，方便开发。在 `SqlMapConfig.xml` 中配置：

```xml
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
</typeAliases>
```

 这样，我们在`mapper.xml` 中就可以使用类型别名：

```xml
<select id="findAuthorById" parameterType="int" resultType="Author">
    SELECT * FROM author WHERE id=#{value}
</select>
```

我们也可以指定包名，MyBatis 会自动扫描包中的 java beans。

```xml
<typeAliases>
  <package name="domain.blog"/>
</typeAliases>
```

这样，`domian.blog` 包中的所有 java bean 的类型别名默认就是类名（首字母大小写都可以）。如果某个类中有`@Alias` 注解，则类型别名是注解中的值。如：

```java
@Alias("myAuthor")
public class Author {
    ...
}
```

Mybatis 中有许多内置的针对常用 java 类的类型别名，常见的有：

| 别名        | 类         |
| --------- | --------- |
| _long     | long      |
| _short    | short     |
| _int      | int       |
| _double   | double    |
| _float    | float     |
| _boolean  | boolean   |
| string    | String    |
| byte      | Byte      |
| long      | Long      |
| short     | Short     |
| int       | Integer   |
| double    | Double    |
| float     | Float     |
| boolean   | Boolean   |
| date      | Date      |
| map       | Map       |
| hashmap   | HashMap   |
| list      | List      |
| arraylist | ArrayList |

## environments

environments 内可配置**事务管理**和**数据库连接**，开头已经给出了一个例子，不多说。

## mappers

mappers可配置从哪里加载映射文件，有几种配置方法：

(1) 通过 resource 加载映射文件

```xml
<mappers>
  <mapper resource="mapper/AuthorMapper.xml" />
  <mapper resource="mapper/BlogMapper.xml" />
</mappers>
```

(2) 通过 mapper 接口加载映射文件

```xml
<mappers>
  <!-- 通过mapper接口加载单个 映射文件
       遵循一些规范：需要将mapper接口类名和mapper.xml映射文件名称保持一致，且在一个目录中
       上边规范的前提是：使用的是mapper代理方法
  -->
  <mapper class="com.anye137.AuthorMapper"/>
  <mapper class="com.anye137.BlogMapper"/>
</mappers>
```

 (3) 指定 mapper 接口的包名（推荐）

```xml
<mappers>
  <!-- 批量加载mapper
       指定mapper接口的包名，mybatis自动扫描包下边所有mapper接口进行加载
       遵循一些规范：需要将mapper接口类名和mapper.xml映射文件名称保持一致，且在一个目录 中
       上边规范的前提是：使用的是mapper代理方法
  -->
  <package name="com.anye137.mapper"/>
</mappers>
```

## 总结

下面是一个配置例子。以后要配置的时候从这里粘贴，再进行小小的改动就行了，哈哈哈。

`db.properties`

```pro
<properties resource="org/MyBatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

`SqlMapConfig`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//MyBatis.org//DTD Config 3.0//EN"
        "http://MyBatis.org/dtd/MyBatis-3-config.dtd">
<configuration>
  
    <properties resource="db.properties">     
    </properties>
  
    <!-- 和spring整合后 environments配置将废除-->
    <environments default="development">
        <environment id="development">
            <!-- 使用jdbc事务管理-->
            <transactionManager type="JDBC" />
            <!-- 数据库连接池,由MyBatis管理-->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}" />
                <property name="url" value="${jdbc.url}" />
                <property name="username" value="${jdbc.username}" />
                <property name="password" value="${jdbc.password}" />
            </dataSource>
        </environment>
    </environments>
      

    <!-- 加载映射文件-->
    <mappers>
        <package name="com.anye137.mapper"/>
    </mappers>
</configuration>
```

<br>

# Spring MVC 配置

`web.xml`

```xml
<web-app id="WebApp_ID" version="2.4"
    xmlns="http://java.sun.com/xml/ns/j2ee" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee 
    http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
    <display-name>Spring MVC Application</display-name>
  
   <servlet>
      <servlet-name>springmvc</servlet-name>
      <servlet-class>
         org.springframework.web.servlet.DispatcherServlet
      </servlet-class>
      <load-on-startup>1</load-on-startup>
   </servlet>
  
   <servlet-mapping>
      <servlet-name>springmvc</servlet-name>
      <url-pattern>*.jsp</url-pattern>
   </servlet-mapping>
  
</web-app>
```

`web.xml` 文件放在应用程序的 `WebContent/WEB-INF` 目录下。在初始化 `DispatcherServlet` 时，Spring MVC 将尝试加载位于该应用程序的 `WebContent/WEB-INF` 目录中的文件名为 [servlet-name]-servlet.xml 的文件。在上面的例子中，文件名将是`springmvc-servlet.xml`。

接下来，`url-pattern`标签表明哪些 URLs 将被`DispatcherServlet` 处理。这里所有以 .jsp 结束的 HTTP 请求将由该`DispatcherServlet`处理。

如果不想使用默认文件名和默认位置，可以通过在 `web.xml` 文件中添加 servlet 监听器 `ContextLoaderListener` 自定义该文件的名称和位置，如下所示：

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  
    <!-- spring mvc的配置文件 -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springMVC.xml</param-value>
    </init-param>
  
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.jsp</url-pattern>
</servlet-mapping>
```

`springMVC.xml` 文件如下：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="
   http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
   http://www.springframework.org/schema/context 
   http://www.springframework.org/schema/context/spring-context-3.0.xsd"> 
     
   <!-- 使用注解的处理器映射器和适配器 -->
   <mvc:annotation-driven></mvc:annotation-driven>
   
   <!-- 指定Controller的包-->
   <context:component-scan base-package="com.anye137.Controller" />
  
   <!-- 视图解析器-->
   <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
      <!-- 配置jsp路径的前缀-->
      <property name="prefix" value="/WEB-INF/jsp/" />
      <!-- 配置jsp路径的后缀-->
      <property name="suffix" value=".jsp" />
   </bean> 
   
</beans>
```

(1) `mvc:annotation-driven` 标签默认加载很多的参数绑定方法，包括了默认使用注解的处理器映射器和适配器。

(2)  `context` 标签用于启用 Spring MVC 注释扫描功能，该功能允许使用注释，如 `@Controller` 和 `@RequestMapping` 等等。

(3) 上面的视图解析器中配置了 jsp 路径的前后缀。比如说：

```java
modelAndView.setViewName("hello");
return modelAndView;
```

这样，一个名称为 hello 的视图将发送给位于 `/WEB-INF/jsp/hello.jsp` 中实现的视图。

<br>

# Spring+Spring MVC+MyBatis 整合配置

终于到重头戏部分了！

其实以后应该不怎么单独使用 MyBatis 或者 Spring MVC，如果要用框架的话一般会整合几个框架，例如 <font color='red'>SSM</font>。 所以下面讲的配置就是本文的重点了。以后写 SSM 项目的话我就可以直接在这里 copy 配置了。嘻嘻！

这里分三步来整合，即 <font color='red'> 整合 dao 层（这里是mapper）</font>， <font color='red'> 整合 service 层</font> 和 <font color='red'>整合 Spring MVC</font>。整合完三部分后，再配置`web.xml` 

## 整合 dao 层

Spring 和 MyBatis 整合，通过 Spring 管理 mapper 接口

src 文件夹下创建数据库配置文件 `db.properties`

```properties
jdbc.driver = com.mysql.jdbc.Driver
jdbc.url = jdbc:mysql://localhost:3306/databaseName?characterEncoding=utf-8
jdbc.username = root
jdbc.password = admin
```

src 文件夹下创建 mybatis 文件夹，在里面创建 Mybatis 配置文件`SqlMapConfig.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 全局setting配置，根据需要添加 -->
    <settings>
        <!-- 获取数据库自增主键值 -->
        <setting name="useGeneratedKeys" value="true" />
    </settings>

    <!-- 配置别名 -->
    <typeAliases>
        <!-- 批量扫描别名 -->
        <package name="com.anye137.pojo">
    </typeAliases>

    <!-- 配置environments，这里不需要配置。由Spring管理配置 -->
    <!-- 配置mapper
    由于使用spring和mybatis的整合包进行mapper扫描，这里不需要配置了。
    必须遵循：mapper.xml和mapper.java文件同名且在一个目录
     -->
</configuration>
```

注意，这里跟上面提到的`SqlMapConfig.xml` 配置有所不同。这里没有配置 **数据源**和 **mapper**。这两个由`applicationContext-dao.xml`配置。此外，该文件还配置了 SqlSessionFactory。

src 文件夹下创建 spring 文件夹，在里面创建

`applicationContext-dao.xml`

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
    http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">

    <!-- 加载db.properties文件中的内容 -->
    <context:property-placeholder location="classpath:db.properties" />
  
    <!-- 配置数据源 ，这里使用dbcp -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driver}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />
        <property name="maxActive" value="30" />
        <property name="maxIdle" value="5" />
    </bean>

    <!-- sqlSessionFactory -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">       
        <property name="dataSource" ref="dataSource" />
        <property name="configLocation" value="classpath:mybatis/SqlMapConfig.xml" />
    </bean>
  
    <!-- mapper扫描器 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 扫描包路径，如果需要扫描多个包，中间使用半角逗号隔开 -->
        <property name="basePackage" value="com.anye137.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>

</beans>
```



## 整合 service 层

通过 Spring 管理 service 接口，以及实现事务控制

src/spring 文件夹内创建 `applicationContext-service.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- 扫描service包下所有使用注解的类型 -->
    <context:component-scan base-package="com.soecode.lyf.service" />

    <!-- 配置事务管理器 -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 数据源dataSource在applicationContext-dao.xml中配置了 -->
        <property name="dataSource" ref="dataSource"/>
    </bean>
  
    <!-- 配置基于注解的声明式事务 -->
    <tx:annotation-driven transaction-manager="transactionManager" />
   
</beans>
```

## 整合 Spring MVC

这一步跟上面讲到的 Spring MVC 配置类似。

src/spring 文件夹内创建`springMVC.xml`

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="
   http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
   http://www.springframework.org/schema/context 
   http://www.springframework.org/schema/context/spring-context-3.0.xsd"> 
     
   <!-- 使用注解的处理器映射器和适配器 -->
   <mvc:annotation-driven></mvc:annotation-driven>
   
   <!-- 指定Controller的包-->
   <context:component-scan base-package="com.anye137.Controller" />
  
   <!-- 视图解析器-->
   <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
      <!-- 配置jsp路径的前缀-->
      <property name="prefix" value="/WEB-INF/jsp/" />
      <!-- 配置jsp路径的后缀-->
      <property name="suffix" value=".jsp" />
   </bean> 
   
</beans>
```

## 配置 web.xml

`web.xml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         id="WebApp_ID" version="3.0">


    <!-- 加载配置文件 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>       
        <param-value>classpath:spring/applicationContext-*.xml</param-value>
    </context-param>
  
    <listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- Spring MVC 配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/springMVC.xml</param-value>
        </init-param>
    </servlet>

    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>*.jsp</url-pattern>
    </servlet-mapping>

    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.htm</welcome-file>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
  
</web-app>
```

至此，SSM 整合大功告成！

**注意所有配置文件的路径，不要搞错了**。

<br>

# 参考资料

(1) [MyBatis学习笔记(5)-配置文件](http://brianway.github.io/2016/03/08/MyBatis-learn-5-configuration/)
(2) [MyBatis Configuration xml](http://www.mybatis.org/mybatis-3/configuration.html)
(3) [Spring MVC 中文文档](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/)
(4) [Spring MVC 学习笔记](http://brianway.github.io/tag/#SpringMVC)