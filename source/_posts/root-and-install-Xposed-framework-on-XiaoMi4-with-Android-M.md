---
title: 小米4 Android 6.0 版本 Root 并安装 Xposed 框架攻略
desc: '小米4 Android 6.0(M) root 攻略，并且安装Xposed框架'
date: 2016-05-16 09:47:16
tags:
---
本人自 Android 开发入坑一周年以来，向来对Root设备不太感冒。我对Root的设备的态度和对苹果的越狱差不多。大学期间有个舍友越狱了自己的 iPhone，据说从此可以下载许多的收费应用和游戏，所以越狱给我的映像就是破解软件的收费限制，Root也差不多。自己成为一名开发者之后，深感 Google 大法好，不 Root 也能把 Android 玩得很好，然而我毕竟还是 too young too simple，在微信群的抢红包大战中败给越狱之后的 iPhone 的抢红包插件之后，我决定研究 Android 上的抢红包插件。

<!--  More -->

上 Github 搜红包插件，果然层出不穷，知乎大神纷纷表示情绪稳定，谈笑间抛出一个名词 -- AccessibilityService。据说该 API 设计的初衷是为有障碍人士提供更方便的操作手机的的选择，只要用户为 App 打开这个这个权限的开关，App 就可以模拟用户对应用进行操作：解析当前界面，点击，等等。我下载了 Github 上一个高 star 的抢红包插件，测试了一下，确实可以帮助用户点开红包，并拆开，比手点是快了点，但是这个东西一点都不稳定，常常红包出现了，却不去点，或者点开了红包不拆开。另一个问题就是，这样抢红包依然太慢，还是要观看拆红包的那个动画，达不到毫秒级抢红包，与越狱后的 iPhone 抢红包插件性能相去甚远。

