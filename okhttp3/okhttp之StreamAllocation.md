# okhttp之StreamAllocation 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* 背景
* 简介
* `StreamAllocation` 的成员变量
* `StreamAllocation` 的构造函数
* `StreamAllocation` 的相关方法
* 小结

---

### 背景
**HTTP** 的版本从最初的 `1.0版本`，到后续的 `1.1版本`，再到后续的 google 推出的**SPDY**，后来再推出 `2.0版本`，HTTP协议越来越完善。`okhttp也是根据2.0和1.1/1.0作为区分，实现了两种连接机制.`

**http2.0**解决了老版本`(1.1和1.0)`最重要两个问题： 
  * **连接无法复用** 
  * **head of line blocking**  [(HOL)问题](https://baike.baidu.com/item/HOL/5993141?fr=aladdin)


**http2.0** 使用 **多路复用** 的技术，多个 `stream` 可以共用一个 `socket` 连接。每个 `tcp`连接都是通过一个 `socket` 来完成的，`socket` 对应一个 `host` 和 `port`，如果有多个`stream`(即多个 [Request](https://www.jianshu.com/p/86049f8eb6ec)) 都是连接在一个 `host` 和 `port`上，那么它们就可以共同使用同一个 `socket` ,这样做的好处就是 **可以减少TCP的一个三次握手的时间**。
在`OKHttp`里面，负责连接的是 [RealConnection](https://www.jianshu.com/p/6ac651afd8fd) 。

---

###  简介
>官方注释是 `StreamAllocation`是用来协调`Connections`、`Streams`和`Calls`这三个实体的。

**HTTP通信** 执行 **网络请求**`Call`  需要在 **连接**`Connection` 上建立一个新的 **流**`Stream`，我们将 `StreamAllocation` 称之 **流** 的桥梁，它负责为一次 **请求** 寻找 **连接** 并建立 **流**，从而完成远程通信。


然后我们先来看看 `StreamAllocation` 这个类是在什么地方初始化的，以及在哪些地方用到？

选中`StreamAllocation` 的构造方法，在 `AndroidStudio` 中按 `Alt + F7`，我们发现是在 `RetryAndFollowUpInterceptor`这个拦截器`intercept(Chain chain)`中创建的.
然后往下层拦截器传递，直到 `ConnectInterceptor` 以及 `CallServerInterceptor`才继续用到。
```
public final class RetryAndFollowUpInterceptor implements Interceptor {
    ...
  @Override public Response intercept(Chain chain) throws IOException {
    ...
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    ...
}
public final class ConnectInterceptor implements Interceptor {
  ...
  @Override public Response intercept(Chain chain) throws IOException {
    ...
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
public final class CallServerInterceptor implements Interceptor {
  ...
  @Override public Response intercept(Chain chain) throws IOException {
    ...
        streamAllocation.noNewStreams();
    ...
  }
 ...
}
```

---

###  StreamAllocation 的成员变量
* 由构造方法中传入的变量：
  * `public final Address address;`  地址
  * `private final ConnectionPool connectionPool;`  连接池
  * `public final Call call;` 请求对象
  * `public final EventListener eventListener;` 事件回调
  * `private final Object callStackTrace;` 日志
  * `private final RouteSelector routeSelector;`  路由选择器
* `private RouteSelector.Selection routeSelection;` 选中的路由集合
* `private Route route;` 路由
* `private RealConnection connection;`  HTTP连接
* `private HttpCodec codec;` HTTP请求的编码和响应的解码

---


###  StreamAllocation 的构造函数
在 [RetryAndFollowUpInterceptor](https://www.jianshu.com/p/0c6324ed363e) 中传入了 ：
* [OkHttpClient](https://www.jianshu.com/p/52780b86e951) 配置的 [连接池](https://www.jianshu.com/p/f9c4458076c7).
* 根据 [Request](https://www.jianshu.com/p/86049f8eb6ec) 构建了 `Address`.
* 构建的 [RealCall](https://www.jianshu.com/p/6bcc7e6e4109) 对象.
* [OkHttpClient](https://www.jianshu.com/p/52780b86e951) 配置的 `eventListener `.
* [RealCall](https://www.jianshu.com/p/6bcc7e6e4109) 中 `captureCallStackTrace()`中配置的 `callStackTrace`
* 根据相关参数构建的 `RouteSelector`.

```
public StreamAllocation(ConnectionPool connectionPool, Address address, Call call,
                        EventListener eventListener, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.call = call;
    this.eventListener = eventListener;
    this.routeSelector = new RouteSelector(address, routeDatabase(), call, eventListener);
    this.callStackTrace = callStackTrace;
}
```

---


###  StreamAllocation 的相关方法
我们在简介中介绍 `StreamAllocation` 在哪里创建的时候，发现拦截器中主要调用的就是 `newStream(...)` 和 `noNewStreams()` 这两个方法。那我们先来看看这两个方法吧。

* `newStream(...)`
   主要就是获取一个 可用的连接 和 对应连接协议的编解码实例，并赋值给变量 `connection` 和 `codec`.
  ```
  public HttpCodec newStream(...) {
    ...
    try {
      //在连接池中找到一个可用的连接  没有则创建一个
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      //获取对应协议的 编解码类 Http1Codec or Http2Codec
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
  ```
* `noNewStreams()` 
  主要就是设置当前的 `connection` 不能再创建新的流。
  ```
  public void noNewStreams() {
    Socket socket;
    Connection releasedConnection;
    synchronized (connectionPool) {
      releasedConnection = connection;
      socket = deallocate(true, false, false);
      if (connection != null) releasedConnection = null;
    }
    closeQuietly(socket);
    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
  }
  ```

继续看下 `newStream(...)` 获取可用连接的流程。

* `findHealthyConnection(...)`
  通过`findConnection(...)`得到一个连接，如果是新创建的连接，则直接返回，否则检查 连接 是否已经可以开始承载新的流，不行则继续`findConnection(...)`.
  ```
  private RealConnection findHealthyConnection(...) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(...);
      //如果是一个新创建的连接 则直接返回
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }
      //否则检查 连接 是否已经可以开始承载新的流
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }
      return candidate;
    }
  }
  ```
* `findConnection(...)`
    * `1.0`  第一个 `synchronized (connectionPool)` 这一段首先尝试使用当前的变量`connection`. 如果当前的`connection`为 `null`, 则通过`Internal.instance.get(connectionPool, address, this, null);` 在 [连接池](https://www.jianshu.com/p/f9c4458076c7) 中获取对应 `address` 的连接 赋值给 `connection`，如果当前的 `connection`不为`null`，则直接返回.
    * `2.0`  如果上一步没有找到可用连接，则看是否有其他可用路由。  
    * `3.0`  第二个 `synchronized (connectionPool)` 这一段，`3.1`如果有其他路由则先去连接池查询看是否有对应连接， `3.2`没有的话则创建一个新的 [RealConnection](https://www.jianshu.com/p/6ac651afd8fd)，并赋值给变量 `connection`. `3.3`在连接池中找到的话则直接返回。
    *  `4.0` 然后新创建的连接开始握手连接，然后放入连接池，然后返回。
  ```
  private RealConnection findConnection(...) throws IOException {
    ...
    //1.0 尝试使用当前的 connection，或者查找 连接池
    synchronized (connectionPool) {
      ...
      releasedConnection = this.connection;
      if (this.connection != null) {
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        releasedConnection = null;
      }
      if (result == null) {
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    ...
    if (result != null) {
      return result;
    }

    //2.0 看是否有其他的路由
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }
    //3.0 
    synchronized (connectionPool) {
      if (newRouteSelection) {
        //3.1 有其他路由则先去连接池查询看是否有对应连接
      }
      if (!foundPooledConnection) {
        ...
        //3.2 没有的话则创建一个新的RealConnection，并赋值给变量 connection
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }
    // 3.3 在连接池中找到的话则直接返回
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }
    //4.0 新创建的连接开始握手连接
    result.connect(...);
    synchronized (connectionPool) {
      reportedAcquired = true;
      Internal.instance.put(connectionPool, result);
      ...
    }
    ...
    return result;
  }
  ```
  流程图大概如下：
  ![okhttp之StreamAllocation.findHealthyConnection(...)](https://upload-images.jianshu.io/upload_images/1709375-2d60006480ec479e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `acquire(...)`从连接池找到对应连接 赋值给`StreamAllocation`
  ```
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();
    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
  ```

* 获取成员变量的一些方法.
  ```
  public HttpCodec codec() {
    synchronized (connectionPool) {
      return codec;
    }
  }
  public Route route() {
    return route;
  }
  public synchronized RealConnection connection() {
    return connection;
  }
  ```

* `release()` 资源释放
  ```
  public void release() {
    Socket socket;
    Connection releasedConnection;
    synchronized (connectionPool) {
      releasedConnection = connection;
      socket = deallocate(false, true, false);
      if (connection != null) releasedConnection = null;
    }
    closeQuietly(socket);
    if (releasedConnection != null) {
      Internal.instance.timeoutExit(call, null);
      eventListener.connectionReleased(call, releasedConnection);
      eventListener.callEnd(call);
    }
  }
  ```

---


###  小结
* 介绍了`HTTP2` 和 `HTTP1` 的区别，`HTTP2`使用 **多路复用** 的技术，减少了同地址的TCP握手时间。

* 介绍了 `StreamAllocation` 的主要方法`newStream(...)`，以及其中获取可用 `Connection` 的具体逻辑.

* `newStream(...)` 返回的 `HttpCodec` 这个 编解码的示例后面也会介绍。

---

### 参考文章
* [OKHttp源码解析(九)](https://www.jianshu.com/p/6166d28983a2)

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
