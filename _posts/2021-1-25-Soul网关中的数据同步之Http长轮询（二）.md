---
layout: post
title: Soul网关中的数据同步之Http长轮询（二）
tags: Soul
---



在上一篇文章中，通过跟踪源码的方式了解了`http长轮询`的执行流程。但是，自己还有一些疑问，本篇文章是在[官网](https://dromara.org/zh-cn/docs/soul/dataSync.html)的基础上进行了拓展，加入一些自己的理解。

> `zookeeper`、`websocket` 数据同步的机制比较简单，而 `http` 同步会相对复杂一些。`Soul `借鉴了 `Apollo`、`Nacos` 的设计思想，取其精华，自己实现了 `http` 长轮询数据同步功能。注意，这里并非传统的 `ajax` 长轮询！

![](https://midnight2104.github.io/img/2021-1-25/http-long-polling.png)



`http 长轮询`机制如上所示，请求逻辑是`Soul网关`主动请求 `soul-admin` 的配置服务。响应逻辑有两种：`soul-admin`端本身的配置修改和`60s`的等待时间到了。

`http` 请求到达` sou-admin` 之后，并非立马响应数据，而是利用 `Servlet3.0` 的异步机制，异步响应数据。首先，将长轮询请求任务 `LongPollingClient` 扔到阻塞队列 `BlocingQueue` 中，并且开启调度任务，每`60s` 执行一次，将队列中的请求拿出，发送对应的响应。如果没有发生配置信息的更改，也需要对请求响应，好让网关知道，不需要一直等待。当然，网关请求配置服务时，也有 `90s` 的超时时间。

```java
class LongPollingClient implements Runnable {
    LongPollingClient(final AsyncContext ac, final String ip, final long timeoutTime) {
        // 省略......
    }
    @Override
    public void run() {
        // 加入定时任务，如果60s之内没有配置变更，则60s后执行，响应http请求
        this.asyncTimeoutFuture = scheduler.schedule(() -> {
            // clients是阻塞队列，保存了来自soul-web的请求信息
            clients.remove(LongPollingClient.this);
            List<ConfigGroupEnum> changedGroups = HttpLongPollingDataChangedListener.compareMD5((HttpServletRequest) asyncContext.getRequest());
            //发送响应
            sendResponse(changedGroups);
        }, timeoutTime, TimeUnit.MILLISECONDS);
        //放到阻塞队列中
        clients.add(this);
    }
}
```

如果这段时间内，`soul-admin`发生了数据信息的更改，此时，会挨个移除队列中的长轮询请求，并响应数据，告知是哪个 `Group` 的数据发生了变更（我们将插件、规则、流量配置、用户配置数据分成不同的组）。网关收到响应信息之后，只知道是哪个 `Group` 发生了配置变更，还需要再次请求该 `Group` 的配置数据。

```java
// soul-admin发生了配置变更，挨个将队列中的请求移除，并予以响应
class DataChangeTask implements Runnable {
    DataChangeTask(final ConfigGroupEnum groupKey) {
        this.groupKey = groupKey;
    }
    @Override
    public void run() {
        try {
            //挨个处理阻塞队列中的请求
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext(); ) {
                LongPollingClient client = iter.next();
                //移除
                iter.remove();
                //响应
                client.sendResponse(Collections.singletonList(groupKey));
            }
        } catch (Throwable e) {
            LOGGER.error("data change error.", e);
        }
    }
}
```



  当 `soul-web` 网关层接收到` http` 响应信息之后，拉取变更信息（如果有变更的话），然后再次请求 `soul-admin` 的配置服务，如此反复循环。

`长轮询`体现在请求任务会一直执行。

```java
class HttpLongPollingTask implements Runnable {

    	//省略其他代码

        @Override
        public void run() {
            //一直循环下去
            while (RUNNING.get()) {
                for (int time = 1; time <= retryTimes; time++) {
                    try {
                        doLongPolling(server);
                    } catch (Exception e) {
                        //一直循环下去
                    }
                }
            }
            log.warn("Stop http long polling.");
        }
    }
```





**轮询**：客户端每隔几秒钟向服务端发送 `http` 请求，服务端在收到请求后，不论是否有数据更新，都直接进行响应。在服务端响应完成，就会关闭这个 `TCP` 连接。这种方式实现非常简单，兼容性也比较好，只要支持 `http` 协议就可以用这种方式实现。缺点就是非常消耗资源，会占用较多的内存和带宽。

**长轮询**：客户端发送请求后服务器端不会立即返回数据，服务器端会阻塞，请求连接挂起，直到服务端有数据更新或者是连接超时才返回，客户端才再次发出请求新建连接、如此反复从而获取最新数据。相比轮询，长轮询减少了很多不必要的 `http` 请求次数，相比之下节约了资源。



参考文献：

- [Soul-数据同步机制-http长轮询同步](https://github.com/itmiwang/SE-Notes/blob/main/SourceCode/Soul/09.Soul-%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E6%9C%BA%E5%88%B6-http%E9%95%BF%E8%BD%AE%E8%AF%A2%E5%90%8C%E6%AD%A5.md)
- [数据同步原理](https://dromara.org/zh-cn/docs/soul/dataSync.html)

