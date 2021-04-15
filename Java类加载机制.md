#### 前言

我们写的Java代码到编译成Class文件，最后被加载到JVM中，经历了哪些过程呢？今天我们就来讲讲Java的类加载机制。

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

> 上图中，装载，链接，初始化属于类加载的三个大的阶段

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

> 注：如果age为final static 修饰的常量，在准备阶段，age就会被初始化为属性所指定的值，即：准备阶段完成后age=10

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

##### 分类

```text
1.Bootstrap ClassLoader 负责加载$JAVA_HOME中 jre/lib/rt.jar 里所有的class或 Xbootclassoath选项指定的jar包。由C++实现，不是ClassLoader子类。 
2.Extension ClassLoader 负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中 jre/lib/*.jar 或 -Djava.ext.dirs指定目录下的jar包。
3.App ClassLoader 负责加载classpath中指定的jar包及 Djava.class.path 所指定目录下的类和 jar包。 
4.Custom ClassLoader 通过java.lang.ClassLoader的子类自定义加载class，属于应用程序根据 自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader。
```

##### 双亲委派机制

1. 检查某个类是否已经加载

   自底向上，从Custom ClassLoader到BootStrap ClassLoader逐层检查，只要某个Classloader已加载，就视为已加载此类，保证此类只所有ClassLoader加载一次。

2. 加载的顺序

   自顶向下，也就是由上层来逐层尝试加载此类。

   ![image](http://www.zhaojun.ink/upload/2021/04/image-823326c820d24375aac31425bcf7abdb.png)

如果一个类加载器收到了加载类的请求，它会先把请求委托给上层加载器去完成，上层加载器又会委托上上层加载器，一直到最顶层的类加载器；如果上层加载器无法完成类的加载工作时，当前类加载器才会尝试自己去加载这个类。

ClassLoader 类中可以看到类加载的逻辑

```java
 protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先, 检查类是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //父加载器不为空，则交给父类加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                        //父加载器为空，由BootstrapClassLoader类加载器加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```






