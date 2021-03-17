---
layout: post
title: Soul源码阅读系列（二）Divide插件
tags: Soul
---

 今天体验的是`Soul`中`divide`插件，它的主要作用是用于`http`的代理。在文章后面一节简单分析了`divide`插件的执行原理。

### Divide插件使用案例

`Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-http`模块演示的通`http`发起请求到`soul`网关，然后再到真实的服务。模块目录及配置信息如下：

![](https://qiniu.midnight2104.com/20210310/1.png)

​	`soul.http`是有关`Soul`的配置，`adminUrl`是`Soul`的后台管理地址，`port`是业务系统的端口，`contextPath`是业务系统的请求路径。

在项目的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。

```xml
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
    <version>${soul.version}</version>
</dependency>
```

在需要被代理的接口上使用注解`@SoulSpringMvcClient`，`@SoulSpringMvcClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![](https://qiniu.midnight2104.com/20210310/2.png)

如果其他接口也想被网关代理，使用方式是一样的，在`@SoulSpringMvcClient`注解中，指定`path`即可。

参考之前的文章，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台管理地址，是一个`SpringBoot`项目，只需要修改一下数据库的地址就可以运行了。项目会自动创建对应的库和表。项目启动后的登录地址是`http://localhost:9095/`，用户名是`admin`，密码是`123456`。后台界面如下：

![](https://qiniu.midnight2104.com/20210310/3.png)



最后运行`SoulTestHttpApplication`，启动`soul-examples-http`项目。

当三个系统（本身的业务系统，Soul后台管理系统`Soul Admin`，Soul核心网关`Soul Bootstrap`）都启动成功后，就能够使用`divide`插件了。

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

以上就是`Soul`作为一个网关起到转发的作用，这个功能模块对应的插件是`divide`插件。接下来，我们跟踪一下`divide`插件的源码，看看它的执行原理。

### Divide插件执行原理

当我们第一次接触时，可能不知道它的执行逻辑在源代码的哪个位置，那怎么办呢？ 

答案是 **猜**，如何猜测呢？我们想要查看的是`divide`插件，那就去插件模块`soul-plugin`看看。然后再找找有没有跟`divide`有关的，发现有一个`soul-plugin-divide`。进入这个模块里面，有一个`DividePlugin`类，它有`doExecute()`方法，那我们也能猜测它可能就是`divide`插件的执行逻辑。

![](https://qiniu.midnight2104.com/20210310/4.png)

有了上面的猜想，我们还需要进行验证，看看对不对，在`doExecute()`方法加上断点进行`debug`调试。将`soul-bootstrap`项目以`debug`模式进行重启，然后发起请求：

```sh
http://localhost:9195/http/order/findById?id=99
```



成功发起请求后，执行逻辑会在我们打断点的地方停住，那么证明我们的猜想是正确的（在这里可以再次体会到命名的重要性）。这个时候要注意观察`IDEA`编辑器提供的方法调用栈信息：

![](https://qiniu.midnight2104.com/20210310/5.png)

方法调用栈里面有很多方法，但是前面四个是`soul`中的方法调用，后面的与`reactor`编程模型有关，先暂时忽略它。在前面四个方法中，调用关系如下：

![](https://qiniu.midnight2104.com/20210310/6.png)

- `SoulWebHandler`：它实现了`WebHandler`，重写了`handle()`方法，用于处理`Soul`网关中所有的请求。
- `DefaultSoulPluginChain`：插件链执行类，以责任链的设计模式处理所有插件。
- `AbstractSoulPlugin`：多个插件的父类，以模板方法设计模式实现各种插件类型。
- `DividePlugin`：`divide`插件，用于处理`http`请求。

分析到这里，就能够清楚的看到`Soul`网关处理一个`http`请求的过程，具体实现就在以上四个类及对应的方法中。实际的代码解析将在下一篇文章中进行分析，因为比较多，做好准备~

到这里，本篇文章就结束了，小结一下：本篇文章通过一个案例演示了`http`请求怎么接入到`Soul`网关中，以及`Divide`插件的执行原理。