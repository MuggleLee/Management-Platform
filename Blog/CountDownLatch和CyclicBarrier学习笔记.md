# CountDownLatch
CountDownLatch：一个或多个线程等待其他线程完成操作。
有一种业务场景下，某一个动作需要等待其它线程完成后才会触发。举个栗子，一个班上50个人，考完试之后需要计算全班同学的总成绩，这种情况使用CountDownLatch并发类就最合适了，每一个人的成绩等于一个线程，需要等待50个线程执行完之后才能执行最后一个动作（计算总成绩）。

  

使用该并发类并不难，熟悉几个主要方法就可以。
 - public CountDownLatch(int count)： //参数count为计数值
 - await()： 调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
 - await(long timeout,TimeUnit unit)： //和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
 - countDown()： //将count值减1
 - getCount()：//获取count值
先写个简单的Demo：
```java
public class CountDownLatchDemo implements Runnable {  
  
    private static final int threadCount = 3;  
  
    private static CountDownLatch countDownLatch = new CountDownLatch(threadCount);  
  
    private static double sum = 0D;  
  
    public String name;  
	  
    public CountDownLatchDemo() {  
    }  
	  
    public CountDownLatchDemo(String name) {  
	this.name = name;  
    }  
  
    public static void main(String[] args) throws InterruptedException {  
        ExecutorService executorService = Executors.newFixedThreadPool(threadCount);  
	executorService.execute(new CountDownLatchDemo("A同学"));  
        executorService.execute(new CountDownLatchDemo("B同学"));  
	executorService.execute(new CountDownLatchDemo("C同学"));  
	countDownLatch.await();  
	System.out.println("所有人的总成绩 " + sum);  
    }  
  
    @Override  
    public void run() {  
        double grade = Math.round(Math.random() * 100);  
	System.out.println("计算 " + name + " 成绩，成绩是" + grade);  
	sum += grade;  
	countDownLatch.countDown();  
    }  
}
```
输出结果：
```
计算 C同学 成绩，成绩是26.0
计算 B同学 成绩，成绩是18.0
计算 A同学 成绩，成绩是53.0
所有人的总成绩 97.0
```
好了，知道大概怎么使用这个并发类，接下来开始撸源码！
```java
public class CountDownLatch {
    //内部使用Sync继承AQS
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            //设置计算值
            setState(count);
        }

        //获取计算值
        int getCount() {
            return getState();
        }

        //尝试获取共享锁
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        //尝试释放锁
        protected boolean tryReleaseShared(int releases) {
            for (; ; ) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    //CountDownLatch唯一的构造器，参数count值为阻塞线程数
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    //阻塞线程，直到count值为0
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //限定阻塞时间，超出限定时间后，count值还没为0都会执行之后的线程
    public boolean await(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //释放锁操作，每调用一次，count值减一
    public void countDown() {
        sync.releaseShared(1);
    }

    //获取count值
    public long getCount() {
        return sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```
大概流程就是，先调用CountDownLatch构造器设置计算值(count)，当主线程(main方法)执行到await()方法的时候就会阻塞主线程，然后执行线程池内的任务，通过执行countDown()方法，将count值不断的减1，直到count值为0的时候就会释放锁，继续执行主线程。


# CyclicBarrier

CyclicBarrier：一组线程相互等待，达到一个共同点再继续执行。

比较困惑的一点是，CyclicBarrier和CountDownLatch两个并发类有什么区别？从概念上和例子上来看，貌似都差不多呀。
参考多篇大佬博客，总结出以下个人观点：

源码上：
CountDownLatch基于AQS实现的；设置了等待线程数后无法重置；
CyclicBarrier基于Condition和ReentrantLock锁实现；设置等待线程数后可以调用reset()重置；

