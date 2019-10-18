# 以爬虫工程师身份谈谈Android端的信息安全 

转载请以链接形式标明出处：http://www.jianshu.com/p/29710cc77d8f

换了工作快二个月了，现在在当前公司爬虫+Android工程师..
很久没写blog了，今天根据这段时间的爬数据 经验来浅谈是如果获取需要的数据的！
希望各位看官阅后在必要的时候有所防范。
不废话了，开更...

手段无非就是抓包 以及 反编译。
抓包一般用Fiddler就可以了（主要是目前就会这个）。
反编译一般就是 apktool 以及 dex2jar。
具体看下面。
顺便说一句，一般有H5或者网页的，都很好得手，一般的网站[Java](http://lib.csdn.net/base/javaee)用Jsoup就好了，像某某招聘网站的数据，开128个线程，几百万数据几个小时就搞定了。
当然，有些网站会做一定的防范措施，那就得见招拆招了。

回到移动端,
先说Fiddler抓包（http,https），
首先就是在PC上下载Fiddle，[下载传送门](http://download.csdn.net/download/lxk_1993/9643642)
然后点击安装，一直点就行（笔者不喜欢安装软件在系统盘，所以一样的可以换下安装位置）。
然后双击运行，
点击左上角的Tools -- Telerik Fiddler Options 
然后如图设置
图一：
![](http://upload-images.jianshu.io/upload_images/1709375-98cf5ff5d15dc32c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
图二：
![](http://upload-images.jianshu.io/upload_images/1709375-6d2ae74af48a75ee?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
图三：
![](http://upload-images.jianshu.io/upload_images/1709375-4e7b38e14bde391c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后win+R ---- cmd --- 回车--ipconfig
图四：
![](http://upload-images.jianshu.io/upload_images/1709375-e0c0ea1ce6e213eb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
获取本机的IPv4地址 （ 例：11.11.11.11 ）。
然后 手机连上同一局域网的wifi，笔记本直接开wifi共享就OK.
然后再手机上点击已连接的wifi选择代理设置，具体手机操作不一。
例：
1.华为长按连接的wifi名  选择 修改网络  然后 在弹出来的界面上点击显示高级选项 ，再点击代理，选择手动，服务器主机名为图四 获得的IPv4地址，服务器端口为图三的Fiddler listens on port 的 值，然后在输下wifi密码 点击连接就好了。
2. 魅族点击连接的wifi名，在下一界面点击右上角的菜单栏（三个竖点），选择第二个选项，下一界面点击第一栏 选择手动，balabala.. 接下来就和华为的一样。

接下来 打开手机的浏览器，输入 刚刚设置的 地址 以及 端口号。
例： 11.11.11.11:8888
在加载后的界面 点击 FiddlerRoot certificate ，安装抓包证书，
有些手机可能是直接下载的，下载好之后（注意下载路径），进入手机的设置界面---高级选项  -- 安全  -- 从SD卡安装（不同的手机不一样），点击 选择 刚刚下载路径下的 证书文件 点击 就ok了，不想用了，直接点击 安全界面的删除凭据就好了。
配置基本就这样了，可以开始愉快的抓包了，有问题文末留言。
![](http://upload-images.jianshu.io/upload_images/1709375-5b16e35cba5f89b8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们大致需要了解的就是上面这些标志的信息了，
叉 表示删除以及清空左边那一大块的url.
Headers 和 Raw 是请求头 和 相应头 信息，有些时候我们需要把请求头 带上，要不是无法获取到数据的。
Webview 、json、xml 那些可以看返回的信息。
获取接口完整地址的话 就是 右击 左边那些地址中的一个地址，选择 copy -- just url，在右边的Raw中可以看到 请求地址 以及 请求方式。
使用大致就这样了，有问题 文末留言。
申明下，别用来做坏事，出事别带上我就行....
还不明白的话，也可以Google 或者Baidu 看看其他的教程。
反编译那些 也 自力更生吧，apktool 以及 dex2jar 就是2行命令 就ok了 ，相信读者的实力是可以完全掌握的。

说了这么多，来回归怎么做一些防范措施吧。
针对Fiddler这类，一般的防御方法有：
1.加密传输的内容，像某某聘 就是。
2.在注册为每个用户设置一个id 以及对应的 唯一识别码 来标识，像某某脉。
3.在第二点的基础上，用户退出时，使当前标识码失效 ，登录时获取新的标识码，像某某兔。
4.然后就是在后台监测用户的访问频率，太快的 直接 封掉，或者更流氓一点，返回一些 错误的信息给他，让采集的人开心的以为数据自己都抓到了。
当前上面这些措施都有相应的破解方法，但是还是能防止大部分人的。
其他的欢迎大家补充。

然后就是反编译这块：
想法是从之前看的一篇文章来的，[传送门在这里](http://mp.weixin.qq.com/s?__biz=MzA4MjEyNTA5Mw==&mid=2652564057&idx=1&sn=49d8c9e3dc211faf8a0f74531daa78e6&scene=22&srcid=0904fU4U16LSbTSIpE5V24L1#rd)
大致就是说，写的代码 要上除了自己或者团队之外的人都看不懂写的是什么。
比如登录界面，一般大家命名是LoginActivity，然后你就写EditActivity来进行语意混淆，或者WXActivity来让他以为是[微信](http://lib.csdn.net/base/wechat)这些第三方的界面，总之就是让名字表达的意思和实际的效果不一样。
再有就是接口地址，一般大家都是放到一个类似Http文件夹下的HttpUrl.java这种，里面就会有一个接口地址。类似11.11.11.11:8080/api这样一目了然。
试试这种
![](http://upload-images.jianshu.io/upload_images/1709375-a60d280453ed8111?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
或者在难点，就是事先加密。然后用的时候在解密。
今天先到这里，后面在更。
提钱祝大家国庆节玩的开心。
嘿嘿。

blog地址：http://blog.csdn.net/lxk_1993

转载请以链接形式标明出处：http://www.jianshu.com/p/29710cc77d8f
