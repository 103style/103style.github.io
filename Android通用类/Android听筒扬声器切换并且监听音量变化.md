>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 



记录一下。

在activity 监听按键：
```
@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    if (keyCode == KeyEvent.KEYCODE_VOLUME_UP || keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
        //设置聊天播放语音是 声音大小 随按键 变化
        int direction = keyCode == KeyEvent.KEYCODE_VOLUME_UP ? AudioManager.ADJUST_RAISE : AudioManager.ADJUST_LOWER;
        int flags = AudioManager.FX_FOCUS_NAVIGATION_UP;
        AudioManager am = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
        if (扬声器模式){
            am.adjustStreamVolume(AudioManager.STREAM_SYSTEM, direction, flags);
        }else {
            am.adjustStreamVolume(AudioManager.STREAM_VOICE_CALL, direction, flags);
        }
        return true;
    }
    return super.onKeyDown(keyCode, event);
}
```

听筒、扬声器 模式切换：
```

private static void init(Context mContext) {
    audioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
}
/**
 * 设置播放模式
 */
public static void setAudioStreamType(Context mContext, boolean speaker) {
    if (speaker) {
        audioManager.setSpeakerphoneOn(true);
        audioManager.setMode(AudioManager.MODE_NORMAL);
        //设置音量，解决有些机型切换后没声音或者声音突然变大的问题
        audioManager.setStreamVolume(AudioManager.STREAM_SYSTEM,
                audioManager.getStreamVolume(AudioManager.STREAM_SYSTEM), AudioManager.FX_KEY_CLICK);
    } else {
        audioManager.setSpeakerphoneOn(false);//关闭扬声器
        //5.0以上
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            audioManager.setMode(AudioManager.MODE_IN_COMMUNICATION);
            //设置音量，解决有些机型切换后没声音或者声音突然变大的问题
            audioManager.setStreamVolume(AudioManager.STREAM_VOICE_CALL,
                    audioManager.getStreamMaxVolume(AudioManager.STREAM_VOICE_CALL), AudioManager.FX_KEY_CLICK);
        } else {
            audioManager.setMode(AudioManager.MODE_IN_CALL);
            audioManager.setStreamVolume(AudioManager.STREAM_VOICE_CALL,
                    audioManager.getStreamMaxVolume(AudioManager.STREAM_VOICE_CALL), AudioManager.FX_KEY_CLICK);
        }
    }
}
```

`audioManager.setMode(int mode)`
mode 类型参照表：
```
/** Used to identify the volume of audio streams for phone calls */
    public static final int STREAM_VOICE_CALL = AudioSystem.STREAM_VOICE_CALL;
    /** Used to identify the volume of audio streams for system sounds */
    public static final int STREAM_SYSTEM = AudioSystem.STREAM_SYSTEM;
    /** Used to identify the volume of audio streams for the phone ring */
    public static final int STREAM_RING = AudioSystem.STREAM_RING;
    /** Used to identify the volume of audio streams for music playback */
    public static final int STREAM_MUSIC = AudioSystem.STREAM_MUSIC;
    /** Used to identify the volume of audio streams for alarms */
    public static final int STREAM_ALARM = AudioSystem.STREAM_ALARM;
    /** Used to identify the volume of audio streams for notifications */
    public static final int STREAM_NOTIFICATION = AudioSystem.STREAM_NOTIFICATION;
    /** Used to identify the volume of audio streams for DTMF Tones */
    public static final int STREAM_DTMF = AudioSystem.STREAM_DTMF;
    /** Used to identify the volume of audio streams for accessibility prompts */
    public static final int STREAM_ACCESSIBILITY = AudioSystem.STREAM_ACCESSIBILITY;
```

### 参考：
* [Android 听筒扬声器切换（多机型兼容、兼容5.0以上）]([https://blog.csdn.net/google_acmer/article/details/54141229](https://blog.csdn.net/google_acmer/article/details/54141229)
)
