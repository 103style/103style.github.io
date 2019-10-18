# Java中的锁的使用和实现介绍 

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

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源。
`源代码基于 1.8.0`

### 目录
* `Lock接口`
* `队列同步器`
* `重入锁`
* `读写锁`
* `LockSupport工具`
* `Condition接口`
* `小结`

---

### Lock接口
在`Java SE 5`之后，并发包中新增了`Lock`接口（以及相关实现类）用来实现锁功能，它提供了与`synchronized`关键字类似的同步功能，只是在使用时需要 **显式** 地获取和释放锁。
虽然它缺少了（通过`synchronized`块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的 **可操作性**、**可中断的获取锁** 以及 **超时获取锁** 等多种`synchronized`关键字所不具备的同步特性。

使用`synchronized`关键字将会 **隐式** 地获取锁，但是它将锁的获取和释放固化了，也就是先获取再释放。

`Lock`的使用：
```
Lock lock = new ReentrantLock();
lock.lock();
try {
} finally {
    lock.unlock();
}
```
注意 ：
>1.在`finally`块中释放锁，目的是保证在获取到锁之后，最终能够被释放。
2.不要将获取锁的过程写在`try`块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放。

`Lock`接口提供的`synchronized`关键字所不具备的主要特性：
* **尝试非阻塞性获取锁**： 当前线程尝试获取锁，如果此时没有其他线程占用此锁，则成功获取到锁。
* **能被中断的获取锁**： 当获取到锁的线程被中断时，中断异常会抛出并且会释放锁。
* **超时获取锁**： 在指定时间内获取锁，如果超过时间还没获取，则返回。

`Lock` 相关的API：
* `void lock();`：获取锁，获取之后返回
* `void lockInterruptibly() throws InterruptedException;`：可中断的获取锁
* `boolean tryLock();`：尝试非阻塞的获取锁
* `boolean tryLock(long time, TimeUnit unit) throws InterruptedException;`： 超时获取锁。 超时时间结束，未获得锁，返回`false`.
* `void unlock();`：释放锁
* `Condition newCondition();`：获取等待通知组件，改组件和锁绑定，当前线程获取到锁才能调用`wait()`方法，调用之后则会释放锁。

---

### 队列同步器
队列同步器`AbstractQueuedSynchronizer`（以下简称同步器），是用来构建锁或者其他同步组件的基础框架，它使用了一个`int` 成员变量表示同步状态，通过内置的`FIFO`队列来完成资源获取线程的排队工作，并发包的作者`Doug Lea`期望它能够成为实现大部分同步需求的基础。

同步器的主要使用方式是继承`AbstractQueuedSynchronizer`，通过同步器提供的3个方法`getState()`、`setState(int newState)`和`compareAndSetState(int expect,int update)`来进行线程安全的状态同步。

同步器是实现锁的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。
可以这样理解二者之间的关系：
* **锁是面向使用者的**，它定义了使用者与锁交互的接口，隐藏了实现细节；
* **同步器面向的是锁的实现者**，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。

##### 队列同步器的接口与示例
`AbstractQueuedSynchronizer`可重写的方法：
* `boolean tryAcquire(int arg)`：独占式获取同步状态。
* `boolean tryRelease(int arg)`：独占式释放同步状态。
* `int tryAcquireShared(int arg)`：共享式获取同步状态。
* `boolean tryReleaseShared(int arg)`：共享释放取同步状态。
* `boolean isHeldExclusively()`：当前同步器是否在独占式模式下被线程占用。

实现自定义同步组件时，将会调用同步器提供 **独占式获取与释放同步状态**、**共享式获取与释放同步状态** 和 **查询同步队列中的等待线程情况** 三类模板方法。

