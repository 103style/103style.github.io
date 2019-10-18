# NDK开发（二）JNI的数据类型 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

[NDK开发文章汇总](https://www.jianshu.com/p/b18426df68f8)

---

### 目录
* 基本数据类型
* 引用数据类型
* JNI的数据类型描述符
* 示例
* 参考文章


---

### 基本数据类型
  | Java数据类型 | JNI本地类型 | C/C++数据类型 | 数据类型描述 |
  |:-:|:-:|:-:|:-:|
  | `boolean` | `jboolean` | `unsigned char` | C/C++无符号8位整数 |
  | `byte` | `jbyte` | `signed char` | C/C++有符号8位整数 |
  | `char` | `jchar` | `unsigned short` | C/C++无符号16位整数 |
  | `short` | `jshort` | `signed short` | C/C++有符号16位整数 |
  | `int` | `jint` | `signed int` | C/C++有符号32位整数 |
  | `long` | `jlong` | `signed long` | C/C++有符号64位整数 |
  | `float` | `jfloat` | `float` | C/C++32位浮点数 |
  | `double` | `jdouble` | `double` | C/C++64位浮点数 |

---

### 引用数据类型
  | Java的类类型 | JNI的引用类型 |
  |:-:|:-:|
  | `java.lang.Object` | `jobject` |
  | ` java.lang.String` | `jstring` |  
  | ` java.lang.Class` | `jclass` |  
  | `Object[]` | `jobjectArray` |  
  | `boolean[]` | `jbooleanArray` |  
  | `byte[]` | `jbyteArray` |  
  | `char[]` | `jcharArray` |  
  | `short[]` | `jshortArray` |  
  | `int[]` | `jintArray` |  
  | `long[]` | `jlongArray` |  
  | `float[]` | `jfloatArray` |  
  | `double[]` | `jdobleArray` |  
  | `java.lang.Throwable` | `jthrowable` |  
  | `void` | `void` |  

---

### JNI的数据类型描述符
| Java类型 | 类型描述符 |
|:-:|:-:|
| `int` | `I` |
| `long` | `J` |
| `byte` | `B` |
| `short` | `S` |
| `char` | `C` |
| `float` | `F` |
| `double` | `D` |
| `boolean` | `Z` |
| `void` | `V` |
| 其他引用类型 |  `L`+`类全名`+`;` |
| 数组 | `[` |
| 方法 | (参数)返回值 |

---

### 示例

* **String 类**
    ```
    Java 类型：java.lang.String
    JNI 描述符：Ljava/lang/String;
    即一个 Java 类对应的描述符，就是 L 加上类的全名，其中 . 要换成 / ，最后 不要忘掉末尾的分号。
    ```

* **数组**
    ```
    Java 类型：String[]
    JNI 描述符：[Ljava/lang/String;
    Java 类型：int[][]
    JNI 描述符：[[I
    数组就是简单的在类型描述符前加 [ 即可，二维数组就是两个 [ ，以此类推。
    ```
* **方法**
    ```
    Java 方法：long f (int n, String s, int[] arr);
    JNI 描述符：(ILjava/lang/String;[I)J
    Java 方法：void f ();
    JNI 描述符：()V
    括号内是每个参数的类型符，括号外就是返回值的类型符。
    ```

---


### 参考文章
* [JNI基础：JNI数据类型和类型描述符](https://blog.csdn.net/afei__/article/details/80899758)

---

以上
