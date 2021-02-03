---
layout: post
title: Soul网关中的Sofa插件执行原理（二）
tags: Soul

---



 在上一篇文章中，我们通过跟踪源码的方式理解了`Sofa`插件的执行原理，将发起的`http`请求转化为`sofa`的泛化调用，但是有个关键的地方没有展开讲：就是服务的配置信息是怎么来的？以及代理对象是怎么得到的？

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



这篇文章，我们就来解决这个问题。先说结论：服务配置信息的来源是`soul-admin`同步到网关。

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



小结：本文再次梳理了`Soul`网关中`Sofa`插件里面的服务是如何被加载的，包括服务的配置信息和 代理对象的生成。