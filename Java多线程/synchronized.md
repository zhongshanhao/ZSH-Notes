---
title: synchronized关键字
date: 2020-08-25
categories:
 -  Java多线程
---

## synchronized

`synchronized`实际是用对象锁保证了临界区内代码的原子性（临界区就是多个线程对共享资源读写操作的代码块），临界区内的代码对外是不可分割的，不会因线程切换所打断。

```java
// 将对象obj当做锁，若一个对象获取该锁，其他对象在获取该锁时将进入阻塞状态，见线程六状态模型
synchronized(obj){
     // 临界区
    
}
```

`synchronized`只能对对象加锁，`synchronized`加在

- 普通方法上，是对当前对象`this`加锁，相当于在方法内部加了`synchronized(this){}`
- 静态方法上，锁住的是`this.class`对象
- 同步代码块上，锁住的是括号内的对象

### synchronized底层原理

每个对象都可以关联一个Monitor对象，如果代码synchronized(obj)执行了，obj对象头中的Mark Word中将存有指向Monitor对象的地址。

![Monitor](https://gitee.com/Krains/FigureBed/raw/master/img/Monitor.png)

- 刚开始Monitor中Owner为null
- 当Thread-2执行synchronized(obj)就会将Monitor的所有者Owner置为Thread-2，Monitor中只能有一个Owner
- 在Thread-2成为obj的Monitor的Owner时，如果Thread-1，Thread-3也来执行synchronized(obj)，就会进入EntryList，这些线程将变成BLOCKED状态
- Thread-2执行完同步代码块的内容，然后唤醒EntryList中等待的线程来竞争锁
- Wait Set中存储的线程是之前获得过锁，但条件不满足而调用Object.wait()而进入WAITING状态的线程，可以通过**获得obj锁**的线程调用notify()方法来唤醒处于WAITING状态的线程，此时被唤醒的线程加入EntryList等待。

### 从字节码看synchronized

```java
public class ThreadTest {
    static final Object lock = new Object();
    static int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        synchronized (lock){
            // counter++这一行代码并不具有原子性，
            // 在转化为字节码文件的时候这一行代码会产生4条指令，如下，
            // 在多线程指令交错的情况下会导致线程安全问题
            counter++;
        }
    }
}
```

```java
public static void main(java.lang.String[]) throws java.lang.InterruptedException;
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // Field lock:Ljava/lang/Object; 获取lock引用
         3: dup
         4: astore_1							// 将lock存入局部变量表
         5: monitorenter			 	 // 将lock对象头中MarkWord置为Monitor指针
         6: getstatic     #3                  // Field counter:I 获取counter的值
         9: iconst_1							// 准备常数1
        10: iadd									// +1
        11: putstatic     #3                  // Field counter:I，将结果放到counter中
        14: aload_1								// 拿到lock引用
        15: monitorexit						// 将lock对象MarkWord重置，唤醒EntryList中的阻塞线程
        16: goto          24					
        19: astore_2							// 此处开始若synchronized代码块中发生了异常，跳到这将lock对象头重置
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             6    16    19   any
            19    22    19   any
      LineNumberTable:
        line 8: 0
        line 9: 6
        line 10: 14
        line 11: 24
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      25     0  args   [Ljava/lang/String;
```

## synchronized优化

### 轻量级锁

轻量级锁的使用场景：如果一个对象虽然有多线程访问，但多线程访问的时间错开的（也就是没有竞争），那么可以使用轻量级锁来优化。

轻量级锁对使用者是透明的，即语法仍然是synchronized

假设有两个同步方法块，利用一个对象加锁

```java
static final Object obj = new Object();
public static void method1(){
    synchronized(obj){
        // 同步块A
        method2();
    }
}

public static void method2(){
    synchronized(obj){
        // 同步块B
    }
}
```

JVM会先在当前线程的栈帧中创建用于存储锁记录的空间，这个空间用来存放锁记录。

加轻量级锁时会创建锁记录（Lock Record）对象，内部可以存储锁定对象的Mark Word和对象引用，然后将其放入栈帧中。

![Lock Record](https://gitee.com/Krains/FigureBed/raw/master/img/Lock%20Record.png)

执行到synchronized(obj)时，让锁记录中Object Reference指向锁对象，并尝试用CAS替换Object的Mark Word，将Mark Word的值存入锁记录

![轻量级锁2](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%812.png)

如果CAS替换成功，对象头中存储了锁记录地址和状态00，表示由该线程给对象加锁，这时图示如下

![轻量级锁3](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%813.png)

如果CAS失败，有两种情况

- 如果是其他线程已经持有了该Object的轻量级锁，这是表明有竞争，进入锁膨胀过程
- 如果是自己执行了synchronize锁重入，那么再添加一条Lock Record作为重入的计数

![轻量级锁4](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%814.png)

当退出synchronized代码块（解锁时）锁记录的值为null，表示有重入，这是重置锁记录，表示重入计数减一

当退出synchronized代码块（解锁时）锁记录的值不为null，这时使用CAS将Mark Word的值恢复给对象头

- 成功，则解锁成功
- 失败，说明轻量级锁进入了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程

#### 自旋优化

轻量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。

自旋重试成功的情况

| 线程1（cpu1上）             | 对象Mark       | 线程2（cpu2上）         |
| --------------------------- | -------------- | ----------------------- |
| -                           | 10（轻量级锁） | -                       |
| 访问同步代码块，获取Monitor | 10（轻量级锁） | -                       |
| 成功（加锁）                | 10（轻量级锁） | -                       |
| 执行同步代码块              | 10（轻量级锁） | -                       |
| 执行同步代码块              | 10（轻量级锁） | 访问同步块，获取Monitor |
| 执行完毕                    | 10（轻量级锁） | 自旋重试                |
| 成功（解锁）                | 01（无锁）     | 自旋重试                |
| -                           | 10（轻量级锁） | 成功（加锁）            |

如果线程2自旋加锁失败，则轻量级锁会膨胀成重量级锁，主要的流程是线程2为锁申请重量级锁，并把自己放入到EntryList里阻塞。

也有可能线程2自旋操作时等不到线程1锁释放，这时线程2进入阻塞状态。

- Java6自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋。
- 自旋会占用CPU时间，单核CPU自旋就是浪费，多核CPU自旋才能发挥优势。
- Java7之后不能控制是否开启自旋功能

#### 锁膨胀

如果在尝试加轻量级锁的过程中，CAS操作无法成功，这时一种情况就是有其他线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

```java
static Object obj = new Object();
public static void method1(){
    synchronized(obj){
        // 同步代码块
    }
}
```

当Thread-1进行轻量级加锁时，Thread-0已经对该对象加了轻量级锁

![轻量级锁5](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%815.png)

这时Thread-1加轻量级锁失败，进入锁膨胀流程

- 即为Object对象申请Monitor锁，让Object指向重量级锁地址
- 然后自己进入Monitor的EntryList，进入BLOCKED状态

![轻量级锁6](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%816.png)

当Thread-0退出同步块解锁时，使用CAS将Mark Word的值恢复给对象头，失败。这时会进入重量级解锁流程，即按照Object中对象头中MarkWord的Monitor地址对象，找到关联的Monitor对象，设置Owner为null，唤醒EntryList中BLOCKED线程。

### 偏向锁

轻量级锁在没有竞争时（就自己这个线程对同一对象加锁），每次重入仍然需要执行CAS操作，然后再加一条锁记录。

Java6中引入了偏向锁来做进一步优化：只有第一次使用CAS将线程ID设置到对象的Mark Word头，之后发现这个线程ID是自己的就表示没有竞争，不用重新CAS，以后只要不发生竞争，这个对象就归该线程所有。

```java
static final Object obj = new Object();
public static void m1(){
    // 同步块 A
    m2();
}

public static void m2(){
    // 同步块 B
    m3();
}

public static void m3(){
    // 同步块 C
}
```

轻量级锁加锁过程

![轻量级锁7](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%817.png)

偏向锁

![偏向锁1](https://gitee.com/Krains/FigureBed/raw/master/img/%E5%81%8F%E5%90%91%E9%94%811.png)

一个对象创建时

- 如果开启了偏向锁（默认开启），那么对象创建后，Mark Word值为0x05即最后3位为101，这是它的thread、epoch、age都为0
- 偏向锁是默认延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加VM参数-xx:BiasedLockingStartupDelay=0来禁用延迟
- 如果没有开启偏向锁，那么对象创建后，Mark Word值为0x01即最后3位为001，这是它的hashCode、age都为0，第一次调用hashCode方法时才会赋值
- 一个线程对一个对象加了偏向锁，那么解锁后该对象头的线程ID仍存储于对象头中
- 可以使用-XX:-UseBiasedLocking禁用偏向锁

**撤销偏向锁**

调用对象hashCode时

调用了对象的hashCode方法，如果此时偏向锁对象MarkWord中存储的是线程id，如果调用hashCode会导致偏向锁被撤销，因为存不下了，如果是轻量级锁和重量级锁则不会，因为

- 轻量级锁会在线程的锁记录（Lock Record）中记录hashCode
- 重量级锁会在Monitor中记录hashCode

其他线程使用对象

当一个线程对一个新对象使用锁时，假设该对象支持偏向锁即MarkWord最后3位为101，会在其对象头上记录线程id，当线程释放该锁后，线程id仍然没有改变，当有其他线程使用偏向锁对象时，此时发现偏向锁偏向的是其他线程，这时会将偏向锁升级为轻量级锁。

调用wait/notify

因为wait与notify是只有重量锁有，因此会将偏向锁升级为重量级锁。

批量重偏向

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程t1的对象仍有机会重写偏向t2，重偏向会重置对象的Thread ID。

当撤销偏向锁阈值超过20次后，JVM会这样觉得，我是不是偏向错了呢？于是会在给这些对象加锁时重新偏向至加锁线程。

当一个对象一开始偏向了线程t1，此时线程t2再对对象加锁（synchronized），会撤销偏向锁，改加轻量锁，解锁后对象头中的MarkWord的状态变为无锁状态，即最后3位为001，但当线程t2又重复对该**类对象**撤销了19次之后，该类对象会重新偏向t2。

批量撤销

当撤销偏向锁阈值超过40次后，JVM会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的。

锁消除

```java
static int x = 0;

public void a(){
    x++;
}

public void b(){
    Object o = new Object();
    synchronized(o){
        x++;
    }
}
```

通过工具的测试，两者运行时间几乎差不多，方法b对对象o加了锁，为什么两者的运行时间几乎一样呢？

JIT即时编译器会对热点代码进一步优化，通过对热点代码进行逃逸分析，局部变量o不会逃离方法的作用范围，其他方法也就无法对o进行加锁，所以JVM做了优化取消了加锁的过程。 

### 各种锁的比较

| 锁       | 优点                                                         | 缺点                                             | 适用场景                             |
| -------- | ------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------ |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距。 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗。 | 适用于只有一个线程访问同步块场景。   |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度。                   | 如果始终得不到锁竞争的线程使用自旋会消耗CPU。    | 追求响应时间。同步块执行速度非常快。 |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU。                            | 线程阻塞，响应时间缓慢。                         | 追求吞吐量。同步块执行时间较长。     |

### 总结锁升级流程

每一个线程在准备获取共享资源时：

第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁” 。

第二步，如果MarkWord不是自己的ThreadId，锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程解锁时将Markword的内容置为空。

第三步，两个线程都把锁对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作， 把锁对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord。

第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋 。

第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于 轻量级锁的状态 。

第六步，如果自旋失败，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己。

### Lock与synchronized的区别

- Lock能够中断正在阻塞队列中的等待的线程，让其不再尝试获取锁
- 能够在指定的截止时间内获取锁，如果截止时间到了仍然无法获取锁，则返回

## wait/notify

当一个线程获得对象obj的锁时，可以调用以下方法

```java
// 让进入obj的Monitor的线程到waitSet等待，无限等待，直到被唤醒
obj.wait()
// 有时限的等待，到n毫秒后结束等待，或是被notify
obj.wait(long n)
// 让Monitor上正在WaitSet等待的线程中挑一个唤醒（随机）
obj.notify()
// 让Monitor上正在WaitSet等待的线程全部唤醒
obj.notifyAll()
```

![wait notify](https://gitee.com/Krains/FigureBed/raw/master/img/wait%20notify.png)

- 若当前拥有Monitor对象的线程发现条件不满足，调用wait方法，即可进入WaitSet变为WAITING状态
- BLOCKED和WAITING的线程都处于阻塞状态，不占用CPU时间片
- BLOCKED线程会在Owner线程释放锁时唤醒
- WAITING线程会在Owner线程调用notify或notifyAll时唤醒，但唤醒后并不意味着立即获得锁，仍需进入EntryList重新竞争
- 要想用wait/notify方法，必须先要获得该对象的Monitor，然后才能调用wait进入WAITING状态或者调用notify唤醒一个处于WaitSet中的线程，让它进入EntryList

`Thread.sleep(long n)`和`Object.wait(long n)`的区别与相同点

- sleep是Thread的方法，而wait是Object的方法
- sleep不需要强制和synchronized配合使用，但wait需要
- sleep在睡眠的同时不会释放对象锁，而wait会释放
- 它们进入的状态都是TIMED_WAITING，如果是不带参的`wait()`方法则会进入WAITING

wait/notify的配合使用方法

```java
public class ThreadTest {
    static final Object obj = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (obj){
                try {
                    // 等到有烟送来才能干活，没烟就一直等待下去
                    while(!hasCigarette)
                        obj.wait();
                    System.out.println("有烟了，可以干活了");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "t1").start();

        new Thread(()->{
            synchronized (obj){
                try {
                    // 等到有外卖送来才能干活
                    while(!hasTakeout)
                        obj.wait();
                    System.out.println("有外卖了，可以干活了");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "t2").start();

        Thread.sleep(2000);

        synchronized (obj){
            hasCigarette = true;
            System.out.println("送烟来了");
            // 无法指定唤醒某线程，因此需要唤醒obj中waitSet中的所有线程，
            // 可以使用ReentrantLock条件变量来指定唤醒某线程
            obj.notifyAll();
        }
    }
}
```

## LockSupport.park()/unpark()

类似于操作系统中使用信号量（信号量只有0和1两个值）实现进程的同步操作（初始信号量S为0），基本使用方法如下

```java
// 暂停当前线程，相当于P操作，将S--
LockSupport.park();

// 恢复某个线程的运行，相当于V操作，将S++，此时S <= 0 会唤醒一个阻塞的线程执行
LockSupport.unpark(暂停线程对象);
```

```java
Thread t1 = new Thread(() -> {
    log.debug("start...");
    sleep(1);
    log.debug("park...");
    // 线程进入WAITING状态
    LockSupport.park();
    log.debug("resume...");
},"t1");
t1.start();

// unpark先于park调用，线程park之后不会进入WAITING状态
// LockSupport.unpark(t1);
sleep(2);
log.debug("unpark...");
// 唤醒线程
LockSupport.unpark(t1);
```

与Object的wait/notify类似，都能够让线程进入WAITING状态，以及能够唤醒线程，但两者有不同：

- wait、notify和notifyAll必须配合Object Monitor一起使用，而park、unpark不必
- park和unpark是以线程为单位来阻塞和唤醒线程的，而notify只能随机唤醒一个等待线程，notifyAll是唤醒所有处于Monitor的WaitSet中的等待线程，没有那么精确
- park和unpark可以先unpark，此时再park线程不会进入WATING状态，但是notify先与wait执行将不能唤醒线程

原理

每个线程都有自己的一个Parker对象，由三部分组成

- counter，只有1和0两个取值
- condition，条件变量
- mutex，用作对条件变量互斥访问用

park()方法

![LockSupport1](https://gitee.com/Krains/FigureBed/raw/master/img/LockSupport1.png)

- 当前线程调用park()方法
- 检查counter的值，如果是1，则将counter清零继续运行，如果是0，获得互斥锁
- 线程进入condition条件变量阻塞
- 又将counter置0

unpark()方法

![LockSupport2](https://gitee.com/Krains/FigureBed/raw/master/img/LockSupport2.png)

- 线程调用Unsafe.unpark(Thread_0)方法，设置counter为1
- 唤醒condition条件变量中的Thread_0
- Thread_0恢复运行
- 设置counter为0

## join原理

是调用者轮询检查线程alive状态

```java
t1.join();
```

等价于下面的代码

```java
synchronized(t1){
    // 调用者线程进入t1的waitSet等待，直到t1运行结束
    while(t1.isAlive()){
        t1.wait(0);
    }
}
```
