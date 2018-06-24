---
layout: post
title: Exception和Error有什么区别
tags: Java
---
- Exception：在程序正常运行中出现的意外，可以捕获，进行相应的处理。
- Error：在正常情况下不太可能出现的情况，绝大部分Error会导致程序不可恢复。

Java异常图：
![png](http://upyun.midnight2104.com/blog/java-Throwable.png)

- Checked异常必须被显式地捕获或者传递（比如IOException），而unchecked异常则可以不必捕获或抛出（比如NullPointerException）。

- Checked异常继承java.lang.Exception类。Unchecked异常继承自java.lang.RuntimeException类。

##### 处理异常方法：
- try-catch-finally
- throw new Object() - throws

##### 注意事项：
```java

try{
 
    // to do something
}
catch(Exception e){ //不要写Exception ，范围太大，更好的是异常要确切一点
    e.printStackTrace();  //不要直接打印，用日志保存，因为在复杂系统中，这个结果不知道在哪儿去了
}
```
- try-catch 代码段会产生额外的性能开销，或者换个角度说，它往往会影响 JVM 对代码进行优化（被try-catch包裹的代码JVM不会进行JIT优化），所以建议仅捕获有必要的代码段，尽量不要一个大的 try 包住整段的代码；
- Java 每实例化一个Exception，都会对当时的栈进行快照（栈快照是记录当前线程状态，变量，代码等信息），这是一个相对比较重的操作。


##### 常见异常：

 - java.lang.NullPointerException（空指针异常）
 ```java
 String s = null;
 s.length();// s对象为空，没法使用，会抛出NullPointerException
 
 ```
 -  java.lang.IndexOutOfBoundsException（数组下标越界异常）
 ```java
  int[] a = {1,2};
	  a[6] = 3; //超过数组的最大长度 2 了，已经越界了
 ```
 - java.lang.ArithmeticException（算术异常）
 ```java
int i = 3 / 0;  //分母为 0 ，抛出算术异常
  ```
 - java.lang.FileNotFoundException（ 文件未找到异常）：文件路径不正确或者根本没有这个文件
 ```java
 File file = new File("F:/mywork/work.txt");

		if (!file.exists()) {
			throw new FileNotFoundException();
		}
 
 ```
 - java.lang.ClassNotFoundException（类没有找到异常）
 ```java
 package com.midnight.test;

public class Test {  
  public static void main(String[] args) {   
	  try {
		Class.forName("com.midnight.test.Hello"); //如果包 com.midnight.test下面没有Hello.java 这个类就会抛异常（编译时没有问题，运行时抛异常）
	} catch (ClassNotFoundException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
  }  
}
 
 ```
- java.lang.NoClassDefFoundError（类没有找到错误。注意这是一个Error）:
类文件存在，但是存在不同的域中
```java
package suba;//当前类会打包到suba

public class Hello {  
  public static void main(String[] args) {   
	System.out.println("hello");
  } 
}
//如果当前类在a目录下，进行编译时没有问题，但是运行时会出错NoClassDefFoundError，因为这个类实际上是这样的suba.Hello（在a的子目录suba下面）
```