---
layout: post
title: Soul网关中的Apache Dubbo插件执行原理（一）
tags: Soul

---



 在之前的文章中，我们体验过了`Apache Dubbo`插件执行流程，本篇文章是通过跟踪源码的方式来理解其中的执行原理。

在`Soul`网关中，`Apache Dubbo`插件负责将`http`协议转换成`dubbo`协议，设计到的插件有：`BodyParamPlugin`，`ApacheDubboPlugin`和`DubboResponsePlugin`。

- `BodyParamPlugin`：负责将请求的`json`放到`exchange`属性中；
- `ApacheDubboPlugin`：使用`Apache Dubbo`进行请求的泛化调用并返回响应结果；
- `DubboResponsePlugin`：包装响应结果。



#### `BodyParamPlugin`插件

该插件在执行链路中是先执行的，负责处理请求类型，比如带有参数的`application/json`，不带参数的查询请求。

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