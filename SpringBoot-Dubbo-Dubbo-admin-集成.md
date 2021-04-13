---
title: SpringBoot Dubbo Dubbo admin 集成
date: 2019-4-26 10:30:26
tags: Dubbo
---

####  创建api项目

这个项目只提供api接口，没有任何接口实现。该接口需单独打包，在服务提供方和消费方共享。
pom文件的打包类型为jar

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>hello-dubbo</artifactId>
        <groupId>com.zhaojun</groupId>
        <version>1.0.0</version>
    </parent>
    
    <modelVersion>4.0.0</modelVersion>
    <artifactId>hello-dubbo-api</artifactId>
    <packaging>jar</packaging>

</project>
```

然后将这个项目安装到maven本地仓库

```shell
mvn clean install
```

#### 服务提供者

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.zhaojun</groupId>
        <artifactId>hello-dubbo</artifactId>
        <version>1.0.0</version>
        <relativePath>../</relativePath>
    </parent>

    <groupId>com.zhaojun</groupId>
    <artifactId>hello-dubbo-provider</artifactId>
    <version>1.0.0</version>
    <name>hello-dubbo-provider</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>log4j</artifactId>
                    <groupId>log4j</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>slf4j-log4j12</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- Dubbo Spring Boot Starter -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.7.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
        </dependency>
		<!-- api -->
        <dependency>
            <groupId>com.zhaojun</groupId>
            <artifactId>hello-dubbo-api</artifactId>
            <version>1.0.0</version>
        </dependency>

    </dependencies>


    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

在启动类加上@EnableDubbo注解,这是[官方文档](http://dubbo.apache.org/zh-cn/blog/dubbo-annotation.html)的解释。


```reStructuredText
通过 @EnableDubbo 可以在指定的包名下（通过 scanBasePackages），或者指定的类中（通过 scanBasePackageClasses）扫描 Dubbo 的服务提供者（以 @Service 标注）以及 Dubbo 的服务消费者（以 Reference 标注）
```

```java
import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author zhaojun0193
 * @date 2019/4/27
 */

@EnableDubbo
@SpringBootApplication
public class HelloDubboProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloDubboProviderApplication.class, args);
    }

}
```

创建服务实现

```java
import com.zhaojun.hello.dubbo.api.DemoService;
import org.apache.dubbo.config.annotation.Service;

/**
 * @author zhaojun0193
 * @date 2019/4/27
 */
@Service(version = "${demo.service.version}")
public class DemoServiceImpl implements DemoService {

    @Override
    public String sayHi() {
        return "Hello Dubbo!";
    }
}
```

服务提供者配置文件

```properties
spring:
  application:
    name: hello-dubbo-provider
  main:
    allow-bean-definition-overriding: true # 允许重名bean覆盖，设置后不会与Spring Boot冲突
dubbo:
  protocol:
    status: server
    name: dubbo
    port: 12345
  registry:
    address: zookeeper://192.168.179.143:2181,192.168.179.143:2182,192.168.179.143:2183 #注册中心地址
    file: dubbo-registry/dubbo-registry.properties  #注册中心的本地缓存配置
demo:
  service:
    version: 1.0.0
```

其中dubbo.registry.file 是个坑 这里有相关的解释 [https://my.oschina.net/greki/blog/550976 ](https://my.oschina.net/greki/blog/550976)

dubbo.registry.address 是配置zookepper的地址。关于zookeeper的安装可以看看这个文章。。。。

#### 服务消费者

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.zhaojun</groupId>
        <artifactId>hello-dubbo</artifactId>
        <version>1.0.0</version>
        <relativePath>../</relativePath>
    </parent>
    <groupId>com.zhaojun</groupId>
    <artifactId>hello-dubbo-consumer</artifactId>
    <version>1.0.0</version>
    <name>hello-dubbo-consumer</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>log4j</artifactId>
                    <groupId>log4j</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>slf4j-log4j12</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- Dubbo Spring Boot Starter -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.7.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
        </dependency>
		<!-- api -->
        <dependency>
            <groupId>com.zhaojun</groupId>
            <artifactId>hello-dubbo-api</artifactId>
            <version>1.0.0</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

创建一个Controller 将Service注入进来。这里使用的是Dubbo的@Reference注解。表面作用和Spring的@Autowired一样。只不过@Reference是分布式的远程调用。这里的version一定要和服务提供者配置的version版本一致。

```java
import com.zhaojun.hello.dubbo.api.DemoService;
import org.apache.dubbo.config.annotation.Reference;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author zhaojun0193
 * @date 2019/4/27
 */