独占锁的示例代码：
```
/**
 * @author https://github.com/103style
 * @date 2019/6/12 17:32
 */
public class TestLock implements Lock {
    private TestQueuedSync sync;
    /**
     * 获取锁
     */
    @Override
    public void lock() {
        sync.acquire(1);
    }
    /**
     * 可中断的获取锁
     */
    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    /**
     * 尝试非阻塞式获取锁
     */
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }
    /**
     * 尝试非阻塞式获取锁
     */
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquire(1);
    }
    /**
     * 释放锁
     */
    @Override
    public void unlock() {
        sync.release(1);
    }
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
    /**
     * 是否有同步队列线程
     */
    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
    /**
     * 锁是否被占用
     */
    public boolean isLock() {
        return sync.isHeldExclusively();
    }
    private static class TestQueuedSync extends AbstractQueuedSynchronizer {
        /**
         * 独占式获取同步状态
         */
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        /**
         * 独占式释放同步状态
         */
        @Override
        protected boolean tryRelease(int arg) {
            if (getState() == 0) {
                throw new IllegalStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        /**
         * 同步状态是否被占用
         */
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        /**
         * 返回一个Condition，每个condition都包含了一个condition队列
         */
        Condition newCondition() {
            return new ConditionObject();
        }
    }
}
```
上述示例代码中，独占锁`TestLock`是一个自定义同步组件，它在同一时刻只允许一个线程占有锁。`TestLock`中定义了一个静态内部类`TestQueuedSync`继承了同步器，在`tryAcquire(int acquires)`方法中，如果经过`compareAndSetState`设置成功，则代表获取了同步状态`1`，而在`tryRelease(int releases)`方法中只是将同步状态重置为`0`。

用户使用`TestLock`时并不会直接和内部同步器的实现`TestQueuedSync`打交道，而是调用`TestLock`提供的方法，在`TestLock`的实现中，以获取锁的`lock()`方法为例，只需要在方法实现中调用同步器的模板方法`acquire(int args)`即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

##### 队列同步器的实现分析
接下来将从实现角度分析同步器是如何完成线程同步的：
* **同步队列** ： 一个`FIFO`双向队列。
  当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点`Node`并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。
  `Node` 保存 **获取同步状态失败的线程引用**、**等待状态** 以及 **前驱和后继节点**，**节点的属性类型** 与 **名称** 以及 **描述** 如下:
  ```
  /**
   * 等待状态：
   *  CANCELLED : 1 在同步队列中等待超时或被中断，需要从队列中取消等待，在该状态将不会变化
   *  SIGNAL : -1  后继节点地线程处于等待状态，当前节点释放获取取消同步状态，后继节点地线程即开始运行
   *  CONDITION : -2  在等待队列中，
   *  PROPAGATE : -3 下一次共享式同步状态获取将会无条件地被传播下去
   *  INITAL : 0  初始状态
   */
  volatile int waitStatus;
  volatile Node prev;//前驱节点
  volatile Node next;//后继节点
  volatile Thread thread;//获取同步状态的线程
  Node nextWaiter;//等待队列中的后继节点。 如果节点是共享的的，这个字段将是一个SHARED常量
  ```
