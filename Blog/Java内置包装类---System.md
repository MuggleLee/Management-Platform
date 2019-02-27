我猜大家平常用得最多的就是System.out.println()这行代码了吧！那为什么输入这行代码就能在控制台输出结果呢？System还有其它什么方法呢？接下来，我根据源码结合日常常用的System方法进行简单的剖析。


首先就从最熟悉的System.out.println()这行代码入手吧！

System类中包含3个静态变量，分别代表标准输入流(in)，标准输出流(out)和标准错误输出流(err)

```java
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
```

由于这3个变量都是静态的，所以System.out.println()这行代码实际上是System类直接

