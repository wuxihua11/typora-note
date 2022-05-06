# 毕业设计项目开发bug解决记录

# 1.openfeign bug记录与解决

## 1.1 spring多线程中或者异步feign调用其它微服务时，拦截器会出现取不到request的情况

由于每一次请求都要获取token请求头，所以在openfeign远程调用时，编写一个拦截器每次请求时都携带一个token用于登录验证，如图：

```java
public void apply(RequestTemplate requestTemplate) {
        HttpServletRequest request = getHttpServletRequestSafely();
        String token=request.getHeader("TOKEN");
        requestTemplate.header("TOKEN",token);
    }
```

但是问题就来了，在spring多线程中或者异步feign调用其它微服务时，拦截器会出现取不到request的情况。

**报错信息如下：**

![image-20220506112642672](https://raw.githubusercontent.com/wuxihua11/pictures/master/202205061126425.png)

**场景：rabbitmq监听中，利用openfeign远程去调用其它微服务。**

### 1.1.1 问题分析

通过源码分析，我们可以看到，是从ThreadLocal中获取RequestAttributes，这就解释了为什么异步或者多线程的情况下获取不到request的原因。

![image-20220506113302358](https://raw.githubusercontent.com/wuxihua11/pictures/master/202205061133916.png)

### 1.1.2 问题解决

当request为空时，我们可以在抛出异常时自定义一个RequestAttributes，就可以正常调用其它微服务，代码如图：

```java
@Configuration
@RequiredArgsConstructor
public class FeignConfig  implements RequestInterceptor {
    private final RedisUtil redisUtil;
    private final Md5TokenGenerator md5TokenGenerator;

    @Override
    public void apply(RequestTemplate requestTemplate) {
        HttpServletRequest request = getHttpServletRequestSafely();
        String token;
        if(request!=null){
            token=request.getHeader("token");
        }
        else {
            //如果为空，重新生成token，并保存到redis
            token = md5TokenGenerator.generate(UUID.randomUUID().toString());
            redisUtil.set(token,"openFeign",60*3);
        }
        requestTemplate.header("TOKEN",token);
    }
    
    public HttpServletRequest getHttpServletRequestSafely() {
        RequestAttributes requestAttributes=getRequestAttributesSafely();
        return requestAttributes instanceof NullRequestAttributes?null:((ServletRequestAttributes)requestAttributes).getRequest();
    }

    public RequestAttributes getRequestAttributesSafely() {
        Object requestAttributes=null;
        try {
            requestAttributes = RequestContextHolder.currentRequestAttributes();
        } catch (IllegalStateException e) {
            requestAttributes=new NullRequestAttributes();
        }
        return (RequestAttributes) requestAttributes;
    }
}
```

