```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
向hashmap中插入一个键值对。onlyIfAbsent设为false，表示不管key是否存在，都强制创建或者更新值。evict用于给LinkedHashMap回调。在HashMap中一共有3处给LinkedHashMap的回调，用以分别维持其访问顺序或者创建顺序。

```java
    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }
```

插入前首先对key进行hash，此方法的注释非常详细。因为map的空间是2^n，索引从0到2^n-1。且定位桶的时候采用的是hash&(2^n-1)，此定位方法只会考虑hash的低n位。为了保证散列的效果，减少冲突，hashmap的实现中加了一个扰乱，将其高位通过按位异或的方式进行扰乱。

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

进入到putVal首先桶数组是否初始化，如果没有就通过扩容进行统一初始化

```java
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
```

接着定位到key散列到的桶，判断当前桶是否为空。如果为空，则创建一个桶，第一个节点就是要插入的键值对。

如果桶不为空，接着便在桶中寻找，判断当前key是否已经存在。p是桶的第一个元素，e是invariant，表示桶中和当前key相同的节点。查找前e为null。

首先判断桶中的第一个元素是否为当前key，如果是更新e，维持invariant。为什么不直接遍历桶中的每个元素判断呢？我猜可能是性能方面的考虑，首先大部分情况下桶中只有一个元素，此时的程序路径更短，性能会高一点。

```java
    if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;
```
接着如果桶已经转化为了红黑树，则交由红黑树去查找或者插入新节点。当红黑树中已经存在当前key时，则返回响应节点。如果不存在，在树中插入一个新节点，返回null，表示之前不存在此key，维持invariant。
```java
    else if (p instanceof TreeNode)
        e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

最后遍历桶中的每个节点（跳过第一个，因为已经判断过了）。如果已经到桶的最后一个节点还没有找到此key，表示桶中不存在此节点。此时创建一个新节点链接到桶的末尾，然后判断桶是否需要树化，注意这里判断条件binCount是根据插入前的桶大小判断的。如果在第二个if中找到已经存在key的节点，则退出。仍然维持了invariant。

```java
    else {
        for (int binCount = 0; ; ++binCount) {
            if ((e = p.next) == null) {
                p.next = newNode(hash, key, value, null);
                if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                    treeifyBin(tab, hash);
                break;
            }
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                break;
            p = e;
        }
    }
```

接着往下看，如果已经存在此key的节点。通过onlyIfAbsent判断下是否需要更新值。因为这是对节点的Access，为了维持LinkedHashMap的访问顺序，这里会进行回调。最后返回旧值给调用方。因为这里没有插入新元素，故也不需要更新map的size，直接进行返回即可。

```java
    if (e != null) { // existing mapping for key
        V oldValue = e.value;
        if (!onlyIfAbsent || oldValue == null)
            e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
```

再看下收尾，先更新modCount，判断是否需要扩容。因为前面直接更新的已经返回了，那么程序走到这里一定是新插入了节点，所以增加size，并回调插入，用以LinkedHashMap维持插入顺序。最后返回null，表示之前不存在此节点的值。

```java
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
```



再来看看扩容的逻辑


首先看下数据结构定义。oldTab，oldCap和oldThr分别代表当前桶数组，桶数组大小和扩容阈值。newCap和newThr分别表示扩容后的新桶数组大小和新扩容阈值。
```java
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
```

接着看第一个if，如果当前桶数组已经初始化过，则直接更新新桶数组大小和新阈值为旧桶数组大小和阈值的2倍。

```java
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
```

如果桶数组没有初始化过，那么会进入下面的elseif。

```java
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
```

那什么情况下oldCap为0，但oldThr又大于0呢，从注释里可以看到是指定初始容量构造的时候，即：

```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

这里初始化会直接将初始容量找一个向上靠近(2^n)的数，直接赋予threshold。也就是说在扩容时，如果之前桶数组没有初始化过，那么oldThr表示的就是要初始化的新桶数组的大小，也就是为什么 `newCap = oldThr`。

如果创建HashMap的时候什么都没指定，那么便会使用默认的桶数组大小和扩容阈值

```java
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
```


接着往下，什么情况下newThr为0呢。我们刚才提到了3个执行的ifelse控制块，只有中间的elseif没有更新这个变量。那么这里便会根据新的桶数组大小计算新的阈值。

```java
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
```

做好上述这些准备后，便正式进入了扩容的逻辑。扩容分为两个部分，一是创建新的桶空间，二是旧数据迁移。在我们分析之前，先看下数据定义：

```java
    threshold = newThr;
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
```

threshold表示新的扩容阈值，newTab和table表示新的桶数组，同时这里也完成了扩容的第一步创建新的桶空间。

接着重点分析数据迁移，706-747行。其实我们不看代码，也可以想出大概的迁移流程：先遍历每个桶，再遍历桶中的每个节点。最后根据节点的hash定位到在新桶空间中的桶，加入进去。JDK本身也是差不多这么实现的。

首先遍历旧桶空间（循环变量j），e表示当前桶中遍历的节点。然后再判断下当前桶是否为空，如果是空就不需要迁移，直接跳过处理下一个桶。

```java
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {/*这里面的代码在下面分析*/}
        }
