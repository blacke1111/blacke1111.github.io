---
title: 摆动排序II
date: 2022-01-06 16:28:04
tags: 排序
categories: 算法 
---

# [摆动排序II](https://leetcode-cn.com/leetbook/read/top-interview-questions/xaygd7/)



**解法一**： 直接排序然后插入相应的位置。

```java
public void wiggleSort(int[] nums) {
        Arrays.sort(nums);
        int n= nums.length%2==0?nums.length-2:nums.length-1;
        int max=nums.length-1;
        int min=0;
        int index=1;
        int[] ints = Arrays.copyOf(nums, nums.length);
        while (n>=0){
            if (n>=0){
                nums[n]=ints[min++];
                n-=2;
            }
            if (index< nums.length){
                nums[index]=ints[max--];
                index+=2;
            }
        }
    }
```

**解法二**：利用大根堆和小根堆， 具体可以看我之前的 [数据流中的中位数](https://blacke1111.github.io/2021/12/11/%E6%95%B0%E6%8D%AE%E6%B5%81%E4%B8%AD%E7%9A%84%E4%B8%AD%E4%BD%8D%E6%95%B0/),具体实现有些差异，主要是当小根堆与大根堆元素相等时，插入的是小根堆还是大根堆。

```java
public void wiggleSort(int[] nums) {
        PriorityQueue<Integer> min=new PriorityQueue<>(nums.length);
        PriorityQueue<Integer> max=new PriorityQueue<Integer>(nums.length,(a,b)->{
            return b-a;
        });
        max.add(nums[0]);
        for (int i = 1; i < nums.length; i++) {
            int num=nums[i];
            if (min.size()==max.size()){
                if (num<min.peek()){
                    max.add(num);
                }else {
                    max.add(min.poll());
                    min.add(num);
                }
            }else {
                if (num>max.peek()){
                    min.add(num);
                }else {
                    min.add(max.poll());
                    max.add(num);
                }
            }
        }
        int index=0;
        while (!max.isEmpty()){
            nums[index]=max.poll();
            index+=2;
        }
        index=nums.length%2==0?nums.length-1:nums.length-2;
        while (!min.isEmpty()){
            nums[index]=min.poll();
            index-=2;
        }
    }
```

