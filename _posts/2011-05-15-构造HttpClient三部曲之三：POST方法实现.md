---
layout: post
title:  构造HttpClient三部曲之三：POST方法实现
---

手机新买，新鲜感未过，几乎一天都在安装试用卸载各种搞毛软件中度过，差点忘了要在这周结束掉HttpClient的博文，趁着还有3个小时才12点赶紧写完。

上一篇介绍了GET方法的实现，这篇主要就介绍介绍POST。从上层来看，GET和POST最大的区别在于GET是一种从服务器获取数据的请求，而POST是向服务器传送数据，进行站点更新。而从协议上来看，POST和GET的最大区别在于：POST请求是包含HTTP消息体的，较GET能够实现更复杂的数据交互。通过上一篇我们已经知道一个GET请求是只有HTTP消息头的，虽然它也能够带上参数：将参数和请求的URL以?分开，但是一来请求的数据量很少，二来因为请求格式的固定化导致不能有效表达一些复杂的请求，而POST则没有这样的问题了。
##HTTP请求之POST
和GET方法不同，POST有自己的HTTP请求体，这样也必然要求POST方法必须在HTTP请求头指明请求体的类型和长度：即content-type和content-length。content-type指明了Http消息体内的数据以怎样的结构进行组织，而content-length则指明了http消息体的长度。一个典型的POST请求如下：

> POST /cwf_nget?param=[] HTTP/1.1 
Host: friend.renren.com 
Connection: keep-alive 
Referer: http://friend.renren.com/ajaxproxy.htm 
Content-Length: 24 
Origin: http://friend.renren.com 
User-Agent: Mozilla/5.0 (Windows NT 6.1) AppleWebKit/534.24 (KHTML, like Gecko) Chrome/11.0.696.60 Safari/534.24 
Content-Type: application/x-www-form-urlencoded 
Accept: */* 
Accept-Encoding: gzip,deflate,sdch 
Accept-Language: zh-CN,zh;q=0.8 
Accept-Charset: GBK,utf-8;q=0.7,*;q=0.3
requestToken=-1000000000

例子中的content-type为application/x-www-form-urlencoded，即需要对HTTP体进行URL编码。而content-length为24，即后面requestToken=-1000000000的长度。当然Content-Type有很多种，比如text/plain，text/xml等等，具体的意义可以参考[这里][1] 。而唯一值得拿出来说的一种content-type是multipart/form-data的情况。顾名思义这种content-type要求HTTP体是以多段的格式发送，主要是为了能够通过HTTP协议上传文件。当然也可以只用这种编码方式来分段上传请求数据，不过不免有点脱裤子放屁之嫌。

一个典型的以multipart/form-data上传文件的Http包如下：
> POST /upload HTTP/1.1 
Host: upload.buzz.163.com 
Connection: keep-alive 
Referer: http://t.163.com/ 
Content-Length: 8993 
Cache-Control: max-age=0 
Origin: http://t.163.com 
User-Agent: Mozilla/5.0 (Windows NT 6.1) AppleWebKit/534.24 (KHTML, like Gecko) Chrome/11.0.696.60 Safari/534.24 
Content-Type: multipart/form-data; boundary=—-WebKitFormBoundaryoF81IsooBo64lBY2 
Accept: application/xml,application/xhtml+xml,text/html;q=0.9,text/plain;q=0.8,image/png,*/*;q=0.5 
Accept-Encoding: gzip,deflate,sdch 
Accept-Language: zh-CN,zh;q=0.8 
Accept-Charset: GBK,utf-8;q=0.7,*;q=0.3 
Cookie: _ntes_nuid=b243d3e6005fcc75916dcf6a05a1231d; vjuids=-115573d47.12ba116f81c.0.6e77627f; ALLYESID4=00101015004134494510218; isGd=0; isFs=0; OUTFOX_SEARCH_USER_ID=-1937482032@10.130.10.67; __utmz=187553192.1299506530.4.3.utmcsr=blog.163.com|utmccn=(referral)|utmcmd=referral|utmcct=/lgh_2002/; USERTRACK=60.186.110.113.1302880928690537; __utma=187553192.1936313005.1288621751.1301116115.1304778661.6; MAIL163_SSN=xiang.wangfeng; Province=0571; City=0571; NTES_LOGINED=true; vjlast=1286897859.1305430125.11;
——WebKitFormBoundaryoF81IsooBo64lBY2 
Content-Disposition: form-data; name="callback"
uploadImageCallback 
——WebKitFormBoundaryoF81IsooBo64lBY2 
Content-Disposition: form-data; name="watermark"
t.163.com/amao 
——WebKitFormBoundaryoF81IsooBo64lBY2 
Content-Disposition: form-data; name="Filedata"; filename="head_5685p40.jpg" 
Content-Type: image/jpeg
??.JFIF…..x.x..?C…………………. 
….. 
… 
.. 
… 
…………………….?C…….    ..    . 
. 
…………………………………………..?…d.d……….?………………………?B………… 
…………..!1A."Q..2aq?.#態?Rbr偙$4DSUct儞擦佯?……………..曘膇X.s菢t=5Y鰦.?.敠婂糩輾纽脡昍獛.h塱7鶛フ3?xA/.[B}.渀hn提挮mN笢菨扦.跍Ni?胘?祻?牛S喁膷 
;.g鉉`.逽枻%.9歩)<€?芋s    鼠狿鍸h?P竐>犚倳攧侽!Θ.俰F?!Z €u€币雽?j.頎??雙iZG%? ?堈..Ie湊趻?.r.-l?.n(#洓’b.,$? y@)虏J堧蟁訷YF攬偧I蘵嶮診    繡`|摥薫?. PG? 
(此处删去N多字节….)
——WebKitFormBoundaryoF81IsooBo64lBY2–


HTTP消息头中指明了content-type为multipart/form-data并给出了boundary。boundary的主要作用是进行分割HTTP消息体中的内容，一般的格式都是前面带几个“-”号，然后跟一堆随机字符。整个HTTP消息体以boundary+两个"-"做结尾。

关于POST相关的代码可以参考[这里][2]。


  [1]: http://www.utoronto.ca/webdocs/HTMLdocs/Book/Book-3ed/appb/mimetype.html
  [2]: http://code.google.com/p/amaoproject/source/browse/trunk/httpclient/httpclient/http_client.cpp