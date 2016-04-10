---
title: RxJava 常见误区（一）：过度使用 Subject
tags: RxJava
desc: RxJava使用过程遇到的一些误区
---
准备写这篇文章的时候看了下 RxJava 在 Github 上已经 12000+ 个 star 了，可见火爆程度，自己使用 RxJava 也已经有一小段时间。最初是在社区对 RxJava 一片赞扬之声下，开始使用 RxJava 来代替项目中一些简单异步请求，到后来才开始接触一些高级玩法，这中间阅读别人的代码加上自己踩的坑，慢慢积累了一些经验，很多都是新手容易犯的错误和 RxJava 容易被误解的地方。这些内容一篇文章写不完，所以我打算写成一个系列，这篇文章是这个系列的第一篇。

<!-- More -->

## 谨慎使用Subject
`Subject`既是`Observable`也是`Observer`，由于它自己本身是`Observer`，所以项目中任何地方都可以调用它的`onNext`方法(只要能获得该 Subject 的引用)。看起来很好对不对？比起`Observable.create`, `Observable.from`, `Observable.just`方便多了，这三个工厂方法都有一个特点，那就是所构建出来的 Observable 发射的元素是确定的，甚至在很多例子中，待发射的元素就像常量一样在编译期就已经可以确定。我在一开始学习这些入门的小例子的时候心里也在想，实际情况哪有这样简单：用户与 UI 交互的事件，移动设备网络类型的改变（ WIFI 与蜂窝网络的切换），服务器推送消息的到达，这些事件何时发生和产生的数量都是在运行时才能得知，怎么可能用这些工厂方法简单地就发射几个固定的值。

直到我遇见了`Subject`。我可以先创建一个一开始什么元素都不发射的`Observable`(`Subject`是`Observable`的子类)，并且同时创建对应的`Subscriber`订阅这个`Observable`，然后在我觉得某个 Ready 的时机，调用这个`Subject`对象的`onNext`方法，向它的`Subscriber`发射元素。逻辑简洁，并且足够灵活，代码如下所示：

``` java
PublishSubject<String> subject = PublishSubject.create();

subject.map(String::length)
    .subscribe(System.out::println);
...
// 在某个Ready的时机
subject.onNext("Puppy");

...
// 当某个时刻subjcet已经完成了使命
subject.onCompleted();
```
## 使用 Subject 可能导致错过真正关心的事件

到目前看来，一切都顺理成章，对比`Observable`，`Subject`优势明显，可以按需在合适的时机发射元素，似乎是`Subject`更能满足日常任务需求，更激进一点，干脆就用`Subject`来代替所有的`Observable`吧。实际上，我也这么做过，但是很快就遇到了问题。举个例子，代码如下：

``` java
PublishSubject<String> operation = PublishSubject.create();
operation
  .subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {
      System.out.println("completed");
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
      System.out.println(s);
    }
});
operation.onNext("Foo");
operation.onNext("Bar");
operation.onCompleted();
```

这段代码很简单，按照预期，它的输出为：

```
Foo
Bar
completed
```

稍微改一下，使用 RxJava 的调度器`Scheduler`指定`operation`对象从 IO 线程发射元素，代码如下（本文中的代码都是从`main`函数启动运行的）：

``` java
PublishSubject<String> operation = PublishSubject.create();
operation
  .subscribeOn(Schedulers.io())
  .subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {
      System.out.println("completed");
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
      System.out.println(s);
    }
});
operation.onNext("Foo");
operation.onNext("Bar");
operation.onCompleted();
sleep(2000);
```
以上代码实际输出的结果为：
```
completed
```
上面这段代码中，除了加上调度器以外，最后还增加了一行代码使当前线程休眠 2 秒，原因是`operation`对象改从 IO 线程发射元素以后，main 线程由于运行到最后一行直接退出了，导致整个进程结束，此时 IO 线程还没有开始发射元素，所以这 2 秒是用来等待 IO 线程启动起来并把该做的事情做完。

