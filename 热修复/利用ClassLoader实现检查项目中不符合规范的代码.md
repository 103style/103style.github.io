# 利用ClassLoader实现检查项目中不符合规范的代码 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


主要实现思路主要是：
* 通过 `context.getClassLoader()`或者 `Thread.currentThread().getContextClassLoader()`获取`ClassLoader`对象。
* 通过反射获取私有变量`pathList`.
* 通过反射获取`pathList`的私有变量 数组 `dexElements` 
* 通过反获取每一个 `dexElements`的私有变量`dexFile`，获得`DexFile`的集合
*  然后通过`DexFile.entries()`获取`dexFile`的所有 指定包名 的类文件
* 然后对文件进行 代码规范的规则校验。



[demo github项目地址](https://github.com/103style/CodeCheckDemo)

以下贴下主要的代码：

* `MainActivity`中的点击校验方法： 
    ```
    private void startCodeCheck() {
        // 获取 BaseDexClassLoader 中 pathList 变量的 dexElements 变量
        Object dexElements = CodeCheckUtils.getDexElements();

        // 获取当前classloader中的DexFile列表
        ArrayList<DexFile> dexFileList = CodeCheckUtils.getDexFileList(dexElements);

        // 获取 DexFile 列表中  含有 packageName 的类
        List<Class> pkgClass = CodeCheckUtils.getPackageClass(dexFileList, getPackageName());

        //获取不符合规则的类
        List<String> illegalList = getIllegalClass(pkgClass);
        if (illegalList == null || illegalList.size() == 0) {
            return;
        }

        //弹框提示修改
        showIllegalClassDialog(illegalList);
    }
    ```

* `demo`的 类校验规则
    ```
    private List<String> getIllegalClass(List<Class> pkgClass) {
        if (pkgClass == null) {
            LogUtils.e(TAG, "pkgClass is null");
            return null;
        }
        List<String> illegalList = new ArrayList<>();

        // TODO: 2019/9/18  这里的检查规则 请根据具体的规则修改
        for (Class itemClass : pkgClass) {
            String name = itemClass.getName();
            LogUtils.d(TAG, name);
            String simpleName = itemClass.getSimpleName();
            if (isSystemOrInnerClass(name, simpleName)) {
                continue;
            }
            name = name.replace(getPackageName(), "");
            String[] arr = name.split("\\.");
            //文件直接在包目录下 子目录下包含文件夹  不合法
            if (arr.length != 3) {
                illegalList.add(simpleName);
                continue;
            }

            if (!arr[2].toLowerCase().endsWith(arr[1].toLowerCase())) {
                illegalList.add(simpleName);
            }
        }
        return illegalList;
    }

    /**
     * 是否是系统生成的类 或者 内部类
     */
    private boolean isSystemOrInnerClass(String name, String simpleName) {
        return name.contains("R$") //资源文件
                | "R".equals(simpleName)//R文件
                | "BuildConfig".equals(simpleName)//配置文件
                | name.contains("$");//内部类
    }
    ```

* `CodeCheckUtils`
    ```
    public class CodeCheckUtils {
    
        private static final String TAG = "CodeCheckUtils";
    
        /**
         * 获取 BaseDexClassLoader 中 pathList 变量的 dexElements 变量
         */
        public static Object getDexElements() {
            ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
            if (classLoader == null) {
                LogUtils.e(TAG, "classLoader is null");
                return null;
            }
            LogUtils.d(TAG, "classLoader = " + classLoader.getClass().getName());
            if (!(classLoader instanceof BaseDexClassLoader)) {
                LogUtils.e(TAG, "classLoader not instanceof BaseDexClassLoader");
                return null;
            }
            BaseDexClassLoader baseDexClassLoader = (BaseDexClassLoader) classLoader;
            Class superclass = baseDexClassLoader.getClass().getSuperclass();
            if (superclass == null) {
                LogUtils.e(TAG, "baseDexClassLoader's  superclass is null");
                return null;
            }
            LogUtils.d(TAG, "baseDexClassLoader's  superclass = " + superclass.getName());
    
            Object declaredField = ReflectUtils.getDeclaredField(baseDexClassLoader, superclass, "pathList");
            if (declaredField == null) {
                LogUtils.e(TAG, "get pathList is null");
                return null;
            }
            return ReflectUtils.getDeclaredField(declaredField, "dexElements");
        }
    
        /**
         * 获取当前classloader中的DexFile列表
         */
        public static ArrayList<DexFile> getDexFileList(Object dexElements) {
            if (dexElements == null) {
                LogUtils.e(TAG, "get dexElements is null");
                return null;
            }
            int length = 0;
            try {
                length = Array.getLength(dexElements);
            } catch (Exception e) {
                LogUtils.e(TAG, "get dexElements length error");
                e.printStackTrace();
            }
    
            if (length == 0) {
                LogUtils.e(TAG, "dexElements length is 0");
                return null;
            }
    
            ArrayList<DexFile> dexFileList = new ArrayList<>();
            Field dexFileField = null;
            for (int i = 0; i < length; i++) {
                Object temp = Array.get(dexElements, i);
                if (dexFileField == null) {
                    dexFileField = ReflectUtils.getClassField(temp, "dexFile");
                    if (dexFileField == null) {
                        continue;
                    }
                }
                Object object = ReflectUtils.getClassFieldValue(temp, dexFileField);
                if (object == null) {
                    return null;
                }
                if (object instanceof DexFile) {
                    dexFileList.add((DexFile) object);
                } else {
                    LogUtils.e(TAG, "");
                }
            }
            return dexFileList;
        }
    
        /**
         * 返回 DexFile 列表中  含有 packageName 的类
         *
         * @param dexFileList DexFile 列表
         * @param packageName 返回包含此包名的类
         */
        public static List<Class> getPackageClass(ArrayList<DexFile> dexFileList, String packageName) {
            if (dexFileList == null || dexFileList.size() == 0) {
                LogUtils.e(TAG, "dexFileList is empty");
                return null;
            }
            List<Class> pckClass = new ArrayList<>();
            for (DexFile dexFile : dexFileList) {
                Enumeration<String> enumeration = dexFile.entries();
                if (enumeration == null) {
                    continue;
                }
                while (enumeration.hasMoreElements()) {
                    String element = enumeration.nextElement();
                    if (element.contains(packageName)) {
                        try {
                            pckClass.add(Class.forName(element));
                        } catch (ClassNotFoundException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
            return pckClass;
        }
    }
    ```


[demo 源码下载](https://github.com/103style/CodeCheckDemo)


效果图：
![效果图](https://upload-images.jianshu.io/upload_images/1709375-0467fcaae09177bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


以上
