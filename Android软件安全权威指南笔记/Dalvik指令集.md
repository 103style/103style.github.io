# Dalvik指令集 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

对于 `Android 4.4` 之前的系统， 可以在 Android 源码 `davik/libdex/DexOpcodes.h`中找到完整的Dalvik指令集。
对于 `Android 4.4` 及之后的以 `ART` 主导的系统， 可以在 Android 源码 `art/runtime/dexinstuctionlist.h`中找到完整的Dalvik指令集。

#### 空操作指令

`nop` ：值为`00`

通常用于对齐代码，不进行实际操作

---

#### 数据操作指令

数据操作指令为`move`，原型为 `move destination, source`

| 指令 | 作用 |
|:-:|:-:|
| `move vA, vB` | 将`4位` `vB` 寄存器的 值 赋值给`4位` `vA` 寄存器 |
| `move/from16 vAA, vBBBB` | 将`16位` `vBBBB` 寄存器的 值 赋值给`8位` `vAA` 寄存器 |
| `move/16 vAAAA, vBBBB` | 将`16位` `vBBBB` 寄存器的 值 赋值给`16位` `vAAAA` 寄存器 |
|||
| `move-wide vA, vB` | 将`4位` `vB` 寄存器的 值 赋值给`4位` `vA` 寄存器 |
| `move-wide/from16 vAA, vBBBB` | 同`move/from16 vAA, vBBBB` |
|||
| `move-object vA, vB` | 将`4位` `vB` 寄存器的 对象 赋值给`4位` `vA` 寄存器 |
| `move-object/from16 vAA, vBBBB` | 将`16位` `vBBBB` 寄存器的 对象 赋值给`8位` `vAA` 寄存器 |
| `move-object/16 vAAAA, vBBBB` | 将`16位` `vBBBB` 寄存器的 对象 赋值给`16位` `vAAAA` 寄存器 |
|||
| `move-result vAA` | 用于将上一个`invoke`类型指令操作的 单字非对象结果 赋予 `vAA` 寄存器 |
| `move-result-wide vAA` | 用于将上一个`invoke`类型指令操作的 双字非对象结果 赋予 `vAA` 寄存器 |
| `move-result-object vAA` | 用于将上一个`invoke`类型指令操作的 对象结果 赋予 `vAA` 寄存器 |
| `move-exception vAA` | 用与将上一个在运行时发生的异常保存到 `vAA` 寄存器，**必须在异常发生时由异常处理器使用** |

---

#### 返回指令

返回指令：函数结束时运行的最后一条指令，基础字节码为`return`

| 指令 | 作用 |
|:-:|:-:|
| `return-void` | 函数从一个`void`方法返回 |
| `return vAA` | 函数返回一个`32位`非对象类型的值，返回值为`8位`寄存器 `vAA` |
| `return-wide vAA` | 函数返回一个`64位`非对象类型的值，返回值为`8位`寄存器 `vAA`  |
| `return-object vAA` | 函数返回一个对象类型的值，返回值为`8位`寄存器 `vAA`  |

---

#### 数据定义指令

| 指令 | 作用 |
|:-:|:-:|
| `const/4 vA, #+B` | 将数值符号扩展为`32位`后赋予寄存器`vA` |
| `const/16  vAA, #+BBBB` | 将数值符号扩展为`32位`后赋予寄存器`vAA` |
| `const vAA, #+BBBBBBBB` | 将数值赋予寄存器`vAA` |
| `const/high16 vAA, #+BBBB0000` | 将数值`右边的0`扩展为`32位`后赋予寄存器`vAA` |
|||
| `const-wide/16  vAA, #+BBBB` | 将数值符号扩展为`64位`后赋予寄存器`vAA` |
| `const-wide/32  vAA, #+BBBBBBBB` | 将数值符号扩展为`64位`后赋予寄存器`vAA` |
| `const-wide vAA, #+BBBBBBBBBBBBBBBB` | 将数值符号赋予寄存器`vAA` |
| `const-wide/high16  vAA, #+BBBB000000000000` | 将数值`右边的0`扩展为`64位`后赋予寄存器`vAA` |
|||
| `const-string vAA, string@BBBB` | 通过字符串索引构造一个字符串，并将其赋予寄存器`vAA` |
| `const-string/jumbo vAA, string@BBBBBBBB` | 通过字符串索引（较大）构造一个字符串，并将其赋予寄存器`vAA` |
|||
| `const-class vAA, type@BBBB` | 通过类型索引获取一个类的引用，并将其赋予寄存器`vAA` |
| `const-class vAAAA, type@BBBBBBBB` | 通过给定类型索引获取一个类的引用，并将其赋予寄存器`vAAAA`，此指令占2字节，值为`0x00ff` |


