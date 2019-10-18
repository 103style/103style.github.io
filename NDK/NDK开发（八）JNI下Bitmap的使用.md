# NDK开发（八）JNI下Bitmap的使用 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

### 目录
* NDK 中的 Bitmap
* 编写测试代码
* 实现JNI下Bitmap使用的逻辑
* 执行测试代码

---

### NDK 中的 Bitmap
`NDK` 已经为我们准备好了操作 `Bitmap` 的相关头文件了，它就是 `<android/bitmap.h>`。

* 像素格式
  ```
  enum AndroidBitmapFormat {
      ANDROID_BITMAP_FORMAT_NONE      = 0,
      ANDROID_BITMAP_FORMAT_RGBA_8888 = 1,
      ANDROID_BITMAP_FORMAT_RGB_565   = 4,
      ANDROID_BITMAP_FORMAT_RGBA_4444 = 7,
      ANDROID_BITMAP_FORMAT_A_8       = 8,
  };
  ```
* `Bitmap`结构体，通过`AndroidBitmap_getInfo(JNIEnv* env, jobject jbitmap, AndroidBitmapInfo* info)`获取。
  ```
  typedef struct {
      //像素宽度
      uint32_t    width;
      //像素高度
      uint32_t    height;
      //每一行占几个字节
      uint32_t    stride;
      //像素格式
      int32_t     format;
      /** Unused. */
      uint32_t    flags;      // 0 for now
  } AndroidBitmapInfo;
  ```

* 提供的方法
  ```
  //获取bitmap的图片信息
  int AndroidBitmap_getInfo(JNIEnv* env, jobject jbitmap,
                            AndroidBitmapInfo* info);
  //获取像素信息
  int AndroidBitmap_lockPixels(JNIEnv* env, jobject jbitmap, void** addrPtr);
  //释放像素信息
  int AndroidBitmap_unlockPixels(JNIEnv* env, jobject jbitmap);
  ```
* 调用函数返回码
  ```
  enum {
      //调用成功
      ANDROID_BITMAP_RESULT_SUCCESS           = 0,
      //参数错误
      ANDROID_BITMAP_RESULT_BAD_PARAMETER     = -1,
      //出现异常
      ANDROID_BITMAP_RESULT_JNI_EXCEPTION     = -2,
     //分配失败
      ANDROID_BITMAP_RESULT_ALLOCATION_FAILED = -3,
  };
  ```
---

###  编写测试代码
* 创建类 [JniBitmapDemo](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/java/com/lxk/ndkdemo/JniBitmapDemo.java)，编写对应的测试代码：
    ```
    public class JniBitmapDemo {
        private static final String TAG = "JniBitmapDemo";
        static {
            System.loadLibrary("bitmap");
        }
    
        public native void passBitmap(Bitmap bitmap);

        public void test() {
            Bitmap bitmap = Bitmap.createBitmap(1, 1, Bitmap.Config.ARGB_8888);
            bitmap.eraseColor(0xff336699); // AARRGGBB
            byte[] bytes = new byte[bitmap.getWidth() * bitmap.getHeight() * 4];
            Buffer dst = ByteBuffer.wrap(bytes);
            bitmap.copyPixelsToBuffer(dst);
            // ARGB_8888 真实的存储顺序是 R-G-B-A
            Log.d(TAG, "R: " + Integer.toHexString(bytes[0] & 0xff));
            Log.d(TAG, "G: " + Integer.toHexString(bytes[1] & 0xff));
            Log.d(TAG, "B: " + Integer.toHexString(bytes[2] & 0xff));
            Log.d(TAG, "A: " + Integer.toHexString(bytes[3] & 0xff));
    
            passBitmap(bitmap);
        }
    }
    ```
* 创建`bitmap.cpp`.
* 添加以下代码到`CMakeLists.txt`：
    ```
    add_library(
            bitmap
            SHARED
            bitmap.cpp)
    target_link_libraries(
            bitmap
            jnigraphics
            ${log-lib})
    ```
---

### 实现JNI下Bitmap使用的逻辑
```
#include <jni.h>
#include <android/bitmap.h>
#include <android/log.h>
#include <cstring>
#include "LogUtils.h"


extern "C"
JNIEXPORT void JNICALL
Java_com_lxk_ndkdemo_JniBitmapDemo_passBitmap(JNIEnv *env, jobject instance, jobject bitmap) {

    if (nullptr == bitmap) {
        LOGE("bitmap is null");
    }

    AndroidBitmapInfo info;
    int result;

    //获取图片信息
    result = AndroidBitmap_getInfo(env, bitmap, &info);

    if (result != ANDROID_BITMAP_RESUT_SUCCESS) {
        LOGE("AndroidBitmap_getInfo failed, result: %d", result);
        return;
    }

    LOGD("bitmap width: %d, height: %d, format: %d, stride: %d", info.width, info.height,
         info.format, info.stride);

    unsigned char *addrPtr;

    // 获取像素信息
    result = AndroidBitmap_lockPixels(env, bitmap, reinterpret_cast<void **>(&addrPtr));

    if (result != ANDROID_BITMAP_RESULT_SUCCESS) {
        LOGE("AndroidBitmap_lockPixels failed, result: %d", result);
        return;
    }

    // 执行图片操作的逻辑
    int length = info.stride * info.height;
    for (int i = 0; i < length; ++i) {
        LOGD("value: %x", addrPtr[i]);
    }

    // 像素信息不再使用后需要解除锁定
    result = AndroidBitmap_unlockPixels(env, bitmap);
    if (result != ANDROID_BITMAP_RESULT_SUCCESS) {
        LOGE("AndroidBitmap_unlockPixels failed, result: %d", result);
    }

}
```
---

### 执行测试代码
```
new JniBitmapDemo().test();
```
会在控制台打印`Bitmap`对应的信息。


如果出现`undefined reference to AndroidBitmap_getInfo`类似的报错信息。
是因为`CMakeLists.txt`中没有添加` jnigraphics`.
```
target_link_libraries(
        bitmap
        jnigraphics
        ${log-lib})
```
---

[Demo地址：https://github.com/103style/NDKDoc/tree/master/NDKDemo](https://github.com/103style/NDKDoc/tree/master/NDKDemo)

以上
