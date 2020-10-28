## OSThread

## TLS

线程局部存储（Thread Local Storage，TLS）用来将数据与一个正在执行的指定线程关联起来。

进程中的全局变量与函数内定义的静态(static)变量，是各个线程都可以访问的共享变量。在一个线程修改的内存内容，对所有线程都生效。这是一个优点也是一个缺点。说它是优点，线程的数据交换变得非常快捷。说它是缺点，一个线程死掉了，其它线程也性命不保; 多个线程访问共享数据，需要昂贵的同步开销，也容易造成同步相关的BUG。

　　如果需要在一个线程内部的各个函数调用都能访问、但其它线程不能访问的变量（被称为static memory local to a thread 线程局部静态变量），就需要新的机制来实现。这就是TLS。

　　线程局部存储在不同的平台有不同的实现，可移植性不太好。幸好要实现线程局部存储并不难，最简单的办法就是建立一个全局表，通过当前线程ID去查询相应的数据，因为各个线程的ID不同，查到的数据自然也不同了。大多数平台都提供了线程局部存储的方法，无需要我们自己去实现

## os_thread.h

```c++
// ...

// 在 ios 平台上，local thread 要求 ios 9+
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

  /// 是否被当前线程持有
  bool IsOwnedByCurrentThread() const;

 private:
  void Lock();
  bool TryLock();  // 如果锁已经锁定了返回 false
  void Unlock();

  MutexData data_;
  NOT_IN_PRODUCT(const char* name_);
    
/// 持有此互斥量的线程 
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

  /// 线程 id
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

  // 线程名
  const char* name() const { return name_; }

  void SetName(const char* name);

  void set_name(const char* name) {
    ASSERT(OSThread::Current() == this);
    ASSERT(name_ == NULL);
    ASSERT(name != NULL);
    name_ = Utils::StrDup(name);
  }

    
  /// 时间线互斥
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
  static void SetCurrent(OSThread* current) { 
      SetCurrentTLS(current); 
  }

/// 当前 VM 线程
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

  /// 线程运行入点和析构器
  typedef void (*ThreadStartFunction)(uword parameter);
  typedef void (*ThreadDestructor)(void* parameter);

  // 开始一个运行给定 function 的线程，若成功则返回 0，否则返回错误码.
  static int Start(const char* name,
                   ThreadStartFunction function,
                   uword parameter);

  /// 创建本地线程
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
  
  /// ThreadId 转换  
  static intptr_t ThreadIdToIntPtr(ThreadId id);
  static ThreadId ThreadIdFromIntPtr(intptr_t id);
    
  /// ThreadId 比较
  static bool Compare(ThreadId a, ThreadId b);

  // 获取到 ThreadJoinId，在调用 OsThread::Join 的时候传递
  // 这个函数对于每个 OSThread 只能调用一次，并且只有当 returned id 最终被传递给 OSThread::Join() 时才应该调用。
  static ThreadJoinId GetCurrentThreadJoinId(OSThread* thread);

  // 在 VM 初始化和关闭的时候调用
  static void Init();

  /// 是否在线程表之中
  static bool IsThreadInList(ThreadId id);

  /// 禁止、允许创建 OS 线程
  static void DisableOSThreadCreation();
  static void EnableOSThreadCreation();

  // 栈缓冲区最大为 16 KB
  static const intptr_t kStackSizeBufferMax = (16 * KB * kWordSize);
  // 栈缓冲区比例 0.5
  static constexpr float kStackSizeBufferFraction = 0.5;

  /// 初始化时使用
  // 无效 ThreadId
  static const ThreadId kInvalidThreadId;
  // 无效 ThreadJoinI
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
    
  /// 从给定的线程中获取 OS Thread
  static OSThread* GetOSThreadFromThread(ThreadState* thread);
    
  // 添加线程到列表
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
  //   typedef DWORD ThreadLocalKey;
  //   typedef DWORD ThreadId;
  //   typedef HANDLE ThreadJoinId;
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
  
  // 关联此 OS 线程的 ThreadPool::Worker
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

  /// 当前 VM 线程
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

// 监视器，不同平台有不同实现
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

  // 等待一定的时间
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

/// 是否被当前线程持有
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

/// OSThread 构造函数
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

/// 初始化 OSThread
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

  /// 在初始化的线程上调用 OSThread::Init
  // 创建新的 OSThread 并且设置为当前 TLS
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

/// 设置当前 TLS
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



## os_thread_win.h

```c++
// ...

namespace dart {

typedef DWORD ThreadLocalKey;
typedef DWORD ThreadId;
typedef HANDLE ThreadJoinId;

static const ThreadLocalKey kUnsetThreadLocalKey = TLS_OUT_OF_INDEXES;

class ThreadInlineImpl {
 private:
  ThreadInlineImpl() {}
  ~ThreadInlineImpl() {}

