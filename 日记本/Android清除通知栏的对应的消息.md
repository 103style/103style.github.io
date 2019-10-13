>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

记录一下

### 大致思路
* 我们收到推送消息的时候会通过 `NotificationManager.notify(int id, Notification notification)` 发送到通知栏。
* 记录每一个显示的 通知栏消息 和 对应的 `id`.
* 按产品要求在进入对应的页面的时候通过 `NotificationManager.cancel(id)` 删除对应的通知栏消息。

---

## 伪代码

通过`sendNotification(...)`显示推送消息，在对应的界面调用类似 `cleanMsgNotify(int notice) `  清除推送消息即可。
```
public static final String CHANNEL_ID = "XXXX";
private static NotificationManager mNotificationManager;
private static List<PushMessageBean> notifyList;

public synchronized static void cleanMsgNotify(int notice) {
    if (mNotificationManager == null
            || notifyList == null || notifyList.size() == 0) {
        return;
    }
    for (int i = notifyList.size() - 1; i >= 0; i--) {
        PushMessageBean t = notifyList.get(i);
        if (t.notice == notice) {
            mNotificationManager.cancel(t.notifyId);
            notifyList.remove(i);
        }
    }
}
public void sendNotification(Context context, PushMessageBean message) {
    if (TextUtils.isEmpty(message.message)) {
        return;
    }
    NotificationCompat.Builder mBuilder;
    int notifyId = (int) System.currentTimeMillis();
    if (mNotificationManager == null) {
        mNotificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    }
    registerNotificationChannel();
    mBuilder = new NotificationCompat.Builder(context, CHANNEL_ID);
    mBuilder.setDefaults(Notification.DEFAULT_SOUND | Notification.DEFAULT_VIBRATE)
            .setAutoCancel(true)
            .setContentText(message.message)
            .setSmallIcon(R.drawable.ic_launchers_round)
            .setVibrate(new long[]{1000})
            .setColor(context.getResources().getColor(R.color.color_primary))
            .setVisibility(Notification.VISIBILITY_PUBLIC)
            .setStyle(new NotificationCompat.BigTextStyle().bigText(message.message));
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {//7.0以上不需要title
        mBuilder.setContentTitle(context.getResources().getString(R.string.app_name));
    }
    message.notifyId = notifyId;
    saveNotification(message);
    mNotificationManager.notify(notifyId, mBuilder.build());
}
private void saveNotification(PushMessageBean message) {
    if (notifyList == null) {
        notifyList = new ArrayList<>();
    }
    notifyList.add(message);
}
private void registerNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        NotificationChannel notificationChannel = mNotificationManager.getNotificationChannel(CHANNEL_ID);
        if (notificationChannel == null) {
            NotificationChannel channel = new NotificationChannel(CHANNEL_ID,
                    CHANNEL_ID, NotificationManager.IMPORTANCE_HIGH);
            channel.enableLights(true); //是否在桌面icon右上角展示小红点
            channel.setLightColor(Color.RED); //小红点颜色
            //channel.setShowBadge(true); //是否在久按桌面图标时显示此渠道的通知
            mNotificationManager.createNotificationChannel(channel);
        }
    }
}
```

---

 以上
