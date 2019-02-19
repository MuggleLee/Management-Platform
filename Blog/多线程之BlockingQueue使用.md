# BlockingQueue


简介：BlockingQueue是一个先进先出的阻塞队列。常用于生产者消费者的场景下。


BlockingQueue继承了Queue接口，所以BlockingQueue接口除了继承Queue接口的add()、offer()、remove()、poll()、element()、peek()之外，还额外添加了put()、take()、remainingCapacity()、contains()、drainTo()方法

BlockingQueue继承了Queue接口，所以在了解BlockingQueue接口之前，先了解一下Queue接口有哪些抽象方法。
