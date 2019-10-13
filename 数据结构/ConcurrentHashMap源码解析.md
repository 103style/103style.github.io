>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)

### 目录
* `ConcurrentHashMap`的用途
* `ConcurrentHashMap`的常量介绍
* `ConcurrentHashMap`的相关函数
* 小结
* 参考文章

---

### ConcurrentHashMap简介
`ConcurrentHashMap` 是在 `HashMap` 的线程安全的版本，不允许 **空键空值**。

和`HashMap`类似，`ConcurrentHashMap`使用了一个`table`来存储`Node`，`ConcurrentHashMap`同样使用记录的`key`的`hashCode`来寻找记录的存储`index`，而处理哈希冲突的方式与`HashMap`也是类似的，冲突的记录将被存储在同一个位置上，形成一条链表，当链表的长度大于`8`的时候会将链表转化为一棵 **红黑树**，从而将查找的复杂度从`O(N)`降到了`O(lgN)`。

![ConcurrentHashMap的结构](https://upload-images.jianshu.io/upload_images/1709375-01dc6913ad934297.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来将详细分析`ConcurrentHashMap`的主要操作方法，以及`ConcurrentHashMap`是如何保证在并发环境下的线程安全的。

---

### ConcurrentHashMap的常量介绍
*  `static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;`

    能转化为数组的最大长度  **2<sup>31</sup> -1 - 8**.

* `private static final int DEFAULT_CONCURRENCY_LEVEL = 16;`

    默认的并发等级

* `private static final int MIN_TRANSFER_STRIDE = 16;`
    
    最小迁移步幅，只在`transfer`方法中用到

* `private static final int RESIZE_STAMP_BITS = 16;`
    
    用于记录`sizeCtl`中的 `resize stamp`

* `private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;`
   
    参与`resize`的最大线程数，如果是默认值，那么位 `(1 << (32-16)) - 1 = 32 - 1 = 31`

* `private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;`
    
    用于记录`sizeCtl`中的 `resize stamp`偏移位

* ` static final int HASH_BITS = 0x7fffffff;`
    哈希散列的值

和 [HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 相同的常量：
```
//链表树化的阈值
static final int TREEIFY_THRESHOLD = 8;
//树边链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//树化时的最小容量
static final int MIN_TREEIFY_CAPACITY = 64;
//最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;
//默认容量
private static final int DEFAULT_CAPACITY = 16;
//负载因子
private static final float LOAD_FACTOR = 0.75f;
```

---


### ConcurrentHashMap的相关函数
* `spread(int h)`：散列计算
* `tableSizeFor(int c)`：根据传入的值计算`ConcurrentHashMap`的容量
* `size()`：计算`ConcurrentHashMap`的大小
* ` initTable()`：初始化`table`
* `get(Object key)`：取数据
* `put(K key, V value)`：存数据
* `remove(Object key)`：移除数据

##### spread(int h)  散列计算
作用同`HashMap`的 `hash`方法。
```
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

##### tableSizeFor(int c)  根据传入的值计算容量
即为计算大于等于传入数的最小的 **2<sup>n</sup>** 的值。
```
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

##### size()  计算大小
对`CounterCell`数组中每一项求和。
```
/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
private transient volatile CounterCell[] counterCells;
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                    (int)n);
}
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

#####  initTable()  初始化table
* `(1.0)` 当`sizeCtl < 0`时，则会让出CPU的时间，等待下次重试。
*  `(2.0)` 当`sizeCtl >= 0`时，则通过 `CAS` 原子操作将 `sizeCtl` 设置为 `-1`。
*  `(3.0)` 设置成功之后，如果table为空，则初始化对应长度的 `Node`数组。并将 当前容量能保存的最大数量(`n * LOAD_FACTOR`)赋值给`sizeCtl`
```
//-1：正在初始化 ,其他小于0的数则表示正在调整大小
private transient volatile int sizeCtl;
//Initializes table, using the size recorded in sizeCtl.
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)//1.0
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//2.0
            try {
                if ((tab = table) == null || tab.length == 0) {//3.0
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

##### get(Object key)  取数据
和[HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 类似，当前的索引处时要获取的值则返回当前值， 否则遍历链表。

![ConcurrentHashMap get操作](https://upload-images.jianshu.io/upload_images/1709375-19a13f3946c91c6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```


##### put(K key, V value)  存数据
![ConcurrentHashMap put操作](https://upload-images.jianshu.io/upload_images/1709375-e624412f6a275277.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public V put(K key, V value) {
    return putVal(key, value, false);
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

##### remove(Object key)  移除数据
![ConcurrentHashMap remove操作](https://upload-images.jianshu.io/upload_images/1709375-3b05ed0bd8f1d1db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public V remove(Object key) {
    return replaceNode(key, null, null);
}
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    for (Node<K, V>[] tab = table; ; ) {
        Node<K, V> f;
        int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K, V> e = f, pred = null; ; ) {
                            K ek;
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    } else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K, V> t = (TreeBin<K, V>) f;
                        TreeNode<K, V> r, p;
                        if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    } else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

---

### 小结
通过上面对 `ConcurrentHashMap`  **添加和删除数据** 的方法的介绍，我们知道在对其进行 **写操作** 的时候，会对当前索引出的节点通过 `CAS` 或者 `synchronized` 来实现多线程同步。在 读取数据 的方法则不会做这些同步操作。


---

### 参考文章
* [Java 8 ConcurrentHashMap源码分析](https://www.jianshu.com/p/cf5e024d9432)

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
