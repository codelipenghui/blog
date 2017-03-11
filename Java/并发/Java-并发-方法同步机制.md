## Java并发-方法同步机制

> &emsp;Java每个对象都会有一个默认的锁，如果访问的方法是同步方法那么会使用对象上的锁，如果对象有多个同步方法，所有方法的调用需要获得对象的锁。<br>

##### 示例

```Java
public class SynMethod {

    synchronized void slowMethod() {
        System.out.println("slowMethod");
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    synchronized void fastMethod() {
        System.out.println("fastMethod");
    }

    public static void main(String[] args) {
        SynMethod method = new SynMethod();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    method.slowMethod();
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    method.fastMethod();
                }
            }
        }).start();

    }
}
```

```Java
输出：
com.intellij.rt.execution.application.AppMain current.SynMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
slowMethod
```

> &emsp;从输出中可以看到fastMethod一直在等待锁。
