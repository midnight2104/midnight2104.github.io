---
layout: post
title: Soul源码阅读系列（五）Apache Dubbo插件
tags: Soul

---

今天体验的是`Soul`中`apache dubbo`插件，如果你的业务系统是由`apache dubbo`构建而成的，又需要网关的支持，那么可以直接使用`Soul`接入。



### Apache Dubbo 插件的案例演示

`Soul`官方在`soul-examples`模块提供了测试样例，其中的`soul-examples-apache-dubbo-service`模块演示的是`Soul`网关对`apache dubbo `系统的支持。模块目录及配置信息如下：



![](https://qiniu.midnight2104.com/20210406/1.png)



`soul.dubbo`是有关`Soul`对`dubbo`插件支持的配置，`adminUrl`是`Soul`的后台管理地址，`contextPath`是业务系统的请求路径。

在项目的`pom`文件中引入`soul`相关依赖，当前版本是`2.2.1`。

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-apache-dubbo</artifactId>
            <version>${soul.version}</version>
        </dependency>
```



在需要被代理的接口上使用注解`@SoulDubboClient`，`@SoulDubboClient`注解会把当前接口注册到`soul`网关中。使用方式如下：

![](https://qiniu.midnight2104.com/20210406/2.png)



如果其他接口也想被网关代理，使用方式是一样的。在`@SoulDubboClient`注解中，指定`path`即可。

运行`TestApacheDubboApplication`，启动`soul-examples-apache-dubbo-service`项目。`Dubbo`是需要注册中心的，可以使用`zookeeper`或者`nacos`。本文使用的是`zookeeper`，启动也很简单。在官网下载，然后解压，直接运行`zkServer.cmd`就可以运行。

参考之前的文章，启动`Soul Admin`和`Soul Bootstrap`。`Soul`的后台界面如下：

![](https://qiniu.midnight2104.com/20210406/3.png)



如果`dubbo`插件没有开启，那就需要手动去开启。

![](https://qiniu.midnight2104.com/20210406/4.png)



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



三个系统（本身的业务系统(这里就是`soul-examples-apache-dubbo-service`)，`Soul`后台管理系统`Soul Admin`，`Soul`核心网关`Soul Bootstrap`）都启动成功后，就可以进行测试了。

![](https://qiniu.midnight2104.com/20210406/5.png)



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



![](https://qiniu.midnight2104.com/20210406/6.png)

```java

//实际dubbo提供的服务
@SoulDubboClient(path = "/insert", desc = "Insert a row of data")
public DubboTest insert(final DubboTest dubboTest) {
    dubboTest.setName("hello world Soul Apache Dubbo: " + dubboTest.getName());
    return dubboTest;
}
```



到这，就演示完了`Soul`对`Apache Dubbo`提供的支持，接下来再看看实际执行的过程。



### Apache Dubbo 插件执行原理



在`Soul`网关中，`Apache Dubbo`插件负责将`http`协议转换成`dubbo`协议，涉及到的插件有：`BodyParamPlugin`，`ApacheDubboPlugin`和`DubboResponsePlugin`。

- `BodyParamPlugin`：负责将请求的`json`放到`exchange`属性中；
- `ApacheDubboPlugin`：使用`Apache Dubbo`进行请求的泛化调用并返回响应结果；
- `DubboResponsePlugin`：包装响应结果。



![](https://qiniu.midnight2104.com/20210406/7.png)



#### `BodyParamPlugin`插件

该插件在执行链路中是最先执行的，负责处理请求类型，比如带有参数的`application/json`，不带参数的查询请求。

```java
//org.dromara.soul.plugin.apache.dubbo.param.BodyParamPlugin#execute   
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        //省略了其他代码
        if (Objects.nonNull(soulContext) && RpcTypeEnum.DUBBO.getName().equals(soulContext.getRpcType())) {
           //省略了其他代码
            //处理application/json
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            //处理application/x-www-form-urlencoded
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            //处理查询请求
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }
```



#### `ApacheDubboPlugin`插件

完成`ApacheDubbo`插件的核心处理逻辑：检查元数据 `-->`检查参数类型`-->`泛化调用。

```java
//org.dromara.soul.plugin.apache.dubbo.ApacheDubboPlugin#doExecute 
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        //检查元数据
        if (!checkMetaData(metaData)) {
			//省略了其他代码
        }
        //检查参数类型
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        //泛化调用
        final Mono<Object> result = dubboProxyService.genericInvoker(body, metaData, exchange);
        //执行下一个插件
        return result.then(chain.execute(exchange));
    }
