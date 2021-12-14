---
title: 基本计算器II
date: 2021-12-13 18:11:22
tags: 栈
categories: 算法 
---

# [基本计算器 II](https://leetcode-cn.com/leetbook/read/top-interview-questions/xa8q4g/)



个人思路：先利用正则表达式把 字符串分割成2部分 ，一部分存储运算符，一部分存储数字

然后入栈的时候先把乘法和除法计算完。

最后出栈计算减法和加法，  最后做出来就超过了百分之5 .。。。。。 我觉得我这里主要时间差就差在利用正则表达式转化成数组，计算时还要转整数。

```java
class Solution {
    public int calculate(String s) {
        s=s.replaceAll(" ","");
        String[] c1 = s.trim().split("\\d{1,}");
        String[] number = s.trim().split("[-+*/]");
        if (number.length==1){
            return  Integer.parseInt(number[0]);
        }
        LinkedList<String> queuec=new LinkedList<>();
        LinkedList<String> queueNumber=new LinkedList<>();
        int res=0;
        int index=1;
        queueNumber.add(number[0]);
        for (int i = 0; i < c1.length; i++) {
                String s1=c1[i];
                if (!c1[i].equals("")) {
                    switch (s1) {
                        case "*": {
                                String poll = queueNumber.getLast();
                                queueNumber.removeLast();
                                queueNumber.add(String.valueOf((Integer.parseInt(number[index++]) * Integer.parseInt(poll))));
                            break;
                        }
                        case "/": {
                                String poll = queueNumber.getLast();
                                queueNumber.removeLast();
                                queueNumber.add(String.valueOf((Integer.parseInt(poll)/Integer.parseInt(number[index++]))));
                            break;
                        }
                        default:{
                            queuec.add(s1);
                            queueNumber.add(number[index++]);
                            break;
                        }
                    }
                }
        }
        res=Integer.parseInt(queueNumber.poll());
        while (!queuec.isEmpty()){
            String s1=queuec.poll();
            switch (s1){
                case "-":{
                    int a=Integer.parseInt(queueNumber.poll());
                    res=res-a;
                 break;
                }
                case "+":{
                    int a=Integer.parseInt(queueNumber.poll());
                    res=res+a;
                    break;
                }
            }
        }
        return res;
    }
}
```



查看题解学习思路：



看了题解写出来的，主要是熟悉思路,大佬的思路还是一如既往的清晰，从一个题目的解法通用到好多地方，

## [【宫水三叶】使用「双栈」解决「究极表达式计算」问题](https://leetcode-cn.com/problems/basic-calculator-ii/solution/shi-yong-shuang-zhan-jie-jue-jiu-ji-biao-c65k/)

```java
 public int calculate(String s) {
        Map<Character, Integer> map = new HashMap<>();
        map.put('-', 1);
        map.put('+', 1);
        map.put('*', 2);
        map.put('/', 2);
        Deque<Integer> deque = new ArrayDeque<>();
        Deque<Character> ops = new ArrayDeque<>();
       s= s.replaceAll(" ","");
        char[] chars = s.toCharArray();
        int n = s.length();
        for (int i = 0; i < n; i++) {
            if (Character.isDigit(chars[i])) {
                int num = 0;
                int j = i;
                while (j < n && Character.isDigit(chars[j])) {
                    num = num * 10 + chars[j++] - '0';
                }
                deque.addLast(num);
                i = j - 1;
            } else {
                //一定要循环。把栈内元素符优先级小于等于当前运算符的都算了
  
                while (!ops.isEmpty() && map.get(ops.peekLast()) >= map.get(chars[i])) {
                    calc(deque, ops);
                }
                ops.addLast(chars[i]);
            }
        }
        while (!ops.isEmpty()) calc(deque, ops);
        return deque.getLast();
    }

    public void calc(Deque<Integer> deque, Deque<Character> ops) {
        //这个判断是必须的，因为没有2个及以上的数据，根本就不需要计算
        if (deque.size() < 2) return;
        int b = deque.pollLast();
        int a = deque.pollLast();
        char c = ops.pollLast();
        switch (c) {
            case '-': {
                deque.addLast(a - b);
                break;
            }
            case '+': {
                deque.addLast(a + b);
                break;
            }
            case '*': {
                deque.addLast(a * b);
                break;
            }
            case '/': {
                deque.addLast(a / b);
                break;
            }
        }
    }
```



