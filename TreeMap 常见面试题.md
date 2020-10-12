# TreeMap 常见面试题

## 红黑树的插入

### 红黑树的 5 个性质

1. 根节点是黑色

2. 所有叶子节点都是黑色，且值为 NULL

3. 新插入的节点是红色

4. 从根节点到叶子节点的所有路径上不能出现两个连续的红色节点

5. 从任一节点到其每个叶子节点的所有路径上都包含相同数目的黑色节点

### 约定俗成的叫法

`x`：新插入的节点（ 也可以叫当前节点 ）

`parentOf(x)`：新插入的节点的**父亲节点**

`parentOf(parentOf(x))`：新插入的节点的**爷爷节点**

`leftOf(parentOf(parentOf(x)))` 或 `rightOf(parentOf(parentOf(x)))`：新插入的节点的**叔叔节点**

### 插入公式

循环的跳出条件 `while (x != null && x != root && x.parent.color == RED) {...}`

如果新插入节点的父亲节点是爷爷节点的左孩子：

* 如果叔叔节点是红色，则只需要将父亲节点和叔叔节点置为黑色，爷爷节点置为红色（ 如果爷爷节点为根节点，那么在跳出 while 循环后会将其再次置为黑色 ）；

* 如果叔叔节点是黑色，

    * 如果新插入元素是父亲节点的左孩子，则需要父亲节点置为黑色，爷爷节点置为红色，然后右旋爷爷节点；（ @1 ）

    * 如果新插入元素是父亲节点的右孩子，则需要先左旋父亲节点，然后再执行 @1 的所有操作。

如果新插入节点的父亲节点是爷爷节点的右孩子：

* 如果叔叔节点是红色，则只需要将父亲节点和叔叔节点置为黑色，爷爷节点置为红色（ 如果爷爷节点为根节点，那么在跳出 while 循环后会将其再次置为黑色 ）；

* 如果叔叔节点是黑色，

    * 如果新插入元素是父亲节点的左孩子，则需要先右旋父亲节点，然后再执行 @2 的所有操作。

    * 如果新插入元素是父亲节点的右孩子，则需要父亲节点置为黑色，爷爷节点置为红色，然后左旋爷爷节点；（ @2 ）

背诵技巧：

无论父亲节点是爷爷节点的左孩子还是右孩子，只要叔叔节点是红色，就不涉及复杂的旋转操作。

如果爷爷节点、父亲节点和新插入节点无法构成三点一线，则需要先反方向旋转一次父亲节点后再修改节点颜色以及后续旋转爷爷节点。

##  TreeMap 和 HashMap 的区别

**1. TreeMap 和 HashMap 都继承自 AbstractMap，同时 TreeMap 还额外实现了 NavigableMap 接口和 SortedMap 接口**

* 实现 **NavigableMap** 接口让 TreeMap 有了对集合内元素的**搜索**能力。例如 firstEntry() 方法可以返回 Map 中包含最小 Key 的 Entry，tailMap() 方法可以返回 Map 中大于或等于指定 Key 的 Entry 集合（ 实现一致性 Hash 算法 ）。

* 实现 **SortMap** 接口让 TreeMap 有了对集合内元素根据 Key 的**排序**能力。默认是按 Key 的升序排序，不过也可以通过指定排序的比较器自定义排序规则。

**2. 数据结构**

TreeMap 使用的是**红黑树**的数据结构；而 1.8 及以后版本的 HashMap 则使用的是**数组 + 单链表 + 红黑树**的数据结构。

**3. 去重的方式**

HashMap 是使用 hashCode() 和 equals() 方法实现去重的。而 TreeMap 则是依靠 Comparator 或 Comparable 来实现基于 Key 的去重。如果 Comparator 不为 null，优先使用 Comparator 的 compare() 方法；如果为 null，则使用 Key 实现的自然排序 Comparable 接口的 compareTo() 方法。 

## TreeMap 的 put 和 remove 流程分析

### put 流程

```java
public V put(K key, V value) {
    Entry<K,V> t = root; // t 表示当前节点，先把 TreeMap 根节点的引用赋值给当前节点
    if (t == null) {
        compare(key, key);
        
        root = new Entry<>(key, value, null); // 如果当前节点为 null，说明为空树，那么新插入的节点直接就作为根节点。compare(key, key) 的意义是提前校验 Key 是否可以比较，即有没有指定的 Comparator 或 Key 有没有继承 Comparable 并覆写 compareTo() 方法，如果都没有则抛出 NPE
        size = 1;
        modCount++;
        return null;
    }
    int cmp; // 用来接收比较的结果
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator; // 传入在构造方法中指定的 Comparator
    if (cpr != null) { 
        do { // 根据二叉排序树的特性，找到适合新节点的插入位置
            parent = t; // 在进行下一轮循环前，保留父节点的位置
            cmp = cpr.compare(key, t.key);
            if (cmp < 0) // 如果比较的结果小于 0，则转向遍历当前节点的左子树
                t = t.left;
            else if (cmp > 0) // 如果比较的结果大于 0，则转向遍历当前节点的右子树
                t = t.right;
            else
                return t.setValue(value); // 如果相等，则会粗暴地用新的值覆盖当前节点旧的值，并返回旧的值
        } while (t != null); // 如果没有相等的 Key，会一直遍历到 NULL 节点
    }
    else { // 在没有指定 Comparator 的情况下，则调用自然排序的 Comparable 进行比较
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent); // 创建新的节点
    if (cmp < 0) // 还记得之前的 parent 变量吗？如果在上面的遍历中没有遇到相等的 Key，则会把新插入的节点与 parent 变量保存的父节点进行比较。如果比较的结果小于 0，则成为父节点的左孩子。
        parent.left = e;
    else
        parent.right = e; // 如果比较的结果大于 0，则成为父节点的右孩子。
    fixAfterInsertion(e); // 还需要对整棵红黑树进行重新涂色和旋转，以满足红黑树平衡的条件
    size++;
    modCount++; // 操作记录加 1
    return null; // 成功插入新节点后，返回为 null
}
```

### remove 流程

```java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    if (p.left != null && p.right != null) { // 如果待删除的节点同时存在左、右孩子，则获取其后继节点。将其 key 和 value 复制给引用 p 指向的节点的 key 和 value，并将引用 p 移动至后继节点。这样做的目的是，将同时拥有左、右孩子的节点的删除操作转换为对其后继节点的删除，减少平衡红黑树的次数
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    }

    Entry<K,V> replacement = (p.left != null ? p.left : p.right); // 现如今引用 p 已指向待删除节点的后继节点，判断该后继节点是否拥有左或右孩子（ 可以简单地理解为后继节点找个接盘侠 ）

    if (replacement != null) { // 如果后继节点存在代替节点，则先进行代替
        replacement.parent = p.parent; // 将代替节点的爹设置为后继节点的爹，简称爹的传递
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left) // 如果后继节点是其父亲节点的左孩子，则将父亲节点的左子树设置为代替节点
            p.parent.left  = replacement;
        else
            p.parent.right = replacement; // 相反地，将父亲节点的右孩子设置为代替节点

        p.left = p.right = p.parent = null; // 删除后继节点

        if (p.color == BLACK) // 如果后继节点的颜色为黑，则还需要平衡红黑树
            fixAfterDeletion(replacement);
    } else if (p.parent == null) {
        root = null;
    } else { // 如果后继节点不存在代替节点，即此时后继节点为叶子节点，则不需要再进行代替操作。然后先平衡红黑树再删除后继节点
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```
