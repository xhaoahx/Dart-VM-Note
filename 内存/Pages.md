# Pages

## pages.h

```c++
// ...

namespace dart {

DECLARE_FLAG(bool, write_protect_code);

// Forward declarations.
class Heap;
class JSONObject;
class ObjectPointerVisitor;
class ObjectSet;
class ForwardingPage;
class GCMarker;

// 老生代页面大小：512KB = 524288B = 4194304 bit = 65536 字
static constexpr intptr_t kOldPageSize = 512 * KB;
static constexpr intptr_t kOldPageSizeInWords = kOldPageSize / kWordSize;
// 页面掩码，即 ~(2^(10 + 9)-1) = 1111..1100...0 (46 个 1，18 个 0)
static constexpr intptr_t kOldPageMask = ~(kOldPageSize - 1);

static constexpr intptr_t kBitVectorWordsPerBlock = 1;
static constexpr intptr_t kBlockSize =
    kObjectAlignment * kBitsPerWord * kBitVectorWordsPerBlock;
static constexpr intptr_t kBlockMask = ~(kBlockSize - 1);
static constexpr intptr_t kBlocksPerPage = kOldPageSize / kBlockSize;

// 包含老生代的页面
class OldPage {
 public:
  enum PageType { 
      // 存储可执行的代码
      kExecutable = 0,
      // 存储数据
      kData 
  };

  // 下一个页面
  OldPage* next() const { return next_; }
  // 设置下一个页面
  void set_next(OldPage* next) { next_ = next; }

  // 是否包含某个地址
  bool Contains(uword addr) const { return memory_->Contains(addr); }
  intptr_t AliasOffset() const { return memory_->AliasOffset(); }

  uword object_start() const { return memory_->start() + ObjectStartOffset(); }
  uword object_end() const { return object_end_; }
  // 已使用的内存大小
  uword used_in_bytes() const { return used_in_bytes_; }
  void set_used_in_bytes(uword value) {
    ASSERT(Utils::IsAligned(value, kObjectAlignment));
    used_in_bytes_ = value;
  }

  ForwardingPage* forwarding_page() const { return forwarding_page_; }
  void AllocateForwardingPage();

  // 页面保存数据的类型
  PageType type() const { return type_; }

  bool is_image_page() const { return !memory_->vm_owns_region(); }

  // 访问对象
  void VisitObjects(ObjectVisitor* visitor) const;
  // 访问对象指针
  void VisitObjectPointers(ObjectPointerVisitor* visitor) const;
  // 找到某对象
  ObjectPtr FindObject(FindObjectVisitor* visitor) const;

  // 写保护
  void WriteProtect(bool read_only);

  // 对象始点偏移，64
  static intptr_t ObjectStartOffset() {
    return Utils::RoundUp(sizeof(OldPage), kMaxObjectAlignment);
  }

  // 对象指针所处的页
  static OldPage* Of(ObjectPtr obj) {
    ASSERT(obj->IsHeapObject());
    ASSERT(obj->IsOldObject());
    return reinterpret_cast<OldPage*>(static_cast<uword>(obj) & kOldPageMask);
  }

  // 某地址所处的页
  static OldPage* Of(uword addr) {
    return reinterpret_cast<OldPage*>(addr & kOldPageMask);
  }

  // 修改给定对象类型为 Executable
  static ObjectPtr ToExecutable(ObjectPtr obj) {
    OldPage* page = Of(obj);
    VirtualMemory* memory = page->memory_;
    const intptr_t alias_offset = memory->AliasOffset();
    if (alias_offset == 0) {
      return obj;  // Not aliased.
    }
    uword addr = ObjectLayout::ToAddr(obj);
    if (memory->Contains(addr)) {
      return ObjectLayout::FromAddr(addr + alias_offset);
    }
    // obj is executable.
    ASSERT(memory->ContainsAlias(addr));
    return obj;
  }

  // Warning: This does not work for objects on image pages.
  static ObjectPtr ToWritable(ObjectPtr obj) {
    OldPage* page = Of(obj);
    VirtualMemory* memory = page->memory_;
    const intptr_t alias_offset = memory->AliasOffset();
    if (alias_offset == 0) {
      return obj;  // Not aliased.
    }
    uword addr = ObjectLayout::ToAddr(obj);
    if (memory->ContainsAlias(addr)) {
      return ObjectLayout::FromAddr(addr - alias_offset);
    }
    // obj is writable.
    ASSERT(memory->Contains(addr));
    return obj;
  }

  // 每个卡片有 128 个插槽
  static const intptr_t kSlotsPerCardLog2 = 7;
  // 每个卡片有 2^(7 + 3) = 1024 Byte = 1KB
  static const intptr_t kBytesPerCardLog2 = kWordSizeLog2 + kSlotsPerCardLog2;

  // 卡表大小
  intptr_t card_table_size() const {
    return memory_->size() >> kBytesPerCardLog2;
  }

  // card_table_ 偏移
  static intptr_t card_table_offset() {
    return OFFSET_OF(OldPage, card_table_);
  }

  // 记忆某个指针
  void RememberCard(ObjectPtr const* slot) {
    // 必须包含这个指针的地址
    ASSERT(Contains(reinterpret_cast<uword>(slot)));
    // 创建卡表，会在此页销毁时被释放
    if (card_table_ == NULL) {
      card_table_ = reinterpret_cast<uint8_t*>(
          calloc(card_table_size(), sizeof(uint8_t)));
    }
    // 以偏移值作为 index
    intptr_t offset =
        reinterpret_cast<uword>(slot) - reinterpret_cast<uword>(this);
    intptr_t index = offset >> kBytesPerCardLog2; // offset >> 10
    ASSERT((index >= 0) && (index < card_table_size()));
    card_table_[index] = 1;
  }
  
  // 访问记忆卡
  void VisitRememberedCards(ObjectPointerVisitor* visitor);

 private:
  void set_object_end(uword value) {
    ASSERT((value & kObjectAlignmentMask) == kOldObjectAlignmentOffset);
    object_end_ = value;
  }
  
  // 创建新的 OldPage 
  // Out of Memory 时返回 NULL.
  static OldPage* Allocate(intptr_t size_in_words,
                           PageType type,
                           const char* name);

  // 释放此页面的虚拟内存。在调用后，指向此页的指针立刻无法访问
  void Deallocate();

  VirtualMemory* memory_;
  OldPage* next_;
  uword object_end_;
  uword used_in_bytes_;
  ForwardingPage* forwarding_page_;
  uint8_t* card_table_;  // Remembered set, not marking.
  PageType type_;

  friend class PageSpace;
  friend class GCCompactor;

  DISALLOW_ALLOCATION();
  DISALLOW_IMPLICIT_CONSTRUCTORS(OldPage);
};

// PageSpaceGarbageCollectionHistory 持有最近一次垃圾回收的时间信息
class PageSpaceGarbageCollectionHistory {
 public:
  PageSpaceGarbageCollectionHistory() {}
  ~PageSpaceGarbageCollectionHistory() {}

  void AddGarbageCollectionTime(int64_t start, int64_t end);

  int GarbageCollectionTimeFraction();

  bool IsEmpty() const { return history_.Size() == 0; }

 private:
  struct Entry {
    int64_t start;
    int64_t end;
  };
  static const intptr_t kHistoryLength = 4;
  RingBuffer<Entry, kHistoryLength> history_;

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(PageSpaceGarbageCollectionHistory);
};


// PageSpaceController 控制堆的大小
class PageSpaceController {
 public:
  // The heap is passed in for recording stats only. The controller does not
  // invoke GC by itself.
  PageSpaceController(Heap* heap,
                      int heap_growth_ratio,
                      int heap_growth_max,
                      int garbage_collection_time_ratio);
  ~PageSpaceController();

  // 返回但 SpaceUsage 达到 after 之后是否触发 GC
  // 此方法可用在分配内存的前后调用，且不会改变 Controller 的状态
  bool ReachedHardThreshold(SpaceUsage after) const;
  bool ReachedSoftThreshold(SpaceUsage after) const;

  // 返回当处于 Idel 状态时，GC 是否值得
  bool ReachedIdleThreshold(SpaceUsage current) const;

  // 在每一次垃圾回收之后调用来更新 controller 状态
  void EvaluateGarbageCollection(SpaceUsage before,
                                 SpaceUsage after,
                                 int64_t start,
                                 int64_t end);
  void EvaluateAfterLoading(SpaceUsage after);
  void HintFreed(intptr_t size);

  void set_last_usage(SpaceUsage current) { last_usage_ = current; }

  /// 状态控制
  void Enable() { is_enabled_ = true; }
  void Disable() { is_enabled_ = false; }
  bool is_enabled() { return is_enabled_; }

 private:
  friend class PageSpace;  // For MergeOtherPageSpaceController

  void RecordUpdate(SpaceUsage before, SpaceUsage after, const char* reason);
  void MergeFrom(PageSpaceController* donor);

  void RecordUpdate(SpaceUsage before,
                    SpaceUsage after,
                    intptr_t growth_in_pages,
                    const char* reason);

  Heap* heap_;

  bool is_enabled_;

  // Usage after last evaluated GC or last enabled.
  SpaceUsage last_usage_;

  // If the garbage collector was not able to free more than heap_growth_ratio_
  // memory, then the heap is grown. Otherwise garbage collection is performed.
  const int heap_growth_ratio_;

  // The desired percent of heap in-use after a garbage collection.
  // Equivalent to \frac{100-heap_growth_ratio_}{100}.
  const double desired_utilization_;

  // Max number of pages we grow.
  const int heap_growth_max_;

  // If the relative GC time goes above garbage_collection_time_ratio_ %,
  // we grow the heap more aggressively.
  const int garbage_collection_time_ratio_;

  // Perform a stop-the-world GC when usage exceeds this amount.
  intptr_t hard_gc_threshold_in_words_;

  // Begin concurrent marking when usage exceeds this amount.
  intptr_t soft_gc_threshold_in_words_;

  // Run idle GC if time permits when usage exceeds this amount.
  intptr_t idle_gc_threshold_in_words_;

  PageSpaceGarbageCollectionHistory history_;

  DISALLOW_IMPLICIT_CONSTRUCTORS(PageSpaceController);
};

/// Page 空间记录，被 Heap 持有用于管理 OldPages
class PageSpace {
 public:
  enum GrowthPolicy { kControlGrowth, kForceGrowth };
  // GC 状态
  enum Phase {
    // 完成
    kDone,
    // 正在标记
    kMarking,
    // 等待终结
    kAwaitingFinalization,
    // 清除 large 对象
    kSweepingLarge,
    // 清除 regular 对象
    kSweepingRegular
  };

  PageSpace(Heap* heap, intptr_t max_capacity_in_words);
  ~PageSpace();

  /// 尝试分配 size 大小的内存
  uword TryAllocate(intptr_t size,
                    OldPage::PageType type = OldPage::kData,
                    GrowthPolicy growth_policy = kControlGrowth) {
    /// 是否是保护模式
    bool is_protected =
        (type == OldPage::kExecutable) && FLAG_write_protect_code;
    bool is_locked = false;
    // 从空闲分区表上分配
    return TryAllocateInternal(size, &freelists_[type], type, growth_policy,
                               is_protected, is_locked);
  }

  /// 以下五个函数均委托为 PageSpaceController 实现 
  bool ReachedHardThreshold() const {
    return page_space_controller_.ReachedHardThreshold(usage_);
  }
  bool ReachedSoftThreshold() const {
    return page_space_controller_.ReachedSoftThreshold(usage_);
  }
  bool ReachedIdleThreshold() const {
    return page_space_controller_.ReachedIdleThreshold(usage_);
  }
  void EvaluateAfterLoading() {
    page_space_controller_.EvaluateAfterLoading(usage_);
  }
  void HintFreed(intptr_t size) { page_space_controller_.HintFreed(size); }

  /// 已经使用的的大小
  int64_t UsedInWords() const { return usage_.used_in_words; }
    
  // 总容量，互斥访问
  int64_t CapacityInWords() const {
    MutexLocker ml(&pages_lock_);
    return usage_.capacity_in_words;
  }
  // 增加容量
  void IncreaseCapacityInWords(intptr_t increase_in_words) {
    MutexLocker ml(&pages_lock_);
    IncreaseCapacityInWordsLocked(increase_in_words);
  }
  // 互斥增加容量
  void IncreaseCapacityInWordsLocked(intptr_t increase_in_words) {
    DEBUG_ASSERT(pages_lock_.IsOwnedByCurrentThread());
    usage_.capacity_in_words += increase_in_words;
    UpdateMaxCapacityLocked();
  }

  void UpdateMaxCapacityLocked();
  void UpdateMaxUsed();

  int64_t ExternalInWords() const { return usage_.external_in_words; }
    
  /// 获取当前容量
  SpaceUsage GetCurrentUsage() const {
    MutexLocker ml(&pages_lock_);
    return usage_;
  }
  int64_t ImageInWords() const {
    int64_t size = 0;
    MutexLocker ml(&pages_lock_);
    for (OldPage* page = image_pages_; page != nullptr; page = page->next()) {
      size += page->memory_->size();
    }
    return size >> kWordSizeLog2;
  }

  bool Contains(uword addr) const;
  bool ContainsUnsafe(uword addr) const;
  bool Contains(uword addr, OldPage::PageType type) const;
  bool DataContains(uword addr) const;
  bool IsValidAddress(uword addr) const { return Contains(addr); }

  void VisitObjects(ObjectVisitor* visitor) const;
  void VisitObjectsNoImagePages(ObjectVisitor* visitor) const;
  void VisitObjectsImagePages(ObjectVisitor* visitor) const;
  void VisitObjectPointers(ObjectPointerVisitor* visitor) const;

  void VisitRememberedCards(ObjectPointerVisitor* visitor) const;

  ObjectPtr FindObject(FindObjectVisitor* visitor,OldPage::PageType type) const;

  // 在此页面上使用 “标记-清理” 或者 “标记-整理” 算法进行 GC
  void CollectGarbage(bool compact, bool finalize);

  void AddRegionsToObjectSet(ObjectSet* set) const;

  void InitGrowthControl() {
    page_space_controller_.set_last_usage(usage_);
    page_space_controller_.Enable();
  }

  /// 设置增长控制
  void SetGrowthControlState(bool state) {
    if (state) {
      page_space_controller_.Enable();
    } else {
      page_space_controller_.Disable();
    }
  }

  /// 增长控制状态
  bool GrowthControlState() { return page_space_controller_.is_enabled(); }

  // Note: Code pages are made executable/non-executable when 'read_only' is
  // true/false, respectively.
  void WriteProtect(bool read_only);
  void WriteProtectCode(bool read_only);

  bool ShouldStartIdleMarkSweep(int64_t deadline);
  bool ShouldPerformIdleMarkCompact(int64_t deadline);

  void AddGCTime(int64_t micros) { gc_time_micros_ += micros; }

  int64_t gc_time_micros() const { return gc_time_micros_; }

  void IncrementCollections() { collections_++; }

  intptr_t collections() const { return collections_; }

#ifndef PRODUCT
  void PrintToJSONObject(JSONObject* object) const;
  void PrintHeapMapToJSONStream(Isolate* isolate, JSONStream* stream) const;
#endif  // PRODUCT

  void AllocateBlack(intptr_t size) {
    allocated_black_in_words_.fetch_add(size >> kWordSizeLog2);
  }

  void AllocatedExternal(intptr_t size) {
    ASSERT(size >= 0);
    intptr_t size_in_words = size >> kWordSizeLog2;
    usage_.external_in_words += size_in_words;
  }
  void FreedExternal(intptr_t size) {
    ASSERT(size >= 0);
    intptr_t size_in_words = size >> kWordSizeLog2;
    usage_.external_in_words -= size_in_words;
  }

  // 获取所有类型的空闲分区表
  FreeList* DataFreeList(intptr_t i = 0) {
    return &freelists_[OldPage::kData + i];
  }
  void AcquireLock(FreeList* freelist);
  void ReleaseLock(FreeList* freelist);

  // 在放置 Data 的空闲分区表上分配
  uword TryAllocateDataLocked(FreeList* freelist,
                              intptr_t size,
                              GrowthPolicy growth_policy) {
    bool is_protected = false;
    bool is_locked = true;
    return TryAllocateInternal(size, freelist, OldPage::kData, growth_policy,
                               is_protected, is_locked);
  }

  // 任务锁
  Monitor* tasks_lock() const { return &tasks_lock_; }
  intptr_t tasks() const { return tasks_; }
  void set_tasks(intptr_t val) {
    ASSERT(val >= 0);
    tasks_ = val;
  }
  
  // 并发标记任务
  intptr_t concurrent_marker_tasks() const { return concurrent_marker_tasks_; }
  void set_concurrent_marker_tasks(intptr_t val) {
    ASSERT(val >= 0);
    concurrent_marker_tasks_ = val;
  }
 
  // GC 阶段
  Phase phase() const { return phase_; }
  void set_phase(Phase val) { phase_ = val; }

  // 尝试在空闲分区表上的 Bump 区分配
  uword TryAllocateDataBumpLocked(intptr_t size) {
    return TryAllocateDataBumpLocked(&freelists_[OldPage::kData], size);
  }
  uword TryAllocateDataBumpLocked(FreeList* freelist, intptr_t size);
    
  
  DART_FORCE_INLINE
  uword TryAllocatePromoLocked(FreeList* freelist, intptr_t size) {
    uword result = freelist->TryAllocateBumpLocked(size);
    if (result != 0) {
      return result;
    }
    return TryAllocatePromoLockedSlow(freelist, size);
  }
  uword TryAllocatePromoLockedSlow(FreeList* freelist, intptr_t size);

  void SetupImagePage(void* pointer, uword size, bool is_executable);

  // 归还任何 Bump 区的分配到空闲分区表
  void AbandonBumpAllocation();
  // Have threads release marking stack blocks, etc.
  void AbandonMarkingForShutdown();

  bool enable_concurrent_mark() const { return enable_concurrent_mark_; }
  void set_enable_concurrent_mark(bool enable_concurrent_mark) {
    enable_concurrent_mark_ = enable_concurrent_mark;
  }

  bool IsObjectFromImagePages(ObjectPtr object);

  // 合并其他空间
  void MergeFrom(PageSpace* donor);

 private:
  // Ids for time and data records in Heap::GCStats.
  enum {
    // Time
    kConcurrentSweep = 0,
    kSafePoint = 1,
    kMarkObjects = 2,
    kResetFreeLists = 3,
    kSweepPages = 4,
    kSweepLargePages = 5,
    // Data
    kGarbageRatio = 0,
    kGCTimeFraction = 1,
    kPageGrowth = 2,
    kAllowedGrowth = 3
  };

  /// 以下函数用于在 OldPage 上分配指定 size 的内存
  // 分配内部调用
  uword TryAllocateInternal(intptr_t size,
                            FreeList* freelist,
                            OldPage::PageType type,
                            GrowthPolicy growth_policy,
                            bool is_protected,
                            bool is_locked);
  // 在新页分配
  uword TryAllocateInFreshPage(intptr_t size,
                               FreeList* freelist,
                               OldPage::PageType type,
                               GrowthPolicy growth_policy,
                               bool is_locked);
  // 在 Large 新页分配
  uword TryAllocateInFreshLargePage(intptr_t size,
                                    OldPage::PageType type,
                                    GrowthPolicy growth_policy);

  void EvaluateConcurrentMarking(GrowthPolicy growth_policy);

  // Makes bump block walkable; do not call concurrently with mutator.
  void MakeIterable() const;

  void AddPageLocked(OldPage* page);
  void AddLargePageLocked(OldPage* page);
  void AddExecPageLocked(OldPage* page);
  void RemovePageLocked(OldPage* page, OldPage* previous_page);
  void RemoveLargePageLocked(OldPage* page, OldPage* previous_page);
  void RemoveExecPageLocked(OldPage* page, OldPage* previous_page);

  OldPage* AllocatePage(OldPage::PageType type, bool link = true);
  OldPage* AllocateLargePage(intptr_t size, OldPage::PageType type);

  // 裁剪指定的 page
  void TruncateLargePage(OldPage* page, intptr_t new_object_size_in_bytes);
  // 释放指定的 page
  void FreePage(OldPage* page, OldPage* previous_page);
  void FreeLargePage(OldPage* page, OldPage* previous_page);
  // 释放所有的 page
  void FreePages(OldPage* pages);

  // OldPage GC 算法和实现
  void CollectGarbageHelper(bool compact,
                            bool finalize,
                            int64_t pre_wait_for_sweepers,
                            int64_t pre_safe_point);
  // 清理 Large page
  void SweepLarge();
  // 清理 page
  void Sweep();
  // 并发清理
  void ConcurrentSweep(IsolateGroup* isolate_group);
  // 整理
  void Compact(Thread* thread);

  static intptr_t LargePageSizeInWordsFor(intptr_t size);

  bool CanIncreaseCapacityInWordsLocked(intptr_t increase_in_words) {
    if (max_capacity_in_words_ == 0) {
      // Unlimited.
      return true;
    }
    intptr_t free_capacity_in_words =
        (max_capacity_in_words_ - usage_.capacity_in_words);
    return ((free_capacity_in_words > 0) &&
            (increase_in_words <= free_capacity_in_words));
  }

  Heap* const heap_;
  
  // 在 freelists_[OldPage::kExecutable] 处有一个储存 executable 数据的页面
  // 从 freelists_[OldPage::kData] 开始，有 FLAG_scavenger_tasks 个用于保存数据的页面。sweeper 循环向页面插入数据，
  // 每个 scavenger workers 使用一个页面而不进行锁定
  const intptr_t num_freelists_;
  FreeList* freelists_;

  // 使用 ExclusivePageIterator 来进行安全的访问
  mutable Mutex pages_lock_;
  OldPage* pages_ = nullptr;
  OldPage* pages_tail_ = nullptr;
  OldPage* exec_pages_ = nullptr;
  OldPage* exec_pages_tail_ = nullptr;
  OldPage* large_pages_ = nullptr;
  OldPage* large_pages_tail_ = nullptr;
  OldPage* image_pages_ = nullptr;

  // 正在为这一代跟踪各种大小。
  intptr_t max_capacity_in_words_;

  // 此与被并发清理器修改
  SpaceUsage usage_;
  RelaxedAtomic<intptr_t> allocated_black_in_words_;

  // Keep track of running MarkSweep tasks.
  mutable Monitor tasks_lock_;
  intptr_t tasks_;
  intptr_t concurrent_marker_tasks_;
  Phase phase_;

#if defined(DEBUG)
  Thread* iterating_thread_;
#endif
  PageSpaceController page_space_controller_;
  GCMarker* marker_;

  int64_t gc_time_micros_;
  intptr_t collections_;
  intptr_t mark_words_per_micro_;

  bool enable_concurrent_mark_;

  friend class BasePageIterator;
  friend class ExclusivePageIterator;
  friend class ExclusiveCodePageIterator;
  friend class ExclusiveLargePageIterator;
  friend class HeapIterationScope;
  friend class HeapSnapshotWriter;
  friend class PageSpaceController;
  friend class ConcurrentSweeperTask;
  friend class GCCompactor;
  friend class CompactorTask;

  DISALLOW_IMPLICIT_CONSTRUCTORS(PageSpace);
};

}  // namespace dart

#endif  // RUNTIME_VM_HEAP_PAGES_H_
```