  /// 获取本地 TLS
  static uword GetThreadLocal(ThreadLocalKey key) {
    ASSERT(key != kUnsetThreadLocalKey);
    return reinterpret_cast<uword>(TlsGetValue(key));
  }

  friend class OSThread;
  friend unsigned int __stdcall ThreadEntry(void* data_ptr);

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(ThreadInlineImpl);
};

class MutexData {
 private:
  MutexData() {}
  ~MutexData() {}

  SRWLOCK lock_;

  friend class Mutex;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(MutexData);
};

class MonitorData {
 private:
  MonitorData() {}
  ~MonitorData() {}

  SRWLOCK lock_;
  CONDITION_VARIABLE cond_;

  friend class Monitor;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(MonitorData);
};

typedef void (*ThreadDestructor)(void* parameter);

/// TLS 整体
class ThreadLocalEntry {
 public:
  ThreadLocalEntry(
      ThreadLocalKey key, 
      ThreadDestructor destructor
  ) : key_(key), destructor_(destructor) {}

  ThreadLocalKey key() const { return key_; }

  ThreadDestructor destructor() const { return destructor_; }

 private:
  ThreadLocalKey key_;
  ThreadDestructor destructor_;

  DISALLOW_ALLOCATION();
};

template <typename T>
class MallocGrowableArray;

/// 用于保存 TLS 的数据结构
class ThreadLocalData : public AllStatic {
 public:
  static void RunDestructors();

 private:
  static void AddThreadLocal(ThreadLocalKey key, ThreadDestructor destructor);
  static void RemoveThreadLocal(ThreadLocalKey key);

  static Mutex* mutex_;
  /// 添加，移除 TLS 都在 thread_locals 列表上进行
  static MallocGrowableArray<ThreadLocalEntry>* thread_locals_;

  static void Init();
  static void Cleanup();

  friend class OS;
  friend class OSThread;
};

}  // namespace dart

#endif  // RUNTIME_VM_OS_THREAD_WIN_H_

```



## os_thread_win.cc

```c++
// ...

namespace dart {

DEFINE_FLAG(int,
            worker_thread_priority,
            kMinInt,
            "The thread priority the VM should use for new worker threads.");

// This flag is flipped by platform_win.cc when the process is exiting.
// TODO(zra): Remove once VM shuts down cleanly.
bool private_flag_windows_run_tls_destructors = true;

/// 用于开始线程的数据包
class ThreadStartData {
 public:
  ThreadStartData(const char* name,
                  OSThread::ThreadStartFunction function,
                  uword parameter)
      : name_(name), function_(function), parameter_(parameter) {}

  const char* name() const { return name_; }
  OSThread::ThreadStartFunction function() const { return function_; }
  uword parameter() const { return parameter_; }

 private:
  const char* name_;
  OSThread::ThreadStartFunction function_;
  uword parameter_;

