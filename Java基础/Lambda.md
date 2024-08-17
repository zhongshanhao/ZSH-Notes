---
title:  Lambda表达式
date: 2020-08-17
categories:
 -  Java基础
---

Java中一切皆对象，因此在Java中函数或者方法无法独立存在，它们不是一个对象，要想像JavaScript进行函数式编程，Java8提出Lambda表达式。

在 Java 中，Lambda 表达式是对象，他们必须依附于一类特别的对象类型——函数式接口(functional interface)。

## 函数式接口(functional interface)

函数式接口是只包含一个抽象方法声明的接口。

java.lang.Runnable 就是一种函数式接口，在 Runnable 接口中只声明了一个方法 void run()，我们使用匿名内部类来实例化函数式接口的对象，有了 Lambda 表达式，这一方式可以得到简化。

每个 Lambda 表达式都能隐式地赋值给函数式接口，例如，我们可以通过 Lambda 表达式创建 Runnable 接口的引用。

```java
Runnable r = () -> System.out.println("hello world");
```

当不指明函数式接口时，编译器会自动解释这种转化：

```java
new Thread(
   () -> System.out.println("hello world")
).start();
```

因此，在上面的代码中，编译器会自动推断：根据线程类的构造函数签名 `public Thread(Runnable r) { }`，将该 Lambda 表达式赋给 Runnable 接口。

例子

```java
// 旧方法，匿名内部类
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello from thread");
    }
}).start();

//新方法:
new Thread(
() -> System.out.println("Hello from thread")
).start();

// 旧方法
Arrays.sort(arr, new Comparator<Integer>(){
    @Override
    public int compare(int x, int y){
        return x - y;
    }
});

// 新方法
Arrays.sort(arr, (x, y) -> x - y);
```

@FunctionalInterface 是Java8新加入的接口，函数式接口中只能有一个抽象方法，如果一个方法声明的抽象方法重写的是其父方法，那么这个抽象方法不计算在内，若定义多个抽象方法，编译器会报错。我们自定义一个函数式接口，在接口上声明@FunctionalInterface，然后使用匿名内部类和Lambda表达式实现该接口。

```java
class Main {

    @FunctionalInterface
    interface People{
        void doSomeWork();
    }

    public static void start(People p){
        p.doSomeWork();
    }

    public static void main(String[] args) {
       //invoke doSomeWork using Annonymous class
        start(new People() {
            @Override
            public void doSomeWork() {
                System.out.println("do work1");
            }
        });
        
        //invoke doSomeWork using Lambda expression 
        start(()-> System.out.println("dodo"));
    }
}
```

 Lambda表达式的载体就是People接口，start()方法可以接受Lambda表达式作为参数。

> Lambda表达式内的引用的变量必须是final或者effectively final变量，effectively final变量指的是在Lambda之后该变量的值不会被修改。

参考链接：http://blog.oneapm.com/apm-tech/226.html