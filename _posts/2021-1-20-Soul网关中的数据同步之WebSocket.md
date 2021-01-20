---
layout: post
title: Soul网关中的数据同步之WebSocket
tags: Soul
---

在前面几篇文章中我们体验了如何将自己的服务接入到`Soul`网关中，接下来几篇我们将要了解的是`Soul`是如何完成数据同步的，在官网中介绍了4种同步方式：基于`WebSocket`的数据同步，基于`ZoomKeeper`的数据同步，基于`Http长轮询`的数据同步和基于`Nacos`的数据同步。我们将依次进行分析，本篇文章分析的是基于`WebSocket`的数据同步。

数据同步的原理在官网已经有讲述了[数据同步原理](https://dromara.org/zh-cn/docs/soul/dataSync.html)：

> `Soul` 数据同步的流程，`Soul` 网关在启动时，会从从配置服务同步配置数据，并且支持推拉模式获取配置变更信息，并且更新本地缓存。而管理员在管理后台，变更用户、规则、插件、流量配置，通过推拉模式将变更信息同步给 `Soul` 网关，具体是 `push` 模式，还是 `pull` 模式取决于配置。

![1](https://midnight2104.github.io/img/2021-1-20/1.png)

> - 如果是 `websocket` 同步策略，则将变更后的数据主动推送给 `soul-web`，并且在网关层，会有对应的 `WebsocketDataHandler` 处理器处理来处 `admin` 的数据推送。



同步的核心逻辑是：在`soul-admin`后台修改数据，先保存到数据库；然后将修改的信息通过同步策略发送到`soul`网关；由网关处理后，保存在`soul`网关内存；使用时，从网关内存获取数据。

本文的分析是想通过跟踪源码的方式来理解同步的核心逻辑，数据同步分析步骤如下：

-   1.修改规则
- 2.更新数据
- 3.接受数据
- 4.使用更新后的数据



##### 1. 修改规则

我们以一个实际调用过程为例，比如在`Soul`网关管理系统中，对一项规则进行修改：将`divide`插件中的`/http/order/findById`规则的重试次数修改为`2`。具体信息如下所示：

![1](https://midnight2104.github.io/img/2021-1-20/2.png)

点击确认后，进入到`soul-admin`的`updateRule()`这个接口。

```java
    @PutMapping("/{id}")
    public SoulAdminResult updateRule(@PathVariable("id") final String id, @RequestBody final RuleDTO ruleDTO) {
        Objects.requireNonNull(ruleDTO);
        ruleDTO.setId(id);
        Integer updateCount = ruleService.createOrUpdate(ruleDTO);
        return SoulAdminResult.success(SoulResultMessage.UPDATE_SUCCESS, updateCount);
    }

```

##### 2.更新数据

进入到后端系统后，会现在数据中更新信息，然后通过`publishEvent()`方法将更新的信息同步到网关。（下面代码只是展示了主要的逻辑，完整的代码请参考`Soul`源码。）

```java
@Transactional(rollbackFor = Exception.class)
    public int createOrUpdate(final RuleDTO ruleDTO) {
        int ruleCount;
        RuleDO ruleDO = RuleDO.buildRuleDO(ruleDTO);
        List<RuleConditionDTO> ruleConditions = ruleDTO.getRuleConditions();
        if (StringUtils.isEmpty(ruleDTO.getId())) {
            //在数据库更新数据
            ruleCount = ruleMapper.insertSelective(ruleDO);
      		//省略了其他代码......
        } else {
            //在数据库更新数据
            ruleCount = ruleMapper.updateSelective(ruleDO);
           //省略了其他代码......
        }
        
        //将更新的数据同步到Soul网关
        publishEvent(ruleDO, ruleConditions);
        return ruleCount;
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



![1](https://midnight2104.github.io/img/2021-1-20/2.png)



当监听器监听到有事件发布后，会执行`onApplicationEvent()`方法，这里面的逻辑是循环处理`DataChangedListener`，通过`switch / case`表达式匹配修改的是什么类型信息，我们这里修改的是规则，所以会匹配到`listener.onRuleChanged()`这个方法。（这里虽然用了循环的方式处理每一个`listener`，但在实际中我们只需要一种数据同步方式就好。）

 本次使用的是`WebSocket`进行数据同步，所以`listener.onRuleChanged()`的实际执行方法是`WebsocketDataChangedListener中的onRuleChanged()`。这里面做的事情是：

- 1.将更新数据转成`WebsocketData`的形式；
- 2.通过`WebsocketCollector`发布数据（数据又转成了`json`）。

```java

public class WebsocketDataChangedListener implements DataChangedListener {
	//省略了其他代码......

    @Override
    public void onRuleChanged(final List<RuleData> ruleDataList, final DataEventTypeEnum eventType) {
        WebsocketData<RuleData> configData =
                new WebsocketData<>(ConfigGroupEnum.RULE.name(), eventType.name(), ruleDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(configData), eventType);
    }

  //省略了其他代码......

}
```

`WebsocketCollector`中的`send()`方法如下，核心逻辑就是`session.getBasicRemote().sendText(message);`。到这儿，`soul-admin`就通过`websocket`就更新规则的数据发布出去了，等待`WebSocketClient`去接收了。

```java
public static void send(final String message, final DataEventTypeEnum type) {
        if (StringUtils.isNotBlank(message)) {
		//省略了其他代码......
            for (Session session : SESSION_SET) {
                try {
                    session.getBasicRemote().sendText(message);
                } catch (IOException e) {
                    log.error("websocket send result is exception: ", e);
                }
            }
        }
    }
```



##### 3.接收数据

在`Soul`网关中，`SoulWebsocketClient`继承了`WebSocketClient`，所以它会处理`soul-admin`通过`WebSocket`发出的数据。处理的入口是在`onMessage()`方法中，它又调用了`handleResult()`方法，核心方法是`websocketDataHandler.executor()`。

```java
public final class SoulWebsocketClient extends WebSocketClient {
    	//省略了其他代码......
   
    @Override
    public void onMessage(final String result) {
        handleResult(result);
    }
    	//省略了其他代码......
    
    @SuppressWarnings("ALL")
    private void handleResult(final String result) {
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
}
```

`WebsocketDataHandler`可以看成是一个工厂类，里面定义好了处理信息的类型：插件，选择器，规则，认证，元数据。

```java

public class WebsocketDataHandler {

    private static final EnumMap<ConfigGroupEnum, DataHandler> ENUM_MAP = new EnumMap<>(ConfigGroupEnum.class);

    public WebsocketDataHandler(final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers,
                                final List<AuthDataSubscriber> authDataSubscribers) {
        ENUM_MAP.put(ConfigGroupEnum.PLUGIN, new PluginDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.SELECTOR, new SelectorDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.RULE, new RuleDataHandler(pluginDataSubscriber));
        ENUM_MAP.put(ConfigGroupEnum.APP_AUTH, new AuthDataHandler(authDataSubscribers));
        ENUM_MAP.put(ConfigGroupEnum.META_DATA, new MetaDataHandler(metaDataSubscribers));
    }

    public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
}
```

根据传入的数据类型，使用对应的`Handler`去处理，比如，我们修改的是规则信息，所以这里会调用`RuleDataHandler`来处理。跟踪进去后，发现`RuleDataHandler`继承了`AbstractDataHandler`类，其他几种数据类型也继承了该类。

![1](https://midnight2104.github.io/img/2021-1-20/2.png)

通过源码发现，这里运用了`模板方法`的设计模式。定义好了通用的方法`handle()`，其他方法都是抽象方法，由子类去实现。在`handle()`方法中通过`switch / case`表达式去匹配操作类型，然后执行实际的方法。

```java
public abstract class AbstractDataHandler<T> implements DataHandler {
   
    //抽象方法
    protected abstract void doUpdate(List<T> dataList);

    //这里省略了其他抽象方法

    //通用方法
    @Override
    public void handle(final String json, final String eventType) {
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList); //根据操作类型匹配实际的执行方法。
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
    }
}
```

在 `RuleDataHandler`中，更新方法的逻辑交给了`pluginDataSubscriber`的`onRuleSubscribe()`方法。而这是一个接口，它的实现类是`CommonPluginDataSubscriber`。

```java
public class RuleDataHandler extends AbstractDataHandler<RuleData> {

    private final PluginDataSubscriber pluginDataSubscriber;

    //省略了其他方法
       
    @Override
    protected void doUpdate(final List<RuleData> dataList) {
        dataList.forEach(pluginDataSubscriber::onRuleSubscribe);
    }

    
}
```

`CommonPluginDataSubscriber`在处理更新的信息，当前我们测试的是更新规则信息，所以会进入更新的逻辑，就是下面代码中的`BaseDataCache.getInstance().cacheRuleData(ruleData);`。

```java

public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    
   //省略其他代码
    
    //处理订阅信息
    @Override
    public void onRuleSubscribe(final RuleData ruleData) {
        subscribeDataHandler(ruleData, DataEventTypeEnum.UPDATE);
    }
     
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
					//省略其他代码
                }
            } else if (data instanceof SelectorData) {
            //省略其他代码
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    //更新通用缓存信息
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    //更新插件中的缓存信息
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } 
                //省略其他代码
                }
            }
        });
    }
}

