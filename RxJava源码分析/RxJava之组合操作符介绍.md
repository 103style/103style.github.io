# RxJava之组合操作符介绍 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

#### 组合相关的操作符 以及 官方介绍
`RxJava` 之 **组合操作符** 官方介绍 ：[Combining Observables](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables)
*   [`combineLatest`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#combineLatest)
*   [`join`]((https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#joins)) and [`groupJoin`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#joins)
*   [`merge`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#merge)
*   [`mergeDelayError`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#mergeDelayError)
*   [`rxjava-joins`--`and`、`then`、`when`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#rxjava-joins)
*   [`startWith`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#startWith)
*   [`switchOnNext`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#switchOnNext)
*   [`zip`](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables#zip)


##### combineLatest
传入的`Observable`事件任何一个发生时，都通过最后和函数返回对应的结果

官方示例：
```
Observable<Long> newsRefreshes = Observable.interval(100, TimeUnit.MILLISECONDS);
Observable<Long> weatherRefreshes = Observable.interval(50, TimeUnit.MILLISECONDS);

Observable.combineLatest(newsRefreshes, weatherRefreshes,
        new BiFunction<Long, Long, String>() {
            @Override
            public String apply(Long newsRefreshTimes, Long weatherRefreshTimes) throws Exception {
                return "Refreshed news " + newsRefreshTimes + " times and weather " + weatherRefreshTimes;
            }
        })
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String item) throws Exception {
                Log.e(TAG, item);
            }
        });
```
输出：
```
Refreshed news 0 times and weather 0
Refreshed news 0 times and weather 1
Refreshed news 0 times and weather 2
Refreshed news 1 times and weather 2
Refreshed news 1 times and weather 3
Refreshed news 1 times and weather 4
Refreshed news 2 times and weather 4
Refreshed news 2 times and weather 5
Refreshed news 2 times and weather 6
Refreshed news 3 times and weather 6
Refreshed news 3 times and weather 7
Refreshed news 3 times and weather 8
```

---

##### merge
合并多个`Observables `为一个`Observables `

官方示例：
```
Observable.just(1, 2, 3)
        .mergeWith(Observable.just(4, 5, 6))
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });
```
输出：
```
1
2
3
4
5
6
```

---


##### mergeDelayError
中途发送的`onError()`直到所有的`Observable`合并完成，才传递给观察者

官方示例：
```
Observable<String> observable1 = Observable.error(new IllegalArgumentException(""));
Observable<String> observable2 = Observable.just("Four", "Five", "Six");
Observable.mergeDelayError(observable1, observable2)
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
```
输出：
```
Four
Five
Six
io.reactivex.exceptions.OnErrorNotImplementedException: The exception was not handled due to missing onError handler in the subscribe() method call.
...
```

---


##### startWith
在`Observable`事件流发出之前，发出`startWith`传的参数

官方示例：
```
Observable<String> names = Observable.just("Spock", "McCoy");
names.startWith("Kirk")
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
```
输出：
```
Kirk
Spock
McCoy
```

---


##### switchOnNext
将发出`Observable`的`Observable`转换为单个`Observable`。

官方示例：
```
Observable<Observable<String>> timeIntervals =
        Observable.interval(1, TimeUnit.SECONDS)
                .map(new Function<Long, Observable<String>>() {
                    @Override
                    public Observable<String> apply(final Long ticks) throws Exception {
                        return Observable.interval(500, TimeUnit.MILLISECONDS)
                                .map(new Function<Long, String>() {
                                    @Override
                                    public String apply(Long innerInterval) throws Exception {
                                        return "outer: " + ticks + " - inner: " + innerInterval;
                                    }
                                });
                    }
                });
Observable.switchOnNext(timeIntervals)
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
                Log.e(TAG, s);
            }
        });
```
输出：
```
outer: 0 - inner: 0
outer: 1 - inner: 0
outer: 2 - inner: 0
outer: 3 - inner: 0
outer: 4 - inner: 0
outer: 5 - inner: 0
outer: 6 - inner: 0
outer: 7 - inner: 0
```

---


##### zip
两个或多个`Observable`中发射的事件 一 一 合并

官方示例：
```
Observable<String> firstNames = Observable.just("James", "Jean-Luc", "Benjamin");
Observable<String> lastNames = Observable.just("Kirk", "Picard", "Sisko");
firstNames.zipWith(lastNames, new BiFunction<String, String, String>() {
    @Override
    public String apply(String s, String s2) throws Exception {
        return s + " " + s2;
    }
}).subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        System.out.println(s);
    }
});
```
输出：
```
James Kirk
Jean-Luc Picard
Benjamin Sisko
```

---

##### join
官方示例：
```
Observable<String> createObserver = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        for (int i = 1; i < 5; i++) {
            emitter.onNext("Right-" + i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}).subscribeOn(Schedulers.newThread());
Observable.just("Left-")
        .join(createObserver,
                new Function<String, ObservableSource<Long>>() {
                    @Override
                    public ObservableSource<Long> apply(String s) throws Exception {
                        return Observable.timer(3000, TimeUnit.MILLISECONDS);
                    }
                },
                new Function<String, ObservableSource<Long>>() {
                    @Override
                    public ObservableSource<Long> apply(String s) throws Exception {
                        return Observable.timer(2000, TimeUnit.MILLISECONDS);
                    }
                },
                new BiFunction<String, String, String>() {
                    @Override
                    public String apply(String s, String s2) throws Exception {
                        return s + s2;
                    }
                })
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
```

---

##### groupJoin
官方示例：
```
Observable<String> createObserver = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        for (int i = 1; i < 5; i++) {
            emitter.onNext("Right-" + i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}).subscribeOn(Schedulers.newThread());
Observable.just("Left-")
        .groupJoin(createObserver,
                new Function<String, ObservableSource<Long>>() {
                    @Override
                    public ObservableSource<Long> apply(String s) throws Exception {
                        return Observable.timer(3000, TimeUnit.MILLISECONDS);
                    }
                },
                new Function<String, ObservableSource<Long>>() {
                    @Override
                    public ObservableSource<Long> apply(String s) throws Exception {
                        return Observable.timer(2000, TimeUnit.MILLISECONDS);
                    }
                },
                new BiFunction<String, Observable<String>, Observable<String>>() {
                    @Override
                    public Observable<String> apply(final String s, Observable<String> stringObservable) throws Exception {
                        return stringObservable.map(new Function<String, String>() {
                            @Override
                            public String apply(String mapString) throws Exception {
                                return s + mapString;
                            }
                        });
                    }
                }).subscribe(new Consumer<Observable<String>>() {
    @Override
    public void accept(Observable<String> stringObservable) throws Exception {
        stringObservable.subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
    }
});
```


以上
