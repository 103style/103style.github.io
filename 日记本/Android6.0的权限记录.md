之前项目对于权限问题的解决方法就是把targetSdkVersion设置为22，而不是25。

对于有强迫症的我，是无法忍受IDE自检出来的黄色警告和提示的，所以直接把targetSdkVersion提到25，compileSdkVersion也提到25，buildToolsVersion也提到25.0.2，v7包，design包也都升到25.3.1。所以随之而来的就要处理对于6.0+的系统的权限适配。

下面是Activity和Fragment中申请权限的代码记录

Activity 以相机和录音权限为例

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) 
        != PackageManager.PERMISSION_GRANTED &&
    ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO)
        != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO}, 0x110);
        } else {
           //已经获取到权限
           //调用接下来的方法 
        }
    } else {
        //6.0之下的机器 相当manifest申明了权限几个
        //调用接下来的方法
    }


    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == 0x110 && grantResults != null && grantResults.length > 0) {
            //获取到权限
            if (checkPermission(grantResults)) {
                //调用接下来的方法       
            } else {
                //用户勾选了不再询问
                //提示用户手动打开权限   
                if (!ActivityCompat.shouldShowRequestPermissionRationale(
            this, Manifest.permission.CAMERA)) {
                    ToastUtils.showShort("相机权限已被禁止，请在应用管理中打开权限");
                } else {
                    ToastUtils.showShort("没有相机权限");
                }

              if (!ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.RECORD_AUDIO)) {
                  ToastUtils.showShort("语音权限已被禁止，请在应用管理中打开权限");
                } else {
                    ToastUtils.showShort("没有语音权限");
                }
            }
        }
    }

    public boolean checkPermission(int[] grantResults) {
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


Fragment 以相机和录音权限为例

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        if (ContextCompat.checkSelfPermission((Context) mContext, Manifest.permission.CAMERA)
                            != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(new String[]{Manifest.permission.CAMERA}, 0x111);
         }
    }


    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
         if (requestCode == 0x110) {
             if (grantResults[0] != PackageManager.PERMISSION_GRANTED) {
                 if (!ActivityCompat.shouldShowRequestPermissionRationale(activity, Manifest.permission.CAMERA)) {
                      ToastUtils.showShort("相机权限已被禁止，请在应用管理中打开权限");
                  } else {
                      ToastUtils.showShort("没有相机权限");
                  }
              } else {
                //调用接下来的方法
              }
          }
    }



2017-05-09
补上写的权限工具类

    public class PermissionUtils {
        /**
         * 检查申请的权限是否全部允许
         */
        public static boolean checkPermission(int[] grantResults) {
            if (grantResults == null || grantResults.length == 0) {
                return true;
            } else {
                int temp = 0;
                for (int i : grantResults) {
                if (i != PackageManager.PERMISSION_GRANTED) {
                    temp++;
                }
            }
            return temp == grantResults.length;
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
     * @param context     上下文
     * @param activity    activity
     * @param permissions 权限数组
     * @param requestCode 申请码
     * @return true 有权限  false 无权限
     */
    public static boolean checkAndApplyfPermissionActivity(Context context, Activity activity, String[] permissions, int requestCode) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            permissions = checkPermissions(context, permissions);
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
    public static boolean checkAndApplyfPermissionFragment(Context context, Fragment mFragment, String[] permissions, int requestCode) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            permissions = checkPermissions(context, permissions);
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
    //        return permissionLists.toArray(new String[permissionLists.size()]);
            return temps;
        }
    }

Activity中修改为

     if (PermissionUtils.checkAndApplyfPermissionActivity(mContext, this,
                    new String[]{Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO, Manifest.permission.WRITE_EXTERNAL_STORAGE}, REQUESRCARMEA)) {
                livePrecheck();
            }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == REQUESRCARMEA && grantResults.length > 0) {
            //获取到权限
            if (PermissionUtils.checkPermission(grantResults)) {
                livePrecheck();
            } else {
                //用户勾选了不再询问
                //提示用户手动打开权限
                PermissionUtils.showPermissionsToast(this, permissions);
            }
        }
    }


Fragment中修改为

    if (PermissionUtils.checkAndApplyfPermissionActivity(mContext, mActivity,
                    new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, IntentExtra.REQUESRALBUM)) {
      //获取到权限
    }

     @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
       if (PermissionUtils.checkPermission(grantResults)) {
              switch (requestCode) {
                  case IntentExtra.REQUESRALBUM:
                    //获取到权限
                      break;
              }
        } else {
            PermissionUtils.showPermissionsToast(activity, permissions);
        }
    }


有好的建议忘留言提示
