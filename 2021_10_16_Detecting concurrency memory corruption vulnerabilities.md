# Detecting concurrency memory corruption vulnerabilities FSE 2019	
	
## Insight:  

a concurrency vulnerability is more related to the orders of events that can be reversed in different executions, no matter whether the corresponding accesses can form data races. 	在重复执行过程中顺序发生颠倒的事件更有可能导致并发漏洞, 检测并发漏洞的关键是确定给定执行中一组事件（内存操作代码块）中的两个或多个是否可交换。

由此定义了同步距离，认为同步距离小的可交换事件更容易引发漏洞。

并引入了第三个事件，两个距离第三事件距离较近的事件彼此更可能交换而引发漏洞

## Exiting Methods 的局限性:

###Data Race Detector:

1. data race关注两个线程同时访问的数据，但并发漏洞可能由多个线程引发。（it is ineffective to apply the approaches for race detection to detect concurrency vulnerabilities. This is because the two concepts are not the same one.）
2. 没有统一的定义race的规则。
3.	false positive高，且遗漏true positive

![limits](https://raw.githubusercontent.com/Anderson-Xia/Note/main/img/2021101601.png)

在上图中，传统的race detector会因为p->test()和free(p)都被m加锁而认为其合法，但实际执行中可能先执行完t2中的三个语句再执行t1，从而引发concurrency UAF
maximal causal models ：受限于求解器，有效性不高

## 本文研究对象

 与内存损坏相关的漏洞——use-after-free, NULL pointer dereference, double-free

## 具体方法

 首先定义了可交换事件，用于判断两个事件的执行顺序是否可逆转（特别的，提到该定义有别于 happens-before（用于判断两个事件的先后顺序, 使用<a href="https://zhuanlan.zhihu.com/p/419944615" target="_blank">Vector Clock</a>可以判断这种关系））随后对三种漏洞类型分别设计检测算法。
