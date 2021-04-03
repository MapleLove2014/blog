线程池的4种状态：

1. RUNNING：可以接收新的任务，处理队列中的任务
2. SHUTDOWN：不接受新的任务，已经执行的任务和已经入队的任务不受影响
3. STOP：不接受新的任务，不执行入队的任务，中断正在执行的任务
4. TIDYING：所有的任务都已停止执行，所有的线程worker都已经结束，并执行terminated回调
5. TERMINATED：terminated回调执行完毕

线程池创建的时候，默认为RUNNING，随着线程池的运行，其状态会增加。

1. 调用shutdown，状态从RUNNING->SHUTDOWN
2. 调用shutdownNow，状态从RUNNING或者SHUTDOWN->STOP
3. 线程池和队列都为空的时候，状态从SHUTDOWN->TIDYING
4. 线程池为空时，状态从STOP->TIDYING
5. 当terminated回调执行完毕，状态从TIDYING->TERMINATED

当线程池处变为状态TERMINATED时，所有结束的线程执行return，结束线程。

不看源码的话，我们可能会考虑使用两个int成员分别保存状态和线程数量。但是这样的缺点是，两个成员同时操作时不是原子的，需要用锁来保证原子性。

那么在Java的实现中，采用了经典的 `AtomicInteger ctl` 来实现，32位整型数的高3位表示状态，低29位表示线程数量。

我们先来看一下这些数据结构的定义，和一些快捷的计算方法：

首先是ctl的定义了，可以看到默认状态RUNNING，初始线程数量0。ctlOf的计算方法等下再说。
```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

接着看下状态和线程数量的掩码：COUNT_BITS表示线程数量使用的低29位。CAPACITY表示的是低29位的掩码(1FFFFFFF)，即低29位全为1，高3位为0.
```java
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```
再来看具体状态定义的值：高三位可以表示0-7共8种状态，在这里使用到了5种。

```java
    private static final int RUNNING    = -1 << COUNT_BITS;  // FFFFFFFF << 29 = 1110 FFFFFFF，高三位=7
    private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 0怎么移动结果都是0 = 0000 0000省略，高三位=0
    private static final int STOP       =  1 << COUNT_BITS;  // =0010 后面全是0，高三位=1
    private static final int TIDYING    =  2 << COUNT_BITS;  // =0100 后面全是0，高三位=2
    private static final int TERMINATED =  3 << COUNT_BITS;  // =0110 后面全是0，高三位=3
```
> 需要注意的是在数值关系上：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED。这个很好理解，左移以为相当于×2，那么RUNNING=-1×2^29，TIDYING=2×2^29

那么程序怎么快捷的根据这个 `ctl` 获取到状态和线程数量呢？我们来看下面的代码：
```java
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
```

获取线程数量只需要拿到低29位就行了，那么使用低29位的掩码进行与操作就行了，掩盖掉高3位。相反的，我们将这个低29位的掩码取个反，那么就掩盖掉了低29位只取高3位状态值，同样也是通过与操作。

还有一个是通过状态和数量构造一个 `ctl`，`rs` 就是刚才定义的5个状态。由于其低29位都是0，所以直接将数量进行或操作就行了。

```java
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

了解上述之后，我们来看下线程的封装Worker：

```java
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }
    }
```

Worker首先是一个AQS，看注释好像是主要用作中断控制（我也没搞懂😅）。另外Worker实现了Runnable，实现run接口，运行任务。

3个核心成员：
1. thread表示当前Worker的工作线程。注意创建此线程的时候，是把当前Worker（`this`）作为Runnable传进去的。当线程start后，会执行Worker的run方法
2. firstTask表示当前运行的任务
3. completedTasks表示当前线程已经运行的任务数量。

我们接着看它是怎么runWorker的。首先我们知道线程池中的线程是可以重复利用的，但是线程又不能重复启动，那么怎么做到重复使用的呢？既然底层不支持，我们可以在应用层实现。即线程启动后进入一个死循环：获取任务，执行完毕后，继续执行任务，直到达到退出条件后再结束线程。

