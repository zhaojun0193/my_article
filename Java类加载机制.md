#### 字节码

Java有一个很出名的口号：“Write Once, Run Anywhere”，他是如何实现这句口号的呢，这其中就离不开字节码。

Sun公司开发了在不同平台上运行的Java虚拟机JVM，用来执行和载入编译后生成的字节码文件。

![](http://www.zhaojun.ink/upload/2021/04/image-5917914bf8d947caa8d9d8ad94f81445.png)

#### 类加载机制

虚拟机把Class文件加载到内存 

并对数据进行校验，转换解析和初始化 

形成可以虚拟机直接使用的Java类型，即java.lang.Class 

#### 类加载过程

![类加载过程](http://www.zhaojun.ink/upload/2021/03/image-6463eaaab58946a2a2f863ba0334eb5c.png)

#### 装载

1. JVM在该阶段会将字节码从不同的数据源（class文件，jar包，也可能是网络）转化为二进制流，加载到内存中
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口

#### 链接

##### 验证

* 文件格式验证

* 元数据验证

* 字节码验证

* 符号引用验证

##### 准备

在这个阶段，JVM会为类的静态变量分配内存，并将初始化为默认值

```java
public static int age = 10; // 准备阶段完成之后age的值为0
```

##### 解析

把类中的符号引用转换为直接引用

```text
符号引用就是一组符号来描述目标，可以是任何字面量。 
直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

在编译时，Java 类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。比如 com.Teacher 类引用了 com.Student 类，编译时 Teacher 类并不知道 Student 类的实际内存地址，因此只能使用符号 com.Student。
```

#### 初始化

对类的静态变量，静态代码块执行初始化操作

```java
public static int age = 10; // 初始化阶段完成之后age的值为10
```

#### 类加载器

类加载器分为四种

```text
1.Bootstrap ClassLoader 负责加载$JAVA_HOME中 jre/lib/rt.jar 里所有的class或 Xbootclassoath选项指定的jar包。由C++实现，不是ClassLoader子类。 
2.Extension ClassLoader 负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中 jre/lib/*.jar 或 -Djava.ext.dirs指定目录下的jar包。
3.App ClassLoader 负责加载classpath中指定的jar包及 Djava.class.path 所指定目录下的类和 jar包。 
4.Custom ClassLoader 通过java.lang.ClassLoader的子类自定义加载class，属于应用程序根据 自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader。
```








