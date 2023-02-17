---
layout: post
title: "LC_52: N-Queens II"
subtitle: "LeetCode"
date: 2022-05-18
author: "Michael Xi"
header-img: "img/leetcode.jpg"
tags: [Leetcode]
---

# Problem

> Queen 是一道经典的 **BackTracking** (回溯)题目
> 

![截屏2022-05-18 下午11.03.31.png](https://s2.loli.net/2023/02/17/aKiz32unby7qUDQ.png)

# Thoughts

> 回溯的本质上是暴力枚举，但是回溯可以 identify 不符合条件的 branches 并返回上一步后继续尝试其他的。

回溯一般会是 For loop of recursions，每一个 Rxecursion 都是在 making a choice，在做选择尝试时我们还需要 constraints，也就是要 identify 不符合条件的情况，以及 Goal，也就是 recursion 的 base case。总结来说，对于回溯，我们需要 make a choice, with contraints, to a Final Goal。
> 

👨🏻‍💻 本题中我们有一个 n*n 的棋盘，我们使用回溯去尝试所有可能的情况，并记录下行得通的情况的数量。具体来说，我们从棋盘的每一层 row 入手，并尝试该 row 的全部columns，层层枚举打开像一棵树。在枚举的过程中，我们会发现有些情况会导致两个 Queen 相吃，违反条件，故终止。最后剩下的情况就都是符合条件的 situations。

![Untitled](https://s2.loli.net/2023/02/17/BDw9hEs28pAW5Jx.png)

# Code

```java
class Solution {

    // Main method: driver method
    public int totalNQueens(int n) {
        ArrayList<ArrayList<Integer>> res = new ArrayList<>();  // All possible outcomes
        ArrayList<Integer> colPlacement = new ArrayList<>(); // Queen Placement at each row
        // Parameters: Dimension of Board ➕ Current rows we're solving ➕ List of each column's Placement ➕ List of res
        backTrack(n, 0, colPlacement, res);
        return res.size();
    }

    // BackTracking by rows!
    public void backTrack(int dim, int currentRow, ArrayList<Integer> colPlacement, ArrayList<ArrayList<Integer>> res) {
        // Base case of Recursion (Finish_Line)
        if (currentRow == dim) {
            res.add(new ArrayList<>(colPlacement));
        }
        // For loop of each column
        for (int col=0; col<dim; col++) {
            colPlacement.add(col);
            if (isValid(colPlacement, currentRow)) {
                backTrack(dim, currentRow + 1, colPlacement, res);
            }
            // If not a valid column choice at current row, remove the last input
            colPlacement.remove(colPlacement.size() - 1);
        }
    }

    // Helper method: determine valid position
    private static boolean isValid(ArrayList<Integer> colPlacement, int currentRow) {
        for (int i=0; i<currentRow; i++) {
            int diff = Math.abs(colPlacement.get(i) - colPlacement.get(currentRow));
            if (diff == 0 || diff == currentRow - i) {
                return false;
            }
        }
        return true;
    }
}
```

![截屏2022-05-18 下午11.15.54.png](https://s2.loli.net/2023/02/17/M7rXpQHAKlSuWGx.png)