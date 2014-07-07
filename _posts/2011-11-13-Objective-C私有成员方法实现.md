---
layout: post
title:  Objective-C私有成员方法实现
---

众所周知，Objective-C是没有私有方法的概念，成员方法只有两种：类成员方法(+)和对象成员方法(-)，这两种都是对外开放的。而本身提供的@private，@protect和@public关键字也仅供成员对象使用而已，比较坑爹。

所以在Objective-C中实现私有成员方法的含义是：创建一个让他人较难调用的方法，而并非实现如C++和JAVA般让非当前类对象无法调用的方法。做法如下：假设我们需要构建一个类叫做AClass，它提供一个公开成员函数publicMethod和一个“私有成员方法”privateMethod，那么在AClass.h文件中我们就有：
> @interface AClass : NSObject 
- (void)publicMethod;
@end

而在.m文件中使用一个类拓展：
> @interface AClass ()
- (void)privateMethod;
@end
@implementation AClass
- (void)publicMethod{...}
- (void)privateMethod;{...} 
@end

这样因为privateMethod隐藏于.m中，调用类并不能直接通过.h看到其声明，等于间接实现了私有成员方法。当然网上也有各种教程是使用分类(Categories)而非类拓展，不过相比于分类，使用类拓展有两个明显的好处：

* 类拓展声明的方法必须放在类的主implementation区间中实现，这样可以消除因为使用有名分类(拓展基本等同于一个匿名分类)而产生不必要的N个implementation段。
* 如果在类拓展中声明的类没有实现，编译器会给出相应的warning，而分类则没有这个便利。

不过正如我一开始所说：因为Objective-C是个动态语言，这也意味你可以发送任何消息给任何一个类对象，而任何一个类对象也可以接受任何方法。如果某个AClass的实例要强行调用privateMethod方法(调用的说法其实不合适，应该说发送消息) ，编译期间是不会有任何error，而运行时也不会有任何问题。但是编译期间会给出一个warning，这个warning的意思是：你个2货，你都声明它是私有方法了却还去调用，有病啊。如果你硬要这么做，后果自负。
