# FreeList

## freelist.h

```c++
// ...

namespace dart {

// FreeListElement describes a freelist element.  Smallest FreeListElement is
// two words in size.  Second word of the raw object is used to keep a next_
// pointer to chain elements of the list together. For objects larger than the
// object size encodable in tags field, the size of the element is embedded in
// the element at the address following the next_ field. All words written by
// the freelist are guaranteed to look like Smis.
// A FreeListElement never has its header mark bit set.

// FreeListElement 描述一个 可利用空间表元素。最小的 FreeListElement 的大小是两个字。raw 对象的第二个单词用于保持一个 
// next_指针，该指针指向链表中的元素。对于大于 tag 字段中可编码的对象大小的对象，元素的大小被嵌入到 next_ 字段后面的地址中。
// 自由职业者编写的所有单词都保证看起来像smi。
// FreeListElement 从不设置其头标记位。
class FreeListElement {
 public:
  /// 下一元素
  FreeListElement* next() const { return next_; }
  uword next_address() const { return reinterpret_cast<uword>(&next_); }

  // 设置下一元素
  void set_next(FreeListElement* next) { next_ = next; }

  intptr_t HeapSize() {
    intptr_t size = ObjectLayout::SizeTag::decode(tags_);
    if (size != 0) return size;
    return *SizeAddress();
  }

  // 从地址、大小构造 FreeListElement
  static FreeListElement* AsElement(uword addr, intptr_t size);

  static void Init();

  static intptr_t HeaderSizeFor(intptr_t size);

  // 用于在 Object::InitOnce 中为空闲列表元素分配类。
  class FakeInstance {
   public:
    FakeInstance() {}
    static cpp_vtable vtable() { return 0; }
    static intptr_t InstanceSize() { return 0; }
    static intptr_t NextFieldOffset() { return -kWordSize; }
    static const ClassId kClassId = kFreeListElement;
    static bool IsInstance() { return true; }

   private:
    DISALLOW_ALLOCATION();
    DISALLOW_COPY_AND_ASSIGN(FakeInstance);
  };

 private:
  // 此 Layout 映射了 RawObject Layout
  RelaxedAtomic<uint32_t> tags_;
#if defined(HASH_IN_OBJECT_HEADER)
  uint32_t hash_;
#endif
  FreeListElement* next_;

  // Returns the address of the embedded size.
  intptr_t* SizeAddress() const {
    uword addr = reinterpret_cast<uword>(&next_) + kWordSize;
    return reinterpret_cast<intptr_t*>(addr);
  }

  // FreeListElements cannot be allocated. Instead references to them are
  // created using the AsElement factory method.
  DISALLOW_ALLOCATION();
  DISALLOW_IMPLICIT_CONSTRUCTORS(FreeListElement);
};

/// 空闲分区表
class FreeList {
 public:
  FreeList();
  ~FreeList();

  // 在空闲分区表上分配 size 大小的空间
  uword TryAllocate(intptr_t size, bool is_protected);
  // 将给定地址上 size 大小的空间归还给分区表
  void Free(uword addr, intptr_t size);

  void Reset();

  void Print() const;

  Mutex* mutex() { return &mutex_; }
  uword TryAllocateLocked(intptr_t size, bool is_protected);
  void FreeLocked(uword addr, intptr_t size);

  // 返回一个较大的元素，至少返回 'minimum_size'，如果不存在，则返回 NULL。
  FreeListElement* TryAllocateLarge(intptr_t minimum_size);
  FreeListElement* TryAllocateLargeLocked(intptr_t minimum_size);

  // 分配一个较小的元素
  uword TryAllocateSmallLocked(intptr_t size) {
    DEBUG_ASSERT(mutex_.IsOwnedByCurrentThread());
    /// 最后
    if (size > last_free_small_size_) {
      return 0;
    }
    // size 映射到 index
    int index = IndexForSize(size);
    // 如果给定下标有一个可用的（即 free_map_.Test(index) == true），则分出一个可用的 FreeListElement，
    // 并且映射成地址
    if (index != kNumLists && free_map_.Test(index)) {
      return reinterpret_cast<uword>(DequeueElement(index));
    }
    
    /// 到达这个位置则表示 index 下标处没有可用的 FreeListElement，
    // 则尝试在下一个有 FreeListElement 的 index 处提取一个 FreeListElement，
    // index < 127, 
    if ((index + 1) < kNumLists) {
      intptr_t next_index = free_map_.Next(index + 1);
      if (next_index != -1) {
        FreeListElement* element = DequeueElement(next_index);
        SplitElementAfterAndEnqueue(element, size, false);
        return reinterpret_cast<uword>(element);
      }
    }
    return 0;
  }
    
  
  uword TryAllocateBumpLocked(intptr_t size) {
    ASSERT(mutex_.IsOwnedByCurrentThread());
    uword result = top_;
    uword new_top = result + size;
    if (new_top <= end_) {
      top_ = new_top;
      unaccounted_size_ += size;
      return result;
    }
    return 0;
  }
    
  /// 提取 unaccounted_size_ 的值
  intptr_t TakeUnaccountedSizeLocked() {
    ASSERT(mutex_.IsOwnedByCurrentThread());
    intptr_t result = unaccounted_size_;
    unaccounted_size_ = 0;
    return result;
  }

  // Ensures OldPage::VisitObjects can successful walk over a partially
  // allocated bump region.
  void MakeIterable() {
    if (top_ < end_) {
      FreeListElement::AsElement(top_, end_ - top_);
    }
  }
  // Returns the bump region to the free list.
  void AbandonBumpAllocation() {
    if (top_ < end_) {
      Free(top_, end_ - top_);
      top_ = 0;
      end_ = 0;
    }
  }

  uword top() const { return top_; }
  uword end() const { return end_; }
  void set_top(uword value) { top_ = value; }
  void set_end(uword value) { end_ = value; }
  void AddUnaccountedSize(intptr_t size) { unaccounted_size_ += size; }

  void MergeFrom(FreeList* donor, bool is_protected);

 private:
  static const int kNumLists = 128;
  static const intptr_t kInitialFreeListSearchBudget = 1000;

  // 将 size 映射成 index
  static intptr_t IndexForSize(intptr_t size) {
    ASSERT(size >= kObjectAlignment);
    // size 是 kObjectAlignment 的整数倍
    ASSERT(Utils::IsAligned(size, kObjectAlignment));

    // size 右移四位映射成 index
    intptr_t index = size >> kObjectAlignmentLog2;
    // 如果是大对象，则分配在 128
    if (index >= kNumLists) {
      index = kNumLists;
    }
    return index;
  }

  intptr_t LengthLocked(int index) const;

  /// 添加一个 element 到指定 index
  void EnqueueElement(FreeListElement* element, intptr_t index);
  FreeListElement* DequeueElement(intptr_t index) {
    // element 链表头
    FreeListElement* result = free_lists_[index];
    FreeListElement* next = result->next();
    // 如果是空链表且不是最后一个 index
    if (next == NULL && index != kNumLists) {
      // 计算原有 size
      intptr_t size = index << kObjectAlignmentLog2;
      // 如果 size 等于 last_free_small_size_（最后一个 small size element）
      if (size == last_free_small_size_) {
        // Note: This is -1 * kObjectAlignment if no other small sizes remain.
        // 
        last_free_small_size_ =
            free_map_.ClearLastAndFindPrevious(index) * kObjectAlignment;
      } else {
        // 标记 index 有 element
        free_map_.Set(index, false);
      }
    }
    free_lists_[index] = next;
    return result;
  }

  void SplitElementAfterAndEnqueue(FreeListElement* element,
                                   intptr_t size,
                                   bool is_protected);

  void PrintSmall() const;
  void PrintLarge() const;

  // 块指针区
  // Bump pointer region.
  uword top_ = 0;
  uword end_ = 0;

  // 从块上区域分配的内存，但是还没有添加到 PageSpace::usage_。此域用于在并行清理时避免昂贵的原子加法
  intptr_t unaccounted_size_ = 0;

  // 锁定空闲分区表的数据结构
  mutable Mutex mutex_;

  // 如果给定的 index 有空闲的的 element，则 free_map[index] = true
  BitSet<kNumLists> free_map_;

  /// 静态分配列表长度 129
  FreeListElement* free_lists_[kNumLists + 1];

  intptr_t freelist_search_budget_ = kInitialFreeListSearchBudget;

  // 最大的的可用 small size 的 element，以字节表示。如果没有，则是负数
  intptr_t last_free_small_size_;

  DISALLOW_COPY_AND_ASSIGN(FreeList);
};

}  // namespace dart

#endif  // RUNTIME_VM_HEAP_FREELIST_H_
```



