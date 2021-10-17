# Detecting concurrency memory corruption vulnerabilities FSE 2019	
	
## Insight:  

a concurrency vulnerability is more related to the orders of events that can be reversed in different executions, no matter whether the corresponding accesses can form data races. 	在重复执行过程中顺序发生颠倒的事件更有可能导致并发漏洞, 检测并发漏洞的关键是确定给定执行中一组事件（内存操作代码块）中的两个或多个是否可交换。

由此定义了同步距离，认为同步距离小的可交换事件更容易引发漏洞。

并引入了第三个事件，两个距离第三事件距离较近的事件彼此更可能交换而引发漏洞

---

## Exiting Methods 的局限性:

###Data Race Detector:

1. data race关注两个线程同时访问的数据，但并发漏洞可能由多个线程引发。（it is ineffective to apply the approaches for race detection to detect concurrency vulnerabilities. This is because the two concepts are not the same one.）
2. 没有统一的定义race的规则。
3.	false positive高，且遗漏true positive

![limits](https://raw.githubusercontent.com/Anderson-Xia/Note/main/img/20211016/2021101601.png)

在上图中，传统的race detector会因为p->test()和free(p)都被m加锁而认为其合法，但实际执行中可能先执行完t2中的三个语句再执行t1，从而引发concurrency UAF
###maximal causal models ：

受限于求解器，有效性不高

---

## 本文研究对象

 与内存损坏相关的漏洞
 1. use-after-free
 2. NULL pointer dereference
 3. double-free

## 具体方法

 首先定义了可交换事件，用于判断两个事件的执行顺序是否可逆转（特别的，提到该定义有别于 happens-before（用于判断两个事件的先后顺序, 使用<a href="https://zhuanlan.zhihu.com/p/419944615" target="_blank">Vector Clock</a>可以判断这种关系））
 
 随后对三种漏洞类型分别设计检测算法。

 使用同步距离来量化两个事件交换的可行性,同一进程内部的一对加锁到解锁是一条同步边，不同进程见解锁到加锁是一条同步边。两事件间含有最少同步边的路径上的同步边数即为其同步距离。
>A sync-edge (⇒) is an edge either 
> 
> (1) from an event acq(m) to its paired event rel(m) in the same thread
>
>(2) from an event rel(m) to a later event acq(m) by two different threads. 
> 
>Based on sync-edge, we define the sync-distance (or distance for short) of two event e1 and e2 as the minimal number of sync-edges that order the two events, denoted as D(e1, e2).

在同步距离的基础上，作者提出了松弛可交换事件——若两事件不存在HPR（happens-before relation），其可交换。反之，若其同步距离小于3，在本文中也视为（松弛）可交换

 本文采用如下图所示算法判断可交换事件

 ![algorithm](https://raw.githubusercontent.com/Anderson-Xia/Note/main/img/20211016/2021101602.png)
 
在如图所示算法中，6-8行判断可交换事件，9-18行判断松弛可交换事件。为判断同步距离是否不大于3，实际上是要判断不同线程中的两事件e1和e2间是否只存在一条同步边。（注意，因为e1,e2存在HPR，所以e1后必定存在一条rel(l)边，e2前必定存在一条acq(l)边,这样才会构成HPR关系，考虑到总同步边数不大于3，则rel(l)和acq(l)间只允许存在一条）

为了实现上图所示的判断，本文在e1释放锁l时维护l的VC（vector clock），命名为PredVC(e1)，并检查e2发生时PredVC(e1)与VC(l)是否一致，如果一致，则说明e1,e2间没有发生有关l的同步事件，则同步距离为3。

##检测算法

针对三种研究对象分别设计检测算法——主要使用VC和Hook事件

###以UAF为例

![algorithm](https://raw.githubusercontent.com/Anderson-Xia/Note/main/img/20211016/2021101603.png)

如上图，对于内存读取事件em，将VC(t)传递给VC(em)，当free事件efp发生后，逐一对比VC中是否存在可以与efp交换的em事件，如果有，则report a uaf。

对于其他两种Vulnerability，同理。