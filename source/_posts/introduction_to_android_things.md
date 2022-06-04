---
title: Android Things介绍
date: 2017-02-22 12:05:49
categories: Android开发
tags:
- Android开发
- 翻译
---

# 介绍

通过提供与 Android 开发相同的工具，一流的 Android 开发框架，以及 Google APIs这些使得开发者在移动平台上大获成功的东西，Android Things使开发连接的嵌入式变得简单。
<!--more-->
![Android Things Platform Architecture](../images/1315506-c906a6bcdd9dd803.png)

相对于手机和平板，嵌入式设备的 Apps 的开发者与硬件外围设备和驱动离得更近。此外，典型的嵌入式设备只呈现一个单独的 app 体验给用户。本文档介绍了核心Android开发和Android Things之间的主要的增加，省略和差异。

Android Things 通过由 Things Support Library 提供的额外 APIs 扩展了核心 Android 框架。这些 APIs 允许 将apps 与在移动设备上不曾发现的新类型硬件进行集成。

Android Things平台也简化了单个应用程序的使用。系统应用程序不再出现，您的应用程序会在系统启动时自动启动，让使用者沉浸在应用程序体验中。

# Things Support Library
## 外围设备 I/O API
外围设备 I/O API 让你的应用程序可以使用工业标准协议和接口与传感器和致动器通信。下面的接口得到了支持：GPIO，PWM，I2C，SPI，UART。

