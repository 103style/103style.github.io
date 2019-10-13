**本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布**

### Android 6.0
* #### 运行时权限
**相机，图库，下载，语音，定位....**
此版本引入了一种新的权限模式，如今，用户可直接在运行时管理应用权限。这种模式让用户能够更好地了解和控制权限，同时为应用开发者精简了安装和自动更新过程。用户可为所安装的各个应用分别授予或撤销权限。
对于以 Android 6.0（API 级别 23）或更高版本为目标平台的应用，请务必在运行时检查和请求权限。要确定您的应用是否已被授予权限，请调用新增的 [checkSelfPermission()](https://developer.android.google.cn/reference/android/content/Context.html#checkSelfPermission\(java.lang.String\))方法。要请求权限，请调用新增的[requestPermissions()](https://developer.android.google.cn/reference/android/app/Activity.html#requestPermissions(java.lang.String[],int)) 方法。即使您的应用并不以 Android 6.0（API 级别 23）为目标平台，您也应该在新权限模式下测试您的应用。如需了解有关在您的应用中支持新权限模式的详情，请参阅[使用系统权限](https://developer.android.google.cn/training/permissions/index.html)。如需了解有关如何评估新模式对应用的影响的提示，请参阅[权限最佳做法](https://developer.android.google.cn/training/permissions/best-practices.html#testing)。
 **权限管理工具类**


    package cn.loveshow.live.util;

    import android.Manifest;
    import android.app.Activity;
    import android.content.Context;
    import android.content.pm.PackageManager;
    import android.os.Build;
    import android.support.annotation.NonNull;
    import android.support.v4.app.ActivityCompat;
    import android.support.v4.app.Fragment;
    import android.support.v4.content.ContextCompat;

    import java.util.ArrayList;
    import java.util.Arrays;


    /**
     * Created by Fungo_Xiaoke on 2017/5/4 14:18.
     * eamil:luoxiaoke@yuntutv.net
     * 权限工具类
     */
    public class PermissionUtils {
    

    /**
     * @param context     上下文
     * @param activity    activity
     * @param permissions 权限数组
     * @param requestCode 申请码
     * @return true 有权限  false 无权限
     */
    public static boolean checkAndApplyfPermissionActivity(Activity activity, String[] permissions, int requestCode) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            permissions = checkPermissions(activity, permissions);
            if (permissions != null && permissions.length > 0) {
                ActivityCompat.requestPermissions(activity, permissions, requestCode);
                return false;
            } else {
                return true;
            }
        } else {
            return true;
        }
    }

    /**
     * @param context     上下文
     * @param mFragment   fragment
     * @param permissions 权限数组
     * @param requestCode 申请码
     * @return true 有权限  false 无权限
     */
    public static boolean checkAndApplyfPermissionFragment( Fragment mFragment, String[] permissions, int requestCode) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            permissions = checkPermissions(mFragment.getActivity(), permissions);
            if (permissions != null && permissions.length > 0) {
                if (mFragment.getActivity() != null) {
                    mFragment.requestPermissions(permissions, requestCode);
                }
                return false;
            } else {
                return true;
            }
        } else {
            return true;
        }
    }

    /**
     * @param context     上下文
     * @param permissions 权限数组
     * @return 还需要申请的权限
     */
    private static String[] checkPermissions(Context context, String[] permissions) {
        if (permissions == null || permissions.length == 0) {
            return new String[0];
        }
        ArrayList<String> permissionLists = new ArrayList<>();
        permissionLists.addAll(Arrays.asList(permissions));
        for (int i = permissionLists.size() - 1; i >= 0; i--) {
            if (ContextCompat.checkSelfPermission(context, permissionLists.get(i)) == PackageManager.PERMISSION_GRANTED) {
                permissionLists.remove(i);
            }
        }

        String[] temps = new String[permissionLists.size()];
        for (int i = 0; i < permissionLists.size(); i++) {
            temps[i] = permissionLists.get(i);
        }
        return temps;
        }


        /**
         * 检查申请的权限是否全部允许
         */
        public static boolean checkPermission(int[] grantResults) {
            if (grantResults == null || grantResults.length == 0) {
                return true;
            } else {
                int temp = 0;
                for (int i : grantResults) {
                    if (i == PackageManager.PERMISSION_GRANTED) {
                        temp++;
                    }
                }
                return temp == grantResults.length;
            }
        }

    /**
     * 没有获取到权限的提示
     *
     * @param permissions 权限名字数组
     */
    public static void showPermissionsToast(Activity activity, @NonNull String[] permissions) {
        if (permissions.length > 0) {
            for (String permission : permissions) {
                showPermissionToast(activity, permission);
            }
         }
       }

    /**
     * 没有获取到权限的提示
     *
     * @param permission 权限名字
     */
    private static void showPermissionToast(Activity activity, @NonNull String permission) {
        if (!ActivityCompat.shouldShowRequestPermissionRationale(activity, permission)) {
            //用户勾选了不再询问,提示用户手动打开权限
            switch (permission) {
                case Manifest.permission.CAMERA:
                    ToastUtils.showShort("相机权限已被禁止，请在应用管理中打开权限");
                    break;
                case Manifest.permission.WRITE_EXTERNAL_STORAGE:
                    ToastUtils.showShort("文件权限已被禁止，请在应用管理中打开权限");
                    break;
                case Manifest.permission.RECORD_AUDIO:
                    ToastUtils.showShort("录制音频权限已被禁止，请在应用管理中打开权限");
                    break;
                case Manifest.permission.ACCESS_FINE_LOCATION:
                    ToastUtils.showShort("位置权限已被禁止，请在应用管理中打开权限");
                    break;
            }
        } else {
            //用户没有勾选了不再询问,拒绝了权限申请
            switch (permission) {
                case Manifest.permission.CAMERA:
                    ToastUtils.showShort("没有相机权限");
                    break;
                case Manifest.permission.WRITE_EXTERNAL_STORAGE:
                    ToastUtils.showShort("没有文件读取权限");
                    break;
                case Manifest.permission.RECORD_AUDIO:
                    ToastUtils.showShort("没有录制音频权限");
                    break;
                case Manifest.permission.ACCESS_FINE_LOCATION:
                    ToastUtils.showShort("没有位置权限");
                    break;
              }
         }
      }
    }

