# AndroidStudio用久了，磁盘越来越不够用？ 

>转载请以链接形式标明出处： [https://www.jianshu.com/p/17635f604495](https://www.jianshu.com/p/17635f604495)
本文出自:[**103style的博客**](https://www.jianshu.com/u/109656c2d96f) 

AndroidStudio用久了，磁盘越来越不够用？

**你应该需要这个**

以 **Windows 10**为例

>**先提醒下，如果文件目录删不掉，应该是文件目录的字符长度太长，所以把那些长的重命名一下再删就好。**
**批量重命名，先 `ctrl+A`全选，鼠标右键选重命名就好**
![文件目录太长](https://upload-images.jianshu.io/upload_images/1709375-780eb0f6c6378456.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



 > 首先来到用户目录`C:\Users\Administrator   (Administrator为用户名)`下 
![用户目录](https://upload-images.jianshu.io/upload_images/1709375-783132dce2107661.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中的
* **.gradle**  是gradle编译相关的文件
* **.lldb**  工程配置了ndk路径时，debug调试生成的文件夹，`如果不需要调试so文件，直接把sdk里面的ndk卸掉吧`
* **.AndroidStudio3.0**  是AndroidStudio版本的文件夹，**如果你用的3.0.1的版本，还有类似2.2.3的目录就可以删掉了**

> 先看**.lldb**目录
直接点击去到`C:\Users\Administrator\.lldb\module_cache\remote-android\.cache`这个目录，
你会看到类似的下图的目录，这个是工程配置了ndk路径（`项目目录下的local.properties文件`）每次debug时都会生成的一个文件夹，
**直接按修改时间排序，删掉当前时间之前的文件夹就好** 
![.lldb](https://upload-images.jianshu.io/upload_images/1709375-41e31043c060fb1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
或者直接注释掉**路径（`项目目录下的local.properties文件`）**里面的类似 `ndk.dir=D\:\\Android\\sdk\\ndk-bundle`这句，然后把整个.lldb目录都删掉
或者直接卸掉ndk,debug秒附加
![卸掉ndk](https://upload-images.jianshu.io/upload_images/1709375-1ca34ec3df06f99b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



> 然后就是**.gradle**相关的，**caches**、**wrapper** 、**daemon**
这三个目录下都缓存了AndroidStudio之前编译的所有工程`gradle/wrapper/gradle-wrapper.properties`文件里面配置的gradle版本，所以我们可以**删掉 比较旧的一些版本以及不需要用的版本**
![gradle/wrapper/gradle-wrapper.properties](https://upload-images.jianshu.io/upload_images/1709375-4e82750d1ea2183f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>**caches目录**
>`C:\Users\Administrator\.gradle\caches`
> 删掉不需要的版本（3.1、3.3.... 之类的）
![caches](https://upload-images.jianshu.io/upload_images/1709375-381c1aa07be886d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>**wrapper目录：** gradle版本的源文件相关，同样删掉不需要的版本
>`C:\Users\Administrator\.gradle\wrapper\dists`
![gradle版本](https://upload-images.jianshu.io/upload_images/1709375-c092f22065a028b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>**daemon目录：** 
>`C:\Users\Administrator\.gradle\daemon`
>gradle编译日志的目录，同样**删掉不需要的版本**，以及**需要版本中的之前的日志文件**
![](https://upload-images.jianshu.io/upload_images/1709375-eba3517555d130ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![gradle编译的日志文件](https://upload-images.jianshu.io/upload_images/1709375-f3bc9b14eed61bc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 补充：
* **配置了ndk,打包的时候ndk会去掉so文件的debug调试信息，所以你会可能会遇到配置了ndk打的包比没配ndk打的包体积小**


### 勘误
>2018-04-09 修正 `.lldb` 文件夹描述错误问题，新增`.lldb`处理方式
>2018-04-13 添加补充 **配了ndk打的包比没配ndk打的包体积小** 的说明





