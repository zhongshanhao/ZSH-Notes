---
title: Java中的线程
date: 2020-08-24
categories:
 -  Java多线程
---

## Java内存模型

Java线程之间的通信由Java内存模型（简称JMM）控制，从抽象的角度来说，JMM定义了线程和主内存之间的抽象关系。

JMM的抽象示意图：

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png" alt="Java内存模型" style="zoom:50%;" />

- 所有的共享变量都存在主内存中
- 每个线程都保存了一份该线程使用到的共享变量的副本
- 如果线程A与线程B之间要通信的话，必须经历下面两个步骤
  - 线程A将本地内存A中更新过的共享变量刷新到主内存中
  - 线程B到主内存中去读取线程A之前已经更新过的共享变量

那么怎么知道这个共享变量的被其他线程更新了呢？这就是JMM的功劳了，也是JMM存在的必要性之一。**JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证**。

> Java中的volatile关键字可以保证多线程操作共享变量的可见性以及禁止指令重排序，synchronized关键字不仅保证可见性，同时也保证了原子性（互斥性）。在更底层，JMM通过内存屏障来实现内存的可见性以及禁止重排序。为了程序员的方便理解，提出了happens-before，它更加的简单易懂，从而避免了程序员为了理解内存可见性而去学习复杂的重排序规则以及这些规则的具体实现方法。

内存模型的三大特性（如何保证？待补充）

- 原子性
- 可见性
- 有序性

synchronized能够保证三大特性，volatile能够保证可见性和有序性

## Java线程生命周期

Java线程生命周期与操作系统中的进程生命周期定义有所不同。

| 状态名称     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NEW          | 初始状态，Thread对象被创建，但是还没有调用start()方法        |
| RUNNABLE     | 运行状态，Java中将运行和就绪态统称为运行态                   |
| BLOCKED      | 阻塞状态，线程获取不到锁资源而进入阻塞状态                   |
| WAITING      | 等待状态，进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断） |
| TIME_WAITING | 超时等待状态，不同于等待状态，可以在指定的时间自行返回       |
| TERMINATED   | 终止状态，表示当前线程已经执行完毕                           |

