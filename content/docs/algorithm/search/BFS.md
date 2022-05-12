---
title: BFS广度优先搜索
weight: 2
---
<!-- TOC -->
# BFS广度优先搜索
## 一、二叉树的层序遍历
### 102.二叉树的层序遍历

题目地址：https://leetcode-cn.com/problems/binary-tree-level-order-traversal/

给你一个二叉树，请你返回其按 层序遍历 得到的节点值。 （即逐层地，从左到右访问所有节点）。

### 107.二叉树的层次遍历 II

题目链接：https://leetcode-cn.com/problems/binary-tree-level-order-traversal-ii/

给定一个二叉树，返回其节点值自底向上的层次遍历。 （即按从叶子节点所在层到根节点所在的层，逐层从左向右遍历）

* 相对于102.二叉树的层序遍历，就是最后把result数组反转一下就可以了。

### 199.二叉树的右视图

题目链接：https://leetcode-cn.com/problems/binary-tree-right-side-view/

给定一棵二叉树，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

* 层序遍历的时候，判断是否遍历到单层的最后面的元素

### 637.二叉树的层平均值

题目链接：https://leetcode-cn.com/problems/average-of-levels-in-binary-tree/

给定一个非空二叉树, 返回一个由每层节点平均值组成的数组。

* 层序遍历的时候把一层求个总和在取一个均值。

### 429.N叉树的层序遍历

题目链接：https://leetcode-cn.com/problems/n-ary-tree-level-order-traversal/

给定一个 N 叉树，返回其节点值的层序遍历。 (即从左到右，逐层遍历)。

* 多个孩子
```C++
for (int i = 0; i < node->children.size(); i++) { // 将节点孩子加入队列            
    if (node->children[i]) que.push(node->children[i]);
}
```

### 515.在每个树行中找最大值

题目链接：https://leetcode-cn.com/problems/find-largest-value-in-each-tree-row/

您需要在二叉树的每一行中找到最大的值。

* 层序遍历，取每一层的最大值

### 116.填充每个节点的下一个右侧节点指针

题目链接：https://leetcode-cn.com/problems/populating-next-right-pointers-in-each-node/

给定一个完美二叉树，其所有叶子节点都在同一层，每个父节点都有两个子节点。

* 单层遍历的时候记录一下本层的头部节点，然后在遍历的时候让前一个节点指向本节点

### 117.填充每个节点的下一个右侧节点指针II

题目地址：https://leetcode-cn.com/problems/populating-next-right-pointers-in-each-node-ii/

普通二叉树。一样

### 104.二叉树的最大深度

题目地址：https://leetcode-cn.com/problems/maximum-depth-of-binary-tree/

给定一个二叉树，找出其最大深度。

二叉树的深度为根节点到最远叶子节点的最长路径上的节点数

### 111.二叉树的最小深度

相对于 104.二叉树的最大深度 ，本题还也可以使用层序遍历的方式来解决，思路是一样的。

* 只有当左右孩子都为空的时候，才说明遍历的最低点了。如果其中一个孩子为空则不是最低点