---
layout: post
title:  NSError NSLocalizedDescription 自动生成
---

# 现状

在 iOS 中我们常常会使用 `NSError` 来封装错误信息，相比于单纯的错误码，`NSError` 包含更多的信息。主要有

* domain 错误发生域
* code 错误码
* userInfo 详细信息

但这也意味着它的使用更繁琐些，一个简单的 `NSError` 初始化方法往往是这样

```objc
NSError *error = [NSError errorWithDomain:NIMLocalErrorDomain
                                         code:NIMLocalErrorCodeInvalidParam
                                     userInfo:nil];
```

这里我们定义了自己的字符串常量 `NIMLocalErrorDomain` 来表示错误发生域和 `NIMLocalErrorCode` 枚举表示对应的错误码。但一个懒得要死的程序员显然对这种繁琐的初始化方法不感冒，然后就引入了宏定义使自己少写几行代码：

```objc
#define NIMLocalError(x)         ([NSError errorWithDomain:NIMLocalErrorDomain \
                                                      code:(x) \
                                                  userInfo:nil])
```

这样我们就可以使用 `NSError *error = NIMLocalError(NIMLocalErrorCodeInvalidParam)` 这种稍微简单的写法。

这是前几天我开脑洞之前云信这边的 `NSError` 的标准写法。


# 问题

但是这样写法有个比较大的问题，当上层开发或者我们的客户在开发过程中碰到错误后，他只能看到 `NSError` 的 `domain` 和 `code` 信息（这也是我们在使用苹果某些 API 时常常会碰到的问题，只有 `doamin`，`code` 和 `the operation could not be completed` 的描述），于是我们只能拿着 `domain` 和 `code` 去 google 一把或者查阅文档来确定真正的错误信息。

一种改进的方法是调整宏定义的写法，将宏定义调整为

```objc
#define NIMLocalError(code,reason) ([NSError errorWithDomain:NIMLocalErrorDomain \
                                                        code:(code) \
                                                    userInfo:@{NSLocalizedDescriptionKey : (reason)}])
```

这样我们就可以调整为 `NSError *error = NIMLocalError(NIMLocalErrorCodeInvalidParam,@"参数错误")`。但这样的做法仍有两个问题

* 不同文件中相同错误需要重复填入相同的描述信息。
* 对于服务器只下发错误码而没有描述信息的情况无法处理。

# 解决方案

一种简单解决方案是将错误码和错误信息的映射关系打包到一个 `plist` 文件或是写死在源码中,后续通过错误码反查描述信息并填充 `NSError`。大致的代码如下

```objc
- (NSError *)error:(NSString *)domain
              code:(NSInteger)code
{
    NSString *message = [self.messages objectForKey:@(code)];
    return message ? [NSError errorWithDomain:domain
                                   code:code
                               userInfo:@{NSLocalizedDescriptionKey : message}] :
                     [NSError errorWithDomain:domain
                                   code:code
                               userInfo:nil];          
}
```

接下来就是苦力活：每添加一个错误码就需要同时在描述文件中添加对应的描述信息。

`而实际情况是：所有错误码枚举，我们都已添加对应的注释，也就是错误码的描述信息。那么问题就来了：为什么我们还需要单独维护一份描述文件呢，直接使用注释信息不就好了么？`

换而言之，我们为什么不直接把枚举定义的头文件当成一种描述文件，直接从中提取注释作为错误信息呢？似乎这是非常困难的事情，需要引入 Objective-C 语法解析库，进而提取语法树。而事实上，由于使用 `VVDocument` 或是 Xcode 自带生成注释方法生成的枚举注释格式非常单一，直接通过简单的行扫描就可以完成。

### 解析过程

通过 `VVDocument` 生成的注释格式如下

```objc
/**
  *  注释内容
 */
 ```
那么我们只需要检查当前行在 `trim` 后发现当前行是以 `* `开头即可确定当前行为注释行，然后提取即可。
而对于枚举表达式，由于 Objective-C 的特殊性，所有枚举往往都会有自己的固定前缀，如
 
 ```objc
 typedef NS_ENUM(NSInteger, NIMRemoteErrorCode) {
    /**
     *  客户端版本错误
     */
    NIMRemoteErrorCodeInvalidVersion      = 201,
    /**
     *  密码错误
     */
    NIMRemoteErrorCodeInvalidPass         = 302,
    /**
     *  CheckSum校验失败
     */
    NIMRemoteErrorCodeInvalidCRC          = 402,
    /**
     *  非法操作或没有权限
     */
    NIMRemoteErrorCodeForbidden           = 403,
}
```

那我们就可以使用前缀进行匹配提取。具体实现如下

```swift
func parseComments(items : [String],domain :String,prefix : String) -> [String:String] {
    var results = [String:String]()
    
    var comment : String? = nil
    for item in items {
        if item.contains(prefix) {
            let exp = item.replacingOccurrences(of: ",", with: " ")
            let tokens = exp.components(separatedBy: "=")

            if tokens.count == 2 {
                let enumName = tokens[0].replacingOccurrences(of: " ", with: "")
                let enumValue = tokens[1].replacingOccurrences(of: " ", with: "")
                if let value = comment , let _ = Int(enumValue) {
                    results[enumName] = value
                    let key = domain + "." + enumValue
                    results[key] = enumName
                }
            }
        }
        else if item.trimmingCharacters(in: .whitespaces).hasPrefix("* ") {
            let word = item.replacingOccurrences(of: "*", with: "").replacingOccurrences(of: " ", with: "")
            comment = word
        }
    }
    return results
}
```
通过简单的匹配（都不需要使用正则，跑）我们就获取了对应关系。最后通过在 Xcode 中设置相应的 `RunScript` 就可以使得生成描述源码的过程优先在编译前执行。

![](/images/xcrunscript.jpg)

如图，至此整个流程就已完整：我们通过 `nim_error_generator` 读取并解析 `NIMGlobalDefs.h` 和 `NIMAVChatDefs.h` 中相应枚举信息，并最终将映射关系写入 `NIMErrorManager.mm`，后者在生成 `NSError` 的时候会根据传入的 `code` 和 `domain` 查找对应的描述信息，并自动填入到 `userinfo`。




