>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

本文是对 [Error:Execution failed for task ':app:transformClassesWithDexForDebug'解决记录](https://blog.csdn.net/lxk_1993/article/details/50511172) 的重新排版


### 3个错误：
* **non-zero exit value 1**
* **non-zero exit value 2**
* **non-zero exit value 3**



##### Error:Execution failed for task ':app:transformClassesWithDexForDebug'.

> `com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished with non-zero exit value 1`

这个是因为依赖包重复了 （像`v4`和`nineoldandroids`），如下图。
app中实现了对easeUI的依赖，但是`app`和`easeUI`都添加了对这个包的依赖。
所以就报这个错误，修改之后再报，就`clean`，`rebuild`一下
![v4包冲突](https://upload-images.jianshu.io/upload_images/1709375-8ba23e83452689fc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![v4包冲突](https://upload-images.jianshu.io/upload_images/1709375-5c210db43e96f40a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

##### Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
> `com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished withnon-zero exit value 2`



这个错误在`app`的`build.gradle`里面添加下面这句就好了。


```
android { 
	defaultConfig { 
		... 
		multiDexEnabled true  
	} 
}
```

---

##### Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
> `com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished withnon-zero exit value 3`



这个错误就在`app.bulid`里面加上这句，再`rebuild` ,之后再运行就行了。
`4g`可以看电脑配置修改（`2g`,`3g`,`6g`,`8g`）。

```
dexOptions {
    javaMaxHeapSize "4g"
}
```
如图：

![dexOptions ](https://upload-images.jianshu.io/upload_images/1709375-7e66d0cd2db71f75?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

以上





