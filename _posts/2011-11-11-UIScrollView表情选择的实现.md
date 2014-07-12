---
layout: post
title:  UIScrollView表情选择的实现
---

前几天做即时通上表情选择的功能，结果不幸用了Windows上的老套路，思路错步步错，越写越感觉进入了死胡同，白白浪费了一下午，然后突然开窍，轻松解决，有必要记录下相关的细节。

情景很简单，做一个选择表情的界面，功能无非是点击弹出一个ScrollView，上面排列一些表情，然后通过拖曳或者点击的方式选择表情。具体可以参考iPad版QQ的操作。

一开始的思路很Windows化，既然UIView有touchesBegan相关的4种事件，基本可以映射成Windows上的MouseDown，MouseMove和MouseUp，那么很简单，在ScrollView的SuperView上响应这四个事件即可：当touchesBegan的时候通过hitTest找到ScrollView中的相应表情，并生成其拷贝，然后在touchesMoved的时候不断改变这份拷贝的位置，最后在touchEnded的时候销毁相应拷贝，并发送选择的通知给接收端即可。但实施的时候碰到了各种问题：

* ScrollView是当前View中的First Responder，也就意味着在ScrollView内部触发的事件都会优先发给它，由他直接处理并丢弃。解决这个问题的方法很简单，自定义一个类，继承ScrollView并在响应完相应触摸事件之后将事件传递给它的NextResponder，也就是它的SuperView处理。
* SrcollView的touchesMoved事件实际上并不被ScrollView直接处理！纳尼？从自己试验和stackoverflow上的一些问答可以知道ScrollView并不直接处理touchesMoved事件，换而言之，即使你继承了ScrollView重写相应方法还是无效。

于是貌似陷入了死胡同。不过在查阅各种资料的时候，突然发现了一种特别的做法(其实应该算是iOS下比较常规的做法，不过从Windows过来的童鞋可能会觉得比较奇怪)：自定义ScrollView内的表情，以它为First Responder去响应触摸事件即可。看了DEMO，突然回忆起一早看的官网上看到的一段话，恍然大悟：

触摸信息有时间和空间两方面，时间方面的信息称为阶段（phrase），表示触摸是否刚刚开始、是否正在移动或处于静止状态，以及何时结束—也就是手指何时从屏幕举起（参见图3-1）。触摸信息还包括当前在视图或窗口中的位置信息，以及之前的位置信息（如果有的话）。当一个手指接触屏幕时，触摸就和某个窗口或视图关联在一起，这个关联在事件的整个生命周期都会得到维护。如果有多个触摸同时发生，则只有和同一个视图相关联的触摸会被一起处理。类似地，如果两个触摸事件发生的间隔时间很短，也只有当它们和同一个视图相关联时，才会被处理为多触击事件。

和Windows不同，由于iOS设备的特殊性，在事件处理方面简单了很多，从Windows的角度来说，iOS上的一个触摸事件基本等同于Windows上对某个控件触发某种消息后进行Capture直到最终ReleaseCapture这么一个过程，只要一个事件开始就不会做HitTest，事件是和刚开始确定的First Responder所关联。这样一来实现拖曳就非常简单，主要自定义一个继承自UIImageView或者UIButton用于表情显示的控件，并重写其触摸事件的实现即可：

触摸事件发生：初始化拖曳状态，对表情对象进行拷贝并显示：
{% highlight objc %}
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    scrollView.scrollEnabled = NO;
    isDraging = NO;
    //生成自己的clone
    self.clone = [Emoticon getEmoticonFromIndex:iconIndex];
    CGPoint newLocation = [[touches anyObject]         locationInView:parentView];
    CGFloat startX = newLocation.x - 128 / 2;
    CGFloat startY = newLocation.y - 128 / 2;
    CGRect rect = CGRectMake(startX, startY, 128, 128);
    [clone setFrame:rect];
    [parentView addSubview:clone];
    clone.center = newLocation;
    [super touchesBegan:touches withEvent:event];
}
{% endhighlight %}
移动中：移动拷贝对象，并设置拖曳状态 (设置这个状态是为了区分点击事件和拖曳事件)
{% highlight objc %}
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
{
    isDraging = YES;
    CGPoint newLocation = [[touches anyObject] locationInView:parentView];
    clone.center = newLocation;
    [super touchesMoved:touches withEvent:event];
}
{% endhighlight %}
触摸事件结束：先响应父类事件(区分点击和拖曳事件)，设置拖曳状态，并发送通知
{% highlight objc %}
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event
{
    [super touchesEnded:touches withEvent:event];
    CGPoint newLocation  = [[touches anyObject] locationInView:parentView];
    //如果不在当前ScrollView内就认为已经选了
    CGRect scrollViewRect = scrollView.frame;
    if (!CGRectContainsPoint(scrollViewRect, newLocation))
    {
        [self postNotification:YES];
    }
    [self cleanClone];
}
{% endhighlight %}
这样一个简单的表情拖曳的功能就算完成了，详细的代码可以参考这里：[点此下载][1] 。(XCode4.2)


  [1]: https://github.com/xiangwangfeng/amao_misc/blob/master/Demos/DragDropDemo.7z