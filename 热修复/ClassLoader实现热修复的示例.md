>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

### 效果图
![修复之前](https://upload-images.jianshu.io/upload_images/1709375-03c6b25a8e141f06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![修复之后](https://upload-images.jianshu.io/upload_images/1709375-e4fad75950e9d208.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---


### 实现思路

主要实现思路主要是：
* 先编写一个有 bug 的程序， 运行安装到手机。

* 修正bug之后，重新 `rebuild`, 然后找到 `app - build - intermediates - dex - debug - mergeProjectDexDebug - out - classes.dex` 移动到 修复包 下载的目录 ， 这里放在 `assets` 目录下，并重命名 `classes.dex` 为 `classes2.dex` 。
![base on AndroidStudio 3.5.1](https://upload-images.jianshu.io/upload_images/1709375-76cc82e3b8f9640e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*  然后点击程序上的 `Move Dex`, 将修正bug之后的dex包 移动到 `android/data/packagename/` 目录下，`在这里目录才有加载dex权限`。

* 然后重启程序，在继承自 `MultiDexApplication` 的 `Application` 中加载对应的 `dex` 文件，获取对应的`dexElements`，然后合并到应用的`dexElements`之前。


---


###  相关代码


[demo 源码下载](https://github.com/103style/HotFixDemo)
[demo.apk download](https://github.com/103style/HotFixDemo/blob/master/apk/demo.apk)



* `MyApplication`
    ```
    public class MyApplication extends MultiDexApplication {
    
        @Override
        protected void attachBaseContext(Context base) {
            super.attachBaseContext(base);
            new FixDemo().loadFixedDex(base);
        }
    }
    ```

*  `MainActivity `
    ```
    public class MainActivity extends AppCompatActivity {
        private TextView bugTv;
        private int i = 10;
        private int a = 0;
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
    
    
            bugTv = findViewById(R.id.bug);
    
            bugTv.setText("Bug： " + i + " / " + a);
    
            findViewById(R.id.bug).setOnClickListener(v -> bugMethod());
            findViewById(R.id.move).setOnClickListener(v -> moveDex("classes2.dex"));
        }
    
        private void bugMethod() {
            Toast.makeText(this, "res = " + i / a, Toast.LENGTH_SHORT).show();
        }
    
    
        private void moveDex(String name) {
            //目录 data/data/packageName/odex
            File fileDir = getDir(FixDemo.DEX_DIR, Context.MODE_PRIVATE);
            AssetManager am = getResources().getAssets();
            try {
                InputStream is = am.open(name);
                String filePath = fileDir.getAbsolutePath() + File.separator + name;
                File file = new File(filePath);
                if (file.exists()) {
                    file.delete();
                }
                FileOutputStream os = new FileOutputStream(filePath);
                int len;
                byte[] buffer = new byte[1024];
                while ((len = is.read(buffer)) != -1) {
                    os.write(buffer, 0, len);
                }
                os.close();
                is.close();
    
                //粘贴完文件
                File f = new File(filePath);
                if (f.exists()) {
                    //文件从sk卡赋值到应用运行目录下，成功则toast提示
                    Toast.makeText(this, "dex移动成功,请重启应用", Toast.LENGTH_SHORT).show();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```

* `FixDemo`
    ```
    public class FixDemo {
    
        public static String DEX_DIR = "odex";
    
        private File odexDir;
    
        /**
         * 开始替换dex
         */
        public void loadFixedDex(Context context) {
            if (null == context) {
                return;
            }
            //遍历所有的修复的dex
            odexDir = context.getDir(DEX_DIR, Context.MODE_PRIVATE);
            File[] listFiles = odexDir.listFiles();
            if (listFiles == null) {
                return;
            }
            HashSet<File> loadedDex = new HashSet<>();
            for (File file : listFiles) {
                if (file.getName().startsWith("classes") || file.getName().endsWith(".dex")) {
                    //先将补丁文件放到一个集合里，然后再进行合并
                    loadedDex.add(file);
                }
            }
            //dex合并
            doDexInject(context, loadedDex);
        }
    
        /**
         * dex 合并
         */
        private void doDexInject(Context appContext, HashSet<File> loadedDex) {
            try {
                String filesDirPath = odexDir.getAbsolutePath() + File.separator + "opt_dex";
                File fopt = new File(filesDirPath);
                if (!fopt.exists()) {
                    fopt.mkdir();
                }
                //1.加载应用程序的dex
                BaseDexClassLoader pathLoader = (BaseDexClassLoader) appContext.getClassLoader();
                for (File dex : loadedDex) {
                    //2.加载指定的修复的dex文件
                    DexClassLoader classLoader = new DexClassLoader(
                            dex.getAbsolutePath(),
                            fopt.getAbsolutePath(),
                            null,
                            pathLoader);
                    //3.合并
                    Object dexObj = getPathList(classLoader);
                    Object pathObj = getPathList(pathLoader);
                    Object mDexElementsList = getDexElements(dexObj);
                    Object pathDexElementsList = getDexElements(pathObj);
                    //将两个list合并为一个
                    Object dexElements = combineArray(mDexElementsList, pathDexElementsList);
                    //重写给PathList里面的Element[] dexElements赋值
                    Object pathList = getPathList(pathLoader);
                    ReflectUtils.setClassFiled(pathList, pathList.getClass(), "dexElements", dexElements);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    
        private Object getPathList(ClassLoader classLoader) {
            try {
                return ReflectUtils.getDeclaredField(classLoader, Class.forName("dalvik.system.BaseDexClassLoader"), "pathList");
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
                return null;
            }
        }
    
        private Object getDexElements(Object obj) {
            return ReflectUtils.getClassField(obj, "dexElements");
        }
    
        /**
         * 合并数组
         */
        private Object combineArray(Object arrayLbs, Object arrayRhs) {
            Class localClass = arrayLbs.getClass().getComponentType();
            int i = Array.getLength(arrayLbs);
            int j = i + Array.getLength(arrayRhs);
            Object result = Array.newInstance(localClass, j);
            for (int k = 0; k < j; ++k) {
                if (k < i) {
                    Array.set(result, k, Array.get(arrayLbs, k));
                } else {
                    Array.set(result, k, Array.get(arrayRhs, k - i));
                }
            }
            return result;
        }
    
    }
    ```

---

以上
