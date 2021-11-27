---
title: 单词搜索II
date: 2021-11-27 13:57:33
tags: 树
categories: 算法
---
# [单词搜索II](https://leetcode-cn.com/problems/word-search-ii/)

<!-- more-->
```java

int[][] move = {{1, 0}, {-1, 0}, {0, -1}, {0, 1}};  //下上左右
    Set<String> res = new HashSet<>();
    public List<String> findWords(char[][] board, String[] words) {
        boolean dp[][] = new boolean[board.length][board[0].length];
        Trie trie = new Trie();
        for (String word : words) {
            trie.insert(word);
        }
        for (int i = 0; i < board.length; i++) {
            for (int j = 0; j < board[i].length; j++) {
                    dfs(board, i, j, trie, dp);
            }
        }
        return new ArrayList<>(res);
    }

    private void dfs(char[][] board, int i, int j, Trie trie, boolean[][] dp) {
        if (!trie.map.containsKey(board[i][j])){
            return;
        }
        trie= trie.map.get(board[i][j]);
        if (!trie.getName().equals("")) {
            res.add(trie.getName());
            trie.setName("");
        }
        for (int[] ints : move) {
           int curi= ints[0]+i; int curj=ints[1]+j;
            if (curi >= 0 && curi <= board.length - 1 && curj >= 0 && curj <=board[0].length - 1 &&!dp[curi][curj]){
                dp[i][j]=true;
                dfs(board,curi,curj,trie,dp);
                dp[i][j]=false;
            }
        }

    }
}

class Trie{
     Map<Character,Trie> map;
    private  String name="";
    public Trie() {
        map=new HashMap<>();
    }

    public void insert(String word){
        Trie temp=this;
        for (int i = 0; i < word.length(); i++) {
            char c = word.charAt(i);
            if (!temp.map.containsKey(c)) {
                temp.map.put(c, new Trie());
            }
            temp = temp.map.get(c);
            if (i == word.length() - 1) {
                temp.setName(word);
            }
        }
    }

        public void setName(String name) {
            this.name = name;
    }

    public String getName() {
        return name;
    }
}
```