# Runtime的基本使用

日常开发中，Runtime类并不常用，所以我对Runtime的基本使用并不了解，借此篇文章加深对Runtime类理解。

Runtime类代表Java程序的运行时环境，每当一个JVM进程启动的时候都会存在一个Runtime对象，随着JVM的存在而存在的。但由于类Runtime的构造器是私有化的，所以不能实例化对象。Runtime类提供了一个静态方法getRuntime()返回一个Runtime对象。

```java
    public static Runtime getRuntime() {
        return currentRuntime;
    }

    private Runtime() {}
```

然后可通过Runtime对象调用一些有趣的方法。

```java
//返回 Java 虚拟机中的空闲内存量，供将来分配对象使用的当前可用内存的近似总量，以字节为单位。
public native long freeMemory();
//返回 Java 虚拟机中的内存总量，目前为当前和后续对象提供的内存总量，以字节为单位。
public native long totalMemory();
//返回 Java 虚拟机试图使用的最大内存量，如果内存本身没有限制，则返回值 Long.MAX_VALUE，以字节为单位。
public native long maxMemory();
// Java 虚拟机的可用的处理器数量
public native int availableProcessors();
```

实例：
```java
public class RuntimeExample {
    public static void main(String[] args) throws IOException {
        Runtime runtime = Runtime.getRuntime();
        System.out.println("Java 虚拟机的可用的处理器数量: " + runtime.availableProcessors());
        System.out.println("Java 虚拟机中的空闲内存量: " + runtime.freeMemory());
        System.out.println("Java 虚拟机中的内存总量: " + runtime.totalMemory());
        System.out.println("Java 虚拟机试图使用的最大内存量: " + runtime.maxMemory());
    }
}
```
输出结果：
```java
Java 虚拟机的可用的处理器数量: 6
Java 虚拟机中的空闲内存量: 124186000
Java 虚拟机中的内存总量: 126877696
Java 虚拟机试图使用的最大内存量: 1870135296
```
