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
* `内存模型基础`
* `volatile的内存语义`
* `锁的内存语义`
* `final域的内存语义`
* `happens-before`
* `双重检查锁定与延迟初始化`
* `Java内存模型综述`
* `小结`

### 内存模型基础

#####  1、并发编程的两个关键问题 
* **线程之间如何通信？**
  通信是指 **以何种机制来交换信息**。
  命令式编程中线程的通信机制主要是以下两种：
  * **共享内存** 的并发模型：通过 **读写内存中的公共状态 来进行隐式通信。**
  * **消息传递** 的并发模型：没有公共状态，只能 **通过发送消息来显示的进行通信。**

* **线程之间如何同步？**
  同步是指 **程序中用于控制不同线程间操作发生相对顺序** 的机制。
  * **共享内存** 的并发模型：同步时显示进行的。我们必须显示指定某段代码需要在线程直线互斥执行。
  * **消息传递** 的并发模型：由于消息发送必须在消息接收之前，因此同步时隐式的。

**Java并发** 采用的是 **共享内存模型**，Java线程之前的通信总是隐式进行的。

#####  2、Java内存模型的抽象结构
在Java中，所有 **实例域**、**静态域** 和 **数组元素** 都储存在堆内存中，**堆内存在线程之前共享**。
本文用  **共享变量** 统一描述 实例域、静态域 和 数组元素 。

**局部变量** 、**方法定义参数**、**异常处理器参数** 不会在内存之间共享，他们不会有内存可见性问题，也不受内存模型影响。

Java线程通信由Java内存模型（简称 `JMM`）控制，JMM 决定一个线程对共享变量的写入何时对另一个线程可见。
从抽象角度看，JMM定义了 **线程** 和 **主内存** 之间的抽象关系：**线程之间的共享变量储存在主内存中，每个线程都有一个私有的本地内存，本地内存储存了 该线程  以读写共享变量的副本。**

