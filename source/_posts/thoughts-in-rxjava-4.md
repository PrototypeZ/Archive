---
title: RxJava 沉思录（四）：总结
tags: RxJava
date: 2018-09-01 15:31:00
desc: 重新认真思考 RxJava
---

本文是 "RxJava 沉思录" 系列的最后一篇分享。本系列所有分享：

+ [RxJava 沉思录（一）：你认为 RxJava 真的好用吗？](/2018/08/29/thoughts-in-rxjava-1/)
+ [RxJava 沉思录（二）：空间维度](/2018/08/30/thoughts-in-rxjava-2/)
+ [RxJava 沉思录（三）：时间维度](/2018/08/31/thoughts-in-rxjava-3/)
+ [RxJava 沉思录（四）：总结](/2018/09/01/thoughts-in-rxjava-4/)

我们在本系列开篇中，曾经留了一个问题：RxJava 是否可以让我们的代码更简洁？作为本系列的最后一篇分享，我们将详细地探讨这个问题。承接前面两篇 “时间维度” 和 “空间维度” 的探讨，我们首先从 **RxJava 的维度** 开始说起。

<!-- More -->

## RxJava 的维度

在前面两篇分享中，我们解读了很多案例，最终得出结论：**RxJava 通过 `Observable` 这个统一的接口，对其相关的事件，在空间维度和事件维度进行重新组织，来简化我们日常的事件驱动编程**。

前文中提到：

> 有了 `Observable` 以后的 RxJava 才刚刚插上了想象力的翅膀。

RxJava 所有想象力的基石和源泉在于 `Observable` 这个统一的接口，有了它，配合我们各种各样的操作符，才可以在时间空间维度玩出花样。

我们回想一下原先我们基于 Callback 的编程范式：

```java
btn.setOnClickListener(v -> {
    // handle click event
})
```

在基于 `Callback` 的编程范式中，我们的 `Callback` 是 **没有维度** 的。它只能够 **响应孤立的事件**，即来一个事件，我处理一个事件。假设同一个事件前后存在依赖关系，或者不同事件之间存在依赖关系，无论是时间维度还是空间维度，如果我们还是继续用 `Callback` 的方式处理，我们必然需要新增许多额外的数据结构来保存中间的上下文信息，同时 `Callback` 本身的逻辑也需要修改，观察者的逻辑会变得不那么纯粹。

但是 RxJava 给我们的事件驱动型编程带来了新的思路，**RxJava 的 `Observable` 一下子把我们的维度拓展到了时间和空间两个维度**。如果事件与事件间存在依赖关系，原先我们需要新增的数据结构以及在 Callback 内写的额外的控制逻辑的代码，现在都可以不用写，我们只需要利用 `Observable` 的操作符对事件在时间和空间维度进行重新组织，就可以实现一样的效果，而观察者的逻辑几乎不需要修改。

所以如果把 RxJava 的编程思想和传统的面向 Callback 的编程思想进行对比，用一个词形容的话，那就是 **降维打击**。

这是我认为目前大多数与 RxJava 有关的技术分享没有提到的一个非常重要的点，并且我认为这才是 RxJava 最精髓最核心的思想。RxJava 对我们日常编程最重要的贡献，就是提升了我们原先对于事件驱动型编程的思考的维度，给人一种大梦初醒的感觉，和这点比起来，所谓的 “链式写法” 这种语法糖什么的，根本不值一提。

## 生产者消费者模式中 RxJava 扮演的角色

