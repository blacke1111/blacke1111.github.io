---
title: 'LC:二叉树中的最大路径和'
date: 2021-11-16 23:45:00
tags: 动态规划
categories: 算法
---
[力扣LC:二叉树中的最大路径和](https://leetcode-cn.com/leetbook/read/top-interview-questions/x2hnpi/):
<!--more-->
```java
private int max=Integer.MIN_VALUE;
    public int maxPathSum(TreeNode root) {
        dfs(root);
        return max;
    }
    public int dfs(TreeNode root){
        if (root==null){
            return 0;
        }
        int left=Math.max(dfs(root.left),0);   //考虑到子节点数如果是负数就不需要+子节点路径数据
        int right=Math.max(dfs(root.right),0);
        int sum=left+right+root.val;
        max=Math.max(max,sum);
        return  root.val+Math.max(left,right);

    }
```
