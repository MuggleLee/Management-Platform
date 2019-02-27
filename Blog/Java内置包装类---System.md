我猜大家平常用得最多的就是System.out.println()这行代码了吧！那为什么输入这行代码就能在控制台输出结果呢？System还有其它什么方法呢？接下来，我根据源码结合日常常用的System方法进行简单的剖析。


首先就从最熟悉的System.out.println()这行代码入手吧！

System类中包含3个成员变量，分别代表标准输入流(in)，标准输出流(out)和标准错误输出流(err)

```java
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
```

由于这3个变量都是静态的，所以System.out.println()这行代码实际上是成员变量out调用PrintStream类的println方法。

但是，这3个成员变量都没有实例化，怎么可以调用类PrintStream中的方法呢？System类中有一个静态方法initializeSystemClass，根据方法名就可以知道是初始化System类的。
```java
    private static void initializeSystemClass() {

        props = new Properties();
        initProperties(props);  // initialized by the VM

        sun.misc.VM.saveAndRemoveProperties(props);

        lineSeparator = props.getProperty("line.separator");
        sun.misc.Version.init();

        FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
        FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
        FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
        setIn0(new BufferedInputStream(fdIn));
        setOut0(newPrintStream(fdOut, props.getProperty("sun.stdout.encoding")));
        setErr0(newPrintStream(fdErr, props.getProperty("sun.stderr.encoding")));

        loadLibrary("zip");

        Terminator.setup();

        sun.misc.VM.initializeOSEnvironment();

        Thread current = Thread.currentThread();
        current.getThreadGroup().add(current);

        setJavaLangAccess();

        sun.misc.VM.booted();
    }

    private static native void setIn0(InputStream in);
    private static native void setOut0(PrintStream out);
    private static native void setErr0(PrintStream err);
```

在initializeSystemClass方法中，调用setIn0，setOut0，setErr0这几个native方法，这几个方法就是分别初始化对应的成员变量。

















