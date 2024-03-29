# Heap

## heap.h

```c++
// Copyright (c) 2012, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#ifndef RUNTIME_VM_HEAP_HEAP_H_
#define RUNTIME_VM_HEAP_HEAP_H_

#if defined(SHOULD_NOT_INCLUDE_RUNTIME)
#error "Should not include runtime"
#endif

#include "include/dart_tools_api.h"

#include "platform/assert.h"
#include "vm/allocation.h"
#include "vm/flags.h"
#include "vm/globals.h"
#include "vm/heap/pages.h"
#include "vm/heap/scavenger.h"
#include "vm/heap/spaces.h"
#include "vm/heap/weak_table.h"
#include "vm/isolate.h"

namespace dart {

// Forward declarations.
class Isolate;
class IsolateGroup;
class ObjectPointerVisitor;
class ObjectSet;
class ServiceEvent;
class TimelineEventScope;
class VirtualMemory;

class Heap {
 public:
  enum Space {
    kNew,
    kOld,
    kCode,
  };

  enum WeakSelector {
    kPeers = 0,
#if !defined(HASH_IN_OBJECT_HEADER)
    kIdentityHashes,
#endif
    kCanonicalHashes,
    kObjectIds,
    kLoadingUnits,
    kNumWeakSelectors
  };

  enum GCType {
    kScavenge,
    kMarkSweep,
    kMarkCompact,
  };

  enum GCReason {
    kNewSpace,     // New space is full.
    kPromotion,    // Old space limit crossed after a scavenge.
    kOldSpace,     // Old space limit crossed.
    kFinalize,     // Concurrent marking finished.
    kFull,         // Heap::CollectAllGarbage
    kExternal,     // Dart_NewFinalizableHandle Dart_NewWeakPersistentHandle
    kIdle,         // Dart_NotifyIdle
    kLowMemory,    // Dart_NotifyLowMemory
    kDebugging,    // service request, etc.
    kSendAndExit,  // SendPort.sendAndExit
  };

  // Pattern for unused new space and swept old space.
  static const uint8_t kZapByte = 0xf3;

  ~Heap();

  Scavenger* new_space() { return &new_space_; }
  PageSpace* old_space() { return &old_space_; }

  uword Allocate(intptr_t size, Space space) {
    ASSERT(!read_only_);
    switch (space) {
      case kNew:
        // Do not attempt to allocate very large objects in new space.
        if (!IsAllocatableInNewSpace(size)) {
          return AllocateOld(size, OldPage::kData);
        }
        return AllocateNew(size);
      case kOld:
        return AllocateOld(size, OldPage::kData);
      case kCode:
        return AllocateOld(size, OldPage::kExecutable);
      default:
        UNREACHABLE();
    }
    return 0;
  }

  // Track external data.
  void AllocatedExternal(intptr_t size, Space space);
  void FreedExternal(intptr_t size, Space space);
  // Move external size from new to old space. Does not by itself trigger GC.
  void PromotedExternal(intptr_t size);

  // Heap contains the specified address.
  bool Contains(uword addr) const;
  bool NewContains(uword addr) const;
  bool OldContains(uword addr) const;
  bool CodeContains(uword addr) const;
  bool DataContains(uword addr) const;

  // Find an object by visiting all pointers in the specified heap space,
  // the 'visitor' is used to determine if an object is found or not.
  // The 'visitor' function should be set up to return true if the
  // object is found, traversal through the heap space stops at that
  // point.
  // The 'visitor' function should return false if the object is not found,
  // traversal through the heap space continues.
  // Returns null object if nothing is found.
  InstructionsPtr FindObjectInCodeSpace(FindObjectVisitor* visitor) const;
  ObjectPtr FindOldObject(FindObjectVisitor* visitor) const;
  ObjectPtr FindNewObject(FindObjectVisitor* visitor);
  ObjectPtr FindObject(FindObjectVisitor* visitor);

  void HintFreed(intptr_t size);
  void NotifyIdle(int64_t deadline);
  void NotifyLowMemory();

  // Collect a single generation.
  void CollectGarbage(Space space);
  void CollectGarbage(GCType type, GCReason reason);

  // Collect both generations by performing a scavenge followed by a
  // mark-sweep. This function may not collect all unreachable objects. Because
  // mark-sweep treats new space as roots, a cycle between unreachable old and
  // new objects will not be collected until the new objects are promoted.
  // Verification based on heap iteration should instead use CollectAllGarbage.
  void CollectMostGarbage(GCReason reason = kFull);

  // Collect both generations by performing an evacuation followed by a
  // mark-sweep. If incremental marking was in progress, perform another
  // mark-sweep. This function will collect all unreachable objects, including
  // those in inter-generational cycles or stored during incremental marking.
  void CollectAllGarbage(GCReason reason = kFull);

  void CheckStartConcurrentMarking(Thread* thread, GCReason reason);
  void StartConcurrentMarking(Thread* thread);
  void CheckFinishConcurrentMarking(Thread* thread);
  void WaitForMarkerTasks(Thread* thread);
  void WaitForSweeperTasks(Thread* thread);
  void WaitForSweeperTasksAtSafepoint(Thread* thread);

  // Enables growth control on the page space heaps.  This should be
  // called before any user code is executed.
  void InitGrowthControl();
  void DisableGrowthControl() { SetGrowthControlState(false); }
  void SetGrowthControlState(bool state);
  bool GrowthControlState();

  // Protect access to the heap. Note: Code pages are made
  // executable/non-executable when 'read_only' is true/false, respectively.
  void WriteProtect(bool read_only);
  void WriteProtectCode(bool read_only) {
    old_space_.WriteProtectCode(read_only);
  }

  // Initialize the heap and register it with the isolate.
  static void Init(IsolateGroup* isolate_group,
                   intptr_t max_new_gen_words,
                   intptr_t max_old_gen_words);

  // Returns a suitable name for a VM region in the heap.
  static const char* RegionName(Space space);

  // Verify that all pointers in the heap point to the heap.
  bool Verify(MarkExpectation mark_expectation = kForbidMarked);

  // Print heap sizes.
  void PrintSizes() const;

  // Return amount of memory used and capacity in a space, excluding external.
  int64_t UsedInWords(Space space) const;
  int64_t CapacityInWords(Space space) const;
  int64_t ExternalInWords(Space space) const;

  int64_t TotalUsedInWords() const;
  int64_t TotalCapacityInWords() const;
  int64_t TotalExternalInWords() const;
  // Return the amount of GCing in microseconds.
  int64_t GCTimeInMicros(Space space) const;

  intptr_t Collections(Space space) const;

  ObjectSet* CreateAllocatedObjectSet(Zone* zone,
                                      MarkExpectation mark_expectation);

  static const char* GCTypeToString(GCType type);
  static const char* GCReasonToString(GCReason reason);

  // Associate a peer with an object.  A non-existent peer is equal to NULL.
  void SetPeer(ObjectPtr raw_obj, void* peer) {
    SetWeakEntry(raw_obj, kPeers, reinterpret_cast<intptr_t>(peer));
  }
  void* GetPeer(ObjectPtr raw_obj) const {
    return reinterpret_cast<void*>(GetWeakEntry(raw_obj, kPeers));
  }
  int64_t PeerCount() const;

#if !defined(HASH_IN_OBJECT_HEADER)
  // Associate an identity hashCode with an object. An non-existent hashCode
  // is equal to 0.
  void SetHash(ObjectPtr raw_obj, intptr_t hash) {
    SetWeakEntry(raw_obj, kIdentityHashes, hash);
  }
  intptr_t GetHash(ObjectPtr raw_obj) const {
    return GetWeakEntry(raw_obj, kIdentityHashes);
  }
#endif

  void SetCanonicalHash(ObjectPtr raw_obj, intptr_t hash) {
    SetWeakEntry(raw_obj, kCanonicalHashes, hash);
  }
  intptr_t GetCanonicalHash(ObjectPtr raw_obj) const {
    return GetWeakEntry(raw_obj, kCanonicalHashes);
  }
  void ResetCanonicalHashTable();

  // Associate an id with an object (used when serializing an object).
  // A non-existant id is equal to 0.
  void SetObjectId(ObjectPtr raw_obj, intptr_t object_id) {
    ASSERT(Thread::Current()->IsMutatorThread());
    SetWeakEntry(raw_obj, kObjectIds, object_id);
  }
  intptr_t GetObjectId(ObjectPtr raw_obj) const {
    ASSERT(Thread::Current()->IsMutatorThread());
    return GetWeakEntry(raw_obj, kObjectIds);
  }
  void ResetObjectIdTable();

  void SetLoadingUnit(ObjectPtr raw_obj, intptr_t object_id) {
    ASSERT(Thread::Current()->IsMutatorThread());
    SetWeakEntry(raw_obj, kLoadingUnits, object_id);
  }
  intptr_t GetLoadingUnit(ObjectPtr raw_obj) const {
    ASSERT(Thread::Current()->IsMutatorThread());
    return GetWeakEntry(raw_obj, kLoadingUnits);
  }

  // Used by the GC algorithms to propagate weak entries.
  intptr_t GetWeakEntry(ObjectPtr raw_obj, WeakSelector sel) const;
  void SetWeakEntry(ObjectPtr raw_obj, WeakSelector sel, intptr_t val);

  WeakTable* GetWeakTable(Space space, WeakSelector selector) const {
    if (space == kNew) {
      return new_weak_tables_[selector];
    }
    ASSERT(space == kOld);
    return old_weak_tables_[selector];
  }
  void SetWeakTable(Space space, WeakSelector selector, WeakTable* value) {
    if (space == kNew) {
      new_weak_tables_[selector] = value;
    } else {
      ASSERT(space == kOld);
      old_weak_tables_[selector] = value;
    }
  }

  void ForwardWeakEntries(ObjectPtr before_object, ObjectPtr after_object);
  void ForwardWeakTables(ObjectPointerVisitor* visitor);

  // Stats collection.
  void RecordTime(int id, int64_t micros) {
    ASSERT((id >= 0) && (id < GCStats::kTimeEntries));
    stats_.times_[id] = micros;
  }

  void RecordData(int id, intptr_t value) {
    ASSERT((id >= 0) && (id < GCStats::kDataEntries));
    stats_.data_[id] = value;
  }

  void UpdateGlobalMaxUsed();

  static bool IsAllocatableInNewSpace(intptr_t size) {
    return size <= kNewAllocatableSize;
  }
  static bool IsAllocatableViaFreeLists(intptr_t size) {
    return size < kAllocatablePageSize;
  }

#ifndef PRODUCT
  void PrintToJSONObject(Space space, JSONObject* object) const;

  // Returns a JSON object with total memory usage statistics for both new and
  // old space combined.
  void PrintMemoryUsageJSON(JSONStream* stream) const;
  void PrintMemoryUsageJSON(JSONObject* jsobj) const;

  // The heap map contains the sizes and class ids for the objects in each page.
  void PrintHeapMapToJSONStream(Isolate* isolate, JSONStream* stream) {
    old_space_.PrintHeapMapToJSONStream(isolate, stream);
  }
#endif  // PRODUCT

  IsolateGroup* isolate_group() const { return isolate_group_; }

  Monitor* barrier() const { return &barrier_; }
  Monitor* barrier_done() const { return &barrier_done_; }

  void SetupImagePage(void* pointer, uword size, bool is_executable) {
    old_space_.SetupImagePage(pointer, size, is_executable);
  }

  static const intptr_t kNewAllocatableSize = 256 * KB;
  static const intptr_t kAllocatablePageSize = 64 * KB;

  Space SpaceForExternal(intptr_t size) const;

  void CollectOnNthAllocation(intptr_t num_allocations);

  void MergeFrom(Heap* donor);

 private:
  class GCStats : public ValueObject {
   public:
    GCStats() {}
    intptr_t num_;
    Heap::GCType type_;
    Heap::GCReason reason_;

    class Data : public ValueObject {
     public:
      Data() {}
      int64_t micros_;
      SpaceUsage new_;
      SpaceUsage old_;

     private:
      DISALLOW_COPY_AND_ASSIGN(Data);
    };

    enum { kTimeEntries = 6 };
    enum { kDataEntries = 4 };

    Data before_;
    Data after_;
    int64_t times_[kTimeEntries];
    intptr_t data_[kDataEntries];

   private:
    DISALLOW_COPY_AND_ASSIGN(GCStats);
  };

  Heap(IsolateGroup* isolate_group,
       intptr_t max_new_gen_semi_words,  // Max capacity of new semi-space.
       intptr_t max_old_gen_words);

  uword AllocateNew(intptr_t size);
  uword AllocateOld(intptr_t size, OldPage::PageType type);

  // Visit all pointers. Caller must ensure concurrent sweeper is not running,
  // and the visitor must not allocate.
  void VisitObjectPointers(ObjectPointerVisitor* visitor);

  // Visit all objects, including FreeListElement "objects". Caller must ensure
  // concurrent sweeper is not running, and the visitor must not allocate.
  void VisitObjects(ObjectVisitor* visitor);
  void VisitObjectsNoImagePages(ObjectVisitor* visitor);
  void VisitObjectsImagePages(ObjectVisitor* visitor) const;

  // Like Verify, but does not wait for concurrent sweeper, so caller must
  // ensure thread-safety.
  bool VerifyGC(MarkExpectation mark_expectation = kForbidMarked);

  // Helper functions for garbage collection.
  void CollectNewSpaceGarbage(Thread* thread, GCReason reason);
  void CollectOldSpaceGarbage(Thread* thread, GCType type, GCReason reason);
  void EvacuateNewSpace(Thread* thread, GCReason reason);

  // GC stats collection.
  void RecordBeforeGC(GCType type, GCReason reason);
  void RecordAfterGC(GCType type);
  void PrintStats();
  void PrintStatsToTimeline(TimelineEventScope* event, GCReason reason);

  void AddRegionsToObjectSet(ObjectSet* set) const;

  // Trigger major GC if 'gc_on_nth_allocation_' is set.
  void CollectForDebugging();

  IsolateGroup* isolate_group_;

  // The different spaces used for allocation.
  Scavenger new_space_;
  PageSpace old_space_;

  WeakTable* new_weak_tables_[kNumWeakSelectors];
  WeakTable* old_weak_tables_[kNumWeakSelectors];

  mutable Monitor barrier_;
  mutable Monitor barrier_done_;

  // GC stats collection.
  GCStats stats_;

  // This heap is in read-only mode: No allocation is allowed.
  bool read_only_;

  bool last_gc_was_old_space_;
  bool assume_scavenge_will_fail_;

  static const intptr_t kNoForcedGarbageCollection = -1;

  // Whether the next heap allocation (new or old) should trigger
  // CollectAllGarbage. Used within unit tests for testing GC on certain
  // sensitive codepaths.
  intptr_t gc_on_nth_allocation_;

  friend class Become;       // VisitObjectPointers
  friend class GCCompactor;  // VisitObjectPointers
  friend class Precompiler;  // VisitObjects
  friend class Unmarker;     // VisitObjects
  friend class ServiceEvent;
  friend class Scavenger;             // VerifyGC
  friend class PageSpace;             // VerifyGC
  friend class IsolateReloadContext;  // VisitObjects
  friend class ClassFinalizer;        // VisitObjects
  friend class HeapIterationScope;    // VisitObjects
  friend class ProgramVisitor;        // VisitObjectsImagePages
  friend class Serializer;            // VisitObjectsImagePages
  friend class HeapTestHelper;
  friend class MetricsTestHelper;

  DISALLOW_COPY_AND_ASSIGN(Heap);
};

class HeapIterationScope : public ThreadStackResource {
 public:
  explicit HeapIterationScope(Thread* thread, bool writable = false);
  ~HeapIterationScope();

  void IterateObjects(ObjectVisitor* visitor) const;
  void IterateObjectsNoImagePages(ObjectVisitor* visitor) const;
  void IterateOldObjects(ObjectVisitor* visitor) const;
  void IterateOldObjectsNoImagePages(ObjectVisitor* visitor) const;

  void IterateVMIsolateObjects(ObjectVisitor* visitor) const;

  void IterateObjectPointers(ObjectPointerVisitor* visitor,
                             ValidationPolicy validate_frames);
  void IterateStackPointers(ObjectPointerVisitor* visitor,
                            ValidationPolicy validate_frames);

 private:
  Heap* heap_;
  PageSpace* old_space_;
  bool writable_;

  DISALLOW_COPY_AND_ASSIGN(HeapIterationScope);
};

class NoHeapGrowthControlScope : public ThreadStackResource {
 public:
  NoHeapGrowthControlScope();
  ~NoHeapGrowthControlScope();

 private:
  bool current_growth_controller_state_;
  DISALLOW_COPY_AND_ASSIGN(NoHeapGrowthControlScope);
};

// Note: During this scope all pages are writable and the code pages are
// non-executable.
class WritableVMIsolateScope : ThreadStackResource {
 public:
  explicit WritableVMIsolateScope(Thread* thread);
  ~WritableVMIsolateScope();
};

class WritableCodePages : StackResource {
 public:
  explicit WritableCodePages(Thread* thread, Isolate* isolate);
  ~WritableCodePages();

 private:
  Isolate* isolate_;
};

#if defined(TESTING)
class GCTestHelper : public AllStatic {
 public:
  // Collect new gen without triggering any side effects. The normal call to
  // CollectGarbage(Heap::kNew) could potentially trigger an old gen collection
  // if there is enough promotion, and this can perturb some tests.
  static void CollectNewSpace() {
    Thread* thread = Thread::Current();
    ASSERT(thread->execution_state() == Thread::kThreadInVM);
    thread->heap()->new_space()->Scavenge();
  }

  // Fully collect old gen and wait for the sweeper to finish. The normal call
  // to CollectGarbage(Heap::kOld) may leave so-called "floating garbage",
  // objects that were seen by the incremental barrier but later made
  // unreachable, and this can perturb some tests.
  static void CollectOldSpace() {
    Thread* thread = Thread::Current();
    ASSERT(thread->execution_state() == Thread::kThreadInVM);
    if (thread->is_marking()) {
      thread->heap()->CollectGarbage(Heap::kMarkSweep, Heap::kDebugging);
    }
    thread->heap()->CollectGarbage(Heap::kMarkSweep, Heap::kDebugging);
    WaitForGCTasks();
  }

  static void CollectAllGarbage() {
    Thread* thread = Thread::Current();
    ASSERT(thread->execution_state() == Thread::kThreadInVM);
    thread->heap()->CollectAllGarbage(Heap::kDebugging);
  }

  static void WaitForGCTasks() {
    Thread* thread = Thread::Current();
    ASSERT(thread->execution_state() == Thread::kThreadInVM);
    thread->heap()->WaitForMarkerTasks(thread);
    thread->heap()->WaitForSweeperTasks(thread);
  }
};
#endif  // TESTING

}  // namespace dart

#endif  // RUNTIME_VM_HEAP_HEAP_H_
```



