>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


**本文操作以 Android Studio 3.5 版本为例**

---

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)


---

### 功能介绍
* 参考 [OpenGL ES 3.0  Hello_Triangle](https://www.jianshu.com/p/90e47b14d9fa) 通过 `JNI` 调用 `OpenGL ES 3.0` 绘制三角形 并 改变背景色。

[Demo源码链接](https://github.com/103style/NDKDoc/tree/master/HelloGLES3)


相关资料：[OpenGL ES 3.0 相关介绍汇总](https://www.jianshu.com/p/216c7e91960a)

---

### 目录
* 创建及配置工程
* 修改JNI代码实现功能逻辑
* 运行程序查看效果
* 相关资源链接

---

###  创建及配置工程
* 创建一个新工程，在 **Choose your project** 时选择 **native c++** 模板。
* 在 `AndroidManifest` 中添加`Open GL ES`版本声明：
    ```
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.lxk.hellogles3">
    
        <uses-feature
            android:glEsVersion="0x00030000"
            android:required="true" />
    
        <application
            ....
        </application>
    
    </manifest>
    ```
* 修改 `native-lib.cpp` 为 `native-gles3.cpp`。
* 修改`CMakeLists.txt` 为以下内容:
    ```
    cmake_minimum_required(VERSION 3.4.1)
    
    ##官方标准配置
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fno-rtti -fno-exceptions -Wall")
    
    add_library(native-gles3
            SHARED
            native-gles3.cpp)
    
    if (${ANDROID_PLATFORM_LEVEL} LESS 11)
        message(FATAL_ERROR "OpenGL 2 is not supported before API level 11 (currently using ${ANDROID_PLATFORM_LEVEL}).")
        return()
    elseif (${ANDROID_PLATFORM_LEVEL} LESS 18)
        add_definitions("-DDYNAMIC_ES3")
        set(OPENGL_LIB GLESv2)
    else ()
        set(OPENGL_LIB GLESv3)
    endif (${ANDROID_PLATFORM_LEVEL} LESS 11)
    
    target_link_libraries(native-gles3
            android
            EGL
            ${OPENGL_LIB}
            log)
    ```
* 创建修改 [相关 java 文件（MainActivity、GLSurfaceView、Renderer）](https://github.com/103style/NDKDoc/tree/master/HelloGLES3/app/src/main/java/com/lxk/hellogles3)。
  **GLSurfaceView 一定要调用**  ` setEGLContextClientVersion(3);` **设置 OpenGL ES 的版本**。



---

### 修改JNI代码实现功能逻辑
* 导入相关头文件：
    ```
    #include <jni.h>
    #include <string>

    #include <GLES3/gl3.h>
    #include <GLES3/gl3ext.h>

    #include <android/log.h>
    #include "LogUtils.h"
    ```
    [LogUtils.h](https://github.com/103style/NDKDoc/blob/master/HelloGLES3/app/src/main/cpp/LogUtils.h)

* 选中 [GLES3Render](https://github.com/103style/NDKDoc/blob/master/HelloGLES3/app/src/main/java/com/lxk/hellogles3/GLES3Render.java) 中报红的 `native` 方法，按 `alt + enter` 快速创建 C 方法。

* 编写 **顶点着色器** 和 **片段着色器** 源码：
    ```
    /**
     * 顶点着色器源码
     */
    auto gl_vertexShader_source =
            "#version 300 es\n"
            "layout(location = 0) in vec4 vPosition;\n"
            "void main() {\n"
            "   gl_Position = vPosition;\n"
            "}\n";
    /**
     * 片段着色器源码
     */
    auto gl_fragmentShader_source =
            "#version 300 es\n"
            "precision mediump float;\n"
            "out vec4 fragColor;\n"
            "void main() {\n"
            "   fragColor = vec4(1.0,1.0,0.0,1.0);\n"
            "}\n";
    ```

* 编写编译着色器源码的方法：
    ```
    /**
     * 编译着色器源码
     * @param shaderType 着色器类型
     * @param shaderSource  源码
     * @return
     */
    GLuint compileShader(GLenum shaderType, const char *shaderSource) {
        //创建着色器对象
        GLuint shader = glCreateShader(shaderType);
        if (!shader) {
            return 0;
        }
        //加载着色器源程序
        glShaderSource(shader, 1, &shaderSource, nullptr);
        //编译着色器程序
        glCompileShader(shader);
    
        //获取编译状态
        GLint compileRes;
        glGetShaderiv(shader, GL_COMPILE_STATUS, &compileRes);
    
        if (!compileRes) {
            //获取日志长度
            GLint infoLen = 0;
            glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLen);
    
            if (infoLen > 0) {
                char *infoLog = static_cast<char *>(malloc(sizeof(char) * infoLen));
                //获取日志信息
                glGetShaderInfoLog(shader, infoLen, nullptr, infoLog);
                LOGE("compile shader error : %s", infoLog);
                free(infoLog);
            }
            //删除着色器
            glDeleteShader(shader);
            return 0;
        }
        return shader;
    }
    ```
* 编写链接着色器程序的代码：
    ```
    /**
     * 链接着色器程序
     */
    GLuint linkProgram(GLuint vertexShader, GLuint fragmentShader) {
        //创建程序
        GLuint programObj = glCreateProgram();
        if (programObj == 0) {
            LOGE("create program error");
            return 0;
        }
        //加载着色器载入程序
        glAttachShader(programObj, vertexShader);
        glAttachShader(programObj, fragmentShader);
    
        //链接着色器程序
        glLinkProgram(programObj);
    
        //检查程序链接状态
        GLint linkRes;
        glGetProgramiv(programObj, GL_LINK_STATUS, &linkRes);
    
        if (!linkRes) {//链接失败
            //获取日志长度
            GLint infoLen;
            glGetProgramiv(programObj, GL_INFO_LOG_LENGTH, &infoLen);
            if (infoLen > 1) {
                //获取并输出日志
                char *infoLog = static_cast<char *>(malloc(sizeof(char) * infoLen));
                glGetProgramInfoLog(programObj, infoLen, nullptr, infoLog);
                LOGE("Error link program : %s", infoLog);
                free(infoLog);
            }
            //删除着色器程序
            glDeleteProgram(programObj);
            return 0;
        }
        return programObj;
    }
    ```
* 在`Java_com_lxk_hellogles3_GLES3Render_surfaceChanged`中编译链接着色器程序：
    ```
    /**
     * 着色器程序
     */
    GLuint program;
    
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_lxk_hellogles3_GLES3Render_surfaceChanged(JNIEnv *env, jobject thiz, jint w, jint h) {
        printGLString("Version", GL_VERSION);
        printGLString("Vendor", GL_VENDOR);
        printGLString("Renderer", GL_RENDERER);
        printGLString("Extension", GL_EXTENSIONS);
    
        LOGD("surfaceChange(%d,%d)", w, h);
    
        //编译着色器源码
        GLuint vertexShader = compileShader(GL_VERTEX_SHADER, gl_vertexShader_source);
        GLuint fragmentShader = compileShader(GL_FRAGMENT_SHADER, gl_fragmentShader_source);
        //链接着色器程序
        program = linkProgram(vertexShader, fragmentShader);
    
        if (!program) {
            LOGE("linkProgram error");
            return;
        }
    
        //设置程序窗口
        glViewport(0, 0, w, h);
    }
    ```

* 设置三角形的顶点坐标数组 ：
    ```
    /**
     * 顶点坐标
     */
    const GLfloat vVertex[] = {
            0.0f, 0.5f, 0.0f,
            -0.5f, -0.5f, 0.0f,
            0.5f, -0.5f, 0.0f
    };
    ```

* 设置 RGB颜色值：
    ```
    static float r;
    static float g;
    static float b;
    
    /**
     * 修改背景颜色
     */
    void changeBg() {
        r += 0.01f;
        if (r > 1.0f) {
            g += 0.01f;
            if (g > 1.0f) {
                b += 0.01f;
                if (b > 1.0f) {
                    r = 0.01f;
                    g = 0.01f;
                    b = 0.01f;
                }
            }
        }
    }
    ```

*  在`Java_com_lxk_hellogles3_GLES3Render_drawFrame`中绘制三角形以及改变背景色：
    ```
    /**
     * 顶点属性索引
     */
    GLuint vertexIndex = 0;
    
    extern "C"
    JNIEXPORT void JNICALL
    Java_com_lxk_hellogles3_GLES3Render_drawFrame(JNIEnv *env, jobject thiz) {
        //改变颜色值
        changeBg();
    
        glClearColor(r, g, b, 1.0f);
        //清空颜色缓冲区
        glClear(GL_COLOR_BUFFER_BIT);
    
        //设置为活动程序
        glUseProgram(program);

        //加载顶点坐标
        glVertexAttribPointer(vertexIndex, 3, GL_FLOAT, GL_FALSE, 0, vVertex);   
        //启用通用顶点属性数组
        glEnableVertexAttribArray(vertexIndex);
        //绘制三角形
        glDrawArrays(GL_TRIANGLES, 0, 3);
        //禁用通用顶点属性数组
        glDisableVertexAttribArray(vertexIndex);
    }
    ```

---

### 运行程序
效果图如下：
![效果图](https://upload-images.jianshu.io/upload_images/1709375-5348a338d641147c.gif?imageMogr2/auto-orient/strip)



---

### 相关资源链接
* [Demo源码链接](https://github.com/103style/NDKDoc/tree/master/HelloGLES3)
* [OpenGL ES 3.0 相关介绍汇总](https://www.jianshu.com/p/216c7e91960a)

---

以上
