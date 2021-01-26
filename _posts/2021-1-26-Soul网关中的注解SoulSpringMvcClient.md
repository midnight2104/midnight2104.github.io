---
layout: post
title: Soul网关中的@SoulSpringMvcClient注解
tags: Soul
---

本篇文章分析的是`@SoulSpringMvcClient`注解，它的作用是：用于标记`SpringMvc`服务中的接口，被标记的接口在系统启动的是时候，将当前接口注册到`soul-admin`后台中。使用方式如下：

```java

@RestController
@RequestMapping("/order")
@SoulSpringMvcClient(path = "/order")
public class OrderController {
	//省略了其他代码

    @GetMapping("/findById")
    @SoulSpringMvcClient(path = "/findById", desc = "Find by id")
    public OrderDTO findById(@RequestParam("id") final String id) {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setId(id);
        orderDTO.setName("hello world findById");
        return orderDTO;
    }
}

```



它的定义如下：

- 注解可以使用在类上，也可以使用在方法上；
- `path`:表示接口的路径；
- `ruleName`：表示规则名称；
- `desc`：接口描述信息；
- `rpcType`：传输类型，默认是`http`，在`soul`网关中，还有`SpringCloud`，`Dubbo`，`Sofa`等类型；
- `enabled`：是否开启（是否被网关代理），默认是`true`:
- `registerMetaData`：是否注册元数据信息，默认是`false`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface SoulSpringMvcClient {
    
    /**
     * Path string.
     *
     * @return the string
     */
    String path();
    
    /**
     * Rule name string.
     *
     * @return the string
     */
    String ruleName() default "";
    
    /**
     * Desc string.
     *
     * @return String string
     */
    String desc() default "";
    
    /**
     * Rpc type string.
     *
     * @return the string
     */
    String rpcType() default "http";
    
    /**
     * Enabled boolean.
     *
     * @return the boolean
     */
    boolean enabled() default true;
    
    /**
     * Register meta data boolean.
     *
     * @return the boolean
     */
    boolean registerMetaData() default false;
}
```



注册过程在启动的时候完成，以`soul`官网中的`soul-examples-http`为例，它的案例演示请参考前面的文章。在配置文件中配置`soul-admin`后台信息:

```java
soul:
  http:
    adminUrl: http://localhost:9095 #soul-admin后台地址
    port: 8188 # soul-admin后台端口
    contextPath: /http # 当前服务的上下文路径
    appName: http # 当前服务的名称
    full: false  # 是否全部被代理，如果为true，那么这个服务的所有接口都会被代理，就不用加注解了。soul-admin端也就不需要规则，只需要一个选择器。
```

业务系统启动时，会通过`start`加载配置文件，`SpringMvcClientBeanPostProcessor`这个后置处理器也会被加载。

![](https://midnight2104.github.io/img/2021-1-26/1.png)

对`@SoulSpringMvcClient`注解的处理就在这个后置处理器中：

- 配置如果是`full=true`，就返回，表示处理所有；
- 获取当前`Bean`的`Controller`，`RestController`，`RequestMapping`，注解，判断是否包含`"*"`。如果是，就代理所有方法，注册当前`Bean`；
- 获取所有方法，注册每个含有`@SoulSpringMvcClient`注解的方法。

```java

public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

	//省略了其他代码

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        //配置是否为true
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        
        //处理类
        if (controller != null || restController != null || requestMapping != null) {
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    //注册
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            
            //遍历方法
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    //注册
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }

    //省略了其他代码
}

```

注册的逻辑就是发请求信息到`soul-admin`中，这个交给了线程池。什么时候发起请求，是线程池在调度处理，发请求的方式是通过`OkHttp`发起`post`请求。

```java
//org.dromara.soul.client.common.utils.RegisterUtils#doRegister    
public static void doRegister(final String json, final String url, final RpcTypeEnum rpcTypeEnum) {
        try {
            //通过OkHttp发起post请求
            String result = OkHttpTools.getInstance().post(url, json);
            if (AdminConstants.SUCCESS.equals(result)) {
                log.info("{} client register success: {} ", rpcTypeEnum.getName(), json);
            } else {
                log.error("{} client register error: {} ", rpcTypeEnum.getName(), json);
            }
        } catch (IOException e) {
            log.error("cannot register soul admin param, url: {}, request body: {}", url, json, e);
        }
    }
```



在`soul-admin`端有个接口来接受当前的注解配置信息，这里面的逻辑就是创建或者更新选择器或插件信息。

```java
@RestController
@RequestMapping("/soul-client")
public class SoulClientController {

    //省略了其他代码

    /**
     * Register spring mvc string.
     *
     * @param springMvcRegisterDTO the spring mvc register dto
     * @return the string
     */
    @PostMapping("/springmvc-register")
    public String registerSpringMvc(@RequestBody final SpringMvcRegisterDTO springMvcRegisterDTO) {
        return soulClientRegisterService.registerSpringMvc(springMvcRegisterDTO);
    }
}
```

通过上面的分析，就明白了`@SoulSpringMvcClient`注解的执行过程及原理。

