---
layout: post
title: Soul网关中的Http服务探活
tags: Soul

---



服务探活机制是为了发现系统中上下游服务的的状态。当有新的服务注册时要通知其他系统，当有服务下线时也要告知其他系统。

`Soul`网关中有对`Http`服务处理的探活，所有的服务对象保存在`soul-admin`的`UPSTREAM_MAP`中，这里面的服务对象有两个来源，一个来自于原有的数据库，一个来自于其他服务的注册。

```java
@Component
public class UpstreamCheckService {
	//保存上游服务
    private static final Map<String, List<DivideUpstream>> UPSTREAM_MAP = Maps.newConcurrentMap();
   
    //省略了其他代码
}
```

在`UpstreamCheckService`类中，在构造器执行完后，会执行`setup()`方法。里面做了两件事情：

- 1.读取数据库的服务信息，保存到`UPSTREAM_MAP`；
- 2.开启定时任务，检查每个方法。

```java
    //在构造器只执行完，执行这个方法
	@PostConstruct
    public void setup() {
        PluginDO pluginDO = pluginMapper.selectByName(PluginEnum.DIVIDE.getName());
        if (pluginDO != null) {
		  //省略了其他代码
            //来源数据库的服务对象
                if (CollectionUtils.isNotEmpty(divideUpstreams)) {
                    UPSTREAM_MAP.put(selectorDO.getName(), divideUpstreams);
                }
            }
        }
        //是否开启探活机制，默认开启
        if (check) {
            //定时任务，每10秒执行一次
            new ScheduledThreadPoolExecutor(Runtime.getRuntime().availableProcessors(), SoulThreadFactory.create("scheduled-upstream-task", false))
                    .scheduleWithFixedDelay(this::scheduled, 10, scheduledTime, TimeUnit.SECONDS);
        }
    }
```



定时任务每10秒就去检查服务的状态，并且更新服务信息。

```java
 private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            //检查每个选择器
            UPSTREAM_MAP.forEach(this::check);
        }
    }

    private void check(final String selectorName, final List<DivideUpstream> upstreamList) {
         //对每个服务进行检查
        for (DivideUpstream divideUpstream : upstreamList) {
            //检查服务
            final boolean pass = UpstreamCheckUtils.checkUrl(divideUpstream.getUpstreamUrl());
            if (pass) {
			//	成功
            } else {
                //失败
                divideUpstream.setStatus(false);
                log.error("check the url={} is fail ", divideUpstream.getUpstreamUrl());
            }
        }
        //都存活
        if (successList.size() == upstreamList.size()) {
            return;
        }
        //部分存活，只保留存活的，去除失活的服务
        if (successList.size() > 0) {
            UPSTREAM_MAP.put(selectorName, successList);
            updateSelectorHandler(selectorName, successList);
        } else { //没有存活的
            UPSTREAM_MAP.remove(selectorName);
            updateSelectorHandler(selectorName, null);
        }
    }
```

检查过程是通过`socket`进行连接，能够连接成功，则服务是好的，否则就认为服务连接失败。

```java
    private static boolean isHostConnector(final String host, final int port) {
        try (Socket socket = new Socket()) {
            //建立socket连接
            socket.connect(new InetSocketAddress(host, port));
        } catch (IOException e) {
            return false;
        }
        return true;
    }
```

有服务失活的时候，要发送更新事件给网关。

```java
    private void updateSelectorHandler(final String selectorName, final List<DivideUpstream> upstreams) {
        SelectorDO selectorDO = selectorMapper.selectByName(selectorName);
        if (Objects.nonNull(selectorDO)) {
			//省略了其他代码
                // publish change event.
                eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE, Collections.singletonList(selectorData)));
            }
        }
    }
```



当有新的服务启动时，`SpringMvcClientBeanPostProcessor`会处理接口信息，向`soul-admin`发起注册请求`"/soul-client/springmvc-register"`。当`soul-admin`端接收到这个请求时，会做三件事情：

- 1.更新数据库信息；
- 2.提交服务信息到`UPSTREAM_MAP`；
- 3.发布事件到网关。

```java
//接收  /soul-client/springmvc-register 请求的实现接口
public String registerSpringMvc(final SpringMvcRegisterDTO dto) {
        if (dto.isRegisterMetaData()) {
            MetaDataDO exist = metaDataMapper.findByPath(dto.getPath());
            if (Objects.isNull(exist)) {
                saveSpringMvcMetaData(dto);
            }
        }
        //处理请求
        String selectorId = handlerSpringMvcSelector(dto);
        handlerSpringMvcRule(selectorId, dto);
        return SoulResultMessage.SUCCESS;
   }

private String handlerSpringMvcSelector(final SpringMvcRegisterDTO dto) {
	//省略了其他代码
    
    // update db 更新数据库信息
    selectorMapper.updateSelective(selectorDO);
    // submit upstreamCheck 提交服务信息到UPSTREAM_MAP
    upstreamCheckService.submit(contextPath, addDivideUpstream);
    // publish change event. 发布事件到网关
    eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE,Collections.singletonList(selectorData)));

    }
```



上面的服务探活是在应用服务和`soul-admin`之间。其实，在网关也有一个`UPSTREAM_MAP`来保存服务信息。它里面的服务来源是`soul-admin`发布事件来通知网关的，网关会对信息进行处理，更新可用服务。

```java
public final class UpstreamCacheManager {
		//保存服务信息
    private static final Map<String, List<DivideUpstream>> UPSTREAM_MAP = Maps.newConcurrentMap();
}
```

在网关这一边，服务信息处理流程是（假设数据同步采用的是`websocket`）：

- `SoulWebsocketClient`: 后台 `wesocket` 信息在这里被监听，并发送给 `WebsocketDataHandler` 处理；
- `WebsocketDataHandler`: 根据事件类型, 选择对应处理器 (`PluginDataHandler`、`RuleDataHandler`等)；
- `AbstractDataHandler`: 根据事件变动类型(`refresh`、`update`等)， 调用处理器对应方法, 具体实现类会调用到 `CommonPluginDataSubscriber` 订阅器；
- `CommonPluginDataSubscriber`: 这里存有所有注册为 `Bean` 的事件处理器，处理主要逻辑在`subscribeDataHandler()`方法中；
- `DividePluginDataHandler`: 更新或移除缓存管理器中服务节点信息。



另外，在网关的`UpstreamCacheManager`中，也有一个每隔30秒的定时任务`scheduled`去检查服务的状态信息，但是这个默认是关闭的。

```java
public final class UpstreamCacheManager {   
    
    //省略了其他代码
    
    private UpstreamCacheManager() {
            boolean check = Boolean.parseBoolean(System.getProperty("soul.upstream.check", "false"));
            if (check) {
                new ScheduledThreadPoolExecutor(1, SoulThreadFactory.create("scheduled-upstream-task", false))
                        .scheduleWithFixedDelay(this::scheduled,
                                30, Integer.parseInt(System.getProperty("soul.upstream.scheduledTime", "30")), TimeUnit.SECONDS);
            }
        }
}
```



小结，本篇文章介绍了在`Soul`中`soul-admin`和`soul-bootstrap`网关对服务的探活机制，主要实现类分别是`UpstreamCheckService`和`UpstreamCacheManager`。

