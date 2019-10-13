>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**


[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

### 目录
* **环境配置**
* **创建支持 C/C++ 的新项目**
* **向现有项目添加 C/C++ 代码**
* **参考文章**

---

### 环境配置
*  [下载安装 Android Studio](https://developer.android.google.cn/studio/index.html)
* 配置 **NDK** 环境
  * 启动 `Android Studio`.
  * 如下图：在界面的 **Configure** 中 打开 **Settings** 界面。
  ![进入Android Studio 设置界面](https://upload-images.jianshu.io/upload_images/1709375-0e723dd1c45266f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 如下图： 在 **左上角** 输入框输入**sdk** →  点击 **Android SDK** → 点击 **SDK Tools** →  然后勾选上 **LLDB**、**CMake**、**NDK** → 然后点击 **OK** → 点击弹出框中的 **OK**.
  ![安装NDK环境步骤图](https://upload-images.jianshu.io/upload_images/1709375-934674fca30e1ea7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * 下载安装完成之后，重启 **Android  Studio**.

---

###  创建支持 C/C++ 的新项目
* 在 **Android Studio** 的界面，点击 **Start a new Android Studio project**
* 如下图，在弹出界面选择 **Native C++** ，然后点击 **Next**。
  ![选择Native C++](https://upload-images.jianshu.io/upload_images/1709375-97b7feb22a7abc12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 在新的界面直接点击 **Next**, 或者修改`Name`, `Package name` , `Save location` 中的值；
* 在新的界面点击 **Finish**.

**Android Studio**将会为我们生成一个模板工程，我们可以直接运行，启动之后界面上会显示 **Hello from C++**。

---

###  支持 C/C++ 的项目文件介绍
从 `Android Studio` 左侧打开 `Project` 窗格并选择 `Android` 视图，如下图：
![支持C/C++ 的项目结构图](https://upload-images.jianshu.io/upload_images/1709375-544fca2d8171da56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们只要关心上图红框标记出来的以下这些文件就好：
* `MainActivity` ：应用视图界面，加载了一个名为`native-lib`的库，定义了一个`native`的方法`stringFromJNI`，然后将`stringFromJNI`返回的值设置到`TextView`上。
  ```
  public class MainActivity extends AppCompatActivity {
      static {
          System.loadLibrary("native-lib");
      }
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          ...
          tv.setText(stringFromJNI());
      }
      public native String stringFromJNI();
  }
  ```
* `CMakeLists.txt` ：CMake 构建脚本。
    ```
     # 设置CMake的最低版本
    cmake_minimum_required(VERSION 3.4.1)
    # 添加源文件或者库
    add_library(
            native-lib # 库的名字
            SHARED # 库的类型  
            native-lib.cpp) # 库的源文件
    # 引用NDK中的库log，命名为log-lib
    find_library(
            log-lib # 库路径对应的变量名
            log) # NDK中的库名
    # 关联库，确保 native-lib 中 能使用 log 库
    target_link_libraries(
            native-lib 
            ${log-lib})
    ```
* `native-lib.cpp` : 示例 C++ 源文件，位于`src/main/cpp/`目录。
    ```
    #include <jni.h>
    #include <string>
    
    extern "C" JNIEXPORT jstring JNICALL
    //MainActivity 中 stringFromJNI方法 对应的 c 方法名字
    Java_com_lxk_myapplication_MainActivity_stringFromJNI(
            JNIEnv *env,
            jobject ) {
        std::string hello = "Hello from C++";
        return env->NewStringUTF(hello.c_str());
    }
    ```
* `build.gradle` ：构建文件
    ```
    android {
        ...
        defaultConfig {
            ...
            externalNativeBuild {
                cmake {
                    cppFlags ""
                }
            }
        }
        externalNativeBuild {
            cmake {
                path "src/main/cpp/CMakeLists.txt" //构建脚本的路径
                version "3.10.2"//CMake的版本
            }
        }
    }
    ```
  

如果刚刚运行过项目的话，点击左侧`Project` 窗格并选择 `Project` 视图，会在 `app/build/intermediates/cmake/debug/armeabi-v7a/`下生成一个 `libnative-lib.so`文件。

**CMake** 使用 `lib库名称.so` 的规范来为库文件命名，库名称即为我们定义的 `native-lib`。不过我们在Java代码中加载时，还是使用我们定义的库名称 `native-lib`。
```
static {
    System.loadLibrary("native-lib");
}
```

---

### 向现有项目添加 C/C++ 代码
向现有 **Android Studio** 项目添加或导入原生代码，则需要按以下基本流程操作：
* **创建新的原生源文件**，并将其添加到 **Android Studio** 项目中，如果您已经拥有原生代码或想要导入预编译原生库，则可跳过此步骤。
* **创建 CMake 编译脚本**，告知 `CMake` 如何将原生源文件编译入库。如果导入和关联预编译库或平台库，您也需要此编译脚本。如果现有的原生库已有 `CMakeLists.txt` 编译脚本，或使用 `ndk-build` 并包含 `Android.mk`编译脚本，则可跳过此步骤。
* **提供一个指向 CMake 或 ndk-build 脚本文件的路径**，将 `Gradle` 关联到原生库。`Gradle` 使用编译脚本将源代码导入您的 **Android Studio** 项目并将原生库（`.so`文件）打包到 `APK` 中。


##### 重新创建一个 Basic Activity的工程。
![Basic Activity](https://upload-images.jianshu.io/upload_images/1709375-42633cf0195775b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 创建新的原生源文件
* 从左侧打开 **Project** 菜单并从下拉菜单中选择 **Project** 视图。
* 右键点击选中 **app/src/** 下的 **main** 目录，然后选择 **New > Directory**，为目录输入一个名称（例如 `cpp`）并点击 **OK**。
* 右键点击您刚刚创建的目录，然后选择 **New > C/C++ Source File**，输入一个名称，例如 **hello-ndk**，如果想创建一个标头文件，请勾选 **Create an associated header** 复选框，点击 **OK**。
  ![](https://upload-images.jianshu.io/upload_images/1709375-b57d99d2dc516f7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 创建 CMake 构建脚本
如果您的原生源文件还没有 `CMake` 构建脚本，则您需要自行创建一个并包含适当的 `CMake` 命令。

**必须将其命名为 CMakeLists.txt**。


* 右键点击选中 **app** ，然后选择 **New > File**，输入**CMakeLists.txt** 作为文件名并点击  **OK**。
* 添加命令到 **CMakeLists.txt** 中
    ```
    cmake_minimum_required(VERSION 3.4.1)
    
    add_library( hello-ndk
                 SHARED
                 src/main/cpp/hello-ndk.cpp)
    ```
* 使用 `add_library()` 向您的 `CMake` 构建脚本添加源文件或库时，**Android Studio** 还会在您同步项目后在 **Project** 视图下显示关联的标头文件。不过，为了确保 `CMake` 可以在编译时定位您的标头文件，您需要将 `include_directories()`命令添加到 `CMake` 构建脚本中并指定标头的路径：
    ```
    add_library(...)
    
    include_directories(src/main/cpp/include/)
    ```
* 添加 **NDK API**，**Android NDK** 提供了一套实用的原生 `API` 和库。将 `find_library()` 命令添加到您的 `CMake` 构建脚本中以定位 **NDK** 库。以 **Android 特定的日志支持库** 为例，为了确保您的原生库可以在 `log` 库中调用函数，您需要使用 `CMake` 构建脚本中的 `target_link_libraries()`命令关联库：
    ```
    add_library(...)

    find_library( log-lib # 库路径的变量名
                  log ) # 对应的库名

    #将预构建库关联到您自己的原生库              
    target_link_libraries( hello-ndk
                           ${log-lib} )
    ```


##### 将 Gradle 关联到您的原生库  
要将 `Gradle` 关联到您的原生库，您需要提供一个指向 `CMake` 或 `ndk-build` 脚本文件的路径。在您构建应用时，`Gradle` 会以依赖项的形式运行 `CMake` 或 `ndk-build`，并将共享的库打包到您的 `APK` 中。

* 点击**Android Studio** 左侧菜单 **Project** 并选择 **Android** 视图。 点击 弹出菜单的第二个选项 **Link C++ Project with Gradle**，如图1，点击文件夹，点击 **Android Studio**图标的按钮可以定位到项目根目录，然后如图2 配置 **CMakeLists.txt** 的路径，点击 **OK**。
![点击可以定位到项目根目录](https://upload-images.jianshu.io/upload_images/1709375-acca2c501ca8536e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/1709375-6ccbf45a0bf0e7fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 然后 **app**目录下的`build.gradle`文件会自动添加以下代码。
    ```
    externalNativeBuild {
        cmake {
            path file('CMakeLists.txt')
        }
    }
    ```

##### 配置Javah命令工具
如下图，按 **Ctrl + Alt + s** 进入 **Setting** 界面，点击 **Tools → External Tools → +** 配置添加外部工具。参数如下：
```
Name :JavaH
Program:$JDKPath$/bin/javah
Parameters: -encoding UTF-8 -d ../cpp -jni $FileClass$
Working directory: $SourcepathEntry$\..\java
```
![配置Javah命令工具](https://upload-images.jianshu.io/upload_images/1709375-dd722a3e223aca74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  

##### 编辑 MainActivity
在 `MainActivity` 添加如下代码：
```
public class MainActivity extends AppCompatActivity {
    static {
        System.loadLibrary("hello-ndk");
    }
    public native String helloNDK();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        //添加下面两行代码
        TextView show = findViewById(R.id.tv_show);
        show.setText(helloNDK());

        ....
    }
    ...
}
```

##### 修改 content_main.xml
修改 `content_main.xml`，给`TextView`添加 `android:id="@+id/tv_show"`
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    ...>

    <TextView
        android:id="@+id/tv_show"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        .../>
</android.support.constraint.ConstraintLayout>
```

#####  生成 .h 头文件
如下图，右键点击`MainActivity`，选择弹出框中的 **External Tools** 的 **JavaH** 。就会在`cpp`目录下生成 `com_example_myapplication_MainActivity.h`文件，你的文件名可能不一样。
![ 生成 .h 头文件](https://upload-images.jianshu.io/upload_images/1709375-1244a53a23f42842.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 编辑hello-ndk.cpp
修改`hello-ndk.cpp`为以下代码：
```
#include <jni.h>
//确认此处名字是否可你生成的头文件的名字一样
#include "com_example_myapplication_MainActivity.h" 

//函数名要和头文件中的名字一致
JNIEXPORT jstring JNICALL Java_com_example_myapplication_MainActivity_helloNDK  
  (JNIEnv * env, jobject){
    return env ->NewStringUTF("Hello NDK");
}
```

##### 运行程序
点击顶部菜单栏 **Build**  中的 **Rebuild Project**，完了之后按 `Shirt + F10` 运行程序，然后界面上就会显示 **Hello NDK** 的字样。

---

### 参考文章
* [官方NDK 入门指南](https://developer.android.google.cn/ndk/guides)

---


[Demo地址](https://github.com/103style/NDKDoc/tree/master/HelloNDK)

以上
