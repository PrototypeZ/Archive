---
title: RxJava 沉思录（一）：你认为 RxJava 真的好用吗？
tags: RxJava
date: 2018-08-29 15:31:00
desc: 重新认真思考 RxJava
---

本人两年前第一次接触 RxJava，和大多数初学者一样，看的第一篇 RxJava 入门文章是扔物线写的[《给 Android 开发者的 RxJava 详解》](https://gank.io/post/560e15be2dca930e00da1083#toc_31)，这篇文章流传之广，相信几乎所有学习 RxJava 的开发者都阅读过。尽管那篇文章定位读者是 RxJava 入门的初学者，但是阅读完之后还是觉得懵懵懂懂，总感觉依然不是很理解这个框架设计理念以及优势。

<!-- More -->

随后工作中有机会使用 RxJava  重构了项目的网络请求以及缓存层，期间陆陆续续又重构了数据访问层，以及项目中其他的一些功能模块，无一例外，我们都选择使用了 RxJava 。

最近翻看一些技术文章，发现涉及 RxJava 的文章还是大多以入门为主，我尝试从一个初学者的角度阅读，发现很多文章都没讲到关键的概念点，举的例子也不够恰当。回想起两年前刚刚学习 RxJava 的自己，虽然看了许多 RxJava 入门的文章，但是始终无法理解 RxJava 究竟好在哪里，所以一定是哪里出问题了。于是有了这一篇反思，希望能和你一起重新思考 RxJava，以及重新思考 RxJava 是否真的让我们的开发变得更轻松。

## 观察者模式有那么神奇吗?

几乎所有 RxJava 入门介绍，都会用一定的篇幅去介绍 “**观察者模式**”，告诉你观察者模式是 RxJava 的核心，是基石:

```java
observable.subscribe(new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
})
```

年少的我不明觉厉：“好厉害，原来这是观察者模式”，但是心里还是感觉有点不对劲：“这代码是不是有点丑？接收到数据的回调名字居然叫 `onNext` ? ”

但是其实观察者并不是什么新鲜的概念，即使你是新手，你肯定也已经写过不少观察者模式的代码了，你能看懂下面一行代码说明你已经对观察者模式了然于胸了：

```java
button.setOnClickListener(v -> doSomething());
```

这就是观察者模式，`OnClickListener` 订阅了 button 的点击事件，就这么简单。原生的写法对比上面 RxJava 那一长串的写法，是不是要简单多了。有人可能会说，RxJava 也可以写成一行表示：

```java
RxView.clicks(button).subscribe(v -> doSomething());
```

先不说这么写需要引入 [RxBinding](https://github.com/JakeWharton/RxBinding) 这个第三方库，不考虑这点，这两种写法最多也只是打个平手，完全体现不出 RxJava 有任何优势。

这就是我要说的第一个论点，如果仅仅只是为了使用 RxJava 的观察者模式，而把原先 Callback 的形式，改为 RxJava 的 `Observable` 订阅模式是没有价值的，你只是把一种观察者模式改写成了另一种观察者模式。我是实用主义者，使用 RxJava 不是为了炫技，所以观察者模式是我们使用 RxJava 的理由吗？当然不是。

## 链式编程很厉害吗?

链式编程也是每次提到 `RxJava` 的时候总会出现的一个高频词汇，很多人形容链式编程是 `RxJava` 解决异步任务的 “杀手锏”：

```java
Observable.from(folders)
    .flatMap((Func1) (folder) -> { Observable.from(file.listFiles()) })
    .filter((Func1) (file) -> { file.getName().endsWith(".png") })
    .map((Func1) (file) -> { getBitmapFromFile(file) })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe((Action1) (bitmap) -> { imageCollectorView.addImage(bitmap) });
```

这段代码出现的频率非常的高，好像是 RxJava 的链式编程给我们带来的好处的最佳佐证。然而平心而论，我看到这个例子的时候，内心是平静的，并没有像大多数文章写得那样，内心产生“它很长，但是很清晰”的心理活动。

首先，`flatMap`, `filter`, `map` 这几个操作符，对于没有函数式编程经验的初学者来讲，并不好理解。其次，虽然这段代码用了很多 RxJava 的操作符，但是其逻辑本质并不复杂，就是在后台线程把某个文件夹里面的以 **png** 结尾的图片文件解析出来，交给 UI 线程进行渲染。

上面这段代码，还带有一个反例，使用 `new Thread()` 的方式实现的版本：

```java
new Thread() {
    @Override
    public void run() {
        super.run();
        for (File folder : folders) {
            File[] files = folder.listFiles();
            for (File file : files) {
                if (file.getName().endsWith(".png")) {
                    final Bitmap bitmap = getBitmapFromFile(file);
                    getActivity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageCollectorView.addImage(bitmap);
                        }
                    });
                }
            }
        }
    }
}.start();
```

对比两种写法，可以发现，之所以 RxJava 版本的缩进减少了，是因为它利用了函数式的操作符，把原本嵌套的 `for` 循环逻辑展平到了同一层次，事实上，我们也可以把上面那个反例的嵌套逻辑展平，既然要用 `lambda` 表达式，那肯定要大家都用才比较公平吧：

```java
new Thread(() -> {
    File[] pngFiles = new File[]{};
    for (File folder : folders) {
        pngFiles = ArrayUtils.addAll(pngFiles, folder.listFiles());
    }
    for (File file : pngFiles) {
        if (file.getName().endsWith(".png")) {
            final Bitmap bitmap = getBitmapFromFile(file);
            getActivity().runOnUiThread(() -> imageCollectorView.addImage(bitmap));
        }
    }
}).start();
```

坦率地讲，这段代码除了 `new Thread().start()` 有槽点以外，没什么大毛病。RxJava 版本确实代码更少，同时省去了一个中间变量 `pngFiles`，这得益于函数式编程的 API，但是实际开发中，这两种写法无论从性能还是项目可维护性上来看，并没有太大的差距，甚至，如果团队并不熟悉函数式编程，后一种写法反而更容易被大家接受。

回到刚才说的“链式编程”，RxJava 把目前 Android Sdk 24 以上才支持的 Java 8 Stream 函数式编程风格带到了带到了低版本 Android 系统上，确实带给我们一些方便，但是仅此而已吗？到目前为止我并没有看到 RxJava 在处理事件尤其是异步事件上有什么特别的手段。

准确的来说，我的关注点并不在大多数文章鼓吹的“链式编程”这一点上，把多个依次执行的异步操作的调用转化为类似同步代码调用那样的自上而下执行，并不是什么新鲜事，而且就这个具体的例子，使用 Android 原生的 `AsyncTask` 或者 `Handler` 就可以满足需求，RxJava 相比原生的写法无法体现它的优势。

除此以外，对于处理异步任务，还有 `Promise` 这个流派，使用类似这样的 API:

```java
promise
    .then(r1 -> task1(r1))
    .then(r2 -> task2(r2))
    .then(r3 -> task3(r3))
    ...
```

难道不是比 RxJava 更加简洁直观吗？而且还不需要引入函数式编程的内容。这种写法，跟所谓的“逻辑简洁”也根本没什么关系，所以从目前看来，RxJava 在我心目只是个 **“哦，还挺不错”** 的框架，但是并没有惊艳到我。

以上是我要说的第二个论点，链式编程的形式只是一种语法糖，通过函数式的操作符可以把嵌套逻辑展平，通过别的方法也可以把嵌套逻辑展平，这只是普通操作，也有其他框架可以做到相似效果。

## RxJava 等于异步加简洁吗?

相信阅读过本文开头介绍的那篇 RxJava 入门文 [《给 Android 开发者的 RxJava 详解》](https://gank.io/post/560e15be2dca930e00da1083#toc_31) 的开发者一定对文中两个小标题印象深刻：

> RxJava 到底是什么？  —— 一个词：**异步**

> RxJava 好在哪？ —— 一个词：**简洁**

首先感谢扔物线，很用心地为初学者准备了这篇简洁朴实的入门文。但是我还是想要指出，这样的表达是**不够严谨的**。

虽然我们使用 RxJava 的场景大多数与异步有关，但是这个框架并不是与异步等价的。举个简单的例子：

```java
Observable.just(1,2,3).subscribe(System.out::println);
```

上面的代码就是同步执行的，和异步没有关系。事实上，RxJava 除非你显式切换到其他的 `Scheduler`，或者你使用的某些操作符隐式指定了其他 `Scheduler`，否则 **RxJava 相关代码就是同步执行的**。 

这种设计和这个框架的野心有关，RxJava 是一种新的 **事件驱动型** 编程范式，它以异步为切入点，试图一统 **同步** 和 **异步** 的世界。
本文前面提到过：

> RxJava 把目前 Android Sdk 24 以上才支持的 Java 8 Stream 函数式编程风格带到了带到了低版本 Android 系统上。

所以只要你愿意，你完全可以在日常的同步编程上使用 RxJava，就好像你在使用 Java 8 的 Stream API。( 但是两者并不等价，因为 RxJava 是事件驱动型编程 )

如果你把日常的同步编程，封装为同步事件的 `Observable`，那么你会发现，同步和异步这两种情况被 RxJava 统一了, 两者具有一样的接口，可以被无差别的对待，同步和异步之间的协作也可以变得比之前更容易。

所以，到此为止，我这里的结论是：**RxJava 不等于异步**。

那么 RxJava 等于 **简洁** 吗？我相信有一些人会说 “是的，RxJava 很简洁”，也有一些人会说 “不，RxJava 太糟糕了，一点都不简洁”。这两种说法我都能理解，其实问题的本质在于对 **简洁** 这个词的定义上。关于这个问题，后续会有一个小节专门讨论，但是我想提前先下一个结论，**对于大多数人，RxJava 不等于简洁**，有时候甚至是更难以理解的代码以及更低的项目可维护性。

## RxJava 是用来解决 Callback Hell 的吗?

很多 RxJava 的入门文都宣扬：RxJava 是用来解决 **Callback Hell** （有些翻译为“回调地狱”）问题的，指的是过多的异步调用嵌套导致的代码呈现出的难以阅读的状态。

我并不赞同这一点。**Callback Hell** 这个问题，最严重的重灾区是在 Web 领域，是使用 JavaScript 最常见的问题之一，以至于专门有一个网站 [callbackhell.com](http://callbackhell.com/) 来讨论这个问题，由于客户端编程和 Web 前端编程具有一定的相似性，Android 编程或多或少也存在这个问题。

上面这个网站中，介绍了几种规避 Callback Hell 的常见方法，无非就是把嵌套的层次移到外层空间来，不要使用匿名的回调函数，为每个回调函数命名。如果是 Java 的话，对应的，避免使用匿名内部类，为每个内部类的对象，分配一个对象名。当然，也可以使用框架来解决这类问题，使用类似 `Promise` 那样的专门为异步编程打造的框架，Android 平台上也有类似的开源版本 [jdeferred](https://github.com/jdeferred/jdeferred)。

在我看来，**jdeferred** 那样的框架，更像是那种纯粹的用来解决 **Callback Hell** 的框架。 至于 RxJava，前面也提到过，它是一个更有野心的框架，正确使用了 RxJava 的话，确实不会有 **Callback Hell** 再出现了，但如果说 RxJava 就是用来解决 **Callback Hell** 的，那就有点高射炮打蚊子的意味了。

## 如何理解 RxJava

也许阅读了前面几小节内容之后，你的心中会和曾经的我一样，对 RxJava 产生一些消极的想法，并且会产生一种疑问：那么 RxJava 存在的意义究竟是什么呢？

举几个常见的例子：

1. 为 View 设置点击回调方法:
```java
btn.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View v) {
        // callback body
    }
});
```

2. Service 组件绑定操作:
```java
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName className, IBinder service) {
        // callback body
    }
    @Override
    public void onServiceDisconnected(ComponentName arg0) {
        // callback body
    }
};

...
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```

3. 使用 Retrofit 发起网络请求:
```java
Call<List<Photo>> call = service.getAllPhotos();
call.enqueue(new Callback<List<Photo>>() {
    @Override
    public void onResponse(Call<List<Photo>> call, Response<List<Photo>> response) {
        // callback body
    }
    @Override
    public void onFailure(Call<List<Photo>> call, Throwable t) {
        // callback body
    }
});
```

在日常开发中我们时时刻刻在面对着类似的回调函数，而且容易看出来，回调函数最本质的功能就是把异步调用的结果返回给我们，剩下的都是大同小异。所以我们能不能不要去记忆各种各样的回调函数，只使用一种回调呢？如果我们定义统一的回调如下：

```java
public class Callback<T> {
    public void onResult(T result);
}
```
那么以上 3 种情况，对应的回调变成了：
1. 为 View 设置点击事件对应的回调为 `Callback<View>`
2. Service 组件绑定操作对应的回调为 `Callback<Pair<CompnentName, IBinder>>` (onServiceConnected)、 `Callback<CompnentName>` (onServiceDisconnected)
3. 使用 Retrofit 发起网络请求对应的回调为 `Callback<List<Photo>>` (onResponse)、 `Callback<Throwable>` (onFailure)

只要按照这种思路，我们可以把所有的异步回调封装成 `Callback<T>` 的形式，我们不再需要去记忆不同的回调，只需要和一种回调交互就可以了。

写到这里，你应该已经明白了，RxJava 存在首先最基本的意义就是 **统一了所有异步任务的回调接口** 。而这个接口就是 `Observable<T>`，这和刚刚的 `Callback<T>` 其实是一个意思。此外，我们可以考虑让这个回调更通用一点 —— 可以被回调多次，对应的，`Observable` 表示的就是一个事件流，它可以发射一系列的事件（`onNext`），包括一个终止信号（`onComplete`）。

如果 RxJava 单单只是统一了回调的话，其实还并没有什么了不起的。统一回调这件事情，除了满足强迫症以外，额外的收益有限，而且需要改造已有代码，短期来看属于负收益。但是 `Observable` 属于 RxJava 的基础设施，**有了 `Observable` 以后的 RxJava 才刚刚插上了想象力的翅膀**。

（未完待续）

本文属于 "RxJava 沉思录" 系列，欢迎阅读本系列的其他分享：

+ [RxJava 沉思录（一）：你认为 RxJava 真的好用吗？](/2018/08/29/thoughts-in-rxjava-1/)
+ [RxJava 沉思录（二）：空间维度](/2018/08/30/thoughts-in-rxjava-2/)
+ [RxJava 沉思录（三）：时间维度](/2018/08/31/thoughts-in-rxjava-3/)
+ [RxJava 沉思录（四）：总结](/2018/09/01/thoughts-in-rxjava-4/)

___
如果您对我的技术分享感兴趣，欢迎关注我的个人公众号：麻瓜日记，不定期更新原创技术分享，谢谢！:)

![](http://prototypez.github.io/images/qrcode.jpg)
