---
title:  LinkedList源码分析
date: 2020-08-18
categories:
 -  Java基础
---

重要属性				

```java
    // 存储结点个数
	transient int size = 0;
	// 指向链表的开头结点
    transient Node<E> first;
	// 指向链表的最后一个结点，开始时first和last都为空
    transient Node<E> last;
```

Node结点

```java
    private static class Node<E> {
        E item;	 		// 元素值
        Node<E> next; 	// 下一个结点
        Node<E> prev;	// 上一个结点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

在头部添加结点

```java
    private void linkFirst(E e) {
        // f为旧头
        final Node<E> f = first;
        // 新建结点prev=null，next=f，作为链表的头
        final Node<E> newNode = new Node<>(null, e, f);
        // first指向新头
        first = newNode;
        // 如果插入前为是空链表，则last也指向头，否者将旧头的prev指向新头
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

在尾部添加结点

```java
    void linkLast(E e) {
        // l为旧尾
        final Node<E> l = last;
        // 新尾的prev指向旧尾，next指向null
        final Node<E> newNode = new Node<>(l, e, null);
        // last指向新尾
        last = newNode;
        // 如果插入前是空链表，则first也指向新尾，否者将旧尾的next指向新尾
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

查询索引为index的元素，索引从0开始

```java
    Node<E> node(int index) {
		// 如果索引在前半部分，则从头开始找，否者从后面开始找
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

ArrayList与LinkedList的异同

底层数据结构的不同：ArrayList底层是数组实现的，LinkedList是双向链表实现的，这延伸到数组和链表的区别

> 在内存中数组是连续的，而链表是不连续的，正是这个底层实现的不同，导致了一下几点不同
>
> - 数组支持随机查找，查找效率高，增加删除的时候需要移动元素，效率低，链表不支持随机查找，查找效率低，增加删除的时候改变指针的指向即可，效率高，当然数组访问效率高不仅仅是因为支持随机查找，另一个重要原因是因为局部性原理，根据该原理提出的高速缓存，访问数组时更可能命中缓存，而不需要频繁去内存中查找。
> - 存储同样的元素时链表用的空间比较大，因为还额外存了下一个元素的指针

添加元素的时候：ArrayList需要考虑扩容，而LinkedList则不需要，因为ArrayList底层是用数组实现的，需要连续的一块空间，当ArrayList扩容时，它需要新开辟一块内存空间，在把原来的数据拷贝到新数组中去，而LinkedList而不用考虑，链表的结点是离散的，不要求连续，所以链表可以分布在内存中任一角落

内存使用：ArrayList不用存指针，在存储相同数量的元素的时候，需要的内存空间更低。

