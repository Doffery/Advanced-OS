# From Collision To Exploitation : Unleashing Use-After-Free Vulnerabilities in Linux Kernel

_By Dong Yuan (2015210938) _

_会议：CCS 2015_

_作者：Wen Xu, Juanru Li, Junliang Shu,Wenbo Yang Tianyi Xie, Yuanyuan Zhang∗ , Dawu Gu_

_机构：Shanghai Jiao Tong University_

## 论文摘要
Linux内核中漏洞数量随着代码量的增加越来越多，在应用中use-after-free的漏洞备受关注，而内核中研究却很少。针对linux内核中use-after-free的漏洞，本文利用提出两个memory collision attack攻击。主要的insight是基于概率性的内存碰撞模型，用户正常申请内存时可能会申请到之前另一个程序刚刚释放的内存。我们提出的两个攻击中，Object-based attack 利用内存回收利用机制实现对已经回收的内存进行覆盖；Physmap-based attack利用physmap和SLAB缓存在物理内存中有重叠区域实现内存操控攻击。

## 论文背景
因为一些保护机制的出现，例如，DEP, SALR, Stack canaries 等， 从 user-level 去 攻击程序变得更艰难，现在不少考虑从OS Kernel对系统进行攻击，从而进一步做其他破坏。因此本文考虑从OS Kernel对系统进行攻击，从而攻击程序。Linux系统中的其他漏洞例如空指针栈溢出漏洞kernel已经有不少的应对措施，而对于利用Heap对系统进行攻击，kernel还存在很多漏洞，本文即针对heap的use-after-free漏洞提出了两种攻击。

其中use-after-free漏洞是指，在某一块内存区域使用完之后，系统回收了内存，而系统中其他程序段没有意识到这块内存已经被回收，依旧对这篇内存进行操作。而攻击着可以在系统回收内存之后利用手段再次使用这块内存并填入自己的恶意内容，如恶意代码等，当程序再次对这块内存进行操作时，系统即被攻击。

## 论文内容
主要介绍两种攻击并且分析了攻击的有效性，并提出两个缓解措施。

### OBJECT-BASED ATTACK
这个攻击主要基于这样一个insight：
```
Our proposed attack strategy lever-ages this observation: Once an allocated vulnerable object is freed, the kernel will recycle the space occupied by that object for a recent allocation.
```
在linux内核中使用SLUB/SLAB对内存进行进行分配，并且会将已经释放的slab留作下一次分配使用。这样可以利用这个特点将自己的恶意代码注入到之前free的内存里去。

论文通过SLUB的kmalloc申请内存，并对之前释放的空间进行重写。

在SLUB中，是对每一个大小相同的object在一个队列中进行重新分配，如果原object的大小不合kmalloc的标准的话（512bits, 1024bits...），攻击者无法利用kmalloc进行有效重写，论文给出的攻击方法是对一整个slab的所有内容进行重写。

如下左图是攻击的伪代码，其中Step 1首先正常使用内存，而后Step 2释放内存，在Step 3中恶意程序“正常“申请内存，企图制造冲突，当冲突成功时，正常程序则会执行恶意的shellcode（Step 4）。

![use_after_free][use_after_fee_pic]

### PHYSMAP-BASED ATTACK
这个攻击方式更底层，利用user space关联到kernel space的映射表，大量写入恶意内容到kernel space。为了增加恶意写入恰好为刚释放的内存的碰撞几率，需要两点：事先占用大量内存，将易受到攻击的objects分配时分散在整个kernel space中，这样命中其中一个object的几率就增大了，还有一点就是大量的内存写入。

文章提出可以使用给physmap设立边界使其不与kernel space重叠或者减少重叠区域的方式增大攻击难度。

如上右图中，是实行physmap-based攻击的伪代码。其中Step 2首先攻击者对physmap中填充内存，而后在Step 3中，释放内存，并在Step 4中制造冲突，最好在Step 5中完成攻击。

### 可行性分析
就两个攻击分别分析可行性。

#### Object-based Attack
首先分析Object-based Attack的可行性，论文实验，在free后的716次填充中，和之前free的内容发生了碰撞，验证了可行性，如果系统再次使用这块内存，则攻击成功。

而后分析了Object-based Attack的优势和局限性。

优势：利用kmalloc简单并且容易控制，攻击者能够指定申请空间大小，方面实行攻击。

局限性：对于内存大小和kmalloc大小不匹配的攻击实施难度较大。

#### PHYSMAP-BASED ATTACK
而后分析了32bit和64bit内核的地址空间，证明了physmap空间和内核使用的虚拟空间的地址存在重合区域，使用文章的方法的确可以产生碰撞。

而后分析类physmap-based attack方法的优势和局限性。

优势，主要有三点：

攻击稳定：只需要不断的往内存里填攻击代码就行。

攻击可实施性很强：因为kernel只要不断的释放内存，就能够被覆盖攻击。

攻击效果很明显：攻击者可以指定执行任意恶意代码，如果攻击成功可以应用的场景很多：不想OBA只能在SLUB下成功。

局限性：如果内存很快被其他程序占用，没有来得及被恶意代码重写的话，攻击不能成功。对于只用过一次的内存，其实被再次调用的可能性比较低。

## 论文实验

首先在32bit和64bit，Linux Kernel，实施了OBA和PBA攻击，分析了成功几率。

![uaf_exp1][uaf_exp1]

Physmap-based的攻击在32bit和64bit的kernel中表现都比较好，但是OBA在64bit中攻击成功率较低，因为在64bit系统中有更多种的object大小的内存分配释放，kmalloc的碰撞概率降低。

![uaf_exp2][uaf_exp2]

然后测试了Android Kernel，利用已经公布了的android kernel的use-after-free漏洞成功对安卓内核实施了攻击，实验证明攻击对Samsung等知名品牌安卓手机都有效～

## 论文缺点&可改进之处
SLAB将内存分给相同类型的object重复利用，攻击难度更大，文章没有提出有效方法。

论文实验都是在没有其他程序抢占内存的情况下进行？free出来的空间可能被其他程序再使用了，这也就不能形成攻击了。

并且针对不同大小的object-based attack，文章假设所有要攻击的都在一个slab,并且全部重写，这样的假设有点强，并且全部重写会kerne探测的可能会更大？

## References

https://en.wikipedia.org/wiki/Slab_allocation
https://www.kernel.org/doc/gorman/html/understand/understand011.html
https://en.wikipedia.org/wiki/Memory_management#HEAP

[use_after_fee_pic]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/use_after_free_attack.png

[uaf_exp1]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/uaf_exp1.png

[uaf_exp2]:https://raw.githubusercontent.com/Doffery/v9-cpu/master/root/usr/paper_report/pic/uaf_exp2.png