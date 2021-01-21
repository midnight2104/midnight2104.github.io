---
layout: post
title: Soul网关中的数据同步之ZooKeeper
tags: Soul
---



在上一篇文章中，跟踪了基于`WebSocket`的数据同步原理，本篇文件将要跟踪基于`ZoomKeeper`的数据同步原理。

> - 基于 zookeeper 的同步原理很简单，主要是依赖 `zookeeper` 的 watch 机制，`soul-web` 会监听配置的节点，`soul-admin` 在启动的时候，会将数据全量写入 `zookeeper`，后续数据发生变更时，会增量更新 `zookeeper` 的节点，与此同时，`soul-web` 会监听配置信息的节点，一旦有信息变更时，会更新本地缓存。

同步的核心逻辑是：在`soul-admin`后台修改数据，先保存到数据库；然后将修改的信息通过同步策略发送到`soul`网关；由网关处理后，保存在`soul`网关内存；使用时，从网关内存获取数据。

本文的分析是想通过跟踪源码的方式来理解同步的核心逻辑，数据同步分析步骤如下：

-   1.修改规则
- 2.更新数据
- 3.接受数据
- 4.使用更新后的数据



##### 1. 修改规则

在演示案例之前，配置一下`soul-admin`端的数据同步方式为`ZooKeeper`（`ZooKeeper`的启动和安装请参考前面的文章）:

```yaml
soul:
  database:
    dialect: mysql
    init_script: "META-INF/schema.sql"
    init_enable: true
  sync:
#    websocket:
#      enabled: true
      zookeeper:
          url: localhost:2181
          sessionTimeout: 5000
          connectionTimeout: 2000
```

在`soul-bootstrap`也配置一下数据同步方式为`ZooKeeper`:

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

        zookeeper:
             url: localhost:2181
             sessionTimeout: 5000
             connectionTimeout: 2000
```



现在，我们以一个实际调用过程为例，比如在`Soul`网关管理系统中，对选择器的配置信息进行修改：查询条件中`id=100`才能匹配成功。具体信息如下所示：

![1](https://midnight2104.github.io/img/2021-1-21/1.png)

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



>  注意一下，这个`DataChangedEventDispatcher`还实现了`InitializingBean`接口，并重写了它的`afterPropertiesSet()`方法，做的事情是：在`bean`初始化的时候，将实现`DataChangedListener`接口的`bean`加载进来。通过查看源码，可以看到4种数据同步的方式都实现了该接口，其中就有我们这次使用的`WebSocket`数据同步方式。



<img src="https://midnight2104.github.io/img/2021-1-20/3.png" alt="1" style="zoom:50%;" />



当监听器监听到有事件发布后，会执行`onApplicationEvent()`方法，这里面的逻辑是循环处理`DataChangedListener`，通过`switch / case`表达式匹配修改的是什么类型信息，我们这里修改的是选择器，所以会匹配到`listener.onSelectorChanged()`这个方法。（这里虽然用了循环的方式处理每一个`listener`，但在实际中我们只需要一种数据同步方式就好。）

 本次使用的是`ZooKeeper`进行数据同步，所以`listener.onSelectorChanged()`的实际执行方法是`ZookeeperDataChangedListener中的onSelectorChanged`。这里面做的事情是：

- 1.构建路径用于存放变更数据；
- 2.更新数据到`ZooKeeper`。

```java

public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        
        for (SelectorData data : changed) {
            //构建路径
            final String selectorRealPath = ZkPathConstants.buildSelectorRealPath(data.getPluginName(), data.getId());

            //省略了其他代码......
            
            final String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getPluginName());
			
            //更新数据到ZooKeeper
            upsertZkNode(selectorRealPath, data);
        }
    }
```

真正更新数据的操作是通过` zkClient.writeData()`完成。

```java

	//真正更新数据的操作是通过 zkClient.writeData()完成
    private void upsertZkNode(final String path, final Object data) {
        if (!zkClient.exists(path)) {
            zkClient.createPersistent(path, true);
        }
        zkClient.writeData(path, data);
    }
```



##### 3.接收数据

在`Soul`网关中，接受数据的操作也是通过`zkClient`进行订阅变更。通过` zkClient.subscribeDataChanges()`向`ZooKeeper`订阅变更的数据，然后去处理。

```java
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {
    	//省略了其他代码......
       
    private void subscribeSelectorDataChanges(final String path) {
        //订阅数据变更
        zkClient.subscribeDataChanges(path, new IZkDataListener() {
            @Override
            public void handleDataChange(final String dataPath, final Object data) {
                //处理数据
                cacheSelectorData((SelectorData) data);
            }

            @Override
            public void handleDataDeleted(final String dataPath) {
                unCacheSelectorData(dataPath);
            }
        });
    }
      
    //处理数据
    private void cacheSelectorData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData)
                .ifPresent(data -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSelectorSubscribe(data)));
    }
}
```

实际处理数据还是由`CommonPluginDataSubscriber`来处理。`CommonPluginDataSubscriber`在处理数据时，根据数据类型和操作类型来分别处理。当前我们测试的是更新选择器信息，所以会进入更新的逻辑，就是下面代码中的`BaseDataCache.getInstance().cacheRuleData(ruleData);`。

```java

public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    
   //省略了其他代码
     
   private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                     //省略了其他代码
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    //更新网关内存中缓存的信息
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                   //更新部分插件信息 Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } 
                }
            } else if (data instanceof RuleData) {
               //省略了其他代码
                }
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



分析到这里，基于`ZooKeeper`数据同步的工作就算完成了。核心逻辑就是就更新的信息放到网关的内存中，使用时再去内存中拿，所以`Soul`网关的效率是很高的。



##### 4. 使用更新后的数据

选择器信息完成更新后，通过`http`去访问`soul`网关，这里以`divide`插件为例。关于`divide`插件的使用请参考之前的文章。

发起一个`GET`请求：`http://localhost:9195/http/order/findById?id=1`，代码会执行到下面这个位置：

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

刚才，我们发起的请求：`http://localhost:9195/http/order/findById?id=1`，是匹配不到选择器的：

```sh
{
    "code": -107,
    "message": "Can not find selector, please check your configuration!",
    "data": null
}
```

因为，在开始的时候，更新了选择器的配置：查询条件中`id=100`才能匹配成功。

所以，我们另外再发起一个`id=100`请求：`http://localhost:9195/http/order/findById?id=100`，就可以成功了。

```sh

{
    "id": "100",
    "name": "hello world findById"
}
```



最后，本文通过源码的方式跟踪了`Soul`网关是如何通过`ZooKeeper`完成数据同步的：数据修改后，通过`Spring`发布修改事件，由`zkClient`发送数据。在网关层也有`zkClient`订阅变更的数据，然后进行处理数据，最后将数据保存到网关内存。