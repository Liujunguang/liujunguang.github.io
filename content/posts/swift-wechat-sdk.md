+++
title = 'Swift Wechat Sdk'
date = 2024-01-14T05:15:52+08:00
draft = true
+++

## 关于 Universal Links

1、我使用类似 https://www.xxx.com/ ，这样手机 Safari 打开首页，就能在顶部看到跳转

2、`apple-app-site-association` 文件的配置，越简单越好，参考这个[链接](https://blog.csdn.net/peng_up/article/details/103894818)

3、微信开放平台“开发信息”的内容修改，不需要从新提交审核，所以审核通过后也可以修改

 
## 关于微信跳转闪回、登录失败、报错

1、记得在 AppDelegate.swift 中开启日志

```swift
// 记得在注册微信前开启日志
WXApi.startLog(by: .detail, logDelegate: self)
WXApi.registerApp(WXAppID, universalLink: WX_UniversalLinks)
...
...
 
func onLog(_ log: String, logLevel level: WXLogLevel) {
    print("wechat-log: ",log)
}
```

2、报错：Error:fail to load Keychain status:-25300, keyData null:1

App跳转微信，又立即闪回App

微信回调没有进入

获取用户信息失败

以上在我这都是因为没有按标准重写 `continueUserActivity` 导致的

```swift
func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey: Any] = [:]) -> Bool {
    WXApi.handleOpen(url, delegate: self)
    return true
}
    
func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
    WXApi.handleOpen(url, delegate: self)
    return true
}
    
func application(_ application: UIApplication, continue userActivity:
    NSUserActivity, restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    return WXApi.handleOpenUniversalLink(userActivity, delegate: self)
}
```

注意：`continueUserActivity` 方法不能写成 private，参考：https://stackoverflow.com/questions/41191472/ios-universal-links-swift-continueuseractivity-not-calling

我就是卡在这个问题上，一直授权登录，获取不到用户信息

查资料的时候看到很多人遇到类似的问题，请注意continue的重新实现方式，像我上面这样应该就没问题。
