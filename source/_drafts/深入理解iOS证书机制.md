---
toc: true
title: 深入理解iOS证书机制
date: 2020-02-26 17:02:52
categories:
tags:
---

## 前言

最近在做一些`Hybrid`开发时为了安全采用了本地的证书校验，发现对`iOS`的证书机制很不了解，借此机会深入学习一下。

## SSL

在进行网络请求或者是`webView`使用时会出现一个叫`didReceiveAuthenticationChallenge`的回调方法。这是当客户端收到服务端安全挑战的回调，客户端可以在此进行处理来满足服务端的安全检查。

产生次安全挑战的原因正式因为[SSL]()，它位于传输层和应用层协议之间，为数据通讯安全做保证。

```objective-c
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
                                             completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler;
```

## iOS的证书

### 安全挑战

在`FoundationKit`中，把安全挑战封装成了`NSURLAuthenticationChallenge`对象，封装了证书等安全信息。



