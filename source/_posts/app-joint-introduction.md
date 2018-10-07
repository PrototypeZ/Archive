---
title: 回归初心：极简 Android 组件化方案 — AppJoint
tags: Android 组件化
date: 2018-09-15 14:26:00
desc: 更适合中小团队的 Android 组件化方案，简洁有效
---

Android 组件化的概念大概从两年前开始有人讨论，到目前为止，技术已经慢慢沉淀下来，越来越多团队开源了自己组件化框架。本人所在团队从去年开始调研组件化框架，在了解社区众多组件化方案之后，决定自研组件化方案。为什么明明已经有很多轮子可以用了，却还是决定要自己造个新轮子呢？

<!-- More -->

主要的原因是在调研了诸多组件化方案之后，发现尽管它们都有各自的优点，但是依然有一些地方不是令人十分满意。而其中最重要的一个因素就是引入组件化方案成本较高，对已有项目改造过大。我想这一点应该很多人都有相同的体会，很多时候 **我们对于项目的重构是需要与新需求的迭代同步进行的** ，几乎很难停下来只做项目的组件化。

另外一点，我不太希望自己的项目和某一款组件化框架 **强耦合**。 Activity 的路由方案也好，跨模块的同步或异步方法调用也好，我希望能够沿用项目已有的调用方式，而不是使用某款组件化框架自己特定的调用方式。例如某个接口已经基于 **RxJava** 封装为了 `Observable` 的接口，我就不太希望因为组件化的关系，这个接口位于另一个模块之后，我就不得不用这个组件化框架定义的方式去调用，我还是希望以 **RxJava** 的方式去调用。

## 回归初心

我认为目前想要进行组件化的项目应该可以分为两类：

+ 包含有一个 **application** 模块，以及一些技术组件的 **library** 模块（业务无关）。
+ 除了 **application** 模块以外，已经存在若干包含业务的 **library** 模块和技术的 **library** 模块。

无论是哪种类型的项目，面临的问题应该都是类似的，**那就是项目大起来以后，编译实在是太慢了**。

除此以外，就是 **跨模块的功能调用非常不便** ，这个问题主要体现在上面列举的第二种类型的项目。本人所在的项目在组件化之前就是上面列举的第二种类型的项目，**application** 模块最早用来承载业务逻辑代码，随着业务发展，大概是某位开发人员觉得， “不行，这样下去 **application** 模块代码数量会失控的”，于是后续新的业务模块都会新开一个 **library** 模块进行开发，就这样断断续续目前有了大概 20+ 个 **library** 模块（业务相关模块，技术模块不包含在内）。

这种做法是符合软件工程思想的，但是也带来了一些棘手的问题，由于 **application** 模块里的业务功能和 **library** 模块里的业务功能在逻辑地位上是平等的，所以难免会有互相调用的情况，但是它们在项目依赖层次上却不是处于相等的地位，**application** 调用 **library** 倒没事，但是反过来调用就成了问题。另外，剩下这 20 + 个 **library** 模块在依赖层次中也不全是属于同一层次的，**library** 模块之间互相依赖也很复杂。

所以我期望的组件化方案要求解决的问题很简单：

+ 业务模块单独编译，单独运行，而不是耗费大量时间全量编译整个 App
+ 跨模块的调用应该优雅，无论两个模块在依赖树中处于什么样的位置，都可以很简单的互相调用
+ 不要有太多的学习成本，沿用目前已有的开发方式，避免代码和具体的组件化框架绑定
+ 组件化的过程可以是渐进的，立即拆分代码不是组件化的前置条件
+ 轻量级，不要引入过多中间层次（例如序列化反序列化）导致不必要的性能开销以及维护复杂度

