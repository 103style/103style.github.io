# AndroidStudio：Unable to start the daemon process 

>转载请以链接形式标明出处： [https://www.jianshu.com/p/64fff6e31965](https://www.jianshu.com/p/64fff6e31965)
本文出自:[**103style的博客**](https://www.jianshu.com/u/109656c2d96f) 

最近不知道怎么的笔记本打开项目一直提示下面这个 **丧心病狂** 的错误。
`AndroidStudio（3.0.1） jdk(1.8.0)`

>Unable to start the daemon process. 
This problem might be caused by incorrect configuration of the daemon. 
For example, an unrecognized jvm option is used. 
Please refer to the user guide chapter on the daemon at 
[http://gradle.org/docs/3.5/userguide/gradle_daemon.html](http://gradle.org/docs/3.5/userguide/gradle_daemon.html) 
Please read below process output to find out more:


于是乎 开始 baidu、google、csdn、stackoverflow... 
找到下面几种解决方法，然而对我来说 **都没用！** **都没用！** **都没用！** 
**你们可以试试这些，抢救一下，有用的话评论下**

 
* >1、修改项目中gradle.properties文件,只要添加以下一行代码:
```org.gradle.jvmargs=-Xmx512m```
>2、重启Android Studio

* >修改项目中gradle.properties文件  (`然而 MaxPermSize AndroidStudio 3.0.1已经废弃了`)
```org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8```

* > `Try deleting your .gradle from C:\Users\<username> directory and try again.` 
删除![C:\Users\<username>\.gradle](https://upload-images.jianshu.io/upload_images/1709375-2ccacfa35c626696.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* > `File Menu - > Invalidate Caches/ Restart->Invalidate and Restart.`
![Invalidate and Restart](https://upload-images.jianshu.io/upload_images/1709375-6c5735fefc3b26a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* > 在下面的位置添加` -Xmx512m`
 ![complier](https://upload-images.jianshu.io/upload_images/1709375-f12b8d79df0bd9be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* > 直接重装AndroidStudio

对的，以上方法 **都没用！**  **都没用！**  **都没用！** 

>有错误日志,然而看不懂
![日志](https://upload-images.jianshu.io/upload_images/1709375-f8e364cb7d068f46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



**最后默默重装系统**
**然后**
**就好了...**
**就好了...**
**就好了...**


![](https://upload-images.jianshu.io/upload_images/1709375-65ebf9498f3693c7.jpg?imageMogr2/auto-orient/strip)


![](https://upload-images.jianshu.io/upload_images/1709375-45bb1ef8c6524d01.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
