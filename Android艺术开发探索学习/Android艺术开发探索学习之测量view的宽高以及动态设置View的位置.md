>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**progress动态更新位置实战**

首先看看我们要实现的效果。

![效果图](http://upload-images.jianshu.io/upload_images/1709375-357b760f872ce33e?imageMogr2/auto-orient/strip)

效果就是这样 看起来这简单。 其实实现起来也很简单。

之前做项目有碰到过这样的需求。

首先获取 `View` 的宽度和高度。刚开始我以为很简单，直接在 `onCreate()` 方法下直接获取 `view` 的宽度，
但是我发现 `w` 一直为 `0`.
然后又想到，在 `onResume` 的时候视图已经出来在我们视野了，在这里获取应该可以了吧。
然后结果让我大失所望。

后面百度找解决方法，用 `ViewTreeObserver` 实现了。

---

然后最近看 **Android艺术开发探索** 的时候又看到了这个问题的解决方法。遂记录下来。

**获取View的宽高的方法有很多，这里给出三种解决方法**:

* **通过 `post` 将一个 `runnable` 投递要消息队列的尾部，然后等待 `looper` 调用此方法的时候，视图也已经初始化好了**。
用法如下：`progressValue` 为你要测量的 `view`.
    ```
    progesssValue.post(new Runnable() {
        @Override
        public void run() {
            int w = progesssValue.getMeasuredWidth();
         }
    });
    ```

* **`ViewTreeObserver` 添加 `addOnGlobalLayoutListener`** `（需要在API 16以上）`
    代码如下:
    ```
    final ViewTreeObserver observer = progesssValue.getViewTreeObserver();
    observer.addOnGlobalLayoutListener(new  ViewTreeObserver.OnGlobalLayoutListener() {
        @TargetApi(Build.VERSION_CODES.JELLY_BEAN)
        @Override
        public void onGlobalLayout() {
            //此处不能写 observer.removeOnGlobalLayoutListener(this); 否则会报错         

           progesssValue.getViewTreeObserver().removeOnGlobalLayoutListener(this);
           int w = progesssValue.getMeasuredWidth();
        }
    });
    ```
* **重写 `Activity` 或者 `View` 的 `onWindowFocusChanged` 这个方法**
    * `onWindowFocusChanged(true)` 表示 `view` 获得了焦点
    * `onWindowFocusChanged` 方法会在 `activity` 获得焦点和失去焦点的时候调用
    ```
    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus) {
            int w = progesssValue.getMeasuredWidth();
        }
    }
    ```

---

然后就是动态改变View的位置，也比较简单，`setOnTouchListener` 就好了。
```
full.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                x1 = (int) event.getRawX();
                break;
            case MotionEvent.ACTION_MOVE:
                x2 = (int) event.getRawX();
                dx = x2 - x1;
                int w = getWindowManager().getDefaultDisplay().getWidth();
                if (Math.abs(dx) > w / 100) {
                    x1 = x2; // 去掉已经用掉的距离， 去掉这句 运行看看会出现效果
                    progesss.setProgress(progesss.getProgress() + dx * 100 / w);
                    setPos();
                }
                break;
            case MotionEvent.ACTION_UP:
                break;
        }
        return true;
    }
});

//设置进度显示在对应的位置
public void setPos() {
    int w = getWindowManager().getDefaultDisplay().getWidth();
    Log.e("w=====", "" + w);
    ViewGroup.MarginLayoutParams params = (ViewGroup.MarginLayoutParams) progesssValue.getLayoutParams();
    int pro = progesss.getProgress();
    int tW = progesssValue.getWidth();
    if (w * pro / 100 < tW * 0.7) {
        params.leftMargin = 0;
    } else if (w * pro / 100 + tW * 0.3 > w) {
        params.leftMargin = w - tW;
    } else {
        params.leftMargin = (int) (w * pro / 100 - tW * 0.7);
    }
    progesssValue.setLayoutParams(params);
    progesssValue.setText(new StringBuffer().append(progesss.getProgress()).append("%"));
}
```

---

博客地址：http://blog.csdn.net/lxk_1993
如果你喜欢我的博客，请关注我。
欢迎留言拍砖。

#### 源码位置：
* **github**:[https://github.com/103style/ViewMeasure 有用的话帮忙star下](https://github.com/103style/ViewMeasure)   修改点击进度位置也可以调整进度
* **csdn资源下载**：[http://download.csdn.net/download/lxk_1993/9466638](http://download.csdn.net/download/lxk_1993/9466638)

---

以上
