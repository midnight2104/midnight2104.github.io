**一、实现ZkRegistryCenter**，主要接口方法如下：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XSnXAB2FxqYJMeiblMnsiaexaaX1mCb8kV00F07pLB3sRQkFsNmfwBv8A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. start()启动方法，作为客户端连接ZooKeeper。定义连接串，命名空间和连接重试策略，随着服务的启动而启动。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XicNRmkhTXIlYWcaEz6HYLwdBvOO85nEvYDMuljAIH83ibiaKgKh8BicJyQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. stop()停止方法，断开和ZooKeeper的的连接。服务被销毁时，连接信息随之关闭。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XPOapHkKyrNkaWFH8Flgq33h3bQllooPs8C6n3UG6ulTwZzHwEKaPUw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. register()服务注册方法，服务提供者provider向zk的注册。服务节点为持久化节点，实例是临时节点，当实例下线时，也会删除节点。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XzovQjhazYDTEnFzBMaoDjLod1cmb2gficPuj44NumyUdjKVkSmdp0Cg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. unregister()服务取消注册方法，服务提供者provider取消向zk的注册。判断服务路径是否存在，不存在直接返回。删除节点时，使用quietly删，没有实例也不要报错。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XN8DrbdJfm3anXM8X0929OszNOcJLd8dDkt5Kpjy5EoibibichISHkgF2g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. fetchAll()获取所有实例方法，服务消费者consumer从zk获取到所有服务提供者provider实例。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XiafNzzrXDe1ATSF60Qa1Hx6K1H1hrs21z62lDEUoYcCSsqPJc9tA2aA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. subscribe()订阅方法，服务消费者consumer从zk获取变动的实例。使用TreeCache缓存了服务节点，并添加监听器，有节点发生变动时就会执行。即再次获取所有服务实例，通过ChangedListener执行特定动作。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5X7ibaBiaWoIszLfZxQjEanwyicYUwcPky431blCYVhjcVmqBiaAzT6QFrDw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**二、服务提供者provider注册**

1. 加载注册中心，生成bean，指定初始化方法start()和销毁方法stop()。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XaNRR3Kobwnc15ZMAibH8tW10Xtwg2oBSDpCVsrytkJVb5D2Lwia7QlAA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 启动服务提供者provider的引导类。注意，*Spring 的所有bean加载完成，项目启动成功才进行服务注册，防止启动过程中就注册，然后启动失败了，已经注册的可能会被调用。*

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XnU4UkGm5KATr1JOSnibDFibW7qkibOs6URn3qZWdgz7r7EZPP00omgAtQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 服务注册，所有bean加载完成后，服务端provider完成注册，调用注册中心RegisterCenter的register()方法完成服务的注册。服务实例instance其实就是ip+端口。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XSc18FzoXQ7bGOGXibbkKpE8iasObhr9wt7bXeWkcCibiaDFUCzriavKX9BA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 服务销毁

使用@PreDestory注解，当服务停止时，便取消服务注册，使用注册中心RegisterCenter的unregister()方法。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XCMibgga6ZZaYGxTgPcviclfqGy9xWujjlhn4egGu29HiaiaE44IxJ2V5Dg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**三、服务消费者consumer**

在服务消费者consumer启动的过程中，向注册中心获取服务接口实例信息，放到代理类中。同时，完成注册中心的订阅，有节点发生变动，就会清空当前已经获取到的服务节点信息，然后再次获取最新的实例信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XRv4KVz2dq5PRREl0rxEbDa632VQic0N2iabgVTPCg6b0BhWmI3Ob7jFg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**四、测试**

1. 本地启动ZooKeeper，默认单机模式，端口是2181，然后使用ZooInspector连接zk，作为可视化界面。
2. 接着启动3个服务提供者provider，端口使用8081, 8082, 8083。成功后，通过ZooInspector会看到如下信息。两个服务都有三个实例。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XxZ6q9Ar1DkDWgCmuFla1wYM9NFNDHdRY1Xm9WibyuwLXKKy4eRWfhicw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 启动服务消费者consumer，连续调用6次接口。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XhPlfdnUl0hLvYGyF1Sjg4bJyQGyqKmViaqoab3lLvF2BEwh742tIIVA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

\4. 观察日志信息，使用轮询的负载均衡也起作用了，也从注册中心获取到了全部节点信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5X8b7EeEIDrugrpj5ZkX8qkMoetucVnmibLsSg6YnqYISHzdwZwYICia7g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 关闭一个服务8081，会看到注册中心也少了一个8081。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XCPFh1tgPI0fyR71Caon4ORXUpSf3BxXDYyIib1mmzz5kriaNGibGejYzg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还有哪些问题？

1. 代码质量不高，需要重构。
2. 过滤器，缓存，mock还不支持。
3. 异常处理不完善，超时重试还不支持。
4. 灰度发布也还不支持。



源码： 

https://github.com/midnight2104/midnight-rpc/tree/lesson5