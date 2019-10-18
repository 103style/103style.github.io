# okhttp之RealCall-execute()流程介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* **前言**
* **OkHttpClient.newCall(Request)**
* **RealCall.execute()**
* **RealInterceptorChain.proceed(request)**
* **小结**

---

### 前言
前面我们对 [OkHttpClient](https://www.jianshu.com/p/52780b86e951) 和 [Request](https://www.jianshu.com/p/86049f8eb6ec) 做了相关的介绍。

此时我们已经构建了 `http客户端` 和 `http请求`，接下来就好通过 `http客户端` 来执行`http请求`。即先通过`OkHttpClient.newCall(Request)`构建`RealCall`，然后通过 `RealCall.execute()` 来执行请求。

---


###  OkHttpClient.newCall(Request)
通过下面的源码我们知道 `OkHttpClient` 的 `newCall` 方法即通过 `RealCall.newRealCall()`构建了一个`RealCall`实例，将 `OkHttpClient` 和 `Request` 赋值给实例的成员变量. 以及初始化了拦截器 `RetryAndFollowUpInterceptor`.
```
//OkHttpClient
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false );
}
//RealCall
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
        @Override protected void timedOut() {
            cancel();
        }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
}
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
}
```

---


### RealCall.execute()
```
public Response execute() throws IOException {
    ...
    try {
        client.dispatcher().executed(this);
        Response result = getResponseWithInterceptorChain();
        ...
    } catch (IOException e) {
        ...
    } finally {
        client.dispatcher().finished(this);
    }
}
```
通过上面的代码我们知道是通过`getResponseWithInterceptorChain();`获取到请求的结果。
```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
            originalRequest, this, eventListener, client.connectTimeoutMillis(),
            client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
}
```
`getResponseWithInterceptorChain`中依次添加了以下的拦截器，后面会具体介绍：
* `client.interceptors()`：我们通过`OkhttpClient`添加的自定义拦截器
* `retryAndFollowUpInterceptor`：重试及重定向拦截器
* `BridgeInterceptor`：桥接拦截器 
* `CacheInterceptor`：缓存拦截器
* `ConnectInterceptor`：连接拦截器
* `client.networkInterceptors()`：
* `CallServerInterceptor`：读写拦截器

然后将 `request请求` 和 `interceptors`这个拦截器集合构建了一个 `RealInterceptorChain`.
然后通过`RealInterceptorChain.proceed(originalRequest);`返回请求结果。

---

### RealInterceptorChain.proceed(request)
```
public Response proceed(...) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();
    calls++;
    ...
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
            connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
            writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ...
    return response;
}
```
我们可以看到这里通过`interceptor.intercept(next);`获取的请求结果。
我们先以`RetryAndFollowUpInterceptor`来介绍下。
```
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    ...
    response = realChain.proceed(request, streamAllocation, null, null);
    ...
}
```
**看到上面代码中的`realChain.proceed(...);`方法，是不是又回到了上面的`RealInterceptorChain.proceed(request)`.**

所以由此可知`RealInterceptorChain.proceed(request)`会 "依次" 去调用拦截器列表每个`interceptors`中的`interceptor.intercept(next)`，如下图：
![RealCall.execute()](https://upload-images.jianshu.io/upload_images/1709375-dfcf0052ac424c6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

拦截器具体做了什么操作呢？请听下回分解。见 [okhttp之五个拦截器的介绍](https://www.jianshu.com/p/0c6324ed363e)


---

### 小结
通过上面的介绍，我们知道了：
* `OkHttpClient.newCall(Request)`构建了一个 `RealCall` 实例。
* `RealCall.execute()`通过添加一系列拦截器，然后依次执行拦截器的`intercept(chain)`方法，然后把响应结果再一层一层回传到给 `RealCall` 。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

