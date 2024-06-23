1. 自定义缓存MidnightCache

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86Aoia29v0sOgg8Fm8WibSbSWPw5RpAicGYqBgK1Cz9j2H35ChaThjKDxWPA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. 实现redis协议

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86AhleSr69f75c5Bpn1tK0ga0SePpK89x4HdEpsCic7fZFN53glFLYx9rg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

integr编码处理

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86AiaibS0GFosrjZL7diaWve9eslXLe9g3DA7ouLqW5kI3RXTvnvaLR7q9pw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

错误编码处理

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86Ax5jn0XjPzdTGISYujNhxKcIibokFjpfXTbMLAmTxj8r0ric4wbiabK4GQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

数组编码处理

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86AslPaQc98RFLJvT8KKKa6rmm7MEPRoSMpjr3fLR4laxqZZCM2CHRwKg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

负责字符串处理

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86AQp5HajeG9qVENfIwicJ8NkD0XXdXcmo5QoibY6OUQRTggLerxjSCw9yg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 测试

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaA4MBphSX50xZ6SzAen86A9e88uyC70icTTSUsuO8lVaG8GTT9icf2c2OrU6aiaTxV0nXKlHEClIw8A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- *REDIS 协议规范*  *https://redis.com.cn/topics/protocol.html**在 RESP 中，数据的类型依赖于首字节：**单行字符串（Simple Strings）： 响应的首字节是 "+"**错误（Errors）： 响应的首字节是 "-"**整型（Integers）： 响应的首字节是 ":"**多行字符串（Bulk Strings）： 响应的首字节是"\$"**数组（Arrays）： 响应的首字节是 "\*"**RESP协议的不同部分总是以 "\r\n" (**CRLF**) 结束。*

源码：https://github.com/midnight2104/midnight-cache