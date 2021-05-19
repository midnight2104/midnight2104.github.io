---
layout: post
title: Soul源码阅读系列（七）Spring Cloud插件
tags: Soul
---

本篇文章主要介绍学习使用`Spring Cloud`插件，如何将`Spring Cloud`服务接入到`Soul`网关。主要内容如下：

 - 在`Soul`中使用`Spring Cloud`服务

   -   查看官方样例
   - 引入依赖
   - 注册`Spring Cloud`服务
   - 运行`Spring Cloud`服务
   - 启动`Soul Admin`和`Soul Bootstrap`
   - 体验`Spring Cloud`服务
   
 - SpringCloudPlugin 源码解析

在前面几篇文章中，已经体验过了`Soul`中`divide`插件，`apache dubbo`插件和`sofa`插件，今天的`Spring Cloud`插件大体逻辑和之前的一致，接入和实现还会简单一些。

#### 1. 在`Soul`中使用`spring cloud`服务

##### 1.1 查看官方样例



  `Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-springcloud`模块演示的是`Soul`网关对`springcloud`服务的支持。模块目录及配置信息如下：

![1](https://midnight2104.github.io/img/2021-1-19/1.png)

有关的配置信息还是和之前一样。在本项目中`Spring Cloud`的注册中心使用的是`nacos`。

> `nacos`可以在官网直接下载，然后解压，在`bin`目录下使用命令`startup.cmd -m standalone`就能启动成功。

```yaml

server:
  port: 8884
  address: 0.0.0.0

spring:
  application:
    name: springCloud-test
  cloud: 
    nacos:
      discovery:
          server-addr: 127.0.0.1:8848 # 注册中心nacos的地址


springCloud-test:
  ribbon.NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule

soul:
  springcloud:
    admin-url: http://localhost:9095 # soul-admin的地址
    context-path: /springcloud

logging:
  level:
    root: info
    org.dromara.soul: debug
  path: "./logs"
```



##### 1.2 引入依赖
在`spring cloud`服务的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。


```xml
	  <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-springcloud</artifactId>
            <version>${soul.version}</version>
        </dependency>

	<!--使用nacos作为注册中心-->
       <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>2.1.0.RELEASE</version>
        </dependency>
```

##### 1.3 注册`spring cloud`服务
在需要被代理的接口上使用注解`@SoulSpringCloudClient`，`@SoulSpringCloudClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![1](https://midnight2104.github.io/img/2021-1-19/2.png)


如果其他接口也想被网关代理，使用方式是一样的。在`@SoulSpringCloudClient`注解中，指定`path`即可。
##### 1.4 运行`spring cloud`服务
运行`SoulTestSpringCloudApplication`，启动`soul-examples-springcloud`项目。成功启动后，可以在控制台看见接口被成功注册到`soul`网关中。

![1](https://midnight2104.github.io/img/2021-1-19/3.png)

##### 1.5 启动`Soul Admin`和`Soul Bootstrap`
参考上一篇的[Soul入门](https://midnight2104.github.io/2021/01/14/Soul%E5%85%A5%E9%97%A8/)，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台界面如下：


![1](https://midnight2104.github.io/img/2021-1-19/4.png)

如果`spring cloud`插件没有开启，需要手动在管理界面开启一下。

![1](https://midnight2104.github.io/img/2021-1-19/5.png)

在`Soul Bootstrap`中，加入相关依赖：

```xml
         <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-httpclient</artifactId>
            <version>${project.version}</version>
        </dependency>
	 <!--soul springCloud plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-springcloud</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-commons</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>

        <!--soul springCloud plugin start end-->

```

`httpclient`插件也是需要的，在`soul`网关将`http`协议转换为`spring cloud`协议后，还需要通过`httpclient`插件发起`web mvc`请求。

提一句，当`spring cloud`服务和`soul-bootstrap`等启动成功后，可以在注册中心看到这两个服务实例。

![1](https://midnight2104.github.io/img/2021-1-19/8.png)

##### 1.6 体验`spring cloud`服务

三个系统（本身的业务系统(这里就是`soul-examples-sofa`)，`Soul`后台管理系统`Soul Admin`，Soul核心网关`Soul Bootstrap`）都启动成功后，就能够体验到`sofa`服务在网关`soul`中的接入。

- 先直连`spring cloud`服务

![1](https://midnight2104.github.io/img/2021-1-19/6.png)

- 再通过`Soul`网关请求`spring cloud`服务

![1](https://midnight2104.github.io/img/2021-1-19/7.png)



```java
	 //实际spring cloud提供的服务
    @GetMapping("/findById")
    @SoulSpringCloudClient(path = "/findById")
    public OrderDTO findById(@RequestParam("id") final String id) {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setId(id);
        orderDTO.setName("hello world spring cloud findById");
        return orderDTO;
    }
```



上面演示的是先通过请求`http://localhost:8884/order/findById?id=99`直连`spring cloud`服务。再通过`Soul`网关发起请求`http://localhost:9195/springcloud/order/findById?id=99`，实际被调用的是`spring cloud`的服务。



#### SpringCloudPlugin 源码解析

```java
// org.dromara.soul.plugin.springcloud.SpringCloudPlugin#doExecute    
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
       //是否匹配上规则    
       if (Objects.isNull(rule)) {
            return Mono.empty();
        }
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
    //从缓存中获取选择器和规则信息
        final SpringCloudRuleHandle ruleHandle = SpringCloudRuleHandleCache.getInstance().obtainHandle(SpringCloudPluginDataHandler.getRuleCacheKey(rule));
        final SpringCloudSelectorHandle selectorHandle = SpringCloudSelectorHandleCache.getInstance().obtainHandle(selector.getId());
        if (StringUtils.isBlank(selectorHandle.getServiceId()) || StringUtils.isBlank(ruleHandle.getPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getCode(), SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }

    // 负载均衡选择实例
        final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
    //构建请求url
        final URI uri = loadBalancer.reconstructURI(serviceInstance, URI.create(soulContext.getRealUrl()));

        String realURL = buildRealURL(uri.toASCIIString(), soulContext.getHttpMethod(), exchange.getRequest().getURI().getQuery());

    //设置请求信息
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        //set time out.
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
    //处理下一个插件   
    return chain.execute(exchange);
    }
```

由`Spring Cloud`构建的微服务是支持`RESTful`，可以通过`http`请求发起调用。

在`SpringCloudPlugin`中设置好请求信息后，就交给后续的插件处理的，是需要要开启`divide`插件。



小结，这篇文章主要介绍了`Soul`对`spring cloud`提供的支持，结合实际案例进行了演示，简单跟读了下源码。

