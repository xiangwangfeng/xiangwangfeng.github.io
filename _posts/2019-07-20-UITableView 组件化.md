---
layout: post
title:  UITableView 组件化
---

# 源起

在 iOS 开发中，UITableView 可以说是最常用的控件。几行代码，实现对应方法，系统就会给你呈现一个 60 帧无比流畅的列表，让初学者成就感爆棚。然而随着开发的深入，我们就会慢慢觉察到当前的 UITableView 实现会有这样或那样的问题。

* 繁琐的重用流程

几乎所有 TableView Adapter 中都有如下的代码 `registerClass(Nib):forCellReuseIdentifier` 进行 cell 重用的注册，后续又需要使用 `dequeueReusableCellWithIdentifier:` 获取对应 cell。苹果的这套重用机制对于开发者来说相当简单友好，但写多了难免觉得重复乏味。同时如何给 cell 设置一个有意义且不重复的 reuseIdentifier 又会成为众多强迫症程序员的烦恼之一。

* 不安全的 model 和 cell 映射关系

随着业务深入，一个 UITableView 往往会包含多种 model，对应不同形式的 cell，那么建立 model 和 cell 的映射关系就会非常蛋疼，无论是if else，switch，还是 map<model,cell> 都不是那么的优雅，每当 model 类型有所增删，开发者往往需要心惊胆战地检查各处实现方法里是否进行了正确的处理。

* 单调的优化过程

业务继续深入，为了保证相关代码整洁，易于拓展和性能高效，除了维护 model 和 cell 关系(`ModelCellMap`)外，我们往往需要引入各种类做职责分离：`DataSource` 管理数据源，`LayoutManager` 负责排版和提供预计算高度能力，`CellHeightCache` 提供高度缓存，`Interactor` 提供事件路由和处理等等，这样可以一定程度减轻代码膨胀的问题。但也不是完美的：套路都是类似的，即使你熟练掌握了这些所谓的设计原则，在实际操作中仍有大量的重复代码。


* 数据源和 UI 不绑定

当 model 变化时，我们往往需要通过当前 model 位置反推出 cell 在 UITableView 中的位置(即 `indexPath`)，然后做相应的更新处理，反之亦然。但这部分工作无非是数组遍历，寻找 index，重复且繁琐，稍有不慎还有出错导致崩溃的可能。

# 组件化方案

为了解决如上问题，同时也受到 IGListKit 和 React.js 的启发，M80TableViewComponent 提出了一种组件化的解决方案，实现类似 React.js 的 "单向数据绑定" 功能，同时将大量的重复计算归纳在组件内部，上层使用者只需要根据当前业务创建相应组件并组合使用即可。

## 基础组件

为了实现整个 UITableView 的流程， M80TableViewComponent 引入三个基础组件：

* M80TableViewComponent
* M80TableViewSectionComponent
* M80TableViewCellComponent

顾名思义，他们分别对应 UITableView，Section 和 UITableViewCell。用前端技术做类比的话，M80TableViewComponent 就是我们定义的 VirtualDOM，而 UITableView 则是真正的 DOM。前者记录虚拟的层次结构，后者仍负责最终的渲染。具体关系参考下图：

![](/images/component_arch.jpg)

## 简单使用

### 定义组件

一个简单的 M80TableViewComponent 定义如下

![](/images/item_component.png)

这是一个用于文本列表显示的组件，只实现最基本组件协议

* 当前组件对应何种 UITableViewCell：   - (Class)cellClass
* 当前组件对应 UITableViewCell 高度是多少: - (CGFloat)height
* 如何通过当前组件配置 UITableViewCell: - (void)configure:(UITableViewCell *)cell



### 和 UITableView 联动

定义完组件后，我们只需要按照顺序将组件加入父组件中，即可完成和 UITableView 的绑定。 
 
![](/images/component_usage.png)

