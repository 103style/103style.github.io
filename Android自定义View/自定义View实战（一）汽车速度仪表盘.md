# 自定义View实战（一）汽车速度仪表盘 

博客地址：http://blog.csdn.net/lxk_1993/article/details/51373269
废话不说  先上效果图。
![](http://upload-images.jianshu.io/upload_images/1709375-bb90cb8d0e8f1c3e?imageMogr2/auto-orient/strip)

是不是很酷炫.
看起来觉得很难？  不难 ， 其实实现起来很容易。
思路：

1.绘制一个实心的圆做仪表盘背景。 

        mPaint.setStyle(Paint.Style.FILL);
        mPaint.setColor(0xFF343434);
        canvas.drawCircle(pointX, pointY, raduis, mPaint);

2.绘制外面的两个圆环 和 里面的 两个圆环。

        //外圈2个圆
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setColor(0xBF3F6AB5);
        mPaint.setStrokeWidth(4 * mDensityDpi);
        canvas.drawCircle(pointX, pointY, raduis, mPaint);
        mPaint.setStrokeWidth(3 * mDensityDpi);
        canvas.drawCircle(pointX, pointY, raduis - 10 * mDensityDpi, mPaint);

        //内圈2个圆
        mPaint.setStrokeWidth(5 * mDensityDpi);
        mPaint.setColor(0xE73F51B5);
        canvas.drawCircle(pointX, pointY, raduis / 2, mPaint);
        mPaint.setColor(0x7E3F51B5);
        canvas.drawCircle(pointX, pointY, raduis / 2 + 5 * mDensityDpi, mPaint);
        mPaint.setStrokeWidth(3 * mDensityDpi);

3.绘制仪表盘的刻度。

    /**
     * 绘制刻度
     */
    private void drawScale(Canvas canvas) {
        for (int i = 0; i < 60; i++) {
            if (i % 6 == 0) {
                canvas.drawLine(pointX - raduis + 10 * mDensityDpi, pointY, pointX - raduis + 50 * mDensityDpi, pointY, mPaint);
            } else {
                canvas.drawLine(pointX - raduis + 10 * mDensityDpi, pointY, pointX - raduis + 30 * mDensityDpi, pointY, mPaint);
            }
            canvas.rotate(6, pointX, pointY);
        }
    }

4.绘制仪表盘的速度标识和中间的速度 和 单位 文字。（这里有好的处理方法请留言）

    /**
     * 绘制速度标识文字
     */
    private void drawText(Canvas canvas, int value) {
        String TEXT = String.valueOf(value);
        switch (value) {
            case 0:
                // 计算Baseline绘制的起点X轴坐标
                baseX = (int) (pointX - sRaduis * Math.cos(Math.PI / 5) + textPaint.measureText(TEXT) / 2 + textScale / 2);
                // 计算Baseline绘制的Y坐标
                baseY = (int) (pointY + sRaduis * Math.sin(Math.PI / 5) + textScale / 2);
                break;
            case 30:
                baseX = (int) (pointX - raduis + 50 * mDensityDpi + textPaint.measureText(TEXT) / 2);
                baseY = (int) (pointY + textScale);
                break;
            case 60:
                baseX = (int) (pointX - sRaduis * Math.cos(Math.PI / 5) + textScale);
                baseY = (int) (pointY - sRaduis * Math.sin(Math.PI / 5) + textScale * 2);
                break;
            case 90:
                baseX = (int) (pointX - sRaduis * Math.cos(2 * Math.PI / 5) - textScale / 2);
                baseY = (int) (pointY - sRaduis * Math.sin(2 * Math.PI / 5) + 2 * textScale);
                break;
            case 120:
                baseX = (int) (pointX + sRaduis * Math.sin(Math.PI / 10) - textPaint.measureText(TEXT) / 2);
                baseY = (int) (pointY - sRaduis * Math.cos(Math.PI / 10) + 2 * textScale);
                break;
            case 150:
                baseX = (int) (pointX + sRaduis * Math.cos(Math.PI / 5) - textPaint.measureText(TEXT) - textScale / 2);
                baseY = (int) (pointY - sRaduis * Math.sin(Math.PI / 5) + textScale * 2);
                break;
            case 180:
                baseX = (int) (pointX + sRaduis - textPaint.measureText(TEXT) - textScale / 2);
                baseY = (int) (pointY + textScale);
                break;
            case 210:
                baseX = (int) (pointX + sRaduis * Math.cos(Math.PI / 5) - textPaint.measureText(TEXT) - textScale / 2);
                baseY = (int) (pointY + sRaduis * Math.sin(Math.PI / 5) - textScale / 2);
                break;

        }
        canvas.drawText(TEXT, baseX, baseY, textPaint);
    }

    /**
     * 绘制中间文字内容
     */
    private void drawCenter(Canvas canvas) {
        //速度
        textPaint.setTextSize(60 * mDensityDpi);
        float tw = textPaint.measureText(String.valueOf(speed));
        baseX = (int) (pointX - tw / 2);
        baseY = (int) (pointY + Math.abs(textPaint.descent() + textPaint.ascent()) / 4);
        canvas.drawText(String.valueOf(speed), baseX, baseY, textPaint);

        //单位
        textPaint.setTextSize(20 * mDensityDpi);
        tw = textPaint.measureText("km/h");
        baseX = (int) (pointX - tw / 2);
        baseY = (int) (pointY + raduis / 4 + Math.abs(textPaint.descent() + textPaint.ascent()) / 4);
        canvas.drawText("km/h", baseX, baseY, textPaint);
    }

5.绘制速度范围的扇形区域。

    /**
     * 绘制速度区域扇形
     */
    private void drawSpeedArea(Canvas canvas) {
        int degree;
        if (speed < 210) {
            degree = speed * 36 / 30;
        } else {
            degree = 210 * 36 / 30;
        }

        canvas.drawArc(speedRectF, 144, degree, true, speedAreaPaint);

        // TODO: 2016/5/12
        //不显示中间的内圈的扇形区域
        mPaint.setColor(0xFF343434);
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawArc(speedRectFInner, 144, degree, true, mPaint);
        mPaint.setStyle(Paint.Style.STROKE);


    }

6.实现点击让 速度动起来。实现runnable 接口。

    @Override
    public void run() {
        int speedChange;
        while (start) {
            switch (type) {
                case 1://油门
                    speedChange = 3;
                    break;
                case 2://刹车
                    speedChange = -5;
                    break;
                case 3://手刹
                    speed = 0;
                default:
                    speedChange = -1;
                    break;
            }
            speed += speedChange;
            if (speed < 1) {
                speed = 0;
            }
            try {
                Thread.sleep(50);
                setSpeed(speed);
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }
    }

在activity中启动线程，设置监听

        //设置监听
        speedUp.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        //按下的时候加速
                        speedControlView.setType(1);
                        break;
                    case MotionEvent.ACTION_UP:
                        //松开做自然减速
                        speedControlView.setType(0);
                        break;
                }
                return true;
            }
        });
        speedDown.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        //按下的时候减速
                        speedControlView.setType(2);
                        break;
                    case MotionEvent.ACTION_UP:
                        //松开做自然减速
                        speedControlView.setType(0);
                        break;
                }
                return true;
            }
        });
        shutDown.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        //按下的时候拉手刹
                        speedControlView.setType(3);
                        break;
                    case MotionEvent.ACTION_UP:
                        //松开做自然减速
                        speedControlView.setType(0);
                        break;
                }
                return true;
            }
        });


    @Override
    protected void onResume() {
        super.onResume();
        if (speedControlView != null) {
            speedControlView.setSpeed(0);
            speedControlView.setStart(true);
        }
        new Thread(speedControlView).start();

    }

补上速度的设置函数

    // 设置速度 并重绘视图
    public void setSpeed(int speed) {
        this.speed = speed;
        postInvalidate();
    }
搞定.
看，是不是很简单。
如果你喜欢我写的文章，请关注我。
我的博客：http://blog.csdn.net/lxk_1993
欢迎留言拍砖。

项目地址：
github: [https://github.com/103style/SpeedControl](https://github.com/103style/SpeedControl) 觉得可以的话点下star 
csdn:[http://download.csdn.net/download/lxk_1993/9518999](http://download.csdn.net/download/lxk_1993/9518999)
