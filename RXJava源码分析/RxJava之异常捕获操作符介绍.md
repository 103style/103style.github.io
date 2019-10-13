>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

#### 异常捕获相关的操作符 以及 官方介绍
`RxJava` 之 **异常捕获操作符** 官方介绍 ：[Error Handling Operators](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators)
*   [`doOnError`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#doonerror)
*   [`onErrorComplete`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#onerrorcomplete)
*   [`onErrorResumeNext`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#onerrorresumenext)
*   [`onErrorReturn`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#onerrorreturn)
*   [`onErrorReturnItem`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#onerrorreturnitem)
*   [`onExceptionResumeNext`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#onexceptionresumenext)
*   [`retry`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#retry)
*   [`retryUntil`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#retryuntil)
*   [`retryWhen`](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators#retrywhen)

---

##### doOnError
出现错误会先走 `doOnError`的回调，然后才走 `onError`.

官方示例：
```
Observable.error(new IOException("Something went wrong"))
        .doOnError(new Consumer<Throwable>() {
            @Override
            public void accept(Throwable error) throws Exception {
                System.err.println("The error message is: " + error.getMessage());
            }
        })
        .subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println("onNext should never be printed!");
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                throwable.printStackTrace();

            }
        }, new Action() {
            @Override
            public void run() throws Exception {
                System.out.println("onComplete should never be printed!");
            }
        });
```
输出：
```
The error message is: Something went wrong
java.io.IOException: Something went wrong
	at io.reactivex.android.samples.Test.main(Test.java:15)
```

---

##### onErrorComplete
`io.reactivex.functions.Predicate` 可以将 `error`事件 修正为 `complete`事件

官方示例：
```
Completable.fromAction(new Action() {
    @Override
    public void run() throws Exception {
        throw new IOException();
    }
}).onErrorComplete(new Predicate<Throwable>() {
    @Override
    public boolean test(Throwable throwable) throws Exception {
        return throwable instanceof IOException;
    }
}).subscribe(
        new Action() {
            @Override
            public void run() throws Exception {
                System.out.println("IOException was ignored");
            }
        },
        new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                System.err.println("onError should not be printed!");
            }
        });
```
输出：
```
IOException was ignored
```

---

##### onErrorResumeNext

官方示例：
```
Observable<Integer> numbers = Observable.generate(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        return 1;
    }
}, new BiFunction<Integer, Emitter<Integer>, Integer>() {
    @Override
    public Integer apply(Integer state, Emitter<Integer> emitter) throws Exception {
        emitter.onNext(state);
        return state + 1;
    }
});

numbers.scan(new BiFunction<Integer, Integer, Integer>() {
    @TargetApi(Build.VERSION_CODES.N)
    @Override
    public Integer apply(Integer i1, Integer i2) throws Exception {
        return Math.multiplyExact(i1, i2);
    }
}).onErrorResumeNext(new ObservableSource<Integer>() {
    @Override
    public void subscribe(Observer<? super Integer> observer) {
        Observable.empty();
    }
}).subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        System.out.println(integer);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) throws Exception {
        System.err.println("onError should not be printed!");
    }
});
```
输出：
```
1
2
6
24
120
720
5040
40320
362880
3628800
39916800
479001600
```

---

##### onErrorReturn
当出现 `error` 事件的时候, 走`onErrorReturn`回调之后 直接`onComplete`

官方示例：
```
Observable.just("1", "2", "2A", "3")
        .map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return Integer.parseInt(s, 10);
            }
        })
        .onErrorReturn(new Function<Throwable, Integer>() {
            @Override
            public Integer apply(Throwable throwable) throws Exception {
                if (throwable instanceof NumberFormatException) {
                    return 0;
                } else {
                    throw new IllegalArgumentException();
                }
            }
        }).subscribe(
        new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                System.err.println("onError should not be printed!");
            }
        });
```
输出：
```
1
2
0
```

---

##### onErrorReturnItem
当出现 `error` 事件的时候, 传递`onErrorReturnItem`的参数给 `onNext` 之后 直接`onComplete`

官方示例：
```
Observable.just("1", "2", "2A", "3")
        .map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return Integer.parseInt(s, 10);
            }
        })
        .onErrorReturnItem(100)
        .subscribe(
                new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(Throwable throwable) throws Exception {
                        System.err.println("onError should not be printed!");
                    }
                });  
```
输出：
```
1
2
100
```

---

##### onExceptionResumeNext
官方示例：
```
Observable<String> exception = Observable.<String>error(new IOException())
        .onExceptionResumeNext(new ObservableSource<String>() {
            @Override
            public void subscribe(Observer<? super String> observer) {
                Observable.just("This value will be used to recover from the IOException");
            }
        });

Observable<String> error = Observable.<String>error(new Error())
        .onExceptionResumeNext(new ObservableSource<String>() {
            @Override
            public void subscribe(Observer<? super String> observer) {
                Observable.just("This value will not be used");
            }
        });

Observable.concat(exception, error)
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println("onNext: " + s);
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                System.err.println("onError: " + throwable);
            }
        });
```
输出：
```
onNext: This value will be used to recover from the IOException
onError: java.lang.Error
```

---

##### retry
遇到 `error` 事件 根据 `retry` 的回调来确定是否重试。

官方示例：
```
Observable<Long> source = Observable.interval(0, 1, TimeUnit.SECONDS)
        .flatMap(new Function<Long, ObservableSource<Long>>() {
            @Override
            public ObservableSource<Long> apply(Long x) throws Exception {
                if (x >= 2) {
                    return Observable.error(new IOException("Something went wrong!"));
                } else {
                    return Observable.just(x);
                }
            }
        });

source.retry(new BiPredicate<Integer, Throwable>() {
    @Override
    public boolean test(Integer integer, Throwable throwable) throws Exception {
        return integer < 3;
    }
}).blockingSubscribe(new Consumer<Long>() {
    @Override
    public void accept(Long x) throws Exception {
        System.out.println("onNext: " + x);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) throws Exception {
        System.out.println(throwable.getMessage());
    }
});
```
输出：
```
onNext: 1
onNext: 0
onNext: 1
onNext: 0
onNext: 1
Something went wrong!
```

---

##### retryUntil
遇到 `error` 事件 根据 `retryUntil` 的回调来确定是否重试。

官方示例：
```
final LongAdder errorCounter = new LongAdder();
Observable<Long> source = Observable.interval(0, 1, TimeUnit.SECONDS)
        .flatMap(new Function<Long, ObservableSource<Long>>() {
            @Override
            public ObservableSource<Long> apply(Long x) throws Exception {
                if (x >= 2) {
                    return Observable.error(new IOException("Something went wrong!"));
                } else {
                    return Observable.just(x);
                }
            }
        }).doOnError(new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                errorCounter.increment();
            }
        });

source.retryUntil(new BooleanSupplier() {
    @Override
    public boolean getAsBoolean() throws Exception {
        return errorCounter.intValue() >= 3;
    }
}).blockingSubscribe(new Consumer<Long>() {
    @Override
    public void accept(Long x) throws Exception {
        System.out.println("onNext: " + x);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) throws Exception {
        System.out.println(throwable.getMessage());
    }
});
```
输出：
```
onNext: 0
onNext: 1
onNext: 0
onNext: 1
onNext: 0
onNext: 1
Something went wrong!
```

---

##### retryWhen
遇到 `error` 事件 根据 `retryWhen` 的回调来确定是否重试。

官方示例：
```
Observable<Long> source = Observable.interval(0, 1, TimeUnit.SECONDS)
        .flatMap(new Function<Long, ObservableSource<Long>>() {
            @Override
            public ObservableSource<Long> apply(Long x) throws Exception {
                if (x >= 2) {
                    return Observable.error(new IOException("Something went wrong!"));
                } else {
                    return Observable.just(x);
                }
            }
        });

source.retryWhen(new Function<Observable<Throwable>, ObservableSource<?>>() {
    @Override
    public ObservableSource<?> apply(Observable<Throwable> throwableObservable) throws Exception {
        return throwableObservable.map(new Function<Throwable, Integer>() {
            @Override
            public Integer apply(Throwable throwable) throws Exception {
                return 1;
            }
        }).scan(new BiFunction<Integer, Integer, Integer>() {
            @Override
            public Integer apply(Integer integer, Integer integer2) throws Exception {
                return integer + integer2;
            }
        }).doOnNext(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println("No. of errors: " + integer);
            }
        }).takeWhile(new Predicate<Integer>() {
            @Override
            public boolean test(Integer integer) throws Exception {
                return integer < 3;
            }
        }).flatMapSingle(new Function<Integer, SingleSource<?>>() {
            @Override
            public SingleSource<?> apply(Integer integer) throws Exception {
                return Single.timer(integer, TimeUnit.SECONDS);
            }
        });
    }
}).blockingSubscribe(new Consumer<Long>() {
    @Override
    public void accept(Long x) throws Exception {
        System.out.println("onNext: " + x);
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) throws Exception {
        throwable.printStackTrace();
    }
}, new Action() {
    @Override
    public void run() throws Exception {
        System.out.println("onComplete");
    }
});
```
输出：
```
onNext: 0
onNext: 1
No. of errors: 1
onNext: 0
onNext: 1
No. of errors: 2
onNext: 0
onNext: 1
No. of errors: 3
onComplete
```

---

以上
