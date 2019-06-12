在 **ExecutorService** 中定义了 *submit* 方法，该方法接收 **Callable** 类型对象，并返回 **Future** 作为结果
```
    <T> Future<T> submit(Callable<T> task);
```
其实现在 **AbstractExecutorService** 中
```java
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) 
            throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```
可以看到，就是通过 **Callable**对象生成一个 **FutureTask** 对象，其中 **FutureTask** 实现了 **RunableFuture** 接口,而 **RunableFuture** 接口又继承了 **Runable** 和 **Future** 接口
```
public class FutureTask<V> implements RunnableFuture<V>{
    //任务状态
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
在 **FutureTask** 中，重写了 *run()* 方法
```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                //正常执行结束，设置状态为Normal
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
可以看到其内部就是调用了 **Callable** 中的 *run()* 方法，并接收其返回结果，并增加对正常情况、异常情况的一些处理
```
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```
正常执行 *run()* 方法并返回，会将状态变为 **NORMAL** ，并调用 *finishCompletion()* 方法唤醒等待线程并移除等待节点
```
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
该方法就是不断的遍历等待队列，移除并唤醒关联的线程。

异常情况的处理也是类似的，只不过设置的状态为 **EXCEPTION* 
```
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
```

---
#### get
**FutureTask** 的 *run()* 方法是在线程池中被调用的，是异步的方式，要想获取其返回结果，需要调用其 *get()* 方法
```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```
该方法就是判断当前的状态是否已完成，是的话直接返回结果，否则则将当前线程添加到等待队列中并阻塞当前线程
```
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                //等待时间到达，将当前线程从等待队列中移除
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```
该方法将当前线程添加到等待队列的适当位置并阻塞当前线程。其 *timed* 参数控制是否是超时等待的方式； *nanos* 参数控制超时等待的时长。

返回结果
```
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
其中 *outcome* 就是前面在 *set()* 方法或者 *setException* 设置进去的正常返回结果或者抛出的异常信息。

---
至此，**Future** 的实现机制应该比较清晰了。