**用法**

       if (PermissionUtils.checkAndApplyfPermissionActivity(this,
            new String[]{Manifest.permission.CAMERA, Manifest.permission.WRITE_EXTERNAL_STORAGE}, 
            REQUESRCARMEA)) {
           //获取到权限的操作  没有权限会申请权限  然后在onRequestPermissionsResult处理申请的结果
       }

       @Override
       public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
           super.onRequestPermissionsResult(requestCode, permissions, grantResults);
           if (PermissionUtils.checkPermission(grantResults)) {
               //申请权限成功
                switch (requestCode) {
                     case 0x001:
                         //dosomething
                         break;
                     case 0x002:
                        //dosomething
                         break;
                     ...
                }
           } else {
               //提示没有什么权限
               PermissionUtils.showPermissionsToast(activity, permissions);
                //or 去权限管理界面
                //gotoPermissionManager（mContext）;
           }
       }


  **没有权限去权限管理界面**

    /**
     * 去应用权限管理界面
     */
    public static void gotoPermissionManager(Context context) {
        Intent intent;
        ComponentName comp;
        //防止刷机出现的问题
        try {
            switch (Build.MANUFACTURER) {
                case "Huawei":
                    intent = new Intent();
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    comp = new ComponentName("com.huawei.systemmanager", "com.huawei.permissionmanager.ui.MainActivity");
                    intent.setComponent(comp);
                    context.startActivity(intent);
                    break;
                case "Meizu":
                    intent = new Intent("com.meizu.safe.security.SHOW_APPSEC");
                    intent.addCategory(Intent.CATEGORY_DEFAULT);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    context.startActivity(intent);
                    break;
                case "Xiaomi":
                    String rom = getSystemProperty("ro.miui.ui.version.name");
                    if ("v5".equals(rom)) {
                        Uri packageURI = Uri.parse("package:" + context.getApplicationInfo().packageName);
                        intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, packageURI);
                    } else {//if ("v6".equals(rom) || "v7".equals(rom)) {
                        intent = new Intent("miui.intent.action.APP_PERM_EDITOR");
                        intent.setClassName("com.miui.securitycenter", "com.miui.permcenter.permissions.AppPermissionsEditorActivity");
                        intent.putExtra("extra_pkgname", context.getPackageName());
                    }
                    context.startActivity(intent);
                    break;
                case "Sony":
                    intent = new Intent();
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    comp = new ComponentName("com.sonymobile.cta", "com.sonymobile.cta.SomcCTAMainActivity");
                    intent.setComponent(comp);
                    context.startActivity(intent);
                    break;
                case "OPPO":
                    intent = new Intent();
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    comp = new ComponentName("com.color.safecenter", "com.color.safecenter.permission.PermissionManagerActivity");
                    intent.setComponent(comp);
                    context.startActivity(intent);
                    break;
                case "LG":
                    intent = new Intent("android.intent.action.MAIN");
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    comp = new ComponentName("com.android.settings", "com.android.settings.Settings$AccessLockSummaryActivity");
                    intent.setComponent(comp);
                    context.startActivity(intent);
                    break;
                case "Letv":
                    intent = new Intent();
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    intent.putExtra("packageName", BuildConfig.APPLICATION_ID);
                    comp = new ComponentName("com.letv.android.letvsafe", "com.letv.android.letvsafe.PermissionAndApps");
                    intent.setComponent(comp);
                    context.startActivity(intent);
                    break;
                default:
                    getAppDetailSettingIntent(context);
                    break;
            }
        } catch (Exception e) {
            getAppDetailSettingIntent(context);
        }
    }

    /**
     * 获取系统属性值
     */
    public static String getSystemProperty(String propName) {
        String line;
        BufferedReader input = null;
        try {
            Process p = Runtime.getRuntime().exec("getprop " + propName);
            input = new BufferedReader(new InputStreamReader(p.getInputStream()), 1024);
            line = input.readLine();
            input.close();
        } catch (IOException ex) {
            Log.e(TAG, "Unable to read sysprop " + propName, ex);
            return null;
        } finally {
            if (input != null) {
                try {
                    input.close();
                } catch (IOException e) {
                    Log.e(TAG, "Exception while closing InputStream", e);
                }
            }
        }
        return line;
    }

      //以下代码可以跳转到应用详情，可以通过应用详情跳转到权限界面(6.0系统测试可用)
     public static void getAppDetailSettingIntent(Context context) {
        Intent localIntent = new Intent();
        localIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        if (Build.VERSION.SDK_INT >= 9) {
            localIntent.setAction("android.settings.APPLICATION_DETAILS_SETTINGS");
            localIntent.setData(Uri.fromParts("package", context.getPackageName(), null));
        } else if (Build.VERSION.SDK_INT <= 8) {
            localIntent.setAction(Intent.ACTION_VIEW);
            localIntent.setClassName("com.android.settings", "com.android.settings.InstalledAppDetails");
            localIntent.putExtra("com.android.settings.ApplicationPkgName", context.getPackageName());
        }
        launchApp(context, localIntent);
    }

      /**
     * 安全的启动APP
     */
    public static boolean launchApp(Context ctx, Intent intent) {
        if (ctx == null)
            throw new NullPointerException("ctx is null");
        try {
            ctx.startActivity(intent);
            return true;
        } catch (ActivityNotFoundException e) {
            Logger.e(e);
            return false;
        }
    }