---

#### 锁指令

| 指令 | 作用 |
|:-:|:-:|
| `monitor-enter vAA` | 为指定对象获取锁 |
| `monitor-exit vAA` | 为指定对象释放锁 |

---

#### 实例操作指令

包括 **类型转换**、**检查** 以及 **创建** 等。

| 指令 | 作用 |
|:-:|:-:|
| `check-cast vAA, type@BBBB` | 将 `vAA` 寄存器中的对象引用转换成指定的类型 |
| `instance-of vA, vB, type@CCCC` | 判断 `vB` 寄存器中的对象引用是否可以转换成指定的类型， 可以 `vA` 寄存器中的值为 `1`， 否则 为 `0` |
| `new-instance vAA, type@BBBB` | 构造一个指定对象的新实例，并赋值给 `vAA` 寄存器，`type`不能时数组类 |
| `check-cast/jumbo vAAAA, type@BBBBBBBB` | 与`check-cast vAA, type@BBBB`类似，只是取值范围更大 |
| `instance-of/jumbo vAAAA, vBBBB, type@CCCCCCC` | 与`instance-of vA, vB, type@CCCC`类似，只是取值范围更大 |
| `new-instance/jumbo vAAAA, type@BBBBBBBB` | 与`new-instance vAA, type@BBBB`类似，只是取值范围更大 |

---

#### 数组操作指令

包括 **获取数组长度**、**新建数组**、**数组赋值**、**数组元素取值和赋值** 等

| 指令 | 作用 |
|:-:|:-:|
| `array-length vA, vB` | 获取 `vB` 寄存器中的数组长度 赋值给 `vA` 寄存器 |
| `new-array vA, vB， type@CCCC` | 构建指定类型(`type@CCCC`)和大小(`vB`)的数组赋值给 `vA` 寄存器 |
| `filled-new-array{vC, vD, vE, vF, vG}, type@BBBB` | 构建指定类型(`type@BBBB`)和大小(`vA`)的数组并填充数组内容。 |
| `filled-new-array/range{vCCCC ...  vNNNN}, type@BBBB` | 与`filled-new-array`类似，只是用 `range` 来指定取值范围 |
| `fill-array-data vAA, +BBBBBBBB` | 用指定的数据类填充数组 |
| `new-array/jumbo vAAAA, vBBBB, type@CCCCCCCC` | 与`new-array`类似，只是取值范围更大 |
| `filled-new-array/jumbo{vCCCC ...  vNNNN}, type@BBBB` | 与`filled-new-array`类似，只是取值范围更大 |
| `arrayop vAA, vBB, vCC` | 对`vBB`寄存器指定的数组元素进行取值和赋值；`vCC`寄存器用于指定数组元素的索引； `vAA`寄存器用于存放读取获取或需要设置的数组元素的值 |



---

#### 异常指令

| 指令 | 作用 |
|:-:|:-:|
| `throw vAA` | 抛出`vAA`寄存器中指定类型的异常 |

---


#### 跳转指令

指从当前地址跳转到指定的偏移出。分为以下三类：
* 无条件跳转指定 `goto`
* 分支跳转指令 `switch`
* 条件跳转指令 `if`

| 指令 | 作用 |
|:-:|:-:|
| `goto +AA` | 无条件跳转到指定偏移处，偏移量 `AA` 不能为0 |
| `goto/16 +AAAA` | 无条件跳转到指定偏移处，偏移量 `AAAA` 不能为0  |
| `goto/32 +AAAAAAAA` | 无条件跳转到指定偏移处  |
| `packed-swtich vAA, +BBBBBBBB` | `vAA` 寄存器为 `swtich` 分支中需要判断的值， `BBBBBBBB`指向一个`packed-swtich-payload`格式的偏移表，表中的值时 **递增** 的偏移量 |
| `sparse-swtich vAA, +BBBBBBBB` | `vAA` 寄存器为 `swtich` 分支中需要判断的值， `BBBBBBBB`指向一个`packed-swtich-payload`格式的偏移表，表中的值时 **无规律** 的偏移量 |
| `if-test vA, vB, +CCCC` | 条件跳转指令用于比较 `vA` 和 `vB` 寄存器的值 |
| `if-testz vAA, +BBBB` | 将 `vAA` 寄存器中的值与 `0` 比较，如果满足或者值为 `0`，则跳转到 `BBBB` 偏移处，`BBBB` 不能为`0` |

