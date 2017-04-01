
---
layout: post
title: app:transformClassesWithDexForDebug'解决记录
date: 2017-03-15
categories: blog
tags: [Android,Error,app:transformClassesWithDexForDebug]
description: csdb链接http://blog.csdn.net/lxk_1993/article/details/50511172
---


3个错误

<font color="#ff4444">non-zero exit value 1 ; non-zero exit value 2 ; non-zero exit value 3</font> 



## non-zero exit value 1 

Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
\> com.Android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished with non-zero exit value 1

这个是因为依赖包重复了 （像v4和nineoldandroids），如图。app中实现了对easeUI的依赖，但是app和easeUI都添加了对这个包的依赖。所以就报这个错误，修改之后再报，就clean，rebuild一下

![](http://img.blog.csdn.net/20160113155838927?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](http://img.blog.csdn.net/20160113155844869?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## non-zero exit value 2

Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
\> com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished withnon-zero exit value 2


这个错误在app的build.gradle里面添加下面这句就好了。

	android {
		   
		defaultConfig {
		
			...	
			multiDexEnabled true
				
		}

	}



## non-zero exit value 3

Error:Execution failed for task ':app:transformClassesWithDexForDebug'.
\> com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'F:\Program Files (x86)\Java\jdk1.8.0_31\bin\java.exe'' finished withnon-zero exit value 3


这个错误就在app.bulid里面加上这句，再rebuild ,之后再运行就行了。4g可以看电脑配置修改（2g,3g,6g,8g）。


	dexOptions {

		javaMaxHeapSize "4g"
			
	}
		
如图：

![](http://img.blog.csdn.net/20160114140024246?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

