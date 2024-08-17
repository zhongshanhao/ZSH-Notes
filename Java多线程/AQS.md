## AQS

全称是 AbstractQueuedSynchronizer，即抽象队列同步器，从字面上可以理解为

- 抽象：抽象类，只实现一些主要逻辑，有些方法则由子类实现，JUC包下许多锁如重入锁、读写锁等都是基于AQS实现的
- 队列：使用先进先出队列存储线程，等待拥有锁的线程释放锁的时候唤醒队列中的线程去竞争锁
- 同步：实现了同步的功能

AQS有什么用？AQS是一个用来构建锁和同步器的框架，使用AQS能够简单高效的构造出应用广泛的同步器，比如ReentrantLock，Semaphore，ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。

特点

用 state 属性来表示资源的状态(分独占模式和共享模式)，子类需要定义如何维护这个同步状态，控制如何获取锁和释放锁

state对于其实现的子类来说有不同的语义

- getState - 获取 state 状态
- setState - 设置 state 状态
- compareAndSetState - CAS 机制设置 state 状态
- 独占模式是只有一个线程能够访问资源，而共享模式可以允许多个线程访问资源

提供了基于FIFO的等待队列，类似与Monitor的EntryList

条件变量来实现等待、唤醒机制，支持多个条件变量，类似与Monitor的WaitSet

子类主要实现这样一些方法(默认抛出 UnsupportedOperationException)来构建同步器

- tryAcquire，独占式获取同步状态
- tryRelease，独占式释放同步状态
- tryAcquireShared，共享式获取同步状态
- tryReleaseShared，共享式释放同步状态
- isHeldExclusively

获取锁方法

```java
// 如果获取锁失败
if (!tryAcquire(arg)) {
	// 入队, 可以选择阻塞当前线程  park unpark
}
```

释放锁方法

```java
// 如果释放锁成功
if (tryRelease(arg)) {
	// 让阻塞线程恢复运行
}
```

阻塞队列结点

```java
static final class Node {
    /** waitStatus值，表示线程已被取消（等待超时或者被中断）*/
    static final int CANCELLED =  1;
    /** waitStatus值，表示后继线程需要被唤醒（unpaking）*/
    static final int SIGNAL = -1;
    /** waitStatus值，表示结点线程等待在condition上，当被signal后，会从等待队列转移到同步到队列中 */
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /** waitStatus值，表示下一次共享式同步状态会被无条件地传播下去 **/
    static final int PROPAGATE = -3;
    /** 等待状态，初始为0 */
    volatile int waitStatus;
    /**当前结点的前驱结点 */
    volatile Node prev;
    /** 当前结点的后继结点 */
    volatile Node next;
    /** 与当前结点关联的排队中的线程 */
    volatile Thread thread;
    /** ...... */
}
```

## AQS在ReentrantLock中的使用

ReentrantLock的实现依赖于AQS，可以从下面的继承关系看出，ReentrantLock的公平锁和非公平锁的实现间接继承了AQS。

