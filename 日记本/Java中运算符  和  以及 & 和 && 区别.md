# Java中运算符  和  以及 & 和 && 区别 

### Java中运算符 "|" 和 "||" 以及 "&" 和 "&&" 区别
* `|`运算符：**不论运算符左侧为true还是false，右侧语句都会进行判断**，如下代码：
    ```
    int a = 1, b = 1;
    if (a++ == 1 | ++b == 2) {
        System.out.println("true");
    }
    System.out.println("a= " + a + "  ,b=  " + b);
    ```
    左侧为`true`，右侧为`true`，输入出结果为：
    ```
    true
    a= 2  ,b=  2
    ```
---
* `||`运算符：**若运算符左边为true，则不再对运算符右侧进行运算**，如下代码：
    ```
    int a = 1, b = 1;
    if (a++ == 1 || ++b == 2) {
        System.out.println("true");
    }
    System.out.println("a= " + a + "  ,b=  " + b);
    ```
    左侧为`true`，所以没有判断运算符右侧语句，输出结果为：
    ```
    true
    a= 2  ,b=  1
    ```
---
* `&`运算符 与 `|`运算符 类似：**不论运算符左侧为true还是false，右侧语句都会进行判断**，如下代码：
    ```
    int a = 1, b = 1;
    if (a++ == 2 & ++b == 2) {
        System.out.println(true);
    } else {
        System.out.println(false);
    }
    System.out.println("a= " + a + "  ,b=  " + b);
    ```
    `&`运算符左侧为`false`，单依然会运行右侧语句输出为：
    ```
    false
    a= 2  ,b=  2
    ```
---
* `&&`运算符 与 `||`运算符 类似：**若运算符左侧为false则不再对右侧语句进行判断**，如下代码：
    ```
    int a = 1, b = 1;
    if (a++ == 2 && ++b == 2) {
        System.out.println(true);
    } else {
        System.out.println(false);
    }
    System.out.println("a= " + a + "  ,b=  " + b);
    ```
  输出结果：
    ```
    false
    a= 2  ,b=  1
    ```
### 参考
* [https://www.cnblogs.com/annofyf/p/9211925.html](https://www.cnblogs.com/annofyf/p/9211925.html)
