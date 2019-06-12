#### 初始化线程
```
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
    
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        //当前线程为新建线程的父线程
        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            if (security != null) {
                g = security.getThreadGroup();
            }

            /*保持和父线程相同的线程组*/
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* 检查是否允许调用线程修改线程组参数 */
        g.checkAccess();

        /*
         * 验证可以在不违反安全约束的情况下构造这个(可能是子类)实例:子类不能覆盖安全敏感的非final方法
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }
        //线程组中未开始线程数量增加 nUnstartedThreads++
        g.addUnstarted();
        //继承父线程中的信息
        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        //从构造线程继承可继承的线程局部变量的初始值
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* 线程ID */
        tid = nextThreadID();
    }
    
    private static synchronized long nextThreadID() {
        return ++threadSeqNumber;
    }
```
从代码里可以看到，new Thread() 创建一个新的Java Thread对象，并没有做太多的事情，只是从父线程中继承了一些基础的线程属性，并没有看到哪里有创建对应系统线程的痕迹，那JVM究竟是在创建的系统线程呢？答案就是调用start方法的时候。

#### 启动线程
```
    public synchronized void start() {
        //线程已经启动过了
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
    
    private native void start0();
```
---
### 从JVM中看看线程创建时的工作

#### JVM中线程相关的数据结构
在Oracle JDK / OpenJDK上，一个Java线程背后有许多数据结构： 

     Java层面     |     HotSpot VM层面     | 操作系统层面
     ---|---|---
java.lang.Thread | JavaThread -> OSThread | native thread

进到HotSpot VM层面，Thread/JavaThread用于记录（相对）平台无关的信息，而平台相关的信息则封装到OSThread里。

Java Thread.java 中定义了 *eetop* 变量用于指向JVM中 JavaThead
```java
    private long           eetop;   //实为指向JavaThread的指针  
```
JavaThread的定义
```
class JavaThread: public Thread {
  friend class VMStructs;
 private:
  JavaThread*    _next;                          // The next thread in the Threads list
  oop            _threadObj;   
  ...
 protected:
  // OS data associated with the thread
  OSThread* _osthread;  // 特定于平台的线程信息
}
```
OSThread的定义
```
class OSThread: public CHeapObj<mtThread> {
  friend class VMStructs;
 private:
  OSThreadStartFunc _start_proc;  // Thread start routine
  void* _start_parm;              // Thread start routine parameter
  volatile ThreadState _state;    // Thread state *hint*
  volatile jint _interrupted;     // Thread.isInterrupted state
  ...
}  
```
然后平台相关的部分各自不同，以Linux为例的话是 *osThread_linux.cpp*
```
  // _pthread_id is the pthread id, which is used by library calls(e.g. pthread_kill).
  pthread_t _pthread_id;
```
从上面抽取的代码可以看出这几个数据结构的关系是（伪代码）： 
```
java.lang.Thread thread;  
JavaThread* jthread = thread->_eetop;  
OSThread* osthread = jthread->_osthread;  
pthread_t pthread_id = osthread->_pthread_id;  
```
---

#### start0
下面来看一下在Java代码中调用start()方法后，线程具体是怎么运行起来的。

