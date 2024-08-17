---
title: CAS
date: 2020-08-25
categories:
 -  Java多线程
---

## 无锁实现多线程并发安全问题

```java
class AccountSafe implements Account {
    private AtomicInteger balance;
    
    public AccountSafe(Integer balance) {
        this.balance = new AtomicInteger(balance);
    }
        
    @Override
    public Integer getBalance() {
        return balance.get();
    }
    
    @Override
    public void withdraw(Integer amount) {
        while (true) {
            int prev = balance.get();
            int next = prev - amount;
            // 对比和交换，如果prev的值与主存中的值一样，则修改，返回true
            // 否者则放弃修改同时返回false
            if (balance.compareAndSet(prev, next)) {
                break;
            }
        }
        // 可以简化为下面的方法
        // balance.addAndGet(-1 * amount);
    }
}
```

其中的关键是 compareAndSet，它的简称就是 CAS (也有 Compare And Swap 的说法)，该方法是原子的。

## 工作方式/实现原理

![cas工作方式](https://gitee.com/Krains/FigureBed/raw/master/img/cas%E5%B7%A5%E4%BD%9C%E6%96%B9%E5%BC%8F.png)

CAS的底层是`lock cmpxchg`指令，在单核和多核CPU下都能够保证比较与交换的原子性。在多核状态下，某个核执行到带 lock 的指令时，CPU 会让总线锁住，当这个核把此指令执行完毕，再开启总线。同时会关闭中断，当前线程不会响应中断，因此这个过程中不会被线程的调度机制所打断，保证了多个线程对内存操作的准确性。

### CAS操作依赖于volatile

原子类中用来存值的变量前加了`volatile`关键字

```java
private volatile int value;
```

获取共享变量时，为了保证该变量的可见性，需要使用volatile修饰。

它可以用来修饰成员变量和静态成员变量，避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作volatile变量都是直接操作主存，即一个线程对volatile变量的修改，对另一个线程可见。

### CAS与synchronized比较

- CAS是基于乐观锁的思想，最乐观的估计，不怕别的线程来改。synchronized是基于悲观锁的思想，防着别的线程来改。
- CAS与volatile实现的是无锁并发、无阻塞并发，synchronized是有锁并发、阻塞并发
- CAS较synchronized来说线程上下文切换没那么频繁，synchronized中一个线程没有获得到锁就会进入阻塞状态，会涉及到上下文的切换，CAS就不会进入阻塞状态，但如果线程数多了CAS也会有上下文切换频繁的问题，因为每个CPU一次只能执行一个线程，线程多了就会进入就绪态。

## JUC提供的原子类

### 原子整数类

- AtomicBoolean
- AtomicInteger
- AtomicLong

以AtomicInteger为例使用，以下是源码的重要方法：

```java
    // volatile保证变量的可见性，每次从主存中读value，写到主存
    private volatile int value;

	// cas操作，如果主存中的值和expect不一致，则设置失败，返回false
	// 如果一致，则用update替换主存中的expect值，返回true
 	// 该操作是原子的，在多线程环境下不会发送线程安全问题
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

	// IntUnaryOperator是一个接口，我们可以实现该接口，自定义对value加减乘除操作
	public final int updateAndGet(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return next;
    }
```

使用例子

```java
        AtomicInteger i = new AtomicInteger(0);

        // 相当于++i操作，先对i加1，在获取i的值
        System.out.println(i.incrementAndGet());

        // 相当于i++操作，先获取i的值，在对i加1
        System.out.println(i.getAndIncrement());

        // updateAndGet接受一个函数（接口实现类），这个函数定义了对value的操作，
        // 能够保证函数的操作的原子性，并且要求这个函数是无副作用的，因为它会多次执行该函数
        System.out.println(i.updateAndGet(p->2*p));
```

> 无副作用是函数式编程中的一个概念，无副作用的意思就是： 一个函数（java里是方法）的多次调用中，只要输入参数的值相同，输出结果的值也必然相同，并且在这个函数执行过程中不会改变程序的任何外部状态（比如全局变量，对象中的属性，都属于外部状态），也不依赖于程序的任何外部状态。

### 原子引用

为什么需要原子引用？

保护其他引用类型变量的原子性

- AtomicReference
- AtomicMarkableReference
- AtomicStampedReference

以AtomicReference为例

```java
interface DecimalAccount {
    // 获取余额
    BigDecimal getBalance();
    // 取款
    void withdraw(BigDecimal amount);
    /**
     * 方法内会启动 1000 个线程,每个线程做 -10 元 的操作
     * 如果初始余额为 10000 那么正确的结果应当是 0
     */
    static void demo(DecimalAccount account) {
        List<Thread> ts = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            ts.add(new Thread(() -> {
                account.withdraw(BigDecimal.TEN);
            }));
        }
        ts.forEach(Thread::start);
        ts.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(account.getBalance());
    }
}
```

使用CAS实现线程安全

```java
class DecimalAccountCAS implements DecimalAccount{
    private AtomicReference<BigDecimal> balance;

    public DecimalAccountCAS(BigDecimal balance){
        this.balance = new AtomicReference<>(balance);
    }

    @Override
    public BigDecimal getBalance() {
        return balance.get();
    }

    @Override
    public void withdraw(BigDecimal amount) {
        // CAS保证操作的原子性
        while(true){
            BigDecimal prev = balance.get();
            BigDecimal next = prev.subtract(amount);
            if(balance.compareAndSet(prev, next)){
                return ;
            }
        }
    }
}
```

使用`synchronized`，相比来说`synchronized`重量级锁开销大

```java
class DecimalAccountLock implements DecimalAccount{
    private BigDecimal balance;

    public  DecimalAccountLock(BigDecimal balance){
        this.balance = balance;
    }

    @Override
    public BigDecimal getBalance() {
        return balance;
    }

    @Override
    public void withdraw(BigDecimal amount) {
        // 注意不能锁balance，balance是会变的
        synchronized (this){
            balance = balance.subtract(amount);
        }
    }
}
```

#### ABA问题

```java
    static AtomicReference<String> ref = new AtomicReference<>("A");
    
    public static void main(String[] args) throws InterruptedException {
        // 获取值 A
        // 这个共享变量被它线程修改过?
        String prev = ref.get();
        other();
        Thread.sleep(1000);
        // 尝试改为 C
        System.out.println("change A->C "+ref.compareAndSet(prev, "C"));
    }
    private static void other() throws InterruptedException {
        new Thread(() -> {
            System.out.println("change A->B " + ref.compareAndSet(ref.get(), "B"));
        }, "t1").start();

        Thread.sleep(100);

        new Thread(() -> {
            System.out.println("change B->A " + ref.compareAndSet(ref.get(), "A"));
        }, "t2").start();
    }
```

输出如下

```java
change A->B true
change B->A true
change A->C true
```

主线程首先获得共享变量的值，如果此时其他线程将共享变量由A该为B，再由B该成A，此时主线程再对共享变量进行cas操作也是可以成功的，就是说AtomicReference无法感知共享变量是否被修改过，这存在一个安全隐患问题。

#### 解决方法

使用AtomicStampedReference类，通过增加一个版本号来判断共享变量是否被修改过

```java
    static AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
    public static void main(String[] args) throws InterruptedException {
        // 获取值 A
        String prev = ref.getReference();
        // 获取版本号
        int stamp = ref.getStamp();
        // 如果中间有其它线程干扰,发生了 ABA 现象
        other();
        Thread.sleep(1000);
        // 尝试改为 C
        System.out.println("版本 "+stamp);
        System.out.println("change A->C "+ref.compareAndSet(prev, "C", stamp, stamp + 1));
    }

    private static void other() throws InterruptedException {
        new Thread(() -> {
            System.out.println("change A->B "+ref.compareAndSet(ref.getReference(), "B",
                    ref.getStamp(), ref.getStamp() + 1));
            System.out.println("更新版本为 "+ref.getStamp());
        }, "t1").start();

        Thread.sleep(500);

        new Thread(() -> {
            System.out.println("change B->A "+ref.compareAndSet(ref.getReference(), "A",
                    ref.getStamp(), ref.getStamp() + 1));
            System.out.println("更新版本为 "+ref.getStamp());
        }, "t2").start();
    }
```

此时版本号不一致导致更新失败

```java
change A->B true
更新版本为 1
change B->A true
更新版本为 2
版本 0
change A->C false
```

