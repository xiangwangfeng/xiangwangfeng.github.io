---
layout: post
title:  构造HttpClient三部曲之一：支持代理的Socket封装
---

最近在重构闪印的网络部分的代码——原因不外乎当时写的时候太着急，很多东西都没有好好的封装，将就着能用就行了。而挪到Ebox中为了传说中的平台无关，像FileStream，Socket之类的用了大量JUCE的封装，觉得异常恶心，而由于种种原因目前很清闲，于是开始重构这部分的代码(实际上是自己从头写起……)，目前也整了个大概了，于是可以写写总结：《构造HttpClient三部曲》。

话说工欲善其事，必先利其器，而构造HttpClient的器就是可支持代理的Socket封装类——当初自己写HttpClient而不是使用WinInet之类的库很大部分原因就是WinInet只支持Http代理而不支持Socks的代理，所以只能自己动手丰衣足食了。(话说很奇怪微软的很多东西都是这样只支持HTTP代理，比如C#中比较有名的ProxySocket 都是别人写的)而C++方面貌似也没有支持代理的HTTP库，倒是C方面有个curl，如果自己懒得封装Http倒是可以拿来用。

言归正传，说ProxySocket的封装。大体来说代理服务器分为三种：http,ftp和socks，更有透明代理(比如公司的翻墙代理就是种透明代理)和非透明代理之分。而这里我只关心http和socks这两种比较常见的非透明代理，至于ftp代理貌似只能从文献中听闻了，基本很少有什么服务器提供这样的代理。当机器通过代理服务器上网时，整个通讯过程分为两个部分：机器和代理通讯，代理和目的地址通讯，这样一来客户端需要关心的就只有：和代理服务器完成一次对传输协议协商握手过程，在这之后就可以把代理服务器看成目标地址了。

##HTTP代理

当客户端连接上一个HTTP代理服务器后并通过它发送请求，代理服务器做的事情就是：建立和目标地址的连接，发送请求，接受反馈并将反馈发回客户端。为了实现这一点，在HTTP协议中规定了这么一个特殊方法：CONNECT。当客户端和HTTP代理服务器连接后，只需要发送如下格式的HTTP请求即可：

CONNECT <destanation_address> : <destanition_port> <http_version><CR><LF>          
Host: <destanation_address> : <destanition_port><CR><LF> 
<optional_header_line><CR><LF> 
<optional_header_line><CR><LF> 
…… 
<CR><LF>而如果代理需要验证用户名密码则需要将"用户名：密码"进行Base64编码后填入：
CONNECT <destanation_address> : <destanition_port> <http_version><CR><LF>          
Host: <destanation_address> : <destanition_port><CR><LF> 
Authorization: Basic <base64后的验证字段><CR><LF> 
Proxy-Authorization: <Basic base64后的验证字段><CR><LF> 
<optional_header_line><CR><LF> 
…… 
<CR><LF>


{% highlight C++ %}
char buff[kmax_file_buffer_size] = {0};
if (!_proxy_config._username.empty())
{
	std::string auth = _proxy_config._username + ":" + _proxy_config._password;
	std::string base64_encode_auth;
	Util::base64Encode(auth,base64_encode_auth);
	sprintf_s(buff,kmax_file_buffer_size,"CONNECT %s:%d HTTP/1.1rnHost: %s:%drnAuthorization: Basic %srnProxy-Authorization: Basic %srnrn",
	_host_name.c_str(),_port_number,_host_name.c_str(),_port_number,base64_encode_auth.c_str(),base64_encode_auth.c_str());
}
else
{
	sprintf_s(buff,kmax_file_buffer_size,"CONNECT %s:%d HTTP/1.1rnHost: %s:%drnrn",
	_host_name.c_str(),_port_number,_host_name.c_str(),_port_number);
}
//发送HTTP代理连接请求
bool send_connect_request = _socket.writeAll(buff,strlen(buff));
if (!send_connect_request)
{
	return false;
}
//获得HTTP代理回复
int ret = _socket.read(buff,sizeof(buff));
if (ret <= 0)
{
	return false;
}
buff[ret] = '';
Util::makeLower(buff,strlen(buff));
return strstr(buff, "200 connection established") != 0;
{% endhighlight %}


而在请求后如果收到正确反馈即表示代理连接成功。(一般就是code = 200)这里值得说明的一点是：上述代码只是对Basic这种验证方式做了处理，这种明文传输的形式是很不安全的。当然还有一种验证方式是NTLM，相对而言比较复杂，不赘述。(实际上是没空去看…)
##Socks4代理
Socks4的代理连接请求相比Http代理要简单不少，而且Socks4是不支持用户验证的。整个Socks4的请求就包含5个字段而已：

* 字段一：Socks版本号，即0×04，占一个字节。
* 字段二：命令码，占一个字节，其中0×01：TCP/IP连接，而0×02：端口绑定。
* 字段三：网络字节序端口，占两个字节。
* 字段四：网络字节序IP地址，占四个字节。
* 字段五：用户ID字段，可变，以null(0)结尾。
而服务器返回的反馈则更加简单，一共包含4个字段：
* 字段一：一个空字节
* 字段二：一个字节，表示反馈状态码，其中0x5A(即90)表示请求被接受。
* 字段三：两个字节，可被忽略
* 字段四：四个字节，可被忽略

可以看出实际上整个Sock4的请求和反馈都是异常简单：SOCK4的反馈甚至只有一个字节是有意义的，很轻松就可以搞定：
{% highlight C++ %}
//Socks4没有用户密码验证
struct Sock4Reqeust
{
	char VN;
	char CD;
	unsigned short port;
	unsigned long ip_address;
	char other[256]; // 变长
} sock4_request;
struct Sock4Reply
{
	char VN;
	char CD;
	unsigned short port;
	unsigned long ip_address;
} sock4_reply;
sock4_request.VN = 0x04; // VN是SOCK版本，应该是4；
sock4_request.CD = 0x01; // CD是SOCK的命令码，1表示CONNECT请求，2表示BIND请求；
sock4_request.port= ntohs(_port_number);
sock4_request.ip_address = SocketHelper::getIntAddress(_host_name.c_str());
sock4_request.other[0] = '';
if (sock4_request.ip_address == INADDR_NONE)
return false;
//发送SOCKS4连接请求
bool send_sock4_requst =  _socket.writeAll((char*)&sock4_request,9);          
if (!send_sock4_requst)
{
	return false;
}
//获得Socks4代理的回复
int ret = _socket.read((char *)&sock4_reply, sizeof(sock4_reply));
if (ret <= 0)
{          
	return false;
}
/*
CD是代理服务器答复，有几种可能：
90，请求得到允许；
91，请求被拒绝或失败；
92，由于SOCKS服务器无法连接到客户端的identd（一个验证身份的进程），请求被拒绝；
93，由于客户端程序与identd报告的用户身份不同，连接被拒绝。
*/
return sock4_reply.CD == 90;
{% endhighlight %}
值得注意的是由于代码中是用结构体表示相应的字段集合，要考虑到字节对齐对结构体字段排放的影响需要指定**#pragma pack(1)**。强制以一个字节对齐。Socks4还有一种变种叫做Socks4a，具体就不介绍了，可以参看这里。

##Socks5代理

Socks5是Socks4的一个升级版本，增加了很多Socks4不支持的特性，比如对IPv6的支持。一个完整的Socks5代理握手协议可以分为如下五个步骤：

* 客户端发起握手请求，参数为可支持的验证方法列表 
* 服务器选择一种验证方式并返回(如果无可支持的验证方式直接返回失败)
* 根据所选验证方式，客户端和服务端进行验证方式上的协商(比如选择了用户名/密码这种方式进行验证，客户端就需要发送用户和密码过去，然后服务器进行验证并返回结果)
* 客户端发起一个类似于Socks4的请求
* 服务端返回一个类似Socks4的反馈
这样经过五个步骤，客户端和服务端的握手就算完成了，接下来就是直接进行数据传输即可。
       Socks5可支持的验证方式包括：
       0×00：无验证
       0×01：GSSAPI
       0×02：用户名/密码
       0×03-0x7F：IANA指定的方法
       0×80-0xFE：保留方法
对于我们这个简单的ProxySocket来说完全可以只关心0×00和0×02即可，其他几种方式一来不常见，二来也没有太大的实现必要。
客户端发起的第一个请求总共包括三个字段：

* 字段1:Socks版本号，即0×05
* 字段2:支持的验证方式数，一个字节
* 字段3:验证方法列表，变长，一个字节表示一个方法

服务端在收到这个请求后，返回一个验证方法的反馈，一共就两个字段：

* 字段1：Socks版本号，一个字节，即0×05
* 字段2：选择的验证方法，一个字节，如果在客户端发来的验证方法列表中没有服务端支持的方法，则返回0xFF

这个时候客户端可以根据服务端饭回来的验证方法进行验证：0×00则直接跳过第三步。一个典型的用户名/密码验证请求如下：

* 字段1：版本号，一个字节，必须指定为0×01
* 字段2：用户名长度，一个字节
* 字段3：用户名，变长
* 字段4：密码长度，一个字节
* 字段5：密码，变长
服务端返回如下的反馈：

* 字段1：版本号，一个字节 (这个其实可以忽略)
* 字段2：状态码，一个字节，0×00表示成功，其他值则表示验证失败，连接需要断开。
这以后的工作就基本和Sock4相似了，客户端发起一个连接请求：

* 字段1：Socks版本号，一个字节，即0×05
* 字段2：命令码，一个字节，0×01：TCP/IP连接，0×02：TCP/IP端口绑定，0×03：关联UDP端口
* 字段3：保留字段，一个字节，必须指定为0×00
* 字段4：地址类型，一个字节，0×01：IPv4地址，0×03：域名，0×04：IPv6地址
* 字段5：目标地址：4字节表示的IPv4地址，或者16字节表示的IPv6地址，如果是域名则是变长：一字节表示域名长度，紧随其后的是域名
* 字段6：网络序的端口号，两个字节
服务器的反馈为：

* 字段1：Sock协议版本，一个字节，必须为0×05
* 字段2：状态码，0×00表示请求成功，其余的状态码可参考相应的RFC文档
* 字段3：保留字段，一个字节，必须制定为0×00
* 字段4：地址类型，一个字节，0×01：IPv4地址，0×03：域名，0×04：IPv6地址
* 字段5：目标地址，4字节表示的IPv4地址，或者16字节表示的IPv6地址，如果是域名则是变长：一字节表示域名长度，紧随其后的是域名
* 字段6：网络序的端口号，两个字节
这样一个完整的Socks5握手协议就算完成了，但是鉴于代码篇幅太长了，这里就不上了，等整个HttpClient介绍完毕后再统一上代码…….（实际上是…某些地方的代码还没整清爽，无颜见公婆啊）

##参考资料
1.Wiki中关于Sock5的介绍：http://en.wikipedia.org/wiki/SOCKS
2.《Http Tunneling》：http://www.codeproject.com/KB/IP/httptunneling.aspx
3.《CAsyncProxySocket – CAsyncSocket derived class to connect through proxies》：http://www.codeproject.com/KB/IP/casyncproxysocket.aspx




