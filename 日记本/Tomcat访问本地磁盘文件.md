# Tomcat访问本地磁盘文件 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

`需先配置Java开发环境`

### 目录
* 下载`Tomcat`
* 配置`Tomcat`
* 添加虚拟路径配置访问本地磁盘文件

---

### 下载Tomcat
首先下载Tomcat，[下载地址](http://tomcat.apache.org/download-90.cgi)，点击下载`zip`包。

然后解压到某一目录`(e.g.  D:\apache-tomcat-9.0.14\)`.

---

### 配置Tomcat
* 按 `win`键，输入 `env`，点击 **编辑系统环境变量**，点击 **环境变量**
  * 新建 **变量名** ：`CATALINA_HOME`，**变量值** ：为`tomcat`的目录`(e.g.  D:\apache-tomcat-9.0.14\)`.
  * 将`Tomcat`目录下的`bin`目录添加到`path`里`(e.g. D:\apache-tomcat-9.0.14\bin)`
  * 将`Tomcat`目录下的`lib`目录添加到`path`里`(e.g. D:\apache-tomcat-9.0.14\lib)`

* 按 `win + R `输入`cmd`，进入到`cmd 命令行`界面， 切换到`tomcat`的`bin`目录.`e.g.`:
  * `D:`
  * `cd apache-tomcat-9.0.14`
  * `cd bin`
  * `service.bat install`
  ![安装Tomcat服务](https://upload-images.jianshu.io/upload_images/1709375-3fcf980f24023df1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 按 `win`键，输入 **服务**，点击 **服务** 进入服务界面，在右侧找到 `Apache Tomcat 9.0 Tomcat9`服务，**右键** 选择 **属性**， 将 **启动类型** 设置为 **自动**，点击 **启动**，然后确定。
  ![启动Tomcat服务](https://upload-images.jianshu.io/upload_images/1709375-7dcc8e029ae63dd5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 打开浏览器，输入 `http://localhost:8080/`，检查是否可以访问。

---

### 添加虚拟路径配置访问本地磁盘文件

假如要访问的文件在 `D:\apache-tomcat-9.0.14\webapps` 这个文件夹中。

* 打开 `tomcat/conf/server.xml` 配置文件，在`<host></host>`之间加入下面代码：
  ```
  <!-- path 表示访问路径  浏览器的访问路径为 http://localhost:8080/webapps/ -->
  <!-- docBase 表示访问路径对应的本地磁盘路径 -->
  <!-- Debug 则是设定debug level,  0表示提供最少的信息，9表示提供最多的信息 -->
  <Context path="/webapps" docBase="D:\apache-tomcat-9.0.14\webapps" debug="0" reloadable="true" />
  ```
* 打开浏览器，输入配置的访问路径，`(e.g. http://localhost:8080/webapps/)`，检查是否能访问。