概念上区别：
CountDownLatch：一个或多个线程等待其他线程执行完成后再执行。（关键词：“其他线程执行完成后”）
例子：线程S需要等待线程A、B、C都执行完成后才会执行。</br>
![enter image description here](https://raw.githubusercontent.com/MuggleLee/PicGo/master/CountDownLatch--Example.png)


CyclicBarrier：多个线程相互等待，到达阻塞点（屏障）后等待其他线程，直到所有线程都达到阻塞点后，阻塞解除并唤醒所有等待线程，所有的线程都继续往下执行。（关键词：“阻塞点”）


举个栗子：小A、小B，小C三位朋友约定在广州塔见面之后去喝早茶，分别从三处不同的地方赶往广州塔碰面。但由于小C迟到，小A，小B等待小C，直到小C都到达广州塔后再一同去喝早茶。使用并发类CyclicBarrier解释就是，线程A，线程B被阻塞，直到线程C执行到阻塞点后阻塞解除，唤醒所有线程。线程A、B、C都继续往下执行。</br>
![enter image description here](https://raw.githubusercontent.com/MuggleLee/PicGo/master/CyclicBarrier--Example.png)


代码示例：
```java
public class CyclicBarrierDemo implements Runnable {  
  
    public static int count = 3;  
  
    public String name;  
  
    public CyclicBarrierDemo(String name) {  
        this.name = name;  
    }  
  
    public CyclicBarrierDemo() {  
    }  
  
    public static CyclicBarrier cyclicBarrier = new CyclicBarrier(count, new Runnable() {  
        @Override  
	public void run() {  
	    System.out.println("大家启程去茶楼，喝茶去噜...");  
	}  
    });  
  
    public static void main(String[] args) {  
        //创建线程池  
        ExecutorService executor = Executors.newFixedThreadPool(count);  
	try {  
            //执行线程  
	    executor.execute(new CyclicBarrierDemo("小A"));  
	    executor.execute(new CyclicBarrierDemo("小B"));  
	    executor.execute(new CyclicBarrierDemo("小C"));  
        } catch (Exception e) {  
	    e.printStackTrace();  
	} finally {  
            executor.shutdown();  
	}  
    }  
  
    @Override  
	public void run() {  
        try {  
            if("小C".equals(name)){  
               System.out.println(name + "路上塞车");  
	       Thread.sleep(1000);  
	    }  
	    System.out.println(this.name + "到达广州塔... ");  
	    cyclicBarrier.await();  
	    System.out.println(this.name + "到达茶楼 ");  
	} catch (InterruptedException e) {  
	    e.printStackTrace();  
	} catch (BrokenBarrierException e) {  
	    e.printStackTrace();  
	}  
    }  
}
```

结果：
```java
小B到达广州塔... 
小C路上塞车
小A到达广州塔... 
小C到达广州塔... 
大家启程去茶楼，喝茶去噜...
小C到达茶楼 
小B到达茶楼 
小A到达茶楼 
```

接下来，膜拜Doug Lea写的源码
```java
public class CyclicBarrier {
    //Generation是CyclicBarrier的一个私有内部类，只有一个成员变量broken来标识当前的barrier是否已“损坏”：
    private static class Generation {
        boolean broken = false;
    }

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition(); //通过lock得到的一个状态变量
    private final int parties;  //表示总的等待线程的数量。
    private final Runnable barrierCommand;  //当屏障(barrier)撤销时，需要执行的操作
    private Generation generation = new Generation();  //当前的Generation。每当屏障失效或者开闸之后都会自动替换掉。从而实现重置的功能。

    private int count;//实际中仍在等待的线程数，每当有一个线程到达屏障点，count值就会减一；当一次新的运算开始后，count的值被重置为parties。

    //唤醒等待线程，设置下一个Generation
    private void nextGeneration() {
        // 唤醒所有等待线程
        trip.signalAll();
        // 重置进入屏障的线程数量
        count = parties;
        // 新生一代
        generation = new Generation();
    }

    //将当前屏障置为破坏状态、重置count、并唤醒所有被阻塞的线程。
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    private int dowait(boolean timed, long nanos)
            throws InterruptedException, BrokenBarrierException,
            TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();   //获取锁
        try {
            //保存此时的generation
            final Generation g = generation;
            //判断屏障是否被破坏
            if (g.broken)
                throw new BrokenBarrierException();
            //判断线程是否被中断
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            //正在等待进入屏障的线程数量减一
            int index = --count;
            //判断等待进入屏障的线程数量是否为0
            if (index == 0) {
                // 运行的动作标识
                boolean ranAction = false;
                try {
                    // 保存运行动作	
                    final Runnable command = barrierCommand;
                    // 如果动作不为空则运行
                    if (command != null)
                        command.run();
                    // 设置运行的动作状态
                    ranAction = true;
                    // 进入下一代，唤醒所有线程;重置count和generation值.
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }
            // index不为0，无限循环
            for (; ; ) {
                try {
                    // 如果没有设置定时就一直等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)  // 如果设置了定时
                        nanos = trip.awaitNanos(nanos);  // 等待定时的时间后，重新唤醒
                } catch (InterruptedException ie) {
                    if (g == generation && !g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }
                // 取消阻塞后，重新判断状态
                if (g.broken)
                    throw new BrokenBarrierException();
                if (g != generation)
                    return index;
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        //表示必须同时到达屏障(barrier)的线程个数
        this.parties = parties;
        //表示处于等待状态的线程个数
        this.count = parties;
        //表示当等待线程数为0时，会执行的动作
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    //返回参与相互等待的线程数
    public int getParties() {
        return parties;
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe);
        }
    }

    public int await(long timeout, TimeUnit unit)
            throws InterruptedException,
            BrokenBarrierException,
            TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    // 判断此屏障是否处于中断状态。
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    //将屏障重置为其初始状态。
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();
            nextGeneration();
        } finally {
            lock.unlock();
        }
    }

    //返回当前在屏障处等待的参与者数目，此方法主要用于调试和断言。
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```

其中，使用CyclicBarrier并发类的核心就是dowait方法，结合源码分析，得出以下的流程图：

![enter image description here](https://raw.githubusercontent.com/MuggleLee/PicGo/master/CyclicBarrier-dowait-flow.png)

# Semaphor
什么是Semaphor？
>Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

什么情况下使用Semaphor？
>用于限制获取某种资源的线程数量。

怎样使用Semaphor？
```java
public class SemaphoreDemo {

    private static final int permits = 4;

    private static Semaphore semaphore = new Semaphore(permits);

    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        for (int i = 0; i < 12; i++) {
            final int index = i;
            executor.execute(() -> {
                try {
                    semaphore.acquire();
                    show(index);
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }

    public static void show(int index) throws InterruptedException {
        Thread.sleep(1000);
        System.out.println(new Date().toString() + " --- " + Thread.currentThread().getName());
    }
}
```

运行结果：
```
Tue Feb 12 10:56:30 CST 2019 --- pool-1-thread-1
Tue Feb 12 10:56:30 CST 2019 --- pool-1-thread-2
Tue Feb 12 10:56:30 CST 2019 --- pool-1-thread-6
Tue Feb 12 10:56:30 CST 2019 --- pool-1-thread-5
Tue Feb 12 10:56:31 CST 2019 --- pool-1-thread-7
Tue Feb 12 10:56:31 CST 2019 --- pool-1-thread-4
Tue Feb 12 10:56:31 CST 2019 --- pool-1-thread-8
Tue Feb 12 10:56:31 CST 2019 --- pool-1-thread-3
Tue Feb 12 10:56:32 CST 2019 --- pool-1-thread-9
Tue Feb 12 10:56:32 CST 2019 --- pool-1-thread-12
Tue Feb 12 10:56:32 CST 2019 --- pool-1-thread-10
Tue Feb 12 10:56:32 CST 2019 --- pool-1-thread-11
```

由运行结果可以看出，每隔一秒就有4个线程执行，可以说明，使用并发类Semaphor可以限制并发线程数量。

使用并发类Semaphor十分简单，只需要掌握几个常用的方法。
- Semaphore(int permits)：创建给定的许可数和使用默认非公平策略。
- Semaphore(int permits,boolean fair)：创建给定的许可数和根据fair判断使用非公平或公平策略。
- acquire()：从信号量获取一个许可，在获取到一个许可前一直将线程阻塞，或者线程被中断。
- acquire(int n)：从信号量获取给定数目的许可，在提供这些许可前一直将线程阻塞，或者线程已被中断。
- release()：释放一个许可，将其返回给信号量。
- release(int n)：释放给定数目的许可，将其返回到信号量。
- availablePermits()：返回此信号量中当前可用的许可数。


接下来，开始撸源码！
```java
public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        //获取当前许可数
        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (; ; ) {
                //获取当前许可数
                int available = getState();
                //剩余许可数
                int remaining = available - acquires;
                //如果剩余许可数小于0或者通过CAS操作成功设置剩余许可数为当前许可数则返回剩余许可数
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        //尝试释放共享锁
        protected final boolean tryReleaseShared(int releases) {
            for (; ; ) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        //通过CAS自旋减少许可数
        final void reducePermits(int reductions) {
            for (; ; ) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        //通过CAS自旋清空许可数
        final int drainPermits() {
            for (; ; ) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (; ; ) {
                //获取共享锁与非公平模式的唯一不同就是，公平模式下会判断是否有线程在等待队列中
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    //创建给定的许可数和使用默认非公平模式。
    public Semaphore(int permits) {
        sync = new Semaphore.NonfairSync(permits);
    }

    //创建给定的许可数和根据fair判断使用非公平或公平模式。
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new Semaphore.FairSync(permits) : new Semaphore.NonfairSync(permits);
    }

    //从信号量获取一个许可，在获取到一个许可前一直将线程阻塞，或者线程被中断
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //从信号量获取一个许可，不会中断
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }

    //尝试获取一个许可
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    //尝试在给定时间内获取一个许可
    public boolean tryAcquire(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //尝试获取给定数量的许可数
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

    //尝试在规定时间内获取给定的许可数
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
            throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }

    //释放一个许可
    public void release() {
        sync.releaseShared(1);
    }

    //释放给定数量的许可
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    //从信号量获取给定许可，不会中断
    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }

    //释放给定数量的许可数
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

    //获取当前可用的许可数
    public int availablePermits() {
        return sync.getPermits();
    }

    //清空许可数
    public int drainPermits() {
        return sync.drainPermits();
    }

    //减少许可数
    protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }

    //是否使用公平模式
    public boolean isFair() {
        return sync instanceof Semaphore.FairSync;
    }

    //判断是否有等待线程
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    //获取等待队列线程的数量
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    //获取等待队列线程集合
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    public String toString() {
        return super.toString() + "[Permits = " + sync.getPermits() + "]";
    }
}
```



参考资料：https://blog.csdn.net/caoxiaohong1005/article/details/80000062
参考书籍：《Java并发编程的艺术》




