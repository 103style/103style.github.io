欢迎评论吐槽拍砖
![](http://upload-images.jianshu.io/upload_images/1709375-064743372400921e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
首先看这些方法这什么时候调用。
官方文档是这样描述的：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1709375-0490feb715839280.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1709375-6c5a9142a466d6d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1709375-737e890c4add8e9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大致的中文描述是
当第一次调用一个Activity就会执行onCreate方法.后面总是接着执行onStart方法.
当Activity处于可见状态的时候就会调用onStart方法.接着如果调用onResume我们就会看到这个界面，调用onStop方法的话就会被隐藏。
当Activity可以得到用户焦点的时候就会调用onResume方法.后面总是调用onPause方法.
当Activity没有被销毁的时候重新调用这个Activity就会调用onRestart方法.后面总是接着执行onStart方法.
当系统将要开始加载另外一个Activity的时候调用onPause方法.这个方法通常用于提交固定的数据、停止动画和其他可能消耗CPU的事情。onPause必须运行的很快，因为将要加载的Activity在它返回的时候才会“恢复”。如果重新返回这个Activity执行onResume方法，如果这个Activity不可见的时候就调用onStop方法.
当Activity处于不可见状态的时候就会调用onStop方法.在Activity将要被销毁的时候 或者 一个新的Activity出现在界面的时候调用. 接下来条用onRestart或者onDestory方法..
当Activity被销毁时会调用onDestory方法.

有两个测试activity。1代表 MainActivity，2代表TestActivity
测试机：genymotion模拟器Google Nexus 4 - 4.4.4 API 19 768*1280  (320dpi)
1.首先是MainActivity正常启动，logcat信息如下：

    xiaoke.hnpolice.com.activitylifecircle E/----1----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStart()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onResume()

2.点击返回，logcat信息如下：

    xiaoke.hnpolice.com.activitylifecircle E/----1----: onPause()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStop()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onDestroy()

3.启动MainActivity,然后点击跳转到TestActivity,logcat信息如下：

    xiaoke.hnpolice.com.activitylifecircle E/----1----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStart()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onResume()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onPause()
    xiaoke.hnpolice.com.activitylifecircle E/----2----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/----2----: onStart()
    xiaoke.hnpolice.com.activitylifecircle E/----2----: onResume()
    xiaoke.hnpolice.com.activitylifecircleE/----1----: onStop()
我们可以看到TestActivity的创建 是在MainActivity的onPause()之后,onStop之前.所以当有面试官问的时候，你就知道怎么回答了吧。

4.启动MainActivity,然后旋转屏幕，,logcat信息如下：

    xiaoke.hnpolice.com.activitylifecircle E/----1----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStart()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onResume()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onPause()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStop()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onDestroy()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onStart()
    xiaoke.hnpolice.com.activitylifecircle E/----1----: onResume()
我们可以看到 MainActivity在进行旋转屏幕操作的时候，先销毁了自己，然后再重新创建。所以你懂的，面试官问的时候你就知道该怎么回答了。

5.我们在onCreate()中出现加上下面这句之后，logcat信息如下。

    List<String> s = null;
    int size = s.size();

    xiaoke.hnpolice.com.activitylifecircle E/----1----: onCreate()
    xiaoke.hnpolice.com.activitylifecircle E/AndroidRuntime: FATAL EXCEPTION: main Process: xiaoke.hnpolice.com.activitylifecircle,PID: 3835 java.lang.RuntimeException: Unable to start activity ComponentInfo {xiaoke.hnpolice.com.activitylifecircle/xiaoke.hnpolice.com.activitylifecircle.MainActivity}: java.lang.NullPointerException ...Caused by: java.lang.NullPointerException at xiaoke.hnpolice.com.activitylifecircle.MainActivity.onCreate(MainActivity.java:30)...

当在onCreate()中出现异常时.MainActivity只会调用onCreate()方法。

博客地址：[http://blog.csdn.net/lxk_1993](http://blog.csdn.net/lxk_1993)
如果你喜欢我的博客，请关注我。欢迎留言拍砖。

测试项目地址：[ https://github.com/103style/ActivityLifeCircle](https://github.com/103style/ActivityLifeCircle)
友情链接 ： [如何托管你的项目到github上详细教程](http://blog.csdn.net/lxk_1993/article/details/50441442)
