# HwLauncher Launcher dialog dismiss failed 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


报错信息
```
HwLauncher: Launcher dialog dismiss failed : java.lang.IllegalArgumentException: 
no dialog with id 1 was ever shown via Activity#showDialog
```

今天在 **华为 Mata7** (`EMUI 3.0 , Android 4.4.2`) 上  覆盖安装 应用后，启动闪退。

然后在 `logcat` 中 发现 上面的那个报错信息。

开始还以为是 我们的应用出了啥问题，后来发现 这个 日志  其实和我们的应用无关。

打印这个日志的原因是：**在 华为Mata7 上 开打手机的 **任务菜单**，有几个应用出来就会打印几条上面那个日志。**

闪退的原因是因为 从之前的未加密的数据库 替换成`sqlcipher`加密的数据库 导致的。

然后解决方法就是  **捕获到异常之后  删除 之前的数据库文件，再重新创建数据库就好了。**

以上 
