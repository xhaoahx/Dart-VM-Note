# Instance

## object.h

```c++
// Instance is the base class for all instance objects (aka the Object class
// in Dart source code.
class Instance : public Object {
 public:
  // Equality and identity testing.
  // 1. OperatorEquals: true iff 'this == other' is true in Dart code.
  // 2. IsIdenticalTo: true iff 'identical(this, other)' is true in Dart code.
  // 3. CanonicalizeEquals: used to canonicalize compile-time constants, e.g.,
  //    using bitwise equality of fields and list elements.
  // Subclasses where 1 and 3 coincide may also define a plain Equals, e.g.,
  // String and Integer.
  virtual bool OperatorEquals(const Instance& other) const;
  bool IsIdenticalTo(const Instance& other) const;
  virtual bool CanonicalizeEquals(const Instance& other) const;
  virtual uint32_t CanonicalizeHash() const;

  intptr_t SizeFromClass() const {
#if defined(DEBUG)
    const Class& cls = Class::Handle(clazz());
    ASSERT(cls.is_finalized() || cls.is_prefinalized());
#endif
    return (clazz()->ptr()->host_instance_size_in_words_ * kWordSize);
  }

  InstancePtr Canonicalize(Thread* thread) const;
  // Caller must hold Isolate::constant_canonicalization_mutex_.
  virtual InstancePtr CanonicalizeLocked(Thread* thread) const;
  virtual void CanonicalizeFieldsLocked(Thread* thread) const;

  InstancePtr CopyShallowToOldSpace(Thread* thread) const;

#if defined(DEBUG)
  // Check if instance is canonical.
  virtual bool CheckIsCanonical(Thread* thread) const;
#endif  // DEBUG

  ObjectPtr GetField(const Field& field) const;

  void SetField(const Field& field, const Object& value) const;

  AbstractTypePtr GetType(Heap::Space space) const;

  // Access the arguments of the [Type] of this [Instance].
  // Note: for [Type]s instead of [Instance]s with a [Type] attached, use
  // [arguments()] and [set_arguments()]
  virtual TypeArgumentsPtr GetTypeArguments() const;
  virtual void SetTypeArguments(const TypeArguments& value) const;

  // Check if the type of this instance is a subtype of the given other type.
  // The type argument vectors are used to instantiate the other type if needed.
  bool IsInstanceOf(const AbstractType& other,
                    const TypeArguments& other_instantiator_type_arguments,
                    const TypeArguments& other_function_type_arguments) const;

  // Check if this instance is assignable to the given other type.
  // The type argument vectors are used to instantiate the other type if needed.
  bool IsAssignableTo(const AbstractType& other,
                      const TypeArguments& other_instantiator_type_arguments,
                      const TypeArguments& other_function_type_arguments) const;

  // Return true if the null instance can be assigned to a variable of [other]
  // type. Return false if null cannot be assigned or we cannot tell (if
  // [other] is a type parameter in NNBD strong mode).
  static bool NullIsAssignableTo(const AbstractType& other);

  bool IsValidNativeIndex(int index) const {
    return ((index >= 0) && (index < clazz()->ptr()->num_native_fields_));
  }

  intptr_t* NativeFieldsDataAddr() const;
  inline intptr_t GetNativeField(int index) const;
  inline void GetNativeFields(uint16_t num_fields,
                              intptr_t* field_values) const;
  void SetNativeFields(uint16_t num_fields, const intptr_t* field_values) const;

  uint16_t NumNativeFields() const {
    return clazz()->ptr()->num_native_fields_;
  }

  void SetNativeField(int index, intptr_t value) const;

  // If the instance is a callable object, i.e. a closure or the instance of a
  // class implementing a 'call' method, return true and set the function
  // (if not NULL) to call.
  bool IsCallable(Function* function) const;

  ObjectPtr Invoke(const String& selector,
                   const Array& arguments,
                   const Array& argument_names,
                   bool respect_reflectable = true,
                   bool check_is_entrypoint = false) const;
  ObjectPtr InvokeGetter(const String& selector,
                         bool respect_reflectable = true,
                         bool check_is_entrypoint = false) const;
  ObjectPtr InvokeSetter(const String& selector,
                         const Instance& argument,
                         bool respect_reflectable = true,
                         bool check_is_entrypoint = false) const;

  // Evaluate the given expression as if it appeared in an instance method of
  // this instance and return the resulting value, or an error object if
  // evaluating the expression fails. The method has the formal (type)
  // parameters given in (type_)param_names, and is invoked with the (type)
  // argument values given in (type_)param_values.
  ObjectPtr EvaluateCompiledExpression(
      const Class& method_cls,
      const ExternalTypedData& kernel_buffer,
      const Array& type_definitions,
      const Array& param_values,
      const TypeArguments& type_param_values) const;

  // Equivalent to invoking hashCode on this instance.
  virtual ObjectPtr HashCode() const;

  // Equivalent to invoking identityHashCode with this instance.
  ObjectPtr IdentityHashCode() const;

  static intptr_t InstanceSize() {
    return RoundedAllocationSize(sizeof(InstanceLayout));
  }

  static InstancePtr New(const Class& cls, Heap::Space space = Heap::kNew);

  // Array/list element address computations.
  static intptr_t DataOffsetFor(intptr_t cid);
  static intptr_t ElementSizeFor(intptr_t cid);

  // Pointers may be subtyped, but their subtypes may not get extra fields.
  // The subtype runtime representation has exactly the same object layout,
  // only the class_id is different. So, it is safe to use subtype instances in
  // Pointer handles.
  virtual bool IsPointer() const;

  static intptr_t NextFieldOffset() { return sizeof(InstanceLayout); }

 protected:
#ifndef PRODUCT
  virtual void PrintSharedInstanceJSON(JSONObject* jsobj, bool ref) const;
#endif

 private:
  // Return true if the runtimeType of this instance is a subtype of other type.
  bool RuntimeTypeIsSubtypeOf(
      const AbstractType& other,
      const TypeArguments& other_instantiator_type_arguments,
      const TypeArguments& other_function_type_arguments) const;

  // Returns true if the type of this instance is a subtype of FutureOr<T>
  // specified by instantiated type 'other'.
  // Returns false if other type is not a FutureOr.
  bool RuntimeTypeIsSubtypeOfFutureOr(Zone* zone,
                                      const AbstractType& other) const;

  // Return true if the null instance is an instance of other type.
  static bool NullIsInstanceOf(
      const AbstractType& other,
      const TypeArguments& other_instantiator_type_arguments,
      const TypeArguments& other_function_type_arguments);

  ObjectPtr* FieldAddrAtOffset(intptr_t offset) const {
    ASSERT(IsValidFieldOffset(offset));
    return reinterpret_cast<ObjectPtr*>(raw_value() - kHeapObjectTag + offset);
  }
  ObjectPtr* FieldAddr(const Field& field) const {
    return FieldAddrAtOffset(field.HostOffset());
  }
  ObjectPtr* NativeFieldsAddr() const {
    return FieldAddrAtOffset(sizeof(ObjectLayout));
  }
  void SetFieldAtOffset(intptr_t offset, const Object& value) const {
    StorePointer(FieldAddrAtOffset(offset), value.raw());
  }
  bool IsValidFieldOffset(intptr_t offset) const;

  // The following raw methods are used for morphing.
  // They are needed due to the extraction of the class in IsValidFieldOffset.
  ObjectPtr* RawFieldAddrAtOffset(intptr_t offset) const {
    return reinterpret_cast<ObjectPtr*>(raw_value() - kHeapObjectTag + offset);
  }
  ObjectPtr RawGetFieldAtOffset(intptr_t offset) const {
    return *RawFieldAddrAtOffset(offset);
  }
  void RawSetFieldAtOffset(intptr_t offset, const Object& value) const {
    StorePointer(RawFieldAddrAtOffset(offset), value.raw());
  }

  static InstancePtr NewFromCidAndSize(SharedClassTable* shared_class_table,
                                       classid_t cid,
                                       Heap::Space heap = Heap::kNew);

  // TODO(iposva): Determine if this gets in the way of Smi.
  HEAP_OBJECT_IMPLEMENTATION(Instance, Object);
  friend class ByteBuffer;
  friend class Class;
  friend class Closure;
  friend class Pointer;
  friend class DeferredObject;
  friend class RegExp;
  friend class SnapshotWriter;
  friend class StubCode;
  friend class TypedDataView;
  friend class InstanceSerializationCluster;
  friend class InstanceDeserializationCluster;
  friend class ClassDeserializationCluster;  // vtable
  friend class InstanceMorpher;
  friend class Obfuscator;  // RawGetFieldAtOffset, RawSetFieldAtOffset
};
```

