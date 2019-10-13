>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### Flowable 的 subscribe 方法
```
public final Disposable subscribe() {
    return subscribe(Functions.emptyConsumer(), Functions.ON_ERROR_MISSING,
            Functions.EMPTY_ACTION, FlowableInternalHelper.RequestMax.INSTANCE);
}

public final Disposable subscribe(Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING,
            Functions.EMPTY_ACTION, FlowableInternalHelper.RequestMax.INSTANCE);
}

public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {
    return subscribe(onNext, onError, Functions.EMPTY_ACTION,
            FlowableInternalHelper.RequestMax.INSTANCE);
}

public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
                                  Action onComplete) {
    return subscribe(onNext, onError, onComplete,
            FlowableInternalHelper.RequestMax.INSTANCE);
}

public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
                                  Action onComplete, Consumer<? super Subscription> onSubscribe) {
    LambdaSubscriber<T> ls = new LambdaSubscriber<T>(onNext, onError, onComplete, onSubscribe);
    subscribe(ls);
    return ls;
}

public final void subscribe(Subscriber<? super T> s) {
    if (s instanceof FlowableSubscriber) {
        subscribe((FlowableSubscriber<? super T>)s);
    } else {
        ObjectHelper.requireNonNull(s, "s is null");
        subscribe(new StrictSubscriber<T>(s));
    }
}

public final void subscribe(FlowableSubscriber<? super T> s) {
    ObjectHelper.requireNonNull(s, "s is null");
    try {
        Subscriber<? super T> z = RxJavaPlugins.onSubscribe(this, s);
        subscribeActual(z);
    } catch (...) {
       ...
    }
}

protected abstract void subscribeActual(Subscriber<? super T> s);
```
前面四个方法都是调用了通过默认的：
* `Functions.emptyConsumer()` : 
    ```
    static final class EmptyConsumer implements Consumer<Object> {
        @Override
        public void accept(Object v) { }

        @Override  
        public String toString() {
            return "EmptyConsumer";
        }
    }
    ```
* `Functions.ON_ERROR_MISSING` : 
    ```
    static final class OnErrorMissingConsumer implements Consumer<Throwable> {
        @Override
        public void accept(Throwable error) {
            RxJavaPlugins.onError(new OnErrorNotImplementedException(error));
        }
    }
    ```
* `Functions.EMPTY_ACTION` : 
    ```
    static final class EmptyAction implements Action {
        @Override
        public void run() { }

        @Override
        public String toString() {
            return "EmptyAction";
        }
    }
    ```
* `FlowableInternalHelper.RequestMax.INSTANCE` : 
    ```    
    public enum RequestMax implements Consumer<Subscription> {
        INSTANCE;
        @Override
        public void accept(Subscription t) throws Exception {
            t.request(Long.MAX_VALUE);
        }
    }
    ```

调用了`subscribe(onNext, onError, onComplete, onSubscribe)`，然后将四个参数包装成一个 `LambdaSubscriber`对象 传递给 子类重写 的 `subscribeActual`方法。


而 `subscribe(Subscriber<? super T> s)` 则通过自己传递 实现`FlowableSubscriber`接口 或者  传递一个`Subscriber`构造成`StrictSubscriber` 传递给 子类重写 的 `subscribeActual`方法。

然后接下来的流程就和 [Rxjava之create操作符源码解析](https://www.jianshu.com/p/ffb85e931f11) 中介绍的类似。

以上。