在 java_lang_Thread.h 中定义了Thread中所有的JNI方法调用
```
    /*
     * Class:     java_lang_Thread
     * Method:    start0
     * Signature: ()V
     */
    JNIEXPORT void JNICALL Java_java_lang_Thread_start0
      (JNIEnv *, jobject);
```
在thread.c 中定义了Thread中定义的Native方法与C++方法间的对应关系。可以看到start0方法对应于JVM_StartThread方法
```
    static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};
```
而JVM_StartThread方法的入口定义在jvm.cpp中
```
//jthread即为对应Java Thread对象
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  // 由于排序问题，在抛出异常时不能保存Threads_lock。
  // 示例:在构造异常时，可能需要获取Heap_lock
  bool throw_illegal_thread_state = false;

  // 在Thread::start中发布jvmti事件之前，我们必须释放Threads_lock。 
  {
    // 确保在操作之前不释放c++线程和OSThread结构
    MutexLocker mu(Threads_lock);

    //由于JDK 5使用线程threadStatus来防止重新启动已经启动的线程，所以我们通常会发现JavaThread为null。
    //但是，对于一个JNI附加的线程，在正在创建的线程对象(带有它的JavaThread集)和更新线程状态之间有一个小窗口，因此我们必须对此进行检查
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      // 我们还可以检查stillborn标志，看看这个线程是否已经停止，但是出于历史原因，我们让线程在开始运行时检测它自己
      //获取Thread对象上的stackSize属性
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      // 分配c++线程结构并创建本机线程。
      //从java检索的堆栈大小是带符号的，但是构造函数采用size_t(无符号类型)，因此避免传递负值，否则会导致非常大的堆栈
      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);

      // 此时，由于内存不足，可能没有为JavaThread创建osthread。检查这种情况并在必要时抛出异常。
      //最后，我们可能想要改变这一点，以便只有在线程创建成功时才获取锁——然后我们还可以进行检查，并在JavaThread构造函数中抛出异常
      if (native_thread->osthread() != NULL) {
        // 注意:当前线程不在“prepare”中使用
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread != NULL, "Starting null thread?");

  if (native_thread->osthread() == NULL) {
    // No one should hold a reference to the 'native_thread'.
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        os::native_thread_creation_failed_msg());
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              os::native_thread_creation_failed_msg());
  }

  //调用start方法启动线程
  Thread::start(native_thread);

JVM_END
```
在JVM_Start中创建了JavaThread，在JVM中，使用JavaThread代表对Java中Thread的抽象。在JavaThread的构造函数中，创建了JVM中使用的 OSThread 系统线程。

```
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread()
#if INCLUDE_ALL_GCS
                       , _satb_mark_queue(&_satb_mark_queue_set),
                       _dirty_card_queue(&_dirty_card_queue_set)
#endif // INCLUDE_ALL_GCS
{
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  //线程入口地址
  set_entry_point(entry_point);
  // Create the native thread itself.
  // %note runtime_23
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  //创建系统进程                                                   
  os::create_thread(this, thr_type, stack_sz);
  /*
    _osthread在这里可能是空的，因为我们耗尽了内存(有太多活动的线程)。我们需要抛出OutOfMemoryError,然而我们不能做这个,
    因为调用者可能持有一个锁和锁之前必须解锁抛出的异常(抛出的异常包括创建异常对象和初始化,初始化将VM通过JavaCall然后所有锁必须解锁)。
    当我们到达这里时，这个线程还是暂停状态。线程必须由创建者显式启动!此外，还必须通过调用Threads:add显式地将线程添加到Threads列表。
    这里没有这样做的原因是必须完全初始化thread对象(请参阅JVM_Start)
  */
}
```