## object.cc

```c++
ObjectPtr Instance::InvokeGetter(const String& getter_name,
                                 bool respect_reflectable,
                                 bool check_is_entrypoint) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  Class& klass = Class::Handle(zone, clazz());
  CHECK_ERROR(klass.EnsureIsFinalized(thread));
  const auto& inst_type_args =
      klass.NumTypeArguments() > 0
          ? TypeArguments::Handle(zone, GetTypeArguments())
          : Object::null_type_arguments();

  const String& internal_getter_name =
      String::Handle(zone, Field::GetterName(getter_name));
  Function& function = Function::Handle(
      zone, Resolver::ResolveDynamicAnyArgs(zone, klass, internal_getter_name));

  if (!function.IsNull() && check_is_entrypoint) {
    // The getter must correspond to either an entry-point field or a getter
    // method explicitly marked.
    Field& field = Field::Handle(zone);
    if (function.kind() == FunctionLayout::kImplicitGetter) {
      field = function.accessor_field();
    }
    if (!field.IsNull()) {
      CHECK_ERROR(field.VerifyEntryPoint(EntryPointPragma::kGetterOnly));
    } else {
      CHECK_ERROR(function.VerifyCallEntryPoint());
    }
  }

  // Check for method extraction when method extractors are not created.
  if (function.IsNull() && !FLAG_lazy_dispatchers) {
    function = Resolver::ResolveDynamicAnyArgs(zone, klass, getter_name);

    if (!function.IsNull() && check_is_entrypoint) {
      CHECK_ERROR(function.VerifyClosurizedEntryPoint());
    }

    if (!function.IsNull() && function.SafeToClosurize()) {
      const Function& closure_function =
          Function::Handle(zone, function.ImplicitClosureFunction());
      return closure_function.ImplicitInstanceClosure(*this);
    }
  }

  const int kTypeArgsLen = 0;
  const int kNumArgs = 1;
  const Array& args = Array::Handle(zone, Array::New(kNumArgs));
  args.SetAt(0, *this);
  const Array& args_descriptor = Array::Handle(
      zone,
      ArgumentsDescriptor::NewBoxed(kTypeArgsLen, args.Length(), Heap::kNew));

  return InvokeInstanceFunction(*this, function, internal_getter_name, args,
                                args_descriptor, respect_reflectable,
                                inst_type_args);
}

ObjectPtr Instance::InvokeSetter(const String& setter_name,
                                 const Instance& value,
                                 bool respect_reflectable,
                                 bool check_is_entrypoint) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  const Class& klass = Class::Handle(zone, clazz());
  CHECK_ERROR(klass.EnsureIsFinalized(thread));
  const auto& inst_type_args =
      klass.NumTypeArguments() > 0
          ? TypeArguments::Handle(zone, GetTypeArguments())
          : Object::null_type_arguments();

  const String& internal_setter_name =
      String::Handle(zone, Field::SetterName(setter_name));
  const Function& setter = Function::Handle(
      zone, Resolver::ResolveDynamicAnyArgs(zone, klass, internal_setter_name));

  if (check_is_entrypoint) {
    // The setter must correspond to either an entry-point field or a setter
    // method explicitly marked.
    Field& field = Field::Handle(zone);
    if (setter.kind() == FunctionLayout::kImplicitSetter) {
      field = setter.accessor_field();
    }
    if (!field.IsNull()) {
      CHECK_ERROR(field.VerifyEntryPoint(EntryPointPragma::kSetterOnly));
    } else if (!setter.IsNull()) {
      CHECK_ERROR(setter.VerifyCallEntryPoint());
    }
  }

  const int kTypeArgsLen = 0;
  const int kNumArgs = 2;
  const Array& args = Array::Handle(zone, Array::New(kNumArgs));
  args.SetAt(0, *this);
  args.SetAt(1, value);
  const Array& args_descriptor = Array::Handle(
      zone,
      ArgumentsDescriptor::NewBoxed(kTypeArgsLen, args.Length(), Heap::kNew));

  return InvokeInstanceFunction(*this, setter, internal_setter_name, args,
                                args_descriptor, respect_reflectable,
                                inst_type_args);
}

ObjectPtr Instance::Invoke(const String& function_name,
                           const Array& args,
                           const Array& arg_names,
                           bool respect_reflectable,
                           bool check_is_entrypoint) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  Class& klass = Class::Handle(zone, clazz());
  CHECK_ERROR(klass.EnsureIsFinalized(thread));

  Function& function = Function::Handle(
      zone, Resolver::ResolveDynamicAnyArgs(zone, klass, function_name));

  if (!function.IsNull() && check_is_entrypoint) {
    CHECK_ERROR(function.VerifyCallEntryPoint());
  }

  // We don't pass any explicit type arguments, which will be understood as
  // using dynamic for any function type arguments by lower layers.
  const int kTypeArgsLen = 0;
  const Array& args_descriptor = Array::Handle(
      zone, ArgumentsDescriptor::NewBoxed(kTypeArgsLen, args.Length(),
                                          arg_names, Heap::kNew));

  const auto& inst_type_args =
      klass.NumTypeArguments() > 0
          ? TypeArguments::Handle(zone, GetTypeArguments())
          : Object::null_type_arguments();

  if (function.IsNull()) {
    // Didn't find a method: try to find a getter and invoke call on its result.
    const String& getter_name =
        String::Handle(zone, Field::GetterName(function_name));
    function = Resolver::ResolveDynamicAnyArgs(zone, klass, getter_name);
    if (!function.IsNull()) {
      if (check_is_entrypoint) {
        CHECK_ERROR(EntryPointFieldInvocationError(function_name));
      }
      ASSERT(function.kind() != FunctionLayout::kMethodExtractor);
      // Invoke the getter.
      const int kNumArgs = 1;
      const Array& getter_args = Array::Handle(zone, Array::New(kNumArgs));
      getter_args.SetAt(0, *this);
      const Array& getter_args_descriptor = Array::Handle(
          zone, ArgumentsDescriptor::NewBoxed(
                    kTypeArgsLen, getter_args.Length(), Heap::kNew));
      const Object& getter_result = Object::Handle(
          zone, InvokeInstanceFunction(*this, function, getter_name,
                                       getter_args, getter_args_descriptor,
                                       respect_reflectable, inst_type_args));
      if (getter_result.IsError()) {
        return getter_result.raw();
      }
      // Replace the closure as the receiver in the arguments list.
      args.SetAt(0, getter_result);
      return InvokeCallableWithChecks(zone, args, args_descriptor);
    }
  }

  // Found an ordinary method.
  return InvokeInstanceFunction(*this, function, function_name, args,
                                args_descriptor, respect_reflectable,
                                inst_type_args);
}

ObjectPtr Instance::EvaluateCompiledExpression(
    const Class& method_cls,
    const ExternalTypedData& kernel_buffer,
    const Array& type_definitions,
    const Array& arguments,
    const TypeArguments& type_arguments) const {
  const Array& arguments_with_receiver =
      Array::Handle(Array::New(1 + arguments.Length()));
  PassiveObject& param = PassiveObject::Handle();
  arguments_with_receiver.SetAt(0, *this);
  for (intptr_t i = 0; i < arguments.Length(); i++) {
    param = arguments.At(i);
    arguments_with_receiver.SetAt(i + 1, param);
  }

  return EvaluateCompiledExpressionHelper(
      kernel_buffer, type_definitions,
      String::Handle(Library::Handle(method_cls.library()).url()),
      String::Handle(method_cls.UserVisibleName()), arguments_with_receiver,
      type_arguments);
}

ObjectPtr Instance::HashCode() const {
  // TODO(koda): Optimize for all builtin classes and all classes
  // that do not override hashCode.
  return DartLibraryCalls::HashCode(*this);
}

ObjectPtr Instance::IdentityHashCode() const {
  return DartLibraryCalls::IdentityHashCode(*this);
}

bool Instance::CanonicalizeEquals(const Instance& other) const {
  if (this->raw() == other.raw()) {
    return true;  // "===".
  }

  if (other.IsNull() || (this->clazz() != other.clazz())) {
    return false;
  }

  {
    NoSafepointScope no_safepoint;
    // Raw bits compare.
    const intptr_t instance_size = SizeFromClass();
    ASSERT(instance_size != 0);
    const intptr_t other_instance_size = other.SizeFromClass();
    ASSERT(other_instance_size != 0);
    if (instance_size != other_instance_size) {
      return false;
    }
    uword this_addr = reinterpret_cast<uword>(this->raw_ptr());
    uword other_addr = reinterpret_cast<uword>(other.raw_ptr());
    for (intptr_t offset = Instance::NextFieldOffset(); offset < instance_size;
         offset += kWordSize) {
      if ((*reinterpret_cast<ObjectPtr*>(this_addr + offset)) !=
          (*reinterpret_cast<ObjectPtr*>(other_addr + offset))) {
        return false;
      }
    }
  }
  return true;
}

uint32_t Instance::CanonicalizeHash() const {
  if (GetClassId() == kNullCid) {
    return 2011;  // Matches null_patch.dart.
  }
  Thread* thread = Thread::Current();
  uint32_t hash = thread->heap()->GetCanonicalHash(raw());
  if (hash != 0) {
    return hash;
  }
  const Class& cls = Class::Handle(clazz());
  NoSafepointScope no_safepoint(thread);
  const intptr_t instance_size = SizeFromClass();
  ASSERT(instance_size != 0);
  hash = instance_size / kWordSize;
  uword this_addr = reinterpret_cast<uword>(this->raw_ptr());
  Instance& member = Instance::Handle();

  const auto unboxed_fields_bitmap =
      thread->isolate()->group()->shared_class_table()->GetUnboxedFieldsMapAt(
          GetClassId());

  for (intptr_t offset = Instance::NextFieldOffset();
       offset < cls.host_next_field_offset(); offset += kWordSize) {
    if (unboxed_fields_bitmap.Get(offset / kWordSize)) {
      if (kWordSize == 8) {
        hash = CombineHashes(hash,
                             *reinterpret_cast<uint32_t*>(this_addr + offset));
        hash = CombineHashes(
            hash, *reinterpret_cast<uint32_t*>(this_addr + offset + 4));
      } else {
        hash = CombineHashes(hash,
                             *reinterpret_cast<uint32_t*>(this_addr + offset));
      }
    } else {
      member ^= *reinterpret_cast<ObjectPtr*>(this_addr + offset);
      hash = CombineHashes(hash, member.CanonicalizeHash());
    }
  }
  hash = FinalizeHash(hash, String::kHashBits);
  thread->heap()->SetCanonicalHash(raw(), hash);
  return hash;
}

#if defined(DEBUG)
class CheckForPointers : public ObjectPointerVisitor {
 public:
  explicit CheckForPointers(IsolateGroup* isolate_group)
      : ObjectPointerVisitor(isolate_group), has_pointers_(false) {}

  bool has_pointers() const { return has_pointers_; }

  void VisitPointers(ObjectPtr* first, ObjectPtr* last) {
    if (first != last) {
      has_pointers_ = true;
    }
  }

 private:
  bool has_pointers_;

  DISALLOW_COPY_AND_ASSIGN(CheckForPointers);
};
#endif  // DEBUG

void Instance::CanonicalizeFieldsLocked(Thread* thread) const {
  const intptr_t class_id = GetClassId();
  if (class_id >= kNumPredefinedCids) {
    // Iterate over all fields, canonicalize numbers and strings, expect all
    // other instances to be canonical otherwise report error (return false).
    Zone* zone = thread->zone();
    Instance& obj = Instance::Handle(zone);
    const intptr_t instance_size = SizeFromClass();
    ASSERT(instance_size != 0);
    const auto unboxed_fields_bitmap =
        thread->isolate()->group()->shared_class_table()->GetUnboxedFieldsMapAt(
            class_id);
    for (intptr_t offset = Instance::NextFieldOffset(); offset < instance_size;
         offset += kWordSize) {
      if (unboxed_fields_bitmap.Get(offset / kWordSize)) {
        continue;
      }
      obj ^= *this->FieldAddrAtOffset(offset);
      obj = obj.CanonicalizeLocked(thread);
      this->SetFieldAtOffset(offset, obj);
    }
  } else {
#if defined(DEBUG)
    // Make sure that we are not missing any fields.
    CheckForPointers has_pointers(Isolate::Current()->group());
    this->raw()->ptr()->VisitPointers(&has_pointers);
    ASSERT(!has_pointers.has_pointers());
#endif  // DEBUG
  }
}

InstancePtr Instance::CopyShallowToOldSpace(Thread* thread) const {
  return Instance::RawCast(Object::Clone(*this, Heap::kOld));
}

InstancePtr Instance::Canonicalize(Thread* thread) const {
  SafepointMutexLocker ml(thread->isolate()->constant_canonicalization_mutex());
  return CanonicalizeLocked(thread);
}

InstancePtr Instance::CanonicalizeLocked(Thread* thread) const {
  if (this->IsCanonical()) {
    return this->raw();
  }
  ASSERT(!IsNull());
  CanonicalizeFieldsLocked(thread);
  Zone* zone = thread->zone();
  const Class& cls = Class::Handle(zone, this->clazz());
  Instance& result =
      Instance::Handle(zone, cls.LookupCanonicalInstance(zone, *this));
  if (!result.IsNull()) {
    return result.raw();
  }
  if (IsNew()) {
    ASSERT((thread->isolate() == Dart::vm_isolate()) || !InVMIsolateHeap());
    // Create a canonical object in old space.
    result ^= Object::Clone(*this, Heap::kOld);
  } else {
    result = this->raw();
  }
  ASSERT(result.IsOld());
  result.SetCanonical();
  return cls.InsertCanonicalConstant(zone, result);
}

#if defined(DEBUG)
bool Instance::CheckIsCanonical(Thread* thread) const {
  Zone* zone = thread->zone();
  Isolate* isolate = thread->isolate();
  Instance& result = Instance::Handle(zone);
  const Class& cls = Class::Handle(zone, this->clazz());
  SafepointMutexLocker ml(isolate->constant_canonicalization_mutex());
  result ^= cls.LookupCanonicalInstance(zone, *this);
  return (result.raw() == this->raw());
}
#endif  // DEBUG

ObjectPtr Instance::GetField(const Field& field) const {
  if (FLAG_precompiled_mode && field.is_unboxing_candidate()) {
    switch (field.guarded_cid()) {
      case kDoubleCid:
        return Double::New(*reinterpret_cast<double_t*>(FieldAddr(field)));
      case kFloat32x4Cid:
        return Float32x4::New(
            *reinterpret_cast<simd128_value_t*>(FieldAddr(field)));
      case kFloat64x2Cid:
        return Float64x2::New(
            *reinterpret_cast<simd128_value_t*>(FieldAddr(field)));
      default:
        if (field.is_non_nullable_integer()) {
          return Integer::New(*reinterpret_cast<int64_t*>(FieldAddr(field)));
        } else {
          UNREACHABLE();
          return nullptr;
        }
    }
  } else {
    return *FieldAddr(field);
  }
}

void Instance::SetField(const Field& field, const Object& value) const {
  if (FLAG_precompiled_mode && field.is_unboxing_candidate()) {
    switch (field.guarded_cid()) {
      case kDoubleCid:
        StoreNonPointer(reinterpret_cast<double_t*>(FieldAddr(field)),
                        Double::Cast(value).value());
        break;
      case kFloat32x4Cid:
        StoreNonPointer(reinterpret_cast<simd128_value_t*>(FieldAddr(field)),
                        Float32x4::Cast(value).value());
        break;
      case kFloat64x2Cid:
        StoreNonPointer(reinterpret_cast<simd128_value_t*>(FieldAddr(field)),
                        Float64x2::Cast(value).value());
        break;
      default:
        if (field.is_non_nullable_integer()) {
          StoreNonPointer(reinterpret_cast<int64_t*>(FieldAddr(field)),
                          Integer::Cast(value).AsInt64Value());
        } else {
          UNREACHABLE();
        }
        break;
    }
  } else {
    field.RecordStore(value);
    const Object* stored_value = field.CloneForUnboxed(value);
    StorePointer(FieldAddr(field), stored_value->raw());
  }
}

AbstractTypePtr Instance::GetType(Heap::Space space) const {
  if (IsNull()) {
    return Type::NullType();
  }
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  const Class& cls = Class::Handle(zone, clazz());
  if (!cls.is_finalized()) {
    // Various predefined classes can be instantiated by the VM or
    // Dart_NewString/Integer/TypedData/... before the class is finalized.
    ASSERT(cls.is_prefinalized());
    cls.EnsureDeclarationLoaded();
  }
  if (cls.IsClosureClass()) {
    Function& signature = Function::Handle(
        zone, Closure::Cast(*this).GetInstantiatedSignature(zone));
    Type& type = Type::Handle(zone, signature.SignatureType());
    if (!type.IsFinalized()) {
      type.SetIsFinalized();
    }
    type ^= type.Canonicalize(thread, nullptr);
    return type.raw();
  }
  Type& type = Type::Handle(zone);
  if (!cls.IsGeneric()) {
    type = cls.DeclarationType();
  }
  if (type.IsNull()) {
    TypeArguments& type_arguments = TypeArguments::Handle(zone);
    if (cls.NumTypeArguments() > 0) {
      type_arguments = GetTypeArguments();
    }
    type = Type::New(cls, type_arguments, TokenPosition::kNoSource,
                     Nullability::kNonNullable, space);
    type.SetIsFinalized();
    type ^= type.Canonicalize(thread, nullptr);
  }
  return type.raw();
}

TypeArgumentsPtr Instance::GetTypeArguments() const {
  ASSERT(!IsType());
  const Class& cls = Class::Handle(clazz());
  intptr_t field_offset = cls.host_type_arguments_field_offset();
  ASSERT(field_offset != Class::kNoTypeArguments);
  TypeArguments& type_arguments = TypeArguments::Handle();
  type_arguments ^= *FieldAddrAtOffset(field_offset);
  return type_arguments.raw();
}

void Instance::SetTypeArguments(const TypeArguments& value) const {
  ASSERT(!IsType());
  ASSERT(value.IsNull() || value.IsCanonical());
  const Class& cls = Class::Handle(clazz());
  intptr_t field_offset = cls.host_type_arguments_field_offset();
  ASSERT(field_offset != Class::kNoTypeArguments);
  SetFieldAtOffset(field_offset, value);
}

/*
Specification of instance checks (e is T) and casts (e as T), where e evaluates
to a value v and v has runtime type S:

Instance checks (e is T) in weak checking mode in a legacy or opted-in library:
  If v == null and T is a legacy type
    return LEGACY_SUBTYPE(T, Null) || LEGACY_SUBTYPE(Object, T)
  If v == null and T is not a legacy type, return NNBD_SUBTYPE(Null, T)
  Otherwise return LEGACY_SUBTYPE(S, T)

Instance checks (e is T) in strong checking mode in a legacy or opted-in lib:
  If v == null and T is a legacy type
    return LEGACY_SUBTYPE(T, Null) || LEGACY_SUBTYPE(Object, T)
  Otherwise return NNBD_SUBTYPE(S, T)

Casts (e as T) in weak checking mode in a legacy or opted-in library:
  If LEGACY_SUBTYPE(S, T) then e as T evaluates to v.
  Otherwise a CastError is thrown.

Casts (e as T) in strong checking mode in a legacy or opted-in library:
  If NNBD_SUBTYPE(S, T) then e as T evaluates to v.
  Otherwise a CastError is thrown.
*/

bool Instance::IsInstanceOf(
    const AbstractType& other,
    const TypeArguments& other_instantiator_type_arguments,
    const TypeArguments& other_function_type_arguments) const {
  ASSERT(!other.IsDynamicType());
  if (IsNull()) {
    return Instance::NullIsInstanceOf(other, other_instantiator_type_arguments,
                                      other_function_type_arguments);
  }
  // In strong mode, compute NNBD_SUBTYPE(runtimeType, other).
  // In weak mode, compute LEGACY_SUBTYPE(runtimeType, other).
  return RuntimeTypeIsSubtypeOf(other, other_instantiator_type_arguments,
                                other_function_type_arguments);
}

bool Instance::IsAssignableTo(
    const AbstractType& other,
    const TypeArguments& other_instantiator_type_arguments,
    const TypeArguments& other_function_type_arguments) const {
  ASSERT(!other.IsDynamicType());
  // In weak mode type casts, whether in legacy or opted-in libraries, the null
  // instance is detected and handled in inlined code and therefore cannot be
  // encountered here as a Dart null receiver.
  ASSERT(Isolate::Current()->null_safety() || !IsNull());
  // In strong mode, compute NNBD_SUBTYPE(runtimeType, other).
  // In weak mode, compute LEGACY_SUBTYPE(runtimeType, other).
  return RuntimeTypeIsSubtypeOf(other, other_instantiator_type_arguments,
                                other_function_type_arguments);
}

// If 'other' type (once instantiated) is a legacy type:
//   return LEGACY_SUBTYPE(other, Null) || LEGACY_SUBTYPE(Object, other).
// Otherwise return NNBD_SUBTYPE(Null, T).
// Ignore value of strong flag value.
bool Instance::NullIsInstanceOf(
    const AbstractType& other,
    const TypeArguments& other_instantiator_type_arguments,
    const TypeArguments& other_function_type_arguments) {
  ASSERT(other.IsFinalized());
  ASSERT(!other.IsTypeRef());  // Must be dereferenced at compile time.
  if (other.IsNullable()) {
    // This case includes top types (void, dynamic, Object?).
    // The uninstantiated nullable type will remain nullable after
    // instantiation.
    return true;
  }
  if (other.IsFutureOrType()) {
    const auto& type = AbstractType::Handle(other.UnwrapFutureOr());
    return NullIsInstanceOf(type, other_instantiator_type_arguments,
                            other_function_type_arguments);
  }
  // No need to instantiate type, unless it is a type parameter.
  // Note that a typeref cannot refer to a type parameter.
  if (other.IsTypeParameter()) {
    auto& type = AbstractType::Handle(other.InstantiateFrom(
        other_instantiator_type_arguments, other_function_type_arguments,
        kAllFree, Heap::kOld));
    if (type.IsTypeRef()) {
      type = TypeRef::Cast(type).type();
    }
    return Instance::NullIsInstanceOf(type, Object::null_type_arguments(),
                                      Object::null_type_arguments());
  }
  return other.IsLegacy() && (other.IsObjectType() || other.IsNeverType());
}

bool Instance::NullIsAssignableTo(const AbstractType& other) {
  Thread* thread = Thread::Current();
  Isolate* isolate = thread->isolate();
  Zone* zone = thread->zone();

  // In weak mode, Null is a bottom type (according to LEGACY_SUBTYPE).
  if (!isolate->null_safety()) {
    return true;
  }
  // "Left Null" rule: null is assignable when destination type is either
  // legacy or nullable. Otherwise it is not assignable or we cannot tell
  // without instantiating type parameter.
  if (other.IsLegacy() || other.IsNullable()) {
    return true;
  }
  if (other.IsFutureOrType()) {
    return NullIsAssignableTo(
        AbstractType::Handle(zone, other.UnwrapFutureOr()));
  }
  return false;
}

bool Instance::RuntimeTypeIsSubtypeOf(
    const AbstractType& other,
    const TypeArguments& other_instantiator_type_arguments,
    const TypeArguments& other_function_type_arguments) const {
  ASSERT(other.IsFinalized());
  ASSERT(!other.IsTypeRef());  // Must be dereferenced at compile time.
  ASSERT(raw() != Object::sentinel().raw());
  // Instance may not have runtimeType dynamic, void, or Never.
  if (other.IsTopTypeForSubtyping()) {
    return true;
  }
  Thread* thread = Thread::Current();
  Isolate* isolate = thread->isolate();
  Zone* zone = thread->zone();
  // In weak testing mode, Null type is a subtype of any type.
  if (IsNull() && !isolate->null_safety()) {
    return true;
  }
  const Class& cls = Class::Handle(zone, clazz());
  if (cls.IsClosureClass()) {
    if (other.IsDartFunctionType() || other.IsDartClosureType() ||
        other.IsObjectType()) {
      return true;
    }
    AbstractType& instantiated_other = AbstractType::Handle(zone, other.raw());
    if (!other.IsInstantiated()) {
      instantiated_other = other.InstantiateFrom(
          other_instantiator_type_arguments, other_function_type_arguments,
          kAllFree, Heap::kOld);
      if (instantiated_other.IsTypeRef()) {
        instantiated_other = TypeRef::Cast(instantiated_other).type();
      }
      if (instantiated_other.IsTopTypeForSubtyping() ||
          instantiated_other.IsDartFunctionType()) {
        return true;
      }
    }
    if (RuntimeTypeIsSubtypeOfFutureOr(zone, instantiated_other)) {
      return true;
    }
    if (!instantiated_other.IsFunctionType()) {
      return false;
    }
    Function& other_signature =
        Function::Handle(zone, Type::Cast(instantiated_other).signature());
    const Function& sig_fun =
        Function::Handle(Closure::Cast(*this).GetInstantiatedSignature(zone));
    return sig_fun.IsSubtypeOf(other_signature, Heap::kOld);
  }
  TypeArguments& type_arguments = TypeArguments::Handle(zone);
  if (cls.NumTypeArguments() > 0) {
    type_arguments = GetTypeArguments();
    ASSERT(type_arguments.IsNull() || type_arguments.IsCanonical());
    // The number of type arguments in the instance must be greater or equal to
    // the number of type arguments expected by the instance class.
    // A discrepancy is allowed for closures, which borrow the type argument
    // vector of their instantiator, which may be of a subclass of the class
    // defining the closure. Truncating the vector to the correct length on
    // instantiation is unnecessary. The vector may therefore be longer.
    // Also, an optimization reuses the type argument vector of the instantiator
    // of generic instances when its layout is compatible.
    ASSERT(type_arguments.IsNull() ||
           (type_arguments.Length() >= cls.NumTypeArguments()));
  }
  AbstractType& instantiated_other = AbstractType::Handle(zone, other.raw());
  if (!other.IsInstantiated()) {
    instantiated_other = other.InstantiateFrom(
        other_instantiator_type_arguments, other_function_type_arguments,
        kAllFree, Heap::kOld);
    if (instantiated_other.IsTypeRef()) {
      instantiated_other = TypeRef::Cast(instantiated_other).type();
    }
    if (instantiated_other.IsTopTypeForSubtyping()) {
      return true;
    }
  }
  if (!instantiated_other.IsType()) {
    return false;
  }
  if (IsNull()) {
    ASSERT(isolate->null_safety());
    if (instantiated_other.IsNullType()) {
      return true;
    }
    if (RuntimeTypeIsSubtypeOfFutureOr(zone, instantiated_other)) {
      return true;
    }
    return !instantiated_other.IsNonNullable();
  }
  // RuntimeType of non-null instance is non-nullable, so there is no need to
  // check nullability of other type.
  return Class::IsSubtypeOf(cls, type_arguments, Nullability::kNonNullable,
                            instantiated_other, Heap::kOld);
}

bool Instance::RuntimeTypeIsSubtypeOfFutureOr(Zone* zone,
                                              const AbstractType& other) const {
  if (other.IsFutureOrType()) {
    const TypeArguments& other_type_arguments =
        TypeArguments::Handle(zone, other.arguments());
    const AbstractType& other_type_arg =
        AbstractType::Handle(zone, other_type_arguments.TypeAtNullSafe(0));
    if (other_type_arg.IsTopTypeForSubtyping()) {
      return true;
    }
    if (Class::Handle(zone, clazz()).IsFutureClass()) {
      const TypeArguments& type_arguments =
          TypeArguments::Handle(zone, GetTypeArguments());
      const AbstractType& type_arg =
          AbstractType::Handle(zone, type_arguments.TypeAtNullSafe(0));
      if (type_arg.IsSubtypeOf(other_type_arg, Heap::kOld)) {
        return true;
      }
    }
    // Retry RuntimeTypeIsSubtypeOf after unwrapping type arg of FutureOr.
    if (RuntimeTypeIsSubtypeOf(other_type_arg, Object::null_type_arguments(),
                               Object::null_type_arguments())) {
      return true;
    }
  }
  return false;
}

bool Instance::OperatorEquals(const Instance& other) const {
  // TODO(koda): Optimize for all builtin classes and all classes
  // that do not override operator==.
  return DartLibraryCalls::Equals(*this, other) == Object::bool_true().raw();
}

bool Instance::IsIdenticalTo(const Instance& other) const {
  if (raw() == other.raw()) return true;
  if (IsInteger() && other.IsInteger()) {
    return Integer::Cast(*this).Equals(other);
  }
  if (IsDouble() && other.IsDouble()) {
    double other_value = Double::Cast(other).value();
    return Double::Cast(*this).BitwiseEqualsToDouble(other_value);
  }
  return false;
}

intptr_t* Instance::NativeFieldsDataAddr() const {
  ASSERT(Thread::Current()->no_safepoint_scope_depth() > 0);
  TypedDataPtr native_fields = static_cast<TypedDataPtr>(*NativeFieldsAddr());
  if (native_fields == TypedData::null()) {
    return NULL;
  }
  return reinterpret_cast<intptr_t*>(native_fields->ptr()->data());
}

void Instance::SetNativeField(int index, intptr_t value) const {
  ASSERT(IsValidNativeIndex(index));
  Object& native_fields = Object::Handle(*NativeFieldsAddr());
  if (native_fields.IsNull()) {
    // Allocate backing storage for the native fields.
    native_fields = TypedData::New(kIntPtrCid, NumNativeFields());
    StorePointer(NativeFieldsAddr(), native_fields.raw());
  }
  intptr_t byte_offset = index * sizeof(intptr_t);
  TypedData::Cast(native_fields).SetIntPtr(byte_offset, value);
}

void Instance::SetNativeFields(uint16_t num_native_fields,
                               const intptr_t* field_values) const {
  ASSERT(num_native_fields == NumNativeFields());
  ASSERT(field_values != NULL);
  Object& native_fields = Object::Handle(*NativeFieldsAddr());
  if (native_fields.IsNull()) {
    // Allocate backing storage for the native fields.
    native_fields = TypedData::New(kIntPtrCid, NumNativeFields());
    StorePointer(NativeFieldsAddr(), native_fields.raw());
  }
  for (uint16_t i = 0; i < num_native_fields; i++) {
    intptr_t byte_offset = i * sizeof(intptr_t);
    TypedData::Cast(native_fields).SetIntPtr(byte_offset, field_values[i]);
  }
}

bool Instance::IsCallable(Function* function) const {
  Class& cls = Class::Handle(clazz());
  if (cls.IsClosureClass()) {
    if (function != nullptr) {
      *function = Closure::Cast(*this).function();
    }
    return true;
  }
  // Try to resolve a "call" method.
  Zone* zone = Thread::Current()->zone();
  Function& call_function = Function::Handle(
      zone, Resolver::ResolveDynamicAnyArgs(zone, cls, Symbols::Call(),
                                            /*allow_add=*/false));
  if (call_function.IsNull()) {
    return false;
  }
  if (function != nullptr) {
    *function = call_function.raw();
  }
  return true;
}

InstancePtr Instance::New(const Class& cls, Heap::Space space) {
  Thread* thread = Thread::Current();
  if (cls.EnsureIsFinalized(thread) != Error::null()) {
    return Instance::null();
  }
  intptr_t instance_size = cls.host_instance_size();
  ASSERT(instance_size > 0);
  ObjectPtr raw = Object::Allocate(cls.id(), instance_size, space);
  return static_cast<InstancePtr>(raw);
}

InstancePtr Instance::NewFromCidAndSize(SharedClassTable* shared_class_table,
                                        classid_t cid,
                                        Heap::Space heap) {
  const intptr_t instance_size = shared_class_table->SizeAt(cid);
  ASSERT(instance_size > 0);
  ObjectPtr raw = Object::Allocate(cid, instance_size, heap);
  return static_cast<InstancePtr>(raw);
}

bool Instance::IsValidFieldOffset(intptr_t offset) const {
  Thread* thread = Thread::Current();
  REUSABLE_CLASS_HANDLESCOPE(thread);
  Class& cls = thread->ClassHandle();
  cls = clazz();
  return (offset >= 0 && offset <= (cls.host_instance_size() - kWordSize));
}

intptr_t Instance::ElementSizeFor(intptr_t cid) {
  if (IsExternalTypedDataClassId(cid) || IsTypedDataClassId(cid) ||
      IsTypedDataViewClassId(cid)) {
    return TypedDataBase::ElementSizeInBytes(cid);
  }
  switch (cid) {
    case kArrayCid:
    case kImmutableArrayCid:
      return Array::kBytesPerElement;
    case kOneByteStringCid:
      return OneByteString::kBytesPerElement;
    case kTwoByteStringCid:
      return TwoByteString::kBytesPerElement;
    case kExternalOneByteStringCid:
      return ExternalOneByteString::kBytesPerElement;
    case kExternalTwoByteStringCid:
      return ExternalTwoByteString::kBytesPerElement;
    default:
      UNIMPLEMENTED();
      return 0;
  }
}

intptr_t Instance::DataOffsetFor(intptr_t cid) {
  if (IsExternalTypedDataClassId(cid) || IsExternalStringClassId(cid)) {
    // Elements start at offset 0 of the external data.
    return 0;
  }
  if (IsTypedDataClassId(cid)) {
    return TypedData::data_offset();
  }
  switch (cid) {
    case kArrayCid:
    case kImmutableArrayCid:
      return Array::data_offset();
    case kOneByteStringCid:
      return OneByteString::data_offset();
    case kTwoByteStringCid:
      return TwoByteString::data_offset();
    default:
      UNIMPLEMENTED();
      return Array::data_offset();
  }
}

const char* Instance::ToCString() const {
  if (IsNull()) {
    return "null";
  } else if (raw() == Object::sentinel().raw()) {
    return "sentinel";
  } else if (raw() == Object::transition_sentinel().raw()) {
    return "transition_sentinel";
  } else if (raw() == Object::unknown_constant().raw()) {
    return "unknown_constant";
  } else if (raw() == Object::non_constant().raw()) {
    return "non_constant";
  } else if (Thread::Current()->no_safepoint_scope_depth() > 0) {
    // Can occur when running disassembler.
    return "Instance";
  } else {
    if (IsClosure()) {
      return Closure::Cast(*this).ToCString();
    }
    // Background compiler disassembly of instructions referring to pool objects
    // calls this function and requires allocation of Type in old space.
    const AbstractType& type = AbstractType::Handle(GetType(Heap::kOld));
    const String& type_name = String::Handle(type.UserVisibleName());
    return OS::SCreate(Thread::Current()->zone(), "Instance of '%s'",
                       type_name.ToCString());
  }
}
```

