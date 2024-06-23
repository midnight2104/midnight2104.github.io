1. 定义SpringValueProcessor处理类

- *实现*BeanPostProcessor后置处理器接口，*扫描所有的Spring value，保存起来。*
- *实现ApplicationListener接口，在配置变更时，更新所有的spring value*

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6Yvl5wOibKD9s6WAKMdcM95icj78iafMfxTPxCvNZibn57CiatwTnO295mPf3Xg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. *实现*BeanPostProcessor后置处理器接口

实现postProcessBeforeInitialization()方法。通过工具类找到所有带有@Value的bean，遍历每个字段。通过helper工具类抽取注解上面的表达式，构建SpringValue对象并保存起来。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6Yvle6b17rwNn60WWMEfr2heqrXdC2HBZ9mKUOQ0qVRdOBcSk18ZhN6rww/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用SpringValue对象封装@Value注解信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvlNgyEVDCOQYh5ialAy5b8jPiabjiamMT2PichsUU71YTwsa3Tue7hK74EhA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

一个SpringValue对象如下：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvlictgTibEKkvFXic5YjkAT4nh0ibvrJqgDC6ajg4mefjG5A8AqXBUwMrSOg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 实现onApplicationEvent方法

- 事件变更时，自动实现该方法；
- 从前面保存的holder中获取SpringValues；
- 遍历集合，解析获取属性值；
- 通过反射重新赋值。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvljibrgMFz8x3Vm7mkXDqTmUnx5Q6To6cookEqDxOd0NBf6SEXWYjgBhQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4. 测试

- 启动 config-server
- 启动config-client

初始值如下：都是dev100

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvlvKbZjpnr8K2iaxibsIiawznbKDlCic0ydxMMaFjo8pMzz2SUrkvETKSW4A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

更新配置中心的值

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6Yvl3V4jhn7u4AovRWqBqZWpAQCqYwp1ZIlrEmib7zpnSZMDoTLwCicHTjvw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

观察config-demo的日志

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvlTiacKN4jvkEAa3ByZVXKGqxlBmTL7OooCJwT4ibOp6UDaJ2BvicknrG5g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

再次获取值，就已经动态发生了变化。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYlzMXoUf7NTk14riacY6YvldtYwh1Y2YrAT7s9nhuGJia80cl55X20wRDlsbvWK5Edaahrd4qIYmxA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

源码：https://github.com/midnight2104/midnight-config/tree/v4