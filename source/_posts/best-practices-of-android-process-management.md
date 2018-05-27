---
title: 如何优雅地构建易维护、可复用的 Android 业务流程
tags: Android 业务流程
desc: 管理复杂业务流程
---

有一定实际 Android 项目开发经验的人，一定曾经在项目中处理过很多重复的业务流程。例如开发一个社交 App ，那么出于用户体验考虑，会需要允许匿名用户（不登录的用户）可以浏览信息流的内容（或者只能浏览受限的内容），当用户想要进一步操作（例如点赞）时，提示用户需要登录或者注册，用户完成这个流程才可以继续刚刚的操作。而如果用户需要进行更深入的互动（例如评论，发布状态），则需要实名认证或者补充手机号这样的流程完成才可以继续操作。

<!-- More -->

而上面列举的还只是比较简单的情况，流程之间还可以互相组合。例如：匿名用户点击了评论，那么需要连续做完：

1.  登录/注册
2.  实名认证

这两个流程才可以继续评论某条信息。另外 1 中，登录流程还可能嵌套“忘记密码”或者“密码找回”这样的流程，也有可能因为服务端检测到用户异地登录插入一个两步验证/手机号验证流程。

## 需要解决的问题

**(一) 流程的体验应当流畅**

根据本人使用市面上 App 的经验，处理业务流程按体验分类可以分为两类，一种是触发流程完成后，回到原页面，没有任何反应，用户需要再点一下刚才的按钮，或者重新操作一遍刚才触发流程的行为，才能进行原来想要的操作。另外一种是，流程完成后，如果之前不满足的某些条件此时已经满足，那么自动帮用户继续刚刚被打断的操作。显然，后一种更符合用户的预期，如果我们需要开发一个新的流程框架，那么这个问题需要被解决。

**(二) 流程需要支持嵌套**

如果在进行一个流程的过程中，某些条件不满足，需要触发一个新的流程，应当可以启动那个流程，完成操作，并且返回继续当前流程。

**(三) 流程步骤间数据传递应当简单**

传统 Activity 之间数据传递是基于 Intent 的，所以数据类型需要支持 `Parcelable` 或者 `Serializable` ，并且需要以 `key-value` 的方式往 Intent 内填充，这是有一定局限性的。此外，流程步骤间有些数据是共享的，有些是独有的，如何方便地去读写这些数据？

> 有人可能会说，那可以把这些数据放到一个公共的空间，想要读写这些数据的 Activity 自行访问这些数据。但是如果真的这样，带来的新问题是：应用进程是可能在任意时刻销毁重建的，重建以后内存中保存的这些数据也消失了。如果不希望看到这样，就需要考虑数据持久化，而持久化的数据也只是被这一次流程用到，何时应该销毁这些数据？持久化的数据需要考虑自身的生命周期的问题，这引入了额外的复杂度。且并没有比使用 Intent 传递方便多少。

**(四) 流程需要适应 Android 组件生命周期**

前面说到了应用进程销毁重建的问题，由于很多操作触发流程以后，启动的流程页面是基于 Activity 实现的，所以完成流程回到的 Activity 实例很有可能不是原来触发流程时的那个 Activity 实例，原来那个实例可能已经被销毁了，必须有合适的手段保证流程完成后，回到触发流程的页面可以正确恢复上下文。

**(五) 流程需要可以简单复用**

还有流程往往是可以复用的，例如登录流程可以在应用的很多地方触发，所以触发后流程结束以后的跳转页面也都是不一样的，不可以在流程结束的页面写死跳转的页面。

**(六) 流程页面在完成后需要比较容易销毁**

流程结束以后，流程每个步骤页面可以简单地销毁，回到最初触发流程的界面。

**(七) 流程进行中回退行为的处理**

如果一个流程包含多个中间步骤，用户进行到中间某个步骤，按返回键时，行为应该如何定义？在大多数情况下，应该支持返回上一个步骤，但是在某些情况下，也应当支持直接返回到流程起始步骤。

## 方案一：基于 startActivityForResult

