---
layout: post
title: Soul网关中的divide插件
tags: Soul
---

 今天体验的是`Soul`中`divide`插件，主要作用是用于`http`的代理。

#### 请求转发

1.`Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-http`模块演示的通`http`发起请求到`soul`网关，然后再到真实的服务。模块目录及配置信息如下：

![1](https://midnight2104.github.io/img/2021-1-15/1.png)

​	`soul.http`是有关`Soul`的配置，`adminUrl`是`Soul`的后台管理地址，`port`是业务系统的端口，`contextPath`是业务系统的请求路径。

2.在项目的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
            <version>${soul.version}</version>
        </dependency>
```

3.在需要被代理的接口上使用注解`@SoulSpringMvcClient`，`@SoulSpringMvcClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![1](https://midnight2104.github.io/img/2021-1-15/2.png)

如果其他接口也想被网关代理，使用方式是一样的，在`@SoulSpringMvcClient`注解中，指定`path`即可。

4.参考上一篇的[Soul入门](https://midnight2104.github.io/2021/01/14/Soul%E5%85%A5%E9%97%A8/)，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台管理地址，是一个`SpringBoot`项目，只需要修改一下数据库的地址就可以运行了。项目会自动创建对应的库和表。项目启动后的登录地址是`http://localhost:9095/`，用户名是`admin`，密码是`123456`。后台界面如下：

![1](https://midnight2104.github.io/img/2021-1-15/3.png)

5.运行`SoulTestHttpApplication`，启动`soul-examples-http`项目。

6.三个系统（本身的业务系统，Soul后台管理系统`Soul Admin`，Soul核心网关`Soul Bootstrap`）都启动成功后，就能够测试一把。

```sh
发起一个Get请求： http://localhost:8188/order/findById?id=99 
得到的响应结果：
{
  "id": "99",
  "name": "hello world findById"
}
```

上面就是一个普通的`http`请求，直接请求业务系统的后端服务，现在通过`Soul`网关来访问该服务。

```sh
同样发起一个Get请求：http://localhost:9195/http/order/findById?id=99
得到的响应结果：
{
  "id": "99",
  "name": "hello world findById"
}
```

这个`localhost:9195`地址就是网关的地址，`/http`是业务系统在网关中的名称。那么，现在的请求就是先通过`Soul`网关，再由网关转发到实际的请求接口。

通过后台管理系统可以发现：主要模块有插件列表和系统管理，在插件列表中可以对各个插件进行管理，每个插件都可以添加多个选择器，每个选择器都可以添加多条规则。实际这就是`Soul`拦截`URL`后的匹配规则：`插件->选择器->规则`，这个后面再细说。

以上就是`Soul`作为一个网关起到转发的作用，这个功能模块对应的插件是`divide`插件。其实，这个插件还能完成负载均衡的功能。

#### 负载均衡

1. 修改`soul-examples-http`项目端口（比如8189），在`Idea`中运行这个项目（设置为可以并行运行）。

   ```yml
   server:
     port: 8189
     address: 0.0.0.0
   
   
   soul:
     http:
       adminUrl: http://localhost:9095
       port: 8189
       contextPath: /http
       appName: http
       full: false
   ```

   ![1](https://midnight2104.github.io/img/2021-1-15/4.png)

2. 启动成功后，在`Soul`的后台系统中的`divide`插件中可以看到这个项目的地址信息了。现在再来发起请求。

   ```xml
   同样发起一个Get请求：http://localhost:9195/http/order/findById?id=99
   得到的响应结果：
   {
     "id": "99",
     "name": "hello world findById"
   }
   ```

   ![1](https://midnight2104.github.io/img/2021-1-15/5.png)

   以上请求总共发现四次，然后我们观察一下`soul-bootstrap`的控制台信息：

   ![1](https://midnight2104.github.io/img/2021-1-15/6.png)

   通过日志信息可以清楚的看到4次请求`localhost:9195`，分别转发到`192.168.1.6:8188`，`192.168.1.6:8189`，`192.168.1.6:8188`，`192.168.1.6:8189`。这个转发还是很均衡，可以通过在后台地址修改权重，将请求分配到指定的地方。

3. 修改权重，将请求转发到`192.168.1.6:8189`。

![1](https://midnight2104.github.io/img/2021-1-15/7.png)

现在再来发起4次请求。

```xml
同样发起一个Get请求：http://localhost:9195/http/order/findById?id=99
得到的响应结果：
{
  "id": "99",
  "name": "hello world findById"
}
```

以上请求总共发现四次，然后我们观察一下`soul-bootstrap`的控制台信息：

![1](https://midnight2104.github.io/img/2021-1-15/8.png)

通过日志信息可以清楚的看到4次请求`localhost:9195`，总是转发到`192.168.1.6:8189`，`192.168.1.6:8189`，`192.168.1.6:8189`，`192.168.1.6:8189`。这样，我们就通过给不同配置设置不同权重，网关将根据权重分配请求。

最后，这篇文章主要介绍了`Soul`中的`divide`插件可以作为`http`的代理，也可以很容易实现负载均衡。





