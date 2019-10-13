>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.5 版本为例**

**本文为参考官方示例  [hello-jniCallback](https://github.com/googlesamples/android-ndk/tree/master/hello-jniCallback) 动手写的 Demo.**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

### 功能介绍
* 通过 `JNI` 获取并调用静态内部类 [JniHandler](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/java/com/lxk/ndkdemo/JniCallbackDemo.java) 的 静态方法`getBuildVersion()` 和 公开方法`getRuntimeMemorySize()`获取 **当前的系统版本** 以及 **当前可用内存**。
* 通过 [`JNI` 线程](https://baike.baidu.com/item/Pthread/4623312) 实现 **开始和停止每秒打印距离开始计时的秒数**。

---

###  编写测试代码
* 编写测试代码 [JniCallbackDemo](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/java/com/lxk/ndkdemo/JniCallbackDemo.java).

    ```
    public class JniCallbackDemo {
        private static final String TAG = "JniCallbackDemo";
        static {
            System.loadLibrary("jni_callback");
        }
        private int timeCount;
        public native void startTiming();
        public native void stopTiming();
    
        private void printTime() {
            Log.e(TAG, "timeCount = " + timeCount);
            timeCount++;
        }
        public static class JniHandler {
            public static String getBuildVersion() {
                return Build.VERSION.RELEASE;
            }
            public long getRuntimeMemorySize() {
                return Runtime.getRuntime().freeMemory();
            }
            private void updateStatus(String msg) {
                if (msg.toLowerCase().contains("error")) {
                    Log.e("JniHandler", "Native Err: " + msg);
                } else {
                    Log.i("JniHandler", "Native Msg: " + msg);
                }
            }
        }
    }
    ```
* 创建 [jni_callback.cpp](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/cpp/jni_callback.cpp).
* 添加以下代码到`CMakeLists.txt`.
    ```
    add_library(
            jni_callback
            SHARED
            jni_callback.cpp)
    target_link_libraries(
            jni_callback
            ${log-lib})
    ```

---

### 通过JNI实现功能逻辑
* **获取当前的系统版本和可用内存**。
    * 因为这个只需要执行一次就好了，我们可以放到`JNI_OnLoad`方法中去实现(`在应用层调用.so库首先会执行 JNI_OnLoad 方法`)。
    * 获取内部类用 `$` 而不是 `.`，例如：
    `env->FindClass("com/lxk/ndkdemo/JniCallbackDemo$JniHandler");`. 
    * 这里构建了一个结构体`jniCallback`保存获取的 **类** 和 **实例**。
    ```
    void queryRuntimeInfo(JNIEnv *env) {
        //获取getBuildVersion的方法id
        jmethodID staticMethodId = env->GetStaticMethodID(jniCallback.jniHandlerClz, "getBuildVersion",
                                                          "()Ljava/lang/String;");
        if (!staticMethodId) {
            LOGE("Failed to retrieve getBuildVersion() methodID");
            return;
        }
        //执行静态方法获取getBuildVersion的方法id
        jstring releaseVersion = static_cast<jstring>(env->CallStaticObjectMethod(
                jniCallback.jniHandlerClz, staticMethodId));
        //获取字符串的地址
        const char *version = env->GetStringUTFChars(releaseVersion, nullptr);
        if (!version) {
            LOGE("Unable to get version string");
            return;
        }
        LOGD("releaseVersion = %s", version);
        //释放字符串的内存
        env->ReleaseStringUTFChars(releaseVersion, version);
        //删除引用
        env->DeleteLocalRef(releaseVersion);
    
        //获取非静态方法 getRuntimeMemorySize 的id
        jmethodID methodId = env->GetMethodID(jniCallback.jniHandlerClz, "getRuntimeMemorySize", "()J");
        if (!methodId) {
            LOGE("Failed to retrieve getRuntimeMemorySize() methodID");
            return;
        }
        //调用 getRuntimeMemorySize
        jlong freeMemorySize = env->CallLongMethod(jniCallback.jniHandlerObj, methodId);
    
        //打印可用内存
        LOGD("Runtime free memory size: %"
                     PRId64, freeMemorySize);
    }
    
    JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
        LOGD("JNI_OnLoad");
        JNIEnv *env;
        //给jniCallback初始化地址
        memset(&jniCallback, 0, sizeof(jniCallback));
    
        jniCallback.javaVM = vm;
    
        if (vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6) != JNI_OK) {
            return JNI_ERR;
        }
        //获取JniCallbackDemo.JniHandler 类
        jclass clz = env->FindClass("com/lxk/ndkdemo/JniCallbackDemo$JniHandler");
        //赋值到结构体
        jniCallback.jniHandlerClz = static_cast<jclass>(env->NewGlobalRef(clz));
        if (!clz) {
            LOGE("FindClass JniCallbackDemo$JniHandler error");
            return JNI_ERR;
        }
        //获取构造函数
        jmethodID initMethodId = env->GetMethodID(jniCallback.jniHandlerClz, "<init>", "()V");
        //构建类
        jobject instance = env->NewObject(jniCallback.jniHandlerClz, initMethodId);
        if (!instance) {
            LOGE("NewObject jniHandler error")
            return JNI_ERR;
        }
        //赋值到结构体
        jniCallback.jniHandlerObj = env->NewGlobalRef(instance);
    
        //调用 JniHandler 的相关方法
        queryRuntimeInfo(env);
    
        jniCallback.done = 0;
        jniCallback.jniCallbackDemoObj = nullptr;
        return JNI_VERSION_1_6;
    }
    ```

* **创建线程执实现开始计时的逻辑：**
    * 通过`pthread_create`创建线程的时候，第一个参数：线程id的指针；第二个参数：线程属性的指针；第一个参数：在线程中运行的函数；第四个参数：运行函数的参数
    ```
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_lxk_ndkdemo_JniCallbackDemo_startTiming(JNIEnv *env, jobject instance) {
        LOGD("jni startTiming");
    
        //线程ID
        pthread_t threadInfo;
        //线程属性
        pthread_attr_t threadAttr;
        //初始化线程属性
        pthread_attr_init(&threadAttr);
        //设置脱离状态的属性
        pthread_attr_setdetachstate(&threadAttr, PTHREAD_CREATE_DETACHED);
    
        //互斥锁
        pthread_mutex_t lock;
        //初始化互斥锁
        pthread_mutex_init(&lock, nullptr);
    
        //获取当前类
        jclass clz = env->GetObjectClass(instance);
        //保存类和 实例 到 结构体中
        jniCallback.jniCallbackDemoClz = static_cast<jclass>(env->NewGlobalRef(clz));
        jniCallback.jniCallbackDemoObj = env->NewGlobalRef(instance);
    
        // StartTiming ：在线程中运行的函数  jniCallback 运行函数的参数
        int result = pthread_create(&threadInfo, &threadAttr, StartTiming, &jniCallback);
        assert(result == 0);
        //删除线程属性
        pthread_attr_destroy(&threadAttr);
    }
    ```
* **线程的执行函数：**
    ```
    /**
     * 调用 类 instance 的 void方法 methodId
     */
    void sendJavaMsg(JNIEnv *env, jobject instance, jmethodID methodId, const char *msg) {
        LOGD("jni sendJavaMsg");
        //获取字符串
        jstring javaMsg = env->NewStringUTF(msg);
        //调用对应方法
        env->CallVoidMethod(instance, methodId, javaMsg);
        //删除本地引用
        env->DeleteLocalRef(javaMsg);
    }

    void *StartTiming(void *context) {
        //获取参数
        JniCallback *JniCallback = static_cast<jni_callback *>(context);
        JavaVM *javaVm = JniCallback->javaVM;
        JNIEnv *env;

        jint res = javaVm->GetEnv((void **) (&env), JNI_VERSION_1_6);
        LOGD("javaVm->GetEnv() res = %d", res);
        if (res != JNI_OK) {
            //链接到虚拟机
            res = javaVm->AttachCurrentThread(&env, nullptr);
            if (JNI_OK != res) {
                LOGE("Failed to AttachCurrentThread, ErrorCode = %d", res);
                return nullptr;
            }
        } else {
            LOGE("javaVm GetEnv JNI_OK");
        }

        //获取 JniHandler 的 updateStatus 函数
        jmethodID statusId = env->GetMethodID(JniCallback->jniHandlerClz, "updateStatus",
                                              "(Ljava/lang/String;)V");

        sendJavaMsg(env, JniCallback->jniHandlerObj, statusId, "TimeThread status: initializing...");
        //获取 JniCallbackDemo 的 printTime 函数
        jmethodID timerId = env->GetMethodID(JniCallback->jniCallbackDemoClz, "printTime", "()V");
        //声明时间变量
        struct timeval beginTime, curTime, usedTime, leftTime;
        const struct timeval kOneSecond = {
                (__kernel_time_t) 1,
                (__kernel_suseconds_t) 0
        };

        sendJavaMsg(env, JniCallback->jniHandlerObj, statusId,
                    "TimeThread status: prepare startTiming ...");

        while (true) {
            //获取当前的时间 赋值给 beginTime
            gettimeofday(&beginTime, nullptr);
            //占有互斥锁
            pthread_mutex_lock(&JniCallback->lock);
            //获取当前的状态
            int done = JniCallback->done;
            if (JniCallback->done) {
                JniCallback->done = 0;
            }
            //释放互斥锁
            pthread_mutex_unlock(&JniCallback->lock);

            if (done) {
                LOGD("JniCallback done");
                break;
            }

            //调用 printTime 函数
            env->CallVoidMethod(JniCallback->jniCallbackDemoObj, timerId);

            //获取当前的时间 赋值给 curTime
            gettimeofday(&curTime, nullptr);

            //计算函数运行消耗的时间
            //usedTime = curTime - beginTime
            timersub(&curTime, &beginTime, &usedTime);

            //计算需要等待的时间
            //leftTime = kOneSecond - usedTime
            timersub(&kOneSecond, &usedTime, &leftTime);

            //构建等待的时间
            struct timespec sleepTime;
            sleepTime.tv_sec = leftTime.tv_sec;
            sleepTime.tv_nsec = leftTime.tv_usec * 1000;

            if (sleepTime.tv_sec <= 1) {
                //睡眠对应纳秒的时间
                nanosleep(&sleepTime, nullptr);
            } else {
                sendJavaMsg(env, JniCallback->jniHandlerObj, statusId,
                            "TimeThread error: processing too long!");
            }
        }
        sendJavaMsg(env, JniCallback->jniHandlerObj, statusId,
                    "TimeThread status: ticking stopped");
        //释放线程
        javaVm->DetachCurrentThread();
        return context;
    }
    ```

* **停止计时的逻辑：**
    ```
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_lxk_ndkdemo_JniCallbackDemo_stopTiming(JNIEnv *env, jobject instance) {
        LOGD("jni stopTiming");
    
        //占用互斥锁
        pthread_mutex_lock(&jniCallback.lock);
        //修改线程运行的条件
        jniCallback.done = 1;
        //释放互斥锁
        pthread_mutex_unlock(&jniCallback.lock);
    
        //初始化等待时间
        struct timespec sleepTime;
        memset(&sleepTime, 0, sizeof(sleepTime));
        sleepTime.tv_nsec = 100000000;
    
        while (jniCallback.done) {
            nanosleep(&sleepTime, nullptr);
        }
    
        //删除引用
        env->DeleteGlobalRef(jniCallback.jniCallbackDemoClz);
        env->DeleteGlobalRef(jniCallback.jniCallbackDemoObj);
        jniCallback.jniCallbackDemoObj = nullptr;
        jniCallback.jniCallbackDemoClz = nullptr;
    
        //删除互斥锁
        pthread_mutex_destroy(&jniCallback.lock);
    }
    ```
---

### 执行测试代码
```
private void testJniCallback() {
    jniCallbackRun(true);
}

private void jniCallbackRun(boolean run) {
    if (jniCallbackDemo == null) {
        jniCallbackDemo = new JniCallbackDemo();
    }
    if (run) {
        Toast.makeText(this, "开始计时，请查看控制台日志输出", Toast.LENGTH_SHORT).show();
        jniCallbackDemo.startTiming();
    } else {
        jniCallbackDemo.stopTiming();
    }
}
```
点击按钮**jni callback test**，在控制台即会输出以下信息。
```
08-22 17:00:34.726 18711-18711/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][63]: JNI_OnLoad
08-22 17:00:34.726 18711-18711/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][42]: releaseVersion = 6.0.1
08-22 17:00:34.726 18711-18711/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][59]: Runtime free memory size: 2439224
08-22 17:00:34.726 18711-18711/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][207]: jni startTiming
08-22 17:00:34.726 18711-18758/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][120]: javaVm->GetEnv() res = -2
08-22 17:00:34.726 18711-18758/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][104]: jni sendJavaMsg
08-22 17:00:34.736 18711-18758/com.lxk.ndkdemo I/JniHandler: Native Msg: TimeThread status: initializing...
08-22 17:00:34.736 18711-18758/com.lxk.ndkdemo D/JNI: [jni_callback.cpp][104]: jni sendJavaMsg
08-22 17:00:34.736 18711-18758/com.lxk.ndkdemo I/JniHandler: Native Msg: TimeThread status: prepare startTiming ...
08-22 17:00:34.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 0
08-22 17:00:35.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 1
08-22 17:00:36.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 2
08-22 17:00:37.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 3
08-22 17:00:38.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 4
08-22 17:00:39.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 5
08-22 17:00:40.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 6
08-22 17:00:41.736 18711-18758/com.lxk.ndkdemo E/JniCallbackDemo: timeCount = 7
...
```
---

### 相关文档
* [jni 线程介绍](https://baike.baidu.com/item/Pthread/4623312)
* [官方hello-jniCallback](https://github.com/googlesamples/android-ndk/tree/master/hello-jniCallback)

---

### 源码
* [jni_callback.cpp 完整代码 ](https://github.com/103style/NDKDoc/blob/master/NDKDemo/app/src/main/cpp/jni_callback.cpp)
* [Demo地址：https://github.com/103style/NDKDoc/tree/master/NDKDemo](https://github.com/103style/NDKDoc/tree/master/NDKDemo)

以上