```

如果桶不为空，就进行数据迁移。

首先旧事重提，如果当前桶只有一个元素(大部分情况)，直接定位到新桶，把节点放进去。如果当前桶已经树化，则直接将扩容逻辑委托给TreeNode处理，具体怎么扩容的，后面再提。

```java
    if (e.next == null)
        newTab[e.hash & (newCap - 1)] = e;
    else if (e instanceof TreeNode)
        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
```

为什么直接覆盖掉新桶？因为HashMap的与的定位方式 `e.hash & (newCap - 1)` 保证了e节点在扩容的时候只能被分配到 `oldBucket = e.hash & (oldCap - 1)` 或者 `oldBucket+oldCap` 两个桶，其他节点也一样，所以可以直接覆盖。举个例子：

当前oldCap = 8 = 1000，那么oldCap - 1 = 111。那么newCap = 16 = 1 0000，newCap - 1 = 1111。
假如当前节点e的hash二进制为....?010，那么该节点在旧桶索引为 ....?010 & 111 = 010。那么当它映射到新空间时，....?010 & 1111 = ? * 2^3 + 010。当有效高位?为0时，e仍然映射到旧位置，如果高位是1，则会映射到 `oldBucket+oldCap`。

理解这种定位方式很重要。扩容时对于旧桶中的节点，我们可以根据这种定位规则分为2类，一类是映射到新桶数组中的旧位置的桶，另外一类是向前跨越oldCap偏移的桶。每一类构造一个链表放到新桶数组就行了。

我们来看代码：


注意注释，每一类的链表都是按照之前的顺序来的。
接着看下变量定义：loHead，loTail分别表示?=0的节点链表，也就是刚才说的第一类。hiHead和hiTail表示?=0的那一类。next表示当前节点e的next节点。

```java
    // preserve order
    Node<K,V> loHead = null, loTail = null;
    Node<K,V> hiHead = null, hiTail = null;
    Node<K,V> next;
```

接着来看分类：遍历每个元素判断 ?等于0还是1，然后按照顺序链接到对应的链表上。

```java
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
```

分类完成后，只需要将每一类放到新桶数组对应的桶就好了。

```java
    if (loTail != null) {
        loTail.next = null;
        newTab[j] = loHead;
    }
    if (hiTail != null) {
        hiTail.next = null;
        newTab[j + oldCap] = hiHead;
    }
```


迁移完当前桶，接着迁移下一个桶，直到所有的桶都处理好，扩容完毕。

对了，上面提到的树节点迁移是怎么做的呢？其实差不多，也是根据定位规则分为两类，然后再放到新的坑位上，具体过程就不分析了。

```java
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next){/* 和链表分类类似 */}
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
```

至此HashMap插入元素和扩容已经分析完了，不过还差treeify和untreeify没有提，如果后面后兴趣再看吧。

这里再说几个面试经常会问的题目：

1. 为什么使用2的n次方作为桶数组大小？

    桶过上文的分析，可以看出2点：1)很方便的按位操作，性能较高。但其实我觉得主要的原因是2)通过按位与的分片方式（和key范围分片类似），保证在扩容时节点只能被放到特定的2个桶中。想象一下如果采用取模的分片方式，这个节点可能会被分到任意的桶中，数据迁移次数比较多。

2. 介绍一下hashmap怎么根据初始容量寻找附近的2的n次方值？

    这里推荐大家看下这个回答 [java - HashMap.tableSizeFor(...). How does this code round up to the next power of 2? - Stack Overflow](https://stackoverflow.com/questions/51118300/hashmap-tablesizefor-how-does-this-code-round-up-to-the-next-power-of-2)


    需要注意的是：2^n = 2^n - 1 + 1, 2^n - 1低n位都是1。
    
    假如一个数为：0000000001010，那这个数对应的2^n -1 = 0000000001111。也就是原数的最高位1开始，后面全部弄成1就行了，最后再+1就得到2^n了。

    首先 00000000010** or 00000000001** = 00000000011**， 注意这里其实已经保证前2位都是1了，那么下次再运算直接右移两位就行了。

    接着右移两位继续计算 00000000011** or 000000000\*\*11 = 0000000001111。
    
    如果后面还有0，下次直接可以右移4位了。

3. HashMap的迭代器是fail-fast吗？

    答案是的，我们来看迭代器的nextNode的源码：

    ```java
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }
    
    ```

    可以看到，当modeCount不等于创建迭代器时的modeCount时，便会直接抛出异常。那么使用迭代器删除节点怎么就没问题呢？接着看代码：

    ```java
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    ```

    在迭代器中remove会更新modCount到expectedModCount上，所以不会有问题。

4. 相比于1.7， 1.8的HashMap都优化了什么？

    1. 引入链表转红黑树
    2. 扩容时不进行rehash，使用高低两链表进行扩容，避免1.7节点迁移量大的问题。同时也保证了一定的顺序性，局部性较好。
    3. 引入扰乱位，优化hash算法
    4. 1.8put使用尾插法，避免1.7头插法在并发下死循环问题