![Java内存模型抽象结构示意图](https://upload-images.jianshu.io/upload_images/1709375-781945e949d252e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图来看，线程A和线程B需要通信的话，需要经历以下步骤：
1、线程A 把 本地内存A 中的 共享变量副本 刷新到 主内存 中。
2、线程B 去读取 主内存 中 线程A 刷新过的 共享变量。

从整体来看，这两个步骤实质上是线程A向线程B发送消息，而通信必须经过主内存。
JMM 通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性的保证。

#####  3、从源代码到指令序列的重排序
执行程序的时候，为了提高性能，**编译器** 和 **处理器** 常常会对指令做 **重排序**。
主要有以下三类：
* **编译器优化的重排序** ：编译器在 **不改变单线程程序语义** 的前提下，可以重新安排语句的执行顺序。 
* **指令级并行的重排序** : 现代处理器采用 **并行技术** 来将**多条指令重叠执行**，**如果不存在数据依赖性**，处理器可以改变对应机器指令的执行顺序。
* **内存系统的重排序** : 由于处理使用缓存和读写缓冲区，这使得加载和存储操作看上去可能乱序执行。

以下描述了源代码到最终执行的指令序列的示意图：
![从源码到最终执行的指令序列的示意图](https://upload-images.jianshu.io/upload_images/1709375-cf62b44b02f988f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中的 1 属于 编译器重排序，2 和 3 属于 处理器重排序。这些重排序可能会导致多线程程序出现内存可见性问题。

对于编译器重排序， JMM的编译器重排序规则 **会禁止特定类型的编译器重排序**。
对于处理器重排序，JMM的处理器重排序规则 **会要求编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序**。  

#####  4、happens-before简介
从 **JDK5** 开始，Java使用新的 **JSR-133** 内存模型。 **JSR-133** 使用 **happens-before** 的概念来阐述操作之间的内存可见性。在 **JMM** 中，**如果一个操作执行的结果需要对另一个操作可见，则这两个操作必须要存在happen-before关系** 。

**happen-before** 规则如下：
* **程序顺序规则**：一个线程中的每个操作，happen-before与该线程中的任意后续操作
* **监视器锁规则**：对一个锁的解锁，happen-before与随后这个锁的加锁
* **volatile变量规则**：对于一个volatile域的写，happen-before与任意后续对这个volatile域的读
* **传递性**： A happen-before B，B happen-before C，则A happen-before C

---

### volatile的内存语义
理解`volatile`特性的一个好方法是把对`volatile`变量的单个读/写，看成是 **使用同一个锁** 对这些单个读/写操作做了同步。

`volatile`变量具有下列特性:
* 可见性：总是能看到（任意线程）对这个`volatile`变量最后的写入。
* 原子性：对任意单个`volatile`变量的读/写具有原子性，但类似于`volatile++`这种复合操作不具有原子性。

`volatile`写的内存语义：当写一个`volatile`变量时，**JMM** 会把该线程对应的本地内存中的共享变量值刷新到主内存。
`volatile`读的内存语义：当读一个`volatile`变量时，**JMM** 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

##### volatile内存语义的实现:
为了实现`volatile`内存语义，**JMM** 会分别限制这两种类型的重排序类型。

以下是 JMM 针对 **编译器** 制定的 `volatile`重排序规则表：
|是否能重排序| 第二个操作|第二个操作|第二个操作|
|:-:|:-:|:-:|:-:|
|第一个操作|普通读写|volatile读|volatile写|
|普通读写|||`(1)`NO|
|volatile读|NO|NO|NO|
|volatile写||NO|NO|
第三行最后一个单元格`(1)`的意思是：在程序中，当第一个操作为普通变量的读或写时，如果第二个操作为volatile写，则编译器不能重排序这两个操作。
在上表中，我们可以知道：
* 当第二个操作是`volatile 写`时，不管第一个操作是什么，都不能重排序。这个规则确保`volatile 写`之前的操作不会被编译器重排序到`volatile 写`之后。
* 当第一个操作是`volatile 读`时，不管第二个操作是什么，都不能重排序。这个规则确保`volatile 读`之后的操作不会被编译器重排序到`volatile 读`之前。
* 当第一个操作是`volatile 写`，第二个操作是`volatile 读`时，不能重排序。

为了实现`volatile`的内存语义，编译器在生成字节码时，会在指令序列中插入 **内存屏障** 来禁止特定类型的 **处理器重排序**。
对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。
为此，JMM采取保守策略。
下面是基于保守策略的JMM内存屏障插入策略。
* 在每个`volatile 写`操作的前面插入一个`StoreStore`屏障，后面插入一个`StoreLoad`屏障。
* 在每个`volatile 读`操作的后面插入一个`LoadLoad`屏障，后面插入一个`LoadStore`屏障。

>当读线程的数量大大超过写线程时，选择在`volatile`写之后插入`StoreLoad`屏障将带来可观的执行效率的提升。

---

### 锁的内存语义
锁是Java并发编程中最重要的同步机制。锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息。

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。和 `volatile 写` 类似。
当线程获取锁时，JMM会把该线程对应的本地内存置为无效。和 `volatile 读` 类似。

锁释放和锁获取的内存语义总结：
* 线程A释放一个锁，实质上是线程A向接下来将要获取这个锁的某个线程发出了（线程A对共享变量所做修改的）消息。
* 线程B获取一个锁，实质上是线程B接收了之前某个线程发出的（在释放这个锁之前对共享变量所做修改的）消息。
* 线程A释放锁，随后线程B获取这个锁，这个过程实质上是线程A通过主内存向线程B发送消息。

类似 [Java并发编程基础](https://www.jianshu.com/p/d50cfcaf8102) 介绍的 **等待/通知** 机制。

[锁的介绍]()

---

### final域的内存语义
与前面介绍的锁和`volatile`相比，对`final`域的读和写更像是普通的变量访问。

对于`final`域，**编译器** 和 **处理器** 要遵守两个 **重排序规则**。
* 在构造函数内对一个`final`域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
* 初次读一个包含`final`域的对象的引用，与随后初次读这个`final`域，这两个操作之间不能重排序。

写`final域` 的重排序规则禁止把`final 域`的写重排序到构造函数之外。

读 `final域` 的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的 `final域`，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。

---


### happens-before
`happens-before` 是 **JMM** 最核心的概念。

重排序规则：只要不改变程序的执行结果（指的是单线程程序和正确同步的多线程程序），编译器和处理器怎么优化都行。

`happens-before`关系的定义如下：
* 如果一个操作`happens-before`另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
* 两个操作之间存在`happens-before`关系，并不意味着Java平台的具体实现必须要按照`happens-before`关系指定的顺序来执行。如果重排序之后的执行结果，与按`happens-before`关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。

**《JSR-133:Java Memory Model and Thread Specification》** 定义了如下`happens-before`规则：
* 程序顺序规则：一个线程中的每个操作，`happens-before`于该线程中的任意后续操作。
* 监视器锁规则：对一个锁的解锁，`happens-before`于随后对这个锁的加锁。
* volatile变量规则：对一个`volatile 域`的写，`happens-before`于任意后续对这个`volatile 域`的读。
* 传递性：如果A `happens-before` B，且B `happens-before` C，那么A `happens-before` C。
* start()规则：如果`线程A`执行操作`ThreadB.start()`（启动`线程B`），那么`A线程`的`ThreadB.start()`操作`happens-before`于`线程B`中的任意操作。
* join()规则：如果`线程A`执行操作`ThreadB.join()`并成功返回，那么`线程B`中的任意操作`happens-before`于`线程A`从`ThreadB.join()`操作成功返回。

---

### 双重检查锁定与延迟初始化
双重检查锁定 示例代码：
```
private static Instance instance; //1
public static Instance getInstance() { //2
    if (instance == null) { //3
        synchronized (Instance.class) { //4
            if (instance == null) { //5
                instance = new Instance() //6
            }
        }
    }
    return instance;
}
```
**存在的问题**：
在线程执行到第`3`行` if (instance == null)`，代码读取到`instance`不为`null`时，`instance`引用的对象有可能还没有完成初始化。

**问题的根源**
前面的双重检查锁定示例代码的第`6`行`instance=new Singleton();`创建了一个对象。
这一行可以分解为：
```
memory = allocate();　　// 1：分配对象的内存空间
ctorInstance(memory);　 // 2：初始化对象
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
```
代码中的`2`和`3`之间，可能会被重排序为：
```
memory = allocate();　　// 1：分配对象的内存空间
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
// 注意，此时对象还没有被初始化！
ctorInstance(memory);　 // 2：初始化对象
```

如下图所示，只要保证2排在4的前面，即使2和3之间重排序了，也不会违反`intra-thread semantics`。
![线程执行时序图](https://upload-images.jianshu.io/upload_images/1709375-d3a7e5c0a390567e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果发生重排序，另一个并发执行的线程B就有可能在示例代码第 `3` 行`if (instance == null) `判断`instance`不为`null`。

**解决方法**：
* 不允许图中 `2` 和 `3` 重排序。
* 允许图中 `2` 和 `3` 重排序，但不允许其他线程“看到”这个重排序。


**基于volatile的解决方案**
只需要给变量 `instance` 添加 `volatile` 修饰符。
```
private volatile static Instance instance;
public static Instance getInstance() {
    if (instance == null) {
        synchronized (Instance.class) {
            if (instance == null) {
                instance = new Instance();
            }
        }
    }
    return instance;
}
```

**基于类初始化的解决方案**
```
public static class InstanceFactory {
    public static Instance getInstance() {
        // 这里将导致InstanceHolder类被初始化
        return InstanceHolder.instance;
    }
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
}
```

初始化一个类，包括执行这个类的静态初始化和初始化在这个类中声明的静态字段。根据Java语言规范，在首次发生下列任意一种情况时，一个类或接口类型`T`将被立即初始化。
* `T`是一个类，而且一个`T`类型的实例被创建。
* `T`是一个类，且`T`中声明的一个静态方法被调用。
* `T`中声明的一个静态字段被赋值。
* `T`中声明的一个静态字段被使用，而且这个字段不是一个常量字段。
* `T`是一个顶级类（Top Level Class，见Java语言规范的`§7.6`），而且一个断言语句嵌套在`T`内部被执行。

在`InstanceFactory`示例代码中，首次执行`getInstance()`方法的线程将导致`InstanceHolder`类被初始化（符合第`4`条）。

由于Java语言是多线程的，多个线程可能在同一时间尝试去初始化同一个类或接口。
因此，在Java中初始化一个类或者接口时，需要做细致的同步处理。

Java语言规范规定，对于每一个类或接口`C`，都有一个唯一的初始化锁`LC`与之对应。从`C`到`LC`的映射，由JVM的具体实现去自由实现。JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类已经被初始化过了。

---

### Java内存模型综述

##### 处理器的内存模型
**顺序一致性内存模型** 是一个 **理论参考模型**，JMM和处理器内存模型在设计时通常会以顺序一致性内存模型为参照。
在设计时，JMM和处理器内存模型会 **对顺序一致性模型做一些放松**，因为如果完全按照顺序一致性模型来实现处理器和JMM，那么很多的处理器和编译器优化都要被禁止，这对执行性能将会有很大的影响。

根据对不同类型的读/写操作组合的执行顺序的放松，可以把常见处理器的内存模型划分为如下几种类型：
* 放松程序中写-读操作的顺序，由此产生了`Total Store Ordering`内存模型（简称为 **TSO**）。
* 在上面的基础上，继续放松程序中写-写操作的顺序，由此产生了`Partial Store Order`内存模型（简称为 **PSO**）。
* 在前面两条的基础上，继续放松程序中读-写和读-读操作的顺序，由此产生了`Relaxed Memory Order`内存模型（简称为 **RMO**）和 **PowerPC** 内存模型。

![处理器内存模型的特征表](https://upload-images.jianshu.io/upload_images/1709375-e03e51edfcf9e303.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中可知：
* 所有处理器内存模型都允许`写-读`重排序，因为都使用了 **写缓存区**。
  由于写缓存区仅对当前处理器可见，这个特性导致当前处理器可以比其他处理器先看到临时保存在自己写缓存区中的写。
* 从上到下，模型由强变弱。越是追求性能的处理器，内存模型设计得会越弱。

由于常见的处理器内存模型比JMM要弱，Java编译器在生成字节码时，会在执行指令序列的适当位置插入 **内存屏障** 来限制处理器的重排序。
![JMM插入内存屏障的示意图](https://upload-images.jianshu.io/upload_images/1709375-e0e41118acae2ce3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 各种内存模型之间的关系
**JMM** 是一个语言级的内存模型。
**处理器内存模型** 是硬件级的内存模型。
**顺序一致性内存模型** 是一个理论参考模型。

从下图可以看出：
常见的`4`种 **处理器内存模型** 比常用的`3`中 **语言内存模型** 要 **弱**，
**处理器内存模型** 和 **语言内存模型** 都比 **顺序一致性内存模型** 要 **弱**。
同处理器内存模型一样，越是追求执行性能的语言，内存模型设计得会越弱。
![各种CPU内存模型的强弱对比示意图](https://upload-images.jianshu.io/upload_images/1709375-0403d1c31514bc5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### JMM的内存可见性保证
按程序类型，Java程序的内存可见性保证可以分为下列3类：
* **单线程程序**：不会出现内存可见性问题。JMM为它们提供了最小安全性保障：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（`0`、`null`、`false`）。
* **正确同步的多线程程序**：程序的执行将具有顺序一致性。这是JMM关注的重点，JMM通过限制编译器和处理器的重排序来为程序员提供内存可见性保证。
* **未同步/未正确同步的多线程程序**：JMM为它们提供了最小安全性保障：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（`0`、`null`、`false`）。


##### JSR-133对旧内存模型的修补
**JSR-133** 对 **JDK 5** 之前的旧内存模型的修补主要有两个：
* 增强`volatile`的内存语义：限制`volatile`变量与普通变量的重排序，使`volatile`的写-读和锁的释放-获取具有相同的内存语义。
* 增强`final`的内存语义：保证final引用不会从构造函数内逸出的情况下，`final`具有了初始化安全性。

---

### 小结
本文我们介绍了：
*  线程之后如何通信以及同步？
* 命令式编程的两种通信机制：**共享内存** 和  **消息传递**。 
  Java并发采用的是 共享内存，通信时隐式进行的。
* Java内存模型的抽象结构。 存储在堆内存的`实例域`、`静态域`和`数组元素`等才能在线程间共享。
* 三种类型的重排序。
* **happens-before** 规则
* **volatile**的特性、内存语义以及实现。
* 锁的内存语义
* **final** 域的内存语义
* 基于 **volatile** 和 **类初始化** 两种方式来处理 **单例模式** 双重检查锁定的优化，以及出现问题的根源介绍。
* 对各内存模型的介绍和对比。


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
