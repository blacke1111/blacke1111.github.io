---
title: 至少有K个重复字符的最长子串
date: 2021-12-20 21:09:58
tags: 归并
categories: 算法   
---

# [至少有K个重复字符的最长子串](https://leetcode-cn.com/leetbook/read/top-interview-questions/xafdmc/)



<!--more-->

主要思想归并：把所有小于k的字符全找出来，然后就可以根据这个条件分区。

```java
int max;
    public int longestSubstring(String s, int k) {
        int []dp=new int[26];
        for (int i = 0; i < s.length(); i++) {
            int a=s.charAt(i)-'a';
            dp[a]=dp[a]+1;
        }
        int flag=1;
        for (int i = 0; i < s.length(); i++) {
            int a=s.charAt(i)-'a';
            if (dp[a]<k){
                flag=0;
                break;
            }
        }
        if (flag==1) {
            max = Math.max(max, s.length());
            return max;
        }
        int start=0;
        for (int i = 0; i < s.length(); i++) {
            int a=s.charAt(i)-'a';
            if (dp[a]<k){
                longestSubstring(s.substring(start,i),k);
                start=i+1;
            }
        }
        longestSubstring(s.substring(start),k);
        return max;
    }
```

