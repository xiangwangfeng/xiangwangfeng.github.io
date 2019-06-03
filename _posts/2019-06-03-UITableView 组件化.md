---
layout: post
title:  UITableView 组件化
---

## UITableView 的问题

在 iOS 开发中，UITableView 可以说是最常用的控件了。几行代码，实现对应方法，系统就会给你呈现一个 60 帧无比流畅的列表，让初学者成就感爆棚。然而随着开发的深入，我们就会慢慢觉察到当前的 UITableView 实现会有这样那样的问题。

* 几乎所有 TableView Adapter 中都有如下的代码 `registerClass(Nib):forCellReuseIdentifier` 来保证 cell 的重用。然而千篇一律的注册方法，写多了难免乏味，而如何给 reuseIdentifier 设置一个有意义的值又会成为各个强迫症程序员的烦恼之一。
* 随着业务深入，一个 UITableView 中往往会包含多种 model 对应多种 cell，那么建立 model 和 cell 的映射关系往往会非常蛋疼，无论是if else，switch，还是 map<model,cell> 都不是那么的优雅，每当 model 类型有所增删，开发者往往需要心惊胆战地 review 各个实现方法里是否正确添加/删除了相应的映射关系。
* 业务继续深入，为了保证相关代码整洁，易于拓展和性能高效，除了维护 model 和 class 关系(`CellFactory`)外，我们往往需要引入不同类型的 adapter 做职责分离：`DataSource` 管理数据源，`LayoutManager` 管理排版和提供预计算高度能力，`CellHeightCache` 进行高度缓存，这种做法可以一定程度减轻代码膨胀的问题。但也不是完美的：套路都是一样的，即使你熟练掌握了这些所谓的设计原则，在实际操作中还是有大量的重复代码，写得多了不免乏味。
* 数据源和 UI 并非绑定关系。当 model 变化时，我们往往需要通过当前 model 的位置反推出对应 cell 在 UITableView 中的位置(`indexPath`)，然后做相应的更新处理，反之亦然。但这部分工作无非是数组遍历，寻找 index，重复且繁琐，稍有不慎还有出错导致崩溃的可能。

## 新的解决方案