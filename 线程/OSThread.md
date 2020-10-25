## OSThread

## TLS

线程局部存储（Thread Local Storage，TLS）用来将数据与一个正在执行的指定线程关联起来。

进程中的全局变量与函数内定义的静态(static)变量，是各个线程都可以访问的共享变量。在一个线程修改的内存内容，对所有线程都生效。这是一个优点也是一个缺点。说它是优点，线程的数据交换变得非常快捷。说它是缺点，一个线程死掉了，其它线程也性命不保; 多个线程访问共享数据，需要昂贵的同步开销，也容易造成同步相关的BUG。

　　如果需要在一个线程内部的各个函数调用都能访问、但其它线程不能访问的变量（被称为static memory local to a thread 线程局部静态变量），就需要新的机制来实现。这就是TLS。

　　线程局部存储在不同的平台有不同的实现，可移植性不太好。幸好要实现线程局部存储并不难，最简单的办法就是建立一个全局表，通过当前线程ID去查询相应的数据，因为各个线程的ID不同，查到的数据自然也不同了。大多数平台都提供了线程局部存储的方法，无需要我们自己去实现

## os_thread.h

```c++
// ...

// On iOS, thread_local requires iOS 9+.
#if !HOST_OS_IOS
#define HAS_C11_THREAD_LOCAL 1
#endif

// Declare the OS-specific types ahead of defining the generic classes.
#if defined(HOST_OS_ANDROID)
#include "vm/os_thread_android.h"
#elif defined(HOST_OS_FUCHSIA)
#include "vm/os_thread_fuchsia.h"
#elif defined(HOST_OS_LINUX)
#include "vm/os_thread_linux.h"
#elif defined(HOST_OS_MACOS)
#include "vm/os_thread_macos.h"
#elif defined(HOST_OS_WINDOWS)
#include "vm/os_thread_win.h"
#else
#error Unknown target os.
#endif

namespace dart {

// Forward declarations.
class Log;
class Mutex;
class ThreadState;
class TimelineEventBlock;

// 互斥量
class Mutex {
 public:
  explicit Mutex(NOT_IN_PRODUCT(const char* name = "anonymous mutex"));
  ~Mutex();

  bool IsOwnedByCurrentThread() const;

 private:
  void Lock();
  bool TryLock();  // 如果锁已经锁定了返回 false
  void Unlock();

  MutexData data_;
  NOT_IN_PRODUCT(const char* name_);
#if defined(DEBUG)
  ThreadId owner_;
#endif  // defined(DEBUG)

  friend class MallocLocker;
  friend class MutexLocker;
  friend class SafepointMutexLocker;
  friend class OSThreadIterator;
  friend class TimelineEventBlockIterator;
  friend class TimelineEventRecorder;
  friend class PageSpace;
  friend void Dart_TestMutex();
  DISALLOW_COPY_AND_ASSIGN(Mutex);
};

// 线程基类，区分是否是操作系统线程
class BaseThread {
 public:
  bool is_os_thread() const { return is_os_thread_; }

 private:
  explicit BaseThread(bool is_os_thread) : is_os_thread_(is_os_thread) {}
  virtual ~BaseThread() {}

  bool is_os_thread_;

  friend class ThreadState;
  friend class OSThread;

  DISALLOW_IMPLICIT_CONSTRUCTORS(BaseThread);
};

// 对于操作系统线程的低级操作
class OSThread : public BaseThread {
 public:
  // 调用工厂构造器 CreateOSThread 来创建 OSThread。如果 Dart VM 处于关闭状态，返回 NULL
  static OSThread* CreateOSThread();
  ~OSThread();

  ThreadId id() const {
    ASSERT(id_ != OSThread::kInvalidThreadId);
    return id_;
  }

#ifdef SUPPORT_TIMELINE
  ThreadId trace_id() const {
    ASSERT(trace_id_ != OSThread::kInvalidThreadId);
    return trace_id_;
  }
#endif

  const char* name() const { return name_; }

  void SetName(const char* name);

  void set_name(const char* name) {
    ASSERT(OSThread::Current() == this);
    ASSERT(name_ == NULL);
    ASSERT(name != NULL);
    name_ = Utils::StrDup(name);
  }

  Mutex* timeline_block_lock() const { return &timeline_block_lock_; }

  // 当持有 |timeline_block_lock_| 时才能安全访问
  TimelineEventBlock* timeline_block() const { return timeline_block_; }

  // Only safe to access when holding |timeline_block_lock_|.
  void set_timeline_block(TimelineEventBlock* block) {
    timeline_block_ = block;
  }

  Log* log() const { return log_; }

  // 栈基址（栈顶）
  uword stack_base() const { return stack_base_; }
  // 栈极限（栈底）
  /// 对于栈来说 stack_base_ > CurrentPointer > stack_limit_
  uword stack_limit() const { return stack_limit_; }
  /// 由于把栈的一部分作为头部空间，因此栈极限相当于 stack_limit_ + stack_headroom_
  uword overflow_stack_limit() const { return stack_limit_ + stack_headroom_; }

  // 是否有头部空间
  bool HasStackHeadroom() { return HasStackHeadroom(stack_headroom_); }
  bool HasStackHeadroom(intptr_t headroom) {
    return GetCurrentStackPointer() > (stack_limit_ + headroom);
  }

  // May fail for the main thread on Linux if resources are low.
  static bool GetCurrentStackBounds(uword* lower, uword* upper);

  // 返回当前的 c++ 堆栈指针。等价于获取本地栈指针，但是可以和 AddressSanitizer 、SafeStack 一并工作
  // 对堆栈溢出检查足够精确，但对对齐检查不够精确。
  static uword GetCurrentStackPointer();

#if defined(USING_SAFE_STACK)
  static uword GetCurrentSafestackPointer();
  static void SetCurrentSafestackPointer(uword ssp);
#endif

  // 用于临时禁用或启用线程中断。
  void DisableThreadInterrupts();
  void EnableThreadInterrupts();
  bool ThreadInterruptsEnabled();

  // 当前正在执行的线程
  static OSThread* TryCurrent() {
    BaseThread* thread = GetCurrentTLS();
    OSThread* os_thread = NULL;
    if (thread != NULL) {
      // 如果是操作系统线程
      if (thread->is_os_thread()) {
        os_thread = reinterpret_cast<OSThread*>(thread);
      } else {
        // 否则是 VM 线程
        ThreadState* vm_thread = reinterpret_cast<ThreadState*>(thread);
        // 从 VM 线程获取操作系统线程
        os_thread = GetOSThreadFromThread(vm_thread);
      }
    }
    return os_thread;
  }

  // 当前正在执行的线程。如果没有当前线程正在执行，则创建新的 OS 线程并返回
  static OSThread* Current() {
    OSThread* os_thread = TryCurrent();
    if (os_thread == NULL) {
      os_thread = CreateAndSetUnknownThread();
    }
    return os_thread;
  }
  static void SetCurrent(OSThread* current) { SetCurrentTLS(current); }

#if defined(HAS_C11_THREAD_LOCAL)
  static ThreadState* CurrentVMThread() { return current_vm_thread_; }
#endif

  // TODO(5411455): Use flag to override default value and Validate the
  // stack size by querying OS.
  /// 计算确定的栈空间大小，即实际栈大小减去头部空间大小
  static uword GetSpecifiedStackSize() {
    intptr_t headroom =
        OSThread::CalculateHeadroom(OSThread::GetMaxStackSize());
    ASSERT(headroom < OSThread::GetMaxStackSize());
    uword stack_size = OSThread::GetMaxStackSize() - headroom;
    return stack_size;
  }
  // 获取当前 TLS
  static BaseThread* GetCurrentTLS() {
    return reinterpret_cast<BaseThread*>(OSThread::GetThreadLocal(thread_key_));
  }
  // 设置当前 TLS
  static void SetCurrentTLS(BaseThread* value);

  typedef void (*ThreadStartFunction)(uword parameter);
  typedef void (*ThreadDestructor)(void* parameter);

  // 开始一个运行给定 function 的线程，若成功则返回 0，否则返回错误码.
  static int Start(const char* name,
                   ThreadStartFunction function,
                   uword parameter);

  static ThreadLocalKey CreateThreadLocal(ThreadDestructor destructor = NULL);
  static void DeleteThreadLocal(ThreadLocalKey key);
  static uword GetThreadLocal(ThreadLocalKey key) {
    return ThreadInlineImpl::GetThreadLocal(key);
  }
  // 获取当前线程 Id
  static ThreadId GetCurrentThreadId();
  static void SetThreadLocal(ThreadLocalKey key, uword value);
  static intptr_t GetMaxStackSize();
  // 等待给定的线程结束
  static void Join(ThreadJoinId id);
  static intptr_t ThreadIdToIntPtr(ThreadId id);
  static ThreadId ThreadIdFromIntPtr(intptr_t id);
  static bool Compare(ThreadId a, ThreadId b);

  // 这个函数对于每个 OSThread 只能调用一次，并且只有当 retunred id 最终被传递给 OSThread::Join() 时才应该调用。
  static ThreadJoinId GetCurrentThreadJoinId(OSThread* thread);

  // 在 VM 初始化和关闭的时候调用
  static void Init();

  static bool IsThreadInList(ThreadId id);

  /// 禁止、允许创建 OS 线程
  static void DisableOSThreadCreation();
  static void EnableOSThreadCreation();

  // 栈缓冲区最大为 16 KB
  static const intptr_t kStackSizeBufferMax = (16 * KB * kWordSize);
  // 栈缓冲区比例 0.5
  static constexpr float kStackSizeBufferFraction = 0.5;

  static const ThreadId kInvalidThreadId;
  static const ThreadJoinId kInvalidThreadJoinId;

 private:
  // 私有构造
  OSThread();

  // These methods should not be used in a generic way and hence
  // are private, they have been added to solve the problem of
  // accessing the VM thread structure from an OSThread object
  // in the windows thread interrupter which is used for profiling.
  // We could eliminate this requirement if the windows thread interrupter
  // is implemented differently.
  ThreadState* thread() const { return thread_; }
  void set_thread(ThreadState* value) { thread_ = value; }

  static void Cleanup();
#ifdef SUPPORT_TIMELINE
  static ThreadId GetCurrentThreadTraceId();
#endif  // PRODUCT
  static OSThread* GetOSThreadFromThread(ThreadState* thread);
  static void AddThreadToListLocked(OSThread* thread);
  static void RemoveThreadFromList(OSThread* thread);
  static OSThread* CreateAndSetUnknownThread();

  // 计算头部空间大小
  static uword CalculateHeadroom(uword stack_size) {
    // 将栈大小的一般且不超过 16KB 的大小作为头部空间
    uword headroom = kStackSizeBufferFraction * stack_size;
    return (headroom > kStackSizeBufferMax) ? kStackSizeBufferMax : headroom;
  }

  // 线程 Key 和 ID
  // 在不同操作系统的有不同的实现；
  // os_thread_win.h:
  // typedef DWORD ThreadLocalKey;
  // typedef DWORD ThreadId;
  // typedef HANDLE ThreadJoinId;
  static ThreadLocalKey thread_key_;
  const ThreadId id_;

#ifdef SUPPORT_TIMELINE
  const ThreadId trace_id_;  // Used to interface with tracing tools.
#endif
  char* name_;  // A name for this thread.

  mutable Mutex timeline_block_lock_;
  TimelineEventBlock* timeline_block_;

  /// 所有的线程都在线程表（链表）中注册
  // 下一个线程指针
  OSThread* thread_list_next_;

  RelaxedAtomic<uintptr_t> thread_interrupt_disabled_;
  Log* log_;
  uword stack_base_;
  uword stack_limit_;
  uword stack_headroom_;
  ThreadState* thread_;
  
  // 持有此 OS 线程的 ThreadPool::Worker
  // 如果 OS 线程还没有被线程池开始，那么此域时 nullptr。
  void* owning_thread_pool_worker_ = nullptr;

  // thread_list_lock_ cannot have a static lifetime because the order in which
  // destructors run is undefined. At the moment this lock cannot be deleted
  // either since otherwise, if a thread only begins to run after we have
  // started to run TLS destructors for a call to exit(), there will be a race
  // on its deletion in CreateOSThread().
  static Mutex* thread_list_lock_;
  /// 线程链表表头
  static OSThread* thread_list_head_;
  static bool creation_enabled_;

#if defined(HAS_C11_THREAD_LOCAL)
  static thread_local ThreadState* current_vm_thread_;
#endif

  friend class IsolateGroup;  // to access set_thread(Thread*).
  friend class OSThreadIterator;
  friend class ThreadInterrupterWin;
  friend class ThreadInterrupterFuchsia;
  friend class ThreadPool;  // to access owning_thread_pool_worker_
};

// 线程迭代器
class OSThreadIterator : public ValueObject {
 public:
  OSThreadIterator();
  ~OSThreadIterator();

  // Returns false when there are no more threads left.
  bool HasNext() const;

  // Returns the current thread and moves forward.
  OSThread* Next();

 private:
  OSThread* next_;
};

// 监视器
class Monitor {
 public:
  enum WaitResult { kNotified, kTimedOut };

  static const int64_t kNoTimeout = 0;

  Monitor();
  ~Monitor();

#if defined(DEBUG)
  /// 是否被当前线程持有
  bool IsOwnedByCurrentThread() const {
    return owner_ == OSThread::GetCurrentThreadId();
  }
#else
  bool IsOwnedByCurrentThread() const {
    UNREACHABLE();
    return false;
  }
#endif

 private:
  bool TryEnter();  // 如果已经锁定，返回 false
  void Enter();
  void Exit();

  // 等待
  WaitResult Wait(int64_t millis);
  WaitResult WaitMicros(int64_t micros);

  // 通知正在等待的线程
  void Notify();
  void NotifyAll();

  MonitorData data_;  // 操作系统指定数据
#if defined(DEBUG)
  ThreadId owner_;
#endif  // defined(DEBUG)

  friend class MonitorLocker;
  friend class SafepointMonitorLocker;
  friend void Dart_TestMonitor();
  DISALLOW_COPY_AND_ASSIGN(Monitor);
};

inline bool Mutex::IsOwnedByCurrentThread() const {
#if defined(DEBUG)
  return owner_ == OSThread::GetCurrentThreadId();
#else
  UNREACHABLE();
  return false;
#endif
}

}  // namespace dart

#endif  // RUNTIME_VM_OS_THREAD_H_

```