其实说起流程这个事情，我们最容易想到的应该就是 Android 原生提供给我们的 [startActivityForResult](https://developer.android.com/reference/android/app/Activity.html?hl=zh-tw#startActivity(android.content.Intent%29) 方法，以 Android 官网中的一个[例子](https://developer.android.com/training/basics/intents/result?hl=zh-cn)（从通讯录中选择一个联系人）为例：

```java
static final int PICK_CONTACT_REQUEST = 1;  // The request code
...
private void pickContact() {
    Intent pickContactIntent = new Intent(Intent.ACTION_PICK, Uri.parse("content://contacts"));
    pickContactIntent.setType(Phone.CONTENT_TYPE); // Show user only contacts w/ phone numbers
    startActivityForResult(pickContactIntent, PICK_CONTACT_REQUEST);
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    // Check which request we're responding to
    if (requestCode == PICK_CONTACT_REQUEST) {
        // Make sure the request was successful
        if (resultCode == RESULT_OK) {
            // The user picked a contact.
            // The Intent's data Uri identifies which contact was selected.

            // Do something with the contact here (bigger example below)
        }
    }
}
```

在上面的例子中，当用户点击按钮（或者其他操作）时，`pickContact` 方法被触发，系统启动通讯录，用户从通讯录中选择联系人以后，回到原页面，继续处理接下来的逻辑。**从通讯录选择用户并返回结果** 就可以被看作为一个流程。

不过上面的流程是属于比较简单的情况，因为流程逻辑只有一个页面，而有时候一个复杂流程可能包含多个页面：例如注册，包含手机号验证界面（接收验证码验证），设置昵称页面，设置密码页面。假设注册流程是从登录界面启动的，那么使用 `startActivityForResult` 来实现注册流程的 Activity 任务栈的变化如下图所示：

![](/images/register-sample.png)

上图的注册流程实现细节如下：

1. 登录界面通过 `startActivityForResult` 启动注册页面的第一个界面 ---- 验证手机号；
2. 手机号验证成功后，验证手机号界面通过 `startActivityForResult` 启动设置昵称页面;
3. 昵称检查合法后，昵称信息通过 `onActivityResult` 返回给验证手机号界面，验证手机号界面通过 `startActivityForResult` 启动设置密码界面，由于设置密码是最后一个流程，验证手机号界面把之前收集好的手机号信息，昵称信息都一并传递给密码界面，密码检查合法后，根据现有的手机号、昵称、密码发起注册；
4. 注册成功后，服务器返回注册用户信息，设置密码界面通过 `onActivityResult` 把注册结果反馈给设置手机号界面;
5. 注册成功，设置手机号界面结束自己，同时把注册成功信息通过 `onActivityResult` 反馈给流程发起者(本例中即登录界面);

通过这个例子可以看出来，**手机号验证界面** 不仅承担了在注册流程中验证手机号的功能，还承担了**注册流程对外的接口的职责**。也就是说，触发注册流程的任意位置，都不需要对注册流程的细节有任何了解，而只需要通过 `startActivityForResult` 和 `onActivityResult` 与流程对外暴露的 Activity 交互即可，如下图：

![](/images/register-sample-2.png)

上面的例子中可能有一点令您疑惑：为什么每个步骤需要返回到验证手机号页面，然后由验证手机号页面负责启动下个步骤呢？一方面，由于验证手机号是流程的第一个页面，它承担了流程调度者的身份，所以由它来进行步骤的分发，这样的好处是每个步骤（除了第一步）之间是解耦和内聚的，每个步骤只需要做好自己的事情并且通过 `onActivityResult` 返回数据即可，假如后续流程的步骤发生增删，维护起来比较简单；另一方面，由于每个步骤做完都返回，当最后一个步骤做完以后，之前流程的中间页面都不存在了，不需要手动去销毁做完的流程页，这样编码起来也比较方便。

但是这么做带来一个小小的副作用：如果在流程的中间步骤按返回键，就会回到流程的第一个步骤，而用户有时候是希望可以回到上一个步骤。为了让用户可以在按返回键的时候返回上一个步骤，就必须要把每个步骤的 Activity 压栈，但是这样做的话最后一步做完之后如何销毁流程相关的所有 Activity 又是一个问题。


为了解决流程相关 Activity 的销毁问题，需要对上面的图做一点修改，如下：

![](/images/register-sample-3.png)

原先，每个步骤做完自己的任务以后只需要结束自己并返回结果，修改后，每个步骤做完自己的任务后不结束自己，也不返回结果，同时需要负责启动流程的下一个步骤（通过 `startActivityForResult`），当它的下一个步骤结束并返回它的结果的时候，这个步骤能在自己的 `onActivityResult` 里接住，它在`onActivityResult`里需要做的是把自己的结果和它的下一个步骤的结果合在一起，传递给它的上一个步骤，并结束自己。

通过这样，实现了用户按返回键所需要的行为，但是这种做法的缺点是造成了流程内步骤间的耦合，一方面是启动顺序之间的耦合，另一方面由于**需要同时携带它下个步骤的结果并返回**造成的数据的耦合。

除此以外我还见过有人会单独使用一个栈，来保存流程中启动过的 Activity , 然后在流程结束后自己去手动依次销毁每个 Activity。我不太喜欢这种方法，它相比上面的方法没有解决实质问题，而且需要额外维护一个数据结构，同时还要考虑生命周期，得不偿失。

最后总结一下前文， `startActivityForResult` 这个方法有着它自己的优势：

> 1.  足够简单，原生支持。
2.  可以处理流程返回结果，继续处理触发流程前的操作。
3.  流程封装良好，可复用。
4.  虽然引入了额外的 `requestCode`，但是在某种程度上保留了请求的上下文。

但是这个原生方案存在的问题也是显而易见的：

> 1.  写法过于 Dirty，发起请求和处理结果的逻辑被分散在两处，不易维护。
2.  页面中如果存在的多个请求，不同流程回调都被杂糅在一个 `onActivityResult` 里，不易维护。
3.  如果一个流程包含多个页面，代码编写会非常繁琐，显得力不从心。
4.  流程步骤间数据共享基于 Intent，没有解决 **问题(三)**。
5.  流程页面的自动销毁和流程进行中回退行为存在矛盾，**问题(六)** 和 **问题(七)** 没有很好地解决。

实际开发中，这几个问题都非常突出，影响开发效率，所以无法直接拿来使用。

## 方案二：EventBus 或者其他基于事件总线的解决方案

基于事件解耦也是一种比较优雅的解决方案，尤其是著名的 EventBus 框架了，它实现了非常经典的发布订阅模型，完成了出色的解耦：
![](/images/EventBus-Publish-Subscribe.png)

我相信很多 Android 开发者都曾经很愉快地使用过这个框架……………………………………最后放弃了它，或者只在小范围使用它。比如说我，目前已经在项目中逐渐删除使用 EventBus 的代码，并且使用 RxJava 作为替代。

通过具体的代码一窥 EventBus 的基本用法：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    EventBus.getDefault().register(this);
    EventBus.getDefault().post(new MessageEvent("hello","world"));

}

