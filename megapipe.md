#MegaPipe: A New Programming Interface for Scalable Network I/O

_By 元东 2015210938_
_& 邓志会 2015210926_

_会议：OSDI 2012_

_作者：Sangjin Han+, Scott Marshall+, Byung-Gon Chun*, and Sylvia Ratnasamy+_

_机构：+University of California, Berkeley， *Yahoo! Research_

## 文章研究的问题
和mTCP解决的相同的问题，解决现有操作系统对短连接，短消息和多核处理不够好的问题：

1.	system call overhead
2.	shared listenng socket
3.	file abstraction

其中123，mTCP均有解决，并且23采用的技术和这篇文章不同；FastSocket也解决了23，并且更elegant。

## 论文背景
随着多核技术的发展，多核系统应用越来越广，而现有的网络API协议不能很好的利用多核拓展到对短消息和高频发连接的现实应用当中去，并且现有系统对于资源的共享而造成的多核性能瓶颈也没有给出更好的解决办法。对于一个理想的网络API来讲，不仅需要可能保持高效的执行速率，还要能够给出一个简单并且易用的程序抽象接口，并且能够支持并行异步的I/O操作，而现在的API不能满足这个条件。而考虑到网络在现代应用中的重要性，提供一个高效的网络处理借口是很有必要的。因而文章提出了MegaPipe，在kernel和user-space中间搭建了一个channel能够高效的处理网络连接和批量处理网络消息。文章主要有三个贡献：
```
首先将Listening Socket分割开了，为多核并行处理提供了可能，增加了效率；

其二设计一个Lightweight socket，不和系统的VFS共享FID，这样又省去了和系统同步的时间；

最后对System call的批量处理，减少了由于短消息造成的大量system call而造成的系统负担。
```
##设计内容：MegaPipe
MegaPipe同时需要更改user-space library和linux内核，同时应用程序也需要根据api做调整。引入了channel用于应用进程和core通信，每两个channel之间相互独立。一个channel可以对应于多个core。channel是一个抽象的概念，表示core与user之间的通信与其他core与另外user通信的独立性。每一个channel有自己的一份data structure，如listening socket等。

和轮训检查是否有package不同，这里采用completion notification model。Kernel采用notification的形式将command结果返回给应用程序。

![mega_socket][mega_socket]

和mTCP&FastSocket vertically划分的概念很相似。

## 设计了Lightweight Socket
原有的socket为了兼容VFS，采用和全局文件共享的FD方式，但是TCP中socket有两个特点：

1. 很少共享
2. 生存时间短暂

因而为了兼容而使其配合全局使用得不偿失。

设计的lwsocket只在每个channel中本地可见，并且不采用最小FD id的方式，采用随机分配方式（万一碰撞呢？），避免了全局的socket使用共享，提高了效率。并且，如果程序指定，MegaPipe可以将lwsocket转化成普通的socket。而mTCP有了自己user-level的socket，FastSocket不仅更轻量，还解决了这个导致的兼容性问题。
## 将System Call按batch方式处理

![mega_exp][mega_exp]

因此system call处理总需要进行mode之间的切换，这会造成很大的时间消耗。因此对system call进行batch处理，可以提升性能。而在短消息系统中，因为大量的短消息通信会造成大量的内存申请释放等系统调用，这样会降低系统性能，并且还有大量的connection申请操作，不断的申请连接和断开连接也会造成系统负担。而采用batch之后批量处理system  call能够成倍的提升system call的执行效率。

在MegaPipe中，这部分操作都是由user-level的library来做的，对application来讲是透明的。

在mTCP这个由mTCP thread来解决，而FastSocket没有解决这个问题。
## 论文系统实现

主要有三个部分：Kernel， User-level library和application modification。

Kernel更改主要是对I/O操作的MegaPipe支持，主要是lwsocket，batch I/O command还有event queue。增加了一个module 1800行左右，并且更改了系统内核400行左右。

Library只是kernel module的一个包装，400行代码。

现在的设计只支持event-driven servers。并且需要针对completion notification model和shared listening socket部分进行对应的更改。

>in a "thread driven" runtimes, when request comes in, a new thread is being created and all the handling is being done in that thread.
>in "event driven" runtimes, when request comes in, the event is dispatched and handler will pick it up. When? In Node.js, there is an "event loop" which basically loops over all the pieces of code that need to be executed and executes them one by one. So the handler will handle the event once event loop invokes it. The important thing here is that all the handlers are being called in the same thread - event loop doesn't have a thread pool to use, it only has one thread.

这个兼容性比mTCP和FastSocket都要差。

## 论文实验
首先实验分析了MegaPipe对多核、不同长度消息的可拓展性，而后针对memcached和nginx做了实验分析。

![mega_expr][mega_expr]

首先可以看到对于多核的可拓展性，随着核数量的增加，MegaPipe的拓展性远远比现有的Linux系统好。并且对于短消息（1byte, 2byte…）性能提升更加明显。

其中有一点就是，多核性能比较时，有一个图性能提升不大，论文分析为由于系统锁和cache的congestion原因，但是文章却没有继续深入解决，是一个遗憾。

## 论文优点
实现了partitioned listening sockets，改善了多核共享listening socket的问题。

实现lwsocket，脱离出VPS，直接指向TCB，避免了和VPS的全局同步。

对System Call进行Batch处理，提高了性能。

## 论文缺点
不仅仅需要更改内核，还需要更改应用程序。

现在仅支持event-driven server，对thread-based性能提升有待验证。

### 对比mTCP&Fastsocket
 mTCP的优势是:
```
 越过了kernel;
 还有batched的packet处理，因此快。
```

Fastsocket的优势：
```
更好的API接口兼容性;

table-level的partition;

还有更好的connection locality。
```

虽然这篇文章在前，但是和mTCP和FastSocket相比仍然具有的优势是：

lightweight socket虽然导致兼容性问题，但却有性能提升，并且能够tranfer到正常的socket。

## 论文评价
在12年实现出这篇文章，并且同时考虑listening socket共享，batch还有lightweight socket

1. Shared resources
2. Broken locality
3. Per packet processing

这三个方面的优化都考虑到了比较难得。

[mega_socket]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/mega_socket.png
[mega_exp]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/mega_exp.png
[mega_expr]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/mega_expr.png
