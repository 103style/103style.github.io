# Java Thread.join()详解 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### Java并发编程的艺术笔记
* [并发编程的挑战](https://www.jianshu.com/p/a2117af9950d)
* [Java并发机制的底层实现原理](https://www.jianshu.com/p/9d8147ba3092)
* [Java内存模型](https://www.jianshu.com/p/235c777d862c)
* [Java并发编程基础](https://www.jianshu.com/p/d50cfcaf8102)
* [Java中的锁的使用和实现介绍](https://www.jianshu.com/p/f3bb5313fce4)
* [Java并发容器和框架](https://www.jianshu.com/p/50e732981232)
* [Java中的12个原子操作类介绍](https://www.jianshu.com/p/cb64f79058c2)
* [Java中的并发工具类](https://www.jianshu.com/p/79d9baaa396e)
* [Java中的线程池](https://www.jianshu.com/p/13c82f1a7ad9)
* [Executor框架](https://www.jianshu.com/p/8933aa93ee74)

---


### 一、使用方式。

`join()`是`Thread`类的一个方法，启动线程后直接调用，例如：
```
Thread t = new AThread(); 
t.start();
t.join();
```

### 二、为什么要用join()方法

在很多情况下，主线程生成并起动了子线程，**如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束**，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是 **主线程需要等待子线程执行完成之后再结束**，这个时候就要用到`join()`方法了。

### 三、join方法的作用

在JDK的API里对于`join()`方法是：

    public final void join()
                throws InterruptedException
    Waits for this thread to die.
    An invocation of this method behaves in exactly the same   way as the invocation
        join(0)
    Throws:
        InterruptedException - if any thread has interrupted the current thread.The interrupted status of the current thread is cleared when this exception is thrown.

即`join()`的作用是：**“等待该线程终止”**，这里需要理解的就是该线程是指的主线程等待子线程的终止。也就是 **在子线程调用了join()方法后面的代码，只有等到子线程结束了才能执行**。

### 四、用实例来理解

写一个简单的例子来看一下join()的用法：
```
class BThread extends Thread {
    public BThread() {
        super("[BThread] Thread");
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + " start.");
        try {
            for (int i = 0; i < 5; i++) {
                System.out.println(threadName + " loop at " + i);
                Thread.sleep(1000);
            }
            System.out.println(threadName + " end.");
        } catch (Exception e) {
            System.out.println("Exception from " + threadName + ".run");
        }
    }
}

class AThread extends Thread {
    BThread bt;

    public AThread(BThread bt) {
        super("[AThread] Thread");
        this.bt = bt;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + " start.");
        try {
            bt.join();
            System.out.println(threadName + " end.");
        } catch (Exception e) {
            System.out.println("Exception from " + threadName + ".run");
        }
    }
}

public class TestDemo {
    public static void main(String[] args) {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + " start.");
        BThread bt = new BThread();
        AThread at = new AThread(bt);
        try {
            bt.start();
            Thread.sleep(2000);
            at.start();
            at.join();
        } catch (Exception e) {
            System.out.println("Exception from main");
        }
        System.out.println(threadName + " end!");
    }
}
```
打印结果：
```
main start. //主线程起动，因为调用了at.join()，要等到at结束了，此线程才能向下执行。 
[BThread] Thread start.
[BThread] Thread loop at 0
[BThread] Thread loop at 1
[AThread] Thread start. //线程at启动，因为调用bt.join()，等到bt结束了才向下执行。 
[BThread] Thread loop at 2
[BThread] Thread loop at 3
[BThread] Thread loop at 4
[BThread] Thread end.
[AThread] Thread end. // 线程AThread在bt.join();阻塞处起动，向下继续执行的结果 
main end! //线程AThread结束，此线程在at.join();阻塞处起动，向下继续执行的结果.   
```

修改一下代码:
```
public class TestDemo {
    public static void main(String[] args) {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + " start.");
        BThread bt = new BThread();
        AThread at = new AThread(bt);
        try {
            bt.start();
            Thread.sleep(2000);
            at.start();
//            at.join();//在此处注释掉对join()的调用
        } catch (Exception e) {
            System.out.println("Exception from main");
        }
        System.out.println(threadName + " end!");
    }
}
```
打印结果：

```
main start. // 主线程起动，因为Thread.sleep(2000)，主线程没有马上结束;
[BThread] Thread start.  //线程BThread起动
[BThread] Thread loop at 0
[BThread] Thread loop at 1
main end! // 在sleep两秒后主线程结束，AThread执行的bt.join();并不会影响到主线程。
[AThread] Thread start. //线程at起动，因为调用了bt.join()，等到bt结束了，此线程才向下执行。
[BThread] Thread loop at 2
[BThread] Thread loop at 3
[BThread] Thread loop at 4
[BThread] Thread end.  //线程BThread结束了
[AThread] Thread end. // 线程AThread在bt.join();阻塞处起动，向下继续执行的结果
```

### 五、从源码看join()方法
在AThread的run方法里，执行了`bt.join();`，进入看一下它的JDK源码：

```
public final void join() throws InterruptedException {
    join(0);
}
```

然后进入`join(long millis)`方法：
```
public final void join(long millis) throws InterruptedException {
    synchronized(lock) {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                lock.wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                lock.wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
}


private volatile long nativePeer;

/**
 * Tests if this thread is alive. A thread is alive if it has
 * been started and has not yet died.
 *
 * @return  <code>true</code> if this thread is alive;
 *          <code>false</code> otherwise.
 */
public final boolean isAlive() {
    return nativePeer != 0;
}

@FastNative
public final native void wait(long millis, int nanos) throws InterruptedException;
```

单纯从代码上看： 
* 如果线程被生成了，但还未被起动，`isAlive()`将返回`false`，调用它的join()方法是没有作用的。将直接继续向下执行。 
* 在`AThread`类中的`run`方法中，`bt.join()`是判断`bt`的`active`状态，如果`bt`的`isActive()`方法返回`false`，在`bt.join()`,这一点就不用阻塞了，可以继续向下进行了。从源码里看，`wait`方法中有参数，也就是不用唤醒谁，只是不再执行`wait`，向下继续执行而已。 
* 在`join()`方法中，对于`isAlive()`和`wait()`方法的作用对象是个比较让人困惑的问题：
`isAlive()`方法的签名是：`public final native boolean isAlive()`，也就是说`isAlive()`是判断当前线程的状态，也就是`bt`的状态。

  **wait()** 方法在`jdk`文档中的解释如下：
  >Causes the current thread to wait until another thread invokes the [`notify()`](http://tool.oschina.net/uploads/apidocs/jdk_7u4/java/lang/Object.html#notify()) method or the [`notifyAll()`](http://tool.oschina.net/uploads/apidocs/jdk_7u4/java/lang/Object.html#notifyAll()) method for this object. In other words, this method behaves exactly as if it simply performs the call `wait(0)`.
	>
	>The current thread must own this object's monitor. The thread releases ownership of this monitor and waits until another thread notifies threads waiting on this object's monitor to wake up either through a call to the `notify` method or the `notifyAll` method. The thread then waits until it can re-obtain ownership of the monitor and resumes execution.

  在这里，当前线程指的是`at`。

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
