---
title: leetcode summary
tags:
  - LeetCode
  - 算法
categories: 编程基础
date: 2019-05-24 18:33:48
---


本文以 CleanHandBook_v1.0.1.pdf为主框架，按照其分类补充了一些其他的题目，并总结各题目的解题要点：



## Chapter 1: Array/String(1-16)

### CleanCodeHandBook

* Two Sum: 在数据中查找和等于特定值的两个数。

  Tips:  HashTable    遍历数组中每个值x，x为key，x的索引index为value 存入 map中。如果只判断是否存在，则用set就可以了。

* Two Sum II – Input array is sorted: 同上题，但数组是已排序的。

  Tips：前后两个指针 Ai+Aj> target ，减小j; < target 增大i.

* Two Sum III – Data structure design: 设计一个支持 find (查找Two Sum) 和 add的数据结构。

  Tips：不同的数据结构，存储空间和查询性能折中。

* Valid Palindrome: 判断一个字符串是否是回文

  Tips：1）前后两个指针；2）过滤空白字符

* Implement strstr(): 是否包含某个子串

​     Tips：两层循环，暴力解法。

* Reverse Words in a String:  翻转句子中的单词。

  Tips：使用一个StringBuilder；从后向前遍历句子中的单词(空格分割)

* Reverse Words in a String II: 同上，前后无空格，不使用额外存储。

  Tips：翻转两趟，第一趟把整句所有字符翻转；第二趟，翻转每个单词中的字符。

* String to Integer (atoi): 字符串转整形

   Tips：负数判断；溢出判断(2147483647)；从高位循环处理

* Valid Number:  判断是否是合法数字

  Tips：多个部分多个循环单独处理即可。

* **Longest Substring Without Repeating Characters**：

* **Longest Substring with At Most Two Distinct Characters** ：

* Missing Ranges：

* Longest Palindromic Substring：

* One Edit Distance：

* Read N Characters Given Read4：

* Read N Characters Given Read4 – Call multiple times：



### 补充





## Chapter 2: Math(17-19)

### CleanCodeHandBook



17. Reverse Integer
18. Plus One
19. Palindrome Number

 ### 补充



## Chapter 3: Linked List(20-24)

### CleanCodeHandBook

20. Merge Two Sorted Lists
21. Add Two Numbers
22. Swap Nodes in Pairs
23. Merge K Sorted Linked Lists
24. Copy List with Random Pointer



### 补充

## Chapter 4: Binary Tree(25-32)

### CleanCodeHandBook

25. Validate Binary Search Tree
26. Maximum Depth of Binary Tree
27. Minimum Depth of Binary Tree
28. Balanced Binary Tree
29. Convert Sorted Array to Balanced Binary Search Tree
30. Convert Sorted List to Balanced Binary Search Tree
31. Binary Tree Maximum Path Sum
32. Binary Tree Upside Down

### 补充

## Chapter 5: Bit Manipulation(33-34)

### CleanCodeHandBook

33. Single Number：

34. Single Number II：

### 补充

## Chapter 6: Misc(35-38)

### CleanCodeHandBook

35. Spiral Matrix
36. Integer to Roman
37. Roman to Integer
38. Clone graph

### 补充

## Chapter 7: Stack(39-41)

### CleanCodeHandBook

39. Min Stack
40. Evaluate Reverse Polish Notation
41. Valid Parentheses

### 补充

## Chapter 8: Dynamic Programming(42-47)

### CleanCodeHandBook

42. Climbing Stairs
43. Unique Paths
44. Unique Paths II
45. Maximum Sum Subarray
46. Maximum Product Subarray
47. Coins in a Line

### 补充

## Chapter 9: Binary Search(48-50)

### CleanCodeHandBook



48. Search Insert Position
49. Find Minimum in Sorted Rotated Array
50. Find Minimum in Rotated Sorted Array II – with duplicates

### 补充