**###ReentrantLock
ReentrantLock和Synchronized一样是可重入锁。
>可重入锁，也叫做递归锁。指的是同一线程外层函数获得锁之后 ，内层递归函数仍然有获取该锁的代码，但不受影响。其最大作用是不会产生死锁。

ReentrantLock与synchronized的区别

 1. 与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
 2. ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合（以后会阐述Condition）。
 3. ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
 4. ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。
 5. ReentrantLock支持中断处理，且性能较synchronized会好些。

###ReentrantLock类继承关系图###

![](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentranrLock-Relationship.png)

ReentrantLock有公平锁和非公平锁两种实现方式，默认使用非公平锁。因为可能存在某些线程阻塞时间长而导致整体执行效率低，所以使用非公平锁比使用公平锁的效率要高。

    public ReentrantLock() {
        sync = new NonfairSync();
    } 
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

ReentrantLock类主要有3个子类：Sync，NonfairSync，FairSync。NonfairSync，FairSync分别都继承Sync，而Sync类继承AbstractQueuedSynchronizer。其实可以说Locks接口实现类大多数都是基于AbstractQueuedSynchronizer(AQS，即队列同步器)实现的。

AbstractQueuedSynchronizer的源码先暂时不讲。先看下NonfairSync的源码实现。

    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }



默认情况下，执行lock()方法调用的是非公平锁。当线程调用lock()方法的时候，执行compareAndSetState(int expect, int update)方法。如果原先状态(state)为0，则说明目前没有线程持有锁，那么设置状态为1，之后执行setExclusiveOwnerThread(Thread thread)设置当前线程是当前拥有独占访问的线程；如果compareAndSetState(int expect, int update)方法返回false则代表该锁持有线程，则调用acquire(int arg)方法。

compareAndSetState(int expect, int update)源码：

     protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

getExclusiveOwnerThread()和setExclusiveOwnerThread(Thread thread)源码：

    /**
     * 返回由setExclusiveOwnerThread最后设置的线程，如果从未设置则返回null。
     */
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
    /**
     * 设置当前拥有独占访问权限的线程。 
     */
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

acquire(int arg)源码(由抽象类AbstractQueuedSynchronizer实现)：

     public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

首先调用tryAcquire(int arg)方法，该方法是由子类ReentrantLock实现。
ReentrantLock中非公平锁tryAcquire(int arg)的源码(实际上是调用nonfairTryAcquire方法)：

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
nonfairTryAcquire(int acquires)的源码：

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

执行nonfairTryAcquire方法，执行getState重新尝试获取锁，之后判断状态(state)是否等于0，如果为0则代表获取到锁，执行compareAndSetState改变状态，再执行setExclusiveOwnerThread设置当前线程是当前拥有独占访问的线程；如果不等于0，则执行getExclusiveOwnerThread()判断是否为当前线程。如果是当前线程重复加锁，则执行setState(int newState)添加状态，该线程继续持有锁，state状态叠加加1(可重入)；
如果线程状态既不为0，也不是当前线程持有锁，则返回false。则会先执行addWaiter(Node mode)，将线程追加到队列中并进行阻塞。

    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

再调用acquireQueued(final Node node, int arg)方法。

    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 返回节点的前驱节点
                final Node p = node.predecessor();
                //如果当前的节点是head说明他是队列中第一个“有效的”节点，因此尝试获取
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
acquireQueued的主要作用是把已经追加到队列的线程节点（addWaiter方法返回值）进行阻塞，但阻塞前又通过tryAccquire重试是否能获得锁，如果重试成功能则无需阻塞，直接返回false;如果获取同步状态失败后，则检查该线程的状态，执行shouldParkAfterFailedAcquire(p, node)方法。
shouldParkAfterFailedAcquire(p, node)源码：

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

首先获取该线程在队列中的前驱节点的状态，如果为SINNAL则表明当前线程需要被阻塞，直接返回true；之后调用parkAndCheckInterrupt()方法阻塞线程；如果ws大于0，表明当前线程的前继节点处于CANCELED的状态，则从当前节点开始往前查找，直到找到第一个不为CAECELED状态的节点；如果ws小于0，则代表前驱正常，执行compareAndSetWaitStatus(pred, ws, Node.SIGNAL)把前驱的状态设置成SIGNAL，返回false。
如果执行shouldParkAfterFailedAcquire(p, node)和parkAndCheckInterrupt()都返回true则interrupted = true;

如果返回false则代表当前线程拥有独占线程；如果返回true则代表当前线程被park，需要等待前驱节点设置为unpark唤醒线程。
最后执行selfInterrupt()阻塞当前线程。

     static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }

这就是执行lock()的过程，调用成功就代表该线程获取到锁。lock()与unlock()之间的代码块称为临界区，一般都是对共享变量操作，当对共享变量操作完成之后执行解锁操作，即执行unlock()。

以下是对执行unlock()操作的讲解。

    public void unlock() {
        sync.release(1);
    }
    
实际上是调用父类AQS的release(int arg)方法

    public final boolean release(int arg) {
        //尝试释放锁
        if (tryRelease(arg)) {
            //找到头结点
            Node h = head;
            // waitStatus为0，证明是初始化的空队列或者后继结点已经被唤醒
            if (h != null && h.waitStatus != 0)
                //唤醒等待队列里的下一个线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

首先调用tryRelease(int arg)方法尝试释放锁，该锁实现在ReentrantLock的内部类Sync，而不是分别在FairSync或NonfairSync中，这也说明了释放锁的过程与锁的公平性无关。

    protected final boolean tryRelease(int releases) {
        //每次释放锁状态减1，若为0说明锁已释放完毕
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //若锁已释放，将表示占有锁的线程变量设为null
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
如果执行unlock()方法的线程没完全释放（即state大于0），则返回false，继续执行该线程加锁后的程序。如果该线程完全释放（即state等于0，该线程没被占用）则返回true。
如果tryRelease返回true，则判断等待队列的头部，如果为空和waitStatus等于0，则证明是初始化的空队列或者后继结点已经被唤醒，直接返回false；否则执行unparkSuccessor(Node node)。

     // 唤醒node节点的后继节点
     private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        // 如果node的waitStatus<0，则使用CAS将等待状态改为0，即初始状态（因为下面马上要将node的后继节点唤醒）
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
            // 定义s为node的后继节点
        Node s = node.next;
        // 如果s为null或者waitStatus为CANCELLED
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从后尾部往前遍历找到一个处于正常阻塞状态的结点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒后继节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }

到此为止，使用ReentrantLock非公平锁的lock和unlock过程已经全部剖析结束。
画了一张图理清思路：
![](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantLock_FlowChart.png)