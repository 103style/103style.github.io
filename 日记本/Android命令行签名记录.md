
 **jarsigner -verbose -keystore 签名文件的全路径 -signedjar 签名之后的文件路径 未签名的apk路径  签名文件别名** 

例：
```
jarsigner -verbose -keystore D:/test.keystore -signedjar D:/singed.apk D:/unsingn.apk  testAlias
```


