
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
>无论在生产者或消费者的线程中，都会先申请可以执行入队或者出队操作的数组元素位置；当申请到之后，直接在已申请到的元素位置操作。

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

1.Disruptor常用的场景是———"生产者-消费者"模型。当发现BlockingQueue等队列达到瓶颈的情况下，可以考虑使用Disruptor代替。
2.Disruptor适用于两个独立的处理过程(两个线程)之间交换数据。

## How：
在学会简单使用Disruptor之前，我们需要了解Disruptor的几个核心组件。

![Disruptor流程图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Disruptor%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### RingBuffer
顾名思义，这是一个环形的缓存区，它是存储消息的地方。其职责是负责对通过 Disruptor 进行交换的数据（事件）进行存储和更新。可以把它用作在不同上下文（线程）间传递数据的buffer。

### Sequence
Sequence是一个递增的序号，可以理解为一个计数器。通过顺序递增的序号来编号管理通过其进行交换的数据（事件），对数据(事件)的处理过程总是沿着序号逐个递增处理。一个 Sequence 用于跟踪标识某个特定的事件处理者( RingBuffer/Consumer )的处理进度。虽然一个 AtomicLong 也可以用于标识进度，但定义 Sequence 来负责该问题还有另一个目的，那就是防止不同的 Sequence 之间的CPU缓存伪共享(Flase Sharing)问题。

### Sequencer
Sequencer 是 Disruptor 的真正核心。此接口有两个实现类 SingleProducerSequencer、MultiProducerSequencer ，它们定义在生产者和消费者之间快速、正确地传递数据的并发算法。

### Sequence Barrier
用于保持对RingBuffer的main published Sequence和Consumer依赖的其它Consumer的 Sequence的引用。Sequence Barrier还定义了决定Consumer是否还有可处理的事件的逻辑。SequenceBarrier用来在消费者之间以及消费者和RingBuffer之间建立依赖关系。在Disruptor中，依赖关系实际上指的是Sequence的大小关系，消费者A依赖于消费者B指的是消费者A的Sequence一定要小于等于消费者B的Sequence，这种大小关系决定了处理某个消息的先后顺序。因为所有消费者都依赖于RingBuffer，所以消费者的Sequence一定小于等于RingBuffer中名为cursor的Sequence，即消息一定是先被生产者放到Ringbuffer中，然后才能被消费者处理。

SequenceBarrier在初始化的时候会收集需要依赖的组件的Sequence，RingBuffer的cursor会被自动的加入其中。需要依赖其他消费者和/或RingBuffer的消费者在消费下一个消息时，会先等待在SequenceBarrier上，直到所有被依赖的消费者和RingBuffer的Sequence大于等于这个消费者的Sequence。当被依赖的消费者或RingBuffer的Sequence有变化时，会通知SequenceBarrier唤醒等待在它上面的消费者。

>个人理解：Sequence Barrier类似一个栅栏，可以管理消费者的消费顺序。在生产-消费模式中，不能存在消费比生产多，肯定消费量是小于等于生产量，所以Sequence Barrier就是控制消费者过度消费，如果消费量大于生产量，这样的程序就不合理了。

### Wait Strategy
当消费者等待在SequenceBarrier上时，有许多可选的等待策略，不同的等待策略在延迟和CPU资源的占用上有所不同，可以视应用场景选择：
**BusySpinWaitStrategy ：** 自旋等待，类似Linux Kernel使用的自旋锁。低延迟但同时对CPU资源的占用也多。
**BlockingWaitStrategy ：** 使用锁和条件变量。CPU资源的占用少，延迟大。
**SleepingWaitStrategy ：** 在多次循环尝试不成功后，选择让出CPU，等待下次调度，多次调度后仍不成功，尝试前睡眠一个纳秒级别的时间再尝试。这种策略平衡了延迟和CPU资源占用，但延迟不均匀。
**YieldingWaitStrategy ：** 在多次循环尝试不成功后，选择让出CPU，等待下次调。平衡了延迟和CPU资源占用，但延迟也比较均匀。
**PhasedBackoffWaitStrategy ：** 上面多种策略的综合，CPU资源的占用少，延迟大。

### Event
在 Disruptor 的语义中，生产者和消费者之间进行交换的数据被称为事件(Event)。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者定义并指定。

### EventProcessor
EventProcessor 持有特定消费者(Consumer)的 Sequence，并提供用于调用事件处理实现的事件循环(Event Loop)。通过把EventProcessor提交到线程池来真正执行，有两类Processor:

其中一类消费者是BatchEvenProcessor。每个BatchEvenProcessor有一个Sequence，来记录自己消费RingBuffer中消息的情况。所以，一个消息必然会被每一个BatchEvenProcessor消费。

另一类消费者是WorkProcessor。每个WorkProcessor也有一个Sequence，多个WorkProcessor还共享一个Sequence用于互斥的访问RingBuffer。一个消息被一个WorkProcessor消费，就不会被共享一个Sequence的其他WorkProcessor消费。这个被WorkProcessor共享的Sequence相当于尾指针。

### EventHandler
Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现。开发者实现EventHandler，然后作为入参传递给EventProcessor的实例。

### Producer
即生产者，只是泛指调用 Disruptor 发布事件的用户代码，Disruptor 没有定义特定接口或类型。

好了，大概了解几个核心组件之后，就可以动手写个简单的Disruptor的Demo吧！

Demo基于下面四个步骤编写：

1.创建生产消费的数据类型。
2.建立一个工厂Event类，用于创建Event类实例对象
3.需要有一个监听事件类，用户处理数据(Event类)
4.实例化Disruptor实例，配置一系列参数，编写Disruptor核心组件
5.编写生产者组件，向Disruptor容器中去投递数据

```java
/**
 * 步骤1：创建生产消费的数据类型。
 * 事件类。定义与Disruptor进行交换数据类型
 */
public class OrderEvent {
    private Long value;

    public Long getValue() {
        return value;
    }

    public void setValue(Long value) {
        this.value = value;
    }
}
```
```java
/**
 * 步骤2：建立一个事件工厂类，用于创建Event类实例对象
 * 定义如何实例化事件类(Event类)。RingBuffer通过EventFactory创建Event实例
 * 需要实现om.lmax.disruptor.EventFactory接口
 */
public class OrderEventFactor implements EventFactory<OrderEvent> {
    @Override
    public OrderEvent newInstance() {
        return new OrderEvent();
    }
}
```
```java
/**
 * 步骤3：创建监听事件类，处理数据
 */
public class OrderEventHandler implements EventHandler<OrderEvent> {
    @Override
    public void onEvent(OrderEvent orderEvent, long l, boolean b) throws Exception {
        System.out.println("消费者 ： " + orderEvent.getValue());
    }
}
```
```java
/**
 * 测试类
 * 4.实例化Disruptor实例，配置一系列参数，编写Disruptor核心组件
 */
public class Main {

    //RingBuffer大小。注意：大小为2的N次方，否则会报错为：java.lang.IllegalArgumentException: bufferSize must be a power of 2
    private int ringBufferSize = 1024 * 1024;

    //创建线程池
    private ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public static void main(String[] args) {
        Main main = new Main();
        main.showDisruptorDemo();
    }

    public void showDisruptorDemo(){
        //创建事件工厂类和事件监听类对象
        OrderEventFactor orderEventFactor = new OrderEventFactor();
        OrderEventHandler orderEventHandler = new OrderEventHandler();

        //1.实例化Disruptor
        Disruptor disruptor = new Disruptor(orderEventFactor,ringBufferSize,executor,ProducerType.SINGLE,new BlockingWaitStrategy());
        //2.监听对象
        disruptor.handleEventsWith(orderEventHandler);
        //3.启动disruptor
        disruptor.start();
        //4.获取实际存储数据容器--RingBuffer
        RingBuffer<OrderEvent> ringBuffer = disruptor.getRingBuffer();

        OrderEventProducer orderEventProducer = new OrderEventProducer(ringBuffer);

        //创建缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(8);

        for (long i = 0; i < 100; i++) {
            byteBuffer.putLong(0,i);
            orderEventProducer.sendData(byteBuffer);
        }
        //关闭Disruptor
        disruptor.shutdown();
        //关闭线程池
        executor.shutdown();
    }
}
```
```java
/**
 * 5.编写生产者组件，向Disruptor容器中去投递数据
 */
public class OrderEventProducer {

    private RingBuffer<OrderEvent> ringBuffer;

    public OrderEventProducer(RingBuffer ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    public void sendData(ByteBuffer byteBuffer) {
        //1.获取序列号
        long sequence = ringBuffer.next();
        try {
            //2.根据序列号找到具体的OrderEvent对象
            OrderEvent orderEvent = ringBuffer.get(sequence);
            //3.赋值
            orderEvent.setValue(byteBuffer.getLong(0));
        } finally {
            //4.提交发布操作
            ringBuffer.publish(sequence);
        }
    }
}
```

输出结果：
```java
消费者 ： 0
消费者 ： 1
消费者 ： 2
...
消费者 ： 97
消费者 ： 98
消费者 ： 99
```

目前根据官方文档和参考各位前辈的博客，顺利写出一个简单的Disruptor的Demo。接下来，继续往深处探究、学习Disruptor的高性能！

敬请期待~




参考资料：
[https://zhuanlan.zhihu.com/p/21355046](https://zhuanlan.zhihu.com/p/21355046)
[https://juejin.im/post/5b744557518825612a228111](https://juejin.im/post/5b744557518825612a228111)
[http://ifeve.com/disruptor-getting-started/](http://ifeve.com/disruptor-getting-started/)
[http://ifeve.com/disruptor-info/](http://ifeve.com/disruptor-info/)



