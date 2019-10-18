# Android检查通知权限是否开启 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


记录一下

```
/**
 * 检查通知权限
 */
private void checkNotification() {
    boolean enabled = NotificationManagerCompat.from(this).areNotificationsEnabled();
    if (enabled) {
        return;
    }
    new AlertDialog.Builder(this)
            .setCancelable(false)
            .setMessage("通知权限未打开的提示文字")
            .setPositiveButton(R.string.ok, (DialogInterface dialog, int which) -> {
                if (android.os.Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
                    Intent intent = new Intent();
                    intent.setAction("android.settings.APP_NOTIFICATION_SETTINGS");
                    intent.putExtra("app_package", this.getPackageName());
                    intent.putExtra("app_uid", this.getApplicationInfo().uid);
                    startActivity(intent);
                } else if (android.os.Build.VERSION.SDK_INT == Build.VERSION_CODES.KITKAT) {
                    Intent intent = new Intent();
                    intent.setAction(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
                    intent.addCategory(Intent.CATEGORY_DEFAULT);
                    intent.setData(Uri.parse("package:" + this.getPackageName()));
                    startActivity(intent);
                } else {
                    Intent localIntent = new Intent();
                    localIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    localIntent.setAction("android.settings.APPLICATION_DETAILS_SETTINGS");
                    localIntent.setData(Uri.fromParts("package", this.getPackageName(), null));
                    startActivity(localIntent);
                }
            })
            .setNegativeButton(R.string.cancel, (DialogInterface dialog, int which) -> {
                dialog.dismiss();
            }).show();
}
```
