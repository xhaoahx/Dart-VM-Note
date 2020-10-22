# Class

## object.h

```c++
class Class : public Object {
 public:
  enum InvocationDispatcherEntry {
    kInvocationDispatcherName,
    kInvocationDispatcherArgsDesc,
    kInvocationDispatcherFunction,
    kInvocationDispatcherEntrySize,
  };

  intptr_t host_instance_size() const {
    ASSERT(is_finalized() || is_prefinalized());
    return (raw_ptr()->host_instance_size_in_words_ * kWordSize);
  }
  intptr_t target_instance_size() const {
    ASSERT(is_finalized() || is_prefinalized());
#if !defined(DART_PRECOMPILED_RUNTIME)
    return (raw_ptr()->target_instance_size_in_words_ *
            compiler::target::kWordSize);
#else
    return host_instance_size();
#endif  // !defined(DART_PRECOMPILED_RUNTIME)
  }
  static intptr_t host_instance_size(ClassPtr clazz) {
    return (clazz->ptr()->host_instance_size_in_words_ * kWordSize);
  }
  static intptr_t target_instance_size(ClassPtr clazz) {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return (clazz->ptr()->target_instance_size_in_words_ *
            compiler::target::kWordSize);
#else
    return host_instance_size(clazz);
#endif  // !defined(DART_PRECOMPILED_RUNTIME)
  }
  void set_instance_size(intptr_t host_value_in_bytes,
                         intptr_t target_value_in_bytes) const {
    ASSERT(kWordSize != 0);
    set_instance_size_in_words(
        host_value_in_bytes / kWordSize,
        target_value_in_bytes / compiler::target::kWordSize);
  }
  void set_instance_size_in_words(intptr_t host_value,
                                  intptr_t target_value) const {
    ASSERT(Utils::IsAligned((host_value * kWordSize), kObjectAlignment));
    StoreNonPointer(&raw_ptr()->host_instance_size_in_words_, host_value);
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(Utils::IsAligned((target_value * compiler::target::kWordSize),
                            compiler::target::kObjectAlignment));
    StoreNonPointer(&raw_ptr()->target_instance_size_in_words_, target_value);
#else
    ASSERT(host_value == target_value);
#endif  // #!defined(DART_PRECOMPILED_RUNTIME)
  }

  intptr_t host_next_field_offset() const {
    return raw_ptr()->host_next_field_offset_in_words_ * kWordSize;
  }
  intptr_t target_next_field_offset() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return raw_ptr()->target_next_field_offset_in_words_ *
           compiler::target::kWordSize;
#else
    return host_next_field_offset();
#endif  // #!defined(DART_PRECOMPILED_RUNTIME)
  }
  void set_next_field_offset(intptr_t host_value_in_bytes,
                             intptr_t target_value_in_bytes) const {
    set_next_field_offset_in_words(
        host_value_in_bytes / kWordSize,
        target_value_in_bytes / compiler::target::kWordSize);
  }
  void set_next_field_offset_in_words(intptr_t host_value,
                                      intptr_t target_value) const {
    ASSERT((host_value == -1) ||
           (Utils::IsAligned((host_value * kWordSize), kObjectAlignment) &&
            (host_value == raw_ptr()->host_instance_size_in_words_)) ||
           (!Utils::IsAligned((host_value * kWordSize), kObjectAlignment) &&
            ((host_value + 1) == raw_ptr()->host_instance_size_in_words_)));
    StoreNonPointer(&raw_ptr()->host_next_field_offset_in_words_, host_value);
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT((target_value == -1) ||
           (Utils::IsAligned((target_value * compiler::target::kWordSize),
                             compiler::target::kObjectAlignment) &&
            (target_value == raw_ptr()->target_instance_size_in_words_)) ||
           (!Utils::IsAligned((target_value * compiler::target::kWordSize),
                              compiler::target::kObjectAlignment) &&
            ((target_value + 1) == raw_ptr()->target_instance_size_in_words_)));
    StoreNonPointer(&raw_ptr()->target_next_field_offset_in_words_,
                    target_value);
#else
    ASSERT(host_value == target_value);
#endif  // #!defined(DART_PRECOMPILED_RUNTIME)
  }

  static bool is_valid_id(intptr_t value) {
    return ObjectLayout::ClassIdTag::is_valid(value);
  }
  intptr_t id() const { return raw_ptr()->id_; }
  void set_id(intptr_t value) const {
    ASSERT(value >= 0 && value < std::numeric_limits<classid_t>::max());
    StoreNonPointer(&raw_ptr()->id_, value);
  }
  static intptr_t id_offset() { return OFFSET_OF(ClassLayout, id_); }
  static intptr_t num_type_arguments_offset() {
    return OFFSET_OF(ClassLayout, num_type_arguments_);
  }

  StringPtr Name() const;
  StringPtr ScrubbedName() const;
  const char* ScrubbedNameCString() const;
  StringPtr UserVisibleName() const;
  const char* UserVisibleNameCString() const;

  const char* NameCString(NameVisibility name_visibility) const;

  // The mixin for this class if one exists. Otherwise, returns a raw pointer
  // to this class.
  ClassPtr Mixin() const;

  // The NNBD mode of the library declaring this class.
  NNBDMode nnbd_mode() const;

  bool IsInFullSnapshot() const;

  virtual StringPtr DictionaryName() const { return Name(); }

  ScriptPtr script() const { return raw_ptr()->script_; }
  void set_script(const Script& value) const;

  TokenPosition token_pos() const { return raw_ptr()->token_pos_; }
  void set_token_pos(TokenPosition value) const;
  TokenPosition end_token_pos() const { return raw_ptr()->end_token_pos_; }
  void set_end_token_pos(TokenPosition value) const;

  int32_t SourceFingerprint() const;

  // This class represents a typedef if the signature function is not null.
  FunctionPtr signature_function() const {
    return raw_ptr()->signature_function_;
  }
  void set_signature_function(const Function& value) const;

  // Return the Type with type parameters declared by this class filled in with
  // dynamic and type parameters declared in superclasses filled in as declared
  // in superclass clauses.
  AbstractTypePtr RareType() const;

  // Return the Type whose arguments are the type parameters declared by this
  // class preceded by the type arguments declared for superclasses, etc.
  // e.g. given
  // class B<T, S>
  // class C<R> extends B<R, int>
  // C.DeclarationType() --> C [R, int, R]
  // The declaration type's nullability is either legacy or non-nullable when
  // the non-nullable experiment is enabled.
  TypePtr DeclarationType() const;

  static intptr_t declaration_type_offset() {
    return OFFSET_OF(ClassLayout, declaration_type_);
  }

  LibraryPtr library() const { return raw_ptr()->library_; }
  void set_library(const Library& value) const;

  // The type parameters (and their bounds) are specified as an array of
  // TypeParameter.
  TypeArgumentsPtr type_parameters() const {
    ASSERT(is_declaration_loaded());
    return raw_ptr()->type_parameters_;
  }
  void set_type_parameters(const TypeArguments& value) const;
  intptr_t NumTypeParameters(Thread* thread) const;
  intptr_t NumTypeParameters() const {
    return NumTypeParameters(Thread::Current());
  }

  // Return a TypeParameter if the type_name is a type parameter of this class.
  // Return null otherwise.
  TypeParameterPtr LookupTypeParameter(const String& type_name) const;

  // The type argument vector is flattened and includes the type arguments of
  // the super class.
  intptr_t NumTypeArguments() const;

  // Return true if this class declares type parameters.
  bool IsGeneric() const { return NumTypeParameters(Thread::Current()) > 0; }

  // If this class is parameterized, each instance has a type_arguments field.
  static const intptr_t kNoTypeArguments = -1;
  intptr_t host_type_arguments_field_offset() const {
    ASSERT(is_type_finalized() || is_prefinalized());
    if (raw_ptr()->host_type_arguments_field_offset_in_words_ ==
        kNoTypeArguments) {
      return kNoTypeArguments;
    }
    return raw_ptr()->host_type_arguments_field_offset_in_words_ * kWordSize;
  }
  intptr_t target_type_arguments_field_offset() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(is_type_finalized() || is_prefinalized());
    if (raw_ptr()->target_type_arguments_field_offset_in_words_ ==
        compiler::target::Class::kNoTypeArguments) {
      return compiler::target::Class::kNoTypeArguments;
    }
    return raw_ptr()->target_type_arguments_field_offset_in_words_ *
           compiler::target::kWordSize;
#else
    return host_type_arguments_field_offset();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }
  void set_type_arguments_field_offset(intptr_t host_value_in_bytes,
                                       intptr_t target_value_in_bytes) const {
    intptr_t host_value, target_value;
    if (host_value_in_bytes == kNoTypeArguments ||
        target_value_in_bytes == RTN::Class::kNoTypeArguments) {
      ASSERT(host_value_in_bytes == kNoTypeArguments &&
             target_value_in_bytes == RTN::Class::kNoTypeArguments);
      host_value = kNoTypeArguments;
      target_value = RTN::Class::kNoTypeArguments;
    } else {
      ASSERT(kWordSize != 0 && compiler::target::kWordSize);
      host_value = host_value_in_bytes / kWordSize;
      target_value = target_value_in_bytes / compiler::target::kWordSize;
    }
    set_type_arguments_field_offset_in_words(host_value, target_value);
  }
  void set_type_arguments_field_offset_in_words(intptr_t host_value,
                                                intptr_t target_value) const {
    StoreNonPointer(&raw_ptr()->host_type_arguments_field_offset_in_words_,
                    host_value);
#if !defined(DART_PRECOMPILED_RUNTIME)
    StoreNonPointer(&raw_ptr()->target_type_arguments_field_offset_in_words_,
                    target_value);
#else
    ASSERT(host_value == target_value);
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }
  static intptr_t host_type_arguments_field_offset_in_words_offset() {
    return OFFSET_OF(ClassLayout, host_type_arguments_field_offset_in_words_);
  }

  static intptr_t target_type_arguments_field_offset_in_words_offset() {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return OFFSET_OF(ClassLayout, target_type_arguments_field_offset_in_words_);
#else
    return host_type_arguments_field_offset_in_words_offset();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  // The super type of this class, Object type if not explicitly specified.
  AbstractTypePtr super_type() const {
    ASSERT(is_declaration_loaded());
    return raw_ptr()->super_type_;
  }
  void set_super_type(const AbstractType& value) const;
  static intptr_t super_type_offset() {
    return OFFSET_OF(ClassLayout, super_type_);
  }

  // Asserts that the class of the super type has been resolved.
  // |original_classes| only has an effect when reloading. If true and we
  // are reloading, it will prefer the original classes to the replacement
  // classes.
  ClassPtr SuperClass(bool original_classes = false) const;

  // Interfaces is an array of Types.
  ArrayPtr interfaces() const {
    ASSERT(is_declaration_loaded());
    return raw_ptr()->interfaces_;
  }
  void set_interfaces(const Array& value) const;

  // Returns the list of classes directly implementing this class.
  GrowableObjectArrayPtr direct_implementors() const {
    return raw_ptr()->direct_implementors_;
  }
  void AddDirectImplementor(const Class& subclass, bool is_mixin) const;
  void ClearDirectImplementors() const;

  // Returns the list of classes having this class as direct superclass.
  GrowableObjectArrayPtr direct_subclasses() const {
    return raw_ptr()->direct_subclasses_;
  }
  void AddDirectSubclass(const Class& subclass) const;
  void ClearDirectSubclasses() const;

  // Check if this class represents the class of null.
  bool IsNullClass() const { return id() == kNullCid; }

  // Check if this class represents the 'dynamic' class.
  bool IsDynamicClass() const { return id() == kDynamicCid; }

  // Check if this class represents the 'void' class.
  bool IsVoidClass() const { return id() == kVoidCid; }

  // Check if this class represents the 'Never' class.
  bool IsNeverClass() const { return id() == kNeverCid; }

  // Check if this class represents the 'Object' class.
  bool IsObjectClass() const { return id() == kInstanceCid; }

  // Check if this class represents the 'Function' class.
  bool IsDartFunctionClass() const;

  // Check if this class represents the 'Future' class.
  bool IsFutureClass() const;

  // Check if this class represents the 'FutureOr' class.
  bool IsFutureOrClass() const { return id() == kFutureOrCid; }

  // Check if this class represents the 'Closure' class.
  bool IsClosureClass() const { return id() == kClosureCid; }
  static bool IsClosureClass(ClassPtr cls) {
    NoSafepointScope no_safepoint;
    return cls->ptr()->id_ == kClosureCid;
  }

  // Check if this class represents a typedef class.
  bool IsTypedefClass() const { return signature_function() != Object::null(); }

  static bool IsInFullSnapshot(ClassPtr cls) {
    NoSafepointScope no_safepoint;
    return LibraryLayout::InFullSnapshotBit::decode(
        cls->ptr()->library_->ptr()->flags_);
  }

  // Returns true if the type specified by cls, type_arguments, and nullability
  // is a subtype of the other type.
  static bool IsSubtypeOf(const Class& cls,
                          const TypeArguments& type_arguments,
                          Nullability nullability,
                          const AbstractType& other,
                          Heap::Space space,
                          TrailPtr trail = nullptr);

  // Check if this is the top level class.
  bool IsTopLevel() const;

  bool IsPrivate() const;

  DART_WARN_UNUSED_RESULT
  ErrorPtr VerifyEntryPoint() const;

  // Returns an array of instance and static fields defined by this class.
  ArrayPtr fields() const { return raw_ptr()->fields_; }
  void SetFields(const Array& value) const;
  void AddField(const Field& field) const;
  void AddFields(const GrowableArray<const Field*>& fields) const;

  // If this is a dart:internal.ClassID class, then inject our own const
  // fields. Returns true if synthetic fields are injected and regular
  // field declarations should be ignored.
  bool InjectCIDFields() const;

  // Returns an array of all instance fields of this class and its superclasses
  // indexed by offset in words.
  // |original_classes| only has an effect when reloading. If true and we
  // are reloading, it will prefer the original classes to the replacement
  // classes.
  ArrayPtr OffsetToFieldMap(bool original_classes = false) const;

  // Returns true if non-static fields are defined.
  bool HasInstanceFields() const;

  // TODO(koda): Unite w/ hash table.
  ArrayPtr functions() const { return raw_ptr()->functions_; }
  void SetFunctions(const Array& value) const;
  void AddFunction(const Function& function) const;
  void RemoveFunction(const Function& function) const;
  FunctionPtr FunctionFromIndex(intptr_t idx) const;
  intptr_t FindImplicitClosureFunctionIndex(const Function& needle) const;
  FunctionPtr ImplicitClosureFunctionFromIndex(intptr_t idx) const;

  FunctionPtr LookupDynamicFunction(const String& name) const;
  FunctionPtr LookupDynamicFunctionAllowAbstract(const String& name) const;
  FunctionPtr LookupDynamicFunctionAllowPrivate(const String& name) const;
  FunctionPtr LookupStaticFunction(const String& name) const;
  FunctionPtr LookupStaticFunctionAllowPrivate(const String& name) const;
  FunctionPtr LookupConstructor(const String& name) const;
  FunctionPtr LookupConstructorAllowPrivate(const String& name) const;
  FunctionPtr LookupFactory(const String& name) const;
  FunctionPtr LookupFactoryAllowPrivate(const String& name) const;
  FunctionPtr LookupFunction(const String& name) const;
  FunctionPtr LookupFunctionAllowPrivate(const String& name) const;
  FunctionPtr LookupGetterFunction(const String& name) const;
  FunctionPtr LookupSetterFunction(const String& name) const;
  FieldPtr LookupInstanceField(const String& name) const;
  FieldPtr LookupStaticField(const String& name) const;
  FieldPtr LookupField(const String& name) const;
  FieldPtr LookupFieldAllowPrivate(const String& name,
                                   bool instance_only = false) const;
  FieldPtr LookupInstanceFieldAllowPrivate(const String& name) const;
  FieldPtr LookupStaticFieldAllowPrivate(const String& name) const;

  DoublePtr LookupCanonicalDouble(Zone* zone, double value) const;
  MintPtr LookupCanonicalMint(Zone* zone, int64_t value) const;

  // The methods above are more efficient than this generic one.
  InstancePtr LookupCanonicalInstance(Zone* zone, const Instance& value) const;

  InstancePtr InsertCanonicalConstant(Zone* zone,
                                      const Instance& constant) const;
  void InsertCanonicalDouble(Zone* zone, const Double& constant) const;
  void InsertCanonicalMint(Zone* zone, const Mint& constant) const;

  void RehashConstants(Zone* zone) const;

  bool RequireLegacyErasureOfConstants(Zone* zone) const;

  static intptr_t InstanceSize() {
    return RoundedAllocationSize(sizeof(ClassLayout));
  }

  bool is_implemented() const {
    return ImplementedBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_implemented() const;

  bool is_abstract() const {
    return AbstractBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_abstract() const;

  ClassLayout::ClassLoadingState class_loading_state() const {
    return ClassLoadingBits::decode(raw_ptr()->state_bits_);
  }

  bool is_declaration_loaded() const {
    return class_loading_state() >= ClassLayout::kDeclarationLoaded;
  }
  void set_is_declaration_loaded() const;

  bool is_type_finalized() const {
    return class_loading_state() >= ClassLayout::kTypeFinalized;
  }
  void set_is_type_finalized() const;

  bool is_synthesized_class() const {
    return SynthesizedClassBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_synthesized_class() const;

  bool is_enum_class() const { return EnumBit::decode(raw_ptr()->state_bits_); }
  void set_is_enum_class() const;

  bool is_finalized() const {
    return ClassFinalizedBits::decode(raw_ptr()->state_bits_) ==
               ClassLayout::kFinalized ||
           ClassFinalizedBits::decode(raw_ptr()->state_bits_) ==
               ClassLayout::kAllocateFinalized;
  }
  void set_is_finalized() const;

  bool is_allocate_finalized() const {
    return ClassFinalizedBits::decode(raw_ptr()->state_bits_) ==
           ClassLayout::kAllocateFinalized;
  }
  void set_is_allocate_finalized() const;

  bool is_prefinalized() const {
    return ClassFinalizedBits::decode(raw_ptr()->state_bits_) ==
           ClassLayout::kPreFinalized;
  }

  void set_is_prefinalized() const;

  bool is_const() const { return ConstBit::decode(raw_ptr()->state_bits_); }
  void set_is_const() const;

  // Tests if this is a mixin application class which was desugared
  // to a normal class by kernel mixin transformation
  // (pkg/kernel/lib/transformations/mixin_full_resolution.dart).
  //
  // In such case, its mixed-in type was pulled into the end of
  // interfaces list.
  bool is_transformed_mixin_application() const {
    return TransformedMixinApplicationBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_transformed_mixin_application() const;

  bool is_fields_marked_nullable() const {
    return FieldsMarkedNullableBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_fields_marked_nullable() const;

  bool is_allocated() const {
    return IsAllocatedBit::decode(raw_ptr()->state_bits_);
  }
  void set_is_allocated(bool value) const;

  bool is_loaded() const { return IsLoadedBit::decode(raw_ptr()->state_bits_); }
  void set_is_loaded(bool value) const;

  uint16_t num_native_fields() const { return raw_ptr()->num_native_fields_; }
  void set_num_native_fields(uint16_t value) const {
    StoreNonPointer(&raw_ptr()->num_native_fields_, value);
  }

  CodePtr allocation_stub() const { return raw_ptr()->allocation_stub_; }
  void set_allocation_stub(const Code& value) const;

#if !defined(DART_PRECOMPILED_RUNTIME)
  intptr_t binary_declaration_offset() const {
    return ClassLayout::BinaryDeclarationOffset::decode(
        raw_ptr()->binary_declaration_);
  }
  void set_binary_declaration_offset(intptr_t value) const {
    ASSERT(value >= 0);
    StoreNonPointer(&raw_ptr()->binary_declaration_,
                    ClassLayout::BinaryDeclarationOffset::update(
                        value, raw_ptr()->binary_declaration_));
  }
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

  intptr_t kernel_offset() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return 0;
#else
    ASSERT(!is_declared_in_bytecode());
    return binary_declaration_offset();
#endif
  }

  void set_kernel_offset(intptr_t value) const {
#if defined(DART_PRECOMPILED_RUNTIME)
    UNREACHABLE();
#else
    ASSERT(!is_declared_in_bytecode());
    set_binary_declaration_offset(value);
#endif
  }

  intptr_t bytecode_offset() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return 0;
#else
    ASSERT(is_declared_in_bytecode());
    return binary_declaration_offset();
#endif
  }

  void set_bytecode_offset(intptr_t value) const {
#if defined(DART_PRECOMPILED_RUNTIME)
    UNREACHABLE();
#else
    ASSERT(is_declared_in_bytecode());
    set_binary_declaration_offset(value);
#endif
  }

  bool is_declared_in_bytecode() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return false;
#else
    return ClassLayout::IsDeclaredInBytecode::decode(
        raw_ptr()->binary_declaration_);
#endif
  }

#if !defined(DART_PRECOMPILED_RUNTIME)
  void set_is_declared_in_bytecode(bool value) const {
    StoreNonPointer(&raw_ptr()->binary_declaration_,
                    ClassLayout::IsDeclaredInBytecode::update(
                        value, raw_ptr()->binary_declaration_));
  }
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

  void DisableAllocationStub() const;

  ArrayPtr constants() const;
  void set_constants(const Array& value) const;

  intptr_t FindInvocationDispatcherFunctionIndex(const Function& needle) const;
  FunctionPtr InvocationDispatcherFunctionFromIndex(intptr_t idx) const;

  FunctionPtr GetInvocationDispatcher(const String& target_name,
                                      const Array& args_desc,
                                      FunctionLayout::Kind kind,
                                      bool create_if_absent) const;

  void Finalize() const;

  ObjectPtr Invoke(const String& selector,
                   const Array& arguments,
                   const Array& argument_names,
                   bool respect_reflectable = true,
                   bool check_is_entrypoint = false) const;
  ObjectPtr InvokeGetter(const String& selector,
                         bool throw_nsm_if_absent,
                         bool respect_reflectable = true,
                         bool check_is_entrypoint = false) const;
  ObjectPtr InvokeSetter(const String& selector,
                         const Instance& argument,
                         bool respect_reflectable = true,
                         bool check_is_entrypoint = false) const;

  // Evaluate the given expression as if it appeared in a static method of this
  // class and return the resulting value, or an error object if evaluating the
  // expression fails. The method has the formal (type) parameters given in
  // (type_)param_names, and is invoked with the (type)argument values given in
  // (type_)param_values.
  ObjectPtr EvaluateCompiledExpression(
      const ExternalTypedData& kernel_buffer,
      const Array& type_definitions,
      const Array& param_values,
      const TypeArguments& type_param_values) const;

  // Load class declaration (super type, interfaces, type parameters and
  // number of type arguments) if it is not loaded yet.
  void EnsureDeclarationLoaded() const;

  ErrorPtr EnsureIsFinalized(Thread* thread) const;
  ErrorPtr EnsureIsAllocateFinalized(Thread* thread) const;

  // Allocate a class used for VM internal objects.
  template <class FakeObject, class TargetFakeObject>
  static ClassPtr New(Isolate* isolate, bool register_class = true);

  // Allocate instance classes.
  static ClassPtr New(const Library& lib,
                      const String& name,
                      const Script& script,
                      TokenPosition token_pos,
                      bool register_class = true);
  static ClassPtr NewNativeWrapper(const Library& library,
                                   const String& name,
                                   int num_fields);

  // Allocate the raw string classes.
  static ClassPtr NewStringClass(intptr_t class_id, Isolate* isolate);

  // Allocate the raw TypedData classes.
  static ClassPtr NewTypedDataClass(intptr_t class_id, Isolate* isolate);

  // Allocate the raw TypedDataView/ByteDataView classes.
  static ClassPtr NewTypedDataViewClass(intptr_t class_id, Isolate* isolate);

  // Allocate the raw ExternalTypedData classes.
  static ClassPtr NewExternalTypedDataClass(intptr_t class_id,
                                            Isolate* isolate);

  // Allocate the raw Pointer classes.
  static ClassPtr NewPointerClass(intptr_t class_id, Isolate* isolate);

  // Register code that has used CHA for optimization.
  // TODO(srdjan): Also register kind of CHA optimization (e.g.: leaf class,
  // leaf method, ...).
  void RegisterCHACode(const Code& code);

  void DisableCHAOptimizedCode(const Class& subclass);

  void DisableAllCHAOptimizedCode();

  void DisableCHAImplementorUsers() { DisableAllCHAOptimizedCode(); }

  // Return the list of code objects that were compiled using CHA of this class.
  // These code objects will be invalidated if new subclasses of this class
  // are finalized.
  ArrayPtr dependent_code() const { return raw_ptr()->dependent_code_; }
  void set_dependent_code(const Array& array) const;

  bool TraceAllocation(Isolate* isolate) const;
  void SetTraceAllocation(bool trace_allocation) const;

  void ReplaceEnum(IsolateReloadContext* reload_context,
                   const Class& old_enum) const;
  void CopyStaticFieldValues(IsolateReloadContext* reload_context,
                             const Class& old_cls) const;
  void PatchFieldsAndFunctions() const;
  void MigrateImplicitStaticClosures(IsolateReloadContext* context,
                                     const Class& new_cls) const;
  void CopyCanonicalConstants(const Class& old_cls) const;
  void CopyDeclarationType(const Class& old_cls) const;
  void CheckReload(const Class& replacement,
                   IsolateReloadContext* context) const;

  void AddInvocationDispatcher(const String& target_name,
                               const Array& args_desc,
                               const Function& dispatcher) const;

  static int32_t host_instance_size_in_words(const ClassPtr cls) {
    return cls->ptr()->host_instance_size_in_words_;
  }

  static int32_t target_instance_size_in_words(const ClassPtr cls) {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return cls->ptr()->target_instance_size_in_words_;
#else
    return host_instance_size_in_words(cls);
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  static int32_t host_next_field_offset_in_words(const ClassPtr cls) {
    return cls->ptr()->host_next_field_offset_in_words_;
  }

  static int32_t target_next_field_offset_in_words(const ClassPtr cls) {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return cls->ptr()->target_next_field_offset_in_words_;
#else
    return host_next_field_offset_in_words(cls);
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  static int32_t host_type_arguments_field_offset_in_words(const ClassPtr cls) {
    return cls->ptr()->host_type_arguments_field_offset_in_words_;
  }

  static int32_t target_type_arguments_field_offset_in_words(
      const ClassPtr cls) {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return cls->ptr()->target_type_arguments_field_offset_in_words_;
#else
    return host_type_arguments_field_offset_in_words(cls);
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

 private:
  TypePtr declaration_type() const { return raw_ptr()->declaration_type_; }

  // Caches the declaration type of this class.
  void set_declaration_type(const Type& type) const;

  bool CanReloadFinalized(const Class& replacement,
                          IsolateReloadContext* context) const;
  bool CanReloadPreFinalized(const Class& replacement,
                             IsolateReloadContext* context) const;

  // Tells whether instances need morphing for reload.
  bool RequiresInstanceMorphing(const Class& replacement) const;

  template <class FakeInstance, class TargetFakeInstance>
  static ClassPtr NewCommon(intptr_t index);

  enum MemberKind {
    kAny = 0,
    kStatic,
    kInstance,
    kInstanceAllowAbstract,
    kConstructor,
    kFactory,
  };
  enum StateBits {
    kConstBit = 0,
    kImplementedBit = 1,
    kClassFinalizedPos = 2,
    kClassFinalizedSize = 2,
    kClassLoadingPos = kClassFinalizedPos + kClassFinalizedSize,  // = 4
    kClassLoadingSize = 2,
    kAbstractBit = kClassLoadingPos + kClassLoadingSize,  // = 6
    kSynthesizedClassBit,
    kMixinAppAliasBit,
    kMixinTypeAppliedBit,
    kFieldsMarkedNullableBit,
    kEnumBit,
    kTransformedMixinApplicationBit,
    kIsAllocatedBit,
    kIsLoadedBit,
    kHasPragmaBit,
  };
  class ConstBit : public BitField<uint32_t, bool, kConstBit, 1> {};
  class ImplementedBit : public BitField<uint32_t, bool, kImplementedBit, 1> {};
  class ClassFinalizedBits : public BitField<uint32_t,
                                             ClassLayout::ClassFinalizedState,
                                             kClassFinalizedPos,
                                             kClassFinalizedSize> {};
  class ClassLoadingBits : public BitField<uint32_t,
                                           ClassLayout::ClassLoadingState,
                                           kClassLoadingPos,
                                           kClassLoadingSize> {};
  class AbstractBit : public BitField<uint32_t, bool, kAbstractBit, 1> {};
  class SynthesizedClassBit
      : public BitField<uint32_t, bool, kSynthesizedClassBit, 1> {};
  class FieldsMarkedNullableBit
      : public BitField<uint32_t, bool, kFieldsMarkedNullableBit, 1> {};
  class EnumBit : public BitField<uint32_t, bool, kEnumBit, 1> {};
  class TransformedMixinApplicationBit
      : public BitField<uint32_t, bool, kTransformedMixinApplicationBit, 1> {};
  class IsAllocatedBit : public BitField<uint32_t, bool, kIsAllocatedBit, 1> {};
  class IsLoadedBit : public BitField<uint32_t, bool, kIsLoadedBit, 1> {};
  class HasPragmaBit : public BitField<uint32_t, bool, kHasPragmaBit, 1> {};

  void set_name(const String& value) const;
  void set_user_name(const String& value) const;
  const char* GenerateUserVisibleName() const;
  void set_state_bits(intptr_t bits) const;

  ArrayPtr invocation_dispatcher_cache() const;
  void set_invocation_dispatcher_cache(const Array& cache) const;
  FunctionPtr CreateInvocationDispatcher(const String& target_name,
                                         const Array& args_desc,
                                         FunctionLayout::Kind kind) const;

  // Returns the bitmap of unboxed fields
  UnboxedFieldBitmap CalculateFieldOffsets() const;

  // functions_hash_table is in use iff there are at least this many functions.
  static const intptr_t kFunctionLookupHashTreshold = 16;

  // Initial value for the cached number of type arguments.
  static const intptr_t kUnknownNumTypeArguments = -1;

  int16_t num_type_arguments() const { return raw_ptr()->num_type_arguments_; }

 public:
  void set_num_type_arguments(intptr_t value) const;

  bool has_pragma() const {
    return HasPragmaBit::decode(raw_ptr()->state_bits_);
  }
  void set_has_pragma(bool has_pragma) const;

 private:
  // Calculates number of type arguments of this class.
  // This includes type arguments of a superclass and takes overlapping
  // of type arguments into account.
  intptr_t ComputeNumTypeArguments() const;

  // Assigns empty array to all raw class array fields.
  void InitEmptyFields();

  static FunctionPtr CheckFunctionType(const Function& func, MemberKind kind);
  FunctionPtr LookupFunction(const String& name, MemberKind kind) const;
  FunctionPtr LookupFunctionAllowPrivate(const String& name,
                                         MemberKind kind) const;
  FieldPtr LookupField(const String& name, MemberKind kind) const;

  FunctionPtr LookupAccessorFunction(const char* prefix,
                                     intptr_t prefix_length,
                                     const String& name) const;

  // Allocate an instance class which has a VM implementation.
  template <class FakeInstance, class TargetFakeInstance>
  static ClassPtr New(intptr_t id,
                      Isolate* isolate,
                      bool register_class = true,
                      bool is_abstract = false);

  // Helper that calls 'Class::New<Instance>(kIllegalCid)'.
  static ClassPtr NewInstanceClass();

  FINAL_HEAP_OBJECT_IMPLEMENTATION(Class, Object);
  friend class AbstractType;
  friend class Instance;
  friend class Object;
  friend class Type;
  friend class InterpreterHelpers;
  friend class Intrinsifier;
  friend class ProgramWalker;
  friend class Precompiler;
};
```

