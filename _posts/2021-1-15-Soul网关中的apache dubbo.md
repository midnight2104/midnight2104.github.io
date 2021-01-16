---
layout: post
title: Soul网关中的apache dubbo插件
tags: Soul
---

 今天体验的是`Soul`中`apache dubbo`插件，如果业务系统是由`apache dubbo`构建而成的，又需要网关的支持，那么可以直接使用`Soul`。

1.`Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-apache-dubbo-service`模块演示的是`Soul`网关对`apache dubbo `系统的支持。模块目录及配置信息如下：

![1](https://midnight2104.github.io/img/2021-1-16/1.png)

​	`soul.dubbo`是有关`Soul`对`dubbo`插件支持的配置，`adminUrl`是`Soul`的后台管理地址，`contextPath`是业务系统的请求路径。

2.在项目的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-apache-dubbo</artifactId>
            <version>${soul.version}</version>
        </dependency>
```

3.在需要被代理的接口上使用注解`@SoulDubboClient，`@SoulDubboClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![1](https://midnight2104.github.io/img/2021-1-16/2.png)

如果其他接口也想被网关代理，使用方式是一样的。在`@SoulDubboClient`注解中，指定`path`即可。

运行`TestApacheDubboApplication`，启动`soul-examples-apache-dubbo-service`项目。`Dubbo`是需要注册中心的，可以使用`zookeeper`或者`nacos`。本文使用的是`zookeeper`，启动也很简单。在官网下载，然后解压，直接运行`zkServer.cmd`就可以运行。

4.参考上一篇的[Soul入门](https://midnight2104.github.io/2021/01/14/Soul%E5%85%A5%E9%97%A8/)，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台界面如下：

![1](https://midnight2104.github.io/img/2021-1-16/3.png)

如果`dubbo`插件没有开启，那么手动开启一下啊。

![1](https://midnight2104.github.io/img/2021-1-16/4.png)

在`Soul Bootstrap`中，加入相关依赖：

```xml
 <!--soul  apache dubbo plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-apache-dubbo</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.5</version>
        </dependency>
        <!-- Dubbo zookeeper registry dependency start -->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>
        <!-- Dubbo zookeeper registry dependency end -->
        <!-- soul  apache dubbo plugin end-->
```

5.三个系统（本身的业务系统(这里就是`soul-examples-apache-dubbo-service`)，`Soul`后台管理系统`Soul Admin`，Soul核心网关`Soul Bootstrap`）都启动成功后，就能够测试一把。

![1](https://midnight2104.github.io/img/2021-1-16/4.png)

```java
	 //实际dubbo提供的服务
    @SoulDubboClient(path = "/findAll", desc = "Get all data")
    public DubboTest findAll() {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setName("hello world Soul Apache, findAll");
        dubboTest.setId(String.valueOf(new Random().nextInt()));
        return dubboTest;
    }
```



上面向网关发起了一个请求`http://localhost:9195/dubbo/findAll`，实际被调用的是`dubbo`的服务。

另外也可以发起一个`POST`请求，下面向网关发起了一个请求`http://localhost:9195/dubbo/insert`，实际被调用的是`dubbo`的服务。

![1](https://midnight2104.github.io/img/2021-1-16/4.png)

```java
   //实际dubbo提供的服务
	@SoulDubboClient(path = "/insert", desc = "Insert a row of data")
    public DubboTest insert(final DubboTest dubboTest) {
        dubboTest.setName("hello world Soul Apache Dubbo: " + dubboTest.getName());
        return dubboTest;
    }
```



最后，这篇文章主要介绍了`Soul`对`Apache Dubbo`提供的支持，结合实际案例进行了演示。

