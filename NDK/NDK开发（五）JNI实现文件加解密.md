# NDK开发（五）JNI实现文件加解密 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---


### 目录
* 编写测试代码
* 实现创建文件逻辑
* 实现JNI加密逻辑
* 实现JNI解密逻辑
* 执行测试代码

---

### 编写测试代码
* 创建`Encryptor`类，编写对应的测试代码：
    ```
    public class Encryptor {
        private static final String TAG = "Encryptor";
        static {
            System.loadLibrary("encryptor");
        }
        private final String BASE_URL = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator;
        /**
         * 加密
         *
         * @param normalPath  文件路径
         * @param encryptPath 加密之后的文件路径
         */
        public native static void encryption(String normalPath, String encryptPath);
        /**
         * 解密
         *
         * @param encryptPath 加密之后的文件路径
         * @param decryptPath 解密之后的文件路径
         */
        public native static void decryption(String encryptPath, String decryptPath);
        /**
         * 创建文件
         *
         * @param normalPath 文件路径
         */
        private native void createFile(String normalPath);
        /**
         * 测试加解密
         */
        public void test() {
            String fileName = "testJni.txt";
            String encryptPath = encryption(fileName);
            decryption(encryptPath);
        }
        /**
         * 加密
         */
        public String encryption(String fileName) {
            String normalPath = BASE_URL + fileName;
            File file = new File(normalPath);
            if (!file.exists()) {
                createFile(normalPath);
            }
            String encryptPath = BASE_URL + "encryption_" + fileName;
            Encryptor.encryption(normalPath, encryptPath);
            Log.d(TAG, "加密完成了...");
            return encryptPath;
        }
        /**
         * 解密
         */
        public void decryption(String encryptPath) {
            if (!new File(encryptPath).exists()) {
                Log.d(TAG, "解密文件不存在");
                return;
            }
            String decryptPath = encryptPath.replace("encryption_", "decryption_");
            Encryptor.decryption(encryptPath, decryptPath);
            Log.d(TAG, "解密完成了...");
        }
    }
    ```
* 创建`encryptor.cpp`
* 添加以下代码到`CMakeLists.txt`：
    ```
    add_library(encryptor
            SHARED
            src/main/cpp/encryptor.cpp)
    target_link_libraries(
            encryptor
            ${log-lib})
    ```

---

