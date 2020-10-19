# BitSet



## bit_set.h

```c++
// ...
namespace dart {

// 就像它在 STL 中的名称一样，BitSet 对象包含固定长度的位序列。
// Just like its namesake in the STL, a BitSet object contains a fixed
// length sequence of bits.
    
// 在 FreeList 中有域 BitSet<kNumLists> free_map_，其中 KnumLists = 128
// FreeList.free_lists_[128] 存放的是大于 128 * (1 << kObjectAlignmentLog2) 大小的内存
template <intptr_t N>
class BitSet {
 public:
  BitSet() { 
      Reset(); 
  }

  /// 将 i 的映射位标记成 value
  void Set(intptr_t i, bool value) {
    // 0 <= i < 128
    ASSERT(i >= 0);
    ASSERT(i < N);
    // kBitPerWord 在 64 位 windows 系统上的值为 64
    // 因此 i & (kBitsPerWord - 1) 的值为 i 的低 6 位
    
    // 假设 value = true
    // 例如 i = 7， 则 i & (kBitsPerWord - 1) = 7，mask = 1<<7 = 2^7 = 1000000， data_[0] |= mask
    // 例如 i = 71，则 i & (kBitsPerWord - 1) = 7，mask = 1<<7 = 2^7 = 1000000， data_[1] |= mask 
    // 例如 i = 72，则 i & (kBitsPerWord - 1) = 8，mask = 1<<8 = 2^8 = 10000000，data_[1] |= mask
    // 且 mask 的最大值为 1<<63
    uword mask = (static_cast<uword>(1) << (i & (kBitsPerWord - 1)));
    if (value) {
      data_[i >> kBitsPerWordLog2] |= mask;  // 置1
    } else {
      data_[i >> kBitsPerWordLog2] &= ~mask; // 置0
    }
  }

  /// 返回 i 的映射位
  bool Test(intptr_t i) const {
    ASSERT(i >= 0);
    ASSERT(i < N);
    uword mask = (static_cast<uword>(1) << (i & (kBitsPerWord - 1)));
    return (data_[i >> kBitsPerWordLog2] & mask) != 0;
  }

  /// 下一位有数据的位
  intptr_t Next(intptr_t i) const {
    ASSERT(i >= 0);
    ASSERT(i < N);
    intptr_t w = i >> kBitsPerWordLog2;
    // mask = 2^65-1 (11111..11111) << (i & (kBitsPerWord - 1)，即后几位为 0
    uword mask = ~static_cast<uword>(0) << (i & (kBitsPerWord - 1));
    /// 如果高位还有数据
    if ((data_[w] & mask) != 0) {
      // Utils::CountTrailingZerosWord 计算末尾 0
      // Utils::CountLeadingZerosWord  计算前导 0
      uword tz = Utils::CountTrailingZerosWord(data_[w] & mask);
      // 还原原数
      return (w << kBitsPerWordLog2) + tz;	// = i + tz
    }
    while (++w < kLengthInWords) {
      if (data_[w] != 0) {
        return (w << kBitsPerWordLog2) +
               Utils::CountTrailingZerosWord(data_[w]);
      }
    }
    return -1;
  }

  /// 最后一位有数据的位
  intptr_t Last() const {
    // w = 1
    for (int w = kLengthInWords - 1; w >= 0; --w) {
      uword d = data_[w];
      // 如果有置位
      if (d != 0) {
        return ((w + 1) << kBitsPerWordLog2) - Utils::CountLeadingZerosWord(d) - 1;
      }
    }
    return -1;
  }

  /// 清除最后一位并且找到前一位
  intptr_t ClearLastAndFindPrevious(intptr_t current_last) {
    ASSERT(Test(current_last));
    ASSERT(Last() == current_last);
    intptr_t w = current_last >> kBitsPerWordLog2;
    uword bits = data_[w];
    // Clear the current last.
    bits ^= (static_cast<uword>(1) << (current_last & (kBitsPerWord - 1)));
    data_[w] = bits;
    // Search backwards for a non-zero word.
    while (bits == 0 && w > 0) {
      bits = data_[--w];
    }
    if (bits == 0) {
      // None found.
      return -1;
    } else {
      // Bitlength incl. w, minus leading zeroes of w, minus 1 to 0-based index.
      return ((w + 1) << kBitsPerWordLog2) -
             Utils::CountLeadingZerosWord(bits) - 1;
    }
  }

  /// 置 0
  void Reset() { memset(data_, 0, sizeof(data_)); }

  intptr_t Size() const { return N; }

 private:
  static const int kLengthInWords = 1 + ((N - 1) / kBitsPerWord); // = 1 + (128 - 1)/64 = 2
  // data[0],data[1] 各保存 64 位
  uword data_[kLengthInWords];
};

}  // namespace dart

#endif  // RUNTIME_VM_BIT_SET_H_
```

