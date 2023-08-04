---
layout: post
title: "Dynamic Programming"
subtitle: "The art of induction"
date: 2023-08-02
author: "Michael Xi"
header-img: "img/post-bg-2015.jpg"
tags: [Algorithm, Dynamic Programming]
---

# What is Dynamic Programming?

Dynamic programming is based on the concept of **induction**. The approach involves breaking down a large problem into smaller sub-problems and solving them **incrementally**. This process starts with a base case and then uses an **induction rule** to build up to the larger problem, transforming a problem of `size = n - 1` into a problem of `size = n`. Through iterations based on the induction rule, we can ultimately solve the complex problem at hand.

### How Does Dynamic Programming Differ from Recursion?

Recursion is also an algorithm that solves a big problem by dividing it into smaller ones. However, there is a significant difference between dynamic programming and recursion:

- **Approach**: Recursion is a top-down approach that solves a big problem by breaking it into smaller subproblems. Each subproblem has a smaller size until it reaches the base case. In contrast, dynamic programming is a bottom-up approach that starts from the smallest size problem and uses induction rules to incrementally solve bigger problems until it reaches the original problem.
- **Memory Management**: Recursion only considers the base case, subproblems, and recursive rules. It does not record any sub-solutions or states. On the other hand, dynamic programming records the sub-solutions or states of each smaller problem. It uses the sub-solutions of `size < n` problems to induce the solution for a `size = n` problem.
- **Problem Types To Solve**: Recursion and dynamic programming are used to solve different types of problems. Recursion is often used when exploring "all possible solutions" or determining "does this solution exist?" questions. Dynamic programming, on the other hand, is ideal for solving optimization problems such as "What is the largest/smallest?”

### Fibonacci Number Example

If our goal is to find the Fibonacci number of an integer, then recursion and dp will solve this problem differently, here is a visualization:

<img src="https://s2.loli.net/2023/08/04/7QM5PZvzsqje6uN.png" width="350"/> <img src="https://s2.loli.net/2023/08/04/vFRwoKaDbtHMBg6.png" width="350"/>



# How to use Dynamic Programming?

Key Information to Write Down

- **DP[i] meaning**: the meaning of DP[i] is defined as exactly what the question is asking. For example, in the Fibonacci problem, dp[i] means the Fibonacci number of i.

- **Base Case**: what is the answer to the smallest problem that we can get right away? For example, fib(1) and fib(2) are both equal to 1 by default.
- **Induction Rule**: how to transform from a smaller state to a bigger one? You can find the rule by enumerating cases. For example, `fib(n) = fib(n-1) + fib(n-2)` is the induction rule for finding the Fibonacci Number.
- **Return**: what do we want to return? It can be the last element of the dp array or global max/min depending on the context of the problem. For example, in the Fibonacci Number problem, we want to return dp[n].



# Practice Makes Perfect

### Q1: Fibonacci Number

> Get the Kth number in the Fibonacci Sequence. (K is 0-indexed, the 0th Fibonacci number is 0 and the 1st Fibonacci number is 1).

**Clarification & Assumption**

- Example inputs and outputs: 
  - 0th fibonacci number is 0
  - 1st fibonacci number is 1
  - 2nd fibonacci number is 1
  - 6th fibonacci number is 8
- Assume if input K < 0, the function returns 0
- Assume the output Fib number does not overflow long

**Result**

- High Level Algorithm
  - Base case: dp[0] = 0, dp[1] = 1, dp[2] = 1
  - Induction rule: dp[i] = dp[i-1] + dp[i-2]
  - Return dp[K]
- Detail Level Implementation
  ```java
  public long fibonacci(int K) {
    // corner case
    if (K <= 0) {
      return 0;
    }
    long[] dp = new long[K + 1];
    // base case
    dp[1] = 1;
    // induction rule
    for (int i = 2; i <= K; i++) {
      dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[K];
  }
  ```

- Time and Space Complexity

  - TC: O(n)  - Iteration
  - SC: O(n)  - dp array in heap