经过改动后的代码并没有接收到`Foo`和`Bar`，如果把最后一行`sleep(2000)`去掉，那么 Console 将不会输出任何内容。这便是我们需要谨慎使用`Subject`的第一个理由： **使用 Subject 可能导致错过真正关心的事件**。

在`RxJava`中，`Observable`可以被分为`Hot Observable`与`Cold Observable`,引用《Learning Reactive Programming with Java 8》中一个形象的比喻（翻译后的意思）：
> 我们可以这样认为，`Cold Observable`在每次被订阅的时候为每一个`Subscriber`单独发送可供使用的所有元素，而`Hot Observable`始终处于运行状态当中，在它运行的过程中，向它的订阅者发射元素（发送广播、事件），我们可以把`Hot Observable`比喻成一个电台，听众从某个时刻收听这个电台开始就可以听到此时播放的节目以及之后的节目，但是无法听到电台此前播放的节目，而`Cold Observable`就像音乐 CD ，人们购买 CD 的时间可能前后有差距，但是收听 CD 时都是从第一个曲目开始播放的。也就是说同一张 CD ，每个人收听到的内容都是一样的， 无论收听时间早或晚。

`Subjcet`是属于`Hot Observable`的。`Cold Observable`可以转化为`Hot Observable`, 但是反过来却不行。回过头来解释上面的例子为什么最后只输出了`completed`: 因为`operation`对象发射元素的线程被指派到了 IO 线程，相应的Subscriber也工作在 IO 线程，而 IO 线程第一次被Scheduler调用，还没起来（正在初始化），发射前两个元素`Foo`,`Bar`是在主线程，主线程的这两个元素往 IO 线程转发的过程中由于 IO 线程还没有起来，就被丢弃了（电台即使没有一个听众，照样可以播音）。`complete`信号比较特殊，在`Reactive X`的设计中，该信号优先级非常高，所以总是可以被优先捕获，不过这是另外一个话题。

所以使用`Subject`的时候，我们必须小心翼翼地设计程序，确保消息发送的时机是在`Subscriber`已经Ready的时候，否则我们就很容易错过我们关心的事件，当代码今后面临重构的时候，其他的程序员也必须知道这个逻辑，否则就很容易引入 Bug 。如果我们不希望错过任何事件，那么我们应该尽可能使用`Cold Observable`，上面的例子如果`operation`对象使用`Observable.just`, `Observable.from`来构造，就不会有这种问题了。

其实，错过事件这种情况一般发生在临界条件下，比如我刚刚声明一个`Subscriber`并且希望立即发送一个事件给它。这时候最好不要使用`Subject`而是使用`Observable.create(OnSubscribe)`。上面有问题的代码改成下面这样, 就可以正常工作了：

``` java
Observable<String> operation = Observable.create(subscriber -> {
  subscriber.onNext("Foo");
  subscriber.onNext("Bar");
  subscriber.onCompleted();
});
operation
  .subscribeOn(Schedulers.io())
  .subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {
      System.out.println("completed");
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
      System.out.println(s);
    }
});
sleep(2000);
```
## Subjcet 不是线程安全的                      

使用`Subject`的第二个问题便是它 **不是线程安全的** ，如果在不同的线程调用它的`onNext`方法，很有可能造成竞态条件（race conditions），我们应该尽可能避免这种情况的出现，因为除非在代码中写足够详细的注释，否则日后维护这段代码的程序员很可能莫名其妙地踩了坑。如果你认为你确实有必要使用`Subject`, 那么请把它转化为`SerializedSubject`，它可以保证如果多个线程同时调用`onNext`方法，依然是线程安全的。

``` java
SerializedSubject<Integer,Integer> subject =
      PublishSubject.<Integer>create().toSerialized();
```

## Subject 使事件的发送变得不可预知