`if-test` 指令如下：

| 指令 | 作用 |
|:-:|:-:|
| `if-eq` | 如果 `vA` 等于 `vB` 则跳转，`Java`语法表示为`if(vA==vB)` |
| `if-ne` | 如果 `vA` 不等于 `vB` 则跳转，`Java`语法表示为`if(vA!=vB)` |
| `if-lt` | 如果 `vA` 小于 `vB` 则跳转，`Java`语法表示为`if(vA<vB)` |
| `if-ge` | 如果 `vA` 大于等于 `vB` 则跳转，`Java`语法表示为`if(vA>=vB)` |
| `if-gt` | 如果 `vA` 大于 `vB` 则跳转，`Java`语法表示为`if(vA>vB)` |
| `if-le` | 如果 `vA` 小于等于 `vB` 则跳转，`Java`语法表示为`if(vA<=vB)` |

`if-testz` 指令如下：

| 指令 | 作用 |
|:-:|:-:|
| `if-eqz` | 如果 `vAA` 为 `0` 则跳转，`Java`语法表示为`if(!vAA)` |
| `if-nez` | 如果 `vAA` 不为 `0` 则跳转，`Java`语法表示为`if(vAA)` |
| `if-ltz` | 如果 `vAA` 小于 `0` 则跳转，`Java`语法表示为`if(vAA<0)` |
| `if-gez` | 如果 `vAA` 大于等于 `0` 则跳转，`Java`语法表示为`if(vAA>=0)` |
| `if-gtz` | 如果 `vAA` 大于 `0` 则跳转，`Java`语法表示为`if(vAA>0)` |
| `if-lez` | 如果 `vAA` 小于等于 `0` 则跳转，`Java`语法表示为`if(vAA<=0)` |

---

#### 比较指令

对两个寄存器的值进行比较，格式 `cmpkid vAA, vBB, vCC`，`vAA` 表示 `vBB` 和 `vCC` 比较的结果。

| 指令 | 作用 |
|:-:|:-:|
| `cmpl-float vAA, vBB, vCC` | 用于比较两个单精度浮点数 (`vAA=0` : `vBB=vCC`；`vAA=-1` : `vBB>vCC`；`vAA=1` : `vBB<vCC`) |
| `cmpg-float vAA, vBB, vCC` | 用于比较两个单精度浮点数(`vAA=0` : `vBB=vCC`；`vAA=1` : `vBB>vCC`；`vAA=-1` : `vBB<vCC`) |
| `cmpl-double vAA, vBB, vCC` | 用于比较两个双精度浮点数(`vAA=0` : `vBB=vCC`；`vAA=-1` : `vBB>vCC`；`vAA=1` : `vBB<vCC`) |
| `cmpg-double vAA, vBB, vCC` | 用于比较两个双精度浮点数(`vAA=0` : `vBB=vCC`；`vAA=1` : `vBB>vCC`；`vAA=-1` : `vBB<vCC`)  |
| `cmp-long vAA, vBB, vCC` | 用于比较两个长整型数(`vAA=0` : `vBB=vCC`；`vAA=1` : `vBB>vCC`；`vAA=-1` : `vBB<vCC`)  |

---

#### 字段操作指令

用于对对象实例的字段进行读写操作。


有以下两种指令集：
* `iinstanceop vA, vB, field@CCCC` : 操作普通字段，以`i`开头 -- `iget`读，`iput`写
* `sstaticop vAA, field@CCCC` : 操作静态字段，以`s`开头 -- `sget`读，`sput`写

| 指令 | 
|:-:|
| `iget` 、`sget` 、`iput` 、`sput` | 
| `iget-wide` 、`sget-wide` 、`iput-` 、`sput-` | 
| `iget-object` 、`sget-object` 、`iput-object` 、`sput-object` | 
| `iget-boolean` 、`sget-boolean` 、`iput-boolean` 、`sput-boolean` | 
| `iget-byte` 、`sget-byte` 、`iput-byte` 、`sput-byte` | 
| `iget-char` 、`sget-char` 、`iput-char` 、`sput-char` | 
| `iget-short` 、`sget-short` 、`iput-short` 、`sput-short` | 

