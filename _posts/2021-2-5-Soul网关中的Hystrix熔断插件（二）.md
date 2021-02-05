---
layout: post
title: Soul网关中的Hystrix熔断插件（二）
tags: Soul

---

在上一篇文章中，体验到了`soul`中`hystrix`插件熔断的功能，今天来分析其中执行过程和相关原理。

在`HystrixPlugin`插件中就是主要的核心处理逻辑：

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
       //省略了其他代码
       //构建command对象
        Command command = fetchCommand(hystrixHandle, exchange, chain);
        return Mono.create(s -> {
            //执行具体请求
            Subscription sub = command.fetchObservable().subscribe(s::success,
                    s::error, s::success);
            s.onCancel(sub::unsubscribe);
            //熔断器打开了
            if (command.isCircuitBreakerOpen()) {
                log.error("hystrix execute have circuitBreaker is Open! groupKey:{},commandKey:{}", hystrixHandle.getGroupKey(), hystrixHandle.getCommandKey());
            }
        }).doOnError(throwable -> { //错误信息处理
            log.error("hystrix execute exception:", throwable);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
            //执行下一个插件
            chain.execute(exchange);
        }).then();
    }
```

通过上面的执行逻辑，可以得到两个个关键点：

- `command`对象的构建；
- `command.fetchObservable().subscribe()`方法的执行；

`command`对象构建过程：

```java
    private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
        //信号量模式
        if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
            return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
                exchange, chain, hystrixHandle.getCallBackUri());
        }
        //线程池模式
        return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
            exchange, chain, hystrixHandle.getCallBackUri());
    }
```

`Hystrix`提供了两种隔离模式：一种是信号量，一种是线程池。本篇文章先分析基于信号量的隔离模式。

`HystrixCommand`继承了`HystrixObservableCommand`，这个类来自于`com.netflix`，就是`Hystrix`熔断器的类。

```java
public class HystrixCommand extends HystrixObservableCommand<Void> implements Command 
```

在`HystrixCommand`中，有重要的实现方法：

```java
@Override
    protected Observable<Void> construct() {
        return RxReactiveStreams.toObservable(chain.execute(exchange));
    }

	//任务执行失败或者熔断打开的时候被执行
    @Override
    protected Observable<Void> resumeWithFallback() {
        return RxReactiveStreams.toObservable(doFallback());
    }

    private Mono<Void> doFallback() {
        if (isFailedExecution()) {
            log.error("hystrix execute have error: ", getExecutionException());
        }
        final Throwable exception = getExecutionException();
        return doFallback(exchange, exception);
    }

	//调用 subscribe 方法后 执行注事件
    @Override
    public Observable<Void> fetchObservable() {
        return this.toObservable();
    }

    @Override
    public URI getCallBackUri() {
        return callBackUri;
    }
```

- `construct + fetchObservable` 负责注册请求任务事件，其中 `fetchObservable` 是 `soul `自定义接口 `Command `的方法，为了提供统一的 `API `给 `doExecutor `调用。
- 事件由 `HystrixPlugin.doExecutor()` 方法中的 `subscribe` 调用真正发起执行
- `resumeWithFallback `触发 `Hrstrix fallbac`k 机制，接口 `Command `中 `default `方法` doFallback `负责真正执行 `fallback `逻辑。



最后，本文主要分析了 `HystrixPlugin doExecutor` 的执行流程，关于`hystrix`熔断本身的机制并没有数量清楚，后续还需要再去看一看。

