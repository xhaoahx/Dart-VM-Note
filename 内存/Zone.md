# Zone

## zone.h

```c++
// ...

namespace dart {

// Zone 支持小块内存的快速分配。分配出去的内存块不能够逐一释放，但是可以在一次操作中释放所有的块
class Zone {
 public:
  // 分配以 ElementType 为元素、以 len 为长度的数组
  template <class ElementType>
  inline ElementType* Alloc(intptr_t len);

  // 将给定的 old_array 重分配长度
  template <class ElementType>
  inline ElementType* Realloc(ElementType* old_array,
                              intptr_t old_len,
                              intptr_t new_len);

  // 在 Zone 上分配 size byte 
  inline uword AllocUnsafe(intptr_t size);

  // 在 zone 上分配拷贝的串
  char* MakeCopyOfString(const char* str);

  // 拷贝给定串的前 N 个字符
  char* MakeCopyOfStringN(const char* str, intptr_t len);

  // 拼接串
  char* ConcatStrings(const char* a, const char* b, char join = ',');

  // 。。。
  char* PrintToString(const char* format, ...) PRINTF_ATTRIBUTE(2, 3);
  char* VPrint(const char* format, va_list args);

  // 计算此 zone 的大小
  uintptr_t SizeInBytes() const;

  // 计算使用的大小
  uintptr_t CapacityInBytes() const;

  // 管理 Handle 分配
  VMHandles* handles() { return &handles_; }

  void VisitObjectPointers(ObjectPointerVisitor* visitor);

  // 上一 zone
  Zone* previous() const { return previous_; }

  // 向前查找给定的 zone
  bool ContainsNestedZone(Zone* other) const {
    while (other != NULL) {
      if (this == other) return true;
      other = other->previous_;
    }
    return false;
  }

  // 从 AllocUnsafe 和 New 分配的地址都符合此内存排列
  static const intptr_t kAlignment = kDoubleSize;

  static void Init();
  static void Cleanup();

  static intptr_t Size() { return total_size_; }

 private:
  Zone();
  ~Zone();  // Delete all memory associated with the zone.

  // 默认块大小
  static const intptr_t kInitialChunkSize = 1 * KB;

  // 默认段大小
  static const intptr_t kSegmentSize = 64 * KB;

  // Zap value used to indicate deleted zone area (debug purposes).
  static const unsigned char kZapDeletedByte = 0x42;

  // Zap value used to indicate uninitialized zone area (debug purposes).
  static const unsigned char kZapUninitializedByte = 0xab;

  // Total size of current zone segments.
  static RelaxedAtomic<intptr_t> total_size_;

  // 拓展 zone 以容纳给定 size
  uword AllocateExpand(intptr_t size);

  // 分配一个 Large segment
  uword AllocateLargeSegment(intptr_t size);

  // 将此 zone 加入 zone 链，在 current 后面
  void Link(Zone* current_zone) { previous_ = current_zone; }

  // 释放此 zone 里的所有内存
  void DeleteAll();

  // 并不释放任何的内存，只做简单的标记
  template <class ElementType>
  void Free(ElementType* old_array, intptr_t len) {
#ifdef DEBUG
    if (len > 0) {
      ASSERT(old_array != nullptr);
      memset(static_cast<void*>(old_array), kZapUninitializedByte,
             len * sizeof(ElementType));
    }
#endif
  }

  // Dump the current allocated sizes in the zone object.
  void DumpZoneSizes();

  // Overflow check (FATAL) for array length.
  template <class ElementType>
  static inline void CheckLength(intptr_t len);

  // This buffer is used for allocation before any segments.
  // This would act as the initial stack allocated chunk so that we don't
  // end up calling malloc/free on zone scopes that allocate less than
  // kChunkSize
  COMPILE_ASSERT(kAlignment <= 8);
  ALIGN8 uint8_t buffer_[kInitialChunkSize];
  MemoryRegion initial_buffer_;

  // The free region in the current (head) segment or the initial buffer is
  // represented as the half-open interval [position, limit). The 'position'
  // variable is guaranteed to be aligned as dictated by kAlignment.
  uword position_;
  uword limit_;

  // Zone segments are internal data structures used to hold information
  // about the memory segmentations that constitute a zone. The entire
  // implementation is in zone.cc.
  class Segment;

  // Total size of all segments in [head_].
  intptr_t small_segment_capacity_ = 0;

  // The current head segment; may be NULL.
  Segment* head_;

  // 在此 zone 中分配的大 segment，可能为 NULL
  Segment* large_segments_;

  // 管理 Handle 分配
  VMHandles handles_;

  // Used for chaining zones in order to allow unwinding of stacks.
  Zone* previous_;

  friend class StackZone;
  friend class ApiZone;
  template <typename T, typename B, typename Allocator>
  friend class BaseGrowableArray;
  template <typename T, typename B, typename Allocator>
  friend class BaseDirectChainedHashMap;
  DISALLOW_COPY_AND_ASSIGN(Zone);
};

 
/// 栈上分配的 Zone
class StackZone : public StackResource {
 public:
  // Create an empty zone and set is at the current zone for the Thread.
  explicit StackZone(ThreadState* thread);

  // Delete all memory associated with the zone.
  virtual ~StackZone();

  // Compute the total size of this zone. This includes wasted space that is
  // due to internal fragmentation in the segments.
  uintptr_t SizeInBytes() const { return zone_->SizeInBytes(); }

  // Computes the used space in the zone.
  intptr_t CapacityInBytes() const { return zone_->CapacityInBytes(); }

  Zone* GetZone() { return zone_; }

 private:
  // 持有 zone
  Zone* zone_;

  template <typename T>
  friend class GrowableArray;
  template <typename T>
  friend class ZoneGrowableArray;

  DISALLOW_IMPLICIT_CONSTRUCTORS(StackZone);
};

// 分配给定 size byte 大小的内存
inline uword Zone::AllocUnsafe(intptr_t size) {
  ASSERT(size >= 0);
  // Round up the requested size to fit the alignment.
  if (size > (kIntptrMax - kAlignment)) {
    FATAL1("Zone::Alloc: 'size' is too large: size=%" Pd "", size);
  }
  // 按 kAlignment 排列的 size
  size = Utils::RoundUp(size, kAlignment);

  // 检查给定的 size 是否在不需要扩展的情况下，能否分配
  uword result;
  // 可用大小
  intptr_t free_size = (limit_ - position_);
  if (free_size >= size) {
    result = position_;
    position_ += size;
  } 
  // 若 size 不足，则需要扩展  
  else {
    result = AllocateExpand(size);
  }

  // Check that the result has the proper alignment and return it.
  ASSERT(Utils::IsAligned(result, kAlignment));
  return result;
}

template <class ElementType>
inline void Zone::CheckLength(intptr_t len) {
  const intptr_t kElementSize = sizeof(ElementType);
  if (len > (kIntptrMax / kElementSize)) {
    FATAL2("Zone::Alloc: 'len' is too large: len=%" Pd ", kElementSize=%" Pd,
           len, kElementSize);
  }
}

// Alloc 实现
template <class ElementType>
inline ElementType* Zone::Alloc(intptr_t len) {
  CheckLength<ElementType>(len);
  return reinterpret_cast<ElementType*>(AllocUnsafe(len * sizeof(ElementType)));
}

// Realloc 实现
template <class ElementType>
inline ElementType* Zone::Realloc(ElementType* old_data,
                                  intptr_t old_len,
                                  intptr_t new_len) {
  CheckLength<ElementType>(new_len);
  const intptr_t kElementSize = sizeof(ElementType);
  uword old_end = reinterpret_cast<uword>(old_data) + (old_len * kElementSize);
  // Resize existing allocation if nothing was allocated in between...
  if (Utils::RoundUp(old_end, kAlignment) == position_) {
    uword new_end =
        reinterpret_cast<uword>(old_data) + (new_len * kElementSize);
    // ...and there is sufficient space.
    if (new_end <= limit_) {
      ASSERT(new_len >= old_len);
      position_ = Utils::RoundUp(new_end, kAlignment);
      return old_data;
    }
  }
  if (new_len <= old_len) {
    return old_data;
  }
  ElementType* new_data = Alloc<ElementType>(new_len);
  if (old_data != 0) {
    memmove(reinterpret_cast<void*>(new_data),
            reinterpret_cast<void*>(old_data), old_len * kElementSize);
  }
  return new_data;
}

}  // namespace dart

#endif  // RUNTIME_VM_ZONE_H_

```

