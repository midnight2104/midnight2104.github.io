---
layout: post
title: Soul源码阅读系列（六）Sofa插件
tags: Soul

---

本篇文章主要介绍学习使用`sofa`插件，如何将`sofa`服务接入到`Soul`网关，以及`sofa`的简单介绍。主要内容如下：



 - 在`Soul`中使用`sofa`服务
   -   查看官方样例
   -   引入依赖
   -   注册`sofa`服务
   -   运行`sofa`服务
   -   启动`Soul Admin`和`Soul Bootstrap`
   -   体验`sofa`服务

 - 关于`sofa`
   - `sofa`是什么
   - `sofa`基本原理

- Sofa插件执行原理
  -   `BodyParamPlugin`插件
  -   `SofaPlugin`插件
  -   `SofaResponsePlugin`插件

- Sofa服务及代理对象的生成



今天体验的是`Soul`中`sofa`插件，如果业务系统是由`sofa`构建而成的，当需要`Soul`网关的支持时，可以将自己的`sofa`服务接入`soul`网关。

#### 1. 在`Soul`中使用`sofa`服务

##### 1.1 查看官方样例



  `Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-sofa`模块演示的是`Soul`网关对`sofa`服务的支持。模块目录及配置信息如下：

![1](https://qiniu.midnight2104.com/20210515/1.png)


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

![1](https://qiniu.midnight2104.com/20210515/2.png)


如果其他接口也想被网关代理，使用方式是一样的。在`@SoulSofaClient`注解中，指定`path`即可。

##### 1.4 运行`sofa`服务

运行`TestSofaApplication`，启动`soul-examples-sofa`项目。`sofa`是需要注册中心的。本文使用的是`zookeeper`，启动也很简单。在官网下载，然后解压，直接运行`zkServer.cmd`就可以运行。

![1](https://qiniu.midnight2104.com/20210515/3.png)

##### 1.5 启动`Soul Admin`和`Soul Bootstrap`

参考上一篇的[Soul入门](https://midnight2104.github.io/2021/01/14/Soul%E5%85%A5%E9%97%A8/)，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台界面如下：


![1](https://qiniu.midnight2104.com/20210515/4.png)

如果`sofa`插件没有开启，需要手动在管理界面开启一下。

![1](https://qiniu.midnight2104.com/20210515/5.png)

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

![1](https://qiniu.midnight2104.com/20210515/7.png)

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

![1](https://qiniu.midnight2104.com/20210515/6.png)

- 1. 当一个 `SOFARPC `的应用启动的时候，如果发现当前应用需要发布` RPC `服务的话，那么 `SOFARPC `会将这些服务注册到服务注册中心上。如图中 `Service` 指向 `Registry`。
- 2. 当引用这个服务的 `SOFARPC `应用启动时，会从服务注册中心订阅到相应服务的元数据信息。服务注册中心收到订阅请求后，会将发布方的元数据列表实时推送给服务引用方。如图中` Registry `指向 `Reference`。
- 3. 当服务引用方拿到地址以后，就可以从中选取地址发起调用了。如图中 `Reference` 指向 `Service`。

上面主要内容是介绍了`Soul`对`sofa`提供的支持，结合实际案例进行了演示。给出了`sofa`的简单介绍。



#### 3. `Sofa`插件执行原理

接下来就通过跟踪源码的方式来理解其中的执行原理。

在`Soul`网关中，`Sofa`插件负责将`http`协议转换成`sofa`协议，涉及到的插件有：`BodyParamPlugin`，`SofaPlugin`和`SofaResponsePlugin`。

- `BodyParamPlugin`：负责将请求的`json`放到`exchange`属性中；
- `SofaPlugin`：使用`Sofa`进行请求的泛化调用并返回响应结果；
- `SofaResponsePlugin`：包装响应结果。



##### 3.1 `BodyParamPlugin`插件

该插件在执行链路中是先执行的，负责处理请求类型，比如带有参数的`application/json`，不带参数的查询请求。

```java
//org.dromara.soul.plugin.sofa.param.BodyParamPlugin#execute  
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
		//省略了其他代码
        if (Objects.nonNull(soulContext) && RpcTypeEnum.SOFA.getName().equals(soulContext.getRpcType())) {

            //处理json
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            //处理x-www-form-urlencoded
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            //处理查询
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }
```



##### 3.2 `SofaPlugin`插件

完成`SofaDubbo`插件的核心处理逻辑：检查元数据 `-->`检查参数类型`-->`泛化调用。

```java
//org.dromara.soul.plugin.sofa.SofaPlugin#doExecute
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
		//省略了其他代码
        //检查元数据
        if (!checkMetaData(metaData)) {
			//省略了其他代码
        }
        //检查参数类型
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
           //省略了其他代码
        }
        //泛化调用
        final Mono<Object> result = sofaProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }
```



`genericInvoker()`方法中参数分别是请求参数`body`，有关服务信息的`metaData`，包含`web`信息的`exchange`。这里面的主要操作是：

- 根据请求路径从缓存获取服务配置信息；
- 获取代理对象；
- 请求参数转化为`sofa`泛化参数；
- `sofa`真正的泛化调用；
- 返回结果。



```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
    
    	//获取服务信息
        ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
		//获取代理对象
        GenericService genericService = reference.refer();
   		 //请求参数转化为泛化调用的参数
        Pair<String[], Object[]> pair;
        if (null == body || "".equals(body) || "{}".equals(body) || "null".equals(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = sofaParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
    	//异步返回结果
        CompletableFuture<Object> future = new CompletableFuture<>();
    	//响应回调
        RpcInvokeContext.getContext().setResponseCallback(new SofaResponseCallback<Object>() {
            @Override
            public void onAppResponse(final Object o, final String s, final RequestBase requestBase) {
                future.complete(o);
            }

            @Override
            public void onAppException(final Throwable throwable, final String s, final RequestBase requestBase) {
                future.completeExceptionally(throwable);
            }

            @Override
            public void onSofaException(final SofaRpcException e, final String s, final RequestBase requestBase) {
                future.completeExceptionally(e);
            }
        });
    
    //真正的泛化调用
        genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
    //返回结果    
    return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.SOFA_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.SOFA_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(SoulException::new);
    }
```



真正的`sofa`泛化调用是`genericService.$invoke()`。到这里，`SofaPlugin`插件的主要工作就完了，后面就是返回结果。



##### 3.3 `SofaResponsePlugin`插件

这个插件就是对结果再一次包装，处理错误信息和成功的结果信息。

```java
//org.dromara.soul.plugin.sofa.response.SofaResponsePlugin#execute   
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            final Object result = exchange.getAttribute(Constants.SOFA_RPC_RESULT);
            //处理错误的信息
            if (Objects.isNull(result)) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            //处理成功的信息
            Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
            return WebFluxResultUtils.result(exchange, success);
        }));
    }
```



至此，就跟踪完了`Soul`网关中对`Sofa`插件处理的核心操作：接入`sofa`服务，将`http`访问协议转化为`sofa`协议，通过`sofa`的泛化调用获取真正的接口服务信息。



#### 4. `Sofa`服务及代理对象的生成



 最后，我们还需要想到的是：服务的配置信息是怎么来的？以及代理对象是怎么得到的？

本文的分析思路和之前的`Apache Dubbo`是一样的。

```java
 public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
     //获取服务
        ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterfaceId())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getServiceName());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
     	//获取代理对象
     GenericService genericService = reference.refer();
     
     //省略了其他代码
     
        genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            //省略了其他代码
        })).onErrorMap(SoulException::new);
    }
```

先说结论：服务配置信息的来源是`soul-admin`同步到网关。

整个流程涉及到的类和方法如下：

`soul-admin`端：

- `WebsocketCollector`:`onMessage()`；
- `SyncDataServiceImpl`:`syncAll()`;
- `MetaDataService`:`syncData()`;

`soul-bootstrap`网关端：

- `SoulWebsocketClient`: `onMessage()`,`handleResult()`;
- `WebsocketDataHandler`:`executor()`;
- `AbstractDataHandler`:`handle()`;
- `MetaDataHandler`:`doRefresh()`;
- `SofaMetaDataSubscriber`:`onSubscribe()`;
- `ApplicationConfigCache`:`initRef()`,`build()`;

在本文的测试中，`soul-admin`端和`soul-bootstrap`网关端之间的数据同步是通过`websocket`进行。所以在网关启动的时候会进行数据同步操作。



在`soul-bootstrap`网关端前面几个类主要处理同步的数据类型，最终流转到`ApplicationConfigCache`，在这里面进行构建服务`build()`。

```java
public ConsumerConfig<GenericService> build(final MetaData metaData) {
        ConsumerConfig<GenericService> reference = new ConsumerConfig<>();
        reference.setGeneric(true);
        reference.setApplication(applicationConfig);
        reference.setRegistry(registryConfig); //注册中心地址
        reference.setInterfaceId(metaData.getServiceName());
        reference.setProtocol(RpcConstants.PROTOCOL_TYPE_BOLT);
        reference.setInvokeType(RpcConstants.INVOKER_TYPE_CALLBACK);
        reference.setRepeatedReferLimit(-1);
        String rpcExt = metaData.getRpcExt();//元数据信息
        SofaParamExtInfo sofaParamExtInfo = GsonUtils.getInstance().fromJson(rpcExt, SofaParamExtInfo.class);
        if (Objects.nonNull(sofaParamExtInfo)) {
            if (StringUtils.isNoneBlank(sofaParamExtInfo.getLoadbalance())) {
                final String loadBalance = sofaParamExtInfo.getLoadbalance();
                reference.setLoadBalancer(buildLoadBalanceName(loadBalance));
            }
            Optional.ofNullable(sofaParamExtInfo.getTimeout()).ifPresent(reference::setTimeout);
            Optional.ofNullable(sofaParamExtInfo.getRetries()).ifPresent(reference::setRetries);
        }
  	  //从注册中心获取代理服务
        Object obj = reference.refer();
        if (obj != null) {
            log.info("init sofa reference success there meteData is :{}", metaData.toString());
            cache.put(metaData.getPath(), reference);//将服务信息放到cache中去
        }
        return reference;
    }
```

在同步过程中，将所有的元数据信息及注册中心的服务放到了`cache`中，在使用的时候，就会到这个`cache`中去拿。就是在文章开始的地方提到的泛化调用。

```java
 public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
     //从缓存的cache中获取服务
        ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterfaceId())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getServiceName());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
     	//获取代理对象
     GenericService genericService = reference.refer();
     
     //省略了其他代码
     
        genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            //省略了其他代码
        })).onErrorMap(SoulException::new);
    }
```



总结：本文结合实际案例演示了`Soul`网关是如何接入`sofa`服务，然后通过源码跟踪的方式分析了`sofa`泛化服务的调用流程，最后分析了`Sofa`插件里面的服务是如何被加载，包括服务的配置信息和 代理对象的生成。