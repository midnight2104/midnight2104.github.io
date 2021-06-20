---
layout: post
title: Soul源码阅读系列（十一）基于Http长轮询的数据同步（一）
tags: Soul
---



在上一篇文章中，跟踪了基于`Nacos`的数据同步原理，本篇文章将要跟踪基于`Http长轮询`的数据同步原理。

> 如果是 `http` 同步策略，`soul-web` 主动发起长轮询请求，默认有 `90s` 超时时间，如果 `soul-admin` 没有数据变更，则会阻塞 `http` 请求，如果有数据发生变更则响应变更的数据信息，如果超过 60s 仍然没有数据变更则响应空数据，网关层接到响应后，继续发起`http`请求，反复同样的请求。



同步的核心逻辑是：在`soul-admin`后台修改数据，先保存到数据库，然后保存到`soul-admin`的内存；在网关有定时任务执行，即发起长轮询，发起`http`请求到`soul-admin`去获取变更的数据。

本文的分析是想通过跟踪源码的方式来理解同步的核心逻辑，数据同步分析步骤如下：

-   1.修改选择器
- 2.更新数据
- 3.接收数据
- 4.使用更新后的数据



##### 1. 修改选择器

在演示案例之前，将`soul-admin`的数据同步方式配置为`http`:

```yaml
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
    init_enable: true
  sync:
#    websocket:
#      enabled: true
#      zookeeper:
#          url: localhost:2181
#          sessionTimeout: 5000
#          connectionTimeout: 2000
      http:
        enabled: true
```

在`soul-bootstrap`也配置一下数据同步方式为`http`:

```yaml
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
#        websocket :
#             urls: ws://localhost:9095/websocket

#        zookeeper:
#             url: localhost:2181
#             sessionTimeout: 5000
#             connectionTimeout: 2000
        http:
             url : http://localhost:9095
```



现在，我们以一个实际调用过程为例，比如在`Soul`网关管理系统中，对选择器的配置信息进行修改：查询条件中`id=99`才能匹配成功。具体信息如下所示：