![reentrantlock](https://gitee.com/Krains/FigureBed/raw/master/img/reentrantlock.png)

ReentrantLock三个内部类

```java
abstract static class Sync extends AbstractQueuedSynchronizer{...}
static final class NonfairSync extends Sync{...}
static final class FairSync extends Sync{...}
```

下面的代码是`ReentranLock`的函数，我们就以此为顺序，依次讲解这些函数背后的实现原理。

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
lock.unlock();
```

`ReentrantLock`构造方法如下，可以选择公平或者非公平锁

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

以公平锁为例

`ReentranLock`的`lock`函数直接调用了`sync`的`lock`函数。也就是调用了`FairSync`的`lock`函数，`FairSync`调用了父类AQS的`acquire`方法，在`acquire`方法内，做了三个操作

- `tryAcquire(arg)`，看看能不能获取到锁
- `addWaiter(Node.EXCLUSIVE)`，如果不能，则将线程加入到AQS队列中
- `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`，在此处将线程阻塞

```java
// ReentranLock
public void lock() {
    sync.lock();
}

// FairSync
final void lock() {
    acquire(1);
}

// AQS
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### 公平锁和非公平锁、可重入原理

在`tryAcquire`方法中，我们能够看到公平锁和非公平锁、可重入的实现原理

公平锁和非公平锁的区别在于，如果当前锁没有被其他线程占用了

- 公平锁先判断AQS队列中是否已经有其他阻塞的线程，如果有，就不去尝试获取锁操作
- 非公平锁会直接尝试获取锁，不管是否有其他阻塞线程在等待

可重入原理

- 如果拥有该锁的线程又一次尝试获取该锁，那么state将会加1，state的数值表示该锁被重入的次数
- 释放的时候会将state减1，只有当state减为0的时候才释放锁

我们根据`acquire`方法的条件判断依次往下看，`tryAcquire`方法中，在与非公平锁不同，公平锁的`tryAcquire`方法仅仅多了`!hasQueuedPredecessors()`这个条件判断，如果没有阻塞的线程，那么当前线程才开始尝试获取锁

```java
//AQS类中的变量.
private volatile int state;
// state == 0 表示该锁没有被占用
// state >= 1 表示该锁已被占用，c的值表示被重入的次数

// 这是FairSync的实现,AQS中未实现,子类按照自己的需要实现该函数
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();

    int c = getState();
    if (c == 0) {
         // 如果当前阻塞队列上没有先来的线程在等待,UnfairSync这里的实现就不一致
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 锁可重入原理
        // 如果当前持有锁的线程就是它自己，那么增加重入次数
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 获取锁失败
    return false;
}
```

addWaiter

如果线程调用`tryAcquire`返回的是`false`，则会进入`addWaiter`方法，将线程加入到`AQS`的阻塞队列中

```java
// cas方法将线程加入到阻塞队列中
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

`acquireQueued`

通过调用`addWaiter`函数，`AQS`将当前线程加入到了等待队列，但是还没有阻塞当前线程的执行，接下来我们就来分析一下`acquireQueued`函数。

由于进入阻塞状态的操作会降低执行效率，所以，`AQS`会尽力避免试图获取独占性变量的线程进入阻塞状态。所以，当线程加入等待队列之后，`acquireQueued`会执行一个for循环，每次都判断当前节点是否应该获得这个变量(在队首了)。如果不应该获取或在再次尝试获取失败，那么就调用`shouldParkAfterFailedAcquire`判断是否应该进入阻塞状态。如果当前节点之前的节点已经进入阻塞状态了，那么就可以判定当前节点不可能获取到锁，为了防止CPU不停的执行for循环，消耗CPU资源，调用`parkAndCheckInterrupt`函数来进入阻塞状态。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { //一直执行,直到获取锁,返回.
            final Node p = node.predecessor(); 
            //node的前驱是head,就说明,node是将要获取锁的下一个节点.
            if (p == head && tryAcquire(arg)) { //所以再次尝试获取独占性变量
                setHead(node); //如果成功,那么就将自己设置为head
                p.next = null; // help GC
                failed = false;
                return interrupted;
                //此时,还没有进入阻塞状态,所以直接返回false,表示不需要中断调用selfInterrupt函数
            }
            //判断是否要进入阻塞状态.如果`shouldParkAfterFailedAcquire`
            //返回true,表示需要进入阻塞
            //调用parkAndCheckInterrupt；否则表示还可以再次尝试获取锁,继续进行for循环
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //调用parkAndCheckInterrupt进行阻塞,然后返回是否为中断状态
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) //前一个节点在等待独占性变量释放的通知,所以,当前节点可以阻塞
        return true;
    if (ws > 0) { //前一个节点处于取消获取独占性变量的状态,所以,可以跳过去
        //返回false
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //将上一个节点的状态设置为signal,返回false,
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); //将AQS对象自己传入
    return Thread.interrupted();
}
```

阻塞和中断

由上述分析，我们知道了`AQS`底层是通过调用`LockSupport`的`park`方法来使线程阻塞的，等待对应的`unpark`操作线程可继续往下执行。

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);	// 设置阻塞对象,用来记录线程被谁阻塞的,用于线程监控和分析工具来定位
    UNSAFE.park(false, 0L);	// UNSAFE的park方法调用的是native方法
    setBlocker(t, null);
}
```

unlock操作

`ReentrantLock`调用了`unlock`方法，其实调用的是父类`AQS`的`release`方法

```java
// ReentrantLock
public void unlock() {
    sync.release(1);
}

// AQS
public final boolean release(int arg) {
    if (tryRelease(arg)) { 
    	// 释放独占性变量,起始就是将status的值减1,因为acquire时是加1
        // 查看阻塞队列，
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//唤醒head的后继节点
        return true;
    }
    return false;
}

// sync 尝试释放锁，返回成功与否，releases是1，表示释放一次锁，可能有重入的情况，所以可能会失败
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

试图找到一个未被取消的线程，将其唤醒

