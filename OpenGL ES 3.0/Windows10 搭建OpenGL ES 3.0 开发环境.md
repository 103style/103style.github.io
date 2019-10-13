>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### OpenGL ES 3.0学习汇总
* [OpenGL ES 3.0 学习记录汇总](https://www.jianshu.com/p/216c7e91960a)


####  准备工作
* 下载 [Visual Studio 2017](https://visualstudio.microsoft.com/zh-hans/downloads/)，`安装教程参考文末的参考资料1,注意下安装目录`, 笔者下载的是企业版，注册码：**NJVYC-BMHX2-G77MM-4XJMR-6Q8QF** ，`参考资料3提供`
* 下载[《opengles3.0编程指南》源码](https://github.com/danginsburg/opengles3-book)
* 下载 [CMake](https://cmake.org/download/) 用来编译源码工程
* 下载 [mail opengl es模拟器](https://developer.arm.com/tools-and-software/graphics-and-gaming/graphics-development-tools/opengl-es-emulator/downloads)，用来运行opengl程序, **建议下载32位zip包**
![mail opengl es模拟器](https://upload-images.jianshu.io/upload_images/1709375-7116be3d60f04d3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 环境配置
* 解压 **mail opengl es模拟器** 压缩包 
* 新开一个窗口找到 **Visual Studio** 的安装目录下的路径 
 `D:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\VC\Tools\MSVC\14.16.27023` ，
  默认安装路径  `C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\VC\Tools\MSVC\14.16.27023`
* **VisualStudioIDE2019Pro** 版本路径： `VisualStudio\VisualStudioIDE2019Pro\VC\Tools\MSVC\14.22.27905`
* 复制 **mail opengl es模拟器 压缩包** 下的下列文件：
  *  把`include` 里的所有文件 `copy` 到  **Visual Studio** 的 `include` 里面。
  * 把 `libEGL.lib`，`libGLESv2.lib`，`libMaliEmulator.lib` 三个文件copy到   **Visual Studio** 的 `lib/x86` 里面。
  * 把 `libEGL.dll`，`libGLESv2.dll`，`libMaliEmulator.dll` 三个文件copy到   **Visual Studio** 的 `bin/Hostx86/x86` 里面。
![示例](https://upload-images.jianshu.io/upload_images/1709375-87432c585c9c4442.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 编译opengles3.0编程指南源码工程
* 打开安装号的 **cMake的gui**
 * `source code 路径` 为 `《opengles3.0编程指南》源码` 的位置
 *  `build thr binaries` 路径 为 任意的一个目录，例如 在源码目录下新建一个bin目录 `D:/projects/opengles/opengles3-book/bin`
 * 然后点击左下的 **Configure** ,选择的 **Visual Studio 15 2017** 的版本
 * 然后就会出现下图中中间的红色区域。
   修改  **EGL_LIBRARY** 为 上一步 环境配置中 `libEGL.lib` 的全路径，
例如`D:/Program Files (x86)/Microsoft Visual Studio/2017/Enterprise/VC/Tools/MSVC/14.16.27023/lib/x86/libEGL.lib`
   修改 **OPENGLES_LIBRARY** 为  `libGLESv2.lib` 的全路径，
例如`D:/Program Files (x86)/Microsoft Visual Studio/2017/Enterprise/VC/Tools/MSVC/14.16.27023/lib/x86/libGLESv2.lib`
* 然后点击 **Generate**，出现下面这样的提示就是编译成功了
    ```
      Selecting Windows SDK version 10.0.17763.0 to target Windows 10.0.17134.
      Configuring done
      Generating done
    ```
* 然后点击 **Open Project**，
![cMake的gui](https://upload-images.jianshu.io/upload_images/1709375-34528896d33d9b53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Configure](https://upload-images.jianshu.io/upload_images/1709375-3b018404cc5bfd29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 运行代码
* **Open Project 之后**
![工程目录i](https://upload-images.jianshu.io/upload_images/1709375-5bbbc1e3393fe797.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* **选择右侧的一个 例子，设置为启动项目**，例如 `Hello_Triangle`
  ![Hello_Triangle](https://upload-images.jianshu.io/upload_images/1709375-64c5e4c991867bd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 然后点击 **本地windows调试器**
![image.png](https://upload-images.jianshu.io/upload_images/1709375-0f450752dbf97815.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 运行效果
![Hello_Triangle 运行效果](https://upload-images.jianshu.io/upload_images/1709375-c2cd404ccb003c87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

OK，到此就结束了，有问题请留言。

#### 参考资料
1. [Visual Studio 2017安装教程](https://jingyan.baidu.com/article/a948d6512f00d70a2dcd2edc.html)
2. [windows配置opengles3开发环境](https://blog.csdn.net/mmy545237835/article/details/80762150)
3. [Windows7下搭建OpenGL ES3.0开发环境(VS2017)](https://blog.csdn.net/u010312436/article/details/82984322)

[原文链接：https://www.jianshu.com/p/b5f8627d9cbc](https://www.jianshu.com/p/b5f8627d9cbc)
