

一、**RPC****的简化版原理**如下图（核心是代理机制）。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=M2FjZDhlMGE1NjBmMDBjMTgyYzc2NTBhZmEzZjcwYTVfRDdqYWxDcUd3OUxGOXFKMWo5NWE0TkNxc2RuU2xxeE1fVG9rZW46U0lDQWJLR2Rqb2c0eGd4MFh3dmNlRVlibnpnXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

1.本地代理存根: Stub

2.本地序列化反序列化

3.网络通信

4.远程序列化反序列化

5.远程服务存根: Skeleton

6.调用实际业务服务

7.原路返回服务结果

8.返回给本地调用方

二、新建一个模块rpc-demo-consumer

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NGFjMDlhYzA1OGFjZjQ4ZmNlMWQzMTlhNGVkNGYwZWJfeG8xaGZ2cWhLTUFtdkxMdjNoeEJMbGd1akluRWlEU1ZfVG9rZW46SUpxOGJ4eTlFb1FndVp4bGJPa2M3a1NIbmpoXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

主要的依赖是rpc-demo-api，即业务服务定义的api接口，rpc-core是核心依赖包。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MmMzNmVhNWY4MTQ4ODBhZWFkODFlNzJiYmRkYTFlODFfOEhwaVE2ZUpTTDBONFNja2phd0s1bU9BcHJhTWdVZWZfVG9rZW46TUgweWJYU21ub3h3dG54ODM1dmNYOENzbndLXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

三、启动方法

在启动方法中注入业务服务接口UserService和OrderService，使用的注解是@RpcConsumer，本文主要的功能就是如何实现这个注解。

在启动的时候会导入ConsumerConfig的配置类。

使用的ApplicationRunner来测试方法调用。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YWYzMjQxNmY3OTRkYjhkYjQ5ODBhMWM0YzdmZjg0MjFfMnZyQzRFckxuT3FXSUVNUUtqOG13VHR4SFFUMnNIN2VfVG9rZW46UmFYSGJ4MlVnbzk5YkF4VFU3WGNWU1RvbnVmXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

四、ConsumerConfig用来启动消费者服务。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MmI5YzZjYzVhNzUxYmZjZmVmNzI5YTc5NjQzYTkwMWJfbE54cU83bjZ1Qlh6eWVCbWRiZDR1NFBKQVh2SGpzQ1hfVG9rZW46QVlRdGJNNVQ0b1JKdmJ4TE4zNGNNZjNabjJlXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

五、ConsumerBootstrap实现了ApplicationContextAware接口，注入ApplicationContext，即Spring的应用上下文，获取bean。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MDg2ZDEzMjhmMzIzM2Y2ZWMwMzRiNWVlN2Y0ZTVjYzZfS1cxVnVQVzlrOVFVbXlZSk9BUkg1Q21XOHpkcFh0dUZfVG9rZW46UFdqT2JCUDdib29DMkZ4YTFvM2NqdTM1bmxjXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

在start()方法中，

1.获取所有的bean；

2.遍历每个bean，查找bean中是否有字段用了@RpcConsumer注解；

3.对被标记了的@RpcConsumer的字段进行遍历；

4.获取到该字段的类型，全限定名；

5.使用动态代理的方式创建消费者对象；

6.把代理对象赋值给空字段。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NDZjOWVhMzJkMTc5MjMxYmJmMWZmMzNjY2ZhM2Q1MDNfNWpGa1FFSVhiVGhFWmFQSXFjQmFGNUN6NWpub2h2ZkhfVG9rZW46SXlGeWIxejV4b1FDbEx4VTZxNGM1YkQ2blFkXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

查找bean中的字段是否使用了@RpcConsumer注解，通过循环遍历的方式进行判断。不断向父类寻找，是因为bean本身可能被Spring进行了CGLIB增强，形成了一个代理类。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=OGU4MGM1ZDU3ZWNjNjk0YTE3OGNmOTdjNzY2ZjNhZjZfZFREa3JralJNbFJQN01rQXVCZ01TZ2U1NE85WENEejZfVG9rZW46SmN2Z2JTQVpNb0JXM054anRUQWNBYTRLbnhCXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

使用Proxy创建代理对象。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=NDBjNmFjYTk4MTQ2ZjUzMzdjZTRjMmZlZDJhMDc1ZmVfSzFOS3VOSzhHM1Q1cGFKaTdBNEpJNEZRSGVRbVg1dmhfVG9rZW46THlybGI0NTRTb3M4REZ4QUQ4a2NCdW0zbldnXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