```

在`BaseDataCache.getInstance().cacheRuleData(ruleData);`代码中，做的事情就是根据传入的变更信息来更新`RULE_MAP`。这个`RULE_MAP`缓存了规则信息，网关在后续使用时，也是从这里获取具体规则去匹配请求。

```java
public final class BaseDataCache {
    private static final ConcurrentMap<String, List<RuleData>> RULE_MAP = Maps.newConcurrentMap();
    
    //省略了其他代码......
    
    //缓存规则
    public void cacheRuleData(final RuleData ruleData) {
        Optional.ofNullable(ruleData).ifPresent(this::ruleAccept);
    }

    //接受规则
    private void ruleAccept(final RuleData data) {
                String selectorId = data.getSelectorId();
                if (RULE_MAP.containsKey(selectorId)) {
                    List<RuleData> existList = RULE_MAP.get(selectorId);
                    //删除原来的规则
                    final List<RuleData> resultList = existList.stream().filter(r -> !r.getId().equals(data.getId())).collect(Collectors.toList());
                    resultList.add(data);
                    //保存新的规则
                    final List<RuleData> collect = resultList.stream().sorted(Comparator.comparing(RuleData::getSort)).collect(Collectors.toList());
                    RULE_MAP.put(selectorId, collect);
                } else {
                    RULE_MAP.put(selectorId, Lists.newArrayList(data));
                }
        }
}

