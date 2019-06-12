```java
/**The current thread must own this object's monitor.The thread
 * releases ownership of this monitor and waits until another thread
 * notifies threads waiting on this object's monitor to wake up
 * either through a call to the {@code notify} method or the
 * {@code notifyAll} method. The thread then waits until it can
 * re-obtain ownership of the monitor and resumes execution.*/
public final native void wait(long timeout) throws InterruptedException;
```

在 *object.c* 中给出了wait方法对应的JNI方法是 *JVM_MonitorWait*
```
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};
```
*JVM_MonitorWait* 定义在 *jvm.cpp* 中。可以看到在其内部主要通过 *ObjectSynchronizer::wait* 方法实现
```c++
JVM_ENTRY(void, JVM_MonitorWait(JNIEnv* env, jobject handle, jlong ms))
  JVMWrapper("JVM_MonitorWait");
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  JavaThreadInObjectWaitState jtiows(thread, ms != 0);
  if (JvmtiExport::should_post_monitor_wait()) {
    JvmtiExport::post_monitor_wait((JavaThread *)THREAD, (oop)obj(), ms);
  }
  ObjectSynchronizer::wait(obj, ms, CHECK);
JVM_END
```
*ObjectSynchronizer::wait* 方法定义在 *synchronizer.cpp* 中
```
//  Wait/Notify/NotifyAll
// NOTE: must use heavy weight monitor to handle wait()
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  //是否使用偏向锁，可通过 -XX:+/-UseBiasedLocking参数设置
  if (UseBiasedLocking) {
    //撤销偏向锁
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    TEVENT (wait - throw IAX) ;
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  //锁膨胀
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  monitor->wait(millis, true, THREAD);

  /* 这个伪调用是为了绕过dtrace bug 6254741。一旦这个问题解决了，我们可以取消下面这一行的注释并删除调用 */
  // DTRACE_MONITOR_PROBE(waited, monitor, obj(), THREAD);
  dtrace_waited_probe(monitor, obj, THREAD);
}

// Inflate light weight monitor to heavy weight monitor
  static ObjectMonitor* inflate(Thread * Self, oop obj);
```
而 *ObjectSynchronizer* 内部又是通过 *ObjectMonitor#wait* 来实现的
```
// Note: a subset of changes to ObjectMonitor::wait()
// will need to be replicated in complete_exit above
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
   Thread * const Self = THREAD ;
   assert(Self->is_Java_thread(), "Must be Java thread!");
   JavaThread *jt = (JavaThread *)THREAD;
   //延迟初始化
   DeferredInitialize () ;

   // Throw IMSX or IEX.
   CHECK_OWNER();

   EventJavaMonitorWait event;

   // 检查一个挂起的中断
   if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
     // post监视器等待事件。注意这是过去时态，我们已经 waiting.
     if (JvmtiExport::should_post_monitor_waited()) {
        // Note: 'false' parameter is passed here because the
        // wait was not timed out due to thread interrupt.
        JvmtiExport::post_monitor_waited(jt, this, false);
     }
     if (event.should_commit()) {
       post_monitor_wait_event(&event, 0, millis, false);
     }
     TEVENT (Wait - Throw IEX) ;
     THROW(vmSymbols::java_lang_InterruptedException());
     return ;
   }

   TEVENT (Wait) ;

   assert (Self->_Stalled == 0, "invariant") ;
   Self->_Stalled = intptr_t(this) ;
   jt->set_current_waiting_monitor(this);

   // 创建一个节点放入队列
   // Critically, after we reset() the event but prior to park(), 必须检查挂起的中断
   ObjectWaiter node(Self);
   node.TState = ObjectWaiter::TS_WAIT ;
   Self->_ParkEvent->reset() ;
   OrderAccess::fence();          // ST into Event; membar ; LD interrupted-flag

   //加入等待队列，在本例中它是一个循环的双链表，但它可以是优先队列或任何数据结构。
   //_WaitSetLock保护等待队列。通常，只有监视器的所有者才访问等待队列，除非park()由于中断超时而返回。
   //竞争是非常罕见的，所以我们使用一个简单的自旋锁，而不是一个重量级的阻塞锁
   Thread::SpinAcquire (&_WaitSetLock, "WaitSet - add") ;
   AddWaiter (&node) ;
   Thread::SpinRelease (&_WaitSetLock) ;

   if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;
   }
   intptr_t save = _recursions; // 记录旧的递归计数
   _waiters++;                  // 递增waiter数量
   _recursions = 0;             // 将递归级别设置为1
   exit (true, Self) ;                    // 退出监控
   guarantee (_owner != Self, "invariant") ;

   // 一旦在上面的exit()调用中取消了ObjectMonitor的所有权，另一个线程就可以entry() ObjectMonitor、执行notify()和exit() ObjectMonitor。
   //如果另一个线程的exit()调用选择这个线程作为继任者，而unpark()调用恰好发生在这个线程发布MONITOR_CONTENDED_EXIT事件时，
   //那么我们将使用RawMonitors运行事件处理程序的风险，并使用unpark()
   //
   //为了避免这个问题，我们重新发布了这个事件。
   //即使没有使用原始的unpark()，这也没有什么害处，因为我们是这个监视器的选定继任者。
   if (node._notified != 0 && _succ == Self) {
      node._event->unpark();
   }

   // The thread is on the WaitSet list - now park() it.
   // On MP systems it's conceivable that a brief spin before we park
   // could be profitable.
   int ret = OS_OK ;
   int WasNotified = 0 ;
   { // State transition wrappers
     OSThread* osthread = Self->osthread();
     OSThreadWaitState osts(osthread, true);
     {
       ThreadBlockInVM tbivm(jt);
       // 线程处于thread_blocking状态，oop访问不安全
       jt->set_suspend_equivalent();

       if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
           // Intentionally empty
       } else
       if (node._notified == 0) {
         if (millis <= 0) {
            Self->_ParkEvent->park () ;
         } else {
            ret = Self->_ParkEvent->park (millis) ;
         }
       }

       // were we externally suspended while we were waiting?
       if (ExitSuspendEquivalent (jt)) {
          // TODO-FIXME: add -- if succ == Self then succ = null.
          jt->java_suspend_self();
       }

     } // 退出线程 safepoint: transition _thread_blocked -> _thread_in_vm


     //节点可能位于WaitSet、EntryList(或cxq)或从WaitSet过渡到EntryList。
     //看看是否需要从WaitSet中删除节点。如果线程不在等待队列中，我们使用双重检查锁定来避免获取_WaitSetLock。
     //
     //注意，在获取TState之前，我们不需要栅栏。在最坏的情况下，我们将获取以前由is线程编写的过时的TS_WAIT值。
     //(也许通过查看处理器自己的存储缓冲区甚至可以满足取操作，尽管考虑到之前的ST和这个加载之间的代码路径的长度，这是非常不可能的)。
     //如果下面的LD获取一个陈旧的TS_WAIT值，那么我们将获得锁，然后重新获取一个新的TState值。也就是说，我们在安全上失败了。

     if (node.TState == ObjectWaiter::TS_WAIT) {
         Thread::SpinAcquire (&_WaitSetLock, "WaitSet - unlink") ;
         if (node.TState == ObjectWaiter::TS_WAIT) {
            DequeueSpecificWaiter (&node) ;       // 从WaitSet中删除节点
            assert(node._notified == 0, "invariant");
            node.TState = ObjectWaiter::TS_RUN ;
         }
         Thread::SpinRelease (&_WaitSetLock) ;
     }

     // 线程现在要么在off-list (TS_RUN)上，要么在EntryList (TS_ENTER)上，要么在cxq (TS_CXQ)上。
     //从这个线程的角度来看，节点的TState变量是稳定的。没有其他线程将异步修改TState。
     guarantee (node.TState != ObjectWaiter::TS_WAIT, "invariant") ;
     OrderAccess::loadload() ;
     if (_succ == Self) _succ = NULL ;
     WasNotified = node._notified ;

     // 重新进入阶段——重新获取监视器。在object.wait()之后重新entry()监视器。
     //保留OBJECT_WAIT状态，直到重新entry()成功完成线程状态为thread_in_vm，并且oop访问仍然是安全的，
     //尽管对象的原始地址可能已经更改。(当然，不要将naked oops缓存到安全点之上)。

     // post monitor waited event. Note that this is past-tense, we are done waiting.
     if (JvmtiExport::should_post_monitor_waited()) {
       JvmtiExport::post_monitor_waited(jt, this, ret == OS_TIMEOUT);
     }

     if (event.should_commit()) {
       post_monitor_wait_event(&event, node._notifier_tid, millis, ret == OS_TIMEOUT);
     }
     //栅栏
     OrderAccess::fence() ;

     assert (Self->_Stalled != 0, "invariant") ;
     Self->_Stalled = 0 ;

     assert (_owner != Self, "invariant") ;
     ObjectWaiter::TStates v = node.TState ;
     if (v == ObjectWaiter::TS_RUN) {
         enter (Self) ;
     } else {
         guarantee (v == ObjectWaiter::TS_ENTER || v == ObjectWaiter::TS_CXQ, "invariant") ;
         ReenterI (Self, &node) ;
         node.wait_reenter_end(this);
     }

     //self(当前线程)重新获得了锁。生命周期——表示Self的节点不能出现在任何队列上。
     //节点即将超出作用域，但即使它是不朽的，我们也不希望与此线程关联的剩余元素留在任何列表中.
     guarantee (node.TState == ObjectWaiter::TS_RUN, "invariant") ;
     assert    (_owner == Self, "invariant") ;
     assert    (_succ != Self , "invariant") ;
   } // OSThreadWaitState()

   jt->set_current_waiting_monitor(NULL);

   guarantee (_recursions == 0, "invariant") ;
   _recursions = save;     // restore the old recursion count
   _waiters--;             // decrement the number of waiters

   // 验证一些后置条件
   assert (_owner == Self       , "invariant") ;
   assert (_succ  != Self       , "invariant") ;
   assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;

   if (SyncFlags & 32) {
      OrderAccess::fence() ;
   }

   //检查通知是否发生
   if (!WasNotified) {
     // 不，它可以是timeout或Thread.interrupt()，或者两者都检查中断事件，否则就是timeout
     if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
       TEVENT (Wait - throw IEX from epilog) ;
       THROW(vmSymbols::java_lang_InterruptedException());
     }
   }

   // 注意:虚假唤醒将被视为超时。
   //监视器通知优先于线程中断。
}
```
自旋获取锁的方法定义在 *thread.cpp* 中
```
void Thread::SpinAcquire (volatile int * adr, const char * LockName) {
  if (Atomic::cmpxchg (1, adr, 0) == 0) {
     return ;   // normal fast-path return
  }

  // Slow-path : We've encountered contention -- Spin/Yield/Block strategy.
  TEVENT (SpinAcquire - ctx) ;
  int ctr = 0 ;
  int Yields = 0 ;
  for (;;) {
     while (*adr != 0) {
        ++ctr ;
        if ((ctr & 0xFFF) == 0 || !os::is_MP()) {
           if (Yields > 5) {
             // Consider using a simple NakedSleep() instead.
             // Then SpinAcquire could be called by non-JVM threads
             Thread::current()->_ParkEvent->park(1) ;
           } else {
             os::NakedYield() ;
             ++Yields ;
           }
        } else {
           SpinPause() ;
        }
     }
     if (Atomic::cmpxchg (1, adr, 0) == 0) return ;
  }
}

void Thread::SpinRelease (volatile int * adr) {
  assert (*adr != 0, "invariant") ;
  OrderAccess::fence() ;      // guarantee at least release consistency.
  // Roach-motel semantics.
  // It's safe if subsequent LDs and STs float "up" into the critical section,
  // but prior LDs and STs within the critical section can't be allowed
  // to reorder or float past the ST that releases the lock.
  *adr = 0 ;
}
```
```
ObjectWaiter * volatile _WaitSet; // LL of threads wait()ing on the monitor
volatile int _WaitSetLock;        // protects Wait Queue - simple spinlock

inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  assert(node != NULL, "should not dequeue NULL node");
  assert(node->_prev == NULL, "node already in list");
  assert(node->_next == NULL, "node already in list");
  // put node at end of queue (circular doubly linked list)
  if (_WaitSet == NULL) {
    _WaitSet = node;
    node->_prev = node;
    node->_next = node;
  } else {
    ObjectWaiter* head = _WaitSet ;
    ObjectWaiter* tail = head->_prev;
    assert(tail->_next == head, "invariant check");
    tail->_next = node;
    head->_prev = node;
    node->_next = head;
    node->_prev = tail;
```