最后一个我们应该谨慎对待`Subject`的原因就是它 **让事件的发送变得不可预知**。由于`Observable.create`使用的例子上面已经给出，再看另外两个工厂方法`Observable.just`和`Observable.from`的例子：

``` java
Observable<String> values = Observable.just("Foo", "Bar");
Observable myObservable = Observable.from(new String[]{"Foo","Bar"});
```
无论是`Observable.create`, `Observable.from` 还是 `Observable.just` , 这些 `Cold Observable` 都有一个显著的优点就是数据的来源可预知，我知道将会发送哪些数据，这些数据是什么类型。但是`Subject`就不一样，我如果创建一个`Subject`，那么代码任何地方只要能 Get 到这个引用，就可以随意使用它发射元素，滥用的后果导致代码越来越难以维护，我不知道其他人是否在某个我不知道的地方发射了我不知道的元素，我相信谁都不愿意维护这样的代码。这是一种反模式，就和 C 语言当初模块化的理念尚未深入人心的时候全局变量带来的灾难一样。

也许看到这里你会想，说了半天好像又回到起点了，`Subject`带给编程的灵活性不推荐用，为了这些理由又要重新用那三个不灵活的工厂方法，确实不能满足需求啊。我们回顾一下之前提到过的编程中经常遇到的实际情况：

> 用户与 UI 交互的事件

> 移动设备网络类型的改变（ WIFI 与蜂窝网络的切换）

> 服务器推送消息的到达

其实这些事件往往都是以注册监听器的接口提供给程序员的，我们完全可以使用`Observable.create`这个工厂方法来创建Observable：

``` java
final class ViewClickOnSubscribe implements Observable.OnSubscribe<Void> {
    final View view;

    ViewClickOnSubscribe(View view) {
        this.view = view;
    }

    @Override public void call(final Subscriber<? super Void> subscriber) {
        verifyMainThread();

        View.OnClickListener listener = new View.OnClickListener() {
            @Override public void onClick(View v) {
                if (!subscriber.isUnsubscribed()) {
                    subscriber.onNext(null);
                }
            }
        };
        view.setOnClickListener(listener);

        subscriber.add(new MainThreadSubscription() {
            @Override protected void onUnsubscribe() {
                view.setOnClickListener(null);
            }
        });
    }
}
```

以上代码来自[Jake Wharton](https://github.com/JakeWharton)的 Android 项目 [RxBinding](https://github.com/JakeWharton/RxBinding) ，目的是将 Android UI 上的用户与控件交互产生的事件转化为`Observable`提供给程序员。上面的代码思路很简单，就是当有一个`Subscriber`想要订阅`View`的点击事件的时候，就为这个`View`在 Android Framework 里注册一个点击的回调（`view.setOnClickListener(listener)`）, 每当点击事件来临的时候就去调用`Subscriber`的`onNext`方法。

我们再对比一下另一种不那么好的写法：

``` java
PublishSubject<String> subject = PublishSubject.create();
View.OnClickListener listener = new View.OnClickListener() {
  @Override public void onClick(View v) {
      subject.onNext(null);
  }
};
view.setOnClickListener(listener);
```

这里的`subject`还只是整个项目局部的代码，我们并不知道其他地方有没有把`subject`对象给怎么样，潜在的风险就是我们刚刚讨论的 **可能会错过临界情况下的事件**、 **线程不安全**、 **事件来源不可预知**。

## 总结
我们已经了解到了`Subject`给我们带来的灵活性以及风险，所以在实际项目中使用的时候我推荐更多地使用`Observable`提供的3个工厂方法，而慎重使用`Subject`，其实90%的情况都可以使用那3个工厂方法解决，如果你确定要使用`Subject`，那么确保：1. 这是一个 `Hot Observable` 且你有对应措施保证不会错过临界的事件；2. 有对应的线程安全措施；3. 模块化代码，确保事件的发送源在掌控中，事件的发送完全可预期。对了，另外加上必要的注释:)
