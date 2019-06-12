#### 主要字段分析

**ThreadPoolExecutor** 是Java提供的线程池默认实现
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // 线程池运行状态runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 从ctl中获取运行状态、工作线程总数
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    //等待任务队列
    private final BlockingQueue<Runnable> workQueue;
    //主要的锁对象
    private final ReentrantLock mainLock = new ReentrantLock();
    //工作线程集合
    private final HashSet<Worker> workers = new HashSet<>();
    
    private final Condition termination = mainLock.newCondition();
    
    private int largestPoolSize;
    
    private long completedTaskCount;
    
    private volatile ThreadFactory threadFactory;
    //拒绝策略
    private volatile RejectedExecutionHandler handler;
    //非核心线程保持alive的时间
    private volatile long keepAliveTime;
    //是否允许核心线程超时失效
    private volatile boolean allowCoreThreadTimeOut;
    //核心线程数量
    private volatile int corePoolSize;
    //允许的最大线程数量
    private volatile int maximumPoolSize;
}
```
在 **ThreadPoolExecutor** 中 使用 ==*ctl*== 变量作为主要控制状态。==*ctl*== 是一个 *AtomicInteger* 类型对象，其内部保存了两个线程池运行控制状态：
- workerCount： 线程运行状态的有效数量；存在低29位中
- runState：    标识线程池的运行状态；存在高3位中

---

#### execute方法分析
下面从 *execute()* 方法开始分析 **ThreadPoolExecutor** 内部到底是怎么工作的
```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         * 1. 如果运行的线程小于corePoolSize，则尝试用给定的命令作为第一个任务启动一个新线程。
         *    对addWorker的调用原子性地检查runState和workerCount，因此可以通过返回false来防止错误警报，
         *    因为错误警报会在不应该添加线程的时候添加线程。
         * 2. 如果一个任务可以成功排队，那么我们仍然需要再次检查是否应该添加一个线程(因为自上次检查以来已有的线程已经死亡)，
         *    或者池在进入这个方法后关闭。因此，我们重新检查状态，如果必要的话，如果停止，则回滚队列;如果没有，则启动一个新线程。
         * 3.如果无法对任务排队，则尝试添加新线程。如果它失败了，我们知道我们被关闭或饱和，所以拒绝任务。
         * 
         */
        int c = ctl.get();
        //小于corePoolSize
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

```
graph LR
A[execute]-->B{小于corePoolSize}
B--Yes-->C[创建新线程执行任务]
B--No-->E{任务队列已满}
E--No-->F[加入任务队列]
E--Yes-->H{小于maxPoolSize}
H--No-->I[创建新线程执行任务]
H--Yes-->J[执行拒绝策略]
```

---

先来看一下内部类 **Worker** 的定义
```
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
{
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;
    
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    
    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }

    public void run() {
        runWorker(this);
    }
}
```
添加工作线程
```
    private boolean addWorker(Runnable firstTask, boolean core) {
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
    }
```

##### 任务执行
```
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
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
    }
```
存活的线程不断通过 *getTask()* 方法从工作队列中获取新的任务执行，由于 *workQueue* 是 **BlockingQueue** 类型的，所以在队列中没有元素的时候，将导致阻塞；并且能保证并发情况下的线程安全性。
```
    private Runnable getTask() {
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
    }
```
