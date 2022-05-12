---
title: STL
---
# STL
## 零、STL总体
### 1、STL包含什么？
1. 容器
2. 迭代器：不暴露容器内部结构，对容器遍历。
3. 算法：排序等常用算法。

### 2、常用的STL/容器？
* 顺序容器
    1. array - 固定大小
    2. vector - 动态数组
    3. list - 双向链表
    4. forward_list - 单向链表
    5. deque - 双端队列
* 关联型
    1. map、multimap、unordered_map、unordered_multimap
    2. set、multiset、unordered_set、unordered_multiset
* 容器适配器
    1. stack - 栈
    2. queue - 队列
    3. priority_queue - 优先队列

### 3、容器内部怎么删除一个元素？
* 顺序型：it = erase(it) 返回下一个有效的it
* 关联型：erase(it++) 返回 void

## 一、vector
### 1、底层原理是什么？
是一个动态数组，装不下的时候->自动申请一片更大地空间，VS 下 1.5 倍或 GCC 下 2 倍，然后复制，释放，原来的迭代器失效。

### 2、为什么是1.5倍或2倍扩容？
* 为什么是成倍增长，而不一个固定大的值？
    1. 复杂度：均摊后可以达到O(1) vs O(n)
    2. 内存的角度：1.5倍的方式可以更好地对内存进行重复利用。因为如果2倍，下一次申请内存会大于之前分配的内存之和，无法再重用。
* 为什么不是以 3 倍或者 4 倍增长呢？ - 可能产生堆空间浪费

### 3、size()和capacity()的区别
* size: 有多少个元素 finish - start
* capacity: 可以容纳多少个元素 end_of_storage - start

### 4、扩容的两种方式reserve()和resize()的区别？
* reserve
    1. 预留空间，没有创建元素对象，
    2. 只改变 capacity()， 比如不能用[]下标去访问。
    3. 只有1个参数。
* resize
    1. 改变容器大小，有创建对象
    2. 改变 capacity()，也改变了size()，可以用[]访问。
    3. 多个参数。

### 5、迭代器失效的情况遇到过吗？
1. 如果插入元素后，改变大小，引起内存重新分配，那就会全部失效
2. 指向的那个元素被删除了，会返回下一个有效的 it = vec.erase(it)

### 6、怎么释放vector的内存？
方法1：
1. vec.clear() - 先清空内容 size() 为 0
2. vec.shrink_to_fit(): 让 capacity 和 size 匹配，也就达到目的了。

方法2：直接vector<int>().swap(foo)

## 二、list
### 1、list 的底层原理知道吗？
底层是一个双向链表，很方便插入和删除。

## 三、deque
### 1、deque 的底层原理知道吗？
底层是一个双端队列，方便在头部和尾部插入和删除。2、什么时候用 vector ? 什么时候用 list ? 什么时候用 deque ?
* vector：随机存取的场景。
* list：插入删除频繁
* deque：需要在头部和尾部操作的时候

## 四、map
### 1、map 底层是什么？
底层是红黑树。
### 2、map 插入方式有哪几种？
1. insert()插入 pair 数据：map.insert( pair<int, string>(1, "xxx"))
2. insert()使用make_pair()函数：map.insert( make_pair(1, "xxx") )
3. insert()插入value_type数据：map.insert( map<int, string>::value_type(1, "xxx" ))
4. 用类似数组的方式直接插入：map[1] = "xx"

### 3、count 和 find 方法有什么区别？
都可以用来判断一个key是否存在。
* count()>0 统计出现的次数，要么 0，要么1
* find() != end() 表示 key 存在。

### 4、map 与 multimap
允不允许重复。

### 5、map 与 set 区别是什么？
set相当于只有 key 没有 value，因为它的value就是key。6、map与unordered_map的区别是什么？怎么选？
1. 查找速度：lgN与O(1)的区别，特别是大数量的情况下。
2. 内存：若要尽量少用内存，用map。
3. hash_map需要考虑hash函数的消耗。

## 五、unordered_map
### 1、底层原理是什么？
是一个哈希表。
* 用开链法解决hash冲突。
* 在设计bucket 数组长度的时候，内置了28个质数，创建时，选择大于等于元素个数的质数，所谓长度（或者说容量）。如果插入的个数超了，就找下一个质数，并重新计算hash值。

* 概念
* 用法
    1. 把所有元素变为0：直接clear()，下次[]访问的时候自动默认为0。
    2. 参考：C++ STL unordered_map容器用法详解 (biancheng.net)

## 六、迭代器
### 1、迭代器的底层原理
* 萃取技术
* 模板piantehua

### 2、迭代器失效的情形
1. 插入时 - vector重新分配内存空间
2. 删除时