---
layout: post
title: Soul网关中的Sofa插件执行原理（一）
tags: Soul

---



 在之前的文章中，我们体验过了`Sofa`插件执行流程，本篇文章是通过跟踪源码的方式来理解其中的执行原理。

在`Soul`网关中，`Sofa`插件负责将`http`协议转换成`sofa`协议，涉及到的插件有：`BodyParamPlugin`，`SofaPlugin`和`SofaResponsePlugin`。

- `BodyParamPlugin`：负责将请求的`json`放到`exchange`属性中；
- `SofaPlugin`：使用`Sofa`进行请求的泛化调用并返回响应结果；
- `SofaResponsePlugin`：包装响应结果。



#### `BodyParamPlugin`插件

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



#### `SofaPlugin`插件

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



#### `SofaResponsePlugin`插件

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

