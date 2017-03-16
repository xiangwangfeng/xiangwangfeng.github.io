---
layout: post
title:  IGListKit diff 实现简析
---

# 前言

`Instagram` 在去年年底开源了基于 `数据驱动` 的 `UICollectionView` 框架 `IGListKit`。整个框架通过 `Adapter` 将过去直接暴露的 `CollectionView` `Datasource` 和 `Delegate` 实现进行包装，并通过它关联 `Model` 和 `IGListSectionController`。这使得上层用户只需要继承并实现所需要的 `IGListSectionController` 接口即可，很好地进行了代码解耦。

整个思路比较新颖又很简洁，即使不直接使用这个框架，也可以按照思路依葫芦画瓢做出自己的简易版方案。不过今天要讲的并不是 `IGListKit` 本身，而是扒一扒它里面的 `IGListDiff` 实现。


相比于前端各种各样 `Virtual DOM diff` 实现，移动端在这方面较为欠缺。在 `UITableView` 和 `UICollectionView` 的使用上，一旦同时发生数据删除，更新，添加时，我们的做法往往是手动计算出变化 `NSIndexPaths` 并调用批量刷新，甚至简单粗暴地调用 `reloadData` 做一次全刷新。而 `IGListKit` 的 `IGListDiff` 正是为这种场景而生：当数据变化产生后，通过调用 `IGListDiff` 自动计算前后两次的差值，为后续批量刷新提供数据。整个算法的复杂度为 O(n)，相当高效。


# IGList Diff


## 算法介绍

`IGListDiff` 使用一个额外的哈希表和两个新旧哈希列表 `hash entry list` 使得比较的算法复杂度从 O(n^2) 变成 O(n)。一个 `Hashentry` 都需要定义四个信息
	
	
```objc
/// hash entry
struct IGListentry {
    /// 记录旧队列中相同 hash 值对象个数
    NSInteger oldCounter = 0;
    /// 记录新队列中相同 hash 值对象个数
    NSInteger newCounter = 0;
    /// 旧队列中当前 hash 对应的对象序号堆栈
    stack<NSInteger> oldIndexes;
    /// 标示数据是否有更新
    BOOL updated = NO;
};
	
```

然后我们就可以进行数据比较了，主要是四个步骤	

* 遍历新队列，计算对象 `hash` 值并找到对应 `entry` ，使得 `newCount++`，同时计入 `new entry list`
* 遍历旧队列，计算对象 `hash` 值并找到对应 `entry` ，使得 `oldCount++`，同时将当前序号入栈 `oldIndexes.push(i)`，并记录 `old entry list`
* 遍历 `new entry list`，检查 `entry` 对应的 `oldIndexes` 信息，如果堆栈中有旧队列序号值，则表示当前 `entry` 至少对应新旧队列中的两个对象，即发生所谓的 `entry match`，`进行记录，方便后续反向查询`。再通过检查 `新队列当前对象` 和 `entry 对应旧对象` 是否相同确认 `update` 状态。
* 再次遍历新旧 `entry list`，检查每个 `entry` 的 `entry match` 状态 
	* 没有 `entry match` 的对象，在新队列中的被标记为 `insert`，而在旧队列中的则被标示为 `delete`
	* 有 `entry match` 的对象通过比较新旧队列序号和 `update` 状态分表表示为 `update`，`move` 和 `not modified`


## 举例

以旧数组 `[1,2,3]` 和新数组 `[1,3,5]` 为例 （数字比较直接忽略 `update` 状态）

* 遍历新数组，得到 `[entry1,entry3,entry5]` 列表，记为 `nl`
* 遍历旧数组，得到 `[entry1,entry2,entry3]` 列表，记为 `ol`
* 遍历 `nl`，由于 `entry1 oldIndexs = [0]`，`entry3 oldIndexes = [2]` 所以他们是 `entry match`，做记录 （`reverse lookup`）
* 遍历 `ol` 
	* `entry1` 有 `entry match`，跳过
	* `entry2` 没有 `entry match`，记为 `delete`
	* `entry3` 有 `entry match`，跳过
* 遍历 `nl`
	* `entry1` 有 `entry match`，同时相对位置不变，记为 `not modified` (其实就是跳过)
	* `entry3` 有 `entry match`，但是相对位置变化，记为 `move`
	* `entry5` 没有 `entry match`，记为 `insert`	
* 输出一个包含 `insert`，`move`，`delete` 和 `update` 列表信息的最终结果