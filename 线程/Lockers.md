# Lockers

## lockers.h

```c++
// ...

namespace dart {

const bool kNoSafepointScope = true;
const bool kDontAssertNoSafepointScope = false;

/*
 * locker 抽象类只能在闭合代码不会触发安全点时使用。
 * 在 DEBUG 下，此类增加当前线程的 no_safepoint_scope_depth，并在锁释放时，减少它
 * 注意：请不要使用传入的互斥锁对象独立于更锁类，例如以下代码不能正确断言
  *    {
 *      MutexLocker ml(m);
 *      ....
 *      m->Exit();
 *      ....
 *      m->Enter();
 *      ...
 *    }
 * 总是使用锁对象，即使锁需要被暂时释放
 *    {
 *      MutexLocker ml(m);
 *      ....
 *      ml.Exit();
 *      ....
 *      ml.Enter();
 *      ...
 *    }
 */
class MutexLocker : public ValueObject {
 public:
  explicit MutexLocker(Mutex* mutex) : mutex_(mutex) {
    ASSERT(mutex != nullptr);
    /// 调用 Lock 来锁定
    mutex_->Lock();
  }

  virtual ~MutexLocker() {
    /// 调用 Unlock 解锁
    mutex_->Unlock();
  }

  /// 锁定、解锁
  void Lock() const {
    mutex_->Lock();
  }
  void Unlock() const {
    mutex_->Unlock();
  }

 private:
  DEBUG_ONLY(bool no_safepoint_scope_;)
  Mutex* const mutex_;

  DISALLOW_COPY_AND_ASSIGN(MutexLocker);
};

/*
 * locker 抽象类只能在闭合代码不会触发安全点时使用。
 * 在 DEBUG 下，此类增加当前线程的 no_safepoint_scope_depth，并在锁释放时，减少它
 * 注意：请不要使用传入的互斥锁对象独立于更锁类，例如以下代码不能正确断言
  *    {
 *      MonitorLocker ml(m);
 *      ....
 *      m->Exit();
 *      ....
 *      m->Enter();
 *      ...
 *    }
 * 总是使用锁对象，即使锁需要被暂时释放
 *    {
 *     MonitorLocker ml(m);
 *      ....
 *      ml.Exit();
 *      ....
 *      ml.Enter();
 *      ...
 *    }
 */
class MonitorLocker : public ValueObject {
 public:
  explicit MonitorLocker(
      Monitor* monitor, 
      bool no_safepoint_scope = true
  ) : monitor_(monitor), 
      no_safepoint_scope_(no_safepoint_scope) 
  {
    ASSERT(monitor != NULL);
    monitor_->Enter();
  }

  virtual ~MonitorLocker() {
    monitor_->Exit();
  }

  void Enter() const {
    monitor_->Enter();
  }
    
  void Exit() const {
    monitor_->Exit();
  }

  Monitor::WaitResult Wait(int64_t millis = Monitor::kNoTimeout) {
    return monitor_->Wait(millis);
  }

  // 等待安全点检查
  Monitor::WaitResult WaitWithSafepointCheck(
      Thread* thread,
      int64_t millis = Monitor::kNoTimeout
  );

  Monitor::WaitResult WaitMicros(int64_t micros = Monitor::kNoTimeout) {
    return monitor_->WaitMicros(micros);
  }

  void Notify() { monitor_->Notify(); }

  void NotifyAll() { monitor_->NotifyAll(); }

 private:
  Monitor* const monitor_;
  bool no_safepoint_scope_;

  DISALLOW_COPY_AND_ASSIGN(MonitorLocker);
};

// 在对象的作用域内离开给定的监视器。
class MonitorLeaveScope : public ValueObject {
 public:
  explicit MonitorLeaveScope(
      MonitorLocker* monitor
  ) : monitor_locker_(monitor) {
    monitor_locker_->Exit();
  }

  virtual ~MonitorLeaveScope() { monitor_locker_->Enter(); }

 private:
  MonitorLocker* const monitor_locker_;

  DISALLOW_COPY_AND_ASSIGN(MonitorLeaveScope);
};

/*
 * 安全点互斥锁
 * 此 locker 抽象类应该在能够触发安全点的闭合代码块内使用
 * 此 locker 保证 因为尝试获得相同锁 而被阻塞的其他线程，会被标记处于安全点。
 */
class SafepointMutexLocker : public ValueObject {
 public:
  explicit SafepointMutexLocker(Mutex* mutex);
  virtual ~SafepointMutexLocker() { mutex_->Unlock(); }

 private:
  Mutex* const mutex_;

  DISALLOW_COPY_AND_ASSIGN(SafepointMutexLocker);
};

/*
 * 安全点监视锁
 * 此 locker 抽象类应该在能够触发安全点的闭合代码块内使用
 * 此 locker 保证 因为尝试获得相同锁 而被阻塞的其他线程，会被标记处于安全点。
 */
class SafepointMonitorLocker : public ValueObject {
 public:
  explicit SafepointMonitorLocker(Monitor* monitor);
  virtual ~SafepointMonitorLocker() { monitor_->Exit(); }

  Monitor::WaitResult Wait(int64_t millis = Monitor::kNoTimeout);

  void NotifyAll() { monitor_->NotifyAll(); }

 private:
  Monitor* const monitor_;

  DISALLOW_COPY_AND_ASSIGN(SafepointMonitorLocker);
};

/// 读写锁
class RwLock {
 public:
  RwLock() {}
  ~RwLock() {}

 private:
  friend class ReadRwLocker;
  friend class WriteRwLocker;

  void EnterRead() {
    MonitorLocker ml(&monitor_);
    while (state_ == -1) {
      ml.Wait();
    }
    ++state_;
  }
  void LeaveRead() {
    MonitorLocker ml(&monitor_);
    ASSERT(state_ > 0);
    if (--state_ == 0) {
      ml.NotifyAll();
    }
  }

  void EnterWrite() {
    MonitorLocker ml(&monitor_);
    while (state_ != 0) {
      ml.Wait();
    }
    state_ = -1;
  }
  void LeaveWrite() {
    MonitorLocker ml(&monitor_);
    ASSERT(state_ == -1);
    state_ = 0;
    ml.NotifyAll();
  }

  Monitor monitor_;
  // [state_] > 0  : 被多个 reader 持有
  // [state_] == 0 : 空闲状态（没有 reader 或者 writer）
  // [state_] == -1: 被一个 writer 持有
  intptr_t state_ = 0;
};

/// 安全点读写锁
class SafepointRwLock {
 public:
  SafepointRwLock() {}
  ~SafepointRwLock() {}

  bool IsCurrentThreadWriter() {
    return writer_id_ == OSThread::GetCurrentThreadId();
  }

 private:
  friend class SafepointReadRwLocker;
  friend class SafepointWriteRwLocker;

  // 如果需要读取锁，返回[true]
  // 如果线程已经持有写锁，而不需要获取读锁，则返回[false]
  bool EnterRead() {
    SafepointMonitorLocker ml(&monitor_);
    if (IsCurrentThreadWriter()) {
      return false;
    }
    while (state_ == -1) {
      ml.Wait();
    }
    ++state_;
    return true;
  }
    
  void LeaveRead() {
    SafepointMonitorLocker ml(&monitor_);
    ASSERT(state_ > 0);
    if (--state_ == 0) {
      ml.NotifyAll();
    }
  }

  void EnterWrite() {
    SafepointMonitorLocker ml(&monitor_);
    if (IsCurrentThreadWriter()) {
      state_--;
      return;
    }
    while (state_ != 0) {
      ml.Wait();
    }
    writer_id_ = OSThread::GetCurrentThreadId();
    state_ = -1;
  }
  void LeaveWrite() {
    SafepointMonitorLocker ml(&monitor_);
    ASSERT(state_ < 0);
    state_++;
    if (state_ < 0) {
      return;
    }
    writer_id_ = OSThread::kInvalidThreadId;
    ml.NotifyAll();
  }

  Monitor monitor_;
  // [state_] > 0  : The lock is held by multiple readers.
  // [state_] == 0 : The lock is free (no readers/writers).
  // [state_] == -1: The lock is held by a single writer.
  intptr_t state_ = 0;
    
  ThreadId writer_id_ = OSThread::kInvalidThreadId;
};

/*
 * 锁定给定的 [RwLock] 用于读取
 *
 * 将会在 writer 持有锁时阻塞
 *
 * If this locker is long'jmped over (e.g. on a background compiler thread) the
 * lock will be freed.
 *
 * NOTE: If the locking operation blocks (due to a writer) it will not check
 * for a pending safepoint operation.
 */
class ReadRwLocker : public StackResource {
 public:
  ReadRwLocker(
      ThreadState* thread_state, 
      RwLock* rw_lock
  ) : StackResource(thread_state),
    rw_lock_(rw_lock) {
    rw_lock_->EnterRead();
  }
  ~ReadRwLocker() { 
      rw_lock_->LeaveRead();
  }

 private:
  RwLock* rw_lock_;
};

/*
 * In addition to what [ReadRwLocker] does, this implementation also gets into a
 * safepoint if necessary.
 */
class SafepointReadRwLocker : public StackResource {
 public:
  SafepointReadRwLocker(
      ThreadState* thread_state, 
      SafepointRwLock* rw_lock
  ) : StackResource(thread_state), 
    rw_lock_(rw_lock) 
  {
    ASSERT(rw_lock_ != nullptr);
    if (!rw_lock_->EnterRead()) {
      // 如果锁没有被获取，也不许被释放
      rw_lock_ = nullptr;
    }
  }
  ~SafepointReadRwLocker() {
    if (rw_lock_ != nullptr) {
      rw_lock_->LeaveRead();
    }
  }

 private:
  SafepointRwLock* rw_lock_;
};

class WriteRwLocker : public StackResource {
 public:
  WriteRwLocker(
      ThreadState* thread_state, 
      RwLock* rw_lock
  ) : StackResource(thread_state), 
    rw_lock_(rw_lock) 
  {
    rw_lock_->EnterWrite();
  }

  ~WriteRwLocker() { rw_lock_->LeaveWrite(); }

 private:
  RwLock* rw_lock_;
};


class SafepointWriteRwLocker : public StackResource {
 public:
  SafepointWriteRwLocker(
      ThreadState* thread_state, 
      SafepointRwLock* rw_lock
  ) : StackResource(thread_state), 
    rw_lock_(rw_lock) 
  {
    rw_lock_->EnterWrite();
  }

  ~SafepointWriteRwLocker() { rw_lock_->LeaveWrite(); }

 private:
  SafepointRwLock* rw_lock_;
};

}  // namespace dart

#endif  // RUNTIME_VM_LOCKERS_H_

```



