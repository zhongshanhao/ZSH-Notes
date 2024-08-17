## 线程池

### 使用线程池的好处

使用线程池主要有以下三个原因

- 创建/销毁线程需要消耗系统资源，线程池可以复用已创建的线程。
- 控制并发的数量。并发数量过多，可能会导致资源消耗过多，从而造成服务器崩溃。（主要原因）
- 可以对线程做统一管理。

创建线程需要时间，每次来一个任务都要创建一个线程，任务执行完毕就结束线程，这极大地消耗系统资源，我们可以使用线程池创建线程，并复用这些线程。

CPU核心数有限，过多的线程不但不会提高系统性能，反而会导致过多的线程切换，频繁的线程上下文切换会导致系统性能下降。使用线程池可以固定申请几个线程，如果当前无可用线程，就先将任务丢到阻塞队列中等待，等到有线程可用的时候才将任务分配给线程。

### ThreadPoolExecutor

![线程池](https://gitee.com/Krains/FigureBed/raw/master/img/%E7%BA%BF%E7%A8%8B%E6%B1%A0.png)

ThreadPoolExecutor是Java线程池的实现，其子类有ScheduleThreadPoolExecutor定时任务线程池。

将从以下4个部分详细讲解线程池的运行机制

- 线程池如何维护自身状态
- 线程池如何管理任务
- 线程池如何管理线程
- 线程池的构造

#### 线程池生命周期管理

ThreadPoolExecutor使用int的高3位来表示线程池状态，低29位表示线程数量，这些信息存储在一个AtomicInteger原子变量 ctl 中，目的是将线程池状态与线程个数合二为一，这样就可以用一次 CAS 原子操作。

```java
// c 为旧值, ctlOf 返回结果为新值
ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))));
// rs 为高 3 位代表线程池状态, wc 为低 29 位代表线程个数,ctl 是合并它们
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池状态如下

| 状态名     | 高3位 | 接收新任务 | 处理队列任务 | 状态转移                        | 说明                                     |
| ---------- | ----- | ---------- | ------------ | ------------------------------- | ---------------------------------------- |
| RUNNING    | 111   | Y          | Y            | 新建时处于该状态                | 线程池正在运行状态                       |
| SHUTDOWN   | 000   | N          | Y            | 调用shutdown()方                | 不会接收新任务，但会处理阻塞队列剩余任务 |
| STOP       | 001   | N          | N            | 调用shutdownNow()               | 会中断正在执行的任务，并抛弃阻塞队列任务 |
| TIDYING    | 010   | -          | -            | 任务执行完毕                    | 任务全执行完毕，活动线程为0即将进入终结  |
| TERMINATED | 011   | -          | -            | 处于TIDYING后执行完terminated() | 终结状态                                 |

生命周期转换图

![582d1606d57ff99aa0e5f8fc59c7819329028](https://gitee.com/Krains/FigureBed/raw/master/img/582d1606d57ff99aa0e5f8fc59c7819329028.png)

#### 构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

- corePoolSize 核心线程数目 (最多保留的线程数)
  
  >  核心线程：线程池中有核心线程和非核心线程，非核心线程数 = maximumPoolSize - corePoolSize。核心线程默认情况下会一直存在于线程池中，即使这个核心线程没有任务，而非核心线程如果长时间闲置，就会被销毁。
  
- maximumPoolSize 最大线程数目

- keepAliveTime 生存时间 - 针对非核心线程

- unit 时间单位 - 针对非核心线程

- workQueue 阻塞队列，维护等待执行的Runnable任务对象

  > 常用的几个阻塞队列：
  >
  > LinkedBlockingQueue：底层是链表，默认大小是Integer.MAX_VALUE，也可以指定大小
  >
  > ArrayBlockingQueue：底层是数组，需要指定大小
  >
  > SynchronousQueue：同步队列，内部容量为0，每个put操作必须等待一个take操作
  >
  > DelayQueue：延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素

- threadFactory 创建线程的工程，用于批量创建线程，统一在创建线程时设置一些参数，如是否守护线程，线程名字、优先级等。

- handler 拒绝策略

  > 拒绝策略 jdk 提供了 4 种实现类，实现类实现RejectedExecutionHandler接口，其它著名框架也提供了实现
  >
  > - AbortPolicy 让调用者抛出 RejectedExecutionException 异常，这是默认策略
  >  - CallerRunsPolicy 让调用者运行任务
  >  - DiscardPolicy 放弃本次任务
  >  - DiscardOldestPolicy 放弃队列最前面的任务，重新执行被拒绝的任务
  >  - Dubbo 的实现，在抛出 RejectedExecutionException异常之前会记录日志，并dump线程栈信息，方便定位问题
  >  - Netty 的实现，创建一个新线程来执行任务
  >  - ActiveMQ 的实现，带超时等待(60s)尝试放入队列，类似我们之前自定义的拒绝策略
  >  - PinPoint 的实现，它使用了一个拒绝策略链，可以将上述策略组合起来形成一种策略

#### Executors帮助创建线程池

根据ThreadPoolExecutor的构造方法，JDK Executors 类中提供了众多工厂方法来帮助创建各种用途的线程池

newFixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

特点

- 核心线程数等于最大线程数，没有救急线程，因此也无需设定超时时间
- 阻塞队列是无界的，可以放任意数量的任务
- 适用于任务量已知，相对耗时的任务

newCachedThreadPool

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

特点

- 核心线程数是0，最大线程数是Integer.MAX_VALUE，救急线程的空闲生存时间是60s，意味着全部都是救急线程，可以被无限创建，线程的空闲生成时间是60S
- 队列采用了 SynchronousQueue 实现，特点是，它没有容量，没有线程来取是放不进去的(一手交钱、一手交货)

- 整个线程池表现为线程数会根据任务量不断增长，没有上限，当任务执行完毕，空闲 1分钟后释放线程。 适合任务数比较密集，但每个任务执行时间较短的情况

newSingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor() {
		return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```

使用场景:

希望多个任务排队执行。线程数固定为 1，任务数多于 1 时，会放入无界队列排队。任务执行完毕，这唯一的线程也不会被释放。

与用单线程执行任务的区别:

- 自己创建一个单线程串行执行任务，如果任务执行失败而终止那么没有任何补救措施，而线程池还会新建一个线程，保证池的正常工作
- Executors.newSingleThreadExecutor() 线程个数始终为1，不能修改		 
	- FinalizableDelegatedExecutorService 应用的是装饰器模式，只对外暴露了 ExecutorService 接口，因此不能调用 ThreadPoolExecutor 中特有的方法
- Executors.newFixedThreadPool(1) 初始时为1，以后还可以修改
	- 对外暴露的是 ThreadPoolExecutor 对象，可以强转后调用 setCorePoolSize 等方法进行修改

ScheduledExecutorService

```java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory, handler);
}
```

ExecutorService接口主要方法

提交任务

```java
// 执行任务
void execute(Runnable command);

// 提交任务 task,用返回值 Future 获得任务执行结果
<T> Future<T> submit(Callable<T> task);

// 提交 tasks 中所有任务
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;

// 提交 tasks 中所有任务,带超时时间
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;

// 提交 tasks 中所有任务,哪个任务先成功执行完毕,返回此任务执行结果,其它任务取消
<T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
```

关闭线程池

```java
/*
线程池状态变为 SHUTDOWN
- 不会接收新任务
- 但已提交任务会执行完
- 此方法不会阻塞调用线程的执行
*/
void shutdown();

/*
线程池状态变为 STOP
- 不会接收新任务
- 会将队列中的任务返回
- 并用 interrupt 的方式中断正在执行的任务
*/
List<Runnable> shutdownNow();
```

#### 源码分析

线程池处理任务的核心方法是`execute`，看看JDK1.8是如何处理线程任务的

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        // 获取当前线程运行状态，状态包括线程状态+当前运行线程个数
        int c = ctl.get();
        // 如果当前线程数小于corePoolSize，则调用addWorker创建核心线程执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        
        // 如果核心线程用完了，那么将任务加入阻塞队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次检查线程池是否处于运行状态，如果否，则移除任务并执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 线程池处于running状态，但是没有线程，则创建救急线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 如果放入workQueue失败，则创建非核心线程执行任务，
    	// 如果这时创建非核心线程失败(当前线程总数不小于maximumPoolSize时)，就会执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

`ctl.get()`是获取线程池状态，用`int`类型表示。第二步中，入队前进行了一次`isRunning`判断，入队之后，又进行了一次`isRunning`判断。

为什么要二次检查线程池的状态?

在多线程的环境下，线程池的状态是时刻发生变化的。很有可能刚获取线程池状态后线程池状态就改变了。判断是否将`command`加入`workqueue`是线程池之前的状态。倘若没有二次检查，万一线程池处于非**RUNNING**状态（在多线程环境下很有可能发生），那么`command`永远不会执行。

#### 工作方式

- 线程池中刚开始没有线程，当一个任务提交给线程池后，线程池会创建一个新线程来执行任务。
- 当线程数达到 corePoolSize 并没有线程空闲，这时如果有任务加入，新加的任务会被加入workQueue 队列排队，直到有空闲的线程依次去缓冲队列中取任务来执行。
- 如果队列选择了有界队列，那么任务超过了队列大小时，可以创建 maximumPoolSize - corePoolSize 数目的线程来救急。
- 如果缓存队列满了，且总线程数到达 maximumPoolSize 仍然有新任务这时会执行拒绝策略。

- 当高峰过去后，超过corePoolSize 的救急线程如果一段时间没有任务做，需要结束节省资源，这个时间由  keepAliveTime 和 unit 来控制。

#### 如何复用线程？

线程池中的线程在执行完一个任务之后，并不会马上销毁，而是试图去阻塞队列中获取任务然后执行，从而达到复用线程的目的。如果阻塞队列中没有任务，线程会被阻塞并挂起，此时不会占用CPU资源，直到超时或者拿到任务，如果超时没拿到任务就会被销毁。

自JDK 1.5 开始，JDK提供了`ScheduledThreadPoolExecutor`类用于计划任务（又称定时任务），这个类有两个用途：

- 在给定的延迟之后运行任务
- 周期性重复执行任务

## 阻塞队列

线程池里边使用了阻塞队列，阻塞队列（BlockingQueue）是Java.util.concurrent包下的重要数据结构，阻塞队列提供了线程安全的队列访问方式：

- 当阻塞队列进行插入数据时，如果队列已满，会将当前线程阻塞，直到队列有位置时才将线程唤醒
- 当阻塞队列进行获取数据时，如果队列为空，会将当前线程阻塞，直到队列非空才将线程唤醒

阻塞队列使用ReentrantLock实现，通过该锁实现线程安全访问，使用两个条件变量实现同步阻塞。

BlockingQueue的多种操作方法

|      | 抛异常     | 返回特定值 | 阻塞    | 超时                        |
| ---- | ---------- | ---------- | ------- | --------------------------- |
| 插入 | add(o)     | offer(o)   | put(o)  | offer(o, timeout, timeunit) |
| 移除 | remove(o)  | poll(o)    | take(o) | poll(timeout, timeunit)     |
| 检查 | element(o) | peek(o)    |         |                             |

四组不同的行为方式解释：

- 抛异常：如果试图的操作无法立即执行，抛一个异常。

- 特定值：如果试图的操作无法立即执行，返回一个特定的值(常常是 true / false)。

- 阻塞：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行。

- 超时：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。返回一个特定值以告知该操作是否成功(典型的是true / false)。

几种常用的阻塞队列（BlockingQueue接口的实现类）

- LinkedBlockingQueue：底层是链表，默认大小是Integer.MAX_VALUE，也可以指定大小

- ArrayBlockingQueue：底层是数组，需要指定大小

- SynchronousQueue：同步队列，内部容量为0，每个put操作必须等待一个take操作
- DelayQueue：延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素

以ArrayBlockingQueue为例，讲解阻塞队列的实现原理

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
        	
	// 实际存储内容的地方，是一个数组
	final Object[] items;
        
    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull; 
    
    // 初始化方法
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
    
    // 插入方法
    // 首先获得锁，如果队列满了，则进入条件变量阻塞
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
    
    // 在插入时唤醒处于 notEmpty 条件变量的线程
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
    
    // 取操作
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    
    // 在取操作时唤醒处于 notFull 的线程
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
        
}
```

通过阻塞队列，可以相当简单实现生产者和消费者模式，生产者只需要管生产产品，放入阻塞队列中，消费者只需要从阻塞队列中取出产品，阻塞队列会帮助同步阻塞相关线程，并提供线程安全的操作。

LinkedBlockingQueue使用两把锁提高并发粒度，一把锁供take取数据的时候使用，一把锁供put放数据的时候使用，能够同时take和put。

```java
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

