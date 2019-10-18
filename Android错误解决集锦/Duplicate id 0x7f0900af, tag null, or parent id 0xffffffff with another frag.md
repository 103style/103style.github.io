# Duplicate id 0x7f0900af, tag null, or parent id 0xffffffff with another frag 

以下是具体报错信息
```
     Caused by: android.view.InflateException: Binary XML file line #39: Binary XML file line #39: Error inflating class fragment
     Caused by: android.view.InflateException: Binary XML file line #39: Error inflating class fragment
     Caused by: java.lang.IllegalArgumentException: Binary XML file line #39: Duplicate id 0x7f0900ae, tag null, or parent id 0xffffffff with another fragment for com.google.android.gms.maps.SupportMapFragment
        at android.support.v4.app.FragmentManagerImpl.onCreateView(FragmentManager.java:3752)
        at android.support.v4.app.FragmentController.onCreateView(FragmentController.java:120)
        at android.support.v4.app.FragmentActivity.dispatchFragmentsOnCreateView(FragmentActivity.java:405)
        at android.support.v4.app.FragmentActivity.onCreateView(FragmentActivity.java:387)
        at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:780)
        at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:730)
        at android.view.LayoutInflater.rInflate(LayoutInflater.java:863)
        at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:824)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:515)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:423)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:374)
        at android.view.View.inflate(View.java:23418)
        at android.support.v4.app.Fragment.onCreateView(RootFrag.java:94)
        at android.support.v4.app.Fragment.performCreateView(Fragment.java:2439)
        at android.support.v4.app.FragmentManagerImpl.moveToState(FragmentManager.java:1460)
        at android.support.v4.app.FragmentManagerImpl.moveFragmentToExpectedState(FragmentManager.java:1784)
        at android.support.v4.app.FragmentManagerImpl.moveToState(FragmentManager.java:1852)
        at android.support.v4.app.FragmentManagerImpl.dispatchStateChange(FragmentManager.java:3269)
        at android.support.v4.app.FragmentManagerImpl.dispatchPause(FragmentManager.java:3245)
        at android.support.v4.app.FragmentController.dispatchPause(FragmentController.java:234)
        at android.support.v4.app.FragmentActivity.onPause(FragmentActivity.java:476)
        at android.app.Activity.performPause(Activity.java:7217)
        at android.app.Instrumentation.callActivityOnPause(Instrumentation.java:1408)
        at android.app.ActivityThread.performPauseActivityIfNeeded(ActivityThread.java:4020)
        at android.app.ActivityThread.performPauseActivity(ActivityThread.java:3997)
        at android.app.ActivityThread.performPauseActivity(ActivityThread.java:3971)
        at android.app.ActivityThread.handlePauseActivity(ActivityThread.java:3945)
        at android.app.ActivityThread.-wrap15(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1725)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6615)

        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:857)
```
应用中有集成Google Map的功能。

在 fragment 使用的时候用的以下方法
* xml中
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    ...
    <fragment xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/frag_map"
        android:name="com.google.android.gms.maps.SupportMapFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
      ...
</RelativeLayout>
```
* java代码
```
private void initMap() {
    FragmentManager fragmentManager = getFragmentManager();
    if (fragmentManager != null) {
        SupportMapFragment mapFragment = (SupportMapFragment) fragmentManager.findFragmentById(R.id.frag_map);
        if (mapFragment != null) {
            mapFragment.getMapAsync(this);
        }
    }
    fusedLocationClient = LocationServices.getFusedLocationProviderClient(activity);
}
```
在一个fragment中使用时 时没有什么问题的，但是当有几个界面当时都用到这个地图功能时，就会出现上面这个问题。 

复现路径：打开一个有 `SupportMapFragment` 的 `fragment` 界面，能正常加载出地图，切换到顶一个`SupportMapFragment`的 `fragment` 界面,就会出现白屏，然后按返回键就会报上面的错误。

然后在 [Android 4.2的变更信息文档](https://developer.android.google.cn/about/versions/android-4.2.html#NestedFragments) 找到以下信息。fragment 是在 4.2版本添加的。
```
Note: You cannot inflate a layout into a fragment when that layout includes a <fragment>.
Nested fragments are only supported when added to a fragment dynamically.
```

fragment 不能加载一个带有<fragment>标签的xml布局，
嵌套的fragment需要用以下方式来动态的加载。
```
Fragment videoFragment = new VideoPlayerFragment();
FragmentTransaction transaction = getChildFragmentManager().beginTransaction();
transaction.add(R.id.video_fragment, videoFragment).commit();
```
所有文章开头的代码修改为以下就ok了。
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    ...
    <FrameLayout
        android:id="@+id/frag_map"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
      ...
</RelativeLayout>
```
```
private void initMap() {
    FragmentManager fragmentManager = getChildFragmentManager();
    SupportMapFragment mapFragment = new SupportMapFragment();
    fragmentManager.beginTransaction().replace(R.id.frag_map, mapFragment).commit();
    mapFragment.getMapAsync(this);
}
```

### 参考
* [https://blog.csdn.net/rockykou/article/details/53312342](https://blog.csdn.net/rockykou/article/details/53312342)
