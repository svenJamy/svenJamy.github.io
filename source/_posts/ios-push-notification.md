---
title: iOS推送机制解析
date: 2018-02-07 22:32:10
tags: iOS
categories: iOS
---

APNs是远程通知功能的核心。它提供了一个强大、安全、高效率的方式让应用程序开发人员从服务端传播信息到iOS（和间接，watchOS），tvOS和macOS设备。

通过这篇文章你会知道iOS推送的整个流程，推送证书的配置及普通推送和silence推送的区别。如果有问题的地方，欢迎留言。<!--more-->

## 原理

首先我们看下苹果文档关于通知机制的介绍：
![](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_simple_2x.png)

在用户启动应用的时候，系统会自动在应用程序和APNs之间建立一个经过认证，加密和持续的IP连接。`provider`就是我们自己的服务端，也是和APNs建立了一个经过加密的安全连接，加密的配置在后面我们会讲到。

APNs的注册和使用流程主要是下面几个步骤：

1. 应用程序注册APNs远端推送通知;
2. iOS从APNS Server获取deviceToken，应用程序接收到device token;
3. 应用程序将device token发送给程序的PUSH服务端;
4. PUSH服务端向APNS服务器发送推送消息;
5. APNS服务将推送消息发送给iPhone对应的应用程序.


## 配置

### 两种鉴权方式的配置

#### 通过 .p12 证书鉴权

如果你之前没有创建过 Push 证书或者是要重新创建一个新的，请在证书列表下面新建。
![](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_1_addCert.png)

新建证书需要注意选择 APNs 证书种类。如图 APNs 证书有开发（Development）和生产（Production）两种。
    注：开发证书用于开发调试使用；生产证书既能用于开发调试，也可用于产品发布。此处我们选择生产证书为例。
![cert_type](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_2_certType.png)

点击 "Continue", 之后选择该证书准备绑定的 AppID。
![cert_to_app](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_3_certToApp.png)

点击 “Continue”，会进入 CSR 说明界面。
![need_CSR](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_4_needCSR.png)

再点 “Continue” 会让你上传 CSR 文件。（ CSR 文件会在下一步创建）
![update_CSR](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_5_uploadCSR.png)

打开系统自带的 KeychainAccess 创建 Certificate Signing Request。如下图操作：
![open_keychain](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_6_openKeychain.png)

填写“用户邮箱”和“常用名称” ，并选择“存储到磁盘”，证书文件后缀为 .certSigningRequest 。

回到浏览器中 CSR 上传页面，上传刚刚生成的后缀为 .certSigningRequest 的文件。
生成证书成功后，点击 “Download” 按钮把证书下载下来，是后缀为 .cer 的文件。
![cert_ready](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_8_certReady.png)

