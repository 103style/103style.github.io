# Could not find class 'android.support.v4.widget.DrawerLayout$1' 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


##### 手机信息
* phone: **huawei mate 7**
* Android version : **4.4.2**
* 系统：**EMUI 3.0**

报错信息：
```
Could not find class 'android.support.v4.widget.DrawerLayout$1', referenced from method android.support.v4.widget.DrawerLayout.<i
Could not find class 'android.view.WindowInsets', referenced from method android.support.v4.widget.DrawerLayout.onDraw
Could not find class 'android.view.WindowInsets', referenced from method android.support.v4.widget.DrawerLayout.onMeasure
Could not find class 'android.view.WindowInsets', referenced from method android.support.v4.widget.DrawerLayout.onMeasure
```

---

网上找了一圈找到下面这个方法，然后对我并没用：
[link](https://stackoverflow.com/questions/46389995/could-not-find-class-android-support-v4-widget-drawerlayout1-referenced-from)
```
Hi ! I have used attr for color in drawable shape.... 
its not support below lollipop. 
I have removed attr and its working fine now. Thanks 
```

----

最后发现 在 `AndroidManifest` 加上已下权限就可以了....
```
//Checks for usages of deprecated classes and methods in XML.
<uses-permission android:name="android.permission.GET_TASKS" />
```
以上
