## Java并发-异常处理

##### 1.Java异常捕获try-catch：<br/>
> &emsp;Java异常处理try-catch只能捕获当前线程抛出的异常，无法捕获其他线程抛出的异常，我们看下面的例子：<br>

```Java

public class ExceptionThread implements Runnable {
    @Override
    public void run() {
        throw new RuntimeException();
    }

    public static void main(String[] args) {
        try {
            new Thread(new ExceptionThread()).start();
        } catch (Exception e) {
            System.out.println("catch the exception");
        }
    }
}

output:

Exception in thread "Thread-0" java.lang.RuntimeException
	at current.ExceptionThread.run(ExceptionThread.java:12)
	at java.lang.Thread.run(Thread.java:745)
```

> &emsp;显然异常没能被捕获，那么Jdk会为我们提供怎样的并发异常处理机制，我们先看下面的一个简单的示例，通过两种方式实现并发异常捕获<br>

##### 2.Jdk为我们提供的并发异常捕获机制：<br/>
```Java
/**
 * 一个抛出异常的任务
 */
class MyExceptionThread implements Runnable {

    @Override
    public void run() {
        throw new RuntimeException();
    }
}

/**
 * 自定义的线程异常处理器
 */
class MyUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("catch the exception");
    }
}

/**
 * 一个产生线程的工程，产生的线程设置了自定义的异常处理器
 */
class HandlerThreadFactory implements ThreadFactory {

    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
        return thread;
    }
}

public class CaptureUncaughtException {

    public static void main(String[] args) {

        /**
         * 第一种方式：
         * Executor中的线程通过HandlerThreadFactory产生
         * Executor执行抛出异常的任务
         *
         * @param args
         */
        ExecutorService exec = Executors.newCachedThreadPool(new HandlerThreadFactory());
        exec.execute(new MyExceptionThread());

        /**
         * 第二种方式：
         * 通过设置Thread的默认异常处理器获得相同的功能
         */
        Thread.setDefaultUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
        ExecutorService exec2 = Executors.newCachedThreadPool();
        exec2.execute(new MyExceptionThread());
    }
}
```

> &emsp;通过实现Thread.UncaughtExceptionHandler并编写uncaughtException方法，然后在线程创建时通过thread.setUncaughtExceptionHandler()设置线程的异常处理器或者通过Thread.setDefaultUncaughtExceptionHandler()是设置默认的线程异常处理来实现并发异常捕获，下面我们通过Jdk源码来探其本质：<br>

##### 3.并发异常处理源码解析：<br/>

- Jvm调用线程的异常处理方法

```Java
/**
 * Dispatch an uncaught exception to the handler. This method is
 * intended to be called only by the JVM.
 */
private void dispatchUncaughtException(Throwable e) {
      getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

- 获得线程异常处理器优先级

> &emsp;首先获得通过setUncaughtExceptionHandler设置进来的异常处理器，如果没有就去他所在的线程组去找线程组的异常处理器，因为ThreadGroup实现了Thread.UncaughtExceptionHandler接口


```Java
public UncaughtExceptionHandler getUncaughtExceptionHandler() {
     return uncaughtExceptionHandler != null ?
         uncaughtExceptionHandler : group;
}
```

> &emsp;ThreadGroup中的实现，如果有父级调用父级的uncaughtException()方法，如果没有父级并且有默认异常处理器就调用默认异常处理器的uncaughtException()方法

```Java
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            System.err.print("Exception in thread \""
                             + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

- 设置默认线程异常处理器 Thread.setDefaultUncaughtExceptionHandler()

```Java

private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;

public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        sm.checkPermission(
            new RuntimePermission("setDefaultUncaughtExceptionHandler")
        );
    }
    /**
     * 设置 defaultUncaughtExceptionHandler 为方法参入传入的实现了UncaughtExceptionHandler接口的实例
     */
    defaultUncaughtExceptionHandler = eh;
}
```

##### 4.总结：<br/>

> &emsp;通过寻找异常处理器决定使用那个异常处理器来回调线程中抛出的异常，寻找的优先级为：专有异常处理>线程组专有uncaughtException>默认异常处理器。我们通过编写自己的handler设置线程的异常处理器总而捕获线程中抛出的异常。

完
