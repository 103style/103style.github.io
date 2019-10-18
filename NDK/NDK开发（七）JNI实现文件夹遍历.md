# NDK开发（七）JNI实现文件夹遍历 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

###  编写测试代码
* 创建类 [JniListDirAllFiles](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/java/com/lxk/ndkdemo/JniListDirAllFiles.java)，编写对应的测试代码：
    ```
    public class JniListDirAllFiles {
        static {
            System.loadLibrary("list_dir_all_file");
        }
        /**
         * 输出文件夹下得所有文件
         *
         * @param dirPath 文件夹路径
         */
        public native void listDirAllFile(String dirPath);

        public void test() {
            listDirAllFile(Config.BASE_URL);
        }
    }
    ```
* 创建`list_dir_all_file.cpp`.
* 添加以下代码到`CMakeLists.txt`：
    ```
    add_library(
            list_dir_all_file
            SHARED
            list_dir_all_file.cpp)
    target_link_libraries(
            list_dir_all_file
            ${log-lib})
    ```
---

### 实现JNI文件夹遍历逻辑
```
#include <jni.h>
#include <dirent.h>
#include <string>
#include <android/log.h>
#include "LogUtils.h"

const int PATH_MAX_LENGTH = 256;

extern "C"
JNIEXPORT void JNICALL
Java_com_lxk_ndkdemo_JniListDirAllFiles_listDirAllFile(JNIEnv *env, jobject instance,
                                                       jstring dirPath_) {
    //空判断
    if (dirPath_ == nullptr) {
        LOGE("dirPath is null!");
        return;
    }
    const char *dirPath = env->GetStringUTFChars(dirPath_, nullptr);
    //长度判读
    if (strlen(dirPath) == 0) {
        LOGE("dirPath length is 0!");
        return;
    }
    //打开文件夹读取流
    DIR *dir = opendir(dirPath);
    if (nullptr == dir) {
        LOGE("can not open dir,  check path or permission!")
        return;
    }

    struct dirent *file;
    while ((file = readdir(dir)) != nullptr) {
        //判断是不是 . 或者 .. 文件夹
        if (strcmp(file->d_name, ".") == 0 || strcmp(file->d_name, "..") == 0) {
            LOGV("ignore . and ..");
            continue;
        }

        if (file->d_type == DT_DIR) {
            //是文件夹则遍历
            //构建文件夹路径
            char *path = new char[PATH_MAX_LENGTH];
            memset(path, 0, PATH_MAX_LENGTH);
            strcpy(path, dirPath);
            strcat(path, "/");
            strcat(path, file->d_name);
            jstring tDir = env->NewStringUTF(path);
            //读取指定文件夹
            Java_com_lxk_ndkdemo_JniListDirAllFiles_listDirAllFile(env, instance, tDir);
            //释放文件夹路径内存
            free(path);
        } else {
            //打印文件名
            LOGD("%s/%s", dirPath, file->d_name);
        }
    }

    //关闭读取流
    closedir(dir);

    env->ReleaseStringUTFChars(dirPath_, dirPath);
}
```
---

### 执行测试代码
```
private void testListDirAllFiles() {
    if (hasFilePermission()) {
        new JniListDirAllFiles().test();
        Toast.makeText(this, "任务完成，检查看日志输出", Toast.LENGTH_SHORT).show();
    }
}
```
会在控制台打印`/storage/emulated/0/NDKDemo/`目录下的所有文件。

---

[Demo地址：https://github.com/103style/NDKDoc/tree/master/NDKDemo](https://github.com/103style/NDKDoc/tree/master/NDKDemo)

以上