另一个容易想到的方法就是在红包到达时，调用微信的拆红包操作 API（假设能逆向微信代码，并且找到对应代码），这里涉及到的技术难点是 Hook 微信的方法，目前有比较成熟的方案便是 [Xposed](https://github.com/rovo89/Xposed)。

## Root 小米4
Xposed 需要 Root 权限，我手上有一个小米4，大概一个月以前跟着 MIUI 升级升到了 Android 6.0.1，发现以前很好用的傻瓜 Root 工具都不能用了，包括 KingRoot， Root精灵，Root大师等等，似乎 Android 6.0.1 对 Root 这件事控制得更加严格了。Google 了一圈 “Android 6.0 小米4 Root”的关键字以后也没有找到现成的教程，本来开始有点心灰意冷，突然看到 MIUI 论坛有个人评论，“我还没发现 SuperSU 不能 Root 的系统”，这下点燃的我的希望。

我找到了一个 SuperSU 的[最新版本](http://www.zdfans.com/3562.html)，然后被一堆名词看得一脸懵逼。
>刷机包使用步骤：

>方法一：Recovery 刷入（推荐）

>1、下载刷机包，复制到设备的SD卡中；

>2、设备进入 CWM/TWRP Recovery（原厂 Recovery 不能刷）；

>3、在 Recovery 中将刚刚复制到 SD 卡的刷机包刷入；

>4、重启设备，更新完成。

![](http://img.doutula.com/production/uploads/image//2016/02/28/20160228614309_TquwJx.jpeg)

作为一个 Android 开发居然对上面这些名词完全没概念真是惭愧。在搜索引擎里补了一番课之后，明白了 Recovery 大概是 Android 系统的一种恢复机制，进入这个模式以后可以对系统进行一些修改，包括写入一些新的内容，甚至重新写入整个系统。如果要刷入 SuperSU 的话需要使用第三方的 Recovery 写入，CWM 和 TWRP 都是第三方的 Recovery。

随即 Google 了 [CWM](https://www.clockworkmod.com/rommanager) 和 [TWRP](https://www.clockworkmod.com/rommanager) 的两个工具的官网， 发现没有对应小米4 ROM的下载，于是转战搜索国内百度，找到了一个国内大神为小米4定制的 [CWM Recovery](http://www.muzisoft.com/recovery/93443.html)，然后我需要为我的小米4刷入这个 Recovery。关闭手机，按住音量下键与开关键开机，手机进入 FastBoot 模式，把手机与电脑使用 USB 线连接起来。把上面提到的 Recovery 解压以后，里面有一个文件名叫**一键刷入Recovry.bat**（这里提一下，本人使用 Windows 系统，如果使用 Mac 或者 Linux 就需要你打开这个文件，参考着修改成对应的版本了。），打开命令行，进入这个文件所在的目录，执行这个文件，就OK了，简直是我等懒人的福音。手机会重启进入 CMW Recovery。

## 刷入 Super SU

在 CMW Recovery 的起始界面，会有“系统1”，“系统2”这样的选项，选择“系统1”进入，这时候你的命令行应该还停留在刚才的目录，在把 *Super SU* 刷入之前，首先我们需要把下载下来的 Super SU 的复制到手机SD卡上，在命令行执行 `adb push /path/to/your/super-su /sdcard/`, 然后在 CMW Recovery 的界面选择从 SD 卡刷入指定 zip 包。

很快就刷完了，这时候从界面上选择返回，重启手机，在重启前，该程序会询问是否禁止小米的自动安装Recovery脚本，该脚本会自动安装小米官方 Recovery，从而导致即使我们在 FastBoot 刷入了第三方 Recovery 无效，因为在等到下次进入时，小米的 Recovery 会覆盖第三方的 Recovery，所以到底要不要覆盖，取决于你自己的需求吧，重启结束后，你会发现，手机已经 Root 了，可以下载一个需要 Root 才能玩的 App，你会发现它会请求 Root 权限了，不过，没有什么特殊需求尽量不要去允许 App 的 Root 权限请求。此时我的心情大概是这样的。

![](http://pics.sc.chinaz.com/Files/pic/faces/3574/01.jpg)

## 安装 Xposed 框架

这个简单，上应用宝市场，直接搜一个**Xposed Installer**，下载就可以了。不过马上就碰到坑了，进入**Xposed Installer**后所有和“安装”，“卸载”有关的按钮都不可点，而且会有错误提示，Xposed 框架未激活。

![](https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcTGzszyt15kpG_4J8sUQIWi4AtU4YLWVTozmPFVQxqwUn-Hem_uRA)

没办法，只好再上 Google 去补课了，在[Xposed](http://repo.xposed.info/module/de.robv.android.xposed.installer)的官网，我发现在一开头的申明就说到，如果是 Android 5.0 及以上的版本，安装方法请移步到[另外一个帖子](http://forum.xda-developers.com/showthread.php?t=3034811)，我跟了过去。帖子的大意就是，你需要首先安装最新的**Xposed Installer**（应用宝上的已经是最新的了），然后你需要刷入一个名为**xposed*.zip**的 zip 包，帖子里提供了链接，需要注意的是 sdk 版本选 23，且架构选 amd，因为小米处理器是32位的。

好在我们之前已经有过通过第三方 Recovery 刷入 zip 包的经验了，我们重复之前的方法，刷入这个 zip 包，这次刷完重启，就是漫长的等待，就和小米每次更新系统时那么慢。

功夫不负有心人，我们重启系统，发现。。。Xposed 框架依然没激活，尽管这时候报错信息已经发生改变了。经过多番查找，原因是刚刚刷入的 zip 包对小米支持不好，所以无效了。我也找到了解决方案，在刚才的论坛中，居然有国外大神发布了[针对 MIUI 修改的 Xposed 的 zip 刷机包](http://forum.xda-developers.com/xposed/unofficial-xposed-miui-t3367634)，简直是真爱米粉，可见小米还是有挺强的国际号召力的。

这里的具体过程我也就不说了，还是重复上面的类似操作，这时候刷机完重启手机，你会发现 Xposed 已经成功激活了！

## 结语

我已经装上了抢红包插件，我要和 iOS 红包插件大战五百回合。

![](http://www.people.com.cn/mediafile/pic/20151001/74/11063092384485792194.jpg)
