---
title: 《跟我学Shiro》学习笔记 第三章:授权
date: 2018-4-16 17:30:26
tags: Shiro
---

#### 前言

本章我们来学习权限认证，在开始前我们先明白一些基本的概念。

**主体（**Subject**）**：即访问应用的用户，在Shiro中使用Subject代表该用户。用户只有授权后才允许访问相应的资源。

**资源（**Resource**）**：在应用中用户可以访问的任何东西，比如访问JSP页面、查看/编辑某些数据、访问某个业务方法、打印文本等等都是资源。用户只要授权后才能访问。

**权限（**Permission**）**：安全策略中的原子授权单位，通过权限我们可以表示在应用中用户有没有操作某个资源的权力。即权限表示在应用中用户能不能访问某个资源，如：访问用户列表页面、查看/新增/修改/删除用户数据（即很多时候都是CRUD（增查改删）式权限控制）等等。

如上可以看出，权限代表了用户有没有操作某个资源的权利，即反映在某个资源上的操作允不允许，不反映谁去执行这个操作。所以后续还需要把权限赋予给用户，即定义哪个用户允许在某个资源上做什么操作（权限），Shiro不会去做这件事情，而是由实现人员提供。

<!-- more -->

**角色**（Role）：角色代表了操作集合，可以理解为权限的集合，一般情况下我们会赋予用户角色而不是权限，即这样用户可以拥有一组权限，赋予权限时比较方便。典型的如：项目经理、技术总监、CTO、开发工程师等都是角色，不同的角色拥有一组不同的权限。

**隐试式角色**：即直接通过角色来验证用户有没有操作权限，粒度较粗。

**显示角色**：在程序中通过权限控制谁能访问某个资源，角色聚合一组权限集合；这样假设哪个角色不能访问某个资源，只需要从角色代表的权限集合中移除即可；无须修改多处代码；即

粒度是以资源/实例为单位的；粒度较细。

请google搜索“RBAC”和“RBAC新解”分别了解“基于角色的访问控制”“基于资源的访问控制(Resource-Based Access Control)”。

#### 3.1授权方式

Shiro支持三种授权方式：

* 编程式：通过写if/else授权代码块完成： 

  ```java
  Subject subject = SecurityUtils.getSubject();  
  if(subject.hasRole(“admin”)) {  
      //有权限  
  } else {  
      //无权限  
  }  
  ```

* 注解式：通过在执行的Java方法上放置相应的注解完成：

  ```java
  @RequiresRoles("admin")  
  public void hello() {  
      //有权限  
  }   
  ```

  没有权限将抛出相应异常。

* JSP/GSP标签：在JSP/GSP页面通过相应的标签完成：

  ```html
  <shiro:hasRole name="admin">  
  <!— 有权限 —>  
  </shiro:hasRole>   
  ```

  后面会详细介绍如何使用。

#### 3.2授权

* **3.2.1 基于角色的访问控制（隐式角色）**

  1. 在ini配置文件配置用户拥有的角色（shiro-role.ini）

     ```ini
     [users]  
     zhang=123,role1,role2  
     wang=123,role1   
     ```

     规则即：“用户名=密码,角色1，角色2”，如果需要在应用中判断用户是否有相应角色，就需要在相应的Realm中返回角色信息，也就是说Shiro不负责维护用户-角色信息，需要应用提供，Shiro只是提供相应的接口方便验证，后续会介绍如何动态的获取用户角色。

  2. 测试用例（com.zhaojun.shiro.chapter3.RoleTest.testHasRole()）

     ```java
     	@Test
         public void testHasRole(){
             String username = "wang";
             String password = "123";
             //登录
             login("classpath:shiro-role.ini",username,password);
             //判断是否拥有角色
             log.info(username + "是否拥有角色role1:"+SecurityUtils.getSubject().hasRole("role1"));
             log.info(username + "是否拥有角色role2:"+SecurityUtils.getSubject().hasRole("role2"));
             log.info(username + "是否拥有角色role1与role2:"+
                     Arrays.toString(SecurityUtils.getSubject().hasRoles(Arrays.asList("role1","role2"))));

         }
     ```

     输出日志：

     ```shll
     21:45:02.389 [main] INFO  com.zhaojun.shiro.chapter3.RoleTest - wang是否拥有角色role1:true
     21:45:02.390 [main] INFO  com.zhaojun.shiro.chapter3.RoleTest - wang是否拥有角色role2:false
     21:45:02.391 [main] INFO  com.zhaojun.shiro.chapter3.RoleTest - wang是否拥有角色role1与role2:[true, false]
     ```

     Shiro提供了hasRole/hasRole用于判断用户是否拥有某个角色/某些权限；但是没有提供如hashAnyRole用于判断是否有某些权限中的某一个。 

     ```java
     @Test(expected = UnauthorizedException.class)
         public void testCheckRole(){
             String username = "wang";
             String password = "123";
             //登录
             login("classpath:shiro-role.ini",username,password);
             //断言拥有角色
             SecurityUtils.getSubject().checkRole("role2");

             SecurityUtils.getSubject().checkRoles("role1","role2");
         }
     ```

     Shiro提供的checkRole/checkRoles和hasRole/hasAllRoles不同的地方是它在判断为假的情况下会抛出UnauthorizedException异常。

  到此基于角色的访问控制（即隐式角色）就完成了，这种方式的缺点就是如果很多地方进行了角色判断，但是有一天不需要了那么就需要修改相应代码把所有相关的地方进行删除；这就是粗粒度造成的问题。