@Subscribe(threadMode = ThreadMode.MainThread)
public void helloEventBus(MessageEvent message){
    mText.setText(message.name);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    EventBus.getDefault().unregister(this);
}

class MessageEvent {
  public final String name;
  public final String password;
  public MessageEvent(String name, String password) {
    this.name = name;
    this.password = password;
  }
}
```

那它有什么不足之处呢？首先，发起一个一般的异步任务，开发者期望在回调中得到的是 **这个任务** 的结果，而在 EventBus 的概念中，回调中传递的是“事件”（例子中的 MessageEvent）。这里稍稍有点不同，理论上，异步任务的结果的数据类型可以就是事件的数据类型，这样两个概念就统一了，然而实际中还是有很多场合无法这样处理, 举个例子：A Activity 和 B Activity 都需要请求一个网络接口，如果把网络请求的响应的对象类型直接作为事件类型提供给它们的 Subscriber，就会产生混乱，如下图。

![](/images/event-bus-weakness-1.png)

图中，A Activity 和 B Activity 都发起同一个网络请求（可能参数不同，例如查天气接口，一个是查北京的天气，另一个是查上海的天气），那么他们的响应结果类是一样的，如果把这个响应结果直接作为事件类型提供给 EventBus 的回调，那么造成的结果就是两个 Activity 都收到两次消息。我把它称为 **事件传播在空间上引起的混乱**。

解决的方案通常是封装一个事件，把 Response 作为这个事件携带的数据：

```java
public class ResponseEvent {
    String sender;
    Response response;
}
```

在把响应对象封装成事件之后，加入了一个 `sender` 字段，用来区分这个响应应该对应哪个 Subscriber ，这样就解决了上述问题。

不仅仅在空间上， **事件传播还可以在时间上引起混乱**，想象一种情况，如果先后发起两个相同类型的请求，但是处理他们的回调是不同的。如果用传统的设置回调的方法，只要给这两个请求设置两个回调就可以了，但是如果使用 EventBus ，由于他们的请求类型相同，所以他们数据返回类型也相同，如果直接把返回数据类型当成事件类型，那么在 EventBus 的事件处理回调中无法区分这两个请求（无法保证一先一后的两个请求一定也是一先一后返回）。解决的方案也类似上面的方案，只要把 `sender` 这个字段换成类似 `timestamp` 这样的字段就可以了。

归根结底，事件传播在空间和时间上引起混乱的深层次原因是，把传统的“为每个异步请求设置一个回调”这种模式，变成了“设置一个回调，用来响应某一种事件”这种模式。传统的方式是一个具体的请求和一个具体的回调之间是强关联，一个具体的回调服务于一个具体的请求，而 EventBus 把两者给解耦开了，**回调和请求之间是弱关联，回调只和事件类型之间是强关联**。

除了上面的问题，事实上还有一个更严峻的问题，具体代码：

```java
// File: ActivityA.java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_a);

    EventBus.getDefault().register(this);

    findViewById(R.id.start).setOnClickListener(
        v -> startActivity(new Intent(this, ActivityB.class))
    )
}

@Subscribe(threadMode = ThreadMode.MainThread)
public void helloEventBus(MessageEvent message){
    mText.setText(message.name);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    EventBus.getDefault().unregister(this);
}

........

