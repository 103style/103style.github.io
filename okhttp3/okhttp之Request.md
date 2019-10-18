# okhttp之Request 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

---

**base on `3.12.0`**

----

### 目录
* **Request 简介**
* **Request相关的配置方法**
* **Header介绍**

---

### Request 简介
`Request`即我们构建的每一个`HTTP`请求。通过配置请求的 **地址**、**http方法**、**请求头** 等信息。

使用方法：
```
Request request = new Request.Builder()
        .url("https://publicobject.com/helloworld.txt")//指定请求地址
        .delete()//指定请求的方法
        .build();
```

`Request`成员变量和构造方法：
```
public final class Request {
    final HttpUrl url;//请求地址
    final String method;//请求方法
    final Headers headers;//头信息
    final @Nullable RequestBody body;//请求体
    //当前请求的标签
    final Map<Class<?>, Object> tags;
    // 头信息的Cache-Control
    private volatile @Nullable CacheControl cacheControl; 

    Request(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = builder.headers.build();
        this.body = builder.body;
        this.tags = Util.immutableMap(builder.tags);
    }
    ...
}
```

###  Request相关的配置方法

默认的配置，默认为`get`请求.
```
public Builder() {
    this.method = "GET";
    this.headers = new Headers.Builder();
}
```

* **设置请求地址 `.url(xxx)`**
  ```
    public Builder url(HttpUrl url) {
        if (url == null) throw new NullPointerException("url == null");
        this.url = url;
        return this;
    }
    public Builder url(String url) {
        if (url == null) throw new NullPointerException("url == null");

        // Silently replace web socket URLs with HTTP URLs.
        if (url.regionMatches(true, 0, "ws:", 0, 3)) {
            url = "http:" + url.substring(3);
        } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
            url = "https:" + url.substring(4);
        }

        return url(HttpUrl.get(url));
    }
    public Builder url(URL url) {
        if (url == null) throw new NullPointerException("url == null");
        return url(HttpUrl.get(url.toString()));
    }
  ```
* **设置http请求方法**
    * `.get()`
    * `.head()`
    * `.post(RequestBody body)`
    * `.delete(@Nullable RequestBody body)`
    * `.delete()`
    * `.put(RequestBody body)`
    * `.patch(RequestBody body)`
    * `.method(String method, @Nullable RequestBody body)`

* **设置http请求的标签**
  * `tag(@Nullable Object tag)`
  * `tag(Class<? super T> type, @Nullable T tag)`

* **设置http请求header中的Cache-Contro**
    * `cacheControl(CacheControl cacheControl)`
    设置此请求的`Cache-Contro`，替换所有已经存在的缓存控制标头。 如果为`null`，这将清除此请求的缓存控制标头。

---

### Header介绍
`Header` 即 `http`请求的头信息，通过以下代码我们知道`header`是由一个 **字符串数组** 组成的。
```
public final class Headers {
  private final String[] namesAndValues;
  Headers(Builder builder) {
    this.namesAndValues = builder.namesAndValues.toArray(new String[builder.namesAndValues.size()]);
  }
  private Headers(String[] namesAndValues) {
    this.namesAndValues = namesAndValues;
  }
  ...
}
```

* **添加头信息的方法:**
    我们通过`.add("Connection:Keep-Alive")`实际上时是吧`Connection:Keep`和`Keep-Alive`依次添加到这个保存头信息的字符串数组中。
    ```
    public Builder add(String line) {
        int index = line.indexOf(":");
        if (index == -1) {
            throw new IllegalArgumentException("Unexpected header: " + line);
        }
        return add(line.substring(0, index).trim(), line.substring(index + 1));
    }
    public Builder add(String name, String value) {
        checkName(name);
        checkValue(value, name);
        return addLenient(name, value);
    }
    Builder addLenient(String name, String value) {
        namesAndValues.add(name);
        namesAndValues.add(value.trim());
        return this;
    }
    ```

* **获取头信息的方法:**
    通过比较单数位的值，来获取对应索引的下一个索引的值。
    ```
    public String get(String name) {
        for (int i = namesAndValues.size() - 2; i >= 0; i -= 2) {
            if (name.equalsIgnoreCase(namesAndValues.get(i))) {
                return namesAndValues.get(i + 1);
            }
        }
        return null;
    }
    ```

* **删除头信息的方法:**
    依次删除单数位的值，相同此索引和下一索引的值。
    ```
    public Builder removeAll(String name) {
        for (int i = 0; i < namesAndValues.size(); i += 2) {
            if (name.equalsIgnoreCase(namesAndValues.get(i))) {
                namesAndValues.remove(i); // name
                namesAndValues.remove(i); // value
                i -= 2;
            }
        }
        return this;
    }
    ```

* **修改头信息的方法:**
    先删除对应的头信息，再添加进数组。
    ```
    public Builder set(String name, String value) {
          (name);
        checkValue(value, name);
        removeAll(name);
        addLenient(name, value);
        return this;
    }
    ```

--- 

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
