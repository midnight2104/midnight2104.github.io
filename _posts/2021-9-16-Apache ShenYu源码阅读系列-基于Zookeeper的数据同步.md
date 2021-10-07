---
layout: post
title: Apache ShenYu源码阅读系列-基于ZooKeeper的数据同步
tags: ShenYu
---

> [Apache ShenYu](https://shenyu.apache.org/zh/docs/index) 是一个异步的，高性能的，跨语言的，响应式的 `API` 网关。

在`ShenYu`网关中，数据同步是指，当在后台管理系统中，数据发送了更新后，如何将更新的数据同步到网关中。`Apache ShenYu` 网关当前支持`ZooKeeper`、`WebSocket`、`Http长轮询`、`Nacos` 、`Etcd` 和 `Consul` 进行数据同步。本文的主要内容是基于`ZooKeeper`的数据同步源码分析。

> 本文基于`shenyu-2.4.0`版本进行源码分析，官网的介绍请参考 [数据同步原理](https://shenyu.apache.org/zh/docs/design/data-sync) 。



### 1. 关于ZooKeeper

[`Apache ZooKeeper`](https://zh.wikipedia.org/wiki/Apache_ZooKeeper)是`Apache`软件基金会的一个软件项目，它为大型分布式计算提供开源的分布式配置服务、同步服务和命名注册。`ZooKeeper`节点将它们的数据存储于一个分层的名字空间，非常类似于一个文件系统或一个前缀树结构。客户端可以在节点读写，从而以这种方式拥有一个共享的配置服务。



### 2. Admin数据同步

我们从一个实际案例进行源码追踪，比如在后台管理系统中，对`Divide`插件中的一条选择器数据进行更新，将权重更新为90：

![](https://qiniu.midnight2104.com/20210916/update-selector-zh.png)

#### 2.1 接收数据

- SelectorController.createSelector()

进入`SelectorController`类中的`updateSelector()`方法，它负责数据的校验，添加或更新数据，返回结果信息。

```java
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/selector")
public class SelectorController {
    
    @PutMapping("/{id}")
    public ShenyuAdminResult updateSelector(@PathVariable("id") final String id, @Valid @RequestBody final SelectorDTO selectorDTO) {
        // 设置当前选择器数据id
        selectorDTO.setId(id);
        // 创建或更新操作
        Integer updateCount = selectorService.createOrUpdate(selectorDTO);
        // 返回结果信息
        return ShenyuAdminResult.success(ShenyuResultMessage.UPDATE_SUCCESS, updateCount);
    }
    
    // ......
}
```



#### 2.2 处理数据

- SelectorServiceImpl.createOrUpdate()

在`SelectorServiceImpl`类中通过`createOrUpdate()`方法完成数据的转换，保存到数据库，发布事件，更新`upstream`。

```java
@RequiredArgsConstructor
@Service
public class SelectorServiceImpl implements SelectorService {
    // 负责事件发布的eventPublisher
    private final ApplicationEventPublisher eventPublisher;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public int createOrUpdate(final SelectorDTO selectorDTO) {
        int selectorCount;
        // 构建数据 DTO --> DO
        SelectorDO selectorDO = SelectorDO.buildSelectorDO(selectorDTO);
        List<SelectorConditionDTO> selectorConditionDTOs = selectorDTO.getSelectorConditions();
        // 判断是添加还是更新
        if (StringUtils.isEmpty(selectorDTO.getId())) {
            // 插入选择器数据
            selectorCount = selectorMapper.insertSelective(selectorDO);
            // 插入选择器中的条件数据
            selectorConditionDTOs.forEach(selectorConditionDTO -> {
                selectorConditionDTO.setSelectorId(selectorDO.getId());
                selectorConditionMapper.insertSelective(SelectorConditionDO.buildSelectorConditionDO(selectorConditionDTO));
            });
            // check selector add
            // 权限检查
            if (dataPermissionMapper.listByUserId(JwtUtils.getUserInfo().getUserId()).size() > 0) {
                DataPermissionDTO dataPermissionDTO = new DataPermissionDTO();
                dataPermissionDTO.setUserId(JwtUtils.getUserInfo().getUserId());
                dataPermissionDTO.setDataId(selectorDO.getId());
                dataPermissionDTO.setDataType(AdminConstants.SELECTOR_DATA_TYPE);
                dataPermissionMapper.insertSelective(DataPermissionDO.buildPermissionDO(dataPermissionDTO));
            }

        } else {
            // 更新数据，先删除再新增
            selectorCount = selectorMapper.updateSelective(selectorDO);
            //delete rule condition then add
            selectorConditionMapper.deleteByQuery(new SelectorConditionQuery(selectorDO.getId()));
            selectorConditionDTOs.forEach(selectorConditionDTO -> {
                selectorConditionDTO.setSelectorId(selectorDO.getId());
                SelectorConditionDO selectorConditionDO = SelectorConditionDO.buildSelectorConditionDO(selectorConditionDTO);
                selectorConditionMapper.insertSelective(selectorConditionDO);
            });
        }
        // 发布事件
        publishEvent(selectorDO, selectorConditionDTOs);

        // 更新upstream
        updateDivideUpstream(selectorDO);
        return selectorCount;
    }
    
    
    // ......
    
}

```

在`Serrvice`类完成数据的持久化操作，即保存数据到数据库，这个比较简单，就不深入追踪了。关于更新`upstream`操作，放到后面对应的章节中进行分析，重点关注发布事件的操作，它会执行数据同步。



`publishEvent()`方法的逻辑是：找到选择器对应的插件，构建条件数据，发布变更数据。

```java
       private void publishEvent(final SelectorDO selectorDO, final List<SelectorConditionDTO> selectorConditionDTOs) {
        // 找到选择器对应的插件
        PluginDO pluginDO = pluginMapper.selectById(selectorDO.getPluginId());
        // 构建条件数据
        List<ConditionData> conditionDataList =                selectorConditionDTOs.stream().map(ConditionTransfer.INSTANCE::mapToSelectorDTO).collect(Collectors.toList());
        // 发布变更数据
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE,
                Collections.singletonList(SelectorDO.transFrom(selectorDO, pluginDO.getName(), conditionDataList))));
    }
```

发布变更数据通过`eventPublisher.publishEvent()`完成，这个`eventPublisher`对象是一个`ApplicationEventPublisher`类，这个类的全限定名是`org.springframework.context.ApplicationEventPublisher`。看到这儿，我们知道了发布数据是通过`Spring`相关的功能来完成的。

> 关于`ApplicationEventPublisher`：
>
> 当有状态发生变化时，发布者调用 `ApplicationEventPublisher` 的 `publishEvent` 方法发布一个事件，`Spring `容器广播事件给所有观察者，调用观察者的 `onApplicationEvent` 方法把事件对象传递给观察者。调用 `publishEvent `方法有两种途径，一种是实现接口由容器注入 `ApplicationEventPublisher` 对象然后调用其方法，另一种是直接调用容器的方法，两种方法发布事件没有太大区别。
>
> - `ApplicationEventPublisher`：发布事件；
> - `ApplicationEvent`：`Spring` 事件，记录事件源、时间和数据；
> - `ApplicationListener`：事件监听者，观察者；



在`Spring`的事件发布机制中，有三个对象，

一个是发布事件的`ApplicationEventPublisher`，在`ShenYu`中通过构造器注入了一个`eventPublisher`。

另一个对象是`ApplicationEvent`，在`ShenYu`中通过`DataChangedEvent`继承了它，表示事件对象。

```java
public class DataChangedEvent extends ApplicationEvent {
//......
}
```

最后一个是 `ApplicationListener`，在`ShenYu`中通过`DataChangedEventDispatcher`类实现了该接口，作为事件的监听者，负责处理事件对象。

```java
@Component
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {

    //......
    
}
```



#### 2.3 分发数据

- DataChangedEventDispatcher.onApplicationEvent()

当事件发布完成后，会自动进入到`DataChangedEventDispatcher`类中的`onApplicationEvent()`方法，进行事件处理。

```java
@Component
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {

  /**
     * 有数据变更时，调用此方法
     * @param event
     */
    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        // 遍历数据变更监听器(一般使用一种数据同步的方式就好了)
        for (DataChangedListener listener : listeners) {
            // 哪种数据发生变更
            switch (event.getGroupKey()) {
                case APP_AUTH: // 认证信息
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN:  // 插件信息
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:    // 规则信息
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR:   // 选择器信息
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA:  // 元数据
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:  // 其他类型，抛出异常
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }
    
}
```

当有数据变更时，调用`onApplicationEvent`方法，然后遍历所有数据变更监听器，判断是哪种数据类型，交给相应的数据监听器进行处理。

`ShenYu`将所有数据进行了分组，一共是五种：认证信息、插件信息、规则信息、选择器信息和元数据。

这里的数据变更监听器（`DataChangedListener`），就是数据同步策略的抽象，它的具体实现有：

![](https://qiniu.midnight2104.com/20210913/data-changed-listener.png)

这几个实现类就是当前`ShenYu`支持的同步策略：

- `WebsocketDataChangedListener`：基于`websocket`的数据同步；
- `ZookeeperDataChangedListener`：基于`zookeeper`的数据同步；
- `ConsulDataChangedListener`：基于`consul`的数据同步；
- `EtcdDataDataChangedListener`：基于`etcd`的数据同步；
- `HttpLongPollingDataChangedListener`：基于`http长轮询`的数据同步；
- `NacosDataChangedListener`：基于`nacos`的数据同步；

既然有这么多种实现策略，那么如何确定使用哪一种呢？

因为本文是基于`Zookeeper`的数据同步源码分析，所以这里以`ZookeeperDataChangedListener`为例，分析它是如何被加载并实现的。

通过在源码工程中进行全局搜索，可以看到，它的实现是在`DataSyncConfiguration`类完成的。

```java
/**
 * 数据同步配置类
 * 通过springboot条件装配实现
 * The type Data sync configuration.
 */
@Configuration
public class DataSyncConfiguration {
    
    
    /**
     * zookeeper数据同步
     * The type Zookeeper listener.
     */
    @Configuration
    @ConditionalOnProperty(prefix = "shenyu.sync.zookeeper", name = "url")  // 条件属性，满足才会被加载
    @Import(ZookeeperConfiguration.class)
    static class ZookeeperListener {

        /**
         * Config event listener data changed listener.
         * 创建Zookeeper数据变更监听器
         * @param zkClient the zk client
         * @return the data changed listener
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataChangedListener.class)
        public DataChangedListener zookeeperDataChangedListener(final ZkClient zkClient) {
            return new ZookeeperDataChangedListener(zkClient);
        }

        /**
         * Zookeeper data init zookeeper data init.
         *  创建 Zookeeper 数据初始化类
         * @param zkClient        the zk client
         * @param syncDataService the sync data service
         * @return the zookeeper data init
         */
        @Bean
        @ConditionalOnMissingBean(ZookeeperDataInit.class)
        public ZookeeperDataInit zookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
            return new ZookeeperDataInit(zkClient, syncDataService);
        }
    }
    
    //省略了其他代码......
}

```

这个配置类是通过`SpringBoot`条件装配类实现的。在`ZookeeperListener`类上面有几个注解：

- `@Configuration`：配置文件，应用上下文；

- `@ConditionalOnProperty(prefix = "shenyu.sync.zookeeper", name = "url")`：属性条件判断，满足条件，该配置类才会生效。也就是说，当我们有如下配置时，就会采用`zookeeper`进行数据同步。

  ```properties
  shenyu:  
    sync:
       zookeeper:
            url: localhost:2181
            sessionTimeout: 5000
            connectionTimeout: 2000
  ```

- ` @Import(ZookeeperConfiguration.class)`：导入另一个类`ZookeeperConfiguration`；

```java
  @EnableConfigurationProperties(ZookeeperProperties.class)  // 启用zk属性配置类
  public class ZookeeperConfiguration {

    /**
     * register zkClient in spring ioc.
     * 向 Spring IOC 容器注册 zkClient
     * @param zookeeperProp the zookeeper configuration
     * @return ZkClient {@linkplain ZkClient}
        */
      @Bean
      @ConditionalOnMissingBean(ZkClient.class)
      public ZkClient zkClient(final ZookeeperProperties zookeeperProp) {
        return new ZkClient(zookeeperProp.getUrl(), zookeeperProp.getSessionTimeout(), zookeeperProp.getConnectionTimeout()); // 读取zk配置信息，并创建zkClient
      }
  }
```

```java
@Data
@ConfigurationProperties(prefix = "shenyu.sync.zookeeper") // zk属性配置
public class ZookeeperProperties {

    private String url;

    private Integer sessionTimeout;

    private Integer connectionTimeout;

    private String serializer;
}
```


当我们主动配置，采用`zookeeper`进行数据同步时，`zookeeperDataChangedListener`就会生成。所以在事件处理方法`onApplicationEvent()`中，就会到相应的`listener`中。在我们的案例中，是对一条选择器数据进行更新，数据同步采用的是`zookeeper`，所以，代码会进入到`ZookeeperDataChangedListener`进行选择器数据变更处理。

```java
    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        // 遍历数据变更监听器(一般使用一种数据同步的方式就好了)
        for (DataChangedListener listener : listeners) {
            // 哪种数据发生变更
            switch (event.getGroupKey()) {
                    
                // 省略了其他逻辑
                    
                case SELECTOR:   // 选择器信息
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());   // 在我们的案例中，会进入到ZookeeperDataChangedListener进行选择器数据变更处理
                    break;
         }
    }
```



#### 2.4 Zookeeper数据变更监听器

- ZookeeperDataChangedListener.onSelectorChanged()

  在`onSelectorChanged()`方法中，判断操作类型，是刷新同步还是更新或创建同步。根据当前选择器数据信息判断节点是否在`zk`中。

```java

/**
 * 使用 zookeeper 发布变更数据
 */
public class ZookeeperDataChangedListener implements DataChangedListener {
    
    // 选择器信息发生改变
    @Override
    public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        // 刷新操作
        if (eventType == DataEventTypeEnum.REFRESH && !changed.isEmpty()) {
            String selectorParentPath = DefaultPathConstants.buildSelectorParentPath(changed.get(0).getPluginName());
            deleteZkPathRecursive(selectorParentPath);
        }
        // 发生变更的数据
        for (SelectorData data : changed) {
            // 构建选择器数据的真实路径
            String selectorRealPath = DefaultPathConstants.buildSelectorRealPath(data.getPluginName(), data.getId());
            // 如果是删除操作
            if (eventType == DataEventTypeEnum.DELETE) {
                // 删除当前数据
                deleteZkPath(selectorRealPath);
                continue;
            }
            // 父节点路径
            String selectorParentPath = DefaultPathConstants.buildSelectorParentPath(data.getPluginName());
            // 创建父节点
            createZkNode(selectorParentPath);
            // 插入或更新数据
            insertZkNode(selectorRealPath, data);
        }
    }

	// 创建 zk 节点
    private void createZkNode(final String path) {
        // 不存在才创建
        if (!zkClient.exists(path)) {
            zkClient.createPersistent(path, true);
        }
    }

    // 插入zk节点
    private void insertZkNode(final String path, final Object data) {
        // 创建节点
        createZkNode(path);
        // 通过 zkClient 写入数据
        zkClient.writeData(path, null == data ? "" : GsonUtils.getInstance().toJson(data));
    }
    
}
```



只要将变动的数据正确写入到`zk`的节点上，`admin`这边的操作就执行完成了。`ShenYu`在使用`zk`进行数据同步时，`zk`的节点是通过精心设计的。



在我们当前的案例中，对`Divide`插件中的一条选择器数据进行更新，将权重更新为90，就会对图中的特定节点更新。

![](https://qiniu.midnight2104.com/20210916/zookeeper-node.png)



我们用时序图将上面的更新流程串联起来。

![](https://qiniu.midnight2104.com/20210916/zk-sync-sequence-admin-zh.png)



### 3. 网关数据同步

假设`ShenYu`网关已经在正常运行，使用的数据同步方式也是`zookeeper`。那么当在`admin`端更新选择器数据后，并且向`zk`发送了变更的数据，那网关是如何接收并处理数据的呢？接下来我们就继续进行源码分析，一探究竟。

#### 3.1 ZkClient接收数据

- ZkClient.subscribeDataChanges()

在网关端有一个`ZookeeperSyncDataService`类，它通过`ZkClient`订阅了数据节点，当数据发生变更时，可以感知到。

```java
/**
 * 使用 zookeeper 缓存数据
 */
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {
    
private void subscribeSelectorDataChanges(final String path) {
       // zkClient订阅数据节点
        zkClient.subscribeDataChanges(path, new IZkDataListener() {
            @Override
            public void handleDataChange(final String dataPath, final Object data) {
                cacheSelectorData(GsonUtils.getInstance().fromJson(data.toString(), SelectorData.class)); // 节点数据被更新
            }

            @Override
            public void handleDataDeleted(final String dataPath) {
                unCacheSelectorData(dataPath);  // 节点数据被删除
            }
        });
    }
 
    // 省略了其他逻辑
}
```



`ZooKeeper`的`Watch`机制，会给订阅的客户端发送节点变更通知。在我们的案例中，更新选择器信息，就会进入到`handleDataChange()`方法。通过`cacheSelectorData()`去处理数据。

#### 3.2 处理数据

- ZookeeperSyncDataService.cacheSelectorData()

经过判空逻辑之后，缓存选择器数据的操作又交给了`PluginDataSubscriber`处理。

```java
    private void cacheSelectorData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData)
                .ifPresent(data -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSelectorSubscribe(data)));
    }
```

`PluginDataSubscriber`是一个接口，它只有一个`CommonPluginDataSubscriber`实现类，负责处理插件、选择器和规则数据。



#### 3.3 通用插件数据订阅者

- PluginDataSubscriber.onSelectorSubscribe()

它没有其他逻辑，直接调用`subscribeDataHandler()`方法。在方法中，更具数据类型（插件、选择器或规则），操作类型（更新或删除），去执行不同逻辑。

```java
/**
 * 通用插件数据订阅者，负责处理所有插件、选择器和规则信息
 * The type Common plugin data subscriber.
 */
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    //......
     // 处理选择器数据
    @Override
    public void onSelectorSubscribe(final SelectorData selectorData) {
        subscribeDataHandler(selectorData, DataEventTypeEnum.UPDATE);
    }    
    
    // 订阅数据处理器，处理数据的更新或删除
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            // 插件数据
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) { // 更新操作
                    // 将数据保存到网关内存
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {  // 删除操作
                    // 从网关内存移除数据
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {  // 选择器数据
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) { // 更新操作
                    // 将数据保存到网关内存
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {  // 删除操作
                    // 从网关内存移除数据
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {  // 规则数据
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) { // 更新操作
                    // 将数据保存到网关内存
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) { // 删除操作
                    // 从网关内存移除数据
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    // 如果每个插件还有自己的处理逻辑，那么就去处理
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
    
}
```



#### 3.4 数据缓存到内存

那么更新一条选择器数据，会进入下面的逻辑：

```java
// 将数据保存到网关内存
BaseDataCache.getInstance().cacheSelectData(selectorData);
// 如果每个插件还有自己的处理逻辑，那么就去处理                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
```

一是将数据保存到网关的内存中。`BaseDataCache`是最终缓存数据的类，通过单例模式实现。选择器数据就存到了`SELECTOR_MAP`这个`Map`中。在后续使用的时候，也是从这里拿数据。

```java
public final class BaseDataCache {
    // 私有变量
    private static final BaseDataCache INSTANCE = new BaseDataCache();
  	// 私有构造器
    private BaseDataCache() {
    }
    
    /**
     * Gets instance.
     *  公开方法
     * @return the instance
     */
    public static BaseDataCache getInstance() {
        return INSTANCE;
    }
    
    /**
    *  缓存选择器数据的Map
     * pluginName -> SelectorData.
     */
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    public void cacheSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(this::selectorAccept);
    }
        
   /**
     * cache selector data.
     * 缓存选择器数据
     * @param data the selector data
     */
    private void selectorAccept(final SelectorData data) {
        String key = data.getPluginName();
        if (SELECTOR_MAP.containsKey(key)) { // 更新操作，先删除再插入
            List<SelectorData> existList = SELECTOR_MAP.get(key);
            final List<SelectorData> resultList = existList.stream().filter(r -> !r.getId().equals(data.getId())).collect(Collectors.toList());
            resultList.add(data);
            final List<SelectorData> collect = resultList.stream().sorted(Comparator.comparing(SelectorData::getSort)).collect(Collectors.toList());
            SELECTOR_MAP.put(key, collect);
        } else {  // 新增操作，直接放到Map中
            SELECTOR_MAP.put(key, Lists.newArrayList(data));
        }
    }
    
}
```



二是如果每个插件还有自己的处理逻辑，那么就去处理。  通过`idea`编辑器可以看到，当新增一条选择器后，有如下的插件还有处理。这里我们就不再展开了。  

![](https://qiniu.midnight2104.com/20210913/handler-selector.png)

经过以上的源码追踪，并通过一个实际的案例，在`admin`端新增更新一条选择器数据，就将`zookeeper`数据同步的流程分析清楚了。

我们还是通过时序图将网关端的数据同步流程串联一下：

![](https://qiniu.midnight2104.com/20210916/zk-sync-sequence-gateway-zh.png)

数据同步的流程已经分析完了，为了不让同步流程被打断，在分析过程中就忽略了其他逻辑。我们还需要分析`Admin`同步数据初始化和网关同步操作初始化的流程。



### 4. Admin同步数据初始化



当`admin`启动后，会将当前的数据信息全量同步到`zk`中，实现逻辑如下：

```java

/**
 * Zookeeper 数据初始化
 */
public class ZookeeperDataInit implements CommandLineRunner {

    private final ZkClient zkClient;

    private final SyncDataService syncDataService;

    /**
     * Instantiates a new Zookeeper data init.
     *
     * @param zkClient        the zk client
     * @param syncDataService the sync data service
     */
    public ZookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
        this.zkClient = zkClient;
        this.syncDataService = syncDataService;
    }

    @Override
    public void run(final String... args) {
        String pluginPath = DefaultPathConstants.PLUGIN_PARENT;
        String authPath = DefaultPathConstants.APP_AUTH_PARENT;
        String metaDataPath = DefaultPathConstants.META_DATA;
        // 判断zk中是否存在数据
        if (!zkClient.exists(pluginPath) && !zkClient.exists(authPath) && !zkClient.exists(metaDataPath)) {
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }
}

```

判断`zk`中是否存在数据，如果不存在，则进行同步。

`ZookeeperDataInit`实现了`CommandLineRunner`接口。它是`springboot`提供的接口，会在所有 `Spring Beans `初始化之后执行`run()`方法，常用于项目中初始化的操作。



- SyncDataService.syncAll()

从数据库查询数据，然后进行全量数据同步，所有的认证信息、插件信息、选择器信息、规则信息和元数据信息。主要是通过`eventPublisher`发布同步事件。这里就跟前面提到的同步逻辑就又联系起来了，`eventPublisher`通过`publishEvent()`发布完事件后，有`ApplicationListener`执行事件变更操作，在`ShenYu`中就是前面提到的`DataChangedEventDispatcher`。

```java
@Service
public class SyncDataServiceImpl implements SyncDataService {
    // 事件发布
    private final ApplicationEventPublisher eventPublisher;
    
     /***
     * 全量数据同步
     * @param type the type
     * @return
     */
    @Override
    public boolean syncAll(final DataEventTypeEnum type) {
        // 同步认证信息
        appAuthService.syncData();
        // 同步插件信息
        List<PluginData> pluginDataList = pluginService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, type, pluginDataList));
        // 同步选择器信息
        List<SelectorData> selectorDataList = selectorService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, type, selectorDataList));
        // 同步规则信息
        List<RuleData> ruleDataList = ruleService.listAll();
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, type, ruleDataList));
        // 同步元数据信息
        metaDataService.syncData();
        return true;
    }
    
}
```





### 5. 网关同步操作初始化

网关这边的数据同步初始化操作主要是订阅`zk`中的节点，当有数据变更时，收到变更数据。这依赖于`ZooKeeper`的`Watch`机制。在`ShenYu`中，负责`zk`数据同步的是`ZookeeperSyncDataService`，也在前面提到过。



`ZookeeperSyncDataService`的功能逻辑是在实例化的过程中完成的：对`zk`中的`shenyu`数据同步节点完成订阅。这里的订阅分两类，一类是已经存在的节点上面数据发生更新，这通过`zkClient.subscribeDataChanges()`方法实现；另一类是当前节点下有新增或删除节点，即子节点发生变化，这通过` zkClient.subscribeChildChanges()`方法实现。



`ZookeeperSyncDataService`的代码有点多，这里我们以插件数据的读取和订阅进行追踪，其他类型的数据操作原理是一样的。

```java

/**
 *  zookeeper 数据同步服务
 */
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {
    // 在实例化的时候，完成从zk中读取数据的操作，并订阅节点
    public ZookeeperSyncDataService( /*省略构造参数参数*/ ) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        // 订阅插件、选择器和规则数据
        watcherData();
        // 订阅认证数据
        watchAppAuth();
        // 订阅元数据
        watchMetaData();
    }
    
    private void watcherData() {
        // 插件节点路径
        final String pluginParent = DefaultPathConstants.PLUGIN_PARENT;
        // 所有插件节点
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        for (String pluginName : pluginZKs) {
            // 订阅当前所有插件、选择器和规则数据
            watcherAll(pluginName);
        }
        // 订阅子节点（新增或删除一个插件）
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    // 需要订阅子节点的所有插件、选择器和规则数据
                    watcherAll(pluginName);
                }
            }
        });
    }
    
    private void watcherAll(final String pluginName) {
        // 订阅插件数据
        watcherPlugin(pluginName);
        // 订阅选择器数据
        watcherSelector(pluginName);
        // 订阅规则数据
        watcherRule(pluginName);
    }

    private void watcherPlugin(final String pluginName) {
        // 当前插件路径
        String pluginPath = DefaultPathConstants.buildPluginPath(pluginName);
        // 是否存在，不存在就创建
        if (!zkClient.exists(pluginPath)) {
            zkClient.createPersistent(pluginPath, true);
        }
        // 读取zk上当前节点数据，并反序列化
        PluginData pluginData = null == zkClient.readData(pluginPath) ? null
                : GsonUtils.getInstance().fromJson((String) zkClient.readData(pluginPath), PluginData.class);
        // 缓存到网关内存中
        cachePluginData(pluginData);
        // 订阅插件节点
        subscribePluginDataChanges(pluginPath, pluginName);
    }
    
   private void cachePluginData(final PluginData pluginData) {
       // 省略实现逻辑，其实就是 CommonPluginDataSubscriber 中的操作，跟前面都能联系起来
    }
    
    private void subscribePluginDataChanges(final String pluginPath, final String pluginName) {
        // 订阅数据变更：更新或删除
        zkClient.subscribeDataChanges(pluginPath, new IZkDataListener() {

            @Override
            public void handleDataChange(final String dataPath, final Object data) {  // 更新操作
                 // 省略实现逻辑，其实就是 CommonPluginDataSubscriber 中的操作，跟前面都能联系起来
            }

            @Override
            public void handleDataDeleted(final String dataPath) {   // 删除操作
                  // 省略实现逻辑，其实就是 CommonPluginDataSubscriber 中的操作，跟前面都能联系起来

            }
        });
    }
    
}    
```



上面的源代码中都给出了注释，相信大家可以看明白。订阅插件数据的主要逻辑如下：

> 1. 构造当前插件路径
> 2. 路径是否存在，不存在就创建
> 3. 读取zk上当前节点数据，并反序列化
> 4. 插件数据缓存到网关内存中
> 5. 订阅插件节点





### 6. 总结

本文通过一个实际案例，对`zookeeper`的数据同步原理进行了源码分析。涉及到的主要知识点如下：

- 基于`zookeeper`的数据同步，主要是通过`watch`机制实现；
- 通过`Spring`完成事件发布和监听；
- 通过抽象`DataChangedListener`接口，支持多种同步策略，面向接口编程；
- 使用单例设计模式实现缓存数据类`BaseDataCache`；
- 通过`SpringBoot`的条件装配和`starter`加载机制实现配置类的加载。


