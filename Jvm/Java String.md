---
title: Java String
date: 2020-08-05
categories:
 -  JVM
---

## String的不可变性

String源码中是这样定义的，String底层在jdk8及以前是用char数组存储的，而jdk9之后改用byte数组存储，由于都加了`final`关键字，String是不可变的。

```java
// jdk8时
private final char[] value;
// jdk9时
private final byte[] value;
```

为什么jdk9改变了String底层的存储？

官方是这么说的：String类的当前实现将字符存储在char数组中，每个字符使用两个字节(16位)。从许多不同的应用程序收集的数据表明，字符串占了堆内存的主要部分，但是大部分字符串对象只包含拉丁字符，这些字符只需要一个字节的存储空间，因此这些字符串的内部char数组中将有一半的空间浪费掉了。

官方建议改变字符串的内部表示从utf-16字符数组到字节数组，同时添加一个encoding-flag的标志位，新的String类将根据字符串的内容存储编码为ISO-8859-1/Latin-1（每个字符一个字节）或者UTF-16（每个字符两个字节）来确定存储一个字符所需的字节数。

简单说jdk9之后使用byte[]加上编码标记，如果编码是ISO-8859-1/Latin-1，就用一个byte存储字符串，如果是UTF-16就用两个byte存储。

```java
数据类型 占字节数
byte		1
char		2
short       2
int  		4  
long		8    
float 		4
double		8    
boolean 	1或者4   根据不同jvm实现有所不同   
引用类型   	 4/8      如果32位机器是4个字节，如果64位机器8字节
```

测试

```java
public void test1(){
    String s1 = "abc";  // 字面量定义方式，"abc"存放在堆中的字符串常量池中，而常量池中只有一个"abc"
    String s2 = "abc";
    
    System.out.println(s1 == s2);  // true 两者指向常量池中同一个"abc"
}
```

```java
public class Main {
    String str = new String("good");
    char [] ch = {'t','e','s','t'};

    public void change(String str, char ch []) {
        str = "test ok";	// 改的只是形参，并没有该this.str，this.str != str
        ch[0] = 'b';
    }

    public static void main(String[] args) {
        Main ex = new Main();
        ex.change(ex.str, ex.ch);
        System.out.println(ex.str); // good，这也说明了String是不可变的，它是直接重新生成一个，而不是在原空间去修改
        System.out.println(ex.ch); // best
    }
}
```

String的String Pool是一个固定大小的Hashtable。如果放进string Pool的string非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用string.intern()(在字符串常量池中生成一个字面量对象)时性能会大幅下降。

## String的内存分配

```java
// 分配在字符串常量池中
String s1 = "test";

// 分配在堆中，看反编译的字节码文件得知"str"是从常量池中获取的(ldc)
String s2 = new String("str");

// 如果常量池中不存在"test",则先在常量池中创建一个"test", 则把它的地址赋给s3
String s3 = s1.intern(); 
```

jdk6中字符串常量池在方法区/永久代中，但在7以后它就移动到堆中了。

```java
class Memory{
    public static void main(String[] args) {
        int i = 1;
        Object obj = new Object();
        Memory mem = new Memory();
        mem.foo(obj);
    }
    
    private void foo(Object param){
        /*
        	Object中的toString()方法，没有变量，会在字符串常量池中创建一个param表示的字符串
            public String toString() {
        		return getClass().getName() + "@" + Integer.toHexString(hashCode());
    		}
        */
        String str = param.toString(); // toString()操作会在字符串常量池中创建一个param表示的字符串
        System.out.println(str);
    }
}
```