  DISALLOW_COPY_AND_ASSIGN(ThreadStartData);
};


// 线程开始函数的实现，传递的参数是 ThreadStartData
static unsigned int __stdcall ThreadEntry(void* data_ptr) {
  if (FLAG_worker_thread_priority != kMinInt) {
    if (SetThreadPriority(GetCurrentThread(), FLAG_worker_thread_priority) ==
        0) {
      FATAL2("Setting thread priority to %d failed: GetLastError() = %d\n",
             FLAG_worker_thread_priority, GetLastError());
    }
  }

  ThreadStartData* data = reinterpret_cast<ThreadStartData*>(data_ptr);

  const char* name = data->name();
  OSThread::ThreadStartFunction function = data->function();
  uword parameter = data->parameter();
  delete data;

  // Create new OSThread object and set as TLS for new thread.
  OSThread* thread = OSThread::CreateOSThread();
  if (thread != NULL) {
    OSThread::SetCurrent(thread);
    thread->set_name(name);

    // Call the supplied thread start function handing it its parameters.
    function(parameter);
  }

  return 0;
}

/// 开始新的线程
int OSThread::Start(const char* name,
                    ThreadStartFunction function,
                    uword parameter) {
  ThreadStartData* start_data = new ThreadStartData(name, function, parameter);
  uint32_t tid;
    
  /// _beginthreadex 是 windows 平台创建线程的函数，等同于 linux 的 pthread_create
  // 其原型：
  //  _ACRTIMP uintptr_t __cdecl _beginthreadex(
  //  _In_opt_  void*                    _Security,
  //  _In_      unsigned                 _StackSize,
  //  _In_      _beginthreadex_proc_type _StartAddress,
  //  _In_opt_  void*                    _ArgList,
  //  _In_      unsigned                 _InitFlag,
  //  _Out_opt_ unsigned*                _ThrdAddr
  //  );
  // ThreadEntry 是入口函数
  uintptr_t thread = _beginthreadex(NULL, OSThread::GetMaxStackSize(),
                                    ThreadEntry, start_data, 0, &tid);
  if (thread == -1L || thread == 0) {
#ifdef DEBUG
    fprintf(stderr, "_beginthreadex error: %d (%s)\n", errno, strerror(errno));
#endif
    return errno;
  }

  // Close the handle, so we don't leak the thread object.
  CloseHandle(reinterpret_cast<HANDLE>(thread));

  return 0;
}

/// 无效 ID
const ThreadId OSThread::kInvalidThreadId = 0;
const ThreadJoinId OSThread::kInvalidThreadJoinId = NULL;

/// 创建 TLS
ThreadLocalKey OSThread::CreateThreadLocal(ThreadDestructor destructor) {
  ThreadLocalKey key = TlsAlloc();
  if (key == kUnsetThreadLocalKey) {
    FATAL1("TlsAlloc failed %d", GetLastError());
  }
  ThreadLocalData::AddThreadLocal(key, destructor);
  return key;
}

// 删除 TLS
void OSThread::DeleteThreadLocal(ThreadLocalKey key) {
  ASSERT(key != kUnsetThreadLocalKey);
  BOOL result = TlsFree(key);
  if (!result) {
    FATAL1("TlsFree failed %d", GetLastError());
  }
  ThreadLocalData::RemoveThreadLocal(key);
}

// 最大栈大小：128 * 8 KB
intptr_t OSThread::GetMaxStackSize() {
  const int kStackSize = (128 * kWordSize * KB);
  return kStackSize;
}

/// ::GetCurrentThreadId 为 Windows API，其原型为：
// WINBASEAPI
// DWORD
// WINAPI
// GetCurrentThreadId(
//     VOID
//     );
ThreadId OSThread::GetCurrentThreadId() {
  return ::GetCurrentThreadId();
}

#ifdef SUPPORT_TIMELINE
ThreadId OSThread::GetCurrentThreadTraceId() {
  return ::GetCurrentThreadId();
}
#endif  // PRODUCT

ThreadJoinId OSThread::GetCurrentThreadJoinId(OSThread* thread) {
  ASSERT(thread != NULL);
  // Make sure we're filling in the join id for the current thread.
  ThreadId id = GetCurrentThreadId();
  ASSERT(thread->id() == id);
  // Make sure the join_id_ hasn't been set, yet.
  DEBUG_ASSERT(thread->join_id_ == kInvalidThreadJoinId);
  HANDLE handle = OpenThread(SYNCHRONIZE, false, id);
  ASSERT(handle != NULL);
#if defined(DEBUG)
  thread->join_id_ = handle;
#endif
  return handle;
}

    
/// WaitForSingleObject 为 Windows API，其原型为：
// WINBASEAPI
// DWORD
// WINAPI
// WaitForSingleObject(
//     _In_ HANDLE hHandle,
//     _In_ DWORD dwMilliseconds
//     );
    
/// CloseHandle 为 Windows API，其原型为：
// WINBASEAPI
// BOOL
// WINAPI
// CloseHandle(
//     _In_ _Post_ptr_invalid_ HANDLE hObject
//     );
void OSThread::Join(ThreadJoinId id) {
  HANDLE handle = static_cast<HANDLE>(id);
  ASSERT(handle != NULL);
  // 不限制超时的等待 handle
  DWORD res = WaitForSingleObject(handle, INFINITE);
  // 关闭 handle
  CloseHandle(handle);
  ASSERT(res == WAIT_OBJECT_0);
}

intptr_t OSThread::ThreadIdToIntPtr(ThreadId id) {
  ASSERT(sizeof(id) <= sizeof(intptr_t));
  return static_cast<intptr_t>(id);
}

ThreadId OSThread::ThreadIdFromIntPtr(intptr_t id) {
  return static_cast<ThreadId>(id);
}

bool OSThread::Compare(ThreadId a, ThreadId b) {
  return a == b;
}

bool OSThread::GetCurrentStackBounds(uword* lower, uword* upper) {
// On Windows stack limits for the current thread are available in
// the thread information block (TIB). Its fields can be accessed through
// FS segment register on x86 and GS segment register on x86_64.
#ifdef _WIN64
  *upper = static_cast<uword>(__readgsqword(offsetof(NT_TIB64, StackBase)));
#else
  *upper = static_cast<uword>(__readfsdword(offsetof(NT_TIB, StackBase)));
#endif
  // Notice that we cannot use the TIB's StackLimit for the stack end, as it
  // tracks the end of the committed range. We're after the end of the reserved
  // stack area (most of which will be uncommitted, most times).
  MEMORY_BASIC_INFORMATION stack_info;
  memset(&stack_info, 0, sizeof(MEMORY_BASIC_INFORMATION));
  size_t result_size =
      VirtualQuery(&stack_info, &stack_info, sizeof(MEMORY_BASIC_INFORMATION));
  ASSERT(result_size >= sizeof(MEMORY_BASIC_INFORMATION));
  *lower = reinterpret_cast<uword>(stack_info.AllocationBase);
  ASSERT(*upper > *lower);
  // When the third last page of the reserved stack is accessed as a
  // guard page, the second last page will be committed (along with removing
  // the guard bit on the third last) _and_ a stack overflow exception
  // is raised.
  //
  // http://blogs.msdn.com/b/satyem/archive/2012/08/13/thread-s-stack-memory-management.aspx
  // explains the details.
  ASSERT((*upper - *lower) >= (4u * 0x1000));
  *lower += 4 * 0x1000;
  return true;
}

#if defined(USING_SAFE_STACK)
NO_SANITIZE_ADDRESS
NO_SANITIZE_SAFE_STACK
uword OSThread::GetCurrentSafestackPointer() {
#error "SAFE_STACK is unsupported on this platform"
  return 0;
}

NO_SANITIZE_ADDRESS
NO_SANITIZE_SAFE_STACK
void OSThread::SetCurrentSafestackPointer(uword ssp) {
#error "SAFE_STACK is unsupported on this platform"
}
#endif

void OSThread::SetThreadLocal(ThreadLocalKey key, uword value) {
  ASSERT(key != kUnsetThreadLocalKey);
  BOOL result = TlsSetValue(key, reinterpret_cast<void*>(value));
  if (!result) {
    FATAL1("TlsSetValue failed %d", GetLastError());
  }
}

    
/// MUTEX 实现，封装了对 读写锁 SRWlock 的操作
Mutex::Mutex(NOT_IN_PRODUCT(const char* name))
#if !defined(PRODUCT)
    : name_(name)
#endif
{
  InitializeSRWLock(&data_.lock_);
#if defined(DEBUG)
  // When running with assertions enabled we do track the owner.
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
}

Mutex::~Mutex() {
#if defined(DEBUG)
  // When running with assertions enabled we do track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
#endif  // defined(DEBUG)
}

void Mutex::Lock() {
  AcquireSRWLockExclusive(&data_.lock_);
#if defined(DEBUG)
  // When running with assertions enabled we do track the owner.
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
}

bool Mutex::TryLock() {
  if (TryAcquireSRWLockExclusive(&data_.lock_) != 0) {
#if defined(DEBUG)
    // When running with assertions enabled we do track the owner.
    owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
    return true;
  }
  return false;
}

void Mutex::Unlock() {
#if defined(DEBUG)
  // When running with assertions enabled we do track the owner.
  ASSERT(IsOwnedByCurrentThread());
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
  ReleaseSRWLockExclusive(&data_.lock_);
}

    
/// Monitor 实现，封装了对 读写锁 SRWlock 的操作
Monitor::Monitor() {
  InitializeSRWLock(&data_.lock_);
  InitializeConditionVariable(&data_.cond_);
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
}

Monitor::~Monitor() {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
#endif  // defined(DEBUG)
}

bool Monitor::TryEnter() {
  // Attempt to pass the semaphore but return immediately.
  if (TryAcquireSRWLockExclusive(&data_.lock_) != 0) {
#if defined(DEBUG)
    // When running with assertions enabled we do track the owner.
    ASSERT(owner_ == OSThread::kInvalidThreadId);
    owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
    return true;
  }
  return false;
}

void Monitor::Enter() {
  AcquireSRWLockExclusive(&data_.lock_);

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
}

void Monitor::Exit() {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)

