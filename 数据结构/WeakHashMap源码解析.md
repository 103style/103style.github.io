# WeakHashMap源码解析 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)

### 目录
* `WeakHashMap`简介
* `WeakHashMap`的常量、成员变量介绍
* `WeakHashMap`的构造函数
* `WeakHashMap`相关的函数
* 小结
* 参考文章

---

### WeakHashMap简介
>`WeakHashMap` 继承于`AbstractMap`，实现了`Map`接口。

>和 [HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 一样，`WeakHashMap` 也是一个**散列表**，它存储的内容也是**键值对(key-value)映射**，而且**键和值都可以是null**。

>不过`WeakHashMap`的**键是“弱键”**。在 `WeakHashMap` 中，当某个键不再正常使用时，会被从`WeakHashMap`中被自动移除。更精确地说，对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的丢弃，这就使该键成为可终止的，被终止，然后被回收。某个键被终止时，它对应的键值对也就从映射中有效地移除了。

>这个“弱键”的原理呢？大致上就是，**通过WeakReference和ReferenceQueue实现的**。 `WeakHashMap` 的 `key` 是“弱键”，即是 `WeakReference` 类型的；`ReferenceQueue`是一个队列，它会保存被GC回收的“弱键”。实现步骤是：
>* 新建`WeakHashMap`，将“**键值对**”添加到`WeakHashMap`中。
    实际上，`WeakHashMap`是通过数组`table`保存`Entry`(键值对)；每一个`Entry`实际上是一个单向链表，即`Entry`是键值对链表。
>* 当**某“弱键”不再被其它对象引用**，并**被GC回收**时。在GC回收该“弱键”时，**这个“弱键”也同时会被添加到ReferenceQueue(queue)队列**中。
>* 当下一次我们需要操作`WeakHashMap`时，会先同步`table`和`queue`。`table`中保存了全部的键值对，而`queue`中保存被GC回收的键值对；同步它们，就是**删除table中被GC回收的键值对**。
    这就是“弱键”如何被自动从`WeakHashMap`中删除的步骤了。

>和 [HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 一样，`WeakHashMap`是不同步的。可以使用 `Collections.synchronizedMap` 方法来构造同步的 `WeakHashMap`。

---

### WeakHashMap的常量、成员变量介绍
和 [HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 一样得常量：
* ` private static final int DEFAULT_INITIAL_CAPACITY = 16;`
    
    默认初始化的`tab`长度

* ` private static final int MAXIMUM_CAPACITY = 1 << 30;`
    
    最大得容量值

* `private static final float DEFAULT_LOAD_FACTOR = 0.75f;`

    默认的加载因子


`WeakHashMap`的全局变量：
* `Entry<K,V>[] table;`

    保存链表的数组

* `private int size;`
    
    当前`WeakHashMap`中键值对的数量

* `private int threshold;`

    扩容阈值

* `private final float loadFactor;`

    当前`WeakHashMap`的加载因子

* `private final ReferenceQueue<Object> queue = new ReferenceQueue<>();`

    被清除的`entry`引用队列

* `int modCount;`

    修改的次数

* `private static final Object NULL_KEY = new Object();`

    空键

* ` private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> `

    继承自`WeakReference`的

---

### WeakHashMap的构造函数
```
public WeakHashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
public WeakHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public WeakHashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Initial Capacity: "+
                initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load factor: "+
                loadFactor);
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;
    table = newTable(capacity);
    this.loadFactor = loadFactor;
    threshold = (int)(capacity * loadFactor);
}
public WeakHashMap(Map<? extends K, ? extends V> m) {
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
            DEFAULT_INITIAL_CAPACITY),
            DEFAULT_LOAD_FACTOR);
    putAll(m);
}
```
通过下面的代码计算合适的`table`长度(`大于指定容量的最小的2的指数幂`)。其他和 [HashMap](https://www.jianshu.com/p/d4fee00fe2f8) 类似。
```
int capacity = 1;
while (capacity < initialCapacity)
    capacity <<= 1;
```

---

### WeakHashMap相关的函数
```
int                    hash(Object k)//计算key的hash
int                    indexFor(int h, int length)//计算key再table上的索引
void                   expungeStaleEntries()//删除引用队列中entry
Entry<K,V>[]           getTable()//获取当前table
void                   clear()
V                      get(Object key)
V                      put(K key, V value)
void                   putAll(Map<? extends K, ? extends V> map)
V                      remove(Object key)
boolean                isEmpty()
int                    size()
void                   resize(int newCapacity)
boolean                containsKey(Object key)
boolean                containsValue(Object value)
void                   transfer(Entry<K,V>[] src, Entry<K,V>[] dest)
```

##### hash(Object k)
函数中注释的解释大致是：
>此函数确保在每个位位置仅由常数倍数相差的哈希码具有有限数量的冲突
```
final int hash(Object k) {
    int h = k.hashCode();
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

##### indexFor(int h, int length)
计算对应`hash`在`table`的索引位置。
```
private static int indexFor(int h, int length) {
    return h & (length-1);
}
```

##### expungeStaleEntries()
删除`table`中有的 `queue`中的`entry`。
```
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);
            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

##### getTable()
获取当前的数组`table`，并在返回前删除引用队列中`entry`
```
private Entry<K,V>[] getTable() {
    expungeStaleEntries();
    return table;
}
```

##### clear()
先清空引用队列`queue`中的元素，然后将`table`中的数据全部设置为`null`，然后再次清空引用队列`queue`中的元素。
```
public void clear() {
    // clear out ref queue. We don't need to expunge entries
    // since table is getting cleared.
    while (queue.poll() != null)
        ;

    modCount++;
    Arrays.fill(table, null);
    size = 0;

    // Allocation of array may have caused GC, which may have caused
    // additional entries to go stale.  Removing these entries from the
    // reference queue will make them eligible for reclamation.
    while (queue.poll() != null)
        ;
}
```

##### get(Object key)
通过`key`的`hash`找到索引，然后遍历链表找到对应的值。
```
public V get(Object key) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int index = indexFor(h, tab.length);
    Entry<K,V> e = tab[index];
    while (e != null) {
        if (e.hash == h && eq(k, e.get()))
            return e.value;
        e = e.next;
    }
    return null;
}
```

##### put(K key, V value)
和 `get`类似，通过`key`的`hash`找到索引，然后检查链表是否有对应的`key`,有个话更新对应的值。

没有的话就通过`new Entry<>(k, value, queue, h, e)`构建一个新的节点添加到之前的链表前面。然后再判断是否要扩容。
```
public V put(K key, V value) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length);

    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        if (h == e.hash && eq(k, e.get())) {
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;
        }
    }

    modCount++;
    Entry<K,V> e = tab[i];
    tab[i] = new Entry<>(k, value, queue, h, e);
    if (++size >= threshold)
        resize(tab.length * 2);
    return null;
}
```

##### putAll(Map<? extends K, ? extends V> map)
用`if (numKeysToBeAdded > threshold)`判断是为了避免 `m`中有重复的`key`.
```
public void putAll(Map<? extends K, ? extends V> m) {
    int numKeysToBeAdded = m.size();
    if (numKeysToBeAdded == 0)
        return;
    
    if (numKeysToBeAdded > threshold) {
        int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
        if (targetCapacity > MAXIMUM_CAPACITY)
            targetCapacity = MAXIMUM_CAPACITY;
        int newCapacity = table.length;
        while (newCapacity < targetCapacity)
            newCapacity <<= 1;
        if (newCapacity > table.length)
            resize(newCapacity);
    }

    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        put(e.getKey(), e.getValue());
}
```

##### remove(Object key)
和`put`类似，`put`是添加或者修改节点，`remove`则为删除节点。
```
public V remove(Object key) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length);
    Entry<K,V> prev = tab[i];
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        if (h == e.hash && eq(k, e.get())) {
            modCount++;
            size--;
            if (prev == e)
                tab[i] = next;
            else
                prev.next = next;
            return e.value;
        }
        prev = e;
        e = next;
    }

    return null;
}
```

##### resize(int newCapacity)
扩容操作，如果忽略`null`元素并处理引用队列导致大量收缩，则恢复旧表。
```
void resize(int newCapacity) {
    Entry<K,V>[] oldTable = getTable();
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry<K,V>[] newTable = newTable(newCapacity);
    transfer(oldTable, newTable);
    table = newTable;
    
    if (size >= threshold / 2) {
        threshold = (int)(newCapacity * loadFactor);
    } else {
        expungeStaleEntries();
        transfer(newTable, oldTable);
        table = oldTable;
    }
}
```
##### transfer(Entry<K,V>[] src, Entry<K,V>[] dest)
删除`src`中的`null key`元素，并将其他元素放到`dest`中的对应位置。
```
private void transfer(Entry<K,V>[] src, Entry<K,V>[] dest) {
    for (int j = 0; j < src.length; ++j) {
        Entry<K,V> e = src[j];
        src[j] = null;
        while (e != null) {
            Entry<K,V> next = e.next;
            Object key = e.get();
            if (key == null) {
                e.next = null;  // Help GC
                e.value = null; //  "   "
                size--;
            } else {
                int i = indexFor(e.hash, dest.length);
                e.next = dest[i];
                dest[i] = e;
            }
            e = next;
        }
    }
}
```

---

### 小结
* `WeakHashMap`也是一个 `数组` + `单链表` 的结构。
* 相比`HashMap`，`WeakHashMap`在每次操作时基本上都有去移除被gc回收的`key`.
* `WeakHashMap`没有实现`HashMap`的树化操作
* 在原有链表添加数据时，`HashMap`添加在尾端，而`WeakHashMap`添加在前端。

通过下图我们发现基本所有的数据操作都调用了`expungeStaleEntries()`来移除被gc回收的`key`.
![getTable](https://upload-images.jianshu.io/upload_images/1709375-80755a01f92a9c0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![expungeStaleEntries](https://upload-images.jianshu.io/upload_images/1709375-76c32e4e3647cc59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---

### 参考文章
* [WeakHashMap详细介绍](https://www.cnblogs.com/skywang12345/p/3311092.html)
* [WeakHashMap和HashMap的区别](https://blog.csdn.net/yangzl2008/article/details/6980709)

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