```



`genericInvoker()`方法中参数分别是请求参数`body`，有关服务信息的`metaData`，包含`web`信息的`exchange`。这里面的主要操作是：

- 根据请求路径从缓存获取服务配置信息；
- 获取代理对象；
- 请求参数转化为`dubbo`泛化参数；
- `apache dubbo`真正的泛化调用；
- 返回结果。



```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
		//省略了其他代码
   	   //根据请求路径从缓存获取服务配置信息
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());

   		 //获取代理对象
        GenericService genericService = reference.get();
    	//请求参数转化为dubbo泛化参数
        Pair<String[], Object[]> pair;
        if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
     
    	//apache dubbo真正的泛化调用，注意这里是异步的泛化调用
        CompletableFuture<Object> future = genericService.$invokeAsync(metaData.getMethodName(), pair.getLeft(), pair.getRight());
    
    //返回结果
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.DUBBO_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        }))//省略了其他代码
    }
```



代码中`reference`的信息如下，这里明确可以看出是`dubbo`协议，有接口名称，超时时间，重试次数，负载均衡等信息。

```xml
<dubbo:reference protocol="dubbo" prefix="dubbo.reference.org.dromara.soul.examples.dubbo.api.service.DubboTestService" uniqueServiceName="org.dromara.soul.examples.dubbo.api.service.DubboTestService" interface="org.dromara.soul.examples.dubbo.api.service.DubboTestService" generic="true" generic="true" sticky="false" timeout="10000" retries="2" loadbalance="random" id="org.dromara.soul.examples.dubbo.api.service.DubboTestService" valid="true" />
```



代码中`genericService`对象信息如下，他是一个代理对象，接口信息是从`zookeeper`中拿到的（在项目演示时用`zookeeper`作为注册中心）。

```xml
invoker :interface org.apache.dubbo.rpc.service.GenericService -> zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?anyhost=true&application=soul_proxy&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=true&interface=org.dromara.soul.examples.dubbo.api.service.DubboTestService&loadbalance=random&methods=findById,insert,findAll&pid=22548&protocol=dubbo&register.ip=192.168.236.60&release=2.7.5&remote.application=test-dubbo-service&retries=2&side=consumer&sticky=false&timeout=10000&timestamp=1611918837429,directory: org.apache.dubbo.registry.integration.RegistryDirectory@46066629
```



真正的`apache dubbo`泛化调是`genericService.$invokeAsync()`，注意这里是异步的泛化调用。到这里，`ApacheDubboPlugin`插件的主要工作就完了，后面就是返回结果。



#### `DubboResponsePlugin`插件

这个插件就是对结果再一次包装，处理错误信息和成功的结果信息。

```java
//org.dromara.soul.plugin.apache.dubbo.response.DubboResponsePlugin#execute    
@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            final Object result = exchange.getAttribute(Constants.DUBBO_RPC_RESULT);
            //错误信息
            if (Objects.isNull(result)) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            //成功信息
            Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
            return WebFluxResultUtils.result(exchange, success);
        }));
    }
```



至此，就跟踪完了`Soul`网关中对`Apache Dubbo`插件处理的核心操作：接入`dubbo`服务，将`http`访问协议转化为`dubbo`协议，通过`dubbo`的泛化调用获取真正的接口服务信息。



### Apache Dubbo  服务来源



![](https://qiniu.midnight2104.com/20210406/8.png)



在前面，我们通过跟踪源码的方式理解了`Apache Dubbo`插件的执行原理，将发起的`http`请求转化为`dubbo`的泛化调用，但是有个关键的地方没有展开讲：就是服务的配置信息是怎么来的？以及代理对象是怎么得到的？

```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // 省略了其他代码

    	//获取服务配置
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
		//获得代理对象
        GenericService genericService = reference.get();

    }
