---
title: volatile关键字
date: 2020-08-26
categories:
 -  Java多线程
---

## 可见性

### 退不出的循环

main 线程对 run 变量的修改对于 t 线程不可见,导致了 t 线程无法停止

```java
static boolean run = true;

public static void main(String[] args) throws InterruptedException {
	Thread t = new Thread(()->{
		while(run){
			// 在run循环内部加打印语句也能够退出循环，因为println源码中加了synchronized，加锁会更新工作内存
            // System.out.println();
		}
	});
	t.start();
	Thread.sleep(1000);  // 等待一秒，线程停不下来，改为1ms后循环次数可能达不到缓存run的阈值，能够结束循环
	run = false; // 线程t不会如预想的停下来
}
```

分析原因：

- 初始状态下，t线程从主内存读取run的值到工作内存中

- 正常情况下每次循环t线程都会到主存中读取run的值到自己的工作内存中，但是这样比较耗时，因此JIT编译器会在执行了多次循环后将run的值缓存到自己工作内存中的高速缓存中，减少对主存中run的访问，提高效率

- 1秒后，main线程修改了run的值，并同步到主存，而t还是从自己的工作内存中的高速缓存中读取这个变量的值，因此不能看到最新的更改

> 主存：线程共享的（堆、方法区）
> 工作内存：线程私有的（本地方法栈、虚拟机栈和程序计数器）

解决方法：

- 给run变量加`volatile`关键字
- 使用`synchronized`，加锁会更新工作内存

volatile(易变关键字)

它可以用来修饰**成员变量**和**静态成员**变量（局部变量没有处于主内存中），它可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作 volatile 变量都是直接操作主存的。

注意`volatile`不能保证对变量操作的原子性，比如两个线程同时执行i++操作，还是会有数据不一致的问题，`volatile`只能保证的是每次读取变量的时候去主存中读，而不能保证在一旦主存中变量改变，工作内存中能够马上看到更新（变量是在更新之前读入）。

## 有序性

```java
int num = 0;
boolean ready = false;

// 线程1 执行此方法
public void actor1(I_Result r) {
	if(ready) {
		r.r1 = num + num;
	} else {
		r.r1 = 1;
	}
}

// 线程2 执行此方法
public void actor2(I_Result r) {
	num = 2;
	ready = true;
}
```

在多线程环境下运行actor1方法和actor2方法r1会出现不同的结果，比如1或者是4，但是也有可能出现0（很少）的情况，因为JVM可能会调换`num=2`和`ready=true`对应指令的顺序，方便进行指令重排。

指令重排是CPU层面的优化，为了提高并发效率，读主存时可以进行对已在工作内存中的变量加一操作，通过指令重排，将很多可以并行的指令排在一起，这样提高执行效率，但是这样可能会影响程序的正确性。

用volatile修饰的变量，通过内存屏障的方式，可以禁用指令重排。

### volatile原理

有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码，即Lock代码，Lock前缀的指令在多核处理器下会引发两件事情

- 将当前处理器缓存行的数据写回到系统内存
- 这个写回内存操作会使在其他CPU里缓存了该内存地址的数据无效

即有volatile变量修饰的共享变量在写的时候会写回到主存，读的时候会到主存中读。

volatile的底层实现原理是内存屏障，Memory Barrier

- 对volatile变量的写指令后会加入写屏障
- 对volatile变量的读指令前会加入读屏障

保证可见性

写屏障保证在该屏障之前的对共享变量的改动，都同步到主存当中

```java
public void actor2(I_Result r) {
	num = 2;
	ready = true; // ready变量有volatile关键字修饰
	// 写屏障
}
```

读屏障保证在该屏障之后，对共享变量的读取，加载的是主存中最新的数据

```java
public void actor1(I_Result r) {
    // 读屏障
	if(ready) {
		r.r1 = num + num;
	} else {
		r.r1 = 1;
	}
}
```

保证有序性

写屏障会确保指令重排序时，不会将写屏障之前的代码排在写屏障之后

读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前

## volatile 解决 double-checked locking 问题

```java
    public final class Singleton {
        private Singleton() { }
        // 加上volatile解决问题
        private static Singleton INSTANCE = null;
        public static Singleton getInstance() {
            if(INSTANCE == null) { // 加了这个判断是为了提高效率，不用每次获取实例都申请锁
                synchronized(Singleton.class) {
                    if (INSTANCE == null) {  
                        // 线程1执行到这，由于指令重排可能导致先将引用（分配内存）给实例，再调用构造方法，
                        // 如果线程2在调用构造方法之前调用getInstance()，那么此时INSTANCE不为null此时
                        // 线程2拿到的是没有执行初始化的实例
                        INSTANCE = new Singleton(); // 指令重排导致出错，线程可能拿到的是并未执行构造器方法的单例
                    }
                }
            }
            return INSTANCE;
        }
    }
```

`INSTANCE = new Singleton()`这行代码从字节码角度来看是这样的

```java
17: new #3					    // class cn/itcast/n5/Singleton
20: dup
21: invokespecial #4  // Method "<init>":()V
24: putstatic 				   // Field INSTANCE:Lcn/itcast/n5/Singleton;
```

- 17：创建对象，将对象引用入栈
- 20：复制一份对象的引用
- 21：利用这个引用，调用构造方法
- 24：将引用赋值给INSTANCE

也许JVM指令重排会优化为：先执行24，再执行21，如果两个线程按如下时间序列执行

![double-checkedlocking](https://gitee.com/Krains/FigureBed/raw/master/img/double-checkedlocking.png)

这样，t1还没有执行构造方法，如果在构造方法中要执行很多初始化操作，那么t2使用的是一个未初始化完毕的单例。

因此，我们可以给变量INSTANCE加上volatile关键字来禁止指令重排，以保证引用赋值在执行构造器方法之后。

