# Sedic: Privacy-Aware Data Intensive Computing on Hybrid Clouds

_会议：CCS 2011_

_作者：Kehuan Zhang, Xiaoyong Zhou, Yangyi Chen and XiaoFengWang, Yaoping Ruan_

_机构：Indiana University, IBM T.J.Watson Research Center_

## 论文摘要
由于云中隐私问题引起重视，使将数据外包放到云中计算受到了限制。首先想到是将计算任务分开将设计隐私的任务放在私有的云中执行，而将普通任务放在公有云中执行，但现今并没有这样的系统。因此文章设计并实现了一个混合云系统，Sedic，结合利用公有云和私有云，在利用云对用户数据进行存储和处理（Map-Reduce）的同时保护好用户数据的隐私。系统既保护了用户隐私数据的隐私，也利用了公有云的计算能力高效计算非隐私部分数据。

## 论文背景
云服务的出现给很多组织或机构解决了一个很大的问题，计算和存储能力的不足。但是在云服务的发展中也出现了一个很重要的问题，即隐私问题。当客户的数据在云系统中存储或者计算时，云服务提供商能够检测存储内容和计算过程，这可能会给公司或者医疗机构等带来隐私问题。当给云服务的数据中有一部分敏感数据的时候不方便将数据直接外包给商用云服务商，而这部分数据的计算任务只能在客户相信的私有云中计算。因此需要一个云框架既包含公有云也包含私有云，能够将敏感数据私有云中计算，其他非敏感部分放在共有云中计算。

文章的贡献是：设计了一个新的并且对用户透明的云计算框架，能够有效处理隐私数据，并且利用公有云的计算资源；并且对于Map-Reduce任务中的reduce造成的公私有云之间的通信量进行了优化减少；实现并且评测了系统的有效性。

## 系统框架
用户首先对需要处理的数据打标签，需要将数据中隐私数据标记出来，然后将数据上传到云中，并且将隐私数据位置（标签）上传给Sedic中的私有云；Sedic分析隐私数据标签，将数据进行partition并且对数据进行replication；而后，Sedic根据数据隐私标签分别在公有云和私有云中执行Mapper任务，然后从公有云和私有云中收集结果完成reduction，而后得出处理结果返回给用户。


## 论文内容

Sedic数据存储&复制：

![sedic_replicate][sedic_replicate]

如 Fig 1 中所示，Sedic 混合云方案将分布式存储系统划分为两块，一块为公有云（左边绿色区域），一块为私有云（右边红色区域）。存储系统的 Name Node 为与私有云中，因为一些对文件的调度和对文件私有信息标识的存储是存放在 Name Node 中，而这些信息不能暴露在共有云中。当一个 Client 想要上传数据时，首先他会向 Name Node 发送创建文件请求，同时将自己的文件的敏感数据标签信息（用于表示敏感信息在文件中的位置）上传至 Name Node 节点。Name Node 为 Client 创建文件并保存敏感数据标签信息，但此时不给用户文件分配存储空间。Client 在向 Data Node 上传数据时，会向 Name Node 请求 Block，这时 Name Node 根据 Client 上传的文件位置查询敏感数据标签，给 Client 分配合适的 Data Node，含有敏感数据分配私有 Data Node 节点，不含有则分配共有 Data Node 节点。如此不断请求Block 直到 Client 所有数据上传完成。

在数据复制的过程中，隐私数据只会在私有云节点之间复制，并且记录的对应的隐私记录的标签也随之复制到其他节点。而后公有云中的数据则尽可能的在公有云之间复制，因为他们更可能在公有云之间处理。

Sedic Map-Reduce过程：

![sedic_map][sedic_map]

首先在map过程中，会根据记录的隐私数据的metadata将数据切割分配给public task和private task，这两个task分别执行在公有云和私有云上。Name node收到每个节点发来的心跳时，从任务的队列中取出对应属性的task分给节点运行。

而后在任务完成后的reduce过程中，将会将所有的数据都收回到私有云中进行reduce。并且系统设置了一个threshold，可以用来限制转移到公有云上的计算的数据量大小，通过限制这个数据量的大小可以限制最后由公有云上完成map之后将结果返回私有云而造成的大量通信开销。但这也和执行效率形成了一个trade-off。

并且论文更改了Hadoop的MapReduce过程，将public的map结果首先在公有云中整合一次，而后再返回到私有云中，这样也节省了云之间的通信开销。

Map-Reduce有自己的工作流程，reduce操作其实是和系统中的fold操作相对应，fold操作是将所有元素聚合成一个list的操作，这也其实也是reduce的目的。虽然真实reduce操作要比这个fold操作复杂的多，但是其实本质上就是fold。我们可以注意到fold操作是具有连接性的，比如说f([a1, a2, a3, a4, a5, a6]) = f([f([a1, a2]), f([a3, a4]), f([a5, a6])])，当只有a3和a4涉及隐私时，我们将f([a1, a2]),和f([a5, a6])外包出去，这样可以减少私有云的工作量。而进一步观察我们可以发现，我们可以将f([a1, a2]),和f([a5, a6])先聚合起来，这是因为fold操作也是具有交换律的。我们现将f([a1, a2，a5, a6])先计算出来，再返回给私有云，这样就减少了私有云的通信负担。

而这个实现的代码框架如图中所示。

![sedic_reduce][sedic_reduce]

## 论文实验
部署了6个节点，其中3个为公有云节点，3个为私有云节点。

测试了3个真实数据集，IDS dataset, Spam data set 和 Twitter data set。

测试了5种不同的JOB：Port Scan Detection， Traffic Statistics，Email Word Count， Spam Keyword Count 还有Grep命令。

![sedic_exp][sedic_exp]

实验显示，利用本方案，相比将所有计算放在私有云中，私有云的计算速度有了1.5~6倍的提升。图中baseline是指只是用私有云计算，可以看到随着隐私数据的增多论文方案的计算时间也随之增加，但一直远远比baseline好的多。

## 论文缺点&可扩展
本文方法的reduce过程还是太粗糙，会造成大量的公有云与私有云之间的通信。

并且不能支持Map-Reduce多次的迭代进行。

最后这种公私有合作真的现实么？


[sedic_replicate]:https://raw.githubusercontent.com/Doffery/v9-cpu/4a5ed0802faf895fd6d7e0002a2ee68f241a8d26/root/usr/paper_report/pic/sedic_data_replicate.PNG

[sedic_map]:https://raw.githubusercontent.com/Doffery/v9-cpu/4a5ed0802faf895fd6d7e0002a2ee68f241a8d26/root/usr/paper_report/pic/sedic_map.PNG

[sedic_reduce]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/sedic_reduce.PNG

[sedic_exp]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/sedic_exp.PNG.png