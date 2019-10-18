# Android SIM卡插拔监听 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 



记录一下
```
import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.telephony.TelephonyManager;

/**
 * 监听sim状态改变的广播，返回sim卡的状态， 有效或者无效。
 * 双卡中只要有一张卡的状态有效即返回状态为有效，两张卡都无效则返回无效。
 */
public class SimStateReceiver extends BroadcastReceiver {
    public final static String ACTION_SIM_STATE_CHANGED = "android.intent.action.SIM_STATE_CHANGED";
    private OnSimChangeListener onSimSwitchListener;

    @Override
    public void onReceive(Context context, Intent intent) {
        if (ACTION_SIM_STATE_CHANGED.equals(intent.getAction())) {
            TelephonyManager tm = (TelephonyManager) context.getSystemService(Service.TELEPHONY_SERVICE);
            int state = tm.getSimState();
            if (onSimSwitchListener != null) {
                onSimSwitchListener.simValid(state == TelephonyManager.SIM_STATE_READY);
            }
        }
    }

    public void setOnSimSwitchListener(OnSimChangeListener onSimSwitchListener) {
        this.onSimSwitchListener = onSimSwitchListener;
    }

    public interface OnSimChangeListener {
        void simValid(boolean valid);
    }
}
```

在**Fragment**  or  **Activity** 中使用：

```
    /**
     * sim 变化广播
     */
    private SimStateReceiver simStateReceiver;
    /**
     * sim监听是否已注册
     */
    private boolean hasRegister = false;

    @Override
    public void onStart() {
        super.onStart();
        registerSimReceiver(true);
    }

    @Override
    public void onStop() {
        super.onStop();
        registerSimReceiver(false);
    }

 /**
     * 监听sim卡状态
     */
    public void registerSimReceiver(boolean register) {
        if (register) {
            if (simStateReceiver == null) {
                simStateReceiver = new SimStateReceiver();
                simStateReceiver.setOnSimSwitchListener(valid -> {
                    if (!valid ) {
                        // TODO:  sim卡不可用
                    }
                });
            }
            IntentFilter intentFilter = new IntentFilter(SimStateReceiver.ACTION_SIM_STATE_CHANGED);
            if (!hasRegister) {
                hasRegister= true;
                context.registerReceiver(simStateReceiver, intentFilter);
            }
        } else {
            if (simStateReceiver != null && hasRegister) {
                hasRegister= false;
                context.unregisterReceiver(simStateReceiver);
            }
        }
    }
```
