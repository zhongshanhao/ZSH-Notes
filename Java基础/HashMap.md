---
title:  HashMap源码分析
date: 2020-08-18
categories:
 -  Java基础
---

## jdk1.7中HashMap的实现

底层是用数组加链表实现的

对比Hashtable

- HashMap是线程不安全的
- 支持插入为null值的key和value

重要参数

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
```
默认初始的哈希桶大小16，注意3点
-  无论是调用空参还是指定大小构造器，在第一次put操作时才真正给数组table分配空间
-  capacity必须是2的幂次，如果不是new时指定不是2的幂次会向上找到最近的2的幂次(`roundUpToPowerOf2()`方法)。 至于为什么必须是2的幂次，由于hashCode是int类型的，这一共是42亿个数，要把这42亿个数放到16个桶中，就要通过hashCode计算下标，如果使用%16操作会比较慢，在源码中计算数组下标时是这样的：h & (length-1)，如果length是2的幂次就能保证长度减一的数其二进制从低位到高位全是1，直接与hashCode相与就能得到0到$2^{n}-1$的数组下标。
-  每次扩容扩为原来的2倍

```java
// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 存储键值对的数量
transient int size;
// 扩容阈值
int threshold;
```

负载因子，当size大于threshold时进行扩容，`threshold=capacity * loadFactor`，capacity是当前数组长度，若当前桶大小为16，16*0.75=12，那么当哈希表中的存储的键值对数达到扩容阈值12时扩容，扩容为原来的2倍

为什么是负载因子选择了0.75？

源码注释中是这么写的：0.75的选择是均衡了时间和空间损耗算出来的值，较高的值导致扩容没那么频繁，会减少空间开销，但是增加了hash冲突，链表变长，增加了查找成本，而较小的值会导致频繁扩容，扩容操作非常耗时，而空间利用率不高。

```java
// 桶
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```

put方法

```java
    public V put(K key, V value) {
        // 可以知道在第一次put操作时才给数组分配空间
        if (table == EMPTY_TABLE) {
            // 在构造函数中用threshold存的初始数组大小
            // 该函数中 threshold=capacity * loadFactor
            inflateTable(threshold);
        }
        // 单独给键值为null时提供了一个put方法，直接丢到table[0]的位置
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        // hash & (length - 1)获取下标
        int i = indexFor(hash, table.length);
        // 遍历链表
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 先比较hash值，如果hash值都不相等，那么两个key一定不相等
            // 然后在比较是不是同一个key，最后在调用equals判断两个key是否相等
            // 如果相等，用新值替换旧值，同时返回旧值
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }

	// 首先判断是否扩容，如果需要扩容就将原来的键值对rehash到新的桶中
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            // 如果key是null，丢到table[0]中
            hash = (null != key) ? hash(key) : 0;
            // 重新获得数组下标
            bucketIndex = indexFor(hash, table.length);
        }

        // 使用头插法将新的键值对插入链表，在多线程环境下，这个操作将会导致循环链表
        createEntry(hash, key, value, bucketIndex);
    }
```

get方法

```java
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        // 通过hash值定位到具体的桶中，遍历链表，先比较hash值是否相等，
        // 如果相等再调用equals比较
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

## jdk8中

做了重大更新，底层用的是数组+链表+红黑树

重要参数

```java
// 将链表转化为红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
// 将红黑树转化为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//当数组容量大于 64 时，链表长度大于8时才会转化成红黑树
static final int MIN_TREEIFY_CAPACITY = 64;
```

为什么将链表转红黑树的阈值设置为8？

源码注释是这样写的：理想状况下，随机的hashCode的值在负载因子的条件下所得出的下标频次服从泊松分布，链表各长度（hash碰撞次数）的概率如下：

```java
     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006
```

当链表长度为8时，这个概率不到千万分之一，所以正常情况下几乎不会碰到链表转化为红黑树的情况。然而JDK又不能阻止用户实现这种不好的hashcode方法，这样，hash冲突的概率就会显著增加，链表就会变得很长，那么这时候链表转红黑树就能够极大优化查找时间。

