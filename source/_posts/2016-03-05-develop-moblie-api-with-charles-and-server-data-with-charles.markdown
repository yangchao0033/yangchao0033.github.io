---
layout: post
title: "让 iOS/Android 网络API开发更加自由-charles模拟服务器返回本地数据"
date: 2016-03-05 14:52:25 +0800
comments: true
categories: ios
---

>**原因：**在与服务器联调接口之后，所有的API都能正常跑通，但是涉及到具体的业务逻辑时，比如界面遇到不同的数据做出特定的布局操作或弹框提示，再或者只有当数据字段符合特定的值时才能做出更多复杂的逻辑操作。如果依赖于服务器或者数据库，那么需要他们去帮你制造假数据来检测你的代码的正确性，这样是一次两次没问题，但是量大的时候，会影响双方的开发进度。能不能采取解耦的思想让移动端和服务端分隔开，在移动端测试开发阶段，自己根据接口文档制造自己需要的特定的返回数据呢？

**解决方案：**这时候我们需要自己的代理服务器来实现--使用 Charles 制作代理服务器。


**具体需求：**当API有所改动时，服务器正在开发相应功能，但是还没有部署到服务器上去，只是在原接口的返回数据中多加了一个字段“id”。现在移动端的业务逻辑写好了，就等数据测试了。

**具体操作步骤：**

<!--more-->

**一、**首先你需要下载一个 Charles，并且安装起来，具体的使用方法这里不做赘述。可以移步[这里](http://www.infoq.com/cn/articles/network-packet-analysis-tool-charles)学习。
**二、**抓取你需要修改的接口
![抓取一个接口.png](抓取一个接口.png)
大家可以看到，这个接口返回的是一个json串。这个请求是手机发起的，我们看到的只是软件抓到的包数据。那么如何加入新的“id”字段呢？)
大家可以看到，这个接口返回的是一个json串。这个请求是手机发起的，我们看到的只是软件抓到的包数据。那么如何加入新的“id”字段呢？

**三、**使用 Charles 的 local map 功能

![保存response数据.png](http://upload-images.jianshu.io/upload_images/1445110-6f4e156cec8977aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

保存 response 数据到本地电脑上去

![选择localmap.png](http://upload-images.jianshu.io/upload_images/1445110-aad32dcbfbdb5a95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择local map

![添加需要map的接口.png](http://upload-images.jianshu.io/upload_images/1445110-d52e6534b924049e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击 Add 添加 map 规则

![map规则.png](http://upload-images.jianshu.io/upload_images/1445110-f4812476a660fc69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这些地方填入你的API信息，以及刚才保存的 response 文件地址

![点击保存map.png](http://upload-images.jianshu.io/upload_images/1445110-d6ec588fb2029af4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击ok保存map规则。之后，只要是从你手机通过 Charles 代理发出的这个接口的请求，都会被重定向到你电脑上本地的 response 文件中。

现在，我们在通过手机调用一下之前的接口，结果呢？结果是还不如原来的，呵呵哒。。。

之前我们还能获取到 title、content和type三个字段以及他们的值，现在直接返回的数据为nil。

找了半天原因，最终通过对比发现了一个问题，两种请求的响应头不一样


![从服务器取到的数据.png](http://upload-images.jianshu.io/upload_images/1445110-4b4c32ac51e3ba6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![本地map的数据.png](http://upload-images.jianshu.io/upload_images/1445110-2a2a2d5e280c4eae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以怎么修改 response 的 content type 类型为 json/application 呢？

**三、**使用 Rewrite 替换特定接口的响应头参数


![打开rewrite.png](http://upload-images.jianshu.io/upload_images/1445110-447880c3c6009b8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开 Rewrite


![rewrite配置页面.png](http://upload-images.jianshu.io/upload_images/1445110-7434a6160598b193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Rewrite 配置页面简介

1、首先创建一个规则集


![添加接口集合和适用范围.png](http://upload-images.jianshu.io/upload_images/1445110-9d417c2fdd967f19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

添加 Rewrite 替换规则


![书写rewrite替换规则.png](http://upload-images.jianshu.io/upload_images/1445110-38946ceab5ba8be0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里相当于是在响应回来之前，通过字段的匹配，替换掉原有的响应头中的Content-Type 类型为 json/application 

所有的对话框都点击 ok 或者 apply 确定保存

现在我们开始测试啦，继续调用该接口。

四、根据需求修改数据

数据修改前刚才看过了，只有 title/content/type 三个字段

```json
{
"title":"标题",
"content":"内容正文",
"type":"类型"
}
```
而现在需要添加 “id” 字段。所以，用 vim 或者其他文本编辑器打开刚才保存的 response 文件，将他修改成这样

```json
{
"id":"00000001",
"title":"标题",
"content":"内容正文",
"type":"类型"
}
```

修改后记得保存一下，现在重新开始请求该接口，查看log日志。

```
{
"id":"00000001",
"title":"标题",
"content":"内容正文",
"type":"类型"
}
```

完美！下面开始愉快的测试我们自己的代码吧~

****
##总结：
本文的目的主要是为了提高客户端和服务器各自独立开发的能力，像制造特定的假数据这种很简单的需求，如果不借助 Charles 代理去自己制造，恐怕你需要自己搭建一个服务器去各种恶补服务器的知识了。其实我们的原理十分简单，就是通过代理软件对网络请求进行拦截，并且返回我们想要的数据。这样做的好处就是，大大加强了客户端的自主开发能力，不需要依赖服务器对我们的特定逻辑进行开发，缩短开发周期。节约大家的时间，试想如果我们自己制造的数据如果在app中能跑通，即便服务器开发出来，如果出现跑不通的错误，那问题最有可能出现在服务器端，当然还要看具体的反馈信息。Charles 还有很多强大的功能，有待大家慢慢探索，祝大家玩得开心~
---
