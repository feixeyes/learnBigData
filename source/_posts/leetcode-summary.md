---
title: CleanHandBook (LeetCode) summary 
tags:
  - LeetCode
  - 算法
categories: 编程基础
date: 2019-05-24 18:33:48
---


本文总结了  CleanHandBook_v1.0.1.pdf 各题目的解题要点：



## Chapter 1: Array/String(1-16)

* Two Sum: 在数据中查找和等于特定值的两个数。

  Tips:  HashTable    遍历数组中每个值x，x为key，x的索引index为value 存入 map中。如果只判断是否存在，则用set就可以了。

* Two Sum II – Input array is sorted: 同上题，但数组是已排序的。

  Tips：前后两个指针 Ai+Aj> target ，减小j; < target 增大i.

* Two Sum III – Data structure design: 设计一个支持 find (查找Two Sum) 和 add的数据结构。

  Tips：与上题不同的是只判断是否存在两个值，而且可以重复。   用一个hashMap记录值，并且记录值出现的次数。

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

  Tips：多个部分多个循环单独处理即可。(空格、+/-、数字、小数点、数字、空格)

* **Longest Substring Without Repeating Characters**：

   Tips: 方案一，HashMap 逐步缩小窗口法；方案二，hashMap 记录位置，跳跃缩小窗口法。

* **Longest Substring with At Most Two Distinct Characters** ：

   Tips: 方案一，三个变量，i,k分别是窗口的边界，j记录上次字符变换的位置。方案二，滑动缩小窗口法。

* Missing Ranges：

   Tips：start -1 ，end+1。

* Longest Palindromic Substring：

   Tips：遍历回文中心，以中心向外扩张，奇偶两种情况。

* One Edit Distance：

   Tips: 分情况处理

* Read N Characters Given Read4：

   Tips:注意文件结束

* Read N Characters Given Read4 – Call multiple times：

   Tips：上次没用完字符的使用



## Chapter 2: Math(17-19)



17. Reverse Integer

Tips：溢出处理 214748364

17. Plus One

Tips：进位是确定的只会是1，所以从后往前不为9则停止。

17. Palindrome Number

Tips: 先循环找到最高位的位数；

## Chapter 3: Linked List(20-24)

20. Merge Two Sorted Lists

Tips: 基础链表遍历

20. Add Two Numbers

Tips:注意进位

20. Swap Nodes in Pairs

    Tips：pre,p,q,r四个指针

21. Merge K Sorted Linked Lists

    Tips: 方案一，分治法，每次缩小一半数量；方案二，大小为k的小顶堆。

22. Copy List with Random Pointer

    Tips:  关键是怎样找到原始节点的clone节点；方案一，hashMap法，第一遍遍历，创建各clone节点，第二遍，关联clone节点的random指针；方案二，clone节点插入原节点后面法，第一步，创建各clone节点，第二步，关联random指针，第三步分成两个链表。

## Chapter 4: Binary Tree(25-32)

25. Validate Binary Search Tree

    Tips: 方案一，中序遍历递增；方案二，限定节点取值范围，递归传递范围。null。

26. Maximum Depth of Binary Tree

    Tips: 递归

27. Minimum Depth of Binary Tree

    Tips:递归；注意如果某个子树为空则不参与比较。

28. Balanced Binary Tree：

    Tips: 递归，传递子树深度子树本身是否平衡，可以用-1表示不平衡

29. Convert Sorted Array to Balanced Binary Search Tree

    Tips:二分法，递归

30. Convert Sorted List to Balanced Binary Search Tree

    Tips: 中序遍历，要知道左右子树的节点数，从原始链表中逐个取数。

25. Binary Tree Maximum Path Sum

    Tips: 可能是左右子树联通，也可能向父节点延伸。

26. Binary Tree Upside Down

    Tips: 当成特殊链表；方案一，自顶向下递归 ，方案二，字底向上 迭代。

## Chapter 5: Bit Manipulation(33-34)

33. Single Number：

Tips:异或

33. Single Number II：(出现三次，只有一个出现一次)

    Tips:方案一， 记录各位1的总个数，模3；方案二，三个变量分别记录 ones,twos,threes.

## Chapter 6: Misc(35-38)

35. Spiral Matrix

    Tips: m,n 记录行、列可以移动的步数；r,l记录当前位置；四个方向单独处理

36. Integer to Roman

    Tips: IV, IX 等当成独立的字符来处理；从大到小除求商即为该字符的个数。

37. Roman to Integer

    Tips: pre如果比当前小，则减去两倍的pre

38. Clone graph

Tips: 记录以及遍历的节点；宽度优先或深度优先

## Chapter 7: Stack(39-41)

39. Min Stack

    Tips:push和pop 时注意栈顶元素和min的关系。

40. Evaluate Reverse Polish Notation

    Tips:栈基本操作，遇到操作符就去栈中两个数，求值后再压入栈。

41. Valid Parentheses

    Tips:栈基本操作，HashMap记录括号关联关系，左括号进栈，遇到右括号则与栈顶比较。

## Chapter 8: Dynamic Programming(42-47)

42. Climbing Stairs

Tips: 递归、迭代，斐波那契数列

42. Unique Paths

    Tips:方案一， 递归  方案二，优化递归，使用momeration技术，方案三，自底向上，迭代。

43. Unique Paths II

    Tips: 同上，注意障碍物上路径为0

44. Maximum Sum Subarray

    Tips: f(Ai)= Max(f(Ai-1)+Ai,Ai)

45. Maximum Product Subarray

​    Tips: 记录到位置i的最大值和最小值

42. Coins in a Line

## Chapter 9: Binary Search(48-50)



48. Search Insert Position

    Tips:基本二分法

49. Find Minimum in Sorted Rotated Array

    Tips: 二分法简单变体

50. Find Minimum in Rotated Sorted Array II – with duplicates

    Tips: 如果都相等则只移动一位索引。

