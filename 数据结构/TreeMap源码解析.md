>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)

### 目录
* 红黑树简介
* `TreeMap`简介
* `TreeMap`的成员变量介绍
* `TreeMap`的构造函数
* `TreeMap`相关的函数
* 小结
* 参考文章

---

### 红黑树简介
[红黑树](https://link.jianshu.com?t=http%3A%2F%2Fwww.cnblogs.com%2Fskywang12345%2Fp%2F3245399.html) 就是一种平衡的二叉查找树，说他平衡的意思是他不会变成“瘸子”，左腿特别长或者右腿特别长。除了符合二叉查找树的特性之外，还具体下列的特性：
* 节点是红色或者黑色
* 根节点是黑色
* 每个叶子的节点都是黑色的空节点（NULL）
* 每个红色节点的两个子节点都是黑色的。
* 从任意节点到其每个叶子的所有路径都包含相同的黑色节点。

![红黑树示例](https://upload-images.jianshu.io/upload_images/1709375-91b27d08fcef07e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---

### TreeMap简介
```
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```
>`TreeMap` 是一个 **有序的key-value集合**，它是通过 [红黑树](https://link.jianshu.com?t=http%3A%2F%2Fwww.cnblogs.com%2Fskywang12345%2Fp%2F3245399.html) 实现的。
`TreeMap` **继承于AbstractMap**，所以它是一个`Map`，即一个`key-value`集合。
`TreeMap` 实现了`NavigableMap`接口，意味着它 **支持一系列的导航方法。**比如返回有序的`key`集合。
`TreeMap` 实现了`Cloneable`接口，意味着 **它能被克隆**。
`TreeMap` 实现了`java.io.Serializable`接口，意味着 **它支持序列化**。

>`TreeMap`基于**红黑树** 实现。该映射根据 **其键的自然顺序进行排序**，或者根据 **创建映射时提供的 Comparator 进行排序**，具体取决于使用的构造方法。
`TreeMap`的基本操作 `containsKey`、`get`、`put` 和 `remove` 的时间复杂度是 `log(n) `。
另外，`TreeMap`是 **非同步** 的。 它的`iterator` 方法返回的 **迭代器是fail-fastl** 的。

排序默认是升序的，数字比较大小，字符串比较首字母，其他类型则需要自己实现`Comparable`接口，否则排序时会报错。

测试代码：
```
public static void main(String[] args) {
    List<Integer> s = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        int t = (int) (Math.random() * 1000);
        s.add(t);
    }
    
    for (int i = 0; i < s.size(); i++) {
        System.out.print(s.get(i) + "----");
    }
    System.out.println();

    TreeMap<Integer, Integer> treeMap = new TreeMap<>();
    for (int i = 0; i < s.size(); i++) {
        treeMap.put(s.get(i), s.get(i));
    }

    Iterator<Map.Entry<Integer, Integer>> iterator = treeMap.entrySet().iterator();
    while (iterator.hasNext()) {
        Map.Entry<Integer, Integer> next = iterator.next();
        System.out.println(next.getKey() + "--" + next.getValue());
    }
}
```
输出结果：
```
953----411----437----516----461----238----201----365----345----191----
191--191
201--201
238--238
345--345
365--365
411--411
437--437
461--461
516--516
953--953
```
数据类型修改为`String`之后输出结果为：
```
65----296----701----75----278----650----507----71----839----547----
278--278
296--296
507--507
547--547
65--65
650--650
701--701
71--71
75--75
839--839
```

---


### TreeMap的成员变量介绍
```
private final Comparator<? super K> comparator;//key的比较器
private transient TreeMapEntry<K,V> root;//数的根节点
private transient int size = 0;//treemap节点的数量
private transient int modCount = 0;//树修改的次数
private transient EntrySet entrySet;//键值对集合
private transient KeySet<K> navigableKeySet;//键集合
private transient NavigableMap<K,V> descendingMap;//降序的NavigableMap
```

---

### TreeMap的构造函数
`TreeMap`的构造函数主要是处理 `key` 的比较器。
```
public TreeMap() {
    comparator = null;
}
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```

---

### TreeMap相关的函数
```
public V get(Object key)
public Map.Entry<K,V> ceilingEntry(K key)
public Map.Entry<K,V> floorEntry(K key)
public Map.Entry<K,V> higherEntry(K key)
public Map.Entry<K,V> lowerEntry(K key)
public V put(K key, V value)
public V replace(K key, V value)
public V remove(Object key) 
public void clear()
public Map.Entry<K,V> firstEntry()
public Map.Entry<K,V> lastEntry()
private void rotateLeft(TreeMapEntry<K,V> p)
private void rotateRight(TreeMapEntry<K,V> p)
private void fixAfterInsertion(TreeMapEntry<K,V> x)
private void fixAfterDeletion(TreeMapEntry<K,V> x)
```

---

##### get(Object key) and getEntry(Object key)
从根节点开始，比较节点的目标的key的大小，
* `目标key>节点key`：查找节点的右子树
* `目标key<节点key`：查找节点的左子树
* `目标key=节点key`：返回节点

当在没有设置`comparator`的时候，`get`方法的`key = null `会抛出`NullPointerException`.
```
public V get(Object key) {
    TreeMapEntry<K,V> p = getEntry(key);
    return (p==null ? null : p.value);
}
final TreeMapEntry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;
    TreeMapEntry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
final TreeMapEntry<K,V> getEntryUsingComparator(Object key) {
    @SuppressWarnings("unchecked")
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        TreeMapEntry<K,V> p = root;
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
    }
    return null;
}
```
---

#####  ceilingEntry(K key) 
获取 **大于等于** `key`的最小节点。当 `key`比树中最大的`key`大时返回`null`。
```
public Map.Entry<K,V> ceilingEntry(K key) {
    return exportEntry(getCeilingEntry(key));
}
final TreeMapEntry<K,V> getCeilingEntry(K key) {
    TreeMapEntry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp < 0) {
            if (p.left != null)
                p = p.left;
            else
                return p;
        } else if (cmp > 0) {
            if (p.right != null) {
                p = p.right;
            } else {
                TreeMapEntry<K,V> parent = p.parent;
                TreeMapEntry<K,V> ch = p;
                while (parent != null && ch == parent.right) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        } else
            return p;
    }
    return null;
}
```
---

##### floorEntry(K key) 
获取 **小于等于** `key`的最大节点。当 `key`比树中最小的`key`小时返回`null`。
```
public K floorKey(K key) {
    return keyOrNull(getFloorEntry(key));
}
final TreeMapEntry<K,V> getFloorEntry(K key) {
    TreeMapEntry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp > 0) {
            if (p.right != null)
                p = p.right;
            else
                return p;
        } else if (cmp < 0) {
            if (p.left != null) {
                p = p.left;
            } else {
                TreeMapEntry<K,V> parent = p.parent;
                TreeMapEntry<K,V> ch = p;
                while (parent != null && ch == parent.left) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        } else
            return p;

    }
    return null;
}
```
---

##### higherEntry(K key)
获取 **大于** `key`的最小节点，不存在则返回 `null`
```
public K higherKey(K key) {
    return keyOrNull(getHigherEntry(key));
}
final TreeMapEntry<K,V> getHigherEntry(K key) {
    TreeMapEntry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp < 0) {
            if (p.left != null)
                p = p.left;
            else
                return p;
        } else {
            if (p.right != null) {
                p = p.right;
            } else {
                TreeMapEntry<K,V> parent = p.parent;
                TreeMapEntry<K,V> ch = p;
                while (parent != null && ch == parent.right) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        }
    }
    return null;
}
```
---

##### lowerEntry(K key)
获取 **小于** `key`的最大节点，不存在则返回`null`.
```
public K lowerKey(K key) {
    return keyOrNull(getLowerEntry(key));
}
final TreeMapEntry<K,V> getLowerEntry(K key) {
    TreeMapEntry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp > 0) {
            if (p.right != null)
                p = p.right;
            else
                return p;
        } else {
            if (p.left != null) {
                p = p.left;
            } else {
                TreeMapEntry<K,V> parent = p.parent;
                TreeMapEntry<K,V> ch = p;
                while (parent != null && ch == parent.left) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        }
    }
    return null;
}
```

---

### put(K key, V value)
* 如果树为`null`，则构建一个`TreeMapEntry`设置为当前的`root`.
* 检查之前`TreeMap`的`comparator`是否为空，不为空则用`comparator`去对比`key`,否则用`k.compareTo(t.key)`比较，然后遍历当前树，找到对应的`key`则修改对应的`value`。
* 如果遍历树没找到，则通过`new TreeMapEntry<>(key, value, parent);` 添加到树上，然后执行`fixAfterInsertion(e)`保证`root`还是一颗 [红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml)。`fixAfterInsertion(e)`下面会介绍。
>红黑树执行插入操作之后，要执行“插入修正操作”。
目的是：保红黑树在进行插入节点之后，仍然是一颗红黑树
```
public V put(K key, V value) {
    TreeMapEntry<K, V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new TreeMapEntry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    TreeMapEntry<K, V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    } else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    TreeMapEntry<K, V> e = new TreeMapEntry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

---

### replace(K key, V value)
通过`getEntry(key)`获取对应的节点，如果节点不为空，则更新值。
```
public V replace(K key, V value) {
    TreeMapEntry<K,V> p = getEntry(key);
    if (p!=null) {
        V oldValue = p.value;
        p.value = value;
        return oldValue;
    }
    return null;
}
```

---

### remove(Object key) 
通过`getEntry(key)`获取对应的节点，如果节点不为空，通过`deleteEntry`删除节点，并通过`fixAfterDeletion(TreeMapEntry<K,V> x)`重排树使之还是一颗 [红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml)。`fixAfterDeletion`下面会介绍。
>红黑树执行删除之后，要执行“删除修正操作”。
目的是保证：红黑树删除节点之后，仍然是一颗红黑树

* 当`p`有左右子树的时候，`successor(p)`，及返回右子树上的最左边的树节点，即大于`p`的`key`的最小值。所以 `replacement`即为右子树上的最左边的树节点的右子树。否则 `replacement`即为`p`的左右子树中的一个。
* 然后根据 `replacement` **不为空**、**是否是根节点**、**为空且不是根节点**  3种情况删除节点`p`.
```
public V remove(Object key) {
    TreeMapEntry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
private void deleteEntry(TreeMapEntry<K, V> p) {
    modCount++;
    size--;
    if (p.left != null && p.right != null) {// p has 2 children
        TreeMapEntry<K, V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    }
    TreeMapEntry<K, V> replacement = (p.left != null ? p.left : p.right);
    if (replacement != null) {
        // Link replacement to parent
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left = replacement;
        else
            p.parent.right = replacement;

        p.left = p.right = p.parent = null;
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

---

### clear()
置空`root`，修改 `size` 为 `0`.
```
public void clear() {
    modCount++;
    size = 0;
    root = null;
}
```

---

### firstEntry()
返回最左端的节点的`key-value`构建的`SimpleImmutableEntry`
```
public Map.Entry<K, V> firstEntry() {
    return exportEntry(getFirstEntry());
}
final TreeMapEntry<K, V> getFirstEntry() {
    TreeMapEntry<K, V> p = root;
    if (p != null)
        while (p.left != null)
            p = p.left;
    return p;
}
static <K, V> Map.Entry<K, V> exportEntry(TreeMapEntry<K, V> e) {
    return (e == null) ? null :
            new AbstractMap.SimpleImmutableEntry<>(e);
}
```

---

### lastEntry()
返回最右端的节点的`key-value`构建的`SimpleImmutableEntry`
```
public Map.Entry<K,V> lastEntry() {
    return exportEntry(getLastEntry());
}
final TreeMapEntry<K,V> getLastEntry() {
    TreeMapEntry<K,V> p = root;
    if (p != null)
        while (p.right != null)
            p = p.right;
    return p;
}
static <K, V> Map.Entry<K, V> exportEntry(TreeMapEntry<K, V> e) {
    return (e == null) ? null :
            new AbstractMap.SimpleImmutableEntry<>(e);
}
```

---

### rotateLeft(TreeMapEntry<K,V> p)
[红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml) 的左旋
![红黑树的左旋](https://upload-images.jianshu.io/upload_images/1709375-4762ae6b2f13b2b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**`p` 相当与上图的`X`.**

```
private void rotateLeft(TreeMapEntry<K,V> p) {
    if (p != null) {
        TreeMapEntry<K,V> r = p.right;
        p.right = r.left;
        if (r.left != null)
            r.left.parent = p;
        r.parent = p.parent;
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}
```

---

### rotateRight(TreeMapEntry<K,V> p)
[红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml) 的右旋
![红黑树的右旋](https://upload-images.jianshu.io/upload_images/1709375-6460dd0996004b99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**`p` 相当与上图的`X`.**

```
private void rotateRight(TreeMapEntry<K,V> p) {
    if (p != null) {
        TreeMapEntry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

---

### fixAfterInsertion(TreeMapEntry<K,V> x)
* 插入的节点，默认设置为`红树`。当`x.parent.color == RED`时，则需要进行旋转知道满足 [红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml) 的条件。旋转的过程可以参考链接中的示例。
```
private void fixAfterInsertion(TreeMapEntry<K, V> x) {
    x.color = RED;

    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            TreeMapEntry<K, V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            TreeMapEntry<K, V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```

---

### fixAfterDeletion(TreeMapEntry<K,V> x)
* 删除的节点只有是 `黑树` 时，才需要进行旋转重新满足 [红黑树](http://www.360doc.com/content/18/0904/19/25944647_783893127.shtml) 的条件。旋转的过程可以参考链接中的示例。
```
private void fixAfterDeletion(TreeMapEntry<K, V> x) {
    while (x != root && colorOf(x) == BLACK) {
        if (x == leftOf(parentOf(x))) {
            TreeMapEntry<K, V> sib = rightOf(parentOf(x));
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }
            if (colorOf(leftOf(sib)) == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            TreeMapEntry<K, V> sib = leftOf(parentOf(x));
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }
            if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

---

### 小结
* `TreeMap`是有序的`key-value`集合。
* `HashMap`不保证数据有序，`LinkedHashMap`保证数据可以保持插入顺序，而`TreeMap`可以保持`key`的大小顺序的时候。

---

### 参考文章
* [JDK源码解析——TreeMap](https://www.jianshu.com/p/f2a0946eead1)
* [Java 集合系列12之 TreeMap详细介绍(源码解析)和使用示例](https://www.cnblogs.com/skywang12345/p/3310928.html)


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
