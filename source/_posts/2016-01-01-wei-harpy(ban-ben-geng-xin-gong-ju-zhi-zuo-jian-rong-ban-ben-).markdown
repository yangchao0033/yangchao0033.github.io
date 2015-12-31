---
layout: post
title: "为Harpy（版本更新工具)制做兼容版本"
date: 2016-01-01 03:58:09 +0800
comments: true
categories: ios
keyword: 版本适配,框架开发,APP版本升级
description: 使用现有的开源框架只做iOS6的兼容版本

---

**中文版：**

# Harpy（兼容版）

###(iOS5-9适配版本,基于[ArtSabintsev/Harpy v3.4.5](https://github.com/ArtSabintsev/Harpy))

### 提醒用户你的应用有新的可用版本，并且及时的跳转到App Store进行更新。

## 关于
**Harpy** 将用户手机上已安装的iOS app版本与当前App Store最新可用版本进行检查对比。如果有新的可用版本时，使用弹窗及时提醒用户最新版本信息，并然用户选择是否需要进一步操作。

<!--more-->

Harry是基于[http://www.semver.org](Semantic Versioning)版本号系统标准执行。
- `Semantic Versioning`是一个三位数的版本号系统（例如:1.0.0）
- Harry同样支持2位数的版本号(例如:1.0)
- Harpy同时支持4位数的版本号（例如:1.0.0.0）

## Swift 支持
当前兼容版本（iOS5-9）暂时不支持swift

## 特点
<!--- [x] CocoaPods Support-->
- [x] 支持三种类型的弹框样式 (详见 **截图 & Alert Types**)
- [x] 提供可选的代理方法 (详见 **Optional Delegate** section)
- [x] 本地化支持超过20+语言

## 屏幕截图

- **左图：**强制用户更新app
- **中图：**提供可选项是否前往更新
- **右图：**提供跳过当前版本更新的选项
- 这些样式全部可以通过`HarpyAletType`枚举进行控制，详见`Harpy.h`

![Forced Update](https://github.com/yangchao0033/Harpy/blob/master/samplePictures/4.pic.jpg?raw=true "Forced Update")
![Optional Update](https://github.com/yangchao0033/Harpy/blob/master/samplePictures/5.pic.jpg?raw=true "Optional Update")
![Skipped Update](https://github.com/yangchao0033/Harpy/blob/master/samplePictures/3.pic.jpg?raw=true "Optional Update")

## 安装

### 手动安装（正在准备CocoaPods）

将‘Harpy’文件夹拖入到你的项目中，并选择'copy if needed',包括 `Harpy.h` 和 `Harpy.m` 文件

## 配置
1. import **Harpy.h** 导入到 AppDelefate 类中 或者 Pre-Complier Header(.pch)文件中
2. 在你的`Appdelegate`中设置**appID**（必要），设置你的**alertType**（可选）
3. 在你的`Appdelegate`中调用`checkVersion`方法，三个检测方法调用位置分别位于Appdelegate的启动的代理方法中，可以自行选择使用
    - 在 `application:didFinishLaunchingWithOptions:` 中调用 `checkVersion`
    - 在 `applicationDidBecomeActive:` 中调用 `checkVersionDaily`  
    - 在 `applicationDidBecomeActive:` 中调用 `checkVersionWeekly` .
    
``` obj-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{

	// 启用Harpy之前确保你的window可用
	[self.window makeKeyAndVisible];

	// 为你的应用设置app id
	[[Harpy sharedInstance] setAppID:@"<#app_id#>"];

	// 设置 UIAlertController 将要基于哪个控制器显示 （适配iOS8+）
	[[Harpy sharedInstance] setPresentingViewController:_window.rootViewController];

  // (可选)设置代理来追踪用户点击事件，活着的使用自定义的界面来展示你的信息
      [[Harpy sharedInstance] setDelegate:self];
	
	// (可选) 设置alertController的tincolor（iOS8+可用）
	[[Harpy sharedInstance] setAlertControllerTintColor:@"<#alert_controller_tint_color#>"];

	// (可选) 设置你的应用名
	[[Harpy sharedInstance] setAppName:@"<#app_name#>"];

	 /* （可选）设置弹框类型 默认为HarpyAlertTypeOption */
	[[Harpy sharedInstance] setAlertType:<#alert_type#>];

	 /* (可选)如果你的应用只在某些国家或地区可用，你必须使用两个字符的country code来设置应用的可用区域 */
	[[Harpy sharedInstance] setCountryCode:@"<#country_code#>"];

	/* (可选) 强制指定应用显示语言, 请使用 Harpy.h 中定义的 HarpyLanguage 进行设置。*/
	[[Harpy sharedInstance] setForceLanguageLocalization:<#HarpyLanguageConstant#>];

	// 执行版本检测
	[[Harpy sharedInstance] checkVersion];
}

- (void)applicationDidBecomeActive:(UIApplication *)application
{

	/*
		执行每天检测你的app是否需要更新版本，需要在`applicationDidBecomeActive:`执行最合适
		因为这对于的你的应用进如后台很长时间后非常有用。
		
		同时，也会在应用第一次启动时执行版本检测
 	*/
	[[Harpy sharedInstance] checkVersionDaily];

	/*
		执行每周检测你的app新版本。同理需要将此代码放置在`applicationDidBecomeActive:`中执行。

		同时，也会在应用第一次启动时执行版本检测
	 */
	[[Harpy sharedInstance] checkVersionWeekly];

}

- (void)applicationWillEnterForeground:(UIApplication *)application
{
	/*
	 执行app新版本检测，放在此是为了让用户从App Sore跳转回来并重新从后台进入你的
	 app，并且没有在从App Store中跳转回来之前更新他们app的时候调用
	 
	 注意：只有当你使用*HarpyAlertTypeForce*样式弹框类型是才使用这种方法

	并且会在你第一次启动应用时检测。
 	*/
	[[Harpy sharedInstance] checkVersion];
}

```  
至此设置全部完成！
  
## 为不同的升级类型设置弹窗样式
如果你喜欢为不同的升级类型比如修改(revision)，补丁(patch)，轻微改动(minor)，重大修改(major)等升级类型仅仅添加下面的几行可选代码即可，添加位置必须在调用版本检查的方法（checkVersion）之前

``` obj-c
	/* 默认情况下Harpy会设置所有的版本升级样式为HarpyAlertTypeOption */
	[[Harpy sharedInstance] setPatchUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setMinorUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setMajorUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setRevisionUpdateAlertType:<#alert_type#>];
```

## 可选的代理和代理方法
如果你想要个处理或者追踪终端用户的的行为，Harpy会为你提供四个代理方法进行监控

```	obj-c
	// 用户界面展示升级提示对话框
	- (void)harpyDidShowUpdateDialog;

	// 用户已经点击升级按钮并且进入到App Sotore
	- (void)harpyUserDidLaunchAppStore;

	// 用户已经点击跳过此次版本更新
	- (void)harpyUserDidSkipVersion;

	// 用户已经点击取消更行对话框
	- (void)harpyUserDidCancel;
```

If you would like to use your own UI, please use the following delegate method to obtain the localized update message if a new version is available:
如果你想使用自己的UI，如果有可用的新版本，使用下面的代理来获得本地化的升级信息（需要设置AlertTpye为HarpyAlertTypeNone）

``` obj-c
- (void)harpyDidDetectNewVersionWithoutAlert:(NSString *)message;
```
## 强制本地化
Harpy 已经本地化了的语言包括 Arabic, Basque, 简体中文, 繁体中文, Danish, Dutch, English, Estonian, French, German, Hebrew, Hungarian, Italian, Japanese, Korean, Latvian, Lithuanian, Malay, Polish, Portuguese (Brazil), Portuguese (Portugal), Russian, Slovenian, Swedish, Spanish, Thai, and Turkish.

你可能想要你的升级对话框*永远*显示正确的语言，而忽略iOS的语言设置（比如在指定国家发行的app）

你可以使用以下代码实现强制本地化

``` obj-c
[[Harpy sharedInstance] setForceLanguageLocalization<#HarpyLanguageConstant#>];
```

## 在App Store上提交的重要注意事项
App Store 审核人员将不会看到升级弹框

**English**：

# Harpy（Compatible version Base On [ArtSabintsev/Harpy v3.4.5](https://github.com/ArtSabintsev/Harpy)）
### Notify users when a new version of your app is available, and prompt them with the App Store link.

---
## About
**Harpy** checks a user's currently installed version of your iOS app against the version that is currently available in the App Store. If a new version is available, an alert can be presented to the user informing them of the newer version, and giving them the option to update the application.

Harpy is built to work with the [http://www.semver.org](Semantic Versioning) system.
- Semantic Versioning is a three number versioning system (e.g., 1.0.0)
- Harpy also supports two-number versioning (e.g., 1.0)
- Harpy also supports four-number versioning (e.g., 1.0.0.0)

## Swift Support
- not support yet

## Features
<!--- [x] CocoaPods Support-->
- [x] Three types of alerts (see **Screenshots & Alert Types**)
- [x] Optional delegate methods (see **Optional Delegate** section)
- [x] Localized for 20+ languages

## Screenshots

- The **left picture** forces the user to update the app.
- The **center picture** gives the user the option to update the app.
- The **right picture** gives the user the option to skip the current update.
- These options are controlled by the `HarpyAlertType` typede that is found in `Harpy.h`.

![Forced Update](https://github.com/ArtSabintsev/Harpy/blob/master/samplePictures/picForcedUpdate.png?raw=true "Forced Update")
![Optional Update](https://github.com/ArtSabintsev/Harpy/blob/master/samplePictures/picOptionalUpdate.png?raw=true "Optional Update")
![Skipped Update](https://github.com/ArtSabintsev/Harpy/blob/master/samplePictures/picSkippedUpdate.png?raw=true "Optional Update")

## Installation Instructions

<!--### CocoaPods Installation
```
pod 'Harpy'
```-->

### Manual Installation

Copy the 'Harpy' folder into your Xcode project. It contains the Harpy.h and Harpy.m files.

## Setup
1. Import **Harpy.h** into your AppDelegate or Pre-Compiler Header (.pch)
1. In your `AppDelegate`, set the **appID**, and optionally, you can set the **alertType**.
1. In your `AppDelegate`, call **only one** of the `checkVersion` methods, as all three perform a check on your application's first launch. Use either:
    - `checkVersion` in `application:didFinishLaunchingWithOptions:`
    - `checkVersionDaily` in `applicationDidBecomeActive:`.
    - `checkVersionWeekly` in `applicationDidBecomeActive:`.


``` obj-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{

	// Present Window before calling Harpy
	[self.window makeKeyAndVisible];

	// Set the App ID for your app
	[[Harpy sharedInstance] setAppID:@"<#app_id#>"];

	// Set the UIViewController that will present an instance of UIAlertController
	[[Harpy sharedInstance] setPresentingViewController:_window.rootViewController];

  // (Optional) Set the Delegate to track what a user clicked on, or to use a custom UI to present your message.
      [[Harpy sharedInstance] setDelegate:self];

	// (Optional) The tintColor for the alertController
	[[Harpy sharedInstance] setAlertControllerTintColor:@"<#alert_controller_tint_color#>"];

	// (Optional) Set the App Name for your app
	[[Harpy sharedInstance] setAppName:@"<#app_name#>"];

	/* (Optional) Set the Alert Type for your app
	 By default, Harpy is configured to use HarpyAlertTypeOption */
	[[Harpy sharedInstance] setAlertType:<#alert_type#>];

	/* (Optional) If your application is not available in the U.S. App Store, you must specify the two-letter
	 country code for the region in which your applicaiton is available. */
	[[Harpy sharedInstance] setCountryCode:@"<#country_code#>"];

	/* (Optional) Overrides system language to predefined language.
	 Please use the HarpyLanguage constants defined in Harpy.h. */
	[[Harpy sharedInstance] setForceLanguageLocalization:<#HarpyLanguageConstant#>];

	// Perform check for new version of your app
	[[Harpy sharedInstance] checkVersion];
}

- (void)applicationDidBecomeActive:(UIApplication *)application
{

	/*
	 Perform daily check for new version of your app
	 Useful if user returns to you app from background after extended period of time
 	 Place in applicationDidBecomeActive:

 	 Also, performs version check on first launch.
 	*/
	[[Harpy sharedInstance] checkVersionDaily];

	/*
	 Perform weekly check for new version of your app
	 Useful if you user returns to your app from background after extended period of time
	 Place in applicationDidBecomeActive:

	 Also, performs version check on first launch.
	 */
	[[Harpy sharedInstance] checkVersionWeekly];

}

- (void)applicationWillEnterForeground:(UIApplication *)application
{
	/*
	 Perform check for new version of your app
	 Useful if user returns to you app from background after being sent tot he App Store,
	 but doesn't update their app before coming back to your app.

 	 ONLY USE THIS IF YOU ARE USING *HarpyAlertTypeForce*

 	 Also, performs version check on first launch.
 	*/
	[[Harpy sharedInstance] checkVersion];
}

```

And you're all set!

## Differentiated Alerts for Patch, Minor, and Major Updates
If you would like to set a different type of alert for revision, patch, minor, and/or major updates, simply add one or all of the following *optional* lines to your setup *before* calling any of the `checkVersion` methods:

``` obj-c
	/* By default, Harpy is configured to use HarpyAlertTypeOption for all version updates */
	[[Harpy sharedInstance] setPatchUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setMinorUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setMajorUpdateAlertType:<#alert_type#>];
	[[Harpy sharedInstance] setRevisionUpdateAlertType:<#alert_type#>];
```

## Optional Delegate and Delegate Methods
If you'd like to handle or track the end-user's behavior, four delegate methods have been made available to you:

```	obj-c
	// User presented with update dialog
	- (void)harpyDidShowUpdateDialog;

	// User did click on button that launched App Store.app
	- (void)harpyUserDidLaunchAppStore;

	// User did click on button that skips version update
	- (void)harpyUserDidSkipVersion;

	// User did click on button that cancels update dialog
	- (void)harpyUserDidCancel;
```

If you would like to use your own UI, please use the following delegate method to obtain the localized update message if a new version is available:

``` obj-c
- (void)harpyDidDetectNewVersionWithoutAlert:(NSString *)message;
```

## Force Localization
Harpy has localizations for Arabic, Basque, Chinese (Simplified), Chinese (Traditional), Danish, Dutch, English, Estonian, French, German, Hebrew, Hungarian, Italian, Japanese, Korean, Latvian, Lithuanian, Malay, Polish, Portuguese (Brazil), Portuguese (Portugal), Russian, Slovenian, Swedish, Spanish, Thai, and Turkish.

You may want the update dialog to *always* appear in a certain language, ignoring iOS's language setting (e.g. apps released in a specific country).

You can enable it like this:

``` obj-c
[[Harpy sharedInstance] setForceLanguageLocalization<#HarpyLanguageConstant#>];
```

## Important Note on App Store Submissions
The App Store reviewer will **not** see the alert.

## Created and maintained by
[Arthur Ariel Sabintsev](http://www.sabintsev.com/)
