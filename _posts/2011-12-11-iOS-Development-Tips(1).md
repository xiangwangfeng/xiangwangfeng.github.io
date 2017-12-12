---
layout: post
title:  iOS Development Tips(1)
---

ipad 即时通开发基本完毕，由于前期基本是边学边做，在很多方面都不是很小心，犯了很多错误，趁着刚洗完澡，手还热乎着，将以前总结的一些 Tips 整理一下，主要是 UI 和动画方面，备用。

## 1. 对横向的 UIPanGesture 和 UITableView 本身的上下滑动做区分。

在 UIGestureRecongizer 的委托中进行判断，通过比较 y 轴速度和一个预设阈值的大小即可：
{% highlight objc %}
(BOOL)gestureRecognizerShouldBegin:(UIPanGestureRecognizer *)gestureRecognizer
{
    CGPoint velocity = [gestureRecognizer velocityInView:self.view];
    return abs(velocity.y) <= kYVelocityThreshold;
}
{% endhighlight %}

## 2. 清除 UITableView 底部多余的分割线

{% highlight objc %}
+ (void)setExtraCellLineHidden: (UITableView *)tableView
{
    UIView *view = [UIView new];
    view.backgroundColor = [UIColor clearColor];
    [tableView setTableFooterView:view]; 
    [view release];
}
{% endhighlight %}
在 iOS4.3 和 iOS5.0 中通过：值得注意的是在 iOS4.3 中可以直接设置 footer 为 nil，但是在 5.0 不行，因为 UITableView 会默认生成一个 Footer。(详见 iOS Release Notes 中的说明：Returning nil from the tableView:viewForHeaderInSection: method (or its footer equivalent) is no longer sufficient to hide a header. You must override tableView:heightForHeaderInSection: and return 0.0 to hide a header.)

## 3. 计算 ScrollView contentSize 的通用方法。

{% highlight objc %}
CGRect contentRect = CGRectZero;
for (UIView *view in scrollView.subviews)
{
    contentRect = CGRectUnion(contentRect, view.frame);
}
scrollView.contentSize = contentRect.size;
{% endhighlight %}

## 4. 在同个 UIView 中让几个互相有关联的手势共存。

调用 requireGestureRecognizerToFail 方法。这个方法可以让某个先满足条件的手势并不立即触发，而是等待该手势 requireGestureRecognizerToFail 指定的手势失败后再触发。这样就可以有效解决 “单击” 和“双击”，“点一下”和 “长按” 等手势之间的关联问题。不过值得一提的是用这个方法区分 UIPanGesture 和 UISwipeGesture 效果并不好，个人建议是：只使用 UIPanGesture，在其 state 为 UIGestureRecognizerStateEnded 时进行速度判断，以此作为依据触发或不触发相应的事件。

## 5. 对于大图片，慎用 imageNamed 这种方法来返回 UIImage。


理由是通过这个方法得到的 UIImage 是一直缓存在内存中，直到程序结束为止——这和我原先以为的系统会做一个引用计数，如果在引用计数为 0 的情况下自动清除该块内存的想法不一致。而且值得一提的是所有用 IB 在 xib 文件内设置图像的方法都是调用 imageNamed 这个方法，也就是说：这些内存在程序结束之前是一直保留着的，对于某些比较吃内存的应用就需要好好规划一下了。不过 UITableViewCell 这种需要重用的控件就很需要它了。

## 6. 如何查找 EXEC_BAD_ACCESS 的原因

发生 EXEC_BAD_ACCESS 的原因很简单：野指针，发送消息到一块已释放的内存。所以有两种方法都可以实现对 EXEC_BAD_ACCESS 原因的查找：

* 设置 NSZombieEnabled，但是这样做有个问题就是所有的内存并不是真正被释放，内存占用会一直增加。（不过 Archive 下是不能设置 NSZombieEnabled，所以也不用担心在 release 程序给用户的时候因为误设了 NSZombieEnabled 而引起问题）
* 用分类或者拓展的方式 “重写” 掉 NSObject 的 responseToSelector 方法
{% highlight objc %}
#ifdef _FOR_DEBUG_
-(BOOL) respondsToSelector:(SEL)aSelector {
printf("selector: %sn", [NSStringFromSelector(aSelector) UTF8String]);
return [super respondsToSelector:aSelector];
}
#endif
{% endhighlight %}

## 7. 使 UIImagePickerController 的按钮显示中文

因为默认设置为英文，所以需要通过添加中文相关的信息：

* 修改 Localization Native Development Region 为 China
* 给工程添加中文资源：Project-》Localizations
* 
## 8.UITableView 的小贴士


* 如果 UITableViewCell 是通过 IB 进行界面排版，那么是不能直接在 IB 里面设置 reuseIdentifier 的，解决方法是：在 UIView 内添加一个 UITableViewCell，或者是当前 UITableViewCell 重写 reuseIdentifier 方法。
* 提高 UITableView 性能的小贴士：设置 UITableViewCell 为不透明，尽量重用 cell，对于 cell 内部图像进行预渲染(适合于像微波这种应用，详见 Improving Image Drawing Performance on iOS)
* 重用 cell 时记得在代码里面写上完整的 cell 设置过程，尤其是通过 IB 进行 cell 设计时很容易犯这样的错误：重用时拿到的 cell 可能是上一个 cell 进行过控件重排版后的结果，而不是 awakeFromNib 时得到的初始状态，所以需要重新排版。
