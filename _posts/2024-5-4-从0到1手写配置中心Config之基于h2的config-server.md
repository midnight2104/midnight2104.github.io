1. 添加依赖

新建的web工程中添加h2的依赖

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIcPk1GiaRBUzS4T8qUTMibwJO4cngKdnOibxZmGykca3ibumBeFLMWTJicAg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. 添加h2的配置

- 设置数据源和密码
- 设置初始化sql语句
- 打开h2的控制台

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIVOJNSadujUJic1IGrw37WZZABFOuIjiblrlP40O3M4vDYAOCUwYuKOdQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

初始化语句创建一个config表，保存服务配置信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIejPWNmGEliaEmhI9hJ216VFSCyR5axDXz0QNFDrXOB5wJI3OLatgKGw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 完成CRUD接口

controller类

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIK9JPcPEnKbccGiay5HZkIExsNLdyzPaJwLW7QL7M8VadKjP9xmWm4uQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

mapper接口

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIWLQgpe5yIk80oXjEoia3ibgGOTGFhbFbnMpakXRgCC0fczdqAIicaOXZQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4. 测试

在web控制台可以看到sql已经初始化完成，crud接口也比较简单。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgb4DuQwkF4To4ltt1vmvpBIoibSSWpOmHR8Mknog6hJm9nTaXKibh2ej2jjGDpN05zhsecBAIOQrnuw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

源码： https://github.com/midnight2104/midnight-config