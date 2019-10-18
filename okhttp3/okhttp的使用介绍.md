# okhttp的使用介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


---

### 目录
* 简介
* 分支介绍
* 使用示例
* 混淆配置

---

### 简介
* [github地址](https://github.com/square/okhttp)
* [官方介绍](https://square.github.io/okhttp/)

**okhttp** 的优势：
* 采用连接池技术减少
* 默认使用 `GZIP` 数据压缩格式，降低传输内容的大小
* 采用缓存避免重复的网络请求
* 支持 `SPDY`、`HTTP/2.0`，对于同一主机的请求可共享同一 `socket` 连接
* 若 `SPDY` 或 `HTTP/2.0` 不可用，还会采用连接池提高连接效率
* 网络出现问题、会自动重连（尝试连接同一主机的多个`ip`地址）
* 使用 `okio` 库简化数据的访问和存储


---

### 分支介绍

目前 **okhttp** 主要有三个分支：
* **4.2.0**：要求 `Android 5.0+ (API level 21+) and on Java 8+`。
  * 源码是用`kotlin`写的。
  * 支持 **TLS 1.3**。
  ```
  implementation("com.squareup.okhttp3:okhttp:4.2.0")
  ```
* **3.14.2**：要求 `Android 5.0+ (API level 21+) and on Java 8+`。
  * 功能同 **4.2.0** 版本，区别是源码是用`java`写的。
  ```
  implementation("com.squareup.okhttp3:okhttp:3.14.2")
  ```
* **3.12.0**：`Android 2.3+ (API level 9+) and Java 7+`.
  * 源码是用`java`写的。
  * 不支持 **TLS 1.2** ，计划在 **2020年12月31日** 前提供关键修复。
  ```
  implementation("com.squareup.okhttp3:okhttp:3.12.0")
  ```

---


### 使用示例

初始化 **OkHttpClient** 和 **ThreadPoolExecutor**：
```
private OkHttpClient client = new OkHttpClient.Builder()
        .build();

private ThreadPoolExecutor service = new ThreadPoolExecutor(0, Integer.MAX_VALUE,
        60L, TimeUnit.SECONDS,
        new SynchronousQueue<>(),
        r -> {
            Thread thread = new Thread(r);
            thread.setName(MainActivity.this.getPackageName());
            return thread;
        });
```


* **同步请求示例**：
    ```
    private void runSync() {
        service.execute(() -> {
            try {
                Request request = new Request.Builder()
                        .url("https://publicobject.com/helloworld.txt")
                        .build();

                try (Response response = client.newCall(request).execute()) {
                    if (response.body() == null) {
                        return;
                    }
                    InputStream is = response.body().source().inputStream();
                    int r;
                    while ((r = is.read()) != -1) {
                        System.out.print((char) r);
                    }
                    System.out.println();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
    ```

* **异步请求示例**：
    ```
    private void runAsync() {
        Request request = new Request.Builder()
                .url("https://publicobject.com/helloworld.txt")
                .build();

        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                System.out.println(e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (response.body() == null) {
                    System.out.println("response.body() is null");
                    return;
                }
                InputStream is = response.body().source().inputStream();
                int r;
                while ((r = is.read()) != -1) {
                    System.out.print((char) r);
                }
                System.out.println();
            }
        });
    }
    ```
* **`post`请求示例**：
    ```
    private void post(String url, String json) throws Exception {
        MediaType JSON = MediaType.get("application/json; charset=utf-8");

        RequestBody body = RequestBody.create(JSON, json);
        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .build();
        Response response = client.newCall(request).execute();
        if (response.body() != null) {
            System.out.println(response.body().string());
        }
    }
    ```
* **取消请求示例**：
    ```
    ScheduledThreadPoolExecutor service = new ScheduledThreadPoolExecutor(1,
            r -> {
                Thread thread = new Thread(r);
                thread.setName(MainActivity.this.getPackageName());
                return thread;
            });

    public void run() {
        Request request = new Request.Builder()
                .url("http://httpbin.org/delay/2")
                .build();

        final long startNanos = System.nanoTime();
        final Call call = client.newCall(request);

        // Schedule a job to cancel the call in 1 second.
        service.schedule(() -> {
            System.out.printf("%.2f Canceling call.%n", (System.nanoTime() - startNanos) / 1e9f);
            call.cancel();
            System.out.printf("%.2f Canceled call.%n", (System.nanoTime() - startNanos) / 1e9f);
        }, 1, TimeUnit.SECONDS);

        System.out.printf("%.2f Executing call.%n", (System.nanoTime() - startNanos) / 1e9f);
        service.execute(() -> {
            try (Response response = call.execute()) {
                System.out.printf("%.2f Call was expected to fail, but completed: %s%n",
                        (System.nanoTime() - startNanos) / 1e9f, response);
            } catch (IOException e) {
                System.out.printf("%.2f Call failed as expected: %s%n",
                        (System.nanoTime() - startNanos) / 1e9f, e);
            }
        });

    }
    ```

* [其他使用方式](https://square.github.io/okhttp/recipes/)
  * [修改Headers](https://square.github.io/okhttp/recipes/#accessing-headers)
  * [上传文件](https://square.github.io/okhttp/recipes/#posting-a-file)
  * [设置响应缓存](https://square.github.io/okhttp/recipes/#response-caching)
  * [更多使用方式](https://square.github.io/okhttp/recipes/)

---


### 混淆配置
```
# JSR 305 annotations are for embedding nullability information.
-dontwarn javax.annotation.**

# A resource is loaded with a relative path so the package of this class must be preserved.
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

# Animal Sniffer compileOnly dependency to ensure APIs are compatible with older versions of Java.
# okio
-dontwarn org.codehaus.mojo.animal_sniffer.*

# OkHttp platform used only on JVM and when Conscrypt dependency is available.
-dontwarn okhttp3.internal.platform.ConscryptPlatform
```

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
