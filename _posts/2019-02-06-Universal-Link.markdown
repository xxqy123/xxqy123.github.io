---
layout: post
title: iOS Universal Links（通用链接） 适配小结
date: 2016-12-26 18:46:24.000000000 +08:00
---

# 简介
Universal Links 是苹果在2015年WWDC上推出的一项新功能，官方功能描述如下：
> In iOS 9 and later, universal links let users open your app when they tap links to your website within WKWebView and UIWebView views and Safari pages, in addition to links that result in a call to openURL:, such as those that occur in Mail, Messages, and other apps.

简单来说即当用户在 WKWebView、UIWebView 或者 Safari 中点击一个链接，如果设备上安装了适配该链接的 app，就可以跳转该 app 对应的页面，否则仍然展示网页 。由于目前微信内置浏览器仍然不支持 openURL 的方式进行应用间的跳转，不少 app 产品通过接入 Universal Links 实现微信浏览器一键跳转到自己app的功能。

# 效果
在“取暖” app 中接入 Universal Links ，微信中打开分享的话题，点击底部“打开App”按钮，直接跳转到“取暖”对应的话题详情页面：
![微信一键跳转到app](http://upload-images.jianshu.io/upload_images/716949-b7a4a47e39daf28a.gif?imageMogr2/auto-orient/strip)

# 流程
###### 1.创建一个包含 JSON 数据的 apple-app-site-association 文件，内容格式如下：

![apple-app-site-association 文件内容](http://upload-images.jianshu.io/upload_images/716949-c15209a21e60754e.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

• appID：app Prefix + Bundle ID, 其中 app Prefix 可以从苹果开发账号页面"Certificates, Identifiers & Profiles" - "Identifiers" - "App IDs" 下找到对应 App IDs 查看，Bundle ID 可以在项目的 target -- General 中找到；

• paths ：设定你的 app 支持的路径列表,只有这些指定的路径的链接,才能被 app 所处理。其中 "*" 表示任意路径（可接于指定路径后面，表示该指定路径下的所有子路径），"?" 表示替换路径中的一个字符。

> Use * to specify your entire website

> Include a specific URL, such as /wwdc/news/, to specify a particular link

> Append * to a specific URL, such as /videos/wwdc/2015/*, to specify a section of your website

> In addition to using * to match any substring, you can also use ? to match any single character. You can combine both wildcards in a single path, such as /foo/*/bar/201?/mypage.

注意点：
1> 正常情况下app Prefix 和开发者账号页面"Membership" - "Membership information" 中的Team ID是相同的，但是对于一些年代比较久远的app，这两个值可能会不同，此时应该选择app Prefix；

![app Prefix](http://upload-images.jianshu.io/upload_images/716949-a12f98f72ea4b5ae.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2> paths 大小写敏感；

3> "?" 表示替换一个字符，但是类似"#"这种特殊字符不可以被替换，例如paths中填写 "/foo/*/bar/201?/mypage"，如果实际域名中路径为 "/foo/abc/bar/201#/mypage"，则无法实现跳转；

4> apple-app-site-association 文件的内容是JSON格式，其本身并没有任何后缀名；

###### 2.将 apple-app-site-association 文件上传到服务器的 ".well-known" 或者根目录下（亲测两个目录均能生效）。

###### 3. 在 Xcode - TARGETS - Capabilities 中打开 "Associated Domains" 开关，在Domains中添加需要进行跳转的域名（所有适配的域名前面都需要加上 "applinks:" 才能生效）；

![配置域名](http://upload-images.jianshu.io/upload_images/716949-173a542df7935519.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完成步骤2和3，可通过苹果官方提供的网址进行验证是否生效，如果域名配置成功、apple-app-site-association 文件上传成功且格式正确，则 "Link to Application" 状态为 "PASSED".

![配置成功1](http://upload-images.jianshu.io/upload_images/716949-1755979ce9d20f91.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际适配过程中，会出现文件正常上传，浏览器中访问地址也可以正常下载 apple-app-site-association 文件，但是通过上述网站进行验证显示状态不通过的情况。这时也可以通过在备忘录中添加记事本或短信中输入App能识别的链接，如果直接点击此链接能直接跳转到你的app，或是长按链接文字,在出现的弹出菜单中第二项是“在'XXX'中打开”，也代表配置成功。

![配置成功2](http://upload-images.jianshu.io/upload_images/716949-ff4d018f2121c4c3.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 4. 在项目的 AppDelegate 里实现回调方法

```
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray *))restorationHandler
{
    if ([userActivity.activityType   isEqualToString:NSUserActivityTypeBrowsingWeb]) {

      NSURL *webpageURL = userActivity.webpageURL;
      NSString *host = webpageURL.host;

      if ([host isEqualToString:@"xxxx.com"]) { // "xxxx.com"为步骤3中配置的域名
        // 解析路径、跳转到指定界面 and so on...
    }
    else {
      [[UIApplication sharedApplication]openURL:webpageURL];
    }
  }
  return YES;
}
```

当 userActivity 类型为 NSUserActivityTypeBrowsingWeb , 表明它是由Universal Links捕获进来，即可以添加解析路径、跳转到指定界面等具体操作。

# 补充
需要特别注意一点：必须跨域才能支持 Universal Links ！

>When a user is browsing your website in Safari and they tap a universal link to a URL in the same domain as the current webpage, iOS respects the user’s most likely intent and opens the link in Safari. If the user taps a universal link to a URL in a different domain, iOS opens the link in your app.

举例假设当前微信浏览器中话题详情页面的 URL 为 "www.baidu.com/XXX/YYY"，底部“打开App”按钮对应的链接URL为 "www.baidu.com/AAA/BBB"，虽然通过上述4步配置了 "www.baidu.com/AAA/BBB" 路径，但由于两者都在百度的域名下，因此实际不能完成跳转。

通过以上4步及跨域支持，即可实现一键从微信跳转到app的功能，但是细心的读者可能会发现，从微信跳转到app的时候，屏幕右上角还有个 "warmup.cc" 的小箭头：

![右上角跳转箭头](http://upload-images.jianshu.io/upload_images/716949-5fdb94194ba43dc4.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击该箭头，会在Safari中打开该页面。此时再回到微信浏览器中点击 "打开App"按钮，神奇的事情出现了：无法跳转到app！！！

解决方案：在Safari中打开该页面，将网页拉倒最顶部，会出现一个悬浮框，点击悬浮框中的打开按钮，又跳回到app中打开指定页面，此时再回到微信浏览器中点击 "打开App"按钮，又能正常跳转到app了。

![Safari中的横条](http://upload-images.jianshu.io/upload_images/716949-112d6b0ee9dca183.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析：Universal Links 功能的接入其实相当于给某些 URL 添加了一种新的打开方式，但是旧的通过浏览器打开 URL 的方式仍然可用，当点击右上角跳转箭头时，相当于又设置这些特定 URL 的默认打开方式为浏览器而非 web，因此一键跳转功能此时会失效。反之通过点击顶部 "打开" 按钮，相当于又将这些特定 URL 的默认打开方式修改为 app ，一键跳转功能恢复正常。

ps：在Safari中打开页面刚进入时，横条是隐藏的，一定要将页面拉到最顶部时才能显示。之前曾考虑通过修改页面布局让其默认显示，但是由于其并不是网页内部元素，以失败告终 T^T .

