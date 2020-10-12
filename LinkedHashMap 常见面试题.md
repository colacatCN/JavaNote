# LinkedHashMap 常见面试题

## 利用 LinkedHashMap 实现 LRU 算法

按照英文的直接原义就是 Least Recently Used，即最近最久未使用算法。最近使用的数据会在未来一段时期内仍然被使用，已经很久没有使用的数据很有可能在未来较长的一段时间内仍然不会被使用。基于这个思想产生了一种缓存淘汰机制，每次从内存中找到最久未使用的数据然后置换出来，并存入新的数据。它的主要衡量指标是使用的时间，附加指标是使用的次数。

## LinkedHashMap

### Entry 的继承体系

```java
Map.Entry<K,V>


    |
    |
    ↓


Node<K,V> --- HashMap.Node            Entry<K,V> --- TreeMap.Entry
    int hash                               K key
    K key                                  V value
    V value                                Entry<K,V> left
    Node<K,V> next                         Entry<K,V> right
                                           Entry<K,V> parent
                                           boolean color = BLACK
    |
    |
    ↓


Entry<K,V> --- LinkedHashMap.Entry
    int hash
    K key
    V value
    Node<K,V> next
    Entry<K,V> before
    Entry<K,V> after


    |
    |
    ↓


TreeNode<K,V> --- HashMap.TreeNode
    int hash
    K key
    V value
    Node<K,V> next
    Entry<K,V> before
    Entry<K,V> after
    TreeNode<K,V> parent
    TreeNode<K,V> left
    TreeNode<K,V> right
    TreeNode<K,V> prev
    boolean red
```

先来简单说明一下上面的继承体系，LinkedHashMap 的内部类 Entry 继承自 HashMap 的内部类 Node，并新增了两个引用，分别是 before 和 after。这两个引用的用途不难理解，也就是用来维护双向链表。同时，TreeNode 继承 LinkedHashMap 的内部类 Entry 后，同样也具备了和其他节点一起组成双向链表的能力。但是这里需要大家考虑一个问题，当使用 HashMap 时 TreeNode 并不需要具备组成双向链表能力。如果继承了 LinkedHashMap 的内部类 Entry，TreeNode 就多了两个用不到的引用，这样做不是会浪费空间吗？

### 核心参数

* `accessOrder`：该字段用来控制是否需要将被访问的节点移动到双向链表的末尾。默认值为 false，即使用双向链表节点插入顺序来排序。

### 建立双向链表

双向链表的建立过程是在第一次插入节点时开始的。初始情况下让 LinkedHashMap 的 head 和 tail 指针同时指向新创建的节点，这样双向链表的骨架就算是建立起来了。之后随着不断有新的节点被插入到 LinkedHashMap 中，将新节点直接接在 tail 指针所指向的节点后面即可。

LinkedHashMap 并没有选择直接覆写父类 HashMap 的 put() 方法，而是直接使用了父类的实现，通过覆写 newNode() 方法实现插入不同类型的数据。在创建新节点的同时把该节点移动到双向链表的末尾。

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

### after 方法三兄弟

* `afterNodeAccess()`：LinkedHashMap 覆写了父类的 get() 方法，在访问完节点后根据 accessOrder 的值判断是否需要将该节点移动到双向链表的末尾。

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

* `afterNodeInsertion()`：LinkedHashMap 在调用父类的 putVal() 完成节点的插入时，判断是否需要删除最近最少访问的节点，删除条件则是由 removeEldestEntry() 来决定。由于设置了 accessOrder 的值为 true，意味着任一节点被访问后都会被移动到双向链表的末尾，因此头节点 header 就是最近最少访问的节点。在父类的 removeNode() 方法内部会回调 LinkedHashMap 的 afterNodeRemoval() 方法来调整双向链表。

```java
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

当需要基于 LinkedHashMap 实现缓存时，可以通过覆写 removeEldestEntry() 方法实现自定义策略的 LRU 缓存。比如可以根据节点数量判断是否移除最近最少被访问的节点，或者根据节点的存活时间判断是否移除该节点（ 大致思路是通过实现 Map.Entry 接口或者继承 HashMap.Node 类来自定义一个节点类型，在其中新增一个字段 createTime 用于记录该节点被创建的时间 ）。

```java
public class SimpleCache<K, V> extends LinkedHashMap<K, V> {

    private static final int MAX_NODE_NUM = 100;

    private int limit;

    public SimpleCache() {
        this(MAX_NODE_NUM);
    }

    public SimpleCache(int limit) {
        super(limit, 0.75f, true);
        this.limit = limit;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > limit;
    }
}
```

* `afterNodeRemoval()`

```java
void afterNodeRemoval(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
