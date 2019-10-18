# RxJava之连接操作符介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

#### 连接相关的操作符 以及 官方介绍
`RxJava` 之 **连接操作符** 官方介绍 ：[Connectable Observable Operators
](https://github.com/ReactiveX/RxJava/wiki/Connectable-Observable-Operators)
*   [**`ConnectableObservable.connect( )`**](http://reactivex.io/documentation/operators/connect.html) 
    >instructs a Connectable Observable to begin emitting items
    指示`Connectable Observable`开始发出项目
*   [**`Observable.publish( )`**](http://reactivex.io/documentation/operators/publish.html)
    >represents an Observable as a Connectable Observable
    将`Observable`表示为可连接的`Observable`
*   [**`Observable.replay( )`**](http://reactivex.io/documentation/operators/replay.html)
    >ensures that all Subscribers see the same sequence of emitted items, even if they subscribe after the Observable begins emitting the items
    确保所有订阅者都看到相同的发射项目序列，即使他们在`Observable`开始发布项目后订阅
*   [**`ConnectableObservable.refCount( )`**](http://reactivex.io/documentation/operators/refcount.html)
    >makes a Connectable Observable behave like an ordinary Observable
    使`Connectable Observable`的行为类似于普通的`Observable`

---


### 示例：

##### 非连接操作
```
ConnectableObservable firstMillion = Observable.range(1, 1000000)
        .sample(7, TimeUnit.MILLISECONDS)
        .publish();

firstMillion.subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #1:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #1 complete");
    }
});

firstMillion.subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #2:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #2 complete");
    }
});

```
输出：
```
Subscriber #1:391999
Sequence #1 complete
Subscriber #2:556663
Sequence #2 complete
```

---

##### publish and connect
官方示例：
```
ConnectableObservable firstMillion = Observable.range(1, 1000000)
        .sample(7, TimeUnit.MILLISECONDS)
        .publish();

firstMillion.subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #1:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #1 complete");
    }
});

firstMillion.subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #2:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #2 complete");
    }
});

firstMillion.connect();
```
输出：
```
Subscriber #1:984513
Subscriber #2:984513
Sequence #1 complete
Sequence #2 complete
```

---

##### publish and refCount
官方示例：
```
ConnectableObservable firstMillion = Observable.range(1, 1000000)
        .sample(7, TimeUnit.MILLISECONDS)
        .publish();

firstMillion.refCount().subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #1:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #1 complete");
    }
});

firstMillion.refCount().subscribe(new Consumer() {
    @Override
    public void accept(Object it) throws Exception {
        System.out.println("Subscriber #2:" + it);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable it) throws Exception {
        System.out.println("Error: " + it.getMessage());
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("Sequence #2 complete");
    }
});
```
输出：
```
Subscriber #1:438899
Sequence #1 complete
Subscriber #2:684698
Sequence #2 complete
```

---


以上
