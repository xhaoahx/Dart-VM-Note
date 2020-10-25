# BitField

```c++
// ...

namespace dart {

// uword 1
static const uword kUwordOne = 1U;

// BitField 是用于在 S 型存储中编码和解码类型 T 值的模板
// position 是 T 在 S 中的位置，size 是 T 在 S 中的 size
template <typename S, typename T, int position, int size>
class BitField {
 public:
  typedef T Type;

  static const intptr_t kNextBit = position + size;

  // 给定的值是否是有效的
  static constexpr bool is_valid(T value) {
    return (static_cast<S>(value) & ~((kUwordOne << size) - 1)) == 0;
  }

  // T 的 mask，使用时必须先逻辑移动
  static constexpr S mask() { return (kUwordOne << size) - 1; }

  // 用于直接从 S 中取出 T 的 mask
  static constexpr S mask_in_place() {
    return ((kUwordOne << size) - 1) << position;
  }

  // S 需要逻辑移动的位数，即 S >> S.shift() & S.mask() == T
  static constexpr int shift() { return position; }

  // T 的大小
  static constexpr int bitsize() { return size; }

  // 将 T 编码成 S
  static UNLESS_DEBUG(constexpr) S encode(T value) {
    DEBUG_ASSERT(is_valid(value));
    return static_cast<S>(value) << position;
  }

  // 从给定的 S 值中解码出 T
  static constexpr T decode(S value) {
    return static_cast<T>((value >> position) & ((kUwordOne << size) - 1));
  }

  // 在 S 中设置 T 的值并返回新的 S
  static UNLESS_DEBUG(constexpr) S update(T value, S original) {
    DEBUG_ASSERT(is_valid(value));
    return (static_cast<S>(value) << position) | (~mask_in_place() & original);
  }
};

}  // namespace dart

#endif  // RUNTIME_VM_BITFIELD_H
```

