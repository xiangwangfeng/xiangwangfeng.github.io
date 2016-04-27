---
layout: post
title:  Program received signal “SIGPIPE”
---

今天按照羊绒童鞋报的流程进行一个必崩bug的重现，发现进行相应操作后程序直接崩在了main方法中，只有一个提示：Program received signal "SIGPIPE"。Log中输出的是socket error为32，即EPIPE。可能的原因是程序在在往一个Socket写数据时却发现这个连接已经被远端关闭。在羊绒童鞋报的流程中提到了需要将ipad休眠，所以基本符合这个猜想：当ipad程序退到后台后，很快就将分配不到任何CPU，而服务器则有可能在这种情况将关闭连接。这之后客户端进行Socket的写操作就会产生这个信号。

Google了一番，发现：一般命令行的程序对于SIGPIPE的处理是直接终止当前进程。而对于多线程程序而言，忽略它应该是一个比较正确的处理。于是对于这个问题可以有如下两种处理方法（任选其一即可）

* 设置SIGPIPE的handler为SIG_IGN，即忽略该信号：
signal(SIGPIPE,SIG_IGN)
* 设置当前socket在进行写操作时不产生SIGPIPE。
int set = 1;
setsockopt(sd, SOL_SOCKET, SO_NOSIGPIPE, (void *)&set, sizeof(int)); 

这样做的好处在于：在某些情况 下我们并不需要一个全局的SIGPIPE handler。但是据说这种并不通用，在linux下没有相应的定义—-但我在mac下测试通过。
参考资料:

* [sigpipe-broken-pipe][1]
* [how to prevent sigpipes or handle them properly][2]


  [1]: http://stackoverflow.com/questions/6824265/sigpipe-broken-pipe
  [2]: http://stackoverflow.com/questions/108183/how-to-prevent-sigpipes-or-handle-them-properly