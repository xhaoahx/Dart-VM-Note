# ThreadPool

## thread_pool.h

```c++
// ...

namespace dart {

class MonitorLocker;

/// 线程池
class ThreadPool {
 public:
  // Task 的之内可以在线程池上运行
  class Task : public IntrusiveDListEntry<Task> {
   protected:
    Task() {}

   public:
    virtual ~Task() {}

    // 重载此函数来指定行为
    virtual void Run() = 0;

   private:
    DISALLOW_COPY_AND_ASSIGN(Task);
  };

  explicit ThreadPool(uintptr_t max_pool_size = 0);

  // 析构函数阻止新建任务，并且等待所有挂起的任务完成
  // Prevent scheduling of new tasks, wait until all pending tasks are done
  // and join worker threads.
  virtual ~ThreadPool();

  // 运行一个任务
  template <typename T, typename... Args>
  bool Run(Args&&... args) {
    return RunImpl(
        std::unique_ptr<Task>(
        	new T(std::forward<Args>(args)...)
        )
    );
  }

  // 当前线程是否在 [this] 线程池上运行
  bool CurrentThreadIsWorker();

  // 标记当前线程被阻塞（例如，运行 native 代码）。这可能暂时地增加 max thread pool size
  void MarkCurrentWorkerAsBlocked();

  // 标记当前线程未被阻塞
  void MarkCurrentWorkerAsUnBlocked();

  // 阻止创建新的任务
  void Shutdown();

  // 已经开始的 worker 数量
  uint64_t workers_started() const { return count_idle_ + count_running_; }
  // 已经停止的 worker 数量
  uint64_t workers_stopped() const { return count_dead_; }

 private:
  // 执行任务的实体
  class Worker : public IntrusiveDListEntry<Worker> {
   public:
    explicit Worker(ThreadPool* pool);

    // 开始此 Worker 的线程，必须在 SetTask() 调用之后调用
    void StartThread();

   private:
    friend class ThreadPool;

    // 新的 worker 线程的入点
    static void Main(uword args);

    // 保存线程池
    ThreadPool* pool_;
    ThreadJoinId join_id_;
      
    // 与运行任务的实际线程之间的引用域 
    OSThread* os_thread_ = nullptr;
    bool is_blocked_ = false;

    DISALLOW_COPY_AND_ASSIGN(Worker);
  };

 protected:
  // 在线程池处于 idle 时调用
  //
  // 子类能够重载此方法来进行某些操作
  // 注意：此方法运行时，线程池会被锁定
  virtual void OnEnterIdleLocked(MonitorLocker* ml) {}

  // shutdown 
  bool ShuttingDownLocked() { return shutting_down_; }

  // 是否可以执行新的任务
  bool TasksWaitingToRunLocked() { return !tasks_.IsEmpty(); }

 private:
  using TaskList = IntrusiveDList<Task>;
  using WorkerList = IntrusiveDList<Worker>;

  bool RunImpl(std::unique_ptr<Task> task);
  void WorkerLoop(Worker* worker);

  Worker* ScheduleTaskLocked(MonitorLocker* ml, std::unique_ptr<Task> task);

  /// worker 生命周期控制
  void IdleToRunningLocked(Worker* worker);
  void RunningToIdleLocked(Worker* worker);
  void IdleToDeadLocked(Worker* worker);
  void ObtainDeadWorkersLocked(WorkerList* dead_workers_to_join);
  void JoinDeadWorkersLocked(WorkerList* dead_workers_to_join);

  Monitor pool_monitor_;
  bool shutting_down_ = false;
  /// 正在运行的 worker
  uint64_t count_running_ = 0;
  /// 空闲的 worker
  uint64_t count_idle_ = 0;
  /// 已经关闭的 worker
  uint64_t count_dead_ = 0;
    
  /// worker 列表
  WorkerList running_workers_;
  WorkerList idle_workers_;
  WorkerList dead_workers_;
  
  /// 待定的任务数量
  uint64_t pending_tasks_ = 0;
  /// 任务列表
  TaskList tasks_;

  Monitor exit_monitor_;
  // 是否所有的 worker 都已经关闭
  std::atomic<bool> all_workers_dead_;

  // 线程池大小
  uintptr_t max_pool_size_ = 0;

  DISALLOW_COPY_AND_ASSIGN(ThreadPool);
};

}  // namespace dart

#endif  // RUNTIME_VM_THREAD_POOL_H_
```



## thread_pool.cc

