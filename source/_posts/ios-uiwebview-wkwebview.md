---
title: UIWebView和WKWebView
date: 2018-01-14 15:45:55
categories:
- iOS
tags: iOS
---

本文主要介绍下UIWebView和WKWebView的基本用法tips及它们与JavaScript之间的调用，持续更新中。。。<!---more--->

## UIWebView

UIWebView是最早接触的控件之一了，使用方法也比较简单：

* 加载数据接口：

```
- (void)loadRequest:(NSURLRequest *)request;
- (void)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;
- (void)loadData:(NSData *)data MIMEType:(NSString *)MIMEType textEncodingName:(NSString *)textEncodingName baseURL:(NSURL *)baseURL;
```

* 如果需要监听webview的状态，判断是否加载对应的URL，或者获取加载的状态等，可以实现UIWebView的代理方法`<UIWebViewDelegate>`：

```
@protocol UIWebViewDelegate <NSObject>

@optional
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;
- (void)webViewDidStartLoad:(UIWebView *)webView;
- (void)webViewDidFinishLoad:(UIWebView *)webView;
- (void)webView:(UIWebView *)webView didFailLoadWithError:(nullable NSError *)error;

@end

```

设置全局的UA（user-Agent）

```
- (void)updateUserAgent {
    NSString *oldAgent = [self.WebView stringByEvaluatingJavaScriptFromString:@"navigator.userAgent"];
    NSString *newAgent = [oldAgent stringByAppendingString:@"new-ua"];
    
    NSDictionary *dic = [[NSDictionary alloc] initWithObjectsAndKeys:
                         newAgent, @"UserAgent", nil];
    
    [[NSUserDefaults standardUserDefaults] registerDefaults:dic];
}
```


* 取消长按webView上的链接弹出actionSheet的问题:

```
-(void)webViewDidFinishLoad:(UIWebView *)webView {
    [webView stringByEvaluatingJavaScriptFromString:@"document.documentElement.style.webkitTouchCallout = 'none';"];
}
```


## WKWebView

WKWebView是iOS 8以后新增加的，使用的时候需要引入WebKit框架，WKWebView有两个delegate,`WKUIDelegate` 和 `WKNavigationDelegate`。`WKNavigationDelegate`主要处理一些跳转、加载等处理操作，`WKUIDelegate`主要处理JS脚本，确认框，警告框等。WKWebView主要有以下几个优点：

* 支持更多的HTML5特性；
* 60fps的刷新率以及内置右滑等手势支持；
* Safari相同的JavaScript引擎；
* 将UIWebViewDelegate与UIWebView拆分成了14类与3个协议(官方文档说明)
* 增加加载进度属性：estimatedProgress；

load方法：

```
- (nullable WKNavigation *)loadRequest:(NSURLRequest *)request;

- (nullable WKNavigation *)loadFileURL:(NSURL *)URL allowingReadAccessToURL:(NSURL *)readAccessURL API_AVAILABLE(macosx(10.11), ios(9.0));

- (nullable WKNavigation *)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;

- (nullable WKNavigation *)loadData:(NSData *)data MIMEType:(NSString *)MIMEType characterEncodingName:(NSString *)characterEncodingName baseURL:(NSURL *)baseURL API_AVAILABLE(macosx(10.11), ios(9.0));

```
导航相关：

```
@property (nonatomic, readonly) BOOL canGoBack;

@property (nonatomic, readonly) BOOL canGoForward;

- (nullable WKNavigation *)goBack;

- (nullable WKNavigation *)goForward;

- (nullable WKNavigation *)reload;

- (nullable WKNavigation *)reloadFromOrigin;

- (void)stopLoading;
```


## iOSH5性能监控技术角度分析
分析移动端H5性能数据，首先我们说说是哪些性能数据。

* 白屏时间；
* 页面耗时，页面耗时指的是开始加载这个网页到整个页面渲染完成的时间；
* 加载链接的一些性能数据，重定向时间，DNS解析时间，TCP链接时间，request请求时间，response响应时间，dom节点解析时间，page渲染时间等等。


## JS交互

### 内建方法
UIWebView和WKWebView都提供了调用JS方法的支持，但是UIWebView只提供了调用JS的方法，不能让JS直接调用OC的方法，如果只是简单的单向调用，可以使用对应的方法：

```
// UIWebView
- (nullable NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;

// WKWebView completionHandler是异步回调block,和UIWebView不同
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id, NSError * _Nullable error))completionHandler;
```

WKWebview提供了API（WKUserContentController）实现和js的通信，不需要借助JavaScriptCore或者webJavaScriptBridge，简单的说就是通过`WKUserContentController`先注册约定好的方法，然后再调用。

### JavaScriptCore

UIWebView默认没有提供直接获取到当前JS环境的context，但是我们可以通过调用对应的JS方法来获取到：

```
-(void)webViewDidFinishLoad:(UIWebView *)webView {
   self.jsContext = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    ｝
```
然后我们就可以通过获取到的JSContext调用JS的方法：

```
// 直接调用
[self.jsContext evaluateScript:@"alert("hello world!")"];
// 需要传递参数
JSValue *JSfunc = self.jsContext[@"login"];
[JSfunc callWithArguments:@[@'name',@'pw']];
```
另外也可以向JSContext注册一个bridge，然后就在JS中通过bridge调用OC提供的方法：

```
#include <JavaScriptCore/JavaScriptCore.h>

@protocol JSBridgeExport <JSExport>
- (void)toast:(NSString *)message;
@end

@interface JSBridgeDemo : NSObject <JSBridgeExport>
@end
@implementation JSBridgeDemo
- (void)toast:(NSString *)message {
    [UIAlertView alert:message];
}
....
@end

self.jsContext[@"lg_bridge"] = [[JSBridgeDemo new];
[self.jsContext evaluateScript:@"lg_bridge. toast(\"test:message send to OC\")"];

```
这样使用一些基本的功能是没有问题的，但是比如我们需要在webview加载好之前来和JS通信，这样是获取不到当前的JSContext。而且每个页面重新加载好后都要重新设置一下对应的注册方法，效率不高。所以这种方法很少使用。

而且`WKWebView`只能使用`WKScriptMessageHandler`来做JS交互
，不能通过JSContext，在适配方面可能也会有问题。

### URL拦截

著名的[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)就是通过这种方式实现的，有兴趣的同学可以去看下源码。