// File: ActivityB.java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_b);

    findViewById(R.id.btn).setOnClickListener(v -> {
        EventBus.getDefault().post(new MessageEvent("hello","world"));
        finish();
    })
}
```

上述代码的意图主要是：在 ActivityA 点击按钮，启动 ActivtyB， ActivtyB 承载了一个业务流程，当 ActivityB 所承担的流程任务完成以后，点击它页面内的按钮，结束流程，把结果数据通过 EventBus 传递回 ActivityA ，同时结束自己，把用户带回 ActivityA。

理想情况下，这样是没有问题，但是如果在开启了 Android 系统中的 “**开发者选项** - **不保留活动**”选项以后，ActivityA 不会收到来自 ActivityB 的任何消息。“**不保留活动**”这个选项其实是模拟了当系统内存不足的时候，会销毁 Activity 栈中用户不可见的 Activity 这一特性。这点在低端机中非常常见，经常玩着玩着手机，突然接个电话，回来发现整个页面都重新加载了。那么原因已经显而易见了：因为 ActivityA 在被系统销毁的时候执行了 `onDestroy`，从 EventBus 中移除了自身回调，因此无法接收到来自 ActivityB 的回调了。能不能不移除回调呢？当然是不能，因为这样会造成内存泄漏，更糟。

熟悉 EventBus 的朋友应该对 `postSticky` 这个方法不陌生，确实，在这种情况下，`postSticky` 这个方法可以让事件多存活一段时间，直到它的消费者出现把它消费掉。但是这个方法也有一些副作用，使用`postSticky`发送的事件需要由 Subscriber 手动把事件移除，这就导致，如果事件有多个消费者，那写代码的时候就不知道应该在什么时候把事件移除，需要增加一个计数器或者别的什么手段，引入了额外的复杂度。`postSticky`的事件只是为了保证 Activity 重走生命周期后内部回调依然可以收到事件，却污染了全局的空间，这种做法我觉得非常不优雅。

写到这里，这篇文章快成了 EventBus 批判文了，其实 EventBus 本身没问题，只是我们使用者要考虑场景，不能滥用，还是有些场合比较适用的，但是对于业务流程处理这个任务来说，我并不认为这是一个很好的应用场景。

上述陈述中，很多例子我都使用了“异步任务”作为例子来阐述，主要是我认为其实在用户操作中我们插入的业务流程也可以视为一种异步任务，反正最后结果都是异步返回给调用者的。所以我认为 EventBus 不适合异步任务的那些点，同样不适合业务流程。

其他的事件总线解决方案基本类似，Android 原生的 Broadcast 如果不考虑它的跨进程特性的话，在处理业务流程这件事情上基本可以认为是个低配版的 EventBus ，所以这里不再赘述。

## 方案三：FLAG_ACTIVITY_CLEAR_TOP 或许是一种方案

由于考虑使用第三方的框架始终无法避开 Android 生命周期的问题（上一节 EventBus 案例中 Activity 的销毁与重建丢失上下文的例子）。我们还是倾向于从 Android 原生框架中寻找符合我们要求的功能组件。这时我从 `Intent Flags` 中找到了 `FLAG_ACTIVITY_CLEAR_TOP`, 官方文档在[这里](https://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_CLEAR_TOP), 我不打算照搬文档，但是想把其中一个例子翻译一下：

> 如果一个 Activity 任务栈有下列 Activity：A, B, C, D. 如果这时 D 调用 `startActivity()`, 并且作为参数的 `Intent` 最后解析为要启动 Activity B（这个 `Intent` 中包含 `FLAG_ACTIVITY_CLEAR_TOP` ）, 那么 C 和 D 都会销毁，B 会接收到这个 `Intent`, 最后这个任务栈应该是这样：A, B。

这段只描述了现象，[文档](https://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_CLEAR_TOP)中还描述了更细节的数据流动，建议仔细阅读消化文档描述，我只把其中最重要的一块单独翻译一下：

> 上面的例子中的 B 会是下面两种结果之一
> 1.  在 `onNewIntent` 回调中接收到来自 D 传递过来的 Intent
> 2.  B 会销毁重建， 而重建的 `Intent` 就是由 D 传递过来的那个 `Intent`

> 如果 B 的 `launchMode` 被申明为 `multiple`(即`standard`)**且** Intent Flags 中**没有** `FLAG_ACTIVITY_SINGLE_TOP`， 那么就是上面的结果2。剩下的情况（`launchMode` 被申明为**非** `multiple` **或** Intent Flags 中**有** `FLAG_ACTIVITY_SINGLE_TOP`），就是结果1.

上面的描述中，B 的结果1 就很适合我们业务流程的封装，为什么这么说呢，这里举一个例子。背景：一个社交 App, 首页信息流。假设所有 Activity 都在一个任务栈中，那么这个任务栈的变化如下图所示：

![](/images/login-sample.png)

(1) 匿名用户浏览了一会，进行了一次点赞操作，此时触发登录流程，登录界面被弹出来；
(2) 用户输完正确的用户名密码后（假设为老用户），服务器接收到登录请求后，检测到风险，发起两步验证（需要短信验证），客户端弹出短信验证页面进行验证;
(3) 用户输入正确的验证码，点击登录，回到信息流页面，同时页面上点赞操作已经成功。

如何实现第3步中描述的现象呢？ 只要在 Activity C 里面，登录成功的逻辑里添加启动 Activity A 的逻辑，同时给这个启动 Activity A 的 Intent 同时加上 `FLAG_ACTIVITY_CLEAR_TOP` `FLAG_ACTIVITY_SINGLE_TOP` 两个 Intent Flag 即可（所有 Activity 的 `launchMode` 均为 `standard`）, 代码如下：

```java
Intent intent = new Intent(ActivityC.this, ActivityA.class);
intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP | Intent.FLAG_ACTIVITY_CLEAR_TOP);
// put your data here
// intent.putExtra("data", "result");
startActivity(intent);
```

使用这种方法，优点如下：
1. 可以把用户重新带回到触发流程之前页面;
2. 有携带数据（本例中可以是用户的登录信息）的回调;
3. 在回调中可以帮助用户继续完成之前被打断的操作;
4. 流程页面全部自动销毁，甚至 Activity C 自身的 `finish()` 方法都不需要调用;
5. 即使打开**不保留活动**依然有效，Acitivity A 的 `onNewIntent` 回调会在 `onCreate` 之后被调用;
6. 对流程进行一半时的页面回退支持良好;

看上去这种方法似乎比 **方案一** 要好很多， 但是其实上面的例子还是有点问题：最后一步 Activity C 显式启动了 Activity A。流程页不应该和触发流程的页面发生任何耦合，不然流程就无法复用，所以应该想一种机制，可以让两者不耦合，同时又可以把流程完成后携带的数据传递给流程触发的地方。目前能想到比较合适的手段就是 **方案一** 中的 `startActivityForResult`了，具体做法是，Activity A 只和 Activity B 通过 `startActivityForResult` 和 `onActivityResult` 进行交互，流程最后一个页面则通过上述的 `onNewIntent` 把流程结束相关数据带回流程第一个页面（Activity B），由 Activity B 通过 `onActivityResult` 把数据传递给流程触发者，具体逻辑如下图所示：

![](/images/login-sample-3.png)

这样流程封装和复用的问题解决了,但是这个方案还是存在一些缺点：

1. 和 `startActivityForResult` 一样，写法 Dirty，如果流程很多，维护较为不易;
2. 即使是同一个流程，在同一个页面中也存在复用的情况，不增加新字段无法在 `onNewIntent` 里面区分;
3. **问题(三)** 没有得到解决，`onNewIntent` 数据传递也是基于 Intent 的, 也没有用于步骤间共享数据的措施，共享的数据可能需要从头传到尾;
4. 步骤之间有轻微的耦合：每个步骤需要负责启动它的下一个步骤;

其中缺点2解释一下，点赞会触发登录，评论也会触发登录，两者登录成功都会返回信息流页面。不增加额外字段，`onNewIntent` 只是接收到了用户的登录信息，并不知道刚刚进行的是点赞还是评论。

这个方案和纯 `startActivityForResult` 的方案（**方案一**）有一种互补的感觉，一个擅长流程页不支持回退的情况，另一种擅长流程页支持回退的情况，而且它们都没有很好解决 **问题(三)** , 我们需要进一步探索是否有更优方案。

## 方案四：利用新开辟的 Activity 栈来完成业务流程

由于我们目前接手的项目中的流程页面，都是基于 Activity 实现的，那么自然而然就能想到应该让处理流程的 Activity 们更加内聚，如果流程相关 Activity 都是在一个独立的 Activity 任务栈中，那么当流程处理完以后，只要在拿到流程的最终结果以后销毁那个任务栈即可，简单又粗暴。

如果依然使用上面那个信息流登录的例子的话，Activity 任务栈的变化应该如下图所示：

![](/images/login-sample-2.png)

要实现图中的效果，那么需要考虑两个问题：

>1. 如何开启一个新的任务栈，把涉及流程的 Activity 都放到里面？
>2. 如何在流程结束以后销毁流程占用的任务栈，同时把流程结果返回到触发流程的页面？

问题1相对而言比较简单，我们把流程相关的所有 Activity 显式设置 `taskAffinity` （例如 com.flowtest.flowA）, 注意不要和 Application 的 packageName 相同，因为 Activity 默认的 `taskAffinity` 就是 packageName。启动流程的时候，在启动流程入口 Activity 的 `Intent` 中增加 `FLAG_ACTIVITY_NEW_TASK` 即可：

```xml
<!-- In AndroidManifest.xml -->
<activity android:name=".ActivityB" android:taskAffinity="com.flowtest.flowA"/>
```

```java
// In ActivityA.java
Intent intent = new Intent(ActivityA.this, ActivityB.class);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```
而流程中的其他 Activity 的启动方式不需要做任何修改，因为它们的 `taskAffinity` 与 流程入口 Activity 相同，所以它们会被自动置入同一个任务栈中。

问题2稍微复杂一点，据我所知，目前 Android 还没有提供在任务栈之间相互通信的手段，那我们只能回过头继续考虑 Activity 之间数据传递的方法。首先，出于流程复用性考虑， 流程依然还是暴露 Activity B, 而流程触发者（Activity A） 通过 Activity B 以及`startActivityForResult` 和 `onActivityResult` 两个方法与流程交互; 其次，流程内部的步骤要和 Activity B 的交互的话，有 `onNewIntent` 以及 `onActivityResult` 这两种回调的方法。

看上去这种思路比较有希望，但是经过几次试验，我放弃了这种做法，原因是一旦开辟一个新的任务栈来处理，手机上**最近应用列表**上，就会多一个App的栏位（多出来的那个代表流程任务栈），也就是说用户在做流程的时候如果按 Home 键切换出去，那他想回来的时候，按 **最近应用列表**，他会看到两个任务，他不知道回哪个。即使流程完成， **最近应用列表** 中还会保留着那个位置，后续依然会给用户造成困惑。另外，任务栈切换时的默认动画和 Activty 默认切换动画不同（虽然可以修改成一样），会在使用过程中感觉有一丝怪异。

## 方案五：使用 Fragment 框架封装流程

到目前为止，上面各种方案中，相对能使用的方案，只有方案一和方案三。方案一中又存在一对矛盾，如果希望流程内所有步骤都能优雅销毁，步骤之间耦合更松散，就没法保证回退行为；回退行为有保证以后，流程步骤的销毁就不够优雅了，步骤之间耦合也紧一些；方案三中，流程步骤销毁的问题和回退得以优雅解决，但是步骤间的耦合没有解决。我们希望一种能够两全其美的方案，步骤之间耦合松散，回退优雅，销毁也容易。

仔细分析两种方案的优缺点，其实不难得出结论：之所以仅靠 Activity 之间交互难以达成上述目标本质上是由于 Activity 任务栈没有开放给我们足够的 API，我们与任务栈能做的事情有限。看到这里其实就容易想到 Android 中，除了 Activity ，Fragment 也是拥有 Back Stack 的，如果我们把流程页以 Fragment 封装，就可以在一个 Activity 内通过 Fragment 切换完成流程；由于 Activity 与 Fragment Back Stack 生命周期同在，Activity 就成了理想的保存 Fragment Back Stack 状态（流程状态）的理想场所；此外，只要调用 Activity 的 `finish()` 方法就可以清空 Fragment Back Stack！

仍然以登录两步验证为例，经过 Fragment 改造以后，触发流程的点只会启动一个 Activity ，并且只和这个 Activity 交互，如下图所示：
![](/images/login-sample-5.png)

Activity A 通过 `startActivityForResult` 启动 ActivityLogin，ActivityLogin 在内部通过 Fragment 把业务流程完成，`finish` 自身，并且把流程结果通过 `onActivityResult` 返回给 Activity A。流程包含的两个步骤被封装成两个 Fragment ， 它们与宿主 Activity 的交互如下图所示：
![](/images/login-sample-4.png)

1. ActivityLogin 启动流程第一个页面 ---- 密码登录，通过 `push` 方法（本例中的方法皆伪代码）把 Fragment A 展示到用户面前，用户登录密码验证成功，通过 `onLoginOk` 方法回调 ActivityLogin，ActivityLogin 保存该步骤必要信息。

2. ActivityLogin 启动流程第二个页面 ---- 两步验证，同时附带上个步骤的信息传递给 Fragment B，也是通过 `push` 方法，手机短信验证成功，通过 `onValidataOk` 方法回调 ActivityLogin， ActivityLogin 把这步的数据和之前步骤的数据打包，通过 `onActivityResult` 传递给流程触发点。 

再回过头看开头，我们对新的流程框架提出了7个待解决问题，再看本方案，我们可以发现，除了 **问题(三)** 还存疑，其余的问题应该说都得到了妥善的解决。

正常情况下，添加 Fragment 是不带有动画的，没有像 Activity 切换那样的默认动画。为了可以使 Fragment 的切换给用户的感觉和 Activity 的体验一致，我建议把 Fragment 的切换动画设置成和 Activity 一样。首先，给 Activity 指定切换动画（不同手机 ROM 的默认 Activity 切换动画不一样，为了使 App 体验一致强烈推荐手动设置切换动画）。

以向左滑动进入、向右滑动推出的动画为例，**styles.xml** 中设置主题如下： 
```xml
<!-- Base application theme. -->
<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowAnimationStyle">@style/ActivityAnimation</item>
    <!-- Customize your theme here. -->
    ...
