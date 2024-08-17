---
title: 剑指 Offer 41. 数据流中的中位数
date: 2020-07-21
tags:
 - 堆
categories:
 -  LeetCode题解
---
::: tip
## [题目链接](https://leetcode-cn.com/problems/shu-ju-liu-zhong-de-zhong-wei-shu-lcof/)
:::

## 暴力法

要想快速找到数据流中的中位数，可以考虑插入排序的思想，每次加入一个数，将这个数插入到数组中，然后能够从已排序数组快速定位到中位数。

如果调用addNum函数N次，它的时间复杂度就是$O(N^2)$，然而题目限制说明会对函数addNum和findMedia调用50000次，这样的时间复杂度是不允许的，我们必须寻找另外一个方法。

## 优先队列/堆

使用一次插入排序的时间复杂度是O(n),怎么优化这个时间呢？
可以使用堆排序，其插入元素的时间复杂度是$O(log_2n)$，但是堆在逻辑上是一颗完全二叉树，物理上它是一个数组，数组是无序的，不能够快速定位到中位数。

我们可以定义一个大根堆minHeap和小根堆maxHeap，两个堆各放一半的元素，将较小的一半放在大根堆中，较大的一半放在小根堆中，这时候中位数：

- minHeap.peek()，若count是奇数，
- (minHeap.peek() + maxHeap())/2，若count是偶数

count是当前元素的个数，当元素为奇数时，把多出来的一个元素放在大根堆minHeap中。

如何保证minHeap的元素都小于maxHeap呢？
我们可以将一个要加入的元素先放入到小根堆maxHeap中，在把小根堆的堆顶元素（即最小的元素）移除放到大根堆中minHeap中，这就保证了minHeap的元素都小于maxHeap的元素。

如果count元素为偶数时，我们还保证两个堆的元素个数相等，要将minHeap堆顶元素移除放入到maxHeap中，此时也满足较小的一半在大根堆中，较大的一半在小根堆中。

```java
class MedianFinder {
    Queue<Integer> minHeap;
    Queue<Integer> maxHeap;
    int count = 0;

    /** initialize your data structure here. */
    public MedianFinder() {
        minHeap = new PriorityQueue<>((n1, n2) -> n2 - n1);
        maxHeap = new PriorityQueue<>();
    }
    
    public void addNum(int num) {
        count++;
        maxHeap.add(num);
        minHeap.add(maxHeap.remove());
        if(count % 2 == 0){
            maxHeap.add(minHeap.remove());
        }
    }
    
    public double findMedian() {
        if(count % 2 == 0){
            return (minHeap.peek() + maxHeap.peek()) / 2.0;
        }else{
            return minHeap.peek();
        }
    }
}
```

### 复杂度分析

假设调用addNum函数n次

- 时间复杂度：$O(nlog_2n)$
- 空间复杂度：$O(n)$