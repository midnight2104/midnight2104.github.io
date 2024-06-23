1. 实现WebHandler接口

- 通过请求路径获取服务名称
- 通过rc拿到所有活着的服务实例
- 负载均衡
- 拿到请求的报文
- 通过webclient发送post请求
- 通过entity获取响应报文
- 组装响应报文

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qyGjpBibAhQYsPibwL7Z5eoiceBCbuItHuEU46PCibgV6zfDX59aGpnbWoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qOtp0ibI1N6ukU3IDZn7K2GLBMia1yYqopaGlE1YwGjmO0uvALg9KnsMQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



2. 添加映射处理器

在spring上下文加载完成后，添加网关的映射处理器。添加映射关系后，需要重新初始化上下文。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qq9ibuLt27KiaQo0e5puTFmo8wqlKmD0A15m0Wtw7ryLWcvr3t6tdF30g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



3. 添加前置过滤器

前置处理器模拟mock操作。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6q6ib5Ust26Shqm6kpibdqMKOlUwDUEsfQ09u9zGBTicUmOeZHbSxORicb1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4. 添加后置过滤器

后置过滤器打印属性信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qo54Fbg7VX4y1rNtDOUs8ibiaKKdRkUI5XCqjBnTkgOuGFCuVib5XTF2Zw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

5. 测试

- 启动注册中心
- 启动服务提供者
- 启动网关
- 发起请求

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qVY7CULPiarGpmLXI1QKO9AaLjtrgrkDUDriaL1H0OkfwG9qcfVvmwUqQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

观察日志，经过了前置过滤器和后置过滤器，返回了调用结果。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgYiba7IiawUhz46ibUpvDkwW6qNrNxKmzUk8HSWlNTK0Qdjm01UumxSgGTcddJRMtws2LGLm0RpZFpPg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)