```java
private void unparkSuccessor(Node node) {
	// ...
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

调用了`unpark`方法后，进入`lock`操作被阻塞的线程就恢复到运行状态,就会再次执行`acquireQueued`中的无限for循环中的操作，再次尝试获取锁。

### 可打断原理

在加锁的时候使用`lockInterruptibly`方法，被打断时抛出异常，使得线程可以继续执行下去，我们来看看它是怎么实现的

```java
ReentrantLock lock = new ReentrantLock();
try {
    // 被打断时抛出异常，使得线程可以继续执行下去
	lock.lockInterruptibly();
} catch (InterruptedException e) {
	e.printStackTrace();
} finally {	
	lock.unlock();
}

// ReentrantLock
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

// AQS
public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取锁，如果获取锁失败，获取可打断锁
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

在`doAcquireInterruptibly`方法中，首先将当前线程加入到阻塞队列中，然后进入一个死循环，如果没有获取到锁，就判断当前是否应该阻塞`shouldParkAfterFailedAcquire`，如果返回`true`，那么就`park`

```java
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        // 将当前线程加入阻塞队列中，
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //将AQS对象自己传入
        return Thread.interrupted();
    }
```

### 锁超时原理

可以使用带时间的`tryLock`尝试加锁，底层调用`LockSupport.parkNanos(this, nanosTimeout)`，在`nanosTimeout`的时间内如果能够获取到锁就往下执行，如果没有就放弃获取。

```java
lock.tryLock(1, TimeUnit.DAYS);
```

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            // 如果时间到了，就返回false，表示获取锁失败
            if (nanosTimeout <= 0L)
                return false;
            // 如果需要被阻塞，且剩下的时间大于自旋时间阈值，就需要调用 parkNanos 阻塞 nanosTimeout 个时间
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 条件变量原理

等待队列

ConditionObject是同步器AQS的内部类，每个Condition对象都包含一个等待队列，一个AQS可以拥有多一个Condition对象，当

- 调用`await`方法，将会以当前线程构造节点，将节点从尾部加入等待队列
- 调用`signal`方法，将会将等待队列中的头节点转移到阻塞队列的尾部

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210808203655437.png" alt="image-20210808203655437" style="zoom:80%;" />

等待

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程加入等待队列中
    Node node = addConditionWaiter();
    // 释放同步状态，也就是释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210808204833826.png" alt="image-20210808204833826" style="zoom: 67%;" />

通知

调用Condition的`signal`方法，

- 将处于等待队列中的首节点移动到同步队列中
- 然后唤醒该线程，在上述`await`方法中，该线程会在`acquireQueued`方法中加入到获取同步状态的竞争中

```java
public final void signal() {
    if (!isHeldExclusively())
    	throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
    	doSignal(first);
}
```

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210808205037725.png" alt="image-20210808205037725" style="zoom:67%;" />

## Semaphore

相比于ReentrantLock独占式获取，Semaphore提供共享式获取，两者最主要的区别在于同一时刻能否有多个线程同时获取到同步状态。以文件读写为例，多个程序能够同时对文件进行读操作，写操作不能多个程序同时进行。

通过重写同步器的`tryAcquireShared(int arg)`方法可以共享式地获取同步状态

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

释放

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

`acquire(1)`流程

```java
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

`release(1)`流程

```java
    public void release() {
        sync.releaseShared(1);
    }
```

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

## ReentrantReadWriteLock

### 读写状态设计

依然使用AQS中的state变量表示读写锁的同步状态，高16位表示读状态，低16位表示写状态

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210808214120813.png" alt="image-20210808214120813" style="zoom:80%;" />

### 写锁的获取和释放

写锁是一个支持重进入的排他锁。如果当前线程已经获取写锁，则增加写状态。

- 如果当前存在读锁，则获取同步状态失败
- 如果不存在读锁，但独占锁不是自己，获取同步状态失败，如果独占锁是自己，增加重入次数
- 如果不存在读锁，且不存在独占锁，尝试获取同步状态

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 存在读锁，不存在读锁，但是独占锁线程不是自己
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

释放

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    // 因为当前是独占锁，因此不需要加锁
    setState(nextc);
    return free;
}
```

### 读锁的获取和释放

锁获取代码，有化简

```java
protected final int tryAcquireShared(int unused){
    for(;;){
        int c = getState();
        int nextc = c + (1 << 16); // 读状态加1
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 如果存在读锁，且读锁不是自己，获取同步状态失败
        if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
            return -1;
        if (compareAndSetState(c, nextc))
            return 1;
    }
}
```

释放

```java
protected final boolean tryReleaseShared(int unused) {
    for (;;) {
        int c = getState();
        int nextc = c - (1 << 16);
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```