1. 在PropertySourcesProcessor中，需要通过http从config-server获取配置。



![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0R5rFdeEGRZczlrIkBey9fEibOosA2xueZ98T5XdUwyLClNAmNL2BsDYg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



2. 使用ConfigMeta包装服务信息

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0Rc42bllFc9pvzf6oN8TmPotrr25NIITwLojQ09v7mIxVF0MQfwU1iaSw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 在MidnightConfigService接口中添加默认实现类

- 继承MidnightRepositoryChangeListener接口；
- 获取默认的MidnightRepository；
- 创建MidnightConfigServiceImpl实现类；
- 添加listener

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RLrKsqujPxtaicul4yGTqibQ3INfCeW3YS8icNCZ4tHemba3PWq5HOBzGg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

4. 定义MidnightRepositoryChangeListener，用于事件监听的回调接口

- onChange() 执行方法
- ChangeEvent变动事件，传递参数

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0R19qyE35FmewqfyEqllSYJibN93QRuRfS2QtAZjzwhg4XM7MJkgoM9HA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

5. 定义MidnightRepository接口

- 提供默认实现类
- getConfig()获取配置
- addListener()添加监听器

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RxBWZDJP9JJOLlprb68yhEklcquPgR2qYia1FMsxvRbjbcMkRA4XYyFg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

6. MidnightRepositoryImpl实现从server获取配置

- meta保存服务配置源信息;
- versionMap保存server版本号，用于比较本地和远程版本信息，用于比对配置是否发生变化；
- configMap保存一个ns服务下所有配置信息；
- executor定时任务，监听服务端配置是否发生变化；
- listeners保存监听器。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RwUOLaJr8JUwVlfwEH6Tp4Abdr7AIjh8tsLuwHNe2MMurOIEP8QZRIw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

7. findAll()接口

发起http请求，从config-server获取所有配置信息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RV87MzDPV88TibuOfWFL5tSog4qicBLmh5vrZzAHamB28Rw7BiaHejsFOA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

8. heartbeat() 心跳检测

- 发起http请求，从配置中心服务端获取最新版本信息；
- 当远程版本信息大于本地版本信息时，说明配置发生了变化。
- 保存最新版本信息；
- 从配置中心重新获取所有配置信息；
- 保存最新的配置信息；
- 发布配置发生变更事件

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RtPwrbK6WqAU24iaYhMdAWIASo8amlia2YOkg7zE2S2W9oF6R0SH1Xypg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

9. onChange() 发布事件

- 更新本地最新配置
- 通过Spring发布环境变更事件。事件类EnvironmentChangeEvent来自于SpringCloud，解决微服务架构下配置热更新问题。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RzAKK9gZibxZsp4kMACU3cozSaFQCffx4iauyNRgqDKY1gUWZdv37NHHg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

10. 重新绑定配置

当EnvironmentChangeEvent事件发布之后，会重新绑定PropertySource，调用getProperty()方法获取最新配置。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RRuThw5r6r2BnfjS7KhhJxCaAs0p2DsZHiccHabNibN57BWKibNZs2NNcg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- @ConfigurationProperties 中的属性就会被更新

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RJ4ldaD1GoKCk5UAlf1v8vLibUvJCQyp8zpyjZCmEWUdEpXbQce4cvGA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

11. 测试

- 启动config-server
- 启动config-client
- 观察日志，最开始的值如下

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RibuX3pDbxnhmTNdtApV09Js7yF0rAYSrhO5fiaMSD0alSslJYrFvpkZA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 更新这两个值后，再观察日志。

发现版本发生了变化，重新获取了配置，并发布了EnvironmentChangeEvent事件。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RVgmrudMBEzWCoxTkIMRv04FOTfcjLoyJzaczeluMY7RFJGJiauQLibFw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 两个值已经被更新了

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/rw1wCRwDbgaFJqdwkNBp0u8JIOVHiaR0RyJta9fnia4AiaoYzPlqdqfs7h2mzAPgzYic9rR3T7dIlxZ2webic9noDnQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

源代码：

https://github.com/midnight2104/midnight-config/tree/v3