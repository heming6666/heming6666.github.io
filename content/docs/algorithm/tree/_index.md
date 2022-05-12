---
weight: 1
title: "二叉树"
bookCollapseSection: true
---
# 二叉树

## 1、解题思路

- **遍历**：通过遍历一遍树可以完成任务，则用 `traverse` 函数配合`外部变量`实现。 **=》回溯**

- **递归分解**：通过子问题/子树的答案得到问题的解，则用 `traverse` 函数递归，利用返回值。**=》 动态规划 =》后序**

## 2、关注点
单独抽出一个节点，它需要做什么？在前/中/后序什么时候做？

## 3、融会贯通
### 3.1、遍历函数 traverse() 的理解
迭代/递归遍历数组、链表、树没什么区别！

{{< columns >}}
#### 数组 
```C++
/* 迭代遍历数组 */
void traverse(int[] arr) {
    for (int i = 0; i < arr.length; i++) {

    }
}

/* 递归遍历数组 */
void traverse(int[] arr, int i) {
    if (i == arr.length) {
        return;
    }
    // 前序位置
    traverse(arr, i + 1);
    // 后序位置
}
```

<---> <!-- magic separator, between columns -->

#### 链表
```C++
/* 迭代遍历单链表 */
void traverse(ListNode head) {
    for (ListNode p = head; p != null; p = p.next) {

    }
}

/* 递归遍历单链表 */
void traverse(ListNode head) {
    if (head == null) {
        return;
    }
    // 前序位置
    traverse(head.next);
    // 后序位置
}
```
{{< /columns >}}

### 3.2、快速排序 -> 前序遍历
构造分界点 -> 递归左右
{{< expand >}}
```C++
vector<int> sortArray(vector<int>& nums) {
        quickSort(nums, 0, nums.size()-1);
        return nums;
    }

    void quickSort (vector<int> &nums, int low, int high) {
        if (low < high) {
            int index = partition(nums,low,high);
            quickSort(nums,low,index-1);
            quickSort(nums,index+1,high);
        } 
    }

    int partition (vector<int> &nums, int low, int high) {
            int mid = low + ((high-low) >> 1);
            if (nums[low] > nums[high]) swap(nums,low,high);
            if (nums[mid] > nums[high]) swap(nums,mid,high);
            if (nums[mid] > nums[low]) swap(nums,mid,low); 

            int pivot = nums[low];
            int start = low;
        
            while (low < high) {
                while (low < high && nums[high] >= pivot) high--;           
                while (low < high && nums[low] <= pivot) low++;
                if (low >= high) break;
                swap(nums, low, high);  
            }
            //基准值归位
            swap(nums,start,low);
            return low;
    }
```
{{< /expand >}}

### 3.3、归并排序 -> 后序遍历
先排左右 -> 合并
{{< expand >}}
```C++
// 定义：排序 nums[lo..hi]
void sort(int[] nums, int lo, int hi) {
    int mid = (lo + hi) / 2;
    // 排序 nums[lo..mid]
    sort(nums, lo, mid);
    // 排序 nums[mid+1..hi]
    sort(nums, mid + 1, hi);

    /****** 后序位置 ******/
    // 合并 nums[lo..mid] 和 nums[mid+1..hi]
    merge(nums, lo, mid, hi);
    /*********************/
}

```
{{< /expand >}}

## 4、题目

