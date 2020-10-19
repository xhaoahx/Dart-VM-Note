# Utils

# utils.h

```c++
const intptr_t kOffsetOfPtr = 32;

// 某个 field 在 type 中的偏移
#define OFFSET_OF(type, field)                                                 \
  (reinterpret_cast<intptr_t>(                                                 \
       &(reinterpret_cast<type*>(kOffsetOfPtr)->field)) -                      \
   kOffsetOfPtr)  // NOLINT

/// 以下6个函数均要求 n 是 2 的次方

// 是否是排列的
// x & (n - 1) 等价于 x % n
// 即判断 x 是否是 n 的整数倍
template <typename T>
static inline bool IsAligned(T x, intptr_t n) {
    ASSERT(IsPowerOfTwo(n));
    return (x & (n - 1)) == 0;
}

// 特化版本
template <typename T>
static inline bool IsAligned(T* x, intptr_t n) {
    return IsAligned(reinterpret_cast<uword>(x), n);
}

// 向下取整，即把 x 向最近最小的 n 的整数倍取整
// 例如：n = 16,则 x = 
// 0~15 -> 0
// 16~31 -> 16
// 32~47 -> 32
// 48~63 -> 48
// 64~79 -> 64
// 80~95 -> 80
// ...
template <typename T>
static inline T RoundDown(T x, intptr_t n) {
    ASSERT(IsPowerOfTwo(n));
    return (x & -n);
}

// 特化版本
template <typename T>
static inline T* RoundDown(T* x, intptr_t n) {
    return reinterpret_cast<T*>(RoundDown(reinterpret_cast<uword>(x), n));
}

// 向上取整，即把 x 向最近最大的 n 的整数倍取整
// 例如：n = 16,则 x = 
// 0~16 -> 16
// 17~32 -> 32
// 33~48 -> 48
// 49~64 -> 64
// 65~80 -> 80
// 81~96 -> 96
// ...
template <typename T>
static inline T RoundUp(T x, intptr_t n) {
    return RoundDown(x + n - 1, n);
}

// 特化版本
template <typename T>
static inline T* RoundUp(T* x, intptr_t n) {
    return reinterpret_cast<T*>(RoundUp(reinterpret_cast<uword>(x), n));
}

```