![对象存储](https://gitee.com/Krains/FigureBed/raw/master/img/%E5%AF%B9%E8%B1%A1%E5%AD%98%E5%82%A8.png)

## 字符串拼接操作

- 常量与常量的拼接结果在常量池，原理是编译期优化
- 常量池中不会存在相同内容的变量
- 只要其中有一个是变量，结果就在堆中。变量拼接的原理是StringBuilder。如果都是字符串常量或者常量引用(带final关键字的)，则仍然是编译期优化。new String("ab")中"ab"是个对象，放在字符串常量池中。
- 如果拼接的结果调用intern()方法，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址

```java
public class StringTest{
    @Test
    public void test1(){
        String s1 = "a" + "b" + "c";
        String s2 = "abc";
        // .java文件编译成.class文件后，经过编译器优化
        // 反编译之后代码是这样的
        // String s1 = "abc";
        // String s2 = "abc"; 因此两个变量都指向字符串常量池中的同一个地址
        System.out.println(s1 == s2);   // true
    }
    
    @Test
    public void test2(){
        String s1 = "javaEE";
        String s2 = "hadoop";
        
        String s3 = "javaEEhadoop";
        String s4 = "javaEE" + "hadoop";  // 编译器优化，直接拼接在一起
        // 如果拼接符号的前后出现了变量，则需要在堆空间中new String(), 具体的内容为拼接的结果：javaEEhadoop
        String s5 = s1 + "hadoop";
        String s6 = "javaEE" + s2;
        /*
				编译后字节码文件是实际上s1+s2的操作实际上是这样的
				StringBuilder s = new StringBuilder();
				s.append(s1);
				s.append(s2);
				s7 = s.toString();
				
				而StringBuilder中toString()中直接new了个String对象
				public String toString() {
        			return new String(value, 0, count);
    			}
        */
        String s7 = s1 + s2;
        
        System.out.println(s3 == s4); // true
        System.out.println(s3 == s5); // false
        System.out.println(s3 == s6); // false
        System.out.println(s3 == s7); // false
        System.out.println(s5 == s6); // false
        System.out.println(s5 == s7); // false
        System.out.println(s6 == s7); // false
		// intern(): 判断字符串常量池中是否存在javaEEhadoop，如果存在，则返回常量池中javaEEhadoop的地址
        // 如果常量池中不存在，则在常量池中加载一份javaEEhadoop，并返回此对象的地址
        String s8 = s6.intern();
        System.out.println(s3 == s8); // true
    }
    
    public static void test3() {
        final String s1 = "a"; // 加了final相当于常量
        final String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2;
        System.out.println(s3 == s4); // true
    }
}
```

"+"拼接字符串方式与StringBuilder的append方法对比

```java
    // 方法1耗费的时间：4005ms，方法2消耗时间：7ms
	public static void method1(int highLevel) {
        String src = "";
        for (int i = 0; i < highLevel; i++) {
            src += "a"; // 每次循环都会创建一个StringBuilder对象、String对象
        }
    }

    public static void method2(int highLevel) {
        StringBuilder sb = new StringBuilder(); 
        for (int i = 0; i < highLevel; i++) {
            sb.append("a");
        }
    }
```

## intern()

intern是一个native方法，调用的是底层C的方法，调用intern方法时，如果池中已经包含了由equals(object)方法确定的与该字符串对象相等的字符串，则返回池中的字符串。否则，该字符串对象将被添加到池中，并返回对该字符串对象的引用。

new String("ab")会创建几个对象

```java
/**
 * new String("ab") 会创建几个对象？ 看字节码就知道是2个对象
 *
 */
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("ab");
    }
}
```

我们转换成字节码来查看

```
 0 new #2 <java/lang/String>
 3 dup
 4 ldc #3 <ab>
 6 invokespecial #4 <java/lang/String.<init>>
 9 astore_1
10 return
```

这里面就是两个对象

- 一个对象是：new关键字在堆空间中创建
- 另一个对象：字符串常量池中的对象

new String("a") + new String("b") 会创建几个对象

```java
/**
 * 我们创建了6个对象
*
* 对象1：new StringBuilder()   因为涉及到字符串拼接操作
* 对象2：new String("a")
* 对象3：常量池的 "a"
* 对象4：new String("b")
* 对象5：常量池的 "b"
* 
* 深入剖析：StringBuilder的toString()
* 对象6：toString中会创建一个 new String("ab")
* 而调用toString方法，不会在常量池中生成ab
 *
 */
public class StringNewTest {
    public static void main(String[] args) {
        String str = new String("a") + new String("b");
    }
}
```

字节码文件为

```
 0 new #2 <java/lang/StringBuilder>    // 1
 3 dup
 4 invokespecial #3 <java/lang/StringBuilder.<init>>
 7 new #4 <java/lang/String>  // 2
10 dup
11 ldc #5 <a>   // 3
13 invokespecial #6 <java/lang/String.<init>>
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 new #4 <java/lang/String>  // 4
22 dup
23 ldc #8 <b>     // 4
25 invokespecial #6 <java/lang/String.<init>>
28 invokevirtual #7 <java/lang/StringBuilder.append>
31 invokevirtual #9 <java/lang/StringBuilder.toString> // 6
34 astore_1
35 return
```

有关intern()的练习

```java
String s = new String("1");  // 执行该行代码之后，在常量池中就有了"1"
s.intern(); // 将该对象放入到常量池。但是调用此方法没有太多的区别，因为已经存在了"1"
String s2 = "1";
System.out.println(s == s2); // jdk6: false   jdk7/8 : false

String s3 = new String("1") + new String("1"); // s3变量记录的地址为：new String("11")，此时常量池中不存在"11"
s3.intern();  // jdk6：创建了一个新的对象"11"，在永久代中
 			  // jdk7: 此时常量池中并没有创建"11"，而是创建一个指向堆空间的new String("11")的引用，也就是说常量池中的"11"对象存的是引用地址，这样节省空间
String s4 = "11";  // 也用的是常量池中"11"对象的引用地址，指向堆空间的"11" ，如果这行代码在s3.intern()之前，为false
System.out.println(s3 == s4); // jdk6: false   jdk7/8:true
```

```java
String s = new String("a") + new String("b");
String s2 = s.intern();
System.out.println(s2 == "ab"); // jdk6: true  jdk7/8: true
System.out.prnitln(s == "ab"); // jdk6:false    jdk7/8: true
```

![intern方法](https://gitee.com/Krains/FigureBed/raw/master/img/intern%E6%96%B9%E6%B3%95.png)

```java
String s1 = new String("ab");    // 会在常量池中生成对象"ab"
// String s1 = new String("a") + new String("b");  // 不会在常量池中生成对象"ab"
String s2 = s1.intern();
System.out.println(s1 == s2); // jdk7/8: false
```

总结string的intern()的使用：

JDK1.6中，将这个字符串对象尝试放入串池。

- 如果串池中有，则并不会放入。返回已有的串池中的对象的地址
- 如果没有，会把此**对象复制一份**，放入串池，并返回串池中的对象地址

JDK1.7起，将这个字符串对象尝试放入串池。

- 如果串池中有，则并不会放入。返回已有的串池中的对象的地址
- 如果没有，则会把**对象的引用地址**复制一份，放入串池，并返回串池中的引用地址