</style>


<!-- Activity 进入、退出动画 -->
<style name="ActivityAnimation" parent="android:Animation.Activity">
    <item name="android:activityOpenEnterAnimation">@anim/push_in_left</item>
    <item name="android:activityCloseEnterAnimation">@anim/push_in_right</item>
    <item name="android:activityCloseExitAnimation">@anim/push_out_right</item>
    <item name="android:activityOpenExitAnimation">@anim/push_out_left</item>
</style>
```

定义进场和退场动画，动画文件放在 **res/anim** 文件夹下：

```xml
<!-- file: push_in_left.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate   
    android:fromXDelta="100%p"    
    android:toXDelta="0"    
    android:duration="400"/>    
</set>

<!-- file: push_in_right.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate
            android:fromXDelta="-25%p"
            android:toXDelta="0"
            android:duration="400"/>
</set>

<!-- file: push_out_right.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate
        android:fromXDelta="0"
        android:toXDelta="100%p"
        android:duration="400"/>
</set>

<!-- file: push_out_left.xml -->
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate
            android:fromXDelta="0"
            android:toXDelta="-25%p"
            android:duration="400"/>
</set>
```

所以，加上 Fragment 的切换动画以后，上面的 `push` 方法的实现如下：
```java
    protected void push(Fragment fragment, String tag) {
        List<Fragment> currentFragments = fragmentManager.getFragments();
        FragmentTransaction transaction = fragmentManager.beginTransaction();
        if (currentFragments.size() != 0) {
            // 流程中，第一个步骤的 Fragment 进场不需要动画，其余步骤需要
            transaction.setCustomAnimations(
                    R.anim.push_in_left,
                    R.anim.push_out_left,
                    R.anim.push_in_right,
                    R.anim.push_out_right
            );
        }
        transaction.add(R.id.fragment_container, fragment, tag);
        if (currentFragments.size() != 0) {
            // 从流程的第二个步骤的 Fragment 进场开始，需要同时隐藏上一个 Fragment，这样才能看到切换动画
            transaction
                    .hide(currentFragments.get(currentFragments.size() - 1))
                    .addToBackStack(tag);
        }
        transaction.commit();
    }