```



接下来我们就来解决这个问题。先说结论：服务配置信息的来源是`soul-admin`同步到网关。

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
- `ApacheDubboMetaDataSubscriber`:`onSubscribe()`;
- `ApplicationConfigCache`:`initRef()`,`build()`;

在接下来的的测试中，`soul-admin`端和`soul-bootstrap`网关端之间的数据同步是通过`websocket`进行。所以在网关启动的时候会进行数据同步操作。



在`soul-bootstrap`网关端前面几个类主要处理同步的数据类型，最终流转到`ApplicationConfigCache`，在这里面进行构建服务`build()`。

```java
public ReferenceConfig<GenericService> build(final MetaData metaData) {
    //构建服务
    ReferenceConfig<GenericService> reference = new ReferenceConfig<>(); 
    //进行泛化调用   
    reference.setGeneric(true);
    //设置应用配置信息
    reference.setApplication(applicationConfig);
    //注册地址，比如zk  
    reference.setRegistry(registryConfig); 
    //接口信息名称
    reference.setInterface(metaData.getServiceName());
    //dubbo协议
    reference.setProtocol("dubbo");
    //其他信息
    String rpcExt = metaData.getRpcExt();
    DubboParamExtInfo dubboParamExtInfo = GsonUtils.getInstance().fromJson(rpcExt, DubboParamExtInfo.class);
        if (Objects.nonNull(dubboParamExtInfo)) {
            //版本
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getVersion())) {
                reference.setVersion(dubboParamExtInfo.getVersion()); 
            }
            //分组
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getGroup())) {
                reference.setGroup(dubboParamExtInfo.getGroup());
            }
            //负载均衡
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getLoadbalance())) {
                final String loadBalance = dubboParamExtInfo.getLoadbalance();
                reference.setLoadbalance(buildLoadBalanceName(loadBalance)); 
            }
            //url
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getUrl())) {
                reference.setUrl(dubboParamExtInfo.getUrl());
            }
            //超时和重试
            Optional.ofNullable(dubboParamExtInfo.getTimeout()).ifPresent(reference::setTimeout);
            Optional.ofNullable(dubboParamExtInfo.getRetries()).ifPresent(reference::setRetries);
        }
        try {
            //从注册中心中获得代理
            Object obj = reference.get(); 
            if (obj != null) {
                log.info("init apache dubbo reference success there meteData is :{}", metaData.toString());
                //放到cache缓存中，后面真正使用的时候就可从缓存中获取，而不是再去通过网络请求信息
                cache.put(metaData.getPath(), reference); 
            }
        } catch (Exception e) {
            log.error("init apache dubbo reference ex:{}", e.getMessage());
        }
        return reference;
    }
```



一个`reference`的信息如下：包括协议，服务接口，超时时间，负载均衡算法，重试次数。

```java
<dubbo:reference protocol="dubbo" prefix="dubbo.reference.org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService" uniqueServiceName="org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService" interface="org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService" generic="true" generic="true" sticky="false" loadbalance="random" retries="2" timeout="10000" id="org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService" valid="true" />
```



一个代理对象信息如下：接口信息，注册中心地址等等其他详细信息。

```xml
invoker :interface org.apache.dubbo.rpc.service.GenericService -> zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?anyhost=true&application=soul_proxy&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=true&interface=org.dromara.soul.examples.dubbo.api.service.DubboMultiParamService&loadbalance=random&methods=batchSave,findByArrayIdsAndName,findByListId,batchSaveAndNameAndId,findByIdsAndName,saveComplexBeanTestAndName,saveComplexBeanTest,findByStringArray&pid=23576&protocol=dubbo&register.ip=192.168.236.60&release=2.7.5&remote.application=test-dubbo-service&retries=2&side=consumer&sticky=false&timeout=10000&timestamp=1612011330405,directory: org.apache.dubbo.registry.integration.RegistryDirectory@406951e0
```



在同步过程中，将所有的元数据信息及注册中心的服务放到了`cache`中，在使用的时候，就会到这个`cache`中去拿。就是在文章开始的地方提到的泛化调用。

```java
//org.dromara.soul.plugin.apache.dubbo.proxy.ApacheDubboProxyService#genericInvoker
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // 省略了其他代码

    	//从cache中获取服务配置
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
		//获得代理对象，这个对象就是在同步过程中保存的代理对象
        GenericService genericService = reference.get();

    }
```



现在就清楚来了`Soul`网关中`Apache Dubbo`插件里面的服务是如何被加载的，包括服务的配置信息和代理对象的生成。



小结，本文通过实际的案例演示了`Soul`网关`Apache Dubbo`的支持，并且通过跟踪源码的方式明白了`Apache Dubbo`插件执行的原理以及服务注册和使用的过程。