# LinkedHashSet源码解析 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`base on jdk_1.8.0_77`

[数据结构源码分析汇总](https://www.jianshu.com/p/126a0fe5ace3)

### 简介
![LinkedHashSet](https://upload-images.jianshu.io/upload_images/1709375-eeafc86bc0a7aba7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上图我们知道`LinkedHashSet`是 `HashSet` 的子类，构造方法也是对应的`HashSet`的方法，并且只重写了`spliterator()`方法。

而 `HashSet<E>`实际上就是通过`HashMap`保存 `key` 为`E`，值为`PRESENT = new Object()`。对应的数据操作即为`HashMap` 的 `key` 的操作。

* [HashSet源码解析](https://www.jianshu.com/p/7b68486427b3)
* [HashMap源码解析](https://www.jianshu.com/p/d4fee00fe2f8)

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