## object.cc

```c++
StringPtr Class::Name() const {
  return raw_ptr()->name_;
}

StringPtr Class::ScrubbedName() const {
  return Symbols::New(Thread::Current(), ScrubbedNameCString());
}

const char* Class::ScrubbedNameCString() const {
  return String::ScrubName(String::Handle(Name()));
}

StringPtr Class::UserVisibleName() const {
#if !defined(PRODUCT)
  ASSERT(raw_ptr()->user_name_ != String::null());
  return raw_ptr()->user_name_;
#endif  // !defined(PRODUCT)
  // No caching in PRODUCT, regenerate.
  return Symbols::New(Thread::Current(), GenerateUserVisibleName());
}

const char* Class::UserVisibleNameCString() const {
#if !defined(PRODUCT)
  ASSERT(raw_ptr()->user_name_ != String::null());
  return String::Handle(raw_ptr()->user_name_).ToCString();
#endif                               // !defined(PRODUCT)
  return GenerateUserVisibleName();  // No caching in PRODUCT, regenerate.
}

const char* Class::NameCString(NameVisibility name_visibility) const {
  switch (name_visibility) {
    case Object::kInternalName:
      return String::Handle(Name()).ToCString();
    case Object::kScrubbedName:
      return ScrubbedNameCString();
    case Object::kUserVisibleName:
      return UserVisibleNameCString();
    default:
      UNREACHABLE();
      return nullptr;
  }
}

ClassPtr Class::Mixin() const {
  if (is_transformed_mixin_application()) {
    const Array& interfaces = Array::Handle(this->interfaces());
    const Type& mixin_type =
        Type::Handle(Type::RawCast(interfaces.At(interfaces.Length() - 1)));
    return mixin_type.type_class();
  }
  return raw();
}

NNBDMode Class::nnbd_mode() const {
  return Library::Handle(library()).nnbd_mode();
}

bool Class::IsInFullSnapshot() const {
  NoSafepointScope no_safepoint;
  return LibraryLayout::InFullSnapshotBit::decode(
      raw_ptr()->library_->ptr()->flags_);
}

AbstractTypePtr Class::RareType() const {
  if (!IsGeneric() && !IsClosureClass() && !IsTypedefClass()) {
    return DeclarationType();
  }
  ASSERT(is_declaration_loaded());
  const Type& type = Type::Handle(
      Type::New(*this, Object::null_type_arguments(), TokenPosition::kNoSource,
                Nullability::kNonNullable));
  return ClassFinalizer::FinalizeType(*this, type);
}

template <class FakeObject, class TargetFakeObject>
ClassPtr Class::New(Isolate* isolate, bool register_class) {
  ASSERT(Object::class_class() != Class::null());
  Class& result = Class::Handle();
  {
    ObjectPtr raw =
        Object::Allocate(Class::kClassId, Class::InstanceSize(), Heap::kOld);
    NoSafepointScope no_safepoint;
    result ^= raw;
  }
  Object::VerifyBuiltinVtable<FakeObject>(FakeObject::kClassId);
  result.set_token_pos(TokenPosition::kNoSource);
  result.set_end_token_pos(TokenPosition::kNoSource);
  result.set_instance_size(FakeObject::InstanceSize(),
                           compiler::target::RoundedAllocationSize(
                               TargetFakeObject::InstanceSize()));
  result.set_type_arguments_field_offset_in_words(kNoTypeArguments,
                                                  RTN::Class::kNoTypeArguments);
  const intptr_t host_next_field_offset = FakeObject::NextFieldOffset();
  const intptr_t target_next_field_offset = TargetFakeObject::NextFieldOffset();
  result.set_next_field_offset(host_next_field_offset,
                               target_next_field_offset);
  COMPILE_ASSERT((FakeObject::kClassId != kInstanceCid));
  result.set_id(FakeObject::kClassId);
  result.set_num_type_arguments(0);
  result.set_num_native_fields(0);
  result.set_state_bits(0);
  if ((FakeObject::kClassId < kInstanceCid) ||
      (FakeObject::kClassId == kTypeArgumentsCid)) {
    // VM internal classes are done. There is no finalization needed or
    // possible in this case.
    result.set_is_declaration_loaded();
    result.set_is_type_finalized();
    result.set_is_allocate_finalized();
  } else if (FakeObject::kClassId != kClosureCid) {
    // VM backed classes are almost ready: run checks and resolve class
    // references, but do not recompute size.
    result.set_is_prefinalized();
  }
  NOT_IN_PRECOMPILED(result.set_is_declared_in_bytecode(false));
  NOT_IN_PRECOMPILED(result.set_binary_declaration_offset(0));
  result.InitEmptyFields();
  if (register_class) {
    isolate->class_table()->Register(result);
  }
  return result.raw();
}

static void ReportTooManyTypeArguments(const Class& cls) {
  Report::MessageF(Report::kError, Script::Handle(cls.script()),
                   cls.token_pos(), Report::AtLocation,
                   "too many type parameters declared in class '%s' or in its "
                   "super classes",
                   String::Handle(cls.Name()).ToCString());
  UNREACHABLE();
}

void Class::set_num_type_arguments(intptr_t value) const {
  if (!Utils::IsInt(16, value)) {
    ReportTooManyTypeArguments(*this);
  }
  StoreNonPointer(&raw_ptr()->num_type_arguments_, value);
}

void Class::set_has_pragma(bool value) const {
  set_state_bits(HasPragmaBit::update(value, raw_ptr()->state_bits_));
}

// Initialize class fields of type Array with empty array.
void Class::InitEmptyFields() {
  if (Object::empty_array().raw() == Array::null()) {
    // The empty array has not been initialized yet.
    return;
  }
  StorePointer(&raw_ptr()->interfaces_, Object::empty_array().raw());
  StorePointer(&raw_ptr()->constants_, Object::null_array().raw());
  StorePointer(&raw_ptr()->functions_, Object::empty_array().raw());
  StorePointer(&raw_ptr()->fields_, Object::empty_array().raw());
  StorePointer(&raw_ptr()->invocation_dispatcher_cache_,
               Object::empty_array().raw());
}

ArrayPtr Class::OffsetToFieldMap(bool original_classes) const {
  if (raw_ptr()->offset_in_words_to_field_ == Array::null()) {
    ASSERT(is_finalized());
    const intptr_t length = raw_ptr()->host_instance_size_in_words_;
    const Array& array = Array::Handle(Array::New(length, Heap::kOld));
    Class& cls = Class::Handle(this->raw());
    Array& fields = Array::Handle();
    Field& f = Field::Handle();
    while (!cls.IsNull()) {
      fields = cls.fields();
      for (intptr_t i = 0; i < fields.Length(); ++i) {
        f ^= fields.At(i);
        if (f.is_instance()) {
          array.SetAt(f.HostOffset() >> kWordSizeLog2, f);
        }
      }
      cls = cls.SuperClass(original_classes);
    }
    StorePointer(&raw_ptr()->offset_in_words_to_field_, array.raw());
  }
  return raw_ptr()->offset_in_words_to_field_;
}

bool Class::HasInstanceFields() const {
  const Array& field_array = Array::Handle(fields());
  Field& field = Field::Handle();
  for (intptr_t i = 0; i < field_array.Length(); ++i) {
    field ^= field_array.At(i);
    if (!field.is_static()) {
      return true;
    }
  }
  return false;
}

class FunctionName {
 public:
  FunctionName(const String& name, String* tmp_string)
      : name_(name), tmp_string_(tmp_string) {}
  bool Matches(const Function& function) const {
    if (name_.IsSymbol()) {
      return name_.raw() == function.name();
    } else {
      *tmp_string_ = function.name();
      return name_.Equals(*tmp_string_);
    }
  }
  intptr_t Hash() const { return name_.Hash(); }

 private:
  const String& name_;
  String* tmp_string_;
};

// Traits for looking up Functions by name.
class ClassFunctionsTraits {
 public:
  static const char* Name() { return "ClassFunctionsTraits"; }
  static bool ReportStats() { return false; }

  // Called when growing the table.
  static bool IsMatch(const Object& a, const Object& b) {
    ASSERT(a.IsFunction() && b.IsFunction());
    // Function objects are always canonical.
    return a.raw() == b.raw();
  }
  static bool IsMatch(const FunctionName& name, const Object& obj) {
    return name.Matches(Function::Cast(obj));
  }
  static uword Hash(const Object& key) {
    return String::HashRawSymbol(Function::Cast(key).name());
  }
  static uword Hash(const FunctionName& name) { return name.Hash(); }
};
typedef UnorderedHashSet<ClassFunctionsTraits> ClassFunctionsSet;

void Class::SetFunctions(const Array& value) const {
  ASSERT(Thread::Current()->IsMutatorThread());
  ASSERT(!value.IsNull());
  StorePointer(&raw_ptr()->functions_, value.raw());
  const intptr_t len = value.Length();
  if (len >= kFunctionLookupHashTreshold) {
    ClassFunctionsSet set(HashTables::New<ClassFunctionsSet>(len, Heap::kOld));
    Function& func = Function::Handle();
    for (intptr_t i = 0; i < len; ++i) {
      func ^= value.At(i);
      // Verify that all the functions in the array have this class as owner.
      ASSERT(func.Owner() == raw());
      set.Insert(func);
    }
    StorePointer(&raw_ptr()->functions_hash_table_, set.Release().raw());
  } else {
    StorePointer(&raw_ptr()->functions_hash_table_, Array::null());
  }
}

void Class::AddFunction(const Function& function) const {
  ASSERT(Thread::Current()->IsMutatorThread());
  const Array& arr = Array::Handle(functions());
  const Array& new_arr =
      Array::Handle(Array::Grow(arr, arr.Length() + 1, Heap::kOld));
  new_arr.SetAt(arr.Length(), function);
  StorePointer(&raw_ptr()->functions_, new_arr.raw());
  // Add to hash table, if any.
  const intptr_t new_len = new_arr.Length();
  if (new_len == kFunctionLookupHashTreshold) {
    // Transition to using hash table.
    SetFunctions(new_arr);
  } else if (new_len > kFunctionLookupHashTreshold) {
    ClassFunctionsSet set(raw_ptr()->functions_hash_table_);
    set.Insert(function);
    StorePointer(&raw_ptr()->functions_hash_table_, set.Release().raw());
  }
}

void Class::RemoveFunction(const Function& function) const {
  ASSERT(Thread::Current()->IsMutatorThread());
  const Array& arr = Array::Handle(functions());
  StorePointer(&raw_ptr()->functions_, Object::empty_array().raw());
  StorePointer(&raw_ptr()->functions_hash_table_, Array::null());
  Function& entry = Function::Handle();
  for (intptr_t i = 0; i < arr.Length(); i++) {
    entry ^= arr.At(i);
    if (function.raw() != entry.raw()) {
      AddFunction(entry);
    }
  }
}

FunctionPtr Class::FunctionFromIndex(intptr_t idx) const {
  const Array& funcs = Array::Handle(functions());
  if ((idx < 0) || (idx >= funcs.Length())) {
    return Function::null();
  }
  Function& func = Function::Handle();
  func ^= funcs.At(idx);
  ASSERT(!func.IsNull());
  return func.raw();
}

FunctionPtr Class::ImplicitClosureFunctionFromIndex(intptr_t idx) const {
  const Array& funcs = Array::Handle(functions());
  if ((idx < 0) || (idx >= funcs.Length())) {
    return Function::null();
  }
  Function& func = Function::Handle();
  func ^= funcs.At(idx);
  ASSERT(!func.IsNull());
  if (!func.HasImplicitClosureFunction()) {
    return Function::null();
  }
  const Function& closure_func =
      Function::Handle(func.ImplicitClosureFunction());
  ASSERT(!closure_func.IsNull());
  return closure_func.raw();
}

intptr_t Class::FindImplicitClosureFunctionIndex(const Function& needle) const {
  Thread* thread = Thread::Current();
  if (EnsureIsFinalized(thread) != Error::null()) {
    return -1;
  }
  REUSABLE_ARRAY_HANDLESCOPE(thread);
  REUSABLE_FUNCTION_HANDLESCOPE(thread);
  Array& funcs = thread->ArrayHandle();
  Function& function = thread->FunctionHandle();
  funcs = functions();
  ASSERT(!funcs.IsNull());
  Function& implicit_closure = Function::Handle(thread->zone());
  const intptr_t len = funcs.Length();
  for (intptr_t i = 0; i < len; i++) {
    function ^= funcs.At(i);
    implicit_closure = function.implicit_closure_function();
    if (implicit_closure.IsNull()) {
      // Skip non-implicit closure functions.
      continue;
    }
    if (needle.raw() == implicit_closure.raw()) {
      return i;
    }
  }
  // No function found.
  return -1;
}

intptr_t Class::FindInvocationDispatcherFunctionIndex(
    const Function& needle) const {
  Thread* thread = Thread::Current();
  if (EnsureIsFinalized(thread) != Error::null()) {
    return -1;
  }
  REUSABLE_ARRAY_HANDLESCOPE(thread);
  REUSABLE_OBJECT_HANDLESCOPE(thread);
  Array& funcs = thread->ArrayHandle();
  Object& object = thread->ObjectHandle();
  funcs = invocation_dispatcher_cache();
  ASSERT(!funcs.IsNull());
  const intptr_t len = funcs.Length();
  for (intptr_t i = 0; i < len; i++) {
    object = funcs.At(i);
    // The invocation_dispatcher_cache is a table with some entries that
    // are functions.
    if (object.IsFunction()) {
      if (Function::Cast(object).raw() == needle.raw()) {
        return i;
      }
    }
  }
  // No function found.
  return -1;
}

FunctionPtr Class::InvocationDispatcherFunctionFromIndex(intptr_t idx) const {
  Thread* thread = Thread::Current();
  REUSABLE_ARRAY_HANDLESCOPE(thread);
  REUSABLE_OBJECT_HANDLESCOPE(thread);
  Array& dispatcher_cache = thread->ArrayHandle();
  Object& object = thread->ObjectHandle();
  dispatcher_cache = invocation_dispatcher_cache();
  object = dispatcher_cache.At(idx);
  if (!object.IsFunction()) {
    return Function::null();
  }
  return Function::Cast(object).raw();
}

void Class::set_signature_function(const Function& value) const {
  ASSERT(value.IsClosureFunction() || value.IsSignatureFunction());
  StorePointer(&raw_ptr()->signature_function_, value.raw());
}

void Class::set_state_bits(intptr_t bits) const {
  StoreNonPointer(&raw_ptr()->state_bits_, static_cast<uint32_t>(bits));
}

void Class::set_library(const Library& value) const {
  StorePointer(&raw_ptr()->library_, value.raw());
}

void Class::set_type_parameters(const TypeArguments& value) const {
  ASSERT((num_type_arguments() == kUnknownNumTypeArguments) ||
         is_declared_in_bytecode() || is_prefinalized());
  StorePointer(&raw_ptr()->type_parameters_, value.raw());
}

intptr_t Class::NumTypeParameters(Thread* thread) const {
  if (!is_declaration_loaded()) {
    ASSERT(is_prefinalized());
    const intptr_t cid = id();
    if ((cid == kArrayCid) || (cid == kImmutableArrayCid) ||
        (cid == kGrowableObjectArrayCid)) {
      return 1;  // List's type parameter may not have been parsed yet.
    }
    return 0;
  }
  if (type_parameters() == TypeArguments::null()) {
    return 0;
  }
  REUSABLE_TYPE_ARGUMENTS_HANDLESCOPE(thread);
  TypeArguments& type_params = thread->TypeArgumentsHandle();
  type_params = type_parameters();
  return type_params.Length();
}

intptr_t Class::ComputeNumTypeArguments() const {
  ASSERT(is_declaration_loaded());
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  Isolate* isolate = thread->isolate();
  const intptr_t num_type_params = NumTypeParameters();

  if ((super_type() == AbstractType::null()) ||
      (super_type() == isolate->object_store()->object_type())) {
    return num_type_params;
  }

  const auto& sup_type = AbstractType::Handle(zone, super_type());
  ASSERT(sup_type.IsType());

  const auto& sup_class = Class::Handle(zone, sup_type.type_class());
  ASSERT(!sup_class.IsTypedefClass());

  const intptr_t sup_class_num_type_args = sup_class.NumTypeArguments();
  if (num_type_params == 0) {
    return sup_class_num_type_args;
  }

  const auto& sup_type_args = TypeArguments::Handle(zone, sup_type.arguments());
  if (sup_type_args.IsNull()) {
    // The super type is raw or the super class is non generic.
    // In either case, overlapping is not possible.
    return sup_class_num_type_args + num_type_params;
  }

  const intptr_t sup_type_args_length = sup_type_args.Length();
  // At this point, the super type may or may not be finalized. In either case,
  // the result of this function must remain the same.
  // The value of num_sup_type_args may increase when the super type is
  // finalized, but the last [sup_type_args_length] type arguments will not be
  // modified by finalization, only shifted to higher indices in the vector.
  // The super type may not even be resolved yet. This is not necessary, since
  // we only check for matching type parameters, which are resolved by default.
  const auto& type_params = TypeArguments::Handle(zone, type_parameters());
  // Determine the maximum overlap of a prefix of the vector consisting of the
  // type parameters of this class with a suffix of the vector consisting of the
  // type arguments of the super type of this class.
  // The number of own type arguments of this class is the number of its type
  // parameters minus the number of type arguments in the overlap.
  // Attempt to overlap the whole vector of type parameters; reduce the size
  // of the vector (keeping the first type parameter) until it fits or until
  // its size is zero.
  auto& type_param = TypeParameter::Handle(zone);
  auto& sup_type_arg = AbstractType::Handle(zone);
  for (intptr_t num_overlapping_type_args =
           (num_type_params < sup_type_args_length) ? num_type_params
                                                    : sup_type_args_length;
       num_overlapping_type_args > 0; num_overlapping_type_args--) {
    intptr_t i = 0;
    for (; i < num_overlapping_type_args; i++) {
      type_param ^= type_params.TypeAt(i);
      sup_type_arg = sup_type_args.TypeAt(sup_type_args_length -
                                          num_overlapping_type_args + i);
      if (!type_param.Equals(sup_type_arg)) break;
    }
    if (i == num_overlapping_type_args) {
      // Overlap found.
      return sup_class_num_type_args + num_type_params -
             num_overlapping_type_args;
    }
  }
  // No overlap found.
  return sup_class_num_type_args + num_type_params;
}

intptr_t Class::NumTypeArguments() const {
  // Return cached value if already calculated.
  intptr_t num_type_args = num_type_arguments();
  if (num_type_args != kUnknownNumTypeArguments) {
    return num_type_args;
  }

  num_type_args = ComputeNumTypeArguments();
  ASSERT(num_type_args != kUnknownNumTypeArguments);
  set_num_type_arguments(num_type_args);
  return num_type_args;
}

ClassPtr Class::SuperClass(bool original_classes) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  Isolate* isolate = thread->isolate();
  if (super_type() == AbstractType::null()) {
    if (id() == kTypeArgumentsCid) {
      // Pretend TypeArguments objects are Dart instances.
      return isolate->class_table()->At(kInstanceCid);
    }
    return Class::null();
  }
  const AbstractType& sup_type = AbstractType::Handle(zone, super_type());
  const intptr_t type_class_id = sup_type.type_class_id();
  if (original_classes) {
    return isolate->GetClassForHeapWalkAt(type_class_id);
  } else {
    return isolate->class_table()->At(type_class_id);
  }
}

void Class::set_super_type(const AbstractType& value) const {
  ASSERT(value.IsNull() || (value.IsType() && !value.IsDynamicType()));
  StorePointer(&raw_ptr()->super_type_, value.raw());
}

TypeParameterPtr Class::LookupTypeParameter(const String& type_name) const {
  ASSERT(!type_name.IsNull());
  Thread* thread = Thread::Current();
  REUSABLE_TYPE_ARGUMENTS_HANDLESCOPE(thread);
  REUSABLE_TYPE_PARAMETER_HANDLESCOPE(thread);
  REUSABLE_STRING_HANDLESCOPE(thread);
  TypeArguments& type_params = thread->TypeArgumentsHandle();
  TypeParameter& type_param = thread->TypeParameterHandle();
  String& type_param_name = thread->StringHandle();

  type_params = type_parameters();
  if (!type_params.IsNull()) {
    const intptr_t num_type_params = type_params.Length();
    for (intptr_t i = 0; i < num_type_params; i++) {
      type_param ^= type_params.TypeAt(i);
      type_param_name = type_param.name();
      if (type_param_name.Equals(type_name)) {
        return type_param.raw();
      }
    }
  }
  return TypeParameter::null();
}

UnboxedFieldBitmap Class::CalculateFieldOffsets() const {
  Array& flds = Array::Handle(fields());
  const Class& super = Class::Handle(SuperClass());
  intptr_t host_offset = 0;
  UnboxedFieldBitmap host_bitmap{};
  // Target offsets might differ if the word size are different
  intptr_t target_offset = 0;
  intptr_t host_type_args_field_offset = kNoTypeArguments;
  intptr_t target_type_args_field_offset = RTN::Class::kNoTypeArguments;
  if (super.IsNull()) {
    host_offset = Instance::NextFieldOffset();
    target_offset = RTN::Instance::NextFieldOffset();
    ASSERT(host_offset > 0);
    ASSERT(target_offset > 0);
  } else {
    ASSERT(super.is_finalized() || super.is_prefinalized());
    host_type_args_field_offset = super.host_type_arguments_field_offset();
    target_type_args_field_offset = super.target_type_arguments_field_offset();
    host_offset = super.host_next_field_offset();
    ASSERT(host_offset > 0);
    target_offset = super.target_next_field_offset();
    ASSERT(target_offset > 0);
    // We should never call CalculateFieldOffsets for native wrapper
    // classes, assert this.
    ASSERT(num_native_fields() == 0);
    set_num_native_fields(super.num_native_fields());

    if (FLAG_precompiled_mode) {
      host_bitmap = Isolate::Current()
                        ->group()
                        ->shared_class_table()
                        ->GetUnboxedFieldsMapAt(super.id());
    }
  }
  // If the super class is parameterized, use the same type_arguments field,
  // otherwise, if this class is the first in the super chain to be
  // parameterized, introduce a new type_arguments field.
  if (host_type_args_field_offset == kNoTypeArguments) {
    ASSERT(target_type_args_field_offset == RTN::Class::kNoTypeArguments);
    const TypeArguments& type_params = TypeArguments::Handle(type_parameters());
    if (!type_params.IsNull()) {
      ASSERT(type_params.Length() > 0);
      // The instance needs a type_arguments field.
      host_type_args_field_offset = host_offset;
      target_type_args_field_offset = target_offset;
      host_offset += kWordSize;
      target_offset += compiler::target::kWordSize;
    }
  } else {
    ASSERT(target_type_args_field_offset != RTN::Class::kNoTypeArguments);
  }

  set_type_arguments_field_offset(host_type_args_field_offset,
                                  target_type_args_field_offset);
  ASSERT(host_offset > 0);
  ASSERT(target_offset > 0);
  Field& field = Field::Handle();
  const intptr_t len = flds.Length();
  for (intptr_t i = 0; i < len; i++) {
    field ^= flds.At(i);
    // Offset is computed only for instance fields.
    if (!field.is_static()) {
      ASSERT(field.HostOffset() == 0);
      ASSERT(field.TargetOffset() == 0);
      field.SetOffset(host_offset, target_offset);

      if (FLAG_precompiled_mode && field.is_unboxing_candidate()) {
        intptr_t field_size;
        switch (field.guarded_cid()) {
          case kDoubleCid:
            field_size = sizeof(DoubleLayout::value_);
            break;
          case kFloat32x4Cid:
            field_size = sizeof(Float32x4Layout::value_);
            break;
          case kFloat64x2Cid:
            field_size = sizeof(Float64x2Layout::value_);
            break;
          default:
            if (field.is_non_nullable_integer()) {
              field_size = sizeof(MintLayout::value_);
            } else {
              UNREACHABLE();
              field_size = 0;
            }
            break;
        }

        const intptr_t host_num_words = field_size / kWordSize;
        const intptr_t host_next_offset = host_offset + field_size;
        const intptr_t host_next_position = host_next_offset / kWordSize;

        const intptr_t target_next_offset = target_offset + field_size;
        const intptr_t target_next_position =
            target_next_offset / compiler::target::kWordSize;

        // The bitmap has fixed length. Checks if the offset position is smaller
        // than its length. If it is not, than the field should be boxed
        if (host_next_position <= UnboxedFieldBitmap::Length() &&
            target_next_position <= UnboxedFieldBitmap::Length()) {
          for (intptr_t j = 0; j < host_num_words; j++) {
            // Activate the respective bit in the bitmap, indicating that the
            // content is not a pointer
            host_bitmap.Set(host_offset / kWordSize);
            host_offset += kWordSize;
          }

          ASSERT(host_offset == host_next_offset);
          target_offset = target_next_offset;
        } else {
          // Make the field boxed
          field.set_is_unboxing_candidate(false);
          host_offset += kWordSize;
          target_offset += compiler::target::kWordSize;
        }
      } else {
        host_offset += kWordSize;
        target_offset += compiler::target::kWordSize;
      }
    }
  }
  set_instance_size(RoundedAllocationSize(host_offset),
                    compiler::target::RoundedAllocationSize(target_offset));
  set_next_field_offset(host_offset, target_offset);

  return host_bitmap;
}

void Class::AddInvocationDispatcher(const String& target_name,
                                    const Array& args_desc,
                                    const Function& dispatcher) const {
  auto& cache = Array::Handle(invocation_dispatcher_cache());
  InvocationDispatcherTable dispatchers(cache);
  intptr_t i = 0;
  for (auto dispatcher : dispatchers) {
    if (dispatcher.Get<kInvocationDispatcherName>() == String::null()) {
      break;
    }
    i++;
  }
  if (i == dispatchers.Length()) {
    const intptr_t new_len =
        cache.Length() == 0
            ? static_cast<intptr_t>(Class::kInvocationDispatcherEntrySize)
            : cache.Length() * 2;
    cache = Array::Grow(cache, new_len);
    set_invocation_dispatcher_cache(cache);
  }
  auto entry = dispatchers[i];
  entry.Set<Class::kInvocationDispatcherName>(target_name);
  entry.Set<Class::kInvocationDispatcherArgsDesc>(args_desc);
  entry.Set<Class::kInvocationDispatcherFunction>(dispatcher);
}

FunctionPtr Class::GetInvocationDispatcher(const String& target_name,
                                           const Array& args_desc,
                                           FunctionLayout::Kind kind,
                                           bool create_if_absent) const {
  ASSERT(kind == FunctionLayout::kNoSuchMethodDispatcher ||
         kind == FunctionLayout::kInvokeFieldDispatcher ||
         kind == FunctionLayout::kDynamicInvocationForwarder);
  auto Z = Thread::Current()->zone();
  auto& function = Function::Handle(Z);
  auto& name = String::Handle(Z);
  auto& desc = Array::Handle(Z);
  auto& cache = Array::Handle(Z, invocation_dispatcher_cache());
  ASSERT(!cache.IsNull());

  InvocationDispatcherTable dispatchers(cache);
  for (auto dispatcher : dispatchers) {
    name = dispatcher.Get<Class::kInvocationDispatcherName>();
    if (name.IsNull()) break;  // Reached last entry.
    if (!name.Equals(target_name)) continue;
    desc = dispatcher.Get<Class::kInvocationDispatcherArgsDesc>();
    if (desc.raw() != args_desc.raw()) continue;
    function = dispatcher.Get<Class::kInvocationDispatcherFunction>();
    if (function.kind() == kind) {
      break;  // Found match.
    }
  }

  if (function.IsNull() && create_if_absent) {
    function = CreateInvocationDispatcher(target_name, args_desc, kind);
    AddInvocationDispatcher(target_name, args_desc, function);
  }
  return function.raw();
}

FunctionPtr Class::CreateInvocationDispatcher(const String& target_name,
                                              const Array& args_desc,
                                              FunctionLayout::Kind kind) const {
```

