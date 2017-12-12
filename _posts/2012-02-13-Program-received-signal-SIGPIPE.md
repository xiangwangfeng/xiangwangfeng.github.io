---
layout: post
title:  Program received signal “SIGPIPE”
---

今天按照羊绒童鞋报的流程进行一个必崩 bug 的重现，发现进行相应操作后程序直接崩在了 main 方法中，只有一个提示：Program received signal "SIGPIPE"。Log 中输出的是 socket error 为 32，即 EPIPE。可能的原因是程序在在往一个 Socket 写数据时却发现这个连接已经被远端关闭。在羊绒童鞋报的流程中提到了需要将 ipad 休眠，所以基本符合这个猜想：当 ipad 程序退到后台后，很快就将分配不到任何 CPU，而服务器则有可能在这种情况将关闭连接。这之后客户端进行 Socket 的写操作就会产生这个信号。

Google 了一番，发现：一般命令行的程序对于 SIGPIPE 的处理是直接终止当前进程。而对于多线程程序而言，忽略它应该是一个比较正确的处理。于是对于这个问题可以有如下两种处理方法（任选其一即可）

* 设置 SIGPIPE 的 handler 为 SIG_IGN，即忽略该信号：
signal(SIGPIPE,SIG_IGN)
* 设置当前 socket 在进行写操作时不产生 SIGPIPE。
int set = 1;
setsockopt(sd, SOL_SOCKET, SO_NOSIGPIPE, (void *)&set, sizeof(int)); 

这样做的好处在于：在某些情况 下我们并不需要一个全局的 SIGPIPE handler。但是据说这种并不通用，在 linux 下没有相应的定义—- 但我在 mac 下测试通过。
参考资料:

* [sigpipe-broken-pipe][1]
* [how to prevent sigpipes or handle them properly][2]


  [1]: http://stackoverflow.com/questions/6824265/sigpipe-broken-pipe
  [2]: http://stackoverflow.com/questions/108183/how-to-prevent-sigpipes-or-handle-them-properly