@RestController
@RequestMapping("/demo")
public class DemoController {

    @Reference(version = "${demo.service.version}")
    private DemoService demoService;

    @RequestMapping(value = "/sayHi",method = RequestMethod.GET)
    public String sayHi(){
        return demoService.sayHi();
    }
}
```

服务消费者配置文件

```properties
spring:
  application:
    name: hello-dubbo-consumer
  main:
    allow-bean-definition-overriding: true # 不设置会与Spring Boot冲突
dubbo:
  protocol:
    name: dubbo
    port: 12345
  registry:
    address: zookeeper://192.168.179.143:2181,192.168.179.143:2182,192.168.179.143:2183 #注册中心地址
    file: dubbo-registry/dubbo-registry.properties
demo:
  service:
    version: 1.0.0
server:
  port: 8888
```

这里和服务提供者配置差不多。但是服务消费者是一个web项目，不要要配置dubbo.status = server

父项目pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zhaojun</groupId>
    <artifactId>hello-dubbo</artifactId>
    <packaging>pom</packaging>

    <version>1.0.0</version>
    <modules>
        <module>hello-dubbo-api</module>
        <module>hello-dubbo-provider</module>
        <module>hello-dubbo-consumer</module>
    </modules>

    <properties>
        <java.version>1.8</java.version>
        <dubbo.version>2.7.1</dubbo.version>
        <spring-boot.version>2.1.4.RELEASE</spring-boot.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Aapche Dubbo  -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo-dependencies-bom</artifactId>
                <version>${dubbo.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo</artifactId>
                <version>${dubbo.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>javax.servlet</groupId>
                        <artifactId>servlet-api</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>log4j</groupId>
                        <artifactId>log4j</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```

到这里我们这个demo就差不多搭建完了。现在启动服务提供者和服务消费者，看看效果。

![1556346381010](D:\images\博客\dubbo\1556346381010.png)

#### Dubbo admin  

1. 将项目克隆到本地

```shell
git clone https://github.com/apache/incubator-dubbo-admin.git
```

2. 修改incubator-dubbo-admin\dubbo-admin-server\src\main\resources\application.properties文件

```properties
admin.registry.address=zookeeper://192.168.179.143:2181
admin.config-center=zookeeper://192.168.179.143:2181
admin.metadata-report.address=zookeeper://192.168.179.143:2181
```

3. 构建项目

```shell
mvn clean package
```

这一步我遇到了一点问题。项目中使用的dubbo版本是2.7.2-SNAPSHOT。而我始终下载不下来这个依赖。

然后我就上github提问，有个朋友说需要配置apache的snapshot仓库，我发现我也是配置了的。

```xml
<repositories>
		<repository>
			<id>apache.snapshots.https</id>
			<name>Apache Development Snapshot Repository</name>
			<url>https://repository.apache.org/content/repositories/snapshots</url>
			<releases>
				<enabled>false</enabled>
			</releases>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
	</repositories>
```

后来网上查了一下，是我mavn的settings.xml 中配置了一个阿里的镜像仓库。但是我是这样配置的。

```xml
<mirror>
        <!--aliyun mirror reponstory -->
        <id>nexus-aliyun</id>
        <mirrorOf>*</mirrorOf> 
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
```

mirrorOf 不应该配置为* 这样的话所有的下载都会走阿里的这个仓库。所以我下载不到dubbo-2.7.2-SNAPSHOT

这里有具体的解释[<http://www.cnblogs.com/whm-blog/p/7102402.html> ](<http://www.cnblogs.com/whm-blog/p/7102402.html> )

4.启动项目

```shell
mvn --projects dubbo-admin-server spring-boot:run
```

或者

```shell
cd dubbo-admin-distribution/target; java -jar dubbo-admin-0.1.jar
```

5.然后访问 http://localhost:8080

![]()

点击服务查询，可以看到，我们刚才发布的服务现在已经能够查询到了。

本篇文章的源码：[https://github.com/zhaojun0193/hello-dubbo](https://github.com/zhaojun0193/hello-dubbo)

转载请注明出处。