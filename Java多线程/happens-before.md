---
title: happens-before
date: 2020-08-26
categories:
 -  Java多线程
---

happens-before 规定了对共享变量的写操作对其它线程的读操作可见，它是可见性与有序性的一套规则总结，抛开以下 happens-before 规则，JMM 并不能保证一个线程对共享变量的写，对于其它线程对该共享变量的读可见

线程解锁m之前对变量的写，对于接下来对 m 加锁的其它线程对该变量的读可见

```java
static int x;
static Object m = new Object();

// 线程t1先于t2执行，由于加了锁，t1对共享变量的修改对t2可见
new Thread(()->{
    synchronized(m) {
    	x = 10;
    }
},"t1").start();

new Thread(()->{
    synchronized(m) {
    	System.out.println(x);
    }
},"t2").start();
```

线程对 volatile 变量的写，对接下来其它线程对该变量的读可见

```java
volatile static int x;

// 线程t1先于t2执行，由于关键字volatile，t1对共享变量的修改对t2可见
new Thread(()->{
	x = 10;
},"t1").start();

new Thread(()->{
	System.out.println(x);
},"t2").start();
```

线程start前对变量的写，对该线程开始后对该变量的读可见

```java
static int x;

x = 10;

new Thread(()->{
	System.out.println(x);
},"t2").start();
```

线程结束前对变量的写，对其它线程得知它结束后的读可见(比如其它线程调用 t1.isAlive() 或 t1.join()等待它结束)

```java
static int x;

Thread t1 = new Thread(()->{
	x = 10;
},"t1");
t1.start();

t1.join();
System.out.println(x);
```

线程 t1 打断 t2(interrupt)前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见(通过t2.interrupted 或 t2.isInterrupted)

```java
static int x;

public static void main(String[] args) {
    Thread t2 = new Thread(()->{
        while(true) {
            if(Thread.currentThread().isInterrupted()) {
                System.out.println(x);
                break;
        	}
    	}
    },"t2");
    t2.start();
    
    new Thread(()->{
        Thread.sleep(1000);
        x = 10;
        t2.interrupt();
    },"t1").start();
    
    while(!t2.isInterrupted()) {
    	Thread.yield();
    }
    System.out.println(x);
}
```

对变量默认值(0，false，null)的写，对其它线程对该变量的读可见

具有传递性，如果 x -> y 并且 y -> z 那么有 x -> z ，配合 volatile 的防指令重排，有下面的例子

```java
volatile static int x;
static int y;

new Thread(()->{
    y = 10;
    x = 20;
},"t1").start();

new Thread(()->{
    // x=20 对 t2 可见, 同时 y=10 也对 t2 可见
    System.out.println(x);
},"t2").start();
```

