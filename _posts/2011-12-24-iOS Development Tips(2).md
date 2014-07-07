---
layout: post
title:  "iOS Development Tips(2)"
---

继上一篇的《iOS Development Tips(1)》 ,继续整理一些日常开发中的小Tips。
##1.UIView设置背景图片的方法。
 UIView作为所有控件的基类很诡异的是没有背景图属性，只有BackgroundColor属性。于是可以通过
> self.view.backgroundColor = [UIColor colorWithPatternImage: [UIImage imageNamed:@"1.png"] ];

的方法进行背景图的设置。这样做的问题在于：有alpha通道的图片在转换后会丧失其alpha值(iOS5.0以前的系统有这问题)。解决的方法经过两次设置：
> view.backgroundColor = [UIColor clearColor];
view.layer.backgroundColor = [UIColor colorWithPatternImage: [UIImage imageNamed:@"1.png"] ];

不过这种方法需要注意的是在iOS下，UIKit和Core Graphics的坐标系是不同的，直接设置会导致背景图上下颠倒。

##2.exclusiveTouch属性的正确用法。
exclusiveTouch属性顾名思义是指UIView会独占整个Touch事件。 对于多点触摸的事件来说这个属性非常重要。比如在我上一篇《UIScrollView表情选择的选实现》 的DEMO其实暗藏着一个多点触摸的bug：可以多个手指选择不同的表情进行拖曳发送，这种表现明显是很诡异的。可以通过设置每个表情控件的exclusiveTouch来屏蔽掉这个“feature”。类似的场景还有很多，不赘述。不过对于exclusiveTouch属性有几个小细节需要注意：
* exclusiveTouch的意义在于：如果当前设置了exclusiveTouch的UIView是整个触摸事件的第一响应者，那么到你所有的手指离开屏幕前其他的UIView是无法接受到整个事件周期内所有的触摸事件。而如果当前UIView设置了exclusiveTouch，却只是在一个事件周期中的某个时期被捎带触碰到，那么还是会投递相应的事件到当前这个UIView。这也就是很多人认为exclusiveTouch属性不起作用的原因。
* 手势识别会忽略exclusiveTouch设置。详见：[《SimpleGestureRecognizers》][1]

##3.两种动画的不同表现。
对于简单的插值动画，一般会使用两种写法，一种是[UIView beginAnimation:context] 一种是[UIView animationWithDuration]。从效果上来说两种基本是一样的。但是有一些细微的差别：

* 对于被卷入动画中的UIView，无论它是否有改变其属性值，都是不接受用户输入的。但是后者有个让人比较恶心的“特性”：在iOS5.0之前的版本，动画期间会禁止掉整个程序的用户输入，而不仅仅是卷入动画的UIView。详见[《UIView Reference》][2]。如果需要在动画期间接受用户输入，需要设置：UIViewAnimationOptionAllowUserInteraction。
* 从程序的便利性上来说更加推荐使用[UIView animationWithDuration]，方便实现动画完成的回调。

##4.ipad上UIPopoverController的使用
对于一般的浮窗，通过往UIPopoverController里添加NavigatorController能够消除掉UIPopoverController内的“横梁”。


  [1]: https://developer.apple.com/library/ios/samplecode/SimpleGestureRecognizers/Introduction/Intro.html
  [2]: https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIView_Class/UIView/UIView.html