```java
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
```

首先，是一个阻塞的循环来获取任务。当然如果该Worker创建线程的时候就已经指定了任务，就不需要再去任务队列里取任务了。
接着再判断线程池的状态，回顾上文如果处于STOP状态，是不会运行新提交任务或者队列中的任务。如果线程池状态正常，则会执行真正的任务 `task.run`。
执行完后，也会执行一些回调，做一些 `bookkeeping` 的活。
最后会再次进入循环，获取任务执行任务。

那么线程什么时候退出呢，从while循环可以看出有2种情形：task为null或者getTask抛出异常。

那么我们来看获取任务 `getTask` ：

```java
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
```

首先还是会判断线程池的状态，如果是SHUTDOWN，且工作队列为空则退出线程。或者STOP不再执行新的任务，退出线程。减少worker数量，返回一个null，让runWorker的while循环退出，结束线程。
线程池状态正常，则会判断当前线程是否可以退出，不执行任务。这又分了好几种情况：
1. 是否允许核心线程超时退出
2. 当前线程池的数量是否已经超过了核心线程数量，即当前线程如果为非核心线程，则超时后需要退出
3. 当前线程数量是否已经超过了最大的线程数量
如果上述3个条件满足其一，则需要考虑是否要将此线程退出。这里还有一个条件就是，如果当前工作队列还有任务，线程池只有一个线程（当前线程）时，线程也不会退出。这一点比较好理解，你退出了就没有线程执行任务了。之后再也没有线程去getTask运行任务了，除非应用程序后面还会继续提交任务，创建线程。所以，直接不退出不就好了。

如果该线程确实有资质运行任务，那就去工作队列中取任务了。这里根据超时会有两种取法([参见blocking-queue]())：核心线程如果设置了allowCoreThreadTimeOut也会进行超时处理，无论是什么线程，超时时间都是用的 `keepAliveTime`。如果该线程不能超时，则会使用无超时的 `take` 获取任务。如果需要超时，就用队列的超时机制来达到超时控制的目的。

> 注意这里，如果走的是take，那么take返回的一定不是null。poll如果超时后，没有取到任务才会返回null

这里奇怪的是为什么需要在死循环中获取任务呢？如果走的是take，拿到后会直接返回。如果是poll超时了，线程直接退出就完了，为什么还要循环继续取呢？为了回答这个问题，我们先来看核心线程和非核心线程的本质。其实无论是Worker还是Thread都没有任何一个标识来标明该线程是不是核心线程。线程是不是核心线程是由当前线程根据当前线程池线程数量和核心线程数量对比后主观判断得到的。也就是说，一个线程是不是核心线程是动态变化的。

比如核心线程数2，最大线程数4，当前线程数3。此时线程A获取任务时，发现3>2，就会认为自己是非核心线程。那按道理来说，如果此时A获取任务超时，就需要退出了。但是这里又会进入循环去获取任务，奇不奇怪？为什么这样做呢，考虑下这种场景：线程A和B同时去取任务，同时都判断出3>2，主观认为自己是非核心线程。假如A和B都获取任务超时了，B执行的快，循环在去取任务时，发现3>2，自己仍然认为自己是非核心线程，那么就会直接退出。等B退出后，A进入循环取任务时发现，3-1=2并不大于2，就会主观认为自己是核心线程。那么此时需要需要超时退出，就得根据核心线程的超时机制做出判断了（是否设置了allowCoreThreadTimeOut）。这也就是为什么这里是一个死循环去取任务了。

啃完这个硬骨头后，我们已经大概清除任务是怎么执行的，怎么获取任务，怎么退出线程等等。接着我们再回到调用方看下任务是怎么提交的，任务的结果是如何通过Future返回的。

