---
title: RxJava 沉思录（三）：时间维度
tags: RxJava
date: 2018-08-31 15:31:00
desc: 重新认真思考 RxJava
---

本文是 "RxJava 沉思录" 系列的第三篇分享。本系列所有分享：

+ [RxJava 沉思录（一）：你认为 RxJava 真的好用吗？](/2018/08/29/thoughts-in-rxjava-1/)
+ [RxJava 沉思录（二）：空间维度](/2018/08/30/thoughts-in-rxjava-2/)
+ [RxJava 沉思录（三）：时间维度](/2018/08/31/thoughts-in-rxjava-3/)
+ [RxJava 沉思录（四）：总结](/2018/09/01/thoughts-in-rxjava-4/)

在上一篇分享中，我们应该已经对 **Observable 在空间维度上重新组织事件的能力** 印象深刻了，那么自然而然的，我们容易联想到时间维度，事实上就我个人而言，我认为 **Observable 在时间维度上的重新组织事件的能力** 相比较其空间维度的能力更为突出。与上一篇类似，本文接下来将通过列举真实的例子来阐述这一论点。

<!-- More -->

## Observable 在时间维度上的重新组织事件的能力

*情景一：点击事件防抖动*
 
这是一个比较常见的情景，用户在手机比较卡顿的时候，点击某个按钮，正常应该启动一个页面，但是手机比较卡，没有立即启动，用户就点了好几下，结果等手机回过神来的时候，就会启动好几个一样的页面。

这个需求用 Callback 的方式比较难处理，但是相信用过 RxJava 的开发者都知道怎么处理：

```java
RxView.clicks(btn)
    .debounce(500, TimeUnit.MILLISECONDS)
    .observerOn(AndroidSchedulers.mainThread())
    .subscribe(o -> {
        // handle clicks
    })
```