```



分析到这里，数据同步的工作就算完成了。核心逻辑就是就更新的信息放到网关的内存中，使用时再去内存中拿，所以`Soul`网关的效率是很高的。



##### 4. 使用更新后的数据

规则信息完成更新后，通过`http`去访问`soul`网关，这里以`divide`插件为例。关于`divide`插件的使用请参考之前的文章。

发起一个`GET`请求：`http://localhost:9195/http/order/findById?id=1`，代码会执行到下面这个位置：

```java
# org.dromara.soul.plugin.base.AbstractSoulPlugin#execute
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);

			//省略了其他代码
            
            //从缓存中获取规则信息
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            //省略了其他代码
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```

代码`BaseDataCache.getInstance().obtainRuleData(selectorData.getId());`就是我们在数据同步时操作的数据缓存类，`RULE_MAP`就是刚才更新的规则信息。

```java
public final class BaseDataCache {
    private static final ConcurrentMap<String, List<RuleData>> RULE_MAP = Maps.newConcurrentMap();
    
    //省略了其他代码......  
	public List<RuleData> obtainRuleData(final String selectorId) {
        return RULE_MAP.get(selectorId);
    }
}
```



最后，本文通过源码的方式跟踪了`Soul`网关是如何通过`WebSocket`完成数据同步的：数据修改后，通过`Spring`发布修改事件，由`WebSocket`发送数据，`SoulWebSocket`会处理数据，最后将数据保存到内存。