```

每个代表流程中一个具体步骤的 Fragment 的职责也是清晰的：收集信息，完成步骤，并把该步骤的结果返回给宿主 Activity。该步骤本身不负责启动下一个步骤，与其他步骤之间也是松耦合的，一个具体的例子如下：

```java
public class PhoneRegisterFragment extends Fragment {

    PhoneValidateCallback mPhoneValidateCallback;

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_simple_content, container, false);
        Button button = view.findViewById(R.id.action);
        EditText input = view.findViewById(R.id.input);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                if (mPhoneValidateCallback != null) {
                    mPhoneValidateCallback.onPhoneValidateOk(input.getText().toString());
                }
            }
        });
        return view;
    }

    public void setPhoneValidateCallback(PhoneValidateCallback phoneValidateCallback) {
        mPhoneValidateCallback = phoneValidateCallback;
    }

    public interface PhoneValidateCallback {
        void onPhoneValidateOk(String phoneNumber);
    }
}
```

这时候，作为一系列流程步骤的宿主 Activity 的职责也明确了：

1. 作为流程对外暴露的接口，对外数据交互（`startActivityForResult` 和 `onActivityResult`）
2. 负责流程步骤的调度，决定步骤间调用的先后顺序
3. 流程步骤间数据共享的通道

举一个例子，注册流程由3个步骤组成：验证手机号、设置昵称、设置密码，流程 Activity 如下所示：
```java
public class RegisterActivity extends BaseActivity {