```c++
// ...

namespace dart {

/// 当 worker 处于 Idle 状态 5000 ms 后，释放此 worker，进入 Dead 状态
DEFINE_FLAG(int,
            worker_timeout_millis,
            5000,
            "Free workers when they have been idle for this amount of time.");

/// 计算超时
static int64_t ComputeTimeout(int64_t idle_start) {
  // 计算微秒
  int64_t worker_timeout_micros =
      FLAG_worker_timeout_millis * kMicrosecondsPerMillisecond;
  if (worker_timeout_micros <= 0) {
    // No timeout.
    return 0;
  } else {
    // 计算等待时间
    int64_t waited = OS::GetCurrentMonotonicMicros() - idle_start;
    // 超时
    if (waited >= worker_timeout_micros) {
      // 我们一定在超时之前调用一次虚假的唤醒。给 worker 最后一次不顾一切的存活机会。我们是仁慈的。
      return 1;
    }
    // 返回剩余时间  
    else {
      return worker_timeout_micros - waited;
    }
  }
}

/// 构造器
ThreadPool::ThreadPool(uintptr_t max_pool_size)
    : all_workers_dead_(false), max_pool_size_(max_pool_size) {}

/// 析构时调用 Shutdown
ThreadPool::~ThreadPool() {
  Shutdown();
}

void ThreadPool::Shutdown() {
  {
    MonitorLocker ml(&pool_monitor_);

    // 阻止新建线程
    shutting_down_ = true;

    // 如果没有正在运行的 worker 或者处于空闲的 worker
    if (running_workers_.IsEmpty() && idle_workers_.IsEmpty()) {
      // 所有的 worker 都已经关闭
      all_workers_dead_ = true;
    } else {
      // 告诉 worker 将剩余工作清空，然后关闭。
      ml.NotifyAll();
    }
  }

  // 等待所有的 worker 关闭
  {
    MonitorLocker eml(&exit_monitor_);
    while (!all_workers_dead_) {
      eml.Wait();
    }
  }
  ASSERT(count_idle_ == 0);
  ASSERT(count_running_ == 0);
  ASSERT(idle_workers_.IsEmpty());
  ASSERT(running_workers_.IsEmpty());

  WorkerList dead_workers_to_join;
  {
    MonitorLocker ml(&pool_monitor_);
    /// 获取所有的关闭的 worker
    ObtainDeadWorkersLocked(&dead_workers_to_join);
  }
  // Join 所有的关闭的 worker
  JoinDeadWorkersLocked(&dead_workers_to_join);

  ASSERT(count_dead_ == 0);
  ASSERT(dead_workers_.IsEmpty());
}

/// 运行任务，正常运行给定的任务时返回 true
bool ThreadPool::RunImpl(std::unique_ptr<Task> task) {
  Worker* new_worker = nullptr;
  {
    MonitorLocker ml(&pool_monitor_);
    // 关闭状态
    if (shutting_down_) {
      return false;
    }
    // 调度任务，返回一个可用的 worker
    new_worker = ScheduleTaskLocked(&ml, std::move(task));
  }
  // 如果成功调度了 worker，创建新的线程来运行一个任务
  if (new_worker != nullptr) {
    new_worker->StartThread();
  }
  return true;
}

/// 当前线程是否是 worker 线程
bool ThreadPool::CurrentThreadIsWorker() {
  auto worker =
      static_cast<Worker*>(OSThread::Current()->owning_thread_pool_worker_);
  return worker != nullptr && worker->pool_ == this;
}

/// 标记当前 worker 被阻塞
/// e.g.
/// ```c++
/// if (mutator_thread != nullptr && mutator_thread->top_exit_frame_info() != 0) {
///    thread_pool()->MarkCurrentWorkerAsUnBlocked();
/// }
/// 即在 worker 所处的线程中，通过线程池来标记 worker 被阻塞  
/// 
void ThreadPool::MarkCurrentWorkerAsBlocked() {
  auto worker =
      static_cast<Worker*>(OSThread::Current()->owning_thread_pool_worker_);
  Worker* new_worker = nullptr;
  if (worker != nullptr) {
    MonitorLocker ml(&pool_monitor_);
    ASSERT(!worker->is_blocked_);
    
    /// 标记阻塞
    worker->is_blocked_ = true;
    // 增加线程池最大容量
    if (max_pool_size_ > 0) {
      ++max_pool_size_;
      // 此线程被阻塞因此不在被当作 worker 使用
      // 如果我们有排队的任务并且没有更多的空闲 worker，我们将会发起一个新的线程（暂时运行超出线程池输入量）来处理排队的任务
      if (idle_workers_.IsEmpty() && pending_tasks_ > 0) {
        new_worker = new Worker(this);
        idle_workers_.Append(new_worker);
        count_idle_++;
      }
    }
  }
    
  /// 新的 worker 来替代此 worker 执行任务
  if (new_worker != nullptr) {
    new_worker->StartThread();
  }
}

