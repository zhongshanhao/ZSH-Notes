---
title: ReentrantLock可重入锁
date: 2020-08-27
categories:
 -  Java多线程
---

## ReentrantLock

与 synchronized 一样，都支持可重入，但相对于 synchronized 它还具备如下特点

- 可中断
- 可以设置超时时间
- 可以设置为公平锁
- 支持多个条件变量

```java
// 获取锁，需成对出现，释放锁放在finally
reentrantLock.lock();
try {
	// 临界区
} finally {
    // 释放锁
    reentrantLock.unlock();
}
```

### 可重入

可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁。

如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住。

### 可打断

加锁时线程t1调用`reentrantLock.lockInterruptibly()`方法表示自己申请的是可打断锁，如果其他线程拥有了这把锁，为了防止线程1无限等待下去，可以在其他线程中调用`t1.interrupt()`打断t1线程的等待状态，让线程t1抛出`InterruptedException`异常，退出等待状态。

### 锁超时

立即失败

某线程调用`lock.tryLock()`尝试获取锁，如果没有获取成功，则放弃获取，如果获取了那么就往下执行。

超时失败

某线程调用`lock.tryLock(1, TimeUnit.SECONDS)`，在1s的时间内如果能够获取到锁就往下执行，如果没有就放弃获取。

### 公平锁

ReentrantLock 默认是不公平的，意思就是当一个线程释放锁之后，处于阻塞状态的线程并不是获取锁的先后顺序来获得锁的。

创建对象时可以使用带参构造器`new ReentrantLock(false)`实现公平锁，公平锁一般没有必要，会降低并发度。

### 条件变量

synchronized 中也有一个条件变量，Monitor中的waitSet，当条件不满足时进入 waitSet 等待。

ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的

- synchronized中调用wait()方法的线程都在一个waitSet等消息
- ReentrantLock 支持多个条件变量，调用await()则在调用该方法的条件变量处等待，唤醒也是根据不同的条件变量来唤醒对应的线程

使用要点：

- await 前需要获得锁
- await 执行后，会释放锁，进入 conditionObject 等待
- await 的线程被唤醒(或打断、或超时)取重新竞争 lock 锁
- 竞争 lock 锁成功后，从 await 后继续执行

例子

```java
    static ReentrantLock lock = new ReentrantLock();
    static Condition waitCigaretteQueue = lock.newCondition();
    static Condition waitbreakfastQueue = lock.newCondition();
    static volatile boolean hasCigrette = false;
    static volatile boolean hasBreakfast = false;
    
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                lock.lock();
                while (!hasCigrette) {
                    try {
                    // 没有烟，等待
                    waitCigaretteQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                }
                log.debug("等到了它的烟");
            } finally {
                lock.unlock();
            }
        }).start();
        new Thread(() -> {
            try {
                lock.lock();
                while (!hasBreakfast) {
                    try {
                        waitbreakfastQueue.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("等到了它的早餐");
            } finally {
                lock.unlock();
            }
        }).start();
        sleep(1);
        sendBreakfast();
        sleep(1);
        sendCigarette();
    }
    
    private static void sendCigarette() {
        lock.lock();
        try {
            log.debug("送烟来了");
            hasCigrette = true;
            // 唤醒
            waitCigaretteQueue.signal();
        } finally {
            lock.unlock();
        }
    }
    
    private static void sendBreakfast() {
        lock.lock();
        try {
            log.debug("送早餐来了");
            hasBreakfast = true;
            waitbreakfastQueue.signal();
        } finally {
            lock.unlock();}
    }
```

## ReentrantLock实现原理