具体效果详见 [Example Project](https://github.com/xiangwangfeng/M80TableViewComponent)


## 特性

看完上述的使用方式后，你很可能将 M80TableViewComponent 当成一种固定数据源组装方式而已，并没有其他新意。但事实上，除了充当固定结构数据源外，它还有如下优势

### 单向绑定

当我们使用组件时，一旦当前 M80TableViewComponent 和 UITableView 关联，后续针对 M80TableViewComponent 的所有操作都会实时反应到 UITableView 之上，包括对 cell component 的移除，刷新，插入，以及 section component 的插入，移除和刷新。我们不再需要繁琐地通过 controller 同时操作 view 和 model 以保证其一致性，只需要单纯操作 component 即可：component 将根据自身层次结构计算出对应的 UI 层次结构，在修改 component 内部结构的同时也会自动获取到对应的 cell 对象进行修改。这样做的好处是上层开发只需要关注 component 即可，而不再关心 indexPath 相关的计算过程，从而规避繁复的 indexPath 计算及计算错误导致的崩溃。

### 灵活组装功能

使用 M80TableViewComponent 可以轻易支持多种不同类型的数据模型，同时由于我们将复用层次从 vc/tableview 下降到 cell/section component 层次，也更方便了在不同场景下的组合使用。


### 自动重用

每一个 M80TableViewCellComponent 在第一次被使用时都会通过 `M80TableViewComponentRegister` 根据上下文信息自动绑定 reuseIdentifier 和 cellClass 的关系，完成 cell 的重用。默认使用当前 cell component 的类名作为 reuseIdentifier，既能保证不与其他 cell 重名，又省去了取名之苦。

### 高度优化和局部刷新

在 iOS 中比较蛋疼的事情是如何判断两个对象相等：在不使用 runtime 的场景下，往往需要业务层添加大量冗余代码用于支持对象比较，而使用了 runtime 又会对业务侵入过多。在 M80TableViewComponent 中我们使用了一种不基于 runtime 且比较轻量的方法：

所有的 M80TableViewCellComponent 都遵循 M80ListDiffable 协议，以用于组件内部的一致性判断:

* - (NSString *)diffableHash;

默认情况下，每个 cell component 在初始化时都会有自己唯一的 cellIdentifier 作为 diffableHash。

以此为出发点，我们就可以进行如下场景的优化。

* 自动 cell 高度缓存
* 通过 ListDiff 算法实现的 section 局部刷新

当开启高度缓存选项时，M80TableViewComponent 计算 cell 高度后会自动记录 diffableHash 和 height 的对应关系。后续再次刷新将自动获取对应高度而无需再次计算。当一个 cell 有多重状态，需要在不同状态下展示不同高度时，则可以通过业务状态返回不同的 diffableHash 进行高度切换。除了高度缓存外，M80TableViewComponent 也提供了一种预计算高度的机制，在组装完 cell component 后，只需要简单调用基类方法 `measure` 就可以直接完成预计算。


而适用局部刷新时，cell component 的 diffableHash 将做为唯一标识:old components 和 new components 根据 diffableHash 被 hash 到不同桶内，冲突桶中的 component 标记为 move，不冲突桶中的 component 则为 add/remove。详细算法可参考 M80ListDiff 函数。在合适的场景下，使用 ListDiff 进行 section 的重新载入，而不是人工计算各种变化信息后进行逐一操作，能够在保证性能的前提下，简化开发过程和良好的界面表现。


## 使用贴士

不同于以往构建 UITableView 的常见用法，使用 M80TableViewComponent 推荐所有操作都针对 component 进行。

* 涉及单个 cell 的操作，直接使用 cell component 本身的方法，如 remove，reload 方法。
* 涉及单个 section 内多个 cell 变化，可以考虑每次重新 setComponents 或调用 reloadUsingListDiff 进行局部刷新。
* 涉及到多 section 多 cell 变化，则可以重新组装所有 component。一方面这样做比较简单，不容易出错。另一方面 component 只是 viewmodel，在真正刷新前的批量操作并不会有过多性能问题。

