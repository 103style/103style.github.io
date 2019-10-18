# Android监听来电状态 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


记录一下.

```
public class PhoneCallReceiver extends BroadcastReceiver {
    private static final String TAG = "PhoneCallReceiver ";
    private OnPhoneCallListener onPhoneCallListener;

    @Override
    public void onReceive(Context context, Intent intent) {
        TelephonyManager tm = (TelephonyManager) context.getSystemService(Service.TELEPHONY_SERVICE);
        int state = tm.getCallState();
        Logg.e(TAG, "state = " + state);
        if (onPhoneCallListener != null) {
            onPhoneCallListener.hasNewCall(state == TelephonyManager.CALL_STATE_RINGING);
        }
    }

    public void setOnOnPhoneCallListener(OnPhoneCallListener onPhoneCallListener) {
        this.onPhoneCallListener = onPhoneCallListener;
    }

    public interface OnPhoneCallListener {
        void hasNewCall(boolean valid);
    }
}
````

* `fragment or activity`：
```
/**
 * 来电响铃的监听
 */
private PhoneCallReceiver phoneCallReceiver;
/**
 * 来电响铃监听是否已注册
 */
 private boolean hasRegisterPhoneCall = false;

// Activity
//@Override
//protected void onRestart() {
//    super.onRestart();
//    registerPhoneCallReceiver(true);
//}
@Override
public void onStart() {
    super.onStart();
    registerPhoneCallReceiver(true);
}
@Override
public void onStop() {
    super.onStop();
    registerPhoneCallReceiver(false);
}
/**
 * 监听来电响铃状态
 */
public void registerPhoneCallReceiver(boolean register) {
    if (register) {
        if (phoneCallReceiver== null) {
            phoneCallReceiver= new PhoneCallReceiver();
            phoneCallReceiver.setOnOnPhoneCallListener(valid -> {
                // TODO: 处理来电 响铃之后的逻辑
            });
        }
        if (!hasRegisterPhoneCall) {
            hasRegisterPhoneCall = true;
            activity.registerReceiver(phoneCallReceiver, new IntentFilter());
        }
    } else {
        if (phoneCallReceiver!= null && hasRegisterPhoneCall) {
            hasRegisterPhoneCall = false;
            activity.unregisterReceiver(phoneCallReceiver);
        }
    }
}
```
* `AndroidManifest.xml`
```
<uses-permission android:name="android.permission.READ_PHONE_STATE" />

<receiver android:name=".PhoneCallReceiver">
    <intent-filter android:priority="1000">
        <action android:name="android.intent.action.PHONE_STATE" />
    </intent-filter>
</receiver>
```
