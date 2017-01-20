---
layout: post
title:  打造高可拓展性 iOS KeyValueStore
---

# 前言

众所周知，在 `iOS` 端做持久化无非两种方式 （习惯性忽略 `CoreData`）

* 文档存储
* 数据库存储

前者以 `plist`，`NSUserDefautls`，`NSKeyedArchiver` 等为代表，优点是使用简单，但序列化和反序列化时往往需要操作整个文件，性能较差，且不适合大量数据存储。

后者以 `SQLite` 为代表，优点是性能高，适宜大数据和复杂结构的存储，缺点是使用较繁琐，拓展和兼容老数据时较为复杂。

而目前 `iOS` 社区对这种情况的改进方向无非是提供性能不错的 `NoSQL` 存储，如 `Realm`，简易 `KeyValueStore`。而后者就是今天探讨的重点。

# 问题

一个简易的 `KeyValueStore`，如 [YTKKeyValueStore](https://github.com/yuantiku/YTKKeyValueStore) 刨去其具体实现细节，核心就是使用 `SQLite` 创建一张只有 `Key` (`TEXT`，主键) 和 `Value` (`BLOB/TEXT`) 的数据库表。思路很简单，能够适应很多场景，但也有它的局限性：

* 无内建缓存
* 存储信息无法分类
* 额外信息无法存储 (非特例化的额外数据)
* 不支持复杂查询

# 解决方案


## 内建缓存

缓存永远是加速 `I/O` 请求的最快途径，这毋庸置疑，关键问题是怎么做。

从理论上来讲，`SQLite` 其实已经提供了缓存支持，但问题在于他提供的缓存内容是裸数据：上层使用时不可避免的需要做反序列化的操作。出于对业务方调用友好和性能考虑，最好的做法自然是在 `I/O` 读取 `Value` 时自动将 `NSData` 转换为对应的 `Model`。然而数据库并不知道怎么做反序列化，存储的 `NSData` 到底是 `JSON`，`ProtoBuf`，`XML`还是别的什么，这一切都只有上层应用知晓。

那么这里的处理就可以变成通过数据库对象向外提供注入序列化和反序列化方法的接口，由业务方自定义转换过程。

```objc

typedef NSData *(^Serializer)(NSString *key, id object);
typedef id(^Deserializer)(NSString *key, NSData *data);


@interface KeyValueStore : NSObject
@property (nonatomic,copy)  Serializer  serializer;
@property (nonatomic,copy)  Deserializer deserializer;
@end


```

当数据库从表中根据 `Key` 获取到对应 `NSData` 对象后，调用自定义 `Deserializer` 方法进行反序列化，同时缓存解析完毕的 `Model` 到内存。存储过程也是同样，通过自定义的 `Serializer` 方法，将传入对象按照自定义规则转换为 `NSData` 并最终持久化。



## 元数据支持

当我们进行存储时，除去原始数据外，往往会需要存储部分元数据，即 `metadata`。举个例子，我们从服务器上拿到一个 `JSON` 对象，随后将它放入 `Value` 字段。但我们往往还需要存储一些额外信息，比如当前对象是什么时候下载，当前对象是什么时候创建的，当前对象是什么时候过期等，这些数据和 `JSON` 对象有关系，却不能强行填入到原 `JSON` 对象中，最好的方式是使用额外的 `metadata` 字段进行存储。

这样我们的数据库表就可以表示为

```sql

CREATE TABLE IF NOT EXISTS dbname (key TEXT PRIMARY KEY,value BLOB,metadata BLOB)

```

而我们的存储接口也从 `setObject:forKey:` 变成了 `setObject:forKey:metadata:`。


## 分组支持

在某些复杂情况下，单一 `Key` 并不能很好支撑复杂需求。比如我们使用 `KeyValueStore` 做朋友圈动态存储，当我们拿到一个 `Key = 1000` 时，我们怎么区别这到底是动态 `Id` 还是评论 `Id` 呢？也就是所谓的 `Key Conflict`。常见的处理方法有三种：

* 分多个数据库文件存储 
* 为 `Key` 添加前缀/后缀  (`"feed_id_1000"` vs `"comment_id_1000"`)
* 使用复合 `Key`，即支持分组

第一种过于繁琐，为了完成简单存储需求引入过多零散文件，而第二种则略显原始，同时将实现细节暴露给上层：上层需要关心如何组装。最好的办法自然使用数据库 `unique` 特性，直接加入分组：

```sql

CREATE TABLE IF NOT EXISTS dbname (rowid INTEGER PRIMARY KEY,bucket TEXT, key TEXT ,value BLOB,metadata BLOB,UNIQUE(bucket,key))

```

那么我们的存储接口也从 `setObject:forKey:metadata:` 变成了 `setObject:forKey:metadata:inBucket:`。

上面提到的自定义序列化和反序列化过程也需要做相应的调整，变成如下定义

```objc

typedef NSData *(^Serializer)(NSString *bucket,NSString *key, id object);
typedef id(^Deserializer)(NSString *bucket,NSString *key, NSData *data);

```

还需要注意的是这里我们引入了 `rowid` 这个 `integer` 型作为自增主键，是为了下面在加入索引支持时能够更加好的支持查询。

## 索引支持

这是限制 `KeyValueStore` 使用场景的的最大因素。前面提到的场景中，`KeyValueStore` 可以获取单条动态或评论的信息，却没办法支持复杂查询：动态流场景往往需要以时间为锚点，向上或向下翻页。


解决方法自然是添加业务方所需要的索引，同时尽量不增加复杂度。我们可以在建立主表的同时建立一个索引表，专门用于支持查询。为支持这种行为，我们同样需要注入由外界提供的生成索引内容的接口。


```objc

typedef NSDictionary*(^KeyValueStoreIndexBlock)(NSString *collection, NSString *key, id object);

```

这样整个存储过程就变成

* `KeyValueStore` 初始化
* 传入索引表 `scheme` 信息 （索引名，索引数据类型集合）
* 建立主表，内容为 `rowId`，`key`，`value`，`metadata`
* 建立索引表，主键为主表的 `rowId`，其他列为前面传入的索引信息
* 写入数据
	* 调用 `setObject:forKey:inBucket` 写入数据
	* 调用 `KeyValueStoreIndexBlock` 获取当前对象的索引信息 `NSDictionary`
	* 将 `NSDictioanry` 按照 `key` 和 `column` 一一对应的规则写入索引表

而整个查询过程则可以简单的变成调用如下方法

```objc
- (NSArray *)query:(NSString *)condition;
```

但传入 `condition = "where timetag > 0"` 时 `KeyValueStore` 自动将对应请求转换为可执行的 `SQL` 语句，并从索引表中获取对应动态对应的 `rowId` 列表，再通过 `rowId` 在主表中进行反查出所有的信息。


# 结论

如上就是对一个简易 `KeyValueStore` 进行拓展的方案：内建缓存，添加分组，元数据，索引支持，经过以上拓展，相信可以覆盖 `99%` 以上的 `iOS App` 存储需求。




