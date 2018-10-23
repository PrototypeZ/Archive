---
title: 一种在 Library 模块中调用 Application 模块功能的方法
tags: Android 组件化
date: 2018-10-21 22:15:00
desc: 组件化的第一步，模块化
---

在[上一篇分享](/2018/09/15/app-joint-introduction/)中，我介绍的主题是如何在一个 Android 项目中使用 [AppJoint](https://github.com/PrototypeZ/AppJoint) 进行组件化。有读者找到我说：我知道组件化很美好，但是项目改造总是需要成本的，而且这个成本是我们目前承受不了的，我们目前能做的最多的只能是 **模块化**，即把主 app 模块的功能拆出来，放到新的 **library** 模块里，但目前面临的最直接的问题就是，**拆出来的新模块如何调用主 app 模块里的方法**。能否不用组件化那一整套工具，先用 **最轻量级的方法** 解决这个跨模块方法调用的问题？

<!-- More -->

答案当然是肯定的，[AppJoint](https://github.com/PrototypeZ/AppJoint) 就是这样一个轻量级的工具，如果你不想组件化，那么你完全可以把它当成一个只是用来解决跨模块方法调用的问题的小工具。

## 问题背景

假设目前已经从 **app** 模块拆出了一个 **module1** 模块，**app** 模块是主 app 模块， 而 **module1** 模块是 library 模块。如果 **module1** 模块想要调用 **app** 模块的功能，那么肯定需要存在一个接口，这个接口两个模块要都能访问到。所以这个接口存在的位置就只能是 **app** 模块和 **module1** 模块都依赖的公共模块。这里的这个公共模块，它可以是已有的公共模块，也可以是新创建的公共模块。我们在这个公共模块里声明这个接口，即 **app** 模块希望被 **module1** 模块调用的方法的接口：

```kotlin
interface AppService {

  // start Activity from app module
  fun startActivityOfApp(context: Context)

  // get Fragment from app module
  fun obtainFragmentOfApp(): Fragment

  // call synchronous method from app module
  fun callMethodSyncOfApp(): String

  // call asynchronous method from app module
  fun callMethodAsyncOfApp(callback: AppCallback<AppEntity>)

  // get RxJava Observable from app module
  fun observableOfApp(): Observable<AppEntity>
}
```

这个接口由于处于公共模块，所以两个模块都能访问到这个接口。接下来我们就来解决这个问题，即在 **module1** 模块里调用 **app** 模块的方法。

## 问题解决

在声明上面这个接口以后，我们在 **app** 模块里实现这个接口：

```kotlin
@ServiceProvider
class AppServiceImpl : AppService {
  override fun startActivityOfApp(context: Context) {
    AppActivity.start(context)
  }

  override fun obtainFragmentOfApp(): Fragment {
    return AppFragment.newInstance()
  }

  override fun callMethodSyncOfApp(): String {
    return "syncMethodResultApp"
  }

  override fun callMethodAsyncOfApp(callback: AppCallback<AppEntity>) {
    Thread { callback.onResult(AppEntity("asyncMethodResultApp")) }.start()
  }

  override fun observableOfApp(): Observable<AppEntity> {
    return Observable.just(AppEntity("rxJavaResultApp"))
  }
}
```

需要注意一点，我们在实现类上标记了 `@ServiceProvider` 注解。然后我们就可以像下面这样，从 **module1** 模块里调用 **app** 模块里的方法了：

```kotlin
val appService = AppJoint.service(AppService::class.java)

// call methods inside AppService
appService.callMethodSyncOfApp()
appService.observableOfApp().subscribe()
appService.startActivityOfApp(context)
...
```

即使我们没有引入组件化的任何概念，我们也能轻松解决模块化中最常遇见的跨模块方法调用的这一类问题。

关于如何在项目中引入 **AppJoint**，可以前往 **AppJoint** 的 Github 主页：[https://github.com/PrototypeZ/AppJoint](https://github.com/PrototypeZ/AppJoint), 核心代码不超过 500 行，您可以使用开箱即用的版本，也可以自行在项目中引入。

## 参考资料

在模块化之后，您可能会对项目的组件化产生兴趣，欢迎继续阅读我的组件化经验分享 [『回归初心：极简 Android 组件化方案 — AppJoint』](http://prototypez.github.io/2018/09/15/app-joint-introduction/)，希望可以给您的项目组件化带去一点点帮助，谢谢！ : )
___
如果您对我的技术分享感兴趣，欢迎关注我的个人公众号：麻瓜日记，不定期更新原创技术分享，谢谢！:)

![](http://prototypez.github.io/images/qrcode.jpg)