![同步队列的基本结构](https://upload-images.jianshu.io/upload_images/1709375-b3af4d21c384aea3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  
  如上图所示，同步器包含了两个节点类型的引用，一个指向 **头节点**，而另一个指向 **尾节点**。
  试想一下，当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：`compareAndSetTail(Node expect,Node update)`，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。


* **独占式同步状态获取与释放**
  通过调用同步器的`acquire(int arg)`方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移出。`acquire(int arg)`代码如下：
    ```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    ```  
    上述代码主要完成了 **同步状态获取**、**节点构造**、**加入同步队列** 以及 **在同步队列中自旋等待**。
    * 首先调用自定义同步器实现的`tryAcquire(int arg)`方法保证线程安全的获取同步状态
    * 如果获取同步状态失败，构造同步节点（独占式`Node.EXCLUSIVE`，同一时刻只能有一个线程成功获取同步状态）并通过`addWaiter(Node node)`方法将该节点加入到同步队列的尾部。
    * 最后调用`acquireQueued(Node node,int arg)`方法，使得该节点以“死循环”的方式获取同步状态
    * 如果获取不到则阻塞节点中的线程，而 **被阻塞线程的唤醒** 主要依靠 **前驱节点的出队**或 **阻塞线程被中断** 来实现。

    我们来看下节点的构造以及加入同步队列的`addWaiter(Node mode)`和`initializeSyncQueue()`方法：
    ```
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                U.putObject(node, Node.PREV, oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
    private final void initializeSyncQueue() {
        Node h;
        if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
            tail = h;
    }
    ```
    上述代码通过在“死循环”中使用`compareAndSetTail(Node expect,Node update)`方法来确保节点能够被线程安全添加。 如果没有尾节点的话，则构建一个新的同步队列。

    接下来看下`acquireQueued(final Node node, int arg)`方法：
    ```
    final boolean acquireQueued(final Node node, int arg) {
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    ```
    在`acquireQueued(final Node node,int arg)`方法中，当前线程在“死循环”中尝试获取同步状态，而只有前驱节点是头节点才能够尝试获取同步状态，这是为什么？原因有两个:
    * **头节点是成功获取到同步状态的节点，而头节点的线程释放了同步状态之后，将会唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点**。
    * **维护同步队列的FIFO原则**。
![节点自旋获取同步状态](https://upload-images.jianshu.io/upload_images/1709375-8ce5e9ccfb9e55a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  由于非首节点线程前驱节点出队或者被中断而从等待状态返回，随后检查自己的前驱是否是头节点，如果是则尝试获取同步状态。可以看到节点和节点之间在循环检查的过程中基本不相互通信，而是简单地判断自己的前驱是否为头节点，这样就使得节点的释放规则符合FIFO，并且也便于对过早通知的处理（过早通知是指前驱节点不是头节点的线程由于中断而被唤醒）。
![独占式同步状态获取流程](https://upload-images.jianshu.io/upload_images/1709375-4881fdc50469a93c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  调用同步器的`release(int arg)`方法可以释放同步状态，然后会唤醒其后继节点（进而使后继节点重新尝试获取同步状态）。`release(int arg)`执行之后会唤醒后继的节点。
    ```
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
    ```

* **共享式同步状态获取与释放** 
    **共享式获取** 与 **独占式获取** 最主要的区别在于 **同一时刻能否有多个线程同时获取到同步状态**。

    以文件的读写为例，**写操作** 要求对资源的 **独占式访问** ，而 **读操作** 可以是 **共享式访问**。

    通过调用`acquireShared(int arg)`可以共享式地获取同步状态，方法代码如下：
    ```
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    ```
    在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是`tryAcquireShared(int arg)`方法返回值 `大于等于0`。
    在`doAcquireShared(int arg)`方法的自旋过程中，如果当前节点的 **前驱为头节点** 时，尝试获取同步状态，如果返回值 `大于等于0`，表示该次获取同步状态成功并从自旋过程中退出。

    共享式获取也需要释放同步状态，通过调用`releaseShared(int arg)`方法可以释放同步状态：
    ```
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
    ```
    该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点。对于能够支持多个线程同时访问的并发组件（比如`Semaphore`），它和独占式主要区别在于`tryReleaseShared(int arg)`方法必须确保同步状态（或者资源数）线程安全释放，一般是通过循环和CAS来保证的，因为释放同步状态的操作会同时来自多个线程。

* **超时获取同步状态**
    通过调用同步器的`doAcquireNanos(int arg,long nanosTimeout)`方法可以超时获取同步状态，即在指定的时间段内获取同步状态，如果获取到同步状态则返回`true`，否则，返回`false`。该方法提供了传统Java同步操作（比如`synchronized`关键字）所不具备的特性。
    ```
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L) {
                    cancelAcquire(node);
                    return false;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > SPIN_FOR_TIMEOUT_THRESHOLD)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    ```
    该方法在自旋过程中，当节点的前驱节点为头节点时尝试获取同步状态，如果获取成功则从该方法返回，这个过程和独占式同步获取的过程类似。获取失败则会重新计算超时时间。

    如果`nanosTimeout`小于等于`spinForTimeoutThreshold`（`1000纳秒`）时，将不会使该线程进行超时等待，而是进入快速的自旋过程。原因在于，**非常短的超时等待无法做到十分精确**。

    **独占式超时获取同步态** 的流程下：
    ![独占式超时获取同步状态的流程](https://upload-images.jianshu.io/upload_images/1709375-fe15d4e980a19368.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **自定义同步组件**
    设计一个同步工具：
    * `在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞。`
    * `能够在同一时刻支持多个线程的访问(共享式访问)。`

    代码如下：`省略了无关的重写方法`
    ```
    /**
     * @author xiaoke.luo@tcl.com 2019/6/14 11:15
     * 自定义同步组件
     * <p>
     * 实现以下功能
     * 1.在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞。
     * 2.能够在同一时刻支持多个线程的访问(共享式访问)。
     */
    public class CustomLock implements Lock {
        private CustomSyncQueue customSyncQueue = new CustomSyncQueue(2);
        public static void main(String[] args) {
            final Lock lock = new CustomLock();
            class Worker extends Thread {
                @Override
                public void run() {
                    while (true) {
                        lock.lock();
                        try {
                            SleepUtils.second(1);
                            System.out.println(Thread.currentThread().getName());
                            SleepUtils.second(1);
                        } finally {
                            lock.unlock();
                        }
                    }
                }
            }
            // 启动10个线程
            for (int i = 0; i < 10; i++) {
                Worker w = new Worker();
                w.setDaemon(true);
                w.start();
            }
            // 每隔1秒换行
            for (int i = 0; i < 10; i++) {
                SleepUtils.second(1);
                System.out.println();
            }
        }
        @Override
        public void lock() {
            customSyncQueue.tryAcquireShared(1);
        }
        @Override
        public void unlock() {
            customSyncQueue.tryReleaseShared(1);
        }
        public static class CustomSyncQueue extends AbstractQueuedSynchronizer {
            public CustomSyncQueue(int count) {
                if (count <= 0) {
                    throw new IllegalStateException("count must >= 0");
                }
                setState(count);
            }
            @Override
            protected int tryAcquireShared(int reduceCount) {
                for (; ; ) {
                    int current = getState();
                    int newCount = current - reduceCount;
                    if (newCount < 0 || compareAndSetState(current, newCount)) {
                        return newCount;
                    }
                }
            }
            @Override
            protected boolean tryReleaseShared(int returnCount) {
                for (; ; ) {
                    int current = getState();
                    int newCount = current + returnCount;
                    if (compareAndSetState(current, newCount)) {
                        return true;
                    }
                }
            }
        }
    }
    ```
    上述代码主要还是 `CustomSyncQueue` 的 `tryAcquireShared` 和 `tryReleaseShared` 方法，当`tryAcquireShared(int reduceCount)`方法返回值`>=0`时，当前线程才获取同步状态。

---

### 重入锁
重入锁`ReentrantLock`，就是支持重进入的锁，它表示该锁能够支持 **一个线程对资源的重复加锁**。
除此之外，该锁的还支持获取锁时的公平和非公平性选择。

我们回顾下`TestLock`的`lock`方法，在 `tryAcquire(int acquires)`方法时没有考虑占有锁的线程再次获取锁的场景，而在调用`tryAcquire(int acquires)`方法时返回了`false`，导致该线程被阻塞。

**在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。** 
事实上，**公平的锁机制往往没有非公平的效率高**，但是，并不是任何场景都是以TPS作为唯一的指标，公平锁能够减少“饥饿”发生的概率，等待越久的请求越是能够得到优先满足。

下面我们来分析下`ReentrantLock` 的实现：
* **实现重进入**
  重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决以下两个问题：
  * **线程再次获取锁**
  * **锁的最终释放**
  
  下面是`ReentrantLock`通过组合自定义同步器来实现锁的获取与释放，以非公平性（默认的）实现：
    ```
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    ``` 
    此方法通过判断 **当前线程是否为获取锁的线程** 来决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回`true`，表示获取同步状态成功。

  下面看释放锁的方法`tryRelease(int releases) `：
    ```
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    ```
    通过检查 `state == 0` 来判断是否需要继续释放锁。

* **公平与非公平获取锁的区别**
  公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是 **FIFO**。
  对于上面介绍的非公平锁实现的`nonfairTryAcquire(int acquires)`，只要 **CAS** 设置同步状态成功，即获取到锁，而公平锁则不同，如下：
    ```
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    ```
    相比非公平锁的实现，公平锁的实现在获取锁的时候多了一个`!hasQueuedPredecessors()`判断：
    ```
    public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
    ```
    即加入了 **同步队列中当前节点是否有前驱节点的判断** ，如果该方法返回 `true`，则表示有线程比当前线程更早地请求获取锁，因此 **需要等待前驱线程获取并释放锁之后才能继续获取锁**。

---

### 读写锁
之前提到锁（如`TestLock`和`ReentrantLock`）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。
读写锁维护了一对锁，一个 **读锁** 和一个 **写锁**，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

一般情况下，**读写锁** 的性能都会比 **排它锁** 好，因为大多数场景 **读是多于写** 的。在读多于写的情况下，**读写锁** 能够提供比 **排它锁** 更好的 **并发性** 和 **吞吐量**。Java并发包提供读写锁的实现是`ReentrantReadWriteLock` ，特性如下：
* **公平性选择** ：支持公平和非公平的方式获取锁，吞吐量非公平优于公平。
* **重进入** ： 读锁在获取锁之后再获取读锁，写锁在获取锁之后再获取读锁和写锁。
* **锁降级** ：遵循获取写锁、获取读锁在释放写锁的次序，写锁能够降级为读锁。

##### 读写锁的接口与示例
`ReadWriteLock`仅定义了获取读锁和写锁的两个方法，即`readLock()`方法和`writeLock()`方法，而其实现类`ReentrantReadWriteLock`，除了接口方法之外，还提供了一些便于外界监控其内部工作状态的方法，这些方法如下：
* `getReadLockCount()`：返回当前读锁获取的次数
* `getReadHoldCount()`：返回当前线程获取读锁的次数
* `isWriteLocked()`：判断写锁是否被获取
* `getWriteHoldCount()`：返回当前写锁被获取的次数

以下是读写锁的使用示例代码：
通过读写锁保证 非线程安全的`HashMap`的读写是线程安全的。
```
static Map<String, Object> map = new HashMap<>();
static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
static Lock r = rwl.readLock();
static Lock w = rwl.writeLock();
/**
 * 获取一个key对应的value
 */
public static final Object get(String key) {
    r.lock();
    try {
        return map.get(key);
    } finally {
        r.unlock();
    }
}
/**
 * 设置key对应的value，并返回旧的value
 */
public static final Object put(String key, Object value) {
    w.lock();
    try {
        return map.put(key, value);
    } finally {
        w.unlock();
    }
}
/**
 * 清空所有的内容
 */
public static final void clear() {
    w.lock();
    try {
        map.clear();
    } finally {
        w.unlock();
    }
}
```

##### 读写锁的实现分析
主要包括：读写状态的设计、写锁的获取与释放、读锁的获取与释放以及锁降级。
* **读写状态的设计**
读写锁将变量切分成了两个部分，高16位表示读，低16位表示写，如下图：
![读写锁状态的划分方式](https://upload-images.jianshu.io/upload_images/1709375-007270a4c8302a36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当前同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。
读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。
假设当前同步状态值为`S`，写状态等于`S&0x0000FFFF`（将高16位全部抹去），读状态等于`S>>16`（无符号补0右移16位）。
当写状态增加`1`时，等于`S+1`，当读状态增加`1`时，等于`S+(1<<16)`，也就是`S+0x00010000`。
根据状态的划分能得出一个推论：`S != 0`时，当写状态`S&0x0000FFFF  = 0`时，则读状态`S>>16 > 0`，即读锁已被获取。

* **写锁的获取与释放**
写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态。
`ReentrantReadWriteLoc`的`tryAcquire`方法如下：
    ```
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            // (Note: if c != 0 and w == 0 then shared count != 0)
            // 存在读锁或者当前获取线程不是已经获取写锁的线程
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            // Reentrant acquire
            setState(c + acquires);
            return true;
        }
        if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }
    ```
    该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个 **读锁是否存在** 的判断。**如果存在读锁，则写锁不能被获取。**

    写锁的释放与`ReentrantLock`的释放过程基本类似，每次释放均减少写状态，当写状态为`0`时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

* **读锁的获取与释放**
  读锁是一个支持重进入的 **共享锁**，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为`0`）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。

  如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。

   `ReentrantReadWriteLock`的`tryAcquireShared`方法:
    ```
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
            return -1;
        int r = sharedCount(c);
        if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
            ...
            return 1;
        }
        return fullTryAcquireShared(current);
    }
    final int fullTryAcquireShared(Thread current) {
        HoldCounter rh = null;
        for (;;) {
            int c = getState();
            if (exclusiveCount(c) != 0) {
                if (getExclusiveOwnerThread() != current)
                    return -1;
                // else we hold the exclusive lock; blocking here
                // would cause deadlock.
            } else if (readerShouldBlock()) {
                // Make sure we're not acquiring read lock reentrantly
                if (firstReader == current) {
                    // assert firstReaderHoldCount > 0;
                } else {
                    if (rh == null) {
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current)) {
                            rh = readHolds.get();
                            if (rh.count == 0)
                                readHolds.remove();
                        }
                    }
                    if (rh.count == 0)
                        return -1;
                }
            }
            if (sharedCount(c) == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (sharedCount(c) == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    if (rh == null)
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                    cachedHoldCounter = rh; // cache for release
                }
                return 1;
            }
        }
    }
    ```
    如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。
    如果当前线程获取了写锁或者写锁未被获取，则当前线程增加读状态，成功获取读锁。

    读锁的每次释放均减少读状态，减少的值是`1<<16`。

* **锁降级** 
    锁降级指的是 **写锁降级成为读锁**。
    如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。
    以下是是锁降级的示例：
    ```
    //当数据发生变更后，update变量（布尔类型且volatile修饰）被设置为false
    public void processData() {
        readLock.lock();
        if (!update) {
            // 必须先释放读锁
            readLock.unlock();
            // 锁降级从写锁获取到开始
            writeLock.lock();
            try {
                if (!update) {
                    // 准备数据的流程（略）
                    update = true;
                }
                readLock.lock();
            } finally {
                writeLock.unlock();
            }
            // 锁降级完成，写锁降级为读锁
        }
        try {
            // 使用数据的流程（略）
        } finally {
            readLock.unlock();
        }
    }
    ```
    上述示例中，当数据发生变更后，`布尔`类型且`volatile`修饰`update`变量被设置为`false`，此时所有访问`processData()`方法的线程都能够感知到变化，但只有一个线程能够获取到写锁，其他线程会被阻塞在读锁和写锁的`lock()`方法上。当前线程获取写锁完成数据准备之后，再获取读锁，随后释放写锁，完成锁降级。

    **锁降级中读锁的获取是否必要呢？**
    答案是必要的。主要是为了 **保证数据的可见性**。
    如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作`线程T`）获取了写锁并修改了数据，那么 **当前线程无法感知`线程T`的数据更新**。
    如果当前线程获取读锁，即遵循锁降级的步骤，则`线程T`将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。

    `RentrantReadWriteLock`不支持锁升级。目的也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。

---

### LockSupport工具
当需要阻塞或唤醒一个线程的时候，都会使用`LockSupport`工具类来完成相应工作。
`LockSupport`定义了一组的公共静态方法，这些方法提供了最基本的 **线程阻塞和唤醒功能**，而`LockSupport`也成为构建同步组件的基础工具。

`LockSupport`提供的 阻塞和唤醒的方法 如下：
* `park()`：阻塞当前线程，只有调用 `unpark(Thread thread)`或者被中断之后才能从`park()`返回。
* `parkNanos(long nanos)`：再`park()`的基础上增加了超时返回。
* `parkUntil(long deadline)`：阻塞线程知道 `deadline` 对应的时间点。
* `park(Object blocker)`：`Java 6`时增加，`blocker`为当前线程在等待的对象。
* `parkNanos(Object blocker, long nanos)`：`Java 6`时增加，`blocker`为当前线程在等待的对象。
* `parkUntil(Object blocker, long deadline)`：`Java 6`时增加，`blocker`为当前线程在等待的对象。
* `unpark(Thread thread)`：唤醒处于阻塞状态的线程 `thread`。

有对象参数的阻塞方法在线程`dump`时，会有更多的现场信息。


---

### Condition接口
任意一个Java对象，都拥有一组监视器方法，定义在`java.lang.Object`），主要包括`wait()`、`wait(long timeout)`、`notify()`以及`notifyAll()`方法，这些方法与`synchronized`同步关键字配合，可以实现等待/通知模式。

`Condition`接口也提供了类似`Object`的监视器方法，与`Lock`配合可以实现 **等待/通知** 模式，但是这两者在使用方式以及功能特性上还是有差别的。

以下是Object的监视器方法与Condition接口的对比：
|对比项|Object|Condition|
|:-:|:-:|:-:|
|前置条件|获取对象的锁|调用`Lock.lock()`获取锁；调用`Lock.newCondition()`获取condition对象|
|调用方式|`object.wait()`|`condition.wait()`|
|等待队列个数|一个|多个|
|当前线程释放锁并进入等待状态|支持|支持|
|当前线程释放锁并进入等待状态，在等待状态中不响应中断|不支持|支持|
|当前线程释放锁并进入超时等待状态|支持|支持|
|当前线程释放锁并进入等待状态到将来某时间|不支持|支持|
|唤醒等待队列的一个线程|支持|支持|
|唤醒等待队列的全部线程|支持|支持|

##### Condition接口与示例
`Condition`定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到`Condition`对象关联的锁。
`Condition`对象是由调用`Lock`对象的`newCondition()`方法创建出来的，换句话说，`Condition`是依赖`Lock`对象的。

`Condition`的使用方式比较简单，需要注意在调用方法前获取锁，如下：
```
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock();
    }
}
public void conditionSignal() throws InterruptedException {
    lock.lock();
    try {
        condition.signal();
    } finally {
        lock.unlock();
    }
}
```

`Condition` 接口方法介绍：
* `void await() throws InterruptedException` ： 当前线程进入等待状态直到被通知或中断
* `void awaitUninterruptibly()` ：当前线程进入等待状态直到被通知，对中断不敏感
* `long awaitNanos(long var1) throws InterruptedException` ：当前线程进入等待状态直到被通知、中断或超时
* `boolean await(long var1, TimeUnit var3) throws InterruptedException` ：当前线程进入等待状态直到被通知、中断或超时
* `boolean awaitUntil(Date var1) throws InterruptedException` ：当前线程进入等待状态直到被通知、中断或到某一时间
* `void signal()` ：唤醒`Condition`上一个在等待的线程
* `void signalAll()` ：唤醒`Condition`上全部在等待的线程

**获取一个Condition必须通过Lock的newCondition()方法。**

通过下面这个有界队列的示例我们来深入了解下 `Condition` 的使用方式:
```
public class BoundedQueue<T> {
    private Object[] items;
    // 添加的下标，删除的下标和数组当前数量
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();

    public BoundedQueue(int size) {
        items = new Object[size];
    }

    // 添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length){
                notFull.await();
            }
            items[addIndex] = t;
            if (++addIndex == items.length)
                addIndex = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    // 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
    @SuppressWarnings("unchecked")
    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            Object x = items[removeIndex];
            if (++removeIndex == items.length)
                removeIndex = 0;
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```
上述代码`add` 和 `remove` 方法 都需要先获取锁保证数据的可见性和排它性。
当储存数组满了的时候时候调用`notFull.await()`，线程即释放锁并进入等待队列。
当储存数组未满时，则添加到数组，并通知 `notEmpty` 中等待的线程。
方法中使用`while`循环是为了防止过早或者意外的通知。


##### Condition的实现分析
主要包括 等待队列、等待和通知。
* **等待队列**
    等待队列是一个`FIFO`的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在`Condition`对象上等待的线程，如果一个线程调用了`Condition.await()`方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。同步队列和等待队列中节点类型都是同步器的静态内部类`AbstractQueuedSynchronizer.Node`。

    线程调用`Condition.await()`，即以当前线程构造节点，并加入等待队列的尾部。
    ![等待队列的基本结构](https://upload-images.jianshu.io/upload_images/1709375-fdd3ea44b0246d87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    如下图所示，`Condition`的实现是同步器的内部类，因此每个`Condition`实例都能够访问同步器提供的方法，相当于每个`Condition`都拥有所属同步器的引用。    
    ![同步队列与等待队列](https://upload-images.jianshu.io/upload_images/1709375-c7fffb9ed8bdd131.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



* **等待**
    调用`Condition`的`await()`等方法，会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态。当从`await()`方法返回时，当前线程一定获取了`Condition`相关联的锁。

    `Condition`的`await()`源码：
    ```
    public final void await() throws InterruptedException {
        if (Thread.interrupted()) {
            throw new InterruptedException();
        } else {
            AbstractQueuedSynchronizer.Node node = this.addConditionWaiter();
            int savedState = AbstractQueuedSynchronizer.this.fullyRelease(node);
            int interruptMode = 0;
            while(!AbstractQueuedSynchronizer.this.isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = this.checkInterruptWhileWaiting(node)) != 0) {
                    break;
                }
            }
            if (AbstractQueuedSynchronizer.this.acquireQueued(node, savedState) && interruptMode != -1) {
                interruptMode = 1;
            }
            if (node.nextWaiter != null) {
                this.unlinkCancelledWaiters();
            }
            if (interruptMode != 0) {
                this.reportInterruptAfterWait(interruptMode);
            }
        }
    }
    ```
    调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。
    当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用`Condition.signal()`方法唤醒，而是对等待线程进行中断，则会抛出`InterruptedException`。
    ![当前线程加入等待队列](https://upload-images.jianshu.io/upload_images/1709375-f3a2ee6a5ac656af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **通知**
    调用`Condition`的`signal()`方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中。

    `Condition`的`signal()`源码：
    ```
    public final void signal() {
        if (!AbstractQueuedSynchronizer.this.isHeldExclusively()) {
            throw new IllegalMonitorStateException();
        } else {
            AbstractQueuedSynchronizer.Node first = this.firstWaiter;
            if (first != null) {
                this.doSignal(first);
            }
        }
    }
    private void doSignal(AbstractQueuedSynchronizer.Node first) {
        do {
            if ((this.firstWaiter = first.nextWaiter) == null) {
                this.lastWaiter = null;
            }
            var1.nextWaiter = null;
        } while (!AbstractQueuedSynchronizer.this.transferForSignal(first) && (first = this.firstWaiter) != null);

    }
    ```
    调用该方法的前置条件是当前线程必须获取了锁，可以看到`signal()`方法进行了`isHeldExclusively()`检查，也就是当前线程必须是获取了锁的线程。
    接着获取等待队列的首节点，将其移动到同步队列并使用`LockSupport`唤醒节点中的线程。
    ![节点从等待队列移动到同步队列](https://upload-images.jianshu.io/upload_images/1709375-2341f3905558d9ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    通过调用同步器的`enq(Node node)`方法，等待队列中的头节点线程安全地移动到同步队列。
    当节点移动到同步队列后，当前线程再使用`LockSupport`唤醒该节点的线程。

    被唤醒后的线程，将从`await()`方法中的`while`循环中退出`isOnSyncQueue(Node node)`方法返回`true`，节点已经在同步队列中，进而调用同步器的`acquireQueued()`方法加入到获取同步状态的竞争中。

    成功获取同步状态之后，被唤醒的线程将从先前调用的`await()`方法返回，此时该线程已经成功地获取了锁。

   ` Condition`的`signalAll()`方法，相当于对等待队列中的每个节点均执行一次signal()方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。
---

### 小结
* `Lock`接口提供的方法`lock()`、`unlock()`等获取和释放锁的介绍
* 队列同步器的使用 以及 自定义队列同步器
* 重入锁 的使用和实现介绍
* 读写锁 的 读锁 和 写锁
* `LockSupport`工具实现 阻塞和唤醒线程
* `Condition`接口实现 等待/通知模式 

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
