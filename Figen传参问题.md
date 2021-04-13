---
title: Figen GET传参遇到的问题以及解决方法
date: 2019-8-22 22:10:26
tags: Figen
---

今天学习feigen遇到了一些问题，在此记录一下。

1. feigen用GET方式传递对象的时候遇到405错误。
2. 配置使用HttpClient的时候遇到`java.lang.NoSuchMethodError: feign.Response.create(ILjava/lang/String;Ljava/util/Map;Lfeign/Response$Body;)Lfeign/Response;`错误。

首先模拟一下feigen利用GET方式传递对象。

**1. 创建服务提供者**

pom.xml 截取

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.7.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <java.version>1.8</java.version>
    <spring-cloud.version>Greenwich.SR2</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

application.yml

```yaml
spring:
  application:
    name: provider
server:
  port: 8080
eureka:
  client:
    service-url:
      defaultZone: http://eureka.springcloud.cn/eureka/
```

这里的Eureka我用的是公益-Eureka Server注册中心，哈哈，不想在本地创建Eureka项目了。

启动类：ProviderApplication.java

```java
@SpringBootApplication
@EnableEurekaClient
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }

}
```

创建一个简单的domain，作为传输参数。

User.java

```java
public class User {
    private Integer id;
    private String name;
    // getter ...
    // setter ...
}
```

创建测试用的服务接口

UserController.java

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @RequestMapping(value = "",method = RequestMethod.GET)
    public String getUser(@RequestBody User user){
        return "user: " + user.getName();
    }
}
```

这里我们用GET方式接收参数。

**2.创建服务消费者**

这里只是比服务提供者多了一个openfeign的依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

application.yml

```yaml
spring:
  application:
    name: consumer
server:
  port: 8081
eureka:
  client:
    service-url:
      defaultZone: http://eureka.springcloud.cn/eureka/
```

启动类：ConsumerApplication.java

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
@EnableEurekaClient
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }

}
```

UserService.java

```java
@FeignClient(value = "provider")
public interface UserService {

    @RequestMapping(value = "/user",method = RequestMethod.GET)
    public String getUser(@RequestBody User user);
}
```

ConsumerController.java

```java
@RestController
public class ConsumerController {

    @Autowired
    private UserService userService;

    @RequestMapping(value = "/getUser",method = RequestMethod.GET)
    @ResponseBody
    public String getUser(User user){
        return userService.getUser(user);
    }
}
```

服务提供者和消费者都创建完毕了，我们来测试一下。

启动项目，用Postman调用一下服务消费者的接口。

![Postman](D:\images\博客\feign\Snipaste_2019-08-23_01-25-51.png)

消费者端控制台日志

```verilog
feign.FeignException$MethodNotAllowed: status 405 reading UserService#getUser(User)
	at feign.FeignException.errorStatus(FeignException.java:100) ~[feign-core-10.2.3.jar:na]
	at feign.FeignException.errorStatus(FeignException.java:86) ~[feign-core-10.2.3.jar:na]
	at feign.codec.ErrorDecoder$Default.decode(ErrorDecoder.java:93) ~[feign-core-10.2.3.jar:na]
	at feign.SynchronousMethodHandler.executeAndDecode(SynchronousMethodHandler.java:149) ~[feign-core-10.2.3.jar:na]
	at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:78) ~[feign-core-10.2.3.jar:na]
	at feign.ReflectiveFeign$FeignInvocationHandler.invoke(ReflectiveFeign.java:103) ~[feign-core-10.2.3.jar:na]
	at com.sun.proxy.$Proxy88.getUser(Unknown Source) ~[na:na]
	at com.zhaojun.consumer.controller.ConsumerController.getUser(ConsumerController.java:20) ~[classes/:na]
```

服务提供者控制台日志

```verilog
2019-08-23 01:25:22.215  WARN 15944 --- [nio-8080-exec-7] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.HttpRequestMethodNotSupportedException: Request method 'POST' not supported]
```

可以看到我们明明定义的是以GET请求的方式调用服务提供者，但是却变成了用POST方式调用。这是为什么呢。

然后通过断点调试发现，feign默认使用的是HttpURLConnection作为请求的客户端，feign-core包下的Client.java的convertAndSend方法下有如下代码

![](D:\images\博客\feign\Snipaste_2019-08-23_02-24-16.png)

这里request.requestBody().asBytes() 是不为空的，requestBody就是为我们传的参数，接下来会执行这段代码

```java
OutputStream out = connection.getOutputStream();
```

我们看看这个方法

![](D:\images\博客\feign\Snipaste_2019-08-23_02-47-16.png)

然后再进入getOutputStream0()方法

![](D:\images\博客\feign\Snipaste_2019-08-23_02-48-21.png)

可以看到，他会判断请求是否是GET如果是的话，就把请求方式变为POST。