  ReleaseSRWLockExclusive(&data_.lock_);
}

Monitor::WaitResult Monitor::Wait(int64_t millis) {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  ThreadId saved_owner = owner_;
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)

  Monitor::WaitResult retval = kNotified;
  if (millis == kNoTimeout) {
    SleepConditionVariableSRW(&data_.cond_, &data_.lock_, INFINITE, 0);
  } else {
    // Wait for the given period of time for a Notify or a NotifyAll
    // event.
    if (!SleepConditionVariableSRW(&data_.cond_, &data_.lock_, millis, 0)) {
      ASSERT(GetLastError() == ERROR_TIMEOUT);
      retval = kTimedOut;
    }
  }

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
  owner_ = OSThread::GetCurrentThreadId();
  ASSERT(owner_ == saved_owner);
#endif  // defined(DEBUG)
  return retval;
}

Monitor::WaitResult Monitor::WaitMicros(int64_t micros) {
  // TODO(johnmccutchan): Investigate sub-millisecond sleep times on Windows.
  int64_t millis = micros / kMicrosecondsPerMillisecond;
  if ((millis * kMicrosecondsPerMillisecond) < micros) {
    // We've been asked to sleep for a fraction of a millisecond,
    // this isn't supported on Windows. Bumps milliseconds up by one
    // so that we never return too early. We likely return late though.
    millis += 1;
  }
  return Wait(millis);
}

void Monitor::Notify() {
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  WakeConditionVariable(&data_.cond_);
}

void Monitor::NotifyAll() {
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  WakeAllConditionVariable(&data_.cond_);
}

