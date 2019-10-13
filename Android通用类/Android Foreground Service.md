>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


为了防止后台服务被系统干掉，我们需要将服务提升为前台服务。

示例代码：

需要在`AndroidManifest` 添加 前台服务的权限 :
`<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>` 
>**FOREGROUND_SERVICE**
>
>Added in **API level 28** `Android 9.0`
>
>public static final String FOREGROUND_SERVICE
>
>Allows a regular application to use **Service.startForeground**.
>
>Protection level: normal
>
>Constant Value: `android.permission.FOREGROUND_SERVICE`

```
public class SampleService extends Service {

    public static final String CHANNEL_ID = "com.github.103style.SampleService";
    public static final String CHANNEL_NAME = "com.github.103style";

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        registerNotificationChannel();
        int notifyId = (int) System.currentTimeMillis();
        NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this, CHANNEL_ID);
        mBuilder
                //必须要有
                .setSmallIcon(R.mipmap.ic_launcher)
                //可选
                //.setSound(null)
                //.setVibrate(null)
                //...
        ;
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
            mBuilder.setContentTitle(getResources().getString(R.string.app_name));
        }
        startForeground(notifyId, mBuilder.build());
    }

    /**
     * 注册通知通道
     */
    private void registerNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationManager mNotificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
            NotificationChannel notificationChannel = mNotificationManager.getNotificationChannel(CHANNEL_ID);
            if (notificationChannel == null) {
                NotificationChannel channel = new NotificationChannel(CHANNEL_ID,
                        CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH);
                //是否在桌面icon右上角展示小红点
                channel.enableLights(true);
                //小红点颜色
                channel.setLightColor(Color.RED);
                //通知显示
                channel.setLockscreenVisibility(Notification.VISIBILITY_PUBLIC);
                //是否在久按桌面图标时显示此渠道的通知
                //channel.setShowBadge(true);
                mNotificationManager.createNotificationChannel(channel);
            }
        }
    }
}
```

以上