基于上述的思想，我们开发了 [AppJoint](https://github.com/PrototypeZ/AppJoint) 这个框架用来帮助我们实现组件化。

![](https://raw.githubusercontent.com/PrototypeZ/AppJoint/master/app-joint-logo.png)

**AppJoint** 是一个非常简单有效的方案，引入 **AppJoint** 进行组件化所有的 API 只包含 **3** 个注解，加 **1** 个方法，这可能是目前最简单的组件化方案了，我们的框架不追求功能要多么复杂强大，只专注于框架本身实用、简单与高效。而且整体实现也非常简单，核心源码 **不到500行**。

## 模块独立运行遇到的问题

本人接触最早的组件化方案是 [DDComponentForAndroid](https://github.com/luojilab/DDComponentForAndroid)，学习这个方案给了我很多启发，在这个方案中，作者提出，可以在 `gradle.properties` 中新增一个变量 `isRunAlone=true` ，用来控制某个业务模块是 **以 library 模块集成到 App 的全量编译中** 还是 **以 application 模块独立编译启动** 。不知道是不是很多人也受了相同的启发，后面很多的组件化框架都是使用类似的方案:

```groovy
if(isRunAlone.toBoolean()){    
    apply plugin: 'com.android.application'
}else{  
    apply plugin: 'com.android.library'
}
```

根据我本人的实践，这种方式有一些缺点。首先有一些开源框架在 **library** 模块中和在 **application** 模块中使用方法是不一样的，例如 [ButterKinfe](https://github.com/JakeWharton/butterknife) , 在 **application** 中使用 `R.id.xxx`，在 **library** 模块中使用 `R2.id.xxx` ，如果想组件化，代码必须保证在两种情况下都可用，所以基本只能抛弃 **ButterKnife** 了，这会给项目带来巨大的改造成本。

除此以外，还有一些开源框架是只能在 **application** 模块中配置的，配置完以后对整个项目的所有 **library** 模块都生效的，例如一些字节码修改的框架（比如 AOP 一类的），这是一种情况。还有一种情况，如果原先项目已经是多模块的情况下，可能多个模块的初始化都是放在 **application** 模块里，因为 **application** 模块是 **上帝模块**，他可以访问到项目中任意一块代码，所以在这里做初始化是最省事的。但是现在拆分为模块之后，因为每个模块需要独立运行，所以模块需要负责自身的初始化，可是有时候这个模块的初始化是只能在 **application** 模块里才可以做的，我们把这段逻辑下放到 **library** 之后，如何初始化就成了问题。

这两种情况，如果我们使用 `gradle.properties` 中的变量来切换 **application** 和 **library** 的话，我们势必需要在这个模块中维护两套逻辑，一套是在 **application** 模式下的启动逻辑，一套是在 **library** 模式下的启动逻辑。原先这个模块是专注自己本身的业务逻辑的，现在不得不为了能够独立作为 **application** 启动，而加入许多其他代码。一方面 `build.gradle` 文件中会充满很多 `if - else`，另一方面 Java 源码中也会加入许多判断是否独立运行的逻辑。

最终 Release App 打包时，这些模块是作为 **library** 存在的，但是我们为了组件化已经在这个模块中加入了很多帮助该模块独立运行（以 **application** 模式）的代码（例如模块需要单独运行，需要一个属于这个模块的 Laucher Activity），虽然这些代码在线上不会生效，可是从洁癖的角度来讲，这些代码其实不应该被打包进去。其实说了这么多无非就是想说明，如果我们希望通过某个变量来控制模块以 **application** 形式还是以 **library** 形式存在，那么我们肯定要在这个模块中加入维护两者的差异的代码，而且可能代码量还不少，最后代码呈现的状态可能是不太优雅的。

此外模块中的 `AndroidManifest.xml` 也需要维护两份：

```groovy
if (isRunAlone.toBoolean()) {
    manifest.srcFile 'src/main/runalone/AndroidManifest.xml'
} else {
    manifest.srcFile 'src/main/AndroidManifest.xml'
}
```

但是 **xml** 毕竟不是代码，没有封装继承这些面向对象的特性，所以每当我们增加、修改、删除四大组件的时候，都需要记得要在两个 `AndroidManifest.xml` 都做对应的修改。除了 `AndroidManifest.xml` 以外，资源文件也存在这个问题，虽然工作量不至于特别巨大，但这样的做法其实已经违背了面向对象的设计原则。

最后还有一个问题，每当模块在 **application** 模式和 **library** 模式之间进行切换的时候，都需要重新 **Gradle Sync** 一次，我想既然是需要组件化的项目那肯定已经是那种编译速度极慢的项目了，即使是 **Gradle Sync** 也需要等待不少时间，这点也是我们不太能接收的。

## 创建多个 Application 模块

我们最后是如何解决模块的单独编译运行这个问题的呢？答案是 **为每个模块新建一个对应的 application 模块** 。也许你会对此表示怀疑：如果为每个业务模块配一个用于独立启动的 **application** 模块，那模块会显得特别多，项目看起来会非常的乱的。但是其实我们可以把所有用于独立启动业务模块的 **application** 模块收录到一个目录中：

```
projectRoot
  +--app
  +--module1
  +--module2
  +--standalone
  |  +--module1Standalone
  |  +--module2Standalone   
```

在上面这个项目结构图中，`app` 模块是全量编译的 **application** 模块入口，`module1` 和 `module2` 是两个业务 **library** 模块， `module1Standalone` 和 `module2Standalone` 是分别使用来独立启动 `module1` 和 `module2` 的 2 个 **application** 模块，这两个模块都被收录在 `standalone` 文件夹下面。事实上，`standalone` 目录下的模块很少需要修改，所以这个目录大多数情况下是属于折叠状态，不会影响整个项目结构的美观。

这样一来，在项目根目录下的 `settings.gradle` 里的代码是这样的：

```groovy
// main app
include ':app'
// library modules
include ':module1'
include ':module2'
// for standalone modules
include ':standalone:module1Standalone'
include ':standalone:module2Standalone'
```

在主 App 模块（`app` 模块）的 `build.gradle` 文件里，我们只需要依赖 `module1` 和 `module2` ，两个 **standalone** 模块只和各自对应的业务模块的独立启动有关，它们不需要被 `app` 模块依赖，所以 `app` 模块的 `build.gradle` 中的依赖部分代码如下：

```groovy
dependencies {
    implementation project(':module1')
    implementation project(':module1')
}
```

那些用于独立运行的 **application** 模块里的 `build.gradle` 文件中，就只有一个依赖，那就是需要被独立运行的 **library** 模块。以 `standalone/module1Standalone` 为例，它对应的 `build.gradle` 中的依赖为：

```groovy
dependencies {
    implementation project(':module1')
}
```

> 在 Android Studio 中创建模块，默认模块是位于项目根目录之下的，如果希望把模块移动到某个文件夹下面，需要对模块右键，选择 "Refactor -- Move" 移动到指定目录之下。

当我们创建好这些 **application** 模块之后，在 Android Studio 的运行小三角按钮旁边，就可以选择我们需要运行哪个模块了：

![](/images/standalone.png)

这样一来，我们首先可以感受到的一点就是模块不再需要改 **gradle.properties** 文件切换 **library** 和 **application** 状态了，也不再需要忍受 **Gradle Sync** 浪费宝贵的开发时间，想全量编译就全量编译，想单独启动就单独启动。

由于专门用于单独启动的 **standalone 模块** 的存在，业务的 **library** 模块只需要按自己是 **library** 模块这一种情况开发即可，不需要考虑自己会变成 **application** 模块，所以无论是新开发一个业务模块还是从一个老的业务模块改造成组件化形式的模块，所要做的工作都会比之前更轻松。而之前提到的，为了让业务模块单独启动所需要的配置、初始化工作都可以放到 **standalone 模块** 里，并且不用担心这些代码被打包到最终 Release 的 App 中，前面例子中提到的用来使模块单独启动的 Launcher Activity，只要把它放到 **standalone 模块** 模块即可。

`AndroidManifest.xml` 和资源文件的维护也变轻松了。四大组件的增删改只需要在业务的 **library** 模块修改即可，不需要维护两份 `AndroidManifest.xml` 了，**standalone 模块** 里的  `AndroidManifest.xml` 只需要包含模块独立启动时和 **library** 模块中的 `AndroidManifest.xml` 不同的地方即可（例如 Launcher Activity 、图标等），编译工具会自动完成两个文件的 merge。 

> 推荐在 **standalone 模块** 内指定一个不同于主 App 的 applicationId，即模块单独启动的 App 与主 App 可以在手机内共存。

我们分析一下这个方案，和原先的比，首先缺点是，引入了很多新的 **standalone 模块**，项目似乎变复杂了。但是优点也是明显的，组件化的逻辑更加清晰，尤其是在老项目改造情况下，所需要付出的工作量更少，而且不需要在开发期间频繁 **Gradle Sync**。 总的来说，改造后的组件化项目更符合软件工程的设计原则，尤其是[开闭原则](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)（open for extension, but closed for modification）。

介绍到这里为止，我们还没有使用任何 [AppJoint](https://github.com/PrototypeZ/AppJoint) 的 API，我们之所以没有借助任何组件化框架的 API 来实现模块的独立启动，是因为本文一开始提出的，**我们不希望项目和任何组件化框架强绑定**, 包括 **AppJoint** 框架本身，**AppJoint** 框架本身的设计是与项目松耦合的，所以使用了 **AppJoint** 框架进行组件化的项目，如果今后希望可以切换到其它更优秀的组件化方案，理论上是很轻松的。

## 为每个模块准备 Application

在组件化之前，我们常常把项目中需要在启动时完成的初始化行为，放在自定义的 `Application` 中，根据本人的项目经验，初始化行为可以分为以下两类：

- **业务相关的初始化**。例如服务器推送长连接建立，数据库的准备，从服务器拉取 CMS 配置信息等。
- **与业务无关的技术组件的初始化**。例如日志工具、统计工具、性能监控、崩溃收集、兼容性方案等。

我们在上一步中，为每个业务模块建立了独立运行的 **standalone 模块** ，但是此时还并不能把业务模块独立启动起来，因为模块的初始化工作并没有完成。我们在前面介绍 **AppJoint** 的设计思想的时候，曾经说过我们希望组件化方案最好 『**不要有太多的学习成本，沿用目前已有的开发方式**』，所以这里我们的解决方案是，在每个业务模块里新建一个自定义的 `Application` 类，用来实现该业务模块的初始化逻辑，这里以在 `module1` 中新建自定义 `Application` 为例：

```java
@ModuleSpec
public class Module1Application extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        // do module1 initialization
        Log.i("module1", "module1 init is called");
    }
}
```

如上面的代码所示，我们在 `module1` 中新建一个自定义的 `Application` 类，名为 `Module1Application`。那我们是不是应该把与这个模块有关的所有初始化逻辑都放在这个类里面呢？并不完全是这样。

首先，对于前面提到的当前模块的 **业务相关的初始化** ，毫无疑问应该放在这个 `Module1Application` 类中，但是针对前面提到的该模块的 **与业务无关的技术组件的初始化** 放在这里就不是很合适了。

首先，从逻辑上考虑，业务无关的技术组件的初始化应该放在一个统一的地方，把它们放在主 App 的自定义 `Application` 类中比较合适，如果每个模块为了自己可以独立编译运行，都要自己初始化一遍，那么所有代码最后一起全量编译的时候，这些初始化行为就会在代码中出现好几次，这样既不合理，也可能会造成潜在问题。

那么，如果我们在 `Module1Application` 中做判断，如果它自身处于独立编译运行状态，就执行技术组件的初始化，反之，若它处于全量编译运行状态中，就不执行技术组件的初始化，由主 App 的 `Application` 来实现这些逻辑，这样是否可以呢？理论上这种方案可行，但是这么做就会遇到和前面提到的 『在 `gradle.properties` 中维护一个变量来控制模块是否独立编译』同样的问题，我们不希望把和业务无关的逻辑（用于业务模块独立启动的逻辑）打包进最终 Release 的 App。 

那应该如何解决这个问题呢？解决方案和前面一小节类似，我们不是为 `module1` 模块准备了一个 `module1Standalone` 模块吗？既然技术相关的组件的初始化并不是 `module1` 模块的核心，只和 `module1` 模块的独立启动有关，那么放在 `module1Standalone` 模块里是最合适的，因为这个模块只会在 `module1` 的独立编译运行中使用到，它的任何代码都不会被打包到最终 Release 的 App 中。我们可以在 `module1Standalone` 中定义一个 `Module1StandaloneApplication` 类，它从 `Module1Application` 继承下来：

```java
public class Module1StandaloneApplication extends Module1Application {

    @Override
    public void onCreate() {
        // module1 init inside super.onCreate()
        super.onCreate();
        // initialization only used for running module1 standalone
        Log.i("module1Standalone", "module1Standalone init is called");
    }
}
```

并且我们在 `module1Standalone` 模块的 `AndroidManifest.xml` 中把 `Module1StandaloneApplication` 设置为 Standalone App 使用的自定义 `Application` 类：

```xml
    <application
        android:icon="@mipmap/module1_launcher"
        android:label="@string/module1_app_name"
        android:theme="@style/AppTheme"
        android:name=".Module1StandaloneApplication">
        <activity android:name=".Module1MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
```

> 在上面的代码中，我们除了设置了自定义的 `Application` 以外，还设置了一个 **Launcher Activity** (`Module1MainActivity`)，这个 `Activity` 即为模块的启动 `Activity`，由于它只存在于模块的独立编译运行期间，App 全量打包时是不包含这个 `Module1MainActivity` 的，所以我们可以在里面定义一些方便模块独立调试的功能，例如快速前往某个页面以及创建 Mock 数据。

这样，只要我们单独运行 `module1Standalone` 这个模块的时候，使用的 `Application` 类就是 `Module1StandaloneApplication`。在开发时，我们需要单独调试 `module1` 时，我们只需要启动 `module1Standalone` 这个模块进行调试即可；而在 App 需要全量编译时，我们则正常启动原来的主 App 。无论是哪种情况， `module1` 这个模块始终是以 `library` 形式存在的，**这意味着，如果我们希望把原先的业务模块改造成组件化模块，需要的改造量缩小很多**，我们改造的过程主要是在 **增加代码**，而不是 **修改代码**，这点符合软件工程中的『**开闭原则**』。

写到这里，我们其实还有一个问题没有解决。`Module1Application` 目前除了被 `Module1StandaloneApplication` 继承以外，没有被任何其它地方引用到。您可能会有疑问：那我们如何保证 App 全量编译运行时，`Module1Application` 里的初始化逻辑会被调用到呢？细心的您可能早就已经发现了：我们在上面定义 `Module1Application` 时，同时标记了一个注解 `@ModuleSpec`:

```java
@ModuleSpec
public class Module1Application extends Application {
    ...
}
```

这个注解的作用是告知 **AppJoint** 框架，我们需要确保当前模块该 `Application` 中的初始化行为，能够在最终全量编译时，被主 App 的 `Application` 类调用到。所以对应的，我们的主 App 模块（`app` 模块）的自定义 `Application` 类也需要被一个注解 -- `AppSpec` 标记，代码如下所示：

```java
@AppSpec
public class App extends Application {
    ...
}
```

上面代码中的 `App` 为主 App 对应的自定义 `Application` 类，我们给这个类上方标记了 `@AppSpec` 注解，这样系统在执行 `App` 自身初始化的同时会一并执行这些子模块的 `Application` 里对应声明周期的初始化。即：
 
+ `App` 执行 `onCreate` 方法时，保证也同时执行 `Module1Application` 和 `Module2Application` 的 `onCreate` 方法 。
+ `App` 执行 `attachBaseContext` 方法时，保证也同时执行 `Module1Application` 和 `Module2Application` 的 `attachBaseContext` 方法。 
+ 依次类推，当 `App` 执行某个生命周期方法时，保证子模块的 `Application` 的对应的生命周期方法也会被执行。

这样，我们通过 **AppJoint** 的 `@ModuleSpec` 和 `@AppSpec` 两个注解，在主 App 的 `Application` 和子模块的 `Application` 之间建立了联系，保证了在全量编译运行时，所有业务模块的初始化行为都能被保证执行。

到这里为止，我们已经处理好了业务模块在 **独立编译运行模式** 和 **全量编译运行模式** 这两种情况下的初始化问题，目前关于 `Application` 还有一个潜在问题，我们的项目在组件化之前，我们经常会在 `Applictaion` 类的 `onCreate` 周期保存当前 `Appliction` 的引用，然后在应用的任何地方都可以使用这个 `Application` 对象，例如下面这样：

```java
public class App extends Application {

    public static App INSTANCE;

    @Override
    public void onCreate() {
        super.onCreate();
        INSTANCE = this;
    }
}
```

这么处理之后，我们可以在项目任意位置通过 `App.INSTANCE` 使用 **Application Context** 对象。但是，现在组件化改造以后，以 `module1` 为例，在独立运行模式时，应用的 `Application` 对象是 `Module1StandaloneApplication` 的实例，而在全量编译运行模式时，应用的 `Application` 对象是主 App 模块的 `App` 的实例，我们如何能像之前一样，做到在项目中任何一个地方都能获取到当前使用的 `Application` 实例呢？

我们可以把项目中所有自定义 `Application` 内部保存的自身的 `Application` 实例的类型，从具体的自定义类，改为标准的 `Application` 类型，以 `Module1Application` 为例：

```java
@ModuleSpec
public class Module1Application extends Application {
    
    public static Application INSTANCE;

    @Override
    public void onCreate() {
        super.onCreate();
        INSTANCE = (Application)getApplicationContext()
        // do module1 initialization
        Log.i("module1", "module1 init is called");
    }
}
```

我们可以看到，如果按原来的写法， `INSTANCE` 的类型一般是具体的自定义类型 `Module1Application`，现在我们改成了 `Application`。同时 `onCreate` 方法里为 `INSTANCE` 赋值的语句不再是 `INSTANCE = this`，而是 `INSTANCE = (Application)getApplicationContext()`。这样处理以后，就可以保证 `module1` 里面的代码，无论是在 App 全量编译模式下，还是独立编译调试模式下，都可以通过 `Module1Application.INSTANCE` 访问当前的 `Application` 实例。这是由于 **AppJoint** 框架 **保证了当主 App 的 `App` 对象被调用 `attachBaseContext` 回调时，所有组件化业务模块的 `Application` 也会被调用 `attachBaseContext` 这个回调**。

这样，我们在 `module1` 这个模块里的任何位置使用 `Module1Application.INSTANCE` 总能正确地获得 `Application` 的实例。对应的，我们使用相同的方法在 `module2` 这个模块里，也可以在任何位置使用 `Module2Application.INSTANCE` 正确地获得 `Application` 的实例，而不需要知道当前处于独立编译运行状态还是全量编译运行状态。

> **一定不要** 依赖某个业务模块自身定义的 `Application` 类的实例（例如 `Module1Application` 的实例），因为在运行时真正使用的 `Application` 实例可能不是它。

我们已经解决业务模块在 **单独编译运行模式** 下和在 **App 全量编译模式** 下，初始化逻辑应该如何组织的问题。我们沿用了我们熟悉的自定义 `Application` 方案，来承载各个模块的初始化行为，同时利用 **AppJoint** 这个胶水，把每个模块的初始化逻辑集成到最终全量编译的 App 中。而这一切和 **AppJoint** 有关的 API 仅仅是两个注解，这里很好的说明了 **AppJoint** 是个学习成本低的工具，我们可以沿用我们已有的开发方式而不是改造我们原有的代码逻辑导致项目和组件化框架造成过度耦合。

## 跨模块方法的调用

虽然目前每个模块已经有独立编译运行的可能了，但是开发一个成熟的 App 我们还有一个重要的问题没有解决，那就是跨模块的方法调用。因为我们的业务模块无论是从业务逻辑上考虑还是从在依赖树上的位置考虑，都应该是具有同等的地位的，体现在依赖层次上，这些业务模块应该是平级的，且互相之间没有依赖：

![](/images/app-joint-structure-1.png)

上图是我们比较理想情况下的组件化的最终状态，`App` 模块不承载任何业务逻辑，它的作用仅仅是作为一个 `application` 壳把 `Module1` ~ `Module(n)` 这个 n 个模块的功能都集成在一起成为一个完整的 App。`Module1` ~ `Module(n)` 这 n 个模块互相之间不存在任何交叉依赖，它们各自仅包含各自的业务逻辑。这种方式虽然完成了业务模块之间的解耦，但是给我们带来的新的挑战：业务模块之间互相调用彼此的功能是非常常见且合理的需求，但是由于这些模块在依赖层次上位于同一层次，所以显然是无法直接调用的。

此外，上图的这种形态是组件化的最终的理想状态，如果我们要将项目改造以达到这种状态，毫无疑问需要付出巨大的时间成本。在业务快速迭代期间，这是我们无法承担的成本，我们只能逐渐地改造项目，也就是说，`App` 模块内的业务代码是被逐渐拆解出来形成新的独立模块的，这意味着在组件化过程的相当长一段时间内，`App` 内还是存在业务代码的，而被拆解出来的模块内的业务逻辑代码，是有可能调用到 `App` 模块内的代码的。这是一种很尴尬的状态，在依赖层次中，位于依赖层次较低位置的代码反而要去调用依赖层次较高位置的代码。

针对这种情况，我们比较容易想到，我们再新建一个模块，例如 `router` 模块，我们在这个模块内定义 **所有业务模块希望暴露给其它模块调用的方法**，如下图：

```
projectRoot
  +--app
  +--module1
  +--module2
  +--standalone
  +--router
  |  +--main
  |  |  +--java
  |  |  |  +--com.yourPackage
  |  |  |  |  +--AppRouter.java
  |  |  |  |  +--Module1Router.java
  |  |  |  |  +--Module2Router.java
```

在上面的项目结构层次中，我们在新建的 `router` 模块下定义了 3 个 **接口**：

+ `AppRouter` 接口声明了 `app` 模块暴露给 `module1`、`module2` 的方法的定义。
+ `Module1Router` 接口声明了 `module1` 模块暴露给 `app`、`module2` 的方法的定义。
+ `Module2Router` 接口声明了 `module2` 模块暴露给 `module1`、`app` 的方法的定义。

以 `AppRouter` 接口文件为例，这个接口的定义如下：

```java
public interface AppRouter {

    /**
     * 普通的同步方法调用
     */
    String syncMethodOfApp();

    /**
     * 以 RxJava 形式封装的异步方法
     */
    Observable<String> asyncMethod1OfApp();

    /**
     * 以 Callback 形式封装的异步方法
     */
    void asyncMethod2OfApp(Callback<String> callback);
}
```

我们在 `AppRouter` 这个接口内定义了 1 个同步方法，2 个异步方法，这些方法是 `app` 模块需要暴露给 `module1` 、 `module2` 的方法，同时 `app` 模块自身也需要提供这个接口的实现，所以首先我们需要在 `app` 、`module1` 、`module2` 这三个模块的 `build.gradle` 文件中依赖 `router` 这个模块：

```groovy
dependencies {
    // Other dependencies
    ...
    api project(":router")
}
```

> 这里依赖 `router` 模块的方式是使用 `api` 而不是 `implementation` 是为了把 `router` 模块的信息暴露给依赖了这些业务模块的 **standalone 模块**，`app` 模块由于没有别的模块依赖它，不受上面所说的限制，可以写成 `implementation` 依赖。

然后我们回到 `app` 模块，为刚刚在 `router` 定义的 `AppRouter` 接口提供一个实现：

```java
@ServiceProvider
public class AppRouterImpl implements AppRouter {

    @Override
    public String syncMethodOfApp() {
        return "syncMethodResult";
    }

    @Override
    public Observable<String> asyncMethod1OfApp() {
        return Observable.just("asyncMethod1Result");
    }

    @Override
    public void asyncMethod2OfApp(final Callback<String> callback) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                callback.onResult("asyncMethod2Result");
            }
        }).start();
    }
}
```

我们可以发现，我们把 `app` 模块内的方法暴露给其它模块的方式和我们平时写代码并没有什么不同，就是声明一个接口提供给其它模块，同时在自己内部编写一个这个接口的实现类。无论是同步还是异步，无论是 Callback 的方式，还是 RxJava 的方式，都可以使用我们原有的开发方式。唯一的区别就是，我们在 `AppRouterImpl` 实现类上方标记了一个 `@ServiceProvider` 注解，这个注解的作用是用来通知 `AppJoint` 框架在 `AppRouter` 和 `AppRouterImpl` 之间建立联系，这样其它模块就可以通过 `AppJoint` 找到一个 `AppRouter` 的实例并调用里面的方法了。

假设现在 `module1` 中需要调用 `app` 模块中的 `asyncMethod1OfApp` 方法，由于 `app` 模块已经把这个方法声明在了 `router` 模块的 `AppRouter` 接口中了，`module1` 由于也依赖了 `router` 模块，所以 `module1` 内可以访问到 `AppRouter` 这个接口，但是却访问不到 `AppRouterImpl` 这个实现类，因为这个类定义在 `app` 模块内，这时候我们可以使用 **AppJoint** 来帮助 `module1` 获取 `AppRouter` 的实例：

```java
AppRouter appRouter = AppJoint.getRouter(AppRouter.class);

// 获得同步调用的结果        
String syncResult = appRouter.syncMethodOfApp();
// 发起异步调用
appRouter.asyncMethod1OfApp()
        .subscribe((result) -> {
            // handle asyncResult
        });
// 发起异步调用
appRouter.asyncMethod2OfApp(new Callback<String>() {
    @Override
    public void onResult(String data) {
        // handle asyncResult
    }
});
```

在上面的代码中，我们可以看到，除了第一步获取 `AppRouter` 接口的实例我们用到了 **AppJoint** 的 API `AppJoint.getRouter` 以外，剩下的代码，`module1` 调用 `app` 模块内的方法的方式，和我们原来的开发方式没有任何区别。`AppJoint.getRouter` 就是 **AppJoint** 所有 API 里唯一的那个方法。

也就是说，如果一个模块需要提供方法供其他模块调用，需要做以下步骤：

+ 把接口声明在 `router` 模块中
+ 在自己模块内部实现上一步中声明的接口，同时在实现类上标记 `@ServiceProvider` 注解

完成这两步以后就可以在其它模块中使用以下方式获取该模块声明的接口的实例，并调用里面的方法：

```java
AppRouter appRouter = AppJoint.getRouter(AppRouter.class);
Module1Router module1Router = AppJoint.getRouter(Module1Router.class);
Module2Router module2Router = AppJoint.getRouter(Module2Router.class);
```

这种方法不仅仅可以保证处于相同依赖层次的业务模块可以互相调用彼此的方法，还可以支持从业务模块中调用 `app` 模块内的方法。这样就可以 **保证我们组件化的过程可以是渐进的** ，我们不需要一口气把 `app` 模块中的所有功能全部拆分到各个业务模块中，我们可以逐渐地把功能拆分出来，以保证我们的业务迭代和组件化改造同时进行。当我们的 `AppRouter` 里面的方法越来越少直到最后可以把这个类从项目中安全删除的时候，我们的组件化改造就完成了。

## 模块独立编译运行模式下跨模块方法的调用

上面一个小结中我们已经介绍了使用 **AppJoint** 在 App 全量编译运行期间，业务模块之间跨模块方法调用的解决方案。在全量编译期间，我们可以通过 `AppJoint.getRouter` 这个方法找到指定模块提供的接口的实例，但是在模块单独编译运行期间，其它的模块是不参与编译的，它们的代码也不会打包进用于模块独立运行的 **standalaone 模块**，我们如何解决在模块单独编译运行模式下，跨模块调用的代码依然有效呢？

以 `module1` 为例，首先为了便于在 `module1` 内部任何地方都可以调用其它模块的方法，我们创建一个 `RouterServices` 类用于存放其它模块的接口的实例：

```java
public class RouterServices {
    // app 模块对外暴露的接口
    public static AppRouter sAppRouter = AppJoint.getRouter(AppRouter.class);
    // module2 模块对外暴露的接口
    public static Module2Router sModule2Router = AppJoint.getRouter(Module2Router.class);
}
```

有了这个类以后，我们在 `module1` 内部如果需要调用其它模块的功能，我们只需要使用 `RouterServices.sAppRouter` 和 `RouterServices.sModule2Router` 这两个对象就可以了。但是如果是刚刚提到的 `module1` 独立编译运行的情况，即启动的 `application` 模块是 `module1Standalone`， 那么 `RouterServices.sAppRouter` 和 `RouterServices.sModule2Router` 这两个对象的值均为 `null` ，这是因为 `app` 和 `module2` 这两个模块此时是没有被编译进来的。

如果我们需要在这种情况下保证已有的 `module1` 内部的通过 `RouterServices.sAppRouter` 和 `RouterServices.sModule2Router` 进行跨模块方法调用的代码依然能工作，我们就需要对这两个引用手动赋值，即我们需要创建 Mock 了 `AppRouter` 和 `Module2Router` 功能的类。这些类由于只对 `module1` 的独立编译运行有意义，所以这些类最合适的位置是放在 `module1Standalone` 这个模块内，以 `AppRouter` 的 Mock 类 `AppRouterMock` 为例：

```java
public class AppRouterMock implements AppRouter {
    @Override
    public String syncMethodOfApp() {
        return "mockSyncMethodOfApp";
    }

    @Override
    public Observable<String> asyncMethod1OfApp() {
        return Observable.just("mockAsyncMethod1OfApp");
    }

    @Override
    public void asyncMethod2OfApp(final Callback<String> callback) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                callback.onResult("mockAsyncMethod2Result");
            }
        }).start();
    }
}
```

已经创建好了 Mock 类，接下来我们要做的是，在 `module1` 独立编译运行的模式下，用 Mock 类的对象，去替换 `RouterServices` 里面的对应的引用，由于这些逻辑只和 `module1` 的独立编译运行有关，我们不希望这些逻辑被打包进真正 Release 的 App 中，那么最合适的地方就是 `Module1StandaloneApplication`里了：

```java
public class Module1StandaloneApplication extends Module1Application {

    @Override
    public void onCreate() {
        // module1 init inside super.onCreate()
        super.onCreate();
        // initialization only used for running module1 standalone
        Log.i("module1Standalone", "module1Standalone init is called");

        // Replace instances inside RouterServices
        RouterServices.sAppRouter = new AppRouterMock();
        RouterServices.sModule2Router = new Module2RouterMock();
    }
}
```

有了上面的初始化动作以后，我们就可以在 `module1` 内部安全地使用 `RouterServices.sAppRouter` 和 `RouterServices.sModule2Router` 这两个对象进行跨模块的方法调用了，无论当前是处于 App 全量编译模式还是 `modul1Standalone` 独立编译运行模式。

## 跨模块启动 Activity 和 Fragment

在组件化改造过程中，除了跨模块的方法调用之外，跨模块启动 Activity 和跨模块引用 Fragment 也是我们经常遇到的需求。目前社区中大多数组件化方案都是使用自定义私有协议，使用 **URL-Scheme** 的方式来实现跨模块 Activity 的启动，这一块已经有很多成熟的方案了，有的组件化方案直接推荐使用 **ARouter** 来实现这块功能。**但是 AppJoint 没有使用这类方案**。

本文开头曾经介绍过，**AppJoint** 所有的 API 只包含 **3** 个注解加 **1** 个方法，而这些 API 我们在前文中已经都介绍完了，也就是说，**我们没有提供专门的 API 来实现跨模块的 Activity / Fragment 调用**。

我们回想一下，在没有实现组件化时，我们启动 Activity 的推荐写法如下，首先在被启动的 Activity 内实现一个静态 `start` 方法:

```java
public class MyActivity extends AppCompatActivity {

    public static void start(Context context, String param1, Integer param2) {
        Intent intent = new Intent(context, MyActivity.class);  
        intent.putExtra("param1", param1);  
        intent.putExtra("param2", param2);  
        context.startActivity(intent);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ...
    }
}
```

然后我们如果在其它 Activity 中启动这个 `MyActivity` 的话，写法如下：

 ```java
 MyActivity.start(param1, param2);
 ```

这里的思想是，服务的提供者应该把复杂的逻辑放在自己这里，而只提供给调用者一个简单的接口，用这个简单的接口隔离具体实现的复杂性，这是符合软件工程思想的。

那么如果目前 `module1` 模块中有一个 `Module1Activity`，现在这个 Activity 希望能够从 `module2` 启动，应该如何写呢？首先，在 `router` 模块的 `Module1Router` 内声明启动 `Module1Activity` 的方法：

```java
public interface Module1Router {
    ...
    // 启动 Module1Activity
    void startModule1Activity(Context context);
}
```

然后在 `module1` 模块里 `Module1Router` 对应的实现类 `Module1RouterImpl` 中实现刚刚定义的方法：

```java
@ServiceProvider
public class Module1RouterImpl implements Module1Router {

    ...

    @Override
    public void startModule1Activity(Context context) {
        Intent intent = new Intent(context, Module1Activity.class);
        context.startActivity(intent);
    }
}
```

这样， `module2` 中就可以通过下面的方式启动 `module1` 中的 `Module1Activity` 了。

```java
RouterServices.sModule1Router.startModule1Activity(context);
```

跨模块获取 `Fragment` 实例也是类似的方法，我们在 `Module1Router` 里继续声明方法： 

```java
public interface Module1Router {
    ...
    // 启动 Module1Activity
    void startModule1Activity(Context context);

    // 获取 Module1Fragment
    Fragment obtainModule1Fragment();
}
```

差不多的写法，我们只要在 `Module1RouterImpl` 里接着实现方法即可:

```java
@ServiceProvider
public class Module1RouterImpl implements Module1Router {
    @Override
    public void startModule1Activity(Context context) {
        Intent intent = new Intent(context, Module1Activity.class);
        context.startActivity(intent);
    }

    @Override
    public Fragment obtainModule1Fragment() {
        Fragment fragment = new Module1Fragment();
        Bundle bundle = new Bundle();
        bundle.putString("param1", "value1");
        bundle.putString("param2", "value2");
        fragment.setArguments(bundle);
        return fragment;
    }
}
```

前面提到过，目前社区大多数组件化方案都是使用 **自定义私有协议，利用 URL-Scheme 的方式来实现跨模块页面跳转** 的，即类似 **ARouter** 的那种方案，为什么 **AppJoint** 不采用这种方案呢？

原因其实很简单，假设项目中没有组件化的需求，我们在同一个模块内进行 Activity 的跳转，肯定不会采用 **URL-Scheme** 方式进行跳转，我们肯定是自己创建 `Intent` 进行跳转的。其实说到底，使用 **URL-Scheme** 进行跳转是 **不得已而为之**，它只是手段，不是目的，因为在组件化之后，模块之间彼此的 Activity 变得不可见了，所以我们转而使用 **URL-Scheme** 的方式进行跳转。

现在 **AppJoint** 重新支持了使用代码进行跳转，只需要把跳转的逻辑抽象为接口中的方法暴露给其它模块，其它模块就可以调用这个方法实现跳转逻辑。除此以外，使用接口提供跳转逻辑相比 **URL-Scheme** 方式还有什么优势呢？

1. 类型安全。充分利用 Java 这种静态类型语言的编译器检查功能，通过接口暴露的跳转方法，无论是传参还是返回值，如果类型错误，在编译期间就能发现错误，而使用 **URL-Scheme** 进行跳转，如果发生类型上的错误，只能在运行期间才能发现错误。

2. 效率高。即使是使用 **URL-Scheme** 进行跳转，底层仍然是构造 `Intent` 进行跳转，但是却额外引入了对跳转 URL 进行构造和解析的过程，涉及到额外的序列化和反序列化逻辑，降低了代码的执行效率。而使用接口提供的跳转逻辑，我们直接构造 `Intent` 进行跳转，不涉及到任何额外的序列化和反序列化操作，和我们日常的 Activity 跳转逻辑执行效率相同。

3. IDE 友好。使用 **URL-Scheme** 进行跳转，IDE 无法提供任何智能提示，只能依靠完善的文档或者开发者自身检查来确保跳转逻辑的正确性，而通过接口提供跳转逻辑可以最大限度发挥 IDE 的智能提示功能，确保我们的跳转逻辑是正确的。

4. 易于重构。使用 **URL-Scheme** 进行跳转，如果遇到跳转逻辑需要重构的情况，例如 Activity 名字的修改，参数名称的修改，参数数量的增删，只能依靠开发者对使用到跳转逻辑的地方一个一个修改，而且无法确保全部都修改正确了，因为编译器无法帮我们检查。而通过接口提供的跳转逻辑代码需要重构时，编译器可以自动帮助我们检查，一旦有地方没有改对，直接在编译期报错，而且 IDE 都提供了智能重构的功能，我们可以方便地对接口中定义的方法进行重构。

5. 学习成本低。我们可以沿用我们熟悉的开发方式，不需要去学习  **URL-Scheme** 跳转框架的 API。这样还可以保证我们的跳转逻辑不与具体的框架强绑定，我们通过接口隔离了跳转逻辑的真正实现，即使使用 **AppJoint** 进行跳转，我们也可以在随时把跳转逻辑切换到其他方案，包括 **URL-Scheme** 方式。

我个人的实践，目前项目中同一进程内的页面跳转已经全部由 **AppJoint** 的方式实现，目前只有跨进程的页面启动交给了 **URL-Scheme** 这种方式（例如从浏览器唤醒 App 某个页面）。

最后再提一点，由于跨模块启动 Activity 沿用了跨模块方法调用的开发方式，在业务模块单独编译运行模式下，我们也需要 Mock 这些启动方法。既然我们是在独立调试某个业务模块，我们肯定不是真的希望跳转到那些页面，我们在 Mock 方法里直接输出 Log 或者 Toast 即可。

## 现在就开始组件化

[AppJoint](https://github.com/PrototypeZ/AppJoint) 的 Github 地址为：[https://github.com/PrototypeZ/AppJoint](https://github.com/PrototypeZ/AppJoint) 。核心代码不超过 500 行，您完全可以快速掌握这个工具加速您的组件化开发，只要 Fork 一份代码即可。如果您不想自己引入工程，我们也提供了一个开箱即用的版本，您可以直接通过 **Gradle** 引入。

1. 在项目根目录的 `build.gradle` 文件中添加 **AppJoint插件** 依赖：

```groovy
buildscript {
    ...
    dependencies {
        ...
        classpath 'io.github.prototypez:app-joint:{latest_version}'
    }
}
```

2. 在主 App 模块和每个组件化的模块添加 **AppJoint** 依赖：

```groovy
dependencies {
    ...
    implementation "io.github.prototypez:app-joint-core:{latest_version}"
}
```

3. 在主 App 模块应用 **AppJoint插件**： 

```groovy
apply plugin: 'com.android.application'
apply plugin: 'app-joint'
```

## 写在最后

通过本文的介绍，我们其实可以发现 **AppJoint** 是个思想很简单的组件化方案。虽然简单，但是却直接而且够用，尽管没有像其它的组件化方案那样提供了各种各样强大的 API，但是却足以胜任大多数中小型项目，这是我们一以贯之的设计理念。

如果您感觉这个项目对您有帮助，希望可以点一个 Star ，谢谢 : )文章很长，感谢您耐心读完。由于本人能力有限，文章可能存在纰漏的地方，欢迎各位指正，再次谢谢大家！
___
如果您对我的技术分享感兴趣，欢迎关注我的个人公众号：麻瓜日记，不定期更新原创技术分享，谢谢！:)

![](http://prototypez.github.io/images/qrcode.jpg)