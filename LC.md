子数组/前缀和/哈希表/滑动窗口(需要满足单调性，有时候不能用)

#### [437. 路径总和 III](https://leetcode-cn.com/problems/path-sum-iii/)

#### [53. 最大子序和](https://leetcode-cn.com/problems/maximum-subarray/)

#### [134. 加油站](https://leetcode-cn.com/problems/gas-station/)

#### [713. 乘积小于K的子数组](https://leetcode-cn.com/problems/subarray-product-less-than-k/)

KMP

#### [214. 最短回文串](https://leetcode-cn.com/problems/shortest-palindrome/)









final原理 ? 

初始化final

```java
// 如果是static自然不会出现线程安全问题，因为static在类加载的时候初始化了，
// 这个阶段JVM保证线程安全问题
final int a = 20;  	
```

字节码

```java
0: aload_0
1: invokespecial #1				// Method java/lang/Object."<init>":()V
4: aload_0
5: bipush 20
7: putfield #2							// Field a:I
				<-- 写屏障
10: return
```

如果没有加final，多线程环境下读到的a的值可能是0或者是20，原因是`int a=20`这个赋值语句不是原子的，它分两步，第一步是给变量a分配空间，初始化为0，第二步才是赋值操作，如果一个线程执行这条语句的第一步之后，还未执行赋值操作，这时其他线程在对变量a的读取，从而导致a等于0的情况出现。

如果加了final，在putfield指令之后会加入写屏障，保证写屏障之前的指令不会出现在写屏障之后，保证其他线程不会出现读到0的情况。







- 热爱技术，善于学习总结，平时在个人博客上总结一些基础知识和算法题解，博客地址：https://krains.gitee.io/
- 热爱编程，喜欢钻研，具有良好的学习和沟通能力并注重团队合作，开拓创新意识强，能够保持不断进取的精神。
- 热爱运动，乐观开朗，积极上进，适应能力强，能吃苦耐劳，有较强的抗压能力，能够始终保持一颗良好的心态。



- 在校学习成绩优异，学分绩84.15/100，成绩在专业排名前6/63
- 在2018年11月获得第十届全国大学生数据竞赛二等奖
- 大三时曾任班级学习委员，获得优秀学生干部
- 在校获得一次三等奖学金，两次二等奖学金



- 熟悉计算机网络协议，如 HTTP、TCP 和 UDP 等。熟悉操作系统基础知识，如进程管理、内存管理等。熟悉基本数据结构和算法，如二分查找、各种排序算法等。
- 熟悉使用 Java，掌握常用的 Java 集合 。
- 熟悉基本的 JVM 原理，包括内存模型、垃圾回收机制等。
- 熟悉基本的 Java 多线程知识。
- 熟练使用 MySQL 数据库，熟悉事务、索引基本原理。



亮点？

熟悉操作系统、计算机网络、数据结构和算法。

熟悉Java基本知识和JVM原理，熟读 Java 常用集合和JUC源码

# 项目

设计模式

线程池的使用

缓存的使用









