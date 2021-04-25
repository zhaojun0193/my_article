#### 前言

前面我们讲到[Java类加载机制](http://www.zhaojun.ink/archives/javaclassload)和 [Java类的初始化](http://www.zhaojun.ink/archives/javaclassinitialize)，了解了Java代码从编译，到加载到虚拟机的基本步骤，接下来我们讲Java对象实例初始化过程，让我们更清楚的了解类的实例化的顺序。

#### 类的实例化

我们在讲类的初始化的时候，提到类初始化其实就是执行`< clinit >()`方法，其实，实例化也有对应的方法，叫做`<init>()`，实例初始化就是执行`<init>()`方法。

* `<init>()`方法可能重载多个，有几个构造器就有几个`<init>()`方法
* `<init>()`由非静态实例变量显示赋值代码和非静态代码块、对应构造器代码组成
* 非静态实例变量显示赋值代码和非静态代码块代码从上到下顺序执行，而对应构造器代码最后执行
* 每次创建实例对象，调用对应构造器，执行的就是对应的`<init>()`方法
* `<init>()`方法的首行是`super()`或`super(实参列表)`,即对应父类的`<init>()`方法

我们总结下：

`<init>()`方法的执行顺序为：父类的非静态实例变量显示赋值代码和非静态代码块 > 父类构造器>子类的非静态实例变量显示赋值代码和非静态代码块>子类构造器。

> 注：非静态实例变量和非静态代码块按照代码的声明顺序从上到下执行

了解了实例化的基本知识之后，结合之前类的初始化知识，我们来做两道面试题巩固一下:

**Father类**

```java
public class Father {
    {
        System.out.println("3");
    }

    private int i = test();
    private static int j = method();

    static {
        System.out.println("1");
    }

    Father() {
        System.out.println("2");
    }


    private int test() {
        System.out.println("4");
        return 1;
    }

    private static int method() {
        System.out.println("5");
        return 1;
    }
}
```

**Son类**

```java
public class Son extends Father{
    private int i = test();
    private static int j = method();

    static {
        System.out.println("6");
    }

    Son() {
        System.out.println("7");
    }

    {
        System.out.println("8");
    }

    private int test() {
        System.out.println("9");
        return 1;
    }

    private static int method() {
        System.out.println("10");
        return 1;
    }

    public static void main(String[] args) {
        Son son = new Son();
        System.out.println("----------");
        Son son1 = new Son();
    }
}
```

**打印结果**

```java
5
1
10
6
3
4
2
9
8
7
----------
3
4
2
9
8
7
```

结果和你想的一样吗？接下来我们来分析一下打印结果：

1. 首先，我们看`main()`方法中，第一行是`new Son()`，我们知道`new`触发发类的初始化，而子类在初始化之前应该先初始化父类，所以我们找到`Father`类中的静态变量和静态代码块，可得出打印顺序为：5，1
2. 接下来初始化子类，我们找到`Son`类中的静态代码块，可得出打印顺序为：10，6
3. 类初始化结束后，进行实例的初始化。首先进行父类的实例初始化，我们先找到`Father`中的非静态变量和非静态代码块，最后是`Father`类的构造器，可得出打印顺序为：3，4，2
4. 接下来进行子类的实例化：我们找到`Son`类中的非静态变量和非静态代码块，最后是`Son`类的构造器,可得出打印顺序为：9，8，7
5. 子类实例化结束后，输出：----------
6. `Son son1 = new Son();`又`new`一次，由于已经进行了类的初始化，所以，只需要再进行一次实例的初始化就好，可得出打印顺序为：3，4，2，9，8，7

**第二题：**

```java
public class A extends B {
    //静态变量
    static int i = 1;

    //静态语句块
    static {
        System.out.println("Class A1:static blocks:" + i);
    }

    //非静态变量
    int j = 1;

    //静态语句块
    static {
        i++;
        System.out.println("Class A2:static blocks:" + i);
    }

    //构造函数
    public A() {
        super();
        i++;
        j++;
        System.out.println("constructor A: " + "i=" + i + ",j=" + j);
    }

    //非静态语句块
    {
        i++;
        j++;
        System.out.println("Class A:common blocks:" + "i=" + i + ",j=" + j);
    }

    //非静态方法
    public void aDisplay() {
        i++;
        System.out.println("Class A:static void aDisplay(): " + "i=" + i + ",j=" + j);
    }

    //静态方法
    public static void aTest() {
        i++;
        System.out.println("Class A:static void aTest():    " + "i=" + i);
    }
}
```

```java
public class B {
    static int i = 1;

    static {
        System.out.println("Class B1:static blocks:" + i);
    }

    int j = 1;

    static {
        i++;
        System.out.println("Class B2:static blocks:" + i);
    }

    public B() {
        i++;
        j++;
        System.out.println("constructor B: " + "i=" + i + ",j=" + j);
    }

    {
        i++;
        j++;
        System.out.println("Class B:common blocks:" + "i=" + i + ",j=" + j);
    }

    public void bDisplay() {
        i++;
        System.out.println("Class B:static void bDisplay(): " + "i=" + i + ",j=" + j);
    }

    public static void bTest() {
        i++;
        System.out.println("Class B:static void bTest():    " + "i=" + i);
    }
}
```

```java
public class ClassLoading {
    public static void main (String args[]) {
        A a=new A();
        a.aDisplay();
    }
}
```

这一题先不要看答案自己分析一下，再和答案进行对比。

```tex
Class B1:static blocks:1
Class B2:static blocks:2
Class A1:static blocks:1
Class A2:static blocks:2
Class B:common blocks:i=3,j=2
constructor B: i=4,j=3
Class A:common blocks:i=3,j=2
constructor A: i=4,j=3
Class A:static void aDisplay(): i=5,j=3
```