## lockers.cc

```c++
// ...

namespace dart {

Monitor::WaitResult MonitorLocker::WaitWithSafepointCheck(
    Thread* thread,
    int64_t millis
) {
  ASSERT(thread == Thread::Current());
  ASSERT(thread->execution_state() == Thread::kThreadInVM);

  /// 设置阻塞状态
  thread->set_execution_state(Thread::kThreadInBlockedState);
  /// 进入安全点
  thread->EnterSafepoint();
  // 等待超时
  Monitor::WaitResult result = monitor_->Wait(millis);
  // 尝试退出安全点
  if (!thread->TryExitSafepoint()) {
    // 退出安全点失败
    monitor_->Exit();
    SafepointHandler* handler = thread->isolate_group()->safepoint_handler();
    handler->ExitSafepointUsingLock(thread);
    monitor_->Enter();
  }
    
  /// VM 线程
  thread->set_execution_state(Thread::kThreadInVM);
  return result;
}

SafepointMutexLocker::SafepointMutexLocker(Mutex* mutex) : mutex_(mutex) {
  ASSERT(mutex != NULL);
  if (!mutex_->TryLock()) {
    // We did not get the lock and could potentially block, so transition
    // accordingly.
    Thread* thread = Thread::Current();
    if (thread != NULL) {
      TransitionVMToBlocked transition(thread);
      mutex->Lock();
    } else {
      mutex->Lock();
    }
  }
}

SafepointMonitorLocker::SafepointMonitorLocker(Monitor* monitor)
    : monitor_(monitor) {
  ASSERT(monitor_ != NULL);
  if (!monitor_->TryEnter()) {
    // We did not get the lock and could potentially block, so transition
    // accordingly.
    Thread* thread = Thread::Current();
    if (thread != NULL) {
      TransitionVMToBlocked transition(thread);
      monitor_->Enter();
    } else {
      monitor_->Enter();
    }
  }
}

Monitor::WaitResult SafepointMonitorLocker::Wait(int64_t millis) {
  Thread* thread = Thread::Current();
  if (thread != NULL) {
    Monitor::WaitResult result;
    {
      TransitionVMToBlocked transition(thread);
      result = monitor_->Wait(millis);
    }
    return result;
  } else {
    return monitor_->Wait(millis);
  }
}

}  // namespace dart

```

