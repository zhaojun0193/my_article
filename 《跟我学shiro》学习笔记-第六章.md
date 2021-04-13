---
title: 《跟我学Shiro》学习笔记 第六章:与Web集成
date: 2018-5-15 11:04:18
tags: Shiro
---

Shiro提供了对Web集成的支持，其通过一个ShiroFilter入口来拦截需要安全控制的URL，然后进行相应的控制，ShiroFilter类似于如Strut2/SpringMVC这种web框架的前端控制器，其是安全控制的入口点，其负责读取配置（如ini配置文件），然后判断URL是否需要登录/权限等工作。 

#### 6.1准备环境

1. 创建webapp应用

   我们利用maven构建我们的项目，使用jetty-maven-plugin插件。具体配置可以查看GitHub源码中的pom.xml。

2. 导入相关依赖

   Servlet：

   ```xml
   <dependency>
       <groupId>javax.servlet</groupId>
       <artifactId>javax.servlet-api</artifactId>
       <version>3.0.1</version>
       <scope>provided</scope>
   </dependency>
   ```

   <!-- more -->

   Shiro-web：

   ```xml
   <dependency>
       <groupId>org.apache.shiro</groupId>
       <artifactId>shiro-web</artifactId>
       <version>1.2.2</version>
   </dependency>
   ```

   其他的依赖参考GitHub源码中的pom.xml。

#### 6.2 ShiroFilter入口

```xml
<listener>
     <listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
</listener>
```

通过EnvironmentLoaderListener来创建相应的WebEnvironment，并自动绑定到ServletContext，默认使用IniWebEnvironment实现。 

可以通过如下配置修改默认实现及其加载的配置文件位置： 

```xml
<context-param>  
   <param-name>shiroEnvironmentClass</param-name>  
   <param-value>org.apache.shiro.web.env.IniWebEnvironment</param-value>  
</context-param>  
<context-param>  
    <param-name>shiroConfigLocations</param-name>  
    <param-value>classpath:shiro.ini</param-value>  
</context-param>   
```

shiroConfigLocations默认是“/WEB-INF/shiro.ini”，IniWebEnvironment默认是先从/WEB-INF/shiro.ini加载，如果没有就默认加载classpath:shiro.ini。 

#### 6.3ini配置

shiro.ini

```ini
[main]
#默认是/login.jsp
authc.loginUrl=/login
roles.unauthorizedUrl=/unauthorized
perms.unauthorizedUrl=/unauthorized
[users]
zhang=123,admin
wang=123
[roles]
admin=user:*,menu:*
[urls]
/login=anon
/unauthorized=anon
/static/**=anon
/authenticated=authc
/role=authc,roles[admin]
/permission=authc,perms["user:create"]
```

其中最重要的就是[urls]部分的配置，其格式是： “url=拦截器[参数]，拦截器[参数]”；即如果当前请求的url匹配[urls]部分的某个url模式，将会执行其配置的拦截器。比如anon拦截器表示匿名访问（即不需要登录即可访问）；authc拦截器表示需要身份认证通过后才能访问；roles[admin]拦截器表示需要有admin角色授权才能访问；而perms["user:create"]拦截器表示需要有“user:create”权限才能访问。 

**url模式使用Ant风格模式**

Ant路径通配符支持?、*、**，注意通配符匹配不包括目录分隔符“/”：

**?****：匹配一个字符**，如”/admin?”将匹配/admin1，但不匹配/admin或/admin2；

*******：匹配零个或多个字符串**，如/admin*将匹配/admin、/admin123，但不匹配/admin/1；

