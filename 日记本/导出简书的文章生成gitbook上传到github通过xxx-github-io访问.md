>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

### 目录
* GitBook相关的安装
* 导出简书的文章
* 配置GitBook工程
  * 创建 gitbookdemo 工程
  * 运行代码创建修改SUMMARY.md
  * 运行代码创建为每个文件夹创建  README.md
  * 为简书下载的文件添加标题
  * 修改工程的README.md
  * 添加相关的插件
  * 编译gitbook工程
* 创建github项目
* 上传文件到github


---


先放上我的 **gitbook**地址： [https://103style.github.io](https://103style.github.io)

![blog 示例](https://upload-images.jianshu.io/upload_images/1709375-1ca295459a856832.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---

### GitBook相关的安装
可以直接 `win+R → cmd`  或者可以先安装 [git](https://git-scm.com/downloads) ，后面也要用到, 通过 `git bash` 进行命令行操作。 
* 安装 **Node.js**
  **GitBook** 是一个基于 Node.js 的命令行工具，下载安装 [Node.js](https://link.jianshu.com/?t=https%3A%2F%2Fnodejs.org%2Fen)，安装完成之后，你可以使用下面的命令来检验是否安装成功。
  ```
  $ node -v
  v7.7.1
  ```

* 安装 **GitBook**
  输入下面的命令来安装 GitBook。
  ```
  npm install gitbook-cli -g
  ```
  安装完成之后，你可以使用下面的命令来检验是否安装成功。
  ```
  $ gitbook -V
  CLI version: 2.3.2
  GitBook version: 3.2.3
  ```

---

### 导出简书的所有文章
登录简书之后， 访问 [https://www.jianshu.com/settings/misc](https://www.jianshu.com/settings/misc) 就可以打包下载所有文章了。
或者 按如下操作：
![导出简书文章1](https://upload-images.jianshu.io/upload_images/1709375-30a4af28fe93345d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![导出简书文章2](https://upload-images.jianshu.io/upload_images/1709375-4a519bcb88bc47fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

###  配置GitBook工程
以 **windows 10** 为例.
* **创建 `gitbookdemo` 工程**
   创建 `gitbookdemo` 文件夹，然后定位到 `gitbookdemo` 目录，运行以下命令。
   以 `git bash` 和 `cmd` 窗口为例：
   * `git bash`窗口
    直接在`gitbookdemo` 文件夹右键，选择  `git bash` 即可。
    ![git bash窗口](https://upload-images.jianshu.io/upload_images/1709375-0d63a6f609ef5fb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   * `cmd`命令窗口
    `Win + R → cmd`  或者 在`gitbookdemo` 文件夹按 **shirt + 鼠标右键**，选择 `Powershell` 。
    ![cmd命令窗口](https://upload-images.jianshu.io/upload_images/1709375-abaf61c25f87b4fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

* 将下载的简书导出的文件复制到 `gitbookdemo` 文件夹。
   示例如下：
   ![示例](https://upload-images.jianshu.io/upload_images/1709375-65eced8c92914276.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

如果文章不多的可以直接手动编辑 `SUMMARY.md`，多的话可以看下面的代码。
下面示例中的 `AndroidStudio问题记录` 和 `Java并发编程的艺术笔记` 为对应文件夹的标题，二级目录则为对应的文章的相对连接。
```
* [Introduction](README.md)

* [文件夹名字](文件夹名字/README.md)
 * [文件夹下的文章名字](文件夹名字/文件夹下的文章带格式的名字)

* [Java并发编程的艺术笔记](Java并发编程的艺术笔记/README.md)
 * [Executor框架](Java并发编程的艺术笔记/Executor框架.md)
 * [Java Thread.join()详解](Java并发编程的艺术笔记/Java Thread.join()详解.md)

```

---

*  **运行代码创建修改`SUMMARY.md`**
    不过生成的文件由于排序的原因，可能需要手动调整下`SUMMARY.md`顺序。
    ```
    public static String TITLE = "* [%s](%s/README.md)\n";
    public static String FILE = " * [%s](%s/%s)\n";
    public static void main(String[] args) throws Exception {
        String path = "D:/gitbookdemo/";
        FileOutputStream fos = new FileOutputStream(path + "SUMMARY.md", true);
        File file = new File(path);
        String[] lists = file.list();
        for (String list : lists) {
            String s = path + list;
            if (new File(s).isDirectory()
                    && !list.startsWith(".")
                    && !list.startsWith("node_modules")
                    && !list.startsWith("_")
            ) {
                fos.write(String.format(TITLE, list, list).getBytes());
                File dir = new File(s);
                String[] dirFiles = dir.list();
                for (String f : dirFiles) {
                    if (f.startsWith("README")) {
                        continue;
                    }
                    String name = f.replace(".md", "");
                    fos.write(String.format(FILE, name, list, f).getBytes());
                }
            }
            fos.write("\n".getBytes());
        }
        fos.close();
    }
    ```  
    `SUMMARY.md`如下：
    ![SUMMARY.md](https://upload-images.jianshu.io/upload_images/1709375-7511bd1cddbe9789.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

* **运行代码创建为每个文件夹创建  `README.md`**
  为每个文件夹创建对应的 `README.md`
    ```
    public static void main(String[] args) throws Exception {
        String path = "D:/gitbookdemo/";
        writeReadme(path);
    }
    private static void writeReadme(String path) throws Exception {
        File file = new File(path);
        String[] lists = file.list();
        for (String list : lists) {
            String s = path + list;
            if (new File(s).isDirectory()
                    && !list.startsWith(".")
                    && !list.startsWith("node_modules")
                    && !list.startsWith("_")
            ) {
                File t = new File(s + "/" + "README.md");
                if (!t.exists()) {
                    t.createNewFile();
                }
                FileOutputStream fos = new FileOutputStream(t);
                fos.write(("# " + list + "\n").getBytes());
                String[] files = new File(s).list();
                for (String f : files) {
                    if (f.startsWith("README")) {
                        continue;
                    }
                    fos.write(String.format("* [%s](%s) \n", f.replace(".md", ""), f).getBytes());
                }
                fos.close();
            }
        }
    }
    ```

---

* **为简书下载的文件添加标题**
   由于下载的 `md` 文件里面默认没带标题，所以执行如下代码添加标题。
   其中 `byte[] buffer = new byte[60*1024];`  是 根据 **最大文件的大小** 设置的缓存。
    ```
    public static void main(String[] args) throws Exception {
        addTitle();
    }
    private static void addTitle() throws Exception {
        String path = "D:/gitbookdemo/";
        File file = new File(path);
        String[] lists = file.list();
        for (String list : lists) {
            String s = path + list;
            if (new File(s).isDirectory()
                    && !list.startsWith(".")
                    && !list.startsWith("node_modules")
                    && !list.startsWith("_")
            ) {
                File dir = new File(s);
                String[] dirFiles = dir.list();
                for (String f : dirFiles) {
                    if (f.startsWith("README")) {
                        continue;
                    }
                    FileInputStream fis = new FileInputStream(s + "/" + f);
                    byte[] buffer = new byte[60*1024];//根据最大文件设置缓存
                      fis.read(buffer);
                    FileOutputStream fos = new FileOutputStream(s + "/" + f);
                    String title = f.replace(".md", "");
                    fos.write(String.format("# %s \n\n", title).getBytes());
                    for (int i = 0; i < buffer.length; i++) {
                        if (buffer[i] == 0){
                            break;
                        }
                        fos.write(buffer[i]);
                    }
                    fis.close();
                    fos.close();
                }
            }
        }
    }
    ```

---

* **修改工程的README `D:\gitbookdemo\README.md`**
   这里是 打开连接加载的首页，可以自行编写。
   示例：
  ```
  # 103style's blog

  * 博文分类
    * [数据结构](数据结构/README.md)
    * [NDK](NDK/README.md)
    * [OpenGL ES 3.0](OpenGL ES 3.0/README.md)
    * [Android通用类](Android通用类/README.md)
  
  * 相关连接
    * [github](https://github.com/103style)
    * [简书](https://www.jianshu.com/u/109656c2d96f)
    * [CSDN](https://blog.csdn.net/lxk_1993)
    * [微博: 103style](https://weibo.com/3250756185/profile)
  ```

---

* **添加相关的插件**，具体可以查看文末的参考文章，也可以跳过直接下一步.
  在`gitbookdemo` 下面新建 `book.json`，只需要修改 `title`、`author`、`description` 即可，`gitbook`版本修改为上面安装时 `gitbook -V` 显示的对应的版本，然后运行 `gitbook install`。
```
{
    "title": "103style",
    "author": "103style",
    "description": "103style的blog",
    "language": "zh-hans",
    "gitbook": "3.2.3",
    "structure": {
        "readme": "README.md"
    },
	"plugins": [
        "splitter",
        "expandable-chapters-small",
        "anchors",
        "anchor-navigation-ex"
	],
	"pluginsConfig": {
	  "anchor-navigation-ex": {
            "showLevel": false
        }
	}
}
```
![示例](https://upload-images.jianshu.io/upload_images/1709375-81fb3eb53cd6f9a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

* **编译gitbook工程**
  通过 `gitbook build` 生成生成 `_book`目录，双击 `_book`目录下的 `index.html`即可在本地浏览器预览效果。
    `可选：` 然后运行下面代码删除之前的 `***.md` 源文件。
    ```
    public static void main(String[] args) throws Exception {
        removeMD();
    }
    private static void removeMD() {
        String path = "D:/gitbookdemo/_book";
        File file = new File(path);
        String[] lists = file.list();
        for (String list : lists) {
            if (list.startsWith(".")
                    || list.startsWith("sear")
                    || list.startsWith("index")
                    || list.startsWith("gitbook")
                    || list.startsWith("package")
            ) {
                continue;
            }
            String[] fs = new File(path + list).list();
            for (String f : fs) {
                if (f.endsWith(".md")) {
                    new File(path + list + "/" + f).delete();
                }
            }
        }
    }
    ```
![预览效果](https://upload-images.jianshu.io/upload_images/1709375-a4be17275596d923.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

### 在github创建项目
没有账号的，可以先注册一个: [注册 github](https://github.com/join?source=header-home)
注册好了之后，点击 `New` 创建项目。
![创建项目](https://upload-images.jianshu.io/upload_images/1709375-6b4827eeecfbc15f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

项目为 **用户名.github.io**， 用户名几位前面那个 **Owner**， 我的用户名是 `103style`, 所以工程名是 `103style.github.io`.
![工程命名示例](https://upload-images.jianshu.io/upload_images/1709375-70a7ea0dea5f81ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

### 上传文件到github
安装 [git](https://git-scm.com/downloads).
安装完成之后，打开上一步创建的工程, **用户名.github.io**,  点击 **Clone or download**， 复制里面的那个链接，`eq: https://github.com/103style/103style.git`
![image.png](https://upload-images.jianshu.io/upload_images/1709375-90c7366803d38472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后在 `gitbookdemo` 目录下, 鼠标右键，点击 `git bash`，
执行命令： `git clone 复制的地址` ，
例如:
```
git clone  https://github.com/103style/103style.git
```
然后  `gitbookdemo`  下，会生成一个 **用户名.github.io** 的文件夹，我的是 `103style.github.io`.


然后将 `gitbookdemo/_book` 目录下的文件全部复制到 **用户名.github.io** 文件夹下。

然后在 `用户名.github.io` 目录下, 鼠标右键，点击 `git bash`，
依次执行命令：
```
git add .

git commit -m "init"

git push orign master:master
```

然后 大功告成。

打开浏览器， 在网址上输入  `https://用户名.github.io`  即可， 例如： `https://103style.github.io`。


---

### 参考文章
* [GitBook 使用教程](https://www.jianshu.com/p/421cc442f06c)

---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
