请实现一个定时任务调度器，有很多任务，每个任务都有一个时间戳，任务会在该时间点开始执行。

定时执行任务是一个很常见的需求，所以这题也是从实践中提炼出来的，做好了将来说不定能用上。

仔细分析一下，这题可以分为三个部分：

* 优先队列。因为多个任务需要按照时间从小到大排序，所以需要用优先队列。
* 生产者。不断往队列里塞任务。
* 消费者。如果队列里有任务过期了，则取出来执行该任务。

如何实现上述要求呢？


### 方案1: PriorityBlockingQueue + Polling

我们很快可以想到第一个办法：

* 用一个`java.util.concurrent.PriorityBlockingQueue`来作为优先队列。因为我们需要一个优先队列，又需要线程安全，用`PriorityBlockingQueue`再合适不过了。你也可以手工实现一个自己的`PriorityBlockingQueue`，用`java.util.PriorityQueue` + `ReentrantLock`，用一把锁把这个队列保护起来，就是线程安全的啦
* 对于生产者，可以用一个`while(true)`，造一些随机任务塞进去
* 对于消费者，起一个线程，在 `while(true)`里每隔几秒检查一下队列，如果有任务，则取出来执行。

这个方案的确可行，总结起来就是**轮询(polling)**。轮询通常有个很大的缺点，就是时间间隔不好设置，间隔太长，任务无法及时处理，间隔太短，会很耗CPU。


### 方案2: PriorityBlockingQueue + 时间差

可以把方案1改进一下，`while(true)`里的逻辑变成：

* 偷看一下堆顶的元素，但并不取出来，如果该任务过期了，则取出来
* 如果没过期，则计算一下时间差，然后 sleep()该时间差

不再是 sleep() 一个固定间隔了，消除了轮询的缺点。


### 方案3: DelayQueue

方案2虽然已经不错了，但是还可以优化一下，Java里有一个[DelayQueue](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/concurrent/DelayQueue.java)，完全符合题目的要求。DelayQueue 设计得非常巧妙，可以看做是一个特化版的`PriorityBlockingQueue`，它把**计算时间差并让消费者等待该时间差**的功能集成进了队列，消费者不需要关心时间差的事情了，直接在`while(true)`里不断`take()`就行了。

DelayQueue的实现原理见下面的代码。

```java
import java.util.PriorityQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

import static java.util.concurrent.TimeUnit.NANOSECONDS;

public class DelayQueue<E extends Delayed> {
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    private final Condition available = lock.newCondition();
    private Thread leader = null;

    public DelayQueue() {}

    /**
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true}
     * @throws NullPointerException if the specified element is null
     */
    public boolean put(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
}
```

这个代码中有几个要点要注意一下。

**1. put()方法**

```java
if (q.peek() == e) {
    leader = null;
    available.signal();
}
```

如果第一个元素等于刚刚插入进去的元素，说明刚才队列是空的。现在队列里有了一个任务，那么就应该唤醒所有在等待的消费者线程。将`leader`重置为null，这些消费者之间互相竞争，自然有一个会被选为leader。


**2. 线程leader的作用**

`leader`这个成员有啥作用？它主要是为了减少不必要的等待时间。比如队列头部还有5秒就要开始了，那么就让消费者线程sleep 5秒，消费者不再需要等待固定的时间间隔了。

想象一下有个多个消费者线程用take方法去取任务,内部先加锁,然后每个线程都去peek头节点。如果leader不为空说明已经有线程在取了，让当前消费者无限等待。

```java
if (leader != null)
   available.await();
```

如果为空说明没有其他消费者去取任务,设置leader为当前消费者，并让改消费者等待指定的时间，

```java
else {
    Thread thisThread = Thread.currentThread();
    leader = thisThread;
    try {
         available.awaitNanos(delay);
    } finally {
         if (leader == thisThread)
             leader = null;
    }
}
```

下次循环会走如下分支，取到任务结束，

```java
if (delay <= 0)
    return q.poll();
```

**3. take()方法中为什么释放first**

```java
first = null; // don't retain ref while waiting
```

我们可以看到 Doug Lea 后面写的注释，那么这行代码有什么用呢？

如果删除这行代码，会发生什么呢？假设现在有3个消费者线程，