无论是同步还是异步，我们日常的事件驱动型编程可以被看成是一种 “**生产者——消费者**” 模型：
![Callback](http://prototypez.github.io/images/rxjava-graph-1.png)

在异步的情况下，我们的代码可以被分为两大块，一块生产事件，一块消费事件，两者通过 Callback 联系起来。而 Callback 是轻量级的，大多数和 Callback 相关的逻辑就仅仅是设置回调和取消设置的回调而已。

如果我们的项目中引入了 RxJava ，我们可以发现，“**生产者——消费者**” 这个模型中，中间多了一层 RxJava 相关的逻辑层：
![RxJava](http://prototypez.github.io/images/rxjava-graph-2.png)

而这一层的作用，我们在之前的讨论中已经明确，是用来对生产者产生的事件进行重新组织的。这个架构之下，生产者这一层的变化不会很大，直接受影响的是消费者这一层，由于 RxJava 这一层对事件进行了“预处理”，消费者这一层代码会比之前轻很多。同时由于 RxJava 取代了原先的 Callback 这一层，RxJava 这一层的代码是会比原先 Callback 这一层更厚。

这么做还会有什么其他的好处呢？首先最直接的好处便是代码会更易于测试。原先生产者和消费者之间是耦合的，由于现在引入了 RxJava，生产者和消费者之间没有直接的耦合关系，测试的时候可以很方便的对生产者和消费者分开进行测试。比如原先网络请求相关逻辑，测试就不是很方便，但是如果我们使用 RxJava 进行解耦以后，观察者仅仅只是耦合 `Observable` 这个接口而已，我们可以自己手动创建用于测试的 `Observable`，这些 `Observable` 负责发射 Mock 的数据，这样就可以很方便的对观察者的代码进行测试，而不需要真正的去发起网络请求。

## 取消订阅与 Scheduler

取消订阅这个功能也是我们在观察者模式中经常用到的一个功能点，尤其是在 Android 开发领域，由于 Activity 生命周期的关系，我们经常需要将网络请求与 Activity 生命周期绑定，即在 Activity 销毁的时候取消所有未完成的网络请求。

常规面向 Callback 的编程方式我们无法在观察者这一层完成取消订阅这一逻辑，我们常常需要找到事件生产者这一层才能完成取消订阅。例如我们需要取消点击事件的订阅时，我们不得不找到点击事件产生的源头，来取消订阅： 

```java
btn.setOnClickListener(null);
```

然而在 RxJava 的世界里，取消订阅这个逻辑终于下放到观察者这一层了。事件的生产者需要在提供 `Observable` 的同时，实现当它的观察者取消订阅时，它应该实现的逻辑（例如释放资源）；事件的观察者当订阅一个 `Observable` 时，它同时会得到一个 `Disposable` ，观察者希望取消订阅事件的时候，只需要通过这个接口通知事件生产者即可，完全不需要了解事件是如何产生的、事件的源头在哪里。

至此，生产者和消费者在 RxJava 的世界里已经完成了彻底的解耦。除此以外，RxJava 还提供了好用的线程池，在 **生产者——消费者** 这个模型里，我们常常会要求两者工作在不同的线程中，切换线程是刚需，RxJava 完全考虑到了这一点，并且把切换线程的功能封装成了 `subscribeOn` 和 `observerOn` 两个操作符，我们可以在事件流处理的任何时机随意切换线程，鉴于这一块已经有很多资料了，这里不再详细展开。

## 面向 Observable 的 AOP：compose 操作符

这一块不属于 RxJava 的核心 Feature，但是如果掌握好这块，可以让我们使用 RxJava 编程效率大大提升。

我们举一个实际的例子，Activity 内发起的网络请求都需要绑定生命周期，即我们需要在 Activity 销毁的时候取消订阅所有未完成的网络请求。假设我目前已经可以获得一个 `Observable<ActivityEvent>`, 这是一个能接收到 Activity 生命周期的 `Observable`（获取方法可以借鉴三方框架 [RxLifecycle](https://github.com/trello/RxLifecycle.git)，或者自己内建一个不可见 Fragment，用来接收生命周期的回调）。

那么用来保证每一个网络请求都能绑定 Activity 生命周期的代码应如下所示：

```java
public interface NetworkApi {
    @GET("/path/to/api")
    Call<List<Photo>> getAllPhotos();
}

public class MainActivity extends Activity {

    Observable<ActivityEvent> lifecycle = ...
    NetworkApi networkApi = ...

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // 发起请求同时绑定生命周期
        networkApi.getAllPhotos()
            .compose(bindToLifecycle())
            .subscribe(result -> {
                // handle results
            });
    }

    private <T> ObservableTransformer<T, T> bindToLifecycle() {
        return upstream -> upstream.takeUntil(
            lifecycle.filter(ActivityEvent.DESTROY::equals)
        );
    }
}
```

> 如果您之前没有接触过 `ObservableTransformer`, 这里做一个简单介绍，它通常和 `compose` 操作符一起使用，用来把一个 `Observable` 进行加工、修饰，甚至替换为另一个 `Observable`。

在这里我们封装了一个 `bindToLifecycle` 方法，它的返回类型是 `ObservableTransformer`，在 `ObservableTransformer` 内部，我们修饰了原 `Observable`, 使其可以在接收到 Activity 的 DESTROY 事件的时候自动取消订阅，这个逻辑是由 `takeUntil` 这个操作符完成的。其实我们可以把这个 `bindToLifecycle` 方法抽取出来，放到公共的工具类，这样任何的 Activity 内部发起的网络请求时，都只需要加一行 `.compose(bindToLifecycle())` 就可以保证绑定生命周期了，从此再也不必担心由于网络请求引起的内存泄漏和崩溃了。

事实上我们还可以有更多玩法， 上面 `ObservableTransformer` 内部的 `upstream` 对象，就是一个 `Observable`，也就是说可以调用它的 `doOnSubscribe` 和 `doOnTerminate` 方法，我们可以在这两个方法里实现 Loading 动画的显隐：

```java
private <T> ObservableTransformer<T, T> applyLoading() {
    return upstream -> upstream
        .doOnSubscribe(() -> {
            loading.show();
        })
        .doOnTerminae(() -> {
            loading.dismiss();
        });    
    );
}
```

这样，我们的网络请求只要调用两个 `compose` 操作符，就可以完成生命周期的绑定以及与之对应的 Loading 动画的显隐了：

```java
networkApi.getAllPhotos()
    .compose(bindToLifecycle())
    .compose(applyLoading())
    .subscribe(result -> {
        // handle results
    });
```

操作符 `compose` 是 RxJava 给我们提供的可以面向 `Observable` 进行 AOP 的接口，善加利用就可以帮我们节省大量的时间和精力。

## RxJava 真的让你的代码更简洁？

在前文中，我们还留了一个问题尚未解答：RxJava 真的更简洁吗？本文中列举了很多实际的例子，我们也看到了，从代码量看，有时候使用 RxJava 的版本比 Callback 的版本更少，有时候两者差不多，有时候 Callback 版本的代码反而更少。所以我们可能无法从代码量上对两者做出公正的考量，所以我们需要从其他方面，例如代码的阅读难度、可维护性上去评判了。

首先我想要明确一点，RxJava 是一个 “**夹带了私货**” 的框架，它本身最重要的贡献是提升了我们思考事件驱动型编程的维度，但是它与此同时又逼迫我们去接受了函数式编程。函数式编程在处理集合、列表这些数据结构时相比较指令式编程具有先天的优势，我理解框架的设计者，由于框架本身提升了我们对事件思考的维度，那么无论是时间维度还是空间维度，一连串发射出来的事件其实就可以被看成许许多多事件的集合，既然是集合，那肯定是使用函数式的风格去处理更加优雅。

![](http://prototypez.github.io/images/rxjava-graph-3.png)

原先的时候，我们接触的函数式编程只是用于处理静态的数据，当我们接触了 RxJava 之后，发现动态的异步事件组成的集合居然也可以使用函数式编程的方式去处理，我不由地佩服框架设计者的脑洞大开。事实上，RxJava 很多操作符都是直接照搬函数式编程中处理集合的函数，例如：`map`, `filter`, `flatMap`, `reduce` 等等。

但是，函数式编程是一把双刃剑，它也会给你带来不利的因素，一方面，这意味着你的团队都需要了解函数式编程的思想，另一方面，函数式的编程风格，意味着代码会比原先更加抽象。

比如在前面的分享中 “*实现一个具有多种类型的 RecyclerView*” 这个例子中， `combineLatest` 这个操作符，完成了原先 `onOk()` 方法、`resultTypes`、`responseList` 一起配合才完成的任务。虽然原先的版本代码不够内聚，不如 RxJava 版本的简练，但是如果从可阅读性和可维护性上来看，我认为原先的版本更好，因为我看到这几个方法和字段，可以推测出这段代码的意图是什么，可是如果是 `combineLatest` 这个操作符，也许我写的那个时候我知道我是什么意图，一旦过一段时间回来看，我对着这个这个 `combineLatest` 操作符可能就一脸懵逼了，我必须从这个事件流最开始的地方从上往下捋一遍，结合实际的业务逻辑，我才能回想起为什么当时要用 `combineLatest` 这个操作符了。

再举一个例子，在 “*社交软件上消息的点赞与取消点赞*” 这个例子中，如果我不是对这种“把事件流中相邻事件进行比较”的编码方式了如指掌的话，一旦隔一段时间，我再次面对这几个 `debounce` 、`zipWith`、`flatMap` 操作符时，我可能会怀疑自己写的代码。自己写的代码都如此，更何况大多数情况下我们需要面对别人写的代码。

这就是为什么 RxJava 写出的代码会更加抽象，**因为 RxJava 的操作符是我们平时处理业务逻辑时常用方法的高度抽象**。 `combineLatest` 是对我们自己写的 `onOk` 等方法的抽象，`zipWith` 帮我们省略了本来要写的中间变量，`debounce` 操作符替代了我们本来要写的计时器逻辑。从功能上来讲两者其实是等价的，只不过 RxJava 给我们提供了高度抽象凝练，更加具有普适性的写法。

在本文前半部分，我们说到过，有的人认为 RxJava 是简洁的，而有的人的看法则完全相反，这件事的本质在于大家对 **简洁** 的期望不同，大多数人认为的简洁指得是代码简单好理解，而高度抽象的代码是不满足这一点的，所以很多人最后发现理解抽象的 RxJava 代码需要花更多的时间，反而不 “简洁” 。认为 RxJava 简洁的人所认为的 **简洁** 更像是那种类似数学概念上的那种 **简洁**，这是因为函数式编程的抽象风格与数学更接近。我们举个例子，大家都知道牛顿第二定律，可是你知道牛顿在《自然哲学的数学原理》上发表牛顿二定律的时候的原始公式表示是什么样的吗：

![Newton's second law](http://prototypez.github.io/images/newton's-second-law.jpg)

公式中的 **p** 表示动量，这是牛顿所认为的"简洁"，而我们大多数人认为简单好记的版本是 “**物体的加速度等于施加在物体上的力除以物体的质量**”。

这就是为什么，我在前面提前下了那个结论：**对于大多数人，RxJava 不等于简洁**，有时候甚至是更难以理解的代码以及更低的项目可维护性。

而目前大多数我看到的有关 RxJava 的技术文章举例说明的所谓 “逻辑简洁” 或者是 “随着程序逻辑的复杂性提高，依然能够保持简洁” 的例子大多数都是不恰当的。一方面他们仅仅停留在 Callback 的维度，举那种依次执行的异步任务的例子，完全没有点到 RxJava 对处理问题的维度的提升这一点；二是举的那些例子实在难以令人信服，至少我并没有觉得那些例子用了 RxJava 相比 Callback 有多么大的提升。

## RxJava 是否适合你的项目

综上所述，我们可以得出这样的结论，**RxJava 是一个思想优秀的框架，而且是那种在工程领域少见的带有学院派气息和理想主义色彩的框架，他是一种新型的事件驱动型编程范式。 RxJava 最重要的贡献，就是提升了我们原先对于事件驱动型编程的思考的维度，允许我们可以从时间和空间两个维度去重新组织事件。**

此外，RxJava 好在哪，真的和“观察者模式”、“链式编程”、“线程池”、“解决 Callback Hell”等等关系没那么大，这些特性相比上面总结的而言，都是微不足道的。

我是不会用“简洁”、“逻辑简洁”、“清晰”、“优雅” 那样空洞的字眼去描述 RxJava 这个框架的，这确实是一个学习曲线陡峭的框架，而且如果团队成员整体对函数式编程认识不够深刻的话，项目的后期维护也是充满风险的。

当然我希望你也不要因此被我吓到，我个人是推崇 RxJava 的，在我本人参与的项目中已经大规模铺开使用了 RxJava。本文前面提到过：
> RxJava 是一种新的 **事件驱动型** 编程范式，它以异步为切入点，试图一统 **同步** 和 **异步** 的世界。

在我参与的项目中，我已经渐渐能感受到这种 **“天下大同”** 的感觉了。这也是为什么我能听到很多人都会说 “一旦用了 RxJava 就很难再放弃了”。

也许这时候你会问我，到底推不推荐大家使用 RxJava ？我认为是这样，如果你认为在你的项目里，Callback 模式已经不能满足你的日常需要，事件之间存在复杂的依赖关系，你需要从更高的维度空间去重新思考你的问题，或者说你需要经常在时间或者空间维度上去重新组织你的事件，那么恭喜你， RxJava 正是为你打造的；如果你认为在你的项目里，目前使用 Callback 模式已经很好满足了你的日常开发需要，简单的业务逻辑也根本玩不出什么新花样，那么 RxJava 就是不适合你的。

（完）

本文属于 "RxJava 沉思录" 系列，欢迎阅读本系列的其他分享：

+ [RxJava 沉思录（一）：你认为 RxJava 真的好用吗？](/2018/08/29/thoughts-in-rxjava-1/)
+ [RxJava 沉思录（二）：空间维度](/2018/08/30/thoughts-in-rxjava-2/)
+ [RxJava 沉思录（三）：时间维度](/2018/08/31/thoughts-in-rxjava-3/)
+ [RxJava 沉思录（四）：总结](/2018/09/01/thoughts-in-rxjava-4/)

___
如果您对我的技术分享感兴趣，欢迎关注我的个人公众号：麻瓜日记，不定期更新原创技术分享，谢谢！:)

![](http://prototypez.github.io/images/qrcode.jpg)
