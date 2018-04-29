#### 1.理解响应式编程

- [原文：introrx.md](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
- [译文：响应式编程介绍](http://blog.csdn.net/womendeaiwoming/article/details/46506017)


#### 2.学习RxJava

- [RxJava Github](https://github.com/ReactiveX/RxJava)

- [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)
- [Android：手把手带你入门神秘的 Rxjava](http://blog.csdn.net/carson_ho/article/details/78179340)
- [Android： RxJava操作符 详细使用手册](http://blog.csdn.net/carson_ho/article/details/79191327)
- [理解RxJava:(一)基础知识](http://www.cnblogs.com/JohnTsai/p/5695560.html)

- [我们为什么要在Android中使用RxJava
](https://www.jianshu.com/p/3a4fb0cf6533)(一篇很粗糙的文章，哈哈)


#### 3.我的理解

利用RxJava进行响应式编程只不过是简化业务代码的一种方式，当精通RxJava后对业务代码的简化是客观的，但是学习成本较高，如果团队有人不了解RxJava的话那么业务代码会出现无法进行下去或各种风格的代码混杂在工程中的情况。RxJava可以学习，但不是业务开发的必需品。（2018-1-30）



#### 4.一些例子的摘要


```
Observable<String> myObservable = Observable.create(
    new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> sub) {
            sub.onNext("Hello, world!");
            sub.onCompleted();
        }
    }
);

Subscriber<String> mySubscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) { System.out.println(s); }

    @Override
    public void onCompleted() { }

    @Override
    public void onError(Throwable e) { }
};

myObservable.subscribe(mySubscriber);
// 输出 "Hello, world!"

```

```
Observable<String> myObservable =
    Observable.just("Hello, world!");
Action1<String> onNextAction = new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println(s);
    }
};


myObservable.subscribe(onNextAction, onErrorAction, onCompleteAction);

myObservable.subscribe(onNextAction);
// 输出 "Hello, world!"

```

```
Observable.just("Hello, world!")
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
              System.out.println(s);
        }
    });
    
Observable.just("Hello, world!")
    .subscribe(s -> System.out.println(s));  
```

**Operators**

```

Observable.just("Hello, world!")
    .map(new Func1<String, String>() {
        @Override
        public String call(String s) {
            return s + " -Dan";
        }
    })
    .subscribe(s -> System.out.println(s));
    
    
/*****/
Observable.just("Hello, world!")
    .map(s -> s.hashCode())
    .map(i -> Integer.toString(i))
    .subscribe(s -> System.out.println(s));    
    
```

**线程的应用**

```
myObservableServices.retrieveImage(url)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```