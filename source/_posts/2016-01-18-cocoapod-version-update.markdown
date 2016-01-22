---
layout: post
title: "CocoaPod version update"
date: 2016-01-18 15:58:44 +0800
comments: true
categories: ios 
keywords: CocoaPod version update"
description: CocoaPod 版本升级，解决无法搜索到最新github库版本
---

###问题描述：
使用 pod 安装第三方库时无法使用 `pod search` 搜索最新的版本

###定位问题：
使用的 pod 版本过低

###方案：
<!--more-->
升级pod版本
```
$ sudo gem update --system
$ gem sources --remove https://rubygems.org/ 
$ gem sources -a https://ruby.taobao.org/ 
$ sudo gem install cocoapods
$ pod setup
```
line 1: 为 `gem` 升级到最新版  
line 2: 删除 `gem` 默认的源  
line 3: 添加淘宝的源，为了解除天朝对gem的隔离（ps：`gem` 到底干了啥，这么苦大仇深？）  
line 4: 安装 `CocoaPod` (会去新加的源去检查是否有新的内容，有则更新，保持你本地pod拥有最新的库信息)  
line 5: `pod` 安装将利用刚才的配置更新本地镜像  


###使用：
继续使用 `pod search 库名`
例如： `pod search ReactiveCocoa`

就能搜到最新的库版本了