* **3.2.2基于资源的访问控制（显示角色）**

  1. 在ini配置文件配置用户拥有的角色及角色-权限关系（shiro-permission.ini） 

     ```ini
     [users]
     zhang=123,role1,role2
     wang=123,role1
     [roles]
     role1=user:create,user:update
     roel2=user:create,user:delete
     ```

  2. 测试用例（com.zhaojun.shiro.chapter3.PermissionTest.testPermission()）

     ```java
     	@Test
         public void testPermission(){
             String username = "zhang";
             String password = "123";
             login("classpath:shiro-permission.ini",username,password);
             //判断是否拥有权限
             log.info(username + "是否拥有user:create权限："+SecurityUtils.getSubject().isPermitted("user:create"));
             log.info(username+"："+SecurityUtils.getSubject().isPermittedAll("user:create","user:update"));
             log.info(username + "是否拥有user:delete权限："+SecurityUtils.getSubject().isPermitted("user:delete"));
         }
     ```

     Shiro提供了isPermitted和isPermittedAll用于判断用户是否拥有某个权限或所有权限，也没有提供如isPermittedAny用于判断拥有某一个权限的接口。

     ```java
     @Test(expected = UnauthorizedException.class)
         public void testCheckPermission(){
             login("classpath:shiro-permission.ini", "zhang", "123");
             Subject subject = SecurityUtils.getSubject();
             //断言拥有权限：user:create
             subject.checkPermission("user:create");
             //断言拥有权限：user:delete and user:update
             subject.checkPermissions("user:delete", "user:update");
             //断言拥有权限：user:view 失败抛出异常
             subject.checkPermissions("user:view");
         }
     ```

     与checkRole一样，若没有相应权限会抛出UnauthorizedException异常。

     到此基于资源的访问控制（显示角色）就完成了，也可以叫基于权限的访问控制，这种方式的一般规则是“资源标识符：操作”，即是资源级别的粒度；这种方式的好处就是如果要修改基本都是一个资源级别的修改，不会对其他模块代码产生影响，粒度小。但是实现起来可能稍微复杂点，需要维护“用户——角色，角色——权限（资源：操作）”之间的关系。  

#### 3.3 Permission

  ##### 字符串权限通配

规则：“资源标识符：操作：对象实例ID”  即对哪个资源的哪个实例可以进行什么操作。其默认支持通配符权限字符串，“:”表示资源/操作/实例的分割；“,”表示操作的分割；“*”表示任意资源/操作/实例。

* **3.3.1 单个资源单个权限** 

  ```java
  subject.checkPermissions("system:user:update"); 
  ```

  用户拥有资源“system:user”的“update”权限。


* **3.3.2 单个资源多个权限**

  ```ini
  role41=system:user:update,system:user:delete  
  ```

  然后通过如下代码判断

  ```java
  subject.checkPermissions("system:user:update", "system:user:delete");  
  ```

  用户拥有资源“system:user”的“update”和“delete”权限。如上可以简写成：

  ```ini
  role42="system:user:update,delete" 
  ```

  接着可以通过如下代码判断 

  ```java
  subject.checkPermissions("system:user:update,delete");  
  ```

  通过“system:user:update,delete”验证"system:user:update, system:user:delete"是没问题的，但是反过来是规则不成立。

* **3.3.3 单个资源全部权限**

  ```ini
  role51="system:user:create,update,delete,view"  
  ```

  然后通过如下代码判断 

  ```java
  subject.checkPermissions("system:user:create,delete,update:view");  
  ```

  用户拥有资源“system:user”的“create”、“update”、“delete”和“view”所有权限。如上可以简写成：

  ```ini
  role52=system:user:*  
  ```

  表示角色52拥有system:user的所有权限。如上可以简写成（推荐上面的写法）：

  ```ini
  role53=system:user  
  ```

  然后通过如下代码判断

  ```java
  subject.checkPermissions("system:user:*");  
  subject.checkPermissions("system:user");   
  ```

  通过“system:user:*”验证“system:user:create,delete,update:view”可以，但是反过来是不成立的。

