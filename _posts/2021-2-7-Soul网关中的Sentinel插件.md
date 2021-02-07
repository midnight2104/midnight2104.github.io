---
layout: post
title: Soul网关中的Sentinel插件
tags: Soul

---

本篇文章分析的是`Sentinel`插件，它也可以提供熔断和限流的功能。

操作前准备：启动`soul-admin`，`soul`网关，`soul-examples-http`测试用例。

#### `Sentinel`功能演示

首先在`soul`网关中添加`Sentinel`插件，引入依赖：

```xml
        <!-- soul sentinel plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-sentinel</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- soul sentinel plugin end-->
```

然后需要在`soul-admin`中依次开启插件，添加选择器，添加规则：

![](https://midnight2104.github.io/img/2021-2-7/1.png)

![](https://midnight2104.github.io/img/2021-2-7/2.png)

![](https://midnight2104.github.io/img/2021-2-7/3.png)



规则字段解析：

- `degrade count`:  熔断阈值;
- `whether to open the degrade (1 or 0)`: 是否开启熔断；
- `degrade type`: 熔断策略，支持秒级 RT/秒级异常比例/分钟级异常数；
- `control behavior`:  流控策略（直接拒绝 / 排队等待 / 慢启动模式），不支持按调用关系限流；
- `grade count`：限流阈值；
- `whether control behavior is enabled (1 or 0)`: 是否开启限流；
- `grade type`: 限流类型，`QPS `或线程数模式。



上述准备工作完成后，就能进行测试了。开启限流，设置流控阈值为1，然后多点几次 `postman`，即会发生限流现象：

![](https://midnight2104.github.io/img/2021-2-7/4.png)



#### `Sentinel`原理分析

`SentinelPlugin` 继承于模板抽象类 `AbstractSoulPlugin`，所以 `doExecutor` 是其执行真正功能逻辑的代码：

```java
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
		//省略了其他代码
        SentinelHandle sentinelHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SentinelHandle.class);
        //限流或降级执行类 SentinelReactorTransformer
        //责任链继续执行
        return chain.execute(exchange).transform(new SentinelReactorTransformer<>(resourceName)).doOnSuccess(v -> {
            //不成功，则抛出降级异常
            if (exchange.getResponse().getStatusCode() != HttpStatus.OK) {
                HttpStatus status = exchange.getResponse().getStatusCode();
                exchange.getResponse().setStatusCode(null);
                throw new SentinelFallbackException(status);
            }
        }).onErrorResume(throwable -> sentinelFallbackHandler.fallback(exchange, UriUtils.createUri(sentinelHandle.getFallbackUri()), throwable));
    }
```

限流或降级执行类是 `SentinelReactorTransformer`，来自于`Sentinel`插件。集成`Sentinel`插件就是以上代码，使用起来还是比较简单。

`Sentinel`规则在`admin`修改后，会同步更新到`SentinelRuleHandle`：

```java
public void handlerRule(final RuleData ruleData) {
        SentinelHandle sentinelHandle = GsonUtils.getInstance().fromJson(ruleData.getHandle(), SentinelHandle.class);

        //更新限流规则
        List<FlowRule> flowRules = FlowRuleManager.getRules()
        if (sentinelHandle.getFlowRuleEnable() == Constants.SENTINEL_ENABLE_FLOW_RULE) {
			//省略了其他代码
        }
        FlowRuleManager.loadRules(flowRules);
    
        //更新熔断降级规则
        List<DegradeRule> degradeRules = DegradeRuleManager.getRules()
        if (sentinelHandle.getDegradeRuleEnable() == Constants.SENTINEL_ENABLE_DEGRADE_RULE) {
		//
        }
        DegradeRuleManager.loadRules(degradeRules);
    }
```



小结，本文结合实际案例演示了`Sentinel`插件，跟踪了`Sentinel`插件源码。