#### JavaThread初始化工作
主要是初始化线程属性及持有的一些资源
```
    void JavaThread::initialize() {
  // Initialize fields

  set_saved_exception_pc(NULL);
  set_threadObj(NULL);  //set The Java level thread object
  _anchor.clear();
  set_entry_point(NULL);
  set_jni_functions(jni_functions());   //包含所有jni函数的结构
  set_callee_target(NULL);
  set_vm_result(NULL);
  set_vm_result_2(NULL);
  set_vframe_array_head(NULL);  // Holds the heap of the active vframeArrays
  set_vframe_array_last(NULL);  // Holds last vFrameArray we popped
  set_deferred_locals(NULL);
  set_deopt_mark(NULL); // Holds special ResourceMark for deoptimization
  set_deopt_compiled_method(NULL);  // nmethod that is currently being deoptimized
  clear_must_deopt_id();    // id of frame that needs to be deopted once we
  // 包含在反优化期间和通过JNI_MonitorEnter/Exit分配的堆栈外监视器
  set_monitor_chunks(NULL);
  set_next(NULL);
  set_thread_state(_thread_new);//设置线程状态
  _terminated = _not_terminated;
  _privileged_stack_top = NULL;
  _array_for_gc = NULL;
  _suspend_equivalent = false;
  _in_deopt_handler = 0;
  _doing_unsafe_access = false;
  _stack_guard_state = stack_guard_unused;
#if INCLUDE_JVMCI
  _pending_monitorenter = false;
  _pending_deoptimization = -1;
  _pending_failed_speculation = NULL;
  _pending_transfer_to_interpreter = false;
  _adjusting_comp_level = false;
  _jvmci._alternate_call_target = NULL;
  assert(_jvmci._implicit_exception_pc == NULL, "must be");
  if (JVMCICounterSize > 0) {
    _jvmci_counters = NEW_C_HEAP_ARRAY(jlong, JVMCICounterSize, mtInternal);
    memset(_jvmci_counters, 0, sizeof(jlong) * JVMCICounterSize);
  } else {
    _jvmci_counters = NULL;
  }
#endif // INCLUDE_JVMCI
  _reserved_stack_activation = NULL;  // stack base not known yet
  (void)const_cast<oop&>(_exception_oop = oop(NULL));
  _exception_pc  = 0;
  _exception_handler_pc = 0;
  _is_method_handle_return = 0;
  _jvmti_thread_state= NULL;
  _should_post_on_exceptions_flag = JNI_FALSE;
  _jvmti_get_loaded_classes_closure = NULL;
  _interp_only_mode    = 0;
  _special_runtime_exit_condition = _no_async_condition;
  _pending_async_exception = NULL;
  _thread_stat = NULL;
  _thread_stat = new ThreadStatistics();
  _blocked_on_compilation = false;
  _jni_active_critical = 0;
  _pending_jni_exception_check_fn = NULL;
  _do_not_unlock_if_synchronized = false;
  _cached_monitor_info = NULL;
  _parker = Parker::Allocate(this);

#ifndef PRODUCT
  _jmp_ring_index = 0;
  for (int ji = 0; ji < jump_ring_buffer_size; ji++) {
    record_jump(NULL, NULL, NULL, 0);
  }
#endif // PRODUCT

  set_thread_profiler(NULL);
  if (FlatProfiler::is_active()) {
    // 这就是我们决定要么给每个线程它自己的剖析器，要么使用一个全局的FlatProfiler，或者直到一些被剖析线程的数量
    ThreadProfiler* pp = new ThreadProfiler();
    pp->engage();
    set_thread_profiler(pp);
  }

  // 为这个线程设置安全点状态信息
  ThreadSafepointState::create(this);

  debug_only(_java_call_counter = 0);

  // JVMTI PopFrame support
  _popframe_condition = popframe_inactive;
  _popframe_preserved_args = NULL;
  _popframe_preserved_args_size = 0;
  _frames_to_pop_failed_realloc = 0;

  pd_initialize();
}
```
#### 创建osthread
在os_linux.cpp中，可以看还是通过系统调用创建系统线程
```
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {
  assert(thread->osthread() == NULL, "caller responsible");

  // 分配 OSThread object
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }

  // 线程类型
  osthread->set_thread_type(thr_type);

  // 初始化状态为 ALLOCATED 而不是 INITIALIZED
  osthread->set_state(ALLOCATED);
  //设置javaThread中的osthread属性
  thread->set_osthread(osthread);

  // 初始化线程属性
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

  // 如果调用者没有指定堆栈大小，则计算堆栈大小
  size_t stack_size = os::Posix::get_initial_stack_size(thr_type, req_stack_size);
  // 在Linux NPTL pthread实现中，保护大小机制没有正确实现。
  // posix标准要求将保护页面的大小添加到堆栈大小中，而Linux则占用了“stacksize”中的空间。
  // 因此，我们根据保护页面的大小调整请求的stack_size，以模拟正确的行为
  stack_size = align_size_up(stack_size + os::Linux::default_guard_size(thr_type), vm_page_size());
  pthread_attr_setstacksize(&attr, stack_size);

  // 配置glibc保护页面
  pthread_attr_setguardsize(&attr, os::Linux::default_guard_size(thr_type));

  ThreadState state;

  {
    pthread_t tid;
    //系统调用
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);

    char buf[64];
    if (ret == 0) {//创建成功
      log_info(os, thread)("Thread started (pthread id: " UINTX_FORMAT ", attributes: %s). ",
        (uintx) tid, os::Posix::describe_pthread_attr(buf, sizeof(buf), &attr));
    } else {
      log_warning(os, thread)("Failed to start thread - pthread_create failed (%s) for attributes: %s.",
        os::errno_name(ret), os::Posix::describe_pthread_attr(buf, sizeof(buf), &attr));
    }

    pthread_attr_destroy(&attr);

    if (ret != 0) {
      // 创建失败，需要清除已经分配的资源
      thread->set_osthread(NULL);
      delete osthread;
      return false;
    }

    // 保存系统线程id
    osthread->set_pthread_id(tid);

    // 等待子线程初始化或中止
    {
      Monitor* sync_with_child = osthread->startThread_lock();
      MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
      while ((state = osthread->get_state()) == ALLOCATED) {
        sync_with_child->wait(Mutex::_no_safepoint_check_flag);
      }
    }
  }

  // 由于达到线程限制而中止
  if (state == ZOMBIE) {
    thread->set_osthread(NULL);
    delete osthread;
    return false;
  }

  // 线程被挂起返回(处于初始化状态)，并在调用链的较高位置启动
  assert(state == INITIALIZED, "race condition");
  return true;
}
```
#### prepare
```
void JavaThread::prepare(jobject jni_thread, ThreadPriority prio) {

  assert(Threads_lock->owner() == Thread::current(), "must have threads lock");
  // 链接Java线程对象<-> c++线程

  // 从JNI句柄(jthread)获取c++ thread对象(oop)，并将其放入一个新句柄中。
  // 然后可以使用句柄“thread_oop”将c++ thread对象传递给其他方法

  // 使用“thread_oop”句柄将新线程(JavaThread *)的Java级别线程对象(jthread)字段设置为c++线程对象

  // 将表示java_lang_Thread的oop的线程字段(JavaThread *)设置为新线程(JavaThread *)

  Handle thread_oop(Thread::current(),
                    JNIHandles::resolve_non_null(jni_thread));
  assert(InstanceKlass::cast(thread_oop->klass())->is_linked(),
    "must be initialized");
  set_threadObj(thread_oop());
  java_lang_Thread::set_thread(thread_oop(), this);

  if (prio == NoPriority) {
    prio = java_lang_Thread::priority(thread_oop());
    assert(prio != NoPriority, "A valid priority should be present");
  }

  // Push the Java priority down to the native thread; needs Threads_lock
  Thread::set_priority(this, prio);

  // 将新线程添加到Threads列表并将其设置为活动状态。我们必须持有线程锁，以便调用Threads::add。
  // 在将线程添加到Threads列表之前，不要阻塞线程，这一点非常重要，因为如果发生了GC，那么GC将不会访问java_thread oop。
  Threads::add(this);
}

//将C++线程的地址设置到Java Thread 的 eetop 字段
void java_lang_Thread::set_thread(oop java_thread, JavaThread* thread) {
  java_thread->address_field_put(_eetop_offset, (address)thread);
}
```

