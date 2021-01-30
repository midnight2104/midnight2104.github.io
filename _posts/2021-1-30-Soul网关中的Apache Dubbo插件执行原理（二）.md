---
layout: post
title: Soul网关中的Apache Dubbo插件执行原理（二）
tags: Soul

---



 在上一篇文章中，我们通过跟踪源码的方式理解了`Apache Dubbo`插件的执行原理，将发起的`http`请求转化为`dubbo`的泛化调用，但是但是有个关键的地方没有展开讲：就是服务的配置信息是怎么来的？以及代理对象是怎么得到的？

```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // 省略了其他代码

    	//获取服务配置
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
		//获得代理对象
        GenericService genericService = reference.get();

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
- `ApacheDubboMetaDataSubscriber`:`onSubscribe()`;
- `ApplicationConfigCache`:`initRef()`,`build()`;

在本文的测试中，`soul-admin`端和`soul-bootstrap`网关端之间的数据同步是通过`websocket`进行。所以在网关启动的时候会进行数据同步操作。



在`soul-bootstrap`网关端前面几个类主要处理同步的数据类型，最终流转到`ApplicationConfigCache`，在这里面进行构建服务`build()`。

```java
public ReferenceConfig<GenericService> build(final MetaData metaData) {
        ReferenceConfig<GenericService> reference = new ReferenceConfig<>(); //构建服务
        reference.setGeneric(true);
        reference.setApplication(applicationConfig);
        reference.setRegistry(registryConfig); //注册地址，比如zk
        reference.setInterface(metaData.getServiceName());
        reference.setProtocol("dubbo"); //dubbo协议
        String rpcExt = metaData.getRpcExt();
        DubboParamExtInfo dubboParamExtInfo = GsonUtils.getInstance().fromJson(rpcExt, DubboParamExtInfo.class);
        if (Objects.nonNull(dubboParamExtInfo)) {
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getVersion())) {
                reference.setVersion(dubboParamExtInfo.getVersion()); //版本
            }
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getGroup())) {
                reference.setGroup(dubboParamExtInfo.getGroup());
            }
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getLoadbalance())) {
                final String loadBalance = dubboParamExtInfo.getLoadbalance();
                reference.setLoadbalance(buildLoadBalanceName(loadBalance)); //负载均衡
            }
            if (StringUtils.isNoneBlank(dubboParamExtInfo.getUrl())) {
                reference.setUrl(dubboParamExtInfo.getUrl());
            }
            Optional.ofNullable(dubboParamExtInfo.getTimeout()).ifPresent(reference::setTimeout);
            Optional.ofNullable(dubboParamExtInfo.getRetries()).ifPresent(reference::setRetries);
        }
        try {
            Object obj = reference.get(); //从注册中心中获得代理
            if (obj != null) {
                log.info("init apache dubbo reference success there meteData is :{}", metaData.toString());
                cache.put(metaData.getPath(), reference); //放到cache中
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



小结：本文再次梳理了`Soul`网关中`Apache Dubbo`插件里面的服务是如何被加载的，包括服务的配置信息和 代理对象的生成。