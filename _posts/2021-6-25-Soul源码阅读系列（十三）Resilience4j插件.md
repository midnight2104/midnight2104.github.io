---
layout: post
title: 2021-6-25-Soul源码阅读系列（十三）Resilience4j插件
tags: Soul
---



本篇文章分析的是`Resilience4j`插件，`Resilience4J`是`Spring Cloud Gateway`推荐的容错方案，它是一个轻量级的容错库。它可以提供熔断和限流的功能。

操作前准备：启动`shenyu-admin`，`shenyu`网关，`shenyu-examples-http`测试用例。

> `Soul` 网关最近换名字了，新的名字叫`ShenYu`，所以文章中可能出现书写不一致的地方。

#### Resilience4j 功能演示

要在`shenyu`网关使用`Resilience4j`插件，需要引入依赖：

```xml
        <!-- shenyu resilience4j plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>shenyu-spring-boot-starter-plugin-resilience4j</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- shenyu resilience4j plugin end-->
```

然后需要在`shenyu-admin`中依次开启插件，添加选择器，添加规则：

![](https://qiniu.midnight2104.com/20210625/1.png)

![](https://qiniu.midnight2104.com/20210625/2.png)

![](https://qiniu.midnight2104.com/20210625/4.png)

规则字段解析：

- `limitRefreshPeriod`: 刷新令牌的时间间隔，单位ms，默认值：500。
- `limitForPeriod`: 每次刷新令牌的数量，默认值：50。
- `timeoutDurationRate`:  等待获取令牌的超时时间，单位ms，默认值：5000。
- `circuitEnable`: 是否开启熔断，0：关闭，1：开启，默认值：0。
- `timeoutDuration`: 熔断超时时间，单位ms，默认值：30000。
- `fallbackUri`: 降级处理的`uri`。
- `slidingWindowSize`: 滑动窗口大小，默认值：100。
- `slidingWindowType`: 滑动窗口类型，0：基于计数，1：基于时间，默认值：0。
- `minimumNumberOfCalls`: 开启熔断的最小请求数，超过这个请求数才开启熔断统计，默认值：100。
- `waitIntervalInOpen`:  熔断器开启持续时间，单位ms，默认值：10。
- `bufferSizeInHalfOpen`: 半开状态下的环形缓冲区大小，必须达到此数量才会计算失败率，默认值：10。
- `failureRateThreshold`:错误率百分比，达到这个阈值，熔断器才会开启，默认值50。



上述是默认参数，在插件中还有参数校验逻辑，如果参数值小于默认值，会直接赋值默认值，因此方便测试效果直接修改源码的配置 ： 每次刷新令牌的数量为2 ，刷新令牌的时间间隔为`1s`，超时时间为`1s`。

```java
public void checkData(final Resilience4JHandle resilience4JHandle) {
        resilience4JHandle.setTimeoutDurationRate(Math.max(resilience4JHandle.getTimeoutDurationRate(), Constants.TIMEOUT_DURATION_RATE));

        // 刷新令牌的时间间隔为1s
        resilience4JHandle.setLimitRefreshPeriod(1000);
        // 每次刷新令牌的数量为2
        resilience4JHandle.setLimitForPeriod(2);

        resilience4JHandle.setCircuitEnable(Math.max(resilience4JHandle.getCircuitEnable(), Constants.CIRCUIT_ENABLE));

        // 超时时间为1s
        resilience4JHandle.setTimeoutDuration(1000);

        resilience4JHandle.setFallbackUri(!"0".equals(resilience4JHandle.getFallbackUri()) ? resilience4JHandle.getFallbackUri() : "");
        resilience4JHandle.setSlidingWindowSize(Math.max(resilience4JHandle.getSlidingWindowSize(), Constants.SLIDING_WINDOW_SIZE));
        resilience4JHandle.setSlidingWindowType(Math.max(resilience4JHandle.getSlidingWindowType(), Constants.SLIDING_WINDOW_TYPE));
        resilience4JHandle.setMinimumNumberOfCalls(Math.max(resilience4JHandle.getMinimumNumberOfCalls(), Constants.MINIMUM_NUMBER_OF_CALLS));
        resilience4JHandle.setWaitIntervalFunctionInOpenState(Math.max(resilience4JHandle.getWaitIntervalFunctionInOpenState(), Constants.WAIT_INTERVAL_FUNCTION_IN_OPEN_STATE));
        resilience4JHandle.setPermittedNumberOfCallsInHalfOpenState(Math.max(resilience4JHandle.getPermittedNumberOfCallsInHalfOpenState(), Constants.PERMITTED_NUMBER_OF_CALLS_IN_HALF_OPEN_STATE));
        resilience4JHandle.setFailureRateThreshold(Math.max(resilience4JHandle.getFailureRateThreshold(), Constants.FAILURE_RATE_THRESHOLD));
    }
```

在要被测试的方法中加上日志信息：

```java
    @GetMapping("/findById")
    @ShenyuSpringMvcClient(path = "/findById", desc = "Find by id")
    public OrderDTO findById(@RequestParam("id") final String id) throws InterruptedException {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setId(id);
        orderDTO.setName("hello world findById");

        log.info("限流测试");

        return orderDTO;
    }
```



#### 限流测试

上述准备工作完成后，就能进行测试了。根据配置：每次刷新令牌的数量为2 ，刷新令牌的时间间隔为`1s`，超时时间为`1s`。在`1s`内只有两个线程能够拿到令牌，可以通过，这样就达到了限流的作用。

使用`SuperBenchmarker`工具，`4`个线程，执行`10s`：

```sh
C:\Users>sb -u http://localhost:9195/http/order/findById?id=2 -c 4 -N 10
Starting at 2021/6/25 12:21:16
[Press C to stop the test]
24      (RPS: 1.5)
---------------Finished!----------------
Finished at 2021/6/25 12:21:33 (took 00:00:16.4476099)
26      (RPS: 1.6)                      Status 200:    26

RPS: 2.2 (requests/second)
Max: 2005ms
Min: 382ms
Avg: 1716.3ms

  50%   below 1997ms
  60%   below 1998ms
  70%   below 1998ms
  80%   below 1999ms
  90%   below 2001ms
  95%   below 2003ms
  98%   below 2005ms
  99%   below 2005ms
99.9%   below 2005ms
```

可以看到结果信息中`RPS: 2.2 (requests/second)`，表示每秒请求数量是`2.2`。

然后，日志中也能看到，在同一秒内，只有两个请求：

```sh
2021-06-25 12:21:30.020  INFO 35060 --- [ctor-http-nio-5] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:30.020  INFO 35060 --- [ctor-http-nio-4] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:31.022  INFO 35060 --- [ctor-http-nio-5] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:31.023  INFO 35060 --- [ctor-http-nio-4] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:32.019  INFO 35060 --- [ctor-http-nio-5] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:32.019  INFO 35060 --- [ctor-http-nio-4] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:33.019  INFO 35060 --- [ctor-http-nio-4] o.a.s.e.http.controller.OrderController  : 限流测试
2021-06-25 12:21:33.019  INFO 35060 --- [ctor-http-nio-5] o.a.s.e.http.controller.OrderController  : 限流测试
```



#### 熔断测试

从配置信息我们知道熔断器默认是关闭，我们需要手动打开，即将参数`circuitEnable`的值设置为`1`。

在`Resilience4JHandle#checkData()`手动设置超时时间为`1s`:

```java
        // 超时时间为1s
        resilience4JHandle.setTimeoutDuration(1000);
```



在被测试的方法中加上线程休眠时间：

```java
    @GetMapping("/findById")
    @ShenyuSpringMvcClient(path = "/findById", desc = "Find by id")
    public OrderDTO findById(@RequestParam("id") final String id) throws InterruptedException {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setId(id);
        orderDTO.setName("hello world findById");

        log.info("限流测试");

        int i = RandomUtils.nextInt(1,3);
        if(i %2 == 0){
            Thread.sleep(2000);
        }

        return orderDTO;
    } 
```



通过`postman`多次发送请求，就会发生因超时出现的熔断。

![](https://qiniu.midnight2104.com/20210625/5.png)

#### Resilience4J原理分析

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
    private Mono<Void> combined(final ServerWebExchange exchange, final ShenyuPluginChain chain, final RuleData rule) {
        //配置信息
        Resilience4JConf conf = Resilience4JBuilder.build(rule);
        return combinedExecutor.run(
                chain.execute(exchange).doOnSuccess(v -> { //执行成功后的处理逻辑
                    HttpStatus status = exchange.getResponse().getStatusCode();
                    if (status == null || !status.is2xxSuccessful()) {
                        exchange.getResponse().setStatusCode(null);
                        throw new CircuitBreakerStatusCodeException(status == null ? HttpStatus.INTERNAL_SERVER_ERROR : status);
                    }
                }), fallback(combinedExecutor, exchange, conf.getFallBackUri()),  // 降级处理
            conf); //配置信息
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