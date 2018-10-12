---
title: Cross module method invocation made easy
tags: calling methods of other modules including the application module
date: 2018-10-10 20:57:00
desc: multi module Android development
---

Are you building an Android App with multiple modules? If so, I guess you maybe facing the same problem as me, that is: **Cross Module Method Invocation**. It's easy to call methods from library modules in our application module. But it's annoying to invoke methods from the application module in our libray modules. 

<!-- More -->

## Solution

In order to solve this problem, I wrote a simple tool called [AppJoint](https://github.com/PrototypeZ/AppJoint). With **AppJoint**, now we can call any methods from any module in anywhere. 

Assuming our project structure is like below:

```
projectRoot
  +--app
  +--module1
  +--module2 
```

Here, `app` is an application module and `module1`/`module2` are library modules. `app` module depends on `module1` and `module2`:

If we wish the `app` module to provide services to other modules like `module1` and `module2`, we need to create a new library first and define service interfaces inside it and make all modules have the dependency of this new module.

For example, I create a new library module named `service` and create several kotlin interfaces which representing the services that each module wants to provide for other modules to use:

```
projectRoot
  +--app
  +--module1
  +--module2 
  +--service
  |  +--main
  |  |  +--kotlin
  |  |  |  +--com.yourPackage
  |  |  |  |  +--AppService.kt
  |  |  |  |  +--Module1Service.kt
  |  |  |  |  +--Module2Service.kt
  
```

+ Methods in `AppService` are provided by the `app` module for other modules to use
+ Methods in `Module1Service` are provided by the `module1` module for other modules to use
+ Methods in `Module2Service` are provided by the `module2` module for other modules to use

All modules should include the `service` module excluding the `service` module itself:

```groovy
dependencies {
    ...
    implementation project(":service")
}
```

Maybe our `AppService.kt` source codes are like below:

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

And then we write an implementation of `AppService` in the `app` module:

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

Note that we add a `@ServiceProvider` annotation on the `AppServiceImpl` class.

Now, if we need to call methods of `AppService` inside `module1` or `module2`, we just need to write codes as below:

```kotlin
val appService = AppJoint.service(AppService::class.java)

// call methods inside AppService
appService.callMethodSyncOfApp()
```

That's all, the `@ServiceProvider` annotation and the `AppJoint.service` method are the only two API.

## Getting started

It's easy to include AppJoint.

1. Add the **AppJoint** plugin dependency to the `build.gradle` file in project root:

```groovy
buildscript {
    ...
    dependencies {
        ...
        classpath 'io.github.prototypez:app-joint:{latest_version}'
    }
}
```

2. Add the **AppJoint** dependency to every module：

```groovy
dependencies {
    ...
    implementation "io.github.prototypez:app-joint-core:{latest_version}"
}
```

3. Apply the **AppJoint** plugin to your main app module： 

```groovy
apply plugin: 'com.android.application'
apply plugin: 'app-joint'
```

## Conclusion

Github of AppJoint: [https://github.com/PrototypeZ/AppJoint](https://github.com/PrototypeZ/AppJoint)

Have fun!