    String phoneNumber;
    String nickName;
    User mUser;

    PhoneRegisterFragment mPhoneRegisterFragment;
    NicknameCheckFragment mNicknameCheckFragment;
    PasswordSetFragment mPasswordSetFragment;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        /**
         * 保证无论 Activity 无论在首次启动还是销毁重建的情况下都能获取正确的 
         * Fragment 实例
         */
        mPhoneRegisterFragment = findOrCreateFragment(PhoneRegisterFragment.class);
        mNicknameCheckFragment = findOrCreateFragment(NicknameCheckFragment.class);
        mPasswordSetFragment = findOrCreateFragment(PasswordSetFragment.class);

        // 如果是首次启动，把流程的第一个步骤代表的 Fragment 压栈
        if (savedInstanceState ==  null) {
            push(mPhoneRegisterFragment);
        }

        // 负责验证完手机号后启动设置昵称
        mPhoneRegisterFragment.setPhoneValidateCallback(new PhoneRegisterFragment.PhoneValidateCallback() {
            @Override
            public void onPhoneValidateOk(String phoneNumber) {
                RegisterActivity.this.phoneNumber = phoneNumber;

                push(mNicknameCheckFragment);
            }
        });

        // 设置完昵称后启动设置密码
        mNicknameCheckFragment.setNicknameCheckCallback(new NicknameCheckFragment.NicknameCheckCallback() {
            @Override
            public void onNicknameCheckOk(String nickname) {
                RegisterActivity.this.nickName = nickName;

                mPasswordSetFragment.setParams(phoneNumber, nickName);
                push(mPasswordSetFragment);
            }
        });