六、代理对象RpcInvocationHandler中定义了远程调用方法的逻辑。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=N2U5ZmJiOTNiYjZhNmU4ZDM2ZTc5MWM4NDNiNzY5ZGRfSDZ6azdtMFdlb1BnOXhYbjJyeG14dkFNU1Z6NFNZQWdfVG9rZW46TTREemJhRWwzb2c0OFB4aGVrNmN1dUZ5bkpjXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

在invoke()方法中定义具体实现： 

1.判断是否本地方法，就不需要走远程调用；

2.封装请求参数；

3.发起远程调用；

4.处理调用结果。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MTZlMDFkM2QxOTc3YjZkZmQ3Y2RmMWZhMGRlN2UxODlfMHJKOFJOcHNGQ0dhaEp5OUFIdDJwN3M0UG9EV2hmVFhfVG9rZW46RmF0ZGJtcU9qbzlGRzF4QllkMGN2RUNhbkVPXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

本地方法判断：即Object类中方法不需要被代理，不需要发起远程调用请求。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=ODNlOThhMzUyMTAyMjQ2YjIwZDVkMjcwNzZkMzYwNmZfYm16VDUzaEZ4Q2xoQ2IyMkt5b29aQ0FPenI2anV6Rm1fVG9rZW46RDdTV2JDY1Rqb2psdTd4cFNCSGNVaVB5bnhiXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

RpcRequest封装了远程调用消费者端请求的数据结构：接口全限定名称、方法名称、方法参数。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=M2E0ZmVjNDM4MmI4NjBiYWQyM2M1ZDc5ODhmN2YxNzlfVHBMU292OHZIOU9KUEFtMldsUTdSS2tQMWVtdGlhU0NfVG9rZW46Skg0OWJjMXpmbzhneld4d29UUGM2d3pybkFkXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

通过post方法发起远程调用，使用的是OkHttpClient。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YjQ5Zjc3ZjI0OGY0ZTMyMWRkYWZlYWZkZTg1YzY5NmFfU1EzY2Rjd0o5MjdNN0E1MVl0ZVN3TnpMNVZGZEtzRjRfVG9rZW46TXhLaGJsY21qb2l2eFF4YXMyZGN2aEVWbkhoXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

RpcResponse定义了消费者端的响应体数据结构：状态、响应结果数据、异常对象。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=MDc5NzRkNTk0MDg3MTgxOWZlMjMzOTJhNmI5NzUyOTRfWmpReTNjYWZFeG9uN3dud0toTWdjZjFkOVduN09Ka2hfVG9rZW46R1NzRGI4NjFRb0V0MU94ZU5mUmM5b2VBbmJmXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

处理调用结果：如果成功，就对调用结果数据进行反序列化；如果失败就封装异常信息。

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=YzQwNTI3YzY3OWY2Njc4YzY2Yzg3NTFmMzRiZjM1OGZfUnhYeEt0ZUljM3ZEcmFZbWhZSVEzZ3lLTjBlaGtLMGJfVG9rZW46WUtlTGJFakVzb01nTXF4VE1jWmNIMlJ3bk5oXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

七、对各种情况的基本测试：

1. 类对象参数的远程调用；
2. 本地方法的远程调用；
3. 基本参数类型的远程调用；
4. String类型参数的远程调用；
5. 另一个业务范围（订单服务）的远程调用；
6. 嵌套注入的远程调用；
7. 异常测试

![img](https://qxjtjpi1tsf.feishu.cn/space/api/box/stream/download/asynccode/?code=Yzk1OTQ1MTczOGZmNTE0MzMxNjM4ODEyOWUxMTA4OThfQnRETDc3Z3gyS3JRUk9HbnJlcmo1MVE4U0pXWVBFczhfVG9rZW46UHpQN2JlYUF3bzlxbWN4VGhRamNkaHdPbk9iXzE3MTA2NTY5NDY6MTcxMDY2MDU0Nl9WNA)

还有哪些问题？

1. 默认使用的Java的动态代理，还有Spring的切面代理，CGLI增强，ByteBuddy的字节码
2. 当前还是localhost请求，后续应该是走网络请求。
3. 基本类型参数并不完整，如果是Double类型参数可能会报错。



工程地址：https://github.com/midnight2104/midnight-rpc/tree/lesson2