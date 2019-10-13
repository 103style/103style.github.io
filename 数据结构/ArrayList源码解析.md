>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)



### 目录
* 简介
* `ArrayList`的常量介绍
* `ArrayList`的构造函数
* `ArrayList`的数据操作函数
* 小结
* 参考文章

---
### 简介
>`ArrayList` 可以理解为动态数组，用 MSDN 中的说法，就是 `Array` 的复杂版本。与 Java 中的数组相比，它的容量能动态增长。`ArrayList` 是 `List` 接口的可变数组的实现。实现了所有可选列表操作，并允许包括 `null` 在内的所有元素。除了实现 `List` 接口外，此类还提供一些方法来操作内部用来存储列表的数组的大小。（此类大致上等同于 `Vector` 类，除了此类是不同步的。）

> 每个 `ArrayList` 实例都有一个 **容量**，该容量是指用来存储列表元素的数组的大小。它总是至少等于列表的大小。随着向 `ArrayList` 中不断添加元素，其容量也自动增长。自动增长会带来数据向新数组的重新拷贝，因此，如果可预知数据量的多少，可在构造 ArrayList 时指定其容量。在添加大量元素前，应用程序也可以使用 `ensureCapacity` 操作来增加 `ArrayList` 实例的容量，这可以减少递增式再分配的数量。

>注意，此实现不是同步的。如果多个线程同时访问一个 ArrayList 实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。（结构上的修改是指任何添加或删除一个或多个元素的操作，或者显式调整底层数组的大小；仅仅设置元素的值不是结构上的修改。）


---

### ArrayList的常量介绍

* `private static final int DEFAULT_CAPACITY = 10;`

    默认的初始化长度。

* `private static final Object[] EMPTY_ELEMENTDATA = {};`

    初始化长度为0时的数组。

* `private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};`

    空数据时数组。

* `private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;`

    `ArrayList`的最大长度。

---

### ArrayList的构造函数

```
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                initialCapacity);
    }
}
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
* 无参数的构造函数，直接将全局变量`elementData`赋值为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`。
* 带长度参数的构造函数：
    * 长度为`0`时将`elementData`赋值为`EMPTY_ELEMENTDATA`
    * 大于`0` 时 则构建一个对应长度的数据
    * 小于`0`则抛出异常
* 带`Collection`参数的构造函数，先将参数转化为数组：
    * 如果长度为`0`，将`elementData`赋值为`EMPTY_ELEMENTDATA`
    * 长度不为`0` 并且不是 `Object[].class`，则将`elementData`赋值为参数转化的数组

---

### ArrayList的数据操作函数
* `grow`：扩容函数
* `indexOf` and `lastIndexOf`：寻找对象对应的索引
* `add(E e)` and `add(int index, E element)`：新增数据
* `addAll(Collection<? extends E> c)` and `addAll(int index, Collection<? extends E> c)`：新增多条数据
* `set(int index, E element)`：修改数据
* `get(int index)`：查询数据
* `remove(int index)`： 移除数据
* `removeRange(int fromIndex, int toIndex)`： 移除多条数据

#####  grow 扩容函数
* `(1.0)` 通过`oldCapacity + (oldCapacity >> 1)`每次扩容增加原长度的一半。
* 如果扩容之后的长度比`minCapacity`小，则赋值长度为`minCapacity`。
* 如果扩容之后的长度大于`MAX_ARRAY_SIZE`，则根据`minCapacity`和 `MAX_ARRAY_SIZE`的大小选择`Integer.MAX_VALUE`还是`MAX_ARRAY_SIZE`.
```
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); //1.0
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
}
```

##### indexOf  and lastIndexOf  查找索引值
一个从头开始，一个从尾开始。对`null` 做 `==` 查找索引值，否则用 `equals` 查找索引值。未找到则返回 `-1`。
```
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

##### add(E e) and add(int index, E element)  新增数据
* 先通过`ensureCapacityInternal`检查是否需要做扩容操作。
* `add(E e)`则是直接添加在数组的最后一个数据之后
* `add(int index, E element)`则是把`index`之后的数据都往后移一位，然后再将 `element`保存在`index` 的位置上。通过`System.arraycopy`进行数组拷贝操作。
```
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
            size - index);
    elementData[index] = element;
    size++;
}
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

##### addAll(Collection<? extends E> c) 和 addAll(int index, Collection<? extends E> c) 新增多个数据
* 也是先通过`ensureCapacityInternal`检查是否需要做扩容操作。 
* `addAll(Collection<? extends E> c)`在检查扩容之后直接通过`System.arraycopy`赋值到原有数组的后面
* 带 索引的方法，则通过`size - index > 0` 来判断是否需要移位操作，然后再通过`System.arraycopy`从`index`位置开始赋值到原数组。
```
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}

public boolean addAll(int index, Collection<? extends E> c) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

##### set(int index, E element) 修改数据
* 先判断索引是否超出范围，然后直接修改`index`处的值，并返回之前的值。
```
public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    E oldValue = (E) elementData[index];
    elementData[index] = element;
    return oldValue;
}
```

##### get(int index) 获取数据
* 先判断索引是否超出范围，然后直返回 `index` 处的值。
```
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index];
}
```

##### remove(int index)  删除数据
* `remove(int index)`如果是删除最后一个数据之前的数据，则会进行移位操作， 然后将最后一个数据修改位`null`。
* `remove(Object o)`则会从头开始找 `o` 所在的索引，然后进行类似`remove(int index)`的操作。
```
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    modCount++;
    E oldValue = (E) elementData[index];

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}

public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

##### removeRange(int fromIndex, int toIndex) 删除多条数据
* 通过`System.arraycopy`将 `toIndex`之后的数据 向前 移动 `toIndex-fromIndex`位，然后将最后的  `toIndex-fromIndex` 个数据设置为 `null`。
```
protected void removeRange(int fromIndex, int toIndex) {
    if (toIndex < fromIndex) {
        throw new IndexOutOfBoundsException("toIndex < fromIndex");
    }
    modCount++;
    int numMoved = size - toIndex;
    System.arraycopy(elementData, toIndex, elementData, fromIndex, numMoved);
    int newSize = size - (toIndex-fromIndex);
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    size = newSize;
}
```

---

### 小结
* 无参构造方法默认为一个 空数组。
* 数组每次扩容为增加原长度的一半。
* 基本上都是通过`System.arraycopy`进行数据操作

---

### 参考文章
* [ArrayList 的实现原理](http://wiki.jikexueyuan.com/project/java-collection/arraylist.html)


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
