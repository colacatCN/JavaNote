# ArrayList 常见面试题

## ArrayList 和 LinkedList 的区别？

**1. 底层数据结构**

ArrayList 底层使用的是 **Object[] 动态数组**；而 LinkedList 底层使用的是**双向链表**。

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

**Step1：add() 方法**

添加元素之前，会先调用 ensureCapacityInternal() 方法确保数组的容量满足此次插入。
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

**Step2：ensureCapacityInternal() 方法**

当准备插入第 1 个元素时，此时 minCapacity 等于 size + 1 又等于 1，在 Math.max() 方法比较后，minCapacity 变为10。
```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

当准备插入第 1 个元素时，此时 elementData.length 一定是等于 0 的，因为调用无参构造方法时会将一个空数组 {} 赋值给 elementData，同时在执行完 calculateCapacity() 方法后 minCapacity 的值变为 10 后，满足 minCapacity - elementData.length > 0 的条件，所以会进入 grow() 方法进行数组的扩容。

但当准备插入第 2 个元素时，此时 elementData 在成功插入第 1 个元素后其容量被扩容至 10 了，使 minCapacity - elementData.length > 0 条件不再成立，所以不会进入 grow() 方法。直到准备插入第 11 个元素时，此时 minCapacity 的值已经要比 elementData.length 大了，所以会再次进入 grow() 方法进行扩容。
```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

**Step3：grow() 方法**

将 oldCapacity 带符号右移一位，相当于 oldCapacity / 2，因此执行完整行语句后得到 newCapacity = oldCapacity * 1.5。

如果新数组的容量 newCapacity 小于传入的参数要求的最小容量 minCapacity，那么新数组的容量以传入的容量参数为准。（ 调用 addAll() 方法时会一口气插入 N 个元素，致使 minCapacity = size + numNew，旧数组即使扩容了 1.5 倍也不一定能满足需求 ）

如果新数组的容量 newCapacity 大于数组能容纳的最大元素个数 MAX_ARRAY_SIZE，那么再判断传入的参数 minCapacity 是否大于 MAX_ARRAY_SIZE。如果 minCapacity 同样大于 MAX_ARRAY_SIZE，那么 newCapacity 等于 Integer.MAX_VALUE，否者 newCapacity 等于 MAX_ARRAY_SIZE。
```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

**Step 4：Arrays.copyOf() 方法**

Arrays.copyOf() 方法的内部实际上调用了 System.arraycopy() 方法。System.arraycopy(Object src, int srcPos, Object dest, int destPos, int length) 方法需要指定目标数组、拷贝的长度、起点、终点以及存入新数组中的位置，高度定制化；而 Arrays.copyOf(U[] original, int newLength, Class<? extends T[]> newType) 会自动在方法内部新建一个数组，拷贝完成后再将新数组返回。
```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```