>在 **Android 4.0** 中， `Dalvik`指令集增加了以下两类指令：
>* `iinstanceop/jumbo vAAAA, vBBBB, field@CCCCCCCC` 
>* `sstaicop/jumbo vAAAA, field@BBBBBBBB` 
>
>和上面两类类似，只是增加了 `jimbo`后缀，且寄存器的指令索引的取值范围更大。

---

#### 方法调用指令

方法调用指令 负责 调用类实例的方法， 基础指令位 `invoke`。

可分为以下两类：
* `invoke-kind {vC, vD, vE, vF, vG}, meth#BBBB`
* `invoke-kind/range {vCCC ... vNNNN}, meth#BBBB`
  相比第一类，只是用`range`来指定寄存器的范围

| 指令 | 作用 |
|:-:|:-:|
| `invoke-virtual` or `invoke-virtual/range` | 调用实例的虚方法 |
| `invoke-super` or `invoke-super/range` | 调用实例的父类方法 |
| `invoke-direct` or `invoke-direct/range` | 调用实例的直接方法 |
| `invoke-static` or `invoke-static/range` | 调用实例的静态方法 |
| `invoke-interface` or `invoke-interface/range` | 调用实例的接口方法 |

---

#### 数据转换指令

| 指令 | 作用 |
|:-:|:-:|
| `neg-int` | 用于对 整数类型 求 补 |
| `not-int` | 用于对 整数类型 求 反 |
|||
| `neg-long` | 用于对 长整型 求 补 |
| `not-long` | 用于对 长整型 求 反 |
|||
| `neg-float` | 用于对 单精度浮点型 求 补 |
| `neg-double` | 用于对 双精度浮点型 求 补 |
|||
| `int-to-float` | 用于 整型 转换为 单精度浮点型 |
| `A-to-B` |  同上，用于 `A( int、long、float、double )` 转换为 `B( int、long、float、double ) ` |
| `int-to-byte` | 用于  整型 转换为 字节型 |
| `int-to-char` | 用于  整型 转换为 字符型 |
| `int-to-short` | 用于  整型 转换为 短整型 |

---

#### 数据运算指令

**数据运算指令** 包含 **算术运算指令** 与 **逻辑运算指令**。

可以分成以下四类：

| 指令类别 | 作用 |
|:-:|:-:|
| `binop vAA, vBB, vCC` | 将 `vBB` 和 `vCC` 寄存器进行运算，将运算结果保存在 `vAA` 中 |
| `binop/2addr vA, vB` | 将 `vA` 和 `vB` 寄存器进行运算，将运算结果保存在 `vA` 中 |
| `binop/lit16 vA, vB, #+CCCC` | 将 `vB` 寄存器和 `CCCC` 常量进行运算，将运算结果保存在 `vA` 中 |
| `binop/lit8 vAA, vBB, #+CC` | 将 `vBB` 寄存器和 `CC` 常量进行运算，将运算结果保存在 `vAA` 中  |

第一类`binop vAA, vBB, vCC`指令可归类如下：

| 指令 | 作用 |
|:-:|:-:|
| `add-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 加法运算 `vBB + vCC` |
| `sub-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 减法运算 `vBB - vCC` |
| `mul-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 乘法运算 `vBB * vCC` |
| `div-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 除法运算 `vBB / vCC` |
| `rem-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 模运算 `vBB % vCC` |
| | |
| `and-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 与运算 `vBB AND vCC`  |
| `or-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 或运算 `vBB OR vCC`  |
| `xor-type` | 将 `vBB` 和 `vCC` 寄存器的值进行 异或运算 `vBB XOR vCC`  |
| | |
| `shl-type` | 将 `vBB` 寄存器中的值（有符号数）左移 `vCC` 位 `vBB<<vCC`  |
| `shr-type` | 将 `vBB` 寄存器中的值（有符号数）右移 `vCC` 位 `vBB>>vCC` |
| `ushr-type` | 将 `vBB` 寄存器中的值（无符号数）右移 `vCC` 位 `vBB>>vCC` |


---

以上
