---
title: 二叉搜索树中第K小的元素
date: 2021-11-29 14:20:29
tags: 动态规划
categories: 算法
---

# [二叉搜索树中第K小的元素](https://leetcode-cn.com/leetbook/read/top-interview-questions/xazo8d/)



中序遍历刚好就是从小到大输出，我使用了一个数组表示当输出第k个最小值时返回，为什么要用数组呢，因为如果用一个整型变量在方法中就是传递是值：

```java

 int count;
    public int kthSmallest(TreeNode root, int k) {
        int [] dp=new int[1];
        count=0;
        dfs(root,dp,k);

        return dp[0];
    }

    private void dfs(TreeNode node, int[] dp, int k) {
        if (count==k||node==null){
            return;
        }
        dfs(node.left,dp,k);
        if (count==k){
            return;
        }
        count++;
        dp[0]=node.val;
        dfs(node.right,dp,k);
    }
```

