---
title:  抽象类与接口
date: 2020-08-17
categories:
 -  Java基础
---

## Java特性

1. 封装

   - 是什么？

     在Java中，一切皆对象。封装，就是把客观事物封装成抽象的类，一个类就是一个封装了数据以及操作这些数据的代码的逻辑实体。

   - 有什么好处？

     我们将复杂的功能封装到一个类中，对外开放一个接口，我们在使用的时候不用管里面复杂的逻辑，直接调用即可完成功能。

2. 继承

   - 是什么？
     继承是一个对象获取另一个对象中属性的方法。

   - 有什么好处？
     继承可以大幅减少冗余的代码，并且可基于父类的基础上扩展代码，增加功能，提高开发效率。

3. 多态

   - 是什么？
    多态就是同一个接口，使用不同的实例而执行不同操作。如用List接口引用了一个参数，调用get方法取决与其实际类型是ArrayList还是LinkedList。
     
   - 有什么好处？
   
     提高了代码的扩展性。

## 修饰符

1. ``` final```

- 作用在类上
  表示该类不可以被继承

- 作用在方法上
  表示该方法不可以被覆写
- 作用在字段上
  表示该字段不能被修改，对于基本类型，其值不能被修改，对于引用类型，表示不能在引用其他对象，但是可以修改引用对象本身。

2. ``` static```

   - 作用在成员变量上
     被称为静态字段，静态字段只有一个共享的空间，在类的所有实例中共享静态字段。

   - 作用在方法上
     被称为静态方法，静态方法能够在不创建实例的情况下通过类名直接调用，如```Math.max()```；
     静态方法不能调用非静态的方法和成员变量。

3. ```private```

   私有访问修饰符，被声明为private的方法、变量和构造方法只能被所属类（其实例或子类也不能访问）访问，被private修饰的方法和变量的作用域只在其类内部（不能通过实例向外抛出），当然在该类的内部类也可以访问，注意类和接口不能声明为private。

4. `protected`

   如果想要类中的变量或方法不被其他类访问但是可以被其子类访问，可以加protected。

5. ```public```

   与private相反，被声明为public的方法、变量、类和接口可以被任何其他类访问。

## 抽象类

如果把一个方法定义为`abstract`，表示这是一个抽象方法，本身没有实现任何方法语句，因为这个抽象方法本身是无法执行的，所以其所在的类无法被实例化，必须要在类上也声明为`abstract`。

无法实例化的抽象类有什么用？

抽象类本身被设计成只能用于被继承，因此，抽象类可以强迫子类实现其定义的抽象方法，否则编译器就会报错，因此，抽象方法实际上相当于定义了“规范”。

面向抽象编程

当我们定义了抽象类`Person`，其中有一个抽象的`run()`方法，当我们实现具体的`Student`、`Teacher`子类的时候，我们可以通过抽象类`Person`类型区引用具体子类的实例
```java
abstract class Person{
    public abstract void run();
}

Person s = new Student();
Person t = new Teacher();
```
这种引用抽象类的好处在于，我们对其进行方法调用，并不关心Person类型变量的具体子类型：
```java
// 不关心Person变量的具体子类型，在运行时能够根据它实际的子类型调用相应的方法
s.run();
t.run();
```
同样的代码，如果引用的是一个新的子类，我们仍然不关心具体类型：
```java
// 同样不关心新的子类是如何实现run()方法的：
Person e = new Employee();
e.run();
```
这种尽量引用高层类型，避免引用实际子类型的方式，称之为面向抽象编程。

面向抽象编程的本质就是：

- 上层代码只定义规范（例如：abstract class Person）；

- 不需要子类就可以实现业务逻辑（正常编译）；

- 具体的业务逻辑由不同的子类实现，调用者并不关心。

## 接口

`interface`是比抽象类还要抽象的纯抽象接口，接口定义的所有方法默认都是`public abstract`的，所以在定义接口时可以省略，接口中的变量会被隐式指定为`public static final`。

从 Java 8 开始，接口也可以拥有默认的方法实现，这是因为不支持默认方法的接口的维护成本太高了。在 Java 8 之前，如果一个接口想要添加新的方法，那么要修改所有实现了该接口的类。

抽象类和接口的区别

语法层面

- 继承：一个类只能单继承一个抽象类，但可以实现多个接口

- 字段：抽象类可以定义各种类型的成员变量，而接口的成员变量只能是`public static final`类型的（常量）

- 方法：抽象类可以定义非抽象方法，接口可以定义default和static（JDK1.8后）方法（为了防止在接口中新增方法时影响其他已经实现了该接口的实现类报错[抽象方法必须被子类重写]而打的补丁）

设计层面

- 接口侧重功能设计，能够将具体实现与调用者隔离，降低模块间的耦合
- 抽象类侧重提升类的复用性，在原有的基础上预留扩展点供开发者灵活实现