* ####  取消支持 Apache HTTP 客户端
Android 6.0 版移除了对 Apache HTTP 客户端的支持。如果您的应用使用该客户端，并以 Android 2.3（API 级别 9）或更高版本为目标平台，请改用[HttpURLConnection](https://developer.android.google.cn/reference/java/net/HttpURLConnection.html) 类。此 API 效率更高，因为它可以通过透明压缩和响应缓存减少网络使用，并可最大限度降低耗电量。要继续使用 Apache HTTP API，您必须先在 build.gradle 文件中声明以下编译时依赖项：

      android {
          useLibrary 'org.apache.http.legacy'
      }

* #### BoringSSL
Android 正在从使用 OpenSSL 库转向使用 [BoringSSL](https://boringssl.googlesource.com/boringssl/) 库。如果您要在应用中使用 Android NDK，请勿链接到并非 NDK API 组成部分的加密库，如libcrypto.so和 libssl.so。这些库并非公共 API，可能会在不同版本和设备上毫无征兆地发生变化或出现故障。此外，您还可能让自己暴露在安全漏洞的风险之下。请改为修改原生代码，以通过 JNI 调用 Java 加密 API，或静态链接到您选择的加密库。

![bugly错误](http://upload-images.jianshu.io/upload_images/1709375-0466f1b103500070.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![bugly错误](http://upload-images.jianshu.io/upload_images/1709375-5c6694299fd98daf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* #### 通知
此版本移除了 Notification.setLatestEventInfo()方法。请改用 [Notification.Builder](https://developer.android.google.cn/reference/android/app/Notification.Builder.html) 类来构建通知。要重复更新通知，请重复使用[Notification.Builder](https://developer.android.google.cn/reference/android/app/Notification.Builder.html) 实例。调用 [build()](https://developer.android.google.cn/reference/android/app/Notification.Builder.html#build()) 方法可获取更新后的 [Notification](https://developer.android.google.cn/reference/android/app/Notification.html) 实例。
adb shell dumpsys notification 命令不再打印输出您的通知文本。请改用 adb shell dumpsys notification --noredact 命令打印输出 notification 对象中的文本。build()方法在4.1以上（16+）的系统才能用。


![notification](http://upload-images.jianshu.io/upload_images/1709375-f35387e6bd961f7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![例子](http://upload-images.jianshu.io/upload_images/1709375-2d191c7536338961.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![gif](http://upload-images.jianshu.io/upload_images/1709375-e89ed772e97f2ab1.gif?imageMogr2/auto-orient/strip)


* #### 音频管理器变更
不再支持通过 [AudioManager](https://developer.android.google.cn/reference/android/media/AudioManager.html) 类直接设置音量或将特定音频流 **静音**。[setStreamSolo()](https://developer.android.google.cn/reference/android/media/AudioManager.html#setStreamSolo(int, boolean)) 方法已弃用，您应该改为调用 [requestAudioFocus()](https://developer.android.google.cn/reference/android/media/AudioManager.html#requestAudioFocus(android.media.AudioManager.OnAudioFocusChangeListener, int, int)) 方法。类似地，[setStreamMute()](https://developer.android.google.cn/reference/android/media/AudioManager.html#setStreamMute(int, boolean)) 方法也已弃用，请改为调用 [adjustStreamVolume()](https://developer.android.google.cn/reference/android/media/AudioManager.html#adjustStreamVolume(int, int, int)) 方法并传入方向值 [ADJUST_MUTE](https://developer.android.google.cn/reference/android/media/AudioManager.html#ADJUST_MUTE) 或 [ADJUST_UNMUTE](https://developer.android.google.cn/reference/android/media/AudioManager.html#ADJUST_UNMUTE)。



* #### Android 密钥库变更
从此版本开始，[Android 密钥库提供程序](https://developer.android.google.cn/training/articles/keystore.html)不再支持 DSA。但仍支持 ECDSA。
停用或重置安全锁定屏幕时（例如，由用户或设备管理员执行此类操作时），系统将不再删除需要闲时加密的密钥，但在上述事件期间会删除需要闲时加密的密钥。




* #### APK 验证
该平台现在执行的 APK 验证更为严格。**如果在清单中声明的文件在 APK 中并不存在**，该 APK 将被视为已损坏。移除任何内容后必须重新签署 APK。

[http://www.jianshu.com/p/95790125b7f4](http://www.jianshu.com/p/95790125b7f4)
[http://blog.csdn.net/lxk_1993/article/details/73784883](http://blog.csdn.net/lxk_1993/article/details/73784883)

### Android7.0

* #### 系统权限更改
为了提高私有文件的安全性，面向 Android 7.0 或更高版本的应用私有目录被限制访问　(0700)。此设置可防止私有文件的元数据泄漏，如它们的大小或存在性。此权限更改有多重副作用：
 * 私有文件的文件权限不应再由所有者放宽，为使用 [MODE_WORLD_READABLE](https://developer.android.google.cn/reference/android/content/Context.html#MODE_WORLD_READABLE) 和/或 [MODE_WORLD_WRITEABLE](https://developer.android.google.cn/reference/android/content/Context.html#MODE_WORLD_WRITEABLE) 而进行的此类尝试将触发 [SecurityException](https://developer.android.google.cn/reference/java/lang/SecurityException.html)
**注**：迄今为止，这种限制尚不能完全执行。应用仍可能使用原生 API 或 [File](https://developer.android.google.cn/reference/java/io/File.html) API 来修改它们的私有目录权限。但是，我们强烈反对放宽私有目录的权限。

 * 传递软件包网域外的 file://URI 可能给接收器留下无法访问的路径。因此，尝试传递 file://URI 会触发FileUriExposedException。分享私有文件内容的推荐方法是使用 [FileProvider](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html)。
 * [DownloadManager](https://developer.android.google.cn/reference/android/app/DownloadManager.html)不再按文件名分享私人存储的文件。旧版应用在访问 [COLUMN_LOCAL_FILENAME](https://developer.android.google.cn/reference/android/app/DownloadManager.html#COLUMN_LOCAL_FILENAME)时可能出现无法访问的路径。面向 Android 7.0 或更高版本的应用在尝试访问 [COLUMN_LOCAL_FILENAME](https://developer.android.google.cn/reference/android/app/DownloadManager.html#COLUMN_LOCAL_FILENAME)时会触发 [SecurityException](https://developer.android.google.cn/reference/java/lang/SecurityException.html)。通过使用  [DownloadManager.Request.setDestinationInExternalFilesDir()](https://developer.android.google.cn/reference/android/app/DownloadManager.Request.html#setDestinationInExternalFilesDir(android.content.Context, java.lang.String, java.lang.String))  或 [DownloadManager.Request.setDestinationInExternalPublicDir()](https://developer.android.google.cn/reference/android/app/DownloadManager.Request.html#setDestinationInExternalPublicDir(java.lang.String, java.lang.String))将下载位置设置为公共位置的旧版应用仍可以访问 [COLUMN_LOCAL_FILENAME](https://developer.android.google.cn/reference/android/app/DownloadManager.html#COLUMN_LOCAL_FILENAME) 中的路径，但是我们强烈反对使用这种方法。对于由 [DownloadManager](https://developer.android.google.cn/reference/android/app/DownloadManager.html) 公开的文件，首选的访问方式是使用[ContentResolver.openFileDescriptor()](https://developer.android.google.cn/reference/android/content/ContentResolver.html#openFileDescriptor(android.net.Uri, java.lang.String))。


* #### 在应用间共享文件
对于面向 Android 7.0 的应用，Android 框架执行的 [StrictMode](https://developer.android.google.cn/reference/android/os/StrictMode.html)API 政策禁止在您的应用外部公开 file://URI。如果一项包含文件 URI 的 intent 离开您的应用，则应用出现故障，并现 FileUriExposedException。异常。要在应用间共享文件，您应发送一项 content://URI，并授予 URI 临时访问权限。进行此授权的最简单方式是使用[FileProvider](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html)类。如需了解有关权限和共享文件的详细信息，请参阅[共享文件](https://developer.android.google.cn/training/secure-file-sharing/index.html)。


* #### FileProvider用法
 **AndroidManiFest.xml**添加

      <application>
      ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="${applicationId}.fileProvider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/provider_paths"/>
        </provider>
      ...
      </application>


![配置applicationId](http://upload-images.jianshu.io/upload_images/1709375-0c5b08203c682528.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **res**目录下新建**xml**文件夹，创建**provider_paths.xml**文件
    
      <?xml version="1.0" encoding="utf-8"?>
      <resources>
        <paths>
        <!--  前面两个是bugly的 -->
        <!-- /storage/emulated/0/Download/${applicationId}/.beta/apk-->
        <external-path
            name="beta_external_path"
            path="Download/" />
        <!--/storage/emulated/0/Android/data/${applicationId}/files/apk/-->
        <external-path
            name="beta_external_files_path"
            path="Android/data/" />

        <external-path
            name="sdcard_files"
            path="" />
        <!--相机相册裁剪-->
        <external-files-path
            name="camera_has_sdcard"
            path="" />
        <files-path
            name="camera_no_sdcard"
            path="" />
        </paths>
        <!--<paths>-->
        <!-- xml文件是唯一设置分享的目录 ，不能用代码设置-->
         <!--1.<files-path>        getFilesDir()  /data/data//files目录-->
         <!--2.<cache-path>        getCacheDir()  /data/data//cache目录-->
         <!--3.<external-path>       Environment.getExternalStorageDirectory()  -->
         <!--4.<external-files-path>    
           Context.getExternalFilesDir(String)  Context.getExternalFilesDir(null)  
           == SDCard/Android/data/你的应用的包名/files/ 目录-->
         <!--5.<external-cache-path>      Context.getExternalCacheDir().-->

         <!--  path :代表设置的目录下一级目录 eg：<external-path path="images/"-->
         <!--整个目录为Environment.getExternalStorageDirectory()+"/images/"-->
         <!--name: 代表定义在Content中的字段 eg：name = "myimages" ，并且请求的内容的文件名为default_image.jpg-->
         <!--则 返回一个URI   content://com.example.myapp.fileprovider/myimages/default_image.jpg-->

       <!--</paths>-->
      </resources>


  
  **确认下路径名**
![路径](http://upload-images.jianshu.io/upload_images/1709375-41841603d864ded3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![路径](http://upload-images.jianshu.io/upload_images/1709375-488fe3787cf86c52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  **FileProvider 头部设置的对应标签**
![FileProvider](http://upload-images.jianshu.io/upload_images/1709375-09f08ad8c4508cc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
   **FileProvider 获取对应路径逻辑 解析xml文件  对比对应的标签  获取对应的路径**
![FileProvider](http://upload-images.jianshu.io/upload_images/1709375-cfa9143566373e1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **修改所有用到Uri的地方   图中的 BuildConfig.APPLICATION_ID  最好还是改成  context.getPackageName() **
![Uri修改](http://upload-images.jianshu.io/upload_images/1709375-462bce69f306b76f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 官方链接：[FileProvider](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html)



* #### APK Signature Scheme v2
 Android 7.0引入了全新的 APK Signature Scheme v2。这是加强对包的校验，启动了新的签名后，像美团的多渠道打包方案在7.0机器上就会报错了。
 解决的办法也很简单，官方提供了关闭v2签名的方法，只需要在gradle上配置一下即可：
      signingConfigs {
          release {
          .......
          v2SigningEnabled false
         }
      }
参考：[Android7.0适配](http://www.jianshu.com/p/55f0e118c62e)
链接： [APK Signature Scheme v2详细介绍](http://www.jianshu.com/p/55f0e118c62e)

### 其他

  **Android6.0**
 * #### USB 连接
默认情况下，现在通过 USB 端口进行的设备连接设置为仅充电模式。要通过 USB 连接访问设备及其内容，用户必须明确地为此类交互授予权限。如果您的应用支持用户通过 USB 端口与设备进行交互，请将必须显式启用交互考虑在内。

* #### 浏览器书签变更
此版本移除了对全局书签的支持。android.provider.Browser.getAllBookmarks() 和 android.provider.Browser.saveBookmark() 方法现已移除。同样，READ_HISTORY_BOOKMARKS 权限和 WRITE_HISTORY_BOOKMARKS 权限也已移除。如果您的应用以 Android 6.0（API 级别 23）或更高版本为目标平台，请勿从全局提供程序访问书签或使用书签权限。您的应用应改为在内部存储书签数据。

* #### 硬件标识符访问权
为给用户提供更严格的数据保护，从此版本开始，对于使用 WLAN API 和 Bluetooth API 的应用，Android 移除了对设备本地硬件标识符的编程访问权。[WifiInfo.getMacAddress()](https://developer.android.google.cn/reference/android/net/wifi/WifiInfo.html#getMacAddress())方法和 [BluetoothAdapter.getAddress()](https://developer.android.google.cn/reference/android/bluetooth/BluetoothAdapter.html#getAddress())方法现在会返回常量值 02:00:00:00:00:00。
现在，要通过蓝牙和 WLAN 扫描访问附近外部设备的硬件标识符，您的应用必须拥有 [ACCESS_FINE_LOCATION](https://developer.android.google.cn/reference/android/Manifest.permission.html#ACCESS_FINE_LOCATION) 或 [ACCESS_COARSE_LOCATION](https://developer.android.google.cn/reference/android/Manifest.permission.html#ACCESS_COARSE_LOCATION) 权限。
 * [WifiManager.getScanResults()](https://developer.android.google.cn/reference/android/net/wifi/WifiManager.html#getScanResults())
 * [BluetoothDevice.ACTION_FOUND](https://developer.android.google.cn/reference/android/bluetooth/BluetoothDevice.html#ACTION_FOUND)
 * [BluetoothLeScanner.startScan()](https://developer.android.google.cn/reference/android/bluetooth/le/BluetoothLeScanner.html#startScan(android.bluetooth.le.ScanCallback))

 **注**：当运行 Android 6.0（API 级别 23）的设备发起后台 WLAN 或蓝牙扫描时，在外部设备看来，该操作的发起来源是一个随机化 MAC 地址。

* #### WLAN 和网络连接变更
此版本对 WLAN API 和 Networking API 引入了以下行为变更。
 * 现在，您的应用只能更改由您创建的 [WifiConfiguration](https://developer.android.google.cn/reference/android/net/wifi/WifiConfiguration.html) 对象的状态。系统不允许您修改或删除由用户或其他应用创建的 [WifiConfiguration](https://developer.android.google.cn/reference/android/net/wifi/WifiConfiguration.html) 对象。
 * 在之前的版本中，如果应用利用带有 disableAllOthers=true
 设置的 [enableNetwork()](https://developer.android.google.cn/reference/android/net/wifi/WifiManager.html#enableNetwork(int, boolean)) 强制设备连接特定 WLAN 网络，设备将会断开与移动数据网络等其他网络的连接。在此版本中，设备不再断开与上述其他网络的连接。如果您的应用的 targetSdkVersion
 为 “20” 或更低，则会固定连接所选 WLAN 网络。如果您的应用的 targetSdkVersion 为 “21”
 或更高，请使用多网络 API（如 [openConnection()](https://developer.android.google.cn/reference/android/net/Network.html#openConnection(java.net.URL))、[bindSocket()](https://developer.android.google.cn/reference/android/net/Network.html#bindSocket(java.net.Socket)) 和新增的[bindProcessToNetwork()](https://developer.android.google.cn/reference/android/net/ConnectivityManager.html#bindProcessToNetwork(android.net.Network))
 方法）来确保通过所选网络传送网络流量。

* #### 相机服务变更
在此版本中，相机服务中共享资源的访问模式已从之前的“先到先得”访问模式更改为高优先级进程优先的访问模式。对服务行为的变更包括：
 * 根据客户端应用进程的“优先级”授予对相机子系统资源的访问权，包括打开和配置相机设备。带有对用户可见 Activity 或前台 Activity 的应用进程一般会被授予较高的优先级，从而使相机资源的获取和使用更加可靠；
 * 当高优先级的应用尝试使用相机时，系统可能会“驱逐”正在使用相机客户端的低优先级应用。在已弃用的 [Camera](https://developer.android.google.cn/reference/android/hardware/Camera.html) API 中，这会导致系统为被驱逐的客户端调用 [onError()](https://developer.android.google.cn/reference/android/hardware/Camera.ErrorCallback.html#onError(int, android.hardware.Camera))。在 [Camera2](https://developer.android.google.cn/reference/android/hardware/camera2/package-summary.html) API 中，这会导致系统为被驱逐的客户端调用 [onDisconnected()](https://developer.android.google.cn/reference/android/hardware/camera2/CameraDevice.StateCallback.html#onDisconnected(android.hardware.camera2.CameraDevice))； 
 * 在配备相应相机硬件的设备上，不同的应用进程可同时独立打开和使用不同的相机设备。但现在，如果在多进程用例中同时访问相机会造成任何打开的相机设备的性能或能力严重下降，相机服务会检测到这种情况并禁止同时访问。即使并没有其他应用直接尝试访问同一相机设备，此变更也可能导致低优先级客户端被“驱逐”。
 * 更改当前用户会导致之前用户帐户拥有的应用内活动相机客户端被驱逐。对相机的访问仅限于访问当前设备用户拥有的用户个人资料。举例来说，这意味着，当用户切换到其他帐户后，“来宾”帐户实际上无法让使用相机子系统的进程保持运行状态。
 
* #### 运行时
ART 运行时环境现在可正确实现 [newInstance()](https://developer.android.google.cn/reference/java/lang/reflect/Constructor.html#newInstance(java.lang.Object...)) 方法的访问规则。此变更修正了之前版本中 Dalvik 无法正确检查访问规则的问题。如果您的应用使用[newInstance()](https://developer.android.google.cn/reference/java/lang/reflect/Constructor.html#newInstance(java.lang.Object...)) 方法，并且您想重写访问检查，请调用 [setAccessible()](https://developer.android.google.cn/reference/java/lang/reflect/AccessibleObject.html#setAccessible(boolean)) 方法（将输入参数设置为 true）。如果您的应用使用 [v7 appcompat 库](https://developer.android.google.cn/tools/support-library/features.html#v7-appcompat)或 [v7 recyclerview 库](https://developer.android.google.cn/tools/support-library/features.html#v7-recyclerview)，则您必须更新应用以使用这些库的最新版本。否则，请务必更新从 XML 引用的任何自定义类，以便能够访问它们的类构造函数。此版本更新了动态链接程序的行为。动态链接程序现在可以识别库的 soname 与其路径之间的差异（[公开错误 6670](https://code.google.com/p/android/issues/detail?id=6670)），并且现在已实现了按 soname 搜索。之前包含错误的 DT_NEEDED 条目（通常是开发计算机文件系统上的绝对路径）却仍工作正常的应用，如今可能会出现加载失败。现已正确实现 dlopen(3) RTLD_LOCAL 标记。请注意，RTLD_LOCAL 是默认值，因此不显式使用 RTLD_LOCAL 的 dlopen(3) 调用将受到影响（除非您的应用显式使用 RTLD_GLOBAL）。使用 RTLD_LOCAL 时，在随后通过调用 dlopen(3) 加载的库中并不能使用这些符号（这与由 DT_NEEDED 条目引用的情况截然不同）。

 在之前版本的 Android 上，如果您的应用请求系统加载包含文本重定位信息的共享库，系统会显示警告，但仍允许加载共享库。从此版本开始，如果您的应用的目标 SDK 版本为 23 或更高，则系统会拒绝加载该库。为帮助您检测库是否加载失败，您的应用应该记录 dlopen(3) 失败日志，并在日志中加入dlerror(3) 调用返回的问题描述文本。要详细了解如何处理文本重定位，请参阅此[指南](https://wiki.gentoo.org/wiki/Hardened/Textrels_Guide)。

* ####  低电耗模式和应用待机模式
此版本引入了针对空闲设备和应用的最新节能优化技术。这些功能会影响所有应用，因此请务必在这些新模式下测试您的应用。
 * **低电耗模式**：如果用户拔下设备的电源插头，并在屏幕关闭后的一段时间内使其保持不活动状态，设备会进入*低电耗*模式，在该模式下设备会尝试让系统保持休眠状态。在该模式下，设备会定期短时间恢复正常工作，以便进行应用同步，还可让系统执行任何挂起的操作。
 * **应用待机模式**：应用待机模式允许系统判定应用在用户未主动使用它时处于空闲状态。当用户有一段时间未触摸应用时，系统便会作出此判定。如果拔下了设备电源插头，系统会为其视为空闲的应用停用网络访问以及暂停同步和作业。

 要详细了解这些节能变更，请参阅[对低电耗模式和应用待机模式进行针对性优化](https://developer.android.google.cn/training/monitoring-device-state/doze-standby.html)。

* #### 文本选择
现在，当用户在您的应用中选择文本时，您可以在一个[浮动工具栏](http://www.google.com/design/spec/patterns/selection.html#selection-text-selection)中显示*“剪切”*、*“复制”*和*“粘贴”*等文本选择操作。其在用户交互实现上与[为单个视图启用上下文操作模式](https://developer.android.google.cn/guide/topics/ui/menus.html#CABforViews)中所述的上下文操作栏类似。
要实现可用于文本选择的浮动工具栏，请在您的现有应用中做出以下更改：
 * 在 [View](https://developer.android.google.cn/reference/android/view/View.html) 对象或 [Activity](https://developer.android.google.cn/reference/android/app/Activity.html) 对象中，将 [ActionMode](https://developer.android.google.cn/reference/android/view/ActionMode.html) 调用从 startActionMode(Callback)更改为 startActionMode(Callback, ActionMode.TYPE_FLOATING)。
 * 改为使用 ActionMode.Callback 的现有实现扩展 [ActionMode.Callback2](https://developer.android.google.cn/reference/android/view/ActionMode.Callback2.html)。
 * 替代 [onGetContentRect()](https://developer.android.google.cn/reference/android/view/ActionMode.Callback2.html#onGetContentRect(android.view.ActionMode, android.view.View, android.graphics.Rect)) 方法，用于提供 [Rect](https://developer.android.google.cn/reference/android/graphics/Rect.html) 内容对象（如文本选择矩形）在视图中的坐标。
 * 如果矩形的定位不再有效，并且这是唯一需要声明为无效的元素，请调[invalidateContentRect()](https://developer.android.google.cn/reference/android/view/ActionMode.html#invalidateContentRect()) 方法。

 请注意，如果您使用 [Android 支持库](https://developer.android.google.cn/tools/support-library/index.html) 22.2 修订版，浮动工具栏不向后兼容，默认情况下 appcompat 会获得对 [ActionMode](https://developer.android.google.cn/reference/android/view/ActionMode.html) 对象的控制权。这会禁止显示浮动工具栏。要在[ActionMode](https://developer.android.google.cn/reference/android/view/ActionMode.html) 中启用 [AppCompatActivity](https://developer.android.google.cn/reference/android/support/v7/app/AppCompatActivity.html) 支持，请调用 [getDelegate()](https://developer.android.google.cn/reference/android/support/v7/app/AppCompatActivity.html#getDelegate())，然后对返回的[setHandleNativeActionModesEnabled()](https://developer.android.google.cn/reference/android/support/v7/app/AppCompatDelegate.html#setHandleNativeActionModesEnabled(boolean)) 对象调用 [AppCompatDelegate](https://developer.android.google.cn/reference/android/support/v7/app/AppCompatDelegate.html)，并将输入参数设置为 false。此调用会将 [ActionMode](https://developer.android.google.cn/reference/android/view/ActionMode.html) 对象的控制权交还给框架。在运行 Android 6.0（API 级别 23）的设备上，框架可以支持 [ActionBar](https://developer.android.google.cn/reference/android/support/v7/app/ActionBar.html) 模式或浮动工具栏模式；而在运行 Android 5.1（API 级别 22）或之前版本的设备上，框架仅支持 [ActionBar](https://developer.android.google.cn/reference/android/support/v7/app/ActionBar.html) 模式。



* #### Android for Work 变更
此版本包含下列针对 Android for Work 的行为变更：
 * **个人上下文中的工作联系人。**Google 拨号器通话记录现在会在用户查看通话记录时显示工作联系人。将 [setCrossProfileCallerIdDisabled()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setCrossProfileCallerIdDisabled(android.content.ComponentName, boolean)) 设置为 true 可在 Google 拨号器通话记录中隐藏托管配置文件联系人。仅当将 [setBluetoothContactSharingDisabled()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setBluetoothContactSharingDisabled(android.content.ComponentName, boolean)) 设置为 false 时，才可以通过蓝牙将工作联系人随个人联系人一起显示给设备。默认情况下，它设置为 true。
 * **WLAN 配置删除**：现在，当删除某个托管配置文件时，将会移除由配置文件所有者添加的 WLAN 配置（例如，通过调用 [addNetwork()](https://developer.android.google.cn/reference/android/net/wifi/WifiManager.html#addNetwork(android.net.wifi.WifiConfiguration))
 方法添加的配置）。
 * **WLAN 配置锁定**：如果 [WIFI_DEVICE_OWNER_CONFIGS_LOCKDOWN](https://developer.android.google.cn/reference/android/provider/Settings.Global.html#WIFI_DEVICE_OWNER_CONFIGS_LOCKDOWN) 不为零，则用户无法再修改或删除任何由活动设备所有者创建的 WLAN 配置。用户仍可创建和修改其自己的 WLAN 配置。活动设备所有者拥有编辑或删除任何 WLAN 配置（包括并非由其创建的配置）的权限。
 * 通过添加 Google 帐户下载设备规范控制器：向托管环境以外的设备添加需要通过设备规范控制器 (DPC) 应用管理的 Google 帐户时，帐户添加流程现在会提示用户安装相应的 WPC。在设备初始设置向导中通过 Settings > Accounts 添加帐户时，也会出现此行为。
 * **对特定 [DevicePolicyManager](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html) API 行为的变更：**
    * 调用 [setCameraDisabled()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setCameraDisabled(android.content.ComponentName, boolean)) 方法只会影响调用该方法的用户的相机；从托管配置文件调用它不会影响主用户运行的相机应用。
    * 此外，[setKeyguardDisabledFeatures()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setKeyguardDisabledFeatures(android.content.ComponentName, int)) 方法现在除了可供设备所有者使用外，还可供配置文件所有者使用。
    * 配置文件所有者可设置以下键盘锁限制：
      * [KEYGUARD_DISABLE_TRUST_AGENTS](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#KEYGUARD_DISABLE_TRUST_AGENTS) 和 [KEYGUARD_DISABLE_FINGERPRINT](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#KEYGUARD_DISABLE_FINGERPRINT)，它们影响配置文件上级用户的键盘锁设置。
     * [KEYGUARD_DISABLE_UNREDACTED_NOTIFICATIONS](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#KEYGUARD_DISABLE_UNREDACTED_NOTIFICATIONS)，它只影响应用在托管配置文件中生成的通知。
    * DevicePolicyManager.createAndInitializeUser() 方法和 DevicePolicyManager.createUser() 方法已弃用。
    * 当给定用户的应用在前台运行时，[setScreenCaptureDisabled()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setScreenCaptureDisabled(android.content.ComponentName, boolean)) 方法现在也会屏蔽辅助结构。
    * [EXTRA_PROVISIONING_DEVICE_ADMIN_PACKAGE_CHECKSUM](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#EXTRA_PROVISIONING_DEVICE_ADMIN_PACKAGE_CHECKSUM) 现在默认为 SHA-256。出于向后兼容性考虑，仍然支持 SHA-1，但未来将会取消该支持。
    * [EXTRA_PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#EXTRA_PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM) 现在只接受 SHA-256。
Android 6.0（API 级别 23）中曾经存在的 Device initializer API 现已删除
EXTRA_PROVISIONING_RESET_PROTECTION_PARAMETERS
 已删除，因此 NFC 占位配置无法通过编程解锁受恢复出厂设置保护的设备。
    * 您现在可以使用 [EXTRA_PROVISIONING_ADMIN_EXTRAS_BUNDLE](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#EXTRA_PROVISIONING_ADMIN_EXTRAS_BUNDLE) extra 在对托管设备进行 NFC 配置期间向设备所有者应用传递数据。
    * Android for Work API 针对 M 运行时权限（包括 Work 配置文件、辅助层及其他内容）进行了优化。新增的 [DevicePolicyManager](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html) 权限 API 不会影响 M 之前版本的应用。
    * 当用户退出通过 [ACTION_PROVISION_MANAGED_PROFILE](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#ACTION_PROVISION_MANAGED_PROFILE) 或 [ACTION_PROVISION_MANAGED_DEVICE](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#ACTION_PROVISION_MANAGED_DEVICE) intent 发起的设置流程的同步部分时，系统现在会返回[RESULT_CANCELED](https://developer.android.google.cn/reference/android/app/Activity.html#RESULT_CANCELED) 结果代码。
 * 对其他 API 的变更：
  * 流量消耗：android.app.usage.NetworkUsageStats 类已重命名为 [NetworkStats](https://developer.android.google.cn/reference/android/app/usage/NetworkStats.html)。
 * 对全局设置的变更：
  * 这些设置不再通过 [setGlobalSettings()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setGlobalSetting(android.content.ComponentName, java.lang.String, java.lang.String)) 进行设置：
   * BLUETOOTH_ON
   * DEVELOPMENT_SETTINGS_ENABLED
   * MODE_RINGER
   * NETWORK_PREFERENCE
   * WIFI_ON
  * 这些全局设置现在可通过 [setGlobalSettings()](https://developer.android.google.cn/reference/android/app/admin/DevicePolicyManager.html#setGlobalSetting(android.content.ComponentName, java.lang.String, java.lang.String)) 进行设置：
   * [WIFI_DEVICE_OWNER_CONFIGS_LOCKDOWN](https://developer.android.google.cn/reference/android/provider/Settings.Global.html#WIFI_DEVICE_OWNER_CONFIGS_LOCKDOWN)

 **Android7.0**
 * #### 无障碍改进
为提高平台对于视力不佳或视力受损用户的易用性，Android 7.0 做出了一些更改。这些更改一般并不要求更改您的应用代码，不过您应仔细检查并使用您的应用测试这些功能，以评估它们对用户体验的潜在影响。

 * #### 电池和内存
Android 7.0 包括旨在延长设备电池寿命和减少 RAM 使用的系统行为变更。这些变更可能会影响您的应用访问系统资源，以及您的应用通过特定隐式 intent 与其他应用交互的方式。

* #### 屏幕缩放
Android 7.0 支持用户设置**显示尺寸**，以放大或缩小屏幕上的所有元素，从而提升设备对视力不佳用户的可访问性。用户无法将屏幕缩放至低于最小屏幕宽度 [sw320dp](http://developer.android.google.cn/guide/topics/resources/providing-resources.html)，该宽度是 Nexus 4 的宽度，也是常规中等大小手机的宽度。
![正常效果](http://upload-images.jianshu.io/upload_images/1709375-3f8e0e26c3b965fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![运行 Android 7.0 系统映像的设备增大显示尺寸后的效果。](http://upload-images.jianshu.io/upload_images/1709375-72e539bf7e2962ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 当设备密度发生更改时，系统会以如下方式通知正在运行的应用：
 * 如果是面向 API 级别 23 或更低版本系统的应用，系统会自动终止其所有后台进程。这意味着如果用户切换离开此类应用，转而打开 *Settings* 屏幕并更改 **Display size** 设置，则系统会像处理内存不足的情况一样终止该应用。如果应用具有任何前台进程，则系统会如[处理运行时更改](https://developer.android.google.cn/guide/topics/resources/runtime-changes.html)中所述将配置变更通知给这些进程，就像对待设备屏幕方向变更一样。
 * 如果是面向 Android 7.0 的应用，则其所有进程（前台和后台）都会收到有关配置变更的通知，如[处理运行时更改](https://developer.android.google.cn/guide/topics/resources/runtime-changes.html)中所述。

 大多数应用并不需要进行任何更改即可支持此功能，不过前提是这些应用遵循 Android 最佳做法。具体要检查的事项：
 * 在屏幕宽度为 [sw320dp](https://developer.android.google.cn/guide/topics/resources/providing-resources.html) 的设备上测试您的应用，并确保其充分运行。
 * 当设备配置发生变更时，更新任何与密度相关的缓存信息，例如缓存位图或从网络加载的资源。当应用从暂停状态恢复运行时，检查配置变更。
注：如果您要缓存与配置相关的数据，则最好也包括相关元数据，例如该数据对应的屏幕尺寸或像素密度。保存这些元数据便于您在配置变更后决定是否需要刷新缓存数据。
 * 避免用像素单位指定尺寸，因为像素不会随屏幕密度缩放。应改为使用[与密度无关像素](https://developer.android.google.cn/guide/practices/screens_support.html) (dp) 单位指定尺寸。


* #### NDK 应用链接至平台库
从 Android 7.0 开始，系统将阻止应用动态链接非公开 NDK 库，这种库可能会导致您的应用崩溃。此行为变更旨在为跨平台更新和不同设备提供统一的应用体验。即使您的代码可能不会链接私有库，但您的应用中的第三方静态库可能会这么做。因此，所有开发者都应进行相应检查，确保他们的应用不会在运行 Android 7.0 的设备上崩溃。如果您的应用使用原生代码，则只能使用[公开 NDK API](https://developer.android.google.cn/ndk/guides/stable_apis.html)。

* #### 低电耗模式
Android 6.0（API 级别 23）引入了低电耗模式，当用户设备未插接电源、处于静止状态且屏幕关闭时，该模式会推迟 CPU 和网络活动，从而延长电池寿命。而 Android 7.0 则通过在设备未插接电源且屏幕关闭状态下、但不一定要处于静止状态（例如用户外出时把手持式设备装在口袋里）时应用部分 CPU 和网络限制，进一步增强了低电耗模式。
![图 1. 低电耗模式如何应用第一级系统活动限制以延长电池寿命的图示。](http://upload-images.jianshu.io/upload_images/1709375-b41f24ea26df75d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当设备处于充电状态且屏幕已关闭一定时间后，设备会进入低电耗模式并应用第一部分限制：关闭应用网络访问、推迟作业和同步。如果进入低电耗模式后设备处于静止状态达到一定时间，系统则会对[PowerManager.WakeLock](https://developer.android.google.cn/reference/android/os/PowerManager.WakeLock.html)
、[AlarmManager](https://developer.android.google.cn/reference/android/app/AlarmManager.html) 闹铃、GPS 和 WLAN 扫描应用余下的低电耗模式限制。无论是应用部分还是全部低电耗模式限制，系统都会唤醒设备以提供简短的维护时间窗口，在此窗口期间，应用程序可以访问网络并执行任何被推迟的作业/同步。
![图 2. 低电耗模式如何在设备处于静止状态达到一定时间后应用第二级系统活动限制的图示。](http://upload-images.jianshu.io/upload_images/1709375-f5550e4f0bb40902.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 请注意，激活屏幕或插接设备电源时，系统将退出低电耗模式并移除这些处理限制。此项新增的行为不会影响有关使您的应用适应 Android 6.0（API 级别 23）中所推出的旧版本低电耗模式的建议和最佳做法，如[对低电耗模式和应用待机模式进行针对性优化](https://developer.android.google.cn/training/monitoring-device-state/doze-standby.html)中所讨论。您仍应遵循这些建议（例如使用 Google 云消息传递 (GCM) 发送和接收消息）并开始安排更新计划以适应新增的低电耗模式行为。

* #### Project Svelte：后台优化
Android 7.0 移除了三项隐式广播，以帮助优化内存使用和电量消耗。此项变更很有必要，因为隐式广播会在后台频繁启动已注册侦听这些广播的应用。删除这些广播可以显著提升设备性能和用户体验。
移动设备会经历频繁的连接变更，例如在 WLAN 和移动数据之间切换时。目前，可以通过在应用清单中注册一个接收器来侦听隐式 [CONNECTIVITY_ACTION](https://developer.android.google.cn/reference/android/net/ConnectivityManager.html#CONNECTIVITY_ACTION)
 广播，让应用能够监控这些变更。由于很多应用会注册接收此广播，因此单次网络切换即会导致所有应用被唤醒并同时处理此广播。
同理，在之前版本的 Android 中，应用可以注册接收来自其他应用（例如相机）的隐式 [ACTION_NEW_PICTURE](https://developer.android.google.cn/reference/android/hardware/Camera.html#ACTION_NEW_PICTURE) 和[ACTION_NEW_VIDEO](https://developer.android.google.cn/reference/android/hardware/Camera.html#ACTION_NEW_VIDEO) 广播。当用户使用相机应用拍摄照片时，这些应用即会被唤醒以处理广播。

 为缓解这些问题，Android 7.0 应用了以下优化措施：
  * 面向 Android 7.0 开发的应用不会收到 [CONNECTIVITY_ACTION](https://developer.android.google.cn/reference/android/net/ConnectivityManager.html#CONNECTIVITY_ACTION) 广播，即使它们已有清单条目来请求接受这些事件的通知。在前台运行的应用如果使用 [BroadcastReceiver](https://developer.android.google.cn/reference/android/content/BroadcastReceiver.html) 请求接收通知，则仍可以在主线程中侦听CONNECTIVITY_CHANGE
  * 应用无法发送或接收 [ACTION_NEW_PICTURE](https://developer.android.google.cn/reference/android/hardware/Camera.html#ACTION_NEW_PICTURE) 或 [ACTION_NEW_VIDEO](https://developer.android.google.cn/reference/android/hardware/Camera.html#ACTION_NEW_VIDEO) 广播。此项优化会影响所有应用，而不仅仅是面向 Android 7.0 的应用。

 如果您的应用使用任何 intent，您仍需要尽快移除它们的依赖关系，以正确适配 Android 7.0 设备。Android 框架提供多个解决方案来缓解对这些隐式广播的需求。例如，[JobScheduler](https://developer.android.google.cn/reference/android/app/job/JobScheduler.html) API 提供了一个稳健可靠的机制来安排满足指定条件（例如连入无限流量网络）时所执行的网络操作。您甚至可以使用 [JobScheduler](https://developer.android.google.cn/reference/android/app/job/JobScheduler.html) 来适应内容提供程序变化。如需了解有关 Android N 中后台优化以及如何改写应用的详细信息，请参阅[后台优化](https://developer.android.google.cn/preview/features/background-optimization.html)。


### 更多信息请猛戳下面的官方链接
[Android 6.0 变更](https://developer.android.google.cn/about/versions/marshmallow/android-6.0-changes.html)
[Android 7.0 变更](https://developer.android.google.cn/about/versions/nougat/android-7.0-changes.html)


转载请以链接形式标明出处： 
[http://www.jianshu.com/p/95790125b7f4](http://www.jianshu.com/p/95790125b7f4)
本文出自:[103style](http://www.jianshu.com/u/109656c2d96f)
or
csdn 
[http://blog.csdn.net/lxk_1993/article/details/73784883](http://blog.csdn.net/lxk_1993/article/details/73784883)
本文出自:[lxk_1993](http://blog.csdn.net/lxk_1993) 