* 一棵二叉树共有几个节点？
	* 普通二叉树
		* 后序遍历 - O(N)
	* 满二叉树
		* 计算树的高度 -> 2^h - 1  ( O(logN)
	* 完全二叉树
		* 记录左、右子树的高度：同，则满；否则普通树
		* O(logN*logN)：一棵完全二叉树的两棵子树，至少有一棵是满二叉树。因此两个递归只有一个会真的递归下去，满的一定会触发hl == hr，只消耗 O(logN) 的复杂度而不会继续递归。综上，算法的递归深度就是树的高度 O(logN)，每次递归所花费的时间就是 while 循环，需要 O(logN)，所以总体的时间复杂度是 O(logN*logN)。
* 翻转二叉树
    * 前序序或后序遍历
* 将二叉树展开为链表
    * 后序遍历
    * 将原先的右子树接到当前右子树的末端
* 填充二叉树每个节点的下一个右侧节点指针
    * 定义：填充以root为根的树的...？不符合，不能跨子树 -> 传入左右节点，分别递归
    * 该做：将传入的两个节点连接
    * 前序
* 构造最大二叉树
    * 定义：将给定的子数组构造最大二叉树
    * 该做：找到最大值，作为root, 并赋值左右节点
    * 前序
    * 涉及数组：辅助函数，控制索引
* 前序和中序遍历结果构造二叉树
    * 定义：给定的子数组构造二叉树
    * 该做：找到root节点，确定左右子树的子数组，赋值左右节点。
    * 前序
    * 涉及数组：辅助函数，控制索引，画图
* 寻找重复子树
    * 定义：给定的子树判断重复子树
    * 该做：
        * 以我为root的子树长啥样
        * 其他子树长啥样
        * 描述与对比：二叉树序列化
    * 后序！
* 二叉树序列化与反序列化
    * 序列化：前序遍历（或后序遍历）
    * 反序列化：因为包含了空指针的信息，前序遍历，先左后右即可。（或先右后左）
    * 层次遍历框架：从队列中取根节点，然后把子节点存进队列
* 剑8：二叉树的下一个节点（给定带父指针二叉树和一个节点，找中序遍历序列的下一个节点）
    * 节点有右子树，最左子节点就是
    * 没有右子树
        * 左叶子节点，那父节点就是下一个
        * 右叶子节点，找到一个是它父节点的左孩子的节点
* 剑26：给两棵二叉树，判断是不是子结构
    * 定义：给定的子树有没有包含
    * 该做：判断结构是否相同
        * 定义：判断给定的子树是否相同
        * 该做：值相等？
        * 前序
    * 前序
* 剑28：判断一颗二叉树是不是对称的
    * 分析：前序遍历和对称前序遍历序列是否一样。1树变2树
    * 定义：给定的两棵树是不是一样的
    * 该做：值相等
    * 前序
* 剑32：从上到下打印二叉树
    * 队列/栈
    * 设置对应的变量
* 剑34：给一棵二叉树和一个整数，打印和为整数的所有路径
    * 定义：给定的子树是否有和为某个整数的路径
    * 该做：加上自己的值。为了打印，遍历时节点入栈，返回父节点之前，在路径上删除当前节点
    * 前序
* 剑55：求二叉树的深度：知道左和右子树的深度，取最大者就可以

* 二叉搜索树中第k小/大的元素
    * 中序遍历即为升序排列的结果
* 把二叉搜索树转换为累加树
    * 利用特性
* 判断BST的合法性
    * 如果当前节点会对下面的子节点有整体影响，使用辅助函数，增加函数参数列表，在参数中携带额外信息，将这种约束传递给子树的所有节点
* 在BST中搜索一个数
    * 针对BST的遍历框架
* 在BST中插入一个数
    * 找到空位置，插入新节点
* 在BST中删除一个数
    * 先找，找到后删除
    * 叶子节点 - return null
    * 一个子节点 - 返回该子节点
    * 两个子节点 - 左子树最大 或 右子树最小
* 给一个正整数n, 计算共有多少种不同结构的BST结构
    * 分析：先确定根节点：n种情况，确认后，左子树组合数 * 右子树组合数
    * 定义：计算闭区间 [lo, hi] 组成的BST个数
    * 后序
    * 消除重叠子问题：备忘录
* 给一个正整数n, 构建所有的BST
    * 分析：先穷举根节点的所有可能 - 递归构造子BST - 穷举左右子树的组合
    * 定义：构造闭区间 [lo, hi] 组成的BST，返回根节点
    * 后序
    * 消除重叠子问题：备忘录
* 给一棵二叉树，找到节点之和最大的那颗BST
    * 分析：左右子树是不是BST？加上自己是不是BST? 记录节点之和
    * 定义：给定子树，返回int[] {是不是BST，最小值，最大值，和}
    * 该做：得到左子树、右子树的结果，判断并计算
    * 后序遍历
    * 如果当前节点要做的事情需要通过左右子树的计算结果推导出来，就要用到后序遍历。
* 剑33：给一个整数数组，判断是不是某二叉搜索树的后序遍历结果
    * 定义：给定子数组，是不是
    * 该做：确定左子树、右子树的子数组