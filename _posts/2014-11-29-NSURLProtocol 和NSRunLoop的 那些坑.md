---
layout: post
title:  NSURLProtocol 和 NSRunLoop 的那些坑
---


最近用 AFNetworking 替换掉了工程里的 ASIHttpRequest，结果陆续碰到很多问题：

* 如何统一地添加全局的 HTTP 头(不仅仅是 UA 而已)
* 如何优雅地进行流量统计
* 对特定的地址进行 CDN 加速( URL 到 IP 的替换)
* 怎么实现 HTTP 的同步请求

前三个需求对于 ASIHttpReqeust 来说都不是问题，只需要在几个统一的点进行修改即可。而使用 AFNetworking 后就没有那么容易了：一方面 AFNetworking 中生成 NSURLRequest 的点比较多，并没有一个统一的路径。其次工程中会有部分直接使用 NSURLConnecion 的场景，无法统一。经 [cyzju](http://msching.github.io/) 提醒发现了 NSURLProtocol 这个大杀器，可惜对应的文档过于简略，唯一比较详细的介绍就只有 [RW 的这篇教程](http://www.raywenderlich.com/59982/nsurlprotocol-tutorial) 而已，掉了很多坑，值得记上一笔。

# NSURLProtocol

## 概念


NSURLProtocol 也是苹果众多黑魔法中的一种，使用它可以轻松地重定义整个 URL Loading System。当你注册自定义 NSURLProtocol 后，就有机会对所有的请求进行统一的处理，基于这一点它可以让你

* 自定义请求和响应
* 提供自定义的全局缓存支持
* 重定向网络请求
* 提供 HTTP Mocking (方便前期测试)
* 其他一些全局的网络请求修改需求

## 使用方法

### 继承 NSURLPorotocl，并注册你的 NSURLProtocol

{% highlight objc %}
[NSURLProtocol registerClass:[YXURLProtocol class]];
{% endhighlight %}

当 NSURLConnection 准备发起请求时，它会遍历所有已注册的 NSURLProtocol，询问它们能否处理当前请求。所以你需要尽早注册这个 Protocol。

### 实现 NSURLProtocol 的相关方法

当遍历到我们自定义的 NSURLProtocol 时，系统先会调用 canInitWithRequest: 这个方法。顾名思义，这是整个流程的入口，只有这个方法返回 YES 我们才能够继续后续的处理。我们可以在这个方法的实现里面进行请求的过滤，筛选出需要进行处理的请求。
{% highlight objc %}
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
    if ([NSURLProtocol propertyForKey:YXURLProtocolHandled inRequest:request])
    {
        return NO;
    }
    NSString *scheme = [[request URL] scheme];
    NSDictionary *dict = [request allHTTPHeaderFields];
    return [dict objectForKey:@"custom_header"] == nil &&
    ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
     [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame);
}
{% endhighlight %}

当筛选出需要处理的请求后，就可以进行后续的处理，需要至少实现如下 4 个方法

{% highlight objc %}
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request
{
    return request;
}

+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a
                       toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}


- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    [YXURLProtocol applyCustomHeaders:mutableReqeust];
    [NSURLProtocol setProperty:@(YES)
                        forKey:YXURLProtocolHandled
                     inRequest:mutableReqeust];
    
    self.connection = [NSURLConnection connectionWithRequest:mutableReqeust
                                                    delegate:self];
}

- (void)stopLoading
{
    [self.connection cancel];
    self.connection = nil;
}
{% endhighlight %}

* canonicalRequestForRequest: 返回规范化后的 request , 一般就只是返回当前 request 即可。
* requestIsCacheEquivalent:toRequest: 用于判断你的自定义 reqeust 是否相同，这里返回默认实现即可。它的主要应用场景是某些直接使用缓存而非再次请求网络的地方。
* startLoading 和 stopLoading 实现请求和取消流程。

### 实现 NSURLConnectionDelegate 和 NSURLConnectionDataDelegate

因为在第二步中我们接管了整个请求过程，所以需要实现相应的协议并使用 NSURLProtocolClient 将消息回传给 URL Loading System。在我们的场景中推荐实现所有协议。
{% highlight objc %}

- (void)connection:(NSURLConnection *)connection
  didFailWithError:(NSError *)error
{
    [self.client URLProtocol:self
            didFailWithError:error];
}

