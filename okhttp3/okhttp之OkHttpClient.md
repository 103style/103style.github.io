>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 简介 
`OkHttpClient`是通过 `builder` 模式来为`http`请求设置相关配置。
> 创建单个`OkHttpClient`实例并将其用于所有`HTTP`调用时，`OkHttp`的性能最佳。
这是因为每个`OkHttpClient`都拥有自己的连接池和线程池。 重用连接和线程可减少延迟并节省内存。 相反，为每个请求创建一个`OkHttpClient`会浪费空闲池上的资源。

当需要多个`OkHttpClient`时，我们可以使用`newBuilder()`自定义共享的`OkHttpClient`实例。 **这将构建共享相同连接池，线程池和配置。**
 

----

###  OkHttpClient 相关的配置方法
##### 默认配置
 ```
public Builder() {
    dispatcher = new Dispatcher();
    protocols = DEFAULT_PROTOCOLS;
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    proxySelector = ProxySelector.getDefault();
    if (proxySelector == null) {
        proxySelector = new NullProxySelector();
    }
    cookieJar = CookieJar.NO_COOKIES;
    socketFactory = SocketFactory.getDefault();
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    certificatePinner = CertificatePinner.DEFAULT;
    proxyAuthenticator = Authenticator.NONE;
    authenticator = Authenticator.NONE;
    connectionPool = new ConnectionPool();
    dns = Dns.SYSTEM;
    followSslRedirects = true;
    followRedirects = true;
    retryOnConnectionFailure = true;
    callTimeout = 0;
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
}
```


##### 超时相关的方法
* 设置`call`完成的超时时间  默认值为`0`表示无超时时间。
    * `callTimeout(long timeout, TimeUnit unit)`
    * `callTimeout(Duration duration)`
* 设置`RealConnection`的 **连接** 超时时间，默认值为 `10s`.
    * `connectTimeout(long timeout, TimeUnit unit)`
    * `connectTimeout(Duration duration)`
* 设置`RealConnection`的 **读取** 超时时间，默认值为 `10s`.
    * `readTimeout(long timeout, TimeUnit unit)`
    * `readTimeout(Duration duration)`
* 设置`RealConnection`的 **写入** 超时时间，默认值为 `10s`.
    * `writeTimeout(long timeout, TimeUnit unit)`
    * `writeTimeout(Duration duration)`
* 设置 `HTTP/2` 和`web socket` `ping`之间的间隔。默认 `0` 表示客户端禁用客户端启动的`ping`.
    * `pingInterval(long interval, TimeUnit unit)`
    * `pingInterval(Duration duration)`

##### 配置Http连接的代理
`proxy` 优先与 `proxySelector`.
* `proxy(@Nullable Proxy proxy)`
    禁用代理可使用 `Proxy.NO_PROXY`.

* `proxySelector(ProxySelector proxySelector)`
    设置未指定`proxy`时的代理策略

#####  设置执行异步请求的策略
* `dispatcher(Dispatcher dispatcher)`

##### 添加和获取可修改的拦截器
* `interceptors()`
    获取可修改的拦截器列表

* `addInterceptor(Interceptor interceptor)`
    添加自定义拦截器

* `networkInterceptors()`
    返回可观察到单个网络请求和响应的拦截器的可修改列表。这些拦截器必须只调用一次。

* `addNetworkInterceptor(Interceptor interceptor)`
    添加网络拦截器

##### 事件响应回调监听
* `eventListener(EventListener eventListener)`
* `eventListenerFactory(EventListener.Factory eventListenerFactory)`

  监听请求相关的回调，如下图：
![event](https://upload-images.jianshu.io/upload_images/1709375-123553fab64ed259.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####  重定向 
* `followSslRedirects(boolean followProtocolRedirects)`
* `followRedirects(boolean followRedirects)`
    是否可以重定向，默认可以。

#####  socket相关
* `socketFactory(SocketFactory socketFactory)`
* `sslSocketFactory(SSLSocketFactory sslSocketFactory)`
* `sslSocketFactory(SSLSocketFactory sslSocketFactory, X509TrustManager trustManager)`
    设置相关的socket工厂。

#####  证书认证相关
* `hostnameVerifier(HostnameVerifier hostnameVerifier)`
    设置用于确认响应证书是否适用于 `HTTPS` 连接的请求主机名。
* `certificatePinner(CertificatePinner certificatePinner)`
    设置证书固定器，以限制受信任的证书。
* `authenticator(Authenticator authenticator)`
    设置用于响应原始服务器质询的身份验证器
* `proxyAuthenticator(Authenticator proxyAuthenticator)`
    设置用于响应代理服务器质询的身份验证器

#####  其他
* `retryOnConnectionFailure(boolean retryOnConnectionFailure)`
    配置客户端连接出现问题时，是否重连。默认重连。
* `cookieJar(CookieJar cookieJar)`
    设置可以接受来自传入`HTTP`响应的`cookie`的处理程序，并提供`cookie`传出HTTP请求。
* `cache(@Nullable Cache cache)`
    设置响应缓存以用于读取和写入缓存的响应。
* `dns(Dns dns)`
    设置用于查找主机名 `IP` 地址的 `DNS` 服务。
* `connectionPool(ConnectionPool connectionPool)`
    设置 `HTTP`、`HTTPS` 连接的连接池。
* `protocols(List<Protocol> protocols)`
    配置此客户端用于与远程服务器通信的协议


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