双击证书后，会在“KeychainAccess”中打开，选择左侧“钥匙串”列表中“登录”，以及“种类”列表中“我的证书”，找到刚才下载的证书，并导出为 .p12 文件, 这里可以设置密码，如123456，如下图：
![export_p12 save_p12](http://docs.jiguang.cn/jpush/client/image/ios_cert/p12_9_exportP12.png)


经过上面那段步骤，我们在桌面上一共生成了三个文件。一个是CSR请求文件，一个是aps_development.cer的SSL证书文件，还有一个刚才生成的Push.p12秘钥文件。一般后台服务器都是Windows系统的，不支持生成的.p12证书格式，这里我们需要转换为.pem证书：

1. .cer的SSL证书转换为.pem文件：

`openssl x509 -in aps_development.cer -inform der -out PushCer.pem`

2. 私钥Push.p12文件转化为.pem文件：

`openssl pkcs12 -nocerts -in Push.p12 -out PushKey.pem`

3. 把上面的两个.pem文件合成一个：

`cat PushCer.pem PushKey.pem > push_author.pem`

4. 测试一下：

`openssl s_client -connect gateway.sandbox.push.apple.com:2195 -cert PushCer.pem -key PushKey.pem`
说明一下：`gateway.sandbox.push.apple.com:2195`这个URL是我们自己的服务器推送到APNs的路径，不管是在开发环境还是正式环境中。

#### 基于 APNs signing Key鉴权
`APNs signing Key`可以用于多个APP的鉴权，所以我们只需要创建一次即可，并且在开发和生产环境均可使用，且不会过期。我们需要在`xcode`和[开发者中心](https://developer.apple.com/)做下面相关的配置：

1. 在开发者后台，进入[Certificates, Identifiers & Profiles](https://developer.apple.com/account/ios/certificate);
2. 点击左侧列表`Keys`中的`All`，看账户中是否已有 auth key，没有则点击 “+” 新创建一个。
3. 填写该 key 的描述并选择服务，如下图。
4. 点击`Continue`让你确认信息，再点击`confirm`，就可以下载该 key了。（注意：记下 key id，而且只可以下载一次，请妥善保存。）

![](http://help.apple.com/xcode/mac/current/en.lproj/Art/ca_apns_authentication_key.png)




### Entitlements

在`xcode`中通过设置这个文件来标识当前应用所具有的额外的能力，比如`Apple Pay`、`push notifications`、`iCloud`、`Siri`等等，具体介绍可以查看[apple文档](https://developer.apple.com/library/content/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/AboutEntitlements.html),默认这个能力都是关闭的，如果需要设置对应的功能，就重写对应的键值对即可。我们可以在`xcode`的`Entitlements`选项卡中设置对应的属性。比如现在我们要`APP`具有接受remote通知的能力：

![xcode setting](http://help.apple.com/xcode/mac/current/en.lproj/Art/ca_enablepushnotifications.png)

在`xcode`里面设置完成之后，然后在APP.entitlements文件里面把下面两个key填入：

Notification entitlement key  | Capability
------------- | -------------
aps-environment  | Receive push notifications in iOS
com.apple.developer.aps-environment  | Receive push notifications in macOS


## 通知注册及处理
在`UIApplicationDelegate`中苹果提供了下面几个和通知相关的方法，我们只需要按照对应的步骤处理即可：

1. 注册remote通知

```
// 在这里注册通知
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(nullable NSDictionary *)launchOptions {
	UIUserNotificationType type = (UIUserNotificationTypeBadge |
                                 UIUserNotificationTypeSound |
                                 UIUserNotificationTypeAlert);
   if ([application respondsToSelector:@selector(isRegisteredForRemoteNotifications)]) {
    UIUserNotificationSettings *mySettings = [UIUserNotificationSettings settingsForTypes:types categories:categories];
    [application registerUserNotificationSettings:mySettings];
    [application registerForRemoteNotifications];
  }
}


// `registerUserNotificationSettings:`成功之后会调用到这个方法
- (void)application:(UIApplication *)application
    didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings { }
    

// `registerForRemoteNotifications`会初始化注册APNs服务，成功之后会调用这个方法，返回的devicetoken需要保存起来上传到自己的服务器端
- (void)application:(UIApplication*)application
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    [myService registerWithToken:token];
    .....
 }
 
// 如果注册失败会调用下面的方法
- (void)application:(UIApplication*)application
    didFailToRegisterForRemoteNotificationsWithError:(NSError*)error


```

## Silent Push

iOS7以后苹果引入了silent push机制，当设备接受到该通知之后，用户是无感知的。但是应用可以在background接受到这个通知并且处理，比如下载一些新的配置等信息。一旦完成处理，必须在处理程序参数中调用completionBlock，否则应用程序将被终止。在接受到silent push通知之后，APP有30秒左右的时间去处理，完成之后调用对应的block。

为了实现Silent Push，需要设置下面几个地方：

1. 在info.plist在添加下面配置项：

```
	<key>UIBackgroundModes</key>
	<array>
		<string>remote-notification</string>
	</array>
```

2. 推送通知的时候，内容体必须包括下面字段：

```
aps {
	content_available: 1
}

```
系统收到push时app大概有下面3种情况：

1. app正在前台运行；
2. app正在后台运行或处于挂起状态，总之app还处于内存中；
3. app进程已经被kill。

当 app 处于前台运行时，收到 push，无需用户响应，系统直接调用`application:didReceiveRemoteNotification:fetchCompletionHandler:`

当 app 处于后台或挂起状态时，收到 push，需要用户响应，此时系统也会调用`application:didReceiveRemoteNotification:fetchCompletionHandler:`

那么，如何区分上述两种情况呢？
可以在`application:didReceiveRemoteNotification:fetchCompletionHandler:`中判断 app的状态，如果`applicationState == UIApplicationStateActive`, 则属于第一种情况，若`applicationState == UIApplicationStateInactive`, 则属于第二种情况。

## reference

[APNSOverview](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html)

[极光推送指南](http://docs.jiguang.cn/jpush/client/iOS/ios_cer_guide/)