/// 添加 TLS
void ThreadLocalData::AddThreadLocal(ThreadLocalKey key,
                                     ThreadDestructor destructor) {
  ASSERT(thread_locals_ != NULL);
  if (destructor == NULL) {
    // We only care about thread locals with destructors.
    return;
  }
  MutexLocker ml(mutex_);
#if defined(DEBUG)
  // Verify that we aren't added twice.
  for (intptr_t i = 0; i < thread_locals_->length(); i++) {
    const ThreadLocalEntry& entry = thread_locals_->At(i);
    ASSERT(entry.key() != key);
  }
#endif
  // Add to list.
  thread_locals_->Add(ThreadLocalEntry(key, destructor));
}

/// 移除 TLS
void ThreadLocalData::RemoveThreadLocal(ThreadLocalKey key) {
  ASSERT(thread_locals_ != NULL);
  MutexLocker ml(mutex_);
  intptr_t i = 0;
  for (; i < thread_locals_->length(); i++) {
    const ThreadLocalEntry& entry = thread_locals_->At(i);
    if (entry.key() == key) {
      break;
    }
  }
  if (i == thread_locals_->length()) {
    // Not found.
    return;
  }
  thread_locals_->RemoveAt(i);
}

// This function is executed on the thread that is exiting. It is invoked
// by |OnDartThreadExit| (see below for notes on TLS destructors on Windows).
void ThreadLocalData::RunDestructors() {
  // If an OS thread is created but ThreadLocalData::Init has not yet been
  // called, this method still runs. If this happens, there's nothing to clean
  // up here. See issue 33826.
  if (thread_locals_ == NULL) {
    return;
  }
  ASSERT(mutex_ != NULL);
  MutexLocker ml(mutex_);
  for (intptr_t i = 0; i < thread_locals_->length(); i++) {
    const ThreadLocalEntry& entry = thread_locals_->At(i);
    // We access the exiting thread's TLS variable here.
    void* p = reinterpret_cast<void*>(OSThread::GetThreadLocal(entry.key()));
    // We invoke the constructor here.
    entry.destructor()(p);
  }
}

Mutex* ThreadLocalData::mutex_ = NULL;
MallocGrowableArray<ThreadLocalEntry>* ThreadLocalData::thread_locals_ = NULL;

/// ThreadLocalData 初始化
void ThreadLocalData::Init() {
  mutex_ = new Mutex();
  thread_locals_ = new MallocGrowableArray<ThreadLocalEntry>();
}

void ThreadLocalData::Cleanup() {
  if (mutex_ != NULL) {
    delete mutex_;
    mutex_ = NULL;
  }
  if (thread_locals_ != NULL) {
    delete thread_locals_;
    thread_locals_ = NULL;
  }
}

}  // namespace dart
```



## os_thread_linux.h

```c++
// ...

namespace dart {

typedef pthread_key_t ThreadLocalKey;
typedef pthread_t ThreadId;
typedef pthread_t ThreadJoinId;

static const ThreadLocalKey kUnsetThreadLocalKey =  static_cast<pthread_key_t>(-1);

class ThreadInlineImpl {
 private:
  ThreadInlineImpl() {}
  ~ThreadInlineImpl() {}

  // 获取与 key 关联的地址值
  static uword GetThreadLocal(ThreadLocalKey key) {
    ASSERT(key != kUnsetThreadLocalKey);
    return reinterpret_cast<uword>(pthread_getspecific(key));
  }

  friend class OSThread;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(ThreadInlineImpl);
};

class MutexData {
 private:
  MutexData() {}
  ~MutexData() {}

  pthread_mutex_t* mutex() { return &mutex_; }

  pthread_mutex_t mutex_;

  friend class Mutex;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(MutexData);
};

class MonitorData {
 private:
  MonitorData() {}
  ~MonitorData() {}

  
  pthread_mutex_t* mutex() { return &mutex_; }
  pthread_cond_t* cond() { return &cond_; }

  // 互斥量
  pthread_mutex_t mutex_;
  // 互斥条件
  pthread_cond_t cond_;

  friend class Monitor;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(MonitorData);
};

}  // namespace dart

#endif  // RUNTIME_VM_OS_THREAD_LINUX_H_

```

## os_thread_linux.cc

```c++
// ...

