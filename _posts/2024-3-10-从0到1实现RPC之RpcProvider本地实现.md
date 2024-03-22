### RPC的简化版原理如下图（核心是代理机制）。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJLFlNsMHTXHRoCztybPTzKNOp8icrGiaicz2ayv2MkvtQYTvqEtrm9jhDA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 1.本地代理存根: Stub
- 2.本地序列化反序列化
- 3.网络通信
- 4.远程序列化反序列化
- 5.远程服务存根: Skeleton
- 6.调用实际业务服务
- 7.原路返回服务结果
- 8.返回给本地调用方

注意处理异常。

### RpcProvider的本地实现

#### 1、工程结构

- rpc-core是核心实现类
- rpc-demo-api是定义实际业务服务的接口
- rpc-demo-provider是业务接口实现类

![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJ6GOVdYwqPVhJjV3pxmgCZrth3pL1fvXZPpUXNcLoa8HF0chReWPwNg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2、RpcProvider在启动过程中把@RpcProvider标记的方法存到Map中去
在启动时加载配置
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJppDnJcUyibF5M3bggiaYqMPyuW5RxTD9Dkfg3A5tsJqMPnv2AF4RMiaNg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在配置中创建启动类
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJVGDadwu2ABhpYhPRnR8XklwVyibkqj6scMOWwppoWJv78V2q97aWghA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

启动类在创建的过程，会将带有@RpcProvider注解的类存放到Map中。
@PostConstruct注解可以在bean的初始化后进行执行，刚好满足我们的常见（存储bean）
使用 ApplicationContextAware接口是为了获取Spring的应用上下文ApplicationContext，从里面获取bean。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJsFxkus1NKWQGAZia0yAMBAEQCqU1cHnTTQ2BaFQfIsWI4Dx7ZUIzJQg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

RpcProvider是自定义的一个注解，用来表示这个类是一个服务提供者。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJE8CSgTj2hibJ3o4MIMeEc0bdjSAibzUibzn5BwXmAz9mJod4HxUZZFibbA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一个服务提供者，订单服务实现类。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJn5T1cHJKxibYlMGZdovZo5nRCgpcr9RXmNDdHRXWmUDLEKWLEq6lxIw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 3、在调用的时候通过方法名称找到应该调用的方法，通过反射完成调用。
发起一个http请求调用，获取订单信息。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJtK3micW2wvWR9d8fjqTtTUr4ianwcbdYRHt10ETibsuAOXauicNk3Z6Sfg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

RpcRequest封装请求参数，包括：接口名称、方法名、参数。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJooxiccQmfe5k96nouSTHGmhz7ofZlc1TJ1MyPWnlITLe97RamvUxptA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


一个请求入口

![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJ7ZhjyqBiaWwDjqVIrqoGm8keODU80dnk9olQFwxqukicqPicvZgpvN2UQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

请求调用，根据服务全限定名（就是一个类的全名）去map存找对应的类，找到后，通过反射发起方法调用。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJR2OvazmbXU0Mgc7HDazHNCjLeu7ohibJV0kHIkibyn2jnBEeGic5Sfksw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

响应类RpcResponse统一封装结果，返回状态，响应结果数据。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJ3ZdhIYiabsQGnogOHVotEdgPy0Z11lB7BDicIibhugF5VNYmqnIXCF2Rw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

实际的调用结果，就成功了。
![](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYuruGQ0aeqMJgPBDpPsKQJ1xkOYXZK5tbYDl0sIKlea8lN6os7ZFzgfeVmCRplKLDOVNZSkJdXkA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


#### 4、有哪些问题？
如果一个接口有多个实现类怎么办？
错误消息怎么处理？
方法有多个重载方法怎么办？

工程地址：https://github.com/midnight2104/midnight-rpc