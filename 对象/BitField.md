# BitField

```c++
// ...

namespace dart {

// uword 1
static const uword kUwordOne = 1U;

// BitField 是用于在 S 型存储中编码和解码类型 T 值的模板
template <typename S, typename T, int position, int size>
class BitField {
 public:
  typedef T Type;

  static const intptr_t kNextBit = position + size;

  // Tells whether the provided value fits into the bit field.
  static constexpr bool is_valid(T value) {
    return (static_cast<S>(value) & ~((kUwordOne << size) - 1)) == 0;
  }

  // Returns a S mask of the bit field.
  static constexpr S mask() { return (kUwordOne << size) - 1; }

  // Returns a S mask of the bit field which can be applied directly to
  // to the raw unshifted bits.
  static constexpr S mask_in_place() {
    return ((kUwordOne << size) - 1) << position;
  }

  // Returns the shift count needed to right-shift the bit field to
  // the least-significant bits.
  static constexpr int shift() { return position; }

  // Returns the size of the bit field.
  static constexpr int bitsize() { return size; }

  // Returns an S with the bit field value encoded.
  static UNLESS_DEBUG(constexpr) S encode(T value) {
    DEBUG_ASSERT(is_valid(value));
    return static_cast<S>(value) << position;
  }

  // 从给定的 S 值总解码出比特
  static constexpr T decode(S value) {
    return static_cast<T>((value >> position) & ((kUwordOne << size) - 1));
  }

  // Returns an S with the bit field value encoded based on the
  // original value. Only the bits corresponding to this bit field
  // will be changed.
  static UNLESS_DEBUG(constexpr) S update(T value, S original) {
    DEBUG_ASSERT(is_valid(value));
    return (static_cast<S>(value) << position) | (~mask_in_place() & original);
  }
};

}  // namespace dart

#endif  // RUNTIME_VM_BITFIELD_H
```

