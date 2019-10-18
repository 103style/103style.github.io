# 自定义View实战（二）QQ健康水滴形加载 

废话不说  先上效果图。
![](http://upload-images.jianshu.io/upload_images/1709375-4df992f382f5a3e0?imageMogr2/auto-orient/strip)
看起来是不是比起那些普通的加载“高大上”一点。
怎么去做了，很简单，真的！
一起来看看怎么实现的吧。
实现思路：
1.首先我们仔细看看这效果图的灰色背景， 你就会说，什么水滴形，不就是个圆和三角形吗!
对嘛，你看，这不就简单了吗，绘制一个实心的圆和三角形。
i)设置圆心、半径和三角形三个点的坐标。
代码如下：

        //设备宽高和dpi密度
        DisplayMetrics displayMetrics = getResources().getDisplayMetrics();
        sWidth = displayMetrics.widthPixels;
        sHeight = displayMetrics.heightPixels;
        //320为我的测试机dpi密度，以次绘制视图
        sDensityDpi = displayMetrics.densityDpi / 320;
        //圆心坐标赋值
        pointerX = pointerY = Math.min(sWidth, sHeight) / 2;
        //半径和三角形边长赋值
        mRaduis = pointerX / 5;
        //赋值顶点坐标
        mTriangleX1 = pointerX;
        mTriangleY1 = (float) (pointerY - 1.5 * mRaduis * Math.sin(Math.PI / 3));
        mTriangleX2 = (float) (pointerX - mRaduis * Math.cos(Math.PI / 3));
        mTriangleX3 = (float) (pointerX + mRaduis * Math.cos(Math.PI / 3));
        mTriangleY2 = mTriangleY3 = (float) (pointerY - mRaduis * Math.sin(Math.PI / 3));
ii)然后开始画圆和三角形，三角形用Path这个类。
代码如下：

        //设置画笔颜色和样式
        mPaint.setColor(0xFFDEE0DD);
        mPaint.setStyle(Paint.Style.FILL);
        //绘制圆
        canvas.drawCircle(pointerX, pointerY, mRaduis, mPaint);
        //绘制顶部三角形
        mPath.moveTo(mTriangleX1, mTriangleY1);
        mPath.lineTo(mTriangleX2, mTriangleY2);
        mPath.lineTo(mTriangleX3, mTriangleY3);
        //lineto起点
        mPath.close();
        canvas.drawPath(mPath, mPaint);

这样这个盗版的水滴形状的背景就绘制出来了。
2.然后就是中间那些蓝色的东西，仔细看看，是不是感觉像一个越来越大的实心弧形，最后那里就是一个小三角形。
画弧，就是上一个[汽车仪表盘](http://blog.csdn.net/lxk_1993/article/details/51373269)里面的速度区域的扇形一样，只是去掉了到圆心的一部分。
i）我们先确定这个弧形的外切圆，其实就是圆形背景的外切圆 缩小了一点。
也就是左上和右下点的坐标调整了一下。
代码如下：

        // 初始化速度范围的2个扇形外切矩形
        progressRectF = new RectF(
                pointerX - mRaduis + 8 * sDensityDpi, 
                pointerY - mRaduis + 8 * sDensityDpi,
                pointerX + mRaduis - 8 * sDensityDpi,
                pointerY + mRaduis - 8 * sDensityDpi
        );
ii）然后我们看看这个弧形的起始角度，它是从底部开始的，所以开始是90度。
然后慢慢的减小，最后多出一个小三角形。画完之后，再还原 相关参数。
代码如下：

        //修改画笔颜色
        mPaint.setColor(0xFF13B5E8);
        startAngle -= 5;
        sweepAngle += 10;
        if (sweepAngle < 310) {
            canvas.drawArc(progressRectF, startAngle, sweepAngle, false, mPaint);
            postInvalidateDelayed(100);
        } else {
            canvas.drawArc(progressRectF, -90, 360, false, mPaint);
            trianglePath1.moveTo(mTriangleX1, (float) (mTriangleY1 + mRaduis * Math.sin(Math.PI / 3) /3 ));
            trianglePath1.lineTo(mTriangleX2 + 2 * (mRaduis / 2 - triangleR / 2), mTriangleY2 + mRaduis / 2 - triangleR / 2);
            trianglePath1.lineTo(mTriangleX3 - 2 * (mRaduis / 2 - triangleR / 2), mTriangleY3 + mRaduis / 2 - triangleR / 2);
            trianglePath1.close();
            canvas.drawPath(trianglePath1, mPaint);
            startAngle = 90;
            sweepAngle = 0;
            triangleHeight = 0;
            postInvalidateDelayed(500);
        }
看，效果图就出来了。

是不是很简单？
博客地址：http://blog.csdn.net/lxk_1993
如果你喜欢我的博客，请关注我。
欢迎留言拍砖。

源码地址:
github：[https://github.com/103style/QQLoading-WaterDrop](https://github.com/103style/QQLoading-WaterDrop) 觉得不错的话，点下star,谢谢
csdn下载：[http://download.csdn.net/download/lxk_1993/9521444](http://download.csdn.net/download/lxk_1993/9521444)
