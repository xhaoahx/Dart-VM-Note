# RawObject

## raw_object.h

```c++
namespace dart {

// For now there are no compressed pointers.
typedef ObjectPtr RawCompressed;

// Forward declarations.
class Isolate;
class IsolateGroup;
#define DEFINE_FORWARD_DECLARATION(clazz) class clazz##Layout;
CLASS_LIST(DEFINE_FORWARD_DECLARATION)
#undef DEFINE_FORWARD_DECLARATION
class CodeStatistics;

/// 用于对象遍历：visitObject
// 将 first 和 last 映射成 type*
#define VISIT_FROM(type, first)                                                \
  type* from() { return reinterpret_cast<type*>(&first); }

#define VISIT_TO(type, last)                                                   \
  type* to() { return reinterpret_cast<type*>(&last); }

#define VISIT_TO_LENGTH(type, last)                                            \
  type* to(intptr_t length) { return reinterpret_cast<type*>(last); }

#define VISIT_NOTHING() int NothingToVisit();

#define ASSERT_UNCOMPRESSED(Type)                                              \
  ASSERT(SIZE_OF_DEREFERENCED_RETURNED_VALUE(Type##Layout, from) == kWordSize)

// For now there are no compressed pointers, so this assert is the same as
// the above.
#define ASSERT_COMPRESSED(Type)                                                \
  ASSERT(SIZE_OF_DEREFERENCED_RETURNED_VALUE(Type##Layout, from) == kWordSize)

#define ASSERT_NOTHING_TO_VISIT(Type)                                          \
  ASSERT(SIZE_OF_RETURNED_VALUE(Type##Layout, NothingToVisit) == sizeof(int))

    
// 虚拟机内置类型数组
// kInt8ArrayElement
// kUint8ArrayElement
// ...
enum TypedDataElementType {
#define V(name) k##name##Element,
  CLASS_LIST_TYPED_DATA(V)
#undef V
};

// 提供快照写
#define SNAPSHOT_WRITER_SUPPORT()                                              \
  void WriteTo(SnapshotWriter* writer, intptr_t object_id,                     \
               Snapshot::Kind kind, bool as_reference);                        \
  friend class SnapshotWriter;

// 对象访问
#define VISITOR_SUPPORT(object)                                                \
  static intptr_t Visit##object##Pointers(object##Ptr raw_obj,                 \
                                          ObjectPointerVisitor* visitor);

// 
#define HEAP_PROFILER_SUPPORT() friend class HeapProfiler;

// 原生对象实现
#define RAW_OBJECT_IMPLEMENTATION(object)                                      \
 private: /* NOLINT */                                                         \
  VISITOR_SUPPORT(object)                                                      \
  friend class object;                                                         \
  // 对象布局
  friend class ObjectLayout;                                                   \
  // 堆
  friend class Heap;                                                           \
  friend class Interpreter;                                                    \
  friend class InterpreterHelpers;                                             \
  friend class Simulator;                                                      \
  friend class SimulatorHelpers;                                               \
  friend class OffsetsTable;                                                   \
  // 禁用分配
  DISALLOW_ALLOCATION();                                                       \
  // 禁用隐式构造
  DISALLOW_IMPLICIT_CONSTRUCTORS(object##Layout)

// 原生堆对象实现
#define RAW_HEAP_OBJECT_IMPLEMENTATION(object)                                 \
 private:                                                                      \
  RAW_OBJECT_IMPLEMENTATION(object);                                           \
  SNAPSHOT_WRITER_SUPPORT()                                                    \
  HEAP_PROFILER_SUPPORT()                                                      \
  friend class object##SerializationCluster;                                   \
  friend class object##DeserializationCluster;                                 \
  friend class Serializer;                                                     \
  friend class Deserializer;                                                   \
  friend class Pass2Visitor;

// RawObject 是所有原生对象的基类。尽管所有的原生对象都持有 tag 域，但并不是所有的原生对象都在堆上分配内存，因此不能解除引用
// 关系， (e.g. RawSmi：原生小整数类)
class ObjectLayout {
 public:
  // tag 域是对象头的一部分，使用以下比特域来保存标志
  enum TagBits {
    kCardRememberedBit = 0,
    kOldAndNotMarkedBit = 1,      // Incremental barrier target.
    kNewBit = 2,                  // Generational barrier target.
    kOldBit = 3,                  // Incremental barrier source.
    kOldAndNotRememberedBit = 4,  // Generational barrier source.
    kCanonicalBit = 5,
    // 保留标志的位置
    kReservedTagPos = 6,
    // 保留标志的大小
    kReservedTagSize = 2,
    // 大小标志的位置
    kSizeTagPos = kReservedTagPos + kReservedTagSize,  // = 8
    // 大小标志的大小
    kSizeTagSize = 8,
    // classId 标志的位置
    kClassIdTagPos = kSizeTagPos + kSizeTagSize,  // = 16
    // classId 标志的大小
    kClassIdTagSize = 16,
    // 对象头哈希
#if defined(HASH_IN_OBJECT_HEADER)
    kHashTagPos = kClassIdTagPos + kClassIdTagSize,  // = 32
    kHashTagSize = 16,
#endif
  };

  static const intptr_t kGenerationalBarrierMask = 1 << kNewBit;
  static const intptr_t kIncrementalBarrierMask = 1 << kOldAndNotMarkedBit;
  static const intptr_t kBarrierOverlapShift = 2;
  COMPILE_ASSERT(kOldAndNotMarkedBit + kBarrierOverlapShift == kOldBit);
  COMPILE_ASSERT(kNewBit + kBarrierOverlapShift == kOldAndNotRememberedBit);

  // The bit in the Smi tag position must be something that can be set to 0
  // for a dead filler object of either generation.
  // See Object::MakeUnusedSpaceTraversable.
  COMPILE_ASSERT(kCardRememberedBit == 0);

  // Encodes the object size in the tag in units of object alignment.
  class SizeTag {
   public:
    typedef intptr_t Type;

    static constexpr intptr_t kMaxSizeTagInUnitsOfAlignment =
        ((1 << ObjectLayout::kSizeTagSize) - 1);
    static constexpr intptr_t kMaxSizeTag =
        kMaxSizeTagInUnitsOfAlignment * kObjectAlignment;

    static UNLESS_DEBUG(constexpr) uword encode(intptr_t size) {
      return SizeBits::encode(SizeToTagValue(size));
    }

    static constexpr uword decode(uword tag) {
      return TagValueToSize(SizeBits::decode(tag));
    }

    static UNLESS_DEBUG(constexpr) uword update(intptr_t size, uword tag) {
      return SizeBits::update(SizeToTagValue(size), tag);
    }

    static UNLESS_DEBUG(constexpr) bool SizeFits(intptr_t size) {
      DEBUG_ASSERT(Utils::IsAligned(size, kObjectAlignment));
      return (size <= kMaxSizeTag);
    }

   private:
    // The actual unscaled bit field used within the tag field.
    class SizeBits
        : public BitField<uint32_t, intptr_t, kSizeTagPos, kSizeTagSize> {};

    static UNLESS_DEBUG(constexpr) intptr_t SizeToTagValue(intptr_t size) {
      DEBUG_ASSERT(Utils::IsAligned(size, kObjectAlignment));
      return !SizeFits(size) ? 0 : (size >> kObjectAlignmentLog2);
    }
    static constexpr intptr_t TagValueToSize(intptr_t value) {
      return value << kObjectAlignmentLog2;
    }
  };

  /// classId 标记，在虚拟机类最多同时存在 65535 个Class
  class ClassIdTag : public BitField<uint32_t,
                                     ClassIdTagType,
                                     kClassIdTagPos,
                                     kClassIdTagSize> {};
  COMPILE_ASSERT(kBitsPerByte * sizeof(ClassIdTagType) == kClassIdTagSize);

  class CardRememberedBit
      : public BitField<uint32_t, bool, kCardRememberedBit, 1> {};

  class OldAndNotMarkedBit
      : public BitField<uint32_t, bool, kOldAndNotMarkedBit, 1> {};

  // 新生代标记
  class NewBit : public BitField<uint32_t, bool, kNewBit, 1> {};

  class CanonicalBit : public BitField<uint32_t, bool, kCanonicalBit, 1> {};

  /// 老生代标记
  class OldBit : public BitField<uint32_t, bool, kOldBit, 1> {};

  /// 未使用的老生代标记
  class OldAndNotRememberedBit
      : public BitField<uint32_t, bool, kOldAndNotRememberedBit, 1> {};

  class ReservedBits
      : public BitField<uint32_t, intptr_t, kReservedTagPos, kReservedTagSize> {
  };

  class Tags {
   public:
    Tags() : tags_(0) {}

    NO_SANITIZE_THREAD
    operator uint32_t() const {
      return *reinterpret_cast<const uint32_t*>(&tags_);
    }

    NO_SANITIZE_THREAD
    uint32_t operator=(uint32_t tags) {
      return *reinterpret_cast<uint32_t*>(&tags_) = tags;
    }

    NO_SANITIZE_THREAD
    bool StrongCAS(uint32_t old_tags, uint32_t new_tags) {
      return tags_.compare_exchange_strong(old_tags, new_tags,
                                           std::memory_order_relaxed);
    }

    NO_SANITIZE_THREAD
    bool WeakCAS(uint32_t old_tags, uint32_t new_tags) {
      return tags_.compare_exchange_weak(old_tags, new_tags,
                                         std::memory_order_relaxed);
    }

    /// 读取 tag 中的某一 Bool 位，TagBitField 类型参数为以上定义的继承自 BitField 的类
    template <class TagBitField>
    NO_SANITIZE_THREAD typename TagBitField::Type Read() const {
      return TagBitField::decode(*reinterpret_cast<const uint32_t*>(&tags_));
    }

    /// 更新 tag 中的某一 Bool 位，TagBitField 类型参数为以上定义的继承自 BitField 的类
    template <class TagBitField>
    NO_SANITIZE_THREAD void UpdateBool(bool value) {
      if (value) {
        tags_.fetch_or(TagBitField::encode(true), std::memory_order_relaxed);
      } else {
        tags_.fetch_and(~TagBitField::encode(true), std::memory_order_relaxed);
      }
    }

    template <class TagBitField>
    NO_SANITIZE_THREAD void UpdateUnsynchronized(
        typename TagBitField::Type value) {
      *reinterpret_cast<uint32_t*>(&tags_) =
          TagBitField::update(value, *reinterpret_cast<uint32_t*>(&tags_));
    }

    template <class TagBitField>
    NO_SANITIZE_THREAD bool TryAcquire() {
      uint32_t mask = TagBitField::encode(true);
      uint32_t old_tags = tags_.fetch_or(mask, std::memory_order_relaxed);
      return !TagBitField::decode(old_tags);
    }

    template <class TagBitField>
    NO_SANITIZE_THREAD bool TryClear() {
      uint32_t mask = ~TagBitField::encode(true);
      uint32_t old_tags = tags_.fetch_and(mask, std::memory_order_relaxed);
      return TagBitField::decode(old_tags);
    }

   private:
    // 原子操作
    std::atomic<uint32_t> tags_;
    COMPILE_ASSERT(sizeof(std::atomic<uint32_t>) == sizeof(uint32_t));
  };

  // 是否是新生代对象
  bool IsNewObject() const {
    uword addr = reinterpret_cast<uword>(this);
    return (addr & kObjectAlignmentMask) == kNewObjectAlignmentOffset;
  }
  // 是否是老生代对象
  bool IsOldObject() const {
    uword addr = reinterpret_cast<uword>(this);
    return (addr & kObjectAlignmentMask) == kOldObjectAlignmentOffset;
  }

  // 支持 Gc 标记位。标记的对象为灰色 （尚未访问） 或黑色 （已访问）。
  bool IsMarked() const {
    ASSERT(IsOldObject());
    return !tags_.Read<OldAndNotMarkedBit>();
  }
  void SetMarkBit() {
    ASSERT(IsOldObject());
    ASSERT(!IsMarked());
    tags_.UpdateBool<OldAndNotMarkedBit>(false);
  }
  void SetMarkBitUnsynchronized() {
    ASSERT(IsOldObject());
    ASSERT(!IsMarked());
    tags_.UpdateUnsynchronized<OldAndNotMarkedBit>(false);
  }
  void ClearMarkBit() {
    ASSERT(IsOldObject());
    ASSERT(IsMarked());
    tags_.UpdateBool<OldAndNotMarkedBit>(true);
  }
  // Returns false if the bit was already set.
  DART_WARN_UNUSED_RESULT
  bool TryAcquireMarkBit() {
    ASSERT(IsOldObject());
    return tags_.TryClear<OldAndNotMarkedBit>();
  }

  // 规范对象具有两个规范对象在逻辑上相等的属性，如果它们是同一个对象（指针相等）。
  // Canonical objects have the property that two canonical objects are
  // logically equal if they are the same object (pointer equal).
  bool IsCanonical() const { return tags_.Read<CanonicalBit>(); }
  void SetCanonical() { tags_.UpdateBool<CanonicalBit>(true); }
  void ClearCanonical() { tags_.UpdateBool<CanonicalBit>(false); }

  bool InVMIsolateHeap() const;

  // 支持 GC 记住位。
  bool IsRemembered() const {
    ASSERT(IsOldObject());
    return !tags_.Read<OldAndNotRememberedBit>();
  }
  void SetRememberedBit() {
    ASSERT(!IsRemembered());
    ASSERT(!IsCardRemembered());
    tags_.UpdateBool<OldAndNotRememberedBit>(false);
  }
  void ClearRememberedBit() {
    ASSERT(IsOldObject());
    tags_.UpdateBool<OldAndNotRememberedBit>(true);
  }

  // 添加到记忆集
  DART_FORCE_INLINE
  void AddToRememberedSet(Thread* thread) {
    ASSERT(!this->IsRemembered());
    this->SetRememberedBit();
    thread->StoreBufferAddObject(ObjectPtr(this));
  }

  bool IsCardRemembered() const { return tags_.Read<CardRememberedBit>(); }
  void SetCardRememberedBitUnsynchronized() {
    ASSERT(!IsRemembered());
    ASSERT(!IsCardRemembered());
    tags_.UpdateUnsynchronized<CardRememberedBit>(true);
  }

  // 获取 classId
  intptr_t GetClassId() const { return tags_.Read<ClassIdTag>(); }

  //
  intptr_t HeapSize() const {
    uint32_t tags = tags_;
    intptr_t result = SizeTag::decode(tags);
    if (result != 0) {
      return result;
    }
    // 在堆上确定 Class 的大小
    result = HeapSizeFromClass(tags);
    ASSERT(result > SizeTag::kMaxSizeTag);
    return result;
  }

  // This variant must not deference this->tags_.
  intptr_t HeapSize(uint32_t tags) const {
    intptr_t result = SizeTag::decode(tags);
    if (result != 0) {
      return result;
    }
    result = HeapSizeFromClass(tags);
    ASSERT(result > SizeTag::kMaxSizeTag);
    return result;
  }

  // 是否包含某个地址
  bool Contains(uword addr) const {
    intptr_t this_size = HeapSize();
    uword this_addr = ObjectLayout::ToAddr(this);
    return (addr >= this_addr) && (addr < (this_addr + this_size));
  }

  void Validate(IsolateGroup* isolate_group) const;
  bool FindObject(FindObjectVisitor* visitor);

  // 此函数可以访问标头中的类 ID，但它无法访问实际的类对象，因为滑动压缩器在移动类对象时使用此函数。
  intptr_t VisitPointers(ObjectPointerVisitor* visitor) {
    // Fall back to virtual variant for predefined classes
    intptr_t class_id = GetClassId();
    if (class_id < kNumPredefinedCids) {
      return VisitPointersPredefined(visitor, class_id);
    }

    // Calculate the first and last raw object pointer fields.
    intptr_t instance_size = HeapSize();
    uword obj_addr = ToAddr(this);
    uword from = obj_addr + sizeof(ObjectLayout);
    uword to = obj_addr + instance_size - kWordSize;
    const auto first = reinterpret_cast<ObjectPtr*>(from);
    const auto last = reinterpret_cast<ObjectPtr*>(to);

#if defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)
    const auto unboxed_fields_bitmap =
        visitor->shared_class_table()->GetUnboxedFieldsMapAt(class_id);

    if (!unboxed_fields_bitmap.IsEmpty()) {
      intptr_t bit = sizeof(ObjectLayout) / kWordSize;
      for (ObjectPtr* current = first; current <= last; current++) {
        if (!unboxed_fields_bitmap.Get(bit++)) {
          visitor->VisitPointer(current);
        }
      }
    } else {
      visitor->VisitPointers(first, last);
    }
#else
    // Call visitor function virtually
    visitor->VisitPointers(first, last);
#endif  // defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)

    return instance_size;
  }

  template <class V>
  intptr_t VisitPointersNonvirtual(V* visitor) {
    // Fall back to virtual variant for predefined classes
    intptr_t class_id = GetClassId();
    if (class_id < kNumPredefinedCids) {
      return VisitPointersPredefined(visitor, class_id);
    }

    // Calculate the first and last raw object pointer fields.
    intptr_t instance_size = HeapSize();
    uword obj_addr = ToAddr(this);
    uword from = obj_addr + sizeof(ObjectLayout);
    uword to = obj_addr + instance_size - kWordSize;
    const auto first = reinterpret_cast<ObjectPtr*>(from);
    const auto last = reinterpret_cast<ObjectPtr*>(to);

#if defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)
    const auto unboxed_fields_bitmap =
        visitor->shared_class_table()->GetUnboxedFieldsMapAt(class_id);

    if (!unboxed_fields_bitmap.IsEmpty()) {
      intptr_t bit = sizeof(ObjectLayout) / kWordSize;
      for (ObjectPtr* current = first; current <= last; current++) {
        if (!unboxed_fields_bitmap.Get(bit++)) {
          visitor->V::VisitPointers(current, current);
        }
      }
    } else {
      visitor->V::VisitPointers(first, last);
    }
#else
    // Call visitor function non-virtually
    visitor->V::VisitPointers(first, last);
#endif  // defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)

    return instance_size;
  }

  // This variant ensures that we do not visit the extra slot created from
  // rounding up instance sizes up to the allocation unit.
  void VisitPointersPrecise(Isolate* isolate, ObjectPointerVisitor* visitor);

  /// 从给定的地址获取得到 ObjectPtr
  static ObjectPtr FromAddr(uword addr) {
    // We expect the untagged address here.
    // 在这里使用未标记的地址
    ASSERT((addr & kSmiTagMask) != kHeapObjectTag);
    return static_cast<ObjectPtr>(addr + kHeapObjectTag);
  }

  // 转换成地址
  static uword ToAddr(const ObjectLayout* raw_obj) {
    return reinterpret_cast<uword>(raw_obj);
  }
  static uword ToAddr(const ObjectPtr raw_obj) {
    return static_cast<uword>(raw_obj) - kHeapObjectTag;
  }

  // 是否是规范对象
  static bool IsCanonical(intptr_t value) {
    return CanonicalBit::decode(value);
  }

 private:
  /// ObjectLayout 的第一个域就是 tags
  Tags tags_;  // Various object tags (bits).
#if defined(HASH_IN_OBJECT_HEADER)
  // On 64 bit there is a hash field in the header for the identity hash.
  uint32_t hash_;
#elif defined(IS_SIMARM_X64)
  // On simarm_x64 the hash isn't used, but we need the padding anyway so that
  // the object layout fits assumptions made about X64.
  uint32_t padding_;
#endif

  intptr_t VisitPointersPredefined(ObjectPointerVisitor* visitor,
                                   intptr_t class_id);

  intptr_t HeapSizeFromClass(uint32_t tags) const;

  // 设置成新的 classId
  void SetClassId(intptr_t new_cid) {
    tags_.UpdateUnsynchronized<ClassIdTag>(new_cid);
  }

  // 对堆对象的所有写入最终都应通过下面的方法之一或 Object 中的对应方法，以确保正确应用写入障碍。

  template <typename type, std::memory_order order = std::memory_order_relaxed>
  type LoadPointer(type const* addr) {
    return reinterpret_cast<std::atomic<type>*>(const_cast<type*>(addr))
        ->load(order);
  }

  template <typename type, std::memory_order order = std::memory_order_relaxed>
  void StorePointer(type const* addr, type value) {
    reinterpret_cast<std::atomic<type>*>(const_cast<type*>(addr))
        ->store(value, order);
    if (value->IsHeapObject()) {
      CheckHeapPointerStore(value, Thread::Current());
    }
  }

  template <typename type>
  void StorePointer(type const* addr, type value, Thread* thread) {
    *const_cast<type*>(addr) = value;
    if (value->IsHeapObject()) {
      CheckHeapPointerStore(value, thread);
    }
  }

  template <typename type>
  void StorePointerUnaligned(type const* addr, type value, Thread* thread) {
    StoreUnaligned(const_cast<type*>(addr), value);
    if (value->IsHeapObject()) {
      CheckHeapPointerStore(value, thread);
    }
  }

  DART_FORCE_INLINE
  void CheckHeapPointerStore(ObjectPtr value, Thread* thread) {
    uint32_t source_tags = this->tags_;
    uint32_t target_tags = value->ptr()->tags_;
    if (((source_tags >> kBarrierOverlapShift) & target_tags &
         thread->write_barrier_mask()) != 0) {
      if (value->IsNewObject()) {
        // Generational barrier: record when a store creates an
        // old-and-not-remembered -> new reference.
        AddToRememberedSet(thread);
      } else {
        // Incremental barrier: record when a store creates an
        // old -> old-and-not-marked reference.
        ASSERT(value->IsOldObject());
#if !defined(TARGET_ARCH_IA32)
        if (ClassIdTag::decode(target_tags) == kInstructionsCid) {
          // Instruction pages may be non-writable. Defer marking.
          thread->DeferredMarkingStackAddObject(value);
          return;
        }
#endif
        if (value->ptr()->TryAcquireMarkBit()) {
          thread->MarkingStackAddObject(value);
        }
      }
    }
  }

  template <typename type, std::memory_order order = std::memory_order_relaxed>
  void StoreArrayPointer(type const* addr, type value) {
    reinterpret_cast<std::atomic<type>*>(const_cast<type*>(addr))
        ->store(value, order);
    if (value->IsHeapObject()) {
      CheckArrayPointerStore(addr, value, Thread::Current());
    }
  }

  template <typename type>
  void StoreArrayPointer(type const* addr, type value, Thread* thread) {
    *const_cast<type*>(addr) = value;
    if (value->IsHeapObject()) {
      CheckArrayPointerStore(addr, value, thread);
    }
  }

  template <typename type>
  DART_FORCE_INLINE void CheckArrayPointerStore(
      type const* addr,
      ObjectPtr value,
      Thread* thread
  ) {
    uint32_t source_tags = this->tags_;
    uint32_t target_tags = value->ptr()->tags_;
    if (((source_tags >> kBarrierOverlapShift) & target_tags &
         thread->write_barrier_mask()) != 0) {
      if (value->IsNewObject()) {
        // Generational barrier: record when a store creates an
        // old-and-not-remembered -> new reference.
        ASSERT(!this->IsRemembered());
        if (this->IsCardRemembered()) {
          RememberCard(reinterpret_cast<ObjectPtr const*>(addr));
        } else {
          this->SetRememberedBit();
          thread->StoreBufferAddObject(static_cast<ObjectPtr>(this));
        }
      } else {
        // Incremental barrier: record when a store creates an
        // old -> old-and-not-marked reference.
        ASSERT(value->IsOldObject());
#if !defined(TARGET_ARCH_IA32)
        if (ClassIdTag::decode(target_tags) == kInstructionsCid) {
          // Instruction pages may be non-writable. Defer marking.
          thread->DeferredMarkingStackAddObject(value);
          return;
        }
#endif
        if (value->ptr()->TryAcquireMarkBit()) {
          thread->MarkingStackAddObject(value);
        }
      }
    }
  }

  // Use for storing into an explicitly Smi-typed field of an object
  // (i.e., both the previous and new value are Smis).
  void StoreSmi(SmiPtr const* addr, SmiPtr value) {
    // Can't use Contains, as array length is initialized through this method.
    ASSERT(reinterpret_cast<uword>(addr) >= ObjectLayout::ToAddr(this));
    *const_cast<SmiPtr*>(addr) = value;
  }
  NO_SANITIZE_THREAD
  void StoreSmiIgnoreRace(SmiPtr const* addr, SmiPtr value) {
    // Can't use Contains, as array length is initialized through this method.
    ASSERT(reinterpret_cast<uword>(addr) >= ObjectLayout::ToAddr(this));
    *const_cast<SmiPtr*>(addr) = value;
  }

 protected:
  friend class StoreBufferUpdateVisitor;  // RememberCard
  void RememberCard(ObjectPtr const* slot);

  friend class Array;
  friend class ByteBuffer;
  friend class CidRewriteVisitor;
  friend class Closure;
  friend class Code;
  friend class Pointer;
  friend class Double;
  friend class DynamicLibrary;
  friend class ForwardPointersVisitor;  // StorePointer
  friend class FreeListElement;
  friend class Function;
  friend class GCMarker;
  friend class ExternalTypedData;
  friend class ForwardList;
  friend class GrowableObjectArray;  // StorePointer
  friend class Heap;
  friend class ClassStatsVisitor;
  template <bool>
  friend class MarkingVisitorBase;
  friend class Mint;
  friend class Object;
  friend class OneByteString;  // StoreSmi
  friend class InstanceLayout;
  friend class Scavenger;
  template <bool>
  friend class ScavengerVisitorBase;
  friend class ImageReader;  // tags_ check
  friend class ImageWriter;
  friend class AssemblyImageWriter;
  friend class BlobImageWriter;
  friend class SnapshotReader;
  friend class Deserializer;
  friend class SnapshotWriter;
  friend class String;
  friend class WeakProperty;            // StorePointer
  friend class Instance;                // StorePointer
  friend class StackFrame;              // GetCodeObject assertion.
  friend class CodeLookupTableBuilder;  // profiler
  friend class Interpreter;
  friend class InterpreterHelpers;
  friend class Simulator;
  friend class SimulatorHelpers;
  friend class ObjectLocator;
  friend class WriteBarrierUpdateVisitor;  // CheckHeapPointerStore
  friend class OffsetsTable;
  friend class Object;

  DISALLOW_ALLOCATION();
  DISALLOW_IMPLICIT_CONSTRUCTORS(ObjectLayout);
};

inline intptr_t ObjectPtr::GetClassId() const {
  return ptr()->GetClassId();
}
```



