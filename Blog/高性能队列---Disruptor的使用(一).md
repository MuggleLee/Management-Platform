
## 背景：
>Disruptor是英国外汇交易公司 LMAX 开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题。

## What：
Disruptor是一个开源的并发框架，能够在**无锁**的情况下实现网络的Queue并发操作。由于 Disruptor开发的系统单线程能支撑每秒600万订单，其性能强大引起了人们的关注。

>对Disruptor感兴趣的朋友可以移步到github查看官方代码[https://github.com/LMAX-Exchange/disruptor](https://github.com/LMAX-Exchange/disruptor)
## Why：
**1.Disruptor的数据结构是环形数组。**
>数组在内存中是占据连续的内存空间，所以在cpu缓存的时候会将在内存中读取到的内存地址后面的一部分数据也会缓存进去，从而达到性能的提升；但是链表在内存中是不连续的存储，所以只能缓存当前的内存地址。因此使用数组在cpu缓存机制中更加适合。

**2.数组的长度是2^n，可以通过位运算能够更快的定位到元素，而且数组的下标是递增的long类型，所以不会存在index溢出。**
>为什么不会内存溢出？因为long的最大值好大呀。假设并发量是1000万一秒，那需要差不多3万年才用完。

**3.无锁队列。**
>无论在生产者或消费者的线程中，都会先申请可以执行入队或者出队的操作的数组元素位置；当申请到之后，直接在已申请到的元素位置操作。

**4.预分配用于存储事件内容的内存空间。**
**5.在cpu缓存行大小为64位或更少的情况下，Disruptor能够避免伪共享问题。**
>通过缓存行填充来确保RingBuffer的序列号不会和其他东西同时存在于一个缓存行中。


***什么是伪共享？***
>参考：
[剖析Disruptor:为什么会这么快？（二）神奇的缓存行填充](https://ifeve.com/disruptor-cacheline-padding/)
[https://juejin.im/post/5c34c65bf265da61257849a8](https://juejin.im/post/5c34c65bf265da61257849a8)
[http://wiki.jikexueyuan.com/project/disruptor-getting-started/fake-share.html](http://wiki.jikexueyuan.com/project/disruptor-getting-started/fake-share.html)
[https://blog.csdn.net/qq_27680317/article/details/78486220](https://blog.csdn.net/qq_27680317/article/details/78486220)

## Where：

Disruptor常用的场景是———"生产者-消费者"模型。当发现BlockingQueue等队列达到瓶颈的情况下，可以考虑使用Disruptor代替。

## How：
在学会简单使用Disruptor之前，我们需要了解Disruptor的几个核心概念。
RingBuffer
顾名思义，这是一个环形的缓存区，它是存储消息的地方。其职责是负责对通过 Disruptor 进行交换的数据（事件）进行存储和更新。
Sequence
Sequencer 
Sequence Barrier
Wait Strategy
Event
EventProcessor
EventHandler
Producer


参考资料：
[https://zhuanlan.zhihu.com/p/21355046](https://zhuanlan.zhihu.com/p/21355046)
[https://juejin.im/post/5b744557518825612a228111](https://juejin.im/post/5b744557518825612a228111)
[http://ifeve.com/disruptor-getting-started/](http://ifeve.com/disruptor-getting-started/)