namespace dart {

DEFINE_FLAG(int,
            worker_thread_priority,
            kMinInt,
            "The thread priority the VM should use for new worker threads.");

#define VALIDATE_PTHREAD_RESULT(result)                                        \
  if (result != 0) {                                                           \
    const int kBufferSize = 1024;                                              \
    char error_buf[kBufferSize];                                               \
    FATAL2("pthread error: %d (%s)", result,                                   \
           Utils::StrError(result, error_buf, kBufferSize));                   \
  }

// Variation of VALIDATE_PTHREAD_RESULT for named objects.
#if defined(PRODUCT)
#define VALIDATE_PTHREAD_RESULT_NAMED(result) VALIDATE_PTHREAD_RESULT(result)
#else
#define VALIDATE_PTHREAD_RESULT_NAMED(result)                                  \
  if (result != 0) {                                                           \
    const int kBufferSize = 1024;                                              \
    char error_buf[kBufferSize];                                               \
    FATAL3("[%s] pthread error: %d (%s)", name_, result,                       \
           Utils::StrError(result, error_buf, kBufferSize));                   \
  }
#endif

#if defined(DEBUG)
#define ASSERT_PTHREAD_SUCCESS(result) VALIDATE_PTHREAD_RESULT(result)
#else
// NOTE: This (currently) expands to a no-op.
#define ASSERT_PTHREAD_SUCCESS(result) ASSERT(result == 0)
#endif

#ifdef DEBUG
#define RETURN_ON_PTHREAD_FAILURE(result)                                      \
  if (result != 0) {                                                           \
    const int kBufferSize = 1024;                                              \
    char error_buf[kBufferSize];                                               \
    fprintf(stderr, "%s:%d: pthread error: %d (%s)\n", __FILE__, __LINE__,     \
            result, Utils::StrError(result, error_buf, kBufferSize));          \
    return result;                                                             \
  }
#else
#define RETURN_ON_PTHREAD_FAILURE(result)                                      \
  if (result != 0) return result;
#endif

///
static void ComputeTimeSpecMicros(struct timespec* ts, int64_t micros) {
  // 秒
  int64_t secs = micros / kMicrosecondsPerSecond;
  // 计算纳秒
  int64_t nanos =
      (micros - (secs * kMicrosecondsPerSecond)) * kNanosecondsPerMicrosecond;
  int result = clock_gettime(CLOCK_MONOTONIC, ts);
  ASSERT(result == 0);
  ts->tv_sec += secs;
  ts->tv_nsec += nanos;
  if (ts->tv_nsec >= kNanosecondsPerSecond) {
    ts->tv_sec += 1;
    ts->tv_nsec -= kNanosecondsPerSecond;
  }
}

/// 线程开始数据
class ThreadStartData {
 public:
  ThreadStartData(
      const char* name,
      OSThread::ThreadStartFunction function,
      uword parameter
  ) : name_(name), function_(function), parameter_(parameter) {}

  const char* name() const { return name_; }
  OSThread::ThreadStartFunction function() const { return function_; }
  uword parameter() const { return parameter_; }

 private:
  const char* name_;
  OSThread::ThreadStartFunction function_;
  uword parameter_;

