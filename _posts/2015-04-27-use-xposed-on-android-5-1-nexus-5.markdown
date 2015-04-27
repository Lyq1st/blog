---
layout: post
title: "Use Xposed On Android 5.1 Nexus 5"
date: 2015-04-27 19:50:23 +0800
comments: true
categories: 
    - Mobile
description: 在nexus 5 安装 lolipop 5.1 使用 xposed framework
keywords: nexus5 安装 lolipop 5.1 android 使用 xposed 
redirect_from: /p/20150427/
---

Xposed今年2月开始支持lolipop，终于可以尝试升级到5.1感受一把了。*[注：以下均为撰文时的结论，以后的变动请随机应变]*

<!-- more -->

###Update 4.4.4 to 5.1

升级有两种方法：OTA和factory image

OTA需要挂代理才能升级，默认不会清除用户数据。
先unlock设备，factory image 升级可以使用以下操作：
> 
```sh
adb reboot bootloader
fastboot oem unlock
fastboot flash bootloader <bootloader-filename>.img
fastboot flash radio <radio-fileame>.img
fastboot reboot-bootloader
fastboot flash recovery recovery.img
fastboot flash boot boot.img
fastboot flash system system.img
fastboot flash cache cache.img
fastboot flash userdata userdata.img
```

使用自带的脚本有些*factory image*版本上，可能会出现各种莫名其妙的问题 最后`userdata.img`，会擦除用户数据，一定要想清楚是不是要擦除数据再执行。

###Root Nexus 5 lolipop 5.1

* 开启DEBUG模式，允许安装第三方APK。
* 刷最新的`TWRP`，目前最新的是[TWRP2.8.5.2][],nexus5上用起来未发现问题，更名为recovery.img。
```sh
	fastboot flash recovery recovery.img
```
* 下载super SU，可以使用[Super SU][]搜索下载APK，下载[UPDATE-SuperSU-v2.46.zip][]
* 进入recovery 在TWRP中install 刷入 Super SU。
* reboot system 安装 Super su，完成root。

###Install Xposed

根据XDA 论坛站中关于xposed帖子来看，当前已经有体验版支持支持5.1了，5.0，5.0.1相对来说支持的还相对好一些。既然要升级索性升级到最高吧。


Installing Xposed on Android 5.1

* 下载[XposedInstaller_3.0-alpha2.apk][]、[xposed-arm-20150308-5.1.zip][]
* 进入recovery 清理 dalvik缓存文件，清理data/data/ 下老版本的xposed数据，主要是conf下的*.xml，还有modulelist.xml，防止卡在开机界面无法开机（如果遇到该情况也可以这样子处理）
* 刷入xposed-arm-20150308-5.1.zip，这个版本不能开启签名校验，就不要勾选了。
* reboot system 安装XposedInstaller_3.0-alpha2.apk，完成xposed的安装。

5.1上测试了几个module，其中 *app ops、BootManager*是可以支持5.1的，*xprivacy*会一直重启和重复优化程序过程，无法开机，可以通过清理data目录解决，实在无解的时候直接到bootloader中fastboot。
```sh
fastboot flash system system.img
```
system.img是从factory image中解压出来的，以防不测可以单独下一份备用。

也有人给出了一个兼容的module的列表[Module list][]，有些可能不准，其中*Greenify*可以直接不依赖于*Xposed*，能够在5.1上使用，目前来看唯一的缺憾就是没能用上*Xprivacy*了。

[XposedInstaller_3.0-alpha2.apk]: http://forum.xda-developers.com/attachment.php?attachmentid=3200857&d=1425849071
[xposed-arm-20150308-5.1.zip]: http://forum.xda-developers.com/attachment.php?attachmentid=3245467&d=1428192863
[TWRP2.8.5.2]: http://techerrata.com/file/twrp2/hammerhead/openrecovery-twrp-2.8.5.2-hammerhead.img
[Super SU]: http://apps.evozi.com/apk-downloader/?id=eu.chainfire.supersu
[UPDATE-SuperSU-v2.46.zip]: http://download.chainfire.eu/supersu
[Module list]: https://docs.google.com/spreadsheets/d/10vutBHBlPtEFtnqNQBuz-IzStlXNGbVS6K0BRpwz7n0/edit#gid=0
