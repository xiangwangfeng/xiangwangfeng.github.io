---
layout: post
title:  构造 HttpClient 三部曲之二：GET 方法实现
---


继续 HttpClient 构造的博文，第二篇：GET 方法的实现。HTTP 协议定义了和服务器交互的不同方法，包括 GET,POST,PUT,DELETE,CONNECT 等等，其中最常用的两个方法就是 GET 和 POST。这篇先讲讲 GET 方法的一些细节。

HTTP 协议的交互主要由请求和响应组成：客户端发起请求，服务端返回响应。而一个简单的 HTTP 请求又可以分成信息头和信息体。但对于 GET 来说，它的请求只有 HTTP 消息头而已。


## HTTP 请求之 GET

一个最简单的 HTTP GET 请求可以写成：
![][1]
而复杂的请求往往会加入很多的请求头域，如：
> GET /logos/2011/hargreaves11-hp-15.jpg HTTP/1.1 
Host: www.google.com.hk 
Connection: keep-alive 
Referer: http://www.google.com.hk/ 
User-Agent: Mozilla/5.0 (Windows NT 6.1) AppleWebKit/534.24 (KHTML, like Gecko) Chrome/11.0.696.60 Safari/534.24 
Accept: */* 
Accept-Encoding: gzip,deflate,sdch 
Accept-Language: zh-CN,zh;q=0.8 
Accept-Charset: GBK,utf-8;q=0.7,*;q=0.3

每一个头域都由一个域名，冒号 (：) 和域值组成。域名是大小写无关，而域值前面可以添加数个空格。之所以将前面那段字特地高亮是因为碰到很多 “不太规范” 的 HTTP Server 返回的域名经常是不 “正规” 的：比如将 Content-Type 写成 Content-type—- 这只是从视觉上不太美观，但是绝对是符合规范的，所以客户端在解析的时候特别要注意忽略大小写的影响。

一些典型的头域有：

* Host：指定请求资源的主机和端口
* Connection：指定连接状态，HTTP1.1 和 1.0 协议的一个很大区别就是 HTTP1.1 可以保持连接，HTTP1.1 默认是持久连接，一次连接可以发起多个请求。
* User-Agent：包含发出请求的用户信息，多用于系统和浏览器检测。关于它的八卦可以参考这里。
* Referer：请求 URI 的源地址。一般可用于反盗链。
* Accept：可接受的文件类型
当然我们也可以往 HTTP 头中塞入一些自定义的头域，这样的效果和在 URL 添加请求参数的效果是一样的。


## HTTP 响应


无论是 GET 方法还是 POST 方法，HTTP 的响应都是一致的：一个 HTTP 消息头和一个 HTTP 消息体。在 HTTP 消息头的第一行指定了：HTTP 版本号，HTTP 响应码和详细消息。而接下来就是一个个头域，直到接受到两个 \ r\n 为止。一个典型的 HTTP 相应的消息头如下：
> HTTP/1.1 200 OK
Cache-Control: private, max-age=30
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Expires: Mon, 25 May 2009 03:20:33 GMT
Last-Modified: Mon, 25 May 2009 03:20:03 GMT
Vary: Accept-Encoding
Server: Microsoft-IIS/7.0
X-AspNet-Version: 2.0.50727
X-Powered-By: ASP.NET
Date: Mon, 25 May 2009 03:20:02 GMT
Content-Length: 12173

对于客户端来说需要着重关注的头域一般只有 Content-Length 和 Transfer-Encoding 而已。如果返回的 HTTP 消息头中有 Content-Length 这个头域则说明 HTTP 体是定长的，客户端只需要继续接受 Content-Length 指定的数据长度的内容即可。而如果服务器在返回数据给客户端时是一边生成数据一边发送，那么很可能就会采用 chunked 的形式，将 Transfer-Encoding 指定为 chunked。这样客户端对 HTTP 消息体的处理就会稍显复杂。
在 HTTP 协议的 RFC 中对 chunk 传输的定义如下：
{% highlight C++ %}
Chunked-Body   = *chunk 
                        last-chunk 
                        trailer 
                        CRLF 
       chunk          = chunk-size [ chunk-extension ] CRLF 
                        chunk-data CRLF 
       chunk-size     = 1*HEX 
       last-chunk     = 1*("0") [ chunk-extension ] CRLF 
       chunk-extension= *( ";" chunk-ext-name [ "=" chunk-ext-val ] )            
       chunk-ext-name = token 
       chunk-ext-val  = token | quoted-string 
       chunk-data     = chunk-size(OCTET) 
       trailer        = *(entity-header CRLF)
{% endhighlight %}
简单来说就是：一个 chunked 编码的 body 可以分为四个部分：一系列的 chunk 段，last chunk，trailer 和 CRLF。一个 chunk 又可以分为 chunk-size 和 chunk-data 两部分。通过解析 chunk-size(16 进制)可以获取到真正的 chunk-data 长度并进行拼接得到最后的 HTTP 体内容。RFC 中关于 chunk 的解析伪代码如下：
{% highlight c %}
  length := 0
　　read chunk-size, chunk-ext (if any) and CRLF
　　while (chunk-size > 0) {
　　read chunk-data and CRLF
　　append chunk-data to entity-body
　　length := length + chunk-size
　　read chunk-size and CRLF
　　}
　　read entity-header
　　while (entity-header not empty) {
　　append entity-header to existing header fields
　　read entity-header
　　}
　　Content-Length := length
　　Remove "chunked" from Transfer-Encoding
{% endhighlight %}
为了构造这个 HTTPClient 我也相应写了 C++ 版的处理方法，不过没有考虑 chunk-extension 和 entity-header 的处理(在我测试的几个网站来看都没有这两个值，所以即使写了也无法验证其正确性)。


[1]:/images/http_get.jpg
