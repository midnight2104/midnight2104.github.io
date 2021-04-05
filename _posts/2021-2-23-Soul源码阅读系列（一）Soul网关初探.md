---
layout: post
title: Soul源码阅读系列（一）Soul网关初探
tags: Soul
---

本篇文章主要内容如下：

- `Soul`是什么
- 如何在本地运行`Soul`
- 对`Soul`进行压测

### Soul 是什么

`Soul`是什么？它可不是灵魂交友软件！

引用`Soul`的官网，它是这样描述`Soul`的：

> 这是一个异步的，高性能的，跨语言的，响应式的`API`网关。我希望能够有一样东西像灵魂一样，保护您的微服务。参考了`Kong`，`Spring-Cloud-Gateway`等优秀的网关后，站在巨人的肩膀上，`Soul`由此诞生！

`Soul`是一个网关，它的特点如下：

- 支持各种语言(`http`协议)，支持 `dubbo`，`spring cloud`协议。
- 插件化设计思想，插件热插拔，易扩展。
- 灵活的流量筛选，能满足各种流量控制。
- 内置丰富的插件支持，鉴权，限流，熔断，防火墙等等。
- 流量配置动态化，性能极高，网关消耗在` 1~2ms`。
- 支持集群部署，支持 `A/B Test`，蓝绿发布。

来自官网的一张架构图

![1](https://qiniu.midnight2104.com/20210304/1.png)



从上面的架构图可以看出，`Soul`网关与语言无关，各种语言（`Java`,`PHP`,`.NET`等）都可以接入到网关中。它通过对不同插件的支持实现各种功能（监控，认证，限流，熔断，不同用户接入等）。在后台管理系统（`soul-admin`）就可以灵活配置各种流量。

### 如何在本地运行 Soul

好了，知道了`Soul`是一个网关，那么接下来就看看怎么使用它。 通过案例演示的方式比直接了解各个概念的方式更能激发兴趣，所以开始`play it`吧！

1. 从官网拉取项目源码 `git clone git@github.com:dromara/soul.git`，当前最新版本是`2.2.1`。

2. 创建并切换分支`git checkout -b myLearn`，就在自己的在本地跑，直接用`master`分支也行。

3. 使用`IDEA`打开项目，然后本地编译一下，确保没有错。

   ```sh
   mvn clean install
   ```

   第一次编译会很慢，需要下载依赖。当然，也可以跳过相关测试和注释，这样会快一点。

   ```sh
   mvn clean install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true
   ```

4. 启动`Soul`的后台管理地址，就是项目源码中的`soul-admin`模块，这是一个`SpringBoot`项目，只需要修改一下数据库的地址就可以运行了。项目会自动创建对应的库和表。

   ![2](https://qiniu.midnight2104.com/20210304/2.png)

   项目启动后的登录地址是`http://localhost:9095/`，默认用户名是`admin`，密码是`123456`。后台界面如下：

   ![3](https://qiniu.midnight2104.com/20210304/3.png)

   主要模块有插件列表和系统管理，在插件列表中可以对各个插件进行管理，每个插件都可以添加多个选择器，每个选择器都可以添加多条规则。实际这就是`Soul`根据请求的`URL`去匹配规则：`插件->选择器->规则`，这个后面再细说。

5. 启动`Soul`的核心模块`soul-bootstrap`，这是网关的核心处理逻辑。不要怕它，这个模块本身不复杂，目录结构如下：

   ![4](https://qiniu.midnight2104.com/20210304/4.png)

   

   启动成功后，就可以访问这个网关了`http://127.0.0.1:9195/`，返回信息如下：

   ```json
   {"code":-107,"message":"Can not find selector, please check your configuration!","data":null}
   ```

   因为还没有接入业务系统，所以没有相关返回值，上面展示的信息是`Soul`网关没有找到相应的选择器，返回的一个提示信息

6. 通过上述步骤，就成功的搭建起`Soul`网关服务了，后面就是在自己的业务系统上使用网关。使用例子可以参考`soul-examples`模块。

7. 本文运行`soul-examples`下面的 `http`服务，结合`divde`插件，发起`http`请求到`soul`网关，体验`http`代理。模块目录及配置信息如下：

   ![5](https://qiniu.midnight2104.com/20210304/5.png)
   
   配置文件中有关`soul`的配置我们后面再详细解释，现在只需要知道这个服务就是一个很普通的`Spring Boot`项目。其中的一个接口信息如下：
   
   ![6](https://qiniu.midnight2104.com/20210304/6.png)
   
   运行`SoulTestHttpApplication`，启动这个项目。成功之后，通过`postman`进行访问：
   
   ![7](https://qiniu.midnight2104.com/20210304/7.png)
   
   上面就是一个普通的`http`请求，直接请求业务系统的后端服务，现在通过`Soul`网关来访问该服务。
   
   ![8](https://qiniu.midnight2104.com/20210304/8.png)
   

这个`localhost:9195`地址就是网关的地址，`/http`是业务系统在网关中的名称。那么，现在的请求就是先通过`Soul`网关，再由网关转发到实际的请求接口。



### 对 Soul 进行压测

成功在本地运行`Soul`网关之后，我们最后再来看看`Soul`网关的性能如何。在`windows`平台可以通过`SuperBenchmarker`工具进行压测。压测设置的参数是200个请求，32个并发，100秒内执行完。

![9](https://qiniu.midnight2104.com/20210304/9.png)   

然后再来看一下直连的情况：

![10](https://qiniu.midnight2104.com/20210304/10.png)   

可以看到 直连的`RPS(3916.9)`还是大于经过网关转发后的`RPS(1526.3 )`，毕竟多了网关这一层的转发。

   

小结一下，本篇文章介绍了`Soul`网关的特点和架构设计。然后，拉取`soul`源码，在本地运行测试案例。最后，对`soul`网关进行了简单的压测。

   

   参考文章：

   - [soul介绍](https://dromara.org/zh-cn/docs/soul/soul.html)
   - [soul学习03——性能测试](https://blog.gaiaproject.club/soul-03/)
   - [Soul网关学习（1） 环境配置](https://www.yuque.com/chenxi-hs7pc/nksy3m/soul-source-1)

   

