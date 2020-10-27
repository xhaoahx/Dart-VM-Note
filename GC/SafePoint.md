# SafePoint

## safe_point.h

```c++
// Copyright (c) 2016, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#ifndef RUNTIME_VM_HEAP_SAFEPOINT_H_
#define RUNTIME_VM_HEAP_SAFEPOINT_H_

#include "vm/globals.h"
#include "vm/isolate.h"
#include "vm/lockers.h"
#include "vm/thread.h"
#include "vm/thread_stack_resource.h"

namespace dart {

// A stack based scope that can be used to perform an operation after getting
// all threads to a safepoint. At the end of the operation all the threads are
// resumed.
class SafepointOperationScope : public ThreadStackResource {
 public:
  explicit SafepointOperationScope(Thread* T);
  ~SafepointOperationScope();

 private:
  DISALLOW_COPY_AND_ASSIGN(SafepointOperationScope);
};

// A stack based scope that can be used to perform an operation after getting
// all threads to a safepoint. At the end of the operation all the threads are
// resumed. Allocations in the scope will force heap growth.
class ForceGrowthSafepointOperationScope : public ThreadStackResource {
 public:
  explicit ForceGrowthSafepointOperationScope(Thread* T);
  ~ForceGrowthSafepointOperationScope();

 private:
  bool current_growth_controller_state_;

  DISALLOW_COPY_AND_ASSIGN(ForceGrowthSafepointOperationScope);
};

// Implements handling of safepoint operations for all threads in an
// IsolateGroup.
class SafepointHandler {
 public:
  explicit SafepointHandler(IsolateGroup* I);
  ~SafepointHandler();

  void EnterSafepointUsingLock(Thread* T);
  void ExitSafepointUsingLock(Thread* T);

  void BlockForSafepoint(Thread* T);

  bool IsOwnedByTheThread(Thread* thread) { return owner_ == thread; }

 private:
  void SafepointThreads(Thread* T);
  void ResumeThreads(Thread* T);

  IsolateGroup* isolate_group() const { return isolate_group_; }
  Monitor* threads_lock() const { return isolate_group_->threads_lock(); }
  bool SafepointInProgress() const {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    return ((safepoint_operation_count_ > 0) && (owner_ != NULL));
  }
  void SetSafepointInProgress(Thread* T) {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    ASSERT(owner_ == NULL);
    ASSERT(safepoint_operation_count_ == 0);
    safepoint_operation_count_ = 1;
    owner_ = T;
  }
  void ResetSafepointInProgress(Thread* T) {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    ASSERT(owner_ == T);
    ASSERT(safepoint_operation_count_ == 1);
    safepoint_operation_count_ = 0;
    owner_ = NULL;
  }
  int32_t safepoint_operation_count() const {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    return safepoint_operation_count_;
  }
  void increment_safepoint_operation_count() {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    ASSERT(safepoint_operation_count_ < kMaxInt32);
    safepoint_operation_count_ += 1;
  }
  void decrement_safepoint_operation_count() {
    ASSERT(threads_lock()->IsOwnedByCurrentThread());
    ASSERT(safepoint_operation_count_ > 0);
    safepoint_operation_count_ -= 1;
  }

  IsolateGroup* isolate_group_;

  // Monitor used by thread initiating a safepoint operation to track threads
  // not at a safepoint and wait for these threads to reach a safepoint.
  Monitor safepoint_lock_;
  int32_t number_threads_not_at_safepoint_;

  // Count that indicates if a safepoint operation is currently in progress
  // and also tracks the number of recursive safepoint operations on the
  // same thread.
  int32_t safepoint_operation_count_;

  // If a safepoint operation is currently in progress, this field contains
  // the thread that initiated the safepoint operation, otherwise it is NULL.
  Thread* owner_;

  friend class Isolate;
  friend class IsolateGroup;
  friend class SafepointOperationScope;
  friend class ForceGrowthSafepointOperationScope;
  friend class HeapIterationScope;
};

/*
 * Set of StackResource classes to track thread execution state transitions:
 *
 * kThreadInGenerated transitioning to
 *   ==> kThreadInVM:
 *       - set_execution_state(kThreadInVM).
 *       - block if safepoint is requested.
 *   ==> kThreadInNative:
 *       - set_execution_state(kThreadInNative).
 *       - EnterSafepoint().
 *   ==> kThreadInBlockedState:
 *       - Invalid transition
 *
 * kThreadInVM transitioning to
 *   ==> kThreadInGenerated
 *       - set_execution_state(kThreadInGenerated).
 *   ==> kThreadInNative
 *       - set_execution_state(kThreadInNative).
 *       - EnterSafepoint.
 *   ==> kThreadInBlockedState
 *       - set_execution_state(kThreadInBlockedState).
 *       - EnterSafepoint.
 *
 * kThreadInNative transitioning to
 *   ==> kThreadInGenerated
 *       - ExitSafepoint.
 *       - set_execution_state(kThreadInGenerated).
 *   ==> kThreadInVM
 *       - ExitSafepoint.
 *       - set_execution_state(kThreadInVM).
 *   ==> kThreadInBlocked
 *       - Invalid transition.
 *
 * kThreadInBlocked transitioning to
 *   ==> kThreadInVM
 *       - ExitSafepoint.
 *       - set_execution_state(kThreadInVM).
 *   ==> kThreadInNative
 *       - Invalid transition.
 *   ==> kThreadInGenerated
 *       - Invalid transition.
 */
class TransitionSafepointState : public ThreadStackResource {
 public:
  explicit TransitionSafepointState(Thread* T) : ThreadStackResource(T) {}
  ~TransitionSafepointState() {}

  SafepointHandler* handler() const {
    ASSERT(thread()->isolate() != NULL);
    ASSERT(thread()->isolate()->safepoint_handler() != NULL);
    return thread()->isolate()->safepoint_handler();
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionSafepointState);
};

// TransitionGeneratedToVM is used to transition the safepoint state of a
// thread from "running generated code" to "running vm code" and ensures
// that the state is reverted back to "running generated code" when
// exiting the scope/frame.
class TransitionGeneratedToVM : public TransitionSafepointState {
 public:
  explicit TransitionGeneratedToVM(Thread* T) : TransitionSafepointState(T) {
    ASSERT(T == Thread::Current());
    ASSERT(T->execution_state() == Thread::kThreadInGenerated);
    T->set_execution_state(Thread::kThreadInVM);
    // Fast check to see if a safepoint is requested or not.
    // We do the more expensive operation of blocking the thread
    // only if a safepoint is requested.
    if (T->IsSafepointRequested()) {
      handler()->BlockForSafepoint(T);
    }
  }

  ~TransitionGeneratedToVM() {
    ASSERT(thread()->execution_state() == Thread::kThreadInVM);
    thread()->set_execution_state(Thread::kThreadInGenerated);
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionGeneratedToVM);
};

// TransitionGeneratedToNative is used to transition the safepoint state of a
// thread from "running generated code" to "running native code" and ensures
// that the state is reverted back to "running generated code" when
// exiting the scope/frame.
class TransitionGeneratedToNative : public TransitionSafepointState {
 public:
  explicit TransitionGeneratedToNative(Thread* T)
      : TransitionSafepointState(T) {
    // Native code is considered to be at a safepoint and so we mark it
    // accordingly.
    ASSERT(T->execution_state() == Thread::kThreadInGenerated);
    T->set_execution_state(Thread::kThreadInNative);
    T->EnterSafepoint();
  }

  ~TransitionGeneratedToNative() {
    // We are returning to generated code and so we are not at a safepoint
    // anymore.
    ASSERT(thread()->execution_state() == Thread::kThreadInNative);
    thread()->ExitSafepoint();
    thread()->set_execution_state(Thread::kThreadInGenerated);
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionGeneratedToNative);
};

// TransitionVMToBlocked is used to transition the safepoint state of a
// thread from "running vm code" to "blocked on a monitor" and ensures
// that the state is reverted back to "running vm code" when
// exiting the scope/frame.
class TransitionVMToBlocked : public TransitionSafepointState {
 public:
  explicit TransitionVMToBlocked(Thread* T) : TransitionSafepointState(T) {
    // A thread blocked on a monitor is considered to be at a safepoint.
    ASSERT(T->execution_state() == Thread::kThreadInVM);
    T->set_execution_state(Thread::kThreadInBlockedState);
    T->EnterSafepoint();
  }

  ~TransitionVMToBlocked() {
    // We are returning to vm code and so we are not at a safepoint anymore.
    ASSERT(thread()->execution_state() == Thread::kThreadInBlockedState);
    thread()->ExitSafepoint();
    thread()->set_execution_state(Thread::kThreadInVM);
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionVMToBlocked);
};

// TransitionVMToNative is used to transition the safepoint state of a
// thread from "running vm code" to "running native code" and ensures
// that the state is reverted back to "running vm code" when
// exiting the scope/frame.
class TransitionVMToNative : public TransitionSafepointState {
 public:
  explicit TransitionVMToNative(Thread* T) : TransitionSafepointState(T) {
    // A thread running native code is considered to be at a safepoint.
    ASSERT(T->execution_state() == Thread::kThreadInVM);
    T->set_execution_state(Thread::kThreadInNative);
    T->EnterSafepoint();
  }

  ~TransitionVMToNative() {
    // We are returning to vm code and so we are not at a safepoint anymore.
    ASSERT(thread()->execution_state() == Thread::kThreadInNative);
    thread()->ExitSafepoint();
    thread()->set_execution_state(Thread::kThreadInVM);
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionVMToNative);
};

// TransitionVMToGenerated is used to transition the safepoint state of a
// thread from "running vm code" to "running generated code" and ensures
// that the state is reverted back to "running vm code" when
// exiting the scope/frame.
class TransitionVMToGenerated : public TransitionSafepointState {
 public:
  explicit TransitionVMToGenerated(Thread* T) : TransitionSafepointState(T) {
    ASSERT(T == Thread::Current());
    ASSERT(T->execution_state() == Thread::kThreadInVM);
    T->set_execution_state(Thread::kThreadInGenerated);
  }

  ~TransitionVMToGenerated() {
    ASSERT(thread()->execution_state() == Thread::kThreadInGenerated);
    thread()->set_execution_state(Thread::kThreadInVM);
    // Fast check to see if a safepoint is requested or not.
    // We do the more expensive operation of blocking the thread
    // only if a safepoint is requested.
    if (thread()->IsSafepointRequested()) {
      handler()->BlockForSafepoint(thread());
    }
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionVMToGenerated);
};

// TransitionNativeToVM is used to transition the safepoint state of a
// thread from "running native code" to "running vm code" and ensures
// that the state is reverted back to "running native code" when
// exiting the scope/frame.
class TransitionNativeToVM : public TransitionSafepointState {
 public:
  explicit TransitionNativeToVM(Thread* T) : TransitionSafepointState(T) {
    // We are about to execute vm code and so we are not at a safepoint anymore.
    ASSERT(T->execution_state() == Thread::kThreadInNative);
    if (T->no_callback_scope_depth() == 0) {
      T->ExitSafepoint();
    }
    T->set_execution_state(Thread::kThreadInVM);
  }

  ~TransitionNativeToVM() {
    // We are returning to native code and so we are at a safepoint.
    ASSERT(thread()->execution_state() == Thread::kThreadInVM);
    thread()->set_execution_state(Thread::kThreadInNative);
    if (thread()->no_callback_scope_depth() == 0) {
      thread()->EnterSafepoint();
    }
  }

 private:
  DISALLOW_COPY_AND_ASSIGN(TransitionNativeToVM);
};

// TransitionToGenerated is used to transition the safepoint state of a
// thread from "running vm code" or "running native code" to
// "running generated code" and ensures that the state is reverted back
// to "running vm code" or "running native code" when exiting the
// scope/frame.
class TransitionToGenerated : public TransitionSafepointState {
 public:
  explicit TransitionToGenerated(Thread* T)
      : TransitionSafepointState(T), execution_state_(T->execution_state()) {
    ASSERT(T == Thread::Current());
    ASSERT((execution_state_ == Thread::kThreadInVM) ||
           (execution_state_ == Thread::kThreadInNative));
    if (execution_state_ == Thread::kThreadInNative) {
      T->ExitSafepoint();
    }
    T->set_execution_state(Thread::kThreadInGenerated);
  }

  ~TransitionToGenerated() {
    ASSERT(thread()->execution_state() == Thread::kThreadInGenerated);
    if (execution_state_ == Thread::kThreadInNative) {
      thread()->set_execution_state(Thread::kThreadInNative);
      thread()->EnterSafepoint();
    } else {
      ASSERT(execution_state_ == Thread::kThreadInVM);
      thread()->set_execution_state(Thread::kThreadInVM);
    }
  }

 private:
  uint32_t execution_state_;
  DISALLOW_COPY_AND_ASSIGN(TransitionToGenerated);
};

// TransitionToVM is used to transition the safepoint state of a
// thread from "running native code" to "running vm code"
// and ensures that the state is reverted back to "running native code"
// when exiting the scope/frame.
// This transition helper is mainly used in the error path of the
// Dart API implementations where we sometimes do not have an explicit
// transition set up.
class TransitionToVM : public TransitionSafepointState {
 public:
  explicit TransitionToVM(Thread* T)
      : TransitionSafepointState(T), execution_state_(T->execution_state()) {
    ASSERT(T == Thread::Current());
    ASSERT((execution_state_ == Thread::kThreadInVM) ||
           (execution_state_ == Thread::kThreadInNative));
    if (execution_state_ == Thread::kThreadInNative) {
      T->ExitSafepoint();
      T->set_execution_state(Thread::kThreadInVM);
    }
    ASSERT(T->execution_state() == Thread::kThreadInVM);
  }

  ~TransitionToVM() {
    ASSERT(thread()->execution_state() == Thread::kThreadInVM);
    if (execution_state_ == Thread::kThreadInNative) {
      thread()->set_execution_state(Thread::kThreadInNative);
      thread()->EnterSafepoint();
    }
  }

 private:
  uint32_t execution_state_;
  DISALLOW_COPY_AND_ASSIGN(TransitionToVM);
};

}  // namespace dart

#endif  // RUNTIME_VM_HEAP_SAFEPOINT_H_
```



