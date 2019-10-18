# okhttp之Connection 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

**base on `3.12.0`**

----

### 目录
* 简介
* `RealConnection` 的成员变量
* `RealConnection` 的构造函数
* `RealConnection` 的相关方法
* 小结 

---


### 简介
`Connection` 是一个定义了四个方法的接口类。定义了 获取 **路由，socket，连接协议，以及HTTPS的TLS握手记录**。
```
public interface Connection {
  Route route();//路由
  Socket socket();//套接字
  @Nullable Handshake handshake();//https协议的TLS握手， 其他协议返回null.
  Protocol protocol();//连接的协议
}
```
`Connection` 的唯一实现类为 `RealConnection`。
当我们进行网络请求的时候， `okhttp` 会拿到 一个 `RealConnection`  来进行对应的网络连接操作。

下面我们来看下 `RealConnection`。

---


### RealConnection 的成员变量
* **静态常量：**
    ```
    /**
     * 解决Android 7.0 的一个报错
     * https://github.com/square/okhttp/issues/3245
     */
    private static final String NPE_THROW_WITH_NULL = "throw with null exception";
    /**
     * 隧道连接 connectTunnel(...) 的最大尝试次数 
     */
    private static final int MAX_TUNNEL_ATTEMPTS = 21;
    ```

* **构造方法传入的 连接池 和 路由：**
    ```
    private final ConnectionPool connectionPool; //连接池
    private final Route route; //路由
    ```

*  **仅在首次调用 `connect(...)` 初始化的变量：**
    ```
    private Socket rawSocket;//低级的 TCP socket
    private Socket socket;// socket
    private Handshake handshake;//HTTPS的TLS握手记录
    private Protocol protocol;//连接协议
    private Http2Connection http2Connection;// http2的连接
    private BufferedSource source;//类似inputstream的输入流
    private BufferedSink sink;//类似outputstream的输出流
    ```
* **由connectionPool管理 用来跟踪连接状态的变量：**
    ```
    /**
     * 为true  则无法在此连接上创建新的流
     */
    public boolean noNewStreams;
    public int successCount; //连接成功次数
    public int allocationLimit = 1; //此连接可以承载的并发流的最大数量。
    /**
     * 此连接承载的当前流
     */
    public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
    /**
     * 达到零时的纳秒时间戳
     */
    public long idleAtNanos = Long.MAX_VALUE;
    ```    


---


### RealConnection 的构造函数
构造函数只是`StreamAllocation`构建连接时调用，传入了当前的 **连接池** 和 **对应请求地址的路由**。
```
public RealConnection(ConnectionPool connectionPool, Route route) {
    this.connectionPool = connectionPool;
    this.route = route;
}
```

---


### RealConnection 的相关public方法
* `connect(...)`
    根据不同的协议建立不同的连接，首先区分 `https` 和 其他 连接通过`connectTunnel(...)`和 `connectSocket(...)` 做不同的准备 , 然后在`establishProtocol(...)`中通过判断是否是`https` 做不同的连接。
    ```
    public void connect(...) {
        //一个RealConnection只能掉一次此方法
        if (protocol != null) throw new IllegalStateException("already connected");
        ...
        while (true) {
            try {
                if (route.requiresTunnel()) {
                    //https 协议 的准备工作
                    connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
                   ...
                } else {
                    //其他通过socket 的准备工作
                    connectSocket(connectTimeout, readTimeout, call, eventListener);
                }
                //确定协议开始连接
                establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
                eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
                break;
            } catch (IOException e) {
                //处理异常之后的释放以及回调操作
                //置空 protocol 等变量
            }
        }
        ...
    }
    private void establishProtocol(...) throws IOException {
        if (route.address().sslSocketFactory() == null) {
            //非https 连接
            if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
                ...
                //通过Http2Connection.Builder(true).build().start() 建立http2连接
                startHttp2(pingIntervalMillis);
                return;
            }
            socket = rawSocket;
            protocol = Protocol.HTTP_1_1;
            return;
        }
        //https 建立安全连接
        eventListener.secureConnectStart(call);
        connectTls(connectionSpecSelector);
        eventListener.secureConnectEnd(call, handshake);

        if (protocol == Protocol.HTTP_2) {
            //通过Http2Connection.Builder(true).build().start() 建立http2连接
            startHttp2(pingIntervalMillis);
        }
    }
    ```

