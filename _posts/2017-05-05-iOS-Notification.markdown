---
layout: post
title: iOS 远程推送
date: 2017-05-05 11:03:24.000000000 +08:00
---

##基本原理
iOS推送分为Local Notifications（本地推送） 和 Remote Notifications（远程推送），这次主要阐述远程推送相关知识。

![远程推送大致流程](http://upload-images.jianshu.io/upload_images/716949-4e6d502949ac3608.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Provider就是我们自己程序的后台服务器，APNS是Apple Push Notification Service的缩写，也就是苹果的推送服务器。

上图可以分为三个阶段：
第一阶段：应用程序的服务器端把要发送的消息、目的iPhone的标识打包，发给APNS。
第二阶段：APNS在自身的已注册Push服务的iPhone列表中，查找有相应标识的iPhone，并把消息发送到iPhone。
第三阶段：iPhone把发来的消息传递给相应的应用程序，并且按照设定弹出Push通知。

下面这张图是说明APNS推送通知的详细工作流程：
![远程推送详细](http://upload-images.jianshu.io/upload_images/716949-e1a1d84e0bf44964.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据图片我们可以概括一下：
1、应用程序注册APNS消息推送。
2、iOS设备从APNS Server获取devicetoken，应用程序接收device token。
3、应用程序将device token发送给程序的PUSH服务端程序。
4、服务端程序向APNS服务发送消息。
5、APNS服务将消息发送给device token 对应的iOS设备上的应用程序。

当然，实现上述步骤需要一个前提：应用程序的推送证书（开发环境&生产环境两类推送证书）和描述文件（Provisioning Profile）配置完备。苹果会在推送前根据描述文件和 App ID 等信息对应用程序的合法性进行验证。

##代码配置
苹果在iOS 10 中引入 UserNotifications.framework 来集中管理和使用 iOS 系统中通知的功能（包含本地通知和远程通知）。在此基础上，Apple 还增加了撤回单条通知，更新已展示通知，中途修改通知内容，在通知中展示图片视频，自定义通知 UI 等一系列新功能，非常强大。远程推送功能，需要对 iOS 设备系统版本进行区分。

### - iOS 10 之前的系统

#####1.注册通知
![注册通知](http://upload-images.jianshu.io/upload_images/716949-744f666cb9b7c607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####2.注册通知成功后，获取device token
![获取device token](http://upload-images.jianshu.io/upload_images/716949-a9eecad56635a4c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####3.实现代理方法，处理远程通知
系统提供了两个通知回调方法
方法1：
![方法1](http://upload-images.jianshu.io/upload_images/716949-21d5d4ae2dbe3d5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法2：
![方法2](http://upload-images.jianshu.io/upload_images/716949-2d209ae404e8dc08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中方法2是2013年WWDC上随iOS7.0系统一齐推出的，因此方法2仅适用于iOS系统版本号>=7.0的设备。
在iOS6和之前，推送的类型是很单一的，无非就是显示标题内容，指定声音等。用户通过解锁进入你的应用后，appDelegate中通过推送打开应用的回调将被调用，然后你再获取数据，进行显示。这和没有后台获取时的打开应用后再获取数据刷新的问题是一样的。iOS7开始，苹果新增推送唤醒功能，我们有机会使设备在接收到远端推送后让系统唤醒设备和我们的后台应用，并先执行一段代码来准备数据和UI，然后再提示用户有推送。这时用户如果解锁设备进入应用后将不会再有任何加载过程，新的内容将直接得到呈现。打开Capabilities页面下的 Background Modes 选项，并勾选 remote notifications 选项即可开启推送唤醒功能。(或者修改info.plist文件)
![开启推送唤醒功能](http://upload-images.jianshu.io/upload_images/716949-07be76371920616e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，iOS7 -  iOS9 的系统中，关于两个方法的区别以及推送唤醒功能开启和关闭状态对两个回调方法的执行有何影响，在进行了一系列的测试后的得到如下数据。 

BackgroundModes - Remote notifications 关闭：

![关闭状态](http://upload-images.jianshu.io/upload_images/716949-164f7ed2e5abf96e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

BackgroundModes - Remote notifications 开启：

![开启状态](http://upload-images.jianshu.io/upload_images/716949-d9dbfc45a1c62890.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

特别说明
1.当app 在前台运行状态收到远程通知时，会立即执行回调方法1或方法2，但不在屏幕顶部弹出通知内容，也不会在通知中心显示通知、更改badgeNumber
2.当app在后台 且 background mode -remote notifications 开启时收到远程通知，会立即执行回调方法1或方法2，但不会在通知中心显示通知，也不会更改badgeNumber，仅在系统桌面顶部显示通知（如下图所示，且数秒后消失）
￼
![](http://upload-images.jianshu.io/upload_images/716949-208329463976c967.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时回调方法1或2 中 数据获取 & 界面更新 已经在app后台完成，如果点击app图标进入app，将展示已更新的数据 & 界面；如果点击顶部通知进入app，会再次执行回调方法1或2，重复 数据获取 & 界面更新。

通过上述测试数据，得到的结论是：
  **1.当app 在前台或者 后台状态时，方法1和方法2的执行时机及次数相同；**

  **2.方法1和方法2的区别在于：在app处于未启动状态时，点击远程通知会调用2，但不会调用1，因此如果只实现方法1，需要在applicationDidFinishedLaunched方法中根据launchOptions判断是否为点击通知启动app，是的话再手动调用方法1；在两个方法均实现的情况下，方法2会覆盖方法1，收到通知时将不会再触发方法1；**

  **3. background mode - remote notifications 开启状态，如果收到远程通知时app在后台，通知将不显示在系统的通知中心，应用程序的badgeNumber 也不作更改，仅在系统桌面顶部显示通知（数秒后消失），方法1或方法2都会立即调用一次（完成后台数据请求 or 页面更新），如果点击通知弹窗，都会再调用一次，此时有可能出现数据重复请求以及界面重复更新的问题；**

#####Q&A:
**1.方法1和2的如何选取？**
如果需要适配iOS7.0以前的版本，选择方法1，同时在applicationDidFinishedLaunched方法中根据launchOptions判断是否为点击通知启动app，是的话再手动调用方法1；
如果适配系统版本 >= 7.0 ， 建议选择方法2 。如果同时开启BackgroundModes - Remote notifications，需要防止 数据重复请求 & 界面重复更新 问题。

**2.如何防止 数据重复请求 & 界面重复更新？**
 1>通过判断application.applicationState，当app在后台时禁止其 请求数据 & 更新界面 （但是这样就失去了开启推送唤醒功能的意义 T^T）（http://stackoverflow.com/questions/20569201/remote-notification-method-called-twice）
![](http://upload-images.jianshu.io/upload_images/716949-f5089309061fb2eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2> 在通知内容中添加unique ID，在回调方法中过滤相同的unique ID对应通知回调的重复执行。（http://stackoverflow.com/questions/20615688/application-didreceiveremotenotification-fetchcompletionhandler-called-more-t）


### - iOS 10 系统
#####1.注册通知
![注册通知](http://upload-images.jianshu.io/upload_images/716949-b185902007f2698a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.注册通知成功后，获取device token (与iOS10之前相同)
![获取device token](http://upload-images.jianshu.io/upload_images/716949-b920ddc04e88f4c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.实现代理方法，处理远程通知
方法1：
![方法1](http://upload-images.jianshu.io/upload_images/716949-06e238d8bce3d9b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法2：
![方法2](http://upload-images.jianshu.io/upload_images/716949-489ed9e989fb2772.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法1和方法2可以同时实现，方法1当且仅当app在前台收到通知时触发，当用户点击通知时，会触发方法2。

**注意点：**
**1.需要在方法1和2中对本地通知和远程通知进行筛选，远程通知满足条件**
```[notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]```
**2.如果开启BackgroundModes - Remote notifications 且实现了**
```- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler```
**方法，收到远程通知时如果app 在前台 或 后台，仍然会调用该方法，此时如果用户再次点击通知，会调用一次方法2，可能出现重复请求数据 & 重复更新界面 的情况，需要作排重处理。**