## zone.cc

```c++
// ...

namespace dart {

RelaxedAtomic<intptr_t> Zone::total_size_ = {0};

// 区域段表示内存块：它们具有在此指针中编码的起始地址和大小（以字节为单位）。它们被链接在一起，形成扩展区域的后备存储。
class Zone::Segment {
 public:
  Segment* next() const { return next_; }
  intptr_t size() const { return size_; }
  // 持有的 VirtualMemory 对象
  VirtualMemory* memory() const { return memory_; }

  uword start() { return address(sizeof(Segment)); }
  uword end() { return address(size_); }

  // 分配或者删除 Segment
  static Segment* New(intptr_t size, Segment* next);
  static void DeleteSegmentList(Segment* segment);
  static void IncrementMemoryCapacity(uintptr_t size);
  static void DecrementMemoryCapacity(uintptr_t size);

 private:
  Segment* next_;
  intptr_t size_;
  VirtualMemory* memory_;
  void* alignment_;

  // Computes the address of the nth byte in this segment.
  uword address(intptr_t n) { return reinterpret_cast<uword>(this) + n; }

  DISALLOW_IMPLICIT_CONSTRUCTORS(Segment);
};

// tcmalloc and jemalloc have both been observed to hold onto lots of free'd
// zone segments (jemalloc to the point of causing OOM), so instead of using
// malloc to allocate segments, we allocate directly from mmap/zx_vmo_create/
// VirtualAlloc, and cache a small number of the normal sized segments.
static constexpr intptr_t kSegmentCacheCapacity = 16;  // 1 MB of Segments
static Mutex* segment_cache_mutex = nullptr;
static VirtualMemory* segment_cache[kSegmentCacheCapacity] = {nullptr};
static intptr_t segment_cache_size = 0;


void Zone::Init() {
  ASSERT(segment_cache_mutex == nullptr);
  segment_cache_mutex = new Mutex(NOT_IN_PRODUCT("segment_cache_mutex"));
}

/// 删除所有的 segment_cache
void Zone::Cleanup() {
  {
    MutexLocker ml(segment_cache_mutex);
    ASSERT(segment_cache_size >= 0);
    ASSERT(segment_cache_size <= kSegmentCacheCapacity);
    while (segment_cache_size > 0) {
      delete segment_cache[--segment_cache_size];
    }
  }
  delete segment_cache_mutex;
  segment_cache_mutex = nullptr;
}

// 创建新的 segment
Zone::Segment* Zone::Segment::New(intptr_t size, Zone::Segment* next) {
  size = Utils::RoundUp(size, VirtualMemory::PageSize());
  VirtualMemory* memory = nullptr;
  // 如果 size == kSegmentSize，则在缓存中取出一块
  if (size == kSegmentSize) {
    MutexLocker ml(segment_cache_mutex);
    ASSERT(segment_cache_size >= 0);
    ASSERT(segment_cache_size <= kSegmentCacheCapacity);
    if (segment_cache_size > 0) {
      memory = segment_cache[--segment_cache_size];
    }
  }
  // 如果缓存无法分配
  if (memory == nullptr) {
    // 分配虚拟内存
    memory = VirtualMemory::Allocate(size, false, "dart-zone");
    // 增加总容量
    total_size_.fetch_add(size);
  }
  if (memory == nullptr) {
    OUT_OF_MEMORY();
  }
    
  // 将给定内存的 start 转换为 Segment 指针，即在 start 出存放 segment 数据
  /// 在 Page 中也由有类似实现
  Segment* result = reinterpret_cast<Segment*>(memory->start());
#ifdef DEBUG
  // Zap the entire allocated segment (including the header).
  memset(reinterpret_cast<void*>(result), kZapUninitializedByte, size);
#endif
  result->next_ = next;
  result->size_ = size;
  result->memory_ = memory;
  result->alignment_ = nullptr;  // Avoid unused variable warnings.

  LSAN_REGISTER_ROOT_REGION(result, sizeof(*result));

  IncrementMemoryCapacity(size);
  return result;
}

void Zone::Segment::DeleteSegmentList(Segment* head) {
  Segment* current = head;
  while (current != NULL) {
    intptr_t size = current->size();
    DecrementMemoryCapacity(size);
    Segment* next = current->next();
    VirtualMemory* memory = current->memory();
#ifdef DEBUG
    // Zap the entire current segment (including the header).
    memset(reinterpret_cast<void*>(current), kZapDeletedByte, current->size());
#endif
    LSAN_UNREGISTER_ROOT_REGION(current, sizeof(*current));

    if (size == kSegmentSize) {
      MutexLocker ml(segment_cache_mutex);
      ASSERT(segment_cache_size >= 0);
      ASSERT(segment_cache_size <= kSegmentCacheCapacity);
      if (segment_cache_size < kSegmentCacheCapacity) {
        segment_cache[segment_cache_size++] = memory;
        memory = nullptr;
      }
    }
    if (memory != nullptr) {
      total_size_.fetch_sub(size);
      delete memory;
    }
    current = next;
  }
}

void Zone::Segment::IncrementMemoryCapacity(uintptr_t size) {
  ThreadState* current_thread = ThreadState::Current();
  if (current_thread != NULL) {
    current_thread->IncrementMemoryCapacity(size);
  } else if (ApiNativeScope::Current() != NULL) {
    // If there is no current thread, we might be inside of a native scope.
    ApiNativeScope::IncrementNativeScopeMemoryCapacity(size);
  }
}

void Zone::Segment::DecrementMemoryCapacity(uintptr_t size) {
  ThreadState* current_thread = ThreadState::Current();
  if (current_thread != NULL) {
    current_thread->DecrementMemoryCapacity(size);
  } else if (ApiNativeScope::Current() != NULL) {
    // If there is no current thread, we might be inside of a native scope.
    ApiNativeScope::DecrementNativeScopeMemoryCapacity(size);
  }
}

// TODO(bkonyi): We need to account for the initial chunk size when a new zone
// is created within a new thread or ApiNativeScope when calculating high
// watermarks or memory consumption.
Zone::Zone()
    : initial_buffer_(buffer_, kInitialChunkSize),
      position_(initial_buffer_.start()),
      limit_(initial_buffer_.end()),
      head_(NULL),
      large_segments_(NULL),
      handles_(),
      previous_(NULL) {
  ASSERT(Utils::IsAligned(position_, kAlignment));
  Segment::IncrementMemoryCapacity(kInitialChunkSize);
#ifdef DEBUG
  // Zap the entire initial buffer.
  memset(initial_buffer_.pointer(), kZapUninitializedByte,
         initial_buffer_.size());
#endif
}

Zone::~Zone() {
  if (FLAG_trace_zones) {
    DumpZoneSizes();
  }
  DeleteAll();
  Segment::DecrementMemoryCapacity(kInitialChunkSize);
}

void Zone::DeleteAll() {
  // Traverse the chained list of segments, zapping (in debug mode)
  // and freeing every zone segment.
  if (head_ != NULL) {
    Segment::DeleteSegmentList(head_);
  }
  if (large_segments_ != NULL) {
    Segment::DeleteSegmentList(large_segments_);
  }
// Reset zone state.
#ifdef DEBUG
  memset(initial_buffer_.pointer(), kZapDeletedByte, initial_buffer_.size());
#endif
  position_ = initial_buffer_.start();
  limit_ = initial_buffer_.end();
  small_segment_capacity_ = 0;
  head_ = NULL;
  large_segments_ = NULL;
  previous_ = NULL;
  handles_.Reset();
}

uintptr_t Zone::SizeInBytes() const {
  uintptr_t size = 0;
  for (Segment* s = large_segments_; s != NULL; s = s->next()) {
    size += s->size();
  }
  if (head_ == NULL) {
    return size + (position_ - initial_buffer_.start());
  }
  size += initial_buffer_.size();
  for (Segment* s = head_->next(); s != NULL; s = s->next()) {
    size += s->size();
  }
  return size + (position_ - head_->start());
}

uintptr_t Zone::CapacityInBytes() const {
  uintptr_t size = 0;
  for (Segment* s = large_segments_; s != NULL; s = s->next()) {
    size += s->size();
  }
  if (head_ == NULL) {
    return size + initial_buffer_.size();
  }
  size += initial_buffer_.size();
  for (Segment* s = head_; s != NULL; s = s->next()) {
    size += s->size();
  }
  return size;
}

uword Zone::AllocateExpand(intptr_t size) {
  ASSERT(size >= 0);
  if (FLAG_trace_zones) {
    OS::PrintErr("*** Expanding zone 0x%" Px "\n",
                 reinterpret_cast<intptr_t>(this));
    DumpZoneSizes();
  }
  // Make sure the requested size is already properly aligned and that
  // there isn't enough room in the Zone to satisfy the request.
  ASSERT(Utils::IsAligned(size, kAlignment));
  intptr_t free_size = (limit_ - position_);
  ASSERT(free_size < size);

  // 每个 Segment 最多分配 max_size 的内存
  intptr_t max_size =
      Utils::RoundDown(kSegmentSize - sizeof(Segment), kAlignment);
  ASSERT(max_size > 0);
  // 需求的 size > Segment 的最大内存
  if (size > max_size) {
    return AllocateLargeSegment(size);
  }

  const intptr_t kSuperPageSize = 2 * MB;
  intptr_t next_size;
  if (small_segment_capacity_ < kSuperPageSize) {
    // When the Zone is small, grow linearly to reduce size and use the segment
    // cache to avoid expensive mmap calls.
    next_size = kSegmentSize;
  } else {
    // When the Zone is large, grow geometrically to avoid Page Table Entry
    // exhaustion. Using 1.125 ratio.
    next_size = Utils::RoundUp(small_segment_capacity_ >> 3, kSuperPageSize);
  }
  ASSERT(next_size >= kSegmentSize);

  // Allocate another segment and chain it up.
  head_ = Segment::New(next_size, head_);
  small_segment_capacity_ += next_size;

  // Recompute 'position' and 'limit' based on the new head segment.
  uword result = Utils::RoundUp(head_->start(), kAlignment);
  position_ = result + size;
  limit_ = head_->end();
  ASSERT(position_ <= limit_);
  return result;
}

uword Zone::AllocateLargeSegment(intptr_t size) {
  ASSERT(size >= 0);
  // Make sure the requested size is already properly aligned and that
  // there isn't enough room in the Zone to satisfy the request.
  ASSERT(Utils::IsAligned(size, kAlignment));
  intptr_t free_size = (limit_ - position_);
  ASSERT(free_size < size);

  // Create a new large segment and chain it up.
  // Account for book keeping fields in size.
  size += Utils::RoundUp(sizeof(Segment), kAlignment);
  large_segments_ = Segment::New(size, large_segments_);

  uword result = Utils::RoundUp(large_segments_->start(), kAlignment);
  return result;
}

/// 串拷贝
char* Zone::MakeCopyOfString(const char* str) {
  intptr_t len = strlen(str) + 1;  // '\0'-terminated.
  char* copy = Alloc<char>(len);
  strncpy(copy, str, len);
  return copy;
}

char* Zone::MakeCopyOfStringN(const char* str, intptr_t len) {
  ASSERT(len >= 0);
  for (intptr_t i = 0; i < len; i++) {
    if (str[i] == '\0') {
      len = i;
      break;
    }
  }
  char* copy = Alloc<char>(len + 1);  // +1 for '\0'
  strncpy(copy, str, len);
  copy[len] = '\0';
  return copy;
}

char* Zone::ConcatStrings(const char* a, const char* b, char join) {
  intptr_t a_len = (a == NULL) ? 0 : strlen(a);
  const intptr_t b_len = strlen(b) + 1;  // '\0'-terminated.
  const intptr_t len = a_len + b_len;
  char* copy = Alloc<char>(len);
  if (a_len > 0) {
    strncpy(copy, a, a_len);
    // Insert join character.
    copy[a_len++] = join;
  }
  strncpy(&copy[a_len], b, b_len);
  return copy;
}

void Zone::DumpZoneSizes() {
  intptr_t size = 0;
  for (Segment* s = large_segments_; s != NULL; s = s->next()) {
    size += s->size();
  }
  OS::PrintErr("***   Zone(0x%" Px
               ") size in bytes,"
               " Total = %" Pd " Large Segments = %" Pd "\n",
               reinterpret_cast<intptr_t>(this), SizeInBytes(), size);
}

void Zone::VisitObjectPointers(ObjectPointerVisitor* visitor) {
  Zone* zone = this;
  while (zone != NULL) {
    zone->handles()->VisitObjectPointers(visitor);
    zone = zone->previous_;
  }
}

char* Zone::PrintToString(const char* format, ...) {
  va_list args;
  va_start(args, format);
  char* buffer = OS::VSCreate(this, format, args);
  va_end(args);
  return buffer;
}

char* Zone::VPrint(const char* format, va_list args) {
  return OS::VSCreate(this, format, args);
}

StackZone::StackZone(ThreadState* thread)
    : StackResource(thread), 
    // 构造创建新 Zone
    zone_(new Zone()) {
  if (FLAG_trace_zones) {
    OS::PrintErr("*** Starting a new Stack zone 0x%" Px "(0x%" Px ")\n",
                 reinterpret_cast<intptr_t>(this),
                 reinterpret_cast<intptr_t>(zone_));
  }

  // This thread must be preventing safepoints or the GC could be visiting the
  // chain of handle blocks we're about the mutate.
  ASSERT(Thread::Current()->MayAllocateHandles());

  /// 相互引用
  zone_->Link(thread->zone());
  thread->set_zone(zone_);
}

StackZone::~StackZone() {
  // This thread must be preventing safepoints or the GC could be visiting the
  // chain of handle blocks we're about the mutate.
  ASSERT(Thread::Current()->MayAllocateHandles());

  ASSERT(thread()->zone() == zone_);
  thread()->set_zone(zone_->previous_);
  if (FLAG_trace_zones) {
    OS::PrintErr("*** Deleting Stack zone 0x%" Px "(0x%" Px ")\n",
                 reinterpret_cast<intptr_t>(this),
                 reinterpret_cast<intptr_t>(zone_));
  }

  delete zone_;
}

}  // namespace dart

```

