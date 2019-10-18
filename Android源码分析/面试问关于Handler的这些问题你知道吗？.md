>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


## 目录
* Handler 相关的问题 `文末参考文章中找到一些以及自己编的一些`
* Handler 相关问题的解答
* Handler 及相关源码的介绍 `base on android-28`

---

### Handler 相关的问题
* 在线程中可以直接调用 Handler 无参的构造方法吗？在主线程和子线程中有没有区别？

* Handler 机制中涉及到哪些类，各自的功能是什么？

* Handler 是可以调用哪些方法发送 Message 到 MessageQueue中？

* MessageQueue 保存 Message 的数据结构是什么样的？

* MessageQueue 添加 Message 的方式是什么样的，头插法、尾插法 还是 其它？

* Looper 是怎么从 MessageQueue 获取 Message 的？

* Looper.loop() 为什么不会阻塞APP？

* 可以监测到 MessageQueue 中无数据的情况吗？可以的话，通过什么方式？

* 在 Looper 中处理多个 Handler 的 Message 时，怎么知道 Message 是要交给哪个 Handler 来处理的？

* 线程 和 Looper 的对应关系是，一对一、一对多、多对一 还是 其它？Looper 与 MessageQueue 呢？

* Handler 发送 Message 到 MessageQueue 中，我们可以通过那些方式来监听轮到 Message 执行了呢？这几种监听都加了的话，是否都能收到回调？

* Handler 容易造成内存泄漏的原因?

* 在子线程中如何获取当前线程的 Looper?

* 如果在任意线程获取主线程的 Looper?

* 如何判断当前线程是不是主线程?

---

### Handler 相关问题的解答

`以下代码中省略了无关代码`

**Q ：在线程中可以直接调用 Handler 无参的构造方法吗？在主线程和子线程中有没有区别？**
**A**：在主线程中可以；在子线程中会抛出`RuntimeException`, 需要先调用 `Looper.prepare()`。主线程在启动的时候已经在调用过`Looper.prepare()`。
```
public Handler() {
    this(null, false);
}
public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException("...");
    }
}
```

---
 
**Q ：Handler 机制中涉及到哪些类，各自的功能是什么？**
**A**：Handler、Looper、MessageQueue、Message
**Handler**：将 `Message` 对象发送到 `MessageQueue` 中去。
**Looper**： `Message` 对象从 `MessageQueue` 中取出来，然后交给 `Handler` 去处理。
**MessageQueue**：负责 `Message` 的保存和取出。
**Message**：

---
 
**Q ：Handler 是可以调用哪些方法发送 Message 到 MessageQueue中？**
**A**：如下
```
public final boolean post(Runnable r)
public final boolean postAtTime(Runnable r, long uptimeMillis)
public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
public final boolean postDelayed(Runnable r, long delayMillis)
public final boolean postDelayed(Runnable r, Object token, long delayMillis)
public final boolean postAtFrontOfQueue(Runnable r)
public final boolean sendMessage(Message msg)
public final boolean sendEmptyMessage(int what)
public final boolean sendEmptyMessageDelayed(int what, long delayMillis)
public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis)
public final boolean sendMessageDelayed(Message msg, long delayMillis)
public boolean sendMessageAtTime(Message msg, long uptimeMillis)
public final boolean sendMessageAtFrontOfQueue(Message msg)
```

---
 
**Q：MessageQueue 保存 Message 的数据结构是什么样的？**
**A**：由 `Message` 组成的单链表。
```
public final class Message implements Parcelable {
    ...
    Message next;
    ...
}
```

---
 
**Q ：MessageQueue 添加 Message 的方式是什么样的，头插法、尾插法 还是 其它？**
**A**： 根据参数 `when` 排序，如 `when <= 0`，则添加到最前面，否则按 `when` 值进行升序排列，插入到对应的位置，如果值`(when > 0)`相等，则按添加顺序排列。
```
enqueueMessage(Message msg, long when){
    Message p = mMessages;
    if (p == null || when == 0 || when < p.when) {
        msg.next = p;
        mMessages = msg;
    } else {
        Message prev;
        for (;;) {
            prev = p;
            p = p.next;
            if (p == null || when < p.when) {
                break;
            }
        }
        msg.next = p; 
        prev.next = msg;
    }
}
```

---
 
**Q ：Looper 是怎么从 MessageQueue 获取 Message 的？**
**A**：通过 `for` 循环，调用 `MessageQueue.next()` 从 `MessageQueue` 中获取可用的 `Message `。
```
public static void loop() {
    for (;;) {
        Message msg = queue.next();
    }
}
```

---
 