put方法

```java
// onlyIfAbsent=true，不覆盖已存在的值
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    	// 第一次put操作时才初始化数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    
		// 如果通过hash值得出的下标找到的桶位置为null，直接存入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            
            // 首先查看第一个结点的key是否相等，如果相等
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            
            // 如果是红黑树，用红黑树的方式插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            
            // 如果是链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 插到尾部
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度大于等于8，链表转红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果链表中有key值与新增的元素相等，则结束循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // p是e的前一个结点，尾插法用
                    p = e;
                }
            }
            // 找到一个与新增key相同的key，用新值更换旧值，并返回旧值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
    	// 如果键值对数量大于阈值，扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

图解

![HashMap put](https://gitee.com/Krains/FigureBed/raw/master/img/HashMap%20put.png)

get方法

```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            
            // 始终先检查第一个结点的key与要查的key是否相等
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 如果是红黑树，就以红黑树方式查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 如果是链表，遍历链表
                do {
                    // 始终先判断hash值是否相等，只有hash值相等两个key才有可能相等
                    // 减少使用equals的次数，优化运行效率
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

jdk7中的问题

在多线程环境下，JDK 7中数组+链表的HashMap实现，链表有可能出现成环，然后查找的时候遍历链表就产生死循环了，因为多线程环境下线程调度不可预知，这种情况很难重现。

7中在解决hash冲突的时候使用的是头插法，在多线程情况下，扩容时会rehash，这个过程可能会出现环形链表，等get的时候就会出现死循环。

8中的改动

在hash冲突的时候使用尾插法，在rehash时保持原有的顺序，它是如何保证顺序的？

在定位key在桶中的索引时，是使用这样的方法的：`(hash & (length - 1))`，每次扩容扩两倍，比如原来容量是16，有4个1，现在扩容为32位，有5个1，我们只需要检查hash第5位是0还是1就可以知道它rehash的位置，如果hash的第5位是0，就还在原位置，如果第5位是1，那么就在原位置加2^4的位置上，因此，我们可以将链表以第5位来将链表拆分成两个部分，在将第5位是0的链表放在原位置，将第5位为1的链表rehash到其他位置。

hashcode&(16-1)与&(32-1)的区别就在于hashcode的第5位是否是1，如果是0，那么两个值相等，如果是1，那么hashcode&15+16=hashcode&32。

```java
Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
```

如果key是null，索引为0，否者将hashCode值无符号右移16位在异或上hashCode

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // >>> 无符号右移，高位通通补0
    }
```

### 常见问题

为什么重写`equals()`方法必须也要重写`hashCode()`

这两个方法都是再`Object`中的，如果默认的`equals`是`==`作用相同，如果是基本数据类型比较的是值，如果是引用类型比较的对象在内存中的地址，`hashCode()`则是根据内存地址生成的（根据不同虚拟机的实现有所不同，有的是生成一个随机数，但是共同点就是hashcod会存储在其对象头中，以防止垃圾回收时将其移动到不同的内存中，前后的hash值不一致），在Object源码注释中，有这样的规范

- 当`obj1.equals(obj2)`返回true时，两者的哈希值必须相等
- 两者哈希值相等，`obj1.equals(obj2)`不一定为真

这样，如果重写了`equals()`方法而没有重写`hashCode()`方法，即使`obj1.equals(obj2)`为真，那么两个有着不同内存地址的不同对象的哈希值将不会相等，违反了规范。在hashMap中，判断两个对象相不相等先判断两者hashCode值相不相等，如果hashCode不相等就没必要比较了，如果相等在调用`equals`方法判断两者对象是否相等。

如果不重写hashcode，那么对象能够正常put进map中，但是只有拿到put时候的对象的引用才能够从map中取出，因为即使两个相等`equals`对象他的hashcode也会不一样，这样就无法取出对应的value了。

