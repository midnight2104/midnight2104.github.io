**一、在过滤器中定义好了前置过滤和后置过滤接口。**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE316OknD0KgdUKUDkYicZM4TdJE5PjI732hJDDH25e3FzF9AKPUFYqBLg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**二、支持MockFilter**

Mock过滤器应用场景是：当发现微服务间调用异常时，开启MockFilter可以对服务请求进行mock，排查是哪个微服务发生了问题。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3916TVENSozqTcEib6eibMTMQiciagQlefd8Niamic2N4exq448TQQAZ8csqA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

前置过滤接口的实现逻辑是：

1. 从RpcRequest中获取服务接口类型；
2. 找到当前调用方法；
3. 获取方法返回值类型；
4. 对返回值类型进行mock

没有实现后置逻辑。

MockUtils的实现逻辑：对基本类型进行mock，对String类型进行mock，对一般类进行mock，还没有实现数组、集合、Map等复杂数据类型。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3YFRW3Z7eSaSrSGCWAhUVA7rwF7nW9ghkm34Q7DMNjsVwRnAE9Ape7Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**二、支持CacheFilter**

CacheFilter的应用场景是：对调用过的请求进行缓存，不再发起远程调用，起缓存作用。

在前置过滤器中获取缓存，在后置过滤器中存放缓存结果。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3ibW5LDK3FqKFwMByicCVjiabetAGSXrMdB98PjrGFDkQibv1jicoIheib53A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**三、在Consumer的应用**

1. 在配置类ConsumerConfig中声明bean。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3hTk0fCibK2GOsLxumyR7UgybKW7icUC3IPw3iapCZTNBJXibM16r5SaL0A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 在启动类ConsumerBootstrap中放到Context中，传递给代理对象。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3fo7Veicia1rts37X2Zu7hBOYDcT30m7j79RQw9DPYzgmI9r1oEFeicuIw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 在代理对象中分别调用前置处理逻辑和后置处理逻辑。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3iaHvqMISwkzXAycKOQ8MV2ibiaDVgMpibQnJKQGwfSwmLqiaGzzLjaMXkoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

四**、测试**

1. 缓存过滤器CacheFilter

调用findById()方法，第一次生成对象后，多次调用，返回的时间戳是一样的，证明缓存过滤器起作用了。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3biaePECIictbaWoroyyMhoAkVy553SJ0svg0Y1Do1ibJEnURic6RwI3ThA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. mock过滤器MockFilter

查看启动类中的测试方法，返回的都是mock值，就成功了。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb0c1RKicpQeicicqibrLu6OOE3eBNIFUKDkdqpIaiblpWFL6owUNxU0GzekT2VKohtziaGNylfu8TX0LYQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

还有什么问题？

1. Filter可以用责任链设计模式；
2. 可以使用其他缓存框架，支持过期时间；
3. 缓存没有考虑写操作。



源码：

https://github.com/midnight2104/midnight-rpc/tree/lesson7