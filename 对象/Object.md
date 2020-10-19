# Object

## object.h

```c++
/// object.h
/// line 280-844
class Object {
 public:
  using ObjectLayoutType = ObjectLayout;
  using ObjectPtrType = ObjectPtr;

  static ObjectPtr RawCast(ObjectPtr obj) { return obj; }

  virtual ~Object() {}

  ObjectPtr raw() const { return raw_; }
  void operator=(ObjectPtr value) { initializeHandle(this, value); }

  uint32_t CompareAndSwapTags(uint32_t old_tags, uint32_t new_tags) const {
    raw()->ptr()->tags_.StrongCAS(old_tags, new_tags);
    return old_tags;
  }
  bool IsCanonical() const { return raw()->ptr()->IsCanonical(); }
  void SetCanonical() const { raw()->ptr()->SetCanonical(); }
  void ClearCanonical() const { raw()->ptr()->ClearCanonical(); }
  intptr_t GetClassId() const {
    return !raw()->IsHeapObject() ? static_cast<intptr_t>(kSmiCid)
                                  : raw()->ptr()->GetClassId();
  }
  inline ClassPtr clazz() const;
  static intptr_t tags_offset() { return OFFSET_OF(ObjectLayout, tags_); }

// Class testers.
#define DEFINE_CLASS_TESTER(clazz)                                             \
  virtual bool Is##clazz() const { return false; }
  CLASS_LIST_FOR_HANDLES(DEFINE_CLASS_TESTER);
#undef DEFINE_CLASS_TESTER

  bool IsNull() const { return raw_ == null_; }

  // Matches Object.toString on instances (except String::ToCString, bug 20583).
  virtual const char* ToCString() const {
    if (IsNull()) {
      return "null";
    } else {
      return "Object";
    }
  }

#ifndef PRODUCT
  void PrintJSON(JSONStream* stream, bool ref = true) const;
  virtual void PrintJSONImpl(JSONStream* stream, bool ref) const;
  virtual const char* JSONType() const { return IsNull() ? "null" : "Object"; }
#endif

  // Returns the name that is used to identify an object in the
  // namespace dictionary.
  // Object::DictionaryName() returns String::null(). Only subclasses
  // of Object that need to be entered in the library and library prefix
  // namespaces need to provide an implementation.
  virtual StringPtr DictionaryName() const;

  bool IsNew() const { return raw()->IsNewObject(); }
  bool IsOld() const { return raw()->IsOldObject(); }
#if defined(DEBUG)
  bool InVMIsolateHeap() const;
#else
  bool InVMIsolateHeap() const { return raw()->ptr()->InVMIsolateHeap(); }
#endif  // DEBUG

  // Print the object on stdout for debugging.
  void Print() const;

  bool IsZoneHandle() const {
    return VMHandles::IsZoneHandle(reinterpret_cast<uword>(this));
  }

  bool IsReadOnlyHandle() const;

  bool IsNotTemporaryScopedHandle() const;

  static Object& Handle(Zone* zone, ObjectPtr raw_ptr) {
    Object* obj = reinterpret_cast<Object*>(VMHandles::AllocateHandle(zone));
    initializeHandle(obj, raw_ptr);
    return *obj;
  }
  static Object* ReadOnlyHandle() {
    Object* obj = reinterpret_cast<Object*>(Dart::AllocateReadOnlyHandle());
    initializeHandle(obj, Object::null());
    return obj;
  }

  static Object& Handle() { return Handle(Thread::Current()->zone(), null_); }

  static Object& Handle(Zone* zone) { return Handle(zone, null_); }

  static Object& Handle(ObjectPtr raw_ptr) {
    return Handle(Thread::Current()->zone(), raw_ptr);
  }

  static Object& ZoneHandle(Zone* zone, ObjectPtr raw_ptr) {
    Object* obj =
        reinterpret_cast<Object*>(VMHandles::AllocateZoneHandle(zone));
    initializeHandle(obj, raw_ptr);
    return *obj;
  }

  static Object& ZoneHandle(Zone* zone) { return ZoneHandle(zone, null_); }

  static Object& ZoneHandle() {
    return ZoneHandle(Thread::Current()->zone(), null_);
  }

  static Object& ZoneHandle(ObjectPtr raw_ptr) {
    return ZoneHandle(Thread::Current()->zone(), raw_ptr);
  }

  static ObjectPtr null() { return null_; }

#if defined(HASH_IN_OBJECT_HEADER)
  static uint32_t GetCachedHash(const ObjectPtr obj) {
    return obj->ptr()->hash_;
  }

  static void SetCachedHash(ObjectPtr obj, uint32_t hash) {
    obj->ptr()->hash_ = hash;
  }
#endif

  // The list below enumerates read-only handles for singleton
  // objects that are shared between the different isolates.
  //
  // - sentinel is a value that cannot be produced by Dart code. It can be used
  // to mark special values, for example to distinguish "uninitialized" fields.
  // - transition_sentinel is a value marking that we are transitioning from
  // sentinel, e.g., computing a field value. Used to detect circular
  // initialization.
  // - unknown_constant and non_constant are optimizing compiler's constant
  // propagation constants.
#define SHARED_READONLY_HANDLES_LIST(V)                                        \
  V(Object, null_object)                                                       \
  V(Array, null_array)                                                         \
  V(String, null_string)                                                       \
  V(Instance, null_instance)                                                   \
  V(Function, null_function)                                                   \
  V(TypeArguments, null_type_arguments)                                        \
  V(CompressedStackMaps, null_compressed_stack_maps)                           \
  V(TypeArguments, empty_type_arguments)                                       \
  V(Array, empty_array)                                                        \
  V(Array, zero_array)                                                         \
  V(ContextScope, empty_context_scope)                                         \
  V(ObjectPool, empty_object_pool)                                             \
  V(PcDescriptors, empty_descriptors)                                          \
  V(LocalVarDescriptors, empty_var_descriptors)                                \
  V(ExceptionHandlers, empty_exception_handlers)                               \
  V(Array, extractor_parameter_types)                                          \
  V(Array, extractor_parameter_names)                                          \
  V(Bytecode, implicit_getter_bytecode)                                        \
  V(Bytecode, implicit_setter_bytecode)                                        \
  V(Bytecode, implicit_static_getter_bytecode)                                 \
  V(Bytecode, method_extractor_bytecode)                                       \
  V(Bytecode, invoke_closure_bytecode)                                         \
  V(Bytecode, invoke_field_bytecode)                                           \
  V(Bytecode, nsm_dispatcher_bytecode)                                         \
  V(Bytecode, dynamic_invocation_forwarder_bytecode)                           \
  V(Instance, sentinel)                                                        \
  V(Instance, transition_sentinel)                                             \
  V(Instance, unknown_constant)                                                \
  V(Instance, non_constant)                                                    \
  V(Bool, bool_true)                                                           \
  V(Bool, bool_false)                                                          \
  V(Smi, smi_illegal_cid)                                                      \
  V(Smi, smi_zero)                                                             \
  V(ApiError, typed_data_acquire_error)                                        \
  V(LanguageError, snapshot_writer_error)                                      \
  V(LanguageError, branch_offset_error)                                        \
  V(LanguageError, speculative_inlining_error)                                 \
  V(LanguageError, background_compilation_error)                               \
  V(LanguageError, out_of_memory_error)                                        \
  V(Array, vm_isolate_snapshot_object_table)                                   \
  V(Type, dynamic_type)                                                        \
  V(Type, void_type)                                                           \
  V(AbstractType, null_abstract_type)

#define DEFINE_SHARED_READONLY_HANDLE_GETTER(Type, name)                       \
  static const Type& name() {                                                  \
    ASSERT(name##_ != nullptr);                                                \
    return *name##_;                                                           \
  }
  SHARED_READONLY_HANDLES_LIST(DEFINE_SHARED_READONLY_HANDLE_GETTER)
#undef DEFINE_SHARED_READONLY_HANDLE_GETTER

  static void set_vm_isolate_snapshot_object_table(const Array& table);

  static ClassPtr class_class() { return class_class_; }
  static ClassPtr dynamic_class() { return dynamic_class_; }
  static ClassPtr void_class() { return void_class_; }
  static ClassPtr type_arguments_class() { return type_arguments_class_; }
  static ClassPtr patch_class_class() { return patch_class_class_; }
  static ClassPtr function_class() { return function_class_; }
  static ClassPtr closure_data_class() { return closure_data_class_; }
  static ClassPtr signature_data_class() { return signature_data_class_; }
  static ClassPtr redirection_data_class() { return redirection_data_class_; }
  static ClassPtr ffi_trampoline_data_class() {
    return ffi_trampoline_data_class_;
  }
  static ClassPtr field_class() { return field_class_; }
  static ClassPtr script_class() { return script_class_; }
  static ClassPtr library_class() { return library_class_; }
  static ClassPtr namespace_class() { return namespace_class_; }
  static ClassPtr kernel_program_info_class() {
    return kernel_program_info_class_;
  }
  static ClassPtr code_class() { return code_class_; }
  static ClassPtr bytecode_class() { return bytecode_class_; }
  static ClassPtr instructions_class() { return instructions_class_; }
  static ClassPtr instructions_section_class() {
    return instructions_section_class_;
  }
  static ClassPtr object_pool_class() { return object_pool_class_; }
  static ClassPtr pc_descriptors_class() { return pc_descriptors_class_; }
  static ClassPtr code_source_map_class() { return code_source_map_class_; }
  static ClassPtr compressed_stackmaps_class() {
    return compressed_stackmaps_class_;
  }
  static ClassPtr var_descriptors_class() { return var_descriptors_class_; }
  static ClassPtr exception_handlers_class() {
    return exception_handlers_class_;
  }
  static ClassPtr deopt_info_class() { return deopt_info_class_; }
  static ClassPtr context_class() { return context_class_; }
  static ClassPtr context_scope_class() { return context_scope_class_; }
  static ClassPtr api_error_class() { return api_error_class_; }
  static ClassPtr language_error_class() { return language_error_class_; }
  static ClassPtr unhandled_exception_class() {
    return unhandled_exception_class_;
  }
  static ClassPtr unwind_error_class() { return unwind_error_class_; }
  static ClassPtr dyncalltypecheck_class() { return dyncalltypecheck_class_; }
  static ClassPtr singletargetcache_class() { return singletargetcache_class_; }
  static ClassPtr unlinkedcall_class() { return unlinkedcall_class_; }
  static ClassPtr monomorphicsmiablecall_class() {
    return monomorphicsmiablecall_class_;
  }
  static ClassPtr icdata_class() { return icdata_class_; }
  static ClassPtr megamorphic_cache_class() { return megamorphic_cache_class_; }
  static ClassPtr subtypetestcache_class() { return subtypetestcache_class_; }
  static ClassPtr loadingunit_class() { return loadingunit_class_; }
  static ClassPtr weak_serialization_reference_class() {
    return weak_serialization_reference_class_;
  }

  // Initialize the VM isolate.
  static void InitNullAndBool(Isolate* isolate);
  static void Init(Isolate* isolate);
  static void InitVtables();
  static void FinishInit(Isolate* isolate);
  static void FinalizeVMIsolate(Isolate* isolate);
  static void FinalizeReadOnlyObject(ObjectPtr object);

  static void Cleanup();

  // Initialize a new isolate either from a Kernel IR, from source, or from a
  // snapshot.
  static ErrorPtr Init(Isolate* isolate,
                       const uint8_t* kernel_buffer,
                       intptr_t kernel_buffer_size);

  static void MakeUnusedSpaceTraversable(const Object& obj,
                                         intptr_t original_size,
                                         intptr_t used_size);

  static intptr_t InstanceSize() {
    return RoundedAllocationSize(sizeof(ObjectLayout));
  }

  template <class FakeObject>
  static void VerifyBuiltinVtable(intptr_t cid) {
    FakeObject fake;
    if (cid >= kNumPredefinedCids) {
      cid = kInstanceCid;
    }
    ASSERT(builtin_vtables_[cid] == fake.vtable());
  }
  static void VerifyBuiltinVtables();

  static const ClassId kClassId = kObjectCid;

  // Different kinds of name visibility.
  enum NameVisibility {
    // Internal names are the true names of classes, fields,
    // etc. inside the vm.  These names include privacy suffixes,
    // getter prefixes, and trailing dots on unnamed constructors.
    //
    // The names of core implementation classes (like _OneByteString)
    // are preserved as well.
    //
    // e.g.
    //   private getter             -> get:foo@6be832b
    //   private constructor        -> _MyClass@6b3832b.
    //   private named constructor  -> _MyClass@6b3832b.named
    //   core impl class name shown -> _OneByteString
    kInternalName = 0,

    // Scrubbed names drop privacy suffixes, getter prefixes, and
    // trailing dots on unnamed constructors.  These names are used in
    // the vm service.
    //
    // e.g.
    //   get:foo@6be832b        -> foo
    //   _MyClass@6b3832b.      -> _MyClass
    //   _MyClass@6b3832b.named -> _MyClass.named
    //   _OneByteString         -> _OneByteString (not remapped)
    kScrubbedName,

    // User visible names are appropriate for reporting type errors
    // directly to programmers.  The names have been scrubbed and
    // the names of core implementation classes are remapped to their
    // public interface names.
    //
    // e.g.
    //   get:foo@6be832b        -> foo
    //   _MyClass@6b3832b.      -> _MyClass
    //   _MyClass@6b3832b.named -> _MyClass.named
    //   _OneByteString         -> String (remapped)
    kUserVisibleName
  };

  // Sometimes simple formating might produce the same name for two different
  // entities, for example we might inject a synthetic forwarder into the
  // class which has the same name as an already existing function, or
  // two different types can be formatted as X<T> because T has different
  // meaning (refers to a different type parameter) in these two types.
  // Such ambiguity might be acceptable in some contexts but not in others, so
  // some formatting methods have two modes - one which tries to be more
  // user friendly, and another one which tries to avoid name conflicts by
  // emitting longer and less user friendly names.
  enum class NameDisambiguation {
    kYes,
    kNo,
  };

 protected:
  // Used for extracting the C++ vtable during bringup.
  Object() : raw_(null_) {}

  uword raw_value() const { return static_cast<uword>(raw()); }

  inline void SetRaw(ObjectPtr value);
  void CheckHandle() const;

  cpp_vtable vtable() const { return bit_copy<cpp_vtable>(*this); }
  void set_vtable(cpp_vtable value) { *vtable_address() = value; }

  static ObjectPtr Allocate(intptr_t cls_id, intptr_t size, Heap::Space space);

  static intptr_t RoundedAllocationSize(intptr_t size) {
    return Utils::RoundUp(size, kObjectAlignment);
  }

  bool Contains(uword addr) const { return raw()->ptr()->Contains(addr); }

  // Start of field mutator guards.
  //
  // All writes to heap objects should ultimately pass through one of the
  // methods below or their counterparts in RawObject, to ensure that the
  // write barrier is correctly applied.

  template <typename type, std::memory_order order = std::memory_order_relaxed>
  type LoadPointer(type const* addr) const {
    return raw()->ptr()->LoadPointer<type, order>(addr);
  }

  template <typename type, std::memory_order order = std::memory_order_relaxed>
  void StorePointer(type const* addr, type value) const {
    raw()->ptr()->StorePointer<type, order>(addr, value);
  }

  // Use for storing into an explicitly Smi-typed field of an object
  // (i.e., both the previous and new value are Smis).
  void StoreSmi(SmiPtr const* addr, SmiPtr value) const {
    raw()->ptr()->StoreSmi(addr, value);
  }
  void StoreSmiIgnoreRace(SmiPtr const* addr, SmiPtr value) const {
    raw()->ptr()->StoreSmiIgnoreRace(addr, value);
  }

  template <typename FieldType>
  void StoreSimd128(const FieldType* addr, simd128_value_t value) const {
    ASSERT(Contains(reinterpret_cast<uword>(addr)));
    value.writeTo(const_cast<FieldType*>(addr));
  }

  template <typename FieldType>
  FieldType LoadNonPointer(const FieldType* addr) const {
    return *const_cast<FieldType*>(addr);
  }

  template <typename FieldType, std::memory_order order>
  FieldType LoadNonPointer(const FieldType* addr) const {
    return reinterpret_cast<std::atomic<FieldType>*>(
               const_cast<FieldType*>(addr))
        ->load(order);
  }

  // Needs two template arguments to allow assigning enums to fixed-size ints.
  template <typename FieldType, typename ValueType>
  void StoreNonPointer(const FieldType* addr, ValueType value) const {
    // Can't use Contains, as it uses tags_, which is set through this method.
    ASSERT(reinterpret_cast<uword>(addr) >= ObjectLayout::ToAddr(raw()));
    *const_cast<FieldType*>(addr) = value;
  }

  template <typename FieldType, typename ValueType, std::memory_order order>
  void StoreNonPointer(const FieldType* addr, ValueType value) const {
    // Can't use Contains, as it uses tags_, which is set through this method.
    ASSERT(reinterpret_cast<uword>(addr) >= ObjectLayout::ToAddr(raw()));
    reinterpret_cast<std::atomic<FieldType>*>(const_cast<FieldType*>(addr))
        ->store(value, order);
  }

  // Provides non-const access to non-pointer fields within the object. Such
  // access does not need a write barrier, but it is *not* GC-safe, since the
  // object might move, hence must be fully contained within a NoSafepointScope.
  template <typename FieldType>
  FieldType* UnsafeMutableNonPointer(const FieldType* addr) const {
    // Allow pointers at the end of variable-length data, and disallow pointers
    // within the header word.
    ASSERT(Contains(reinterpret_cast<uword>(addr) - 1) &&
           Contains(reinterpret_cast<uword>(addr) - kWordSize));
    // At least check that there is a NoSafepointScope and hope it's big enough.
    ASSERT(Thread::Current()->no_safepoint_scope_depth() > 0);
    return const_cast<FieldType*>(addr);
  }

// Fail at link time if StoreNonPointer or UnsafeMutableNonPointer is
// instantiated with an object pointer type.
#define STORE_NON_POINTER_ILLEGAL_TYPE(type)                                   \
  template <typename ValueType>                                                \
  void StoreNonPointer(type##Ptr const* addr, ValueType value) const {         \
    UnimplementedMethod();                                                     \
  }                                                                            \
  type##Ptr* UnsafeMutableNonPointer(type##Ptr const* addr) const {            \
    UnimplementedMethod();                                                     \
    return NULL;                                                               \
  }

  CLASS_LIST(STORE_NON_POINTER_ILLEGAL_TYPE);
  void UnimplementedMethod() const;
#undef STORE_NON_POINTER_ILLEGAL_TYPE

  // Allocate an object and copy the body of 'orig'.
  static ObjectPtr Clone(const Object& orig, Heap::Space space);

  // End of field mutator guards.

  ObjectPtr raw_;  // The raw object reference.

 protected:
  void AddCommonObjectProperties(JSONObject* jsobj,
                                 const char* protocol_type,
                                 bool ref) const;

 private:
  static intptr_t NextFieldOffset() {
    // Indicates this class cannot be extended by dart code.
    return -kWordSize;
  }

  static void InitializeObject(uword address, intptr_t id, intptr_t size);

  static void RegisterClass(const Class& cls,
                            const String& name,
                            const Library& lib);
  static void RegisterPrivateClass(const Class& cls,
                                   const String& name,
                                   const Library& lib);

  /* Initialize the handle based on the raw_ptr in the presence of null. */
  static void initializeHandle(Object* obj, ObjectPtr raw_ptr) {
    if (raw_ptr != Object::null()) {
      obj->SetRaw(raw_ptr);
    } else {
      obj->raw_ = Object::null();
      Object fake_object;
      obj->set_vtable(fake_object.vtable());
    }
  }

  cpp_vtable* vtable_address() const {
    uword vtable_addr = reinterpret_cast<uword>(this);
    return reinterpret_cast<cpp_vtable*>(vtable_addr);
  }

  static cpp_vtable builtin_vtables_[kNumPredefinedCids];

  // The static values below are singletons shared between the different
  // isolates. They are all allocated in the non-GC'd Dart::vm_isolate_.
  static ObjectPtr null_;
  static BoolPtr true_;
  static BoolPtr false_;

  static ClassPtr class_class_;           // Class of the Class vm object.
  static ClassPtr dynamic_class_;         // Class of the 'dynamic' type.
  static ClassPtr void_class_;            // Class of the 'void' type.
  static ClassPtr type_arguments_class_;  // Class of TypeArguments vm object.
  static ClassPtr patch_class_class_;     // Class of the PatchClass vm object.
  static ClassPtr function_class_;        // Class of the Function vm object.
  static ClassPtr closure_data_class_;    // Class of ClosureData vm obj.
  static ClassPtr signature_data_class_;  // Class of SignatureData vm obj.
  static ClassPtr redirection_data_class_;  // Class of RedirectionData vm obj.
  static ClassPtr ffi_trampoline_data_class_;  // Class of FfiTrampolineData
                                               // vm obj.
  static ClassPtr field_class_;                // Class of the Field vm object.
  static ClassPtr script_class_;               // Class of the Script vm object.
  static ClassPtr library_class_;    // Class of the Library vm object.
  static ClassPtr namespace_class_;  // Class of Namespace vm object.
  static ClassPtr kernel_program_info_class_;  // Class of KernelProgramInfo vm
                                               // object.
  static ClassPtr code_class_;                 // Class of the Code vm object.
  static ClassPtr bytecode_class_;      // Class of the Bytecode vm object.
  static ClassPtr instructions_class_;  // Class of the Instructions vm object.
  static ClassPtr instructions_section_class_;  // Class of InstructionsSection.
  static ClassPtr object_pool_class_;      // Class of the ObjectPool vm object.
  static ClassPtr pc_descriptors_class_;   // Class of PcDescriptors vm object.
  static ClassPtr code_source_map_class_;  // Class of CodeSourceMap vm object.
  static ClassPtr compressed_stackmaps_class_;  // Class of CompressedStackMaps.
  static ClassPtr var_descriptors_class_;       // Class of LocalVarDescriptors.
  static ClassPtr exception_handlers_class_;    // Class of ExceptionHandlers.
  static ClassPtr deopt_info_class_;            // Class of DeoptInfo.
  static ClassPtr context_class_;            // Class of the Context vm object.
  static ClassPtr context_scope_class_;      // Class of ContextScope vm object.
  static ClassPtr dyncalltypecheck_class_;   // Class of ParameterTypeCheck.
  static ClassPtr singletargetcache_class_;  // Class of SingleTargetCache.
  static ClassPtr unlinkedcall_class_;       // Class of UnlinkedCall.
  static ClassPtr
      monomorphicsmiablecall_class_;         // Class of MonomorphicSmiableCall.
  static ClassPtr icdata_class_;             // Class of ICData.
  static ClassPtr megamorphic_cache_class_;  // Class of MegamorphiCache.
  static ClassPtr subtypetestcache_class_;   // Class of SubtypeTestCache.
  static ClassPtr loadingunit_class_;        // Class of LoadingUnit.
  static ClassPtr api_error_class_;          // Class of ApiError.
  static ClassPtr language_error_class_;     // Class of LanguageError.
  static ClassPtr unhandled_exception_class_;  // Class of UnhandledException.
  static ClassPtr unwind_error_class_;         // Class of UnwindError.
  // Class of WeakSerializationReference.
  static ClassPtr weak_serialization_reference_class_;

#define DECLARE_SHARED_READONLY_HANDLE(Type, name) static Type* name##_;
  SHARED_READONLY_HANDLES_LIST(DECLARE_SHARED_READONLY_HANDLE)
#undef DECLARE_SHARED_READONLY_HANDLE

  friend void ClassTable::Register(const Class& cls);
  friend void ObjectLayout::Validate(IsolateGroup* isolate_group) const;
  friend class Closure;
  friend class SnapshotReader;
  friend class InstanceDeserializationCluster;
  friend class OneByteString;
  friend class TwoByteString;
  friend class ExternalOneByteString;
  friend class ExternalTwoByteString;
  friend class Thread;

#define REUSABLE_FRIEND_DECLARATION(name)                                      \
  friend class Reusable##name##HandleScope;
  REUSABLE_HANDLE_LIST(REUSABLE_FRIEND_DECLARATION)
#undef REUSABLE_FRIEND_DECLARATION

  DISALLOW_ALLOCATION();
  DISALLOW_COPY_AND_ASSIGN(Object);
};


// Breaking cycles and loops.
ClassPtr Object::clazz() const {
  uword raw_value = static_cast<uword>(raw_);
  if ((raw_value & kSmiTagMask) == kSmiTag) {
    return Smi::Class();
  }
  ASSERT(!IsolateGroup::Current()->compaction_in_progress());
  return Isolate::Current()->class_table()->At(raw()->GetClassId());
}

DART_FORCE_INLINE
void Object::SetRaw(ObjectPtr value) {
  NoSafepointScope no_safepoint_scope;
  raw_ = value;
  intptr_t cid = value->GetClassIdMayBeSmi();
  // Free-list elements cannot be wrapped in a handle.
  ASSERT(cid != kFreeListElement);
  ASSERT(cid != kForwardingCorpse);
  if (cid >= kNumPredefinedCids) {
    cid = kInstanceCid;
  }
  set_vtable(builtin_vtables_[cid]);
#if defined(DEBUG)
  if (FLAG_verify_handles && raw_->IsHeapObject()) {
    Isolate* isolate = Isolate::Current();
    Heap* isolate_heap = isolate->heap();
    // TODO(rmacnak): Remove after rewriting StackFrame::VisitObjectPointers
    // to not use handles.
    if (!isolate_heap->new_space()->scavenging()) {
      Heap* vm_isolate_heap = Dart::vm_isolate()->heap();
      uword addr = ObjectLayout::ToAddr(raw_);
      if (!isolate_heap->Contains(addr) && !vm_isolate_heap->Contains(addr)) {
        ASSERT(FLAG_write_protect_code);
        addr = ObjectLayout::ToAddr(OldPage::ToWritable(raw_));
        ASSERT(isolate_heap->Contains(addr) || vm_isolate_heap->Contains(addr));
      }
    }
  }
#endif
}
```