  DISALLOW_COPY_AND_ASSIGN(ThreadStartData);
};

// TODO(bkonyi): remove this call once the prebuilt SDK is updated.
// Spawned threads inherit their spawner's signal mask. We sometimes spawn
// threads for running Dart code from a thread that is blocking SIGPROF.
// This function explicitly unblocks SIGPROF so the profiler continues to
// sample this thread.
static void UnblockSIGPROF() {
  sigset_t set;
  sigemptyset(&set);
  sigaddset(&set, SIGPROF);
  int r = pthread_sigmask(SIG_UNBLOCK, &set, NULL);
  USE(r);
  ASSERT(r == 0);
  ASSERT(!CHECK_IS_BLOCKING(SIGPROF));
}

 
// 线程入口函数
static void* ThreadStart(void* data_ptr) {
  if (FLAG_worker_thread_priority != kMinInt) {
    if (setpriority(PRIO_PROCESS, syscall(__NR_gettid),
                    FLAG_worker_thread_priority) == -1) {
      FATAL2("Setting thread priority to %d failed: errno = %d\n",
             FLAG_worker_thread_priority, errno);
    }
  }

  // 传入的线程开始数据
  ThreadStartData* data = reinterpret_cast<ThreadStartData*>(data_ptr);

  const char* name = data->name();
  // 获取运行函数
  OSThread::ThreadStartFunction function = data->function();
  uword parameter = data->parameter();
  delete data;

  // Set the thread name. There is 16 bytes limit on the name (including \0).
  char truncated_name[16];
  snprintf(truncated_name, ARRAY_SIZE(truncated_name), "%s", name);
  pthread_setname_np(pthread_self(), truncated_name);

  // Create new OSThread object and set as TLS for new thread.
  OSThread* thread = OSThread::CreateOSThread();
  if (thread != NULL) {
    OSThread::SetCurrent(thread);
    thread->set_name(name);
    UnblockSIGPROF();
    // Call the supplied thread start function handing it its parameters.
    function(parameter);
  }

  return NULL;
}

int OSThread::Start(const char* name,
                    ThreadStartFunction function,
                    uword parameter) {
  pthread_attr_t attr;
   
  // 初始化线程属性
  int result = pthread_attr_init(&attr);
  RETURN_ON_PTHREAD_FAILURE(result);

  // 设置栈大小
  result = pthread_attr_setstacksize(&attr, OSThread::GetMaxStackSize());
  RETURN_ON_PTHREAD_FAILURE(result);

  ThreadStartData* data = new ThreadStartData(name, function, parameter);

  pthread_t tid;
  // 创建线程
  result = pthread_create(&tid, &attr, ThreadStart, data);
  RETURN_ON_PTHREAD_FAILURE(result);

  // 设置析构器
  result = pthread_attr_destroy(&attr);
  RETURN_ON_PTHREAD_FAILURE(result);

  return 0;
}

const ThreadId OSThread::kInvalidThreadId = static_cast<ThreadId>(0);
const ThreadJoinId OSThread::kInvalidThreadJoinId =
    static_cast<ThreadJoinId>(0);

ThreadLocalKey OSThread::CreateThreadLocal(ThreadDestructor destructor) {
  pthread_key_t key = kUnsetThreadLocalKey;
  int result = pthread_key_create(&key, destructor);
  VALIDATE_PTHREAD_RESULT(result);
  ASSERT(key != kUnsetThreadLocalKey);
  return key;
}

void OSThread::DeleteThreadLocal(ThreadLocalKey key) {
  ASSERT(key != kUnsetThreadLocalKey);
  int result = pthread_key_delete(key);
  VALIDATE_PTHREAD_RESULT(result);
}

void OSThread::SetThreadLocal(ThreadLocalKey key, uword value) {
  ASSERT(key != kUnsetThreadLocalKey);
  int result = pthread_setspecific(key, reinterpret_cast<void*>(value));
  VALIDATE_PTHREAD_RESULT(result);
}

intptr_t OSThread::GetMaxStackSize() {
  const int kStackSize = (128 * kWordSize * KB);
  return kStackSize;
}

ThreadId OSThread::GetCurrentThreadId() {
  return pthread_self();
}

#ifdef SUPPORT_TIMELINE
ThreadId OSThread::GetCurrentThreadTraceId() {
  return syscall(__NR_gettid);
}
#endif  // PRODUCT

ThreadJoinId OSThread::GetCurrentThreadJoinId(OSThread* thread) {
  ASSERT(thread != NULL);
  // Make sure we're filling in the join id for the current thread.
  ASSERT(thread->id() == GetCurrentThreadId());
  // Make sure the join_id_ hasn't been set, yet.
  DEBUG_ASSERT(thread->join_id_ == kInvalidThreadJoinId);
  pthread_t id = pthread_self();
#if defined(DEBUG)
  thread->join_id_ = id;
#endif
  return id;
}

void OSThread::Join(ThreadJoinId id) {
  int result = pthread_join(id, NULL);
  ASSERT(result == 0);
}

intptr_t OSThread::ThreadIdToIntPtr(ThreadId id) {
  ASSERT(sizeof(id) == sizeof(intptr_t));
  return static_cast<intptr_t>(id);
}

ThreadId OSThread::ThreadIdFromIntPtr(intptr_t id) {
  return static_cast<ThreadId>(id);
}

bool OSThread::Compare(ThreadId a, ThreadId b) {
  return pthread_equal(a, b) != 0;
}

bool OSThread::GetCurrentStackBounds(uword* lower, uword* upper) {
  pthread_attr_t attr;
  // May fail on the main thread.
  if (pthread_getattr_np(pthread_self(), &attr) != 0) {
    return false;
  }

  void* base;
  size_t size;
  int error = pthread_attr_getstack(&attr, &base, &size);
  pthread_attr_destroy(&attr);
  if (error != 0) {
    return false;
  }

  *lower = reinterpret_cast<uword>(base);
  *upper = *lower + size;
  return true;
}

#if defined(USING_SAFE_STACK)
NO_SANITIZE_ADDRESS
NO_SANITIZE_SAFE_STACK
uword OSThread::GetCurrentSafestackPointer() {
#error "SAFE_STACK is unsupported on this platform"
  return 0;
}

NO_SANITIZE_ADDRESS
NO_SANITIZE_SAFE_STACK
void OSThread::SetCurrentSafestackPointer(uword ssp) {
#error "SAFE_STACK is unsupported on this platform"
}
#endif

Mutex::Mutex(NOT_IN_PRODUCT(const char* name))
#if !defined(PRODUCT)
    : name_(name)
#endif
{
  pthread_mutexattr_t attr;
  int result = pthread_mutexattr_init(&attr);
  VALIDATE_PTHREAD_RESULT_NAMED(result);

#if defined(DEBUG)
  result = pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
  VALIDATE_PTHREAD_RESULT_NAMED(result);
#endif  // defined(DEBUG)

  result = pthread_mutex_init(data_.mutex(), &attr);
  // Verify that creating a pthread_mutex succeeded.
  VALIDATE_PTHREAD_RESULT_NAMED(result);

  result = pthread_mutexattr_destroy(&attr);
  VALIDATE_PTHREAD_RESULT_NAMED(result);

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
}

Mutex::~Mutex() {
  int result = pthread_mutex_destroy(data_.mutex());
  // Verify that the pthread_mutex was destroyed.
  VALIDATE_PTHREAD_RESULT_NAMED(result);

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
#endif  // defined(DEBUG)
}

void Mutex::Lock() {
  int result = pthread_mutex_lock(data_.mutex());
  // Specifically check for dead lock to help debugging.
  ASSERT(result != EDEADLK);
  ASSERT_PTHREAD_SUCCESS(result);  // Verify no other errors.
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
}

bool Mutex::TryLock() {
  int result = pthread_mutex_trylock(data_.mutex());
  // Return false if the lock is busy and locking failed.
  if (result == EBUSY) {
    return false;
  }
  ASSERT_PTHREAD_SUCCESS(result);  // Verify no other errors.
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
  return true;
}

void Mutex::Unlock() {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
  int result = pthread_mutex_unlock(data_.mutex());
  // Specifically check for wrong thread unlocking to aid debugging.
  ASSERT(result != EPERM);
  ASSERT_PTHREAD_SUCCESS(result);  // Verify no other errors.
}

Monitor::Monitor() {
  pthread_mutexattr_t mutex_attr;
  int result = pthread_mutexattr_init(&mutex_attr);
  VALIDATE_PTHREAD_RESULT(result);

#if defined(DEBUG)
  result = pthread_mutexattr_settype(&mutex_attr, PTHREAD_MUTEX_ERRORCHECK);
  VALIDATE_PTHREAD_RESULT(result);
#endif  // defined(DEBUG)

  result = pthread_mutex_init(data_.mutex(), &mutex_attr);
  VALIDATE_PTHREAD_RESULT(result);

  result = pthread_mutexattr_destroy(&mutex_attr);
  VALIDATE_PTHREAD_RESULT(result);

  pthread_condattr_t cond_attr;
  result = pthread_condattr_init(&cond_attr);
  VALIDATE_PTHREAD_RESULT(result);

  result = pthread_condattr_setclock(&cond_attr, CLOCK_MONOTONIC);
  VALIDATE_PTHREAD_RESULT(result);

  result = pthread_cond_init(data_.cond(), &cond_attr);
  VALIDATE_PTHREAD_RESULT(result);

  result = pthread_condattr_destroy(&cond_attr);
  VALIDATE_PTHREAD_RESULT(result);

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)
}

