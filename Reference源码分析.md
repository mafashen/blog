#### Java中的几种引用类型
Java中的引用
- 强引用 

是指创建一个对象并把这个对象赋给一个引用变量。强引用有引用变量指向时永远不会被垃圾回收，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。
- 软引用 SoftReference

如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；
如果内存空间不足了，就会回收这些对象的内存。

软引用可用来实现内存敏感的高速缓存,比如网页缓存、图片缓存等。使用软引用能防止内存泄露，增强程序的健壮性。   

SoftReference的特点是它的一个实例保存对一个Java对象的软引用， 该软引用的存在不妨碍垃圾收集线程对该Java对象的回收。
- 弱引用 WeakReference

弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。

弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被JVM回收，这个软引用就会被加入到与之关联的引用队列中。
- 虚引用 PhantomReference

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。

如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。

　　要注意的是，虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。
　　

<p>Java中的几种引用类型都继承自Reference,在其内部的静态构造快中，启动了一个ReferenceHandler线程作为守护线程，并且优先级设置为最大优先级，那么ReferenceHandler主要是做什么的呢？</p>

```java
static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();

        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean waitForReferenceProcessing()
                throws InterruptedException
            {
                return Reference.waitForReferenceProcessing();
            }
        });
    }
```
```java
private static class ReferenceHandler extends Thread {
    private static void ensureClassInitialized(Class<?> clazz) {
            try {
                Class.forName(clazz.getName(), true, clazz.getClassLoader());
            } catch (ClassNotFoundException e) {
                throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
            }
        }

        static {
            //预加载并初始化clean类，这样，如果延迟加载/初始化它时出现内存不足，我们就不会在以后的运行循环中遇到麻烦.
            ensureClassInitialized(Cleaner.class);
            ensureClassInitialized(InterruptedException.class);
        }

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, null, name, 0, false);
        }

        public void run() {
            while (true) {
                processPendingReferences();
            }
        }
}
```
Reference继承自Thread，实现了run()方法，在run()方法中死循环调用Reference中的processPendingReferences()方法
```
private static void processPendingReferences() {
        // 只有单例引用处理线程调用waitForReferencePendingList()和getAndClearReferencePendingList().
        //这些是单独的操作，以避免与调用waitForReferenceProcessing()的其他线程竞争.
        waitForReferencePendingList();
        Reference<Object> pendingList;
        synchronized (processPendingLock) {
            pendingList = getAndClearReferencePendingList();
            processPendingActive = true;
        }
        while (pendingList != null) {
            Reference<Object> ref = pendingList;
            pendingList = ref.discovered;
            ref.discovered = null;
            //如果引用的对象时Cleaner实例，直接调用其clean()方法进行清理工作
            if (ref instanceof Cleaner) {
                ((Cleaner)ref).clean();
                // Notify any waiters that progress has been made.
                // 改善了NIO的延迟.Bits waiters, which are the only important ones.
                synchronized (processPendingLock) {
                    processPendingLock.notifyAll();
                }
            } else {
                //如果设置了引用引用队列，则加入到队列中
                ReferenceQueue<? super Object> q = ref.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(ref);
            }
        }
        // Notify any waiters of completion of current round.
        synchronized (processPendingLock) {
            processPendingActive = false;
            processPendingLock.notifyAll();
        }
    }
```

#### ReferenceQueue
```
    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (lock) {
            // //获取r的queue属性，检查有没有指定引用队列或已经入队或出队
            ReferenceQueue<?> queue = r.queue;
            if ((queue == NULL) || (queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            //标记为已经入队
            r.queue = ENQUEUED;
            //如果head为null，则将这个Reference作为head
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            //唤醒调用remove的线程
            lock.notifyAll();
            return true;
        }
    }
    
    public Reference<? extends T> poll() {
        if (head == null)
            return null;
        synchronized (lock) {
            return reallyPoll();
        }
    }
```

**Reference**中定义了几个成员变量值得关注，之后关于JVM的分析中，将会看到JVM是如何使用与设置这些值的
```java
    /* 引用队列中下一个Reference对象，ReferenceQueue通过这个来维持队列的顺序
     * When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    @SuppressWarnings("rawtypes")
    volatile Reference next;

    /* 由JVM控制，表示下一个要被回收的对象
     * When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    private transient Reference<T> discovered;  /* used by VM */
    
    //引用列表中下一个要进入引用队列的对象(这个引用的对象已经被GC)，由ReferenceHandler线程负责加入队列.pending由JVM赋值
    private static Reference<Object> pending = null;
```
在Reference内部实现中，引用的对象有4种状态，分别是：
- active状态： 

