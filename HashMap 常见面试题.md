# HashMap 常见面试题

## 为什么 table 的长度是 2 的 n 次幂？

* 加快哈希计算：为了找到 key 落在哈希表的哪个槽位里，通常需要计算 hash(key) % lengh，但取余的计算效率要比与运算低很多，所以为了保证与运算的结果和取余相同，采用 hash(key) & ( lengh - 1 )。

* 减少哈希冲突：2 的 n 次幂 = 1 个 1 + ( n - 1 ) 个 0，而 2 的 n 次幂减 1 = n 个 1，这样便保证了 hash(key) & ( lengh - 1 ) 在作与运算时，各位得到的结果可能为 0，也可能为 1，达到了均匀散列的效果。

## 1.7 和 1.8 有哪些区别

1. 数据结构

1.7 使用的是**数组 + 单链表**的数据结构；而 1.8 及以后的版本则使用的是**数组 + 单链表 + 红黑树**的数据结构，当单链表的深度达到 8 且整体元素的个数超过 64 的时候，就会自动扩容把单链表转换成红黑树，从而把查询的时间复杂度从 O(n) 降低到 O(logN)。

2. 插入数据的方式

1.7 采用的是头插法，以单链表的形式进行纵向延伸，这种方式容易出现链表的逆序和环形链表的死链问题；而在 1.8 及以后的版本中由于引入了红黑树因此改用尾插法，能够有效避免上述问题。
```java
// 1.7
void createEntry(int hash, K key, V value, int bucketIndex) { 
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);  
    size++;  
} 

// 1.8
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
    // 忽略部分代码
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
    // 忽略部分代码
}
```

3. 扩容的时机

1.7：如果整体元素的个数达到 threshold 这个扩容阈值且数组下标位置已经存在元素才进行扩容。
```java
if ((size >= threshold) && (null != table[bucketIndex])) {
    resize(2 * table.length);
}
```

1.8：
* table 未被初始化
```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```
* 插入后整体元素的个数大于 threshold
```java
if (++size > threshold)
    resize();
```
* 单链表的长度超过 8 但整体元素的个数小于 64
```java
if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
    resize();
```

4. 扩容后数据存储位置的计算方式

1.7：将 hash(key) 和扩容后新数组的长度减 1 作与运算。
```java
static int indexFor(int h, int length) {  
     return h & (length-1);  
 } 
```

1.8：重新映射元素时需要考虑节点类型。对于树形节点，需先拆分红黑树再映射。对于链表类型节点，则需先对链表进行分组然后再映射。需要的注意的是，分组后组内节点相对位置保持不变。
```java
// 假设初始容量为 16
n - 1: 0000 1111

           &

35:    0010 0011 = 3

27:    0001 1011 = 3

19:    0001 0011 = 3

43:    0001 1011 = 3

// 插入上述四个元素后
[0]

[1]

[2]

[3] --> [35] --> [27] --> [19] --> [43] 

[4]

[.]
```
可见 hash(35)、hash(27)、hash(19)、hash(43) 均只有后四位参与到与运算中，当新数组容量扩容至 32 时，此时参与与运算的位数由 4 位变为了 5 位，现在就变成了如下：
```java
n - 1: 0001 1111

           &

35:    0010 0011 = 3

27:    0001 1011 = 19 = oldCap + 原位置

19:    0001 0011 = 3

43:    0001 1011 = 19

// 重新插入上述四个元素后
[0]

[1]

[2]

[3] --> [35] --> [19]

[.]

[19] --> [27] --> [43]
```
这种方式就相当于只需要判断 hash(key) 中新增参与运算的位是 0 还是 1 就能迅速计算出扩容后该元素的存储位置。因此利用 hash(key) & oldCap 如果值为 0，将 loHead 和 loTail 指向这个节点，如果后续还有节点 hash & oldCap 为 0 的话，则将节点继续加入 loHead 指向的单链表中，并将 loTail 指向该节点。如果值为非 0 的话，则让 hiHead 和 hiTail 指向该节点。完成遍历后，可能会得到两条链表，此时就完成了链表分组。


## 为什么在 1.8 中对 HashMap 进行优化的时候，把单链表转换为红黑树的阈值是 8，而不是 7 或者不是 20 呢？

在 1.8 中目前的策略是当单链表的长度大于等于 8 时会自动转换为红黑树；相反的，当红黑树中的元素个数小于等于 6 时会自动退化回单链表，两者中间存在一个差值 7。假设一下，如果设计成单链表的长度超过 8 时就直接树化，小于 8 时则退化回链表，如果一个 HashMap 不停地往同一个槽位插入和删除元素，单链表的个数在 8 左右徘徊，就会频繁地发生树转链表、链表转树，导致整体的执行效率大幅度降低。

还有一点重要的就是由于 TreeNode 的大小大约是普通 Node 的两倍，因此仅在单链表的长度超过 8 且 HashMap 整体包含的元素个数超过 64 时才会使用红黑树，同时 HashMap 中元素的分布情况遵循泊松分布，单链表的长度超过 8 的概率是非常非常小。
