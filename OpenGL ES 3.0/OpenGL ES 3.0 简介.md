# OpenGL ES 3.0 简介 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### OpenGL ES 3.0学习汇总
* [OpenGL ES 3.0 学习记录汇总](https://www.jianshu.com/p/216c7e91960a)

---

### 简介
* OpenGL ES (OpenGL for Emberdded Systems)是以 **手持** 和 **嵌入式设备** 为目标的高级3D图形应用编程接口（API）。

* **Android 4.3 +** 和 **iPhone 5s +** 已经支持OpenGL ES 3.0。
* OpenGL ES 3.0 **向后兼容 OpenGL ES 2.0**。
* OpenGL ES 3.0 是从 **OpenGL 3.3规范** 衍生而来。

---

### OpenGL ES 3.0 图形管线
OpenGL ES 3.0实现了具有 **可编程着色功能** 的图形管线，由 **OpenGL ES 3.0 API** 和 **OpenGL ES 着色语言3.0规范** 组成。 
下图中深色背景的 **顶点着色器** 和 **片段着色器** 为可编程阶段。
![OpenGL ES 3.0 图形管线](https://upload-images.jianshu.io/upload_images/1709375-9ec27c71b7294d3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

#### 顶点着色器

顶点着色器 **实现了顶点操作的通用可编程方法**。  

顶点着色器的输入包括：
  * **着色器程序**一一描述顶点上执行操作的顶点着色器程序源代码或者可执行文件。
  * **顶点着色器输人**(或者属性)一一用顶点数组提供的每个顶点的数据。
  * **统一变量**(uniform)一一顶点(或者片段)着色器使用的不变数据。
  * **采样器**一一代表顶点着色器使用纹理的特殊统一变量类型。
    
 下图是顶点着色器的输入输出模型。
   ![OpenGL ES 3.0 顶点着色器 输入输出模型](https://upload-images.jianshu.io/upload_images/1709375-75dabf78031057d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

顶点着色器取得一个位置及相关的颜色数据作为输入属性，用一个 4x4矩阵变换位置，并输出变换后的位置和颜色。 

以下为顶点着色器示例：
```
#version 300 es   //提供着色语言的版本 必须出现在第一行
uniform mat4 u_mvpMatrix; # 统一变量 储存组合的模型视图和投影矩阵

in vec4 a_postion; # 顶点着色器输入的位置属性
in vec4 a_color;  # 顶点着色器输入的颜色属性

out vec4 v_color; # 顶点着色器输出的颜色属性
void main(){
  v_color = a_color; # 读取输入属性并写入输出属性上
  # 内建变量gl_Position 是自动声明的
  gl_Position = u_mvpMatrix * a_postion； # 变换后的位置写入gl_Positon输出
}
```
>`mat4` : 基本类型，  4 * 4浮点矩阵
`vec4` : 基本类型，4分量浮点向量
`uniform` : 储存限定符，在图元处理中值不改变，统一变量组成了着色器、OpenGL ES 和 应用程序的链接。
`in` : 参数限定符，用于传入函数的函数参数
`out` : 参数限定符，用于传出函数，但是传入时没有初始化的参数

---

#### 图元装配
OpenGL ES 3.0 图形管线，在 顶点着色器 之后就是 图元装配 了。
图元是 三角形、直线 或者 点精灵 等几何对象。
图元的每个顶点被发送到顶点着色器的不同拷贝，在图元装配期间，这些顶点被组合成图元。

对于每个图元，**会抛弃图元不在 视锥体(`屏幕可见的区域`)内的部分**，在视锥体内的区域的部分经过裁剪之后，将顶点位置转换为屏幕坐标。然后传递到管线的下一阶段 —— **光栅化阶段**。

---

#### 光栅化
光栅化阶段 会绘制对应的 图元。
**光栅化** 是将 **图元** 转化为 **二维片段** 的过程，然后这些片段再由 **片段着色器** 处理。这些二维片段代表可在屏幕上绘制的像素。

下图为光栅化流程:
![光栅化流程](https://upload-images.jianshu.io/upload_images/1709375-36a83e860fbaa825.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

#### 片段着色器
为 **片段上的操作** 实现了通用的可编程方法。

采用 如下输入 对每个光栅化阶段的片段执行这个着色器。    
  * **着色器程序**——描述片段上所执行操作的片段着色器程序源代码或者可执行文件。
  * **输入变量**——光栅化单元用插值为每个片段生成的顶点着色器输出。
  * **统一变量**——片段（或者顶点）着色器使用的不变数据。
  * **采样器**——代表片段着色器所用纹理的特殊统一变量类型。

  ![OpenGL ES 3.0 片段着色器](https://upload-images.jianshu.io/upload_images/1709375-22383bd014af2270.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 片段着色器可以抛弃片段，也可以生成一个或多个颜色值作为输出。
 片段着色器一般只输出一个颜色值，在 **渲染多重目标** 的时候会为每一个渲染目标输出一个颜色值。
 **光栅化阶段** 生成的 **颜色**、**深度**、**模板** 和 **屏幕坐标位置（X<sub>w</sub>,Y<sub>w</sub>）** 变成 OpenGL ES 3.0管线 **逐片段操作** 阶段的输入。

 片段着色器示例：
   ```
   #version 300 es  //提供着色语言的版本 
   precision mediump float;  # 默认的精度限定符

   in vec4 v_color; # 片段着色器的输入，即顶点着色器的输出
    
   out vec4 fragColor; # 片段着色器的输出变量， 即传递到逐片段操作的颜色
   void main()
   {
      fragColor = v_color; # 输出颜色设置为输入颜色  片段着色器的输入在图元之间进行线性插值 然后传递给片段着色器
   }
   ```
> **precision precision-qualifier type;**
`precision` : 可以用来确定默认精度修饰符, `type`可以是`int`或`float`或`采样器类型`，`precision-qualifier`可以是`lowp`, `mediump`, 或者`highp`。任何其他类型和修饰符都会引起错误。包括全局变量声明，函数返回值声明，函数参数声明，和本地变量声明等。没有声明精度修饰符的变量将使用和它最近的precision语句中的精度。
>
>`highp`:  精度和精度限定符，满足顶点语言的最低需求。
`lowp`: 精度和精度限定符，范围和精度低于`mediump`,但是任然能够表现所有颜色通道的所有颜色值。
`mediump` : 精度和精度限定符，范围和精度介于 `highp`和`lowp`之间。
>
>`FP = 浮点`

|| FP范围 | FP幅值范围 | FP精度 |整数范围 `有符号`|整数范围 `无符号`|
|:------:|:------:|:------:|:------:|:------:|:------:|
|`highp` | （2<sup>-126</sup>，2<sup>127</sup>）| 0.0, (2<sup>-126</sup>,2<sup>127</sup>)|2<sup>-24</sup>| （2<sup>-31</sup>，2<sup>31</sup>-1）|（0，2<sup>32</sup>-1）
| `mediump` | （2<sup>-14</sup>，2<sup>14</sup>） | (2<sup>-14</sup>,2<sup>14</sup>) |2<sup>-10</sup>|（2<sup>-15</sup>，2<sup>15</sup>-1）|（0，2<sup>16</sup>-1）
| `lowp` | （-2，2） | (2<sup>-8</sup>,2) | 2<sup>-8</sup>|（2<sup>-7</sup>，2<sup>7</sup>-1）|（0，2<sup>8</sup>-1）

---

#### 逐片段操作
  光栅化阶段生成的屏幕坐标为（X<sub>w</sub>,Y<sub>w</sub>）的片段只能修改 **帧缓冲区** 中位置为（X<sub>w</sub>,Y<sub>w</sub>）的像素。
  下图描述了OpenGL ES 3.0 逐片段操作阶段。
  ![OpenGL ES 3.0 逐片段操作阶段](https://upload-images.jianshu.io/upload_images/1709375-7dd8bdfacc07b410.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
* **像素归属测试**——确定帧缓区中的位置（X<sub>w</sub>,Y<sub>w</sub>）的像素是不是归OpenGL ES 所有。被遮挡的像素则不属于OpenGL ES 的上下文，从而不显示这些像素。**此过程在OpenGL ES 内部处理，不由开发人员控制**。
* **剪裁测试**—— 确定（X<sub>w</sub>,Y<sub>w</sub>）是否位于OpenGL ES 状态的一部分的裁剪矩形范围内，抛弃范围之外的片段。
* **模板测试、深度测试**—— 这些测试在输入片段的 **模板** 和 **深度值** 上进行 ，以确定片段是都该被拒绝。
* **混合**——将新生成的颜色和保存在帧缓冲区（X<sub>w</sub>,Y<sub>w</sub>）位置的颜色值组合起来。
* **抖动**——用于最小化 因为使用有限精度在帧缓冲区中保存颜色值而产生的伪像。

在逐片段操作阶段的最后，片段 被拒绝 或者   在帧缓冲区（X<sub>w</sub>,Y<sub>w</sub>）位置写入片段的颜色、深度或者模板值。
写入片段的颜色、深度或者模板值 取决于 **启用的相应写入掩码**。写入掩码可以更精细的控制写入的值。例如，`可以设置颜色缓冲区的写入掩码 使得 任何红色值都不被写入到颜色缓冲区`。

[原文链接:https://www.jianshu.com/p/1ecfc9a43552](https://www.jianshu.com/p/1ecfc9a43552)