Reference刚开始被构造时处于这个状态。当对象的可达性发生改变（不再可达）的某个时间后，会被更改为pending状态（前提是构造Reference对象时传入了引用队列）。 
- pending状态： 

处于这个状态时，说明引用列表即将被ReferenceHandler线程加入到引用队列的对象（前提是构造Reference对象时传入了引用队列）。 
- enqueued状态： 

这个引用的对象即将被垃圾回收时处于这个状态，此时已经被JVM加入到了引用队列（如果构造时指定的话），当从引用队列中取出时，状态随之变为inactive状态。 
- inactive状态： 

引用的对象已经被垃圾回收，一旦处于这个状态就无法改变了。

这些状态都是和JVM密切相关的。

---
#### 从JVM看引用操作
在 *processPendingReferences* 方法中，调用了两个Native方法，下面一起看一下JVM中这两个方法具体是做什么的。
```
    /*
     * Wait until the VM's pending list may be non-null.
     */
    private static native void waitForReferencePendingList();
    
    /*
     * Atomically get and clear (set to null) the VM's pending list.
     */
    private static native Reference<Object> getAndClearReferencePendingList();
```
在 *reference.c* 中定义了JNI方法入口
```c
JNIEXPORT void JNICALL
Java_java_lang_ref_Reference_waitForReferencePendingList(JNIEnv *env, jclass ignore)
{
    JVM_WaitForReferencePendingList(env);
}
```
在 *jvm.cpp* 中定义了*JVM_WaitForReferencePendingList* 这个方法的入口，可以看到在这个方法中，会在调用 *Universe::has_reference_pending_list()* 返回false的情况下阻塞
```c++
JVM_ENTRY(void, JVM_WaitForReferencePendingList(JNIEnv* env))
  JVMWrapper("JVM_WaitForReferencePendingList");
  MonitorLockerEx ml(Heap_lock);
  while (!Universe::has_reference_pending_list()) {
    ml.wait();
  }
JVM_END
```
而在 *universe.cpp* 中 *has_reference_pending_list()* 只是简单的判断其内部定义的 _reference_pending_list 字段是否为空
```
//初始值
oop Universe::_reference_pending_list                 = NULL;

bool Universe::has_reference_pending_list() {
  assert_pll_ownership();
  return _reference_pending_list != NULL;
}
```
那么 *Universe::_reference_pending_list* 的值会在什么时候变化呢？
通过 *_reference_pending_list* 关键字搜索 *universe.cpp* 文件，发现只有下面所示的两个方法有对 *_reference_pending_list* 的值进行变更
```c++
void Universe::set_reference_pending_list(oop list) {
  assert_pll_ownership();
  _reference_pending_list = list;
}

oop Universe::swap_reference_pending_list(oop list) {
  assert_pll_locked(is_locked);
  return (oop)Atomic::xchg_ptr(list, &_reference_pending_list);
}
```
而其中 *set_reference_pending_list* 方法通过全局搜索显示只有一处地方调用，并且是设置为NULL，那么对 *_reference_pending_list* 字段的操作应该就主要是通过 *swap_reference_pending_list* 方法了。

