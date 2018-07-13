---
layout: post
title: 谈谈final、finally、 finalize 有什么区别
tags: Java
---
### final
- 修饰类表示该类不可以被继承
- 修饰字段或变量表示不可以被修改（注意修饰引用变量时，引用不可以修改，但是引用指向的类是可以被修改的）
- 修饰方法时表示不可以被重写。

```java
        final  int i = 9;
        i = 10; //会报错，变量 i 被final修饰，不可以改变
        
        
        final List<Integer> list1 = new ArrayList<>();
        List<Integer> list2 = new ArrayList<>();
        list1 = list2; //会报错，变量 list1 被final修饰，不可以改变
        list1.add(1); //正常执行，引用不可以修改，但是引用指向的类是可以被修改的

```
### finally
> finally是用于保证代码一定会被执行，常用于异常捕获，关闭资源（不过在Java7以后推荐使用try-with-resources的方式关闭资源,资源可以自动关闭，代码更优雅）。

```java
 //常规读取文件的方式，需要处理很多异常
 public static  void main(String[] args){
        InputStream is = null;
        OutputStream os = null;
        try{
            is = new FileInputStream("F:\\inputFile.txt");
            os = new FileOutputStream("F:\\outputFile.txt");

            byte[] b = new byte[1024];
            int len = 0;
            while((len = is.read(b)) != -1 ){
                os.write(b,0,len);
            }
        }catch (FileNotFoundException e){
            e.printStackTrace();
        }catch (IOException e){
            e.printStackTrace();
        }
        finally {
            try{
                is.close();
            }catch (IOException e){
                e.printStackTrace();
            }
            try{
                os.close();
            }catch (IOException e){
                e.printStackTrace();
            }
        }

    }

```
```java
//使用try-with-resources的方式关闭资源
try( InputStream is = new FileInputStream("F:\\inputFile.txt");
             OutputStream os = new FileOutputStream("F:\\outputFile.txt") ){

            byte[] b = new byte[1024];
            int len = 0;
            while((len = is.read(b)) != -1 ){
                os.write(b,0,len);
            }
        }catch (FileNotFoundException e){
            e.printStackTrace();
        }catch (IOException e){
            e.printStackTrace();
        }

```
> 注意：几个特例中finally不会执行
```java
//1. 程序被强制退出
    try{
            System.exit(-1);
        }finally {
            System.out.println("aa");
        }
        
//2.try中有死循环
    try{
            while(true){
                System.out.println("ss");
            }
        }finally {
            System.out.println("aa");
        }        
        
//3.线程被杀死
```

### finalize
> `finalize`是`Object`对象自带的方法，用于对象被清除前，确保资源的回收。但是自己实现这个方法进行资源回收时，会降低性能。所以并不推荐使用它，并且在 `JDK 9 `开始被标记为 `deprecated`。`Java`平台目前在逐步使用 `java.lang.ref.Cleaner`来替换掉原有的 `finalize` 实现。`Cleaner` 的实现利用了幻象引用（`PhantomReference`），这是一种常见的所谓 `post-mortem `清理机制。


> 参考资源：[极客时间--Java核心技术36讲](https://time.geekbang.org/column/article/6906)