## safe_point.cc

```c++
// Copyright (c) 2016, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#include "vm/heap/safepoint.h"

#include "vm/heap/heap.h"
#include "vm/thread.h"
#include "vm/thread_registry.h"

namespace dart {

DEFINE_FLAG(bool, trace_safepoint, false, "Trace Safepoint logic.");

SafepointOperationScope::SafepointOperationScope(Thread* T)
    : ThreadStackResource(T) {
  ASSERT(T != nullptr && T->isolate_group() != nullptr);

  SafepointHandler* handler = T->isolate_group()->safepoint_handler();
  ASSERT(handler != NULL);

  // Signal all threads to get to a safepoint and wait for them to
  // get to a safepoint.
  handler->SafepointThreads(T);
}

SafepointOperationScope::~SafepointOperationScope() {
  Thread* T = thread();
  ASSERT(T != nullptr && T->isolate_group() != nullptr);

  // Resume all threads which are blocked for the safepoint operation.
  SafepointHandler* handler = T->isolate_group()->safepoint_handler();
  ASSERT(handler != NULL);
  handler->ResumeThreads(T);
}

ForceGrowthSafepointOperationScope::ForceGrowthSafepointOperationScope(
    Thread* T)
    : ThreadStackResource(T) {
  ASSERT(T != NULL);
  IsolateGroup* IG = T->isolate_group();
  ASSERT(IG != NULL);

  SafepointHandler* handler = IG->safepoint_handler();
  ASSERT(handler != NULL);

  // Signal all threads to get to a safepoint and wait for them to
  // get to a safepoint.
  handler->SafepointThreads(T);

  // N.B.: Change growth policy inside the safepoint to prevent racy access.
  Heap* heap = IG->heap();
  current_growth_controller_state_ = heap->GrowthControlState();
  heap->DisableGrowthControl();
}

ForceGrowthSafepointOperationScope::~ForceGrowthSafepointOperationScope() {
  Thread* T = thread();
  ASSERT(T != NULL);
  IsolateGroup* IG = T->isolate_group();
  ASSERT(IG != NULL);

  // N.B.: Change growth policy inside the safepoint to prevent racy access.
  Heap* heap = IG->heap();
  heap->SetGrowthControlState(current_growth_controller_state_);

  // Resume all threads which are blocked for the safepoint operation.
  SafepointHandler* handler = IG->safepoint_handler();
  ASSERT(handler != NULL);
  handler->ResumeThreads(T);

  if (current_growth_controller_state_) {
    ASSERT(T->CanCollectGarbage());
    // Check if we passed the growth limit during the scope.
    if (heap->old_space()->ReachedHardThreshold()) {
      heap->CollectGarbage(Heap::kMarkSweep, Heap::kOldSpace);
    } else {
      heap->CheckStartConcurrentMarking(T, Heap::kOldSpace);
    }
  }
}

SafepointHandler::SafepointHandler(IsolateGroup* isolate_group)
    : isolate_group_(isolate_group),
      safepoint_lock_(),
      number_threads_not_at_safepoint_(0),
      safepoint_operation_count_(0),
      owner_(NULL) {}

SafepointHandler::~SafepointHandler() {
  ASSERT(owner_ == NULL);
  ASSERT(safepoint_operation_count_ == 0);
  isolate_group_ = NULL;
}

void SafepointHandler::SafepointThreads(Thread* T) {
  ASSERT(T->no_safepoint_scope_depth() == 0);
  ASSERT(T->execution_state() == Thread::kThreadInVM);

  {
    // First grab the threads list lock for this isolate
    // and check if a safepoint is already in progress. This
    // ensures that two threads do not start a safepoint operation
    // at the same time.
    MonitorLocker sl(threads_lock());

    // Now check to see if a safepoint operation is already in progress
    // for this isolate, block if an operation is in progress.
    while (SafepointInProgress()) {
      // If we are recursively invoking a Safepoint operation then we
      // just increment the count and return, otherwise we wait for the
      // safepoint operation to be done.
      if (owner_ == T) {
        increment_safepoint_operation_count();
        return;
      }
      sl.WaitWithSafepointCheck(T);
    }

    // Set safepoint in progress state by this thread.
    SetSafepointInProgress(T);

    // Go over the active thread list and ensure that all threads active
    // in the isolate reach a safepoint.
    Thread* current = isolate_group()->thread_registry()->active_list();
    while (current != NULL) {
      MonitorLocker tl(current->thread_lock());
      if (!current->BypassSafepoints()) {
        if (current == T) {
          current->SetAtSafepoint(true);
        } else {
          uint32_t state = current->SetSafepointRequested(true);
          if (!Thread::IsAtSafepoint(state)) {
            // Thread is not already at a safepoint so try to
            // get it to a safepoint and wait for it to check in.
            if (current->IsMutatorThread()) {
              current->ScheduleInterruptsLocked(Thread::kVMInterrupt);
            }
            MonitorLocker sl(&safepoint_lock_);
            ++number_threads_not_at_safepoint_;
          }
        }
      }
      current = current->next();
    }
  }
  // Now wait for all threads that are not already at a safepoint to check-in.
  {
    MonitorLocker sl(&safepoint_lock_);
    intptr_t num_attempts = 0;
    while (number_threads_not_at_safepoint_ > 0) {
      Monitor::WaitResult retval = sl.Wait(1000);
      if (retval == Monitor::kTimedOut) {
        num_attempts += 1;
        if (FLAG_trace_safepoint && num_attempts > 10) {
          // We have been waiting too long, start logging this as we might
          // have an issue where a thread is not checking in for a safepoint.
          for (Thread* current =
                   isolate_group()->thread_registry()->active_list();
               current != NULL; current = current->next()) {
            if (!current->IsAtSafepoint()) {
              OS::PrintErr("Attempt:%" Pd
                           " waiting for thread %s to check in\n",
                           num_attempts, current->os_thread()->name());
            }
          }
        }
      }
    }
  }
}

void SafepointHandler::ResumeThreads(Thread* T) {
  // First resume all the threads which are blocked for the safepoint
  // operation.
  MonitorLocker sl(threads_lock());

  // First check if we are in a recursive safepoint operation, in that case
  // we just decrement safepoint_operation_count and return.
  ASSERT(SafepointInProgress());
  if (safepoint_operation_count() > 1) {
    decrement_safepoint_operation_count();
    return;
  }
  Thread* current = isolate_group()->thread_registry()->active_list();
  while (current != NULL) {
    MonitorLocker tl(current->thread_lock());
    if (!current->BypassSafepoints()) {
      if (current == T) {
        current->SetAtSafepoint(false);
      } else {
        uint32_t state = current->SetSafepointRequested(false);
        if (Thread::IsBlockedForSafepoint(state)) {
          tl.Notify();
        }
      }
    }
    current = current->next();
  }
  // Now reset the safepoint_in_progress_ state and notify all threads
  // that are waiting to enter the isolate or waiting to start another
  // safepoint operation.
  ResetSafepointInProgress(T);
  sl.NotifyAll();
}

void SafepointHandler::EnterSafepointUsingLock(Thread* T) {
  MonitorLocker tl(T->thread_lock());
  T->SetAtSafepoint(true);
  if (T->IsSafepointRequested()) {
    MonitorLocker sl(&safepoint_lock_);
    ASSERT(number_threads_not_at_safepoint_ > 0);
    number_threads_not_at_safepoint_ -= 1;
    sl.Notify();
  }
}

void SafepointHandler::ExitSafepointUsingLock(Thread* T) {
  MonitorLocker tl(T->thread_lock());
  ASSERT(T->IsAtSafepoint());
  while (T->IsSafepointRequested()) {
    T->SetBlockedForSafepoint(true);
    tl.Wait();
    T->SetBlockedForSafepoint(false);
  }
  T->SetAtSafepoint(false);
}

void SafepointHandler::BlockForSafepoint(Thread* T) {
  ASSERT(!T->BypassSafepoints());
  MonitorLocker tl(T->thread_lock());
  if (T->IsSafepointRequested()) {
    T->SetAtSafepoint(true);
    {
      MonitorLocker sl(&safepoint_lock_);
      ASSERT(number_threads_not_at_safepoint_ > 0);
      number_threads_not_at_safepoint_ -= 1;
      sl.Notify();
    }
    while (T->IsSafepointRequested()) {
      T->SetBlockedForSafepoint(true);
      tl.Wait();
      T->SetBlockedForSafepoint(false);
    }
    T->SetAtSafepoint(false);
  }
}

}  // namespace dart

```