Monitor::~Monitor() {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
#endif  // defined(DEBUG)

  int result = pthread_mutex_destroy(data_.mutex());
  VALIDATE_PTHREAD_RESULT(result);

  result = pthread_cond_destroy(data_.cond());
  VALIDATE_PTHREAD_RESULT(result);
}

bool Monitor::TryEnter() {
  int result = pthread_mutex_trylock(data_.mutex());
  // Return false if the lock is busy and locking failed.
  if (result == EBUSY) {
    return false;
  }
  ASSERT_PTHREAD_SUCCESS(result);  // Verify no other errors.
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
  return true;
}

void Monitor::Enter() {
  int result = pthread_mutex_lock(data_.mutex());
  VALIDATE_PTHREAD_RESULT(result);

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
  owner_ = OSThread::GetCurrentThreadId();
#endif  // defined(DEBUG)
}

void Monitor::Exit() {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)

  int result = pthread_mutex_unlock(data_.mutex());
  VALIDATE_PTHREAD_RESULT(result);
}

Monitor::WaitResult Monitor::Wait(int64_t millis) {
  Monitor::WaitResult retval = WaitMicros(millis * kMicrosecondsPerMillisecond);
  return retval;
}

Monitor::WaitResult Monitor::WaitMicros(int64_t micros) {
#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  ThreadId saved_owner = owner_;
  owner_ = OSThread::kInvalidThreadId;
#endif  // defined(DEBUG)

  Monitor::WaitResult retval = kNotified;
  if (micros == kNoTimeout) {
    // Wait forever.
    int result = pthread_cond_wait(data_.cond(), data_.mutex());
    VALIDATE_PTHREAD_RESULT(result);
  } else {
    struct timespec ts;
    ComputeTimeSpecMicros(&ts, micros);
    int result = pthread_cond_timedwait(data_.cond(), data_.mutex(), &ts);
    ASSERT((result == 0) || (result == ETIMEDOUT));
    if (result == ETIMEDOUT) {
      retval = kTimedOut;
    }
  }

#if defined(DEBUG)
  // When running with assertions enabled we track the owner.
  ASSERT(owner_ == OSThread::kInvalidThreadId);
  owner_ = OSThread::GetCurrentThreadId();
  ASSERT(owner_ == saved_owner);
#endif  // defined(DEBUG)
  return retval;
}

void Monitor::Notify() {
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  int result = pthread_cond_signal(data_.cond());
  VALIDATE_PTHREAD_RESULT(result);
}

void Monitor::NotifyAll() {
  // When running with assertions enabled we track the owner.
  ASSERT(IsOwnedByCurrentThread());
  int result = pthread_cond_broadcast(data_.cond());
  VALIDATE_PTHREAD_RESULT(result);
}

}  // namespace dart

#endif  // defined(HOST_OS_LINUX)

```

