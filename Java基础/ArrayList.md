---
title:  ArrayList源码分析
date: 2020-08-18
categories:
 -  Java基础
---

源码中的重要属性

- DEFAULT_CAPACITY：默认初始数组大小，默认值是10，无参构造器初始化是空数组，在第一次add()操作后扩容成10
- elementData：真正存放数据的数组
- size：当前数组中真正存放的元素个数

构造器

提供了三种构造器，分别是无参、指定大小、指定初始数据的构造器

```java
    // 无参构造器，初始化为一个空数组，在第一次add操作时将数组扩容为默认初始数组大小10
	public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

	// 指定初始大小
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

	// 指定初始数据
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        // 如果长度初始数据长度不为0
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            // 转化为Object[]类型的数组
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

put方法

```java
    public boolean add(E e) {
        // 确保数组容量足够，如果不够就扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

private void ensureCapacityInternal(int minCapacity) { 
    //确保容量足够 
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity)); 
} 

private void ensureExplicitCapacity(int minCapacity) { 
    modCount++; 
    
    // 如果我们需要的最小容量 大于 当前数组的长度，那就需要扩容了 
    if (minCapacity - elementData.length > 0) 
        grow(minCapacity); 
}

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 每次扩容1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩容后的容量 < 期望的容量，那就让期望容量成为新容量，因为至少需要这么多的
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果扩容后的容量 > jvm能分配的最大值，那么就用 Integer 的最大值，上界
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        //通过复制进行扩容 Arrays.copyOf是通过System.arraycopy实现的，它是native方法
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

remove方法

```java
    public boolean remove(Object o) {
        // 如果查找的元素是null，使用==来判断
        // 否则使用equals
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 将最后一个元素的引用置null，帮助GC清理，否者会产生内存泄露
        elementData[--size] = null; // clear to let GC do its work
    }
```

- ArrayList支持放入null值，所以在查找元素时需要特判元素是否为null，如果是null，用==判断，否者用equals
- 扩容和删除操作都是通过`System.arraycopy`实现的

