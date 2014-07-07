---
layout: post
title:  Objective-C内存管理总结
---

最近一周都在看那本《iPhone3开发基础教程》，写写一些搞毛的DEMO，日子过得很舒坦。这本书对于我这种零基础的童鞋来说的确是本好书，看完一遍基本算是入门了。但从某种意义上这本书就等同于市面上泛滥的Windows下的《VC6从入门到精通》这一类的书，很多地方只是教你怎么做，却没教你为什么。有些地方看得云里雾里的，不过通过苹果官网的iOS Developer Library 各种文档也倒能查漏补缺。比如下面要讲的Objective-C相关的内存管理(总结附带翻译[这一篇][1])。

Objective-C的内存管理方式介于C/C++和C#之间，即不像C/C++需要程序员包办一切，也不像C#一样有较完备的垃圾回收机制(当然在Objective-C 2.0以后有引入了GC机制，但在ipad和iphone上是不支持的。)，而是独辟蹊径：基于引用计数进行管理。当然引用计数这个并不是个多新鲜的话题，很多程序在逻辑层对资源进行管理也会用到，不过语法上有这方面的支持的语言Objective-C还是我见过的第一个。

##基本原理
Objective-C的内存管理模型基于对象的所有权。如果你拥有一个对象，那么你就有责任去释放它。一个对象可以有多个拥有者—-而且在其生存期必须有一个拥有者，否则系统将自动销毁该对象。对于对象的所有权和释放需要遵守以下四个原则：

* 任何你创建的对象你都能获得其所有权。
         这里的创建包括使用alloc，new，copy等关键字来获得一个对象。
* 你可以通过retain来获得一个对象的所有权。
        除了创建对象外，获得某个对象所有权的唯一方法就只有给该对象发retain消息。一旦通过retain获得了该对象的所有权，就必须遵守上面红字标注的原则：如果你拥有一个对象，那么你就有责任去释放它。
* 如果你不再需要一个对象了，你就必须释放其所有权。
* 你不能释放非你所有的对象所有权。
        如果只是字面理解的话，上面的原则简单又明白，但是实际操作中我们总是会将“拥有所有权”和“拥有这个对象的指针”混为一谈。比如下面这个例子：
{% highlight objc %}
Person *person = [[Person alloc] init];
Person *another_person = person;
NSString *name = person.fullName;
[person release];
{% endhighlight %}

从内存管理的角度来说，上面这段代码是完全正确的。很多初学者可能对于another_person这个对象不需要释放是可以理解的，但是对name为何不需要release表示不解。不过只要对照上面四个原则就可以明白：
name这个对象并非通过创建或者retain获得，所以name这个对象是指向fullname指向的内存区域，但是你对这个对象并没有拥有权，所以遵守规则4不需要release。当然这也是有例外的，但是前提是Person这个类对fullName这个property进行了错误的重新实现。但是如果是这样，问题是Person类而不是当前这段代码。这个问题详见下文。  

##技术内幕：引用计数
上述原则的原理很简单，所有的对象创建销毁都基于引用计数：创建一个对象，引用计数变为1。发送一次retain消息，引用计数+1。发送一次release消息，引用计数-1。发送一次autorelease消息，引用计数在某个阶段结束后-1。一旦一个对象的引用计数变为了0，则自动销毁该对象。

有一个非常方便的方法可以查看一个对象的当前引用计数，即给对象发送 retainCount消息，但正常情况下并不推荐使用这个方法来管理内存：一来如果显式的调用retainCount来判断对象引用数并加以控制等于放弃了Objective-C提供的整个内存管理模型，这种方式不仅低效而且容易发生问题。二来内置的一些类为了效率等问题对retainCount会做一些特殊处理(比如 NSString)，所以这种方法并不普适。
##自动释放
上个段落中提到autorelease，但却没有做具体的分析。在我看来autorelease这个方法是Objective-C的一个很重要的概念和创新的地方。众所周知，C/C++编程时有个很重要的对内存管理的原则：谁创建谁销毁。但是这个原则在有些情况下却无法遵守：比如让一个普通函数返回一块内存区域。函数调用者必须在函数调用完毕后自己负责销毁这块内存。当然也有一种比较别扭的做法，将一个2级指针作为[in,out]参数，宣称这块内存是我特地叫这个函数帮我创建的，所以稍后我会自己负责销毁它。而autorelease这个关键字的存在就使得所有的一切都变得简单：调用者永远不用关心函数返回对象是否需要自己去销毁，只需要关心怎么使用而已：如果需要在当前函数作用域之外仍拥有这一块内存，调用retain。
##autorelease的原理
对于Objective-C来说，在每个RunLoop都会隐式的创建一个NSAutoreleasepool的实例，在RunLoop结束后发送release消息给该实例，这个实例受到消息后就会遍历在自己生命周期内所有收到autorelease消息的对象，并给它们发送一个release消息，这样就实现了对收到autorelease消息对象内存的管理。需要注意的是如果直接使用pthread库创建线程，需要自己额外创建autorelease pool。
##一些细节
1.函数返回值的写法：
返回函数内生成对象，需要以 [instance autorelease]的方式返回
          取值函数，返回类成员，推荐以[[intance retain] autorelease]的方式返回 (大多数情况下完全可以通过property自动实现)
          
2.property retain属性的实现
对于类成员的访问，一般会采用自动生成property的方式，而使用最多的属性莫过于retain (尤其是UI的一些控件)。如果一个类成员property属性为retain，那么其大致实现如下：
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