## 基本数据类型

Java中的几种基本数据类型是什么？对应的包装类型是什么？各自占用多少字节呢？
Java中有8种基本数据类型，分别为：

| 基本类型 | 位数 | 字节 | 默认值  |
| -------- | ---- | ---- | ------- |
| int      | 32   | 4    | 0       |
| short    | 16   | 2    | 0       |
| long     | 64   | 8    | 0L      |
| byte     | 8    | 1    | 0       |
| char     | 16   | 2    | 'u0000' |
| float    | 32   | 4    | 0f      |
| double   | 64   | 8    | 0d      |
| boolean  | 1    |      | false   |

对于boolean，官方文档未明确定义，它依赖于 JVM 厂商的具体实现，一般当做int变量来处理，占4个字节。逻辑上理解是占用 1位，但是实际中会考虑计算机高效存储因素。

Java中负数是如何存储的？

以-1为例，取其绝对值的原码，在取该原码的补码。

为何如此？

计算机中只有加法，所以计算减法的时候需要将减法传为加法。计算 a - b 就相当于 a + (-b)，对应的把他们的二进制相加就能够得到结果，正数用原码表示，而负数用其相反数原码的补码表示。

什么是包装类？

Java 设计当初就提供了 8 种 基本数据类型及对应的 8 种包装数据类型。Java中一切皆对象，包装类型正是为了解决基本数据类型无法面向对象编程所提供的。

什么情况下只能用包装类？

- 使用泛型的时候
- 成员变量中基本数据类型初始化为0，而包装类型为null，如果值初始化不允许是0，则可以使用包装类型
- 方法参数中可能接受到的是空值

自动装箱

自动装箱即自动将基本数据类型转换成包装类型，实际上自动装箱调用的是`Integer.valueOf()`方法。

```java
Integer i = 1;
Integer i = Integer.valueOf(1);

public static Integer valueOf(int i) {
    // 默认缓存[-128, 127]的数，上界可以虚拟机参数-XX:AutoBoxCacheMax来配置
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

自动拆箱

自动拆箱即自动将包装类型转换成包装类型，即调用的是`intValue()`方法

```java
Integer a = new Integer(1);
int i = a
int i = a.intValue();


private final int value;

public int intValue(){
    return value;
}
```

自动拆箱和自动装箱的坑

```java
Integer n1 = 123;
Integer n2 = 123;
Integer n3 = 128;
Integer n4 = 128;
System.out.println(n1 == n2); // true，自动装箱，n1和n2都是用的在方法区的常量缓存
System.out.println(n3 == n4); // false
System.out.println(new Integer(1) == new Integer(1)) // 不同的对象，比较的是引用地址
System.out.printlb(1 == new Integer(1))   // true，先拆箱，再比较
```

## 泛型

参数类型，把类型当作参数一样转递

```java
// 自定义泛型类
public class ObjectTool<T> {
    
    private T obj;

    public void setObj(T obj) {
        this.obj = obj;
    }

    public T getObj() {
        return obj;
    }
    
    // 自定义泛型方法
    public <O> void show(O t){
        System.out.println(t);
    }
}
```

常用的K、V、E、T、？

本质上这些都是通配符，没有啥区别，只不过是编码时的一种约定俗成的东西。通常情况下，各个通配符的含义如下：

- ？表示不确定的Java类型
- T（type）表示具体的一个Java类型
- K V（key value）分别代表Java键值中的Key Value类型
- E（element）代表集合中的元素类型

PECS原则（Producer Extends. Consumer Super）

Producer Extends 上界通配符

```java
public class Test {
    static class Food {}
    static class Fruit extends Food {}
    static class Apple extends Fruit {}

    public static void main(String[] args) throws IOException {
        List<? extends Fruit> fruits = new ArrayList<>();
        //不能加入任何元素
        fruits.add(new Food());     // compile error
        fruits.add(new Fruit());    // compile error
        fruits.add(new Apple());    // compile error

        //集合元素的类型，符合extends Fruit，可赋值给 变量fruits
        fruits = new ArrayList<Food>(); // compile error
        fruits = new ArrayList<Fruit>(); // compile success
        fruits = new ArrayList<Apple>(); // compile success  注1
        fruits.add(new Apple());   // compile error 注2 


        fruits = new ArrayList<? extends Fruit>(); // 在java中会出现 compile error: 通配符类型无法实例化  

        Fruit object = fruits.get(0);    // compile success
    }
}
```

- **写入时**：编译器会阻止将Apple类加入fruits，在向fruits中添加元素时，编译器会检查类型是否符合要求，因为编译器只是知道fruits是Fruit某个子类，但是不知道具体是哪个子类，所以只好阻止所有的子类添加。
- **读取时**：可以用父类Fruit接受fruits里的元素，里边元素都是Fruit的子类，Java可以通过父类引用子类

Consumer Super 下界通配符

```java
public class Test {
    static class Food {}
    static class Fruit extends Food {}
    static class Apple extends Fruit {}