### 实现创建文件逻辑
[ fopen()函数介绍](https://www.runoob.com/cprogramming/c-function-fopen.html)。


```
#include <cstdio>

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapplication_Encryptor_createFile(JNIEnv *env, jobject instance,
                                                    jstring normalPath_) {
    //获取字符串保存在JVM中内存中
    const char *normalPath = env->GetStringUTFChars(normalPath_, nullptr);
    //打开 normalPath  wb:只写打开或新建一个二进制文件；只允许写数据
    FILE *fp = fopen(normalPath, "wb");

    //把字符串写入到指定的流 stream 中，但不包括空字符。
    fputs("Hi, this file is created by JNI, and my name is 103style.", fp);

    //关闭流 fp。刷新所有的缓冲区
    fclose(fp);
    //释放JVM保存的字符串的内存
    env->ReleaseStringUTFChars(normalPath_, normalPath);
}
```

---

### 实现JNI加密逻辑
* [ fopen()函数介绍](https://www.runoob.com/cprogramming/c-function-fopen.html)
* [fputs()函数介绍](https://www.runoob.com/cprogramming/c-function-fputs.html)
* [fgetc()函数介绍](https://www.runoob.com/cprogramming/c-function-fgetc.html)
```
#include <android/log.h>
#include <cstdio>
#include <cstring>

#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"Encryptor",FORMAT,##__VA_ARGS__);

char password[] = "103style";

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapplication_Encryptor_encryption(JNIEnv *env, jclass type, jstring normalPath_,
                                                    jstring encryptPath_) {
    //获取字符串保存在JVM中内存中
    const char *normalPath = env->GetStringUTFChars(normalPath_, nullptr);
    const char *encryptPath = env->GetStringUTFChars(encryptPath_, nullptr);

    LOGE("normalPath = %s, encryptPath = %s", normalPath, encryptPath);

    //rb:只读打开一个二进制文件，允许读数据。
    //wb:只写打开或新建一个二进制文件；只允许写数据
    FILE *normal_fp = fopen(normalPath, "rb");
    FILE *encrypt_fp = fopen(encryptPath, "wb");

    if (normal_fp == nullptr) {
        LOGE("%s", "文件打开失败");
        return;
    }

    //一次读取一个字符
    int ch = 0;
    int i = 0;
    size_t pwd_length = strlen(password);
    while ((ch = fgetc(normal_fp)) != EOF) { //End of File
        //写入(异或运算)
        fputc(ch ^ password[i % pwd_length], encrypt_fp);
        i++;
    }

    //关闭流 normal_fp和encrypt_fp。刷新所有的缓冲区
    fclose(normal_fp);
    fclose(encrypt_fp);

    //释放JVM保存的字符串的内存
    env->ReleaseStringUTFChars(normalPath_, normalPath);
    env->ReleaseStringUTFChars(encryptPath_, encryptPath);
}
```

---

### 实现JNI解密逻辑
* [ fopen()函数介绍](https://www.runoob.com/cprogramming/c-function-fopen.html)
* [fputs()函数介绍](https://www.runoob.com/cprogramming/c-function-fputs.html)
* [fgetc()函数介绍](https://www.runoob.com/cprogramming/c-function-fgetc.html)

```
#include <android/log.h>
#include <cstdio>
#include <cstring>

#define LOGE(FORMAT, ...) __android_log_print(ANDROID_LOG_ERROR,"Encryptor",FORMAT,##__VA_ARGS__);

char password[] = "103style";

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapplication_Encryptor_decryption(JNIEnv *env, jclass type, jstring encryptPath_,
                                                    jstring decryptPath_) {
    ////获取字符串保存在JVM中内存中
    const char *encryptPath = env->GetStringUTFChars(encryptPath_, nullptr);
    const char *decryptPath = env->GetStringUTFChars(decryptPath_, nullptr);

    LOGE("encryptPath = %s, decryptPath = %s", encryptPath, decryptPath);

    //rb:只读打开一个二进制文件，允许读数据。
    //wb:只写打开或新建一个二进制文件；只允许写数据
    FILE *encrypt_fp = fopen(encryptPath, "rb");
    FILE *decrypt_fp = fopen(decryptPath, "wb");

    if (encrypt_fp == nullptr) {
        LOGE("%s", "加密文件打开失败");
        return;
    }

    int ch;
    int i = 0;
    size_t pwd_length = strlen(password);
    while ((ch = fgetc(encrypt_fp)) != EOF) {
        fputc(ch ^ password[i % pwd_length], decrypt_fp);
        i++;
    }

    //关闭流 encrypt_fp 和 decrypt_fp。刷新所有的缓冲区
    fclose(encrypt_fp);
    fclose(decrypt_fp);

    //释放JVM保存的字符串的内存
    env->ReleaseStringUTFChars(encryptPath_, encryptPath);
    env->ReleaseStringUTFChars(decryptPath_, decryptPath);
}
```

---


### 执行测试代码
```
private void testEncryptor() {
    if (ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
            != PackageManager.PERMISSION_GRANTED) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            requestPermissions(new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, 0x1024);
            return;
        }
    }
    new Encryptor().test();
}
```
执行程序会在 **手机根目录** 生成以下三个文件：
* `testJni.txt`：原文件
* `encryption_testJni.txt`：加密之后的文件
* `decryption_testJni.txt`：解密之后的文件

---

[demo地址: https://github.com/103style/NDKDoc/tree/master/HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK)

以上
