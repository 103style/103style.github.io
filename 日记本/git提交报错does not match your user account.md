# git提交报错does not match your user account 

`git` 提交报错 **does not match your user account**

出现这个错误的原因是：

因为修改 `git` 的 `user.name`或`user.email` 然后 `commit` 了代码，然后 `push` 的时候报错。

所以我们修改 `git` 的 `user.name` 和 `user.email` 为要求的值。
```
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

然后执行以下代码，更新 提交 `commit` 代码的 `author`
```
git commit --amend --reset-author
```

以上
