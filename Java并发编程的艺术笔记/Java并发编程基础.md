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



### 目录
* 线程的简介
* 启动和终止线程
* 线程间通信
* 小结

---

### 线程的简介
* **[什么是线程](https://baike.baidu.com/item/%E7%BA%BF%E7%A8%8B/103101?fr=aladdin)**
    线程是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

* **为什么要使用多线程**
    目前的处理器核心越来越多，使用多线程能有更快的响应时间，并能有更好的编程模型。

* **线程优先级**
    现代操作系统基本采用时分的形式调度运行的线程，操作系统分出每一个时间片会根据线程的优先级来分配，优先级越高的最先获取执行资源。

    在Java线程中，通过一个整型成员变量`priority`来控制优先级，优先级的范围从`1~10`，在线程构建的时候可以通过`setPriority(int)`方法来修改优先级，默认优先级是`5`，优先级高的线程分配时间片的数量要多于优先级低的线程。

    线程优先级的设置:
    * **频繁阻塞**（休眠或者I/O操作）的线程需要设置 **较高优先级**，
    * **偏重计算**（需要较多CPU时间或者偏运算）的线程则设置 **较低的优先级**，确保处理器不会被独占。

    在不同的 **JVM** 以及 **操作系统** 上，线程规划会存在差异，有些操作系统甚至会忽略对线程优先级的设定。

  **线程优先级不能作为程序正确性的依赖**，因为操作系统可以完全不用理会 `Java` 线程对于优先级的设定。

* **线程的状态**
  * `NEW`  初始状态
  * `RUNNABLE` 运行状态
  * `BLOCKED` 阻塞状态
  * `WAITING` 等待状态
  * `TIME_WAITING` 超时等待状态
  * `TERMINATED` 终止状态

下图是状态变化的介绍：
![Java线程状态变迁](https://upload-images.jianshu.io/upload_images/1709375-ccfe7bc796ff7067.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **Daemon线程**
  **Daemon** 线程是一种支持型线程，因为它主要被用作程序中后台调度以及支持性工作。
  这意味着，**当一个Java虚拟机中不存在非`Daemon`线程的时候，Java虚拟机将会退出**（`Daemon`线程不一定会执行完）。
  可以通过调用`Thread.setDaemon(true)`将线程设置为`Daemon`线程。需在启动之前设置。

---

### 启动和终止线程
线程随着 `thread.start()` 开始启动  到  `run()` 方法执行完毕 结束。

我们可以通过 `Thread.interrupted()` 方法中断线程。
>中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。
线程通过检查自身是否被中断来进行响应，线程通过方法`isInterrupted()`来进行判断是否被中断，也可以调用静态方法`Thread.interrupted()`对当前线程的中断标识位进行复位。如果该线程已经处于终结状态，即使该线程被中断过，在调用该线程对象的`isInterrupted()`时依旧会返回`false`。
许多声明抛出`InterruptedException`的方法（例如`Thread.sleep(long millis)`方法）这些方法在抛出`InterruptedException`之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出`InterruptedException`，此时调用`isInterrupted()`方法将会返回`false`。

下面看一个例子：
```
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while (true) {
            try {
                TimeUnit.SECONDS.sleep(10);
                System.out.println("time = " + System.currentTimeMillis() / 1000 + ", i = " + i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, "t1");
    Thread t2 = new Thread(() -> {
        while (true) {
            i++;
        }
    }, "t2");

    //设置为 daemon 线程 并启动
    t1.setDaemon(true);
    t2.setDaemon(true);
    t1.start();
    t2.start();

    //让t1 t2 运行3s
    TimeUnit.SECONDS.sleep(3);

    //中断线程
    t1.interrupt();
    t2.interrupt();
    //获取中断状态
    System.out.println("time = " + System.currentTimeMillis() / 1000 + ", t1.isInterrupted() = " + t1.isInterrupted());
    System.out.println("time = " + System.currentTimeMillis() / 1000 + ", t2.isInterrupted() = " + t2.isInterrupted());
    //防止 t1 t2 立即退出
    TimeUnit.SECONDS.sleep(15);
}
```
输出结果：
```
time = 1560134045, t1.isInterrupted() = false
time = 1560134045, t2.isInterrupted() = true
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.tcl.executors.Test.lambda$main$0(Test.java:16)
	at java.lang.Thread.run(Thread.java:745)
time = 1560134055, i = -576615207
```
根据输出结果，我们知道在线程`sleep`的时候，调用 `isInterrupted()` 会导致 `sleep interrupted` 异常，并且中断标记也被清除了。


已经被废弃的 `suspend()`（暂停）、`resume()`（恢复） 和 `stop()`（停止）。
废弃原因是，在调用方法之后，线程不会保证占用的资源被正常释放。
示例：
```
public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(() -> {
        while (true) {
            System.out.println("time = " + System.currentTimeMillis() / 1000);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    t.setDaemon(true);
    t.start();
    TimeUnit.SECONDS.sleep(3);

    t.suspend();
    System.out.println("suspend time = " + System.currentTimeMillis() / 1000);
    TimeUnit.SECONDS.sleep(3);

    t.resume();
    System.out.println("resume time = " + System.currentTimeMillis() / 1000);
    TimeUnit.SECONDS.sleep(3);

    t.stop();
    System.out.println("stop time = " + System.currentTimeMillis() / 1000);
    TimeUnit.SECONDS.sleep(3);
}
```
输出结果：
```
time = 1560134529
time = 1560134530
time = 1560134531
suspend time = 1560134532
resume time = 1560134535
time = 1560134535
time = 1560134536
time = 1560134537
stop time = 1560134538
```

---

### 线程间通信
##### volatile和synchronized关键字
`volatile`修饰的变量，程序访问时都需要在共享内存中去读取，对它的改变也必须更新共享内存，保证了线程对变量访问的可见性。

`synchronized`：对于 **同步块** 的实现使用了`monitorenter`和`monitorexit`指令，而 **同步方法** 则是依靠方法修饰符上的`ACC_SYNCHRONIZED`来完成的。无论采用哪种方式，其本质是对一个对象的监视器`monitor`进行获取，而这个获取过程是排他的，也就是同一时刻只能有一个线程获取到由`synchronized`所保护对象的监视器。

---

##### 等待/通知机制——wait和notify
指一个`线程A`调用了`对象O`的`wait()`方法进入等待状态，而另一个`线程B`调用了`对象O`的`notify()`或者`notifyAll()`方法，`线程A`收到通知后从`对象O`的`wait()`方法返回，进而执行后续操作。
等待：`wait()`、`wait(long)`、`wait(long, int)`
通知：`notify()`、`notifyAll()`
示例：
```
private static Object object = new Object();

public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        try {
            TimeUnit.SECONDS.sleep(1);
            synchronized (object) {
                System.out.println("t1 start object.wait(), time = " + System.currentTimeMillis() / 1000);
                object.wait();
                System.out.println("t1 after object.wait(), time = " + System.currentTimeMillis() / 1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    Thread t2 = new Thread(() -> {
        try {
            TimeUnit.SECONDS.sleep(5);
            synchronized (object) {
                System.out.println("t2 start object.notify(), time = " + System.currentTimeMillis() / 1000);
                object.notify();
                System.out.println("t2 after object.notify(), time = " + System.currentTimeMillis() / 1000);
            }

            synchronized (object) {
                System.out.println("t2  hold lock again, time = " + System.currentTimeMillis() / 1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    t1.start();
    t2.start();
}
```
输出结果：
```
t1 start object.wait(), time = 1560138112
t2 start object.notify(), time = 1560138116
t2 after object.notify(), time = 1560138116
t2  hold lock again, time = 1560138116
t1 after object.wait(), time = 1560138116
```
>1.使用`wait()`、`notify()`和`notifyAll()`时需要先对调用对象加锁,否则会报`java.lang.IllegalMonitorStateException`异常。
2.调用`wait()`方法后，线程状态由`RUNNING`变为`WAITING`，并将当前线程放置到对象的等待队列。
3.`notify()`或`notifyAll()`方法调用后，等待线程依旧不会从`wait()`返回，需要调用`notify()`或`notifAll()`的线程释放锁之后，等待线程才有机会从`wait()`返回。
4.`notify()`方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而`notifyAll()`方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由`WAITING`变为`BLOCKED`。
5.从`wait()`方法返回的前提是获得了调用对象的锁。

---

##### 等待/通知的经典范式
包括 **等待方**（消费者）和 **通知方**（生产者）。
等待方遵循以下原则：
* 获取对象的锁。
* 如果条件不满足，那么调用对象的`wait`方法，被通知后任要检查条件。
* 条件不满足则执行对应的逻辑。

对应代码如下：
```
synchronized (对象) {
    while (条件不满足) {
        对象.wait();
    }
    对应的处理逻辑
}
```

通知方遵循以下原则：
* 获取对象的锁。
* 改变条件。
* 通知所有在等待在对象上的线程。
```
synchronized (对象) {
    改变条件
    对象.notifyAll();
}
```

---

##### 管道输入/输出流 
`PipedOutputStream`、`PipedInputStream`、`PipedReader` 和 `PipedWriter`。
示例代码：
```
private static PipedWriter writer;
private static PipedReader reader;

public static void main(String[] args) throws InterruptedException, IOException {
    writer = new PipedWriter();
    reader = new PipedReader();
    //绑定输入输出
    writer.connect(reader);
    Thread t = new Thread(() -> {
        int res;
        try {
            while ((res = reader.read()) != -1) {
                System.out.print((char) res);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
    t.start();

    int res;
    while ((res = System.in.read()) != -1) {
        System.out.println(res);
        writer.write(res);
        //按回车结束
        if (res == 10) {
            break;
        }
    }
    writer.close();
}
```
输出：
```
Hi!
72
105
33
10
Hi!
```

---

##### Thread.join()
`thread.join()` 即当前线程需要在 `thread` 线程执行完之后才能继续执行，[Java Thread.join()详解](https://www.jianshu.com/p/595be9eab056)，这里已经做了详细介绍了，就不再赘述了。

---

##### ThreadLocal
`ThreadLocal`,即线程变量，是一个以`ThreadLocal`对象为 **键** 、`任意对象` 为 **值** 的存储结构。
这个结构被附带在线程上，也就是说一个线程可以根据一个`ThreadLocal`对象查询到绑定在这个线程上的一个值。
可以通过 `set(T t)` 设置， `get()` 获取。
示例如下：
```
private static ThreadLocal<String> threadLocal = new ThreadLocal<>();

public static void main(String[] args) throws InterruptedException {
    String time = String.valueOf(System.currentTimeMillis() / 1000);
    System.out.println("time = " + time);
    threadLocal.set(time);
    Thread t = new Thread(() -> {
        String time1 = String.valueOf(System.currentTimeMillis());
        System.out.println("time1 = " + time1);
        threadLocal.set(time1);
    });
    t.start();
    TimeUnit.SECONDS.sleep(5);
    System.out.println("threadLocal.get() = " + threadLocal.get());
}
```
输出结果：
```
time = 1560146178
time1 = 1560146178263
threadLocal.get() = 1560146178
```
可以看到`线程t`中对`threadLocal`设置的值，并不影响`main线程`中的值。

` set(T value)`方法的源代码：
可以看到即把 **当前ThreadLocal对象** 为 `key`，**传入的参数** 为`value` 保存在 `ThreadLocalMap`中。
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

---

### 小结
本文我们介绍了：
* 什么是线程，为什么使用多线程，线程的优先级、状态变化 以及 `Deamon`线程。
* 线程启动`start()` 和 中断线程`interrupt()`，以及过期的`suspend()`、`resume()`、`stop()`的作用。
* 通过 `volatile` 来 `synchronized`来保证变量在多线程中的可见性，实现线程间通信。
* 用 线程的 **等待/通知** 机制 来 实现线程间通信，使用的注意事项 以及 **等待方** 和 **通知方** 需要遵循的原则。
* 通过管道输入输出流 `PipedOutputStream`、`PipedInputStream`、`PipedWirter`、`PipedReader`的介绍。
* Thread.join() 的使用介绍。
* `ThreadLocal`的使用介绍。


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
