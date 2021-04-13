---
title: 《跟我学Shiro》学习笔记 第二章:身份验证
date: 2018-4-13 17:18:56
tags: shiro
---

### 前言

上一章主要对Shiro功能，运行原理，架构设计进行了介绍，这一章我们主要学习Shiro的身份验证。本章的代码我会上传至GitHub，链接见文末。

**身份验证**：即在应用中谁能证明他就是他本人。一般提供如他们的身份ID一些标识信息来表明他就是他本人，如提供身份证，用户名/密码来证明。

在shiro中，用户需要提供principals （身份）和credentials（证明）给shiro，从而应用能验证用户身份：

**principals**：身份，即主体的标识属性，可以是任何东西，如用户名、邮箱等，唯一即可。一个主体可以有多个principals，但只有一个Primary principals，一般是用户名/密码/手机号。

**credentials**：证明/凭证，即只有主体知道的安全值，如密码/数字证书等。

最常见的principals和credentials组合就是用户名/密码了。接下来先进行一个基本的身份认证。

<!-- more -->

### 2.1环境准备

引入相关依赖

```xml
<dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.9</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.10</version>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.0.13</version>
        </dependency>

        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.1.3</version>
        </dependency>
    
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-core</artifactId>
            <version>1.2.2</version>
        </dependency>
    </dependencies>
```

### 2.2登录/登出

1. 首先准备一些	用户身份/凭据（shiro.ini）

   ```ini
   [users]  
   zhang=123  
   wang=123  
   ```

   这里使用ini配置文件，[users]指定了两个主体：zhang/123、wang/123。