    public static void main(String[] args) throws IOException {
        List<? super Fruit> fruits = new ArrayList<>();
        //Fruit 及其子类，可被看做是Fruit，从而添加成功
        fruits.add(new Food());     // compile error
        fruits.add(new Fruit());    // compile success
        fruits.add(new Apple());    // compile success
        
		//集合元素的类型，符合super Fruit，可赋值给变量fruits，赋值后fruits不再是逆变类型
        fruits = new ArrayList<Food>(); // compile success
        fruits = new ArrayList<Fruit>(); // compile success
        fruits = new ArrayList<Apple>(); // compile error

        fruits = new ArrayList<? super Fruit>(); // compile error: 通配符类型无法实例化      

        Fruit object = fruits.get(0); // compile error，
    }
}
```

- **写入时**：出于对类型安全的考虑，我们可以加入Fruit对象或者其任何子类对象，但由于编译器并不知道List的内容究竟是Fruit的哪个超类，因此不允许加入特定的任何超类型（Food）。
- **读取时**：编译器在不知道是什么类型的情况下只能返回Object对象，因为Object是任何Java类的最终祖先类。

这两种通配符主要在于一个点，就是Java中父类可以引用子类型，而反过来却不行。

**PECS原则总结**

从上述两方面的分析，总结PECS原则如下：

如果要从集合中读取类型T的数据，并且**不能写入**，可以使用 ? extends 通配符；(Producer Extends)

如果要从集合中写入类型T的数据，并且**不需要读取**，可以使用 ? super 通配符；(Consumer Super)

如果既要存又要取，那么就不要使用任何通配符。

[参考链接](https://blog.csdn.net/xx326664162/article/details/52175283)

泛型擦除

泛型是提供给javac编译器使用的，在编译时，能够检查输入的类型，如果不匹配就会编译错误。在编译器编译完带有泛型的java程序后，生成的class文件中将不再带有泛型信息，这个过程称之为"擦除"。

## 反射

反射可以在运行时检查类、接口、方法和变量等信息，还可以在运行时实例化新对象，调用方法以及设置和获取变量值。

### Class对象

分析一个类之前，需要获得该类对应的 java.lang.Class 对象，获取该对象的方法有两种

```java
Class<Thread> threadClass = Thread.class;
Class<?> thread = Class.forName("java.lang.Thread");
```

反射的基本使用

```java
Class<Thread> threadClass = Thread.class;
System.out.println("全限定类名：" + threadClass.getName());
System.out.println("类修饰符：" + Modifier.toString(threadClass.getModifiers()));
System.out.println("包信息：" + threadClass.getPackage());

// 获取该类实现的接口信息，接口也是一个类
Class<?>[] interfaces = threadClass.getInterfaces();
// 获取所有public修饰的构造方法以及其参数
Constructor<?>[] constructors = threadClass.getConstructors();
for(Constructor constructor : constructors){
    Parameter[] parameters = constructor.getParameters();
    System.out.print("构造方法：" + constructor.getName() + " 参数：");
    for(Parameter p : parameters){
        System.out.print(p + ",");
    }
    System.out.println();
}
// 获得特定的构造方法，然后使用反射创建对象
Constructor<Thread> constructor = threadClass.getConstructor(Runnable.class);
Thread thread = constructor.newInstance(new Runnable() {
    @Override
    public void run() {
        System.out.println("使用反射调用方法");
    }
});
// 获取start方法，然后传入实例，调用方法
Method method = threadClass.getMethod("start");
method.invoke(thread);
System.out.println();
```

输出

```cmd
全限定类名：java.lang.Thread
类修饰符：public
包信息：package java.lang, Java Platform API Specification, version 1.8
构造方法：java.lang.Thread 参数：java.lang.Runnable arg0,
构造方法：java.lang.Thread 参数：
构造方法：java.lang.Thread 参数：java.lang.ThreadGroup arg0,java.lang.Runnable arg1,java.lang.String arg2,
构造方法：java.lang.Thread 参数：java.lang.Runnable arg0,java.lang.String arg1,
构造方法：java.lang.Thread 参数：java.lang.ThreadGroup arg0,java.lang.String arg1,
构造方法：java.lang.Thread 参数：java.lang.String arg0,
构造方法：java.lang.Thread 参数：java.lang.ThreadGroup arg0,java.lang.Runnable arg1,
构造方法：java.lang.Thread 参数：java.lang.ThreadGroup arg0,java.lang.Runnable arg1,java.lang.String arg2,long arg3,

使用反射调用方法
```