## raw_object.cc

```c++
// ...

namespace dart {

bool ObjectLayout::InVMIsolateHeap() const {
  // All "vm-isolate" objects are pre-marked and in old space
  // (see [Object::FinalizeVMIsolate]).
  if (!IsOldObject() || !IsMarked()) return false;

  auto heap = Dart::vm_isolate()->heap();
  ASSERT(heap->UsedInWords(Heap::kNew) == 0);
  return heap->old_space()->ContainsUnsafe(ToAddr(this));
}

void ObjectPtr::Validate(IsolateGroup* isolate_group) const {
  // All Smi values are valid.
  if (!IsHeapObject()) {
    return;
  }
  // Slightly more readable than a segfault.
  if (tagged_pointer_ == kHeapObjectTag) {
    FATAL("RAW_NULL encountered");
  }
  ptr()->Validate(isolate_group);
}

void ObjectLayout::Validate(IsolateGroup* isolate_group) const {
  if (static_cast<uword>(Object::void_class_) == kHeapObjectTag) {
    // Validation relies on properly initialized class classes. Skip if the
    // VM is still being initialized.
    return;
  }
  // Validate that the tags_ field is sensible.
  uint32_t tags = tags_;
  if (IsNewObject()) {
    if (!NewBit::decode(tags)) {
      FATAL1("New object missing kNewBit: %x\n", tags);
    }
    if (OldBit::decode(tags)) {
      FATAL1("New object has kOldBit: %x\n", tags);
    }
    if (OldAndNotMarkedBit::decode(tags)) {
      FATAL1("New object has kOldAndNotMarkedBit: %x\n", tags);
    }
    if (OldAndNotRememberedBit::decode(tags)) {
      FATAL1("New object has kOldAndNotRememberedBit: %x\n", tags);
    }
  } else {
    if (NewBit::decode(tags)) {
      FATAL1("Old object has kNewBit: %x\n", tags);
    }
    if (!OldBit::decode(tags)) {
      FATAL1("Old object missing kOldBit: %x\n", tags);
    }
  }
  const intptr_t class_id = ClassIdTag::decode(tags);
  if (!isolate_group->shared_class_table()->IsValidIndex(class_id)) {
    FATAL1("Invalid class id encountered %" Pd "\n", class_id);
  }
  if (class_id == kNullCid &&
      isolate_group->shared_class_table()->HasValidClassAt(class_id)) {
    // Null class not yet initialized; skip.
    return;
  }
  intptr_t size_from_tags = SizeTag::decode(tags);
  intptr_t size_from_class = HeapSizeFromClass(tags);
  if ((size_from_tags != 0) && (size_from_tags != size_from_class)) {
    FATAL3(
        "Inconsistent size encountered "
        "cid: %" Pd ", size_from_tags: %" Pd ", size_from_class: %" Pd "\n",
        class_id, size_from_tags, size_from_class);
  }
}

// Can't look at the class object because it can be called during
// compaction when the class objects are moving. Can use the class
// id in the header and the sizes in the Class Table.
// Cannot deference ptr()->tags_. May dereference other parts of the object.
intptr_t ObjectLayout::HeapSizeFromClass(uint32_t tags) const {
  intptr_t class_id = ClassIdTag::decode(tags);
  intptr_t instance_size = 0;
  switch (class_id) {
    case kCodeCid: {
      const CodePtr raw_code = static_cast<const CodePtr>(this);
      intptr_t pointer_offsets_length =
          Code::PtrOffBits::decode(raw_code->ptr()->state_bits_);
      instance_size = Code::InstanceSize(pointer_offsets_length);
      break;
    }
    case kInstructionsCid: {
      const InstructionsPtr raw_instructions =
          static_cast<const InstructionsPtr>(this);
      intptr_t instructions_size = Instructions::Size(raw_instructions);
      instance_size = Instructions::InstanceSize(instructions_size);
      break;
    }
    case kInstructionsSectionCid: {
      const InstructionsSectionPtr raw_section =
          static_cast<const InstructionsSectionPtr>(this);
      intptr_t section_size = InstructionsSection::Size(raw_section);
      instance_size = InstructionsSection::InstanceSize(section_size);
      break;
    }
    case kContextCid: {
      const ContextPtr raw_context = static_cast<const ContextPtr>(this);
      intptr_t num_variables = raw_context->ptr()->num_variables_;
      instance_size = Context::InstanceSize(num_variables);
      break;
    }
    case kContextScopeCid: {
      const ContextScopePtr raw_context_scope =
          static_cast<const ContextScopePtr>(this);
      intptr_t num_variables = raw_context_scope->ptr()->num_variables_;
      instance_size = ContextScope::InstanceSize(num_variables);
      break;
    }
    case kOneByteStringCid: {
      const OneByteStringPtr raw_string =
          static_cast<const OneByteStringPtr>(this);
      intptr_t string_length = Smi::Value(raw_string->ptr()->length_);
      instance_size = OneByteString::InstanceSize(string_length);
      break;
    }
    case kTwoByteStringCid: {
      const TwoByteStringPtr raw_string =
          static_cast<const TwoByteStringPtr>(this);
      intptr_t string_length = Smi::Value(raw_string->ptr()->length_);
      instance_size = TwoByteString::InstanceSize(string_length);
      break;
    }
    case kArrayCid:
    case kImmutableArrayCid: {
      const ArrayPtr raw_array = static_cast<const ArrayPtr>(this);
      intptr_t array_length = Smi::Value(raw_array->ptr()->length_);
      instance_size = Array::InstanceSize(array_length);
      break;
    }
    case kObjectPoolCid: {
      const ObjectPoolPtr raw_object_pool =
          static_cast<const ObjectPoolPtr>(this);
      intptr_t len = raw_object_pool->ptr()->length_;
      instance_size = ObjectPool::InstanceSize(len);
      break;
    }
#define SIZE_FROM_CLASS(clazz) case kTypedData##clazz##Cid:
      CLASS_LIST_TYPED_DATA(SIZE_FROM_CLASS) {
        const TypedDataPtr raw_obj = static_cast<const TypedDataPtr>(this);
        intptr_t array_len = Smi::Value(raw_obj->ptr()->length_);
        intptr_t lengthInBytes =
            array_len * TypedData::ElementSizeInBytes(class_id);
        instance_size = TypedData::InstanceSize(lengthInBytes);
        break;
      }
#undef SIZE_FROM_CLASS
    case kFfiPointerCid:
      instance_size = Pointer::InstanceSize();
      break;
    case kTypeArgumentsCid: {
      const TypeArgumentsPtr raw_array =
          static_cast<const TypeArgumentsPtr>(this);
      intptr_t array_length = Smi::Value(raw_array->ptr()->length_);
      instance_size = TypeArguments::InstanceSize(array_length);
      break;
    }
    case kPcDescriptorsCid: {
      const PcDescriptorsPtr raw_descriptors =
          static_cast<const PcDescriptorsPtr>(this);
      intptr_t length = raw_descriptors->ptr()->length_;
      instance_size = PcDescriptors::InstanceSize(length);
      break;
    }
    case kCodeSourceMapCid: {
      const CodeSourceMapPtr raw_code_source_map =
          static_cast<const CodeSourceMapPtr>(this);
      intptr_t length = raw_code_source_map->ptr()->length_;
      instance_size = CodeSourceMap::InstanceSize(length);
      break;
    }
    case kCompressedStackMapsCid: {
      const CompressedStackMapsPtr maps =
          static_cast<const CompressedStackMapsPtr>(this);
      intptr_t length = CompressedStackMaps::PayloadSizeOf(maps);
      instance_size = CompressedStackMaps::InstanceSize(length);
      break;
    }
    case kLocalVarDescriptorsCid: {
      const LocalVarDescriptorsPtr raw_descriptors =
          static_cast<const LocalVarDescriptorsPtr>(this);
      intptr_t num_descriptors = raw_descriptors->ptr()->num_entries_;
      instance_size = LocalVarDescriptors::InstanceSize(num_descriptors);
      break;
    }
    case kExceptionHandlersCid: {
      const ExceptionHandlersPtr raw_handlers =
          static_cast<const ExceptionHandlersPtr>(this);
      intptr_t num_handlers = raw_handlers->ptr()->num_entries_;
      instance_size = ExceptionHandlers::InstanceSize(num_handlers);
      break;
    }
    case kFreeListElement: {
      uword addr = ObjectLayout::ToAddr(this);
      FreeListElement* element = reinterpret_cast<FreeListElement*>(addr);
      instance_size = element->HeapSize();
      break;
    }
    case kForwardingCorpse: {
      uword addr = ObjectLayout::ToAddr(this);
      ForwardingCorpse* element = reinterpret_cast<ForwardingCorpse*>(addr);
      instance_size = element->HeapSize();
      break;
    }
    case kImageHeaderCid: {
      instance_size = ImageHeader::InstanceSize();
      break;
    }
    case kWeakSerializationReferenceCid: {
      instance_size = WeakSerializationReference::InstanceSize();
      break;
    }
    default: {
      // Get the (constant) instance size out of the class object.
      // TODO(koda): Add Size(ClassTable*) interface to allow caching in loops.
      auto isolate_group = IsolateGroup::Current();
#if defined(DEBUG)
#if !defined(DART_PRECOMPILED_RUNTIME)
      auto reload_context = isolate_group->reload_context();
      const bool use_saved_class_table =
          reload_context != nullptr ? reload_context->UseSavedSizeTableForGC()
                                    : false;
#else
      const bool use_saved_class_table = false;
#endif

      auto class_table = isolate_group->shared_class_table();
      ASSERT(use_saved_class_table || class_table->SizeAt(class_id) > 0);
      if (!class_table->IsValidIndex(class_id) ||
          (!class_table->HasValidClassAt(class_id) && !use_saved_class_table)) {
        FATAL3("Invalid cid: %" Pd ", obj: %p, tags: %x. Corrupt heap?",
               class_id, this, static_cast<uint32_t>(tags));
      }
#endif  // DEBUG
      instance_size = isolate_group->GetClassSizeForHeapWalkAt(class_id);
    }
  }
  ASSERT(instance_size != 0);
#if defined(DEBUG)
  intptr_t tags_size = SizeTag::decode(tags);
  if ((class_id == kArrayCid) && (instance_size > tags_size && tags_size > 0)) {
    // TODO(22501): Array::MakeFixedLength could be in the process of shrinking
    // the array (see comment therein), having already updated the tags but not
    // yet set the new length. Wait a millisecond and try again.
    int retries_remaining = 1000;  // ... but not forever.
    do {
      OS::Sleep(1);
      const ArrayPtr raw_array = static_cast<const ArrayPtr>(this);
      intptr_t array_length = Smi::Value(raw_array->ptr()->length_);
      instance_size = Array::InstanceSize(array_length);
    } while ((instance_size > tags_size) && (--retries_remaining > 0));
  }
  if ((instance_size != tags_size) && (tags_size != 0)) {
    FATAL3("Size mismatch: %" Pd " from class vs %" Pd " from tags %x\n",
           instance_size, tags_size, tags);
  }
#endif  // DEBUG
  return instance_size;
}

intptr_t ObjectLayout::VisitPointersPredefined(ObjectPointerVisitor* visitor,
                                               intptr_t class_id) {
  ASSERT(class_id < kNumPredefinedCids);

  intptr_t size = 0;

  switch (class_id) {
#define RAW_VISITPOINTERS(clazz)                                               \
  case k##clazz##Cid: {                                                        \
    clazz##Ptr raw_obj = static_cast<clazz##Ptr>(this);                        \
    size = clazz##Layout::Visit##clazz##Pointers(raw_obj, visitor);            \
    break;                                                                     \
  }
    CLASS_LIST_NO_OBJECT(RAW_VISITPOINTERS)
#undef RAW_VISITPOINTERS
#define RAW_VISITPOINTERS(clazz) case kTypedData##clazz##Cid:
    CLASS_LIST_TYPED_DATA(RAW_VISITPOINTERS) {
      TypedDataPtr raw_obj = static_cast<TypedDataPtr>(this);
      size = TypedDataLayout::VisitTypedDataPointers(raw_obj, visitor);
      break;
    }
#undef RAW_VISITPOINTERS
#define RAW_VISITPOINTERS(clazz) case kExternalTypedData##clazz##Cid:
    CLASS_LIST_TYPED_DATA(RAW_VISITPOINTERS) {
      auto raw_obj = static_cast<ExternalTypedDataPtr>(this);
      size = ExternalTypedDataLayout::VisitExternalTypedDataPointers(raw_obj,
                                                                     visitor);
      break;
    }
#undef RAW_VISITPOINTERS
    case kByteDataViewCid:
#define RAW_VISITPOINTERS(clazz) case kTypedData##clazz##ViewCid:
      CLASS_LIST_TYPED_DATA(RAW_VISITPOINTERS) {
        auto raw_obj = static_cast<TypedDataViewPtr>(this);
        size =
            TypedDataViewLayout::VisitTypedDataViewPointers(raw_obj, visitor);
        break;
      }
#undef RAW_VISITPOINTERS
    case kByteBufferCid: {
      InstancePtr raw_obj = static_cast<InstancePtr>(this);
      size = InstanceLayout::VisitInstancePointers(raw_obj, visitor);
      break;
    }
    case kFfiPointerCid: {
      PointerPtr raw_obj = static_cast<PointerPtr>(this);
      size = PointerLayout::VisitPointerPointers(raw_obj, visitor);
      break;
    }
    case kFfiDynamicLibraryCid: {
      DynamicLibraryPtr raw_obj = static_cast<DynamicLibraryPtr>(this);
      size =
          DynamicLibraryLayout::VisitDynamicLibraryPointers(raw_obj, visitor);
      break;
    }
#define RAW_VISITPOINTERS(clazz) case kFfi##clazz##Cid:
      CLASS_LIST_FFI_TYPE_MARKER(RAW_VISITPOINTERS) {
        // NativeType do not have any fields or type arguments.
        size = HeapSize();
        break;
      }
#undef RAW_VISITPOINTERS
#define RAW_VISITPOINTERS(clazz) case k##clazz##Cid:
      CLASS_LIST_WASM(RAW_VISITPOINTERS) {
        // These wasm types do not have any fields or type arguments.
        size = HeapSize();
        break;
      }
#undef RAW_VISITPOINTERS
    case kFreeListElement: {
      uword addr = ObjectLayout::ToAddr(this);
      FreeListElement* element = reinterpret_cast<FreeListElement*>(addr);
      size = element->HeapSize();
      break;
    }
    case kForwardingCorpse: {
      uword addr = ObjectLayout::ToAddr(this);
      ForwardingCorpse* forwarder = reinterpret_cast<ForwardingCorpse*>(addr);
      size = forwarder->HeapSize();
      break;
    }
    case kNullCid:
    case kNeverCid:
      size = HeapSize();
      break;
    default:
      FATAL3("Invalid cid: %" Pd ", obj: %p, tags: %x. Corrupt heap?", class_id,
             this, static_cast<uint32_t>(tags_));
      break;
  }

#if defined(DEBUG)
  ASSERT(size != 0);
  const intptr_t expected_size = HeapSize();

  // In general we expect that visitors return exactly the same size that
  // HeapSize would compute. However in case of Arrays we might have a
  // discrepancy when concurrently visiting an array that is being shrunk with
  // Array::MakeFixedLength: the visitor might have visited the full array while
  // here we are observing a smaller HeapSize().
  ASSERT(size == expected_size ||
         (class_id == kArrayCid && size > expected_size));
  return size;  // Prefer larger size.
#else
  return size;
#endif
}

void ObjectLayout::VisitPointersPrecise(Isolate* isolate,
                                        ObjectPointerVisitor* visitor) {
  intptr_t class_id = GetClassId();
  if (class_id < kNumPredefinedCids) {
    VisitPointersPredefined(visitor, class_id);
    return;
  }

  // N.B.: Not using the heap size!
  uword next_field_offset = isolate->GetClassForHeapWalkAt(class_id)
                                ->ptr()
                                ->host_next_field_offset_in_words_
                            << kWordSizeLog2;
  ASSERT(next_field_offset > 0);
  uword obj_addr = ObjectLayout::ToAddr(this);
  uword from = obj_addr + sizeof(ObjectLayout);
  uword to = obj_addr + next_field_offset - kWordSize;
  const auto first = reinterpret_cast<ObjectPtr*>(from);
  const auto last = reinterpret_cast<ObjectPtr*>(to);

#if defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)
  const auto unboxed_fields_bitmap =
      visitor->shared_class_table()->GetUnboxedFieldsMapAt(class_id);

  if (!unboxed_fields_bitmap.IsEmpty()) {
    intptr_t bit = sizeof(ObjectLayout) / kWordSize;
    for (ObjectPtr* current = first; current <= last; current++) {
      if (!unboxed_fields_bitmap.Get(bit++)) {
        visitor->VisitPointer(current);
      }
    }
  } else {
    visitor->VisitPointers(first, last);
  }
#else
  visitor->VisitPointers(first, last);
#endif  // defined(SUPPORT_UNBOXED_INSTANCE_FIELDS)
}

bool ObjectLayout::FindObject(FindObjectVisitor* visitor) {
  ASSERT(visitor != NULL);
  return visitor->FindObject(static_cast<ObjectPtr>(this));
}

// Most objects are visited with this function. It calls the from() and to()
// methods on the raw object to get the first and last cells that need
// visiting.
#define REGULAR_VISITOR(Type)                                                  \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    /* Make sure that we got here with the tagged pointer as this. */          \
    ASSERT(raw_obj->IsHeapObject());                                           \
    ASSERT_UNCOMPRESSED(Type);                                                 \
    visitor->VisitPointers(raw_obj->ptr()->from(), raw_obj->ptr()->to());      \
    return Type::InstanceSize();                                               \
  }

// It calls the from() and to() methods on the raw object to get the first and
// last cells that need visiting.
//
// Though as opposed to Similar to [REGULAR_VISITOR] this visitor will call the
// specializd VisitTypedDataViewPointers
#define TYPED_DATA_VIEW_VISITOR(Type)                                          \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    /* Make sure that we got here with the tagged pointer as this. */          \
    ASSERT(raw_obj->IsHeapObject());                                           \
    ASSERT_UNCOMPRESSED(Type);                                                 \
    visitor->VisitTypedDataViewPointers(raw_obj, raw_obj->ptr()->from(),       \
                                        raw_obj->ptr()->to());                 \
    return Type::InstanceSize();                                               \
  }

// For variable length objects. get_length is a code snippet that gets the
// length of the object, which is passed to InstanceSize and the to() method.
#define VARIABLE_VISITOR(Type, get_length)                                     \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    /* Make sure that we got here with the tagged pointer as this. */          \
    ASSERT(raw_obj->IsHeapObject());                                           \
    intptr_t length = get_length;                                              \
    visitor->VisitPointers(raw_obj->ptr()->from(),                             \
                           raw_obj->ptr()->to(length));                        \
    return Type::InstanceSize(length);                                         \
  }

// For now there are no compressed pointers:
#define COMPRESSED_VISITOR(Type) REGULAR_VISITOR(Type)
#define VARIABLE_COMPRESSED_VISITOR(Type, get_length)                          \
  VARIABLE_VISITOR(Type, get_length)

// For fixed-length objects that don't have any pointers that need visiting.
#define NULL_VISITOR(Type)                                                     \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    /* Make sure that we got here with the tagged pointer as this. */          \
    ASSERT(raw_obj->IsHeapObject());                                           \
    ASSERT_NOTHING_TO_VISIT(Type);                                             \
    return Type::InstanceSize();                                               \
  }

// For objects that don't have any pointers that need visiting, but have a
// variable length.
#define VARIABLE_NULL_VISITOR(Type, get_length)                                \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    /* Make sure that we got here with the tagged pointer as this. */          \
    ASSERT(raw_obj->IsHeapObject());                                           \
    ASSERT_NOTHING_TO_VISIT(Type);                                             \
    intptr_t length = get_length;                                              \
    return Type::InstanceSize(length);                                         \
  }

// For objects that are never instantiated on the heap.
#define UNREACHABLE_VISITOR(Type)                                              \
  intptr_t Type##Layout::Visit##Type##Pointers(                                \
      Type##Ptr raw_obj, ObjectPointerVisitor* visitor) {                      \
    UNREACHABLE();                                                             \
    return 0;                                                                  \
  }

REGULAR_VISITOR(Class)
REGULAR_VISITOR(Bytecode)
REGULAR_VISITOR(Type)
REGULAR_VISITOR(TypeRef)
REGULAR_VISITOR(TypeParameter)
REGULAR_VISITOR(PatchClass)
REGULAR_VISITOR(Function)
COMPRESSED_VISITOR(Closure)
REGULAR_VISITOR(ClosureData)
REGULAR_VISITOR(SignatureData)
REGULAR_VISITOR(RedirectionData)
REGULAR_VISITOR(FfiTrampolineData)
REGULAR_VISITOR(Script)
REGULAR_VISITOR(Library)
REGULAR_VISITOR(LibraryPrefix)
REGULAR_VISITOR(Namespace)
REGULAR_VISITOR(ParameterTypeCheck)
REGULAR_VISITOR(SingleTargetCache)
REGULAR_VISITOR(UnlinkedCall)
REGULAR_VISITOR(MonomorphicSmiableCall)
REGULAR_VISITOR(ICData)
REGULAR_VISITOR(MegamorphicCache)
REGULAR_VISITOR(ApiError)
REGULAR_VISITOR(LanguageError)
REGULAR_VISITOR(UnhandledException)
REGULAR_VISITOR(UnwindError)
REGULAR_VISITOR(ExternalOneByteString)
REGULAR_VISITOR(ExternalTwoByteString)
COMPRESSED_VISITOR(GrowableObjectArray)
COMPRESSED_VISITOR(LinkedHashMap)
COMPRESSED_VISITOR(ExternalTypedData)
TYPED_DATA_VIEW_VISITOR(TypedDataView)
REGULAR_VISITOR(ReceivePort)
REGULAR_VISITOR(StackTrace)
REGULAR_VISITOR(RegExp)
REGULAR_VISITOR(WeakProperty)
REGULAR_VISITOR(MirrorReference)
REGULAR_VISITOR(UserTag)
REGULAR_VISITOR(SubtypeTestCache)
REGULAR_VISITOR(LoadingUnit)
REGULAR_VISITOR(KernelProgramInfo)
VARIABLE_VISITOR(TypeArguments, Smi::Value(raw_obj->ptr()->length_))
VARIABLE_VISITOR(LocalVarDescriptors, raw_obj->ptr()->num_entries_)
VARIABLE_VISITOR(ExceptionHandlers, raw_obj->ptr()->num_entries_)
VARIABLE_VISITOR(Context, raw_obj->ptr()->num_variables_)
VARIABLE_COMPRESSED_VISITOR(Array, Smi::Value(raw_obj->ptr()->length_))
VARIABLE_COMPRESSED_VISITOR(
    TypedData,
    TypedData::ElementSizeInBytes(raw_obj->GetClassId()) *
        Smi::Value(raw_obj->ptr()->length_))
VARIABLE_VISITOR(ContextScope, raw_obj->ptr()->num_variables_)
NULL_VISITOR(Mint)
NULL_VISITOR(Double)
NULL_VISITOR(Float32x4)
NULL_VISITOR(Int32x4)
NULL_VISITOR(Float64x2)
NULL_VISITOR(Bool)
NULL_VISITOR(Capability)
NULL_VISITOR(SendPort)
NULL_VISITOR(TransferableTypedData)
NULL_VISITOR(ImageHeader)
REGULAR_VISITOR(Pointer)
NULL_VISITOR(DynamicLibrary)
VARIABLE_NULL_VISITOR(Instructions, Instructions::Size(raw_obj))
VARIABLE_NULL_VISITOR(InstructionsSection, InstructionsSection::Size(raw_obj))
VARIABLE_NULL_VISITOR(PcDescriptors, raw_obj->ptr()->length_)
VARIABLE_NULL_VISITOR(CodeSourceMap, raw_obj->ptr()->length_)
VARIABLE_NULL_VISITOR(CompressedStackMaps,
                      CompressedStackMaps::PayloadSizeOf(raw_obj))
VARIABLE_NULL_VISITOR(OneByteString, Smi::Value(raw_obj->ptr()->length_))
VARIABLE_NULL_VISITOR(TwoByteString, Smi::Value(raw_obj->ptr()->length_))
// Abstract types don't have their visitor called.
UNREACHABLE_VISITOR(AbstractType)
UNREACHABLE_VISITOR(CallSiteData)
UNREACHABLE_VISITOR(TypedDataBase)
UNREACHABLE_VISITOR(Error)
UNREACHABLE_VISITOR(Number)
UNREACHABLE_VISITOR(Integer)
UNREACHABLE_VISITOR(String)
UNREACHABLE_VISITOR(FutureOr)
// Smi has no heap representation.
UNREACHABLE_VISITOR(Smi)
#if defined(DART_PRECOMPILED_RUNTIME)
NULL_VISITOR(WeakSerializationReference)
#else
REGULAR_VISITOR(WeakSerializationReference)
#endif

intptr_t FieldLayout::VisitFieldPointers(FieldPtr raw_obj,
                                         ObjectPointerVisitor* visitor) {
  ASSERT(raw_obj->IsHeapObject());
  ASSERT_UNCOMPRESSED(Field);
  visitor->VisitPointers(raw_obj->ptr()->from(), raw_obj->ptr()->to());

  if (visitor->trace_values_through_fields()) {
    if (Field::StaticBit::decode(raw_obj->ptr()->kind_bits_)) {
      visitor->isolate_group()->ForEachIsolate(
          [&](Isolate* isolate) {
            intptr_t index =
                Smi::Value(raw_obj->ptr()->host_offset_or_field_id_);
            visitor->VisitPointer(&isolate->field_table()->table()[index]);
          },
          /*at_safepoint=*/true);
    }
  }
  return Field::InstanceSize();
}

bool CodeLayout::ContainsPC(const ObjectPtr raw_obj, uword pc) {
  if (!raw_obj->IsCode()) return false;
  auto const raw_code = static_cast<const CodePtr>(raw_obj);
  const uword start = Code::PayloadStartOf(raw_code);
  const uword size = Code::PayloadSizeOf(raw_code);
  return (pc - start) <= size;  // pc may point just past last instruction.
}

intptr_t CodeLayout::VisitCodePointers(CodePtr raw_obj,
                                       ObjectPointerVisitor* visitor) {
  visitor->VisitPointers(raw_obj->ptr()->from(), raw_obj->ptr()->to());

  CodeLayout* obj = raw_obj->ptr();
  intptr_t length = Code::PtrOffBits::decode(obj->state_bits_);
#if defined(TARGET_ARCH_IA32)
  // On IA32 only we embed pointers to objects directly in the generated
  // instructions. The variable portion of a Code object describes where to
  // find those pointers for tracing.
  if (Code::AliveBit::decode(obj->state_bits_)) {
    uword entry_point = Code::PayloadStartOf(raw_obj);
    for (intptr_t i = 0; i < length; i++) {
      int32_t offset = obj->data()[i];
      visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(entry_point + offset));
    }
  }
  return Code::InstanceSize(length);
#else
  // On all other architectures, objects are referenced indirectly through
  // either an ObjectPool or Thread.
  ASSERT(length == 0);
  return Code::InstanceSize(0);
#endif
}

bool BytecodeLayout::ContainsPC(ObjectPtr raw_obj, uword pc) {
  if (raw_obj->IsBytecode()) {
    BytecodePtr raw_bytecode = static_cast<BytecodePtr>(raw_obj);
    uword start = raw_bytecode->ptr()->instructions_;
    uword size = raw_bytecode->ptr()->instructions_size_;
    return (pc - start) <= size;  // pc may point past last instruction.
  }
  return false;
}

intptr_t ObjectPoolLayout::VisitObjectPoolPointers(
    ObjectPoolPtr raw_obj,
    ObjectPointerVisitor* visitor) {
  const intptr_t length = raw_obj->ptr()->length_;
  ObjectPoolLayout::Entry* entries = raw_obj->ptr()->data();
  uint8_t* entry_bits = raw_obj->ptr()->entry_bits();
  for (intptr_t i = 0; i < length; ++i) {
    ObjectPool::EntryType entry_type =
        ObjectPool::TypeBits::decode(entry_bits[i]);
    if ((entry_type == ObjectPool::EntryType::kTaggedObject) ||
        (entry_type == ObjectPool::EntryType::kNativeEntryData)) {
      visitor->VisitPointer(&entries[i].raw_obj_);
    }
  }
  return ObjectPool::InstanceSize(length);
}

bool InstructionsLayout::ContainsPC(const InstructionsPtr raw_instr, uword pc) {
  const uword start = Instructions::PayloadStart(raw_instr);
  const uword size = Instructions::Size(raw_instr);
  // We use <= instead of < here because the saved-pc can be outside the
  // instruction stream if the last instruction is a call we don't expect to
  // return (e.g. because it throws an exception).
  return (pc - start) <= size;
}

intptr_t InstanceLayout::VisitInstancePointers(InstancePtr raw_obj,
                                               ObjectPointerVisitor* visitor) {
  // Make sure that we got here with the tagged pointer as this.
  ASSERT(raw_obj->IsHeapObject());
  uint32_t tags = raw_obj->ptr()->tags_;
  intptr_t instance_size = SizeTag::decode(tags);
  if (instance_size == 0) {
    instance_size = visitor->isolate_group()->GetClassSizeForHeapWalkAt(
        raw_obj->GetClassId());
  }

  // Calculate the first and last raw object pointer fields.
  uword obj_addr = ObjectLayout::ToAddr(raw_obj);
  uword from = obj_addr + sizeof(ObjectLayout);
  uword to = obj_addr + instance_size - kWordSize;
  visitor->VisitPointers(reinterpret_cast<ObjectPtr*>(from),
                         reinterpret_cast<ObjectPtr*>(to));
  return instance_size;
}

intptr_t ImmutableArrayLayout::VisitImmutableArrayPointers(
    ImmutableArrayPtr raw_obj,
    ObjectPointerVisitor* visitor) {
  return ArrayLayout::VisitArrayPointers(raw_obj, visitor);
}

void ObjectLayout::RememberCard(ObjectPtr const* slot) {
  OldPage::Of(static_cast<ObjectPtr>(this))->RememberCard(slot);
}

DEFINE_LEAF_RUNTIME_ENTRY(void,
                          RememberCard,
                          2,
                          uword /*ObjectPtr*/ object_in,
                          ObjectPtr* slot) {
  ObjectPtr object = static_cast<ObjectPtr>(object_in);
  ASSERT(object->IsOldObject());
  ASSERT(object->ptr()->IsCardRemembered());
  OldPage::Of(object)->RememberCard(slot);
}
END_LEAF_RUNTIME_ENTRY

const char* PcDescriptorsLayout::KindToCString(Kind k) {
  switch (k) {
#define ENUM_CASE(name, init)                                                  \
  case Kind::k##name:                                                          \
    return #name;
    FOR_EACH_RAW_PC_DESCRIPTOR(ENUM_CASE)
#undef ENUM_CASE
    default:
      return nullptr;
  }
}

bool PcDescriptorsLayout::ParseKind(const char* cstr, Kind* out) {
  ASSERT(cstr != nullptr && out != nullptr);
#define ENUM_CASE(name, init)                                                  \
  if (strcmp(#name, cstr) == 0) {                                              \
    *out = Kind::k##name;                                                      \
    return true;                                                               \
  }
  FOR_EACH_RAW_PC_DESCRIPTOR(ENUM_CASE)
#undef ENUM_CASE
  return false;
}
#undef PREFIXED_NAME

}  // namespace dart
```

