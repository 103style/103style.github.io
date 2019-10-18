# AndroidStudio App出现红叉问题 解决方法记录 

先献上两个报错信息
  * ` Please select Android SDK`
  * `Run configuration Error:Broken configuration due to unavailable plugin or invalid configuration data`

### 正文

今天打开Android Studio 原本好好的项目，就直接编译不过了，如下图, **或者不是叉叉，是个问号**。
![image.png](https://upload-images.jianshu.io/upload_images/1709375-b5030b039b066254.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看了下 **Version control**，也没啥改动的...

然后就网上查了下，记录下解决方案。


* **点击红叉处按钮，选择 Edit Configurations**
你会看到这样的页面，以及**红色框框标注出来的报错信息**
![参考文章里面的截图1](https://upload-images.jianshu.io/upload_images/1709375-44e085c750f76ed9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**笔者先碰到的报错信息如下：** ，
`Run configuration Error:Broken configuration due to unavailable plugin or invalid configuration data`

**大致就是Android Studio的插件出问题了。**
 
**解决方法**
  * 默认快捷键（**ctrl+alt+s**）打开设置页面，或者点击左上角的**File - Setting**。 选中 **plugins**
![image.png](https://upload-images.jianshu.io/upload_images/1709375-3966bb2f1b211301.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 你会发现很多插件都变成红色的了，所以我们先吧 **变红的勾全部取消掉，然后在选上，如果选上还是红色就先不选中**，直接点 **Apply** 再点**ok** ,然后重启Studio。 如果之前没选中的再次选中，然后再Apply -- ok 重启Studio.
 
**如果到这里进来重新编译没问题了，那你很幸运。**

如果进来 **app 上还是有个 红叉**。
我们继续 **点击红叉处按钮，选择 Edit Configurations**
你就会看到  文章开头的 **第一个报错信息**
![参考文章里面的截图](https://upload-images.jianshu.io/upload_images/1709375-44e085c750f76ed9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**解决方法：**
  * 默认快捷键（**ctrl+alt+s**）打开设置页面，或者点击左上角的**File - Setting**。 再左上角输入栏输入 **sdk**，选中**Android sdk**。

  * ![image.png](https://upload-images.jianshu.io/upload_images/1709375-a680cb2927fa330c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  * **然后按下图操作 一直点 就ok了.**
  * ![参考文章的截图2](https://upload-images.jianshu.io/upload_images/1709375-ac4f4218a2edf4ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**参考文章：**
* https://blog.csdn.net/hust_twj/article/details/78855444
* https://blog.csdn.net/lb_383691051/article/details/79570711



