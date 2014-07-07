---
layout: post
title:  "iOS Development Tips(1)"
---

ipad即时通开发基本完毕，由于前期基本是边学边做，在很多方面都不是很小心，犯了很多错误，趁着刚洗完澡，手还热乎着，将以前总结的一些Tips整理一下，主要是UI和动画方面，备用。

##1.对横向的UIPanGesture和UITableView本身的上下滑动做区分。
在UIGestureRecongizer的委托中进行判断，通过比较y轴速度和一个预设阈值的大小即可：
{% highlight objc %}
(BOOL)gestureRecognizerShouldBegin:(UIPanGestureRecognizer *)gestureRecognizer
{
    CGPoint velocity = [gestureRecognizer velocityInView:self.view];
    return abs(velocity.y) <= kYVelocityThreshold;
}
{% endhighlight %}
##2.清除UITableView底部多余的分割线
{% highlight objc %}
+ (void)setExtraCellLineHidden: (UITableView *)tableView
{
    UIView *view = [UIView new];
    view.backgroundColor = [UIColor clearColor];
    [tableView setTableFooterView:view]; 
    [view release];
}
{% endhighlight %}
在iOS4.3和iOS5.0中通过：值得注意的是在iOS4.3中可以直接设置footer为nil，但是在5.0不行，因为UITableView会默认生成一个Footer。(详见iOS Release Notes中的说明：Returning nil from the tableView:viewForHeaderInSection: method (or its footer equivalent) is no longer sufficient to hide a header. You must override tableView:heightForHeaderInSection: and return 0.0 to hide a header.)
##3.计算ScrollView contentSize的通用方法。
{% highlight objc %}
CGRect contentRect = CGRectZero;
for (UIView *view in scrollView.subviews)
{
    contentRect = CGRectUnion(contentRect, view.frame);
}
scrollView.contentSize = contentRect.size;
{% endhighlight %}
##4.在同个UIView中让几个互相有关联的手势共存。
调用requireGestureRecognizerToFail方法。这个方法可以让某个先满足条件的手势并不立即触发，而是等待该手势requireGestureRecognizerToFail指定的手势失败后再触发。这样就可以有效解决“单击”和“双击”，“点一下”和“长按”等手势之间的关联问题。不过值得一提的是用这个方法区分UIPanGesture和UISwipeGesture效果并不好，个人建议是：只使用UIPanGesture，在其state为UIGestureRecognizerStateEnded时进行速度判断，以此作为依据触发或不触发相应的事件。
##5.对于大图片，慎用imageNamed这种方法来返回UIImage。
理由是通过这个方法得到的UIImage是一直缓存在内存中，直到程序结束为止——这和我原先以为的系统会做一个引用计数，如果在引用计数为0的情况下自动清除该块内存的想法不一致。而且值得一提的是所有用IB在xib文件内设置图像的方法都是调用imageNamed这个方法，也就是说：这些内存在程序结束之前是一直保留着的，对于某些比较吃内存的应用就需要好好规划一下了。不过UITableViewCell这种需要重用的控件就很需要它了。
##6.如何查找EXEC_BAD_ACCESS的原因
发生EXEC_BAD_ACCESS的原因很简单：野指针，发送消息到一块已释放的内存。所以有两种方法都可以实现对EXEC_BAD_ACCESS原因的查找：

* 设置NSZombieEnabled，但是这样做有个问题就是所有的内存并不是真正被释放，内存占用会一直增加。（不过Archive下是不能设置NSZombieEnabled，所以也不用担心在release程序给用户的时候因为误设了NSZombieEnabled而引起问题）
* 用分类或者拓展的方式“重写”掉NSObject的responseToSelector方法
{% highlight objc %}
#ifdef _FOR_DEBUG_
-(BOOL) respondsToSelector:(SEL)aSelector {
printf("selector: %sn", [NSStringFromSelector(aSelector) UTF8String]);
return [super respondsToSelector:aSelector];
}
#endif
{% endhighlight %}
##7.使UIImagePickerController的按钮显示中文
因为默认设置为英文，所以需要通过添加中文相关的信息：

* 修改Localization Native Development Region 为China
* 给工程添加中文资源：Project-》Localizations
##8.UITableView的小贴士

* 如果UITableViewCell是通过IB进行界面排版，那么是不能直接在IB里面设置reuseIdentifier的，解决方法是：在UIView内添加一个UITableViewCell，或者是当前UITableViewCell重写reuseIdentifier方法。
* 提高UITableView性能的小贴士：设置UITableViewCell为不透明，尽量重用cell，对于cell内部图像进行预渲染(适合于像微波这种应用，详见Improving Image Drawing Performance on iOS)
* 重用cell时记得在代码里面写上完整的cell设置过程，尤其是通过IB进行cell设计时很容易犯这样的错误：重用时拿到的cell可能是上一个cell进行过控件重排版后的结果，而不是awakeFromNib时得到的初始状态，所以需要重新排版。
