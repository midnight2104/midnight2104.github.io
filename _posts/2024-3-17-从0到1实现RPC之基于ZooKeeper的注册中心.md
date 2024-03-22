**一、实现ZkRegistryCenter**，主要接口方法如下：

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YWUyOWFlNjg3NmE5OTQ5NGNkNDU4ZGRlMTc3MWQ3MTRfT2FFM2lVTkNSSTd1N2xkSmJMWTd0OUpZSUJzMHR0a1dfVG9rZW46RERPcGJwU2lDb2tNYkl4dzBRcmNNMG80bkhmXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

1. start()启动方法，作为客户端连接ZooKeeper。定义连接串，命名空间和连接重试策略，随着服务的启动而启动。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NjUzNzdmNTk5ODIwN2NhMzQxYWNkZjcyODg1MmJiN2RfU2pkRDNPWjlMcG5wMXFFUE9oYlBST3A4bll2NnZQZkFfVG9rZW46WWtsY2IyblJZb2w0UWx4Mno0SmNOSUNubmNjXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

2. stop()停止方法，断开和ZooKeeper的的连接。服务被销毁时，连接信息随之关闭。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjQzZDI4NzJhOTAzMzZiMjE3OWVhMzhhOTQ1YjEzMmZfdVdpZ3VoVDNzSjQ3bHdUOFdycjJIdHk2QUN1blMxUEdfVG9rZW46UTR3R2I5Y2o1b0tYM1B4MmsyNmN1RG5rbkxoXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

3. register()服务注册方法，服务提供者provider向zk的注册。服务节点为持久化节点，实例是临时节点，当实例下线时，也会删除节点。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YjEyZGNkNDJhOTllMjg0YTA5NjI4YjFlNWI2MGM1YzVfRkFVR0t6R2kyYmc2ZzJWTGZaV1hOakJLQkZiM2o0aWRfVG9rZW46VmhxemJrb0V5bzNHWmZ4Y1VFMWNmVER1bm1oXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

4. unregister()服务取消注册方法，服务提供者provider取消向zk的注册。判断服务路径是否存在，不存在直接返回。删除节点时，使用quietly删，没有实例也不要报错。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDY2NmVkYmE4MDU3N2EwYTRiMWE0OTQ1MDNlMmE4Yjdfa0F6cXlrcDNnMHJZRVhUZFZDSHlGMDlFcnRwdGdQY1lfVG9rZW46U2U4TmJuR3FVb1I5RUR4TWdBQmNFS08wblJmXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

5. fetchAll()获取所有实例方法，服务消费者consumer从zk获取到所有服务提供者provider实例。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MjM4NmFmMDNmNjg2NzZlNDBkMzkyZWRmYmEyN2VkNmRfVFV0bzlYVzJBUWZmTXM4blVLSjZPOHhkSjd0VEJpVDJfVG9rZW46V0JlVmJhV2Jjb1NFcTF4UDYySGM5d1IyblhmXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

6. subscribe()订阅方法，服务消费者consumer从zk获取变动的实例。使用TreeCache缓存了服务节点，并添加监听器，有节点发生变动时就会执行。即再次获取所有服务实例，通过ChangedListener执行特定动作。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NmZmMTc0Yzk3YzQ2ZGQxMzMzZDAwY2YxYjc5ZDEzYjdfOWN1Q2JVU0VxaVJpQ1hPa1Nwd0UyVFQxa3JRYWl5OUlfVG9rZW46R29CWWJwSjRMb0p4R2R4S2dLZ2NNS1kxbm1mXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

**二、服务提供者provider注册**

1. 加载注册中心，生成bean，指定初始化方法start()和销毁方法stop()。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=OTY4NzdiYzY4MTg2OGIwYTNiZTllMDRlY2NhMzdlZTlfRTFONTFUc3UyaG5vU0FNREx4ZUpiME42b1M5Ulk2eDhfVG9rZW46SUNZU2J4UFZqb2tYeTV4SlRVcGNTSkJ0bnJqXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