![](https://midnight2104.github.io/img/2021-1-23/1.png)

点击确认后，进入到`soul-admin`的`updateSelector()`这个接口。

```java
    @PutMapping("/{id}")
    public SoulAdminResult updateSelector(@PathVariable("id") final String id, @RequestBody final SelectorDTO selectorDTO) {
        Objects.requireNonNull(selectorDTO);
        selectorDTO.setId(id);
        Integer updateCount = selectorService.createOrUpdate(selectorDTO);
        return SoulAdminResult.success(SoulResultMessage.UPDATE_SUCCESS, updateCount);
    }

```

##### 2.更新数据

进入到后端系统后，会现在数据中更新信息，然后通过`publishEvent()`方法将更新的信息同步到网关。（下面代码只是展示了主要的逻辑，完整的代码请参考`Soul`源码。）

```java
 @Transactional(rollbackFor = RuntimeException.class)
    public int createOrUpdate(final SelectorDTO selectorDTO) {
        int selectorCount;
        SelectorDO selectorDO = SelectorDO.buildSelectorDO(selectorDTO);
		//将更新的数据保存到soul-admin的数据库
        
        //省略了其他代码
        
        //将更新的数据同步到网关
        publishEvent(selectorDO, selectorConditionDTOs);
        return selectorCount;
    }

```

在`publishEvent()`方法中调用了`eventPublisher.publishEvent()`，这个`eventPublisher`对象是一个`ApplicationEventPublisher`类，这个类的全限定名是`org.springframework.context.ApplicationEventPublisher`。看到这儿，我们知道了发布数据是通过`Spring`相关的功能来完成的。

```java
    private void publishEvent(final RuleDO ruleDO, final List<RuleConditionDTO> ruleConditions) {
		//省略了其他代码......
        // publish change event.
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, DataEventTypeEnum.UPDATE,
                Collections.singletonList(RuleDO.transFrom(ruleDO, pluginDO.getName(), conditionDataList))));
    }
```

`Spring`完成事件发布后，肯定有对应的监听器来处理它，这个监听器是`ApplicationListener`接口。在`Soul`中是通过`DataChangedEventDispatcher`这个类来完成具体监听工作的，它实现了`ApplicationListener`接口。

```java
//处理监听事件
@Component
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {
	//省略了其他代码......
    
    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH: //认证授权
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN: //修改了插件
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:  //修改了规则
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR: //修改了选择器
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA: //修改了元数据
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }

    //在bean初始化的时候，将实现DataChangedListener接口的bean加载进来。
    @Override
    public void afterPropertiesSet() {
        Collection<DataChangedListener> listenerBeans = applicationContext.getBeansOfType(DataChangedListener.class).values();
        this.listeners = Collections.unmodifiableList(new ArrayList<>(listenerBeans));
    }

}
```



>  注意一下，这个`DataChangedEventDispatcher`还实现了`InitializingBean`接口，并重写了它的`afterPropertiesSet()`方法，做的事情是：在`bean`初始化的时候，将实现`DataChangedListener`接口的`bean`加载进来。通过查看源码，可以看到4种数据同步的方式都实现了该接口，其中就有我们这次使用的`Nacos`数据同步方式。



<img src="https://midnight2104.github.io/img/2021-1-20/3.png" alt="1" style="zoom:50%;" />



当监听器监听到有事件发布后，会执行`onApplicationEvent()`方法，这里面的逻辑是循环处理`DataChangedListener`，通过`switch / case`表达式匹配修改的是什么类型信息，我们这里修改的是选择器，所以会匹配到`listener.onSelectorChanged()`这个方法。（这里虽然用了循环的方式处理每一个`listener`，但在实际中我们只需要一种数据同步方式就好。）

 本次使用的是`http长轮询`进行数据同步，所以`listener.onSelectorChanged()`的实际执行方法是`HttpLongPollingDataChangedListener#onSelectorChanged`，它继承了`AbstractDataChangedListener`。这里面做的事情是：

- 1.更新选择器信息到缓存；
- 2.设置响应。

```java

    public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        if (CollectionUtils.isEmpty(changed)) {
            return;
        }
        //更新选择器信息到缓存
        this.updateSelectorCache();
        //设置响应
        this.afterSelectorChanged(changed, eventType);
    }

```

真正更新数据的操作是通过`updateCache`完成，将新的数据放到`CACHE`中，这个`CACHE`是`ConcurrentMap`类型。网关有定时任务来这个`CACHE`里获取数据。

```java
 	 //更新选择器信息到缓存
    protected void updateSelectorCache() {
        this.updateCache(ConfigGroupEnum.SELECTOR, selectorService.listAll());
    }

    protected <T> void updateCache(final ConfigGroupEnum group, final List<T> data) {
        String json = GsonUtils.getInstance().toJson(data);
        ConfigDataCache newVal = new ConfigDataCache(group.name(), json, Md5Utils.md5(json), System.currentTimeMillis());
        //更新新的数据
        ConfigDataCache oldVal = CACHE.put(newVal.getGroup(), newVal);
        log.info("update config cache[{}], old: {}, updated: {}", group, oldVal, newVal);
    }
```

设置响应的过程是在定时任务中完成的。

```java
//scheduler 是个定时任务  
@Override
    protected void afterSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.SELECTOR));
    }

//定时任务  
    class DataChangeTask implements Runnable {
		//省略其他代码

        @Override
        public void run() {
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
                //省略其他代码
            }
        }
    }

	//发送响应
    void sendResponse(final List<ConfigGroupEnum> changedGroups) {
			//省略其他代码
            generateResponse((HttpServletResponse) asyncContext.getResponse(), changedGroups);
            asyncContext.complete();
        }

	//产生响应
    private void generateResponse(final HttpServletResponse response, final List<ConfigGroupEnum> changedGroups) {
        try {
            response.setHeader("Pragma", "no-cache");
            response.setDateHeader("Expires", 0);
            response.setHeader("Cache-Control", "no-cache,no-store");
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setStatus(HttpServletResponse.SC_OK);
            response.getWriter().println(GsonUtils.getInstance().toJson(SoulAdminResult.success(SoulResultMessage.SUCCESS, changedGroups)));
        } catch (IOException ex) {
            log.error("Sending response failed.", ex);
        }
    }
```



##### 3.接收数据

在`Soul`网关中，接收数据的操作是主动通过`http长轮询`发起`http`请求到`soul-admin`获取数据。处理逻辑在`org.dromara.soul.sync.data.http.HttpSyncDataService`类中，`Soul`网关启动时就会执行。

```java
private void start() {
        // It could be initialized multiple times, so you need to control that.
        if (RUNNING.compareAndSet(false, true)) {
            // fetch all group configs.
            this.fetchGroupConfig(ConfigGroupEnum.values());
            int threadSize = serverList.size();
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // 发起长轮询
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
        } else {
            log.info("soul http long polling was started, executor=[{}]", executor);
        }
    }
```

`http长轮询`的任务是：不停的进行轮询，先向`soul-admin`发起请求查看是否有配置信息（包括插件，选择器，规则和元数据）变更；如果有配置信息变更，再发起请求获取变更的数据；最后更新网关的缓存数据。

```java
如果有配置信息变更class HttpLongPollingTask implements Runnable {
		//省略其他代码
        @Override
        public void run() {
            while (RUNNING.get()) {
                for (int time = 1; time <= retryTimes; time++) {
                    try {
                        doLongPolling(server);
                    } catch (Exception e) {
                   //省略其他代码
                    }
                }
            }
            log.warn("Stop http long polling.");
        }
    }

private void doLongPolling(final String server) {
    //省略其他代码
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>(8);
        for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
		//先发起请求，查看是否有配置信息变更
        String listenerUrl = server + "/configs/listener";
        JsonArray groupJson = null;
        try {
            String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
        } catch (RestClientException e) {
//省略其他代码
        }
         //如果有配置信息变更
        if (groupJson != null) {
            if (ArrayUtils.isNotEmpty(changedGroups)) {
                log.info("Group config changed: {}", Arrays.toString(changedGroups));
                //获取变更的配置信息
                this.doFetchGroupConfig(server, changedGroups);
            }
        }
    }
    
    private void doFetchGroupConfig(final String server, final ConfigGroupEnum... groups) {
        //省略其他代码
        
        //再发起请求，获取变更的配置信息
        String url = server + "/configs/fetch?" + StringUtils.removeEnd(params.toString(), "&");

        try {
            json = this.httpClient.getForObject(url, String.class);
        } catch (RestClientException e) {
//省略其他代码
        }
        // 使用获取的配置信息更新缓存
        boolean updated = this.updateCacheWithJson(json);
        if (updated) {
            log.info("get latest configs: [{}]", json);
            return;
        }
    }

```



在代码中`this.updateCacheWithJson(json)`，使用获取的配置信息更新缓存的处理操作，实际还是由`CommonPluginDataSubscriber`来处理。`CommonPluginDataSubscriber`在处理数据时，根据数据类型和操作类型来分别处理。当前我们测试的是更新选择器信息，所以会进入更新的逻辑，就是下面代码中的`BaseDataCache.getInstance().cacheRuleData(ruleData);`。

```java

public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    
   //省略了其他代码
     
   private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
  				//省略处理插件的逻辑
            } else if (data instanceof SelectorData) { //处理选择器信息
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) { //更新操作
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) { //删除操作
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                //省略处理规则的逻辑
            }
        });
    }
}

```

在`BaseDataCache.getInstance().cacheSelectData(selectorData);`代码中，做的事情就是根据传入的变更信息来更新`SELECTOR_MAP`。这个`SELECTOR_MAP`缓存了选择器信息，网关在后续使用时，也是从这里获取具体的选择器去匹配请求。

```java
public final class BaseDataCache {
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    //省略了其他代码......
    
    //缓存选择器
    public void cacheSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(this::selectorAccept);
    }

    //接受选择器
    private void selectorAccept(final SelectorData data) {
        String key = data.getPluginName();
        if (SELECTOR_MAP.containsKey(key)) {
            List<SelectorData> existList = SELECTOR_MAP.get(key);
            //删除之前的选择器
            final List<SelectorData> resultList = existList.stream().filter(r -> !r.getId().equals(data.getId())).collect(Collectors.toList());
            resultList.add(data);
            //保存现在的选择器
            final List<SelectorData> collect = resultList.stream().sorted(Comparator.comparing(SelectorData::getSort)).collect(Collectors.toList());
            SELECTOR_MAP.put(key, collect);
        } else {
            SELECTOR_MAP.put(key, Lists.newArrayList(data));
        }
    }
}

```



分析到这里，基于`http长轮询`数据同步的工作就算完成了。核心逻辑是：网关主动请求`soul-admin`获取变更的配置信息，将变更的信息放到网关的内存中，使用时再去内存中拿，所以`Soul`网关的效率是很高的。



##### 4. 使用更新后的数据

选择器信息完成更新后，通过`http`去访问`soul`网关，这里以`divide`插件为例。关于`divide`插件的使用请参考之前的文章。

发起一个`GET`请求：`http://localhost:9195/http/order/findById?id=100`，代码会执行到下面这个位置：

```java
# org.dromara.soul.plugin.base.AbstractSoulPlugin#execute
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            //获取选择器信息
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            
            //省略了其他代码
            
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```

代码`BaseDataCache.getInstance().obtainSelectorData(pluginName);`就是我们在数据同步时操作的数据缓存类，`RULE_MAP`就是刚才更新的规则信息。

```java
public final class BaseDataCache {
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    //省略了其他代码......  
    public List<SelectorData> obtainSelectorData(final String pluginName) {
        return SELECTOR_MAP.get(pluginName);
    }
}
```

刚才，我们发起的请求：`http://localhost:9195/http/order/findById?id=100`，是匹配不到选择器的：

```sh
{
    "code": -107,
    "message": "Can not find selector, please check your configuration!",
    "data": null
}
```

因为，在开始的时候，更新了选择器的配置：查询条件中`id=99`才能匹配成功。

所以，我们另外再发起一个`id=99`请求：`http://localhost:9195/http/order/findById?id=99`，就可以成功了。

```sh

{
    "id": "99",
    "name": "hello world findById"
}
```



最后，本文通过源码的方式跟踪了`Soul`网关是如何通过`http长轮询`完成数据同步的：数据修改后，先保存到 `soul-admin`的内存，然后通过`Soul`网关主动向`soul-admin`发起`http`请求获取配置信息，然后进行处理数据，最后将数据保存到网关内存。