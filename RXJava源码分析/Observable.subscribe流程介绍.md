>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

本文基于 `RxJava 2.x` 版本

---


我们直接看`Observable `的`subscribe`方法
```
public final Disposable subscribe() {
    return subscribe(Functions.emptyConsumer(), Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION, Functions.emptyConsumer());
}
public final Disposable subscribe(Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION, Functions.emptyConsumer());
}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {
    return subscribe(onNext, onError, Functions.EMPTY_ACTION, Functions.emptyConsumer());
}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
                                  Action onComplete) {
    return subscribe(onNext, onError, onComplete, Functions.emptyConsumer());
}
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
                                  Action onComplete, Consumer<? super Disposable> onSubscribe) {
    ...
    LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);
    subscribe(ls);
    return ls;
}
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        ...
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        ...
    }
}
protected abstract void subscribeActual(Observer<? super T> observer);
```


* `subscribe()`
* `subscribe(Consumer<? super T> onNext)`
* `subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError)`
* `subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete) `
* `subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Consumer<? super Disposable> onSubscribe) `

我们可以看到 前面四个方法都是调用了第五个方法，对参数`onNext`、`onError`、`onComplete`、`onSubscribe`的默认赋值。

然后四参数的方法将`onNext`、`onError`、`onComplete`、`onSubscribe`构建成一个`LambdaObserver`对象，传递给了`subscribe(Observer<? super T> observer)`方法。

而 `subscribe(Observer<? super T> observer)`则是调用了抽象方法`subscribeActual(Observer<? super T> observer)`，这个方法由上一个操作符返回的`Observer`对象重写实现。

---

接下来我们先来看看这些参数的默认值：
* `Functions.emptyConsumer()`：`accept(Object v)`回调的空实现
  ```
  public static <T> Consumer<T> emptyConsumer() {
      return (Consumer<T>)EMPTY_CONSUMER;
  }

  static final Consumer<Object> EMPTY_CONSUMER = new EmptyConsumer();

  static final class EmptyConsumer implements Consumer<Object> {
      @Override
      public void accept(Object v) { }

      @Override
      public String toString() {
          return "EmptyConsumer";
      }
  }
  ```
* `Functions.ON_ERROR_MISSING`： 输出报错信息
  ```
  public static final Consumer<Throwable> ON_ERROR_MISSING = new OnErrorMissingConsumer();
  static final class OnErrorMissingConsumer implements Consumer<Throwable> {
      @Override
      public void accept(Throwable error) {
          RxJavaPlugins.onError(new OnErrorNotImplementedException(error));
      }
  }
  public static void onError(@NonNull Throwable error) {
      ...
      error.printStackTrace(); // NOPMD
      uncaught(error);
  }
  ```
* `Functions.EMPTY_ACTION`：`run()`回调的空实现
  ```
  public static final Action EMPTY_ACTION = new EmptyAction();
  static final class EmptyAction implements Action {
      @Override
      public void run() { }

      @Override
      public String toString() {
          return "EmptyAction";
      }
  }
  ```

---

然后我们来看看`LambdaObserver`类：
```
public final class LambdaObserver<T> extends AtomicReference<Disposable>
        implements Observer<T>, Disposable, LambdaConsumerIntrospection {

    final Consumer<? super T> onNext;
    final Consumer<? super Throwable> onError;
    final Action onComplete;
    final Consumer<? super Disposable> onSubscribe;

    public LambdaObserver(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete,
            Consumer<? super Disposable> onSubscribe) {
        super();
        this.onNext = onNext;
        this.onError = onError;
        this.onComplete = onComplete;
        this.onSubscribe = onSubscribe;
    }

    @Override
    public void onSubscribe(Disposable d) {
        if (DisposableHelper.setOnce(this, d)) {
            try {
                onSubscribe.accept(this);
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                d.dispose();
                onError(ex);
            }
        }
    }

    @Override
    public void onNext(T t) {
        if (!isDisposed()) {
            try {
                onNext.accept(t);
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                get().dispose();
                onError(e);
            }
        }
    }

    @Override
    public void onError(Throwable t) {
        if (!isDisposed()) {
            lazySet(DisposableHelper.DISPOSED);
            try {
                onError.accept(t);
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                RxJavaPlugins.onError(new CompositeException(t, e));
            }
        } else {
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public void onComplete() {
        if (!isDisposed()) {
            lazySet(DisposableHelper.DISPOSED);
            try {
                onComplete.run();
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                RxJavaPlugins.onError(e);
            }
        }
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return get() == DisposableHelper.DISPOSED;
    }

    @Override
    public boolean hasCustomOnError() {
        return onError != Functions.ON_ERROR_MISSING;
    }
}
```
我们只要关注`onSubscribe(Disposable d)`、`onNext(T t)`、`onError(Throwable t)`、`onComplete()`这几个实现，分别是调用`onSubscribe`、`onNext`、`onError`、`onComplete`几个对象的回调方法。

以上
