# Number

## object.h

### Number

```c++
class Number : public Instance {
 public:
  // TODO(iposva): Add more useful Number methods.
  StringPtr ToString(Heap::Space space) const;

  // Numbers are canonicalized differently from other instances/strings.
  // Caller must hold Isolate::constant_canonicalization_mutex_.
  virtual InstancePtr CanonicalizeLocked(Thread* thread) const;

#if defined(DEBUG)
  // Check if number is canonical.
  virtual bool CheckIsCanonical(Thread* thread) const;
#endif  // DEBUG

 private:
  OBJECT_IMPLEMENTATION(Number, Instance);

  friend class Class;
};

```

### Integer

```c++
class Integer : public Number {
 public:
  static IntegerPtr New(const String& str, Heap::Space space = Heap::kNew);

  // Creates a new Integer by given uint64_t value.
  // Silently casts value to int64_t with wrap-around if it is greater
  // than kMaxInt64.
  static IntegerPtr NewFromUint64(uint64_t value,
                                  Heap::Space space = Heap::kNew);

  // Returns a canonical Integer object allocated in the old gen space.
  // Returns null if integer is out of range.
  static IntegerPtr NewCanonical(const String& str);
  static IntegerPtr NewCanonical(int64_t value);

  static IntegerPtr New(int64_t value, Heap::Space space = Heap::kNew);

  // Returns true iff the given uint64_t value is representable as Dart integer.
  static bool IsValueInRange(uint64_t value);

  virtual bool OperatorEquals(const Instance& other) const {
    return Equals(other);
  }
  virtual bool CanonicalizeEquals(const Instance& other) const {
    return Equals(other);
  }
  virtual uint32_t CanonicalizeHash() const { return AsTruncatedUint32Value(); }
  virtual bool Equals(const Instance& other) const;

  virtual ObjectPtr HashCode() const { return raw(); }

  virtual bool IsZero() const;
  virtual bool IsNegative() const;

  virtual double AsDoubleValue() const;
  virtual int64_t AsInt64Value() const;
  virtual int64_t AsTruncatedInt64Value() const { return AsInt64Value(); }
  virtual uint32_t AsTruncatedUint32Value() const;

  virtual bool FitsIntoSmi() const;

  // Returns 0, -1 or 1.
  virtual int CompareWith(const Integer& other) const;

  // Converts integer to hex string.
  const char* ToHexCString(Zone* zone) const;

  // Return the most compact presentation of an integer.
  IntegerPtr AsValidInteger() const;

  // Returns null to indicate that a bigint operation is required.
  IntegerPtr ArithmeticOp(Token::Kind operation,
                          const Integer& other,
                          Heap::Space space = Heap::kNew) const;
  IntegerPtr BitOp(Token::Kind operation,
                   const Integer& other,
                   Heap::Space space = Heap::kNew) const;
  IntegerPtr ShiftOp(Token::Kind operation,
                     const Integer& other,
                     Heap::Space space = Heap::kNew) const;

  static int64_t GetInt64Value(const IntegerPtr obj) {
    intptr_t raw_value = static_cast<intptr_t>(obj);
    if ((raw_value & kSmiTagMask) == kSmiTag) {
      return (raw_value >> kSmiTagShift);
    } else {
      ASSERT(obj->IsMint());
      return static_cast<const MintPtr>(obj)->ptr()->value_;
    }
  }

 private:
  OBJECT_IMPLEMENTATION(Integer, Number);
  friend class Class;
};
```

### Smi

```c++
class Smi : public Integer {
 public:
  static const intptr_t kBits = kSmiBits;
  static const intptr_t kMaxValue = kSmiMax;
  static const intptr_t kMinValue = kSmiMin;

  intptr_t Value() const { return RawSmiValue(raw()); }

  virtual bool Equals(const Instance& other) const;
  virtual bool IsZero() const { return Value() == 0; }
  virtual bool IsNegative() const { return Value() < 0; }

  virtual double AsDoubleValue() const;
  virtual int64_t AsInt64Value() const;
  virtual uint32_t AsTruncatedUint32Value() const;

  virtual bool FitsIntoSmi() const { return true; }

  virtual int CompareWith(const Integer& other) const;

  static intptr_t InstanceSize() { return 0; }

  static SmiPtr New(intptr_t value) {
    SmiPtr raw_smi = static_cast<SmiPtr>(
        (static_cast<uintptr_t>(value) << kSmiTagShift) | kSmiTag);
    ASSERT(RawSmiValue(raw_smi) == value);
    return raw_smi;
  }

  static SmiPtr FromAlignedAddress(uword address) {
    ASSERT((address & kSmiTagMask) == kSmiTag);
    return static_cast<SmiPtr>(address);
  }

  static ClassPtr Class();

  static intptr_t Value(const SmiPtr raw_smi) { return RawSmiValue(raw_smi); }

  static intptr_t RawValue(intptr_t value) {
    return static_cast<intptr_t>(New(value));
  }

  static bool IsValid(int64_t value) { return compiler::target::IsSmi(value); }

  void operator=(SmiPtr value) {
    raw_ = value;
    CHECK_HANDLE();
  }
  void operator^=(ObjectPtr value) {
    raw_ = value;
    CHECK_HANDLE();
  }

 private:
  static intptr_t NextFieldOffset() {
    // Indicates this class cannot be extended by dart code.
    return -kWordSize;
  }

  Smi() : Integer() {}
  BASE_OBJECT_IMPLEMENTATION(Smi, Integer);
  OBJECT_SERVICE_SUPPORT(Smi);
  friend class Api;  // For ValueFromRaw
  friend class Class;
  friend class Object;
  friend class ReusableSmiHandleScope;
  friend class Thread;
}
```



## Object.cc

### Number

```c++

```

### Integer

```c++

```

### Smi

```c++

```



