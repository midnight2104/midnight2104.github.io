---
layout: post
title: Soul网关中的Resilience4j插件
tags: Soul

---

本篇文章分析的是`Resilience4j`插件，它可以提供熔断和限流的功能。

操作前准备：启动`soul-admin`，`soul`网关，`soul-examples-http`测试用例。

#### Resilience4j功能演示

要在`soul`网关使用`Resilience4j`插件，需要引入依赖：

```xml
        <!-- soul resilience4j plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-resilience4j</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- soul resilience4j plugin end-->
```

然后需要在`soul-admin`中依次开启插件，添加选择器，添加规则：

![](https://midnight2104.github.io/img/2021-2-6/1.png)

![](https://midnight2104.github.io/img/2021-2-6/2.png)

![](https://midnight2104.github.io/img/2021-2-6/3.png)

规则字段解析：

- `token filling period(limitRefreshPeriod)`: 刷新令牌的时间间隔，单位ms，默认值：500。
- `token filling number(limitForPeriod)`: 每次刷新令牌的数量，默认值：50。
- `control behavior timeout(timeoutDurationRate)`: 熔断超时时间，单位ms，默认值：30000。
- `circuit enable(circuitEnable)`: 是否开启熔断，0：关闭，1：开启，默认值：0。
- `circuit timeout (timeoutDuration)`: 熔断超时时间，单位ms，默认值：30000。
- `fallback uri(fallbackUri)`: 降级处理的`uri`。
- `sliding window size(slidingWindowSize)`: 滑动窗口大小，默认值：100。
- `sliding window type(slidingWindowType)`: 滑动窗口类型，0：基于计数，1：基于时间，默认值：0。
- `enabled error minimum calculation threshold(minimumNumberOfCalls)`: 开启熔断的最小请求数，超过这个请求数才开启熔断统计，默认值：100。
- `degrade opening duration(waitIntervalFunctionInOpenState)`: 熔断器开启持续时间，单位ms，默认值：10。
- `half open threshold(permittedNumberOfCallsInHalfOpenState)`: 半开状态下的环形缓冲区大小，必须达到此数量才会计算失败率，默认值：10。
- `degrade failure rate(failureRateThreshold)`:错误率百分比，达到这个阈值，熔断器才会开启，默认值50。



上述准备工作完成后，就能进行测试了。每次刷新令牌的数量调成了只有一个，刷新时间要 100s，所以只要多点几次 postman，即会发生限流现象：

![](https://midnight2104.github.io/img/2021-2-6/4.png)





#### Resilience4j原理分析

`Resilience4JPlugin` 继承于模板抽象类 `AbstractSoulPlugin`，所以 `doExecutor` 是其执行真正功能逻辑的代码：

```java
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
		//省略了其他代码
        //是否开启熔断
        if (resilience4JHandle.getCircuitEnable() == 1) {
            return combined(exchange, chain, rule);
        }
        //限流处理
        return rateLimiter(exchange, chain, rule);
    }
```

主要功能是判断是否开启熔断，如果是，就进入组合模式（即处理熔断又处理限流）；否则，只做限流处理。

- 组合模式

```java
    private Mono<Void> combined(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        //设置管理后台配置的规则
        Resilience4JConf conf = Resilience4JBuilder.build(rule);
        return combinedExecutor.run(
            	//执行责任链后续操作
                chain.execute(exchange).doOnSuccess(v -> {
                    //省略了其他代码
                }), 
            fallback(combinedExecutor, exchange, conf.getFallBackUri()), //降级处理
            conf); // Resilience4J相关配置
    }
```

`combinedExecutor.run()`方法负责执行真实的逻辑：先执行熔断的功能，后执行限流的功能。再处理超时，错误信息，最后进行降级处理。

```java
    public <T> Mono<T> run(final Mono<T> run, final Function<Throwable, Mono<T>> fallback, final Resilience4JConf resilience4JConf) {

        //链式调用
        Mono<T> to = run.transformDeferred(CircuitBreakerOperator.of(circuitBreaker))//先处理熔断
                .transformDeferred(RateLimiterOperator.of(rateLimiter))//后限流
                .timeout(resilience4JConf.getTimeLimiterConfig().getTimeoutDuration())//超时处理
                .doOnError(/*...*/);//错误处理
        if (fallback != null) {
            to = to.onErrorResume(fallback);//降级处理
        }
        return to;
    }
```



- 限流模式

```java
    private Mono<Void> rateLimiter(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        return ratelimiterExecutor.run(
                chain.execute(exchange),  //责任链继续执行
            fallback(ratelimiterExecutor, exchange, null), //降级处理
            Resilience4JBuilder.build(rule))//规则配置
                .onErrorResume(throwable -> ratelimiterExecutor.withoutFallback(exchange, throwable));
    }
```

限流的功能主要在`ratelimiterExecutor.run()`中，负责生成限流器并执行。

```java
    @Override
    public <T> Mono<T> run(final Mono<T> toRun, final Function<Throwable, Mono<T>> fallback, final Resilience4JConf conf) {
        //限流器
        RateLimiter rateLimiter = Resilience4JRegistryFactory.rateLimiter(conf.getId(), conf.getRateLimiterConfig());
        Mono<T> to = toRun.transformDeferred(RateLimiterOperator.of(rateLimiter));//执行限流功能
        if (fallback != null) { //降级
            return to.onErrorResume(fallback);
        }
        return to;
    }
```



小结，本文结合实际案例演示了`Resilience4j`插件，并分析了组合模式和限流模式的执行原理。