# VirtualMemory



## virtual_memory.h

```c++
// ...

namespace dart {

// 虚拟内存
class VirtualMemory {
 public:
  // 访问保护
  enum Protection {
    // 无法访问
    kNoAccess,
    // 只读
    kReadOnly,
    // 读写
    kReadWrite,
    // 读可执行
    kReadExecute,
    // 读写可执行
    kReadWriteExecute
  };

  // The reserved memory is unmapped on destruction.
  ~VirtualMemory();

  // 内存起始地址
  uword start() const { return region_.start(); }
  // 内存终止地址
  uword end() const { return region_.end(); }
  // 起始地址指针
  void* address() const { return region_.pointer(); }
  // 内存大小
  intptr_t size() const { return region_.size(); }
  // alias 区域相对于内存起始地址的偏移量
  intptr_t AliasOffset() const { return alias_.start() - region_.start(); }

  static void Init();

  // Returns true if dual mapping is enabled.
  static bool DualMappingEnabled();

  // 是否包含某地址
  bool Contains(uword addr) const { return region_.Contains(addr); }
  // alias 是否包含某地址（如果存在 alias 区域）
  bool ContainsAlias(uword addr) const {
    return (AliasOffset() != 0) && alias_.Contains(addr);
  }

  // 改变某个虚拟内存的保护模式
  static void Protect(void* address, intptr_t size, Protection mode);
  // 改变此虚拟内存的保护模式
  void Protect(Protection mode) { return Protect(address(), size(), mode); }

  // 保留并提交一个具有 size 大小的虚拟内存段。如果不能分配所请求大小的段，则返回 NULL。
  static VirtualMemory* Allocate(intptr_t size,
                                 bool is_executable,
                                 const char* name) {
    // 使用 page 大小作为对其量
    return AllocateAligned(size, PageSize(), is_executable, name);
  }
  
  // 保留并提交一个具有 size 大小的虚拟内存段（以对其的方式）。如果不能分配所请求大小的段，则返回 NULL。
  static VirtualMemory* AllocateAligned(intptr_t size,
                                        intptr_t alignment,
                                        bool is_executable,
                                        const char* name);

  // 返回缓存的 page 大小
  static intptr_t PageSize() {
    ASSERT(page_size_ != 0);
    return page_size_;
  }

  // 是否处于同一 page 中
  static bool InSamePage(uword address0, uword address1);

  // 裁剪此虚拟内存至新的大小
  void Truncate(intptr_t new_size);

  // VM 是否持有此内存区域
  // 直接添加到 Dart 堆的快照的一部分为 False，该堆属于嵌入器，不得被 VM 释放或更改其保护状态
  bool vm_owns_region() const { return reserved_.pointer() != NULL; }

  static VirtualMemory* ForImagePage(void* pointer, uword size);

  // 释放此内存区域
  void release() {
    // 确保没有 page 内存泄漏
    const uword size_ = size();
    ASSERT(address() == reserved_.pointer() && size_ == reserved_.size());
    reserved_ = MemoryRegion(nullptr, 0);
  }

 private:
  static intptr_t CalculatePageSize();

  // 释放子内存。在支持子内存段的操作系统上， 这可以将虚拟内存还给系统。成功时返回 true 。
  static void FreeSubSegment(void* address, intptr_t size);

  // 这些构造函数仅在内部保留新的虚拟空间时使用，它们不会自己保留任何虚拟地址空间。
  VirtualMemory(const MemoryRegion& region,
                const MemoryRegion& alias,
                const MemoryRegion& reserved)
      : region_(region), alias_(alias), reserved_(reserved) {}

  // 只给定内存区域 region 的时候，会将 region_ 和 alias 都引用为 region
  VirtualMemory(const MemoryRegion& region, const MemoryRegion& reserved)
      : region_(region), alias_(region), reserved_(reserved) {}

  /// 内存区域
  MemoryRegion region_;

  // 采用不同保护方式的可选的辅助内存区域到虚拟内存空间的映射，例如允许代码执行。
  MemoryRegion alias_;

  // 基础保留尚未回给 Os 。其地址可能不同意区域由于对齐分配。其大小可能不同意区域由于 Truncate 。
  MemoryRegion reserved_;

  static uword page_size_;

  // 进制隐式转换构造器
  DISALLOW_IMPLICIT_CONSTRUCTORS(VirtualMemory);
};

}  // namespace dart

#endif  // RUNTIME_VM_VIRTUAL_MEMORY_H_

```



virtual_memory.cc

```c++
// ...
namespace dart {

bool VirtualMemory::InSamePage(uword address0, uword address1) {
  return (Utils::RoundDown(address0, PageSize()) ==
          Utils::RoundDown(address1, PageSize()));
}

void VirtualMemory::Truncate(intptr_t new_size) {
  ASSERT(Utils::IsAligned(new_size, PageSize()));
  ASSERT(new_size <= size());
  if (reserved_.size() ==  region_.size()) { 
    
    FreeSubSegment(
        reinterpret_cast<void*>(start() + new_size),
        size() - new_size
    );
    reserved_.set_size(new_size);
    if (AliasOffset() != 0) {
      FreeSubSegment(
          reinterpret_cast<void*>(alias_.start() + new_size),
          alias_.size() - new_size
      );
    }
  }
    
  // 调用 Subregion 来减小 region 和 alias
  region_.Subregion(region_, 0, new_size);
  alias_.Subregion(alias_, 0, new_size);
}

VirtualMemory* VirtualMemory::ForImagePage(void* pointer, uword size) {
  // Memory for precompilated instructions was allocated by the embedder, so
  // create a VirtualMemory without allocating.
  MemoryRegion region(pointer, size);
  MemoryRegion reserved(0, 0);  // NULL reservation indicates VM should not
                                // attempt to free this memory.
  VirtualMemory* memory = new VirtualMemory(region, region, reserved);
  ASSERT(!memory->vm_owns_region());
  return memory;
}

}  // namespace dart

```