![reentrantlock](https://gitee.com/Krains/FigureBed/raw/master/img/reentrantlock.png)

### 非公平锁实现原理

#### 加锁解锁流程

默认为非公平锁实现

```java
public ReentrantLock(){
    sync = new NonfairSync();
}
```

NonfairSync继承自Sync，而Sync类继承自AQS

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

- state状态，state=0时表示该锁没有被线程占用，state=1时表示该锁已被占用，state>1表示该锁被重入的次数
- head指针，维护了一个双向链表，每个结点是竞争锁失败时进入阻塞状态的线程
- exclusiveOwnerThread指向的是当前拥有该锁的线程

没有竞争时

![非公平锁1](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%811.png)

第一个竞争者出现时

![非公平锁2](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%812.png)

Thread-1执行了

- CAS尝试将state由0改为1，结果失败（1）
- 进入tryAcquire逻辑，这时state已经是1，结果仍然失败（2）
- 接下来进入addWaiter逻辑，构造Node队列
  - 图中黄色三角表示该Node的waitStatus状态，其中0为默认正常状态
  - Node的创建是懒惰的
  - 其中第一个Node称为哨兵，用来占位，并不关联线程

![非公平锁3](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%813.png)

当前线程会进入acquireQueue逻辑

- acquireQueue会在一个死循环中不断尝试获得锁，失败后进入park阻塞
- 如果自己是紧邻着head（排第二位），那么再次tryAcquire尝试获取锁，当然这是state仍为1，失败（3）
- 进入shouldParkAfterFailedAcquire逻辑，将前驱node，即head的waitStatus改为-1，这次返回false

![非公平锁4](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%814.png)

- shouldParkAfterFailedAcquire 执行完毕回到 acquireQueued ，再次 tryAcquire 尝试获取锁，当然这时
  state 仍为 1，失败（4）
- 当再次进入 shouldParkAfterFailedAcquire 时，这时因为其前驱 node 的 waitStatus 已经是 -1，这次返回
  true
- 进入 parkAndCheckInterrupt，Thread-1 park(灰色表示)

![非公平锁5](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%815.png)

再次有多个线程经历上述过程竞争失败，变成这个样子

![非公平锁6](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%816.png)

Thread-0释放锁，进入tryRelease流程，如果成功

- 设置exclusiveOwnerThread为null
- state = 0

![非公平锁7](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%817.png)

- 当前队列不为 null，并且 head 的 waitStatus = -1，进入 unparkSuccessor 流程

- 找到队列中离 head 最近的一个 Node(没取消的)，unpark 恢复其运行，本例中即为 Thread-1

- 回到 Thread-1 的 acquireQueued 流程

![非公平锁8](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%818.png)

如果加锁成功(没有竞争)，会设置

- exclusiveOwnerThread 为 Thread-1，state = 1

- head 指向刚刚 Thread-1 所在的 Node，该 Node 清空 Thread

- 原本的 head 因为从链表断开，而可被垃圾回收

 如果这时候有其它线程来竞争(非公平的体现)，例如这时有 Thread-4 来了

![非公平锁9](https://gitee.com/Krains/FigureBed/raw/master/img/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%819.png)

如果不巧又被 Thread-4 占了先

- Thread-4 被设置为 exclusiveOwnerThread，state = 1
- Thread-1 再次进入 acquireQueued 流程，获取锁失败，重新进入 park 阻塞

大致流程

加锁

- 如果锁没有被占用时，此时来了一个线程，使用CAS操作尝试将state从0改为1，exclusiveOwnerThread设置为该线程，这时候该线程就拥有了改锁
- 这时候又来了一个线程，首先也是使用CAS操作尝试将state从0改为1，显然失败了，此时调用tryAcquire再次使用CAS尝试获得锁，不成功就将该线程加入链表中（两次CAS尝试）
- 如果这个线程所在结点的前驱结点是head的话，又将进行两次CAS操作尝试获取锁，不成功就将其阻塞（使用park()），等待拥有锁线程释放锁，会将它唤醒，尝试获取锁，失败则阻塞，如果前驱结点不是head的话，那么将不进行CAS，而是进入阻塞。（如果线程所在结点的前驱是head，还进行两次CAS）

解锁

- 如果拥有锁的线程要解锁，设置exclusiveOwnerThread为null，设置state为0
- 找到链表中离head最近的一个结点（没取消的），调用unpark()方法恢复其运行
- 如果此时没有竞争者，那么由该线程获得锁，如果有此时外部有线程也想获得锁，那么两个线程竞争（非公平），那么竞争失败的线程加入到链表中，然后阻塞

#### 可重入原理

- 如果拥有该锁的线程又一次尝试获取该锁，那么state将会加1，state的数值表示该锁被重入的次数
- 释放的时候会将state减1，只有当state减为0的时候才释放锁

#### 可打断原理

如果此时其他线程打断正在阻塞的线程，那么该线程会抛出异常，从而退出死循环等待获得锁。

```java
static final class NonfairSync extends Sync {
    public final void acquireInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
		// 如果没有获得到锁, 进入 (一)
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
    
    // (一) 可打断的获取锁流程
    private void doAcquireInterruptibly(int arg) throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            // 处于阻塞队列中的线程都运行到这个死循环，如果前驱结点是head，那么该线程
            // 在锁被释放的时候有能力去竞争锁，这里是可打断的，如果某个线程被打断，就能够
            // 抛出异常，从而退出死循环
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt()) {
                    // 在 park 过程中如果被 interrupt 会进入此
                    // 这时候抛出异常, 而不会再次进入 for (;;)
                    throw new InterruptedException();
                }
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
}
```

## 条件变量实现原理

每个条件变量其实就对应着一个等待队列，类似与Monitor中的waitSet，但ReentrantLock支持多个条件变量，其实现类是ConditionObject。

### await流程

开始 Thread-0 持有锁，调用 await，进入 ConditionObject 的 addConditionWaiter 流程

创建新的Node状态为-2（Node.CONDITION），关联Thread-0，加入等待队列尾部

![条件变量1](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F1.png)

接下来进入 AQS 的 fullyRelease（将state置0） 流程，释放同步器上的锁

![条件变量2](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F2.png)

unpark AQS 队列中的下一个节点，竞争锁，假设没有其他竞争线程，那么 Thread-1 竞争成功

![条件变量3](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F3.png)

park 阻塞 Thread-0

![条件变量4](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F4.png)

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
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

### signal流程

假设 Thread-1 要来唤醒 Thread-0

![条件变量5](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F5.png)

进入 ConditionObject 的 doSignal 流程，取得等待队列中第一个 Node，即 Thread-0 所在 Node

![条件变量6](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F6.png)

执行 transferForSignal 流程，将该 Node 加入 AQS 队列尾部，将 Thread-0 的 waitStatus 改为 0，Thread-3 的
waitStatus 改为 -1

![条件变量7](https://gitee.com/Krains/FigureBed/raw/master/img/%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F7.png)

### 与管程的条件变量相比

每个对象都可以用继承自Object的wait/notify方法来实现等待/通知机制。而Condition接口也提供了类似Object监视器的方法，通过与Lock配合来实现等待/通知模式。两者的区别如下：

| 对比项                                         | Object监视器                  | Condition                                                   |
| ---------------------------------------------- | ----------------------------- | ----------------------------------------------------------- |
| 前置条件                                       | 获取对象的锁                  | 调用Lock.lock获取锁，调用Lock.newCondition获取Condition对象 |
| 调用方式                                       | 直接调用，比如object.notify() | 直接调用，比如condition.await()                             |
| 等待队列的个数                                 | 一个                          | 多个                                                        |
| 当前线程释放锁进入等待状态                     | 支持                          | 支持                                                        |
| 当前线程释放锁进入等待状态，在等待状态中不中断 | 不支持                        | 支持                                                        |
| 当前线程释放锁并进入超时等待状态               | 支持                          | 支持                                                        |
| 当前线程释放锁并进入等待状态直到将来的某个时间 | 不支持                        | 支持                                                        |
| 唤醒等待队列中的一个线程                       | 支持                          | 支持                                                        |
| 唤醒等待队列中的全部线程                       | 支持                          | 支持                                                        |

