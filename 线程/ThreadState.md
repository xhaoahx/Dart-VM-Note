# ThreadState

## thread_state.h

```c++
// Copyright (c) 2019, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#ifndef RUNTIME_VM_THREAD_STATE_H_
#define RUNTIME_VM_THREAD_STATE_H_

#include "include/dart_api.h"
#include "vm/os_thread.h"

namespace dart {

class HandleScope;
class LongJumpScope;
class Zone;

// ThreadState 辅助线程本地状态的容器
// 例如：它持有用于分配的 Zone 栈，和用于栈展开的 StackResources 栈
//
// 注意：此类在编译器和运行时之间共享，因此它不能暴露任何的运行时内部
class ThreadState : public BaseThread {
 public:
  // 当前正在执行的线程。如果还未初始化，返回 NULL
  static ThreadState* Current() {
#if defined(HAS_C11_THREAD_LOCAL)
    return OSThread::CurrentVMThread();
#else
    BaseThread* thread = OSThread::GetCurrentTLS();
    if (thread == NULL || thread->is_os_thread()) {
      return NULL;
    }
    return static_cast<ThreadState*>(thread);
#endif
  }

  explicit ThreadState(bool is_os_thread);
  virtual ~ThreadState();

  // 此线程对应的操作系统线程
  OSThread* os_thread() const { return os_thread_; }
  // 设置操作系统线程
  void set_os_thread(OSThread* os_thread) { os_thread_ = os_thread; }

  // 此线程用最顶层的用于分配的 Zone
  Zone* zone() const { return zone_; }

  // 给定的 Zone 是否被此线程持有
  bool ZoneIsOwnedByThread(Zone* zone) const;

  // 增加内存容量
  void IncrementMemoryCapacity(uintptr_t value) {
    current_zone_capacity_ += value;
    // 更新 zone 水线
    if (current_zone_capacity_ > zone_high_watermark_) {
      zone_high_watermark_ = current_zone_capacity_;
    }
  }

  // 减少内存容量
  void DecrementMemoryCapacity(uintptr_t value) {
    ASSERT(current_zone_capacity_ >= value);
    current_zone_capacity_ -= value;
  }

  // 当前内存容量
  uintptr_t current_zone_capacity() const { return current_zone_capacity_; }
  // zone 水线
  uintptr_t zone_high_watermark() const { return zone_high_watermark_; }

  // 将 zone 水线设置为当前 zone 容量
  void ResetHighWatermark() { zone_high_watermark_ = current_zone_capacity_; }

  // StackResources 相关操作
  StackResource* top_resource() const { return top_resource_; }
  void set_top_resource(StackResource* value) { top_resource_ = value; }
  static intptr_t top_resource_offset() {
    return OFFSET_OF(ThreadState, top_resource_);
  }

  LongJumpScope* long_jump_base() const { return long_jump_base_; }
  void set_long_jump_base(LongJumpScope* value) { long_jump_base_ = value; }

  // Handle 相关
  bool IsValidZoneHandle(Dart_Handle object) const;
  intptr_t CountZoneHandles() const;
  bool IsValidScopedHandle(Dart_Handle object) const;
  intptr_t CountScopedHandles() const;

  // 是否可用分配新的 Handle
  virtual bool MayAllocateHandles() = 0;

  HandleScope* top_handle_scope() const {
#if defined(DEBUG)
    return top_handle_scope_;
#else
    return 0;
#endif
  }

  void set_top_handle_scope(HandleScope* handle_scope) {
#if defined(DEBUG)
    top_handle_scope_ = handle_scope;
#endif
  }

 private:
  void set_zone(Zone* zone) { zone_ = zone; }

  OSThread* os_thread_ = nullptr;
  Zone* zone_ = nullptr;
  uintptr_t current_zone_capacity_ = 0;
  uintptr_t zone_high_watermark_ = 0;
  StackResource* top_resource_ = nullptr;
  LongJumpScope* long_jump_base_ = nullptr;

  // This field is only used in the DEBUG builds, but we don't exclude it
  // because it would cause RELEASE and DEBUG builds to have different
  // offsets for various Thread fields that are used from generated code.
  HandleScope* top_handle_scope_ = nullptr;

  friend class ApiZone;
  friend class StackZone;
};

}  // namespace dart

#endif  // RUNTIME_VM_THREAD_STATE_H_

```

## thread_state.cc

```c++
// Copyright (c) 2019, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#include "vm/thread_state.h"

#include "vm/handles_impl.h"
#include "vm/zone.h"

namespace dart {

ThreadState::ThreadState(bool is_os_thread) : BaseThread(is_os_thread) {
  // 
  if (zone_ == nullptr) {
    ASSERT(current_zone_capacity_ == 0);
  } else {
    Zone* current = zone_;
    uintptr_t total_zone_capacity = 0;
    while (current != nullptr) {
      total_zone_capacity += current->CapacityInBytes();
      current = current->previous();
    }
    ASSERT(current_zone_capacity_ == total_zone_capacity);
  }
}

ThreadState::~ThreadState() {}

/// 给定的 zone 是否被本线程持有
bool ThreadState::ZoneIsOwnedByThread(Zone* zone) const {
  ASSERT(zone != nullptr);
  Zone* current = zone_;
  while (current != nullptr) {
    if (current == zone) {
      return true;
    }
    current = current->previous();
  }
  return false;
}

bool ThreadState::IsValidZoneHandle(Dart_Handle object) const {
  Zone* zone = this->zone();
  while (zone != NULL) {
    if (zone->handles()->IsValidZoneHandle(reinterpret_cast<uword>(object))) {
      return true;
    }
    zone = zone->previous();
  }
  return false;
}

intptr_t ThreadState::CountZoneHandles() const {
  intptr_t count = 0;
  Zone* zone = this->zone();
  while (zone != NULL) {
    count += zone->handles()->CountZoneHandles();
    zone = zone->previous();
  }
  ASSERT(count >= 0);
  return count;
}

bool ThreadState::IsValidScopedHandle(Dart_Handle object) const {
  Zone* zone = this->zone();
  while (zone != NULL) {
    if (zone->handles()->IsValidScopedHandle(reinterpret_cast<uword>(object))) {
      return true;
    }
    zone = zone->previous();
  }
  return false;
}

intptr_t ThreadState::CountScopedHandles() const {
  intptr_t count = 0;
  Zone* zone = this->zone();
  while (zone != NULL) {
    count += zone->handles()->CountScopedHandles();
    zone = zone->previous();
  }
  ASSERT(count >= 0);
  return count;
}

}  // namespace dart

```