所以，我们请求服务提供者的时候会出现405的错误。

那么，为什么我们get请求会有requestBody呢？

因为在opfeign-core包下的SpringEncoder类下的encode中

![](D:\images\博客\feign\Snipaste_2019-08-23_03-15-01.png)

将我们的请求参数转成了json，然后设置header=application/json;charset=UTF-8,然后再将json转成byte数组放到了requesBody中。

所以，HttpURLConnection判断我们的requestBody不为空，然后认为是POST请求。那么我们如何解决这个问题呢？

**方法一、使用httpclient代替HttpURLConnection**

既然HttpURLConnection会把带有body的GET请求转成POST那么我们不用他好了。

首先，引入相关依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>

<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>10.2.3</version>
</dependency>
```

这里注意feign-httpclient版本要与feign-core版本一致。

我这里就犯了一个错误，我根据网上以及书上的的教材进行配置，他们说需要引入这个依赖

```xml
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>8.18.0</version>
</dependency>
```

但是他们的spring-boot是2.0.3.RELEASE，spring-cloud版本是Finchley.RELEASE版的，而我的版本和他们不同，导致在进行调用的时候产生了`java.lang.NoSuchMethodError: feign.Response.create(ILjava/lang/String;Ljava/util/Map;Lfeign/Response$Body;)Lfeign/Response`异常。

这是因为create方法在Feign 10就已经移除掉了

![](D:\images\博客\feign\Snipaste_2019-08-23_09-52-42.png)

好了，引入依赖后，在配置文件中开启使用httpclient

```yaml
feign:
  httpclient:
    enabled: true
```

在我使用这个版本这个值是默认为true的，但是还是配上吧。

![](D:\images\博客\feign\Snipaste_2019-08-23_10-07-19.png)

然后重新启动项目后用Postman调用,发现已经成功了。

![](D:\images\博客\feign\Snipaste_2019-08-23_10-15-37.png)

**方法二、增加拦截器处理requestBody中的参数**

如果我们不想用httpclient，也不想让GET请求中包含requestBody，那我们就需要把requestTemplate中的requestBody置为空，并且将条件拼接到url后面。这里参考《重新定义Spring Cloud实战》一书给出代码（根据SpringBoot版本不同代码有所改动）。

```java
@Component
public class FeignRequestInterceptor implements RequestInterceptor {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void apply(RequestTemplate template) {
        // feign 不支持 GET 方法传 POJO, json body转query
        if (template.method().equals("GET") && template.requestBody() != null) {
            try {
                JsonNode jsonNode = objectMapper.readTree(template.requestBody().asBytes());
                template.body(Request.Body.empty());

                Map<String, Collection<String>> queries = new HashMap<>();
                buildQuery(jsonNode, "", queries);
                template.queries(queries);
            } catch (IOException e) {
                //提示:根据实践项目情况处理此处异常，这里不做扩展。
                e.printStackTrace();
            }
        }
    }

    private void buildQuery(JsonNode jsonNode, String path, Map<String, Collection<String>> queries) {
        if (!jsonNode.isContainerNode()) {   // 叶子节点
            if (jsonNode.isNull()) {
                return;
            }
            Collection<String> values = queries.get(path);
            if (null == values) {
                values = new ArrayList<>();
                queries.put(path, values);
            }
            values.add(jsonNode.asText());
            return;
        }
        if (jsonNode.isArray()) {   // 数组节点
            Iterator<JsonNode> it = jsonNode.elements();
            while (it.hasNext()) {
                buildQuery(it.next(), path, queries);
            }
        } else {
            Iterator<Map.Entry<String, JsonNode>> it = jsonNode.fields();
            while (it.hasNext()) {
                Map.Entry<String, JsonNode> entry = it.next();
                if (StringUtils.hasText(path)) {
                    buildQuery(entry.getValue(), path + "." + entry.getKey(), queries);
                } else {  // 根节点
                    buildQuery(entry.getValue(), entry.getKey(), queries);
                }
            }
        }
    }
}
```

可以看到请求改造前后的变化。

改造前：

![](D:\images\博客\feign\Snipaste_2019-08-23_11-16-30.png)

改造后：

![](D:\images\博客\feign\Snipaste_2019-08-23_11-18-12.png)

这样就变成了传统的get方法获取数据，那么服务提供者这边需要把@RequestBody注解去掉，因为现在RequestBody已经是空了。

```java
@RestController
@RequestMapping("/user")
public class UserController {

    /**
     * 去掉 @RequestBody
     */
    @RequestMapping(value = "",method = RequestMethod.GET)
    public String getUser(User user){
        return "user: " + user.getName();
    }
}
```

接下来测试一下，发现传参成功了。

![](D:\images\博客\feign\Snipaste_2019-08-23_11-22-55.png)

大功告成！