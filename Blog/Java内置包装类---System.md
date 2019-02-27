在刚开始学习Java的时候，我想每一个人都有写过System.out.println("Hello World !")吧！那为什么输入这行代码就能在控制台输出结果呢？System还有其它什么方法呢？接下来，我根据源码结合日常常用的System方法进行简单的剖析。


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
其中，src 表示源数组，srcPos 表示	，dest 表示目标数组，destPos 表示要复制到的目标数组的起始位置，length 表示复制的个数


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
常用的系统属性：
|系统属性|描述|
|-|-|
|java.version|Java 运行时环境版本|
|java.home|Java 安装目录|
|java.class.path|Java 类路径|
|os.name|操作系统的名称|
|file.separator|文件分隔符（在 UNIX 系统中是“/”）|
|path.separator|路径分隔符（在 UNIX 系统中是“:”）|
|line.separator|行分隔符（在 UNIX 系统中是“/n”）|
|user.name|用户的账户名称|
|user.dir|用户的当前工作目录|

譬如想要获取用户的当前工作目录
示例：
```java
public class SystemExample {
    public static void main(String[] args) {
        String userDir = System.getProperty("user.dir");
        System.out.println(userDir);
    }
}
```


**3.exit() 方法**
```java
public static void exit(int status) {
    Runtime.getRuntime().exit(status);
}
```
执行该方法可以终止目前正在运行的Java虚拟机。参数为"0"代表正常终止，参数不为"0"代表异常终止。

<font color="red" size="2px">***这是唯一一个能够退出程序并不执行finally的情况。**</font>

示例：
```java
public class SystemExample {
    public static void main(String[] args) {
        System.out.println("Hello World !");
        System.exit(0);
        System.out.println("Hello Java");
    }
}
```
输出结果：
```java
Hello World !
```
由输出结果可知，执行System.exit(0)终止了JVM，所以下面的"Hello Java"没有执行到。

**4.currentTimeMillis()方法**
```java
public static native long currentTimeMillis();
```
这个方法是获取当前系统的时间戳，时间的格式为当前系统时间与 GMT 时间（格林尼治时间）1970 年 1 月 1 日 0 时 0 分 0 秒所差的毫秒数。
示例：
```java
public class SystemExample {
    public static void main(String[] args) {
        long time = System.currentTimeMillis();
        System.out.println(time);
    }
}
```



参考资料：http://c.biancheng.net/view/904.html















