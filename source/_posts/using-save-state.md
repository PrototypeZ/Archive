---
title: 不需要再手写 onSaveInstanceState 了，因为你的时间非常值钱
tags: Android onSaveInstanceState
desc: 自动化组件状态保存与恢复
---

如果你是一个有经验的 Android 程序员，那么你肯定手写过许多 `onSaveInstanceState` 以及 `onRestoreInstanceState` 方法用来保持 Activity 的状态，因为 Activity 在变为不可见以后，系统随时可能把它回收用来释放内存。**重写 Activity 中的  `onSaveInstanceState` 方法** 是 Google 推荐的用来保持 Activity 状态的做法。
<!-- More -->

## Google 推荐的最佳实践

`onSaveInstanceState` 方法会提供给我们一个 `Bundle` 对象用来保存我们想保存的值，但是 `Bundle` 存储是基于 key - value 这样一个形式，所以我们需要定义一些额外的 `String` 类型的 key 常量，最后我们的项目中会充斥着这样代码：

```java
static final String STATE_SCORE = "playerScore";
static final String STATE_LEVEL = "playerLevel";
// ...


@Override
public void onSaveInstanceState(Bundle savedInstanceState) {
    // Save the user's current game state
    savedInstanceState.putInt(STATE_SCORE, mCurrentScore);
    savedInstanceState.putInt(STATE_LEVEL, mCurrentLevel);

    // Always call the superclass so it can save the view hierarchy state
    super.onSaveInstanceState(savedInstanceState);
}
```

保存完状态之后，为了能在系统重新实例化这个 Activity 的时候恢复先前被系统杀死前的状态，我们在 `onCreate` 方法里把原来保存的值重新取出来：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); // Always call the superclass first

    // Check whether we're recreating a previously destroyed instance
    if (savedInstanceState != null) {
        // Restore value of members from saved state
        mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
        mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
    } else {
        // Probably initialize members with default values for a new instance
    }
    // ...
}
```


当然，恢复这个操作也可以在 `onRestoreInstanceState` 这个方法实现：

```java
public void onRestoreInstanceState(Bundle savedInstanceState) {
    // Always call the superclass so it can restore the view hierarchy
    super.onRestoreInstanceState(savedInstanceState);

    // Restore state members from saved instance
    mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
    mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
}
```


## 解放你的双手

上面的方案当然是正确的。但是并不优雅，为了保持变量的值，引入了两个方法 ( `onSaveInstanceState` 和 `onRestoreInstanceState` ) 和两个常量 ( 为了存储两个变量而定义的两个常量，仅仅为了放到 `Bundle` 里面)。

为了更好地解决这个问题，我写了 [SaveState](https://github.com/PrototypeZ/SaveState) 这个插件：

![save-state-logo](https://raw.githubusercontent.com/PrototypeZ/SaveState/master/logo.png)

在使用了 [SaveState](https://github.com/PrototypeZ/SaveState) 这个插件以后，保持 Activity 的状态的写法如下：

```java
public class MyActivity extends Activity {

    @AutoRestore
    int myInt;

    @AutoRestore
    IBinder myRpcCall;

    @AutoRestore
    String result;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // Your code here
    }
}
```

没错，你只需要在需要保持的变量上标记 `@AutoRestore` 注解即可，无需去管那几个烦人的 Activity 回调，也不需要定义多余的 `String` 类型 key 常量。

那么，除了 Activity 以外，Fragment 能自动保持状态吗？答案是： Yes！

```java
public class MyFragment extends Fragment {

    @AutoRestore
    User currentLoginUser;

    @AutoRestore
    List<Map<String, Object>> networkResponse;

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        // Your code here
    }
}
```

使用方法和 Activity 一模一样！不止如此，使用场景还可以推广到 `View`， 从此，你的自定义 View，也可以把状态保持这个任务交给 [SaveState](https://github.com/PrototypeZ/SaveState) ：

```java
public class MyView extends View {

    @AutoRestore
    String someText;

    @AutoRestore
    Size size;

    @AutoRestore
    float[] myFloatArray;

    public MainView(Context context) {
        super(context);
    }

    public MainView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public MainView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

}
```

## 现在就使用 SaveState

引入 [SaveState](https://github.com/PrototypeZ/SaveState) 的方法也十分简单：

首先，在项目根目录的 `build.gradle` 文件中增加以下内容：

```groovy
buildscript {

    repositories {
        google()
        jcenter()
    }
    dependencies {
        // your other dependencies

        // dependency for save-state
        classpath "io.github.prototypez:save-state:${latest_version}"
    }
}
```

然后，在 **application** 和 **library** 模块的 `build.gradle` 文件中应用插件：

```groovy
apply plugin: 'com.android.application'
// apply plugin: 'com.android.library'
apply plugin: 'save.state'
```

万事具备！再也不需要写烦人的回调，因为你的时间非常值钱！做了一点微小的工作，如果我帮你节省下来了喝一杯咖啡的时间，希望你可以帮我点一个 Star，谢谢 :)


SaveState Github 地址：[https://github.com/PrototypeZ/SaveState](https://github.com/PrototypeZ/SaveState)