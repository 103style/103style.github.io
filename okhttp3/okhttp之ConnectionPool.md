>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

**base on `3.12.0`**

----

### 目录
* 简介
* `ConnectionPool` 的成员变量
* `ConnectionPool` 的构造函数
* `ConnectionPool` 的相关方法
* 小结

---


### 简介
`ConnectionPool` 即连接池，用来管理  `HTTP` 和 `HTTP/2` 连接的重用，以减少网络延迟。 

相同的 `HTTP` 请求可以共用一个连接`(RealConnection)`， `ConnectionPool` 实现了将哪些连接保持打开状态以备将来使用的策略。

即  `ConnectionPool` 是用来管理 `RealConnection` 用的。

---


### ConnectionPool 的成员变量
* 用于清除过期的连接，每个连接池最多只能运行一个线程。
  线程池执行程序允许对池本身进行垃圾回收。  
  ```
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
  ```
* 最大空闲连接数，在`cleanup(...)`清理的时候用到
  ```
  private final int maxIdleConnections;
  ```
* 每个空闲连接的存活时间的纳秒数
  ```
  private final long keepAliveDurationNs;
  ```
* 清除过期连接的任务
  ```
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
  ```
* 保存可复用连接的队列
  ```
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  ```
* 连接地址要避免的失败路由黑名单
  ```
  final RouteDatabase routeDatabase = new RouteDatabase();
  ```
* 是否是清理过期连接的标记位
  ```
  boolean cleanupRunning;
  ```

---

### ConnectionPool 的构造函数
默认配置的 每个地址的空闲连接数为 **5个**，每个空闲连接的存活时间为 **5分钟**.
```
public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
}
public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    if (keepAliveDuration <= 0) {
        throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
}
```

---

### ConnectionPool 的相关方法
##### 公开方法
* `public synchronized int idleConnectionCount()`
  获取当前空闲连接的个数
  ```
  public synchronized int idleConnectionCount() {
    int total = 0;
    for (RealConnection connection : connections) {
      if (connection.allocations.isEmpty()) total++;
    }
    return total;
  }
  ```

* `public synchronized int connectionCount()`
  获取当前连接的个数
  ```
  public synchronized int connectionCount() {
    return connections.size();
  }
  ```

* `public void evictAll()`
  关闭和移除连接池中所有的空闲连接
  ```
  public void evictAll() {
    List<RealConnection> evictedConnections = new ArrayList<>();
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();
        if (connection.allocations.isEmpty()) {
          connection.noNewStreams = true;
          evictedConnections.add(connection);
          i.remove();
        }
      }
    }
    for (RealConnection connection : evictedConnections) {
      closeQuietly(connection.socket());
    }
  }
  ```

##### 私有方法
* `RealConnection get(Address address, StreamAllocation streamAllocation, Route route)`
  获取可复用的连接，没有则返回`null`.
  ```
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
  ```

* `Socket deduplicate(Address address, StreamAllocation streamAllocation)`
  如果可能，将`streamAllocation`保留的连接替换为共享连接。
  ```
  @Nullable Socket deduplicate(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, null)
          && connection.isMultiplexed()
          && connection != streamAllocation.connection()) {
        return streamAllocation.releaseAndAcquire(connection);
      }
    }
    return null;
  }
  ```

* `void put(RealConnection connection)`
   先清除过期的连接，再将可复用的连接保存到队列`connections`中
  ```
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
  ```

* `boolean connectionBecameIdle(RealConnection connection)`
    告诉连接池，`connection`已变为空闲连接。
    如果配置了 `RealConnection.noNewStreams= true 不允许复用` 或者 `maxIdleConnections==0 不允许有空闲连接`，则直接从队列中删除该连接。
  ```
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (connection.noNewStreams || maxIdleConnections == 0) {
      connections.remove(connection);
      return true;
    } else {
      notifyAll(); 
      return false;
    }
  }
  ```

* `long cleanup(long now)`
    清除过期的空闲连接。
    * `1.0` 循环遍历`connections` 主要是寻找队列中  正在使用的连接`inUseConnectionCount`、空闲连接的个数`idleConnectionCount`、空闲等待最久的连接`longestIdleConnection `
    * `2.0` 空闲等待最久的连接等待时间超过了`keepAliveDurationNs`，或者 空闲连总数大于了允许的最大空闲连接数`maxIdleConnections`，则从队列中移除当前连接，并关闭，然后`cleanupRunnable`继续执行 `cleanup(long now)`.
    * `3.0` 如果有空闲的连接，则`cleanupRunnable`等待时间为`keepAliveDurationNs - longestIdleDurationNs` 
    * `4.0` 如果连接都在使用中，则`cleanupRunnable`等待时间为`keepAliveDurationNs` 
    * `5.0` 否则`cleanupRunnable`任务完成
  ```
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;
    synchronized (this) {
      // 1.0
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
        idleConnectionCount++;
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) { //2.0
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {//3.0
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {//4.0
        return keepAliveDurationNs;
      } else {//5.0
        cleanupRunning = false;
        return -1;
      }
    }
    closeQuietly(longestIdleConnection.socket());//关闭连接
    return 0;
  }
  ```

---

### 小结
* 连接池`ConnectionPool`是用来管理  `HTTP` 和 `HTTP/2` 连接的重用，以减少网络延迟。 

* 连接池默认时每个地址的空闲连接数为 **5个**，每个空闲连接的存活时间为 **5分钟**.

* 连接池每次添加一个新的连接时，都会先清理当前连接池中过期的连接，通过 清理线程池`executor` 执行清理任务`cleanupRunnable`。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
