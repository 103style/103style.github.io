# AndroidStudio 找不到Genymotion设备
* AndroidStudio 版本： 3.3.2
* Genymotion 版本：3.0.1

笔者是因为使用了Android sdk 下面的 adb，然后运行的时候一直找不到genymotion设备。

然后，就在genymotion的setting界面 把 adb 配置为 genymotion默认的adb。

然后把genymotion安装目录下的tools文件夹路径添加到 系统的环境变量 Path 中，win+R → cmd 回车 然后运行下面的命令 
* adb kill-server
* adb start-server
* adb devices

就看到启动的genymotion模拟器了a