**Q：Looper.loop() 为什么不会阻塞APP？** 
**A**：因为在没有可用消息的时候会休眠，然后 当 `MessageQueue` 有可用消息之后(`新增的 when<=0 的消息或者到达指定when时`)会通过 `epoll机制` 唤醒。可参考文末参考文章 [我所理解的Handler](https://www.jianshu.com/p/a3b5a5b33e0a) 中关于`Java端与Native端建立连接` 的介绍。
```
public static void loop() {
    for (;;) {
        Message msg = queue.next();
        if (msg == null) {
            return;
        }
    }
}
```

---
 
**Q：可以监测到 MessageQueue 中无数据的情况吗？可以的话，通过什么方式？** 
**A**：在 `api 23+` 可以通过执行 `handler.getLooper().getQueue().addIdleHandler()`来添加监听，`queueIdle()`中 `return false` 表示只监听一次无数据的情况，`return true` 则表示监听每次无数据的情况，
```
handler.getLooper().getQueue().addIdleHandler(
        new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                return false;
            }
        });
```

---
 
**Q：在 Looper 中处理多个 Handler 的 Message 时，怎么知道 Message 是要交给哪个 Handler 来处理的？** 
**A**：因为 `Message` 中的 `target` 变量即为对应的 `Handler`.
```
public static void loop() {
    for (;;) {
        Message msg = queue.next(); 
        try {
            msg.target.dispatchMessage(msg);
        } finally {
        }
    }
}
```

---
 
**Q：线程 和 Looper 的对应关系是，一对一、一对多、多对一 还是 其它？Looper 与 MessageQueue 呢？** 
**A**：一个线程只能有一个 `Looper`，当设置之后再调用`prepare()`会抛出`RuntimeException`； 但是一个 `Looper` 可以设置给多个线程，所以是 **多对一** 的对应关系。`Looper` 与 `MessageQueue` 则是一对一关系，`MessageQueue` 是 `Looper` 成员变量。
```
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

---
 
**Q ：Handler 发送 Message 到 MessageQueue 中，我们可以通过那些方式来监听轮到 Message 执行了呢？这几种监听都加了的话，是否都能收到回调？**
**A**：可以通过三种方式：`1`通过`Message.obtain(handler, callback)`给`Message` 设置 `callback`；`2`通过在创建 `Handler` 的时候设置 `Callback`；`3`可以重写 `Handler`的 `handleMessage(msg)` 方法。

当三个方式都设置了，也只会有一个回调，优先级按上述的`1`、`2`、`3`排列，优先`Message` 自己的 `callback`， 然后再是`Handler` 的  `Callback`，最后才是重写 `Handler` 的 `handleMessage(msg)` 方法。

```
public static void loop() {
    for (;;) {
        Message msg = queue.next(); 
        try {
            msg.target.dispatchMessage(msg);
        } finally {
        }
    }
}
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```


---
 
**Q：Handler 容易造成内存泄漏的原因?** 
**A**：`Message.target` 存有 `Handler` 的引用，以知道自身由哪一个`Handler` 来处理。因此，当 `Handler` 为非静态内部类、或持有关键对象的其它表现形式时（如`Activity` 常表现为 `Context` ），就引用了其它外部对象。当 `Message` 得不到处理时，被 `Handler` 持有的外部对象会一直处于内存泄漏状态。



---
 
**Q ：在子线程中如何获取当前线程的 Looper?**
**A**：调用静态方式 `Looper.myLooper()`，在子线程中没有调用  `Looper.prepare()`时，返回`null`.
```
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

---
 
**Q ：如果在任意线程获取主线程的 Looper?**
**A**：调用静态方式 `Looper.getMainLooper()`.
```
public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}
```

---
 
**Q ：如何判断当前线程是不是主线程?**
**A**：调用 `Looper.getMainLooper().isCurrentThread()`  或者 `Looper.getMainLooper() == Looper.myLooper()`


---

### Handler 及相关源码的介绍

**Handler的构造函数**
```
public Handler() {
    this(null, false);
}
public Handler(Callback callback) {
    this(callback, false);
}
public Handler(Looper looper) {
    this(looper, null, false);
}
public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}
public Handler(boolean async) {
    this(null, async);
}
public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException("...");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

**Looper的创建**
```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

**MessageQueue**
```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit(); // jni调用c层 以实现 epoll机制
}
```

**Message**
```
long when;
Bundle data;
Handler target;
Runnable callback;
Message next;
```

**Looper.loop() 循环遍历消息队列**
```
public static void loop() {
    final Looper me = myLooper();
    //...
    final MessageQueue queue = me.mQueue;
    for (;;) {
        //获取可以执行的消息
        Message msg = queue.next();
        if (msg == null) {
            //睡眠 等待唤醒
            return;
        }
        try {
            msg.target.dispatchMessage(msg);
        //...
        msg.recycleUnchecked();
    }
}
```

**MessageQueue.enqueueMessage(...) 保存消息到队列中**
```
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //队列会空 或 when=0 或 小于首个消息的when  添加到最前面
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            Message prev;
            //找到when对应的位置
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        //判断是否需要唤醒
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```


**MessageQueue.next() 寻找队列可用消息**
```
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        //获取NativeMessageQueue地址失败，无法正常使用epoll机制
        return null;
    }
    //用来保存注册到消息队列中的空闲消息处理器（IdleHandler）的个数
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    for (;;) {
        //...
        //检查当前线程的消息队列中是否有新的消息需要处理，尝试进入休眠
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //获取有效的消息
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }

            if (msg != null) {
                if (now < msg.when) {
                    //没达到消息需要被处理的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    mBlocked = false;
                    //返回有效消息  并从链表中删除当前消息
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                //没有更多消息，休眠时间无限
                nextPollTimeoutMillis = -1;
            }
            //...
            //获取空闲处理回调
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //进入休眠 等待唤醒
                mBlocked = true;
                continue;
            }
            //...
        }
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            //执行空闲处理
            //....
        }

        //重置变量
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```
**Handler.dispatchMessage()**
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

---


### 参考文章
[你真的懂Handler吗？Handler问答](https://www.jianshu.com/p/f70ee1765a61)
[我所理解的Handler](https://www.jianshu.com/p/a3b5a5b33e0a)

如果哪里描述错了，请评论帮忙指出，感谢！

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