## object.cc

```c++
/// object.cc
/// line 2724-2531

void Object::InitNullAndBool(Isolate* isolate) {
  // Should only be run by the vm isolate.
  ASSERT(isolate == Dart::vm_isolate());

  // TODO(iposva): NoSafepointScope needs to be added here.
  ASSERT(class_class() == null_);

  Heap* heap = isolate->heap();

  // Allocate and initialize the null instance.
  // 'null_' must be the first object allocated as it is used in allocation to
  // clear the object.
  {
    uword address = heap->Allocate(Instance::InstanceSize(), Heap::kOld);
    null_ = static_cast<InstancePtr>(address + kHeapObjectTag);
    // The call below is using 'null_' to initialize itself.
    InitializeObject(address, kNullCid, Instance::InstanceSize());
    null_->ptr()->SetCanonical();
  }

  // Allocate and initialize the bool instances.
  // These must be allocated such that at kBoolValueBitPosition, the address
  // of true is 0 and the address of false is 1, and their addresses are
  // otherwise identical.
  {
    // Allocate a dummy bool object to give true the desired alignment.
    uword address = heap->Allocate(Bool::InstanceSize(), Heap::kOld);
    InitializeObject(address, kBoolCid, Bool::InstanceSize());
    static_cast<BoolPtr>(address + kHeapObjectTag)->ptr()->value_ = false;
  }
  {
    // Allocate true.
    uword address = heap->Allocate(Bool::InstanceSize(), Heap::kOld);
    true_ = static_cast<BoolPtr>(address + kHeapObjectTag);
    InitializeObject(address, kBoolCid, Bool::InstanceSize());
    true_->ptr()->value_ = true;
    true_->ptr()->SetCanonical();
  }
  {
    // Allocate false.
    uword address = heap->Allocate(Bool::InstanceSize(), Heap::kOld);
    false_ = static_cast<BoolPtr>(address + kHeapObjectTag);
    InitializeObject(address, kBoolCid, Bool::InstanceSize());
    false_->ptr()->value_ = false;
    false_->ptr()->SetCanonical();
  }

  // Check that the objects have been allocated at appropriate addresses.
  ASSERT(static_cast<uword>(true_) ==
         static_cast<uword>(null_) + kTrueOffsetFromNull);
  ASSERT(static_cast<uword>(false_) ==
         static_cast<uword>(null_) + kFalseOffsetFromNull);
  ASSERT((static_cast<uword>(true_) & kBoolValueMask) == 0);
  ASSERT((static_cast<uword>(false_) & kBoolValueMask) != 0);
  ASSERT(static_cast<uword>(false_) ==
         (static_cast<uword>(true_) | kBoolValueMask));
}

void Object::InitVtables() {
  {
    Object fake_handle;
    builtin_vtables_[kObjectCid] = fake_handle.vtable();
  }

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    clazz fake_handle;                                                         \
    builtin_vtables_[k##clazz##Cid] = fake_handle.vtable();                    \
  }
  CLASS_LIST_NO_OBJECT_NOR_STRING_NOR_ARRAY(INIT_VTABLE)
#undef INIT_VTABLE

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    Array fake_handle;                                                         \
    builtin_vtables_[k##clazz##Cid] = fake_handle.vtable();                    \
  }
  CLASS_LIST_ARRAYS(INIT_VTABLE)
#undef INIT_VTABLE

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    String fake_handle;                                                        \
    builtin_vtables_[k##clazz##Cid] = fake_handle.vtable();                    \
  }
  CLASS_LIST_STRINGS(INIT_VTABLE)
#undef INIT_VTABLE

  {
    Instance fake_handle;
    builtin_vtables_[kFfiNativeTypeCid] = fake_handle.vtable();
  }

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    Instance fake_handle;                                                      \
    builtin_vtables_[kFfi##clazz##Cid] = fake_handle.vtable();                 \
  }
  CLASS_LIST_FFI_TYPE_MARKER(INIT_VTABLE)
#undef INIT_VTABLE

  {
    Instance fake_handle;
    builtin_vtables_[kFfiNativeFunctionCid] = fake_handle.vtable();
  }

  {
    Pointer fake_handle;
    builtin_vtables_[kFfiPointerCid] = fake_handle.vtable();
  }

  {
    DynamicLibrary fake_handle;
    builtin_vtables_[kFfiDynamicLibraryCid] = fake_handle.vtable();
  }

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    Instance fake_handle;                                                      \
    builtin_vtables_[k##clazz##Cid] = fake_handle.vtable();                    \
  }
  CLASS_LIST_WASM(INIT_VTABLE)
#undef INIT_VTABLE

#define INIT_VTABLE(clazz)                                                     \
  {                                                                            \
    TypedData fake_internal_handle;                                            \
    builtin_vtables_[kTypedData##clazz##Cid] = fake_internal_handle.vtable();  \
    TypedDataView fake_view_handle;                                            \
    builtin_vtables_[kTypedData##clazz##ViewCid] = fake_view_handle.vtable();  \
    ExternalTypedData fake_external_handle;                                    \
    builtin_vtables_[kExternalTypedData##clazz##Cid] =                         \
        fake_external_handle.vtable();                                         \
  }
  CLASS_LIST_TYPED_DATA(INIT_VTABLE)
#undef INIT_VTABLE

  {
    TypedDataView fake_handle;
    builtin_vtables_[kByteDataViewCid] = fake_handle.vtable();
  }

  {
    Instance fake_handle;
    builtin_vtables_[kByteBufferCid] = fake_handle.vtable();
    builtin_vtables_[kNullCid] = fake_handle.vtable();
    builtin_vtables_[kDynamicCid] = fake_handle.vtable();
    builtin_vtables_[kVoidCid] = fake_handle.vtable();
    builtin_vtables_[kNeverCid] = fake_handle.vtable();
  }
}

void Object::Init(Isolate* isolate) {
  // Should only be run by the vm isolate.
  ASSERT(isolate == Dart::vm_isolate());

  InitVtables();

  Heap* heap = isolate->heap();

// Allocate the read only object handles here.
#define INITIALIZE_SHARED_READONLY_HANDLE(Type, name)                          \
  name##_ = Type::ReadOnlyHandle();
  SHARED_READONLY_HANDLES_LIST(INITIALIZE_SHARED_READONLY_HANDLE)
#undef INITIALIZE_SHARED_READONLY_HANDLE

  *null_object_ = Object::null();
  *null_array_ = Array::null();
  *null_string_ = String::null();
  *null_instance_ = Instance::null();
  *null_function_ = Function::null();
  *null_type_arguments_ = TypeArguments::null();
  *empty_type_arguments_ = TypeArguments::null();
  *null_abstract_type_ = AbstractType::null();
  *null_compressed_stack_maps_ = CompressedStackMaps::null();
  *bool_true_ = true_;
  *bool_false_ = false_;

  // Initialize the empty and zero array handles to null_ in order to be able to
  // check if the empty and zero arrays were allocated (RAW_NULL is not
  // available).
  *empty_array_ = Array::null();
  *zero_array_ = Array::null();

  Class& cls = Class::Handle();

  // Allocate and initialize the class class.
  {
    intptr_t size = Class::InstanceSize();
    uword address = heap->Allocate(size, Heap::kOld);
    class_class_ = static_cast<ClassPtr>(address + kHeapObjectTag);
    InitializeObject(address, Class::kClassId, size);

    Class fake;
    // Initialization from Class::New<Class>.
    // Directly set raw_ to break a circular dependency: SetRaw will attempt
    // to lookup class class in the class table where it is not registered yet.
    cls.raw_ = class_class_;
    ASSERT(builtin_vtables_[kClassCid] == fake.vtable());
    cls.set_instance_size(
        Class::InstanceSize(),
        compiler::target::RoundedAllocationSize(RTN::Class::InstanceSize()));
    const intptr_t host_next_field_offset = Class::NextFieldOffset();
    const intptr_t target_next_field_offset = RTN::Class::NextFieldOffset();
    cls.set_next_field_offset(host_next_field_offset, target_next_field_offset);
    cls.set_id(Class::kClassId);
    cls.set_state_bits(0);
    cls.set_is_allocate_finalized();
    cls.set_is_declaration_loaded();
    cls.set_is_type_finalized();
    cls.set_type_arguments_field_offset_in_words(Class::kNoTypeArguments,
                                                 RTN::Class::kNoTypeArguments);
    cls.set_num_type_arguments(0);
    cls.set_num_native_fields(0);
    cls.InitEmptyFields();
    isolate->class_table()->Register(cls);
  }

  // Allocate and initialize the null class.
  cls = Class::New<Instance, RTN::Instance>(kNullCid, isolate);
  cls.set_num_type_arguments(0);
  isolate->object_store()->set_null_class(cls);

  // Allocate and initialize Never class.
  cls = Class::New<Instance, RTN::Instance>(kNeverCid, isolate);
  cls.set_num_type_arguments(0);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();
  isolate->object_store()->set_never_class(cls);

  // Allocate and initialize the free list element class.
  cls =
      Class::New<FreeListElement::FakeInstance,
                 RTN::FreeListElement::FakeInstance>(kFreeListElement, isolate);
  cls.set_num_type_arguments(0);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();

  // Allocate and initialize the forwarding corpse class.
  cls = Class::New<ForwardingCorpse::FakeInstance,
                   RTN::ForwardingCorpse::FakeInstance>(kForwardingCorpse,
                                                        isolate);
  cls.set_num_type_arguments(0);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();

  // Allocate and initialize the sentinel values.
  {
    *sentinel_ ^=
        Object::Allocate(kNeverCid, Instance::InstanceSize(), Heap::kOld);

    *transition_sentinel_ ^=
        Object::Allocate(kNeverCid, Instance::InstanceSize(), Heap::kOld);
  }

  // Allocate and initialize optimizing compiler constants.
  {
    *unknown_constant_ ^=
        Object::Allocate(kNeverCid, Instance::InstanceSize(), Heap::kOld);
    *non_constant_ ^=
        Object::Allocate(kNeverCid, Instance::InstanceSize(), Heap::kOld);
  }

  // Allocate the remaining VM internal classes.
  cls = Class::New<TypeArguments, RTN::TypeArguments>(isolate);
  type_arguments_class_ = cls.raw();

  cls = Class::New<PatchClass, RTN::PatchClass>(isolate);
  patch_class_class_ = cls.raw();

  cls = Class::New<Function, RTN::Function>(isolate);
  function_class_ = cls.raw();

  cls = Class::New<ClosureData, RTN::ClosureData>(isolate);
  closure_data_class_ = cls.raw();

  cls = Class::New<SignatureData, RTN::SignatureData>(isolate);
  signature_data_class_ = cls.raw();

  cls = Class::New<RedirectionData, RTN::RedirectionData>(isolate);
  redirection_data_class_ = cls.raw();

  cls = Class::New<FfiTrampolineData, RTN::FfiTrampolineData>(isolate);
  ffi_trampoline_data_class_ = cls.raw();

  cls = Class::New<Field, RTN::Field>(isolate);
  field_class_ = cls.raw();

  cls = Class::New<Script, RTN::Script>(isolate);
  script_class_ = cls.raw();

  cls = Class::New<Library, RTN::Library>(isolate);
  library_class_ = cls.raw();

  cls = Class::New<Namespace, RTN::Namespace>(isolate);
  namespace_class_ = cls.raw();

  cls = Class::New<KernelProgramInfo, RTN::KernelProgramInfo>(isolate);
  kernel_program_info_class_ = cls.raw();

  cls = Class::New<Code, RTN::Code>(isolate);
  code_class_ = cls.raw();

  cls = Class::New<Bytecode, RTN::Bytecode>(isolate);
  bytecode_class_ = cls.raw();

  cls = Class::New<Instructions, RTN::Instructions>(isolate);
  instructions_class_ = cls.raw();

  cls = Class::New<InstructionsSection, RTN::InstructionsSection>(isolate);
  instructions_section_class_ = cls.raw();

  cls = Class::New<ObjectPool, RTN::ObjectPool>(isolate);
  object_pool_class_ = cls.raw();

  cls = Class::New<PcDescriptors, RTN::PcDescriptors>(isolate);
  pc_descriptors_class_ = cls.raw();

  cls = Class::New<CodeSourceMap, RTN::CodeSourceMap>(isolate);
  code_source_map_class_ = cls.raw();

  cls = Class::New<CompressedStackMaps, RTN::CompressedStackMaps>(isolate);
  compressed_stackmaps_class_ = cls.raw();

  cls = Class::New<LocalVarDescriptors, RTN::LocalVarDescriptors>(isolate);
  var_descriptors_class_ = cls.raw();

  cls = Class::New<ExceptionHandlers, RTN::ExceptionHandlers>(isolate);
  exception_handlers_class_ = cls.raw();

  cls = Class::New<Context, RTN::Context>(isolate);
  context_class_ = cls.raw();

  cls = Class::New<ContextScope, RTN::ContextScope>(isolate);
  context_scope_class_ = cls.raw();

  cls = Class::New<ParameterTypeCheck, RTN::ParameterTypeCheck>(isolate);
  dyncalltypecheck_class_ = cls.raw();

  cls = Class::New<SingleTargetCache, RTN::SingleTargetCache>(isolate);
  singletargetcache_class_ = cls.raw();

  cls = Class::New<UnlinkedCall, RTN::UnlinkedCall>(isolate);
  unlinkedcall_class_ = cls.raw();

  cls =
      Class::New<MonomorphicSmiableCall, RTN::MonomorphicSmiableCall>(isolate);
  monomorphicsmiablecall_class_ = cls.raw();

  cls = Class::New<ICData, RTN::ICData>(isolate);
  icdata_class_ = cls.raw();

  cls = Class::New<MegamorphicCache, RTN::MegamorphicCache>(isolate);
  megamorphic_cache_class_ = cls.raw();

  cls = Class::New<SubtypeTestCache, RTN::SubtypeTestCache>(isolate);
  subtypetestcache_class_ = cls.raw();

  cls = Class::New<LoadingUnit, RTN::LoadingUnit>(isolate);
  loadingunit_class_ = cls.raw();

  cls = Class::New<ApiError, RTN::ApiError>(isolate);
  api_error_class_ = cls.raw();

  cls = Class::New<LanguageError, RTN::LanguageError>(isolate);
  language_error_class_ = cls.raw();

  cls = Class::New<UnhandledException, RTN::UnhandledException>(isolate);
  unhandled_exception_class_ = cls.raw();

  cls = Class::New<UnwindError, RTN::UnwindError>(isolate);
  unwind_error_class_ = cls.raw();

  cls = Class::New<WeakSerializationReference, RTN::WeakSerializationReference>(
      isolate);
  weak_serialization_reference_class_ = cls.raw();

  ASSERT(class_class() != null_);

  // Pre-allocate classes in the vm isolate so that we can for example create a
  // symbol table and populate it with some frequently used strings as symbols.
  cls = Class::New<Array, RTN::Array>(isolate);
  isolate->object_store()->set_array_class(cls);
  cls.set_type_arguments_field_offset(Array::type_arguments_offset(),
                                      RTN::Array::type_arguments_offset());
  cls.set_num_type_arguments(1);
  cls = Class::New<Array, RTN::Array>(kImmutableArrayCid, isolate);
  isolate->object_store()->set_immutable_array_class(cls);
  cls.set_type_arguments_field_offset(Array::type_arguments_offset(),
                                      RTN::Array::type_arguments_offset());
  cls.set_num_type_arguments(1);
  cls = Class::New<GrowableObjectArray, RTN::GrowableObjectArray>(isolate);
  isolate->object_store()->set_growable_object_array_class(cls);
  cls.set_type_arguments_field_offset(
      GrowableObjectArray::type_arguments_offset(),
      RTN::GrowableObjectArray::type_arguments_offset());
  cls.set_num_type_arguments(1);
  cls = Class::NewStringClass(kOneByteStringCid, isolate);
  isolate->object_store()->set_one_byte_string_class(cls);
  cls = Class::NewStringClass(kTwoByteStringCid, isolate);
  isolate->object_store()->set_two_byte_string_class(cls);
  cls = Class::New<Mint, RTN::Mint>(isolate);
  isolate->object_store()->set_mint_class(cls);
  cls = Class::New<Double, RTN::Double>(isolate);
  isolate->object_store()->set_double_class(cls);

  // Ensure that class kExternalTypedDataUint8ArrayCid is registered as we
  // need it when reading in the token stream of bootstrap classes in the VM
  // isolate.
  Class::NewExternalTypedDataClass(kExternalTypedDataUint8ArrayCid, isolate);

  // Needed for object pools of VM isolate stubs.
  Class::NewTypedDataClass(kTypedDataInt8ArrayCid, isolate);

  // Allocate and initialize the empty_array instance.
  {
    uword address = heap->Allocate(Array::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kImmutableArrayCid, Array::InstanceSize(0));
    Array::initializeHandle(empty_array_,
                            static_cast<ArrayPtr>(address + kHeapObjectTag));
    empty_array_->StoreSmi(&empty_array_->raw_ptr()->length_, Smi::New(0));
    empty_array_->SetCanonical();
  }

  Smi& smi = Smi::Handle();
  // Allocate and initialize the zero_array instance.
  {
    uword address = heap->Allocate(Array::InstanceSize(1), Heap::kOld);
    InitializeObject(address, kImmutableArrayCid, Array::InstanceSize(1));
    Array::initializeHandle(zero_array_,
                            static_cast<ArrayPtr>(address + kHeapObjectTag));
    zero_array_->StoreSmi(&zero_array_->raw_ptr()->length_, Smi::New(1));
    smi = Smi::New(0);
    zero_array_->SetAt(0, smi);
    zero_array_->SetCanonical();
  }

  // Allocate and initialize the canonical empty context scope object.
  {
    uword address = heap->Allocate(ContextScope::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kContextScopeCid, ContextScope::InstanceSize(0));
    ContextScope::initializeHandle(
        empty_context_scope_,
        static_cast<ContextScopePtr>(address + kHeapObjectTag));
    empty_context_scope_->StoreNonPointer(
        &empty_context_scope_->raw_ptr()->num_variables_, 0);
    empty_context_scope_->StoreNonPointer(
        &empty_context_scope_->raw_ptr()->is_implicit_, true);
    empty_context_scope_->SetCanonical();
  }

  // Allocate and initialize the canonical empty object pool object.
  {
    uword address = heap->Allocate(ObjectPool::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kObjectPoolCid, ObjectPool::InstanceSize(0));
    ObjectPool::initializeHandle(
        empty_object_pool_,
        static_cast<ObjectPoolPtr>(address + kHeapObjectTag));
    empty_object_pool_->StoreNonPointer(&empty_object_pool_->raw_ptr()->length_,
                                        0);
    empty_object_pool_->SetCanonical();
  }

  // Allocate and initialize the empty_descriptors instance.
  {
    uword address = heap->Allocate(PcDescriptors::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kPcDescriptorsCid,
                     PcDescriptors::InstanceSize(0));
    PcDescriptors::initializeHandle(
        empty_descriptors_,
        static_cast<PcDescriptorsPtr>(address + kHeapObjectTag));
    empty_descriptors_->StoreNonPointer(&empty_descriptors_->raw_ptr()->length_,
                                        0);
    empty_descriptors_->SetCanonical();
  }

  // Allocate and initialize the canonical empty variable descriptor object.
  {
    uword address =
        heap->Allocate(LocalVarDescriptors::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kLocalVarDescriptorsCid,
                     LocalVarDescriptors::InstanceSize(0));
    LocalVarDescriptors::initializeHandle(
        empty_var_descriptors_,
        static_cast<LocalVarDescriptorsPtr>(address + kHeapObjectTag));
    empty_var_descriptors_->StoreNonPointer(
        &empty_var_descriptors_->raw_ptr()->num_entries_, 0);
    empty_var_descriptors_->SetCanonical();
  }

  // Allocate and initialize the canonical empty exception handler info object.
  // The vast majority of all functions do not contain an exception handler
  // and can share this canonical descriptor.
  {
    uword address =
        heap->Allocate(ExceptionHandlers::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kExceptionHandlersCid,
                     ExceptionHandlers::InstanceSize(0));
    ExceptionHandlers::initializeHandle(
        empty_exception_handlers_,
        static_cast<ExceptionHandlersPtr>(address + kHeapObjectTag));
    empty_exception_handlers_->StoreNonPointer(
        &empty_exception_handlers_->raw_ptr()->num_entries_, 0);
    empty_exception_handlers_->SetCanonical();
  }

  // Allocate and initialize the canonical empty type arguments object.
  {
    uword address = heap->Allocate(TypeArguments::InstanceSize(0), Heap::kOld);
    InitializeObject(address, kTypeArgumentsCid,
                     TypeArguments::InstanceSize(0));
    TypeArguments::initializeHandle(
        empty_type_arguments_,
        static_cast<TypeArgumentsPtr>(address + kHeapObjectTag));
    empty_type_arguments_->StoreSmi(&empty_type_arguments_->raw_ptr()->length_,
                                    Smi::New(0));
    empty_type_arguments_->StoreSmi(&empty_type_arguments_->raw_ptr()->hash_,
                                    Smi::New(0));
    empty_type_arguments_->ComputeHash();
    empty_type_arguments_->SetCanonical();
  }

  // The VM isolate snapshot object table is initialized to an empty array
  // as we do not have any VM isolate snapshot at this time.
  *vm_isolate_snapshot_object_table_ = Object::empty_array().raw();

  cls = Class::New<Instance, RTN::Instance>(kDynamicCid, isolate);
  cls.set_is_abstract();
  cls.set_num_type_arguments(0);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();
  dynamic_class_ = cls.raw();

  cls = Class::New<Instance, RTN::Instance>(kVoidCid, isolate);
  cls.set_num_type_arguments(0);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();
  void_class_ = cls.raw();

  cls = Class::New<Type, RTN::Type>(isolate);
  cls.set_is_allocate_finalized();
  cls.set_is_declaration_loaded();
  cls.set_is_type_finalized();

  cls = dynamic_class_;
  *dynamic_type_ = Type::New(cls, Object::null_type_arguments(),
                             TokenPosition::kNoSource, Nullability::kNullable);
  dynamic_type_->SetIsFinalized();
  dynamic_type_->ComputeHash();
  dynamic_type_->SetCanonical();

  cls = void_class_;
  *void_type_ = Type::New(cls, Object::null_type_arguments(),
                          TokenPosition::kNoSource, Nullability::kNullable);
  void_type_->SetIsFinalized();
  void_type_->ComputeHash();
  void_type_->SetCanonical();

  // Since TypeArguments objects are passed as function arguments, make them
  // behave as Dart instances, although they are just VM objects.
  // Note that we cannot set the super type to ObjectType, which does not live
  // in the vm isolate. See special handling in Class::SuperClass().
  cls = type_arguments_class_;
  cls.set_interfaces(Object::empty_array());
  cls.SetFields(Object::empty_array());
  cls.SetFunctions(Object::empty_array());

  cls = Class::New<Bool, RTN::Bool>(isolate);
  isolate->object_store()->set_bool_class(cls);

  *smi_illegal_cid_ = Smi::New(kIllegalCid);
  *smi_zero_ = Smi::New(0);

  String& error_str = String::Handle();
  error_str = String::New(
      "Internal Dart data pointers have been acquired, please release them "
      "using Dart_TypedDataReleaseData.",
      Heap::kOld);
  *typed_data_acquire_error_ = ApiError::New(error_str, Heap::kOld);
  error_str = String::New("SnapshotWriter Error", Heap::kOld);
  *snapshot_writer_error_ =
      LanguageError::New(error_str, Report::kError, Heap::kOld);
  error_str = String::New("Branch offset overflow", Heap::kOld);
  *branch_offset_error_ =
      LanguageError::New(error_str, Report::kBailout, Heap::kOld);
  error_str = String::New("Speculative inlining failed", Heap::kOld);
  *speculative_inlining_error_ =
      LanguageError::New(error_str, Report::kBailout, Heap::kOld);
  error_str = String::New("Background Compilation Failed", Heap::kOld);
  *background_compilation_error_ =
      LanguageError::New(error_str, Report::kBailout, Heap::kOld);
  error_str = String::New("Out of memory", Heap::kOld);
  *out_of_memory_error_ =
      LanguageError::New(error_str, Report::kBailout, Heap::kOld);

  // Allocate the parameter arrays for method extractor types and names.
  *extractor_parameter_types_ = Array::New(1, Heap::kOld);
  extractor_parameter_types_->SetAt(0, Object::dynamic_type());
  *extractor_parameter_names_ = Array::New(1, Heap::kOld);
  // Fill in extractor_parameter_names_ later, after symbols are initialized
  // (in Object::FinalizeVMIsolate). extractor_parameter_names_ object
  // needs to be created earlier as VM isolate snapshot reader references it
  // before Object::FinalizeVMIsolate.

  *implicit_getter_bytecode_ =
      CreateVMInternalBytecode(KernelBytecode::kVMInternal_ImplicitGetter);

  *implicit_setter_bytecode_ =
      CreateVMInternalBytecode(KernelBytecode::kVMInternal_ImplicitSetter);

  *implicit_static_getter_bytecode_ = CreateVMInternalBytecode(
      KernelBytecode::kVMInternal_ImplicitStaticGetter);

  *method_extractor_bytecode_ =
      CreateVMInternalBytecode(KernelBytecode::kVMInternal_MethodExtractor);

  *invoke_closure_bytecode_ =
      CreateVMInternalBytecode(KernelBytecode::kVMInternal_InvokeClosure);

  *invoke_field_bytecode_ =
      CreateVMInternalBytecode(KernelBytecode::kVMInternal_InvokeField);

  *nsm_dispatcher_bytecode_ = CreateVMInternalBytecode(
      KernelBytecode::kVMInternal_NoSuchMethodDispatcher);

  *dynamic_invocation_forwarder_bytecode_ = CreateVMInternalBytecode(
      KernelBytecode::kVMInternal_ForwardDynamicInvocation);

  // Some thread fields need to be reinitialized as null constants have not been
  // initialized until now.
  Thread* thr = Thread::Current();
  ASSERT(thr != NULL);
  thr->ClearStickyError();
  thr->clear_pending_functions();

  ASSERT(!null_object_->IsSmi());
  ASSERT(!null_array_->IsSmi());
  ASSERT(null_array_->IsArray());
  ASSERT(!null_string_->IsSmi());
  ASSERT(null_string_->IsString());
  ASSERT(!null_instance_->IsSmi());
  ASSERT(null_instance_->IsInstance());
  ASSERT(!null_function_->IsSmi());
  ASSERT(null_function_->IsFunction());
  ASSERT(!null_type_arguments_->IsSmi());
  ASSERT(null_type_arguments_->IsTypeArguments());
  ASSERT(!null_compressed_stack_maps_->IsSmi());
  ASSERT(null_compressed_stack_maps_->IsCompressedStackMaps());
  ASSERT(!empty_array_->IsSmi());
  ASSERT(empty_array_->IsArray());
  ASSERT(!zero_array_->IsSmi());
  ASSERT(zero_array_->IsArray());
  ASSERT(!empty_context_scope_->IsSmi());
  ASSERT(empty_context_scope_->IsContextScope());
  ASSERT(!empty_descriptors_->IsSmi());
  ASSERT(empty_descriptors_->IsPcDescriptors());
  ASSERT(!empty_var_descriptors_->IsSmi());
  ASSERT(empty_var_descriptors_->IsLocalVarDescriptors());
  ASSERT(!empty_exception_handlers_->IsSmi());
  ASSERT(empty_exception_handlers_->IsExceptionHandlers());
  ASSERT(!sentinel_->IsSmi());
  ASSERT(sentinel_->IsInstance());
  ASSERT(!transition_sentinel_->IsSmi());
  ASSERT(transition_sentinel_->IsInstance());
  ASSERT(!unknown_constant_->IsSmi());
  ASSERT(unknown_constant_->IsInstance());
  ASSERT(!non_constant_->IsSmi());
  ASSERT(non_constant_->IsInstance());
  ASSERT(!bool_true_->IsSmi());
  ASSERT(bool_true_->IsBool());
  ASSERT(!bool_false_->IsSmi());
  ASSERT(bool_false_->IsBool());
  ASSERT(smi_illegal_cid_->IsSmi());
  ASSERT(smi_zero_->IsSmi());
  ASSERT(!typed_data_acquire_error_->IsSmi());
  ASSERT(typed_data_acquire_error_->IsApiError());
  ASSERT(!snapshot_writer_error_->IsSmi());
  ASSERT(snapshot_writer_error_->IsLanguageError());
  ASSERT(!branch_offset_error_->IsSmi());
  ASSERT(branch_offset_error_->IsLanguageError());
  ASSERT(!speculative_inlining_error_->IsSmi());
  ASSERT(speculative_inlining_error_->IsLanguageError());
  ASSERT(!background_compilation_error_->IsSmi());
  ASSERT(background_compilation_error_->IsLanguageError());
  ASSERT(!out_of_memory_error_->IsSmi());
  ASSERT(out_of_memory_error_->IsLanguageError());
  ASSERT(!vm_isolate_snapshot_object_table_->IsSmi());
  ASSERT(vm_isolate_snapshot_object_table_->IsArray());
  ASSERT(!extractor_parameter_types_->IsSmi());
  ASSERT(extractor_parameter_types_->IsArray());
  ASSERT(!extractor_parameter_names_->IsSmi());
  ASSERT(extractor_parameter_names_->IsArray());
  ASSERT(!implicit_getter_bytecode_->IsSmi());
  ASSERT(implicit_getter_bytecode_->IsBytecode());
  ASSERT(!implicit_setter_bytecode_->IsSmi());
  ASSERT(implicit_setter_bytecode_->IsBytecode());
  ASSERT(!implicit_static_getter_bytecode_->IsSmi());
  ASSERT(implicit_static_getter_bytecode_->IsBytecode());
  ASSERT(!method_extractor_bytecode_->IsSmi());
  ASSERT(method_extractor_bytecode_->IsBytecode());
  ASSERT(!invoke_closure_bytecode_->IsSmi());
  ASSERT(invoke_closure_bytecode_->IsBytecode());
  ASSERT(!invoke_field_bytecode_->IsSmi());
  ASSERT(invoke_field_bytecode_->IsBytecode());
  ASSERT(!nsm_dispatcher_bytecode_->IsSmi());
  ASSERT(nsm_dispatcher_bytecode_->IsBytecode());
  ASSERT(!dynamic_invocation_forwarder_bytecode_->IsSmi());
  ASSERT(dynamic_invocation_forwarder_bytecode_->IsBytecode());
}

void Object::FinishInit(Isolate* isolate) {
  // The type testing stubs we initialize in AbstractType objects for the
  // canonical type of kDynamicCid/kVoidCid need to be set in this
  // method, which is called after StubCode::InitOnce().
  Code& code = Code::Handle();

  code = TypeTestingStubGenerator::DefaultCodeForType(*dynamic_type_);
  dynamic_type_->SetTypeTestingStub(code);

  code = TypeTestingStubGenerator::DefaultCodeForType(*void_type_);
  void_type_->SetTypeTestingStub(code);
}

void Object::Cleanup() {
  null_ = static_cast<ObjectPtr>(RAW_NULL);
  true_ = static_cast<BoolPtr>(RAW_NULL);
  false_ = static_cast<BoolPtr>(RAW_NULL);
  class_class_ = static_cast<ClassPtr>(RAW_NULL);
  dynamic_class_ = static_cast<ClassPtr>(RAW_NULL);
  void_class_ = static_cast<ClassPtr>(RAW_NULL);
  type_arguments_class_ = static_cast<ClassPtr>(RAW_NULL);
  patch_class_class_ = static_cast<ClassPtr>(RAW_NULL);
  function_class_ = static_cast<ClassPtr>(RAW_NULL);
  closure_data_class_ = static_cast<ClassPtr>(RAW_NULL);
  signature_data_class_ = static_cast<ClassPtr>(RAW_NULL);
  redirection_data_class_ = static_cast<ClassPtr>(RAW_NULL);
  ffi_trampoline_data_class_ = static_cast<ClassPtr>(RAW_NULL);
  field_class_ = static_cast<ClassPtr>(RAW_NULL);
  script_class_ = static_cast<ClassPtr>(RAW_NULL);
  library_class_ = static_cast<ClassPtr>(RAW_NULL);
  namespace_class_ = static_cast<ClassPtr>(RAW_NULL);
  kernel_program_info_class_ = static_cast<ClassPtr>(RAW_NULL);
  code_class_ = static_cast<ClassPtr>(RAW_NULL);
  bytecode_class_ = static_cast<ClassPtr>(RAW_NULL);
  instructions_class_ = static_cast<ClassPtr>(RAW_NULL);
  instructions_section_class_ = static_cast<ClassPtr>(RAW_NULL);
  object_pool_class_ = static_cast<ClassPtr>(RAW_NULL);
  pc_descriptors_class_ = static_cast<ClassPtr>(RAW_NULL);
  code_source_map_class_ = static_cast<ClassPtr>(RAW_NULL);
  compressed_stackmaps_class_ = static_cast<ClassPtr>(RAW_NULL);
  var_descriptors_class_ = static_cast<ClassPtr>(RAW_NULL);
  exception_handlers_class_ = static_cast<ClassPtr>(RAW_NULL);
  context_class_ = static_cast<ClassPtr>(RAW_NULL);
  context_scope_class_ = static_cast<ClassPtr>(RAW_NULL);
  dyncalltypecheck_class_ = static_cast<ClassPtr>(RAW_NULL);
  singletargetcache_class_ = static_cast<ClassPtr>(RAW_NULL);
  unlinkedcall_class_ = static_cast<ClassPtr>(RAW_NULL);
  monomorphicsmiablecall_class_ = static_cast<ClassPtr>(RAW_NULL);
  icdata_class_ = static_cast<ClassPtr>(RAW_NULL);
  megamorphic_cache_class_ = static_cast<ClassPtr>(RAW_NULL);
  subtypetestcache_class_ = static_cast<ClassPtr>(RAW_NULL);
  loadingunit_class_ = static_cast<ClassPtr>(RAW_NULL);
  api_error_class_ = static_cast<ClassPtr>(RAW_NULL);
  language_error_class_ = static_cast<ClassPtr>(RAW_NULL);
  unhandled_exception_class_ = static_cast<ClassPtr>(RAW_NULL);
  unwind_error_class_ = static_cast<ClassPtr>(RAW_NULL);
}

void Object::FinalizeVMIsolate(Isolate* isolate) {
  // Should only be run by the vm isolate.
  ASSERT(isolate == Dart::vm_isolate());

  // Finish initialization of extractor_parameter_names_ which was
  // Started in Object::InitOnce()
  extractor_parameter_names_->SetAt(0, Symbols::This());

  // Set up names for all VM singleton classes.
  Class& cls = Class::Handle();

  SET_CLASS_NAME(class, Class);
  SET_CLASS_NAME(dynamic, Dynamic);
  SET_CLASS_NAME(void, Void);
  SET_CLASS_NAME(type_arguments, TypeArguments);
  SET_CLASS_NAME(patch_class, PatchClass);
  SET_CLASS_NAME(function, Function);
  SET_CLASS_NAME(closure_data, ClosureData);
  SET_CLASS_NAME(signature_data, SignatureData);
  SET_CLASS_NAME(redirection_data, RedirectionData);
  SET_CLASS_NAME(ffi_trampoline_data, FfiTrampolineData);
  SET_CLASS_NAME(field, Field);
  SET_CLASS_NAME(script, Script);
  SET_CLASS_NAME(library, LibraryClass);
  SET_CLASS_NAME(namespace, Namespace);
  SET_CLASS_NAME(kernel_program_info, KernelProgramInfo);
  SET_CLASS_NAME(code, Code);
  SET_CLASS_NAME(bytecode, Bytecode);
  SET_CLASS_NAME(instructions, Instructions);
  SET_CLASS_NAME(instructions_section, InstructionsSection);
  SET_CLASS_NAME(object_pool, ObjectPool);
  SET_CLASS_NAME(code_source_map, CodeSourceMap);
  SET_CLASS_NAME(pc_descriptors, PcDescriptors);
  SET_CLASS_NAME(compressed_stackmaps, CompressedStackMaps);
  SET_CLASS_NAME(var_descriptors, LocalVarDescriptors);
  SET_CLASS_NAME(exception_handlers, ExceptionHandlers);
  SET_CLASS_NAME(context, Context);
  SET_CLASS_NAME(context_scope, ContextScope);
  SET_CLASS_NAME(dyncalltypecheck, ParameterTypeCheck);
  SET_CLASS_NAME(singletargetcache, SingleTargetCache);
  SET_CLASS_NAME(unlinkedcall, UnlinkedCall);
  SET_CLASS_NAME(monomorphicsmiablecall, MonomorphicSmiableCall);
  SET_CLASS_NAME(icdata, ICData);
  SET_CLASS_NAME(megamorphic_cache, MegamorphicCache);
  SET_CLASS_NAME(subtypetestcache, SubtypeTestCache);
  SET_CLASS_NAME(loadingunit, LoadingUnit);
  SET_CLASS_NAME(api_error, ApiError);
  SET_CLASS_NAME(language_error, LanguageError);
  SET_CLASS_NAME(unhandled_exception, UnhandledException);
  SET_CLASS_NAME(unwind_error, UnwindError);

  // Set up names for classes which are also pre-allocated in the vm isolate.
  cls = isolate->object_store()->array_class();
  cls.set_name(Symbols::_List());
  cls = isolate->object_store()->one_byte_string_class();
  cls.set_name(Symbols::OneByteString());
  cls = isolate->object_store()->never_class();
  cls.set_name(Symbols::Never());

  // Set up names for the pseudo-classes for free list elements and forwarding
  // corpses. Mainly this makes VM debugging easier.
  cls = isolate->class_table()->At(kFreeListElement);
  cls.set_name(Symbols::FreeListElement());
  cls = isolate->class_table()->At(kForwardingCorpse);
  cls.set_name(Symbols::ForwardingCorpse());

  {
    ASSERT(isolate == Dart::vm_isolate());
    Thread* thread = Thread::Current();
    WritableVMIsolateScope scope(thread);
    HeapIterationScope iteration(thread);
    FinalizeVMIsolateVisitor premarker;
    ASSERT(isolate->heap()->UsedInWords(Heap::kNew) == 0);
    iteration.IterateOldObjectsNoImagePages(&premarker);
    // Make the VM isolate read-only again after setting all objects as marked.
    // Note objects in image pages are already pre-marked.
  }
}

void Object::FinalizeReadOnlyObject(ObjectPtr object) {
  NoSafepointScope no_safepoint;
  intptr_t cid = object->GetClassId();
  if (cid == kOneByteStringCid) {
    OneByteStringPtr str = static_cast<OneByteStringPtr>(object);
    if (String::GetCachedHash(str) == 0) {
      intptr_t hash = String::Hash(str);
      String::SetCachedHash(str, hash);
    }
    intptr_t size = OneByteString::UnroundedSize(str);
    ASSERT(size <= str->ptr()->HeapSize());
    memset(reinterpret_cast<void*>(ObjectLayout::ToAddr(str) + size), 0,
           str->ptr()->HeapSize() - size);
  } else if (cid == kTwoByteStringCid) {
    TwoByteStringPtr str = static_cast<TwoByteStringPtr>(object);
    if (String::GetCachedHash(str) == 0) {
      intptr_t hash = String::Hash(str);
      String::SetCachedHash(str, hash);
    }
    ASSERT(String::GetCachedHash(str) != 0);
    intptr_t size = TwoByteString::UnroundedSize(str);
    ASSERT(size <= str->ptr()->HeapSize());
    memset(reinterpret_cast<void*>(ObjectLayout::ToAddr(str) + size), 0,
           str->ptr()->HeapSize() - size);
  } else if (cid == kExternalOneByteStringCid) {
    ExternalOneByteStringPtr str =
        static_cast<ExternalOneByteStringPtr>(object);
    if (String::GetCachedHash(str) == 0) {
      intptr_t hash = String::Hash(str);
      String::SetCachedHash(str, hash);
    }
  } else if (cid == kExternalTwoByteStringCid) {
    ExternalTwoByteStringPtr str =
        static_cast<ExternalTwoByteStringPtr>(object);
    if (String::GetCachedHash(str) == 0) {
      intptr_t hash = String::Hash(str);
      String::SetCachedHash(str, hash);
    }
  } else if (cid == kCodeSourceMapCid) {
    CodeSourceMapPtr map = CodeSourceMap::RawCast(object);
    intptr_t size = CodeSourceMap::UnroundedSize(map);
    ASSERT(size <= map->ptr()->HeapSize());
    memset(reinterpret_cast<void*>(ObjectLayout::ToAddr(map) + size), 0,
           map->ptr()->HeapSize() - size);
  } else if (cid == kCompressedStackMapsCid) {
    CompressedStackMapsPtr maps = CompressedStackMaps::RawCast(object);
    intptr_t size = CompressedStackMaps::UnroundedSize(maps);
    ASSERT(size <= maps->ptr()->HeapSize());
    memset(reinterpret_cast<void*>(ObjectLayout::ToAddr(maps) + size), 0,
           maps->ptr()->HeapSize() - size);
  } else if (cid == kPcDescriptorsCid) {
    PcDescriptorsPtr desc = PcDescriptors::RawCast(object);
    intptr_t size = PcDescriptors::UnroundedSize(desc);
    ASSERT(size <= desc->ptr()->HeapSize());
    memset(reinterpret_cast<void*>(ObjectLayout::ToAddr(desc) + size), 0,
           desc->ptr()->HeapSize() - size);
  }
}

void Object::set_vm_isolate_snapshot_object_table(const Array& table) {
  ASSERT(Isolate::Current() == Dart::vm_isolate());
  *vm_isolate_snapshot_object_table_ = table.raw();
}

// Make unused space in an object whose type has been transformed safe
// for traversing during GC.
// The unused part of the transformed object is marked as an TypedDataInt8Array
// object.
void Object::MakeUnusedSpaceTraversable(const Object& obj,
                                        intptr_t original_size,
                                        intptr_t used_size) {
  ASSERT(Thread::Current()->no_safepoint_scope_depth() > 0);
  ASSERT(!obj.IsNull());
  ASSERT(original_size >= used_size);
  if (original_size > used_size) {
    intptr_t leftover_size = original_size - used_size;

    uword addr = ObjectLayout::ToAddr(obj.raw()) + used_size;
    if (leftover_size >= TypedData::InstanceSize(0)) {
      // Update the leftover space as a TypedDataInt8Array object.
      TypedDataPtr raw =
          static_cast<TypedDataPtr>(ObjectLayout::FromAddr(addr));
      uword new_tags =
          ObjectLayout::ClassIdTag::update(kTypedDataInt8ArrayCid, 0);
      new_tags = ObjectLayout::SizeTag::update(leftover_size, new_tags);
      const bool is_old = obj.raw()->IsOldObject();
      new_tags = ObjectLayout::OldBit::update(is_old, new_tags);
      new_tags = ObjectLayout::OldAndNotMarkedBit::update(is_old, new_tags);
      new_tags = ObjectLayout::OldAndNotRememberedBit::update(is_old, new_tags);
      new_tags = ObjectLayout::NewBit::update(!is_old, new_tags);
      // On architectures with a relaxed memory model, the concurrent marker may
      // observe the write of the filler object's header before observing the
      // new array length, and so treat it as a pointer. Ensure it is a Smi so
      // the marker won't dereference it.
      ASSERT((new_tags & kSmiTagMask) == kSmiTag);
      uint32_t tags = raw->ptr()->tags_;
      uint32_t old_tags;
      // TODO(iposva): Investigate whether CompareAndSwapWord is necessary.
      do {
        old_tags = tags;
        // We can't use obj.CompareAndSwapTags here because we don't have a
        // handle for the new object.
      } while (!raw->ptr()->tags_.WeakCAS(old_tags, new_tags));

      intptr_t leftover_len = (leftover_size - TypedData::InstanceSize(0));
      ASSERT(TypedData::InstanceSize(leftover_len) == leftover_size);
      raw->ptr()->StoreSmi(&(raw->ptr()->length_), Smi::New(leftover_len));
      raw->ptr()->RecomputeDataField();
    } else {
      // Update the leftover space as a basic object.
      ASSERT(leftover_size == Object::InstanceSize());
      ObjectPtr raw = static_cast<ObjectPtr>(ObjectLayout::FromAddr(addr));
      uword new_tags = ObjectLayout::ClassIdTag::update(kInstanceCid, 0);
      new_tags = ObjectLayout::SizeTag::update(leftover_size, new_tags);
      const bool is_old = obj.raw()->IsOldObject();
      new_tags = ObjectLayout::OldBit::update(is_old, new_tags);
      new_tags = ObjectLayout::OldAndNotMarkedBit::update(is_old, new_tags);
      new_tags = ObjectLayout::OldAndNotRememberedBit::update(is_old, new_tags);
      new_tags = ObjectLayout::NewBit::update(!is_old, new_tags);
      // On architectures with a relaxed memory model, the concurrent marker may
      // observe the write of the filler object's header before observing the
      // new array length, and so treat it as a pointer. Ensure it is a Smi so
      // the marker won't dereference it.
      ASSERT((new_tags & kSmiTagMask) == kSmiTag);
      uint32_t tags = raw->ptr()->tags_;
      uint32_t old_tags;
      // TODO(iposva): Investigate whether CompareAndSwapWord is necessary.
      do {
        old_tags = tags;
        // We can't use obj.CompareAndSwapTags here because we don't have a
        // handle for the new object.
      } while (!raw->ptr()->tags_.WeakCAS(old_tags, new_tags));
    }
  }
}

void Object::VerifyBuiltinVtables() {
#if defined(DEBUG)
  ASSERT(builtin_vtables_[kIllegalCid] == 0);
  ASSERT(builtin_vtables_[kFreeListElement] == 0);
  ASSERT(builtin_vtables_[kForwardingCorpse] == 0);
  ClassTable* table = Isolate::Current()->class_table();
  for (intptr_t cid = kObjectCid; cid < kNumPredefinedCids; cid++) {
    if (table->HasValidClassAt(cid)) {
      ASSERT(builtin_vtables_[cid] != 0);
    }
  }
#endif
}

void Object::RegisterClass(const Class& cls,
                           const String& name,
                           const Library& lib) {
  ASSERT(name.Length() > 0);
  ASSERT(name.CharAt(0) != '_');
  cls.set_name(name);
  lib.AddClass(cls);
}

void Object::RegisterPrivateClass(const Class& cls,
                                  const String& public_class_name,
                                  const Library& lib) {
  ASSERT(public_class_name.Length() > 0);
  ASSERT(public_class_name.CharAt(0) == '_');
  String& str = String::Handle();
  str = lib.PrivateName(public_class_name);
  cls.set_name(str);
  lib.AddClass(cls);
}


// line 1609-
// Initialize a new isolate from source or from a snapshot.
//
// There are three possibilities:
//   1. Running a Kernel binary.  This function will bootstrap from the KERNEL
//      file.
//   2. There is no vm snapshot.  This function will bootstrap from source.
//   3. There is a vm snapshot.  The caller should initialize from the snapshot.
//
// A non-NULL kernel argument indicates (1).
// A NULL kernel indicates (2) or (3).
ErrorPtr Object::Init(Isolate* isolate,
                      const uint8_t* kernel_buffer,
                      intptr_t kernel_buffer_size) {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  ASSERT(isolate == thread->isolate());
  TIMELINE_DURATION(thread, Isolate, "Object::Init");

#if defined(DART_PRECOMPILED_RUNTIME)
  const bool bootstrapping = false;
#else
  const bool is_kernel = (kernel_buffer != NULL);
  const bool bootstrapping =
      (Dart::vm_snapshot_kind() == Snapshot::kNone) || is_kernel;
#endif  // defined(DART_PRECOMPILED_RUNTIME).

  if (bootstrapping) {
#if !defined(DART_PRECOMPILED_RUNTIME)
    // Object::Init version when we are bootstrapping from source or from a
    // Kernel binary.
    // This will initialize isolate group object_store, shared by all isolates
    // running in the isolate group.
    ObjectStore* object_store = isolate->object_store();

    Class& cls = Class::Handle(zone);
    Type& type = Type::Handle(zone);
    Array& array = Array::Handle(zone);
    Library& lib = Library::Handle(zone);
    TypeArguments& type_args = TypeArguments::Handle(zone);

    // All RawArray fields will be initialized to an empty array, therefore
    // initialize array class first.
    cls = Class::New<Array, RTN::Array>(isolate);
    ASSERT(object_store->array_class() == Class::null());
    object_store->set_array_class(cls);

    // VM classes that are parameterized (Array, ImmutableArray,
    // GrowableObjectArray, and LinkedHashMap) are also pre-finalized, so
    // CalculateFieldOffsets() is not called, so we need to set the offset of
    // their type_arguments_ field, which is explicitly declared in their
    // respective Raw* classes.
    cls.set_type_arguments_field_offset(Array::type_arguments_offset(),
                                        RTN::Array::type_arguments_offset());
    cls.set_num_type_arguments(1);

    // Set up the growable object array class (Has to be done after the array
    // class is setup as one of its field is an array object).
    cls = Class::New<GrowableObjectArray, RTN::GrowableObjectArray>(isolate);
    object_store->set_growable_object_array_class(cls);
    cls.set_type_arguments_field_offset(
        GrowableObjectArray::type_arguments_offset(),
        RTN::GrowableObjectArray::type_arguments_offset());
    cls.set_num_type_arguments(1);

    // Initialize hash set for canonical types.
    const intptr_t kInitialCanonicalTypeSize = 16;
    array = HashTables::New<CanonicalTypeSet>(kInitialCanonicalTypeSize,
                                              Heap::kOld);
    object_store->set_canonical_types(array);

    // Initialize hash set for canonical type parameters.
    const intptr_t kInitialCanonicalTypeParameterSize = 4;
    array = HashTables::New<CanonicalTypeParameterSet>(
        kInitialCanonicalTypeParameterSize, Heap::kOld);
    object_store->set_canonical_type_parameters(array);

    // Initialize hash set for canonical_type_arguments_.
    const intptr_t kInitialCanonicalTypeArgumentsSize = 4;
    array = HashTables::New<CanonicalTypeArgumentsSet>(
        kInitialCanonicalTypeArgumentsSize, Heap::kOld);
    object_store->set_canonical_type_arguments(array);

    // Setup type class early in the process.
    const Class& type_cls =
        Class::Handle(zone, Class::New<Type, RTN::Type>(isolate));
    const Class& type_ref_cls =
        Class::Handle(zone, Class::New<TypeRef, RTN::TypeRef>(isolate));
    const Class& type_parameter_cls = Class::Handle(
        zone, Class::New<TypeParameter, RTN::TypeParameter>(isolate));
    const Class& library_prefix_cls = Class::Handle(
        zone, Class::New<LibraryPrefix, RTN::LibraryPrefix>(isolate));

    // Pre-allocate the OneByteString class needed by the symbol table.
    cls = Class::NewStringClass(kOneByteStringCid, isolate);
    object_store->set_one_byte_string_class(cls);

    // Pre-allocate the TwoByteString class needed by the symbol table.
    cls = Class::NewStringClass(kTwoByteStringCid, isolate);
    object_store->set_two_byte_string_class(cls);

    // Setup the symbol table for the symbols created in the isolate.
    Symbols::SetupSymbolTable(isolate);

    // Set up the libraries array before initializing the core library.
    const GrowableObjectArray& libraries =
        GrowableObjectArray::Handle(zone, GrowableObjectArray::New(Heap::kOld));
    object_store->set_libraries(libraries);

    // Pre-register the core library.
    Library::InitCoreLibrary(isolate);

    // Basic infrastructure has been setup, initialize the class dictionary.
    const Library& core_lib = Library::Handle(zone, Library::CoreLibrary());
    ASSERT(!core_lib.IsNull());

    const GrowableObjectArray& pending_classes =
        GrowableObjectArray::Handle(zone, GrowableObjectArray::New());
    object_store->set_pending_classes(pending_classes);

    // Now that the symbol table is initialized and that the core dictionary as
    // well as the core implementation dictionary have been setup, preallocate
    // remaining classes and register them by name in the dictionaries.
    String& name = String::Handle(zone);
    cls = object_store->array_class();  // Was allocated above.
    RegisterPrivateClass(cls, Symbols::_List(), core_lib);
    pending_classes.Add(cls);
    // We cannot use NewNonParameterizedType(), because Array is
    // parameterized.  Warning: class _List has not been patched yet. Its
    // declared number of type parameters is still 0. It will become 1 after
    // patching. The array type allocated below represents the raw type _List
    // and not _List<E> as we could expect. Use with caution.
    type =
        Type::New(Class::Handle(zone, cls.raw()), TypeArguments::Handle(zone),
                  TokenPosition::kNoSource, Nullability::kNonNullable);
    type.SetIsFinalized();
    type ^= type.Canonicalize(thread, nullptr);
    object_store->set_array_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_array_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_array_type(type);

    cls = object_store->growable_object_array_class();  // Was allocated above.
    RegisterPrivateClass(cls, Symbols::_GrowableList(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Array, RTN::Array>(kImmutableArrayCid, isolate);
    object_store->set_immutable_array_class(cls);
    cls.set_type_arguments_field_offset(Array::type_arguments_offset(),
                                        RTN::Array::type_arguments_offset());
    cls.set_num_type_arguments(1);
    ASSERT(object_store->immutable_array_class() !=
           object_store->array_class());
    cls.set_is_prefinalized();
    RegisterPrivateClass(cls, Symbols::_ImmutableList(), core_lib);
    pending_classes.Add(cls);

    cls = object_store->one_byte_string_class();  // Was allocated above.
    RegisterPrivateClass(cls, Symbols::OneByteString(), core_lib);
    pending_classes.Add(cls);

    cls = object_store->two_byte_string_class();  // Was allocated above.
    RegisterPrivateClass(cls, Symbols::TwoByteString(), core_lib);
    pending_classes.Add(cls);

    cls = Class::NewStringClass(kExternalOneByteStringCid, isolate);
    object_store->set_external_one_byte_string_class(cls);
    RegisterPrivateClass(cls, Symbols::ExternalOneByteString(), core_lib);
    pending_classes.Add(cls);

    cls = Class::NewStringClass(kExternalTwoByteStringCid, isolate);
    object_store->set_external_two_byte_string_class(cls);
    RegisterPrivateClass(cls, Symbols::ExternalTwoByteString(), core_lib);
    pending_classes.Add(cls);

    // Pre-register the isolate library so the native class implementations can
    // be hooked up before compiling it.
    Library& isolate_lib = Library::Handle(
        zone, Library::LookupLibrary(thread, Symbols::DartIsolate()));
    if (isolate_lib.IsNull()) {
      isolate_lib = Library::NewLibraryHelper(Symbols::DartIsolate(), true);
      isolate_lib.SetLoadRequested();
      isolate_lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kIsolate, isolate_lib);
    ASSERT(!isolate_lib.IsNull());
    ASSERT(isolate_lib.raw() == Library::IsolateLibrary());

    cls = Class::New<Capability, RTN::Capability>(isolate);
    RegisterPrivateClass(cls, Symbols::_CapabilityImpl(), isolate_lib);
    pending_classes.Add(cls);

    cls = Class::New<ReceivePort, RTN::ReceivePort>(isolate);
    RegisterPrivateClass(cls, Symbols::_RawReceivePortImpl(), isolate_lib);
    pending_classes.Add(cls);

    cls = Class::New<SendPort, RTN::SendPort>(isolate);
    RegisterPrivateClass(cls, Symbols::_SendPortImpl(), isolate_lib);
    pending_classes.Add(cls);

    cls =
        Class::New<TransferableTypedData, RTN::TransferableTypedData>(isolate);
    RegisterPrivateClass(cls, Symbols::_TransferableTypedDataImpl(),
                         isolate_lib);
    pending_classes.Add(cls);

    const Class& stacktrace_cls =
        Class::Handle(zone, Class::New<StackTrace, RTN::StackTrace>(isolate));
    RegisterPrivateClass(stacktrace_cls, Symbols::_StackTrace(), core_lib);
    pending_classes.Add(stacktrace_cls);
    // Super type set below, after Object is allocated.

    cls = Class::New<RegExp, RTN::RegExp>(isolate);
    RegisterPrivateClass(cls, Symbols::_RegExp(), core_lib);
    pending_classes.Add(cls);

    // Initialize the base interfaces used by the core VM classes.

    // Allocate and initialize the pre-allocated classes in the core library.
    // The script and token index of these pre-allocated classes is set up in
    // the parser when the corelib script is compiled (see
    // Parser::ParseClassDefinition).
    cls = Class::New<Instance, RTN::Instance>(kInstanceCid, isolate);
    object_store->set_object_class(cls);
    cls.set_name(Symbols::Object());
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    cls.set_is_const();
    core_lib.AddClass(cls);
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_object_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_object_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_object_type(type);
    type = type.ToNullability(Nullability::kNullable, Heap::kOld);
    object_store->set_nullable_object_type(type);

    cls = Class::New<Bool, RTN::Bool>(isolate);
    object_store->set_bool_class(cls);
    RegisterClass(cls, Symbols::Bool(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Instance, RTN::Instance>(kNullCid, isolate);
    object_store->set_null_class(cls);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    RegisterClass(cls, Symbols::Null(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Instance, RTN::Instance>(kNeverCid, isolate);
    cls.set_num_type_arguments(0);
    cls.set_is_allocate_finalized();
    cls.set_is_declaration_loaded();
    cls.set_is_type_finalized();
    cls.set_name(Symbols::Never());
    object_store->set_never_class(cls);

    ASSERT(!library_prefix_cls.IsNull());
    RegisterPrivateClass(library_prefix_cls, Symbols::_LibraryPrefix(),
                         core_lib);
    pending_classes.Add(library_prefix_cls);

    RegisterPrivateClass(type_cls, Symbols::_Type(), core_lib);
    pending_classes.Add(type_cls);

    RegisterPrivateClass(type_ref_cls, Symbols::_TypeRef(), core_lib);
    pending_classes.Add(type_ref_cls);

    RegisterPrivateClass(type_parameter_cls, Symbols::_TypeParameter(),
                         core_lib);
    pending_classes.Add(type_parameter_cls);

    cls = Class::New<Integer, RTN::Integer>(isolate);
    object_store->set_integer_implementation_class(cls);
    RegisterPrivateClass(cls, Symbols::_IntegerImplementation(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Smi, RTN::Smi>(isolate);
    object_store->set_smi_class(cls);
    RegisterPrivateClass(cls, Symbols::_Smi(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Mint, RTN::Mint>(isolate);
    object_store->set_mint_class(cls);
    RegisterPrivateClass(cls, Symbols::_Mint(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<Double, RTN::Double>(isolate);
    object_store->set_double_class(cls);
    RegisterPrivateClass(cls, Symbols::_Double(), core_lib);
    pending_classes.Add(cls);

    // Class that represents the Dart class _Closure and C++ class Closure.
    cls = Class::New<Closure, RTN::Closure>(isolate);
    object_store->set_closure_class(cls);
    RegisterPrivateClass(cls, Symbols::_Closure(), core_lib);
    pending_classes.Add(cls);

    cls = Class::New<WeakProperty, RTN::WeakProperty>(isolate);
    object_store->set_weak_property_class(cls);
    RegisterPrivateClass(cls, Symbols::_WeakProperty(), core_lib);

// Pre-register the mirrors library so we can place the vm class
// MirrorReference there rather than the core library.
#if !defined(DART_PRECOMPILED_RUNTIME)
    lib = Library::LookupLibrary(thread, Symbols::DartMirrors());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartMirrors(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kMirrors, lib);
    ASSERT(!lib.IsNull());
    ASSERT(lib.raw() == Library::MirrorsLibrary());

    cls = Class::New<MirrorReference, RTN::MirrorReference>(isolate);
    RegisterPrivateClass(cls, Symbols::_MirrorReference(), lib);
#endif

    // Pre-register the collection library so we can place the vm class
    // LinkedHashMap there rather than the core library.
    lib = Library::LookupLibrary(thread, Symbols::DartCollection());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartCollection(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }

    object_store->set_bootstrap_library(ObjectStore::kCollection, lib);
    ASSERT(!lib.IsNull());
    ASSERT(lib.raw() == Library::CollectionLibrary());
    cls = Class::New<LinkedHashMap, RTN::LinkedHashMap>(isolate);
    object_store->set_linked_hash_map_class(cls);
    cls.set_type_arguments_field_offset(
        LinkedHashMap::type_arguments_offset(),
        RTN::LinkedHashMap::type_arguments_offset());
    cls.set_num_type_arguments(2);
    RegisterPrivateClass(cls, Symbols::_LinkedHashMap(), lib);
    pending_classes.Add(cls);

    // Pre-register the async library so we can place the vm class
    // FutureOr there rather than the core library.
    lib = Library::LookupLibrary(thread, Symbols::DartAsync());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartAsync(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kAsync, lib);
    ASSERT(!lib.IsNull());
    ASSERT(lib.raw() == Library::AsyncLibrary());
    cls = Class::New<FutureOr, RTN::FutureOr>(isolate);
    cls.set_type_arguments_field_offset(FutureOr::type_arguments_offset(),
                                        RTN::FutureOr::type_arguments_offset());
    cls.set_num_type_arguments(1);
    RegisterClass(cls, Symbols::FutureOr(), lib);
    pending_classes.Add(cls);

    // Pre-register the developer library so we can place the vm class
    // UserTag there rather than the core library.
    lib = Library::LookupLibrary(thread, Symbols::DartDeveloper());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartDeveloper(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kDeveloper, lib);
    ASSERT(!lib.IsNull());
    ASSERT(lib.raw() == Library::DeveloperLibrary());
    cls = Class::New<UserTag, RTN::UserTag>(isolate);
    RegisterPrivateClass(cls, Symbols::_UserTag(), lib);
    pending_classes.Add(cls);

    // Setup some default native field classes which can be extended for
    // specifying native fields in dart classes.
    Library::InitNativeWrappersLibrary(isolate, is_kernel);
    ASSERT(object_store->native_wrappers_library() != Library::null());

    // Pre-register the typed_data library so the native class implementations
    // can be hooked up before compiling it.
    lib = Library::LookupLibrary(thread, Symbols::DartTypedData());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartTypedData(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kTypedData, lib);
    ASSERT(!lib.IsNull());
    ASSERT(lib.raw() == Library::TypedDataLibrary());
#define REGISTER_TYPED_DATA_CLASS(clazz)                                       \
  cls = Class::NewTypedDataClass(kTypedData##clazz##ArrayCid, isolate);        \
  RegisterPrivateClass(cls, Symbols::_##clazz##List(), lib);

    DART_CLASS_LIST_TYPED_DATA(REGISTER_TYPED_DATA_CLASS);
#undef REGISTER_TYPED_DATA_CLASS
#define REGISTER_TYPED_DATA_VIEW_CLASS(clazz)                                  \
  cls = Class::NewTypedDataViewClass(kTypedData##clazz##ViewCid, isolate);     \
  RegisterPrivateClass(cls, Symbols::_##clazz##View(), lib);                   \
  pending_classes.Add(cls);

    CLASS_LIST_TYPED_DATA(REGISTER_TYPED_DATA_VIEW_CLASS);

    cls = Class::NewTypedDataViewClass(kByteDataViewCid, isolate);
    RegisterPrivateClass(cls, Symbols::_ByteDataView(), lib);
    pending_classes.Add(cls);

#undef REGISTER_TYPED_DATA_VIEW_CLASS
#define REGISTER_EXT_TYPED_DATA_CLASS(clazz)                                   \
  cls = Class::NewExternalTypedDataClass(kExternalTypedData##clazz##Cid,       \
                                         isolate);                             \
  RegisterPrivateClass(cls, Symbols::_External##clazz(), lib);

    cls = Class::New<Instance, RTN::Instance>(kByteBufferCid, isolate,
                                              /*register_class=*/false);
    cls.set_instance_size(0, 0);
    cls.set_next_field_offset(-kWordSize, -compiler::target::kWordSize);
    isolate->class_table()->Register(cls);
    RegisterPrivateClass(cls, Symbols::_ByteBuffer(), lib);
    pending_classes.Add(cls);

    CLASS_LIST_TYPED_DATA(REGISTER_EXT_TYPED_DATA_CLASS);
#undef REGISTER_EXT_TYPED_DATA_CLASS
    // Register Float32x4, Int32x4, and Float64x2 in the object store.
    cls = Class::New<Float32x4, RTN::Float32x4>(isolate);
    RegisterPrivateClass(cls, Symbols::_Float32x4(), lib);
    pending_classes.Add(cls);
    object_store->set_float32x4_class(cls);

    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, Symbols::Float32x4(), lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_float32x4_type(type);

    cls = Class::New<Int32x4, RTN::Int32x4>(isolate);
    RegisterPrivateClass(cls, Symbols::_Int32x4(), lib);
    pending_classes.Add(cls);
    object_store->set_int32x4_class(cls);

    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, Symbols::Int32x4(), lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_int32x4_type(type);

    cls = Class::New<Float64x2, RTN::Float64x2>(isolate);
    RegisterPrivateClass(cls, Symbols::_Float64x2(), lib);
    pending_classes.Add(cls);
    object_store->set_float64x2_class(cls);

    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, Symbols::Float64x2(), lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_float64x2_type(type);

    // Set the super type of class StackTrace to Object type so that the
    // 'toString' method is implemented.
    type = object_store->object_type();
    stacktrace_cls.set_super_type(type);

    // Abstract class that represents the Dart class Type.
    // Note that this class is implemented by Dart class _AbstractType.
    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    RegisterClass(cls, Symbols::Type(), core_lib);
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_type_type(type);

    // Abstract class that represents the Dart class Function.
    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    RegisterClass(cls, Symbols::Function(), core_lib);
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_function_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_function_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_function_type(type);

    cls = Class::New<Number, RTN::Number>(isolate);
    RegisterClass(cls, Symbols::Number(), core_lib);
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_number_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_number_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_number_type(type);

    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, Symbols::Int(), core_lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_int_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_int_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_int_type(type);
    type = type.ToNullability(Nullability::kNullable, Heap::kOld);
    object_store->set_nullable_int_type(type);

    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, Symbols::Double(), core_lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_double_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_double_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_double_type(type);
    type = type.ToNullability(Nullability::kNullable, Heap::kOld);
    object_store->set_nullable_double_type(type);

    name = Symbols::_String().raw();
    cls = Class::New<Instance, RTN::Instance>(kIllegalCid, isolate,
                                              /*register_class=*/true,
                                              /*is_abstract=*/true);
    RegisterClass(cls, name, core_lib);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    type = Type::NewNonParameterizedType(cls);
    object_store->set_string_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_string_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_string_type(type);

    cls = object_store->bool_class();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_bool_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_bool_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_bool_type(type);

    cls = object_store->smi_class();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_smi_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_smi_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_smi_type(type);

    cls = object_store->mint_class();
    type = Type::NewNonParameterizedType(cls);
    object_store->set_mint_type(type);
    type = type.ToNullability(Nullability::kLegacy, Heap::kOld);
    object_store->set_legacy_mint_type(type);
    type = type.ToNullability(Nullability::kNonNullable, Heap::kOld);
    object_store->set_non_nullable_mint_type(type);

    // The classes 'void' and 'dynamic' are phony classes to make type checking
    // more regular; they live in the VM isolate. The class 'void' is not
    // registered in the class dictionary because its name is a reserved word.
    // The class 'dynamic' is registered in the class dictionary because its
    // name is a built-in identifier (this is wrong).  The corresponding types
    // are stored in the object store.
    cls = object_store->null_class();
    type = Type::New(cls, Object::null_type_arguments(),
                     TokenPosition::kNoSource, Nullability::kNullable);
    type.SetIsFinalized();
    type ^= type.Canonicalize(thread, nullptr);
    object_store->set_null_type(type);
    ASSERT(type.IsNullable());

    // Consider removing when/if Null becomes an ordinary class.
    type = object_store->object_type();
    cls.set_super_type(type);

    cls = object_store->never_class();
    type = Type::New(cls, Object::null_type_arguments(),
                     TokenPosition::kNoSource, Nullability::kNonNullable);
    type.SetIsFinalized();
    type ^= type.Canonicalize(thread, nullptr);
    object_store->set_never_type(type);

    // Create and cache commonly used type arguments <int>, <double>,
    // <String>, <String, dynamic> and <String, String>.
    type_args = TypeArguments::New(1);
    type = object_store->int_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_int(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->legacy_int_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_legacy_int(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->non_nullable_int_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_non_nullable_int(type_args);

    type_args = TypeArguments::New(1);
    type = object_store->double_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_double(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->legacy_double_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_legacy_double(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->non_nullable_double_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_non_nullable_double(type_args);

    type_args = TypeArguments::New(1);
    type = object_store->string_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_string(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->legacy_string_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_legacy_string(type_args);
    type_args = TypeArguments::New(1);
    type = object_store->non_nullable_string_type();
    type_args.SetTypeAt(0, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_non_nullable_string(type_args);

    type_args = TypeArguments::New(2);
    type = object_store->string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, Object::dynamic_type());
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_string_dynamic(type_args);
    type_args = TypeArguments::New(2);
    type = object_store->legacy_string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, Object::dynamic_type());
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_legacy_string_dynamic(type_args);
    type_args = TypeArguments::New(2);
    type = object_store->non_nullable_string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, Object::dynamic_type());
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_non_nullable_string_dynamic(type_args);

    type_args = TypeArguments::New(2);
    type = object_store->string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_string_string(type_args);
    type_args = TypeArguments::New(2);
    type = object_store->legacy_string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_legacy_string_legacy_string(type_args);
    type_args = TypeArguments::New(2);
    type = object_store->non_nullable_string_type();
    type_args.SetTypeAt(0, type);
    type_args.SetTypeAt(1, type);
    type_args = type_args.Canonicalize(thread, nullptr);
    object_store->set_type_argument_non_nullable_string_non_nullable_string(
        type_args);

    lib = Library::LookupLibrary(thread, Symbols::DartFfi());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartFfi(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kFfi, lib);

    cls = Class::New<Instance, RTN::Instance>(kFfiNativeTypeCid, isolate);
    cls.set_num_type_arguments(0);
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    object_store->set_ffi_native_type_class(cls);
    RegisterClass(cls, Symbols::FfiNativeType(), lib);

#define REGISTER_FFI_TYPE_MARKER(clazz)                                        \
  cls = Class::New<Instance, RTN::Instance>(kFfi##clazz##Cid, isolate);        \
  cls.set_num_type_arguments(0);                                               \
  cls.set_is_prefinalized();                                                   \
  pending_classes.Add(cls);                                                    \
  RegisterClass(cls, Symbols::Ffi##clazz(), lib);
    CLASS_LIST_FFI_TYPE_MARKER(REGISTER_FFI_TYPE_MARKER);
#undef REGISTER_FFI_TYPE_MARKER

    cls = Class::New<Instance, RTN::Instance>(kFfiNativeFunctionCid, isolate);
    cls.set_type_arguments_field_offset(Pointer::type_arguments_offset(),
                                        RTN::Pointer::type_arguments_offset());
    cls.set_num_type_arguments(1);
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    RegisterClass(cls, Symbols::FfiNativeFunction(), lib);

    cls = Class::NewPointerClass(kFfiPointerCid, isolate);
    object_store->set_ffi_pointer_class(cls);
    pending_classes.Add(cls);
    RegisterClass(cls, Symbols::FfiPointer(), lib);

    cls = Class::New<DynamicLibrary, RTN::DynamicLibrary>(kFfiDynamicLibraryCid,
                                                          isolate);
    cls.set_instance_size(DynamicLibrary::InstanceSize(),
                          compiler::target::RoundedAllocationSize(
                              RTN::DynamicLibrary::InstanceSize()));
    cls.set_is_prefinalized();
    pending_classes.Add(cls);
    RegisterClass(cls, Symbols::FfiDynamicLibrary(), lib);

    lib = Library::LookupLibrary(thread, Symbols::DartWasm());
    if (lib.IsNull()) {
      lib = Library::NewLibraryHelper(Symbols::DartWasm(), true);
      lib.SetLoadRequested();
      lib.Register(thread);
    }
    object_store->set_bootstrap_library(ObjectStore::kWasm, lib);

#define REGISTER_WASM_TYPE(clazz)                                              \
  cls = Class::New<Instance, RTN::Instance>(k##clazz##Cid, isolate);           \
  cls.set_num_type_arguments(0);                                               \
  cls.set_is_prefinalized();                                                   \
  pending_classes.Add(cls);                                                    \
  RegisterClass(cls, Symbols::clazz(), lib);
    CLASS_LIST_WASM(REGISTER_WASM_TYPE);
#undef REGISTER_WASM_TYPE

    // Finish the initialization by compiling the bootstrap scripts containing
    // the base interfaces and the implementation of the internal classes.
    const Error& error = Error::Handle(
        zone, Bootstrap::DoBootstrapping(kernel_buffer, kernel_buffer_size));
    if (!error.IsNull()) {
      return error.raw();
    }

    isolate->class_table()->CopySizesFromClassObjects();

    ClassFinalizer::VerifyBootstrapClasses();

    // Set up the intrinsic state of all functions (core, math and typed data).
    compiler::Intrinsifier::InitializeState();

    // Set up recognized state of all functions (core, math and typed data).
    MethodRecognizer::InitializeState();

    // Adds static const fields (class ids) to the class 'ClassID');
    lib = Library::LookupLibrary(thread, Symbols::DartInternal());
    ASSERT(!lib.IsNull());
    cls = lib.LookupClassAllowPrivate(Symbols::ClassID());
    ASSERT(!cls.IsNull());
    const bool injected = cls.InjectCIDFields();
    ASSERT(injected);

    isolate->object_store()->InitKnownObjects();
#endif  // !defined(DART_PRECOMPILED_RUNTIME)
  } else {
    // Object::Init version when we are running in a version of dart that has a
    // full snapshot linked in and an isolate is initialized using the full
    // snapshot.
    ObjectStore* object_store = isolate->object_store();

    Class& cls = Class::Handle(zone);

    // Set up empty classes in the object store, these will get initialized
    // correctly when we read from the snapshot.  This is done to allow
    // bootstrapping of reading classes from the snapshot.  Some classes are not
    // stored in the object store. Yet we still need to create their Class
    // object so that they get put into the class_table (as a side effect of
    // Class::New()).
    cls = Class::New<Instance, RTN::Instance>(kInstanceCid, isolate);
    object_store->set_object_class(cls);

    cls = Class::New<LibraryPrefix, RTN::LibraryPrefix>(isolate);
    cls = Class::New<Type, RTN::Type>(isolate);
    cls = Class::New<TypeRef, RTN::TypeRef>(isolate);
    cls = Class::New<TypeParameter, RTN::TypeParameter>(isolate);

    cls = Class::New<Array, RTN::Array>(isolate);
    object_store->set_array_class(cls);

    cls = Class::New<Array, RTN::Array>(kImmutableArrayCid, isolate);
    object_store->set_immutable_array_class(cls);

    cls = Class::New<GrowableObjectArray, RTN::GrowableObjectArray>(isolate);
    object_store->set_growable_object_array_class(cls);

    cls = Class::New<LinkedHashMap, RTN::LinkedHashMap>(isolate);
    object_store->set_linked_hash_map_class(cls);

    cls = Class::New<Float32x4, RTN::Float32x4>(isolate);
    object_store->set_float32x4_class(cls);

    cls = Class::New<Int32x4, RTN::Int32x4>(isolate);
    object_store->set_int32x4_class(cls);

    cls = Class::New<Float64x2, RTN::Float64x2>(isolate);
    object_store->set_float64x2_class(cls);

#define REGISTER_TYPED_DATA_CLASS(clazz)                                       \
  cls = Class::NewTypedDataClass(kTypedData##clazz##Cid, isolate);
    CLASS_LIST_TYPED_DATA(REGISTER_TYPED_DATA_CLASS);
#undef REGISTER_TYPED_DATA_CLASS
#define REGISTER_TYPED_DATA_VIEW_CLASS(clazz)                                  \
  cls = Class::NewTypedDataViewClass(kTypedData##clazz##ViewCid, isolate);
    CLASS_LIST_TYPED_DATA(REGISTER_TYPED_DATA_VIEW_CLASS);
#undef REGISTER_TYPED_DATA_VIEW_CLASS
    cls = Class::NewTypedDataViewClass(kByteDataViewCid, isolate);
#define REGISTER_EXT_TYPED_DATA_CLASS(clazz)                                   \
  cls = Class::NewExternalTypedDataClass(kExternalTypedData##clazz##Cid,       \
                                         isolate);
    CLASS_LIST_TYPED_DATA(REGISTER_EXT_TYPED_DATA_CLASS);
#undef REGISTER_EXT_TYPED_DATA_CLASS

    cls = Class::New<Instance, RTN::Instance>(kFfiNativeTypeCid, isolate);
    object_store->set_ffi_native_type_class(cls);

#define REGISTER_FFI_CLASS(clazz)                                              \
  cls = Class::New<Instance, RTN::Instance>(kFfi##clazz##Cid, isolate);
    CLASS_LIST_FFI_TYPE_MARKER(REGISTER_FFI_CLASS);
#undef REGISTER_FFI_CLASS

#define REGISTER_WASM_CLASS(clazz)                                             \
  cls = Class::New<Instance, RTN::Instance>(k##clazz##Cid, isolate);
    CLASS_LIST_WASM(REGISTER_WASM_CLASS);
#undef REGISTER_WASM_CLASS

    cls = Class::New<Instance, RTN::Instance>(kFfiNativeFunctionCid, isolate);

    cls = Class::NewPointerClass(kFfiPointerCid, isolate);
    object_store->set_ffi_pointer_class(cls);

    cls = Class::New<DynamicLibrary, RTN::DynamicLibrary>(kFfiDynamicLibraryCid,
                                                          isolate);

    cls = Class::New<Instance, RTN::Instance>(kByteBufferCid, isolate,
                                              /*register_isolate=*/false);
    cls.set_instance_size_in_words(0, 0);
    isolate->class_table()->Register(cls);

    cls = Class::New<Integer, RTN::Integer>(isolate);
    object_store->set_integer_implementation_class(cls);

    cls = Class::New<Smi, RTN::Smi>(isolate);
    object_store->set_smi_class(cls);

    cls = Class::New<Mint, RTN::Mint>(isolate);
    object_store->set_mint_class(cls);

    cls = Class::New<Double, RTN::Double>(isolate);
    object_store->set_double_class(cls);

    cls = Class::New<Closure, RTN::Closure>(isolate);
    object_store->set_closure_class(cls);

    cls = Class::NewStringClass(kOneByteStringCid, isolate);
    object_store->set_one_byte_string_class(cls);

    cls = Class::NewStringClass(kTwoByteStringCid, isolate);
    object_store->set_two_byte_string_class(cls);

    cls = Class::NewStringClass(kExternalOneByteStringCid, isolate);
    object_store->set_external_one_byte_string_class(cls);

    cls = Class::NewStringClass(kExternalTwoByteStringCid, isolate);
    object_store->set_external_two_byte_string_class(cls);

    cls = Class::New<Bool, RTN::Bool>(isolate);
    object_store->set_bool_class(cls);

    cls = Class::New<Instance, RTN::Instance>(kNullCid, isolate);
    object_store->set_null_class(cls);

    cls = Class::New<Instance, RTN::Instance>(kNeverCid, isolate);
    object_store->set_never_class(cls);

    cls = Class::New<Capability, RTN::Capability>(isolate);
    cls = Class::New<ReceivePort, RTN::ReceivePort>(isolate);
    cls = Class::New<SendPort, RTN::SendPort>(isolate);
    cls = Class::New<StackTrace, RTN::StackTrace>(isolate);
    cls = Class::New<RegExp, RTN::RegExp>(isolate);
    cls = Class::New<Number, RTN::Number>(isolate);

    cls = Class::New<WeakProperty, RTN::WeakProperty>(isolate);
    object_store->set_weak_property_class(cls);

    cls = Class::New<MirrorReference, RTN::MirrorReference>(isolate);
    cls = Class::New<UserTag, RTN::UserTag>(isolate);
    cls = Class::New<FutureOr, RTN::FutureOr>(isolate);
    cls =
        Class::New<TransferableTypedData, RTN::TransferableTypedData>(isolate);
  }
  return Error::null();
}
```

