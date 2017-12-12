---
layout: post
title:  iOS Development Tips(2)
---

继上一篇的《iOS Development Tips(1)》 , 继续整理一些日常开发中的小 Tips。

## 1.UIView 设置背景图片的方法。

 UIView 作为所有控件的基类很诡异的是没有背景图属性，只有 BackgroundColor 属性。于是可以通过
> self.view.backgroundColor = [UIColor colorWithPatternImage: [UIImage imageNamed:@"1.png"] ];

的方法进行背景图的设置。这样做的问题在于：有 alpha 通道的图片在转换后会丧失其 alpha 值 (iOS5.0 以前的系统有这问题)。解决的方法经过两次设置：
> view.backgroundColor = [UIColor clearColor];
view.layer.backgroundColor = [UIColor colorWithPatternImage: [UIImage imageNamed:@"1.png"] ];

不过这种方法需要注意的是在 iOS 下，UIKit 和 Core Graphics 的坐标系是不同的，直接设置会导致背景图上下颠倒。

## 2.exclusiveTouch 属性的正确用法。

exclusiveTouch 属性顾名思义是指 UIView 会独占整个 Touch 事件。 对于多点触摸的事件来说这个属性非常重要。比如在我上一篇《UIScrollView 表情选择的选实现》 的 DEMO 其实暗藏着一个多点触摸的 bug：可以多个手指选择不同的表情进行拖曳发送，这种表现明显是很诡异的。可以通过设置每个表情控件的 exclusiveTouch 来屏蔽掉这个 “feature”。类似的场景还有很多，不赘述。不过对于 exclusiveTouch 属性有几个小细节需要注意：
* exclusiveTouch 的意义在于：如果当前设置了 exclusiveTouch 的 UIView 是整个触摸事件的第一响应者，那么到你所有的手指离开屏幕前其他的 UIView 是无法接受到整个事件周期内所有的触摸事件。而如果当前 UIView 设置了 exclusiveTouch，却只是在一个事件周期中的某个时期被捎带触碰到，那么还是会投递相应的事件到当前这个 UIView。这也就是很多人认为 exclusiveTouch 属性不起作用的原因。
* 手势识别会忽略 exclusiveTouch 设置。详见：[《SimpleGestureRecognizers》][1]

## 3. 两种动画的不同表现。

对于简单的插值动画，一般会使用两种写法，一种是 [UIView beginAnimation:context] 一种是 [UIView animationWithDuration]。从效果上来说两种基本是一样的。但是有一些细微的差别：

* 对于被卷入动画中的 UIView，无论它是否有改变其属性值，都是不接受用户输入的。但是后者有个让人比较恶心的 “特性”：在 iOS5.0 之前的版本，动画期间会禁止掉整个程序的用户输入，而不仅仅是卷入动画的 UIView。详见 [《UIView Reference》][2]。如果需要在动画期间接受用户输入，需要设置：UIViewAnimationOptionAllowUserInteraction。
* 从程序的便利性上来说更加推荐使用 [UIView animationWithDuration]，方便实现动画完成的回调。

## 4.ipad 上 UIPopoverController 的使用

对于一般的浮窗，通过往 UIPopoverController 里添加 NavigatorController 能够消除掉 UIPopoverController 内的 “横梁”。


  [1]: https://developer.apple.com/library/ios/samplecode/SimpleGestureRecognizers/Introduction/Intro.html
  [2]: https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIView_Class/UIView/UIView.html