# HashMap浅析

## 基本思想

Map在底层需要有一个数据结构，每次从中获取或放入元素的时候，需要**计算该元素的hash值**，进而找到**该元素在存储的数据结构中的位置**。由于对于不同的元素而言，它们的hash值也可能是一样的，因此不可避免地会出现hash碰撞的问题：当前位置已经被其他元素占用。

hash碰撞有很多种解决方法，比如线性探测、平方探测、链地址法等等。

## 1.8版本实现

### 底层数据存储结构

Java中HashMap的实现在底层是利用了`Node<K,V>[] table`这样一个数组来实现。`Node<K,V>`的结构定义如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    ...
}
```

可以看到，`Node<K,V>`实现了`Map.Entry<K,V>`的接口。一个`Node`就是`Map`中的一个条目，同时包含了`key`和`value`。同时，`Node<K,V> next`属性也为之后hash冲突时使用链表解决冲突提供了方便。（冲突时，数组的对应位置已经有一个`Node`，在这个`Node`对应的链表最后插入当前元素即可解决冲突）。而当链表太长时（不小于`TREEIFY_THRESHOLD - 1`，且`table`的长度`>= MIN_TREEIFY_CAPACITY`时），就会将链表转化为红黑树（`treeifyBin`方法）。

### get方法的实现

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

可以看见，`get`方法内部调用了`getNode(hash(key), key)`来获取相应元素。接下来来看一下`getNode`的实现

#### `getNode(int hash, Object key)`

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 1. (tab = table) != null && (n = ta.length) > 0：当前存储元素的table不为空
    /* 2. first = tab[(n-1) & hash] != null: 通过hash与n-1(table长度-1)的位与运算计算该元素应该在的位置（HashMap的实现中，存放数据的table的长度永远为2的幂次）。如果不为空，说明该元素可能存在(因为可能发生hash碰撞，这时获得的就是链表的第一个元素，即first)
    */
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 若first元素与要检查的元素相同，返回first
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 若在当前位置上是个链表或红黑树
        if ((e = first.next) != null) {
            // 如果是红黑树，到红黑树中查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 否则顺着链表一个个查找，找到了就返回
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### put方法的实现

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

可以看到，内部还是调用了`putVal`方法来完成`put`操作

#### putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // 若table为空，resize一下给table分配空间
        n = (tab = resize()).length;
    // 如果根据hash值计算的位置上没有元素，那么直接放就可以了
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 若找到的元素的key与要放的元素的key相同
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 令e = p
            e = p;
        // 若当前位置上是棵红黑树
        else if (p instanceof TreeNode)
            // 红黑树的插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 当前位置为链表，binCount统计该链表的元素个数
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 找到了插入的位置，插入
                    p.next = newNode(hash, key, value, null);
                    // 若binCount不小于TREEIFY_THRESHOLD-1，尝试将链表转化为红黑树
                    // 因为有可能是当前table长度太小，所以不一定是转化为红黑树，可能是resize扩容和rehash
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 或者找到了相同的key，到外部处理
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 不为空说明目前没有插入成功，但有一个key相同的元素，目前e指向的就是这个已经存在的元素
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 若onlyIfAbsent为false，或者原始的value为空，那么更新value
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); // 用于LinkedHashMap，HashMap中该方法方法体为空
            return oldValue;
        }
    }
    ++modCount; // 修改次数
    // 当前元素个数超过threshold，扩容、rehash
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict); // 用于LinkedHashMap，HashMap中该方法方法体为空
    return null;
}
```