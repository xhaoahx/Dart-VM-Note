# MemoryRegion



## memory_region.h

```c++
// ...
    
namespace dart {

// 内存区域在 debug 模式下使用边界检查访问内存时十分有用
// 能够安全地传递值而不用关注区域的所有权    
class MemoryRegion : public ValueObject {
 public:
  // 无参数构造函数，使用 null 指针且大小为 0  
  MemoryRegion() : pointer_(NULL), size_(0) {}
  // 双参数构造函数，提供固定的起始指针和内存区域大小  
  MemoryRegion(void* pointer, uword size) : pointer_(pointer), size_(size) {}
  // 拷贝构造函数  
  MemoryRegion(const MemoryRegion& other) : ValueObject() { *this = other; }
  // 赋值构造函数
  MemoryRegion& operator=(const MemoryRegion& other) {
    pointer_ = other.pointer_;
    size_ = other.size_;
    return *this;
  }

  // 起始地址  
  void* pointer() const { return pointer_; }
  // 区域大小
  uword size() const { return size_; }
  // 设置大小
  void set_size(uword new_size) { size_ = new_size; }

  // 起始地址
  uword start() const { return reinterpret_cast<uword>(pointer_); }
    
  // 终止边界地址
  uword end() const { return start() + size_; }

  // 在给定的 offset 处获取一个 T 类型的变量（以指针的形式）
  template <typename T>
  T Load(uword offset) const {
    return *ComputeInternalPointer<T>(offset);
  }

  // 在给定的 offset 处储存一个 T 类型的变量
  template <typename T>
  void Store(uword offset, T value) const {
    *ComputeInternalPointer<T>(offset) = value;
  }

  // 不对齐的存储
  template <typename T>
  void StoreUnaligned(uword offset, T value) const {
    dart::StoreUnaligned(ComputeInternalPointer<T>(offset), value);
  }

  // 获得指向 offset 的给定类型的指针
  template <typename T>
  T* PointerTo(uword offset) const {
    return ComputeInternalPointer<T>(offset);
  }

  // 是否包含某个地址
  bool Contains(uword address) const {
    return (address >= start()) && (address < end());
  }

  void CopyFrom(uword offset, const MemoryRegion& from) const;

  // 在一个已存在的内存区域上划分一个子内存区域，其起始偏移为给定的 offset
  void Subregion(const MemoryRegion& from, uword offset, uword size) {
    ASSERT(from.size() >= size);
    ASSERT(offset <= (from.size() - size));
    pointer_ = reinterpret_cast<void*>(from.start() + offset);
    size_ = size;
  }

  // 拓展一个已存在的内存区域
  void Extend(const MemoryRegion& region, uword extra) {
    pointer_ = region.pointer();
    size_ = (region.size() + extra);
  }

 private:
  template <typename T>
  T* ComputeInternalPointer(uword offset) const {
    ASSERT(size() >= sizeof(T));
    ASSERT(offset <= size() - sizeof(T));
    return reinterpret_cast<T*>(start() + offset);
  }

  void* pointer_;
  uword size_;
};

}  // namespace dart

```



## memory_region.cc

```c++
// ...
namespace dart {

// 拷贝一块内存区域
void MemoryRegion::CopyFrom(uword offset, const MemoryRegion& from) const {
  ASSERT(from.pointer() != NULL && from.size() > 0);
  ASSERT(this->size() >= from.size());
  ASSERT(offset <= this->size() - from.size());
  memmove(reinterpret_cast<void*>(start() + offset), from.pointer(),
          from.size());
}

}  // namespace dart

```

