---
title: CountDownLatch与thread-join()的区别
date: 2018-4-9 14:30:31
tags: java并发
---

今天学习CountDownLatch这个类，作用感觉和join很像，然后就百度了一下，看了他们之间的区别。所以在此记录一下。

首先来看一下join，在当前线程中，如果调用某个thread的join方法，那么当前线程就会被阻塞，直到thread线程执行完毕，当前线程才能继续执行。join的原理是，不断的检查thread是否存活，如果存活，那么让当前线程一直wait，直到thread线程终止，线程的this.notifyAll 就会被调用。

我们来看一下这个应用场景：假设现在公司有三个员工A,B,C，他们要开会。但是A需要等B,C准备好之后再才能开始，B,C需要同时准备。我们先用join模拟上面的场景。

Employee.java:
```java
public class Employee extends Thread{
        
	private String employeeName;
	
	private long time;

	public Employee(String employeeName,long time){
		this.employeeName = employeeName;
		this.time = time;
	}
	
	@Override
	public void run() {
		try {
			System.out.println(employeeName+ "开始准备");
			Thread.sleep(time);
			System.out.println(employeeName+" 准备完成");
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```
JoinTest.java:
```java
public class JoinTest {

	public static void main(String[] args) throws InterruptedException {
		Employee a = new Employee("A", 3000);
		Employee b = new Employee("B", 3000);
		Employee c = new Employee("C", 4000);
		
		b.start();
		c.start();
		
		b.join();
		c.join();
		System.out.println("B,C准备完成");
		a.start();
	}
}
```
最后输出结果如下:
```shell
C开始准备
B开始准备
B 准备完成
C 准备完成
B,C准备完成
A开始准备
A 准备完成
```
可以看到,A总是在B,C准备完成之后才开始执行的。

CountDownLatch中我们主要用到两个方法一个是await()方法，调用这个方法的线程会被阻塞，另外一个是countDown()方法，调用这个方法会使计数器减一，当计数器的值为0时，因调用await()方法被阻塞的线程会被唤醒，继续执行。
<!-- more -->
接下来，我们用CountDownLatch来模拟一下。

Employee.java:
```java
public class Employee extends Thread{

	private String employeeName;
	
	private long time;
	
	private CountDownLatch countDownLatch;

	public Employee(String employeeName,long time, CountDownLatch countDownLatch){
		this.employeeName = employeeName;
		this.time = time;
		this.countDownLatch = countDownLatch;
	}
	
	@Override
	public void run() {
		try {
			System.out.println(employeeName+ "开始准备");
			Thread.sleep(time);
			System.out.println(employeeName+" 准备完成");
			countDownLatch.countDown();
		} catch (Exception e) {
			e.printStackTrace();
		}
		
	}
}
```
CountDownLatchTest.java:
```java
public class CountDownLatchTest {
	public static void main(String[] args) throws InterruptedException {
		CountDownLatch countDownLatch = new CountDownLatch(2);
		Employee a = new Employee("A", 3000,countDownLatch);
		Employee b = new Employee("B", 3000,countDownLatch);
		Employee c = new Employee("C", 4000,countDownLatch);
		
		b.start();
		c.start();
		countDownLatch.await();
		System.out.println("B,C准备完成");
		a.start();
	}
}
```

输出结果如下:
```
B开始准备
C开始准备
B 准备完成
C 准备完成
B,C准备完成
A开始准备
A 准备完成
```
上面可以看到，CountDownLatch与join都能够模拟上述的场景，那么他们有什么不同呢?这时候我们试想另外一个场景就能看到他们的区别了。

假设A，B，C的工作都分为两个阶段，A只需要等待B，C各自完成他们工作的第一个阶段就可以执行了。

我们来修改一下Employee类：
```java
public class Employee extends Thread{

	private String employeeName;
	
	private long time;
	
	private CountDownLatch countDownLatch;

	public Employee(String employeeName,long time, CountDownLatch countDownLatch){
		this.employeeName = employeeName;
		this.time = time;
		this.countDownLatch = countDownLatch;
	}
	
	@Override
	public void run() {
		try {
			System.out.println(employeeName+ " 第一阶段开始准备");
			Thread.sleep(time);
			System.out.println(employeeName+" 第一阶段准备完成");
			
			countDownLatch.countDown();
			
			System.out.println(employeeName+ " 第二阶段开始准备");
			Thread.sleep(time);
			System.out.println(employeeName+" 第二阶段准备完成");
			
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}

```

CountDownLatchTest类不需要做修改，输出结果入下：

```
B 第一阶段开始准备
C 第一阶段开始准备
B 第一阶段准备完成
B 第二阶段开始准备
C 第一阶段准备完成
C 第二阶段开始准备
B,C第一阶段准备完成
A 第一阶段开始准备
B 第二阶段准备完成
A 第一阶段准备完成
A 第二阶段开始准备
C 第二阶段准备完成
A 第二阶段准备完成
```
从结果可以看出，A在B，C第一阶段准备完成的时候就开始执行了，不需要等到第二阶段准备完成。这种场景下，用join是没法实现的。

总结：调用join方法需要等待thread执行完毕才能继续向下执行,而CountDownLatch只需要检查计数器的值为零就可以继续向下执行，相比之下，CountDownLatch更加灵活一些，可以实现一些更加复杂的业务场景。


参考：[https://blog.csdn.net/zhutulang/article/details/48504487](https://blog.csdn.net/zhutulang/article/details/48504487)
