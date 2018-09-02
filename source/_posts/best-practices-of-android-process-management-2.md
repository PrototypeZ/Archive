---
title: 如何优雅地构建易维护、可复用的 Android 业务流程(二)
tags: Android 业务流程
date: 2018-06-25 00:00:00
desc: 管理复杂业务流程
---

这是关于如何在 Android 中封装业务流程经验分享的第二篇，第一篇在[这里](http://prototypez.github.io/2018/04/30/best-practices-of-android-process-management/)。所谓 **业务流程** ，指的是一系列页面的集合，这些页面肩负着一个特定职责，负责和用户交互，从用户端收集信息。业务流程有时候由用户主动触发，而有时候是由于某些条件不满足而触发，当流程完成以后，有时候只是简单地回到发起流程的页面，用流程的结果更新那个页面；而有时候是继续之前 **由于触发流程而中断** 的操作；还有些时候是则是转入新的流程。

<!-- More -->

## 回顾

在[上一篇分享](http://prototypez.github.io/2018/04/30/best-practices-of-android-process-management/)中，我根据自己在公司项目中的实践，列举了七点流程框架应该解决的问题。同时也记录了在选型过程中调研和尝试的若干种方案，它们的优点与不足，以及最后为什么放弃的原因，这些方案分别是：

+ 简单的基于 `startActivityForResult`/`onActivityResult`
+ 基于 EventBus 或者其他基于事件总线的方案
+ 简单的基于 `FLAG_ACTIVITY_CLEAR_TOP` 或者设置 launchMode 为 singleTop / singleTask  
+ 开辟新的 Activity 任务栈

上一篇分享中提出的最后一个方案 -- **使用 Fragment 框架封装流程**，是我相对而言比较满意的方案，它并不是完美的方案，至少它没有全部解决我自己提出的对流程框架的七个问题，但是从现阶段来看，在复杂程度，易用性和可靠性上，这种方案已经足够满足我们日常开发所需，在我发现更好的方案之前，我认为还是值得介绍一下的:)

简单回顾一下，这种方案就是一个 Activity 对应一个流程，这个 Activity 就是这个流程对外暴露的接口，具体暴露的接口是 `startActivityForResult` 和 `onActivityResult`, 任何触发流程的位置，都只通过这两个方法，和代表流程的 Activity 进行交互。

流程的每一个步骤（页面），被封装为一个个 Fragment, Fragment 只和宿主 Activity 交互，通常就是把本步骤的数据传递给宿主 Activity 以及通知宿主 Activity 本步骤已做完。宿主 Activity 即流程 Activity。

流程 Activity 除了担当本流程对外接口任务以外，还要承担流程内部步骤间的流转，其实就是对代表步骤的 Fragment 的添加与移除。

以登录流程为例（包含两个步骤：用户名密码验证、需要手机验证码的两步验证）, 整个流程与流程触发点（例如首页信息流点赞操作）的交互以及流程内部 Fragment 栈如下图所示：

![](/images/login-sample-5.png)

而流程宿主 Activity 与代表流程的每个具体步骤的 Fragment 的交互可以用下图来表示：

![](/images/login-sample-4.png)

## 如何优化流程与外部交互接口

上一篇分享中，最后有提到基于 `startActivityForResult` 和 `onActivityResult` 两个方法来发起流程和接收流程结果是 **不优雅** 的，而且这种写法也不利于流程在其他位置被复用。例如登录操作在点赞时可能被触发，在评论时也可能被触发，常规写法只会让 Activity / Fragment 过于臃肿。

我们希望的结果是，发起流程可以被封装为一个普通的异步操作，然后我们就可以像对待普通的异步任务那样，为这个流程指派一个观察者来监听异步结果。但是封装的难点在于我们并不容易在 `startActivityForResult` 的位置获取一个对象，这个对象可以在 `onActivityResult` 的时候获得回调，根源在于 `onActivityResult` 并不属于 Activity / Fragment 的生命周期函数，所以无论是 Google 官方的 [Lifecycle Component](https://developer.android.com/topic/libraries/architecture/lifecycle) 还是第三方的 [RxLifecycle](https://github.com/trello/RxLifecycle) 都不包含这个回调。

但是我们还是有机会通过别的办法在 `startActivityForResult` 的位置拿到 `onActivityResult` 这个回调的观察者。其中一种方案就是借鉴 [Glide](https://github.com/bumptech/glide.git) 的思想，Glide 可以为异步操作绑定 Activity 的生命周期，它的原理就是为发起请求的 Activity 添加一个 **不可见的 Fragment**，大家知道，Fragment 也可以发起 `startActivityForResult` 操作，并通过 `onActivityResult` 接受结果。 

到这里，我们的思路就清晰了。我们的 Activity 自己不需要发起 `startActivityForResult`，而是新建一个不可见的 Fragemnt ，然后把这个任务交给它，Fragment 就相当于一个观察者， Activity 持有这个 Fragment 对象，Fragment 可以在自己收到 `onActivityResult` 的时候把结果通知 Activity。

这种做法有两个好处：首先，
1. 如果我们自己创建一个观察者，那么通常会被放在全局作用空间。那么就需要仔细考虑对象生命周期绑定问题，以防止可能造成的内存泄漏。Fragment 属于 Android 框架的一部分，只要正常使用（例如不要错误使用 static 引用，谨慎使用匿名内部类），就不会造成内存泄漏。

2. 自己创建的观察者对象，在 **进程被杀死重新创建** 或者 **后台Activity被回收** 的情况下，可能无法正常恢复，一方面可能导致无法接收到 `onActivityResult` 的结果，另一方面有可能导致应用 Crash（通常是因为空指针）。而如果我们把观察者的任务交给 Fragment，由于 Fragment 被 Activity 的 FragmentManager 管理，即使 Activity 由于系统的原因被销毁重新创建了，还是可以保证观察者自身被正确恢复，并且正常收到 `onActivityResult` 回调。

## 使用 RxJava 进行封装

上一小节提到，我们希望可以像对待一个普通的异步任务一样，对待业务流程。对于像我们这样的已经在项目中引入 [RxJava](https://github.com/ReactiveX/RxJava) 的团队来说，使用 RxJava 封装自然是首选。

首先，`onActivityResult` 这个回调中回传给我们的 3 个参数，我们单独封装为一个类：

```java
public class ActivityResult {

    private int requestCode;
    private int resultCode;
    private Intent data;

    public ActivityResult(int requestCode, int resultCode, Intent data) {
        this.requestCode = requestCode;
        this.resultCode = resultCode;
        this.data = data;
    }

    // getters & setters
}
```

本文在第一节的时候提到：流程 **有时候是由于某些条件不满足而触发** 的，举一个简单的例子：社交 App 的点赞操作需要登录态才可以进行，那么处理点赞事件的代码很有可能是这样的：

```java
likeBtn.setOnClickListener(v -> {
    if (LoginInfo.isLogin()) {
        doLike();
    } else {
        startLoginProcess()
            .subscribe(this::doLike);
    }
})
```

上面的代码中，我们假设 `startLoginProcess` 为一个封装好的登录流程，它的返回类型为 `Observable<ActivityResult>`。像这样的条件检测并且发起流程的类似代码很多，一个异步任务，把原本逻辑上流畅的一个代码流程给拆成两部分。为了可以让这部分更优雅，我们其实可以把 `LoginInfo.isLogin()` 为 `true` 这种情况也视为 `startLoginProcess` 这个 `Observable` 所发射的数据。到目前为止 `ActivityResult` 这个对象我们已经单独封装好了，我们可以自行实例化这个对象，无需依赖 `onActivityResult` 回调来构造这个对象：

```java
public Observable<ActivityResult> loginState() {
    if (LoginInfo.isLogin()) {
        return Observable.just(new ActivityResult(REQUEST_CODE_LOGIN, RESULT_OK, new Intent()));
    } else {
        return startLoginProcess();
    }
}
```

这样一来，检测用户是否登录，在 **用户已经登录** 和 **用户一开始没登录但是通过登录流程以后登录成功** 的情况下，执行点赞操作的代码就变成了下面这样：

```java
likeBtn.setOnClickListener(v -> {
    loginState()
        .subscribe(this::doLike);
})
```

虽然整体上代码量并没有变少，但是逻辑上更加清晰了，`loginState` 这个方法也更容易被其他地方所复用了。

接下来的部分是 承担着 `onActivityResult` 这个方法的观察者 的责任的 Fragment，根据刚刚的讨论结果，这个 Fragment 需要提供一个方法，允许除了在它自己得到 `onActivityResult` 回调时实例化一个 `ActivityResult` 这种情况以外，也可以手动插入一个 `ActivityResult` 对象，这样的封装可以在应对 **条件检测 - 发起流程** 这类情景时，对外部有更加简洁和一致的接口。具体代码如下：

```java
public class ActivityResultFragment extends Fragment {

    private final BehaviorSubject<ActivityResult> mActivityResultSubject = BehaviorSubject.create();

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        mActivityResultSubject.onNext(new ActivityResult(requestCode, resultCode, data));
    }

    public static Observable<ActivityResult> getActivityResultObservable(Activity activity) {
        FragmentManager fragmentManager = activity.getFragmentManager();
        ActivityResultFragment fragment = (ActivityResultFragment) fragmentManager.findFragmentByTag(
                ActivityResultFragment.class.getCanonicalName());
        if (fragment == null) {
            fragment = new ActivityResultFragment();
            fragmentManager.beginTransaction()
                    .add(fragment, ActivityResultFragment.class.getCanonicalName())
                    .commit();
            fragmentManager.executePendingTransactions();
        }
        return fragment.mActivityResultSubject;
    }

    public static void startActivityForResult(Activity activity, Intent intent, int requestCode) {
        FragmentManager fragmentManager = activity.getFragmentManager();
        ActivityResultFragment fragment = (ActivityResultFragment) fragmentManager.findFragmentByTag(
                ActivityResultFragment.class.getCanonicalName());
        if (fragment == null) {
            fragment = new ActivityResultFragment();
            fragmentManager.beginTransaction()
                    .add(fragment, ActivityResultFragment.class.getCanonicalName())
                    .commit();
            fragmentManager.executePendingTransactions();
        }
        fragment.startActivityForResult(intent, requestCode);
    }

    public static void insertActivityResult(Activity activity, ActivityResult activityResult) {
        FragmentManager fragmentManager = activity.getFragmentManager();
        ActivityResultFragment fragment= (ActivityResultFragment) fragmentManager.findFragmentByTag(
                ActivityResultFragment.class.getCanonicalName());
        if (fragment == null) {
            fragment = new ActivityResultFragment();
            fragmentManager.beginTransaction()
                    .add(fragment, ActivityResultFragment.class.getCanonicalName())
                    .commit();
            fragmentManager.executePendingTransactions();
        }
        fragment.mActivityResultSubject.onNext(activityResult);
    }
}
```

在 `ActivityResultFragment` 这个类中：

1.  `mActivityResultSubject` 即为发射 `ActivityResult` 的 Observable ；

2. `getActivityResultObservable` 这个方法是用于在 Activity 中获取不可见 Fragment 对应的 `Observable<ActivityResult>`，借鉴了 Glide 的思想；

3. `onActivityResult` 这个方法里，Fragment 把自己接收到的数据封装为 `ActivityResult` 传递给 `mActivityResultSubject`；

4. `startActivityForResult` 这个方法是用来被 Activity 调用的，Activity 把本来应该由自己发起的 `startActivityForResult` 交给由这个 Fragment 来发起；

5. `insertActivityResult` 这个方法的作用前面解释过了，是为了给调用流程的调用者，提供一致的接口，主要优化 **条件检测 - 发起流程** 这种情景。

## 可复用流程的封装

到目前为止，基于 RxJava，需要封装流程所需的基础设施已经准备完毕。我们来尝试封装一个流程，以登录流程为例，按上一篇讨论的结果，登录流程可能包含多个页面（用户名、密码验证，手机验证码两步验证等），也可能有子流程（忘记密码），但是对于“登录”这个流程，它对外只暴露一个代表它这个流程的 Activity，无论它内部跳转多么复杂，外部与这个登录流程交互也非常简单，只需要通过 `startActivityForResult` 和 `onActivityResult` 这两个方法。而这两个方法在上一节已经可以被很方便的封装，我们以登录流程为例，假设登录流程只对外暴露 `LoginActivity` 这一个 Activity，代码如下：

```java
public class LoginActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // your other code

        loginBtn.setOnClickListener(v -> {
            // 简单起见，略去请求部分，直接登录成功
            this.setResult(RESULT_OK);
            finish();
        });
    }

    public static Observable<ActivityResult> loginState(Activity activity) {
        if (LoginInfo.isLogin()) {
            ActivityResultFragment.insertActivityResult(
                activity,
                new ActivityResult(REQUEST_CODE_LOGIN, RESULT_OK, new Intent())
            );
        } else {
            Intent intent = new Intent(activity, LoginActivity.class);
            ActivityResultFragment.startActivityForResult(activity, intent, REQUEST_CODE_LOGIN);
        }
        return ActivityResultFragment.getActivityResultObservable(activity)
            .filter(ar -> ar.getRequestCode() == REQUEST_CODE_LOGIN)
            .filter(ar -> ar.getResultCode() == RESULT_OK);
    }
}
```

这个 Activity 对外提供一个静态的 `loginState` 方法，返回类型为 `Observable<ActivityResult>`，在已经登录的情况下，Observable 会立即发送一个 `ActivityResult` 表示登录成功，在非登录态下, 会唤起登录流程，如果登录流程最后的结果是登录成功，Observable 也会发送一个 `ActivityResult` 表示登录成功，所以凡是使用到这个登录流程的地方，对这个登录流程的调用，应该如下代码所示(仍然以点赞操作为例)：

```java
likeBtn.setOnClickListener(v -> {
    LoginActivity.loginState(this)
        .subscribe(this::doLike);
})
```

原先需要书写复杂的 `startActivityForResult` 和 `onActivityResult` 两个方法，才能完成和登录流程交互，而且还需要在发起流程前先确认是否当前已经是登录态，现在只需要一行 `LoginActivity.loginState()`, 然后指定一个 Observer 即可达到一样的效果，更重要的是，写法变简单了以后，整个登录流程变得非常容易复用，任何需要检查登录态然后再做操作的地方，都只需要这一行代码即可完成登录态检测，实现了 **流程的高度可复用**。 

这种写法可以在流程完成以后，继续之前的操作（例如本例中点赞），不需要用户重新进行一遍先前被流程所打断的操作（例如本例中的点赞操作）。但是细心的你可能并不这么认为，因为上面的代码本质上是有问题的，问题在于在上面 `LoginActivity.loginState()` 的调用在 `likeBtn.setOnClickListener` 的内部回调，那么考虑极端情况，如果登录流程被唤起，而发起登录流程的 Activity 不幸被系统回收，那么当登录流程做完回到的发起登录流程的 Activity 将会是系统重新创建的 Activity，这个全新的 Activity 是没有执行过 `likeBtn.setOnClickListener` 的内部回调的任何代码的，所以 `.subscribe()` 方法指定的观察者不会受到任何回调，`this::doLike` 不会被执行。

为了可以让封装的流程兼容这种情况，可以采用这种方案：修改 `loginState` 方法，使其返回 [ObservableTransformer](http://reactivex.io/RxJava/javadoc/io/reactivex/ObservableTransformer.html), 我们重命名 `loginState` 方法为 `ensureLogin`:

```java
public static ObservableTransformer<T, ActivityResult> ensureLogin(Activity activity) {
    return upstream -> {
        Observable<ActivityResult> loginOkResult = ActivityResultFragment.getActivityResultObservable(activity)
            .filter(ar -> ar.getRequestCode() == REQUEST_CODE_LOGIN)
            .filter(ar -> ar.getResultCode() == RESULT_OK);

        upstream.subscribe(t -> {
            if (LoginInfo.isLogin()) {
                ActivityResultFragment.insertActivityResult(
                    activity,
                    new ActivityResult(REQUEST_CODE_LOGIN, RESULT_OK, new Intent())
                );
            } else {
                Intent intent = new Intent(activity, LoginActivity.class);
                ActivityResultFragment.startActivityForResult(activity, intent, REQUEST_CODE_LOGIN);
            }
        }

        return loginOkResult;
    }
}
```

> 如果您之前没有接触过 `ObservableTransformer`, 这里做一个简单介绍，它通常和 `compose` 操作符一起使用，用来把一个 `Observable` 进行加工、修饰，甚至替换为另一个 `Observable`。

登录流程的封装，现在对外体现为 `ensureLogin` 这一个方法，那么其它代码如何调用这个登录流程呢，还是以点赞操作为例，现在的代码应该是这样：

```java
RxView.clicks(likeBtn)
    .compose(LoginActivity.ensureLogin(this))
    .subscribe(this::doLike);
```

这里的 `RxView.clicks` 使用了 [RxBinding](https://github.com/JakeWharton/RxBinding) 这个开源库，用于把 View 的事件，转化为 `Observable`，当然其实你也可以自己封装。改完这种写法以后，刚刚提到的极端情况下也可以正常工作了，即使发起流程的页面在流程被唤起后被系统回收，在流程完成以后回到发起页，发起页被重新创建了，发起页的 `Observer` 依然可以正常收到流程结果，之前被中端的操作得以继续执行。

现在我们可以稍微总结一下，根据上一篇和本篇提出建议，如何封装一个业务流程：

1. 一个业务流程对应一个 Activity，这个 Activity 作为对外的接口以及流程内部步骤的调度者；
2. 一个流程内部的一个步骤对应一个 Fragment，这个 Fragment 只负责完成自己的任务以及把自己的数据反馈给 Activity；
3. 流程对外暴露的接口应该封装为一个 `ObservableTransformer`，流程发起者应该提供发起流程的 `Observable`（例如以 `RxView.clicks` 的形式提供），两者通过 `compose` 操作符关联起来。

这是我个人实践出的一套封装流程的经验，它并不是完美的方案，但是在可靠性、可复用程度、接口简单程度上已经足以胜任我个人的日常开发，所以才有了这两篇分享。

我们已经封装了一个最简单的流程 -- 登录流程，但是实际项目中往往会遇到更严峻的挑战，例如流程组合与流程嵌套。


## 复杂流程实践：流程组合

举例：某款基金销售 App，在用户点击购买基金时，可能存在如下图流程：
![](/images/process-combine-1.png)

可用从上图中看出，某个未登录用户想要购买一款基金的最长路径包含：**登录 - 绑卡 - 风险测评 - 投资者适当性管理** 这几个步骤。但是，并不是所有用户都要经历所有这些步骤，例如，如果用户已登录并且已做过风险测评，那这个用户只需要再做 **绑卡 - 适当性管理** 这两步就可以了。

这样的一个需求，如果用传统的写法来写，可以预见肯定会在 click 事件处理的地方罗列很多 `if - else` ：
 ```java
// ...
// 设置点击事件处理函数
buyFundBtn.setOnClickListener(v -> handleBuyFund());


// ...
// 处理结果
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    switch (requestCode) {
        case REQUEST_LOGIN:
        case REQUEST_ADD_BANKCARD:
        case REQUEST_RISK_TEST:
        case REQUEST_INVESTMNET_PROMPT:
            if (resultCode == RESULT_OK) handleBuyFund();  
            break;
    }
} 

// ...
 
private void handleBuyFund() {
    // 判断是否已登录
    if (!isLogin()) {
        startLogin();
        return;
    }
    // 判断是否已绑卡
    if (!hasBankcard()) {
        startAddBankcard();
        return;
    }
    // 判断是否已做风险测试
    if (!isRisktestDone()) {
        startRiskTest();
        return;
    }
    // 判断是否需要按照投资者适当性管理规定，给予用户必要提示
    if (investmentPrompt()) {
        startInvestmentPrompt();
        return;
    }

    startBuyFundActivity();
}
 ```

上面这种写法一方面代码比较长，另一方面，流程发起和结果处理分散在两处，代码较为不易维护。我们分析一下，整个大的流程是几个小流程的组合，我们可以把上面的图上的流程换一种画法：

![](/images/process-combine-2.png)

按照上文的思想，我们令每个流程对外暴露一个 Activity，并且已经使用 RxJava `ObservableTransformer` 封装好，那么前面复杂的代码可以简化为：

```java
RxView.clicks(buyFundBtn)
    // 确保未登录情况下，发起登录流程，已登录情况下自动流转至下一个流程
    .compose(ActivityLogin.ensureLogin(this))
    // 确保未绑卡情况下，发起绑卡流程，已绑卡情况下自动流转至下一个流程
    .compose(ActivityBankcardManage.ensureHavingBankcard(this))
    // 确保未风险测评情况下，发起风险测评流程，已测评情况下自动流转至下一个流程
    .compose(ActivityRiskTest.ensureRiskTestDone(this))
    // 确保需要适当性提示情况下，发起适当性提示，已提示或不需要提示情况下自动流转至下一个流程
    .compose(ActivityInvestmentPrompt.ensureInvestmentPromptOk(this))
    // 所有条件都满足，进入购买基金页
    .subscribe(v -> startBuyFundActivity(this));
```

通过 RxJava 的良好封装，我们做到了可以用更少的代码来表达更复杂的逻辑。上面的例子中的 4 个被组合的流程，它们有一个共同的特点，就是彼此独立，互相不依赖其它剩余流程的结果，现实中，我们可能会遇到这样的情况: B 流程启动，需要依赖 A 流程完成的结果，为了能满足这种情况，我们只需要对上面的封装稍作修改。

假设绑卡流程需要依赖登录流程完成后的用户信息，那么首先，在登录流程结束调用 `setResult` 的位置, 传递用户信息：
```java
this.setResult(
    RESULT_OK, 
    IntentBuilder.newInstance().putExtra("user", user).build()
);
finish();
```
然后，修改 `ensureLogin` 方法，使经过 `ObservableTransformer` 处理后，返回的新的 `Observable` 由发射 `ActivityResult` 改为发射 `User`： 
```java
public static ObservableTransformer<T, User> ensureLogin(Activity activity) {
    return upstream -> {
        Observable<ActivityResult> loginOkResult = ActivityResultFragment.getActivityResultObservable(activity)
            .filter(ar -> ar.getRequestCode() == REQUEST_CODE_LOGIN)
            .filter(ar -> ar.getResultCode() == RESULT_OK)
            .map(ar -> (User)ar.getData.getParcelableExtra("user"));

        upstream.subscribe(t -> {
            if (LoginInfo.isLogin()) {
                ActivityResultFragment.insertActivityResult(
                    activity,
                    new ActivityResult(
                        REQUEST_CODE_LOGIN, 
                        RESULT_OK, 
                        IntentBuilder.newInstance().putExtra("user", LoginInfo.getUser()).build()
                    )
                );
            } else {
                Intent intent = new Intent(activity, LoginActivity.class);
                ActivityResultFragment.startActivityForResult(activity, intent, REQUEST_CODE_LOGIN);
            }
        }

        return loginOkResult;
    }
}
```

与此同时，原来的 `ensureHavingBankcard` 方法的 `ObservableTransformer` 方法接受的 Observable 原来是任意类型 T 的，由于我们现在规定，绑卡流程需要依赖登录流程的结果 User ，所以我们把 T 类型，改为 User 类型：

```java
public static ObservableTransformer<User, ActivityResult> ensureHavingBankcard(Activity activity) {
    return upstream -> {
        Observable<ActivityResult> bankcardOk = ActivityResultFragment.getActivityResultObservable(activity)
            .filter(ar -> ar.getRequestCode() == REQUEST_ADD_BANKCARD)
            .filter(ar -> ar.getResultCode() == RESULT_OK);

        upstream.subscribe(user -> {
            if (getBankcardNum() > 0) {
                ActivityResultFragment.insertActivityResult(
                    activity,
                    new ActivityResult(
                        REQUEST_ADD_BANKCARD, 
                        RESULT_OK, 
                        new Intent()
                    )
                );
            } else {
                Intent intent = new Intent(activity, AddBankcardActivity.class);
                intent.putExtra("user", user);
                ActivityResultFragment.startActivityForResult(activity, intent, REQUEST_ADD_BANKCARD);
            }
        }

        return bankcardOk;
    }
}
```

这样，这两个流程之间就有了依赖关系，绑卡依赖登录流程返回的结果，但是组合这两个流程的写法还是不会有任何改变：

```java
RxView.clicks(someBtn)
    .compose(ActivityLogin.ensureLogin(this))
    .compose(ActivityBankcardManage.ensureHavingBankcard(this))
    .subscribe(v -> doSomething());
```

除此以外，绑卡流程还是可复用的，它是依赖可以返回 User 的流程的，所以只要是其他可以返回 User 作为结果的流程，都可以与绑卡流程组合。

## 复杂流程实践：流程嵌套

举例：登录流程中的登录页面，除了可以选择用户名密码登录外，往往还提供其他选项，最典型的就是注册和忘记密码两个功能：

![](/images/process-insert-1.png)

从直觉上，我们肯定是认为注册和忘记密码应该是不属于登录这个流程的，它们是相对独立的两个流程，也就是说在登录这流程内部，嵌入了其它的流程，我把这种情况称之为流程的嵌套。

按照同样的套路，我们应该先把注册、忘记密码这两个流程使用 `ObservableTransformer` 进行封装，然后我们把上图流程按照本文思想整理一下，如下：

![](/images/process-insert-2.png)

可以看到，现在的区别是，发起流程的地方不再是一个普通的 Activity，而是另一个流程中的某个步骤，按照先前的讨论，流程中的步骤是由 Fragment 承载的。所以这里有两种处理方法，一种是 Fragment 把发起流程的任务交给宿主 Activity，由宿主 Activity 分配给属于它的“看不见的 Fragment” 去发起流程并处理结果，另一种是直接由该 Fragmnet 发起流程，由于 Fragment 也有属于它自己的 ChildFragmentManager，所以只需要对“**使用 RxJava 进行封装**”这一节中的相关方法做一些修改即可支持由 Fragment 内部发起流程，具体修改内容为把 `activity.getFragmentManager()` 改为 `fragment.getChildFragmentManager()` 即可。

在具体应用中，本人使用的是后一种，即 **直接由 Fragment 发起流程**，因为被嵌套的流程往往和主流程有关联，即嵌套流程的结果有可能改变主流程的流转分支，所以直接由 Fragment 发起流程并处理结果比较方便一点，如果交给宿主 Activity 可能需要额外多写一些代码进行 Activity - Fragment 的通信才能实现相同效果。

首先，在没有嵌套流程的情况下，登录流程的第一个步骤登录步骤（用户名、密码验证），代码应该如下：

```java
public class LoginFragment extends Fragment {
    // UI references.
    private EditText mPhoneView;
    private EditText mPasswordView;

    LoginCallback mCallback;

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_login_by_pwd, container, false);
        // Set up the login form.
        mPhoneView = view.findViewById(R.id.phone);
        mPasswordView = view.findViewById(R.id.password);

        Button signInButton = view.findViewById(R.id.sign_in);
        signInButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                String phone = mPhoneView.getText().toString();
                String pwd = mPasswordView.getText().toString();

                if (mCallback != null) {
                    // mock login ok
                    mCallback.onLoginOk(true, new User(
                            UUID.randomUUID().toString(),
                            "Jack",
                            mPhoneView.getText().toString()
                    ));
                }
            }
        });

        return view;
    }

    public void setLoginCallback(LoginCallback callback) {
        this.mCallback = callback;
    }

    public interface LoginCallback {
        void onLoginOk(boolean needSmsVerify, User user);
    }
}
```

在上面的代码中，`LoginCallback` 这个接口作用是，登录这个步骤，收集完信息，与服务器交互完毕后，把结果回传给宿主 Activity，由 Activity 决定后续步骤的流转。上面的例子中做了一部分简化，在 `onClick` 处理函数里没有发起和服务端的交互，而是直接 Mock 了一个请求成功的结果。

现在的需求是，在登录这个步骤里，嵌入两个步骤：

1. 一个是注册流程，而且注册成功后直接视为登录成功，不需要再走剩余的登录流程步骤；
2. 另一个是忘记密码流程，忘记密码流程本质是重置密码，但是即使密码重置成功还是需要用户使用新密码登录，不会直接在重置密码后自动登录。

根据需求，我们在上述代码中加入嵌入这两个流程的代码：

```java
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_login_by_pwd, container, false);
        // Set up the login form.
        mPhoneView = view.findViewById(R.id.phone);
        mPasswordView = view.findViewById(R.id.password);

        Button signInButton = view.findViewById(R.id.sign_in);
        Button mPwdResetBtn = view.findViewById(R.id.pwd_reset);
        Button mRegisterBtn = view.findViewById(R.id.register);

        // 直接登录
        signInButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                String phone = mPhoneView.getText().toString();
                String pwd = mPasswordView.getText().toString();

                if (mCallback != null) {
                    // mock login ok
                    mCallback.onLoginOk(true, new User(
                            UUID.randomUUID().toString(),
                            "Jack",
                            mPhoneView.getText().toString()
                    ));
                }
            }
        });

        // 发起注册流程
        RxView.clicks(mRegisterBtn)
            .compose(RegisterActivity.startRegister(this))
            .subscribe(user -> {
                if (mCallback != null) {
                    mCallback.onRegisterOk(user);
                }
            });

        // 发起忘记密码流程
        RxView.clicks(mPwdResetBtn)
            .compose(PwdResetActivity.startPwdReset(this))
            .subscribe();

        return view;
    }

    public interface LoginCallback {
        void onLoginOk(boolean needSmsVerify, User user);
        void onRegisterOk(User user);
    }
```

在上面的代码里，`RegisterActivity.startRegister` 和 `PwdResetActivity.startPwdReset` 两个方法即为使用了 `ObservableTransformer` 封装的注册流程和忘记密码流程。同时可以看到，`LoginCallback` 这个接口里多了一个方法 `onRegisterOk`，也就是说登录这个步骤不再只有 `onLoginOk` 这一种情况通知宿主 Activity 了，在内嵌注册流程成功的情况下，也可以通知宿主 Activity，然后让宿主 Activity 决定后续流转，当然这种情况，根据需求注册成功也是属于登录成功的一种，宿主 Activity 通过 `setResult` 方法把整个登录流程的状态标记为登录成功，`finish` 自己，同时把用户信息传递给发起登录流程的地方。

但是为什么内嵌的注册流程需要把流程的结果回传给登录流程的宿主 Activity，而内嵌的忘记密码流程没有设置一个类似的方法回调登录流程的宿主 Activity 呢？因为注册成功这件事影响了登录流程的走向（注册成功直接视为登录成功，登录流程状态置为成功，并通知发起登录流程的地方本次登录结果为成功），而忘记密码流程最后的重置密码成功并不影响登录流程走向（即使重置密码成功依然需要在登录界面使用新密码登录）。

按照上面的分析，登录流程的宿主 Activity，负责分发流程步骤的逻辑的相关代码如下所示：

```java
public class LoginActivity extends Activity {

    // ... 

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        //...
        // 用户名密码验证步骤
        loginFragment.setLoginCallback(new LoginFragment.LoginCallback() {
            @Override
            public void onLoginOk(boolean needSmsVerify, User user) {
                // 用户名密码验证成功
                if (needSmsVerify) {
                    // 需要短信验证码的两步验证
                    push(loginBySmsFragment);
                } else {
                    // 登录成功
                    setResult(
                        RESULT_OK, 
                        IntentBuilder.newInstance().putExtra("user", user).build()
                    );
                    finish();
                }
            }

            @Override
            public void onRegisterOk(User user) {
                // 注册成功, 直接登录
                setResult(
                    RESULT_OK, 
                    IntentBuilder.newInstance().putExtra("user", user).build()
                );
                finish();
            }
        }

        // 短信验证码两步验证步骤
        loginBySmsFragment.setSmsLoginCallback(new LoginBySmsFragment.LoginBySmsCallback() {
            @Override
            public void onSmsVerifyOk(User user) {
                // 短信验证成功
                setResult(
                    RESULT_OK, 
                    IntentBuilder.newInstance().putExtra("user", user).build()
                );
                finish();
            }
        });
    }

    // ...
}
```

可以看到，即使是流程嵌套的情况下，使用 RxJava 封装的流程依然不会使流程跳转的代码显得十分混乱，这点十分可贵，因为这意味着今后流程相关代码不会成为项目中难以维护的模块，而是清晰且高内聚的。

## 流程上下文保存

到目前为止，我们还剩最后一个问题需要解决，即涉及流程的相关上下文保存。具体包含两部分，一是流程触发点，发起流程的位置，需要对发起流程前的上下文进行保存，另一部分是流程中间步骤的结果，也需要进行保存。

**1. 流程中间步骤的结果的保存**

要对流程中间步骤的结果进行保存是因为，按照我们前面的讨论，流程中每个步骤（即Fragment），会和用户交互，然后把该步骤的结果传递给宿主 Activity，那么假设流程没有做完，并且该步骤的结果可能会被后续步骤使用，那么宿主 Activity 是有必要保存这个结果的，那么通常这个结果会以这个 Activity 的成员变量的形式被保存，问题在于 Activity 一旦被置于后台，就随时可能被系统回收，此时可能流程并没有做完，如果没有对 Activity 的成员变量做保存和恢复处理，当下次 Activity 回到前台以后，这个流程的状态就处于不确定状态了，甚至可能崩溃。

解决的方案很显然，继承 Activity 的 `onSaveInstanceState` 和 `onRestoreInstanceState`（或者 `onCreate`） 这两个方法，在这两个方法内部实现变量的保存与恢复操作。如果你觉得实现这两个方法会使你的代码非常丑陋，那么我推荐你使用 [SaveState](https://github.com/PrototypeZ/SaveState) 这个工具，使用它，你只需要在需要保存和恢复的成员变量上标记一个 `@AutoRestore` 注解，框架就会自动帮你保存和恢复成员变量，你不需要写任何额外的代码。

**2. 发起流程前的上下文的保存**

和 **1** 的原因类似，流程一旦被唤起，发起流程的 Activity 就处于后台状态，这是一种可能被系统回收的状态。举个例子, 有一个理财产品列表页，用户未登录状态，现在要求用户点击任何一个理财产品，先把用户带到登录界面，待登录流程完成后，把用户带到具体的理财产品购买页。列表的点击事件设置一般分两种，一是为列表中每个 Item 设置一个点击处理函数，另一种是为所有 Item 设置同一个点击处理函数。以给列表所有 Item 设置同一个点击处理函数为例：

```java

// 所有 Item 的点击事件对应的 Observable，其发射的元素为点击位置
Observable<Integer> itemClicks = ...

itemClicks
    .compose(LoginActivity.ensureLogin(activity))
    .subscribe(/** 不知道怎么写 **/);
```

`subscribe` 里面的观察者不知道怎么写了是因为 `LoginActivity.ensureLogin` 这个 `ObservableTransformer` 会把 `Observable<T>` 转为 `Observable<ActivityResult>`, 所以观察者里只知道登录成功了，不知道最初是点击哪个理财产品触发的登录操作，所以不知道应该如何去启动购买页面。

我们遇到的困境是当流程完成以后，我们不知道发起流程前的上下文是什么，导致我们无法在观察者里做正确的后续逻辑。一种直观的解决方案就是，我们把发起流程时的上下文数据打包进 `startActivityForResult` 的 Intent 里面，用一个保留的 Key 值去存储，同时确保流程完成时， `setResult` 调用时，会把刚刚流程传入的上下文数据，同样以一个保留的 Key 值回传给发起流程的地方。

如果这样处理以后，我们回过头看刚才的情况，我们再实现一个 `LoginActivity.ensureLoginWithContext` 方法，它的返回值为 `ObservableTransformer<Bundle, Bundle>` ：

```java
public static ObservableTransformer<Bundle, Bundle> ensureLoginWithContext(AppCompatActivity activity) {
    return upstream -> {
        upstream.subscribe(contextData -> {
            if (LoginInfo.isLogin()) {
                ActivityResultFragment.insertActivityResult(
                    activity,
                    new ActivityResult(REQUEST_LOGIN, RESULT_OK, null, contextData)
                );
            } else {
                Intent intent = new Intent(activity, LoginActivity.class);
                ActivityResultFragment.startActivityForResult(activity, intent, REQUEST_LOGIN, contextData);
            }
        });
        return ActivityResultFragment.getActivityResultObservable(activity)
                .filter(ar -> ar.getRequestCode() == REQUEST_LOGIN)
                .filter(ar -> ar.getResultCode() == RESULT_OK)
                .map(ActivityResult::getRequestContextData);
    };
}
```


上面的代码中的 `ensureLoginWithContext` 和原先的 `ensureLogin` 方法相比，除了返回值的泛型类型不同以外，在内部实现里，调用的 `ActivityResult` 的构造方法以及 `startActivityForResult` 方法和原先的版本比都多了一个 `Bundle` 类型的 `contextData` 参数，这个参数即为需要保存的流程发起前的上下文。最后看整个方法的 return 语句，多了一个 `map` 操作符，用来把 `ActivityResult` 里保存的流程的上下文重新取出来。这里的逻辑就是刚刚提到的：在流程发起前，将流程发起前的上下文信息通过 `Bundle` 传递给流程，最后流程结束时再原封不动返回给流程发起的地方，以便流程发起点可以知晓它发起流程前的状态。这几个方法的具体实现可以参考 [Sq](https://github.com/PrototypeZ/Sq) 这个框架。


在经过这样处理以后，列表 Item 的点击事件发起登录流程的代码如下所示：
```java
itemClicks
        .map(index -> BundleBuilder.newInstance().putInt("index", index).build())
        .compose(LoginActivity.ensureLoginWithContext(this))
        .map(bundle -> bundle.getInt("index"))
        .subscribe(index -> {
            // modification of item in position $index
            adapter.notifyItemChanged(index);
        });
```
在 `compose` 操作符前后，分别多了一个 `map` 操作符，分别负责把上下文打包以及从流程结果中把原来的上下文解包取出来。

流程上下文的保存还有一个注意点，就是流程在结束时，即调用 `setResult` 时，需要保证把先前传入的的上下文再塞回去到结果里去，只有做到了这点，上面的代码才是有效的，这些工作如果总是手动来做会很繁琐，您可以选择自己封装，或者直接使用下一节介绍的开箱即用的工具。

## 如何快速使用

到这里为止，对于封装业务流程相关所有经验分享已经介绍完毕，如果您看到这里，对于本文以及本文的上一篇提出的流程方案感兴趣，您有两种方法集成到自己的项目里，一是参照文中的代码，自己实现（核心代码都已在文中，稍作修改即可）; 另一种方法是直接使用封装好的版本，这个项目的名字是 [Sq](https://github.com/PrototypeZ/Sq), 您只需要把依赖添加到 Gradle，开箱即用。

[![](https://raw.githubusercontent.com/PrototypeZ/Sq/master/sq-logo.png)](https://github.com/PrototypeZ/Sq)

## 总结

文章很长，感谢您耐心读完。由于本人能力有限，文章可能存在纰漏的地方，欢迎各位指正。关于如何对业务流程进行封装，因为我并没有看到过很多技术文章对这一块进行讨论，所以我个人的见解会有不全面的地方，如果您有更好的方案，欢迎一起讨论。谢谢大家！