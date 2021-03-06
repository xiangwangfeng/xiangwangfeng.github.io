---
layout: post
title:  Objective-C 内存管理总结
---

最近一周都在看那本《iPhone3 开发基础教程》，写写一些搞毛的 DEMO，日子过得很舒坦。这本书对于我这种零基础的童鞋来说的确是本好书，看完一遍基本算是入门了。但从某种意义上这本书就等同于市面上泛滥的 Windows 下的《VC6 从入门到精通》这一类的书，很多地方只是教你怎么做，却没教你为什么。有些地方看得云里雾里的，不过通过苹果官网的 iOS Developer Library 各种文档也倒能查漏补缺。比如下面要讲的 Objective-C 相关的内存管理(总结附带翻译[这一篇][1])。

Objective-C 的内存管理方式介于 C/C++ 和 C# 之间，即不像 C/C++ 需要程序员包办一切，也不像 C# 一样有较完备的垃圾回收机制(当然在 Objective-C 2.0 以后有引入了 GC 机制，但在 ipad 和 iphone 上是不支持的。)，而是独辟蹊径：基于引用计数进行管理。当然引用计数这个并不是个多新鲜的话题，很多程序在逻辑层对资源进行管理也会用到，不过语法上有这方面的支持的语言 Objective-C 还是我见过的第一个。

## 基本原理

Objective-C 的内存管理模型基于对象的所有权。如果你拥有一个对象，那么你就有责任去释放它。一个对象可以有多个拥有者—- 而且在其生存期必须有一个拥有者，否则系统将自动销毁该对象。对于对象的所有权和释放需要遵守以下四个原则：

* 任何你创建的对象你都能获得其所有权。
         这里的创建包括使用 alloc，new，copy 等关键字来获得一个对象。
* 你可以通过 retain 来获得一个对象的所有权。
        除了创建对象外，获得某个对象所有权的唯一方法就只有给该对象发 retain 消息。一旦通过 retain 获得了该对象的所有权，就必须遵守上面红字标注的原则：如果你拥有一个对象，那么你就有责任去释放它。
* 如果你不再需要一个对象了，你就必须释放其所有权。
* 你不能释放非你所有的对象所有权。
        如果只是字面理解的话，上面的原则简单又明白，但是实际操作中我们总是会将 “拥有所有权” 和“拥有这个对象的指针”混为一谈。比如下面这个例子：
{% highlight objc %}
Person *person = [[Person alloc] init];
Person *another_person = person;
NSString *name = person.fullName;
[person release];
{% endhighlight %}

从内存管理的角度来说，上面这段代码是完全正确的。很多初学者可能对于 another_person 这个对象不需要释放是可以理解的，但是对 name 为何不需要 release 表示不解。不过只要对照上面四个原则就可以明白：
name 这个对象并非通过创建或者 retain 获得，所以 name 这个对象是指向 fullname 指向的内存区域，但是你对这个对象并没有拥有权，所以遵守规则 4 不需要 release。当然这也是有例外的，但是前提是 Person 这个类对 fullName 这个 property 进行了错误的重新实现。但是如果是这样，问题是 Person 类而不是当前这段代码。这个问题详见下文。  

## 技术内幕：引用计数

上述原则的原理很简单，所有的对象创建销毁都基于引用计数：创建一个对象，引用计数变为 1。发送一次 retain 消息，引用计数 + 1。发送一次 release 消息，引用计数 - 1。发送一次 autorelease 消息，引用计数在某个阶段结束后 - 1。一旦一个对象的引用计数变为了 0，则自动销毁该对象。

有一个非常方便的方法可以查看一个对象的当前引用计数，即给对象发送 retainCount 消息，但正常情况下并不推荐使用这个方法来管理内存：一来如果显式的调用 retainCount 来判断对象引用数并加以控制等于放弃了 Objective-C 提供的整个内存管理模型，这种方式不仅低效而且容易发生问题。二来内置的一些类为了效率等问题对 retainCount 会做一些特殊处理(比如 NSString)，所以这种方法并不普适。

## 自动释放

上个段落中提到 autorelease，但却没有做具体的分析。在我看来 autorelease 这个方法是 Objective-C 的一个很重要的概念和创新的地方。众所周知，C/C++ 编程时有个很重要的对内存管理的原则：谁创建谁销毁。但是这个原则在有些情况下却无法遵守：比如让一个普通函数返回一块内存区域。函数调用者必须在函数调用完毕后自己负责销毁这块内存。当然也有一种比较别扭的做法，将一个 2 级指针作为 [in,out] 参数，宣称这块内存是我特地叫这个函数帮我创建的，所以稍后我会自己负责销毁它。而 autorelease 这个关键字的存在就使得所有的一切都变得简单：调用者永远不用关心函数返回对象是否需要自己去销毁，只需要关心怎么使用而已：如果需要在当前函数作用域之外仍拥有这一块内存，调用 retain。

## autorelease 的原理

对于 Objective-C 来说，在每个 RunLoop 都会隐式的创建一个 NSAutoreleasepool 的实例，在 RunLoop 结束后发送 release 消息给该实例，这个实例受到消息后就会遍历在自己生命周期内所有收到 autorelease 消息的对象，并给它们发送一个 release 消息，这样就实现了对收到 autorelease 消息对象内存的管理。需要注意的是如果直接使用 pthread 库创建线程，需要自己额外创建 autorelease pool。

## 一些细节

1. 函数返回值的写法：
返回函数内生成对象，需要以 [instance autorelease]的方式返回
          取值函数，返回类成员，推荐以 [[intance retain] autorelease] 的方式返回 (大多数情况下完全可以通过 property 自动实现)
          
2.property retain 属性的实现
对于类成员的访问，一般会采用自动生成 property 的方式，而使用最多的属性莫过于 retain (尤其是 UI 的一些控件)。如果一个类成员 property 属性为 retain，那么其大致实现如下：
{% highlight objc%}
// set
if (property != newValue)
{
    [property release];
    property = [newValue retain];
}
//get
return [[property retain] autorelease];
{% endhighlight %}



  [1]: https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html#//apple_ref/doc/uid/20000994-BAJHFBGH