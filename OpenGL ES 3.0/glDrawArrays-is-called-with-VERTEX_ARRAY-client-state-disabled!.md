>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 



`OpenGL ES 3`

如下，在调用 `glEnableVertexAttribArray` 之后还是报错 `glDrawArrays is called with VERTEX_ARRAY client state disabled!`
```
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, vVertex);

glEnableVertexAttribArray(0);
```

解决方法如下：

* 检查 `AndroidManifest` 中是否添加了`Open GL ES`版本声明：
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

*  检查 **GLSurfaceView 是否调用**  ` setEGLContextClientVersion(3);` **设置 OpenGL ES 的版本**。

以上
