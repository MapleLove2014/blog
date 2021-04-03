阻塞队列数据结构保证了元素进入的FIFO顺序，列头的节点是下一个移出的节点，新增加的节点追加到列尾。

当向一个空队列取元素时，该线程会被阻塞，直到被添加元素的线程唤醒。同样，当向一个满队列中添加元素时，线程也会被阻塞，直到被取元素的线程唤醒。这是一个经典的生产者消费者模型，在操作系统课程中也会讲到。可以用两个信号量、一个互斥锁和P，V操作进行模拟。感兴趣可以参考这篇[文章](https://afteracademy.com/blog/the-producer-consumer-problem-in-operating-system)。

初始两个信号量：SEM_EMPTY_SPACE=10和SEM_FILLED=0，互斥量MUTEX，用以保证同时只有一个线程操作队列

添加元素线程：
```
P(SEM_EMPTY_SPACE)
P(MUTEX)
add elements
V(MUTEX)
V(SEM_FILLED)
```
取元素线程：
```
P(SEM_FILLED)
P(MUTEX)
take elements
V(MUTEX)
V(SEM_EMPTY_SPACE)
```

OK，那么在Java中是怎么实现的？

我们先来看ArrayBlockingQueue，它是一个用有限容量数组存储元素的循环阻塞队列。提供了两个主要的添加元素方法offer和put，offer分别支持轮询（如果队列满，马上返回false）和阻塞超时机制。put方法则队列满就无限阻塞，直到等取元素线程通知。相对应的，对于取元素，提供了take，poll和peek。take会在队列为空时阻塞，直到添加元素线程通知。poll对应于offer，分别支持轮询和超时。peek也会返回对头的元素，但是相对于poll和take，不会删除队头。

先来看下主要的数据定义：items保存所有的元素，takeIndex和putIndex分别表示下次要取和添加元素的位置，count时队中的元素个数。

```java
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;
```

接着看一下并发控制，一个可重入锁作为互斥量。和上文模拟不同的是，这里使用两个临界布尔条件notEmpty和notFull作为信号通知，那么程序还需要再进行判断是否可以添加元素和删除元素。
```java
    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

先来看put，先获取互斥量，接着判断队列是否已满。如果队列已满，则阻塞等待取元素线程的notFull信号通知。

```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

通知后，进入enqueue入队方法。将元素入队，同时发送notEmpty通知取元素的线程。

```java
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
```

看完了入队，接着看下出队，以take为例：

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

同样先获取互斥量，然后判断队列是否为空。如果为空，则阻塞等待入队线程的notEmpty信号。通知后，进入dequeue出队方法：

```java
    private E dequeue() {
        final Object[] items = this.items;
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```

同样，出队后发出notFull信号给入队线程。

对于LinkedBlockingQueue，实现原理相同，只不过是采用链表的形式来存储元素。LinkedBlockingDeque和LinkedBlockingQueue类似，不同之处在于使用双向链表的方式存储元素。弱化了FIFO的概念，即程序可以从队列和队尾入队和出队。另外这两个队列如果不指定容量，默认是无限队列（最大Integer.MAX_VALUE）。