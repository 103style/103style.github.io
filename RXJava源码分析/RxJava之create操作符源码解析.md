
>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

本文基于 `RxJava 2.x` 版本

##### create操作符例子：
```
Observable
        .create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter) throws Exception {

            }
        })
        .subscribe(new Observer<Object>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(Object o) {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
```

---


**首先我们看`create` 方法:**
```
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
`RxJavaPlugins` 类的 `onAssembly` 方法：
```
static volatile Function<? super Observable, ? extends Observable> onObservableAssembly;

public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```
在源码中查看引用可知 `onObservableAssembly` 只有在测试的时候才不为 `null`。
**所以`Observable.create(ObservableOnSubscribe<T> source)`实际上就是返回了 `ObservableCreate`对象**

---


`ObservableCreate` 类，可以看到 `ObservableCreate` 是 `Observable` 的子类，并实现了父类的 `subscribeActual` 方法。
```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {...}
    ...
}
```

---

**然后我们看`subscribe`方法:   实际上是调用了 `Observable` 的抽象方法 `subscribeActual(observer);`**
```
public final void subscribe(Observer<? super T> observer) {
    ...
    subscribeActual(observer);
    ...
}

protected abstract void subscribeActual(Observer<? super T> observer);
```
又因为  `create`操作符返回的 `ObservableCreate` 是 `Observable` 的子类，
**所以实际上调用的是`ObservableCreate` 的 `subscribeActual(observer);`**

具体可阅读 [Observable subscribe流程介绍](https://www.jianshu.com/p/21afa541fa98)

---

`ObservableCreate` 的 `subscribeActual(observer)`方法： 

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
* 首先创建了 `CreateEmitter`对象，
* 然后调用了 `subscribe` 方法传进来的 `Observer` 对象的 `onSubscribe()` 方法
* 然后调用了`create` 操作符 传进来的 `ObservableOnSubscribe` 对象的 `subscribe(ObservableEmitter<T> emitter)`方法

---


因为 `CreateEmitter` 类实现了 `ObservableEmitter<T>` 和 `Disposable` 接口，
所以我们可以在 `create` 操作符 传进来的 `ObservableOnSubscribe` 对象的 `subscribe(ObservableEmitter<T> emitter)`方法里调用`onNext`、`onError`、`onComplete`等方法。
```
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {
    ...
    final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }
    ...
}
```
`ObservableEmitter` 接口：
```
public interface ObservableEmitter<T> extends Emitter<T> {
    void setDisposable(@Nullable Disposable d);
    void setCancellable(@Nullable Cancellable c);
    boolean isDisposed();
    ObservableEmitter<T> serialize();
    boolean tryOnError(@NonNull Throwable t);
}