2. 启动服务提供者provider的引导类。注意，*Spring 的所有bean加载完成，项目启动成功才进行服务注册，防止启动过程中就注册，然后启动失败了，已经注册的可能会被调用。*

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=M2Q1ZTkzNzQ4YWI2NDRkNzgyNTM4N2MyZGMzNjM4NjVfNG92WXVuNkdjNEVpcTdnMzZpcVg1clhqVHZOUGZUS2tfVG9rZW46WmpsdWJzcmhDb3B2bE94TnFmdWNZWXhhbjNlXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

3. 服务注册，所有bean加载完成后，服务端provider完成注册，调用注册中心RegisterCenter的register()方法完成服务的注册。服务实例instance其实就是ip+端口。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YWM0OWJjODlkNGE3ZWQ5NWUzZWE1ZWQ1Yzc1ODY3ZmZfQkRrRng4MXJTeTVLR256Z1E0aUY5RVBhenRPelZiaGlfVG9rZW46SjdzTWJidmY4b0c4bTB4QmNSamNLZnlrbmpoXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

4. 服务销毁

使用@PreDestory注解，当服务停止时，便取消服务注册，使用注册中心RegisterCenter的unregister()方法。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NTY3OTA4MzNlNTIxMWExYmY0NWViY2IzMGQ1YjZjYjlfTkFSNVRpZkxRWmFJblVQRGJISXJ5NXhpU29VcktneGlfVG9rZW46RkpKbGJlU01hbzdiRTh4eWZCUWNCZDlubmpiXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

**三、服务消费者consumer**

在服务消费者consumer启动的过程中，向注册中心获取服务接口实例信息，放到代理类中。同时，完成注册中心的订阅，有节点发生变动，就会清空当前已经获取到的服务节点信息，然后再次获取最新的实例信息。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NmQxYTU0ODMxYzI1MDg4YWI3YzQ2NWJjYzgwNTIzN2Rfb0hibGd0b2lraThtQ1RaMmVoOHlKZGQzSWdLRGFGZEhfVG9rZW46R2hFcGJValIxb3JjaTB4UDlEWmMxdDlsblo2XzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

**四、测试**

1. 本地启动ZooKeeper，默认单机模式，端口是2181，然后使用ZooInspector连接zk，作为可视化界面。
2. 接着启动3个服务提供者provider，端口使用8081, 8082, 8083。成功后，通过ZooInspector会看到如下信息。两个服务都有三个实例。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=OTcwNTIxMDk2MTVmNDY5Y2UzYTJhZTg2OTRhNjczOWZfd2V1VjNRN1ZpeDhQdDRjUm95VGhHSDNVRHJuclRaa09fVG9rZW46VjFSOWJQczF6b0lpUk14WEZiMmNHeUpIbnRmXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

3. 启动服务消费者consumer，连续调用6次接口。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MmI1YjkyYzBkOTgxNDc5NWQyNmY4Y2NlNWNlNzFhYzZfTERXaGVuUDZKRTFSazJhNXdEbmhyTFdqcW9mbHNCMUFfVG9rZW46QUNwYmJvR0p2b24xdGp4VklHZ2NvNWFXbkdoXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

4. 观察日志信息，使用轮询的负载均衡也起作用了，也从注册中心获取到了全部节点信息。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NTY1ZTFhYTQ5ZGU5NWU5M2E2MWRiZThlZmE2OTA3ODRfZlBuV0dtRENINDlEd2x6bGIzclJ5eE93QTlFcmtNZHhfVG9rZW46U2tNd2I5cUNhb05LOUl4VXRtYWNXdHdlbm1VXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

5. 关闭一个服务8081，会看到注册中心也少了一个8081。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=OGUyYTUyYzIwMmJjOWJkY2E1NTM1MWEzMzQyY2RlOTJfcnl0RXhVanhjZzBubFVUaTA5dFQ1YVJQOXFKcFpvSnVfVG9rZW46TkpyN2JJRU1Fb2FPNjd4SVpZamNpMTEzbk1mXzE3MTExMDg1MzM6MTcxMTExMjEzM19WNA)

还有哪些问题？

1. 代码质量不高，需要重构。
2. 过滤器，缓存，mock还不支持。
3. 异常处理不完善，超时重试还不支持。
4. 灰度发布也还不支持。