* 线程A进来获取first,然后进入 else 的 else ,设置了leader为当前线程A，并让A等待一段时间
* 线程B进来获取first, 进入else的阻塞操作,然后无限期等待，这时线程B是持有first引用的
* 线程A等待指定时间后被唤醒，获取对象成功，出队，这个对象理应被GC回收，但是它还被线程B持有着，GC链可达，所以不能回收这个first
* 只要线程B无限期的睡眠，那么这个本该被回收的对象就不能被GC销毁掉，那么就会造成内存泄露


#### Task对象

```java
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

public class Task implements Delayed {
    private String name;
    private long startTime;  // milliseconds

    public Task(String name, long delay) {
        this.name = name;
        this.startTime = System.currentTimeMillis() + delay;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long diff = startTime - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return (int)(this.startTime - ((Task) o).startTime);
    }

    @Override
    public String toString() {
        return "task " + name + " at " + startTime;
    }
}
```

JDK中有一个接口`java.util.concurrent.Delayed`，可以用于表示具有过期时间的元素，刚好可以拿来表示任务这个概念。


#### 生产者

```java
import java.util.Random;
import java.util.UUID;

public class TaskProducer implements Runnable {
    private final Random random = new Random();
    private DelayQueue<Task> q;

    public TaskProducer(DelayQueue<Task> q) {
        this.q = q;
    }

    @Override
    public void run() {
        while (true) {
            try {
                int delay = random.nextInt(10000);
                Task task = new Task(UUID.randomUUID().toString(), delay);
                System.out.println("Put " + task);
                q.put(task);
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

生产者很简单，就是一个死循环，不断地产生一些是时间随机的任务。


#### 消费者

```java
public class TaskConsumer implements Runnable {
    private DelayQueue<Task> q;

    public TaskConsumer(DelayQueue<Task> q) {
        this.q = q;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Task task = q.take();
                System.out.println("Take " + task);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

当 DelayQueue 里没有任务时，`TaskConsumer`会无限等待，直到被唤醒，因此它不会消耗CPU。


#### 定时任务调度器

```java
public class TaskScheduler {
    public static void main(String[] args) {
        DelayQueue<Task> queue = new DelayQueue<>();
        new Thread(new TaskProducer(queue), "Producer Thread").start();
        new Thread(new TaskConsumer(queue), "Consumer Thread").start();
    }
}
```

DelayQueue这个方案，每个消费者线程只需要等待所需要的时间差，因此响应速度更快。它内部用了一个优先队列，所以插入和删除的时间复杂度都是$$\log n$$。

JDK里还有一个[ScheduledThreadPoolExecutor](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/concurrent/ScheduledThreadPoolExecutor.java)，原理跟DelayQueue类似，封装的更完善，平时工作中可以用它，不过面试中，还是拿DelayQueue来讲吧，它封装得比较薄，容易讲清楚原理。


### 方案4: HashedWheelTimer

TODO


**Follow up: 如何设计一个分布式的定时任务调度器呢？**


### 参考资料

* [java.util.concurrent.DelayQueue](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/concurrent/DelayQueue.java)
* [HashedWheelTimer.java - Github](https://github.com/netty/netty/blob/4.1/common/src/main/java/io/netty/util/HashedWheelTimer.java)
* [delayQueue原理理解之源码解析 - 简书](http://www.jianshu.com/p/e0bcc9eae0ae)
* [细说延时任务的处理 - 简书](http://www.jianshu.com/p/7beebbc61229)
* [延迟任务的实现总结 - nick hao - 博客园](http://www.cnblogs.com/haoxinyue/p/6663720.html)
* [定时器（Timer）的实现](http://novoland.github.io/%E5%B9%B6%E5%8F%91/2014/07/26/%E5%AE%9A%E6%97%B6%E5%99%A8%EF%BC%88Timer%EF%BC%89%E7%9A%84%E5%AE%9E%E7%8E%B0.html)
* [java.util.concurrent.DelayQueue Example](https://examples.javacodegeeks.com/core-java/util/concurrent/delayqueue/java-util-concurrent-delayqueue-example/)
* [HashedWheelTimer 原理 - ZimZz - 博客园](http://www.cnblogs.com/zemliu/p/3928285.html)
* [Hash算法系列-具体算法（HashedWheelTimer） - CSDN](http://blog.csdn.net/yq76034150/article/details/6783398)
