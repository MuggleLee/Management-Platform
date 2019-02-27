我猜大家平常用得最多的就是System.out.println()这行代码了吧！那为什么输入这行代码就能在控制台输出结果呢？System还有其它什么方法呢？接下来，我根据源码结合日常常用的System方法进行简单的剖析。


首先就从最熟悉的System.out.println()这行代码入手吧！

System类中包含3个成员变量，分别代表标准输入流(in)，标准输出流(out)和标准错误输出流(err)

```java
public final static InputStream in = null;
public final static PrintStream out = null;
public final static PrintStream err = null;
```

由于这3个变量都是静态的，所以System.out.println()这行代码实际上是成员变量out调用PrintStream类的println方法。（现在先不过多介绍PrintStream，就知道println方法是可以输出字段的）

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

在initializeSystemClass方法中，分别调用setIn0，setOut0，setErr0这几个native方法来初始化对应的成员变量。

那System类中还有什么常用的方法呢？

**1.arraycopy()方法**

```java
public static native void arraycopy(Object src,int srcPos,Object dest,int destPos,int length);
```
其中，src 表示源数组，srcPos 表示从源数组中复制的起始位置，dest 表示目标数组，destPos 表示要复制到的目标数组的起始位置，length 表示复制的个数


示例：
```java
public class ArrayCopyExample {
    public static void main(String[] args) {
        //原数组
        Object[] destArray = {1,2,3,4,5,6,7,8};
        //目标数组
        Object[] destPosArray = {"a","b","c","d","e"};
        System.arraycopy(destArray,0,destPosArray,0,3);
        for (int i = 0; i < destPosArray.length; i++) {
            System.out.println(destPosArray[i]);
        }
    }
}
```

输出结果：
```java
1
2
3
d
e
```

**2.getProperty()和getProperties()方法**
```java
   public static String getProperty(String key) {
        checkKey(key);
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertyAccess(key);
        }
        return props.getProperty(key);
    }
    public static Properties getProperties() {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertiesAccess();
        }
        return props;
    }
```


通过执行getProperty()方法传入key参数可以获取系统属性。那可以获取哪些系统属性呢？
可以调用getProperties()方法查看System这个类可以获取哪些系统属性。

```java
public class SystemExample {
    public static void main(String[] args) {
        Properties properties = System.getProperties();
        Set set = properties.stringPropertyNames();
        System.out.println(set.size());
        Iterator<String> it = set.iterator();
        while(it.hasNext()){
            String name = it.next();
            System.out.println(name);
        }
    }
}
```

|系统属性|column2|
|-|-|
|content1|content2|







参考资料：http://c.biancheng.net/view/904.html















