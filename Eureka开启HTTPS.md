---
title: Eureka启用HTTPS
date: 2019-8-21 10:30:26
tags: Eureka
---

1. 为Eureka Client生成证书

   ```shell
   client:keytool -genkeypair -alias client -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore client.p12 -validity 3650
   ```

   ```shell
   输入密钥库口令:
   再次输入新口令:
   您的名字与姓氏是什么?
     [Unknown]:
   您的组织单位名称是什么?
     [Unknown]:
   您的组织名称是什么?
     [Unknown]:
   您所在的城市或区域名称是什么?
     [Unknown]:
   您所在的省/市/自治区名称是什么?
     [Unknown]:
   该单位的双字母国家/地区代码是什么?
     [Unknown]:
   CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown是否正确?
     [否]:  y
   ```
   Eureka Client证书的密码设置的是：client

2. 为Eureka Serveer生成证书

   ```shell
   keytool -genkeypair -alias server -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore server.p12 -validity 3650
   ```

   ```shell
   输入密钥库口令:
   再次输入新口令:
   您的名字与姓氏是什么?
     [Unknown]:
   您的组织单位名称是什么?
     [Unknown]:
   您的组织名称是什么?
     [Unknown]:
   您所在的城市或区域名称是什么?
     [Unknown]:
   您所在的省/市/自治区名称是什么?
     [Unknown]:
   该单位的双字母国家/地区代码是什么?
     [Unknown]:
   CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown是否正确?
     [否]:  y
   ```
   
   Eureka Client证书的密码设置的是：server
   
   经过上面两个操作之后，当前目录下会生成两个.p12文件，分别是client.p12和server.p12。
   
3. 下面分别导出两个p12证书，如下：

      ```shell
      keytool -export -alias client -file client.crt --keystore client.p12
      ```

      ```shell
      输入密钥库口令:
      存储在文件 <client.crt> 中的证书
      ```

      ```shell
      keytool -export -alias server -file server.crt --keystore server.p12
      ```

      ```shell
      输入密钥库口令:
      存储在文件 <server.crt> 中的证书
      ```

4. 接下来，将server.crt文件导入client.p12中，使client端信任server的证书

   ```shell
   keytool -import -alias server -file server.crt -keystore client.p12
   ```

   ```shell
   输入密钥库口令:
   所有者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
   发布者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
   序列号: 1835fea4
   有效期开始日期: Wed Aug 21 11:37:51 CST 2019, 截止日期: Sat Aug 18 11:37:51 CST 2029
   
   证书指纹:
            MD5: 6E:4B:26:19:44:DD:1A:F2:DE:F8:B8:25:A0:17:28:DA
            SHA1: 87:5C:4F:84:6B:D6:E5:6D:4E:1C:61:B6:87:99:1E:AD:85:6F:31:75
            SHA256: E4:47:2B:F5:D2:73:0E:81:64:B7:0F:A1:0A:99:B4:41:8F:D9:A3:5A:E4:15:7C:58:36:00:B5:E9:AF:8F:81:23
            签名算法名称: SHA256withRSA
            版本: 3
   
   扩展:
   
   #1: ObjectId: 2.5.29.14 Criticality=false
   SubjectKeyIdentifier [
   KeyIdentifier [
   0000: 19 48 C8 13 BB 25 DE FF   9B 72 9F EC B0 D4 6C 91  .H...%...r....l.
   0010: 2F F5 B2 14                                        /...
   ]
   ]
   
   是否信任此证书? [否]:  y
   证书已添加到密钥库中
   ```

   这里输入client.p12的密码：client。

   将client.crt文件导入server.p12中，使server端信任client的证书

   ```shell
   keytool -import -alias client -file client.crt -keystore server.p12
   ```

   ```shell
   输入密钥库口令:
   所有者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
   发布者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
   序列号: 6e0b1e14
   有效期开始日期: Wed Aug 21 11:39:23 CST 2019, 截止日期: Sat Aug 18 11:39:23 CST 2029
   
   证书指纹:
            MD5: C3:75:BC:D1:01:21:E0:E1:EA:C7:88:D0:BD:2C:3B:D3
            SHA1: 7E:58:1C:86:5F:28:B0:6F:69:A2:47:E6:32:3D:B6:8A:32:20:34:4E
            SHA256: C2:97:D0:68:DF:32:E2:CC:C0:7D:23:74:89:E3:37:EC:92:DB:41:6B:EE:13:C0:D7:5A:D0:C9:45:F1:86:CB:C1
            签名算法名称: SHA256withRSA
            版本: 3
   
   扩展:
   
   #1: ObjectId: 2.5.29.14 Criticality=false
   SubjectKeyIdentifier [
   KeyIdentifier [
   0000: E8 48 CB CB 3A E9 96 B4   03 50 B7 FA 53 8A E3 71  .H..:....P..S..q
   0010: FD C3 EB 74                                        ...t
   ]
   ]
   
   是否信任此证书? [否]:  y
   证书已添加到密钥库中
   ```

   这里输入server.p12的密码：server。