请参考 [Peripheral I/O API Guides](https://developer.android.com/things/sdk/pio/index.html) 获得更多关于如何使用这些 APIs 的信息。

## 用户驱动 API
用户驱动扩展现有 Android 框架服务，并允许应用程序向框架注入硬件事件，而其它应用程序可以使用标准 Android APIs 访问。

请参考 [User Driver API Guides](https://developer.android.com/things/sdk/drivers/index.html) 获得更多关于如何使用这些 APIs 的信息。

# 行为变化
## 核心应用程序包
Android Things 不包含系统应用程序和 content providers 的标准集合。避免在你的应用程序中使用 [common intents](https://developer.android.com/guide/components/intents-common.html) 和如下的这些 content provider：

[CalendarContract](https://developer.android.com/reference/android/provider/CalendarContract.html)
[ContactsContract](https://developer.android.com/reference/android/provider/ContactsContract.html)
[DocumentsContract](https://developer.android.com/reference/android/provider/DocumentsContract.html)
[DownloadManager](https://developer.android.com/reference/android/app/DownloadManager.html)
[MediaStore](https://developer.android.com/reference/android/provider/MediaStore.html)
[Settings](https://developer.android.com/reference/android/provider/Settings.html)
[Telephony](https://developer.android.com/reference/android/provider/Telephony.html)
[UserDictionary](https://developer.android.com/reference/android/provider/UserDictionary.html)
[VoicemailContract](https://developer.android.com/reference/android/provider/VoicemailContract.html)

## 显示是可选的
Android Things 使用与传统的 Android 应用程序可用的相同的 [UI toolkit](https://developer.android.com/guide/topics/ui/index.html) 来支持图形用户界面。在图形模式下，应用程序窗口占据显示器的全部空间。Android Things 不包含系统状态栏或导航按钮，给予应用程序对可视用户体验的完全控制。

然而，显示设备对于 Android Things 而言不是 *必需的*。在没有图形显示器的设备上，activities 依然是你的 Android Things 应用程序的一个主要组件。这是由于框架传送所有 [输入事件](https://developer.android.com/guide/topics/ui/ui-events.html) 给前台activity，它握有焦点。你的应用程序无法通过任何其它应用程序组件接收按键事件或移动事件，比如一个 [service](https://developer.android.com/guide/components/services.html)。

## Home activity支持
Android Things 期待一个应用程序在它的清单中暴露一个 "home activity" 作为系统在启动时自动启动的主入口。这个 activity 必须包含一个包含 [CATEGORY_DEFAULT](https://developer.android.com/reference/android/content/Intent.html#CATEGORY_DEFAULT) 和 IOT_LAUNCHER 的 intent filter。

为了开发的简单，这个相同的 activity 应该包含一个 [CATEGORY_LAUNCHER](https://developer.android.com/reference/android/content/Intent.html#CATEGORY_LAUNCHER) intent filter 以使得 Android Studio 在部署和调试时可以起动它以作为默认的 activity。
```
<application
    android:label="@string/app_name">
    <activity android:name=".HomeActivity">
        <!-- Launch activity as default from Android Studio -->
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>

        <!-- Launch activity automatically on boot -->
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.IOT_LAUNCHER"/>
            <category android:name="android.intent.category.DEFAULT"/>
        </intent-filter>
    </activity>
</application>
```

## 支持 Google 服务
Android Things 支持  [Google APIs for Android](https://developers.google.com/android/) 的一个子集。作为一个通用的规则，需要用户输入或保密认证的 API 对于应用程序不可用。下表分列了 Android Things 中 API 的支持情况。

| 支持的APIs        | 不可用的 APIs           | 
| ------------- |:-------------:| 
| [Cast](https://developers.google.com/cast/)      | [AdMob](https://firebase.google.com/docs/admob/) |
| [Drive](https://developers.google.com/drive/)      | [Android Pay](https://developers.google.com/android-pay/)      |
| [Firebase Analytics](https://firebase.google.com/docs/analytics/) | [Firebase App Indexing](https://firebase.google.com/docs/app-indexing/)|
| [Firebase Cloud Messaging (FCM)](https://firebase.google.com/docs/cloud-messaging/) | [Firebase Authentication](https://firebase.google.com/docs/auth/)|
| [Firebase Crash Reporting](https://firebase.google.com/docs/crash/) | [Firebase Dynamic Links](https://firebase.google.com/docs/dynamic-links/)|
| [Firebase Realtime Database](https://firebase.google.com/docs/database/) | [Firebase Invites](https://firebase.google.com/docs/invites/) |
| [Firebase Remote Config](https://firebase.google.com/docs/remote-config/) | [Firebase Notifications](https://firebase.google.com/docs/notifications/) |
| [Firebase Storage](https://firebase.google.com/docs/storage/) | [Maps](https://developers.google.com/maps/)|
| [Fit](https://developers.google.com/fit/) | [Play Games](https://developers.google.com/games/services/)|
| [Instance ID](https://developers.google.com/instance-id/) | [Search](https://developers.google.com/search/)|
| [Location](https://developers.google.com/awareness-location/) | [Sign-In](https://developers.google.com/identity/) |
| [Nearby](https://developers.google.com/nearby/) |  |
| [Places](https://developers.google.com/places/) |  |
| [Mobile Vision](https://developers.google.com/vision/) |  |

## 权限
[运行时请求权限](https://developer.android.com/training/permissions/requesting.html) 不再得到支持，因为嵌入式设备不保证有一个 UI 来接受运行时对话框。在你的应用程序的清单文件中 [声明权限](https://developer.android.com/training/permissions/declaring.html) 你需要的。所有在你应用程序的清单文件中声明的 normal 和 dangerous 权限将在安装时获取。

## 通知
由于在 Android Things 中没有系统范围的状态栏和窗口 shade，[通知](https://developer.android.com/guide/topics/ui/notifiers/notifications.html) 不再得到支持。避免在你的应用程序中调用 [NotificationManager](https://developer.android.com/reference/android/app/NotificationManager.html)。

#总结
考虑到IoT的大多数应用场景，如设备的性能如 CPU 和内存没有手机和平板的那么足，通常没有显示设备，通常都是一个设备单个应用，通常无需与用户进行交互等，同时常常需要操作各种各样不同的外围设备，Android Things 经过在手机和平板上使用的核心 Android 裁剪和增强而来，并保留大部分核心 Android 的东西。开发工具，用于Java 应用程序开发的框架中的大部分得到保留。系统 Apps 及与用户交互有关的许多组件被裁剪。而增强了在应用程序中直接操作外围硬件设备的能力。

大部分内容译自 [官方文档](https://developer.android.com/things/sdk/index.html#behavior_changes)。

# 参考文档
[谷歌物联网Android Things到底是什么？这10点让你搞懂](http://digi.china.com/digi/20161219/201612199840.html)

[树莓派+Android Things](http://www.cnblogs.com/joelan/p/6269350.html)

[Android Things给物联网设备带来基于TensorFlow的机器学习和计算机视觉](http://www.infoq.com/cn/news/2017/02/android-things-dev-preview-2)

[我与Android Things的24小时](http://chuansong.me/n/1506196551221)