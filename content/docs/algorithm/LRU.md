# LRU
1. 要用哈希表 + 双向链表
2. 哈希表：快速查找节点
3. 双向链表
    1. 队头：最近访问的，或新加入的
    2. 队尾：最久未被访问

题目要求两个方法：
* get（）
    1. 如果不存在，返回-1
    2. 如果存在：
        1. 通过key找哈希表，得到节点
        2. moveToHead 将该节点移到队头
        3. 返回节点的值
* put（）
    1. cache.count(key)不存在的话
        1. new新链表节点
        2. 加到哈希表
        3. addToHead 添加到头部
        4. size++判断是否超出容量，如果超出：
            1. removeTail 删除尾部
            2. 删除哈希表cache.erase
            3. size--
            4. delete链表节点
    2. 否则就是已经存在：
        1. cache[key]  拿到节点
        2. 更新node->value为新的值
        3. moveToHead 将该节点移到队头

主要方法：
* 查找由哈希表负责
* 插入：要插哈希表，要插链表头,size++
* 删除：要删哈希表，要删链表尾，size--
* 变更：链表先删再插


注意:
- 用双向链表，且带伪头节点和伪尾节点
- 每次要注意是不是要同时操作链表和哈希表