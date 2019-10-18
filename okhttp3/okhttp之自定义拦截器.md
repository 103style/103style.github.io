# okhttp之自定义拦截器 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* 前言
* `LogInterceptor`实践
* `addInterceptor` 与 `addNetworkInterceptor` 的区别

---

### 前言
前面我们在 [RealCall.execute()流程介绍](https://www.jianshu.com/p/6bcc7e6e4109) 和 [okhttp之五个拦截器的介绍](https://www.jianshu.com/p/0c6324ed363e) 中介绍了拦截器的执行顺序 和 每个自带拦截器的作用。 

我们知道 我们自定义的拦截器会最先执行，在由响应结果之后也会最后处理。

没看过 [RealCall.execute()流程介绍](https://www.jianshu.com/p/6bcc7e6e4109) 和 [okhttp之五个拦截器的介绍](https://www.jianshu.com/p/0c6324ed363e) 的小伙伴可以先去看看。

官方关于拦截器的介绍 ：[戳我](https://square.github.io/okhttp/interceptors/)


---


### LogInterceptor 实践
自定义拦截器主要的逻辑就是：
* 实现`Interceptor`接口，重写 `intercept(Interceptor.Chain chain)`方法
* 调用 ` Response response = chain.proceed(request);` 传递给下一层拦截器获取他的返回结果。
如下：
```
/**
 * @author https://github.com/103style
 * @date 2019/9/10 14:15
 */
public class LogInterceptor implements Interceptor {

    @Override
    public Response intercept(Interceptor.Chain chain) throws IOException {
        //此三行代码是每个自定义拦截器中必须的
        Request request = chain.request();
        Response response = chain.proceed(request);
        return response;
    }
}
```
**`intercept(...)` 中的三行代码是每个自定义拦截器中必须的**。


通过这三行代码，我们可以获取到 请求 和 响应 的信息。然后根据具体的业务需求去做对应的操作，比如**日志打印**，**json转化**，**数据解密** 等。

当然也可以获取`Connection`和 `Call`以及以下操作 ：
```
Connection connection();
Call call();
int connectTimeoutMillis();
Chain withConnectTimeout(int timeout, TimeUnit unit);
int readTimeoutMillis();
Chain withReadTimeout(int timeout, TimeUnit unit);
int writeTimeoutMillis();
Chain withWriteTimeout(int timeout, TimeUnit unit);
```


打印日志的示例代码如下：
```
/**
 * @author https://github.com/103style
 * @date 2019/9/10 14:15
 */
public class LogInterceptor implements Interceptor {

    private static final String TAG = "LogInterceptor";

    @Override
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Request request = chain.request();

        long t1 = System.nanoTime();
        Log.d(TAG, "request = " + request.toString());
        
        Response response = chain.proceed(request);

        long t2 = System.nanoTime();
        //1e6d = 10的6的方
        Log.d(TAG, "time cost = " + (t2 - t1) / 1e6d + "ms \n response = " + response.toString());

        return response;
    }
}
```

然后通过 `OkHttpClient`配置拦截器:
```
client = new OkHttpClient.Builder()
        .addInterceptor(new LogInterceptor())
        .build();
```
or
```
client = new OkHttpClient.Builder()
        .addNetworkInterceptor(new LogInterceptor())
        .build();
```

两种方式主要的区别是 `addInterceptor` 是最先执行的拦截器， `addNetworkInterceptor`是在`ConnectInterceptor`之后执行的拦截器。 可以在 [RealCall.execute()流程介绍](https://www.jianshu.com/p/6bcc7e6e4109) 知道。

**官方的解释**：
>**`addInterceptor`**：
无需担心中间响应，例如重定向和重试。
即使从缓存提供`HTTP`响应，也总是被调用一次。
遵守应用程序的原始意图。不关心`OkHttp`注入的标头，例如`If-None-Match`。
允许短路而不是`Chain.proceed()`。
允许重试并多次调用`Chain.proceed()`。
>
>**`addNetworkInterceptor`**：
能够对诸如重定向和重试之类的中间响应进行操作。
不会在读取缓存时调用。
观察数据，就像通过网络传输数据一样。
访问`Connection`带有请求的。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
