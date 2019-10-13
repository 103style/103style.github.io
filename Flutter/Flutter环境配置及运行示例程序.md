>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

[官方配置介绍](https://flutter-io.cn/docs/get-started/install)

---


以下为`Windows`示例：

* 下载`Flutter sdk`，建议在盘符根目录下下载(`e.g.  D:\`)。
```
git clone https://github.com/flutter/flutter.git
```
![示例](https://upload-images.jianshu.io/upload_images/1709375-7e87b2a4361d7f06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

* 更新 `path` 环境变量
  * 在开始菜单的搜索功能键入`env`，然后选择 `编辑账户的环境变量`
  * 在 `用户变量` 一栏中，检查是否有 `Path` 这个条目：
    * 有的话，则把`flutter\bin`的完整目录添加到`Path`变量中(`e.g. D:\flutter\bin`).
    * 如果不存在的话，在用户环境变量中 **点击新建** 创建一个新的 `Path` 变量，然后将 `flutter\bin` 所在的完整路径作为新变量的值。
 
---

* 运行 `flutter doctor`  
   按 `win + R`,输入`cmd`, 然后输入`flutter doctor` 回车检查`flutter`环境。然后会提示你那些配置没配好。笔者提示的是 **没有找到 Android sdk 的目录**，需要在环境变量里 添加一个 `ANDROID_HOME`，值为你 `Android sdk的目录`,  例如`D:/example/sdk`。
---

* 安装开发工具，以 `Android Studio` 为例
  * [下载Android Studio](https://developer.android.google.cn/studio)
  * 然后双击运行下载的`exe`文件。
  * 然后安装 `Flutter` 和 `Dart` 插件
  ![安装插件](https://upload-images.jianshu.io/upload_images/1709375-25904db1844ae3a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 输入`dart`，然后勾选上，在输入`flutter`，然后勾选上，点击`ok`
  ![image.png](https://upload-images.jianshu.io/upload_images/1709375-8431ce9ea80c1b2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
---

* 运行示例程序
  * 重新启动 `Android Studio`
  * 点击`Open an existing Android Studio project`
  * 选中之前下载`flutter sdk`目录下中的`example` 里的 `flutter_gallery`，路径(`e.g. D:\flutter\examples\flutter_gallery`)
  * 然后同时按住 `ctrl` 、`alt` 、`s`三个键，输入 `flutter`，然后修改`Flutter SDK path`为刚刚下载的地址(`e.g.  D:\flutter`)。
  ![image.png](https://upload-images.jianshu.io/upload_images/1709375-4348ebeb4ea5d607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 等待编译通过，运行程序。快捷键`Shirt + F10`
    ![运行](https://upload-images.jianshu.io/upload_images/1709375-db84ca249ad32564.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





  
