## Java并发-简单的有界阻塞队列实现wait-notify

> &emsp;一个简单的有界阻塞队列的实现来介绍阻塞队列的基本原理。<br>
> - 需要使用一把锁来实现阻塞
> - 需要一个容器来存放对象
> - 需要一个计数器，以及最大存放量来控制有界队列盛放状态
> - 构造有界队列时指定最大的可放入量 <br>
>


##### 示例

```Java
public class SimpleBlockingQueue {

    /**
     * 锁
     */
    private final Object lock = new Object();

    /**
     * 定义队列容器
     */
    private final LinkedList container = new LinkedList();
//
    /**
     * 定义计数器
     */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * 容器最小存放量
     */
    private final int minSize = 0;

    /**
     * 容器最大存放量
     */
    private final int maxSize;

    /**
     * 构造方法指定容器最大大小
     */
    public SimpleBlockingQueue(int size) {
        this.maxSize = size;
    }

    /**
     * 放入方法
     */
    public void put(Object obj) {
        synchronized (lock) {
            if (count.get() == maxSize) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("新加入元素：" + obj);

            /**
             * 添加新元素到容器中
             */
            container.add(obj);

            /**
             * 处理计数器
             */
            count.incrementAndGet();

            /**
             * 当容器为空时，执行take方法的线程会一直wait，当添加一个元素进去后需要通知其拿走
             */
            lock.notify();
        }
    }

    /**
     * 取走方法
     */
    public Object take() {

        Object ret = null;
        synchronized (lock) {
            if (count.get() == minSize) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            /**
             * 删除并返回容器中的第一个元素
             */
            ret = container.removeFirst();

            System.out.println("取走元素：" + ret);

            /**
             * 处理计数器
             */
            count.decrementAndGet();

            /**
             * 当容器被塞满时，put方法会一直wait，当拿走一个元素后需要通知其放入
             */
            lock.notify();

            return ret;
        }
    }

    /**
     * 获得当前容器大小的方法
     */
    public int getSize() {
        return count.get();
    }

    public static void main(String[] args) {
        SimpleBlockingQueue queue = new SimpleBlockingQueue(5);
        queue.put("1");
        queue.put("2");
        queue.put("3");
        queue.put("4");
        queue.put("5");

        System.out.println("当前queue长度:" + queue.getSize());

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                queue.put("6");
                queue.put("7");
            }
        }, "t1");

        t1.start();

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                queue.take();
                try {
                    /**
                     * 为了演示拿走一个放入一个的效果，为拿走操作增加耗时
                     */
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                queue.take();
            }
        }, "t2");

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        t2.start();
    }
}

输出：
新加入元素：1
新加入元素：2
新加入元素：3
新加入元素：4
新加入元素：5
当前queue长度:5
取走元素：1
新加入元素：6
取走元素：2
新加入元素：7
```
