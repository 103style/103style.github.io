# AndroidStudio依赖的包文件导入失败 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


解决方法：
关闭AS，把 **`C:\Users\计算机用户名\.gradle\`** 下的 **`caches`** 目录全删了，然后重新启动项目就好了。

----


最近遇到一个莫名其妙的问题：
之前AS打开项目还运行的好好的，
然后第二天一打开，就一直编译失败，
发现是 依赖的第三方库的文件找不到，类似以下语句报红：
```
import com.github.greendao.module.CacheDbHelper;
```

之前遇到过类似的错误，也是报红，但是能正常跑起来，只要点击下图的对应操作，清空缓存就好。
![清空缓存](https://upload-images.jianshu.io/upload_images/1709375-a588a68ae939bf06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**但是这次死活没有效果，而且还运行不起来。**

* 然后尝试重启计算机，也没用...

* 然后我又新建了一个项目，导入这个第三方引用，然而发现并没有什么问题，所以并不是依赖的问题。

* 接着又下载了`Android Studio 3.5 beta4` 的版本，导入项目发现还是有问题。



最后没有办法只有关掉AS，然后把 **`C:\Users\计算机用户名\.gradle\`** 下的 **`caches`** 目录全删了。
![caches目录](https://upload-images.jianshu.io/upload_images/1709375-036eb1e85c3cb92e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
然后重新运行 就ok了。

以上
