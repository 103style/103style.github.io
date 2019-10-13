
`onCreateView` 的时候
用 
`LayoutInflater.from(mContext).inflate(layoutRes, parent, false);`
替换
`LayoutInflater.from(mContext).inflate(layoutRes, null);`

用第二种 会导致 布局不能撑满屏幕 , [具体原因](https://www.jianshu.com/p/9a6db88b8ad3)