/// 标记当前 worker 未被阻塞
void ThreadPool::MarkCurrentWorkerAsUnBlocked() {
  auto worker =
      static_cast<Worker*>(OSThread::Current()->owning_thread_pool_worker_);
  if (worker != nullptr) {
    MonitorLocker ml(&pool_monitor_);
    if (worker->is_blocked_) {
      worker->is_blocked_ = false;
        
      /// 对称操作
      if (max_pool_size_ > 0) {
        --max_pool_size_;
        ASSERT(max_pool_size_ > 0);
      }
    }
  }
}

/// Worker 循环

/* 任务调度流程如下：
 * 1.调用 ThreadPool::Run<TaskImpl>(args) 来运行一个任务
 * 2.调用 RunImpl
 * 3.调用 ScheduleTaskLocked 来获取一个 worker
 * 4.如果没有可用的 worker，则新建并且初始化一个 worker 返回
 * 5.新的得到的 worker 调用 StartThread 来开始新的线程，并以 ThreadPool::Worker::Main 为入口函数，运行
 *   worThreadPool::WorkerLoop 循环
 * 6.worThreadPool::WorkerLoop 不断取出待定的任务并执行 Task::Run 函数（即自定义的实现）
 */

void ThreadPool::WorkerLoop(Worker* worker) {
  WorkerList dead_workers_to_join;
  
  while (true) {
    MonitorLocker ml(&pool_monitor_);

    /// 如果有任务需要进行
    if (!tasks_.IsEmpty()) {
        
      /// 从空闲状态变为运行状态
      IdleToRunningLocked(worker);
        
      
      while (!tasks_.IsEmpty()) {
        /// 取得第一个任务
        std::unique_ptr<Task> task(tasks_.RemoveFirst());
        // 减少排队数量
        pending_tasks_--;
        
        MonitorLeaveScope mls(&ml);
          
        /// 调用 Task 的 Run 实现
        task->Run();
        ASSERT(Isolate::Current() == nullptr);
        task.reset();
      }
        
      // 此时没有待定的任务，进入空闲状态
      RunningToIdleLocked(worker);
    }

    if (running_workers_.IsEmpty()) {
      ASSERT(tasks_.IsEmpty());
      OnEnterIdleLocked(&ml);
      if (!tasks_.IsEmpty()) {
        continue;
      }
    }

    // 处于关闭状态，关闭此 worker
    if (shutting_down_) {
      ObtainDeadWorkersLocked(&dead_workers_to_join);
      IdleToDeadLocked(worker);
      break;
    }

    // 进入 Sleep 状态直到有新的任务到达，否则在超时之后被关闭
    const int64_t idle_start = OS::GetCurrentMonotonicMicros();
    bool done = false;
    while (!done) {
      const auto result = ml.WaitMicros(ComputeTimeout(idle_start));

      // 跳出循环来处理任务
      if (!tasks_.IsEmpty()) break;

      // 处于关闭状态或者超时，则 done = true，关闭 worker
      if (shutting_down_ || result == Monitor::kTimedOut) {
        done = true;
        break;
      }
    }
    // 关闭 worker
    if (done) {
      ObtainDeadWorkersLocked(&dead_workers_to_join);
      IdleToDeadLocked(worker);
      break;
    }
  }

  // 在我们将 worker 转换为关闭状态之前，我们获取之前已经关闭的所有 worker，并且加入他们。
  // 因为 worker 在关闭之后，都会加入之前关闭的 worker，所以非加入的 worker 待定为 1
  JoinDeadWorkersLocked(&dead_workers_to_join);
}

// 空闲状态到运行状态
void ThreadPool::IdleToRunningLocked(Worker* worker) {
  ASSERT(idle_workers_.ContainsForDebugging(worker));
  idle_workers_.Remove(worker);
  running_workers_.Append(worker);
  count_idle_--;
  count_running_++;
}

