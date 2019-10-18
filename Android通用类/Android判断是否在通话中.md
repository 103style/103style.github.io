# Android判断是否在通话中 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

最后的判断代码：
```
/**
 * 是否正在电话通话中
 */
private boolean phoneIsInUse() {
    TelephonyManager mTelephonyManager = (TelephonyManager) activity.getSystemService(Context.TELEPHONY_SERVICE);
    int state = mTelephonyManager.getCallState();
    return state != TelephonyManager.CALL_STATE_IDLE;
}
```

---

开始在网上搜了搜，找到下面这两个：

* 然后 却找不到 `ITelephony` 类了。
    ```
    private boolean phoneIsInUse() {
        boolean phoneInUse = false;
        try {
            ITelephony phone = ITelephony.Stub.asInterface(ServiceManager.checkService("phone"));
            if (phone != null) phoneInUse = !phone.isIdle();
        } catch (RemoteException e) {
            Log.w(TAG, "phone.isIdle() failed", e);
        }
        return phoneInUse;
    }
    ```
*  `6.0`之后才可以用这个， 且需要判断 `READ_PHONE_STATE` 权限.
    ```
    public static boolean phoneIsInUse(Context context){
        TelecomManager tm = (TelecomManager)context.getSystemService(Context.TELECOM_SERVICE);
        return tm.isInCall();
    }
    ```
