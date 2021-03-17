
LinkedHashMap（LHM）是保持顺序的Map，是HashMap的升级版，扩展和重写了很多HashMap的方法。它的主要内容都继承自HashMap，如果不熟悉请先参考HashMap源码分析的博客。

它特别之处在于会维持两种顺序，默认是插入顺序（least recently inserted），所有节点会按照插入的顺序遍历。它也可以额外维持一种访问顺序（least recently accessed）。在LinkedHashMap的顺序定义中是一种顺序排序（从小到大）。这种顺序根据这两种策略，分别是新插入的节点大于旧插入的节点和新访问的节点大于之前访问的节点。另外需要注意的是在访问顺序中，新插入的节点仍然会作为最大的节点，因为插入的逻辑是一致的，都是放在链表尾部。还有一点是在插入节点后，LinkedHashMap提供了一种淘汰(evict)策略的扩展，这对于有限容量JVM本地缓存（LRI，LRA）的实现提供了很好的帮助。


首先来看数据定义，一个布尔类型的变量表示顺序的选择，默认false，即插入顺序。

```java
    final boolean accessOrder;
```

使用双向链表维持节点的全局顺序
```
    transient LinkedHashMap.Entry<K,V> head;
    transient LinkedHashMap.Entry<K,V> tail;
```

继承了HashMap的Node，增加before和after两个成员，用于维持双向链表。
```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

首先来看它是怎么维持插入顺序的，在HashMap中创建一个节点是通过下面的代码实现的：

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }
```

为了维持链表顺序，LHM重写了这个方法：

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p = new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
```

先把这个节点的Entry创建出来，然后在linkNodeLast中将其追加到链表的末尾，这样就维持了插入顺序。

还记得在HashMap中put完元素后，会调用回调afterNodeInsertion吗，LHM就实现了这个回调，用以实现节点淘汰。比如我们要实现一个LRI或者LRA的本地缓存，由于LHM是继承自HashMap，即不限制容量（最大容量为1 << 30）。在极端情况下可能会造成JVM OOM问题。为此LHM实现了这个回调，通过用户的自定义扩展removeEldestEntry判断是否可以删除最老的元素（默认不可以），来删除这个最老的节点，也就是链表的头节点。

```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

看完插入顺序我们接着看访问顺序，这个得需要构造函数 `LinkedHashMap#LinkedHashMap(int, float, boolean)`指定。我们来看当节点被访问时做了什么：

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

当定义了访问顺序时，会调用afterNodeAccess。这个方法如果看过HashMap也会很熟悉，因为在其中也会被回调。

可以看到在这个方法中，将当前访问的节点放在了链表尾部，即最大顺序。

```java
    void afterNodeAccess(Node<K,V> e) { // move node to last
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

最后我们看下LHM是怎么遍历的，先看迭代器的构造函数。由于此时还没有遍历元素，current为null。由于按照顺序遍历，next定位到链表头节点，表示下次遍历的节点。然后记录下modCount，用以检测是否在迭代过程中修改过。

```java
    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }
```

获取下一个节点比较清晰明了了，返回遍历的检点，更新next为下一个节点。
```java
    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after;
        return e;
    }
```

我们继续看下iterator是怎么删除节点的，和HashMap的差不多。删除后，更新modCount。

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