![Java+线程状态变迁](https://gitee.com/Krains/FigureBed/raw/master/img/Java+%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E5%8F%98%E8%BF%81.png)

> 纠错：左上角Object.join()应为Thread.join()

## 创建线程的三种方式

方法一：继承Thread，覆写run()方法

方法二：实现Runnable接口，然后交给Thread执行

例子：

```java
    @Test
    public void test1(){
        Thread t1 = new Thread("t1"){
          @Override
          public void run(){
              System.out.println(1);
          }
        };
        t1.start();
    }

    @Test
    public void test2(){
        // 使用Lambda接口简化类的创建
        Runnable task = () -> System.out.println(2);
        Thread t2 = new Thread(task, "t2");
        t2.start();
    }
```

区别

- 方法1把线程和任务合并在一起，方法2将两者分开了
- 用`Runnable`更容易与线程池等高级 API 配合
- 用`Runnable`让任务类脱离了`Thread`继承体系，更灵活

方法三：FutureTask配合Thread，FutureTask 能够接收 Callable 类型的参数，Callable也是一个函数式接口，只有一个call()方法，创建好FutureTask任务交给Thread执行，它用来处理有返回结果的情况

使用例子：

```java
        // 创建任务对象，指定返回结果类型
        FutureTask<String> task = new FutureTask<>(()->{
            Thread.sleep(1000);
            return "sss";
        });

        // 新建线程去执行任务
        new Thread(task, "thread1").start();

        // 调用者线程阻塞，直到task任务执行结束返回结果
        String result = task.get();
        System.out.println(result);
```

## Thread类

常用方法

```java
// 启动一个新线程运行run方法
start();  

// 线程运行时的代码
run();

// 等待 调用 该方法的线程结束，当前线程才继续执行
join();

// 最多等待n毫秒
join(long n);
    
// 获取当前正在执行的线程
currentThread();

// 让当前执行的线程休眠n毫秒
sleep(n);

// 提示线程调度器让出当前线程对CPU的使用
yield();

// 打断线程，调用sleep、wait、join的线程会进入等待状态，可用该方法打断阻塞状态的线程，并抛出异常和清除打断标记
// 如果线程正在运行，打断标记为真
interrupt();
```

### Thread类中interrupt()、interrupted() 和 isInterrupted() 方法区别

**interrupt()的作用**

- 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
- 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。调用`isInterrupted()`方法，如果返回的是true，说明被其他线程打断过。

**Interrupted()**

作用是测试当前线程是否被中断（检查中断标志），返回一个boolean并清除中断状态，第二次再调用时中断状态已经被清除，将返回一个false。

**isInterrupted()**

返回中断标记，不会清除中断标记位，返回true，说明被中断了，否则是false

**关于start()的两个引申问题**

1. 反复调用同一个线程的start()方法是否可行？
2. 假如一个线程执行完毕（此时处于TERMINATED状态），再次调用这个线程的start()方法是否可行？

要分析这两个问题，我们先来看看start()的源码：

```java
public synchronized void start() {
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {

        }
    }
}
```

看不到对threadStatus的修改，通过端点调试，两个问题的答案都是不可行，在调用一次start()之后，threadStatus的值会改变（threadStatus !=0），此时再次调用start()方法会抛出IllegalThreadStateException异常。

**变量的线程安全分析**

成员变量和静态变量是否线程安全？

- 如果它们没有共享，则线程安全
- 如果被共享了
  - 对变量只有读操作，则线程安全
  - 对变量有读写操作，则这段代码是临界区，需要考虑线程安全问题

局部变量是否线程安全？

- 局部变量是线程安全的，因为每个线程都创建了一份栈帧，局部变量存在局部变量表中，不是共享的

- 但局部变量引用的对象则未必，如果该对象逃离了方法的作用范围，则需要考虑线程安全问题。

## ThreadLocal

ThreadLocal可以给不同线程存储属于自己的一份数据，这个数据是线程私有的

```java
// 定义为成员变量
ThreadLocal<Integer> threadLocal1 = new ThreadLocal<Integer>();
ThreadLocal<Integer> threadLocal2 = new ThreadLocal<Integer>();

threadLocal1.set(999);
threadLocal2.set(888);
```

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210323095151105.png" alt="image-20210323095151105" style="zoom:50%;" />

每个线程持有一个ThreadLocalMap变量，该变量是ThreadLocal的静态内部类，由ThreadLocal管理

```java
public class Thread implements Runnable {
	/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    // ...
}

public class ThreadLocal<T> {
    
	// 注意： Entry继承了弱引用，Entry里的key就是用弱引用所引用的 
	static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold;
        
        
        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
    }
}
```

从上述代码可以看到，ThreadLocalMap里有个Entry数组，是真正存储数据的地方，初始容量是16，扩容阈值时当前数组长度的 2 / 3。

我们看最重要的set方法，该map使用的是哈希碰撞的解决方法是 开放地址法(再散列法)。

set方法中，以一个ThreadLocal的实例为key，通过该key的哈希值&len-1，得到下标，

- 如果对应位置为null，直接将新的Entry放入桶
- 如果不为null，判断存储的Entry的key是否与当前ThreadLocal实例相等，相等则替换
- 不相等则寻找下一个位置，重复1-3操作

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

扩容

```java
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;
			
            // 将原数组的Entry重新rehash到新的数组中
            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```

需要注意的点

ThreadLocal可能会造成内存泄漏问题

![91ef76c6a7efce1b563edc5501a900dbb58f6512](D:\_temp\网络图片\91ef76c6a7efce1b563edc5501a900dbb58f6512.jpeg)

造成内存泄漏的原因

线程运行时，如果`ThreadLocalRef`被置为了null，那么对`ThreadLocal`的强引用将会断开，如果此时进行GC，那么`Entry`里面的key将会被回收，那么`value`将不会被访问到，如果线程不结束(使用线程池)，那么value将不会被回收，从而造成了内存泄漏问题。

解决方法，使用完`ThreadLocal`之后，执行`remove`操作，会将对`value`的强引用也回收掉，避免出现内存泄漏情况。

key使用弱引用的原因

使用**弱引用**可以多一层保障：弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set(),get(),remove()的时候会被清除，但如果一直不调用相应的方法，内存泄漏的隐患还是会存在。

因此，ThreadLocal内存泄漏的根源是：**由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏，而不是因为弱引用。**

`ThreadLocalMap`也有帮助回收`value`的`expungeStaleEntry`方法，会在哈希定位获取`Entry`失败时进行扫描回收。

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

参考链接

[1].http://concurrent.redspider.group/article/02/6.html