我们来看执行任务的API，execute(?)和submit(?)。前者表示使用线程池异步执行任务，不关注任务的运行情况，即使任务发生异常等等。submit则会以提交任务的名义，返回一个句柄Future，用户获取任务的运行情况，包括结果，是否结束，是否发生异常等等。

我们这里分析下submit，下面看代码：

```java
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```

很清晰，先将任务封装到FutureTask，然后执行任务，返回Future句柄给调用发。我们重点看下 `execute` ： 

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

其实这里已经注释的很明白了。提交任务3步走：
1. 如果当前线程数还未达到核心线程数，那么直接创建一个Worker执行这个任务。
>  注意这里并不会先将任务入队，在取任务进行执行。

2. 如果已到达核心线程数，就判断是否任务能入队。如果能入队，但是线程池没有运行，则会直接拒绝该任务。另外如果此时线程池没有任何线程，则创建一个非核心线程，从队列中取任务执行。如果都不是，任务入队后，就结束了，让已存在的线程后面到队列中取任务运行。
> 至于Worker到底是直接运行给定的任务，还是从队列中取，根据构造函数的一个参数判断即可。回想下runWorker方法的的While循环，如果task本身不为空，是不会再去队列中取的。

3. 如果任务队列已满，则尝试创建一个非核心线程。如果创建失败（线程池STOP状态或者达到最大线程数量等）则会拒绝该任务。

我们来看下 `addWorker`，这个方法说实话可读性非常差。

```java
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
```

我们先看前面的条件检查。首先第一个if语句来维持线程池状态的定义：
1. STOP或者更后面的状态，不能创建线程
2. SHUTDOWN且任务队列为空，不能创建线程
接着根据是否核心的标识判断当前线程池的数量是否超过对应核心标识的线程数量，如果超过了，也不能创建。

这里通过CAS和重试来进行乐观并发控制，如果CAS失败，进行重试。如果CAS成功，占住了创建线程的坑位，会跳出这个循环 `break retry` ，向下执行创建线程。

我们接着往下看：
```java
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
```

这里就清晰多了，创建一个Worker，然后进行一些判断。这里有一点：如果线程池是SHUTDOWN状态，任务为null，也可以创建线程？我没搞懂这是为什么，有知道的读者可以在评论区留言指出。

线程创建后，便立即使用start启动线程，返回创建线程成功。Worker启动后，便会按照上文提到的流程运行任务。

那么我们拿到Future后是怎么获得结果的呢？（这一块不说太详细了，简单说下核心流程）

当调用Future.get阻塞等待结果时，FutureTask会将该线程作为一个WaitNode节点添加到其实现的lock-free的[Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)。如果有多个线程调用get，则这些WaitNode会通过链表的结构链接在一起。然后通过 `LockSupport.park(this)` 将当前线程阻塞。

那什么时候唤醒呢，当时时任务执行完毕。FutureTask的run方法执行完毕后会进行 `set(result)`，在 `finishCompletion` 方法中通过 `LockSupport.unpark(t)` 唤醒上面等待结果的所有线程。

```java
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```

说到这里关于JAVA线程池的主要实现原理，核心源码大概分析完了（拒绝策略比较简单就不提了）。有一个扩展以后如果有时间再说吧，就是我们的系统有一个统一的traceId来记录一个请求的执行过程。这个traceId可能前端生成的，也可以是系统网关层生成的。比如这个请求需要在一个异步任务中执行，如果出现了异常，通过异常日志中的traceId，我们可以还原这个请求在系统中的执行路径。

最后总结下线程池的运行流程吧：

1. 如果线程数未达到核心线程数，则创建线程执行此任务
2. 如果达到核心线程数，则新提交的任务入队
3. 如果队列已满，且总线程数未超过最大线程数，就创建线程执行此任务
4. 如果队列已满，且总线程数达到最大线程数，执行拒绝策略
5. 如果非核心线程获取任务超时keepAliveTime，退出线程。如果设置了allowCoreThreadTimeOut，核心线程超时也会退出。