// 运行状态到空闲状态
void ThreadPool::RunningToIdleLocked(Worker* worker) {
  ASSERT(tasks_.IsEmpty());

  ASSERT(running_workers_.ContainsForDebugging(worker));
  running_workers_.Remove(worker);
  idle_workers_.Append(worker);
  count_running_--;
  count_idle_++;
}

// 空闲状态到关闭状态
void ThreadPool::IdleToDeadLocked(Worker* worker) {
  ASSERT(tasks_.IsEmpty());

  ASSERT(idle_workers_.ContainsForDebugging(worker));
  idle_workers_.Remove(worker);
  dead_workers_.Append(worker);
  count_idle_--;
  count_dead_++;

  // Notify shutdown thread that the worker thread is about to finish.
  if (shutting_down_) {
      
    /// 所有的 worker 都关闭之后
    if (running_workers_.IsEmpty() && idle_workers_.IsEmpty()) {
      all_workers_dead_ = true;
      MonitorLocker eml(&exit_monitor_);
      // 释放锁
      eml.Notify();
    }
  }
}

/// 将 dead_workers_ 加入到  dead_workers_to_join 中
void ThreadPool::ObtainDeadWorkersLocked(WorkerList* dead_workers_to_join) {
  dead_workers_to_join->AppendList(&dead_workers_);
  ASSERT(dead_workers_.IsEmpty());
  count_dead_ = 0;
}

void ThreadPool::JoinDeadWorkersLocked(WorkerList* dead_workers_to_join) {
  auto it = dead_workers_to_join->begin();
  while (it != dead_workers_to_join->end()) {
    Worker* worker = *it;
    // 移除 it
    it = dead_workers_to_join->Erase(it);

    // 调用 操作系统 Join
    OSThread::Join(worker->join_id_);
    delete worker;
  }
  ASSERT(dead_workers_to_join->IsEmpty());
}

// 调度任务
ThreadPool::Worker* ThreadPool::ScheduleTaskLocked(MonitorLocker* ml,
                                                   std::unique_ptr<Task> task) {
  // 将任务入列
  tasks_.Append(task.release());
  pending_tasks_++;
  ASSERT(pending_tasks_ >= 1);

  // 通知存在的空闲 worker
  if (count_idle_ >= pending_tasks_) {
    ASSERT(!idle_workers_.IsEmpty());
    /// 释放锁
    ml->Notify();
    return nullptr;
  }

  // 如果我们超出了最大可运行的线程数量，则不会开始新的线程
  if (max_pool_size_ > 0 && (count_idle_ + count_running_) >= max_pool_size_) {
    if (!idle_workers_.IsEmpty()) {
      ml->Notify();
    }
    return nullptr;
  }

  // 创建新的 worker
  auto new_worker = new Worker(this);
  idle_workers_.Append(new_worker);
  count_idle_++;
  return new_worker;
}

ThreadPool::Worker::Worker(ThreadPool* pool)
    : pool_(pool), join_id_(OSThread::kInvalidThreadJoinId) {}

void ThreadPool::Worker::StartThread() {
  // 创建新的线程
  // 入口点为 hreadPool::Worker::Main
  // 这里把自己 (reinterpret_cast<uword>(this)，即 &Worker) 作为参数传入
  int result = OSThread::Start("DartWorker", &Worker::Main, reinterpret_cast<uword>(this));
  if (result != 0) {
    FATAL1("Could not start worker thread: result = %d.", result);
  }
}

void ThreadPool::Worker::Main(uword args) {
  /// 所处的线程
  OSThread* os_thread = OSThread::Current();
  ASSERT(os_thread != nullptr);

  /// 参数即为传入的 worker，可用类型转换
  Worker* worker = reinterpret_cast<Worker*>(args);
  ThreadPool* pool = worker->pool_;

  /// 相互引用，建立此 thread 和对应 worker 之间的联系
  os_thread->owning_thread_pool_worker_ = worker;
  worker->os_thread_ = os_thread;

  // 保存 join_id_ 
  worker->join_id_ = OSThread::GetCurrentThreadJoinId(os_thread);

  /// worker 的生命周期，即一直进行 WorkerLoop 循环，直到超时或者关闭
  pool->WorkerLoop(worker);

  // 清除引用
  worker->os_thread_ = nullptr;
  os_thread->owning_thread_pool_worker_ = nullptr;

  // 回调线程关闭函数
  if (Dart::thread_exit_callback() != NULL) {
    (*Dart::thread_exit_callback())();
  }
}

}  // namespace dart
```

