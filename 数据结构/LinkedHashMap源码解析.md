>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)

### 目录
* `LinkedHashMap`简介
* `LinkedHashMap`的成员变量介绍
* `LinkedHashMap`的构造函数
* `LinkedHashMap`重写的函数
* 小结
* 参考文章

---

### LinkedHashMap简介
`HashMap` 是无序的，`HashMap` 在 `put` 的时候是根据 `key` 的 `hashcode` 进行 `hash` 然后放入对应的地方。所以在按照一定顺序 `put` 进 `HashMap` 中，然后遍历出 `HashMap` 的顺序跟 `put` 的顺序不同。

```
class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```
 `LinkedHashMap` 是 `HashMap` 的子类。它保留插入的顺序，如果需要输出的顺序和输入时的相同，那么就选用 `LinkedHashMap`。

`LinkedHashMap` 是 `Map` 接口的哈希表和链接列表实现，具有可预知的迭代顺序。此实现提供所有可选的映射操作，并允许使用 `null值` 和 `null键`。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。

`LinkedHashMap` 实现与 `HashMap` 的不同之处在于，`LinkedHashMap` 维护着一个运行于所有条目的 **双重链接列表**。此链接列表定义了迭代顺序，该迭代顺序可以是插入顺序或者是访问顺序。

注意，此实现不是同步的。如果多个线程同时访问链接的哈希映射，而其中至少一个线程从结构上修改了该映射，则它必须保持外部同步。

根据链表中元素的顺序可以分为：**按插入顺序的链表**，和 **按访问顺序(`调用 get 方法`)** 的链表。默认是按插入顺序排序，**如果指定按访问顺序排序，那么调用`get`方法后，会将这次访问的元素移至链表尾部**，不断访问可以形成按访问顺序排序的链表。

---


### LinkedHashMap的成员变量介绍
* `transient LinkedHashMapEntry<K,V> head;`
    
    双向链表的头节点

* `transient LinkedHashMapEntry<K,V> tail;`

    双向链表的尾节点

* `final boolean accessOrder;`
    * `false`：按插入顺序排序
    * `true`：按访问顺序排序


* `LinkedHashMapEntry`在`HashMap.Node`的基础上添加了 `before`和`after`节点在实现双向链表。
    ```
    static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    ```

---

### LinkedHashMap的构造函数
相比[HashMap](https://www.jianshu.com/p/d4fee00fe2f8)来说，只是添加了`accessOrder`默认为`false`，以及可以设置`accessOrder`的构造函数。

```
public LinkedHashMap() {
    super();
    accessOrder = false;
}
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}
public LinkedHashMap(int initialCapacity, float loadFactor, 
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

---

### LinkedHashMap重写的函数
`LinkedHashMap`重写了`HashMap`的以下方法来实现按对应排序输出。
* `afterNodeAccess(Node<K,V> p)`：数据访问之后
* `afterNodeInsertion(boolean evict)`：数据插入完成之后
* `afterNodeRemoval(Node<K,V> p)`：数据移除之后
* `get(Object key)` and `getOrDefault(Object key, V defaultValue)`


##### afterNodeAccess
* 如果时按访问顺序排序的话(`accessOrder = true`)，并且当前节点不是尾节点，则将当前节点移动到双向链表末端
```
void afterNodeAccess(Node<K, V> e) {
    LinkedHashMapEntry<K, V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMapEntry<K, V> p =
                (LinkedHashMapEntry<K, V>) e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

##### afterNodeInsertion
* 因为`removeEldestEntry`返回`false` 所以啥也没做.
```
void afterNodeInsertion(boolean evict) {
    LinkedHashMapEntry<K, V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

##### afterNodeRemoval
* 即删除双向队列中的节点
```
void afterNodeRemoval(Node<K, V> e) {
    LinkedHashMapEntry<K, V> p =
            (LinkedHashMapEntry<K, V>) e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

#####  get方法
* 只是在设置了按访问顺序排序时调用了`afterNodeAccess`方法，来做双向两边的变化操作。
```
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return defaultValue;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

---

### 小结
* `LinkedHashMap`只是在`HashMap`的基础上维护了一个双向链表来保证按`accessOrder`对应的顺序来输出。
* 在每次数据操作之后都会修改双向链表来保证对应的顺序。

---

### 参考文章
* [LinkedHashMap 的实现原理](http://wiki.jikexueyuan.com/project/java-collection/linkedhashmap.html)

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