- (NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(NSURLResponse *)response
{
    if (response != nil) 
    {
        [[self client] URLProtocol:self wasRedirectedToRequest:request redirectResponse:response];
    }
    return request;
}

- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection
{
    return YES;
}

- (void)connection:(NSURLConnection *)connection
didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
{
    [self.client URLProtocol:self
didReceiveAuthenticationChallenge:challenge];
}

- (void)connection:(NSURLConnection *)connection
didCancelAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge 
{
    [self.client URLProtocol:self
didCancelAuthenticationChallenge:challenge];
}


- (void)connection:(NSURLConnection *)connection
didReceiveResponse:(NSURLResponse *)response
{
    [self.client URLProtocol:self
          didReceiveResponse:response
          cacheStoragePolicy:NSURLCacheStorageAllowed];
}

- (void)connection:(NSURLConnection *)connection
    didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self
                 didLoadData:data];
}

- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse
{
    return cachedResponse;
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection
{
    [self.client URLProtocolDidFinishLoading:self];
}
{% endhighlight %}
在每个 delgate 的实现中我都刨去了工程中的特定实现(流量统计)，只保留了需要实现的最小 Protocol 集合。

## NSURLProtocol 那些坑

从上面的介绍来看，NSURLProtocol 还是比较简单，但是实际使用的过程中却容易掉进各种坑，一方面是文档不够详尽，另一方面也是对于苹果这套 URL Loading Sytem 并不熟悉，不能将整个调用过程有机地统一。

* 坑 1：企图在 canonicalRequestForRequest: 进行 request 的自定义操作，导致各种递归调用导致连接超时。这个 API 的表述其实很暧昧:

**It is up to each concrete protocol implementation to define what “canonical” means. A protocol should guarantee that the same input request always yields the same canonical form.**

所谓的 canonical form 到底是什么呢？而围观了包括 [NSEtcHosts](https://github.com/mattt/NSEtcHosts) 和 [RNCachingURLProtocol](https://github.com/rnapier/RNCachingURLProtocol) 在内的实现，它们都是直接返回当前 request。在这个方法内进行 request 的修改非常容易导致递归调用(即使通过 setProperty:forKey:inRequest: 对请求打了标记)

* 坑 2：没有实现足够的回调方法导致各种奇葩问题。如 connection:willSendRequest:redirectResponse:
内如果没有通过 [self client] 回传消息，那么需要重定向的网页就会出现问题: host 不对或者造成跨域调用导致资源无法加载。


# 同步 AFNetworking 请求

虽然 Mattt 各种鄙视同步做网络请求，但是我们不可否认某些场景下使用同步调用会带来不少便利。一种比较简单的实现是使用信号量做同步：
{% highlight objc %}
@implementation AFHTTPRequestOperation (YX)
- (void)yxStartSynchronous
{
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
        dispatch_semaphore_signal(semaphore);
    } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        dispatch_semaphore_signal(semaphore);
    }];
    [self start];
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
}
@end
{% endhighlight %}
但是这样带来的问题是在 UI 线程调用同步请求就会导致线程堵死崩溃(好吧，就不应该允许 UI 线 程上这么做)。一种改进的方法是使用 NSRunLoop

即：
{% highlight objc %}
    while (_shouldBlock)
    {
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                                 beforeDate:[NSDate laterDate]];
   }
{% endhighlight %}


但是这种写法是有 ** 大坑 ** 的：如果当前 NSRunLoop 并没有任何 NSTimer 或 Input Source，runMode:beforeDate: 方法将立刻返回 NO，于是造成死循环，占用大量 CPU，进而导致 NSURLConnection 请求超时。 规避的方法是往 RunLoop 中添加 NSTimer 或者空 NSPort 使得 NSRunLoop 挂起而不占用 CPU。(ASIHttpRequest 则是在当前 RunLoop 中添加了 0.25 秒触发一次的刷新 Timer)

**If no input sources or timers are attached to the run loop, this method exits immediately and returns NO; otherwise, it returns after either the first input source is processed or limitDate is reached. Manually removing all known input sources and timers from the run loop does not guarantee that the run loop will exit immediately. OS X may install and remove additional input sources as needed to process requests targeted at the receiver’s thread. Those sources could therefore prevent the run loop from exiting.**







