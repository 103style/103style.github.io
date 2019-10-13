>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* 简介
* `Dispatcher`成员变量介绍
* `Dispatcher`构造方法介绍
* `Dispatcher`主要方法介绍
* 小结

---

### 简介
首先我们来介绍下 `Dispatcher`，官方描述是这样的：
>Policy on when async requests are executed.
   执行异步请求时的策略

所以`Dispatcher`是我们进行异步请求是 `okhttp` 给我们提供的 **执行异步请求时的策略**.
```
public final class Dispatcher {...}
```
我们可以看到 `Dispatcher` 类是由 `final`  修饰的，代表它不能被继承。


我们先来看下 `Dispatcher`是什么时候设置的。
通过查看下面 `OkHttpClient` 的代码，我们知道在我们创建 `OkHttpClient` 的时候，如果我们没有通过 `builder.dispatcher(Dispatcher dispatcher)` 修改 `Dispatcher`的配置的话，默认的 `dispatcher` 就是 默认配置的 `Dispatcher`类。
```
public class OkHttpClient implements ... {
    ...
    public Builder newBuilder() {
        return new Builder(this);
    }
    public static final class Builder {
        Dispatcher dispatcher;
        ...
        public Builder() {
            dispatcher = new Dispatcher();
            ...
        }
        public Builder dispatcher(Dispatcher dispatcher) {
            if (dispatcher == null) throw new IllegalArgumentException("dispatcher == null");
            this.dispatcher = dispatcher;
            return this;
        }
    }
}
```

---

### Dispatcher成员变量介绍
* `int maxRequests = 64;`
    默认同时执行的最大请求数， 可以通过`setMaxRequests(int)`修改.

* `int maxRequestsPerHost = 5;`
    每个主机默认请求的最大数目， 可以通过`setMaxRequestsPerHost(int)`修改.

* `private @Nullable Runnable idleCallback;`
    调度没有请求任务时的回调.

* `ExecutorService executorService;`
    执行异步请求的线程池，**默认是 核心线程为0，最大线程数为`Integer.MAX_VALUE`，空闲等待为60s**.

* `Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();`
    异步请求的执行顺序的队列.

* `Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();`
    运行中的异步请求队列.

* `Deque<RealCall> runningSyncCalls = new ArrayDeque<>();`
    运行中的同步请求队列.

---


### Dispatcher构造函数
```
public Dispatcher(ExecutorService executorService) {
  this.executorService = executorService;
}
public Dispatcher() {}
```
可以自己设置执行任务的线程池。

---

### Dispatcher主要方法介绍
*  **配置和获取 同时执行请求的最大任务数、同主机允许同时执行的最大任务数**
    ```
    public void setMaxRequests(int maxRequests) {...}
    public synchronized int getMaxRequests() {...}
    public void setMaxRequestsPerHost(int maxRequestsPerHost) {...}
    public synchronized int getMaxRequestsPerHost() {...}
    ```
* **设置没有请求任务时的回调**
    ```
    public synchronized void setIdleCallback(@Nullable Runnable idleCallback) {
        this.idleCallback = idleCallback;
    }
    ```
* **添加到请求队列**
    ```
    void enqueue(AsyncCall call) {
        synchronized (this) {
            readyAsyncCalls.add(call);
        }
        promoteAndExecute();
    }
    synchronized void executed(RealCall call) {
        runningSyncCalls.add(call);
    }
    ```
* **执行等待队列中的请求任务**
    ```
    private boolean promoteAndExecute() {
        assert (!Thread.holdsLock(this));
        List<AsyncCall> executableCalls = new ArrayList<>();
        boolean isRunning;
        synchronized (this) {
            //获取等待中的任务队列
            for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
                AsyncCall asyncCall = i.next();
                // 超过可以同时运行的最大请求任务数
                if (runningAsyncCalls.size() >= maxRequests) break; 
                // 超过同一主机同时运行的最大请求任务数
                if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; 
                i.remove();
                executableCalls.add(asyncCall);
                runningAsyncCalls.add(asyncCall);
            }
            isRunning = runningCallsCount() > 0;
        }
        for (int i = 0, size = executableCalls.size(); i < size; i++) {
            AsyncCall asyncCall = executableCalls.get(i);
            asyncCall.executeOn(executorService());
        }
        return isRunning;
    }
    ```
* **获取执行任务的线程池.**
    如果没有通过构造方法`Dispatcher(ExecutorService executorService)` 设置线程池的话，默认就是 核心线程为0，最大线程数为`Integer.MAX_VALUE`，空闲等待为60s，用`SynchronousQueue`保存等待任务 的线程池。
    ```
    public synchronized ExecutorService executorService() {
        if (executorService == null) {
            executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
        }
        return executorService;
    }
    ```
* **获取和当前请求的host一样的运行中的请求个数**.
    ```
    private int runningCallsForHost(AsyncCall call) {
        int result = 0;
        for (AsyncCall c : runningAsyncCalls) {
            if (c.get().forWebSocket) continue;
            if (c.host().equals(call.host())) result++;
        }
        return result;
    }
    ```
* **取消所有请求任务**
    ```
    public synchronized void cancelAll() {
        for (AsyncCall call : readyAsyncCalls) {
            call.get().cancel();
        }
        for (AsyncCall call : runningAsyncCalls) {
            call.get().cancel();
        }
        for (RealCall call : runningSyncCalls) {
            call.cancel();
        }
    }
    ```
* **结束请求任务**
    ```
    void finished(AsyncCall call) {}
    void finished(RealCall call) {}
    private <T> void finished(Deque<T> calls, T call) {
        Runnable idleCallback;
        synchronized (this) {
            if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
            idleCallback = this.idleCallback;
        }
        boolean isRunning = promoteAndExecute();

        if (!isRunning && idleCallback != null) {
            idleCallback.run();
        }
    }
    ```
* **获取等待中、执行中的请求任务**
    ```
    public synchronized List<Call> queuedCalls() {
        List<Call> result = new ArrayList<>();
        for (AsyncCall asyncCall : readyAsyncCalls) {
            result.add(asyncCall.get());
        }
        return Collections.unmodifiableList(result);
    }
    public synchronized List<Call> runningCalls() {
        List<Call> result = new ArrayList<>();
        result.addAll(runningSyncCalls);
        for (AsyncCall asyncCall : runningAsyncCalls) {
            result.add(asyncCall.get());
        }
        return Collections.unmodifiableList(result);
    }
    public synchronized int queuedCallsCount() {
        return readyAsyncCalls.size();
    }
    public synchronized int runningCallsCount() {
        return runningAsyncCalls.size() + runningSyncCalls.size();
    }
    ```

---

### 小结
`Dispatcher`是我们进行异步请求是 `okhttp` 给我们提供的 **执行异步请求时的策略**。

`Dispatcher`因为是`final`修饰的类，所以我们我能继承它，但是我们可以通过创建一个`Dispatcher`对象，然后修改 **执行任务的线程池**、 **最大并发数**、 **同主机最大并发数** 等。

执行任务的方法是`promoteAndExecute()`.


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