#### 线程入口
##### OsThread线程入口
os_linux.cpp 中 定义的OsThread的线程入口
```
// 所有新创建的线程的线程启动例程
static void *thread_native_entry(Thread *thread) {
  //尝试随机化 hot 堆栈帧的缓存行。当相同堆栈的线程跟踪彼此的高速缓存行时，这将有所帮助。
  //线程可以来自同一个JVM实例，也可以来自不同的JVM实例。这种优势对于使用超线程技术的处理器尤其明显
  static int counter = 0;
  int pid = os::current_process_id();
  alloca(((pid ^ counter++) & 7) * 128);

  thread->initialize_thread_current();

  OSThread* osthread = thread->osthread();
  Monitor* sync = osthread->startThread_lock();

  osthread->set_thread_id(os::current_thread_id());

  log_info(os, thread)("Thread is alive (tid: " UINTX_FORMAT ", pthread id: " UINTX_FORMAT ").",
    os::current_thread_id(), (uintx) pthread_self());

  if (UseNUMA) {
    int lgrp_id = os::numa_get_group_id();
    if (lgrp_id != -1) {
      thread->set_lgrp_id(lgrp_id);
    }
  }
  // 初始化此线程的信号掩码
  os::Linux::hotspot_sigmask(thread);

  // 初始化浮点控制寄存器 x86 set_fpu_control_word(0x27f);
  os::Linux::init_thread_fpu_state();

  // handshaking with parent thread
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);

    // 通知父线程
    osthread->set_state(INITIALIZED);
    sync->notify_all();

    // 阻塞直到调用了 os::start_thread()
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }

  // call one more level start routine
  thread->run();

  log_info(os, thread)("Thread finished (tid: " UINTX_FORMAT ", pthread id: " UINTX_FORMAT ").",
    os::current_thread_id(), (uintx) pthread_self());

  // 如果一个线程没有将自己删除(“delete this”)作为其终止序列的一部分，
  // 那么我们必须确保在实际终止之前清除线程本地存储。任何线程都不应该因其终止而被异步删除。
  if (Thread::current_or_null_safe() != NULL) {
    assert(Thread::current_or_null_safe() == thread, "current thread is wrong");
    thread->clear_thread_current();
  }

  return 0;
}
```
##### thread->run
JavaThread重载了父类Thread的run方法，所以OsThread中thread_native_entry调用的thread->run()将进入这里
```
void JavaThread::run() {
  // 初始化线程本地alloc缓冲区相关字段
  this->initialize_tlab();

  // 用于测试堆栈跟踪回退的有效性
  this->record_base_of_stack_pointer();

  // 记录实际堆栈基数和大小
  this->record_stack_base_and_size();
  //创建栈保护页
  this->create_stack_guard_pages();
  //缓存全局变量
  this->cache_global_variables();

  // 线程现在已经足够初始化，可以由安全点代码在VM中处理。将线程状态从_thread_new更改为_thread_in_vm
  ThreadStateTransition::transition_and_fence(this, _thread_new, _thread_in_vm);

  assert(JavaThread::current() == this, "sanity check");
  assert(!Thread::current()->owns_locks(), "sanity check");

  DTRACE_THREAD_PROBE(start, this);

  // 这个操作可能会阻塞。我们在完成了对新线程的所有safepoint检查之后调用它
  this->set_active_handles(JNIHandleBlock::allocate_block());

  if (JvmtiExport::should_post_thread_life()) {
    JvmtiExport::post_thread_start(this);
  }

  EventThreadStart event;
  if (event.should_commit()) {
    event.set_thread(THREAD_TRACE_ID(this));
    event.commit();
  }

  // 我们调用另一个函数来完成其余的工作，因此我们确信从那里使用的堆栈地址将低于刚刚计算的堆栈基数
  thread_main_inner();

  // 注意，此时线程不再有效!
}


void JavaThread::thread_main_inner() {
  assert(JavaThread::current() == this, "sanity check");
  assert(this->threadObj() != NULL, "just checking");

  // 执行线程入口点，除非该线程有挂起异常或在启动前已停止。
  // 注意:由于JVM_StopThread，我们可能已经有了挂起的异常!
  if (!this->has_pending_exception() &&
        // 获取Java Thread 中定义的 stillborn 字段值
      !java_lang_Thread::is_stillborn(this->threadObj())) {
    {
      ResourceMark rm(this);
      this->set_native_thread_name(this->get_thread_name());
    }
    HandleMark hm(this);
    //_entry_point 前面新建 JavaThread 传入的方法指针，此处直接调用
    this->entry_point()(this, this);
  }

  DTRACE_THREAD_PROBE(stop, this);
  //线程退出
  this->exit(false);
  delete this;
}
```
##### JavaThread地方方法入口
jvm.cpp文件中定义了JavaThread的方法入口
```c++
    static void thread_entry(JavaThread* thread, TRAPS) {
      HandleMark hm(THREAD);
      Handle obj(THREAD, thread->threadObj());
      JavaValue result(T_VOID);
      JavaCalls::call_virtual(&result,
                              obj,
                              KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                              vmSymbols::run_method_name(),
                              vmSymbols::void_method_signature(),
                              THREAD);
}
```
通过call_virtual调用Thread中的run方法.在vmSymbols.hpp中，run_method_name通过宏定义，所以最后将调用到Java Thread 的 run 方法
```
template(run_method_name, "run")              
````

#### 线程运行
在 JVM_StartThread 中的最后，创建完JavaThread和OsThread后，紧接着调用了Thread::start()方法使线程开始工作
```
void Thread::start(Thread* thread) {
  // Start is different from resume in that its safety is guaranteed by context or
  // being called from a Java method synchronized on the Thread object.
  if (!DisableStartThread) {
    if (thread->is_Java_thread()) {
      // 在线程开始允许前将线程状态更改为 RUNNABLE .
      // 因为不能知道在启动后线程的具体状态，所以不能在线程开始运行后再设置状态
      // 可能是 MONITOR_WAIT 或者  SLEEPING 或者是其他的一些状态
      java_lang_Thread::set_thread_status(((JavaThread*)thread)->threadObj(),
                                          java_lang_Thread::RUNNABLE);
    }
    os::start_thread(thread);
  }
}
```
```os.cpp
/*
    初始化状态与挂起状态是不同的，因为线程第一次启动时的条件与恢复挂起时的条件不同。
    这些差异使得我们很难在启动线程时应用更严格的检查，而在恢复线程时则需要执行这些检查。
    但是，当start_thread作为线程的结果被调用时。在Java线程上启动，操作在Java线程对象上同步。
    因此，不可能存在启动线程的竞争，因此当我们处理线程时，线程也不会退出。
    启动Java线程的非Java线程要么必须在不可能有竞争的上下文中这样做，要么应该执行适当的锁定。
*/
void os::start_thread(Thread* thread) {
  // guard suspend/resume
  MutexLockerEx ml(thread->SR_lock(), Mutex::_no_safepoint_check_flag);
  OSThread* osthread = thread->osthread();
  //设置osthread的状态为RUNNABLE，而前面创建OsThread时，线程入口在while条件
  //while (osthread->get_state() == INITIALIZED) 
  osthread->set_state(RUNNABLE);
  pd_start_thread(thread);
}
```
pd_start_thread方法定义在 os_linux.cpp 中， 可以看到在OsThread的调用notify唤醒之前创建osthread时，thread_native_entry中阻塞直到调用os::start_thread()
```
void os::pd_start_thread(Thread* thread) {
  OSThread * osthread = thread->osthread();
  assert(osthread->get_state() != INITIALIZED, "just checking");
  Monitor* sync_with_child = osthread->startThread_lock();
  MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
  //sync->wait(Mutex::_no_safepoint_check_flag);
  sync_with_child->notify();
}
```
---
#### 其他需要注意的地方
##### JVM中Thread的继承结构
```
// Class hierarchy
// - Thread
//   - VMThread
//   - WatcherThread
//   - ConcurrentMarkSweepThread
//   - JavaThread
//     - CompilerThread
```
##### 创建osthread时的信号处理
```
void os::Linux::hotspot_sigmask(Thread* thread) {

  //Save caller's signal mask before setting VM signal mask
  sigset_t caller_sigmask;
  pthread_sigmask(SIG_BLOCK, NULL, &caller_sigmask);

  OSThread* osthread = thread->osthread();
  osthread->set_caller_sigmask(caller_sigmask);

  pthread_sigmask(SIG_UNBLOCK, os::Linux::unblocked_signals(), NULL);

  if (!ReduceSignalUsage) {
    if (thread->is_VM_thread()) {
      // Only the VM thread handles BREAK_SIGNAL ...
      pthread_sigmask(SIG_UNBLOCK, vm_signals(), NULL);
    } else {
      // ... all other threads block BREAK_SIGNAL
      pthread_sigmask(SIG_BLOCK, vm_signals(), NULL);
    }
  }
}
```