## os_thread.cc

```c++
// ...

namespace dart {

// The single thread local key which stores all the thread local data
// for a thread.
ThreadLocalKey OSThread::thread_key_ = kUnsetThreadLocalKey; // = TLS_OUT_OF_INDEXES = ((DWORD)0xFFFFFFFF)
OSThread* OSThread::thread_list_head_ = NULL;
Mutex* OSThread::thread_list_lock_ = NULL;
bool OSThread::creation_enabled_ = false;

#if defined(HAS_C11_THREAD_LOCAL)
thread_local ThreadState* OSThread::current_vm_thread_ = NULL;
#endif

/// 操作系统线程构造函数
OSThread::OSThread()
    : BaseThread(true),  // is_os_thread = true
      id_(OSThread::GetCurrentThreadId()),
#if defined(DEBUG)
      join_id_(kInvalidThreadJoinId),
#endif
#ifdef SUPPORT_TIMELINE
      trace_id_(OSThread::GetCurrentThreadTraceId()),
#endif
      name_(NULL),
      timeline_block_lock_(),
      timeline_block_(NULL),
      thread_list_next_(NULL),
      thread_interrupt_disabled_(1),  // Thread interrupts disabled by default.
      log_(new class Log()),
      stack_base_(0),
      stack_limit_(0),
      stack_headroom_(0),
      thread_(NULL) {
  // 获取栈基址和边界，调用系统函数
  if (!GetCurrentStackBounds(&stack_limit_, &stack_base_)) {
    FATAL("Failed to retrieve stack bounds");
  }

  // 计算头部空间
  stack_headroom_ = CalculateHeadroom(stack_base_ - stack_limit_);

  ASSERT(stack_base_ != 0);
  ASSERT(stack_limit_ != 0);
  ASSERT(stack_base_ > stack_limit_);
  ASSERT(stack_base_ > GetCurrentStackPointer());
  ASSERT(stack_limit_ < GetCurrentStackPointer());
  RELEASE_ASSERT(HasStackHeadroom());
}

/// 工厂构造器
OSThread* OSThread::CreateOSThread() {
  ASSERT(thread_list_lock_ != NULL);
  
  // 锁定线程列表
  MutexLocker ml(thread_list_lock_);
  if (!creation_enabled_) {
    return NULL;
  }
  OSThread* os_thread = new OSThread();
  // 添加至线程列表
  AddThreadToListLocked(os_thread);
  return os_thread;
}

OSThread::~OSThread() {
  if (!is_os_thread()) {
    // If the embedder enters an isolate on this thread and does not exit the
    // isolate, the thread local at thread_key_, which we are destructing here,
    // will contain a dart::Thread instead of a dart::OSThread.
    FATAL("Thread exited without calling Dart_ExitIsolate");
  }
  // 从线程列表中移除
  RemoveThreadFromList(this);
  delete log_;
  log_ = NULL;
#if defined(SUPPORT_TIMELINE)
  if (Timeline::recorder() != NULL) {
    Timeline::recorder()->FinishBlock(timeline_block_);
  }
#endif
  timeline_block_ = NULL;
  free(name_);
}

/// 设置线程名称
void OSThread::SetName(const char* name) {
  MutexLocker ml(thread_list_lock_);
  // Clear the old thread name.
  if (name_ != NULL) {
    free(name_);
    name_ = NULL;
  }
  set_name(name);
}

// Disable AdressSanitizer and SafeStack transformation on this function. In
// particular, taking the address of a local gives an address on the stack
// instead of an address in the shadow memory (AddressSanitizer) or the safe
// stack (SafeStack).
NO_SANITIZE_ADDRESS
NO_SANITIZE_SAFE_STACK
DART_NOINLINE
uword OSThread::GetCurrentStackPointer() {
  uword stack_allocated_local = reinterpret_cast<uword>(&stack_allocated_local);
  return stack_allocated_local;
}

/// 启用、禁用线程中断
void OSThread::DisableThreadInterrupts() {
  ASSERT(OSThread::Current() == this);
  thread_interrupt_disabled_.fetch_add(1u);
}

void OSThread::EnableThreadInterrupts() {
  ASSERT(OSThread::Current() == this);
  uintptr_t old = thread_interrupt_disabled_.fetch_sub(1u);
  if (FLAG_profiler && (old == 1)) {
    // 唤醒 ThreadInterrupter
    ThreadInterrupter::WakeUp();
  }
  if (old == 0) {
    // We just decremented from 0, this means we've got a mismatched pair
    // of calls to EnableThreadInterrupts and DisableThreadInterrupts.
    FATAL("Invalid call to OSThread::EnableThreadInterrupts()");
  }
}

/// 是否启用线程中断
bool OSThread::ThreadInterruptsEnabled() {
  return thread_interrupt_disabled_ == 0;
}

 /// 删除线程
static void DeleteThread(void* thread) {
  MSAN_UNPOISON(&thread, sizeof(thread));
  delete reinterpret_cast<OSThread*>(thread);
}

/// 初始化操作系统线程
void OSThread::Init() {
  // 初始化全局操作系统线程锁
  if (thread_list_lock_ == NULL) {
    thread_list_lock_ = new Mutex();
  }
  ASSERT(thread_list_lock_ != NULL);

  // 创建本地线程 key
  if (thread_key_ == kUnsetThreadLocalKey) {
    /// 创建本地线程 key
    // CreateThreadLocal 与平台实现有关，DeleteThread 为析构函数
    thread_key_ = CreateThreadLocal(DeleteThread);
  }
  ASSERT(thread_key_ != kUnsetThreadLocalKey);

  // 允许创建操作系统线程
  EnableOSThreadCreation();

  // 创建新的操作系统线程并且设置为当前 TLS
  OSThread* os_thread = CreateOSThread();
  ASSERT(os_thread != NULL);
  OSThread::SetCurrent(os_thread);
  os_thread->set_name("Dart_Initialize");
}

void OSThread::Cleanup() {
// We cannot delete the thread local key and thread list lock,  yet.
// See the note on thread_list_lock_ in os_thread.h.
#if 0
  if (thread_list_lock_ != NULL) {
    // Delete the thread local key.
    ASSERT(thread_key_ != kUnsetThreadLocalKey);
    DeleteThreadLocal(thread_key_);
    thread_key_ = kUnsetThreadLocalKey;

    // Delete the global OSThread lock.
    ASSERT(thread_list_lock_ != NULL);
    delete thread_list_lock_;
    thread_list_lock_ = NULL;
  }
#endif
}

// 创建未命名线程
OSThread* OSThread::CreateAndSetUnknownThread() {
  ASSERT(OSThread::GetCurrentTLS() == NULL);
  OSThread* os_thread = CreateOSThread();
  if (os_thread != NULL) {
    OSThread::SetCurrent(os_thread);
    os_thread->set_name("Unknown");
  }
  return os_thread;
}

// 是否在线程表中
bool OSThread::IsThreadInList(ThreadId id) {
  if (id == OSThread::kInvalidThreadId) {
    return false;
  }
  OSThreadIterator it;
  while (it.HasNext()) {
    ASSERT(OSThread::thread_list_lock_->IsOwnedByCurrentThread());
    OSThread* t = it.Next();
    // An address test is not sufficient because the allocator may recycle
    // the address for another Thread. Test against the thread's id.
    if (t->id() == id) {
      return true;
    }
  }
  return false;
}

/// 关闭、启用创建系统线程
void OSThread::DisableOSThreadCreation() {
  MutexLocker ml(thread_list_lock_);
  creation_enabled_ = false;
}

void OSThread::EnableOSThreadCreation() {
  MutexLocker ml(thread_list_lock_);
  creation_enabled_ = true;
}

// 从给定的 ThreadState 上获取操作系统线程
OSThread* OSThread::GetOSThreadFromThread(ThreadState* thread) {
  ASSERT(thread->os_thread() != NULL);
  return thread->os_thread();
}

// 添加到线程表
void OSThread::AddThreadToListLocked(OSThread* thread) {
  ASSERT(thread != NULL);
  ASSERT(thread_list_lock_ != NULL);
  ASSERT(OSThread::thread_list_lock_->IsOwnedByCurrentThread());
  ASSERT(creation_enabled_);
  ASSERT(thread->thread_list_next_ == NULL);

  // 插入在线程链表头部
  thread->thread_list_next_ = thread_list_head_;
  thread_list_head_ = thread;
}

// 从线程表中移除
void OSThread::RemoveThreadFromList(OSThread* thread) {
  bool final_thread = false;
  {
    ASSERT(thread != NULL);
    ASSERT(thread_list_lock_ != NULL);
    MutexLocker ml(thread_list_lock_);
    OSThread* current = thread_list_head_;
    OSThread* previous = NULL;

    // Scan across list and remove |thread|.
    while (current != NULL) {
      if (current == thread) {
        // We found |thread|, remove from list.
        if (previous == NULL) {
          thread_list_head_ = thread->thread_list_next_;
        } else {
          previous->thread_list_next_ = current->thread_list_next_;
        }
        thread->thread_list_next_ = NULL;
        final_thread = !creation_enabled_ && (thread_list_head_ == NULL);
        break;
      }
      previous = current;
      current = current->thread_list_next_;
    }
  }
  // Check if this is the last thread. The last thread does a cleanup
  // which removes the thread local key and the associated mutex.
  if (final_thread) {
    Cleanup();
  }
}

void OSThread::SetCurrentTLS(BaseThread* value) {
  // Provides thread-local destructors.
  SetThreadLocal(thread_key_, reinterpret_cast<uword>(value));

#if defined(HAS_C11_THREAD_LOCAL)
  // Allows the C compiler more freedom to optimize.
  if ((value != NULL) && !value->is_os_thread()) {
    current_vm_thread_ = static_cast<Thread*>(value);
  } else {
    current_vm_thread_ = NULL;
  }
#endif
}

// 线程表迭代器，默认从线程链表头开始迭代
OSThreadIterator::OSThreadIterator() {
  ASSERT(OSThread::thread_list_lock_ != NULL);
  // 锁定线程表
  OSThread::thread_list_lock_->Lock();
  next_ = OSThread::thread_list_head_;
}

OSThreadIterator::~OSThreadIterator() {
  ASSERT(OSThread::thread_list_lock_ != NULL);
  // Unlock the thread list when done.
  OSThread::thread_list_lock_->Unlock();
}

bool OSThreadIterator::HasNext() const {
  ASSERT(OSThread::thread_list_lock_ != NULL);
  ASSERT(OSThread::thread_list_lock_->IsOwnedByCurrentThread());
  return next_ != NULL;
}

OSThread* OSThreadIterator::Next() {
  ASSERT(OSThread::thread_list_lock_ != NULL);
  ASSERT(OSThread::thread_list_lock_->IsOwnedByCurrentThread());
  OSThread* current = next_;
  next_ = next_->thread_list_next_;
  return current;
}

}  // namespace dart

```

