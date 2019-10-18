# NDK开发（六）JNI实现文件拆分和合并 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

### 目录
* 编写测试代码
* 实现创建文件逻辑
* 实现JNI文件拆分逻辑
* 实现JNI文件合并逻辑
* 执行测试代码

---


### 编写测试代码
* 添加权限
    ```
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    ```
* 创建`JniFileOperation`类，编写对应的测试代码：
    ```
    public class JniFileOperation {
        private static final String TAG = "JniFileOperation";
        static {
            System.loadLibrary("file_operation");
        }
        /**
         * 原文件名
         */
        private String fileName = "split_test.txt";
        /**
         * 合并拆分之后文件的文件名
         */
        private String mergeFileName = "split_test_merged.txt";
        /**
         * 文件拆分格式
         */
        private String splitFileFormat = "split_test_%d.txt";
        /**
         * 拆分的数量
         */
        private int splitCount = 4;
        public native void createFile(String fileName);
        /**
         * 拆分
         *
         * @param path        文件路径
         * @param pathPattern 拆分之后文件的路径格式
         * @param splitCount  拆分成几个
         */
        public native void split(String path, String pathPattern, int splitCount);
        /**
         * 合并
         *
         * @param pathMerge   合并之后的文件路径
         * @param pathPattern 要合并的文件的路径格式
         * @param count       要合并的文件数量
         */
        public native void merge(String pathMerge, String pathPattern, int count);
    
        /**
         * 测试文件 拆分与合并
         */
        public void test() {
            String filePath = Config.getBaseUrl() + fileName;
            Log.e(TAG, "filePath = " + filePath);
    
            File file = new File(filePath);
            if (!file.exists()) {
                Log.e(TAG, "开始创建文件");
                createFile(filePath);
            }
    
            String pathPattern = Config.getBaseUrl() + splitFileFormat;
            split(filePath, pathPattern, splitCount);
            Log.e(TAG, "文件拆分成功");

            String mergePath = Config.getBaseUrl() + mergeFileName;
            merge(mergePath, pathPattern, splitCount);
            Log.e(TAG, "文件合并成功");
        }
    }
    ```
* 创建`file_operation.cpp`
* 添加以下代码到`CMakeLists.txt`：
    ```
    add_library(
            file_operation
            SHARED
            file_operation.cpp)
    target_link_libraries(
            file_operation
            ${log-lib})
    ```

---

### 实现创建文件逻辑
```
#include <jni.h>
#include <cstdio>
#include <cstdlib>
#include <android/log.h>

#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"com_lxk_ndkdemo",FORMAT,##__VA_ARGS__);

const int SIZE = 100;

/**
 * 创建拆分原文件
 */
extern "C"
JNIEXPORT void JNICALL
Java_com_lxk_ndkdemo_JniFileOperation_createFile(JNIEnv *env, jobject instance, jstring fileName_) {
    const char *fileName = env->GetStringUTFChars(fileName_, nullptr);

    //创建写文件流
    FILE *fp = fopen(fileName, "wb");

    //写文件
    for (int i = 0; i < SIZE; i++) {
        fputs("0123456789\n", fp);
    }
    //关闭流
    fclose(fp);
    LOGE("%s", "创建文件成功");
    env->ReleaseStringUTFChars(fileName_, fileName);
}
```


---

