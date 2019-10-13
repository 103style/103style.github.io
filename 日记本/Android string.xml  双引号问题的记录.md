我们知道 在`string.xml`中加了 `"` ，但是跑到手机上时不显示引号，我们知道原因是没有加 `\` 进行转译，加上转译符号就好了。


然后就是  当字符串中有 `"`、`...`之类的符号时， `AndroidStudio` 会让这个字符串变黄并提示你要改成 `xxx`， `"`对应的提示就是`&quot;`，开始以为改成`&quot;`之后效果会自动对 `"` 进行转译，然后实际上并没有。


看以下5个字符串显示的对比。
```
<string name="hello1">"Hello1"</string>
<string name="hello2">\"Hello2\"</string>
<string name="hello3">&quot;Hello3&quot;</string>
<string name="hello4">\&quot;Hello4&quot;</string>
<string name="hello5">\"Hello5\&quot;</string>
```
![预览效果](https://upload-images.jianshu.io/upload_images/1709375-d89ff9463671062b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上
