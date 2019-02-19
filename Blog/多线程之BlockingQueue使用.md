# BlockingQueue


简介：BlockingQueue是一个先进先出的阻塞队列。常用于生产者消费者的场景下。


BlockingQueue继承了Queue接口，所以BlockingQueue接口除了继承Queue接口的add()、offer()、remove()、poll()、element()、peek()之外，还额外添加了put()、take()、remainingCapacity()、contains()、drainTo()方法

BlockingQueue继承了Queue接口，所以在了解BlockingQueue接口之前，先了解一下Queue接口有哪些抽象方法。

```java
public interface Queue<E> extends Collection<E> {

    boolean add(E e);////将一个非空非null元素插入到该队列，如果插入成功返回true,不成功抛出异常

    boolean offer(E e);////将一个非空非null元素插入到该队列，如果插入成功返回true,不成功返回false

    E remove();//删除当前队列的头部元素，并返回头部元素,如果为空，抛出异常

    E poll();//删除当前队列的头部元素，并返回头部元素,如果为空，返回null

    E element();//获取当前队列的头部元素，如果为空，抛出异常

    E peek();//获取当前队列的头部元素，如果为空，返回null
}
```


BlockingQueue接口继承Queue接口方法基础上，还额外添加了如下几个方法：
```java
public interface BlockingQueue<E> extends Queue<E> {

    //将给定元素设置到队列中，如果设置成功返回true, 否则返回false。
    boolean add(E e);

    //将给定的元素设置到队列中，如果设置成功返回true, 否则返回false. 如果e值为空则抛出空指针异常。
    boolean offer(E e);

    //将元素设置到队列中，如果队列中没有多余的空间，该方法会一直阻塞，直到队列中有多余的空间。
    void put(E e) throws InterruptedException;

    //将元素设置到队列中，如果队列中没有多余的空间，该方法会一直阻塞，直到队列中有多余的空间。
    boolean offer(E e, long timeout, TimeUnit unit)
            throws InterruptedException;

    //从队列中获取值。如果队列中没有值，线程会一直阻塞，直到队列中有值，并且该方法取得了该值。
    E take() throws InterruptedException;

    //在给定的时间里，从队列中获取值，如果没有取到会抛出异常。
    E poll(long timeout, TimeUnit unit)
            throws InterruptedException;

    //获取队列中剩余的空间
    int remainingCapacity();

    //从队列中移除指定的值。
    boolean remove(Object o);

    //判断队列中是否拥有该值。
    public boolean contains(Object o);

    //将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c);

    //指定最多数量限制将队列中值，全部移除，并发设置到给定的集合中。
    int drainTo(Collection<? super E> c, int maxElements);
}

```

总结归纳如下：
||抛出异常|返回特殊值|阻塞|超时|
|-|-|-|-|-|
|插入|add|offer|put|offer(e,timeout,unit)|
|移除|remove|poll|take|poll(time, unit)|
|检查|element|peek/contains|



