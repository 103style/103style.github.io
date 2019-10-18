# 如何托管你的项目到github上详细教程 

本文将详细介绍如何托管你的项目到github上

文章博客地址：
[http://blog.csdn.net/lxk_1993/article/details/50441442](http://blog.csdn.net/lxk_1993/article/details/50441442)

一、首先你要有三个东西  github账号、上传工具msysgit 和 你的项目。
1.注册一个github账号
要托管到github，那你就应该要有一个属于你自己的github帐号，所以你应该先注册一个账号。
打开浏览器，
在地址栏输入地址：https://github.com（如下图），
填写用户名（name20151231）、
邮箱(1195644309@qq.com)、
密码("？？？？？？"),
点击Sign up for GitHub ,
跳转到下一界面，
拖动到页面最下面点击Finish Sign Up即可完成注册，
github也会给你注册用的邮箱发一份邮件让你认证邮箱。
点击Verify email address或者下面的链接即可认证成功。

**![](http://upload-images.jianshu.io/upload_images/1709375-9d9d99a10080b281?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**
****
****
2.github账号有了，
我们就开始安装上传工具msysgit了。
因为官方下载安装包比较慢，
下面分享一个windows的安装包。
msysgit2015.12.31最新安装包（windows）
百度云分享下载地址：[http://pan.baidu.com/s/1hrgTIdu](http://pan.baidu.com/s/1hrgTIdu)**
安装教程:[http://jingyan.baidu.com/article/e52e36154233ef40c70c5153.html](http://jingyan.baidu.com/article/e52e36154233ef40c70c5153.html)

安装的时候 最后会有两个教程上没有的直接选默认的，点next就好。
右击桌面显示下图所示表示安装成功（windows10）
![](http://upload-images.jianshu.io/upload_images/1709375-ce72082e07205c8a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 自己写好的一个项目。

二、开始上传项目到github.
1.首先进入github主页，登录你刚注册的账号。
点击New repository.

![](http://upload-images.jianshu.io/upload_images/1709375-ac122eead940c9ac?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.输入你的Repository name（仓库名），即可创建。
 Description和下面的可以不填。
**![](http://upload-images.jianshu.io/upload_images/1709375-fded700209230926?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**

3.点击途中红框的地址即可复制这个仓库的地址。也可以自己复制前面的地址。
**![](http://upload-images.jianshu.io/upload_images/1709375-caa2c3182953580c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**

4.来到你的项目的根目录。如图，笔者的项目名字是shareTest. **
**![](http://upload-images.jianshu.io/upload_images/1709375-40a5ef79144ffda0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5.然后鼠标右击界面空白地方，点击Git Bash Here 出现如图所示界面
![](http://upload-images.jianshu.io/upload_images/1709375-8293fe1d2a3e7bda?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. 输入git clone 加上 第三步 复制的地址，然后回车。
笔者输入的就是
git clone https://github.com/name20151231/MyProject.git

![](http://upload-images.jianshu.io/upload_images/1709375-4ad933a81732bdab?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

7.然后你的项目目录下会有一个新的文件夹(笔者的是MyProject,您的就是你刚刚创建的仓库名字)。如图

**![](http://upload-images.jianshu.io/upload_images/1709375-76890ed0665aacb0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**
****
****
8.然后选中除了这个文件夹（笔者的是MyProject,您的就是你刚刚创建的仓库名字）之外的文件，全部复制到 这个文件夹(MyProject)里面去。
**![](http://upload-images.jianshu.io/upload_images/1709375-d6889a92ed43c011?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**

9.然后输入 cd MyProject  回车进入该仓库的根目录目录。（笔者的是MyProject,您的就是你刚刚创建的仓库名字）**
**![](http://upload-images.jianshu.io/upload_images/1709375-32189a934c081e44?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

10.输入  git add .  (后面这个点要带上，add后面还有个空格)。
将这些文件添加到你本地的仓库。
**![](http://upload-images.jianshu.io/upload_images/1709375-c154d3a3647181e4?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**

11.输入 git  commit -m "logs"  
引号里面是你对本次提交的说明信息，
笔者写的信息是 logs。

12.最后输入 git push -u origin master   
然后会弹框提示你
输入你的用户名（就是刚刚注册github账号的名字，笔者的是 name20151231）,
点击ok,
又弹框提示你输入密码（你注册用户明对应的密码）。
再点击点击ok，
项目就上传完毕了，
打开github点击你刚刚创建的仓库 ，
就看到 项目都在里面了。
如图所示。
输入 exit 就可以退出msysgit.
**![](http://upload-images.jianshu.io/upload_images/1709375-5344d66bfcaaa979?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**
****
**![](http://upload-images.jianshu.io/upload_images/1709375-e9b0043d046134cd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**
****
**![](http://upload-images.jianshu.io/upload_images/1709375-7c3e4c9c81fd4500?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**
****
**![](http://upload-images.jianshu.io/upload_images/1709375-2f6aafc1ee988635?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)**


补一张具体的流程图
![git提交.png](http://upload-images.jianshu.io/upload_images/1709375-3ea2b5e768417269.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参考资料：
http://jingyan.baidu.com/article/f7ff0bfc7181492e27bb1360.html
