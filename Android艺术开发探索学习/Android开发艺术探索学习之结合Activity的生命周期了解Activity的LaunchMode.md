# Android开发艺术探索学习之结合Activity的生命周期了解Activity的LaunchMode 

首先还是先介绍下Activity的launchMode.一共有四种.
1.**standard.**
2.**singleTop**.
3.**singleTask**.
4.**singleInstance**.

  第一种standard.就是不管怎么样每次启动都会创建一个新的实例，也就是系统默认的启动方式。
我们设置ActivityA的启动方式为standard.设置点击执行
    
    startActivity(**new**Intent(ActivityA.**this**, ActivityA.**class**));
点击两次，我们看到打印的logcat信息如下：

    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----:  onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----:   onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStop()
在这里ActivityA调用了三次oncreate()方法。所以一共有3个不同的ActivityA实例。

  第二种是singleTop,栈顶复用模式。就是 如果这个ActivityA已经位于栈的顶部，那么从ActivityA启动ActivityA，就不会重新创建ActivityA。将ActivityA的启动模式改为singTop，像上面那样测试，打印的logcat信息如下。

    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----:     onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
所以可以看出只创建了一个Activity.
但是如果ActivityA不在栈顶，我们增加一个启动模式为standard的ActivityB。 测试 A 启动 B, B在启动A.logcat信息如下。 

    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----:   onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----:   onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStop()
显然从B到A的时候又创建了一个ActivityA. 所以带singleTop这种启动模式的Activity，只有这个Activity在栈顶的时候，在启动这个Activity才不会重新创建新的Activity.否则就和standard没什么区别。而且日常开发中我们很少会有 Activity自己在启动自己这样的情况。

  第三种:singleTask.栈内复用模式。假设A的启动模式是singleTask.那么在一个栈中只会存在一个A的实例。并且当A不在栈顶的时候，再启动A的话，会直接销毁 栈中位于 A 上面的所有Activity实例。我们再新增launchMode为standard的B和C。
然后启动A,从A启动B,从B启动C，在从C启动A.打印的logcat信息如下.

    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onDestroy()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onRestart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onDestroy()
从logcat信息我们可以看到，当从C启动A的时候，在重新启动A之前，也就是C的onPause()之前会依次销毁栈内在A和C之间的Activity实例，然后当启动完A之后再销毁C。假设A是singTask模式，BCDE都是标准模式。然后一次启动ABCDE，然后在启动A.在E的onPause方法之前会依次条用B、C、D的onStop和onDestroy方法.然后当启动完A之后再调用E的 onstop和 ondestroy销毁E.

    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----:   onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onCreate()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----B----: onDestroy()
    com.hnpolice.xiaoke.activitylaunchmode E/----C----: onDestroy()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onPause()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onRestart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onStart()
    com.hnpolice.xiaoke.activitylaunchmode E/----A----: onResume()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onStop()
    com.hnpolice.xiaoke.activitylaunchmode E/----D----: onDestroy()
这种模式在开发中做程序退出的时候会用到。一种俗称“懒人式”的程序退出方法。就是把首界面的启动模式设置为singleTask.然后监听back键。第一次提示“再按一次退出程序”，第二次就直接finish掉首界面。程序退出就完成了。

  第四种启动模式：singleInstance。这种模式就是singleTask的加强模式。除了singleTask的所有特性之外。还规定了这种模式的Activity只能单独的位于一个任务栈中。

  大家看完要是不明白可以看看这个，
这里有篇文章[http://blog.csdn.net/liuhe688/article/details/6754323](http://blog.csdn.net/liuhe688/article/details/6754323)，
比较详细的介绍了Activity的launchMode。

博客地址：[http://blog.csdn.net/lxk_1993](http://blog.csdn.net/lxk_1993)
如果你喜欢我的博客，请关注我。欢迎留言拍砖。
 友情链接：
 如果你还不是很了解Acitivity的生命周期的话可以点这里：
 Activity的生命周期详解
 http://blog.csdn.net/lxk_1993/article/details/50731594
