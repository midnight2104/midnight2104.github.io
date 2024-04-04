

一、**RPC**的简化版原理如下图（核心是代理机制）。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5Xq61KNSO1YN4K9f5ianHNOdocQu3Zz0Zlpzkls8qL3nvM1tJDhz3nj4Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1.本地代理存根: Stub

2.本地序列化反序列化

3.网络通信

4.远程序列化反序列化

5.远程服务存根: Skeleton

6.调用实际业务服务

7.原路返回服务结果

8.返回给本地调用方

二、新建一个模块rpc-demo-consumer

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5Xk0dibT1Wz9wrY3O5ykg9dAsI0rAOqkn2Giad0G0ia34D7pHuMnrTNkdsQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

主要的依赖是rpc-demo-api，即业务服务定义的api接口，rpc-core是核心依赖包。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XbF1ZtNNXia2SEMyS59VweBAkyjMKJcd4C1DNAI1p44SMpfdThIlN43A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

三、启动方法

在启动方法中注入业务服务接口UserService和OrderService，使用的注解是@RpcConsumer，本文主要的功能就是如何实现这个注解。

在启动的时候会导入ConsumerConfig的配置类。

使用的ApplicationRunner来测试方法调用。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XPpPB8uXTvzqeS8PpBMzicPBfqGUuFI8Xcdsq9ExMe5dOqIuDrNk2jibw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

四、ConsumerConfig用来启动消费者服务。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XdotGpm9wd5c7GBrdiaRo4xbpmnhlMhqvKozXpEZ10DRvoOuLsQNExXA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

五、ConsumerBootstrap实现了ApplicationContextAware接口，注入ApplicationContext，即Spring的应用上下文，获取bean。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XPjxchOn6zaNFvCpRuLbEGYbGCAjU1esTCsKdefqTZQ23zUg0uMq0cw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在start()方法中，

1.获取所有的bean；

2.遍历每个bean，查找bean中是否有字段用了@RpcConsumer注解；

3.对被标记了的@RpcConsumer的字段进行遍历；

4.获取到该字段的类型，全限定名；

5.使用动态代理的方式创建消费者对象；

6.把代理对象赋值给空字段。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XEFqlSRgluIoZJibGM8kQO1PT9pFkTvIwKR0089Wllwsp9VtS4QhmIrA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

查找bean中的字段是否使用了@RpcConsumer注解，通过循环遍历的方式进行判断。不断向父类寻找，是因为bean本身可能被Spring进行了CGLIB增强，形成了一个代理类。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5X1fbnCy1sfB4YKnGGTZDPWKO6jicVhx6PMxPUeyTdbNhibBzDQwmL3iafw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用Proxy创建代理对象。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5X6eicTx7zoGOKPgjL5ejPgL4RYGKcJ6L9bbUtMyhPnOZmibdricBL3iaTmw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

六、代理对象RpcInvocationHandler中定义了远程调用方法的逻辑。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XwlIAIKZHnfLHUWu25VGWRQJZpdWJQ8NcjEm9BoO5ffErt51IxFokGg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在invoke()方法中定义具体实现：1.判断是否本地方法，就不需要走远程调用；

2.封装请求参数；

3.发起远程调用；

4.处理调用结果。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XhRRgTTx3uK5NW1uRhYluNUIXVRwGKZDL3ibUz7gjqGViavFPyGoaZLicQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

本地方法判断：即Object类中方法不需要被代理，不需要发起远程调用请求。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5Xv2bcVPcEJOXgzhtre1PClWlpeqlddXbtyicJ33n6CKfbag1CzAVr80A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

RpcRequest封装了远程调用消费者端请求的数据结构：接口全限定名称、方法名称、方法参数。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XkSfu1rDicFdM8Q549juQECCxtKk6ur9n7UKlbutyR8Saxialu5HejVOQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过post方法发起远程调用，使用的是OkHttpClient。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XLWU1wSR5myMWwRCm4vHJvxqpXibZIcjiaOaicLcLEQVUTZR05XSsG9s2g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

RpcResponse定义了消费者端的响应体数据结构：状态、响应结果数据、异常对象。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XCGa0SD4Lc5M5iaM8FcmPBVcEpp3S5ek3ibSu94HJ95p1Cia1dOUmqwp4Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

处理调用结果：如果成功，就对调用结果数据进行反序列化；如果失败就封装异常信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XibIGIa2sJNBhJUKwKepxakEoUuq4ghwhuvjwvVK1ABzaxrXKbNVjKWA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

七、对各种情况的基本测试：

1.类对象参数的远程调用；2.本地方法的远程调用；3.基本参数类型的远程调用；4. String类型参数的远程调用；5. 另一个业务范围（订单服务）的远程调用；6. 嵌套注入的远程调用；7. 异常测试

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZUEm4WYqt9mfiaD5X8qGy5XVMZE4VddZBgA2uiaG8xRQOMUK0edYTAKA4Zbj7KangOWQ7pNWuMLwvw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还有哪些问题？

1. 默认使用的Java的动态代理，还有Spring的切面代理，CGLI增强，ByteBuddy的字节码
2. 当前还是localhost请求，后续应该是走网络请求。
3. 基本类型参数并不完整，如果是Double类型参数可能会报错。



工程地址：https://github.com/midnight2104/midnight-rpc