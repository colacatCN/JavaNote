# HashMap 常见面试题

## 为什么 table 的长度是 2 的 n 次幂？

* 加快哈希计算：为了找到 key 落在哈希表的哪个槽位里，通常需要计算 hash(key) % lengh，但取余的计算效率要比与运算低很多，所以为了保证与运算的结果和取余相同，采用 hash(key) & ( lengh - 1 )。

* 减少哈希冲突：2 的 n 次幂 = 1 个 1 + ( n - 1 ) 个 0，而 2 的 n 次幂减 1 = n 个 1，这样便保证了 hash(key) & ( lengh - 1 ) 在作与运算时，各位得到的结果可能为 0，也可能为 1，达到了均匀散列的效果。

## 1.7 和 1.8 有哪些区别

**1. 数据结构**

1.7 使用的是**数组 + 单链表**的数据结构；而 1.8 及以后的版本则使用的是**数组 + 单链表 + 红黑树**的数据结构，当单链表的深度达到 8 且整体元素的个数超过 64 的时候，就会自动扩容把单链表转换成红黑树，从而把查询的时间复杂度从 O(n) 降低到 O(logN)。

**2. 插入数据的方式**

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

**3. 扩容的时机**

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

**4. 扩容后数据存储位置的计算方式**

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

## HashMap 存在哪些线程不安全的问题？

**1. 数据丢失**

1.7：如果多个线程同时执行 resize，每个线程又都会 new Entry[newCapacity]，这是线程内的局部数组实例，线程之间是不可见的。当迁移完成后，执行 resize 的线程会将该数组实例赋值给**共享变量 table**，从而覆盖其他线程之前的操作，因此在“新表”中插入的数据都会被无情地抛弃。
```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;         
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
       return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable)
{
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```
HashMap 1.7 由于**并发扩容**引起的线程安全问题大致可以总结为以下两种：
* 已遍历区间新增元素会丢失（ 线程 A 在对旧表进行扩容时，线程 B 同时还在往旧表中写入数据 ）
* “新表”被覆盖（ 线程 A 和 B 同时对旧表进行扩容，线程 A 稍微提前了一内内先完成了然后快乐地在新表中写入数据，过了一会儿线程 B 也完成了扩容并将它的“新表”赋值给了 table，导致线程 A 的努力都白费了 ）

由于**并发插入**引起的线程安全问题：在 createEntry() 方法中会将新添加的元素直接放在 slot 槽位上（ 头插法 ），使新添加的元素在下次提取时可以被更快地访问。如果两个线程同时执行到 Entry<K, V> e = table[bucketIndex] 时，那么前一个线程的赋值就会被另一个覆盖掉。
```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K, V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

1.8：当线程 A 已确定了落槽位发现没有其他元素，若此时让出 CPU 时间分片，而在该重新获得时间前，线程 B 也被分配到了同样的落槽位并提前将数据存了进去，那线程 A 持有的就是一个过期的落槽位，它并不会理会线程 B 的操作，直接粗暴地覆盖之前其他线程的记录，造成了数据丢失。
```java
// putVal()
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

**2. size 不准确**

一方面，成员变量 size 只是被 transient 关键字修饰（ 不参与序列化 ），也就是说，在各个线程工作内存中的 size 副本并不会及时同步到主内存中，因此导致 size 的值会不断地被覆盖。另一方面，在 putVal() 方法中当存完数据后判断是否需要扩容，++size 由于是非原子性操作会导致相同的问题。
```java
/**
 * The number of key-value mappings contained in this map.
 */
transient int size;

// putVal()
if (++size > threshold)
    resize();
```

## HashMap 的 put 和 resize 流程分析

### put 流程

Step 1：如果 table 为空或者 table 的长度为 0，初始化 table（ resize() ）默认长度为 16，扩容阈值为 12。
```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

Step 2：计算待插入元素的落槽位 i，如果 table[i] 为空，则将该元素插入槽位。
```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

Step3：如果 table[i] 不为空

3.1 判断待插入元素的 hash 值是否和槽位内的元素相同，如果相同，则用 newValue 覆盖 oldValue，并返回 oldValue；如果不相同，跳转至 3.2；
```java
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```
    
3.2 如果槽位内的元素属于 TreeNode 类型，则交由 putTreeVal() 方法处理；
```java
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
    
3.3 如果槽位内的元素属于普通的 Node 类型，则遍历链表，同时利用 binCount 变量统计当前链表的长度。在遍历时如果链表中的元素和待插入元素的 hash 值相同，则用 newValue 覆盖 oldValue，并返回 oldValue；否则将会遍历到链表末尾，创建一个新的 Node 节点用于存放数据，并根据 binCount 的值判断当前链表的长度是否满足树化的条件；
```java
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
```

Step4：操作统计 + 1，并判断插入元素后是否满足扩容的条件。
```java
++modCount;
if (++size > threshold)
    resize();
