# Java并发机制的底层实现原理 

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



Java代码  编译之后 得到 Java字节码，被 类加载器加载到`JVM`中，最终 转化为汇编指令。

#### volatile
`volatile`是轻量级的`synchronized`，被`volatile`修饰的变量，**在一个线程能读到这个变量被另一个线程修改之后的值。**
`volatile`不会引起线程上下文切换和调度。

#### volatile的两条实现原则
* Lock前缀指令会引起处理器缓存回写到内存.
* 一个处理器的缓存回写到内存会导致其他处理器的缓存无效.

#### synchronized实现同步
* 对于 普通同步方法，锁是 当前实例对象。
* 对于 静态同步方法，锁是 当前类的Class对象。
* 对于 同步方法块，锁是 Synchonized括号里配置的对象。

#### Synchonized在JVM里的实现原理
**JVM** 基于进入和退出`Monitor`对象来实现方法同步和代码块同步。
* 代码块同步是使用`monitorenter`和`monitorexit`指令实现的
* 方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明。但是，方法的同步同样可以使用`monitorenter`和`monitorexit`指令来实现。

`monitorenter`指令是在编译后插入到同步代码块的开始位置，`monitorexit`是插入到方法结束处和异常处。
JVM要保证每个`monitorenter`必须有对应的`monitorexit`与之配对。
任何对象都有一个`monitor`与之关联，当且一个`monitor`被持有后，它将处于锁定状态。线程执行到`monitorenter`指令时，将会尝试获取对象所对应的`monitor`的所有权，即尝试获得对象的锁。

#### Java对象头
`synchronized`用的锁是存在Java对象头里的。在`32位` 虚拟机中，`1字宽` 等于`4字节`，即`32bit`。
* **数组类型**,虚拟机用`3个字宽`存储对象头。
* **非数组类型**，虚拟机用`2字宽`存储对象头。

####  锁的4种状态
级别从低到高依次是:
* 无锁状态
* 偏向锁状态
* 轻量级锁状态
* 重量级锁状态

[Java中的锁介绍](https://www.jianshu.com/p/f3bb5313fce4)

### 原子操作的实现原理
原子（`atomic`）本意是“不能被进一步分割的最小粒子”，而原子操作（`atomic operation`）意为“不可被中断的一个或一系列操作”。
* **处理器如何实现原子操作**
  * **使用总线锁保证原子性**：所谓总线锁就是使用处理器提供的一个`LOCK＃`信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。
  * **使用缓存锁保证原子性**。 
  以下两种情况不会使用缓存锁：
    * 当处理器不支持缓存锁定。
    * 当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行时，则处理器会调用总线锁定。
* **Java如何实现原子操作**
  * **使用循环CAS实现原子操作**， [Java中的12个原子操作类介绍](https://www.jianshu.com/p/cb64f79058c2)。
  * CAS实现原子操作的三大问题：
    * **ABA问题**：因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。
    `ABA`问题的解决思路就是 **使用版本号**。在变量前面追加上版本号，每次变量更新的时候把`版本号`加`1`，那么`A→B→A`就会变成`1A→2B→3A`。
    原子操作类`AtomicStampedReference`的`compareAndSet`方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。
    * **循环时间长开销大**：自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。
    * **只能保证一个共享变量的原子操作**：当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。
  * **使用锁机制实现原子操作**
    锁机制保证了只有获得锁的线程才能够操作锁定的内存区域。


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
