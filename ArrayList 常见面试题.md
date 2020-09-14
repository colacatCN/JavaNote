# ArrayList 常见面试题

## ArrayList 和 LinkedList 的区别？

**1. 底层数据结构**

ArrayList 底层使用的是 **Object[]** 动态数组；而 LinkedList 底层使用的是**双向链表**。

**2. 插入和删除是否会受到元素位置的影响**

ArrayList 采用了动态数组存储数据，所以插入或删除元素的时间复杂度受元素位置的影响。比如：执行 add(E e) 方法时 ArrayList 会默认将元素追加到数组的末尾，时间复杂度就是 O(1)。但是如果要在指定位置 i 插入或删除元素 add(int index, E element)，此时的时间复杂度就会提高到 O(n - i)，因为在进行上述操作的时候数组中的第 i 号以及之后的 (n - i) 个元素都要执行向后或向前移一位的操作。

LinkedList 采用了双向链表存储数据，所以对于执行 add(E e) 方法进行的元素插入是不受元素位置的影响，时间复杂度近似 O(1)。但是如果要在指定位置 i 插入或删除元素 add(int index, E element)，此时的时间复杂度就近似为 o(n)，因为需要先将指针移动到指定位置再进行插入或删除。

**3. 是否支持快速随机访问**

快速随机访问就是通过索引号快速地获取元素。ArrayList 支持；而 LinkedList 不支持元素的随机访问。

**4. 是否保证线程安全**

ArrayList 和 LinkedList 都是不同步的，也就是无法保证线程安全；

**5. 内存空间占用**

ArrayList 的空间浪费主要体现在数组的末尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间，因为一个 Node 节点不仅要存放数据，还要存放直接后继和直接前驱。

ArrayList 的成员变量 elementData 是被 transient 关键字所修饰，表明其并不会直接参与到序列化的过程中。ArrayList 通过重写 writeObject() 方法使它先调用 defaultWriteObject() 方法序列化那些非 transient 的成员变量，然后通过遍历 elementData 数组，将其中非 null 的元素一并加入序列化，这样一来不仅可以提高序列化的效率，还能够减少空间的开销。

## ArrayList 扩容机制