```

### resize 流程

Step1：根据当前的实际情况初始化变量 newCap 和 newThr，创建 newTab；
如果调用构造方法时手工指定了初始容量，那么经过 tableSizeFor() 方法处理后得到的 newCap 的值是一个大于初始容量且最近的 2 的整数次幂的数（ 至于为什么是 2 的整数次幂请翻到第一道面试题 ），同时 threshold 的值就等于 newCap * loadFactor；如果此时 oldTab 中已经包含有元素了，那么将 newTab 扩容至 oldTab 的两倍的同时，也将 threshold 也扩大两倍。
```java
Node<K,V>[] oldTab = table;
int oldCap = (oldTab == null) ? 0 : oldTab.length;
int oldThr = threshold;
int newCap, newThr = 0;
if (oldCap > 0) {
    if (oldCap >= MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return oldTab;
    }
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
        newThr = oldThr << 1; // double threshold
}
else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
}
if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
}
threshold = newThr;
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
```

Step2：不同于 1.7 的做法，为了避免并发扩容带来的线程安全问题，1.8 在创建完 newTab 后直接将其引用赋值给了 table，然后再把 oldTab 中的元素逐步搬迁到 newTab 中（ **@1** ）。
接着开始遍历 oldTab，如果槽位中的元素没有后继元素（ 它屁股后面没有挂着链表或红黑树 ），则根据 newCap 重新计算其在 newTab 中的落槽位（ **@2** ）；否则判断其数据类型，如果是 TreeNode 类型则交由 split() 方法来切分红黑树，如果是普通的 Node 类型，此时 1.8 的做法又有别于 1.7，它通过一种“独特”的方式将 oldTab 上的链表拆分成两条不同的链表挂在 newTab 上（ **@4：e.hash & oldCap**，详情请见 1.7 和 1.8 的不同点这道面试题 ）。
```java
table = newTab; // @1
if (oldTab != null) {
    for (int j = 0; j < oldCap; ++j) {
        Node<K,V> e;
        if ((e = oldTab[j]) != null) {
            oldTab[j] = null;
            if (e.next == null)
                newTab[e.hash & (newCap - 1)] = e; // @2
            else if (e instanceof TreeNode)
                ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // @3
            else { // @4
                Node<K,V> loHead = null, loTail = null;
                Node<K,V> hiHead = null, hiTail = null;
                Node<K,V> next;
                do {
                    next = e.next;
                    if ((e.hash & oldCap) == 0) {
                        if (loTail == null)
                            loHead = e;
                        else
                            loTail.next = e;
                        loTail = e;
                    }
                    else {
                        if (hiTail == null)
                            hiHead = e;
                        else
                            hiTail.next = e;
                        hiTail = e;
                    }
                } while ((e = next) != null);
                if (loTail != null) {
                    loTail.next = null;
                    newTab[j] = loHead;
                }
                if (hiTail != null) {
                    hiTail.next = null;
                    newTab[j + oldCap] = hiHead;
                }
            }
        }
    }
}
```

Step3：按理说切分红黑树是没必要单独拎出来作为一个步骤来讲，耐不住这个是我目前唯一一个能看得懂的红黑树操作，不讲可惜了！在正式分析 split() 方法前需要重温一下 TreeNode 的结构，其中 prev 是用来删除下一个节点用的，标记当前节点的前驱节点。
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;
    boolean red;

    // 省略了构造方法和成员方法
}
```

和切分链表的操作类似，切分红黑树同样需要两对指针用于记录不需要移动的链表头部和尾部和需要移动的链表头部和尾部。接着开始遍历整棵红黑树，取出 e 的下一个节点后，先把它赋值为空方便 GC 回收，再将它的 hash 值和 oldCap 按位与判断在 newTab 中的位置是否需要移动。完整地遍历一遍后，如果 loHead 和 hiHead 各自指向的链表中元素个数小于等于 6 的话，则将原先的红黑树退化回链表；否则在 newTab 中重新构建出一棵新的红黑树。
```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

## 分析一下 HashMap 1.7 的死链问题
[HashMap 死链示意图](https://github.com/ColacatCN/MarkdownPhotos/blob/master/HashMap%20%E6%AD%BB%E9%93%BE%E7%A4%BA%E6%84%8F%E5%9B%BE.png)