在 *referenceProcessor.cpp* 中的 *enqueue_discovered_reflist* 方法中发现有调用 *swap_reference_pending_list* 方法,从类名称中可以猜测出这个类的职责是针对引用处理的，在这里我们看到会处理所有的待回收的引用对象，并设置Reference对象的 **discovered** 字段。
```
void ReferenceProcessor::enqueue_discovered_reflist(DiscoveredList& refs_list) {
  /*
    给定一个通过“discovered”字段(java.lang.ref.reference.discovered)链接的引用列表，
    自循环它们的“next”字段，从而将它们与活动引用区分开来，然后将它们添加到挂起列表中。
    Java线程将通过发现的字段看到链接在一起的引用对象。
    我们没有尝试在引用处理器的所有地方执行写屏障更新，而是在这里执行屏障，在这里我们遍历所有链接的引用对象。
    注意，在引用处理过程中不要弄脏任何卡片，这很重要，因为这会导致G1卡表验证失败。
  */
  log_develop_trace(gc, ref)("ReferenceProcessor::enqueue_discovered_reflist list " INTPTR_FORMAT, p2i(&refs_list));

  oop obj = NULL;
  oop next_d = refs_list.head();
  // Walk down the list, self-looping the next field so that the References are not considered active.
  while (obj != next_d) {
    obj = next_d;
    //类型判断，必须是初始化Reference对象时指定的类型
    assert(obj->is_instance(), "should be an instance object");
    assert(InstanceKlass::cast(obj->klass())->is_reference_instance_klass(), "should be reference object");
    //取Java Reference对象中的discovered字段
    next_d = java_lang_ref_Reference::discovered(obj);
    log_develop_trace(gc, ref)("        obj " INTPTR_FORMAT "/next_d " INTPTR_FORMAT, p2i(obj), p2i(next_d));
    assert(java_lang_ref_Reference::next(obj) == NULL,
           "Reference not active; should not be discovered");
    // 然后进行自循环，使Ref不处于活动状态
    java_lang_ref_Reference::set_next_raw(obj, obj);
    if (next_d != obj) {
      oopDesc::bs()->write_ref_field(java_lang_ref_Reference::discovered_addr(obj), next_d);
    } else {
      // 这是最后一个对象
      //将refs_list交换到pending list，并将obj的discovered设置为我们从pending list中读取的内容
      oop old = Universe::swap_reference_pending_list(refs_list.head());
      java_lang_ref_Reference::set_discovered_raw(obj, old); // old may be NULL
      //写 Reference 对象的 discovered 字段
      oopDesc::bs()->write_ref_field(java_lang_ref_Reference::discovered_addr(obj), old);
    }
  }
}
```

```c++
// Parallel enqueue task
class RefProcEnqueueTask: public AbstractRefProcTaskExecutor::EnqueueTask {
public:
  RefProcEnqueueTask(ReferenceProcessor& ref_processor,
                     DiscoveredList      discovered_refs[],
                     int                 n_queues)
    : EnqueueTask(ref_processor, discovered_refs, n_queues)
  { }

  virtual void work(unsigned int work_id) {
    assert(work_id < (unsigned int)_ref_processor.max_num_q(), "Index out-of-bounds");
    // Simplest first cut: static partitioning.
    int index = work_id;
    // “index”上的增量必须与创建ReferenceProcessor所用的最大队列数(n_queues)相对应。这是因为所发现的引用列表分配和索引的“clever”方式
    assert(_n_queues == (int) _ref_processor.max_num_q(), "Different number not expected");
    for (int j = 0;
         j < ReferenceProcessor::number_of_subclasses_of_ref();
         j++, index += _n_queues) {
      //将引用入队
      _ref_processor.enqueue_discovered_reflist(_refs_lists[index]);
      _refs_lists[index].set_head(NULL);
      _refs_lists[index].set_length(0);
    }
  }
};
```
*_discovered_refs* 类似邻接表，数组中每个元素指向一个链表，链表中每个节点是一个需要被回收掉的对象
```
// Master array of discovered oops - referenceProcessor.hpp
DiscoveredList* _discovered_refs;
  
_discovered_refs     = NEW_C_HEAP_ARRAY(DiscoveredList,
            _max_num_q * number_of_subclasses_of_ref(), mtGC);
            
// 将不再存活的引用加入到队列中
void ReferenceProcessor::enqueue_discovered_reflists(AbstractRefProcTaskExecutor* task_executor) {
  if (_processing_is_mt && task_executor != NULL) {
    // 并行方式
    RefProcEnqueueTask tsk(*this, _discovered_refs, _max_num_q);
    task_executor->execute(tsk);
  } else {
    // 串行方式: 调用父类中的方法
    for (uint i = 0; i < _max_num_q * number_of_subclasses_of_ref(); i++) {
      enqueue_discovered_reflist(_discovered_refs[i]);
      _discovered_refs[i].set_head(NULL);
      _discovered_refs[i].set_length(0);
    }
  }
}
```
在 *concurrentMarkSweepGeneration.cpp* 中看到调用 *enqueue_discovered_references* 方法，从名称可以知道，这个文件是和CMS垃圾收集器有关的，另外其他一些GC类也有调用此方法

