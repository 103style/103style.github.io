>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* **前言**
* **重试及重定向拦截器** `RetryAndFollowUpInterceptor`
* **桥接拦截器** `BridgeInterceptor`
* **缓存拦截器** `CacheInterceptor`
* **连接拦截器** `ConnectInterceptor`
* **读写拦截器** `CallServerInterceptor`
* **小结**

---


###  前言
前面我们对 [okhttp的RealCall.execute()流程做了介绍](https://www.jianshu.com/p/6bcc7e6e4109)，在文末有提到五个自带的拦截器，本文就来介绍五个拦截器主要的工作。

还没看过 [RealCall.execute()流程](https://www.jianshu.com/p/6bcc7e6e4109) 的小伙伴可以先去看看。

**这里先简单介绍下五个拦截器的作用**：
* `RetryAndFollowUpInterceptor`：负责请求的重试和重定向
* `BridgeInterceptor`：给请求添加对应的 `header` 信息，处理响应结果的 `header` 信息
* `CacheInterceptor`：根据当前获取的状态选择 网络请求 、读取缓存、更新缓存。
* `ConnectInterceptor`：建立 `http` 连接。
* `CallServerInterceptor`：读写网络数据。

话不多说，我们接下来直接看这写拦截器的 `intercept(chain)` 主要做了什么事情。

---

###  重试及重定向拦截器 RetryAndFollowUpInterceptor

首先介绍下`RetryAndFollowUpInterceptor` 主要做了哪些逻辑：
*  **首先获取以及初始化相关的实例.**

* **获取下层拦截器返回的结果，出现异常 则根据是否可以恢复来判断中断 还是 重新开始循环.**

* **根据返回的信息 判断是否需要重定向？**
  当重定向次数大于 `MAX_FOLLOW_UPS = 20` 时则抛出异常.

* **然后判断重定向返回的信息是否出现异常。**
   * 出现则抛出异常并释放资源.
   * 不出现则用重定向返回的信息构建 `request`重新传给下层拦截器.

>下面我们来结合具体代码看看.

* **获取以及初始化相关的实例**：
    我们可以看到这里主要是获取 `RealInterceptorChain`、`RealCall`对象，然后构建了一个`StreamAllocation`，然后进入死循环执行下面的逻辑。
    `StreamAllocation`主要是 **用来建立执行HTTP请求所需网络设施的组件**，后面我们会详细介绍。
    ```
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation =streamAllocation;
    //然后开始进入死循环
    while (true) {...}
    ```
* **获取并处理下层拦截器返回的结果**
    这里主要是获取下层拦截器返回的结果，然后判断是否可以重试。
    ```
    Response response;
    boolean releaseConnection = true;
    try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
    } catch (RouteException e) {
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
            throw e.getFirstConnectException();
        }
        releaseConnection = false;
        continue;
    } catch (IOException e) {
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
    } finally {
        if (releaseConnection) {
            //不能重试则释放资源
            streamAllocation.streamFailed(null);
            streamAllocation.release();
        }
    }
    ```
    判断是否可以重试的逻辑：
    ```
    private boolean recover(IOException e, StreamAllocation streamAllocation,
                            boolean requestSendStarted, Request userRequest) {
        streamAllocation.streamFailed(e);
        //client.retryOnConnectionFailure(false) 配置了不允许重试
        if (!client.retryOnConnectionFailure()) return false;
        //无法发送请求内容
        if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;
        //异常是致命的
        if (!isRecoverable(e, requestSendStarted)) return false;
        //没有可重试的路径
        if (!streamAllocation.hasMoreRoutes()) return false;

        //上面都不满足 则可以重试
        return true;
    }
    ```

* **根据返回的信息判断是否需要重定向**
    ```
    Request followUp;
    try {
        followUp = followUpRequest(response, streamAllocation.route());
    } catch (IOException e) {
        streamAllocation.release();
        throw e;
    }
    if (followUp == null) {
        streamAllocation.release();
        return response;
    }
    ```
    `followUpRequest`：
    主要是根据返回信息的响应码做对应的操作。
    ```
    private Request followUpRequest(Response userResponse, Route route) throws IOException {
        if (userResponse == null) throw new IllegalStateException();
        int responseCode = userResponse.code();
        final String method = userResponse.request().method();
        switch (responseCode) {
            case HTTP_PROXY_AUTH://407  需要代理验证
                return client.proxyAuthenticator().authenticate(route, userResponse);
            case HTTP_UNAUTHORIZED://401  没有认证
                return client.authenticator().authenticate(route, userResponse);
            case HTTP_PERM_REDIRECT://308
            case HTTP_TEMP_REDIRECT://307
                if (!method.equals("GET") && !method.equals("HEAD")) {
                    return null;
                }
            case HTTP_MULT_CHOICE://300
            case HTTP_MOVED_PERM://301
            case HTTP_MOVED_TEMP://302
            case HTTP_SEE_OTHER://303
                //client不允许重定向
                if (!client.followRedirects()) return null;
                boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
                if (!sameScheme && !client.followSslRedirects()) return null;

                //根据相应的状态修改request的header信息
                //...
                return requestBuilder.url(url).build();
            case HTTP_CLIENT_TIMEOUT://408
                ...
            case HTTP_UNAVAILABLE://503
                ...
            default:
                return null;
        }
    }
    ```

* **然后判断重定向返回的信息是否出现异常。**
    ```
    //关闭之前响应数据的流信息
    closeQuietly(response.body());
    //超过重定向次数
    if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }
    //不能重用的请求体
    if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
    }
    //跨主机导致连接地址不一样
    if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
                createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
    } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
                + " didn't close its backing stream. Bad interceptor?");
    }
    //将重定向的请求赋值给request 
    request = followUp;
    priorResponse = response;
    ```

---

###  桥接拦截器 BridgeInterceptor
`BridgeInterceptor` 的主要做了以下工作：
* 给 `http请求` 添加对应的 `header` 信息.
* 如果下层拦截器返回的数据的 `Content-Encoding` 是 `gzip`，则通过 `GzipSource` 获取返回的数据。

>废话不多说，看代码：

* **给http请求添加对应的header信息**.
    * `RequestBody`不为空则根据内容添加`Content-Type`、`Content-Length`、`Transfer-Encoding`.
    * 给没有`Host`、`Connection`、`User-Agent`头信息的 `request` 补上.
    * 根据对应条件给 `request` 补上`Accept-Encoding`、`Cookie`.
    ```
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    RequestBody body = userRequest.body();
    if (body != null) {
        MediaType contentType = body.contentType();
        if (contentType != null) {
            requestBuilder.header("Content-Type", contentType.toString());
        }
        long contentLength = body.contentLength();
        if (contentLength != -1) {
            requestBuilder.header("Content-Length", Long.toString(contentLength));
            requestBuilder.removeHeader("Transfer-Encoding");
        } else {
            requestBuilder.header("Transfer-Encoding", "chunked");
            requestBuilder.removeHeader("Content-Length");
        }
    }

    if (userRequest.header("Host") == null) {
        requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    if (userRequest.header("Connection") == null) {
        requestBuilder.header("Connection", "Keep-Alive");
    }

    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
        transparentGzip = true;
        requestBuilder.header("Accept-Encoding", "gzip");
    }
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
        requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
        requestBuilder.header("User-Agent", Version.userAgent());
    }
    ```
* **处理gzip格式数据**、
    * 主要就是通过`GzipSource` 包装 `gzip` 格式的数据，具体可以看`GzipSource.read(...)` 。
    ```
    Response networkResponse = chain.proceed(requestBuilder.build());
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
    Response.Builder responseBuilder = networkResponse.newBuilder()
            .request(userRequest);
    if (transparentGzip
            && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
            && HttpHeaders.hasBody(networkResponse)) {
        GzipSource responseBody = new GzipSource(networkResponse.body().source());
        Headers strippedHeaders = networkResponse.headers().newBuilder()
                .removeAll("Content-Encoding")
                .removeAll("Content-Length")
                .build();
        responseBuilder.headers(strippedHeaders);
        String contentType = networkResponse.header("Content-Type");
        responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }
    return responseBuilder.build();
    ```

---

###  缓存拦截器 CacheInterceptor
`CacheInterceptor` 的作用主要是：
 * **根据当前时间获取当前request的缓存**。
 * **根据缓存中缓存的request和response做对应处理**。
 * **有缓存时，则根据条件判断是否缓存到本地**。

>接下来看代码实现：

* **根据当前时间获取当前request的缓存**
    ```
    Response cacheCandidate = cache != null
            ? cache.get(chain.request())
            : null;
    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    if (cache != null) {
        cache.trackResponse(strategy);
    }
    if (cacheCandidate != null && cacheResponse == null) {
        closeQuietly(cacheCandidate.body());
    }
    ```

* **根据缓存中缓存的request和 response做对应处理**
    * 网络不可用，又没有缓存则返回 `504`错误.
    * 网络不可用，缓存可用，则直接返回缓存。
    * 网络可用，缓存不可用，则通过下层拦截器获取网络数据。
    ```
    //网络不可用，又没有缓存则返回 `504`错误.
    if (networkRequest == null && cacheResponse == null) {
        return new Response.Builder()
                .request(chain.request())
                .protocol(Protocol.HTTP_1_1)
                .code(504)
                .message("Unsatisfiable Request (only-if-cached)")
                .body(Util.EMPTY_RESPONSE)
                .sentRequestAtMillis(-1L)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();
    }
    //网络不可用，缓存可用，则直接返回缓存
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .build();
    }
    Response networkResponse = null;
    try {
        //请求网络数据
        networkResponse = chain.proceed(networkRequest);
    } finally {
        if (networkResponse == null && cacheCandidate != null) {
            closeQuietly(cacheCandidate.body());
        }
    }
    ```

* **缓存不为空时，根据条件是否缓存到本地**
    * 如果请求返回码是`HTTP_NOT_MODIFIED: 304`，则更新缓存。
    * 是 `HEAD` 类型的请求，则更新缓存。
    * 是`POST`、`PATCH`、`PUT`、`DELETE`、`MOVE`请求则删除缓存中的`request`。
    ```
    if (cacheResponse != null) {
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {
            //304
            Response response = cacheResponse.newBuilder()
                    .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                    .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                    .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                    .cacheResponse(stripBody(cacheResponse))
                    .networkResponse(stripBody(networkResponse))
                    .build();
            networkResponse.body().close();
            cache.trackConditionalCacheHit();
            cache.update(cacheResponse, response);
            return response;
        } else {
            closeQuietly(cacheResponse.body());
        }
    }
    Response response = networkResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
    if (cache != null) {
        if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
            // HEAD 请求
            CacheRequest cacheRequest = cache.put(response);
            return cacheWritingResponse(cacheRequest, response);
        }
        if (HttpMethod.invalidatesCache(networkRequest.method())) {
            //POST、PATCH、PUT、DELETE、MOVE
            try {
                cache.remove(networkRequest);
            } catch (IOException ignored) {
            }
        }
    }

    return response;
    ```

---

###  连接拦截器 ConnectInterceptor
`ConnectInterceptor`主要就是通过`StreamAllocation`、`HttpCodec` 和 `RealConnection` 合理建立 `http`连接。

这三个类后面会单独讲解，主要就是通过 **在连接池中寻找可以的连接，没有则创建，并通过`okio`来操作数据流，然后由`CallServerInterceptor`继续处理**。
```
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

---

###  读写拦截器 CallServerInterceptor
`CallServerInterceptor`即为获取请求的响应数据，并回传给上一层拦截器。
* 处理带有 `RequestBody` 并符合条件的 `request`。
* 然后通过`Response.Builder`构建响应数据，并根据相应数据的返回码做响应处理。

>开始看代码

* **处理带有RequestBody并符合条件的request**
    处理带有`RequestBody`的非 `GET` 和 `HEAD` 请求。
    * 当`header`的 `Expect` 为 `100-continue`时，则为`Response.Builder`添加头信息
    * 如果服务器允许发送请求`body`发送，则通过`okio`写入请求数据.
    ```
    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        //等待HTTP/1.1 100响应
        if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
            httpCodec.flushRequest();
            realChain.eventListener().responseHeadersStart(realChain.call());
            responseBuilder = httpCodec.readResponseHeaders(true);
        }
        if (responseBuilder == null) {
            //如果响应满足 HTTP/1.1 100
            realChain.eventListener().requestBodyStart(realChain.call());
            long contentLength = request.body().contentLength();
            CountingSink requestBodyOut =
                    new CountingSink(httpCodec.createRequestBody(request, contentLength));
            BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

            request.body().writeTo(bufferedRequestBody);
            bufferedRequestBody.close();
            realChain.eventListener()
                    .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
        } else if (!connection.isMultiplexed()) {
            //不满足HTTP/1.1 100 则防止连接被重用
            streamAllocation.noNewStreams();
        }
    }
    //结束请求
    httpCodec.finishRequest();
    ```

* **构建并处理响应数据**
    ```
    if (responseBuilder == null) {
        realChain.eventListener().responseHeadersStart(realChain.call());
        //读取响应头信息
        responseBuilder = httpCodec.readResponseHeaders(false);
    }
    Response response = responseBuilder
            .request(request)
            .handshake(streamAllocation.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();

    int code = response.code();
    if (code == 100) {
        //服务器返回100 则重新请求
        responseBuilder = httpCodec.readResponseHeaders(false);
        response = responseBuilder
                .request(request)
                .handshake(streamAllocation.connection().handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();

        code = response.code();
    }
    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);
    if (forWebSocket && code == 101) {
        response = response.newBuilder()
                .body(Util.EMPTY_RESPONSE)
                .build();
    } else {
        response = response.newBuilder()
                .body(httpCodec.openResponseBody(response))
                .build();
    }
    //Connection 为 close 则禁止创建新的流
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
            || "close".equalsIgnoreCase(response.header("Connection"))) {
        streamAllocation.noNewStreams();
    }
     //204 205 的请求返回Content-Length 必须为 0
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
        throw new ProtocolException(
                "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
    return response;
    ```


---

###  小结
通过上面的介绍我们基本了解了五个拦截器对应的作用，这里再回顾下：
* `RetryAndFollowUpInterceptor`：负责请求的重试和重定向
* `BridgeInterceptor`：给请求添加对应的 `header` 信息，处理响应结果的 `header` 信息
* `CacheInterceptor`：根据当前获取的状态选择 网络请求 、读取缓存、更新缓存。
* `ConnectInterceptor`：建立 `http` 连接。
* `CallServerInterceptor`：读写网络数据。

每个拦截器的都有自己负责的功能。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

