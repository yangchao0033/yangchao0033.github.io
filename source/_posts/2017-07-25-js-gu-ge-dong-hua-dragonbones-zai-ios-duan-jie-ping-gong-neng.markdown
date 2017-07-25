---
layout: post
title: "JavaScript 骨骼动画 (DragonBones) 在 iOS 端截屏功能"
date: 2017-07-25 23:30:10 +0800
comments: true
categories: ios 
keywords: DragonBones, JavaScript, 骨骼动画, iOS 截屏功能 
Description: 记录在使用 JavaScript 骨骼动画时遇到截屏分享保存问题
---

### 需求背景：
使用 DragonBones 在 Webview 中绘制骨骼动画，并对当前的骨骼页面进行图层截取，实现保存本地和分享功能。

### 问题：
使用 DragonBones 制作骨骼动画时遇到一个问题，使用 WKWebView 加载骨骼动画正常，再对 WKWebView 的父容器 view （控制器 view ）进行截图是出现空白。

#### 解决思路及方案一：

<!--more-->

通过资料查阅，发现 WKWebView 在截图时比较特殊,需要调用 UIView 的分类，也就是 
```objc
- (nullable UIView *)snapshotViewAfterScreenUpdates:(BOOL)afterUpdates NS_AVAILABLE_IOS(7_0);
```
进行View图层的快照获取，推测和 WKWebView 的渲染原理相关。因为正常流程都是通过 UIGraphics 去捕捉 UIView 的上下文来生成 UIImage 对象，进而分享或保存到本地。
```objc
+ (UIImage *)getSharingImageWithView:(UIView *)originView {
    UIImage *snapshotImage = nil;
    UIGraphicsBeginImageContextWithOptions(originView.bounds.size, NO, [UIScreen mainScreen].scale);
    [originView.layer renderInContext:UIGraphicsGetCurrentContext()];
    snapshotImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return snapshotImage;
}
```
对 WkWebView、TableView 等滚动视图循环截屏可参考 [TYSnapshotScroll](https://github.com/TonyReet/TYSnapshotScroll)。
当然，例如 TYSnapshotScroll 提供的 WKWebView 截图方式（snapshotViewAfterScreenUpdates）截取方式对类似 github、google 等首页的截取非常完美。

但是，在加载骨骼动画时只有动画背景，没有任务形象。

#### 解决思路及方案二：

根据从安卓和前端收集的信息来看:
1. h5 骨骼动画底层使用了 canvas 绘图(一个类似于 iOS CoreGraphic 的库，专门负责绘制复杂图形。)
2. JavaScript 使用 canvas 绘图时，会存在这样的关系，直接上图


![canvas 的七大姑八大姨](http://upload-images.jianshu.io/upload_images/1445110-6f6cd7e375e337f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



3. 上图中最有价值的线索是，canvas 会调用 webGL 进行渲染，同时会调用硬件加速功能
4. 安卓端在手动关闭硬件加速后，该页面截屏恢复正常。

然鹅，只有 Safari 中有对 WebGL 有控制权，WKWebView 并不能通过代码关闭硬件加速。
所以只能放弃使用 canvas 动态绘制的 webView 进行图像提取。

所以，我们采取了采取了双端结合的思路：
* 由 JavaScript 通过 canvas 输出 base64 图片字符串，以及人物背景图片的URL
JavaScript生成图片：
```js
var fullQuality = canvas.toDataURL('image/jpeg', 1.0); // 参数二控制图片质量
// data:image/jpeg;base64,/9j/4AAQSkZJRgABAQ...9oADAMBAAIRAxEAPwD/AD/6AP/Z"
var mediumQuality = canvas.toDataURL('image/jpeg', 0.5);
var lowQuality = canvas.toDataURL('image/jpeg', 0.1);
```
Native 解析图片：
```objc
NSData *decodedImageData = [NSData dataWithContentsOfURL:[NSURL URLWithString:base64String]];
UIImage *decodedImage = [UIImage imageWithData:decodedImageData
                                         scale:[UIScreen mainScreen].scale];
personImageView.image = decodedImage;
```
* 由 Native 对使用这些资源进行布局渲染，并且控制截屏时用户信息、二维码等元素的展示，效果如图：

![骨骼动画截屏](http://upload-images.jianshu.io/upload_images/1445110-9d3101208a5f022e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 总结
对于部分特殊控件的图像处理可能要会出现位置的异常，需要对对应的底层实现技术点进行剖析，倒推引起的原因。

#### 参考：
[HTML5 Canvas,WebGL,CSS Shaders,GLSL的暧昧关系](http://www.zhangxinxu.com/wordpress/2011/10/html5-canvas-webgl-css-shaders-glsl%E7%9A%84%E6%9A%A7%E6%98%A7%E5%85%B3%E7%B3%BB/)
[egret](https://www.egret.com/)
[dragonBone](http://dragonbones.com/cn/index.html)
