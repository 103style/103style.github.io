# AndroidStudio自定义代码模块来实现代码规范 

>转载请以链接形式标明出处： [https://www.jianshu.com/p/c5ad9d7f8d98](https://www.jianshu.com/p/c5ad9d7f8d98)
本文出自:[**103style的博客**](https://www.jianshu.com/u/109656c2d96f) 


**Android Studio 自定义代码模块 让我们更快更轻松的写代码**

* 说到添加作者信息，我想大家都知道下图这样的添加方式

![作者信息](http://upload-images.jianshu.io/upload_images/1709375-2edfdce975c67d4d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但是这样有一个问题 我们在新建Activity的时候 并不会起作用，真的很烦

* 写布局的时候 一些重复的Textview、Imageview、Linearlayout等View or ViewGorup之类的（当然可以用同统一的style去管理），心很累


**先看看效果图**

![作者信息.gif](http://upload-images.jianshu.io/upload_images/1709375-70891d824e95c15d?imageMogr2/auto-orient/strip)

![xml快速添加控件.gif](http://upload-images.jianshu.io/upload_images/1709375-75e3d9a18cc91e0b.gif?imageMogr2/auto-orient/strip)
 **里面的具体代码 可以自己设置 或者用统一的style管理**


##### 所以就有了自定义代码模板来实现
**步骤如下**
* 打开Android Studio 来到一个项目界面
* 按Ctrl+Alt+ s ，打开设置界面的快捷键
* 在输入框中输入Live 

如下图 

![操作步骤.gif](http://upload-images.jianshu.io/upload_images/1709375-e39f23b37da83e0a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
##### 为了方面管理，我们可以先新建一个group，如下图：
![添加group](http://upload-images.jianshu.io/upload_images/1709375-76b55f1015cd1de0?imageMogr2/auto-orient/strip)

##### 然后再里面写我们自定义模板
![设置模板内容](http://upload-images.jianshu.io/upload_images/1709375-ce44def716835506?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* Abbreviation 后面填 你想设置的快捷键，如 auth_java 
* Template text 里面填 你想设置的代码内容 
如:
/**
 \* create by Fungo_Xiaoke $DATE$ $TIME$
 \* email:luoxiaoke@yuntutv.net
 */

**$$中间包起来的是变量名， 我们我可点击Edit variables来设置它的值**

![设置变量](http://upload-images.jianshu.io/upload_images/1709375-7ce4514e58977bcc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
**Define 选择设置的这个快捷键（auth_java）生效的范围，这里我选的Java**

 
**下面是两个实例**

![例子1.gif](http://upload-images.jianshu.io/upload_images/1709375-34433fa3617ff4a1?imageMogr2/auto-orient/strip)

![例子2.gif](http://upload-images.jianshu.io/upload_images/1709375-5193954950b9ab96?imageMogr2/auto-orient/strip)

**然后就能方便的使用了。 为变量设不同的值，大家可以试试里面对应的表达式是什么效果。**

**另外其他的可以根据实际情况去设计和发挥。**
* 单例模式，  
* 添加改代码的备注信息   
* 注释信息  
* 自定义View的一些常用逻辑 **...**

#### e.g:
![自定义view.gif](http://upload-images.jianshu.io/upload_images/1709375-f4a8df7a3ee86f29.gif?imageMogr2/auto-orient/strip)


![单例.gif](http://upload-images.jianshu.io/upload_images/1709375-3c7144a25ffc31d4.gif?imageMogr2/auto-orient/strip)

##### 是不是快很多!


转载请以链接形式标明出处：
[http://www.jianshu.com/p/c5ad9d7f8d98](http://www.jianshu.com/p/c5ad9d7f8d98)
本文出自:[103style](http://www.jianshu.com/u/109656c2d96f)

如果觉得下得不错，点个关注呗。
有问题也可以下面留言。