## pages.cc

```c++
// ...

namespace dart {

// 期望的进行垃圾回收后老生代的空间占比==20
DEFINE_FLAG(int,
            old_gen_growth_space_ratio,
            20,
            "The desired maximum percentage of free space after old gen GC");
// 期望的对老生代进行垃圾回收的时间占比=3
DEFINE_FLAG(int,
            old_gen_growth_time_ratio,
            3,
            "The desired maximum percentage of time spent in old gen GC");
// 每次老生代页面数可增长的最大量=280    
DEFINE_FLAG(int,
            old_gen_growth_rate,
            280,
            "The max number of pages the old generation can grow at a time");
    
DEFINE_FLAG(bool,
            print_free_list_before_gc,
            false,
            "Print free list statistics before a GC");
DEFINE_FLAG(bool,
            print_free_list_after_gc,
            false,
            "Print free list statistics after a GC");
DEFINE_FLAG(bool, log_growth, false, "Log PageSpace growth policy decisions.");

// 创建老生代页面
OldPage* OldPage::Allocate(intptr_t size_in_words,
                           PageType type,
                           const char* name) {
  const bool executable = type == kExecutable;

  /// 调用虚拟内存进行分配
  VirtualMemory* memory = VirtualMemory::AllocateAligned(
      size_in_words << kWordSizeLog2, kOldPageSize, executable, name);
  /// 内存不足，可能进行 GC 操作
  if (memory == NULL) {
    return NULL;
  }

  // 封装成 OldPage 对象
  OldPage* result = reinterpret_cast<OldPage*>(memory->address());
  ASSERT(result != NULL);
  result->memory_ = memory;
  result->next_ = NULL;
  result->used_in_bytes_ = 0;
  result->forwarding_page_ = NULL;
  result->card_table_ = NULL;
  result->type_ = type;

  LSAN_REGISTER_ROOT_REGION(result, sizeof(*result));

  return result;
}

// 释放页表
void OldPage::Deallocate() {
  if (card_table_ != NULL) {
    free(card_table_);
    card_table_ = NULL;
  }

  bool image_page = is_image_page();

  if (!image_page) {
    LSAN_UNREGISTER_ROOT_REGION(this, sizeof(*this));
  }

  // 释放 VirtualMemory
  delete memory_;

  // For a heap page from a snapshot, the OldPage object lives in the malloc
  // heap rather than the page itself.
  if (image_page) {
    free(this);
  }
}

/// 遍历访问此页上的所有对象
void OldPage::VisitObjects(ObjectVisitor* visitor) const {
  ASSERT(Thread::Current()->IsAtSafepoint());
  NoSafepointScope no_safepoint;
  uword obj_addr = object_start();
  uword end_addr = object_end();
  while (obj_addr < end_addr) {
    ObjectPtr raw_obj = ObjectLayout::FromAddr(obj_addr);
    visitor->VisitObject(raw_obj);
    // raw_obj: ObjectPtr
    // raw_obj->ptr(): ObjectLayout*
    // raw_obj->ptr()->HeapSize(): uword
    // obj_addr 用于遍历对象，每遍历一个对象就加上它的 size（实际是 Instance_Size）
    obj_addr += raw_obj->ptr()->HeapSize();
  }
  ASSERT(obj_addr == end_addr);
}

/// 遍历对象指针（以OBjectPtr 的方式访问）
void OldPage::VisitObjectPointers(ObjectPointerVisitor* visitor) const {
  ASSERT(Thread::Current()->IsAtSafepoint() ||
         (Thread::Current()->task_kind() == Thread::kCompactorTask));
  NoSafepointScope no_safepoint;
  uword obj_addr = object_start();
  uword end_addr = object_end();
  while (obj_addr < end_addr) {
    ObjectPtr raw_obj = ObjectLayout::FromAddr(obj_addr);
    obj_addr += raw_obj->ptr()->VisitPointers(visitor);
  }
  ASSERT(obj_addr == end_addr);
}

/// 访问记忆卡
void OldPage::VisitRememberedCards(ObjectPointerVisitor* visitor) {
  ASSERT(Thread::Current()->IsAtSafepoint() ||
         (Thread::Current()->task_kind() == Thread::kScavengerTask));
  NoSafepointScope no_safepoint;

  if (card_table_ == NULL) {
    return;
  }

  bool table_is_empty = false;

  ArrayPtr obj = static_cast<ArrayPtr>(ObjectLayout::FromAddr(object_start()));
  ASSERT(obj->IsArray());
  ASSERT(obj->ptr()->IsCardRemembered());
  ObjectPtr* obj_from = obj->ptr()->from();
  ObjectPtr* obj_to = obj->ptr()->to(Smi::Value(obj->ptr()->length_));

  const intptr_t size = card_table_size();
  for (intptr_t i = 0; i < size; i++) {
    if (card_table_[i] != 0) {
      ObjectPtr* card_from =
          reinterpret_cast<ObjectPtr*>(this) + (i << kSlotsPerCardLog2);
      ObjectPtr* card_to = reinterpret_cast<ObjectPtr*>(card_from) +
                           (1 << kSlotsPerCardLog2) - 1;
      // Minus 1 because to is inclusive.

      if (card_from < obj_from) {
        // First card overlaps with header.
        card_from = obj_from;
      }
      if (card_to > obj_to) {
        // Last card(s) may extend past the object. Array truncation can make
        // this happen for more than one card.
        card_to = obj_to;
      }

      visitor->VisitPointers(card_from, card_to);

      bool has_new_target = false;
      for (ObjectPtr* slot = card_from; slot <= card_to; slot++) {
        if ((*slot)->IsNewObjectMayBeSmi()) {
          has_new_target = true;
          break;
        }
      }

      if (has_new_target) {
        // Card remains remembered.
        table_is_empty = false;
      } else {
        card_table_[i] = 0;
      }
    }
  }

  if (table_is_empty) {
    free(card_table_);
    card_table_ = NULL;
  }
}

ObjectPtr OldPage::FindObject(FindObjectVisitor* visitor) const {
  uword obj_addr = object_start();
  uword end_addr = object_end();
  if (visitor->VisitRange(obj_addr, end_addr)) {
    while (obj_addr < end_addr) {
      ObjectPtr raw_obj = ObjectLayout::FromAddr(obj_addr);
      uword next_obj_addr = obj_addr + raw_obj->ptr()->HeapSize();
      if (visitor->VisitRange(obj_addr, next_obj_addr) &&
          raw_obj->ptr()->FindObject(visitor)) {
        return raw_obj;  // Found object, return it.
      }
      obj_addr = next_obj_addr;
    }
    ASSERT(obj_addr == end_addr);
  }
  return Object::null();
}

void OldPage::WriteProtect(bool read_only) {
  ASSERT(!is_image_page());

  VirtualMemory::Protection prot;
  if (read_only) {
    if ((type_ == kExecutable) && (memory_->AliasOffset() == 0)) {
      prot = VirtualMemory::kReadExecute;
    } else {
      prot = VirtualMemory::kReadOnly;
    }
  } else {
    prot = VirtualMemory::kReadWrite;
  }
  memory_->Protect(prot);
}

// The initial estimate of how many words we can mark per microsecond (usage
// before / mark-sweep time). This is a conservative value observed running
// Flutter on a Nexus 4. After the first mark-sweep, we instead use a value
// based on the device's actual speed.
static const intptr_t kConservativeInitialMarkSpeed = 20;

/// PageSpace 构造函数
PageSpace::PageSpace(Heap* heap, intptr_t max_capacity_in_words)
    : heap_(heap),
      // freeList 的数量
      num_freelists_(Utils::Maximum(FLAG_scavenger_tasks, 1) + 1),
      
      freelists_(new FreeList[num_freelists_]),
      pages_lock_(),
      max_capacity_in_words_(max_capacity_in_words),
      usage_(),
      allocated_black_in_words_(0),
      tasks_lock_(),
      tasks_(0),
      concurrent_marker_tasks_(0),
      phase_(kDone),
#if defined(DEBUG)
      iterating_thread_(NULL),
#endif
      // 控制器
      page_space_controller_(heap,
                             // flag 位
                             FLAG_old_gen_growth_space_ratio,
                             FLAG_old_gen_growth_rate,
                             FLAG_old_gen_growth_time_ratio),
      marker_(NULL),
      gc_time_micros_(0),
      collections_(0),
      mark_words_per_micro_(kConservativeInitialMarkSpeed),
      enable_concurrent_mark_(FLAG_concurrent_mark) {
  // 此时没有持有锁，但是依然可用更新：因为没有其他对象可以访问此类
  UpdateMaxCapacityLocked();
  UpdateMaxUsed();

  for (intptr_t i = 0; i < num_freelists_; i++) {
    freelists_[i].Reset();
  }
}

PageSpace::~PageSpace() {
  {
    MonitorLocker ml(tasks_lock());
    while (tasks() > 0) {
      ml.Wait();
    }
  }
  /// 释放所有的 Page
  FreePages(pages_);
  FreePages(exec_pages_);
  FreePages(large_pages_);
  FreePages(image_pages_);
  ASSERT(marker_ == NULL);
  delete[] freelists_;
}

// 对于给定的 size，返回一个合适的 LargePage Size
intptr_t PageSpace::LargePageSizeInWordsFor(intptr_t size) {
  intptr_t page_size = Utils::RoundUp(size + OldPage::ObjectStartOffset(),
                                      VirtualMemory::PageSize());
  return page_size >> kWordSizeLog2;
}

    
/// 向 PageSpace 中添加新的 OldPage
void PageSpace::AddPageLocked(OldPage* page) {
  if (pages_ == nullptr) {
    pages_ = page;
  } else {
    pages_tail_->set_next(page);
  }
  pages_tail_ = page;
}

void PageSpace::AddLargePageLocked(OldPage* page) {
  if (large_pages_ == nullptr) {
    large_pages_ = page;
  } else {
    large_pages_tail_->set_next(page);
  }
  large_pages_tail_ = page;
}

void PageSpace::AddExecPageLocked(OldPage* page) {
  if (exec_pages_ == nullptr) {
    exec_pages_ = page;
  } else {
    if (FLAG_write_protect_code) {
      exec_pages_tail_->WriteProtect(false);
    }
    exec_pages_tail_->set_next(page);
    if (FLAG_write_protect_code) {
      exec_pages_tail_->WriteProtect(true);
    }
  }
  exec_pages_tail_ = page;
}

/// 在 PageSpace 中移除指定的 OldPage
void PageSpace::RemovePageLocked(OldPage* page, OldPage* previous_page) {
  if (previous_page != NULL) {
    previous_page->set_next(page->next());
  } else {
    pages_ = page->next();
  }
  if (page == pages_tail_) {
    pages_tail_ = previous_page;
  }
}

void PageSpace::RemoveLargePageLocked(OldPage* page, OldPage* previous_page) {
  if (previous_page != NULL) {
    previous_page->set_next(page->next());
  } else {
    large_pages_ = page->next();
  }
  if (page == large_pages_tail_) {
    large_pages_tail_ = previous_page;
  }
}

void PageSpace::RemoveExecPageLocked(OldPage* page, OldPage* previous_page) {
  if (previous_page != NULL) {
    previous_page->set_next(page->next());
  } else {
    exec_pages_ = page->next();
  }
  if (page == exec_pages_tail_) {
    exec_pages_tail_ = previous_page;
  }
}

// 创建一个页面
OldPage* PageSpace::AllocatePage(OldPage::PageType type, bool link) {
  {
    MutexLocker ml(&pages_lock_);
    if (!CanIncreaseCapacityInWordsLocked(kOldPageSizeInWords)) {
      return nullptr;
    }
    IncreaseCapacityInWordsLocked(kOldPageSizeInWords);
  }
  const bool is_exec = (type == OldPage::kExecutable);
  const char* name = Heap::RegionName(is_exec ? Heap::kCode : Heap::kOld);
    
  /// 使用虚拟内存分配一个 kOldPageSizeInWords 的内存，并且封装成 OldPage 对象
  OldPage* page = OldPage::Allocate(kOldPageSizeInWords, type, name);
  if (page == nullptr) {
    RELEASE_ASSERT(!FLAG_abort_on_oom);
    IncreaseCapacityInWords(-kOldPageSizeInWords);
    return nullptr;
  }

  /// 以下部分互斥访问
  MutexLocker ml(&pages_lock_);
  if (link) {
    if (is_exec) {
      AddExecPageLocked(page);
    } else {
      AddPageLocked(page);
    }
  }

  page->set_object_end(page->memory_->end());
  if ((type != OldPage::kExecutable) && 
      (heap_ != nullptr) &&
      (heap_->isolate_group() != Dart::vm_isolate()->group())
     ) {
    page->AllocateForwardingPage();
  }
  return page;
}

/// 创建 Large 
OldPage* PageSpace::AllocateLargePage(intptr_t size, OldPage::PageType type) {
  const intptr_t page_size_in_words = LargePageSizeInWordsFor(size);
  {
    MutexLocker ml(&pages_lock_);
    if (!CanIncreaseCapacityInWordsLocked(page_size_in_words)) {
      return nullptr;
    }
    IncreaseCapacityInWordsLocked(page_size_in_words);
  }
  const bool is_exec = (type == OldPage::kExecutable);
  const char* name = Heap::RegionName(is_exec ? Heap::kCode : Heap::kOld);
  OldPage* page = OldPage::Allocate(page_size_in_words, type, name);

  MutexLocker ml(&pages_lock_);
  if (page == nullptr) {
    IncreaseCapacityInWordsLocked(-page_size_in_words);
    return nullptr;
  }
  if (is_exec) {
    AddExecPageLocked(page);
  } else {
    AddLargePageLocked(page);
  }

  // 此页上只有一个对象 (at least until Array::MakeFixedLength is called).
  page->set_object_end(page->object_start() + size);
  return page;
}

// 裁剪 Large Page 到 new_object_size_in_bytes
void PageSpace::TruncateLargePage(OldPage* page,
                                  intptr_t new_object_size_in_bytes) {
  const intptr_t old_object_size_in_bytes =
      page->object_end() - page->object_start();
  ASSERT(new_object_size_in_bytes <= old_object_size_in_bytes);
  const intptr_t new_page_size_in_words =
      LargePageSizeInWordsFor(new_object_size_in_bytes);
  VirtualMemory* memory = page->memory_;
  const intptr_t old_page_size_in_words = (memory->size() >> kWordSizeLog2);
  if (new_page_size_in_words < old_page_size_in_words) {
    memory->Truncate(new_page_size_in_words << kWordSizeLog2);
    IncreaseCapacityInWords(new_page_size_in_words - old_page_size_in_words);
    page->set_object_end(page->object_start() + new_object_size_in_bytes);
  }
}

// 释放 Page
void PageSpace::FreePage(OldPage* page, OldPage* previous_page) {
  bool is_exec = (page->type() == OldPage::kExecutable);
  {
    MutexLocker ml(&pages_lock_);
    IncreaseCapacityInWordsLocked(-(page->memory_->size() >> kWordSizeLog2));
    if (is_exec) {
      RemoveExecPageLocked(page, previous_page);
    } else {
      RemovePageLocked(page, previous_page);
    }
  }
  // TODO(iposva): Consider adding to a pool of empty pages.
  page->Deallocate();
}

void PageSpace::FreeLargePage(OldPage* page, OldPage* previous_page) {
  ASSERT(page->type() != OldPage::kExecutable);
  MutexLocker ml(&pages_lock_);
  IncreaseCapacityInWordsLocked(-(page->memory_->size() >> kWordSizeLog2));
  RemoveLargePageLocked(page, previous_page);
  page->Deallocate();
}

// 释放所有 Page
void PageSpace::FreePages(OldPage* pages) {
  OldPage* page = pages;
  while (page != NULL) {
    OldPage* next = page->next();
    page->Deallocate();
    page = next;
  }
}

void PageSpace::EvaluateConcurrentMarking(GrowthPolicy growth_policy) {
  if (growth_policy != kForceGrowth) {
    ASSERT(GrowthControlState());
    if (heap_ != NULL) {  // Some unit tests.
      Thread* thread = Thread::Current();
      if (thread->CanCollectGarbage()) {
        heap_->CheckFinishConcurrentMarking(thread);
        heap_->CheckStartConcurrentMarking(thread, Heap::kOldSpace);
      }
    }
  }
}

/// 在新的 Page 中分配给定的 size
uword PageSpace::TryAllocateInFreshPage(intptr_t size,
                                        FreeList* freelist,
                                        OldPage::PageType type,
                                        GrowthPolicy growth_policy,
                                        bool is_locked) {
  ASSERT(Heap::IsAllocatableViaFreeLists(size));

  EvaluateConcurrentMarking(growth_policy);

  uword result = 0;
  SpaceUsage after_allocation = GetCurrentUsage();
  after_allocation.used_in_words += size >> kWordSizeLog2;
  // Can we grow by one page?
  after_allocation.capacity_in_words += kOldPageSizeInWords;
    
    
  if (growth_policy == kForceGrowth ||
      !page_space_controller_.ReachedHardThreshold(after_allocation)) {
    OldPage* page = AllocatePage(type);
    if (page == NULL) {
      return 0;
    }
    // Start of the newly allocated page is the allocated object.
    result = page->object_start();
    // Note: usage_.capacity_in_words is increased by AllocatePage.
    usage_.used_in_words += (size >> kWordSizeLog2);
    // 计算剩余的空闲空间，即新 Page 除去请求的 size 之外的空间大小
    uword free_start = result + size;
    intptr_t free_size = page->object_end() - free_start;
      
    // 把新空间加入到空闲分区表
    if (free_size > 0) {
      if (is_locked) {
        freelist->FreeLocked(free_start, free_size);
      } else {
        freelist->Free(free_start, free_size);
      }
    }
  }
  return result;
}

/// 在新的 Large Page 上分配
uword PageSpace::TryAllocateInFreshLargePage(intptr_t size,
                                             OldPage::PageType type,
                                             GrowthPolicy growth_policy) {
  ASSERT(!Heap::IsAllocatableViaFreeLists(size));

  EvaluateConcurrentMarking(growth_policy);

  intptr_t page_size_in_words = LargePageSizeInWordsFor(size);
  if ((page_size_in_words << kWordSizeLog2) < size) {
    // On overflow we fail to allocate.
    return 0;
  }

  uword result = 0;
  SpaceUsage after_allocation = GetCurrentUsage();
  after_allocation.used_in_words += size >> kWordSizeLog2;
  after_allocation.capacity_in_words += page_size_in_words;
  if (growth_policy == kForceGrowth ||
      !page_space_controller_.ReachedHardThreshold(after_allocation)) {
    OldPage* page = AllocateLargePage(size, type);
    if (page != NULL) {
      result = page->object_start();
      // Note: usage_.capacity_in_words is increased by AllocateLargePage.
      usage_.used_in_words += (size >> kWordSizeLog2);
    }
  }
  return result;
}

/// 在 PageSpace 中分配给定的 size 的内存
uword PageSpace::TryAllocateInternal(intptr_t size,
                                     FreeList* freelist,
                                     OldPage::PageType type,
                                     GrowthPolicy growth_policy,
                                     bool is_protected,
                                     bool is_locked) {
  ASSERT(size >= kObjectAlignment);
  ASSERT(Utils::IsAligned(size, kObjectAlignment));
  uword result = 0;
  // 如果可以在空闲分区表上分配，即 size < 64 KB
  if (Heap::IsAllocatableViaFreeLists(size)) {
    if (is_locked) {
      result = freelist->TryAllocateLocked(size, is_protected);
    } else {
      result = freelist->TryAllocate(size, is_protected);
    }
    // 如果在空闲分区表上的空间不足
    if (result == 0) {
      // 在新的 page 上分配
      result = TryAllocateInFreshPage(size, freelist, type, growth_policy,
                                      is_locked);
      // usage_ is updated by the call above.
    } else {
      usage_.used_in_words += (size >> kWordSizeLog2);
    }
  // 否则在新的 Large 页面上进行分配
  } else {
    result = TryAllocateInFreshLargePage(size, type, growth_policy);
    // usage_ is updated by the call above.
  }
  ASSERT((result & kObjectAlignmentMask) == kOldObjectAlignmentOffset);
  return result;
}

void PageSpace::AcquireLock(FreeList* freelist) {
  freelist->mutex()->Lock();
}

void PageSpace::ReleaseLock(FreeList* freelist) {
  intptr_t size = freelist->TakeUnaccountedSizeLocked();
  usage_.used_in_words += (size >> kWordSizeLog2);
  freelist->mutex()->Unlock();
}

/// 以下为 Page 迭代器
class BasePageIterator : ValueObject {
 public:
  explicit BasePageIterator(const PageSpace* space) : space_(space) {}

  OldPage* page() const { return page_; }

  bool Done() const { return page_ == NULL; }

  void Advance() {
    ASSERT(!Done());
    page_ = page_->next();
    if ((page_ == NULL) && (list_ == kRegular)) {
      list_ = kExecutable;
      page_ = space_->exec_pages_;
    }
    if ((page_ == NULL) && (list_ == kExecutable)) {
      list_ = kLarge;
      page_ = space_->large_pages_;
    }
    if ((page_ == NULL) && (list_ == kLarge)) {
      list_ = kImage;
      page_ = space_->image_pages_;
    }
    ASSERT((page_ != NULL) || (list_ == kImage));
  }

 protected:
  enum List { kRegular, kExecutable, kLarge, kImage };

  void Initialize() {
    list_ = kRegular;
    page_ = space_->pages_;
    if (page_ == NULL) {
      list_ = kExecutable;
      page_ = space_->exec_pages_;
      if (page_ == NULL) {
        list_ = kLarge;
        page_ = space_->large_pages_;
        if (page_ == NULL) {
          list_ = kImage;
          page_ = space_->image_pages_;
        }
      }
    }
  }

  const PageSpace* space_ = nullptr;
  List list_;
  OldPage* page_ = nullptr;
};

// Provides unsafe access to all pages. Assumes pages are walkable.
class UnsafeExclusivePageIterator : public BasePageIterator {
 public:
  explicit UnsafeExclusivePageIterator(const PageSpace* space)
      : BasePageIterator(space) {
    Initialize();
  }
};

// Provides exclusive access to all pages, and ensures they are walkable.
class ExclusivePageIterator : public BasePageIterator {
 public:
  explicit ExclusivePageIterator(const PageSpace* space)
      : BasePageIterator(space), ml_(&space->pages_lock_) {
    space_->MakeIterable();
    Initialize();
  }

 private:
  MutexLocker ml_;
  NoSafepointScope no_safepoint;
};

// Provides exclusive access to code pages, and ensures they are walkable.
// NOTE: This does not iterate over large pages which can contain code.
class ExclusiveCodePageIterator : ValueObject {
 public:
  explicit ExclusiveCodePageIterator(const PageSpace* space)
      : space_(space), ml_(&space->pages_lock_) {
    space_->MakeIterable();
    page_ = space_->exec_pages_;
  }
  OldPage* page() const { return page_; }
  bool Done() const { return page_ == NULL; }
  void Advance() {
    ASSERT(!Done());
    page_ = page_->next();
  }

 private:
  const PageSpace* space_;
  MutexLocker ml_;
  NoSafepointScope no_safepoint;
  OldPage* page_;
};

    
/// 使得列表可以迭代
void PageSpace::MakeIterable() const {
  // Assert not called from concurrent sweeper task.
  // TODO(koda): Use thread/task identity when implemented.
  ASSERT(IsolateGroup::Current()->heap() != NULL);
  for (intptr_t i = 0; i < num_freelists_; i++) {
    freelists_[i].MakeIterable();
  }
}

void PageSpace::AbandonBumpAllocation() {
  for (intptr_t i = 0; i < num_freelists_; i++) {
    freelists_[i].AbandonBumpAllocation();
  }
}

void PageSpace::AbandonMarkingForShutdown() {
  delete marker_;
  marker_ = NULL;
}

void PageSpace::UpdateMaxCapacityLocked() {
  if (heap_ == NULL) {
    // Some unit tests.
    return;
  }
  ASSERT(heap_ != NULL);
  ASSERT(heap_->isolate_group() != NULL);
  auto isolate_group = heap_->isolate_group();
  isolate_group->GetHeapOldCapacityMaxMetric()->SetValue(
      static_cast<int64_t>(usage_.capacity_in_words) * kWordSize);
}

void PageSpace::UpdateMaxUsed() {
  if (heap_ == NULL) {
    // Some unit tests.
    return;
  }
  ASSERT(heap_ != NULL);
  ASSERT(heap_->isolate_group() != NULL);
  auto isolate_group = heap_->isolate_group();
  isolate_group->GetHeapOldUsedMaxMetric()->SetValue(UsedInWords() * kWordSize);
}

// 是否包含某一地址，在所有的 page 上查找
bool PageSpace::Contains(uword addr) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (it.page()->Contains(addr)) {
      return true;
    }
  }
  return false;
}

bool PageSpace::ContainsUnsafe(uword addr) const {
  for (UnsafeExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (it.page()->Contains(addr)) {
      return true;
    }
  }
  return false;
}

// 查找指定类型的地址
bool PageSpace::Contains(uword addr, OldPage::PageType type) const {
  if (type == OldPage::kExecutable) {
    // Fast path executable pages.
    for (ExclusiveCodePageIterator it(this); !it.Done(); it.Advance()) {
      if (it.page()->Contains(addr)) {
        return true;
      }
    }
    return false;
  }
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if ((it.page()->type() == type) && it.page()->Contains(addr)) {
      return true;
    }
  }
  return false;
}

bool PageSpace::DataContains(uword addr) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if ((it.page()->type() != OldPage::kExecutable) &&
        it.page()->Contains(addr)) {
      return true;
    }
  }
  return false;
}

void PageSpace::AddRegionsToObjectSet(ObjectSet* set) const {
  ASSERT((pages_ != NULL) || (exec_pages_ != NULL) || (large_pages_ != NULL));
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    set->AddRegion(it.page()->object_start(), it.page()->object_end());
  }
}

/// 访问对象
void PageSpace::VisitObjects(ObjectVisitor* visitor) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    it.page()->VisitObjects(visitor);
  }
}

void PageSpace::VisitObjectsNoImagePages(ObjectVisitor* visitor) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (!it.page()->is_image_page()) {
      it.page()->VisitObjects(visitor);
    }
  }
}

void PageSpace::VisitObjectsImagePages(ObjectVisitor* visitor) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (it.page()->is_image_page()) {
      it.page()->VisitObjects(visitor);
    }
  }
}

void PageSpace::VisitObjectPointers(ObjectPointerVisitor* visitor) const {
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    it.page()->VisitObjectPointers(visitor);
  }
}

void PageSpace::VisitRememberedCards(ObjectPointerVisitor* visitor) const {
  ASSERT(Thread::Current()->IsAtSafepoint() ||
         (Thread::Current()->task_kind() == Thread::kScavengerTask));

  // Wait for the sweeper to finish mutating the large page list.
  MonitorLocker ml(tasks_lock());
  while (phase() == kSweepingLarge) {
    ml.Wait();  // No safepoint check.
  }

  // Large pages may be added concurrently due to promotion in another scavenge
  // worker, so terminate the traversal when we hit the tail we saw while
  // holding the pages lock, instead of at NULL, otherwise we are racing when we
  // read OldPage::next_ and OldPage::remembered_cards_.
  OldPage* page;
  OldPage* tail;
  {
    MutexLocker ml(&pages_lock_);
    page = large_pages_;
    tail = large_pages_tail_;
  }
  while (page != nullptr) {
    page->VisitRememberedCards(visitor);
    if (page == tail) break;
    page = page->next();
  }
}

/// 查找对象
ObjectPtr PageSpace::FindObject(FindObjectVisitor* visitor,
                                OldPage::PageType type) const {
  if (type == OldPage::kExecutable) {
    // Fast path executable pages.
    for (ExclusiveCodePageIterator it(this); !it.Done(); it.Advance()) {
      ObjectPtr obj = it.page()->FindObject(visitor);
      if (obj != Object::null()) {
        return obj;
      }
    }
    return Object::null();
  }

  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (it.page()->type() == type) {
      ObjectPtr obj = it.page()->FindObject(visitor);
      if (obj != Object::null()) {
        return obj;
      }
    }
  }
  return Object::null();
}

void PageSpace::WriteProtect(bool read_only) {
  if (read_only) {
    // Avoid MakeIterable trying to write to the heap.
    AbandonBumpAllocation();
  }
  for (ExclusivePageIterator it(this); !it.Done(); it.Advance()) {
    if (!it.page()->is_image_page()) {
      it.page()->WriteProtect(read_only);
    }
  }
}

void PageSpace::WriteProtectCode(bool read_only) {
  if (FLAG_write_protect_code) {
    MutexLocker ml(&pages_lock_);
    NoSafepointScope no_safepoint;
    // No need to go through all of the data pages first.
    OldPage* page = exec_pages_;
    while (page != NULL) {
      ASSERT(page->type() == OldPage::kExecutable);
      page->WriteProtect(read_only);
      page = page->next();
    }
    page = large_pages_;
    while (page != NULL) {
      if (page->type() == OldPage::kExecutable) {
        page->WriteProtect(read_only);
      }
      page = page->next();
    }
  }
}

/// 是否应该开始空闲标记清理
bool PageSpace::ShouldStartIdleMarkSweep(int64_t deadline) {
  // To make a consistent decision, we should not yield for a safepoint in the
  // middle of deciding whether to perform an idle GC.
  NoSafepointScope no_safepoint;

  if (!page_space_controller_.ReachedIdleThreshold(usage_)) {
    return false;
  }

  {
    MonitorLocker locker(tasks_lock());
    if (tasks() > 0) {
      // 并发的清理任务正在运行。如果我们现在开始标记清理算法就必须等待它结束。
      // 等待时间不会被包括到 mark_words_per_micro_ 中
      return false;
    }
  }

  // This uses the size of new-space because the pause time to start concurrent
  // marking is related to the size of the root set, which is mostly new-space.
  int64_t estimated_mark_completion =
      OS::GetCurrentMonotonicMicros() +
      heap_->new_space()->UsedInWords() / mark_words_per_micro_;
  return estimated_mark_completion <= deadline;
}

bool PageSpace::ShouldPerformIdleMarkCompact(int64_t deadline) {
  // To make a consistent decision, we should not yield for a safepoint in the
  // middle of deciding whether to perform an idle GC.
  NoSafepointScope no_safepoint;

  // Discount two pages to account for the newest data and code pages, whose
  // partial use doesn't indicate fragmentation.
  const intptr_t excess_in_words =
      usage_.capacity_in_words - usage_.used_in_words - 2 * kOldPageSizeInWords;
  const double excess_ratio = static_cast<double>(excess_in_words) /
                              static_cast<double>(usage_.capacity_in_words);
  const bool fragmented = excess_ratio > 0.05;

  if (!fragmented && !page_space_controller_.ReachedIdleThreshold(usage_)) {
    return false;
  }

  {
    MonitorLocker locker(tasks_lock());
    if (tasks() > 0) {
      // A concurrent sweeper is running. If we start a mark sweep now
      // we'll have to wait for it, and this wait time is not included in
      // mark_words_per_micro_.
      return false;
    }
  }

  // Assuming compaction takes as long as marking.
  intptr_t mark_compact_words_per_micro = mark_words_per_micro_ / 2;
  if (mark_compact_words_per_micro == 0) {
    mark_compact_words_per_micro = 1;  // Prevent division by zero.
  }

  int64_t estimated_mark_compact_completion =
      OS::GetCurrentMonotonicMicros() +
      UsedInWords() / mark_compact_words_per_micro;
  return estimated_mark_compact_completion <= deadline;
}

/// 调用此函数在 OldPage 上回收垃圾
void PageSpace::CollectGarbage(bool compact, bool finalize) {
  ASSERT(GrowthControlState());

  if (!finalize) {
#if defined(TARGET_ARCH_IA32)
    return;  // Barrier not implemented.
#else
    if (!enable_concurrent_mark()) return;  // 禁用
    if (FLAG_marker_tasks == 0) return;     // 禁用
#endif
  }

  Thread* thread = Thread::Current();
  const int64_t pre_safe_point = OS::GetCurrentMonotonicMicros();
  SafepointOperationScope safepoint_scope(thread);

  const int64_t pre_wait_for_sweepers = OS::GetCurrentMonotonicMicros();
  // 等待挂起的任务完成，然后对驱动程序任务进行说明。
  Phase waited_for;
  {
    MonitorLocker locker(tasks_lock());
    
    waited_for = phase();
    
    if (!finalize &&
        (phase() == kMarking || phase() == kAwaitingFinalization)) {
      // 并发标记正在运行
      return;
    }

    while (tasks() > 0) {
      locker.Wait();
    }
    ASSERT(phase() == kAwaitingFinalization || phase() == kDone);
    set_tasks(1);
  }
    
  // 如果显示 GC 信息
  if (FLAG_verbose_gc) {
    const int64_t wait =
        OS::GetCurrentMonotonicMicros() - pre_wait_for_sweepers;
    if (waited_for == kMarking) {
      THR_Print("Waited %" Pd64 " us for concurrent marking to finish.\n",
                wait);
    } else if (waited_for == kSweepingRegular || waited_for == kSweepingLarge) {
      THR_Print("Waited %" Pd64 " us for concurrent sweeping to finish.\n",
                wait);
    }
  }

    
  // 确保此 Isolate 所有的线程都处于安全点（包括暂停的线程或者本地代码）。我们在新生代 GC和老生代 GC 之间设置警戒，以确保如
  // 果两个线程同时进行收集，则失败的线程会跳过收集，直接进入分配。
  {
    CollectGarbageHelper(compact, finalize, pre_wait_for_sweepers,
                         pre_safe_point);
  }

  // 完成回收，通知所有线程恢复
  {
    MonitorLocker ml(tasks_lock());
    set_tasks(tasks() - 1);
    ml.NotifyAll();
  }
}

/// OldPage GC 实现
void PageSpace::CollectGarbageHelper(bool compact,
                                     bool finalize,
                                     int64_t pre_wait_for_sweepers,
                                     int64_t pre_safe_point) {
  Thread* thread = Thread::Current();
  ASSERT(thread->IsAtSafepoint());
  auto isolate_group = heap_->isolate_group();
  ASSERT(isolate_group == IsolateGroup::Current());

  const int64_t start = OS::GetCurrentMonotonicMicros();

  // Perform various cleanup that relies on no tasks interfering.
  isolate_group->shared_class_table()->FreeOldTables();
  isolate_group->ForEachIsolate(
      [&](Isolate* isolate) { isolate->field_table()->FreeOldTables(); },
      /*at_safepoint=*/true);

  NoSafepointScope no_safepoints;

  // 打印空闲分区表
  if (FLAG_print_free_list_before_gc) {
    for (intptr_t i = 0; i < num_freelists_; i++) {
      OS::PrintErr("Before GC: Freelist %" Pd "\n", i);
      freelists_[i].Print();
    }
  }

  // GC 之前确认 
  if (FLAG_verify_before_gc) {
    OS::PrintErr("Verifying before marking...");
    heap_->VerifyGC(phase() == kDone ? kForbidMarked : kAllowMarked);
    OS::PrintErr(" done.\n");
  }

  // Make code pages writable.
  if (finalize) WriteProtectCode(false);

  // 记录内存使用
  SpaceUsage usage_before = GetCurrentUsage();

  // 标记所有可达的老生代对象
  if (marker_ == NULL) {
    ASSERT(phase() == kDone);
    marker_ = new GCMarker(isolate_group, heap_);
  } else {
    ASSERT(phase() == kAwaitingFinalization);
  }

  if (!finalize) {
    ASSERT(phase() == kDone);
    marker_->StartConcurrentMark(this);
    return;
  }

  /// 标记所有的可达对象
  marker_->MarkObjects(this);
  usage_.used_in_words = marker_->marked_words() + allocated_black_in_words_;
  allocated_black_in_words_ = 0;
  mark_words_per_micro_ = marker_->MarkedWordsPerMicro();
  delete marker_;
  marker_ = NULL;

  int64_t mid1 = OS::GetCurrentMonotonicMicros();

  // Abandon the remainder of the bump allocation block.
  AbandonBumpAllocation();
  // 重置空闲分区表
  for (intptr_t i = 0; i < num_freelists_; i++) {
    freelists_[i].Reset();
  }

  int64_t mid2 = OS::GetCurrentMonotonicMicros();
  int64_t mid3 = 0;

  {
    if (FLAG_verify_before_gc) {
      OS::PrintErr("Verifying before sweeping...");
      heap_->VerifyGC(kAllowMarked);
      OS::PrintErr(" done.\n");
    }

    // 可执行的页面总是被立即清除，以简化内存保护

    TIMELINE_FUNCTION_GC_DURATION(thread, "SweepExecutable");
    GCSweeper sweeper;
    OldPage* prev_page = NULL;
    OldPage* page = exec_pages_;
    FreeList* freelist = &freelists_[OldPage::kExecutable];
    MutexLocker ml(freelist->mutex());
    while (page != NULL) {
      OldPage* next_page = page->next();
      // 清理指定的页面，返回清理后页面使用的内存大小
      bool page_in_use = sweeper.SweepPage(page, freelist, true /*is_locked*/);
      // 有正在使用的内存则串联成新的链表
      if (page_in_use) {
        prev_page = page;
      } 
      // 如果 page_in_use == 0 则页面为空，释放此页 
      else {
        FreePage(page, prev_page);
      }
      // Advance to the next page.
      page = next_page;
    }

    mid3 = OS::GetCurrentMonotonicMicros();
  }

  // 如果使用整理算法
  if (compact) {
    // 清理 Large page
    SweepLarge();
    // 整理
    Compact(thread);
    set_phase(kDone);
  } else if (FLAG_concurrent_sweep) {
    // 并发清理
    ConcurrentSweep(isolate_group);
  } else {
    // 不使用整理算法
    SweepLarge();
    Sweep();
    set_phase(kDone);
  }

  // Make code pages read-only.
  if (finalize) WriteProtectCode(true);

  int64_t end = OS::GetCurrentMonotonicMicros();

  // Record signals for growth control. Include size of external allocations.
  page_space_controller_.EvaluateGarbageCollection(
      usage_before, GetCurrentUsage(), start, end);

  heap_->RecordTime(kConcurrentSweep, pre_safe_point - pre_wait_for_sweepers);
  heap_->RecordTime(kSafePoint, start - pre_safe_point);
  heap_->RecordTime(kMarkObjects, mid1 - start);
  heap_->RecordTime(kResetFreeLists, mid2 - mid1);
  heap_->RecordTime(kSweepPages, mid3 - mid2);
  heap_->RecordTime(kSweepLargePages, end - mid3);

  if (FLAG_print_free_list_after_gc) {
    for (intptr_t i = 0; i < num_freelists_; i++) {
      OS::PrintErr("After GC: Freelist %" Pd "\n", i);
      freelists_[i].Print();
    }
  }

  UpdateMaxUsed();
  if (heap_ != NULL) {
    heap_->UpdateGlobalMaxUsed();
  }
}

// 清理 Large page
void PageSpace::SweepLarge() {
  TIMELINE_FUNCTION_GC_DURATION(Thread::Current(), "SweepLarge");

  GCSweeper sweeper;
  OldPage* prev_page = nullptr;
  OldPage* page = large_pages_;
  while (page != nullptr) {
    OldPage* next_page = page->next();
    const intptr_t words_to_end = sweeper.SweepLargePage(page);
    if (words_to_end == 0) {
      FreeLargePage(page, prev_page);
    } else {
      TruncateLargePage(page, words_to_end << kWordSizeLog2);
      prev_page = page;
    }
    // Advance to the next page.
    page = next_page;
  }
}

// 清理算法实现
void PageSpace::Sweep() {
  TIMELINE_FUNCTION_GC_DURATION(Thread::Current(), "Sweep");

  GCSweeper sweeper;

  intptr_t shard = 0;
  const intptr_t num_shards = Utils::Maximum(FLAG_scavenger_tasks, 1);
  // 锁定空闲分区表
  for (intptr_t i = 0; i < num_shards; i++) {
    DataFreeList(i)->mutex()->Lock();
  }

  OldPage* prev_page = nullptr;
  OldPage* page = pages_;
  while (page != nullptr) {
    OldPage* next_page = page->next();
    ASSERT(page->type() == OldPage::kData);
    shard = (shard + 1) % num_shards;
    // 清理页面，并且返回正在使用的内存大小
    bool page_in_use =
        sweeper.SweepPage(page, DataFreeList(shard), true /*is_locked*/);
    if (page_in_use) {
      prev_page = page;
    } else {
      FreePage(page, prev_page);
    }
    // Advance to the next page.
    page = next_page;
  }

  for (intptr_t i = 0; i < num_shards; i++) {
    DataFreeList(i)->mutex()->Unlock();
  }

  if (FLAG_verify_after_gc) {
    OS::PrintErr("Verifying after sweeping...");
    heap_->VerifyGC(kForbidMarked);
    OS::PrintErr(" done.\n");
  }
}

// 并发清理
void PageSpace::ConcurrentSweep(IsolateGroup* isolate_group) {
  // Start the concurrent sweeper task now.
  GCSweeper::SweepConcurrent(
      isolate_group,
      pages_, 
      pages_tail_, 
      large_pages_,
      large_pages_tail_, 
      &freelists_[OldPage::kData]
  );
}

// 整理
void PageSpace::Compact(Thread* thread) {
  thread->isolate_group()->set_compaction_in_progress(true);
  GCCompactor compactor(thread, heap_);
  compactor.Compact(pages_, &freelists_[OldPage::kData], &pages_lock_);
  thread->isolate_group()->set_compaction_in_progress(false);

  if (FLAG_verify_after_gc) {
    OS::PrintErr("Verifying after compacting...");
    heap_->VerifyGC(kForbidMarked);
    OS::PrintErr(" done.\n");
  }
}

uword PageSpace::TryAllocateDataBumpLocked(FreeList* freelist, intptr_t size) {
  ASSERT(size >= kObjectAlignment);
  ASSERT(Utils::IsAligned(size, kObjectAlignment));

  intptr_t remaining = freelist->end() - freelist->top();
  if (UNLIKELY(remaining < size)) {
    // Checking this first would be logical, but needlessly slow.
    if (!Heap::IsAllocatableViaFreeLists(size)) {
      return TryAllocateDataLocked(freelist, size, kForceGrowth);
    }
    FreeListElement* block = freelist->TryAllocateLargeLocked(size);
    if (block == NULL) {
      // Allocating from a new page (if growth policy allows) will have the
      // side-effect of populating the freelist with a large block. The next
      // bump allocation request will have a chance to consume that block.
      // TODO(koda): Could take freelist lock just once instead of twice.
      return TryAllocateInFreshPage(size, freelist, OldPage::kData,
                                    kForceGrowth, true /* is_locked*/);
    }
    intptr_t block_size = block->HeapSize();
    if (remaining > 0) {
      freelist->FreeLocked(freelist->top(), remaining);
    }
    freelist->set_top(reinterpret_cast<uword>(block));
    freelist->set_end(freelist->top() + block_size);
    remaining = block_size;
  }
  ASSERT(remaining >= size);
  uword result = freelist->top();
  freelist->set_top(result + size);

  freelist->AddUnaccountedSize(size);

// Note: Remaining block is unwalkable until MakeIterable is called.
#ifdef DEBUG
  if (freelist->top() < freelist->end()) {
    // Fail fast if we try to walk the remaining block.
    COMPILE_ASSERT(kIllegalCid == 0);
    *reinterpret_cast<uword*>(freelist->top()) = 0;
  }
#endif  // DEBUG
  return result;
}

uword PageSpace::TryAllocatePromoLockedSlow(FreeList* freelist, intptr_t size) {
  uword result = freelist->TryAllocateSmallLocked(size);
  if (result != 0) {
    freelist->AddUnaccountedSize(size);
    return result;
  }
  return TryAllocateDataBumpLocked(freelist, size);
}

void PageSpace::SetupImagePage(void* pointer, uword size, bool is_executable) {
  // Setup a OldPage so precompiled Instructions can be traversed.
  // Instructions are contiguous at [pointer, pointer + size). OldPage
  // expects to find objects at [memory->start() + ObjectStartOffset,
  // memory->end()).
  uword offset = OldPage::ObjectStartOffset();
  pointer = reinterpret_cast<void*>(reinterpret_cast<uword>(pointer) - offset);
  ASSERT(Utils::IsAligned(pointer, kObjectAlignment));
  size += offset;

  VirtualMemory* memory = VirtualMemory::ForImagePage(pointer, size);
  ASSERT(memory != NULL);
  OldPage* page = reinterpret_cast<OldPage*>(malloc(sizeof(OldPage)));
  page->memory_ = memory;
  page->next_ = NULL;
  page->object_end_ = memory->end();
  page->used_in_bytes_ = page->object_end_ - page->object_start();
  page->forwarding_page_ = NULL;
  page->card_table_ = NULL;
  if (is_executable) {
    page->type_ = OldPage::kExecutable;
  } else {
    page->type_ = OldPage::kData;
  }

  MutexLocker ml(&pages_lock_);
  page->next_ = image_pages_;
  image_pages_ = page;
}

bool PageSpace::IsObjectFromImagePages(dart::ObjectPtr object) {
  uword object_addr = ObjectLayout::ToAddr(object);
  OldPage* image_page = image_pages_;
  while (image_page != nullptr) {
    if (image_page->Contains(object_addr)) {
      return true;
    }
    image_page = image_page->next();
  }
  return false;
}

static void AppendList(OldPage** pages,
                       OldPage** pages_tail,
                       OldPage** other_pages,
                       OldPage** other_pages_tail) {
  ASSERT((*pages == nullptr) == (*pages_tail == nullptr));
  ASSERT((*other_pages == nullptr) == (*other_pages_tail == nullptr));

  if (*other_pages != nullptr) {
    if (*pages_tail == nullptr) {
      *pages = *other_pages;
      *pages_tail = *other_pages_tail;
    } else {
      const bool is_execute = FLAG_write_protect_code &&
                              (*pages_tail)->type() == OldPage::kExecutable;
      if (is_execute) {
        (*pages_tail)->WriteProtect(false);
      }
      (*pages_tail)->set_next(*other_pages);
      if (is_execute) {
        (*pages_tail)->WriteProtect(true);
      }
      *pages_tail = *other_pages_tail;
    }
    *other_pages = nullptr;
    *other_pages_tail = nullptr;
  }
}

static void EnsureEqualImagePages(OldPage* pages, OldPage* other_pages) {
#if defined(DEBUG)
  while (pages != nullptr) {
    ASSERT((pages == nullptr) == (other_pages == nullptr));
    ASSERT(pages->object_start() == other_pages->object_start());
    ASSERT(pages->object_end() == other_pages->object_end());
    pages = pages->next();
    other_pages = other_pages->next();
  }
#endif
}

/// 合并空间
void PageSpace::MergeFrom(PageSpace* donor) {
  donor->AbandonBumpAllocation();

  ASSERT(donor->tasks_ == 0);
  ASSERT(donor->concurrent_marker_tasks_ == 0);
  ASSERT(donor->phase_ == kDone);
  DEBUG_ASSERT(donor->iterating_thread_ == nullptr);
  ASSERT(donor->marker_ == nullptr);

  for (intptr_t i = 0; i < num_freelists_; ++i) {
    ASSERT(donor->freelists_[i].top() == 0);
    ASSERT(donor->freelists_[i].end() == 0);
    const bool is_protected =
        FLAG_write_protect_code && i == OldPage::kExecutable;
    freelists_[i].MergeFrom(&donor->freelists_[i], is_protected);
    donor->freelists_[i].Reset();
  }

  // The freelist locks will be taken in MergeOtherFreelist above, and the
  // locking order is the freelist locks are taken before the page list locks,
  // so don't take the pages lock until after MergeOtherFreelist.
  MutexLocker ml(&pages_lock_);
  MutexLocker ml2(&donor->pages_lock_);

  AppendList(&pages_, &pages_tail_, &donor->pages_, &donor->pages_tail_);
  AppendList(&exec_pages_, &exec_pages_tail_, &donor->exec_pages_,
             &donor->exec_pages_tail_);
  AppendList(&large_pages_, &large_pages_tail_, &donor->large_pages_,
             &donor->large_pages_tail_);
  // We intentionall do not merge [image_pages_] beause [this] and [other] have
  // the same mmap()ed image page areas.
  EnsureEqualImagePages(image_pages_, donor->image_pages_);

  // We intentionaly do not increase [max_capacity_in_words_] because this can
  // lead [max_capacity_in_words_] to become larger and larger and eventually
  // wrap-around and become negative.
  allocated_black_in_words_ += donor->allocated_black_in_words_;
  gc_time_micros_ += donor->gc_time_micros_;
  collections_ += donor->collections_;

  usage_.capacity_in_words += donor->usage_.capacity_in_words;
  usage_.used_in_words += donor->usage_.used_in_words;
  usage_.external_in_words += donor->usage_.external_in_words;

  page_space_controller_.MergeFrom(&donor->page_space_controller_);

  ASSERT(FLAG_concurrent_mark || donor->enable_concurrent_mark_ == false);
}

PageSpaceController::PageSpaceController(Heap* heap,
                                         int heap_growth_ratio,
                                         int heap_growth_max,
                                         int garbage_collection_time_ratio)
    : heap_(heap),
      is_enabled_(false),
      heap_growth_ratio_(heap_growth_ratio),
      desired_utilization_((100.0 - heap_growth_ratio) / 100.0),
      heap_growth_max_(heap_growth_max),
      garbage_collection_time_ratio_(garbage_collection_time_ratio),
      idle_gc_threshold_in_words_(0) {
  const intptr_t growth_in_pages = heap_growth_max / 2;
  RecordUpdate(last_usage_, last_usage_, growth_in_pages, "initial");
}

PageSpaceController::~PageSpaceController() {}

bool PageSpaceController::ReachedHardThreshold(SpaceUsage after) const {
  if (!is_enabled_) {
    return false;
  }
  if (heap_growth_ratio_ == 100) {
    return false;
  }
  return after.CombinedUsedInWords() > hard_gc_threshold_in_words_;
}

bool PageSpaceController::ReachedSoftThreshold(SpaceUsage after) const {
  if (!is_enabled_) {
    return false;
  }
  if (heap_growth_ratio_ == 100) {
    return false;
  }
  return after.CombinedUsedInWords() > soft_gc_threshold_in_words_;
}

bool PageSpaceController::ReachedIdleThreshold(SpaceUsage current) const {
  if (!is_enabled_) {
    return false;
  }
  if (heap_growth_ratio_ == 100) {
    return false;
  }
  return current.CombinedUsedInWords() > idle_gc_threshold_in_words_;
}

void PageSpaceController::EvaluateGarbageCollection(SpaceUsage before,
                                                    SpaceUsage after,
                                                    int64_t start,
                                                    int64_t end) {
  ASSERT(end >= start);
  history_.AddGarbageCollectionTime(start, end);
  const int gc_time_fraction = history_.GarbageCollectionTimeFraction();
  heap_->RecordData(PageSpace::kGCTimeFraction, gc_time_fraction);

  // Assume garbage increases linearly with allocation:
  // G = kA, and estimate k from the previous cycle.
  const intptr_t allocated_since_previous_gc =
      before.CombinedUsedInWords() - last_usage_.CombinedUsedInWords();
  intptr_t grow_heap;
  if (allocated_since_previous_gc > 0) {
    const intptr_t garbage =
        before.CombinedUsedInWords() - after.CombinedUsedInWords();
    ASSERT(garbage >= 0);
    // It makes no sense to expect that each kb allocated will cause more than
    // one kb of garbage, so we clamp k at 1.0.
    const double k = Utils::Minimum(
        1.0, garbage / static_cast<double>(allocated_since_previous_gc));

    const int garbage_ratio = static_cast<int>(k * 100);
    heap_->RecordData(PageSpace::kGarbageRatio, garbage_ratio);

    // Define GC to be 'worthwhile' iff at least fraction t of heap is garbage.
    double t = 1.0 - desired_utilization_;
    // If we spend too much time in GC, strive for even more free space.
    if (gc_time_fraction > garbage_collection_time_ratio_) {
      t += (gc_time_fraction - garbage_collection_time_ratio_) / 100.0;
    }

    // Number of pages we can allocate and still be within the desired growth
    // ratio.
    const intptr_t grow_pages =
        (static_cast<intptr_t>(after.CombinedUsedInWords() /
                               desired_utilization_) -
         (after.CombinedUsedInWords())) /
        kOldPageSizeInWords;
    if (garbage_ratio == 0) {
      // No garbage in the previous cycle so it would be hard to compute a
      // grow_heap size based on estimated garbage so we use growth ratio
      // heuristics instead.
      grow_heap =
          Utils::Maximum(static_cast<intptr_t>(heap_growth_max_), grow_pages);
    } else {
      // Find minimum 'grow_heap' such that after increasing capacity by
      // 'grow_heap' pages and filling them, we expect a GC to be worthwhile.
      intptr_t max = heap_growth_max_;
      intptr_t min = 0;
      intptr_t local_grow_heap = 0;
      while (min < max) {
        local_grow_heap = (max + min) / 2;
        const intptr_t limit = after.CombinedUsedInWords() +
                               (local_grow_heap * kOldPageSizeInWords);
        const intptr_t allocated_before_next_gc =
            limit - (after.CombinedUsedInWords());
        const double estimated_garbage = k * allocated_before_next_gc;
        if (t <= estimated_garbage / limit) {
          max = local_grow_heap - 1;
        } else {
          min = local_grow_heap + 1;
        }
      }
      local_grow_heap = (max + min) / 2;
      grow_heap = local_grow_heap;
      ASSERT(grow_heap >= 0);
      // If we are going to grow by heap_grow_max_ then ensure that we
      // will be growing the heap at least by the growth ratio heuristics.
      if (grow_heap >= heap_growth_max_) {
        grow_heap = Utils::Maximum(grow_pages, grow_heap);
      }
    }
  } else {
    heap_->RecordData(PageSpace::kGarbageRatio, 100);
    grow_heap = 0;
  }
  heap_->RecordData(PageSpace::kPageGrowth, grow_heap);
  last_usage_ = after;

  intptr_t max_capacity_in_words = heap_->old_space()->max_capacity_in_words_;
  if (max_capacity_in_words != 0) {
    ASSERT(grow_heap >= 0);
    // Fraction of asymptote used.
    double f = static_cast<double>(after.CombinedUsedInWords() +
                                   (kOldPageSizeInWords * grow_heap)) /
               static_cast<double>(max_capacity_in_words);
    ASSERT(f >= 0.0);
    // Increase weight at the high end.
    f = f * f;
    // Fraction of asymptote available.
    f = 1.0 - f;
    ASSERT(f <= 1.0);
    // Discount growth more the closer we get to the desired asymptote.
    grow_heap = static_cast<intptr_t>(grow_heap * f);
    // Minimum growth step after reaching the asymptote.
    intptr_t min_step = (2 * MB) / kOldPageSize;
    grow_heap = Utils::Maximum(min_step, grow_heap);
  }

  RecordUpdate(before, after, grow_heap, "gc");
}

void PageSpaceController::EvaluateAfterLoading(SpaceUsage after) {
  // Number of pages we can allocate and still be within the desired growth
  // ratio.
  intptr_t growth_in_pages;
  if (desired_utilization_ == 0.0) {
    growth_in_pages = heap_growth_max_;
  } else {
    growth_in_pages = (static_cast<intptr_t>(after.CombinedUsedInWords() /
                                             desired_utilization_) -
                       (after.CombinedUsedInWords())) /
                      kOldPageSizeInWords;
  }

  // Apply growth cap.
  growth_in_pages =
      Utils::Minimum(static_cast<intptr_t>(heap_growth_max_), growth_in_pages);

  RecordUpdate(after, after, growth_in_pages, "loaded");
}

void PageSpaceController::RecordUpdate(SpaceUsage before,
                                       SpaceUsage after,
                                       intptr_t growth_in_pages,
                                       const char* reason) {
  // Save final threshold compared before growing.
  hard_gc_threshold_in_words_ =
      after.CombinedUsedInWords() + (kOldPageSizeInWords * growth_in_pages);

  // Start concurrent marking when old-space has less than half of new-space
  // available or less than 5% available.
#if defined(TARGET_ARCH_IA32)
  const intptr_t headroom = 0;  // No concurrent marking.
#else
  // Note that heap_ can be null in some unit tests.
  const intptr_t new_space =
      heap_ == nullptr ? 0 : heap_->new_space()->CapacityInWords();
  const intptr_t headroom =
      Utils::Maximum(new_space / 2, hard_gc_threshold_in_words_ / 20);
#endif
  soft_gc_threshold_in_words_ = hard_gc_threshold_in_words_ - headroom;

  // Set a tight idle threshold.
  idle_gc_threshold_in_words_ =
      after.CombinedUsedInWords() + (2 * kOldPageSizeInWords);

#if defined(SUPPORT_TIMELINE)
  Thread* thread = Thread::Current();
  if (thread != nullptr) {
    TIMELINE_FUNCTION_GC_DURATION(thread, "UpdateGrowthLimit");
    tbes.SetNumArguments(6);
    tbes.CopyArgument(0, "Reason", reason);
    tbes.FormatArgument(1, "Before.CombinedUsed (kB)", "%" Pd "",
                        RoundWordsToKB(before.CombinedUsedInWords()));
    tbes.FormatArgument(2, "After.CombinedUsed (kB)", "%" Pd "",
                        RoundWordsToKB(after.CombinedUsedInWords()));
    tbes.FormatArgument(3, "Hard Threshold (kB)", "%" Pd "",
                        RoundWordsToKB(hard_gc_threshold_in_words_));
    tbes.FormatArgument(4, "Soft Threshold (kB)", "%" Pd "",
                        RoundWordsToKB(soft_gc_threshold_in_words_));
    tbes.FormatArgument(5, "Idle Threshold (kB)", "%" Pd "",
                        RoundWordsToKB(idle_gc_threshold_in_words_));
  }
#endif

  if (FLAG_log_growth) {
    THR_Print("%s: threshold=%" Pd "kB, idle_threshold=%" Pd "kB, reason=%s\n",
              heap_->isolate_group()->source()->name,
              hard_gc_threshold_in_words_ / KBInWords,
              idle_gc_threshold_in_words_ / KBInWords, reason);
  }
}

void PageSpaceController::HintFreed(intptr_t size) {
  intptr_t size_in_words = size << kWordSizeLog2;
  if (size_in_words > idle_gc_threshold_in_words_) {
    idle_gc_threshold_in_words_ = 0;
  } else {
    idle_gc_threshold_in_words_ -= size_in_words;
  }

  // TODO(rmacnak): Hasten the soft threshold at some discount?
}

void PageSpaceController::MergeFrom(PageSpaceController* donor) {
  last_usage_.capacity_in_words += donor->last_usage_.capacity_in_words;
  last_usage_.used_in_words += donor->last_usage_.used_in_words;
  last_usage_.external_in_words += donor->last_usage_.external_in_words;
}

void PageSpaceGarbageCollectionHistory::AddGarbageCollectionTime(int64_t start,
                                                                 int64_t end) {
  Entry entry;
  entry.start = start;
  entry.end = end;
  history_.Add(entry);
}

int PageSpaceGarbageCollectionHistory::GarbageCollectionTimeFraction() {
  int64_t gc_time = 0;
  int64_t total_time = 0;
  for (int i = 0; i < history_.Size() - 1; i++) {
    Entry current = history_.Get(i);
    Entry previous = history_.Get(i + 1);
    gc_time += current.end - current.start;
    total_time += current.end - previous.end;
  }
  if (total_time == 0) {
    return 0;
  } else {
    ASSERT(total_time >= gc_time);
    int result = static_cast<int>(
        (static_cast<double>(gc_time) / static_cast<double>(total_time)) * 100);
    return result;
  }
}

}  // namespace dart
```