> `debounce` 操作符产生一个新的 `Observable`, 这个 `Observable` 只发射原 `Observable` 中时间间隔小于指定阈值的最大子序列的最后一个元素。 [参考资料：Debounce](http://reactivex.io/documentation/operators/debounce.html)

虽然这个例子比较简单，但是它很好的表达了 **Observable 可以在时间维度上对其发射的事件进行重新组织** , 从而做到之前 Callback 形式不容易做到的事情。

*情景二：社交软件上消息的点赞与取消点赞*

点赞与取消点赞是社交软件上经常出现的需求，假设我们目前有下面这样的点赞与取消点赞的代码：

```java
boolean like;

likeBtn.setOnClickListener(v -> {
    if (like) {
        // 取消点赞
        sendCancelLikeRequest(postId);
    } else {
        // 点赞
        sendLikeRequest(postId);
    }
    like = !like;
});
```

*以下图片素材资源来自 [Dribbble](https://dribbble.com/shots/2547034-Twitter-like-button-mashup)*
![Dribbble](https://cdn.dribbble.com/users/75982/screenshots/2547034/twitter_like_button.gif)

如果你碰巧实现了一个非常酷炫的点赞动画，用户可能会玩得不亦乐乎，这个时候可能会对后端服务器造成一定的压力，因为每次点赞与取消点赞都会发起网络请求，假如很多用户同时在玩这个点赞动画，服务器可能会不堪重负。

和前一个例子的防抖动思路差不多，我们首先想到需要防抖动：

```java
boolean like;
PublishSubject<Boolean> likeAction = PublishSubject.create();

likeBtn.setOnClickListener(v -> {
    likeAction.onNext(like);
    like = !like;
});

likeAction.debounce(1000, TimeUnit.MILLISECONDS)
    .observerOn(AndroidSchedulers.mainThread())
    .subscribe(like -> {
        if (like) {
            sendCancelLikeRequest(postId);
        } else {
            sendLikeRequest(postId);
        }
    });
```

写到这个份上，其实已经可以解决服务器压力过大的问题了，但是还是有优化空间，假设当前是已赞状态，用户快速点击 2 下，按照上面的代码，还是会发送一次点赞的请求，由于当前是已赞状态，再发送一次点赞请求是没有意义的，所以我们优化的目标就是将这一类事件过滤掉：

```java
Observable<Boolean> debounced = likeAction.debounce(1000, TimeUnit.MILLISECONDS);
debounced.zipWith(
    debounced.startWith(like),
    (last, current) -> last == current ? new Pair<>(false, false) : new Pair<>(true, current)
)
    .flatMap(pair -> pair.first ? Observable.just(pair.second) : Observable.empty())
    .subscribe(like -> {
        if (like) {
            sendCancelLikeRequest(postId);
        } else {
            sendLikeRequest(postId);
        }
    });
```

> `zipWith` 操作符可以把两个 `Observable` 发射的相同序号(同为第 **x** 个)的元素，进行运算转换，得到的新元素作为新的 `Observable` 对应序号所发射的元素。[参考资料：ZipWith](http://reactivex.io/documentation/operators/zip.html)

上面的代码，我们可以看到，首先我们对事件流做了一次 `debounce` 操作，得到 `debounced` 事件流，然后我们把 `debounced` 事件流和 `debounced.startWith(like)` 事件流做了一次 `zipWith` 操作。相当于新的这个 `Observable` 中发射的第 **n** 个元素（**n >= 2**）是由 `debounced` 事件流中的第 **n** 和 第 **n-1** 个元素运算得到的（新的这个 `Observable` 中发射的第 **1** 个元素是由 `debounced` 事件流中的第 **1** 个元素和原始点赞状态 `like` 运算而来）。

运算的结果是得到一个 `Pair` 对象，它是一个双布尔类型二元组，二元组第一个元素为 **true** 代表这个事件不该被忽略，应该被观察者观察到；若为 **false** 则应该被忽略。二元组的第二个元素仅在第一个元素为 **true** 的情况下才有意义，**true** 表示应该发起一次点赞操作，而 **false** 表示应该发起一次取消点赞操作。上面提到的“**运算**”具体运算的规则是，比较两个元素，若相等，则把二元组的第一个元素置为 **false**，若不相等，则把二元组的第一个元素置为 **true**, 同时把二元组的第二个元素置为 `debounced` 事件流发射的那个元素。

随后的 `flatMap` 操作符完成了两个逻辑，一是过滤掉二元组第一个元素为 **false** 的二元组，二是把二元组转化回最初的 `Boolean` 事件流。其实这个逻辑也可由 `filter` 和 `map` 两个操作符配合完成，这里为了简单用了一个操作符。

虽然上面用了不少篇幅解释了每个操作符的意义，但其实核心思想是简单的，就是在原先 `debounce` 操作符的基础上，把得到的事件流里每个元素和它的上一个元素做比较，如果这个元素和上个元素相同（例如在已赞状态下再次发起点赞操作）, 就把这个元素过滤掉，这样最终的观察者里只会在在真正需要改变点赞状态的时候才会发起网络请求了。

我们考虑用 Callback 实现相同逻辑，虽然比较本次操作与上次操作这样的逻辑通过 Callback 也可以做到，但是 `debounce` 这个操作符完成的任务，如果要使用 Callback 来实现就非常复杂了，我们需要定义一个计时器，还要负责启动与关闭这个计时器，我们的 Callback 内部会掺杂进很多和观察者本身无关的逻辑，相比 RxJava 版本的纯粹相去甚远。

*情景三：检测双击事件*

首先，我们需要定义双击事件，不妨先规定两次点击小于 500 毫秒则为一次双击事件。我们先使用 Callback 的方式实现：

```java
long lastClickTimeStamp;

btn.setOnClickListener(v -> {
    if (System.currentTimeMillis() - lastClickTimeStamp < 500) {
        // handle double click
    }
});
```
上面的代码很容易理解，我们引入一个中间变量 `lastClickTimeStamp`, 通过比较点击事件发生时和上一次点击事件的时间差是否小于 500 毫秒，来确认是否发生了一次双击事件。那么如何通过 RxJava 来实现呢？就和上一个例子一样，我们可以在时间维度对 `Observable` 发射的事件进行重新组织，只过滤出与上次点击事件间隔小于 500 毫秒的点击事件，代码如下：

```java
Observable<Long> clicks = RxView.clicks(btn)
    .map(o -> System.currentTimeMillis())
    .share();
    
clicks.zipWith(clicks.skip(1), (t1, t2) -> t2 - t1)
    .filter(interval -> interval < 500)
    .subscribe(o -> {
        // handle double click
    });
```

我们再一次用到了 `zipWith` 操作符来对事件流自身相邻的两个元素做比较，另外这次代码中使用了 `share` 操作符，用来保证点击事件的 `Observable` 被转为 **Hot Observable**。

> 在`RxJava`中，`Observable`可以被分为`Hot Observable`与`Cold Observable`,引用《Learning Reactive Programming with Java 8》中一个形象的比喻（翻译后的意思）：我们可以这样认为，`Cold Observable`在每次被订阅的时候为每一个`Subscriber`单独发送可供使用的所有元素，而`Hot Observable`始终处于运行状态当中，在它运行的过程中，向它的订阅者发射元素（发送广播、事件），我们可以把`Hot Observable`比喻成一个电台，听众从某个时刻收听这个电台开始就可以听到此时播放的节目以及之后的节目，但是无法听到电台此前播放的节目，而`Cold Observable`就像音乐 CD ，人们购买 CD 的时间可能前后有差距，但是收听 CD 时都是从第一个曲目开始播放的。也就是说同一张 CD ，每个人收听到的内容都是一样的， 无论收听时间早或晚。

仅仅是上面这个双击检测的例子，还不能体现 RxJava 的优越性，我们把需求改得更复杂一点：如果用户在“短时间”内连续多次点击，只能算一次双击操作。这个需求是合理的，因为如果按照上面 Callback 的写法，虽然可以检测出双击操作，但是如果用户快速点击 **n** 次（间隔均小于 500 毫秒，**n >= 2**）, 就会触发 **n - 1** 次双击事件，假设双击处理函数里需要发起网络请求，会对服务器造成压力。要实现这个需求其实也简单，和上一个例子类似，我们用到了 `debounce` 操作符：

```java
Observable<Object> clicks = RxView.clicks(btn).share()

clicks.buffer(clicks.debounce(500, TimeUnit.MILLISECONDS))
    .filter(events -> events.size >= 2)
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(o -> {
        // handle double click
    });
```

> `buffer` 操作符接受一个 `Observable` 为参数，这个 `Observable` 所发射的元素是什么不重要，重要的是这些元素发射的时间点，这些时间点会在时间维度上把原来那个 `Observable` 所发射的元素划分为一系列元素的组，`buffer` 操作符返回的新的 `Observable` 发射的元素即为那些“组”。  
[参考资料: Buffer](http://reactivex.io/documentation/operators/buffer.html)

上面的代码通过 `buffer` 和 `debounce` 两个操作符很巧妙的把点击事件流转化为了我们关心的 “短时间内点击次数超过 2 次” 的事件流，而且新的事件流中任意两个相邻事件间隔必定大于 500 毫秒。

在这个例子中，如果我们想要使用 Callback 去实现相似逻辑，代码量肯定是巨大的，而且鲁棒性也无法保证。

*情景四：搜索框中需要根据用户输入内容显示搜索提示*

以支付宝为例，当在搜索框输入“蚂蚁”关键词后，下方自动刷新和关键词相关的结果：

![](/images/alipay-demo.png)

为了简化这个例子，我们不妨定义根据关键词搜索的接口如下：

```java
public interface Api {
    @GET("path/to/api")
    Observable<List<String>> queryKeyword(String keyword);
}
```

查询接口现在已经确定下来，我们考虑一下在实现这个需求的过程中需要考虑哪些因素：

1. 防止用户输入过快，触发过多网络请求，需要对输入事件做一下防抖动。
2. 用户在输入关键词过程中可能触发多次请求，那么，如果后一次请求的结果先返回，前一次请求的结果后返回，这种情况应该保证界面展示的是后一次请求的结果。
3. 用户在输入关键词过程中可能触发多次请求，那么，如果后一次请求的结果返回时，前一次请求的结果尚未返回的情况下，就应该取消前一次请求。

综合考虑上面的因素以后，我们使用 RxJava 实现的对应的代码如下：

```java
RxTextView.textChanges(input)
    .debounce(300, TimeUnit.MILLISECONDS)
    .switchMap(text -> api.queryKeyword(text.toString()))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(results -> {
        // handle results
    });
```

> `switchMap` 这个操作符与 `flatMap` 操作符类似，但是区别是如果原 `Observable` 中的两个元素，通过 `switchMap` 操作符都转为 `Observable` 之后，如果后一个元素对应的 `Observable` 发射元素时，前一个元素对应的 `Observable` 尚未发射完所有元素，那么前一个元素对应的 `Observable` 会被自动取消订阅，尚未发射完的元素也不会体现在 `switchMap` 操作符调用后产生的新的 `Observable` 发射的元素中。
[参考资料：SwitchMap](http://reactivex.io/documentation/operators/flatmap.html)

我们分析上面的代码，可以发现: `debounce` 操作符解决了问题 **1**，`switchMap` 操作符解决了问题 **2**、**3**。这个例子可以很好的说明，**RxJava 的 `Observable` 可以通过一系列操作符从时间的维度上重新组织事件，从而简化观察者的逻辑**。这个例子如果使用 Callback 来实现，肯定是十分复杂的，需要设置计时器以及一堆中间变量，观察者中也会掺杂进很多额外的逻辑，用来保证事件与事件的依赖关系。

（未完待续）

本文属于 "RxJava 沉思录" 系列，欢迎阅读本系列的其他分享：

+ [RxJava 沉思录（一）：你认为 RxJava 真的好用吗？](/2018/08/29/thoughts-in-rxjava-1/)
+ [RxJava 沉思录（二）：空间维度](/2018/08/30/thoughts-in-rxjava-2/)
+ [RxJava 沉思录（三）：时间维度](/2018/08/31/thoughts-in-rxjava-3/)
+ [RxJava 沉思录（四）：总结](/2018/09/01/thoughts-in-rxjava-4/)

___
***
---
如果您对我的技术分享感兴趣，欢迎关注我的个人公众号：麻瓜日记，不定期更新原创技术分享，谢谢！:)

![](/images/qrcode.jpg)