5. 创建eureka-server工程，将server.p12文件放到resources目录下，并在application.yml下配置如下信息。

   ```yaml
   server:
     port: 8761
     ssl:
       enabled: true
       key-store: classpath:server.p12
       key-alias: server
       key-store-type: PKCS12
       key-store-password: server
   eureka:
     instance:
       hostname: localhost
       secure-port: 443
       secure-port-enabled: true
       non-secure-port-enabled: false  #该实例应该接收通信的非安全端口是否启用，默认为true
       home-page-url: https://${eureka.instance.hostname}:${server.port}/
       status-page-url: https://${eureka.instance.hostname}:${server.port}/
     client:
       fetch-registry: false
       register-with-eureka: false
       service-url:
         defaultZone: https://${eureka.instance.hostname}:${server.port}/eureka
     server:
       wait-time-in-ms-when-sync-empty: 0
       enable-self-preservation: false
   
   ```
   启动eureka-server，访问https://localhost:8761/发现可以使用https进行访问。

   创建eureka-client工程，将client.p12文件放到resources目录下，并在application.yml下配置如下信息。

   ```yaml
   server:
     port: 8080
   eureka:
     client:
       securePortEnabled: true
       ssl:
         key-store: client.p12
         key-store-password: client
       service-url:
         defaultZone: https://localhost:8761/eureka
   spring:
     application:
       name: eureka-client
   ```

   我们这里没有指定整个应用实例启用HTTPS，仅仅是开启访问eureka-server的HTTPS配置。通过自定义配置eureka.client.ssl.key-store和eureka.client.ssl.key-store-password两个属性，指定eureka-client访问eureka-server的sslContext配置。这里需要在代码里指定DiscoveryClient.DiscoveryClientOptionalArgs。

   在eureka-client项目下新增一个配置类EurekaHttpsClientConfig，代码如下。

   ```java
   @Configuration
   public class EurekaHttpsClientConfig {
   
       @Value("${eureka.client.ssl.key-store}")
       String keyStoreFileName;
   
       @Value("${eureka.client.ssl.key-store-password}")
       String keyStorePassword;
   
       @Bean
       public DiscoveryClient.DiscoveryClientOptionalArgs discoveryClientOptionalArgs() throws CertificateException, NoSuchAlgorithmException, KeyStoreException, IOException, KeyManagementException {
           EurekaJerseyClientImpl.EurekaJerseyClientBuilder builder = new EurekaJerseyClientImpl.EurekaJerseyClientBuilder();
           builder.withClientName("eureka-https-client");
           SSLContext sslContext = new SSLContextBuilder()
                   .loadTrustMaterial(
                           this.getClass().getClassLoader().getResource(keyStoreFileName),keyStorePassword.toCharArray()
                   )
                   .build();
           builder.withCustomSSL(sslContext);
   
           builder.withMaxTotalConnections(10);
           builder.withMaxConnectionsPerHost(10);
   
           DiscoveryClient.DiscoveryClientOptionalArgs args = new DiscoveryClient.DiscoveryClientOptionalArgs();
           args.setEurekaJerseyClient(builder.build());
           return args;
       }
   }
   ```

   上面的代码还需要httpclient的依赖

   ```xml
   <dependency>
       <groupId>org.apache.httpcomponents</groupId>
       <artifactId>httpclient</artifactId>
       <version>4.5.5</version>
   </dependency>
   ```

   启动eureka-client，发现eureka-client已经成功注册到eureka-server上了。

   