### 实现JNI文件拆分逻辑
```
/**
 * 根据文件的路径，获得文件的大小
 */
long get_file_size(const char *path) {
    //rb：只读打开一个二进制文件，允许读数据
    //使用给定的模式 "rb" 打开 path 所指向的文件
    FILE *fp = fopen(path, "rb");
    if (fp == nullptr) {
        LOGE("%s", "文件打开失败");
        return 0;
    }
    //SEEK_SET	文件的开头
    //SEEK_CUR	文件指针的当前位置
    //SEEK_END	文件的末尾
    //设置流 fp 的文件位置为 0， 0 意味着从给定的 SEEK_END 位置查找的字节数。
    fseek(fp, 0, SEEK_END);
    //返回给定流 fp 的当前文件位置。
    return ftell(fp);
}
/**
 * 拆分文件
 */
extern "C"
JNIEXPORT void JNICALL
Java_com_lxk_ndkdemo_JniFileOperation_split(JNIEnv *env, jobject instance, jstring path_,
                                            jstring pathPattern_, jint splitCount) {
    const char *path = env->GetStringUTFChars(path_, nullptr);
    const char *pathPattern = env->GetStringUTFChars(pathPattern_, nullptr);

    //malloc：分配所需的内存空间，并返回一个指向它的指针。
    char **patches = new char *[splitCount];

    //获取文件长度
    long fileSize = get_file_size(path);

    //获取单个文件长度
    long per_size = fileSize / splitCount;

    //设置每个子文件的路径
    for (int i = 0; i < splitCount; i++) {
        patches[i] = new char[256];
        sprintf(patches[i], pathPattern, i);
        LOGE("patches[%d] = %s", i, patches[i]);
    }

    //创建fp流读取path对应的文件
    FILE *fp = fopen(path, "rb");
    if (fp == nullptr) {
        LOGE("%s", "文件打开失败");
        return;
    }
    //读取分割文件的流
    FILE *index_fp = nullptr;
    int index = 0;
    for (int i = 0; i < fileSize; i++) {
        if (i % per_size == 0) {
            if (index_fp != nullptr) {
                fclose(index_fp);
            }
            index_fp = fopen(patches[index], "wb");
            index++;
            if (index_fp == nullptr) {
                LOGE("文件%s打开失败", patches[index]);
                return;
            }
        }
        fputc(fgetc(fp), index_fp);

        //读完之后释放流
        if (i + 1 == fileSize) {
            fclose(index_fp);
        }
    }
    fclose(fp);

    //释放内存
    for (int i = 0; i < splitCount; i++) {
        free(patches[i]);
    }
    free(patches);

    env->ReleaseStringUTFChars(path_, path);
    env->ReleaseStringUTFChars(pathPattern_, pathPattern);
}
```

---

### 实现JNI文件合并逻辑
```
/**
 * 合并拆分文件
 */
extern "C"
JNIEXPORT void JNICALL
Java_com_lxk_ndkdemo_JniFileOperation_merge(JNIEnv *env, jobject instance, jstring pathMerge_,
                                            jstring pathPattern_, jint count) {
    const char *pathMerge = env->GetStringUTFChars(pathMerge_, nullptr);
    const char *pathPattern = env->GetStringUTFChars(pathPattern_, nullptr);

    //创建合并文件的写流
    FILE *fp = fopen(pathMerge, "wb");

    for (int i = 0; i < count; i++) {
        char *index = new char[256];
        sprintf(index, pathPattern, i);
        //读取每个分割文件
        FILE *index_fp = fopen(index, "rb");
        if (index_fp == nullptr) {
            LOGE("文件%s读取失败", index)
            return;
        }
        //依次写入合并文件
        int ch;
        while ((ch = fgetc(index_fp)) != EOF) {
            fputc(ch, fp);
        }
        //关闭当前的分割文件流
        fclose(index_fp);
        //释放拆分文件名的内存
        free(index);
    }
    fclose(fp);

    env->ReleaseStringUTFChars(pathMerge_, pathMerge);
    env->ReleaseStringUTFChars(pathPattern_, pathPattern);
}
```

---

### 执行测试代码
```
private void testFileOperation() {
    if (hasFilePermission()) {
        new JniFileOperation().test();
        Toast.makeText(this, "任务完成，测试文件路径:" + Config.getBaseUrl(), Toast.LENGTH_SHORT).show();
    }
}
```
会在`/storage/emulated/0/NDKDemo/`下生成对应的测试文件。

---

[Demo地址：https://github.com/103style/NDKDoc/tree/master/NDKDemo](https://github.com/103style/NDKDoc/tree/master/NDKDemo)


以上
