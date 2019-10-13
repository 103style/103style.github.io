>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.4.2 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

前先阅读 [JNI的数据类型](https://www.jianshu.com/p/8eac2a6dfe4e)。


---


### 目录
* JNI访问Java成员变量
* JNI访问Java静态变量
* JNI访问Java非静态方法
* JNI访问Java静态方法
* JNI访问Java构造方法
* 小结
* 参考文章

---

### JNI访问Java成员变量
我们在 [Demo HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK) 这里继续添加示例。

* 首先定义变量`showText`：
    ```
    public String showText = "Hello World";
    ```

* 添加`native`方法`accessField()`：
    ```
    public native void accessField();
    ```
    选中`accessField`,按 `Alt+Enter`快捷添加`.cpp`中方法`Java_com_example_myapplication_MainActivity_accessField`

* 在方法中实现修改属性的逻辑：
    ```
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_example_myapplication_MainActivity_accessField(JNIEnv *env, jobject instance) {
        //获取类
        jclass jcla = env->GetObjectClass(instance);
        //获取类的成员变量showText的id
        jfieldID jfId = env->GetFieldID(jcla, "showText", "Ljava/lang/String;");

        jstring after = env->NewStringUTF("Hello NDK");
        //修改属性id对应的值
        env->SetObjectField(instance, jfId, after);
    }
    ```

* 调用验证结果：
    ```
    TextView accessFiledShow = findViewById(R.id.tv_access_filed_show);
    String res = "before: " + showText;
    //通过ndk 修改成员变量
    accessField();
    res += ", after:" + showText;
    accessFiledShow.setText(res);
    ```

---

### JNI访问Java静态变量
我们在 [Demo HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK) 这里继续添加示例。
* 同上添加静态属性和 `native`方法:
    ```
    private static String staticString = "static string";

    public native void accessStaticField();
    ```
* 按 `Alt+Enter`快捷添加`.cpp`中方法`Java_com_example_myapplication_MainActivity_accessStaticField`，添加如下代码:
    ```
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_example_myapplication_MainActivity_accessStaticField(JNIEnv *env, jobject instance) {
        //获取类
        jclass oClass = env->GetObjectClass(instance);
        //获取静态变量id
        jfieldID staticFid = env->GetStaticFieldID(oClass, "staticString", "Ljava/lang/String;");
    
        //设置静态变量
        jstring after = env->NewStringUTF("static field update in jni");
        env->SetStaticObjectField(oClass, staticFid, after);
    }
    ```
* 调用验证结果：
    ```
    TextView accessStaticFiledShow = findViewById(R.id.tv_static_access_filed_show);
    res = "before: " + staticString;
    //通过ndk 修改静态变量
    accessStaticField();
    res += ", after:" + staticString;
    accessStaticFiledShow.setText(res);
    ```

---

### JNI访问Java非静态方法
我们在 [Demo HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK) 这里继续添加示例。
* 同上添加`Java`非静态方法和 `native`方法:
    ```
    public native String accessMethod();

    public String getAuthName(String name) {
        Log.e(TAG, "name = " + name);
        return name;
    }
    ```
* 按 `Alt+Enter`快捷添加`.cpp`中方法`Java_com_example_myapplication_MainActivity_accessMethod`，添加如下代码：
    ```
    extern "C"
    JNIEXPORT jstring JNICALL
    Java_com_example_myapplication_MainActivity_accessMethod(JNIEnv *env, jobject instance) {
        //获取类
        jclass jCla = env->GetObjectClass(instance);
        //获取方法id  第二个参数：方法名  第三个参数：(参数)返回值 的类型描述
        jmethodID methodID = env->GetMethodID(jCla, "getAuthName",
                                              "(Ljava/lang/String;)Ljava/lang/String;");
        jstring res = env->NewStringUTF("103style");
        jobject objRes = env->CallObjectMethod(instance, methodID, res);
        return static_cast<jstring>(objRes);
    }
    ```
* 调用验证结果：
    ```
    TextView tvName = findViewById(R.id.tv_auth_name);
    res = "authName = " + accessMethod();
    tvName.setText(res);
    ```

---

### JNI访问Java静态方法
我们在 [Demo HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK) 这里继续添加示例。
* 同上添加`Java`静态方法和 `native`方法:
    ```
    private static int getRandomValue(int max) {
        return new Random().nextInt(max);
    }

   public native int accessStaticMethod(int max);
    ```
* 按 `Alt+Enter`快捷添加`.cpp`中方法`Java_com_example_myapplication_MainActivity_accessStaticMethod__I`，添加如下代码：
    ```
    extern "C"
    JNIEXPORT jint JNICALL
    Java_com_example_myapplication_MainActivity_accessStaticMethod__I(JNIEnv *env, jobject instance,
                                                                      jint max) {
        //获取类
        jclass jCla = env->GetObjectClass(instance);
        //获取静态方法的id
        jmethodID methodID = env->GetStaticMethodID(jCla, "getRandomValue", "(I)I");
        //调用静态方法
        jint res = env->CallStaticIntMethod(jCla, methodID, max);
        //返回结果
        return res;
    }
    ```

* 调用验证结果：
    ```
    TextView staticMethodShow = findViewById(R.id.tv_static_method_show);
    res = "accessStaticMethod(100) = " + accessStaticMethod(100);
    staticMethodShow.setText(res);
    ```

---

### JNI访问Java构造方法
我们在 [Demo HelloNDK](https://github.com/103style/NDKDoc/tree/master/HelloNDK) 这里继续添加示例。

* 同上添加 `native`方法:
    ```
    public native Date accessConstructor();
    ```
* 按 `Alt+Enter`快捷添加`.cpp`中方法`Java_com_example_myapplication_MainActivity_accessConstructor`，添加如下代码：
    ```
    extern "C"
    JNIEXPORT jobject JNICALL
    Java_com_example_myapplication_MainActivity_accessConstructor(JNIEnv *env, jobject instance) {
        //得到TestClass对应的jclass
    //    jclass jCla = env->FindClass("com/example/myapplication/TestClass");
        jclass jCla = env->FindClass("java/util/Date");
        //获取构造函数的methodId  构造函数为 void函数 对应的方法名为<init>
        jmethodID methodID = env->GetMethodID(jCla, "<init>", "()V");
        jobject jTestClass = env->NewObject(jCla, methodID);
        return jTestClass;
    }
    ```

* 调用验证结果：
    ```
    TextView staticConstShow = findViewById(R.id.tv_const_show);
    res = accessConstructor().toString();
    staticConstShow.setText(res);
    ```

---

### 小结
* `JNI`获取类的成员变量的ID调用`GetFieldID`获取，通过`Set[类型]Field`修改变量值。
* `JNI`获取类的成员变量的ID调用`GetStaticFieldID`获取，通过`SetStatic[类型]Field`修改变量值。
* `JNI`获取类的方法的ID调用`GetMethodID`获取，通过`Call[类型]Method`调用方法。
* `JNI`获取类的静态方法的ID调用`GetStaticMethodID`获取，通过`CallStatic[类型]Method`调用方法。
* `JNI`获取类的构造方法的ID调用`GetMethodID`获取，通过`NewObject`构造，构造函数名为`"<init>"`。
* [Demo地址](https://github.com/103style/NDKDoc/tree/master/HelloNDK)


---

### 参考文章
* [Android Studio NDK开发（三）：属性访问](https://www.jianshu.com/p/d3171f51b80d)
* [Android Studio NDK开发（四）：方法访问](https://www.jianshu.com/p/949c66746e84)


---

以上
