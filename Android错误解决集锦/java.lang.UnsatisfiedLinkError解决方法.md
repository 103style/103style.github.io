# java.lang.UnsatisfiedLinkError解决方法 

就像这样的错误

`Java.lang.UnsatisfiedLinkError: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.pckgname.live-2/base.apk"],nativeLibraryDirectories=[/data/app/com.pckgname.live-2/lib/arm64, /vendor/lib64, /system/lib64]]] couldn't find "libvinit.so" `

`java.lang.UnsatisfiedLinkError:dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.belongsoft.smartvillage-2/base.apk"],nativeLibraryDirectories=[/data/app/com.belongsoft.smartvillage-2b/arm, endorb, /systemb]]] couldn't find "libffmpeg.so" `

错误的解决方法：
* 对于`Android Studio`
如图
在工程目录下的`gradle.properties`里面加上
`android.useDeprecatedNdk=true`
在`app`的`build.gradle`中添加如下代码,然后`rebuild`.

例子：
```
android {
    defaultConfig {
        multiDexEnabled true
        ndk {
            abiFilters "armeabi", "armeabi-v7a", "x86", "mips"
        }
    }
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
}
```

* **对于Eclipse**
在`lib`目录下建下面这些文件夹  然后把所有报错的`so`文件都放一份到每个文件夹下。
  * `armeabi`
  * `armeabi-v7a`
  * `x86`
  * `mips`
