>转载请以链接形式标明出处： [https://www.jianshu.com/p/3924ab03a97b](https://www.jianshu.com/p/3924ab03a97b)
本文出自:[**103style的博客**](https://www.jianshu.com/u/109656c2d96f) 


**android:layout_marginEnd隐藏的坑，巨坑**
**类似问题 setMargins无效**


相信稍微有强迫症的开发小伙伴都会看到**xml中的类似的这种warning提示**

    “Consider addingandroid:layout_marginEnd="@dimen/px_30_w750" to better support right-to-left layouts less... ”
**在你写了左边距和右边距不相等的时候，就会提示你**


然而这种平时是不会有什么问题的！
当你需要  **动态改变 控件位置**的时候，  
比如这样，

     if (test != null) {
            RelativeLayout.LayoutParams testLP = (RelativeLayout.LayoutParams) test.getLayoutParams();
            testLP .setMargins(0, 0,
                    DensityUtil.getSize(landsreen ? R.dimen.px_130_w750 : R.dimen.px_30_w750),
                    DensityUtil.getSize(landsreen ? R.dimen.px_140_w750 : R.dimen.px_316_w750));
            test.setLayoutParams(testLP );
        }


然而**setMargins**的源码改变的是**rightMargin**
**setMarginEnd**的源码改变的才是**endMargin**

     public void setMargins(int left, int top, int right, int bottom) {
            leftMargin = left;
            topMargin = top;
            rightMargin = right;
            bottomMargin = bottom;
            mMarginFlags &= ~LEFT_MARGIN_UNDEFINED_MASK;
            mMarginFlags &= ~RIGHT_MARGIN_UNDEFINED_MASK;
            if (isMarginRelative()) {
                mMarginFlags |= NEED_RESOLUTION_MASK;
            } else {
                mMarginFlags &= ~NEED_RESOLUTION_MASK;
            }
        }


     public void setMarginEnd(int end) {
            endMargin = end;
            mMarginFlags |= NEED_RESOLUTION_MASK;
        }


**然后在API LEVEL 17的时候  如果你同时写了 android:layout_marginEnd  和 android:layout_marginRight  , 他会去读 android:layout_marginEnd....** 

然后 你设置的setMargins 就起不了作用了...


实际效果是这样的 

**具体 android:layout_marginEnd 和 android:layout_marginRight   在布局的时候怎么添加的源码  我就先不研究了，后面有时间再补上**
**需要了解的可以自行看看**


转载请以链接形式标明出处：[http://www.jianshu.com/p/3924ab03a97b](http://www.jianshu.com/p/3924ab03a97b)
本文出自:[103style](http://www.jianshu.com/u/109656c2d96f)
