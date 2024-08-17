---
title: 剑指 Offer 59 - I. 滑动窗口的最大值
date: 2020-07-21
tags:
 - 单调队列
categories:
 -  LeetCode题解
---

::: tip

## [题目链接](https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/)

:::

使用单调队列，队列头部维护了当前窗口的最大值。

为了保证队列头部是窗口内的最大值，有两个操作

- 当新元素进入窗口时，若当前元素比队尾的元素要大，就要将当前队尾的元素出队，直到窗口为空或者队列尾部的元素值大于等于当前元素。

- 当窗口内的元素移出窗口时，要判断当前元素是否处于队列头部，若是，则将队首元素移除。

对滑动窗口不熟悉的同学可以看我总结的滑动窗口：[滑动窗口/双指针](https://krains.gitee.io/blogs/Algorithm&Data%20Structure/Algorithm/%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3.html)

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        int n = nums.length;
        List<Integer> list = new ArrayList<>();
        Deque<Integer> queue = new LinkedList<>();

        int i = 0;
        int j = 0;
        while(j < n){
            // 新元素进入窗口
            while(!queue.isEmpty() && queue.getLast() < nums[j]){
                queue.removeLast();
            }
            queue.addLast(nums[j++]);
            
            while(i < j && j - i > k){
                // nums[i]移出窗口
                if(nums[i] == queue.getFirst())
                    queue.removeFirst();
                i++;
            }

            if(j - i == k)
                list.add(queue.getFirst());
        }

        int[] ans = new int[list.size()];
        for(i = 0; i < list.size(); i++){
            ans[i] = list.get(i);
        }
        return ans;
    }
}
```

### 复杂度分析

- 时间复杂度：$O(n)$
- 空间复杂度：$O(n)$，n是nums的长度。