* `isEligible(...)`
    如果主机完全一致满足条件。 然后就只有满足对应条件的 `HTTP/2连接` 才合格。
    ```
    public boolean isEligible(Address address, @Nullable Route route) {
        // 如果此连接不接受新的流 则不合格
        if (allocations.size() >= allocationLimit || noNewStreams) return false;
        // 如果地址的非主机字段不重叠，则不合格
        if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;
        // 如果主机完全匹配，此连接可以携带地址。
        if (address.url().host().equals(this.route().address().url().host())) {
            return true; // This connection is a perfect match.
        }
        // 1.必须是HTTP/2连接.
        if (http2Connection == null) return false;
        // 2.路由必须共享一个IP地址
        if (route == null) return false;
        if (route.proxy().type() != Proxy.Type.DIRECT) return false;
        if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
        if (!this.route.socketAddress().equals(route.socketAddress())) return false;
        // 3.此连接的服务器证书必须包含新主机。
        if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
        if (!supportsUrl(address.url())) return false;
        // 4.证书固定必须与主机匹配。
        try {
            address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
        } catch (SSLPeerUnverifiedException e) {
            return false;
        }
        return true;
    }
    ```

* `supportsUrl(HttpUrl url)`
    需要与此连接的端口一致，主机名一致或者证书匹配。
    ```
    public boolean supportsUrl(HttpUrl url) {
        if (url.port() != route.address().url().port()) {
            return false;//端口不一致
        }
        if (!url.host().equals(route.address().url().host())) {
            // 主机不匹配  但是证书匹配也可以
            return handshake != null && OkHostnameVerifier.INSTANCE.verify(
                    url.host(), (X509Certificate) handshake.peerCertificates().get(0));
        }
        return true;
    }
    ```


* `newCodec(...)`
    根据是否是`HTTP_2`创建不同的 `HttpCodec`.
    ```
    public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
                              StreamAllocation streamAllocation) throws SocketException {
        if (http2Connection != null) {
            return new Http2Codec(client, chain, streamAllocation, http2Connection);
        } else {
            socket.setSoTimeout(chain.readTimeoutMillis());
            source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
            sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
            return new Http1Codec(client, streamAllocation, source, sink);
        }
    }
    ```

* `cancel()`
    关闭原始套接字，以免最终无法执行同步IO。
    ```
    public void cancel() {
        closeQuietly(rawSocket);
    }
    public static void closeQuietly(Socket socket) {
        if (socket != null) {
            try {
                socket.close();
            ...
            } catch (Exception ignored) {
            }
        }
    }
    ```

* `isHealthy(boolean doExtensiveChecks)`
    此连接是否准备好承载新的流, 主要就是判断连接是否已经中断或关闭
    ```
    public boolean isHealthy(boolean doExtensiveChecks) {
        if (socket.isClosed() || socket.isInputShutdown() || socket.isOutputShutdown()) {
            return false;
        }
        if (http2Connection != null) {
            return !http2Connection.isShutdown();
        }
        if (doExtensiveChecks) {
            try {
                int readTimeout = socket.getSoTimeout();
                try {
                    socket.setSoTimeout(1);
                    if (source.exhausted()) {
                        return false;
                    }
                    return true;
                } finally {
                    socket.setSoTimeout(readTimeout);
                }
            } catch (SocketTimeoutException ignored) {
            } catch (IOException e) {
                return false;
            }
        }
        return true;
    }
    ```


---


### 小结 

`Connection` 是 用来 获取 **路由，socket，连接协议，以及HTTPS的TLS握手记录**， 以及根据 是否是 `HTTPS` 通过不同的方式建立连接。以及提供了复用流时的一些方法。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
