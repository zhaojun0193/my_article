---
title: 《跟我学Shiro》学习笔记 第四章:编码/加密
date: 2018-4-25 14:15:00
tags: Shiro
---

#### 前言

密码存储应该采用加密/生产密码摘要存储。而不是采用明文存储，这期我们就来学习一下Shiro在编码与加密方面的功能。

#### 4.1编码/解码

Shiro提供了base64和16进制字符串编码/解码的API支持，方便一些编码解码操作。Shiro内部的一些数据的存储/表示都使用了base64和16进制字符串。

```java
@Test
public void base64encode(){
    String str = "hello";
    String base64Encoded = Base64.encodeToString(str.getBytes());

    log.info("编码后："+base64Encoded);

    String str2 = Base64.decodeToString(base64Encoded);

    log.info("解码后："+str2);
}
```

通过如上方式可以进行base64编码/解码操作。

<!-- more -->

```java
@Test
public void hexEncode(){
    String str = "hello";
    String base64Encoded = Hex.encodeToString(str.getBytes());
    log.info("编码后："+base64Encoded);
    String str2 = new String(Hex.decode(base64Encoded.getBytes()));
    log.info("解码后："+str2);
}
```

通过如上方式可以进行16进制字符串编码/解码操作。

#### 4.2散列算法

散列算法一般用于生成数据的摘要信息，是一种不可逆的算法，一般适合存储密码之类的数据，常见的散列算法如MD5、SHA等。一般进行散列时最好提供一个salt（盐），比如加密密码“admin”，产生的散列值是“21232f297a57a5a743894a0e4a801fc3”，可以到一些md5解密网站很容易的通过散列值得到密码“admin”，即如果直接对密码进行散列相对来说破解更容易，此时我们可以加一些只有系统知道的干扰数据，如用户名和ID（即盐）；这样散列的对象是“密码+用户名+ID”，这样生成的散列值相对来说更难破解。

```java
@Test
public void md5(){
    String str = "hello";
    String salt = "123";
    String md5 = new Md5Hash(str,salt).toString();
    log.info(md5);
}
```

md5算法散列。

```java
@Test
public void sha256(){
    String str = "hello";
    String salt = "123";
    String sha256 = new Sha256Hash(str,salt).toString();
    log.info(sha256);
}
```

sha256散列算法。

Shiro还提供了通用的散列支持：

```java
@Test
public void messageDigest(){
    String str = "hello";
    String salt = "123";
    //内部使用MessageDigest
    String simpleHash = new SimpleHash("SHA-1", str, salt).toString();
    log.info(simpleHash);
}
```

通过调用SimpleHash时指定散列算法，其内部使用了Java的MessageDigest实现。

为了方便使用，Shiro提供了HashService，默认提供了DefaultHashService实现。

```java
@Test
public void testHashService() {
    //默认算法SHA-512
    DefaultHashService hashService = new DefaultHashService();
    hashService.setHashAlgorithmName("SHA-512");
    //私盐，默认无
    hashService.setPrivateSalt(new SimpleByteSource("123"));
    //是否生成公盐，默认false
    hashService.setGeneratePublicSalt(true);
    //用于生成公盐。默认就这个
    hashService.setRandomNumberGenerator(new SecureRandomNumberGenerator());
    //生成Hash值的迭代次数
    hashService.setHashIterations(1);

    HashRequest request = new HashRequest.Builder()
    .setAlgorithmName("MD5").setSource(ByteSource.Util.bytes("hello"))
    .setSalt(ByteSource.Util.bytes("123")).setIterations(2).build();
    String hex = hashService.computeHash(request).toHex();
    log.info(hex);
}
```

1. 首先创建一个DefaultHashService，默认使用SHA-512算法；
2. 可以通过hashAlgorithmName属性修改算法；
3. 可以通过privateSalt设置一个私盐，其在散列时自动与用户传入的公盐混合产生一个新盐；
4. 可以通过generatePublicSalt属性在用户没有传入公盐的情况下是否生成公盐；
5. 可以设置randomNumberGenerator用于生成公盐；
6. 可以设置hashIterations属性来修改默认加密迭代次数；
7. 需要构建一个HashRequest，传入算法、数据、公盐、迭代次数。

#### 4.3加密/解密

Shiro还提供对称式加密/解密算法的支持，如AES、Blowfish等；当前还没有提供对非对称加密/解密算法支持，未来版本可能提供。

AES算法实现：

```java
@Test
public void aes() {
    AesCipherService aesCipherService = new AesCipherService();
    //设置key长度
    aesCipherService.setKeySize(128);
    //生成key
    Key key = aesCipherService.generateNewKey();
    String text = "hello";
    //加密
    String encrptText = aesCipherService.encrypt(text.getBytes(), key.getEncoded()).toHex();
    log.info(encrptText);
    //解密  
    String text2 = new String(aesCipherService.decrypt(Hex.decode(encrptText), key.getEncoded()).getBytes());

    log.info(text2);
}
```

#### 4.4PasswordService/CredentialsMatcher

Shiro提供了PasswordService及CredentialsMatcher用于提供加密密码及验证密码服务。

```java
public interface PasswordService {  
    //输入明文密码得到密文密码  
    String encryptPassword(Object plaintextPassword) throws IllegalArgumentException;  
}  
```

```java
public interface CredentialsMatcher {  
    //匹配用户输入的token的凭证（未加密）与系统提供的凭证（已加密）  
    boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info);  
}   
```

