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

    boolean add(E e);

    boolean offer(E e);

    void put(E e) throws InterruptedException;

    boolean offer(E e, long timeout, TimeUnit unit)
            throws InterruptedException;

    E take() throws InterruptedException;

    E poll(long timeout, TimeUnit unit)
            throws InterruptedException;

    int remainingCapacity();

    boolean remove(Object o);

    public boolean contains(Object o);

    int drainTo(Collection<? super E> c);

    int drainTo(Collection<? super E> c, int maxElements);
}

```

总结归纳成表格如下：
||抛出异常|返回特殊值|返回特殊值|
|-|-|-|-|
|插入|add|offer|put|
|移除|remove|poll|take|
|检查|element|peek|



