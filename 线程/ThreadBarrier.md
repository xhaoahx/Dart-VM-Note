# ThreadBarrier

## thread_barrier.h

```c++
// ...

namespace dart {

// 线程屏障包括：
// * 固定（在构造时）数量 n 的参与进程 {T1,T2,T3,...,Tn}
// * 未知的回合数
// 要求：
// * 存在 R，这样每个参与的线程都对 Sync() 进行 R 次调用，然后是它唯一一次对 Exit() 的调用。
// 保证：
// * 对于任何两个线程 Ti 和 Tj 并且回合数 r <= R,
//   everything done by Ti before its r'th call to Sync() happens before
//   everything done by Tj after its r'th call to Sync().
// 注意：
// * 构造barrier的线程不需要参与。 
//
// Example usage with 3 threads (1 controller + 2 workers) and 3 rounds:
//
// T1:
// ThreadBarrier barrier(3);
// Dart::thread_pool()->Run(              T2:
//     new FooTask(&barrier));            fooSetup();
// Dart::thread_pool()->Run(              ...                T3:
//     new BarTask(&barrier));            ...                barSetup();
// barrier.Sync();                        barrier_->Sync();  barrier_->Sync();
// /* 两个线程都完成了 Setup */   ...                ...
// prepareWorkForTasks();                 ...                ...
// barrier.Sync();                        barrier_->Sync();  barrier_->Sync();
// /* Idle while tasks are working */     fooWork();         barWork();
// barrier.Sync();                        barrier_->Sync();  barrier_->Sync();
// collectResultsFromTasks();             barrier_->Exit();  barrier_->Exit();
// barrier.Exit();
//
// 注意调用 Sync() 是在时间顺序上 "排队"，但是却不保证 Exit 的顺序
//
class ThreadBarrier {
 public:
  explicit ThreadBarrier(
      intptr_t num_threads,
      Monitor* monitor,
      Monitor* done_monitor
  ) : num_threads_(num_threads),
      monitor_(monitor),
      // 把 remaining_ 赋值成 num_threads
      remaining_(num_threads),
      parity_(false),
      done_monitor_(done_monitor),
      done_(false) 
  {
    ASSERT(remaining_ > 0);
  }

  void Sync() {
    MonitorLocker ml(monitor_);
    ASSERT(remaining_ > 0);
      
    // 如果还有其他的任务没有调用 Sync()
    if (--remaining_ > 0) {
      // 此线程并不是最后一个到达的
      bool old_parity = parity_;
      while (parity_ == old_parity) {
        ml.Wait();
      }
    } else {
      // 最后一个到达的线程开启下一回合
      // 恢复 remaining
      remaining_ = num_threads_;
      // 修改 parity_ 的值以退出 while (parity_ == old_parity) 循环
      parity_ = !parity_;
      // 通知所有等待的线程
      ml.NotifyAll();
    }
  }

  void Exit() {
    bool last = false;
    /// 互斥修改 last
    {
      MonitorLocker ml(monitor_);
      ASSERT(remaining_ > 0);
      last = (--remaining_ == 0);
    }
    if (last) {
      // 最后一个线程设置 done_.
      MonitorLocker ml(done_monitor_);
      ASSERT(!done_);
      done_ = true;
      // 通知析构器以停止等待
      ml.Notify();
    }
  }

  ~ThreadBarrier() {
    MonitorLocker ml(done_monitor_);
    // 等待所有线程退出
    while (!done_) {
      ml.Wait();
    }
    ASSERT(remaining_ == 0);
  }

 private:
  const intptr_t num_threads_;

  Monitor* monitor_;
  intptr_t remaining_;
  bool parity_;

  Monitor* done_monitor_;  // TODO(koda): Try to optimize this away.
  bool done_;

  DISALLOW_COPY_AND_ASSIGN(ThreadBarrier);
};

}  // namespace dart

#endif  // RUNTIME_VM_THREAD_BARRIER_H_
```