Shiro默认提供了PasswordService实现DefaultPasswordService；CredentialsMatcher实现PasswordMatcher及HashedCredentialsMatcher（更强大）。



**DefaultPasswordService配合PasswordMatcher实现简单的密码加密与验证服务**

1. 定义Realm

   ```java
   public class MyRealm extends AuthorizingRealm {
   
       private PasswordService passwordService;
   
       public void setPasswordService(PasswordService passwordService){
           this.passwordService = passwordService;
       }
   
       @Override
       protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
           return null;
       }
   
       @Override
       protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
           return new SimpleAuthenticationInfo("zhang",passwordService.encryptPassword("123"),getName());
       }
   }
   ```

   为了方便，直接注入一个passwordService来加密密码，实际使用时需要在Service层使用passwordService加密密码并存到数据库。

2. ini配置（shiro-passwordservice.ini）

   ```ini
   [main]
   passwordService=org.apache.shiro.authc.credential.DefaultPasswordService
   hashService=org.apache.shiro.crypto.hash.DefaultHashService
   passwordService.hashService=$hashService
   hashFormat=org.apache.shiro.crypto.hash.format.Shiro1CryptFormat
   passwordService.hashFormat=$hashFormat
   hashFormatFactory=org.apache.shiro.crypto.hash.format.DefaultHashFormatFactory
   passwordService.hashFormatFactory=$hashFormatFactory
   
   passwordMatcher=org.apache.shiro.authc.credential.PasswordMatcher
   passwordMatcher.passwordService=$passwordService
   
   myRealm=com.zhaojun.shiro.chapter4.realm.MyRealm
   myRealm.passwordService=$passwordService
   myRealm.credentialsMatcher=$passwordMatcher
   securityManager.realms=$myRealm
   ```

   * passwordService使用DefaultPasswordService，如果有必要也可以自定义；
   * hashService定义散列密码使用的HashService，默认使用DefaultHashService（默认SHA-256算法）；
   * hashFormat用于对散列出的值进行格式化，默认使用Shiro1CryptFormat，另外提供了Base64Format和HexFormat，对于有salt的密码请自定义实现ParsableHashFormat然后把salt格式化到散列值中；
   * hashFormatFactory用于根据散列值得到散列的密码和salt；因为如果使用如SHA算法，那么会生成一个salt，此salt需要保存到散列后的值中以便之后与传入的密码比较时使用；默认使用DefaultHashFormatFactory；
   * passwordMatcher使用PasswordMatcher，其是一个CredentialsMatcher实现；
   * 将credentialsMatcher赋值给myRealm，myRealm间接继承了AuthenticatingRealm，其在调用getAuthenticationInfo方法获取到AuthenticationInfo信息后，会使用credentialsMatcher来验证凭据是否匹配，如果不匹配将抛出IncorrectCredentialsException异常。

   **HashedCredentialsMatcher实现密码验证服务**

   Shiro提供了CredentialsMatcher的散列实现HashedCredentialsMatcher，和之前的PasswordMatcher不同的是，它只用于密码验证，且可以提供自己的盐，而不是随机生成盐，且生成密码散列值的算法需要自己写，因为能提供自己的盐。

   1. 生成密码散列值

      此处我们使用MD5算法，“密码+盐（用户名+随机数）”的方式生成散列值：

      ```java
      String algorithmName = "md5";  
      String username = "liu";  
      String password = "123";  
      String salt1 = username;  
      String salt2 = new SecureRandomNumberGenerator().nextBytes().toHex();  
      int hashIterations = 2;  
        
      SimpleHash hash = new SimpleHash(algorithmName, password, salt1 + salt2, hashIterations);  
      String encodedPassword = hash.toHex();   
      ```

   如果要写用户模块，需要在新增用户/重置密码时使用如上算法保存密码，将生成的密码及salt2存入数据库（因为我们的散列算法是：md5(md5(密码+username+salt2))）。

   2. 生成Realm

      ```java
      @Override
          protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
              //用户名及salt1
              String username = "liu";
              //加密后的密码
              String password = "202cb962ac59075b964b07152d234b70";
              String salt2 = "202cb962ac59075b964b07152d234b70";
              //盐是用户名+随机数
              SimpleAuthenticationInfo ai =
                      new SimpleAuthenticationInfo(username, password, getName());
              ai.setCredentialsSalt(ByteSource.Util.bytes(username+salt2));
              return ai;
          }
      ```

      如果使用JdbcRealm，需要修改获取用户信息（包括盐）的sql：“select password, password_salt from users where username = ?”，而我们的盐是由username+password_salt组成，所以需要通过如下ini配置修改：

      ```ini
      jdbcRealm.saltStyle=COLUMN  
      jdbcRealm.authenticationQuery=select password, concat(username,password_salt) from users where username = ?  
      jdbcRealm.credentialsMatcher=$credentialsMatcher 
      ```

      1、saltStyle表示使用密码+盐的机制，authenticationQuery第一列是密码，第二列是盐；

      2、通过authenticationQuery指定密码及盐查询SQL；

----

   张开涛的博客：[http://jinnianshilongnian.iteye.com/category/305053](http://jinnianshilongnian.iteye.com/category/305053)

   我的博客：[https://zhaojun0193.github.io](https://zhaojun0193.github.io)

   本文代码地址：[https://github.com/zhaojun0193/shiro-example](https://github.com/zhaojun0193/shiro-example)
