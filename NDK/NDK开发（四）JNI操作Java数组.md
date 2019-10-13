>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)


前先阅读 [JNI的数据类型](https://www.jianshu.com/p/8eac2a6dfe4e)。

---


### 目录
* 准备
* 构建数组
* 对数组进行排序

---

### 准备
* 创建`JniArrayOperation`类，编写对应的测试代码：
    ```
    public class JniArrayOperation {
        static {
            System.loadLibrary("array_operation");
        }
        public native int[] getIntArray(int length);
        public native void sortIntArray(int[] arr);
        public void test() {
            //获取随机的20个数
            int[] array = getIntArray(20);
            for (int i : array) {
                System.out.print(i + ", ");
            }
            System.out.println();
            //对数组进行排序
            sortIntArray(array);
            System.out.println("After sort:");
            for (int i : array) {
                System.out.print(i + ", ");
            }
            System.out.println();
        }
    }
    ```
* 创建`array_operation.cpp`
* 修改`CMakeLists.txt`
    ```
    add_library( array_operation
            SHARED
            src/main/cpp/array_operation.cpp)
    target_link_libraries( hello-ndk array_operation
                           ${log-lib} )
    ```


---

### 构建数组
```
#include <jni.h>
#include <random>

//数组元素最大值
const jint max = 100;

extern "C"
JNIEXPORT jintArray JNICALL
Java_com_example_myapplication_JniArrayOperation_getIntArray(JNIEnv *env, jobject instance,
                                                             jint length) {
    //创建一个指定大小的数组
    jintArray array = env->NewIntArray(length);

    jint *elementsP = env->GetIntArrayElements(array, nullptr);

    //设置 0-100的随机元素
    jint *startP = elementsP;
    for (; startP < elementsP + length; startP++) {
        *startP = static_cast<jint>(random() % max);
    }
    env->ReleaseIntArrayElements(array, elementsP, 0);
    return array;
}
```

---

### 对数组进行排序
```
int compare(const void *a, const void *b) {
    return *(int *) a - *(int *) b;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapplication_JniArrayOperation_sortIntArray(JNIEnv *env, jobject instance,
                                                              jintArray arr_) {
    //获取数组起始元素的指针
    jint *arr = env->GetIntArrayElements(arr_, nullptr);
    //获取数组长度
    jint len = env->GetArrayLength(arr_);
    //排序
    qsort(arr, len, sizeof(jint), compare);

    //第三个参数 同步
    //0：Java数组进行更新，并且释放C/C++数组
    //JNI_ABORT：Java数组不进行更新，但是释放C/C++数组
    //JNI_COMMIT：Java数组进行更新，不释放C/C++数组(函数执行完后，数组还是会释放的)
    env->ReleaseIntArrayElements(arr_, arr, 0);
}
```


调用`new JniArrayOperation().test();`运行程序：
```
System.out: 18, 87, 91, 23, 17, 65, 14, 97, 19, 66, 92, 54, 8, 54, 50, 56, 68, 19, 61, 39, 
System.out: After sort:
System.out: 8, 14, 17, 18, 19, 19, 23, 39, 50, 54, 54, 56, 61, 65, 66, 68, 87, 91, 92, 97, 
```

---

[demo地址: https://github.com/103style/NDKDoc/tree/master/HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK)


以上
