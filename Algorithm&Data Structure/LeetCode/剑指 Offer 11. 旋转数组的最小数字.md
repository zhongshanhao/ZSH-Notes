---
title: 剑指 Offer 11. 旋转数组的最小数字
date: 2020-07-22
tags:
 - 二分查找
categories:
 -  LeetCode题解
---
::: tip
## [题目链接](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)
:::

二分法就是利用数组的有序性进行排除，每次排除不满足条件的一半，最后剩下的元素就是满足条件的。

因为数组是局部有序，因此要将数组划分成两个部分，对**有序**的一部分进行讨论。

将数组根据mid划分为两个区间，[left, mid-1]和[mid, right]，若

- numbers[mid] == numbers[right]，中间元素与右边界的元素相等，则可以排除掉元素numbers[right]，因为即使numbers[mid]是答案，在区间内[left, right-1]内仍然有numbers[mid]。

> mid和right会不会指向同一个元素？也就是mid与right会相等吗？
> 结论：在循环内部mid一定不等于right
> 前提条件：left != right, 这也是进入循环的条件
> mid =(left + right) / 2 ，由于整型除法的向下取整特性，mid != right，这也说明排除掉的一定是重复的元素

- numbers[mid] < numbers[right]，区间[mid, right]有序，且单调递增，说明numbers[mid]在区间[mid, right]是最小的，而我们要找的是数组中的最小值，因此可令right = mid，排除掉区间(mid, right]，而numbers[mid]不能被排除，它**可能**是我们要找的最小值。

- numbers[mid] > numbers[right]，区间[left, mid]有序，且数组是旋转过的，它把数组的前几个元素挪到了后面，因此它的最小值一定取自区间[mid+1, right]。

能够使用numbers[mid]与numbers[left]做局部区间有序的判断？

> 不能，因为它不能同时解决整体有序和局部有序的情况，看下面两个样例
>
> 3 4 5 1 2
>
> 第一次二分，mid=2，可以判断左侧局部有序，因此区间缩小为[3, 4]，这时是正确的
>
> 1 2 3 4 5
>
> 第一次二分，mid=2，按照上面的逻辑，区间也缩小为[3, 4]，这时把答案排除在了外面，它是错误的

```java
class Solution {
    public int minArray(int[] numbers) {
        int left = 0;
        int right = numbers.length - 1;
        
        while(left < right){
            int mid = left + (right - left) / 2;
            if(numbers[mid] == numbers[right]){
                right--;
            }else if(numbers[right] > numbers[mid]){
                right = mid;
            }else if(numbers[right] < numbers[mid]){
                left = mid + 1;
            }
        }

        return numbers[left];
    }
}
```

### 复杂度分析

- 时间复杂度：$O(log_2N)$，N是数组的长度，$O(log_2N)$是它的平均复杂度

- 空间复杂度：$O(1)$