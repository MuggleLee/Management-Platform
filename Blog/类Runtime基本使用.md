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

```