2. 测试用例（com.zhaojun.shiro.chapter2.LoginLogoutTest）

   ```java
   	@Test
       public void testHelloWorld() {
           //1. 获取SecurityManagerFactory，此处使用ini配置文件初始化SecurityManager
           Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
           /*
            2. 得到SecurityManager实例，并绑定到SecurityUtils上
            接着获取SecurityManager并绑定到SecurityUtils，这是一个全局设置，设置一次即可
            SecurityManager是在org.apache.shiro.mgt包下
            */
           SecurityManager securityManager = factory.getInstance();
           SecurityUtils.setSecurityManager(securityManager);
           /*
              3. 得到Subject及创建用户名/密码身份验证Token（即用户身份/凭证）
              通过SecurityUtils得到Subject，其会自动绑定到当前线程；
              如果在web环境在请求结束时需要解除绑定；然后获取身份验证的Token，如用户名/密码
            */
           Subject subject = SecurityUtils.getSubject();
           UsernamePasswordToken token = new UsernamePasswordToken("wang", "123");

           try {
               /*
                  4. 登录，即验证身份
                  调用subject.login方法进行登录，其会自动委托给SecurityManager.login方法进行登录
                */
               subject.login(token);
               log.info("用户登录成功");
           } catch (AuthenticationException e) {
               /*
               如果身份验证失败请捕获AuthenticationException或其子类，常见的如： DisabledAccountException（禁用的帐号）、
               LockedAccountException（锁定的帐号）、UnknownAccountException（错误的帐号）、
               ExcessiveAttemptsException（登录失败次数过多）、IncorrectCredentialsException （错误的凭证）、
               ExpiredCredentialsException（过期的凭证）等，具体请查看其继承关系；对于页面的错误消息展示，
               最好使用如“用户名/密码错误”而不是“用户名错误”/“密码错误”，防止一些恶意用户非法扫描帐号库
                */
               log.error("身份验证失败:{}", e.getLocalizedMessage());
           }

           Assert.assertEquals(true, subject.isAuthenticated());

           //6. 退出
           subject.logout();
       }
   ```

   从以上代码可以总结出身份验证的步骤：

   1. 收集用户身份/凭证，即如用户名/密码；
   2. 调用Subject.login进行登录，如果失败将得到相应的AuthenticationException异常，根据异常提示用户错误信息；否则登录成功；
   3. 最后调用Subject.logout进行退出操作。

   如上测试的几个问题：

   1. 用户名/密码硬编码在ini配置文件，以后需要改成如数据库存储，且密码需要加密存储；
   2. 用户身份Token可能不仅仅是用户名/密码，也可能还有其他的，如登录时允许用户名/邮箱/手机号同时登录。

   ### 2.3身份认证流程

   ![身份认证流程图](http://oucja4p5v.bkt.clouddn.com//image/shiro/%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B.png)

流程如下：

1. 首先调用Subject.login(token)进行登录，其会自动委托给Security Manager，调用之前必须通过SecurityUtils. setSecurityManager()设置；
2. SecurityManager负责真正的身份验证逻辑；它会委托给Authenticator进行身份验证；
3. Authenticator才是真正的身份验证者，Shiro API中核心的身份认证入口点，此处可以自定义插入自己的实现；
4. Authenticator可能会委托给相应的AuthenticationStrategy进行多Realm身份验证，默认ModularRealmAuthenticator会调用AuthenticationStrategy进行多Realm身份验证；
5. Authenticator会把相应的token传入Realm，从Realm获取身份验证信息，如果没有返回/抛出异常表示身份验证失败了。此处可以配置多个Realm，将按照相应的顺序及策略进行访问。

### 2.4Realm

Realm：域，Shiro从从Realm获取安全数据（如用户、角色、权限），就是说SecurityManager要验证用户身份，那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法；也需要从Realm得到用户相应的角色/权限进行验证用户是否能进行操作；可以把Realm看成DataSource，即安全数据源。如我们之前的ini配置方式将使用org.apache.shiro.realm.text.IniRealm。

org.apache.shiro.realm.Realm接口如下： 

```java
String getName(); //返回一个唯一的Realm名字  
boolean supports(AuthenticationToken token); //判断此Realm是否支持此Token  
AuthenticationInfo getAuthenticationInfo(AuthenticationToken token)  
 throws AuthenticationException;  //根据Token获取认证信息  
```

#### 2.4.1单Realm配置

1. 自定义Realm实现：

   ```java
   @Slf4j
   public class MyRealm1 implements Realm {
       public String getName() {
           return "myRealm1";
       }

       public boolean supports(AuthenticationToken authenticationToken) {
           //仅支持UsernamePasswordToken类型的token
           return authenticationToken instanceof UsernamePasswordToken;
       }

       public AuthenticationInfo getAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
           String username = (String) authenticationToken.getPrincipal();

           String password = new String((char[])authenticationToken.getCredentials());

           log.info("username:{},password:{}",username,password);

           if(!"zhang".equals(username)){
               log.info("用户名错误");
               throw new UnknownAccountException();
           }

           if(!"123".equals(password)){
               log.info("密码错误");
               throw new IncorrectCredentialsException();
           }
           //若身份认证成功，返回一个AuthenticationInfo实现
           return new SimpleAuthenticationInfo(username,password,getName());
       }
   }
   ```

2. ini配置文件指定自定义Realm实现(shiro-realm.ini)  

   ```ini
   #申明一个realm
   myRealm1= com.zhaojun.shiro.chapter2.realm.MyRealm1
   #指定securityManager的realms实现
   securityManager.realms=$myRealm1
   ```

   通过$name来引入之前的realm定义

3. 测试用例请参考com.zhaojun.shiro.chapter2.LoginLogoutTest的testCustomRealm测试方法，只需要把之前的shiro.ini配置文件改成shiro-realm.ini即可。

#### 2.4.2多Realm配置

1. ini配置文件（shiro-multi-realm.ini）  

   ```ini
   #申明多个realm
   myRealm1= com.zhaojun.shiro.chapter2.realm.MyRealm1
   myRealm2= com.zhaojun.shiro.chapter2.realm.MyRealm2
   #指定securityManager的realms实现
   securityManager.realms=$myRealm1,$myRealm2
   ```

   securityManager会按照realms指定的顺序进行身份认证。此处我们使用显示指定顺序的方式指定了Realm的顺序，如果删除“securityManager.realms=\$myRealm1\$myRealm2”，那么securityManager会按照realm声明的顺序进行使用（即无需设置realms属性，其会自动发现），当我们显示指定realm后，其他没有指定realm将被忽略，如“securityManager.realms=\$myRealm1”，那么myRealm2不会被自动设置进去。

2. 测试用例请参考com.zhaojun.shiro.chapter2.LoginLogoutTest的testCustomMultiRealm测试方法，只需要把之前的shiro.ini配置文件改成shiro-multi-realm.ini即可。


#### 2.4.3Shiro默认提供的Realm

![Realm继承关系](http://oucja4p5v.bkt.clouddn.com//image/shiro%E9%BB%98%E8%AE%A4Realm.png)

以后一般继承AuthorizingRealm（授权）即可；其继承了AuthenticatingRealm（即身份验证），而且也间接继承了CachingRealm（带有缓存实现）。其中主要默认实现如下：

**org.apache.shiro.realm.text.IniRealm**：[users]部分指定用户名/密码及其角色；[roles]部分指定角色即权限信息；

**org.apache.shiro.realm.text.PropertiesRealm**： user.username=password,role1,role2指定用户名/密码及其角色；role.role1=permission1,permission2指定角色及权限信息；

**org.apache.shiro.realm.jdbc.JdbcRealm**：通过sql查询相应的信息，如“select password from users where username = ?”获取用户密码，“select password, password_salt from users where username = ?”获取用户密码及盐；“select role_name from user_roles where username = ?”获取用户角色；“select permission from roles_permissions where role_name = ?”获取角色对应的权限信息；也可以调用相应的api进行自定义sql；

#### 2.4.4JdbcRealm使用

1. 引入数据库相关依赖

   ```xml
   <dependency>
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId>
       <version>5.1.25</version>
   </dependency>
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>druid</artifactId>
       <version>0.2.23</version>
   </dependency>
   ```

2. 新建数据库shiro，并新建三张表，users（用户名/密码）、user_roles（用户/角色）、roles_permissions（角色/权限）。具体请参照/shiro-example-chapter2/src/main/sql/shiro.sql。并添加一条用户记录：zhang/123。

3. ini配置（shiro-jdbc-realm.ini） 

   ```ini
   jdbcRealm=org.apache.shiro.realm.jdbc.JdbcRealm
   dataSource=com.alibaba.druid.pool.DruidDataSource
   dataSource.driverClassName=com.mysql.jdbc.Driver
   dataSource.url=jdbc:mysql://localhost:3306/shiro
   dataSource.username=root
   dataSource.password=123
   jdbcRealm.dataSource=$dataSource
   securityManager.realms=$jdbcRealm
   ```

4. 测试用例参照com.zhaojun.shiro.chapter2.LoginLogoutTest的testJDBCRealm方法。与之前的方法基本一致，只是替换引用的配置文件即可。

   JdbcRealm会自己进行数据库查询，JdbcRealm.java相关源码如下：

   ```java
   //查询语句
   protected String authenticationQuery = "select password from users where username = ?";

   //获取认证信息
   protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
           UsernamePasswordToken upToken = (UsernamePasswordToken)token;
           String username = upToken.getUsername();
           if (username == null) {
               throw new AccountException("Null usernames are not allowed by this realm.");
           } else {
               Connection conn = null;
               SimpleAuthenticationInfo info = null;

               try {
                   String salt;
                   try {
                       conn = this.dataSource.getConnection();
                       String password = null;
                       salt = null;
                       switch(this.saltStyle) {
                       case NO_SALT:
                           password = this.getPasswordForUser(conn, username)[0];
                           break;
                       case CRYPT:
                           throw new ConfigurationException("Not implemented yet");
                       case COLUMN:
                           String[] queryResults = this.getPasswordForUser(conn, username);
                           password = queryResults[0];
                           salt = queryResults[1];
                           break;
                       case EXTERNAL:
                           password = this.getPasswordForUser(conn, username)[0];
                           salt = this.getSaltForUser(username);
                       }
   				   //查询结果为空，抛出账户错误异常
                       if (password == null) {
                           throw new UnknownAccountException("No account found for user [" + username + "]");
                       }
   				  //若查询结果不为空，则创建一个用户认证信息
                       info = new SimpleAuthenticationInfo(username, password.toCharArray(), this.getName());
                       if (salt != null) {
                           info.setCredentialsSalt(Util.bytes(salt));
                       }
                   } catch (SQLException var12) {
                       salt = "There was a SQL error while authenticating user [" + username + "]";
                       if (log.isErrorEnabled()) {
                           log.error(salt, var12);
                       }

                       throw new AuthenticationException(salt, var12);
                   }
               } finally {
                   JdbcUtils.closeConnection(conn);
               }

               return info;
           }
       }
   //通过用户名查询密码，判断用户是否存在
   private String[] getPasswordForUser(Connection conn, String username) throws SQLException {
           boolean returningSeparatedSalt = false;
           String[] result;
           switch(this.saltStyle) {
           case NO_SALT:
           case CRYPT:
           case EXTERNAL:
               result = new String[1];
               break;
           case COLUMN:
           default:
               result = new String[2];
               returningSeparatedSalt = true;
           }

           PreparedStatement ps = null;
           ResultSet rs = null;

           try {
               ps = conn.prepareStatement(this.authenticationQuery);
               ps.setString(1, username);
               //执行查询
               rs = ps.executeQuery();

               for(boolean foundResult = false; rs.next(); foundResult = true) {
                   if (foundResult) {
                       throw new AuthenticationException("More than one user row found for user [" + username + "]. Usernames must be unique.");
                   }

                   result[0] = rs.getString(1);
                   if (returningSeparatedSalt) {
                       result[1] = rs.getString(2);
                   }
               }
           } finally {
               JdbcUtils.closeResultSet(rs);
               JdbcUtils.closeStatement(ps);
           }

           return result;
       }
   ```
#### 2.5Authenticator及AuthenticationStrategy

   Authenticator的职责是验证用户帐号，是Shiro API中身份验证核心的入口点： 

   ```java
   public AuthenticationInfo authenticate(AuthenticationToken authenticationToken)  
               throws AuthenticationException;   
   ```

   如果验证成功，将返回AuthenticationInfo验证信息；此信息中包含了身份及凭证；如果验证失败将抛出相应的AuthenticationException实现。


   SecurityManager接口继承了Authenticator，另外还有一个ModularRealmAuthenticator实现，其委托给多个Realm进行验证，验证规则通过AuthenticationStrategy接口指定，默认提供的实现：

   **FirstSuccessfulStrategy**：只要有一个Realm验证成功即可，只返回第一个Realm身份验证成功的认证信息，其他的忽略；

   **AtLeastOneSuccessfulStrategy**：只要有一个Realm验证成功即可，和FirstSuccessfulStrategy不同，返回所有Realm身份验证成功的认证信息；

   **AllSuccessfulStrategy**：所有Realm验证成功才算成功，且返回所有Realm身份验证成功的认证信息，如果有一个失败就失败了。



   ModularRealmAuthenticator默认使用AtLeastOneSuccessfulStrategy策略。



   假设我们有三个realm：

   myRealm1： 用户名/密码为zhang/123时成功，且返回身份/凭据为zhang/123；

   myRealm2： 用户名/密码为wang/123时成功，且返回身份/凭据为wang/123；

   myRealm3： 用户名/密码为zhang/123时成功，且返回身份/凭据为zhang@163.com/123，和myRealm1不同的是返回时的身份变了；

1. ini配置文件（shiro-authenticator-all-success.ini）

   ```ini
   #指定securityManager的authenticator实现
   authenticator=org.apache.shiro.authc.pam.ModularRealmAuthenticator
   securityManager.authenticator=$authenticator

   #指定securityManager.authenticator的authenticationStrategy
   allSuccessfulStrategy=org.apache.shiro.authc.pam.AllSuccessfulStrategy
   securityManager.authenticator.authenticationStrategy=$allSuccessfulStrategy

   #申明多个realm
   myRealm1= com.zhaojun.shiro.chapter2.realm.MyRealm1
   myRealm2= com.zhaojun.shiro.chapter2.realm.MyRealm2
   myRealm3= com.zhaojun.shiro.chapter2.realm.MyRealm3
   #指定securityManager的realms实现
   securityManager.realms=$myRealm1,$myRealm2,$myRealm3
   ```

2. 测试代码（com.zhaojun.shiro.chapter2.AuthenticatorTest）

   ```java
   private void login(String configFile){
           //获取SecurityManager工厂，使用配置文件初始化工厂
           Factory<SecurityManager> factory = new IniSecurityManagerFactory(configFile);
           //得到SecurityManager实例，并绑定到SecurityUtils
           SecurityManager securityManager = factory.getInstance();
           SecurityUtils.setSecurityManager(securityManager);

           //获取Subject
           Subject subject = SecurityUtils.getSubject();

           //创建Token
           UsernamePasswordToken token = new UsernamePasswordToken("zhang","123");

           //登录
           subject.login(token);
       }

       @Test
       public void testAllSuccessfulStrategy(){
           login("classpath:shiro-authenticator-all-success.ini");
           Subject subject = SecurityUtils.getSubject();

           //得到一个身份集合
           PrincipalCollection principals = subject.getPrincipals();
           List list = principals.asList();
           for(Object i : list){
               log.info(i.toString());
           }
       }
   ```

   这时程序会抛出UnknownAccountException异常，因为我们用的是AllSuccessfulStrategy，需要所有Realm验证成功才算成功。

   3. 新建配置文件（shiro-authenticator-least-one-success.ini）

      ```ini
      #指定securityManager的authenticator实现
      authenticator=org.apache.shiro.authc.pam.ModularRealmAuthenticator
      securityManager.authenticator=$authenticator

      #指定securityManager.authenticator的authenticationStrategy
      atLeastOneSuccessfulStrategy=org.apache.shiro.authc.pam.AtLeastOneSuccessfulStrategy
      securityManager.authenticator.authenticationStrategy=$atLeastOneSuccessfulStrategy

      #申明多个realm
      myRealm1= com.zhaojun.shiro.chapter2.realm.MyRealm1
      myRealm2= com.zhaojun.shiro.chapter2.realm.MyRealm2
      myRealm3= com.zhaojun.shiro.chapter2.realm.MyRealm3
      #指定securityManager的realms实现
      securityManager.realms=$myRealm1,$myRealm2,$myRealm3
      ```

   4. 测试代码（com.zhaojun.shiro.chapter2.AuthenticatorTest）

      ```java
      	@Test
          public void testAtLeastOneSuccessfulStrategy(){
              login("classpath:shiro-authenticator-least-one-success.ini");
              Subject subject = SecurityUtils.getSubject();

              //得到一个身份集合
              PrincipalCollection principals = subject.getPrincipals();
              List list = principals.asList();
              for(Object i : list){
                  log.info(i.toString());
              }
          }
      ```

      可以看到输出日志,返回了两个身份信息

      ```shell
      00:40:32.294 [main] INFO  c.z.shiro.chapter2.AuthenticatorTest - zhang
      00:40:32.294 [main] INFO  c.z.shiro.chapter2.AuthenticatorTest - zhang@163.com
      ```

   到此，身份验证就基本搞定了，重点是身份验证的基本流程，以及常用的Realm的用法，如何自定义Realm，和常用的认证策略。


----

张开涛的博客：[http://jinnianshilongnian.iteye.com/category/305053](http://jinnianshilongnian.iteye.com/category/305053)

我的博客：[https://zhaojun0193.github.io](https://zhaojun0193.github.io)

本文代码地址：[https://github.com/zhaojun0193/shiro-example](https://github.com/zhaojun0193/shiro-example)