public interface Emitter<T> {
    void onNext(@NonNull T value);
    void onError(@NonNull Throwable error);
    void onComplete();
}
```
`Disposable` 接口：
```
public interface Disposable {
    void dispose();
    boolean isDisposed();
}
```

---


因为`CreateEmitter` 又重写了`onNext`、`onError`、`onComplete`等方法。
**所以 `create` 操作符 传进来的 `ObservableOnSubscribe` 对象的 `subscribe(ObservableEmitter<T> emitter)`方法里调用`onNext`、`onError`、`onComplete`等方法实际上调用了 `CreateEmitter` 的 `onNext`、`onError`、`onComplete`等方法。**

`CreateEmitter` 的 `onNext`、`onError`、`onComplete`方法：
```
    @Override
    public void onNext(T t) {
        if (t == null) {
            onError(new NullPointerException("..."));
            return;
        }
        if (!isDisposed()) {
            observer.onNext(t);
        }
    }

    @Override
    public void onError(Throwable t) {
        if (!tryOnError(t)) {
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public boolean tryOnError(Throwable t) {
        if (t == null) {
            t = new NullPointerException("...");
        }
        if (!isDisposed()) {
            try {
                observer.onError(t);
            } finally {
                dispose();
            }
            return true;
        }
        return false;
    }

    @Override
    public void onComplete() {
        if (!isDisposed()) {
            try {
                observer.onComplete();
            } finally {
                dispose();
            }
        }
    }
```
* 对`onNext`、`onError`传进来的值做了空判断。
* 如果 `!isDisposed()` 则继续执行 `observer` 对象的 `onNext`、`onError`、`onComplete`等方法。
  ( **`observer` 对象为 `create`操作符 之后的 `subscribe()`方法传进来的 `Observer<T>` 对象**)
* 并在 `onComplete` 和 `onError` 方法最后执行 `dispose()` 方法。

---


接下来我们来看 `CreateEmitter` 的 `dispose() ` 和 `isDisposed()`方法
```
    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
```
继续看 `get()`方法，看下面代码可知 `get()` 返回的是一个 `Disposable` 对象
```
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {...}


public class AtomicReference<V> implements Serializable {
    private volatile V value;

    public AtomicReference(V var1) {
        this.value = var1;
    }

    public AtomicReference() {
    }

    public final V get() {
        return this.value;
    }
```

---

继续看 `DisposableHelper` 的 `isDisposed(Disposable d) ` 和 `dispose(AtomicReference<Disposable> field) `方法：
```
public enum DisposableHelper implements Disposable {
    /**
     * The singleton instance representing a terminal, disposed state, don't leak it.
     */
    DISPOSED
    ;

    public static boolean isDisposed(Disposable d) {
        return d == DISPOSED;
    }
    ...
    public static boolean dispose(AtomicReference<Disposable> field) {
        Disposable current = field.get();
        Disposable d = DISPOSED;
        if (current != d) {
            current = field.getAndSet(d);
            if (current != d) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
        return false;
    }
    ...
}
```

* `isDisposed(Disposable d)` 则是判断 `d` 是否和枚举值 `DISPOSED` 相等。
* `dispose(AtomicReference<Disposable> field) ` 方法即是 将  `CreateEmitter` 的 `isDisposed()` 中调用 `get()` 获取的对象赋值为  `DisposableHelper` 的枚举值  `DISPOSED` 。
  **所以调用`dispose(AtomicReference<Disposable> field) `方法后， `isDisposed(Disposable d)`即返回`true`。**

----

`CreateEmitter` 的 `setDisposable(Disposable d)` 和 `setCancellable(Cancellable c)`：
```
static final class CreateEmitter<T> 
        extends AtomicReference<Disposable>
        implements ObservableEmitter<T>, Disposable {
    ...
    @Override
    public void setDisposable(Disposable d) {
        DisposableHelper.set(this, d);
    }
    ...
    @Override
    public void setCancellable(Cancellable c) {
        setDisposable(new CancellableDisposable(c));
    }
    ...
}
```
**`get()`方法返回的 `Disposable`需要在 `setDisposable` 或者 `setCancellable` 设置。**
**所以如果没有调用这两个方法，`get()`方法返回的值为 `null`。**

**所以 `isDisposed(Disposable d) ` 为 `true`.**

**`dispose(AtomicReference<Disposable> field)` 方法中因为 `current` 为 `null`, 所以直接返回 `false`。**

---

如果我们在create操作符中调用了 `setDisposable` 或者 `setCancellable` 方法，如下：
```
Observable
        .create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter) throws Exception {
                emitter.setCancellable(new Cancellable() {
                    @Override
                    public void cancel() throws Exception {}
                });
                 //or
                emitter.setDisposable(new Disposable() {
                    @Override
                    public void dispose() { }

                    @Override
                    public boolean isDisposed() {
                        return false;
                    }
                }); 
            }
        })
        .subscribe(new Observer<Object>() {...});
```
 `emitter.setCancellable` 最后也是调用了  `setDisposable(new CancellableDisposable(c));` 方法。
 **所以`emitter.setDisposable` or `emitter.setCancellable` 都是通过 `DisposableHelper.set(this, d);` 去赋值 `CreateEmitter` 中的 `value` 值，我们可以通过 上述的`get()`获取。**

---

`DisposableHelper.set(this, d)`： 
```
public static boolean set(AtomicReference<Disposable> field, Disposable d) {
    for (;;) {
        Disposable current = field.get();  
        if (current == DISPOSED) {
            if (d != null) {
                d.dispose();
            }
            return false;
        }
        if (field.compareAndSet(current, d)) {
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
}
```
* 首先获取当前的 `value`值 `current`, 由上面的分析我们得知 默认为 `null`。
所以直接走到 `field.compareAndSet(current, d)`，表示更新 `CreateEmitter` 的 `value` 为 `d` ，返回 `true`则表示 传递的参数 `current` 和 `d` 值 `not equal`。
又因为`current` 为 `null`。所以直接 `return true`。

* 当我们调用了 `setDisposable` 或者 `setCancellable`之后， 再次调用  `setDisposable` 或者 `setCancellable`。
此时 `current` 的值则不为 `null`。
如果在此之前调用过 `dispose()`方法，则 `current` 即为 `DISPOSED`。所以再次 `setDisposable` 则无效。
否则 比较 当前的 `value` 是否和 `d` 相等，如果不相等  `field.compareAndSet(current, d)`则返回 `true`，更新 `CreateEmitter` 的 `value` 为 `d`，并释放 `current`。

所以我们多次调用  `setDisposable` 或者 `setCancellable` ，如果中间没有调用  `dispose();` ，后一次设置会覆盖前面一次设置，最后有效的为最后一次设置。

**以上**
#### 参考文章
* [Android RxJava 2.0：手把手带你 源码分析RxJava](https://www.jianshu.com/p/e1c48a00951a)
