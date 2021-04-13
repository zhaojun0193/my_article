---
title: SpringMVC实现RESTful接口,关于静态资源的访问
date: 2018-4-11 12:51:00
tags: SpringMVC
---

#### 前言

* **什么是REST**

  REST全称是Representational State Transfer，中文意思是表述（编者注：通常译为表征）性状态转移。 它首次出现在2000年Roy Fielding的博士论文中，Roy Fielding是HTTP规范的主要编写者之一。 他在论文中提到："我这篇文章的写作目的，就是想在符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构。REST指的是一组架构约束条件和原则。" 如果一个架构符合REST的约束条件和原则，我们就称它为RESTful架构。

  REST本身并没有创造新的技术、组件或服务，而隐藏在RESTful背后的理念就是使用Web的现有特征和能力， 更好地使用现有Web标准中的一些准则和约束。虽然REST本身受Web技术的影响很深， 但是理论上REST架构风格并不是绑定在HTTP上，只不过目前HTTP是唯一与REST相关的实例。 所以我们这里描述的REST也是通过HTTP实现的REST。

* **REST架构的主要原则**

  网络上所有的事物都被抽象为资源

  每个资源都有一个唯一的资源标识符

  同一个资源有多重表现形式(xml，json等)

  对资源的各种操作不会改变资源标识符

  所有操作都是无状态的

  符合REST原则的架构方式即可称为RESTful

<!-- more -->

#### web.xml的相关配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4"
         xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">

    <display-name>echart-learning</display-name>

    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>

    <!--加载applicationContext.xml-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </context-param>

    <!--编码过滤器-->
    <filter>
        <filter-name>encodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>encodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <!--Spring监听器-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!--SpringMvc-->
    <servlet>
        <servlet-name>SpringMvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>SpringMvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
        
```

其中需要注意的点是:

```xml
<servlet-mapping>
        <servlet-name>SpringMvc</servlet-name>
        <url-pattern>/</url-pattern>  <!-- 这里需要配置成 / -->
</servlet-mapping>
```

#### spring-mvc.xml的相关配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <!-- 包扫描 -->
    <mvc:annotation-driven/>
    <context:component-scan base-package="com.zhaojun.controller" />

    <!-- 视图解析器 -->
    <bean id="viewResolver"
          class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/view/" />
        <property name="suffix" value=".jsp"></property>
    </bean>

    <!--静态资源映射-->
    <mvc:resources mapping="/resource/**" location="/resource/"/>
</beans>
```

需要注意的是，由于上面web.xml中,我们springmvc的拦截方式是 / ，这样会导致静态资源也被拦截住，所以需要配置静态资源的映射，这里有五种方法。我使用了第二种，下面是五种方法的相关说明。

1. 用另外一个servlet来处理静态资源。如果我们的静态资源放在webapp文件夹下的resource文件夹中，那么我们可以用名字为default的servlet来处理静态资源。因此我们需要在web.xml中加上如下配置:

   ```xml
   <servlet-mapping>
           <servlet-name>default</servlet-name>
           <url-pattern>resource/*</url-pattern>
   </servlet-mapping>
   ```

   这标识default的servlet会处理URL中为resource/*的请求，这样，当你把image，css，js等文件,放在resource文件夹中，就不会被拦截住了。

2. 采用Spring自带的 `<mvc:resources>`方法，在spring-mvc.xml进行以下配置：

   ```xml
   <mvc:annotation-driven/>
   <mvc:resources mapping="/resource/**" location="/resource/"/>
   ```

   这样，就不需要新添加一个servlet的配置来处理静态资源了。

3. 在SpringMVC3.0之后推荐使用:

   ```xml
   <mvc:default-servlet-handler/>
   ```

   在spring-mvc.xml中配置`<mvc:default-servlet-handler />`后，会在SpringMVC上下文中定义一个org.springframework.web.servlet.resource.DefaultServletHttpRequestHandler，它会像一个检查员，对进入DispatcherServlet的URL进行筛查，如果发现是静态资源的请求，就将该请求转由Web应用服务器默认的Servlet处理，如果不是静态资源的请求，才由DispatcherServlet继续处理。

   一般Web应用服务器默认的Servlet名称是"default"，因此DefaultServletHttpRequestHandler可以找到它。如果你所有的Web应用服务器的默认Servlet名称不是"default"，则需要通过default-servlet-name属性显示指定：`<mvc:default-servlet-handler default-servlet-name="所使用的Web服务器默认使用的Servlet名称" />`

以下两种在SpringMVC3.0之前可以使用。

 4. 在spring-mvc.xml中配置:

    ```xml
    <mvc:resources location="/img/" mapping="/img/**"/>   
     <mvc:resources location="/js/" mapping="/js/**"/>    
     <mvc:resources location="/css/" mapping="/css/**"/> 
    ```

5. 在web.xml进行如下配置:

    ```xml
    <servlet-mapping>  
         <servlet-name>default</servlet-name>  
         <url-pattern>*.css</url-pattern>  
    </servlet-mapping>  
      
    <servlet-mapping>  
        <servlet-name>default</servlet-name>  
        <url-pattern>*.gif</url-pattern>  
      
    </servlet-mapping>  
         
    <servlet-mapping>  
         <servlet-name>default</servlet-name>  
         <url-pattern>*.jpg</url-pattern>  
    </servlet-mapping>  
         
    <servlet-mapping>  
         <servlet-name>default</servlet-name>  
         <url-pattern>*.js</url-pattern>  
    </servlet-mapping>  
    ```

------

参考博文: [https://blog.csdn.net/suyu_yuan/article/details/52775828](https://blog.csdn.net/suyu_yuan/article/details/52775828)

转载请注明出处。