* **3.3.4 所有资源全部权限**

  ```ini
  role61=*:view  
  ```

  然后通过如下代码判断:

  ```java
  subject.checkPermissions("user:view");  
  ```

  用户拥有所有资源的“view”所有权限。假设判断的权限是“"system:user:view”，那么需要“role5=*:*:view”这样写才行。

* **3.3.5实例级别的权限**

  1. 单个实例权限

     ```ini
     role71=user:view:1  
     ```

     对资源user的1实例拥有view权限。

     然后通过如下代码判断 

     ```java
     subject.checkPermissions("user:view:1");  
     ```

  2. 单个实例多个权限

     ```ini
     role72="user:update,delete:1"  
     ```

     对资源user的1实例拥有update、delete权限。

     然后通过如下代码判断

     ```java
     ubject.checkPermissions("user:delete,update:1");  
     subject.checkPermissions("user:update:1", "user:delete:1"); 
     ```

  3. 单个实例所有权限

     ```ini
     role73=user:*:1  
     ```

     对资源user的1实例拥有所有权限。

     然后通过如下代码判断 :

     ```java
     subject.checkPermissions("user:auth:1");  
     ```

  4. 所有实例所有权限

     ```ini
     role74=user:*:*  
     ```

     对资源user的1实例拥有所有权限。

     然后通过如下代码判断 :

     ```java
     subject.checkPermissions("user:view:1", "user:auth:2"); 
     ```

     ​

* **3.3.6 Shiro 对权限字符串缺失部分的处理**

  如“user:view”等价于“user:view:*”；而“organization”等价于“organization:*”或者“organization:*:*”。可以这么理解，这种方式实现了前缀匹配。

  另外如“user:*”可以匹配如“user:delete”、“user:delete”可以匹配如“user:delete:1”、“user:*:1”可以匹配如“user:view:1”、“user”可以匹配“user:view”或“user:view:1”等。即*可以匹配所有，不加*可以进行前缀匹配；但是如“*:view”不能匹配“system:user:view”，需要使用“*:*:view”，即后缀匹配必须指定前缀（多个冒号就需要多个*来匹配）。

* **3.3.7WildcardPermission**

  如下两种方式是等价的：

  ```java
  subject.checkPermission("menu:view:1");  
  subject.checkPermission(new WildcardPermission("menu:view:1"));  
  ```

  因此没什么必要的话使用字符串更方便。

  ​	


#### 3.4授权流程

![授权流程](http://oucja4p5v.bkt.clouddn.com//image/shiro/%E6%8E%88%E6%9D%83%E6%B5%81%E7%A8%8B.png)



流程如下：

1、首先调用Subject.isPermitted*/hasRole*接口，其会委托给SecurityManager，而SecurityManager接着会委托给Authorizer；

2、Authorizer是真正的授权者，如果我们调用如isPermitted(“user:view”)，其首先会通过PermissionResolver把字符串转换成相应的Permission实例；

3、在进行授权之前，其会调用相应的Realm获取Subject相应的角色/权限用于匹配传入的角色/权限；

4、Authorizer会判断Realm的角色/权限是否和传入的匹配，如果有多个Realm，会委托给ModularRealmAuthorizer进行循环判断，如果匹配如isPermitted*/hasRole*会返回true，否则返回false表示授权失败。

ModularRealmAuthorizer进行多Realm匹配流程：

1、首先检查相应的Realm是否实现了实现了Authorizer；

2、如果实现了Authorizer，那么接着调用其相应的isPermitted*/hasRole*接口进行匹配；

3、如果有一个Realm匹配那么将返回true，否则返回false。

如果Realm进行授权的话，应该继承AuthorizingRealm，其流程是：

1.1、如果调用hasRole*，则直接获取AuthorizationInfo.getRoles()与传入的角色比较即可；

1.2、首先如果调用如isPermitted(“user:view”)，首先通过PermissionResolver将权限字符串转换成相应的Permission实例，默认使用WildcardPermissionResolver，即转换为通配符的WildcardPermission；

2、通过AuthorizationInfo.getObjectPermissions()得到Permission实例集合；通过AuthorizationInfo. getStringPermissions()得到字符串集合并通过PermissionResolver解析为Permission实例；然后获取用户的角色，并通过RolePermissionResolver解析角色对应的权限集合（默认没有实现，可以自己提供）；

3、接着调用Permission. implies(Permission p)逐个与传入的权限比较，如果有匹配的则返回true，否则false。 

----

张开涛的博客：[http://jinnianshilongnian.iteye.com/category/305053](http://jinnianshilongnian.iteye.com/category/305053)

我的博客：[https://zhaojun0193.github.io](https://zhaojun0193.github.io)

本文代码地址：[https://github.com/zhaojun0193/shiro-example](https://github.com/zhaojun0193/shiro-example)

  