***\*****：匹配路径中的零个或多个路径，如/admin/**将匹配/admin/a或/admin/a/b。

**url模式匹配顺序**

url模式匹配顺序是按照在配置中的声明顺序匹配，即从头开始使用第一个匹配的url模式对应的拦截器链。如：

```ini
/bb/**=filter1  
/bb/aa=filter2  
/**=filter3   
```

如果请求的url是“/bb/aa”，因为按照声明顺序进行匹配，那么将使用filter1进行拦截。 

拦截器将在下一节详细介绍。接着我们来看看身份验证、授权及退出在web中如何实现。 

**1.  身份验证**

**1.1 配置身份验证的url**

```ini
/authenticated=authc  
/role=authc,roles[admin]  
/permission=authc,perms["user:create"]   
```

即访问这些地址时会首先判断用户有没有登录，如果没有登录默会跳转到登录页面，默认是/login.jsp，可以通过在[main]部分通过如下配置修改：  

```ini
authc.loginUrl=/login  
```

**1.2 登录servlet**

```java
@WebServlet("/login")
public class LoginServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        req.getRequestDispatcher("/WEB-INF/view/login.jsp").forward(req,resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String resultMessage = null;

        String username = req.getParameter("username");
        String password = req.getParameter("password");

        Subject subject = SecurityUtils.getSubject();

        UsernamePasswordToken token = new UsernamePasswordToken(username,password);

        try{
            subject.login(token);
        }catch (UnknownAccountException e){
            resultMessage = "用户名/密码错误";
        }catch (IncorrectCredentialsException e){
            resultMessage = "用户名/密码错误";
        }catch (AuthenticationException e){
            //其他错误，比如锁定，如果想单独处理请单独catch处理
            resultMessage = "其他错误：" + e.getMessage();
        }

        if(resultMessage != null){
            req.setAttribute("result",resultMessage);
            req.getRequestDispatcher("/WEB-INF/view/login.jsp").forward(req,resp);
        }else {
            req.getRequestDispatcher("/WEB-INF/view/loginSuccess.jsp").forward(req,resp);
        }
    }
}
```

1. doGet请求时展示登录页面；

2. doPost时进行登录，登录时收集username/password参数，然后提交给Subject进行登录。如果有错误再返回到登录页面；否则跳转到登录成功页面（此处应该返回到访问登录页面之前的那个页面，或者没有上一个页面时访问主页）。

**1.3 login.jsp**

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<form action="/login" method="post">
    用户名：<input type="text" name="username">
    密 码：<input type="password" name="password">
    <input type="submit" value="登录"> <span style="color: red">${result}</span>
</form>
</body>
</html>
```

**1.4测试**
首先输入http://localhost:8080/login 进行登录,登录成功后访问http://localhost:8080/authenticated 来显示当前登录的用户。

当前实现的一个缺点就是，永远返回到同一个成功页面（比如首页），在实际项目中比如支付时如果没有登录将跳转到登录页面，登录成功后再跳回到支付页面；对于这种功能大家可以在登录时把当前请求保存下来，然后登录成功后再重定向到该请求即可。 

**2. 基于Basic的拦截器身份验证**

**2.1 shiro-basicfilterlogin.ini配置**

```ini
[main]
authcBasic.applicationName=please login

perms.unauthorizedUrl=/unauthorized
roles.unauthorizedUrl=/unauthorized
[users]
zhang=123,admin
wang=123

[roles]
admin=user:*,menu:*

[urls]
/role=authcBasic,roles[admin]
```

1、authcBasic是org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter类型的实例，其用于实现基于Basic的身份验证；applicationName用于弹出的登录框显示信息使用，如图： 

![](http://oucja4p5v.bkt.clouddn.com//image/shiro/basic.png)

2、[urls]部分配置了/role地址需要走authcBasic拦截器，即如果访问/role时还没有通过身份验证那么将弹出如上图的对话框进行登录，登录成功即可访问。 

**2.2 web.xml**

把shiroConfigLocations改为shiro-basicfilterlogin.ini即可。

**2.3 测试**

输入http://localhost:8080/role ，会弹出之前的Basic验证对话框输入“zhang/123”即可登录成功进行访问。

**3.基于表单的拦截器身份验证**

基于表单的拦截器身份验证和【1】类似，但是更简单，因为其已经实现了大部分登录逻辑；我们只需要指定：登录地址/登录失败后错误信息存哪/成功的地址即可。 

**3.1 shiro-formfilterlogin.ini**

```ini
[main]
authc.loginUrl=/formfilterlogin
authc.usernameParam=username
authc.passwordParam=password
authc.successUrl=/
authc.failureKeyAttribute=shiroLoginFailure

perms.unauthorizedUrl=/unauthorized
roles.unauthorizedUrl=/unauthorized

[users]
zhang=123,admin
wang=123

[roles]
admin=user:*,menu:*

[urls]
/static/**=anon
/formfilterlogin=authc
/role=authc,roles[admin]
/permission=authc,perms["user:create"]

```

1、authc是org.apache.shiro.web.filter.authc.FormAuthenticationFilter类型的实例，其用于实现基于表单的身份验证；通过loginUrl指定当身份验证时的登录表单；usernameParam指定登录表单提交的用户名参数名；passwordParam指定登录表单提交的密码参数名；successUrl指定登录成功后重定向的默认地址（默认是“/”）（如果有上一个地址会自动重定向带该地址）；failureKeyAttribute指定登录失败时的request属性key（默认shiroLoginFailure）；这样可以在登录表单得到该错误key显示相应的错误消息； 

**3.2 web.xml**

把shiroConfigLocations改为shiro- formfilterlogin.ini即可。 

**3.3 登录servlet**

```java
@WebServlet("/formfilterlogin")
public class FormFilterLoginServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doPost(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String errorClassName = (String) req.getAttribute("shiroLoginFailure");
        if (UnknownAccountException.class.getName().equals(errorClassName)) {
            req.setAttribute("error", "用户名/密码错误");
        } else if (IncorrectCredentialsException.class.getName().equals(errorClassName)) {
            req.setAttribute("error", "用户名/密码错误");
        } else if (errorClassName != null) {
            req.setAttribute("error", "未知错误：" + errorClassName);
        }
        req.getRequestDispatcher("/WEB-INF/view/formfilterlogin.jsp").forward(req, resp);
    }
}

```

**3.4 测试**

输入http://localhost:8080/role ，会跳转到“/formfilterlogin”登录表单，提交表单如果authc拦截器登录成功后，会直接重定向会之前的地址“/role”；假设我们直接访问“/formfilterlogin”的话登录成功将直接到默认的successUrl。 



**4. 授权（角色/权限验证）**

**4.1 shiro.ini**

```ini
[main]  
roles.unauthorizedUrl=/unauthorized  
perms.unauthorizedUrl=/unauthorized  
 [urls]  
/role=authc,roles[admin]  
/permission=authc,perms["user:create"]  
```

通过unauthorizedUrl属性指定如果授权失败时重定向到的地址。roles是org.apache.shiro.web.filter.authz.RolesAuthorizationFilter类型的实例，通过参数指定访问时需要的角色，如“[admin]”，如果有多个使用“，”分割，且验证时是hasAllRole验证，即且的关系。Perms是org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter类型的实例，和roles类似，只是验证权限字符串。 

**4.2 RoleServlet/PermissionServlet**

```java
@WebServlet("/role")
public class RoleServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Subject subject = SecurityUtils.getSubject();

        subject.checkRoles("admin");

        req.getRequestDispatcher("/WEB-INF/view/hasRole.jsp").forward(req,resp);
    }
}
```

**4.4测试**

首先访问http://localhost:8080/login ，使用帐号“zhang/123”进行登录，再访问/role或/permission时会跳转到成功页面（因为其授权成功了）；如果使用帐号“wang/123”登录成功后访问这两个地址会跳转到“/unauthorized”即没有授权页面。

**5. 退出**

**5.1 shiro.ini**

```ini
[urls]
/logout=anon
```

指定/logout使用anon拦截器即可，即不需要登录即可访问。 

**5.2 LogoutServlet**

```java
@WebServlet("/logout")
public class LogoutServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Subject subject = SecurityUtils.getSubject();

        subject.logout();

        req.getRequestDispatcher("/WEB-INF/view/logoutSuccess.jsp").forward(req,resp);
    }
}
```

**5.3测试**

首先访问http://localhost:8080/login ，使用帐号“zhang/123”进行登录，登录成功后访问/logout即可退出。 

Shiro也提供了logout拦截器用于退出，其是org.apache.shiro.web.filter.authc.LogoutFilter类型的实例，我们可以在shiro.ini配置文件中通过如下配置完成退出：  

```ini
[main]  
logout.redirectUrl=/login  
  
[urls]  
/logout2=logout   
```

通过logout.redirectUrl指定退出后重定向的地址；通过/logout2=logout指定退出url是/logout2。这样当我们登录成功后然后访问/logout2即可退出。 

----

张开涛的博客：[http://jinnianshilongnian.iteye.com/category/305053](http://jinnianshilongnian.iteye.com/category/305053)

我的博客：[https://zhaojun0193.github.io](https://zhaojun0193.github.io)

本文代码地址：[https://github.com/zhaojun0193/shiro-example](https://github.com/zhaojun0193/shiro-example)