## heap.cc

```c++
// Copyright (c) 2012, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#include <memory>
#include <utility>

#include "vm/heap/heap.h"

#include "platform/assert.h"
#include "platform/utils.h"
#include "vm/compiler/jit/compiler.h"
#include "vm/dart.h"
#include "vm/flags.h"
#include "vm/heap/pages.h"
#include "vm/heap/safepoint.h"
#include "vm/heap/scavenger.h"
#include "vm/heap/verifier.h"
#include "vm/heap/weak_table.h"
#include "vm/isolate.h"
#include "vm/lockers.h"
#include "vm/object.h"
#include "vm/object_set.h"
#include "vm/os.h"
#include "vm/raw_object.h"
#include "vm/service.h"
#include "vm/service_event.h"
#include "vm/service_isolate.h"
#include "vm/stack_frame.h"
#include "vm/tags.h"
#include "vm/thread_pool.h"
#include "vm/timeline.h"
#include "vm/virtual_memory.h"

namespace dart {

DEFINE_FLAG(bool, write_protect_vm_isolate, true, "Write protect vm_isolate.");
DEFINE_FLAG(bool,
            disable_heap_verification,
            false,
            "Explicitly disable heap verification.");

// We ensure that the GC does not use the current isolate.
class NoActiveIsolateScope {
 public:
  NoActiveIsolateScope() : thread_(Thread::Current()) {
    saved_isolate_ = thread_->isolate_;
    thread_->isolate_ = nullptr;
  }
  ~NoActiveIsolateScope() {
    ASSERT(thread_->isolate_ == nullptr);
    thread_->isolate_ = saved_isolate_;
  }

 private:
  Thread* thread_;
  Isolate* saved_isolate_;
};

Heap::Heap(IsolateGroup* isolate_group,
           intptr_t max_new_gen_semi_words,
           intptr_t max_old_gen_words)
    : isolate_group_(isolate_group),
      new_space_(this, max_new_gen_semi_words),
      old_space_(this, max_old_gen_words),
      barrier_(),
      barrier_done_(),
      read_only_(false),
      last_gc_was_old_space_(false),
      assume_scavenge_will_fail_(false),
      gc_on_nth_allocation_(kNoForcedGarbageCollection) {
  UpdateGlobalMaxUsed();
  for (int sel = 0; sel < kNumWeakSelectors; sel++) {
    new_weak_tables_[sel] = new WeakTable();
    old_weak_tables_[sel] = new WeakTable();
  }
  stats_.num_ = 0;
}

Heap::~Heap() {
  for (int sel = 0; sel < kNumWeakSelectors; sel++) {
    delete new_weak_tables_[sel];
    delete old_weak_tables_[sel];
  }
}

uword Heap::AllocateNew(intptr_t size) {
  ASSERT(Thread::Current()->no_safepoint_scope_depth() == 0);
  CollectForDebugging();
  Thread* thread = Thread::Current();
  uword addr = new_space_.TryAllocate(thread, size);
  if (LIKELY(addr != 0)) {
    return addr;
  }
  if (!assume_scavenge_will_fail_ && new_space_.GrowthControlState()) {
    // This call to CollectGarbage might end up "reusing" a collection spawned
    // from a different thread and will be racing to allocate the requested
    // memory with other threads being released after the collection.
    CollectGarbage(kNew);

    addr = new_space_.TryAllocate(thread, size);
    if (LIKELY(addr != 0)) {
      return addr;
    }
  }

  // It is possible a GC doesn't clear enough space.
  // In that case, we must fall through and allocate into old space.
  return AllocateOld(size, OldPage::kData);
}

uword Heap::AllocateOld(intptr_t size, OldPage::PageType type) {
  ASSERT(Thread::Current()->no_safepoint_scope_depth() == 0);
  if (old_space_.GrowthControlState()) {
    CollectForDebugging();
    uword addr = old_space_.TryAllocate(size, type);
    if (addr != 0) {
      return addr;
    }
    Thread* thread = Thread::Current();
    // Wait for any GC tasks that are in progress.
    WaitForSweeperTasks(thread);
    addr = old_space_.TryAllocate(size, type);
    if (addr != 0) {
      return addr;
    }
    // All GC tasks finished without allocating successfully. Collect both
    // generations.
    CollectMostGarbage();
    addr = old_space_.TryAllocate(size, type);
    if (addr != 0) {
      return addr;
    }
    // Wait for all of the concurrent tasks to finish before giving up.
    WaitForSweeperTasks(thread);
    addr = old_space_.TryAllocate(size, type);
    if (addr != 0) {
      return addr;
    }
    // Force growth before attempting another synchronous GC.
    addr = old_space_.TryAllocate(size, type, PageSpace::kForceGrowth);
    if (addr != 0) {
      return addr;
    }
    // Before throwing an out-of-memory error try a synchronous GC.
    CollectAllGarbage(kLowMemory);
    WaitForSweeperTasks(thread);
  }
  uword addr = old_space_.TryAllocate(size, type, PageSpace::kForceGrowth);
  if (addr != 0) {
    return addr;
  }
  // Give up allocating this object.
  OS::PrintErr("Exhausted heap space, trying to allocate %" Pd " bytes.\n",
               size);
  return 0;
}

void Heap::AllocatedExternal(intptr_t size, Space space) {
  ASSERT(Thread::Current()->no_safepoint_scope_depth() == 0);
  if (space == kNew) {
    Isolate::Current()->AssertCurrentThreadIsMutator();
    new_space_.AllocatedExternal(size);
    if (new_space_.ExternalInWords() <= (4 * new_space_.CapacityInWords())) {
      return;
    }
    // Attempt to free some external allocation by a scavenge. (If the total
    // remains above the limit, next external alloc will trigger another.)
    CollectGarbage(kScavenge, kExternal);
    // Promotion may have pushed old space over its limit. Fall through for old
    // space GC check.
  } else {
    ASSERT(space == kOld);
    old_space_.AllocatedExternal(size);
  }

  if (old_space_.ReachedHardThreshold()) {
    CollectGarbage(kMarkSweep, kExternal);
  } else {
    CheckStartConcurrentMarking(Thread::Current(), kExternal);
  }
}

void Heap::FreedExternal(intptr_t size, Space space) {
  if (space == kNew) {
    new_space_.FreedExternal(size);
  } else {
    ASSERT(space == kOld);
    old_space_.FreedExternal(size);
  }
}

void Heap::PromotedExternal(intptr_t size) {
  new_space_.FreedExternal(size);
  old_space_.AllocatedExternal(size);
}

bool Heap::Contains(uword addr) const {
  return new_space_.Contains(addr) || old_space_.Contains(addr);
}

bool Heap::NewContains(uword addr) const {
  return new_space_.Contains(addr);
}

bool Heap::OldContains(uword addr) const {
  return old_space_.Contains(addr);
}

bool Heap::CodeContains(uword addr) const {
  return old_space_.Contains(addr, OldPage::kExecutable);
}

bool Heap::DataContains(uword addr) const {
  return old_space_.DataContains(addr);
}

void Heap::VisitObjects(ObjectVisitor* visitor) {
  new_space_.VisitObjects(visitor);
  old_space_.VisitObjects(visitor);
}

void Heap::VisitObjectsNoImagePages(ObjectVisitor* visitor) {
  new_space_.VisitObjects(visitor);
  old_space_.VisitObjectsNoImagePages(visitor);
}

void Heap::VisitObjectsImagePages(ObjectVisitor* visitor) const {
  old_space_.VisitObjectsImagePages(visitor);
}

HeapIterationScope::HeapIterationScope(Thread* thread, bool writable)
    : ThreadStackResource(thread),
      heap_(isolate()->heap()),
      old_space_(heap_->old_space()),
      writable_(writable) {
  isolate()->safepoint_handler()->SafepointThreads(thread);

  {
    // It's not safe to iterate over old space when concurrent marking or
    // sweeping is in progress, or another thread is iterating the heap, so wait
    // for any such task to complete first.
    MonitorLocker ml(old_space_->tasks_lock());
#if defined(DEBUG)
    // We currently don't support nesting of HeapIterationScopes.
    ASSERT(old_space_->iterating_thread_ != thread);
#endif
    while ((old_space_->tasks() > 0) ||
           (old_space_->phase() != PageSpace::kDone)) {
      if (old_space_->phase() == PageSpace::kAwaitingFinalization) {
        ml.Exit();
        heap_->CollectOldSpaceGarbage(thread, Heap::kMarkSweep,
                                      Heap::kFinalize);
        ml.Enter();
      }
      while (old_space_->tasks() > 0) {
        ml.Wait();
      }
    }
#if defined(DEBUG)
    ASSERT(old_space_->iterating_thread_ == NULL);
    old_space_->iterating_thread_ = thread;
#endif
    old_space_->set_tasks(1);
  }

  if (writable_) {
    heap_->WriteProtectCode(false);
  }
}

HeapIterationScope::~HeapIterationScope() {
  if (writable_) {
    heap_->WriteProtectCode(true);
  }

  {
    MonitorLocker ml(old_space_->tasks_lock());
#if defined(DEBUG)
    ASSERT(old_space_->iterating_thread_ == thread());
    old_space_->iterating_thread_ = NULL;
#endif
    ASSERT(old_space_->tasks() == 1);
    old_space_->set_tasks(0);
    ml.NotifyAll();
  }

  isolate()->safepoint_handler()->ResumeThreads(thread());
}

void HeapIterationScope::IterateObjects(ObjectVisitor* visitor) const {
  heap_->VisitObjects(visitor);
}

void HeapIterationScope::IterateObjectsNoImagePages(
    ObjectVisitor* visitor) const {
  heap_->new_space()->VisitObjects(visitor);
  heap_->old_space()->VisitObjectsNoImagePages(visitor);
}

void HeapIterationScope::IterateOldObjects(ObjectVisitor* visitor) const {
  old_space_->VisitObjects(visitor);
}

void HeapIterationScope::IterateOldObjectsNoImagePages(
    ObjectVisitor* visitor) const {
  old_space_->VisitObjectsNoImagePages(visitor);
}

void HeapIterationScope::IterateVMIsolateObjects(ObjectVisitor* visitor) const {
  Dart::vm_isolate()->heap()->VisitObjects(visitor);
}

void HeapIterationScope::IterateObjectPointers(
    ObjectPointerVisitor* visitor,
    ValidationPolicy validate_frames) {
  isolate_group()->VisitObjectPointers(visitor, validate_frames);
}

void HeapIterationScope::IterateStackPointers(
    ObjectPointerVisitor* visitor,
    ValidationPolicy validate_frames) {
  isolate_group()->VisitStackPointers(visitor, validate_frames);
}

void Heap::VisitObjectPointers(ObjectPointerVisitor* visitor) {
  new_space_.VisitObjectPointers(visitor);
  old_space_.VisitObjectPointers(visitor);
}

InstructionsPtr Heap::FindObjectInCodeSpace(FindObjectVisitor* visitor) const {
  // Only executable pages can have RawInstructions objects.
  ObjectPtr raw_obj = old_space_.FindObject(visitor, OldPage::kExecutable);
  ASSERT((raw_obj == Object::null()) ||
         (raw_obj->GetClassId() == kInstructionsCid));
  return static_cast<InstructionsPtr>(raw_obj);
}

ObjectPtr Heap::FindOldObject(FindObjectVisitor* visitor) const {
  return old_space_.FindObject(visitor, OldPage::kData);
}

ObjectPtr Heap::FindNewObject(FindObjectVisitor* visitor) {
  return new_space_.FindObject(visitor);
}

ObjectPtr Heap::FindObject(FindObjectVisitor* visitor) {
  // The visitor must not allocate from the heap.
  NoSafepointScope no_safepoint_scope;
  ObjectPtr raw_obj = FindNewObject(visitor);
  if (raw_obj != Object::null()) {
    return raw_obj;
  }
  raw_obj = FindOldObject(visitor);
  if (raw_obj != Object::null()) {
    return raw_obj;
  }
  raw_obj = FindObjectInCodeSpace(visitor);
  return raw_obj;
}

void Heap::HintFreed(intptr_t size) {
  old_space_.HintFreed(size);
}

void Heap::NotifyIdle(int64_t deadline) {
  Thread* thread = Thread::Current();
  SafepointOperationScope safepoint_operation(thread);

  // Check if we want to collect new-space first, because if we want to collect
  // both new-space and old-space, the new-space collection should run first
  // to shrink the root set (make old-space GC faster) and avoid
  // intergenerational garbage (make old-space GC free more memory).
  if (new_space_.ShouldPerformIdleScavenge(deadline)) {
    TIMELINE_FUNCTION_GC_DURATION(thread, "IdleGC");
    CollectNewSpaceGarbage(thread, kIdle);
  }

  // Check if we want to collect old-space, in decreasing order of cost.
  // Because we use a deadline instead of a timeout, we automatically take any
  // time used up by a scavenge into account when deciding if we can complete
  // a mark-sweep on time.
  if (old_space_.ShouldPerformIdleMarkCompact(deadline)) {
    // We prefer mark-compact over other old space GCs if we have enough time,
    // since it removes old space fragmentation and frees up most memory.
    // Blocks for O(heap), roughtly twice as costly as mark-sweep.
    TIMELINE_FUNCTION_GC_DURATION(thread, "IdleGC");
    CollectOldSpaceGarbage(thread, kMarkCompact, kIdle);
  } else if (old_space_.ReachedHardThreshold()) {
    // Even though the following GC may exceed our idle deadline, we need to
    // ensure than that promotions during idle scavenges do not lead to
    // unbounded growth of old space. If a program is allocating only in new
    // space and all scavenges happen during idle time, then NotifyIdle will be
    // the only place that checks the old space allocation limit.
    // Compare the tail end of Heap::CollectNewSpaceGarbage.
    // Blocks for O(heap).
    TIMELINE_FUNCTION_GC_DURATION(thread, "IdleGC");
    CollectOldSpaceGarbage(thread, kMarkSweep, kIdle);
  } else if (old_space_.ShouldStartIdleMarkSweep(deadline) ||
             old_space_.ReachedSoftThreshold()) {
    // If we have both work to do and enough time, start or finish GC.
    // If we have crossed the soft threshold, ignore time; the next old-space
    // allocation will trigger this work anyway, so we try to pay at least some
    // of that cost with idle time.
    // Blocks for O(roots).
    PageSpace::Phase phase;
    {
      MonitorLocker ml(old_space_.tasks_lock());
      phase = old_space_.phase();
    }
    if (phase == PageSpace::kAwaitingFinalization) {
      TIMELINE_FUNCTION_GC_DURATION(thread, "IdleGC");
      CollectOldSpaceGarbage(thread, Heap::kMarkSweep, Heap::kFinalize);
    } else if (phase == PageSpace::kDone) {
      TIMELINE_FUNCTION_GC_DURATION(thread, "IdleGC");
      StartConcurrentMarking(thread);
    }
  }
}

void Heap::NotifyLowMemory() {
  CollectMostGarbage(kLowMemory);
}

void Heap::EvacuateNewSpace(Thread* thread, GCReason reason) {
  ASSERT((reason != kOldSpace) && (reason != kPromotion));
  if (thread->isolate_group() == Dart::vm_isolate()->group()) {
    // The vm isolate cannot safely collect garbage due to unvisited read-only
    // handles and slots bootstrapped with RAW_NULL. Ignore GC requests to
    // trigger a nice out-of-memory message instead of a crash in the middle of
    // visiting pointers.
    return;
  }
  {
    SafepointOperationScope safepoint_operation(thread);
    RecordBeforeGC(kScavenge, reason);
    VMTagScope tagScope(thread, reason == kIdle ? VMTag::kGCIdleTagId
                                                : VMTag::kGCNewSpaceTagId);
    TIMELINE_FUNCTION_GC_DURATION(thread, "EvacuateNewGeneration");
    new_space_.Evacuate();
    RecordAfterGC(kScavenge);
    PrintStats();
    NOT_IN_PRODUCT(PrintStatsToTimeline(&tbes, reason));
    last_gc_was_old_space_ = false;
  }
}

void Heap::CollectNewSpaceGarbage(Thread* thread, GCReason reason) {
  NoActiveIsolateScope no_active_isolate_scope;
  ASSERT((reason != kOldSpace) && (reason != kPromotion));
  if (thread->isolate_group() == Dart::vm_isolate()->group()) {
    // The vm isolate cannot safely collect garbage due to unvisited read-only
    // handles and slots bootstrapped with RAW_NULL. Ignore GC requests to
    // trigger a nice out-of-memory message instead of a crash in the middle of
    // visiting pointers.
    return;
  }
  {
    SafepointOperationScope safepoint_operation(thread);
    RecordBeforeGC(kScavenge, reason);
    {
      VMTagScope tagScope(thread, reason == kIdle ? VMTag::kGCIdleTagId
                                                  : VMTag::kGCNewSpaceTagId);
      TIMELINE_FUNCTION_GC_DURATION_BASIC(thread, "CollectNewGeneration");
      new_space_.Scavenge();
      RecordAfterGC(kScavenge);
      PrintStats();
      NOT_IN_PRODUCT(PrintStatsToTimeline(&tbes, reason));
      last_gc_was_old_space_ = false;
    }
    if (reason == kNewSpace) {
      if (old_space_.ReachedHardThreshold()) {
        CollectOldSpaceGarbage(thread, kMarkSweep, kPromotion);
      } else {
        CheckStartConcurrentMarking(thread, kPromotion);
      }
    }
  }
}

void Heap::CollectOldSpaceGarbage(Thread* thread,
                                  GCType type,
                                  GCReason reason) {
  NoActiveIsolateScope no_active_isolate_scope;

  ASSERT(reason != kNewSpace);
  ASSERT(type != kScavenge);
  if (FLAG_use_compactor) {
    type = kMarkCompact;
  }
  if (thread->isolate_group() == Dart::vm_isolate()->group()) {
    // The vm isolate cannot safely collect garbage due to unvisited read-only
    // handles and slots bootstrapped with RAW_NULL. Ignore GC requests to
    // trigger a nice out-of-memory message instead of a crash in the middle of
    // visiting pointers.
    return;
  }
  {
    SafepointOperationScope safepoint_operation(thread);
    thread->isolate_group()->ForEachIsolate(
        [&](Isolate* isolate) {
          // Discard regexp backtracking stacks to further reduce memory usage.
          isolate->CacheRegexpBacktrackStack(nullptr);
        },
        /*at_safepoint=*/true);

    RecordBeforeGC(type, reason);
    VMTagScope tagScope(thread, reason == kIdle ? VMTag::kGCIdleTagId
                                                : VMTag::kGCOldSpaceTagId);
    TIMELINE_FUNCTION_GC_DURATION_BASIC(thread, "CollectOldGeneration");
    old_space_.CollectGarbage(type == kMarkCompact, true /* finish */);
    RecordAfterGC(type);
    PrintStats();
    NOT_IN_PRODUCT(PrintStatsToTimeline(&tbes, reason));

    // Some Code objects may have been collected so invalidate handler cache.
    thread->isolate_group()->ForEachIsolate(
        [&](Isolate* isolate) {
          isolate->handler_info_cache()->Clear();
          isolate->catch_entry_moves_cache()->Clear();
        },
        /*at_safepoint=*/true);
    last_gc_was_old_space_ = true;
    assume_scavenge_will_fail_ = false;
  }
}

void Heap::CollectGarbage(GCType type, GCReason reason) {
  Thread* thread = Thread::Current();
  switch (type) {
    case kScavenge:
      CollectNewSpaceGarbage(thread, reason);
      break;
    case kMarkSweep:
    case kMarkCompact:
      CollectOldSpaceGarbage(thread, type, reason);
      break;
    default:
      UNREACHABLE();
  }
}

void Heap::CollectGarbage(Space space) {
  Thread* thread = Thread::Current();
  if (space == kOld) {
    CollectOldSpaceGarbage(thread, kMarkSweep, kOldSpace);
  } else {
    ASSERT(space == kNew);
    CollectNewSpaceGarbage(thread, kNewSpace);
  }
}

void Heap::CollectMostGarbage(GCReason reason) {
  Thread* thread = Thread::Current();
  CollectNewSpaceGarbage(thread, reason);
  CollectOldSpaceGarbage(
      thread, reason == kLowMemory ? kMarkCompact : kMarkSweep, reason);
}

void Heap::CollectAllGarbage(GCReason reason) {
  Thread* thread = Thread::Current();

  // New space is evacuated so this GC will collect all dead objects
  // kept alive by a cross-generational pointer.
  EvacuateNewSpace(thread, reason);
  if (thread->is_marking()) {
    // If incremental marking is happening, we need to finish the GC cycle
    // and perform a follow-up GC to purge any "floating garbage" that may be
    // retained by the incremental barrier.
    CollectOldSpaceGarbage(thread, kMarkSweep, reason);
  }
  CollectOldSpaceGarbage(
      thread, reason == kLowMemory ? kMarkCompact : kMarkSweep, reason);
  WaitForSweeperTasks(thread);
}

void Heap::CheckStartConcurrentMarking(Thread* thread, GCReason reason) {
  {
    MonitorLocker ml(old_space_.tasks_lock());
    if (old_space_.phase() != PageSpace::kDone) {
      return;  // Busy.
    }
  }

  if (old_space_.ReachedSoftThreshold()) {
    // New-space objects are roots during old-space GC. This means that even
    // unreachable new-space objects prevent old-space objects they reference
    // from being collected during an old-space GC. Normally this is not an
    // issue because new-space GCs run much more frequently than old-space GCs.
    // If new-space allocation is low and direct old-space allocation is high,
    // which can happen in a program that allocates large objects and little
    // else, old-space can fill up with unreachable objects until the next
    // new-space GC. This check is the concurrent-marking equivalent to the
    // new-space GC before synchronous-marking in CollectMostGarbage.
    if (last_gc_was_old_space_) {
      CollectNewSpaceGarbage(thread, kFull);
    }

    StartConcurrentMarking(thread);
  }
}

void Heap::StartConcurrentMarking(Thread* thread) {
  TIMELINE_FUNCTION_GC_DURATION_BASIC(thread, "StartConcurrentMarking");
  old_space_.CollectGarbage(/*compact=*/false, /*finalize=*/false);
}

void Heap::CheckFinishConcurrentMarking(Thread* thread) {
  bool ready;
  {
    MonitorLocker ml(old_space_.tasks_lock());
    ready = old_space_.phase() == PageSpace::kAwaitingFinalization;
  }
  if (ready) {
    CollectOldSpaceGarbage(thread, Heap::kMarkSweep, Heap::kFinalize);
  }
}

void Heap::WaitForMarkerTasks(Thread* thread) {
  MonitorLocker ml(old_space_.tasks_lock());
  while ((old_space_.phase() == PageSpace::kMarking) ||
         (old_space_.phase() == PageSpace::kAwaitingFinalization)) {
    while (old_space_.phase() == PageSpace::kMarking) {
      ml.WaitWithSafepointCheck(thread);
    }
    if (old_space_.phase() == PageSpace::kAwaitingFinalization) {
      ml.Exit();
      CollectOldSpaceGarbage(thread, Heap::kMarkSweep, Heap::kFinalize);
      ml.Enter();
    }
  }
}

void Heap::WaitForSweeperTasks(Thread* thread) {
  ASSERT(!thread->IsAtSafepoint());
  MonitorLocker ml(old_space_.tasks_lock());
  while (old_space_.tasks() > 0) {
    ml.WaitWithSafepointCheck(thread);
  }
}

void Heap::WaitForSweeperTasksAtSafepoint(Thread* thread) {
  ASSERT(thread->IsAtSafepoint());
  MonitorLocker ml(old_space_.tasks_lock());
  while (old_space_.tasks() > 0) {
    ml.Wait();
  }
}

void Heap::UpdateGlobalMaxUsed() {
  ASSERT(isolate_group_ != NULL);
  // We are accessing the used in words count for both new and old space
  // without synchronizing. The value of this metric is approximate.
  isolate_group_->GetHeapGlobalUsedMaxMetric()->SetValue(
      (UsedInWords(Heap::kNew) * kWordSize) +
      (UsedInWords(Heap::kOld) * kWordSize));
}

void Heap::InitGrowthControl() {
  new_space_.InitGrowthControl();
  old_space_.InitGrowthControl();
}

void Heap::SetGrowthControlState(bool state) {
  new_space_.SetGrowthControlState(state);
  old_space_.SetGrowthControlState(state);
}

bool Heap::GrowthControlState() {
  ASSERT(new_space_.GrowthControlState() == old_space_.GrowthControlState());
  return old_space_.GrowthControlState();
}

void Heap::WriteProtect(bool read_only) {
  read_only_ = read_only;
  new_space_.WriteProtect(read_only);
  old_space_.WriteProtect(read_only);
}

void Heap::Init(IsolateGroup* isolate_group,
                intptr_t max_new_gen_words,
                intptr_t max_old_gen_words) {
  ASSERT(isolate_group->heap() == nullptr);
  std::unique_ptr<Heap> heap(
      new Heap(isolate_group, max_new_gen_words, max_old_gen_words));
  isolate_group->set_heap(std::move(heap));
}

const char* Heap::RegionName(Space space) {
  switch (space) {
    case kNew:
      return "dart-newspace";
    case kOld:
      return "dart-oldspace";
    case kCode:
      return "dart-codespace";
    default:
      UNREACHABLE();
  }
}

void Heap::AddRegionsToObjectSet(ObjectSet* set) const {
  new_space_.AddRegionsToObjectSet(set);
  old_space_.AddRegionsToObjectSet(set);
  set->SortRegions();
}

void Heap::CollectOnNthAllocation(intptr_t num_allocations) {
  // Prevent generated code from using the TLAB fast path on next allocation.
  new_space_.AbandonRemainingTLABForDebugging(Thread::Current());
  gc_on_nth_allocation_ = num_allocations;
}

void Heap::MergeFrom(Heap* donor) {
  ASSERT(!donor->read_only_);
  ASSERT(donor->old_space()->tasks() == 0);

  new_space_.MergeFrom(donor->new_space());
  old_space_.MergeFrom(donor->old_space());

  for (intptr_t i = 0; i < kNumWeakSelectors; ++i) {
    // The new space rehashing should not be necessary.
    new_weak_tables_[i]->MergeFrom(donor->new_weak_tables_[i]);
    old_weak_tables_[i]->MergeFrom(donor->old_weak_tables_[i]);
  }

  StoreBufferBlock* block =
      donor->isolate_group()->store_buffer()->TakeBlocks();
  while (block != nullptr) {
    StoreBufferBlock* next = block->next();
    block->set_next(nullptr);
    isolate_group()->store_buffer()->PushBlock(block,
                                               StoreBuffer::kIgnoreThreshold);
    block = next;
  }
}

void Heap::CollectForDebugging() {
  if (gc_on_nth_allocation_ == kNoForcedGarbageCollection) return;
  if (Thread::Current()->IsAtSafepoint()) {
    // CollectAllGarbage is not supported when we are at a safepoint.
    // Allocating when at a safepoint is not a common case.
    return;
  }
  gc_on_nth_allocation_--;
  if (gc_on_nth_allocation_ == 0) {
    CollectAllGarbage(kDebugging);
    gc_on_nth_allocation_ = kNoForcedGarbageCollection;
  } else {
    // Prevent generated code from using the TLAB fast path on next allocation.
    new_space_.AbandonRemainingTLABForDebugging(Thread::Current());
  }
}

ObjectSet* Heap::CreateAllocatedObjectSet(Zone* zone,
                                          MarkExpectation mark_expectation) {
  ObjectSet* allocated_set = new (zone) ObjectSet(zone);

  this->AddRegionsToObjectSet(allocated_set);
  Isolate* vm_isolate = Dart::vm_isolate();
  vm_isolate->heap()->AddRegionsToObjectSet(allocated_set);

  {
    VerifyObjectVisitor object_visitor(isolate_group(), allocated_set,
                                       mark_expectation);
    this->VisitObjectsNoImagePages(&object_visitor);
  }
  {
    VerifyObjectVisitor object_visitor(isolate_group(), allocated_set,
                                       kRequireMarked);
    this->VisitObjectsImagePages(&object_visitor);
  }
  {
    // VM isolate heap is premarked.
    VerifyObjectVisitor vm_object_visitor(isolate_group(), allocated_set,
                                          kRequireMarked);
    vm_isolate->heap()->VisitObjects(&vm_object_visitor);
  }

  return allocated_set;
}

bool Heap::Verify(MarkExpectation mark_expectation) {
  if (FLAG_disable_heap_verification) {
    return true;
  }
  HeapIterationScope heap_iteration_scope(Thread::Current());
  return VerifyGC(mark_expectation);
}

bool Heap::VerifyGC(MarkExpectation mark_expectation) {
  auto thread = Thread::Current();
  StackZone stack_zone(thread);

  ObjectSet* allocated_set =
      CreateAllocatedObjectSet(stack_zone.GetZone(), mark_expectation);
  VerifyPointersVisitor visitor(isolate_group(), allocated_set);
  VisitObjectPointers(&visitor);

  // Only returning a value so that Heap::Validate can be called from an ASSERT.
  return true;
}

void Heap::PrintSizes() const {
  OS::PrintErr(
      "New space (%" Pd64 "k of %" Pd64
      "k) "
      "Old space (%" Pd64 "k of %" Pd64 "k)\n",
      (UsedInWords(kNew) / KBInWords), (CapacityInWords(kNew) / KBInWords),
      (UsedInWords(kOld) / KBInWords), (CapacityInWords(kOld) / KBInWords));
}

int64_t Heap::UsedInWords(Space space) const {
  return space == kNew ? new_space_.UsedInWords() : old_space_.UsedInWords();
}

int64_t Heap::CapacityInWords(Space space) const {
  return space == kNew ? new_space_.CapacityInWords()
                       : old_space_.CapacityInWords();
}

int64_t Heap::ExternalInWords(Space space) const {
  return space == kNew ? new_space_.ExternalInWords()
                       : old_space_.ExternalInWords();
}

int64_t Heap::TotalUsedInWords() const {
  return UsedInWords(kNew) + UsedInWords(kOld);
}

int64_t Heap::TotalCapacityInWords() const {
  return CapacityInWords(kNew) + CapacityInWords(kOld);
}

int64_t Heap::TotalExternalInWords() const {
  return ExternalInWords(kNew) + ExternalInWords(kOld);
}

int64_t Heap::GCTimeInMicros(Space space) const {
  if (space == kNew) {
    return new_space_.gc_time_micros();
  }
  return old_space_.gc_time_micros();
}

intptr_t Heap::Collections(Space space) const {
  if (space == kNew) {
    return new_space_.collections();
  }
  return old_space_.collections();
}

const char* Heap::GCTypeToString(GCType type) {
  switch (type) {
    case kScavenge:
      return "Scavenge";
    case kMarkSweep:
      return "MarkSweep";
    case kMarkCompact:
      return "MarkCompact";
    default:
      UNREACHABLE();
      return "";
  }
}

const char* Heap::GCReasonToString(GCReason gc_reason) {
  switch (gc_reason) {
    case kNewSpace:
      return "new space";
    case kPromotion:
      return "promotion";
    case kOldSpace:
      return "old space";
    case kFinalize:
      return "finalize";
    case kFull:
      return "full";
    case kExternal:
      return "external";
    case kIdle:
      return "idle";
    case kLowMemory:
      return "low memory";
    case kDebugging:
      return "debugging";
    case kSendAndExit:
      return "send_and_exit";
    default:
      UNREACHABLE();
      return "";
  }
}

int64_t Heap::PeerCount() const {
  return new_weak_tables_[kPeers]->count() + old_weak_tables_[kPeers]->count();
}

void Heap::ResetCanonicalHashTable() {
  new_weak_tables_[kCanonicalHashes]->Reset();
  old_weak_tables_[kCanonicalHashes]->Reset();
}

void Heap::ResetObjectIdTable() {
  new_weak_tables_[kObjectIds]->Reset();
  old_weak_tables_[kObjectIds]->Reset();
}

intptr_t Heap::GetWeakEntry(ObjectPtr raw_obj, WeakSelector sel) const {
  if (raw_obj->IsNewObject()) {
    return new_weak_tables_[sel]->GetValue(raw_obj);
  }
  ASSERT(raw_obj->IsOldObject());
  return old_weak_tables_[sel]->GetValue(raw_obj);
}

void Heap::SetWeakEntry(ObjectPtr raw_obj, WeakSelector sel, intptr_t val) {
  if (raw_obj->IsNewObject()) {
    new_weak_tables_[sel]->SetValue(raw_obj, val);
  } else {
    ASSERT(raw_obj->IsOldObject());
    old_weak_tables_[sel]->SetValue(raw_obj, val);
  }
}

void Heap::ForwardWeakEntries(ObjectPtr before_object, ObjectPtr after_object) {
  const auto before_space =
      before_object->IsNewObject() ? Heap::kNew : Heap::kOld;
  const auto after_space =
      after_object->IsNewObject() ? Heap::kNew : Heap::kOld;

  for (int sel = 0; sel < Heap::kNumWeakSelectors; sel++) {
    const auto selector = static_cast<Heap::WeakSelector>(sel);
    auto before_table = GetWeakTable(before_space, selector);
    intptr_t entry = before_table->RemoveValueExclusive(before_object);
    if (entry != 0) {
      auto after_table = GetWeakTable(after_space, selector);
      after_table->SetValueExclusive(after_object, entry);
    }
  }

  // We only come here during hot reload, in which case we assume that none of
  // the isolates is in the middle of sending messages.
  isolate_group()->ForEachIsolate(
      [&](Isolate* isolate) {
        RELEASE_ASSERT(isolate->forward_table_new() == nullptr);
        RELEASE_ASSERT(isolate->forward_table_old() == nullptr);
      },
      /*at_safepoint=*/true);
}

void Heap::ForwardWeakTables(ObjectPointerVisitor* visitor) {
  // NOTE: This method is only used by the compactor, so there is no need to
  // process the `Heap::kNew` tables.
  for (int sel = 0; sel < Heap::kNumWeakSelectors; sel++) {
    WeakSelector selector = static_cast<Heap::WeakSelector>(sel);
    GetWeakTable(Heap::kOld, selector)->Forward(visitor);
  }

  // Isolates might have forwarding tables (used for during snapshoting in
  // isolate communication).
  isolate_group()->ForEachIsolate(
      [&](Isolate* isolate) {
        auto table_old = isolate->forward_table_old();
        if (table_old != nullptr) table_old->Forward(visitor);
      },
      /*at_safepoint=*/true);
}

#ifndef PRODUCT
void Heap::PrintToJSONObject(Space space, JSONObject* object) const {
  if (space == kNew) {
    new_space_.PrintToJSONObject(object);
  } else {
    old_space_.PrintToJSONObject(object);
  }
}

void Heap::PrintMemoryUsageJSON(JSONStream* stream) const {
  JSONObject obj(stream);
  PrintMemoryUsageJSON(&obj);
}

void Heap::PrintMemoryUsageJSON(JSONObject* jsobj) const {
  jsobj->AddProperty("type", "MemoryUsage");
  jsobj->AddProperty64("heapUsage", TotalUsedInWords() * kWordSize);
  jsobj->AddProperty64("heapCapacity", TotalCapacityInWords() * kWordSize);
  jsobj->AddProperty64("externalUsage", TotalExternalInWords() * kWordSize);
}
#endif  // PRODUCT

void Heap::RecordBeforeGC(GCType type, GCReason reason) {
  stats_.num_++;
  stats_.type_ = type;
  stats_.reason_ = reason;
  stats_.before_.micros_ = OS::GetCurrentMonotonicMicros();
  stats_.before_.new_ = new_space_.GetCurrentUsage();
  stats_.before_.old_ = old_space_.GetCurrentUsage();
  for (int i = 0; i < GCStats::kTimeEntries; i++)
    stats_.times_[i] = 0;
  for (int i = 0; i < GCStats::kDataEntries; i++)
    stats_.data_[i] = 0;
}

static double AvgCollectionPeriod(int64_t run_time, intptr_t collections) {
  if (collections <= 0 || run_time <= 0) {
    return 0.0;
  }
  return MicrosecondsToMilliseconds(run_time) /
         static_cast<double>(collections);
}

void Heap::RecordAfterGC(GCType type) {
  stats_.after_.micros_ = OS::GetCurrentMonotonicMicros();
  int64_t delta = stats_.after_.micros_ - stats_.before_.micros_;
  if (stats_.type_ == kScavenge) {
    new_space_.AddGCTime(delta);
    new_space_.IncrementCollections();
  } else {
    old_space_.AddGCTime(delta);
    old_space_.IncrementCollections();
  }
  stats_.after_.new_ = new_space_.GetCurrentUsage();
  stats_.after_.old_ = old_space_.GetCurrentUsage();
#ifndef PRODUCT
  // For now we'll emit the same GC events on all isolates.
  if (Service::gc_stream.enabled()) {
    isolate_group_->ForEachIsolate([&](Isolate* isolate) {
      if (!Isolate::IsSystemIsolate(isolate)) {
        ServiceEvent event(isolate, ServiceEvent::kGC);
        event.set_gc_stats(&stats_);
        Service::HandleEvent(&event);
      }
    });
  }
#endif  // !PRODUCT
  if (Dart::gc_event_callback() != nullptr) {
    isolate_group_->ForEachIsolate([&](Isolate* isolate) {
      if (!Isolate::IsSystemIsolate(isolate)) {
        Dart_GCEvent event;
        auto isolate_id = Utils::CStringUniquePtr(
            OS::SCreate(nullptr, ISOLATE_SERVICE_ID_FORMAT_STRING,
                        isolate->main_port()),
            std::free);
        int64_t isolate_uptime_micros = isolate->UptimeMicros();

        event.isolate_id = isolate_id.get();
        event.type = GCTypeToString(stats_.type_);
        event.reason = GCReasonToString(stats_.reason_);

        // New space - Scavenger.
        {
          intptr_t new_space_collections = new_space_.collections();

          event.new_space.collections = new_space_collections;
          event.new_space.used = stats_.after_.new_.used_in_words * kWordSize;
          event.new_space.capacity =
              stats_.after_.new_.capacity_in_words * kWordSize;
          event.new_space.external =
              stats_.after_.new_.external_in_words * kWordSize;
          event.new_space.time =
              MicrosecondsToSeconds(new_space_.gc_time_micros());
          event.new_space.avg_collection_period =
              AvgCollectionPeriod(isolate_uptime_micros, new_space_collections);
        }

        // Old space - Page.
        {
          intptr_t old_space_collections = old_space_.collections();

          event.old_space.collections = old_space_collections;
          event.old_space.used = stats_.after_.old_.used_in_words * kWordSize;
          event.old_space.capacity =
              stats_.after_.old_.capacity_in_words * kWordSize;
          event.old_space.external =
              stats_.after_.old_.external_in_words * kWordSize;
          event.old_space.time =
              MicrosecondsToSeconds(old_space_.gc_time_micros());
          event.old_space.avg_collection_period =
              AvgCollectionPeriod(isolate_uptime_micros, old_space_collections);
        }

        (*Dart::gc_event_callback())(&event);
      }
    });
  }
}

void Heap::PrintStats() {
#if !defined(PRODUCT)
  if (!FLAG_verbose_gc) return;

  if ((FLAG_verbose_gc_hdr != 0) &&
      (((stats_.num_ - 1) % FLAG_verbose_gc_hdr) == 0)) {
    OS::PrintErr(
        "[              |                      |     |       |      "
        "| new gen     | new gen     | new gen "
        "| old gen       | old gen       | old gen     "
        "| sweep | safe- | roots/| stbuf/| tospc/| weaks/|               ]\n"
        "[ GC isolate   | space (reason)       | GC# | start | time "
        "| used (kB)   | capacity kB | external"
        "| used (kB)     | capacity (kB) | external kB "
        "| thread| point |marking| reset | sweep |swplrge| data          ]\n"
        "[              |                      |     |  (s)  | (ms) "
        "|before| after|before| after| b4 |aftr"
        "| before| after | before| after |before| after"
        "| (ms)  | (ms)  | (ms)  | (ms)  | (ms)  | (ms)  |               ]\n");
  }

  // clang-format off
  OS::PrintErr(
    "[ %-13.13s, %10s(%9s), "  // GC(isolate-group), type(reason)
    "%4" Pd ", "  // count
    "%6.2f, "  // start time
    "%5.1f, "  // total time
    "%5" Pd ", %5" Pd ", "  // new gen: in use before/after
    "%5" Pd ", %5" Pd ", "  // new gen: capacity before/after
    "%3" Pd ", %3" Pd ", "  // new gen: external before/after
    "%6" Pd ", %6" Pd ", "  // old gen: in use before/after
    "%6" Pd ", %6" Pd ", "  // old gen: capacity before/after
    "%5" Pd ", %5" Pd ", "  // old gen: external before/after
    "%6.2f, %6.2f, %6.2f, %6.2f, %6.2f, %6.2f, "  // times
    "%" Pd ", %" Pd ", %" Pd ", %" Pd ", "  // data
    "]\n",  // End with a comma to make it easier to import in spreadsheets.
    isolate_group()->source()->name,
    GCTypeToString(stats_.type_),
    GCReasonToString(stats_.reason_),
    stats_.num_,
    MicrosecondsToSeconds(isolate_group_->UptimeMicros()),
    MicrosecondsToMilliseconds(stats_.after_.micros_ -
                               stats_.before_.micros_),
    RoundWordsToKB(stats_.before_.new_.used_in_words),
    RoundWordsToKB(stats_.after_.new_.used_in_words),
    RoundWordsToKB(stats_.before_.new_.capacity_in_words),
    RoundWordsToKB(stats_.after_.new_.capacity_in_words),
    RoundWordsToKB(stats_.before_.new_.external_in_words),
    RoundWordsToKB(stats_.after_.new_.external_in_words),
    RoundWordsToKB(stats_.before_.old_.used_in_words),
    RoundWordsToKB(stats_.after_.old_.used_in_words),
    RoundWordsToKB(stats_.before_.old_.capacity_in_words),
    RoundWordsToKB(stats_.after_.old_.capacity_in_words),
    RoundWordsToKB(stats_.before_.old_.external_in_words),
    RoundWordsToKB(stats_.after_.old_.external_in_words),
    MicrosecondsToMilliseconds(stats_.times_[0]),
    MicrosecondsToMilliseconds(stats_.times_[1]),
    MicrosecondsToMilliseconds(stats_.times_[2]),
    MicrosecondsToMilliseconds(stats_.times_[3]),
    MicrosecondsToMilliseconds(stats_.times_[4]),
    MicrosecondsToMilliseconds(stats_.times_[5]),
    stats_.data_[0],
    stats_.data_[1],
    stats_.data_[2],
    stats_.data_[3]);
  // clang-format on
#endif  // !defined(PRODUCT)
}

void Heap::PrintStatsToTimeline(TimelineEventScope* event, GCReason reason) {
#if !defined(PRODUCT)
  if ((event == NULL) || !event->enabled()) {
    return;
  }
  intptr_t arguments = event->GetNumArguments();
  event->SetNumArguments(arguments + 13);
  event->CopyArgument(arguments + 0, "Reason", GCReasonToString(reason));
  event->FormatArgument(arguments + 1, "Before.New.Used (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.new_.used_in_words));
  event->FormatArgument(arguments + 2, "After.New.Used (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.new_.used_in_words));
  event->FormatArgument(arguments + 3, "Before.Old.Used (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.old_.used_in_words));
  event->FormatArgument(arguments + 4, "After.Old.Used (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.old_.used_in_words));

  event->FormatArgument(arguments + 5, "Before.New.Capacity (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.new_.capacity_in_words));
  event->FormatArgument(arguments + 6, "After.New.Capacity (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.new_.capacity_in_words));
  event->FormatArgument(arguments + 7, "Before.Old.Capacity (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.old_.capacity_in_words));
  event->FormatArgument(arguments + 8, "After.Old.Capacity (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.old_.capacity_in_words));

  event->FormatArgument(arguments + 9, "Before.New.External (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.new_.external_in_words));
  event->FormatArgument(arguments + 10, "After.New.External (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.new_.external_in_words));
  event->FormatArgument(arguments + 11, "Before.Old.External (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.before_.old_.external_in_words));
  event->FormatArgument(arguments + 12, "After.Old.External (kB)", "%" Pd "",
                        RoundWordsToKB(stats_.after_.old_.external_in_words));
#endif  // !defined(PRODUCT)
}

Heap::Space Heap::SpaceForExternal(intptr_t size) const {
  // If 'size' would be a significant fraction of new space, then use old.
  static const int kExtNewRatio = 16;
  if (size > (CapacityInWords(Heap::kNew) * kWordSize) / kExtNewRatio) {
    return Heap::kOld;
  } else {
    return Heap::kNew;
  }
}

NoHeapGrowthControlScope::NoHeapGrowthControlScope()
    : ThreadStackResource(Thread::Current()) {
  Heap* heap = isolate()->heap();
  current_growth_controller_state_ = heap->GrowthControlState();
  heap->DisableGrowthControl();
}

NoHeapGrowthControlScope::~NoHeapGrowthControlScope() {
  Heap* heap = isolate()->heap();
  heap->SetGrowthControlState(current_growth_controller_state_);
}

WritableVMIsolateScope::WritableVMIsolateScope(Thread* thread)
    : ThreadStackResource(thread) {
  if (FLAG_write_protect_code && FLAG_write_protect_vm_isolate) {
    Dart::vm_isolate()->heap()->WriteProtect(false);
  }
}

WritableVMIsolateScope::~WritableVMIsolateScope() {
  ASSERT(Dart::vm_isolate()->heap()->UsedInWords(Heap::kNew) == 0);
  if (FLAG_write_protect_code && FLAG_write_protect_vm_isolate) {
    Dart::vm_isolate()->heap()->WriteProtect(true);
  }
}

WritableCodePages::WritableCodePages(Thread* thread, Isolate* isolate)
    : StackResource(thread), isolate_(isolate) {
  isolate_->heap()->WriteProtectCode(false);
}

WritableCodePages::~WritableCodePages() {
  isolate_->heap()->WriteProtectCode(true);
}

}  // namespace dart
```

