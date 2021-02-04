---
layout: post
title: Soul网关中的Hystrix熔断插件（一）
tags: Soul

---

本篇文章要体验`Soul`网关中对`Hystrix`熔断插件的支持。当系统承接的流量太大时，为防止这些大流量将系统压垮，常常考虑使用熔断机制，将请求断开，以保护系统。

参考之前的流程，启动`soul-admin`和`soul`网关，以及业务系统（本次的测试演示使用的是`soul-examples-http`）。注意在`soul`网关中，加入`hystrix`插件。

```xml
        <!-- soul hystrix plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-hystrix</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- soul hystrix plugin end-->
```

启动`soul-admin`后，在管理控制台也需要手动打开`hystrix`插件。

![](https://midnight2104.github.io/img/2021-2-4/1.png)

开启后，在`hystrix`插件中新建一个选择器：

![](https://midnight2104.github.io/img/2021-2-4/2.png)

然后创建相应的选择器规则：

![](https://midnight2104.github.io/img/2021-2-4/3.png)

规则中的字段含义：

- 跳闸最小请求数量 ：最小的请求量，至少要达到这个量才会触发熔断
- 错误百分比阀值 ： 这段时间内，发生异常的百分比。
- 最大并发量 ： 最大的并发量
- 跳闸休眠时间`(ms)` ：熔断以后恢复的时间。
- 分组`Key`： 一般设置为:`contextPath`，如果不设置，默认值就是业务系统的`contextPath`。
- 命令`Key`: 一般设置为具体的路径接口。



以上准备工作就做完了，现在开始压测，压测工具使用的是`SuperBenchmarker`。压测命令如下，并发请求15，压`10s`。

`sb -u http://localhost:9195/http/order/findById?id=123 -c 15 -N 10`

得到的结果信息如下：

```sh
---------------Finished!----------------
Finished at 2021/2/4 20:33:48 (took 00:00:13.6597429)
Status 500:    52239
Status 200:    18

RPS: 4673.3 (requests/second)
Max: 72ms
Min: 0ms
Avg: 0.1ms

  50%   below 0ms
  60%   below 0ms
  70%   below 0ms
  80%   below 0ms
  90%   below 0ms
  95%   below 0ms
  98%   below 2ms
  99%   below 3ms
99.9%   below 8ms

```

可以看到其中有很多请求都是返回的`500`，这是后端网关主动断开的请求。查看后端网关的日志信息：

```xml
2021-02-04 20:33:47.629  INFO 18504 --- [work-threads-17] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :hystrix-rule-test
2021-02-04 20:33:47.629 ERROR 18504 --- [work-threads-17] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-02-04 20:33:47.629  INFO 18504 --- [work-threads-18] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :hystrix
2021-02-04 20:33:47.629  INFO 18504 --- [work-threads-18] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :hystrix-rule-test
2021-02-04 20:33:47.629 ERROR 18504 --- [work-threads-18] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-02-04 20:33:47.629  INFO 18504 --- [-work-threads-1] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :hystrix
2021-02-04 20:33:47.629  INFO 18504 --- [-work-threads-1] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :hystrix-rule-test
2021-02-04 20:33:47.629  INFO 18504 --- [-work-threads-2] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :hystrix
2021-02-04 20:33:47.629  INFO 18504 --- [-work-threads-2] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix rule success match , rule name :hystrix-rule-test
2021-02-04 20:33:47.629 ERROR 18504 --- [-work-threads-2] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-02-04 20:33:47.630 ERROR 18504 --- [-work-threads-1] o.d.soul.plugin.hystrix.HystrixPlugin    : hystrix execute have circuitBreaker is Open! groupKey:/http,commandKey:/order/findById
2021-02-04 20:33:47.630  INFO 18504 --- [-work-threads-3] o.d.soul.plugin.base.AbstractSoulPlugin  : hystrix selector success match , selector name :hystrix
```

日志信息中明确记录了`Error`信息：`hystrix execute have circuitBreaker is Open!`。在业务系统无法接受请求时便主动断开（我们配置的规则中最大并发量是1）。这里就测试出了`Soul`网关中`Hystrix`熔断器的作用。

我们再试试把`hystrix`插件关掉再测一下：

```
$ sb -u http://localhost:9195/http/order/findById?id=123 -c 15 -N 10
Starting at 2021/2/4 20:44:11
[Press C to stop the test]
29871   (RPS: 2229.8)
---------------Finished!----------------
Finished at 2021/2/4 20:44:25 (took 00:00:13.5005204)
Status 200:    29871

RPS: 2691.4 (requests/second)
Max: 62ms
Min: 0ms
Avg: 1.2ms

  50%   below 1ms
  60%   below 1ms
  70%   below 1ms
  80%   below 1ms
  90%   below 2ms
  95%   below 3ms
  98%   below 4ms
  99%   below 4ms
99.9%   below 9ms
```



这一次的请求就都成功了。



最后，本文通过实际的案例体验了`soul`网关中的熔断插件`hystrix`。明天开始分析原理~

