# java.lang.NullPointerException Attempt to invoke virtual method 'int android.widget.Editor 

**bugly的报错**
```
java.lang.NullPointerException 
Attempt to invoke virtual method 'int android.widget.Editor
on a null object reference android.widget.Editor.touchPositionIsInSelection(Editor.java:955)
android.widget.Editor.touchPositionIsInSelection(Editor.java:901)
android.widget.Editor.performLongClick(Editor.java:1002)
android.widget.TextView.performLongClick(TextView.java:9261)
android.view.View$CheckForLongPress.run(View.java:21151)
android.os.Handler.handleCallback(Handler.java:742)
android.os.Handler.dispatchMessage(Handler.java:95)
android.os.Looper.loop(Looper.java:154)
android.app.ActivityThread.main(ActivityThread.java:5523)
java.lang.reflect.Method.invoke(Native Method)
com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:739)
com.android.internal.os.ZygoteInit.main(ZygoteInit.java:629)
```

![主要出现在小米的机型](http://upload-images.jianshu.io/upload_images/1709375-a505bbe22d1b1669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**问题原因：**

**view.setMovementMethod(LinkMovementMethod.getInstance());**

**一般用于 点击 textview中的 可跳转链接的时候使用**

**但是长按这个View的时候 就会报这个异常。**


```
    public final void setMovementMethod(MovementMethod movement) {
        if (mMovement != movement) {
            mMovement = movement;

            if (movement != null && !(mText instanceof Spannable)) {
                setText(mText);
            }

            fixFocusableAndClickableSettings();

            // SelectionModifierCursorController depends on textCanBeSelected, which depends on
            // mMovement
            if (mEditor != null) mEditor.prepareCursorControllers();
        }
    }

    private void fixFocusableAndClickableSettings() {
        if (mMovement != null || (mEditor != null && mEditor.mKeyListener != null)) {
            setFocusable(FOCUSABLE);
            setClickable(true);
            setLongClickable(true);
        } else {
            setFocusable(FOCUSABLE_AUTO);
            setClickable(false);
            setLongClickable(false);
        }
    }
```

**可以看到,** 

**会调用到setLongClickable(true);方法,**

**此时长按就会出现如上log所示的崩溃,** 

**而解决方案就一目了然了,** 

**有两种,都是屏蔽长按事件罢了:** 

* **调用 TextView.setLongClickable(false); 禁用长按事件** 

* **android:longClickable="false"** 

* **消费掉长按事件即可;** 

```
TextView.setOnLongClickListener(new View.OnLongClickListener() {
	@Override
	public boolean onLongClick(View v) {
		return true;
	}
});
```


 **不过我看到Android-26的sdk已经修复了这个问题了，有加空判断**

![android-26 sdk 已处理](http://upload-images.jianshu.io/upload_images/1709375-eaf583f3bca4d02a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