- **Follow-Up: How to reduce space complexity?**

  Each dp[i] state only relies on the previous two states, so it's unnecessary to keep track of all previous states in a O(n) long dp array. The improved function is:

  ```java
  public long fibonacci(int K) {
    // corner case
    if (K <= 0) {
      return 0;
    }
    // base case
    long prev = 0;
    long cur = 1;
    // induction rule
    for (int i = 2; i <= K; i++) {
      long tmp = prev + cur;
      prev = cur;
      cur = tmp;
    }
    return cur;
  }
  ```



### Q2: Longest Ascending SubArray

> Given an unsorted array, find the length of the longest subarray in which the numbers are in ascending order.

**Clarification & Assumption**

- Example inputs and outputs: 
  - {7, 2, 3, 1, 5, 8, 9, 6}, longest ascending subarray is {1, 5, 8, 9}, length is 4
  - {1, 2, 3, 3, 4, 4, 5}, longest ascending subarray is {1, 2, 3}, length is 3
- Assume the given array is not null

**Result**

- High Level Algorithm

  - Base case: if the input array has length 1, then the longest ascending subarray has length 1

  - Induction rule

    - Case 1: If current element is "keep ascending"

      `dp[i] = dp[i - 1] + 1`

    - Case 2: if current element is "not ascending"

      `dp[i] = 1`

  - Return the globalMax of longest ascending subarray length

- Detail Level Implementation

  ```java
  public int longest(int[] array) {
      // corner case
      if (array.length == 0) {
        return 0;
      }
      // base case
      int[] dp = new int[array.length];
      dp[0] = 1;
      // induction rule
      int globalMax = 1;
      for (int i = 1; i < array.length; i++) {
        if (array[i] > array[i - 1]) {
          dp[i] = dp[i - 1] + 1;
        } else {
          dp[i] = 1;
        }
        globalMax = (globalMax < dp[i]) ? dp[i] : globalMax;
      }
      return globalMax;
    }
  ```

- Time and Space Complexity
  - TC: O(n)
  - SC: O(n)



### Q3: Max Product Of Cutting Rope

> Given a rope with positive integer-length *n*, how to cut the rope into *m* integer-length parts with length *p*[0], *p*[1], ...,*p*[*m*-1], in order to get the maximal product of *p*[0]*p*[1]* ... *p*[*m*-1]? *m* is determined by you and must be greater than 0 **(at least one cut must be made**). Return the max product you can have.

**Clarification & Assumption**

- Example: if n = 12, the max product of cutting rope is 3 * 3 * 3 * 3 = 81
- Assume input n >= 2, so that at least one cut can be made

**Result**

- High Level Algorithm

  - Base case: if the rope has length 0 or 1, then it's invalid, so `dp[0] = -1` and `dp[1] = -1`

  - Induction rule

    - **What does dp[i] represents?** dp[i] represents the max product of cutting rope given a rope with length i.

    - To find the maximum product of a cut rope, we need to consider all possible cutting positions and calculate the product of the different ways of cutting. 

    - Let's assume we cut at position j, where j starts from 1 since we must cut at least once. After the cut, the rope is divided into two parts. 

      - For the left part, we can refer to the previous state's maximum product of cutting with the given length of the left rope. Note that the maximum product of the left part of the rope can be smaller than not cutting at all. 
      
      - For the right part of the rope, we simply multiply to get the product. In the end, the max product at the current rope length is recorded in the dp array.
      
        `dp[i] = Max(j, dp[j]) * (i - j)`
      

  - Return the `dp[n]`, which means the max product of cutting rope with rope length = n

- Detail Level Implementation

  ```java
  public int maxProduct(int length) {
    int[] dp = new int[length + 1];
    // base case
    dp[0] = -1;
    dp[1] = -1;
    // induction - increment rope length
    for (int i = 2; i <= length; i++) {
      // all possible cutting position
      int max = 0;
      for (int j = 1; j < i; j++) {
        int currentCutMaxProduct = Math.max(j, dp[j]) * (i - j);
        max = Math.max(max, currentCutMaxProduct);
      }
      dp[i] = max;
    }
    return dp[length];
  }
  ```
- Time and Space Complexity
  - TC: O(n^2)
  - SC: O(n)