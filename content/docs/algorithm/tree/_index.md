---
weight: 1
title: "二叉树"
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