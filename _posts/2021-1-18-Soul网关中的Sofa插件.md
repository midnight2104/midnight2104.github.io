---
layout: post
title: Soul网关中的sofa插件
tags: Soul
---

本篇文章主要介绍学习使用`sofa`插件，如何将`sofa`服务接入到`Soul`网关，以及`sofa`的简单介绍。主要内容如下：



 - 在`Soul`中使用`sofa`服务

   -   查看官方样例
   - 引入依赖
   - 注册`sofa`服务
   - 运行`sofa`服务
   - 启动`Soul Admin`和`Soul Bootstrap`
   - 体验`sofa`服务
 - 关于`sofa`

   - `sofa`是什么

   - `sofa`基本原理
    
    



今天体验的是`Soul`中`sofa`插件，如果业务系统是由`sofa`构建而成的，当需要`Soul`网关的支持时，可以将自己的`sofa`服务接入`soul`网关。





 #### 1. 在`Soul`中使用`sofa`服务



 ##### 1.1 查看官方样例



  `Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-sofa`模块演示的是`Soul`网关对`sofa`服务的支持。模块目录及配置信息如下：

![1](https://midnight2104.github.io/img/2021-1-18/1.png)


​	`soul.sofa`是有关`Soul`对`sofa`插件支持的配置，`adminUrl`是`Soul`的后台管理地址，`contextPath`是业务系统的请求路径上下文。

##### 1.2 引入依赖
在`sofa`服务的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。


```xml
        <properties>
            <rpc-sofa-boot-starter.version>6.0.4</rpc-sofa-boot-starter.version>
        </properties>       
		<dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>rpc-sofa-boot-starter</artifactId>
            <version>${rpc-sofa-boot-starter.version}</version>
        </dependency>
 		<dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-sofa</artifactId>
            <version>${soul.version}</version>
            <exclusions>
                <exclusion>
                    <artifactId>guava</artifactId>
                    <groupId>com.google.guava</groupId>
                </exclusion>
            </exclusions>
        </dependency>
```

##### 1.3 注册`sofa`服务
在需要被代理的接口上使用注解`@SoulSofaClient`，`@SoulSofaClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![1](https://midnight2104.github.io/img/2021-1-18/2.png)


如果其他接口也想被网关代理，使用方式是一样的。在`@SoulSofaClient`注解中，指定`path`即可。
##### 1.4 运行`sofa`服务
运行`TestSofaApplication`，启动`soul-examples-sofa`项目。`sofa`是需要注册中心的。本文使用的是`zookeeper`，启动也很简单。在官网下载，然后解压，直接运行`zkServer.cmd`就可以运行。

![1](https://midnight2104.github.io/img/2021-1-18/3.png)

##### 1.5 启动`Soul Admin`和`Soul Bootstrap`
参考上一篇的[Soul入门](https://midnight2104.github.io/2021/01/14/Soul%E5%85%A5%E9%97%A8/)，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台界面如下：


![1](https://midnight2104.github.io/img/2021-1-18/4.png)

如果`sofa`插件没有开启，需要手动在管理界面开启一下。

![1](https://midnight2104.github.io/img/2021-1-18/5.png)

在`Soul Bootstrap`中，加入相关依赖：

```xml
 	<!--soul sofa plugin start-->
        <dependency>
           <groupId>com.alipay.sofa</groupId>
           <artifactId>sofa-rpc-all</artifactId>
           <version>5.7.6</version>
       </dependency>
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
       <dependency>
           <groupId>org.dromara</groupId>
           <artifactId>soul-spring-boot-starter-plugin-sofa</artifactId>
                <!-- 当前版本是2.2.1-->
           <version>${project.version}</version>
       </dependency>
     <!-- soul  sofa plugin end-->
```

##### 1.6 体验`sofa`服务
三个系统（本身的业务系统(这里就是`soul-examples-sofa`)，`Soul`后台管理系统`Soul Admin`，Soul核心网关`Soul Bootstrap`）都启动成功后，就能够体验到`sofa`服务在网关`soul`中的接入。

![1](https://midnight2104.github.io/img/2021-1-18/sofa.png)

```java
	 //实际sofa提供的服务
    @SoulSofaClient(path = "/findAll", desc = "Get all data")
    public DubboTest findAll() {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setName("hello world Soul Sofa , findAll");
        dubboTest.setId(String.valueOf(new Random().nextInt()));
        return dubboTest;
    }
```



上面向网关发起了一个请求`http://localhost:9195/sofa/findAll`，实际被调用的是`sofa`的服务。



#### 2. 关于`sofa`



##### 2.1 sofa是什么

在之前，我还没有使用过`sofa`，这里直接引用了官方的介绍：

> `SOFARPC` 是蚂蚁金服开源的一款基于 `Java` 实现的 `RPC` 服务框架，为应用之间提供远程服务调用能力，具有高可伸缩性，高容错性，目前蚂蚁金服所有的业务的相互间的 `RPC` 调用都是采用 `SOFARPC`。`SOFARPC` 为用户提供了负载均衡，流量转发，链路追踪，链路数据透传，故障剔除等功能。
>
> `SOFARPC` 还支持不同的协议，目前包括 [bolt](https://www.sofastack.tech/projects/sofa-rpc/bolt)，[RESTful](https://www.sofastack.tech/projects/sofa-rpc/restful)，[dubbo](https://www.sofastack.tech/projects/sofa-rpc/dubbo)，[H2C](https://www.sofastack.tech/projects/sofa-rpc/h2c) 协议进行通信。其中` bolt` 是蚂蚁金融服务集团开放的基于 Netty 开发的网络通信框架。


个人理解：`sofa`是一个轻量级的`dubbo`。
##### 2.2 `sofa`基本原理

![1](https://midnight2104.github.io/img/2021-1-18/6.png)

- 1. 当一个 `SOFARPC `的应用启动的时候，如果发现当前应用需要发布` RPC `服务的话，那么 `SOFARPC `会将这些服务注册到服务注册中心上。如图中 `Service` 指向 `Registry`。
- 2. 当引用这个服务的 `SOFARPC `应用启动时，会从服务注册中心订阅到相应服务的元数据信息。服务注册中心收到订阅请求后，会将发布方的元数据列表实时推送给服务引用方。如图中` Registry `指向 `Reference`。
- 3. 当服务引用方拿到地址以后，就可以从中选取地址发起调用了。如图中 `Reference` 指向 `Service`。

最后，这篇文章主要介绍了`Soul`对`sofa`提供的支持，结合实际案例进行了演示。给出了`sofa`的简单介绍。



参考地址：

- [SOFARPC 介绍](https://www.sofastack.tech/projects/sofa-rpc/overview/)
- [sofa用户接入soul](https://dromara.org/zh-cn/docs/soul/user-sofa.html)

