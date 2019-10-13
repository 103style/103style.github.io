*  先来到VirtualBox的下载位置，如图，笔者位置在D:/vb文件夹下

![下载目录](http://upload-images.jianshu.io/upload_images/1709375-fb4adb6c4e47d3ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  然后按住win+R(win就是左下角ctrl和alt之间那个键)，输入cmd，然后回车

*  如果在C盘的话，就直接cd 文件目录（右击下载的文件，选择最下面的属性可以看到位置），需要切换盘的话先 盘符加冒号回车（例  D:），然后再cd 文件目录（例 cd D:\vb）

*  然后复制下载文件的名字，笔者的是（VirtualBox-5.1.18-114002-Win.exe），如果文件名 没有.exe后缀，在cmd上输入的时候要加上.exe, 然后在cmd命令行输入  带.exe的文件名 -extract - path 导出的文件目录，笔者导出文件目录直接为当前目录  (例 VirtualBox-5.1.18-114002-Win.exe -extract -path D:\vb),如图：
![导出msi文件](http://upload-images.jianshu.io/upload_images/1709375-cefaf60c8b589bab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  然后就会发现导出的目录出现如图的文件，然后根据你的电脑位数来安装，amd64表示64位的，x86表示32位的，
![文件](http://upload-images.jianshu.io/upload_images/1709375-cb658c5fa4c91bdf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  然后鼠标右击左下角的windows图片，选择命令提示符（管理员）  
![管理员权限运行cmd命令](http://upload-images.jianshu.io/upload_images/1709375-11c928d196d1af7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  然后输入msiexec  /package 刚刚提取文件的目录，加上对应位数的msi文件，然后回车，就可以安装了，并不会有2502 2503的错误了，安装好后，重启就ok了
   例子msiexec /package D:\vb\ VirtualBox-5.1.18-r114002-MultiArch_amd64.msi 
   如图：
![安装](http://upload-images.jianshu.io/upload_images/1709375-ec5b8c259720d0df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