## freelist.cc

```c++
// ...

namespace dart {

// 从给定的地址和大小转换成 element
// 这个方法会破坏原有地址的数据，以保存空闲内存的信息（通常在释放其内存时调用）
FreeListElement* FreeListElement::AsElement(uword addr, intptr_t size) {
  // Precondition: the (page containing the) header of the element is
  // writable.
  ASSERT(size >= kObjectAlignment);
  ASSERT(Utils::IsAligned(size, kObjectAlignment));

  // 由于给定的地址包含对象头
  FreeListElement* result = reinterpret_cast<FreeListElement*>(addr);

  uint32_t tags = 0;
  tags = ObjectLayout::SizeTag::update(size, tags);
  tags = ObjectLayout::ClassIdTag::update(kFreeListElement, tags);
  ASSERT((addr & kNewObjectAlignmentOffset) == kOldObjectAlignmentOffset);
  tags = ObjectLayout::OldBit::update(true, tags);
  tags = ObjectLayout::OldAndNotMarkedBit::update(true, tags);
  tags = ObjectLayout::OldAndNotRememberedBit::update(true, tags);
  tags = ObjectLayout::NewBit::update(false, tags);
  result->tags_ = tags;
#if defined(HASH_IN_OBJECT_HEADER)
  // Clearing this is mostly for neatness. The identityHashCode
  // of free list entries is not used.
  result->hash_ = 0;
#endif
  if (size > ObjectLayout::SizeTag::kMaxSizeTag) {
    *result->SizeAddress() = size;
  }
  result->set_next(NULL);
  return result;
  // Postcondition: the (page containing the) header of the element is
  // writable.
}

void FreeListElement::Init() {
  ASSERT(sizeof(FreeListElement) == kObjectAlignment);
  ASSERT(OFFSET_OF(FreeListElement, tags_) == Object::tags_offset());
}


intptr_t FreeListElement::HeaderSizeFor(intptr_t size) {
  if (size == 0) return 0;
  return ((size > ObjectLayout::SizeTag::kMaxSizeTag) ? 3 : 2) * kWordSize;
}

FreeList::FreeList() : mutex_() {
  Reset();
}

FreeList::~FreeList() {
}
    
// 未锁定分配
uword FreeList::TryAllocate(intptr_t size, bool is_protected) {
  MutexLocker ml(&mutex_);
  return TryAllocateLocked(size, is_protected);
}

// 分配
uword FreeList::TryAllocateLocked(intptr_t size, bool is_protected) {
  DEBUG_ASSERT(mutex_.IsOwnedByCurrentThread());
  // Precondition: is_protected is false or else all free list elements are
  // in non-writable pages.

  // Postcondition: if allocation succeeds, the allocated block is writable.
  int index = IndexForSize(size);
  // 如果 index 小于 128（即可用使用小内存分配），并且对应的 index 有 element
  if ((index != kNumLists) && free_map_.Test(index)) {
    FreeListElement* element = DequeueElement(index);
    if (is_protected) {
      VirtualMemory::Protect(reinterpret_cast<void*>(element), size,
                             VirtualMemory::kReadWrite);
    }
    /// 返回空闲的内存，以供进一步使用。这时的内存还保留着 element 的信息
    return reinterpret_cast<uword>(element);
  }

  /// 到达这里，说明 index 处没有空闲的 element
  
  // 如果不是最大的小内存 index
  if ((index + 1) < kNumLists) {
      
    /// 寻找下一块可用使用的空闲 element（这一内存对应的 next_index 显然大于 index）
    intptr_t next_index = free_map_.Next(index + 1);
      
    // 找到了可以用的 index
    if (next_index != -1) {
      // Dequeue an element from the list, split and enqueue the remainder in
      // the appropriate list.
      FreeListElement* element = DequeueElement(next_index);
      if (is_protected) {
        // Make the allocated block and the header of the remainder element
        // writable.  The remainder will be non-writable if necessary after
        // the call to SplitElementAfterAndEnqueue.
        // If the remainder size is zero, only the element itself needs to
        // be made writable.
        intptr_t remainder_size = element->HeapSize() - size;
        intptr_t region_size =
            size + FreeListElement::HeaderSizeFor(remainder_size);
        VirtualMemory::Protect(reinterpret_cast<void*>(element), region_size,
                               VirtualMemory::kReadWrite);
      }
      // 将这一 element 之后的 element 入列（即链表操作）
      SplitElementAfterAndEnqueue(element, size, is_protected);
      return reinterpret_cast<uword>(element);
    }
  }

  FreeListElement* previous = NULL;
  // current 在 large element 处
  FreeListElement* current = free_lists_[kNumLists];
    
  // 到达这里，small size element 已经不能满足分配，所以寻找一个更大的块
  // We are willing to search the freelist further for a big block.
  // For each successful free-list search we:
  //   * increase the search budget by #allocated-words
  //   * decrease the search budget by #free-list-entries-traversed
  //     which guarantees us to not waste more than around 1 search step per
  //     word of allocation
  //
  // If we run out of search budget we fall back to allocating a new page and
  // reset the search budget.
  // freelist_search_budget_ = 1000
  intptr_t tries_left = freelist_search_budget_ + (size >> kWordSizeLog2);
  while (current != NULL) {
    if (current->HeapSize() >= size) {
      // Found an element large enough to hold the requested size. Dequeue,
      // split and enqueue the remainder.
      intptr_t remainder_size = current->HeapSize() - size;
      intptr_t region_size =
          size + FreeListElement::HeaderSizeFor(remainder_size);
      if (is_protected) {
        // Make the allocated block and the header of the remainder element
        // writable.  The remainder will be non-writable if necessary after
        // the call to SplitElementAfterAndEnqueue.
        VirtualMemory::Protect(reinterpret_cast<void*>(current), region_size,
                               VirtualMemory::kReadWrite);
      }

      if (previous == NULL) {
        free_lists_[kNumLists] = current->next();
      } else {
        // If the previous free list element's next field is protected, it
        // needs to be unprotected before storing to it and reprotected
        // after.
        bool target_is_protected = false;
        uword target_address = 0L;
        if (is_protected) {
          uword writable_start = reinterpret_cast<uword>(current);
          uword writable_end = writable_start + region_size - 1;
          target_address = previous->next_address();
          target_is_protected =
              !VirtualMemory::InSamePage(target_address, writable_start) &&
              !VirtualMemory::InSamePage(target_address, writable_end);
        }
        if (target_is_protected) {
          VirtualMemory::Protect(reinterpret_cast<void*>(target_address),
                                 kWordSize, VirtualMemory::kReadWrite);
        }
        previous->set_next(current->next());
        if (target_is_protected) {
          VirtualMemory::Protect(reinterpret_cast<void*>(target_address),
                                 kWordSize, VirtualMemory::kReadExecute);
        }
      }
      SplitElementAfterAndEnqueue(current, size, is_protected);
      freelist_search_budget_ =
          Utils::Minimum(tries_left, kInitialFreeListSearchBudget);
      return reinterpret_cast<uword>(current);
    } else if (tries_left-- < 0) {
      freelist_search_budget_ = kInitialFreeListSearchBudget;
      return 0;  // Trigger allocation of new page.
    }
    previous = current;
    current = current->next();
  }
  return 0;
}

// 释放给定地址
void FreeList::Free(uword addr, intptr_t size) {
  MutexLocker ml(&mutex_);
  FreeLocked(addr, size);
}

void FreeList::FreeLocked(uword addr, intptr_t size) {
  DEBUG_ASSERT(mutex_.IsOwnedByCurrentThread());
  intptr_t index = IndexForSize(size);
  // 调用 AsELement 会破坏地址原有的信息以保存内存的信息
  FreeListElement* element = FreeListElement::AsElement(addr, size);
  // 将 element 归还给空闲分区表
  EnqueueElement(element, index);
}

void FreeList::Reset() {
  MutexLocker ml(&mutex_);
  free_map_.Reset();
  last_free_small_size_ = -1;
  for (int i = 0; i < (kNumLists + 1); i++) {
    free_lists_[i] = NULL;
  }
}

// 将 element 加至指定的 index
void FreeList::EnqueueElement(FreeListElement* element, intptr_t index) {
  FreeListElement* next = free_lists_[index];
  if (next == NULL && index != kNumLists) {
    free_map_.Set(index, true);
    // 最大的可用 size
    last_free_small_size_ =
        Utils::Maximum(last_free_small_size_, index << kObjectAlignmentLog2);
  }
  element->set_next(next);
  free_lists_[index] = element;
}

// 求可用 element 数量
intptr_t FreeList::LengthLocked(int index) const {
  DEBUG_ASSERT(mutex_.IsOwnedByCurrentThread());
  ASSERT(index >= 0);
  ASSERT(index < kNumLists);
  intptr_t result = 0;
  FreeListElement* element = free_lists_[index];
  while (element != NULL) {
    ++result;
    element = element->next();
  }
  return result;
}

void FreeList::SplitElementAfterAndEnqueue(FreeListElement* element,
                                           intptr_t size,
                                           bool is_protected) {
  // Precondition required by AsElement and EnqueueElement: either
  // element->Size() == size, or else the (page containing the) header of
  // the remainder element starting at element + size is writable.
  intptr_t remainder_size = element->HeapSize() - size;
  if (remainder_size == 0) return;

  uword remainder_address = reinterpret_cast<uword>(element) + size;
  element = FreeListElement::AsElement(remainder_address, remainder_size);
  intptr_t remainder_index = IndexForSize(remainder_size);
  EnqueueElement(element, remainder_index);

  // Postcondition: when allocating in a protected page, the fraction of the
  // remainder element which does not share a page with the allocated element is
  // no longer writable. This means that if the remainder's header is not fully
  // contained in the last page of the allocation, we need to re-protect the
  // page it ends on.
  if (is_protected) {
    const uword remainder_header_size =
        FreeListElement::HeaderSizeFor(remainder_size);
    if (!VirtualMemory::InSamePage(
            remainder_address - 1,
            remainder_address + remainder_header_size - 1)) {
      VirtualMemory::Protect(
          reinterpret_cast<void*>(
              Utils::RoundUp(remainder_address, VirtualMemory::PageSize())),
          remainder_address + remainder_header_size -
              Utils::RoundUp(remainder_address, VirtualMemory::PageSize()),
          VirtualMemory::kReadExecute);
    }
  }
}

// 分配大内存
FreeListElement* FreeList::TryAllocateLarge(intptr_t minimum_size) {
  MutexLocker ml(&mutex_);
  return TryAllocateLargeLocked(minimum_size);
}

FreeListElement* FreeList::TryAllocateLargeLocked(intptr_t minimum_size) {
  DEBUG_ASSERT(mutex_.IsOwnedByCurrentThread());
  FreeListElement* previous = NULL;
  // 指向 large element 区域
  FreeListElement* current = free_lists_[kNumLists];
  // TODO(koda): Find largest.
  // We are willing to search the freelist further for a big block.
  intptr_t tries_left =
      freelist_search_budget_ + (minimum_size >> kWordSizeLog2);
  while (current != NULL) {
    FreeListElement* next = current->next();
    if (current->HeapSize() >= minimum_size) {
      if (previous == NULL) {
        free_lists_[kNumLists] = next;
      } else {
        previous->set_next(next);
      }
      freelist_search_budget_ =
          Utils::Minimum(tries_left, kInitialFreeListSearchBudget);
      return current;
    } else if (tries_left-- < 0) {
      freelist_search_budget_ = kInitialFreeListSearchBudget;
      return 0;  // 触发新页分配
    }
    previous = current;
    current = next;
  }
  return NULL;
}

    
// 合并分区表
void FreeList::MergeFrom(FreeList* donor, bool is_protected) {
  // The [other] free list is from a dying isolate. There are no other threads
  // accessing it, so there is no need to lock here.
  MutexLocker ml(&mutex_);
  for (intptr_t i = 0; i < (kNumLists + 1); ++i) {
    FreeListElement* donor_head = donor->free_lists_[i];
    if (donor_head != nullptr) {
      // If we didn't have a freelist element before we have to set the bit now,
      // since we will get 1+ elements from [other].
      FreeListElement* old_head = free_lists_[i];
      if (old_head == nullptr && i != kNumLists) {
        free_map_.Set(i, true);
      }

      // Chain other's list in.
      FreeListElement* last = donor_head;
      while (last->next() != nullptr) {
        last = last->next();
      }

      if (is_protected) {
        VirtualMemory::Protect(reinterpret_cast<void*>(last), sizeof(*last),
                               VirtualMemory::kReadWrite);
      }
      last->set_next(old_head);
      if (is_protected) {
        VirtualMemory::Protect(reinterpret_cast<void*>(last), sizeof(*last),
                               VirtualMemory::kReadExecute);
      }
      free_lists_[i] = donor_head;
    }
  }

  last_free_small_size_ =
      Utils::Maximum(last_free_small_size_, donor->last_free_small_size_);
}

}  // namespace dart
```

