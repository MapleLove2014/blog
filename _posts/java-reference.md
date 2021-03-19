java共有5种引用类型，默认为强引用。分别通过WeakReference、SoftReference和PhantomReference显示生命弱引用，软引用和虚引用。如果我们打开java的java.lang.ref包会发现还有一个FinalReference。它有什么用，和其他引用有啥区别呢，我们文末再提。

![ref]({{ "/assets/img/notes/ref.png"}})

先说强引用，当我们new出的对象被引用时，此时便是强引用。当GC判断对象没有被引用时，即不可达，便会对其进行垃圾回收。

软引用，是通过SoftReference显示构造出来的引用。被引用的对象在内存空间不够时会被GC垃圾回收，也就是说JVM会保证在抛出OOM前回收掉这些弱引用对象，并且将其放入ReferenceQueue（RQ，它的作用放在最后说）。通常被用作内存敏感的缓存，比如Guava的Cache `CacheBuilder.newBuilder().softValues()` ，通过软引用保证内存空间不足时，清除掉这些缓存的value。

弱引用，通过WeakReference显示构造出来的引用。被引用的对象在任何一次GC中都可能被回收，并被放入RQ中。它最经典的使用场景就是ThreadLocal（TL），TL通过静态内部类ThreadLocalMap（TLM）保存当前线程的所有创建的ThreadLocal实例和值的映射，通过 `WeakReference<ThreadLocal<?>>` 来保存。这里需要注意，线程退出此ThreadLocal实例的作用域时需要手动清空，不然可能会产生内存泄漏的问题，这个有时间写个专题来说。

然后再来说下虚引用，同样通过PhantomReference创建。如果一个对象只被虚引用引用的话，该对象不能再被程序直接引用。`PhantomReference#get` 返回的永远是 `null` 。虚引用引用的对象在任何时候都可能被GC回收，同样在回收前会放入RQ中。它常用来做一些 `clean-up` 的活，比如释放内存，关闭文件等等。比如java nio包下的堆外内存 `DirectByteBuffer` 的构造函数 `DirectByteBuffer(int cap)` ，最后会添加一个cleaner：

```java
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
```

Cleaner（`public class Cleaner extends PhantomReference<Object>`）是一个虚引用，`create` 会把这个虚引用引用到构造的 `DirectByteBuffer` 实例上，同时Cleaner也会把所有的cleaner链在一起。

```java
    private Cleaner(Object target, Runnable deallocator) {
        super(target, dummyQueue);
        this.thunk = deallocator;
    }
```

当应用程序中不再使用DirectByteBuffer，即无任何高于Phantom的引用存在，GC便会回收这个虚引用。由于这个是堆外内存，需要程序手动释放。GC将此虚引用放入RQ前会先将这些虚引用链在一起，即 `Reference#pending` pending链表。同时Reference类加载时会启动一个 `ReferenceHandler` 高优先级守护线程：

```java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```

这个线程会执行 `Reference#tryHandlePending` 方法，如果判断当前pending虚引用是一个Cleaner，则会执行 `Cleaner#clean` 方法：

```java
    // Fast path for cleaners
    if (c != null) {
        c.clean();
        return true;
    }

    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) q.enqueue(r);
```

在本例中，cleaner会执行 `thunk#run` 方法，即 `Deallocator#run`，然后手动释放创建 `DirectByteBuffer` 时申请的堆外内存空间。

```java
    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
```

> 需要注意的是，由于这是一个守护线程去clean，具体什么时候执行是不可预测的。同时clean应该只做一些非阻塞的非常简单的清理工作。

最后看一下 `FinalReference` ，这个类是 `package private` ，所以外部应用程序时无法直接使用的。它有一个子类 `Finalizer` ，内部由VM将实现了 `Object#finalize` 方法的类注册到Finalizer链：

```java
    /* Invoked by VM */
    static void register(Object finalizee) { new Finalizer(finalizee);}
```

注册的时机由 `RegisterFinalizersAtInit` VM参数决定。默认值为 true，在构造函数返回之前注册。如果设置了false，则在对象空间分配好之后进行注册。

和上文的 `ReferenceHandler` 线程类似，Finalizer同样也会启动一个高优先级（优先级低于ReferenceHandler）的守护线程：
```java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        Thread finalizer = new FinalizerThread(tg);
        finalizer.setPriority(Thread.MAX_PRIORITY - 2);
        finalizer.setDaemon(true);
        finalizer.start();
    }
```

当GC检测到一个FinalReference引用的对象不可达时，便会将其添加到RQ中。`Finalizer` 线程会一直轮询RQ，如果有便执行其 `finalize` 方法：

```java
    private void runFinalizer(JavaLangAccess jla) {
        // 省略
                jla.invokeFinalize(finalizee);
        // 省略
    }
```
> 同样需要注意，这是一个守护线程，由于线程调度的原因，什么时候执行 `finalize` 是未知的。极端情况下，一些依赖于 `finalize` 释放资源的对象可能一直没有被回收，容易导致内存泄漏问题。

说完了这5种引用类型，在插一嘴RQ：RQ的存在可以在对象被GC回收前给应用程序一个通知，比如应用程序可以启动一个线程轮询RQ，打印出现在是否有不可达的非强引用对象。

> 注意进入RQ的引用是不能通过get拿出被引用的对象，从RQ中取出的Reference调用get返回的是null，和被引用的对象没有任何关系。

RQ可以被用在`WeakHashMap（WHM）`中通过轮询RQ获取 `WeakReachable` 的弱引用，也就是WHM的Entry。

```java
    Entry(Object key, V value,
            ReferenceQueue<Object> queue,
            int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
```
> 注意这里弱引用的是 `key`，也就是说如果key没有被强引用，这个Entry就会被回收。

拿到即将被回收的Entry之后定位到桶，将该Entry从桶中删除，同时将value置为null，让GC回收value。

```java
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

最后再总结下吧：
在这片文章中，我们分别介绍了Java中的5中引用类型，以及其GC时机和使用场景。
并通过DirectByteBuffer的例子介绍了通过虚引用的Cleaner进行释放堆外内存。
在FinalReference中介绍了Finalizer的注册和执行以及使用的注意事项。
最后通过WeakHashMap介绍了ReferenceQueue的使用场景和作用。




















参考：
https://www.infoq.cn/article/jvm-source-code-analysis-finalreference

https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html

https://github.com/AdoptOpenJDK/openjdk-jdk11/blob/master/src/java.base/share/classes/jdk/internal/ref/Cleaner.java

https://learningviacode.blogspot.com/2014/02/reference-queues.html

https://docs.oracle.com/javase/8/docs/api/java/lang/ref/package-summary.html#package.description

https://stackoverflow.com/questions/58225656/which-reference-finalizer-finalreference-or-weak-phantom-soft-reference-have-h

https://stackoverflow.com/questions/5511279/what-is-a-weakhashmap-and-when-to-use-it