在 *referenceProcessor.cpp* 中可以看到对各种引用类型的处理
```
ReferenceProcessorStats ReferenceProcessor::process_discovered_references(
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor,
  GCTimer*                     gc_timer) {

  assert(!enqueuing_is_done(), "If here enqueuing should not be complete");
  // Stop treating discovered references specially.
  disable_discovery();

  // 如果discovery是并发的，那么可能有人已经修改了j.l.r.SoftReference中的静态字段的值。
  //从启用discovery到现在，使用反射或UnSafe的方式保存软引用时间戳时钟。
  //无条件地更新ReferenceProcessor中的静态字段，以便在处理发现的软引用时使用新值。

  _soft_ref_timestamp_clock = java_lang_ref_SoftReference::clock();

  ReferenceProcessorStats stats(
      total_count(_discoveredSoftRefs),
      total_count(_discoveredWeakRefs),
      total_count(_discoveredFinalRefs),
      total_count(_discoveredPhantomRefs));

  // Soft references
  {
    GCTraceTime(Debug, gc, ref) tt("SoftReference", gc_timer);
    process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();

  // Weak references
  {
    GCTraceTime(Debug, gc, ref) tt("WeakReference", gc_timer);
    process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  // Final references
  {
    GCTraceTime(Debug, gc, ref) tt("FinalReference", gc_timer);
    process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  // Phantom references
  {
    GCTraceTime(Debug, gc, ref) tt("PhantomReference", gc_timer);
    process_discovered_reflist(_discoveredPhantomRefs, NULL, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  // 弱全局JNI引用。使用上面的常规弱引用同时遍历这些参数会更有意义(语义上)，//但JDK1.2规范不是这样的。参考#4126360。
  //因此，native代码可以使用JNI弱引用来绕过虚引用，并恢复“post-mortem”对象
  {
    GCTraceTime(Debug, gc, ref) tt("JNI Weak Reference", gc_timer);
    if (task_executor != NULL) {
      task_executor->set_single_threaded_mode();
    }
    process_phaseJNI(is_alive, keep_alive, complete_gc);
  }

  log_debug(gc, ref)("Ref Counts: Soft: " SIZE_FORMAT " Weak: " SIZE_FORMAT " Final: " SIZE_FORMAT " Phantom: " SIZE_FORMAT,
                     stats.soft_count(), stats.weak_count(), stats.final_count(), stats.phantom_count());
  log_develop_trace(gc, ref)("JNI Weak Reference count: " SIZE_FORMAT, count_jni_refs());

  return stats;
}
```
从以上的分析中可以得出结论，JVM在进行垃圾回收的时候，在最终标记阶段（CMS），会对Reference进行处理，设置Reference对象上的 *discovered、next* 字段，而在将唤醒ReferenceHandler线程，并把对象加入到引用队列中

---
#### Netty中对于虚引用的应用实例
ResourceLeakDetector中定义的内部成员类继承自PhantomReference
```java
private final class DefaultResourceLeak extends PhantomReference<Object> implements ResourceLeak {

    @Override
    public boolean close() {
        if (freed.compareAndSet(false, true)) {
            synchronized (head) {
                active --;
                prev.next = next;
                next.prev = prev;
                prev = null;
                next = null;
            }
            return true;
        }
        return false;
    }
}
```
申请资源时，都会调用open方法，使用 **DefaultResourceLeak** 包装引用实体，并且调用 *reportLeak* 方法报告资源泄漏的情况
```
    public ResourceLeak open(T obj) {
        Level level = ResourceLeakDetector.level;
        if (level == Level.DISABLED) {
            return null;
        }

        if (level.ordinal() < Level.PARANOID.ordinal()) {
            if (leakCheckCnt ++ % samplingInterval == 0) {
                reportLeak(level);
                return new DefaultResourceLeak(obj);
            } else {
                return null;
            }
        } else {
            reportLeak(level);
            return new DefaultResourceLeak(obj);
        }
    }
```
而 *reportLeak* 方法的主体其实比较简单，就是不断的从引用队列中取引用，并判断是否调用过了 *close* 方法，DefaultResourceLeak中重写了 *ResourceLeak#close* 方法，使用了一个 **==AtomicBoolean==** 对象记录对 *close* 方法的调用情况，如果没有调用过，此时引用持有的对象已经被回收了，那肯定是内存泄漏了。
```
    private void reportLeak(Level level) {
        //...
        
        // Detect and report previous leaks.
        for (;;) {
            @SuppressWarnings("unchecked")
            DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
            if (ref == null) {
                break;
            }

            ref.clear();

            if (!ref.close()) {
                continue;
            }

            //...
        }
    }
```
