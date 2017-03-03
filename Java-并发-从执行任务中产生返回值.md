## Java并发-从执行任务中产生返回值

##### 1.实现方法：<br/>
> &emsp;从Java SE5 引入了参数泛型化的Callable接口, 类型参数表示从方法call()中返回的值，想要在任务执行中拿到任务的返回值必须实现Callable下的call()方法;<br>


> &emsp;必须使用ExecutorService.submit()提交，submit()方法会产生实现了Future接口的对象，他用Callable返回结果进行了参数化，可以调用Future的get()方法来获取任务执行的返回结果

##### 2.示例

```Java
public class CallableTest {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        ArrayList<Future<String>> results = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            results.add(exec.submit(new TaskWithResult(i)));
        }

        for (Future<String> result : results) {
            try {
                System.out.println(result.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
                return;
            } catch (ExecutionException e) {
                e.printStackTrace();
            } finally {
                exec.shutdown();
            }
        }
    }
}

class TaskWithResult implements Callable<String> {

    private int id;

    public TaskWithResult(int id) {
        this.id = id;
    }

    @Override
    public String call() throws Exception {
        return "call id=" + id;
    }
}

输出：
call id=0
call id=1
call id=2
call id=3
call id=4
call id=5
call id=6
call id=7
call id=8
call id=9
```
##### 3.重点源码解析
- ExecutorService.submit()

> &emsp;AbstractExecutorService中共有3个submit方法，如下：

```Java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
> &emsp;调用了newTaskFor()方法来返回一个FutureTask对象，实现了RunnableFuture接口，RunnableFuture接口继承了Future接口和RRunnable接口，newTaskFor()方法源代码如下所示：

```Java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```

> 下面我们来看看FutureTask中的重要方法，run()方法：

```Java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        /**
        * 构造器参数
        */
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
              /**
              * 调用call方法
              */
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                /**
                * 设置返回结果
                */
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

> set()方法源码

```Java
/**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```

> 接下来我们来看看FutureTask中的获得返回值方法，get()方法：

```Java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        /**
        * 阻塞方法，等待结束后再返回
        */
        s = awaitDone(false, 0L);
    return report(s);
}
```

> 到此FutureTask中的重要方法已经介绍完毕，在构造完毕FutureTask后，由于FutureTask实现了Runnable接口，并实现了run()方法，别忘了在构造出来的FutureTask实例交给Executor.execute()去执行。向上见AbstractExecutorService.submit()方法。

完