        // 设置完密码后，注册流程结束
        mPasswordSetFragment.setRegisterCallback(new PasswordSetFragment.PasswordSetCallback() {
            @Override
            public void onRegisterOk(User user) {
                mUser = user;
                Intent intent = new Intent();
                intent.putExtra("user", mUser);
                setResult(RESULT_OK, intent);
                finish();
            }
        });
    }
}
```

其中 `findOrCreateFragment` 方法的实现如下：

```java
public  <T extends Fragment> T findOrCreateFragment(@NonNull Class<T> fragmentClass) {
    String tag = fragmentClass.fragmentClass.getCanonicalName();
    FragmentManager fragmentManager = getSupportFragmentManager();
    T fragment = (T) fragmentManager.findFragmentByTag(tag);
    if (fragment == null) {
        try {
            fragment = fragmentClass.newInstance();
        } catch (InstantiationException e) {
                e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
    return fragment;
}
```

看到这里，您也许会对 `findOrCreateFragment` 这个方法实现有一定疑问，主要是针对使用 `Class.newInstance` 这个方法对 Fragment 进行实例化这行代码。通常来说，Google 推荐在 Fragment 里自己实现一个 `newInstance` 方法来负责对 Fragment 的实例化，同时，Fragment 应该包含一个无参构造函数，Fragment 初始化的参数不应该以构造函数的参数的形式存在，而是应该通过 `Fragment.setArguments` 方法进行传递，符合上面要求的 `newInstance` 方法应该形如：

```java
public static MyFragment newInstance(int someInt) {
    MyFragment myFragment = new MyFragment();

    Bundle args = new Bundle();
    args.putInt("someInt", someInt);
    myFragment.setArguments(args);

    return myFragment;
}
```

> 因为使用 Fragment.setArguments 方法设置的参数，可以在 Activity 销毁重建时（重建过程也包含重建原来 Activity 管理的那些 Fragment），传递给那些被 Activity 恢复的 Fragment。

但是这边的代码为什么要这么处理呢？首先，Activity 只要进入后台，就有可能在某个时刻被杀死，所以当我们回到某个 Activity 的时候，我们应该有意识：这个 Activity 可能是刚刚离开前的那个 Activity，也有可能是已经被杀死，但是重新被创建的新 Activity。如果是重新被创建的情况，那么之前 Activity 内的状态可能已经丢失了。也就是说对于给每个流程步骤的 Fragment 设置的回调（`setPhoneValidateCallback`、 `setNicknameCheckCallback` 、 `setRegisterCallback`）有可能已经无效了，因为 Activity 重新创建以后，内存中是一个新的对象，这个对象只经历了 `onCreate`、`onStart`、`onResume` 这些回调，如果给 Fragment 设置回调的调用不在这些生命周期函数里，那么这些状态就已经丢失了（可以通过 **开发者选项里** 的 **不保留活动** 选项进行验证）。

但是有一个解决方法，就是把设置 Fragment 回调的调用写在 Activity 的 `onCreate` 函数里（因为无论是全新的 Activity 还是重建的 Activity 都会走 `onCreate` 生命周期），如本例中的 `onCreate` 方法的写法。但是这就要求在 `onCreate` 函数内，需要获取所有 Fragment 的实例（无论是首次全新创建的 Fragment，还是被恢复情况下，利用 FragmentManager 查找到的系统帮我们自动恢复的那个 Fragment）。

但是流程中，很常见的情况是，某个步骤启动所需要的参数，依赖于上个步骤。如果使用 Google 推荐的那个最佳实践，很显然，我们在初始化的时候需要准备好所有参数，这是不现实的，Activity 的 `onCreate` 函数里肯定没有准备好靠后的步骤的 Fragment 初始化所需要的参数。

这里就产生了一个矛盾：一方面为了保证销毁重建情况下，流程继续可用，需要在 `onCreate`
期间获得所有 Fragment 实例；另一方面，无法在 `onCreate` 期间准备好所有 Fragment 初始化所需要的参数，用来以 Google 最佳实践实例化 Fragment。

这里的解决方案就是上面的 `findOrCreateFragment` 方法，不完全使用 Google 最佳实践。利用 **Fragment 应该包含一个无参构造函数** 这一点，通过反射，实例化 Fragment。
```java
fragment = fragmentClass.newInstance();
```

利用 **Fragment 初始化的参数不应该以构造函数的参数存在，而是应该通过 `Fragment.setArguments` 方法进行传递** 这一点，在每个步骤结束的回调里启动下一个步骤的代码（本例中的 `push` 方法）之前，通过 `Fragment.setArguments` 方法传值。 `PasswordSetFragment.setParams` 的方法如下(底层就是 `Fragment.setArguments` 方法)：

```java
    public void setParams(String phone, String nickname) {
        Bundle bundle = new Bundle();
        bundle.putString("phone", phone);
        bundle.putString("nickname", nickname);
        setArguments(bundle);
    }
```

其实通过静态分析代码可以发现，调用 `push` 方法显示的 Fragment 实例，都是在 FragmentManger 中尚未存在的，也就是说，都是那些只被通过反射实例化以后，却还没有真正走过任何 Fragment 生命周期函数的 **准新 Fragment**。所以说，虽然我们代码上好像和谷歌推荐的写法不一样了，但本质上依然遵循谷歌推荐的最佳实践。

看到这里，这个通过 Fragment Back Stack 实现的流程框架的所有关键细节就都说完了。这个方案对比 **方案一** 和 **方案三** 显然是更好的方案，因为它综合了这两个方案的优点。我们来总结一下这个方案的优点：

1. 流程步骤间是解耦的，每个步骤职责清晰，只需要完成自己的事并且通知给宿主;
2. 回退支持良好，用户体验流畅;
3. 销毁流程只需要调用 Activity 的 `finish` 方法，非常轻量级;
4. 只有一个 Activity 代表这个流程暴露给外部，封装良好而且易于复用;
5. 流程步骤间数据的共享变得更简单

再回顾一下本文一开始提出的流程框架需要解决的 7 个问题，可以发现除了 **问题(三)** 没有完全解决以外，其余问题应该都是得到了较为满意的解决。我们来看一下  **问题(三)**，这个问题的提出的前提是，流程的每个步骤是基于 Activity 实现的，虽然使用基于 Fragment 的方案以后，Fragment 回调给 Activity 的数据不再受 Bundle 支持格式的限制，但是从 Activity `push` 启动 Fragment 需要先调用 `setArguments` 方法，而这个方法支持的格式依然受 Bundle 的限制。如果我们希望 Android 在 Activity 销毁后重建时正确恢复 Fragment ，我们只能接受这一点。

另外，虽然 Fragment 传递给 Activity 的数据格式不受限制了，考虑到 Activity 有可能销毁重建，为了保持 Activity 的状态，我们还是需要实现 Activity 的 `onSaveInstanceState` 方法和 `onRestoreInstanceState` 方法，而这两个方法依然是和 Bundle 打交道的：




```java
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putString("phoneNumber", phoneNumber);
        outState.putString("nickName", nickName);
        outState.putSerializable("user", mUser);
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);
        if (savedInstanceState == null) return;
        phoneNumber = savedInstanceState.getString("phoneNumber");
        nickName = savedInstanceState.getString("nickName");
        mUser = (User) savedInstanceState.getSerializable("user");
    }
```

也就是说，如果我们希望我们的 Activity/Fragment 能历经生命周期摧残，而始终以正确的姿态被系统恢复，那么我们就要保证我们的数据是能够被打包进 Bundle 的。我们牺牲了编码上的便利性，换取代码执行的正确性。所以目前看来，**问题(三)**虽然没有被我们解决或者绕过，但是其实本质上它的存在是可以被接受的。

## 总结

在探讨和比较了上面这么多方案以后，我们终于找到相对而言最适合解决方案 ---- **方案(五)：基于 Fragment 封装流程框架**。但是这还不是终点，虽然在理论指标上，这个方案满足了我们的需求，但是实际开发中，还是有一些小问题等待被解决。比如：
1. 流程对外暴露的接口是 startActivityForResult/onActivityResult，基于这个 API 进行开发，很难称得上是“优雅”；
2. 发起流程的上下文应该如何保存，`requestCode` 能保存的信息量有限，尤其是在 ListView / RecyclerView 的场合下；
3. 或许我们应该借助一个框架来帮助我们实现流程框架，而不是手写很多重复代码；

等等。

在下一篇分享中，我将继续介绍如何更优雅地去使用、封装流程框架，欢迎继续关注！
<!-- 应该在result数据中，把请求数据也带上 -->
