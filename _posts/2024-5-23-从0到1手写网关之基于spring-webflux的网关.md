1. 引入依赖

- 添加rpc-core用于从注册中心获取服务
- 添加webflux的依赖

注意：spring web 和 spring webflux 不兼容

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZ6t0uIMb9a6icibZINgLRY1HuORnC28nX7BNmx4c5rxibtmv5EZMcgbQLZ6cHtqgomYcE1mL3YfpU8Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 实现GatewayHandler

- 1.通过请求路径获取服务名称
- 2.通过rc拿到所有或者的实例
- 3.负载均衡
- 4.拿到请求报文
- 5.通过webclient发送post请求
- 6.通过entity获取响应报文
- \7. 组装响应报文

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZ6t0uIMb9a6icibZINgLRY1Hup5OAMibuIBH6kaMDg8jwuEdnwabHBNRYdrFfr2ggHeSBrSMKo55mlg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZ6t0uIMb9a6icibZINgLRY1HQDxDV3a9kaf4JU0GsuiaoTWS0wTwtMIlAcnuZWmPKTnpicJ6EdFPHkrA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 实现GatewayRouter

- 实现请求的路径映射

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZ6t0uIMb9a6icibZINgLRY1HBKqH3KGz4zwpRDcn9ALJoAYvjhyePJPO1arN87ic83EmLJibaWeQ7pBA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 测试

- 启动注册中心、配置中心、和服务提供者
- 启动网关
- 发起请求调用

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgZ6t0uIMb9a6icibZINgLRY1HWJq6ErdLHcRB09v1ibvUCliavOic91qlwLBmOibxYhxiaXD7N22m7ZHlBQQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

源码：https://github.com/midnight2104/midnight-gateway