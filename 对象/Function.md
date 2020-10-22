# Function

## function.h

```c++
class Function : public Object {
 public:
  StringPtr name() const { return raw_ptr()->name_; }
  StringPtr UserVisibleName() const;  // Same as scrubbed name.
  const char* UserVisibleNameCString() const;

  const char* NameCString(NameVisibility name_visibility) const;

  void PrintName(const NameFormattingParams& params,
                 BaseTextBuffer* printer) const;
  StringPtr QualifiedScrubbedName() const;
  StringPtr QualifiedUserVisibleName() const;

  virtual StringPtr DictionaryName() const { return name(); }

  StringPtr GetSource() const;

  // Return the type of this function's signature. It may not be canonical yet.
  // For example, if this function has a signature of the form
  // '(T, [B, C]) => R', where 'T' and 'R' are type parameters of the
  // owner class of this function, then its signature type is a parameterized
  // function type with uninstantiated type arguments 'T' and 'R' as elements of
  // its type argument vector.
  // A function type is non-nullable by default.
  TypePtr SignatureType(
      Nullability nullability = Nullability::kNonNullable) const;
  TypePtr ExistingSignatureType() const;

  // Update the signature type (with a canonical version).
  void SetSignatureType(const Type& value) const;

  // Set the "C signature" function for an FFI trampoline.
  // Can only be used on FFI trampolines.
  void SetFfiCSignature(const Function& sig) const;

  // Retrieves the "C signature" function for an FFI trampoline.
  // Can only be used on FFI trampolines.
  FunctionPtr FfiCSignature() const;

  bool FfiCSignatureContainsHandles() const;

  // Can only be called on FFI trampolines.
  // -1 for Dart -> native calls.
  int32_t FfiCallbackId() const;

  // Can only be called on FFI trampolines.
  void SetFfiCallbackId(int32_t value) const;

  // Can only be called on FFI trampolines.
  // Null for Dart -> native calls.
  FunctionPtr FfiCallbackTarget() const;

  // Can only be called on FFI trampolines.
  void SetFfiCallbackTarget(const Function& target) const;

  // Can only be called on FFI trampolines.
  // Null for Dart -> native calls.
  InstancePtr FfiCallbackExceptionalReturn() const;

  // Can only be called on FFI trampolines.
  void SetFfiCallbackExceptionalReturn(const Instance& value) const;

  // Return a new function with instantiated result and parameter types.
  FunctionPtr InstantiateSignatureFrom(
      const TypeArguments& instantiator_type_arguments,
      const TypeArguments& function_type_arguments,
      intptr_t num_free_fun_type_params,
      Heap::Space space) const;

  // Build a string of the form '<T>(T, {B b, C c}) => R' representing the
  // internal signature of the given function. In this example, T is a type
  // parameter of this function and R is a type parameter of class C, the owner
  // of the function. B and C are not type parameters.
  StringPtr Signature() const;

  // Build a string of the form '<T>(T, {B b, C c}) => R' representing the
  // user visible signature of the given function. In this example, T is a type
  // parameter of this function and R is a type parameter of class C, the owner
  // of the function. B and C are not type parameters.
  // Implicit parameters are hidden.
  StringPtr UserVisibleSignature() const;

  void PrintSignature(NameVisibility name_visibility,
                      BaseTextBuffer* printer) const;

  // Returns true if the signature of this function is instantiated, i.e. if it
  // does not involve generic parameter types or generic result type.
  // Note that function type parameters declared by this function do not make
  // its signature uninstantiated, only type parameters declared by parent
  // generic functions or class type parameters.
  bool HasInstantiatedSignature(Genericity genericity = kAny,
                                intptr_t num_free_fun_type_params = kAllFree,
                                TrailPtr trail = nullptr) const;

  ClassPtr Owner() const;
  void set_owner(const Object& value) const;
  ClassPtr origin() const;
  ScriptPtr script() const;
  ObjectPtr RawOwner() const { return raw_ptr()->owner_; }

  // The NNBD mode of the library declaring this function.
  // TODO(alexmarkov): nnbd_mode() doesn't work for mixins.
  // It should be either removed or fixed.
  NNBDMode nnbd_mode() const { return Class::Handle(origin()).nnbd_mode(); }

  RegExpPtr regexp() const;
  intptr_t string_specialization_cid() const;
  bool is_sticky_specialization() const;
  void SetRegExpData(const RegExp& regexp,
                     intptr_t string_specialization_cid,
                     bool sticky) const;

  StringPtr native_name() const;
  void set_native_name(const String& name) const;

  AbstractTypePtr result_type() const { return raw_ptr()->result_type_; }
  void set_result_type(const AbstractType& value) const;

  // The parameters, starting with NumImplicitParameters() parameters which are
  // only visible to the VM, but not to Dart users.
  // Note that type checks exclude implicit parameters.
  AbstractTypePtr ParameterTypeAt(intptr_t index) const;
  void SetParameterTypeAt(intptr_t index, const AbstractType& value) const;
  ArrayPtr parameter_types() const { return raw_ptr()->parameter_types_; }
  void set_parameter_types(const Array& value) const;
  static intptr_t parameter_types_offset() {
    return OFFSET_OF(FunctionLayout, parameter_types_);
  }

  // Parameter names are valid for all valid parameter indices, and are not
  // limited to named optional parameters. If there are parameter flags (eg
  // required) they're stored at the end of this array, so the size of this
  // array isn't necessarily NumParameters(), but the first NumParameters()
  // elements are the names.
  StringPtr ParameterNameAt(intptr_t index) const;
  void SetParameterNameAt(intptr_t index, const String& value) const;
  ArrayPtr parameter_names() const { return raw_ptr()->parameter_names_; }
  static intptr_t parameter_names_offset() {
    return OFFSET_OF(FunctionLayout, parameter_names_);
  }

  // Sets up the function's parameter name array, including appropriate space
  // for any possible parameter flags. This may be an overestimate if some
  // parameters don't have flags, and so TruncateUnusedParameterFlags() should
  // be called after all parameter flags have been appropriately set.
  //
  // Assumes that the number of fixed and optional parameters for the function
  // has already been set.
  void CreateNameArrayIncludingFlags(Heap::Space space) const;

  // Truncate the parameter names array to remove any unused flag slots. Make
  // sure to only do this after calling SetIsRequiredAt as necessary.
  void TruncateUnusedParameterFlags() const;

  // The required flags are stored at the end of the parameter_names. The flags
  // are packed into Smis.
  bool IsRequiredAt(intptr_t index) const;
  void SetIsRequiredAt(intptr_t index) const;

  // The type parameters (and their bounds) are specified as an array of
  // TypeParameter.
  TypeArgumentsPtr type_parameters() const {
    return raw_ptr()->type_parameters_;
  }
  void set_type_parameters(const TypeArguments& value) const;
  static intptr_t type_parameters_offset() {
    return OFFSET_OF(FunctionLayout, type_parameters_);
  }
  intptr_t NumTypeParameters(Thread* thread) const;
  intptr_t NumTypeParameters() const {
    return NumTypeParameters(Thread::Current());
  }

  // Returns true if this function has the same number of type parameters with
  // equal bounds as the other function. Type parameter names are ignored.
  bool HasSameTypeParametersAndBounds(const Function& other,
                                      TypeEquality kind) const;

  // Return the number of type parameters declared in parent generic functions.
  intptr_t NumParentTypeParameters() const;

  // Print the signature type of this function and of all of its parents.
  void PrintSignatureTypes() const;

  // Return a TypeParameter if the type_name is a type parameter of this
  // function or of one of its parent functions.
  // Unless NULL, adjust function_level accordingly (in and out parameter).
  // Return null otherwise.
  TypeParameterPtr LookupTypeParameter(const String& type_name,
                                       intptr_t* function_level) const;

  // Return true if this function declares type parameters.
  bool IsGeneric() const { return NumTypeParameters(Thread::Current()) > 0; }

  // Return true if any parent function of this function is generic.
  bool HasGenericParent() const;

  // Not thread-safe; must be called in the main thread.
  // Sets function's code and code's function.
  void InstallOptimizedCode(const Code& code) const;
  void AttachCode(const Code& value) const;
  void SetInstructions(const Code& value) const;
  void ClearCode() const;
  void ClearBytecode() const;

  // Disables optimized code and switches to unoptimized code.
  void SwitchToUnoptimizedCode() const;

  // Ensures that the function has code. If there is no code it compiles the
  // unoptimized version of the code.  If the code contains errors, it calls
  // Exceptions::PropagateError and does not return.  Normally returns the
  // current code, whether it is optimized or unoptimized.
  CodePtr EnsureHasCode() const;

  // Disables optimized code and switches to unoptimized code (or the lazy
  // compilation stub).
  void SwitchToLazyCompiledUnoptimizedCode() const;

  // Compiles unoptimized code (if necessary) and attaches it to the function.
  void EnsureHasCompiledUnoptimizedCode() const;

  // Return the most recently compiled and installed code for this function.
  // It is not the only Code object that points to this function.
  CodePtr CurrentCode() const { return CurrentCodeOf(raw()); }

  bool SafeToClosurize() const;

  static CodePtr CurrentCodeOf(const FunctionPtr function) {
    return function->ptr()->code_;
  }

  CodePtr unoptimized_code() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return static_cast<CodePtr>(Object::null());
#else
    return raw_ptr()->unoptimized_code_;
#endif
  }
  void set_unoptimized_code(const Code& value) const;
  bool HasCode() const;
  static bool HasCode(FunctionPtr function);
#if !defined(DART_PRECOMPILED_RUNTIME)
  static inline bool HasBytecode(FunctionPtr function);
#endif

  static intptr_t code_offset() { return OFFSET_OF(FunctionLayout, code_); }

  static intptr_t result_type_offset() {
    return OFFSET_OF(FunctionLayout, result_type_);
  }

  static intptr_t entry_point_offset(
      CodeEntryKind entry_kind = CodeEntryKind::kNormal) {
    switch (entry_kind) {
      case CodeEntryKind::kNormal:
        return OFFSET_OF(FunctionLayout, entry_point_);
      case CodeEntryKind::kUnchecked:
        return OFFSET_OF(FunctionLayout, unchecked_entry_point_);
      default:
        UNREACHABLE();
    }
  }

  static intptr_t unchecked_entry_point_offset() {
    return OFFSET_OF(FunctionLayout, unchecked_entry_point_);
  }

#if !defined(DART_PRECOMPILED_RUNTIME)
  bool IsBytecodeAllowed(Zone* zone) const;
  void AttachBytecode(const Bytecode& bytecode) const;
  BytecodePtr bytecode() const { return raw_ptr()->bytecode_; }
  inline bool HasBytecode() const;
#else
  inline bool HasBytecode() const { return false; }
#endif

  virtual intptr_t Hash() const;

  // Returns true if there is at least one debugger breakpoint
  // set in this function.
  bool HasBreakpoint() const;

  ContextScopePtr context_scope() const;
  void set_context_scope(const ContextScope& value) const;

  // Enclosing function of this local function.
  FunctionPtr parent_function() const;

  // Enclosed generated closure function of this local function.
  // This will only work after the closure function has been allocated in the
  // isolate's object_store.
  FunctionPtr GetGeneratedClosure() const;

  // Enclosing outermost function of this local function.
  FunctionPtr GetOutermostFunction() const;

  void set_extracted_method_closure(const Function& function) const;
  FunctionPtr extracted_method_closure() const;

  void set_saved_args_desc(const Array& array) const;
  ArrayPtr saved_args_desc() const;

  void set_accessor_field(const Field& value) const;
  FieldPtr accessor_field() const;

  bool IsRegularFunction() const {
    return kind() == FunctionLayout::kRegularFunction;
  }

  bool IsMethodExtractor() const {
    return kind() == FunctionLayout::kMethodExtractor;
  }

  bool IsNoSuchMethodDispatcher() const {
    return kind() == FunctionLayout::kNoSuchMethodDispatcher;
  }

  bool IsInvokeFieldDispatcher() const {
    return kind() == FunctionLayout::kInvokeFieldDispatcher;
  }

  bool IsDynamicInvokeFieldDispatcher() const {
    return IsInvokeFieldDispatcher() &&
           IsDynamicInvocationForwarderName(name());
  }

  // Performs all the checks that don't require the current thread first, to
  // avoid retrieving it unless they all pass. If you have a handle on the
  // current thread, call the version that takes one instead.
  bool IsDynamicClosureCallDispatcher() const {
    if (!IsDynamicInvokeFieldDispatcher()) return false;
    return IsDynamicClosureCallDispatcher(Thread::Current());
  }
  bool IsDynamicClosureCallDispatcher(Thread* thread) const;

  bool IsDynamicInvocationForwarder() const {
    return kind() == FunctionLayout::kDynamicInvocationForwarder;
  }

  bool IsImplicitGetterOrSetter() const {
    return kind() == FunctionLayout::kImplicitGetter ||
           kind() == FunctionLayout::kImplicitSetter ||
           kind() == FunctionLayout::kImplicitStaticGetter;
  }

  // Returns true iff an implicit closure function has been created
  // for this function.
  bool HasImplicitClosureFunction() const {
    return implicit_closure_function() != null();
  }

  // Returns the closure function implicitly created for this function.  If none
  // exists yet, create one and remember it.  Implicit closure functions are
  // used in VM Closure instances that represent results of tear-off operations.
  FunctionPtr ImplicitClosureFunction() const;
  void DropUncompiledImplicitClosureFunction() const;

  // Return the closure implicitly created for this function.
  // If none exists yet, create one and remember it.
  InstancePtr ImplicitStaticClosure() const;

  InstancePtr ImplicitInstanceClosure(const Instance& receiver) const;

  // Returns the target of the implicit closure or null if the target is now
  // invalid (e.g., mismatched argument shapes after a reload).
  FunctionPtr ImplicitClosureTarget(Zone* zone) const;

  intptr_t ComputeClosureHash() const;

  // Redirection information for a redirecting factory.
  bool IsRedirectingFactory() const;
  TypePtr RedirectionType() const;
  void SetRedirectionType(const Type& type) const;
  StringPtr RedirectionIdentifier() const;
  void SetRedirectionIdentifier(const String& identifier) const;
  FunctionPtr RedirectionTarget() const;
  void SetRedirectionTarget(const Function& target) const;

  FunctionPtr ForwardingTarget() const;
  void SetForwardingChecks(const Array& checks) const;

  FunctionLayout::Kind kind() const {
    return KindBits::decode(raw_ptr()->kind_tag_);
  }
  static FunctionLayout::Kind kind(FunctionPtr function) {
    return KindBits::decode(function->ptr()->kind_tag_);
  }

  FunctionLayout::AsyncModifier modifier() const {
    return ModifierBits::decode(raw_ptr()->kind_tag_);
  }

  static const char* KindToCString(FunctionLayout::Kind kind);

  bool IsGenerativeConstructor() const {
    return (kind() == FunctionLayout::kConstructor) && !is_static();
  }
  bool IsImplicitConstructor() const;
  bool IsFactory() const {
    return (kind() == FunctionLayout::kConstructor) && is_static();
  }

  static bool ClosureBodiesContainNonCovariantChecks() {
    return FLAG_precompiled_mode || FLAG_lazy_dispatchers;
  }

  // Whether this function can receive an invocation where the number and names
  // of arguments have not been checked.
  bool CanReceiveDynamicInvocation() const { return IsFfiTrampoline(); }

  bool HasThisParameter() const {
    return IsDynamicFunction(/*allow_abstract=*/true) ||
           IsGenerativeConstructor() || (IsFieldInitializer() && !is_static());
  }

  bool IsDynamicFunction(bool allow_abstract = false) const {
    if (is_static() || (!allow_abstract && is_abstract())) {
      return false;
    }
    switch (kind()) {
      case FunctionLayout::kRegularFunction:
      case FunctionLayout::kGetterFunction:
      case FunctionLayout::kSetterFunction:
      case FunctionLayout::kImplicitGetter:
      case FunctionLayout::kImplicitSetter:
      case FunctionLayout::kMethodExtractor:
      case FunctionLayout::kNoSuchMethodDispatcher:
      case FunctionLayout::kInvokeFieldDispatcher:
      case FunctionLayout::kDynamicInvocationForwarder:
        return true;
      case FunctionLayout::kClosureFunction:
      case FunctionLayout::kImplicitClosureFunction:
      case FunctionLayout::kSignatureFunction:
      case FunctionLayout::kConstructor:
      case FunctionLayout::kImplicitStaticGetter:
      case FunctionLayout::kFieldInitializer:
      case FunctionLayout::kIrregexpFunction:
        return false;
      default:
        UNREACHABLE();
        return false;
    }
  }
  bool IsStaticFunction() const {
    if (!is_static()) {
      return false;
    }
    switch (kind()) {
      case FunctionLayout::kRegularFunction:
      case FunctionLayout::kGetterFunction:
      case FunctionLayout::kSetterFunction:
      case FunctionLayout::kImplicitGetter:
      case FunctionLayout::kImplicitSetter:
      case FunctionLayout::kImplicitStaticGetter:
      case FunctionLayout::kFieldInitializer:
      case FunctionLayout::kIrregexpFunction:
        return true;
      case FunctionLayout::kClosureFunction:
      case FunctionLayout::kImplicitClosureFunction:
      case FunctionLayout::kSignatureFunction:
      case FunctionLayout::kConstructor:
      case FunctionLayout::kMethodExtractor:
      case FunctionLayout::kNoSuchMethodDispatcher:
      case FunctionLayout::kInvokeFieldDispatcher:
      case FunctionLayout::kDynamicInvocationForwarder:
        return false;
      default:
        UNREACHABLE();
        return false;
    }
  }
  bool IsInFactoryScope() const;

  bool NeedsArgumentTypeChecks() const {
    return (IsClosureFunction() && ClosureBodiesContainNonCovariantChecks()) ||
           !(is_static() || (kind() == FunctionLayout::kConstructor));
  }

  bool NeedsMonomorphicCheckedEntry(Zone* zone) const;
  bool HasDynamicCallers(Zone* zone) const;
  bool PrologueNeedsArgumentsDescriptor() const;

  bool MayHaveUncheckedEntryPoint() const;

  TokenPosition token_pos() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return TokenPosition();
#else
    return raw_ptr()->token_pos_;
#endif
  }
  void set_token_pos(TokenPosition value) const;

  TokenPosition end_token_pos() const {
#if defined(DART_PRECOMPILED_RUNTIME)
    return TokenPosition();
#else
    return raw_ptr()->end_token_pos_;
#endif
  }
  void set_end_token_pos(TokenPosition value) const {
#if defined(DART_PRECOMPILED_RUNTIME)
    UNREACHABLE();
#else
    StoreNonPointer(&raw_ptr()->end_token_pos_, value);
#endif
  }

  intptr_t num_fixed_parameters() const {
    return FunctionLayout::PackedNumFixedParameters::decode(
        raw_ptr()->packed_fields_);
  }
  void set_num_fixed_parameters(intptr_t value) const;

  uint32_t packed_fields() const { return raw_ptr()->packed_fields_; }
  void set_packed_fields(uint32_t packed_fields) const;
  static intptr_t packed_fields_offset() {
    return OFFSET_OF(FunctionLayout, packed_fields_);
  }
  // Reexported so they can be used by the flow graph builders.
  using PackedHasNamedOptionalParameters =
      FunctionLayout::PackedHasNamedOptionalParameters;
  using PackedNumFixedParameters = FunctionLayout::PackedNumFixedParameters;
  using PackedNumOptionalParameters =
      FunctionLayout::PackedNumOptionalParameters;

  bool HasOptionalParameters() const {
    return PackedNumOptionalParameters::decode(raw_ptr()->packed_fields_) > 0;
  }
  bool HasOptionalNamedParameters() const {
    return HasOptionalParameters() &&
           PackedHasNamedOptionalParameters::decode(raw_ptr()->packed_fields_);
  }
  bool HasOptionalPositionalParameters() const {
    return HasOptionalParameters() && !HasOptionalNamedParameters();
  }
  intptr_t NumOptionalParameters() const {
    return PackedNumOptionalParameters::decode(raw_ptr()->packed_fields_);
  }
  void SetNumOptionalParameters(intptr_t num_optional_parameters,
                                bool are_optional_positional) const;

  intptr_t NumOptionalPositionalParameters() const {
    return HasOptionalPositionalParameters() ? NumOptionalParameters() : 0;
  }

  intptr_t NumOptionalNamedParameters() const {
    return HasOptionalNamedParameters() ? NumOptionalParameters() : 0;
  }

  intptr_t NumParameters() const;

  intptr_t NumImplicitParameters() const;

#if defined(DART_PRECOMPILED_RUNTIME)
#define DEFINE_GETTERS_AND_SETTERS(return_type, type, name)                    \
  static intptr_t name##_offset() {                                            \
    UNREACHABLE();                                                             \
    return 0;                                                                  \
  }                                                                            \
  return_type name() const { return 0; }                                       \
                                                                               \
  void set_##name(type value) const { UNREACHABLE(); }
#else
#define DEFINE_GETTERS_AND_SETTERS(return_type, type, name)                    \
  static intptr_t name##_offset() {                                            \
    return OFFSET_OF(FunctionLayout, name##_);                                 \
  }                                                                            \
  return_type name() const { return raw_ptr()->name##_; }                      \
                                                                               \
  void set_##name(type value) const {                                          \
    StoreNonPointer(&raw_ptr()->name##_, value);                               \
  }
#endif

  JIT_FUNCTION_COUNTERS(DEFINE_GETTERS_AND_SETTERS)

#undef DEFINE_GETTERS_AND_SETTERS

#if !defined(DART_PRECOMPILED_RUNTIME)
  intptr_t binary_declaration_offset() const {
    return FunctionLayout::BinaryDeclarationOffset::decode(
        raw_ptr()->binary_declaration_);
  }
  void set_binary_declaration_offset(intptr_t value) const {
    ASSERT(value >= 0);
    StoreNonPointer(&raw_ptr()->binary_declaration_,
                    FunctionLayout::BinaryDeclarationOffset::update(
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
    return FunctionLayout::IsDeclaredInBytecode::decode(
        raw_ptr()->binary_declaration_);
#endif
  }

#if !defined(DART_PRECOMPILED_RUNTIME)
  void set_is_declared_in_bytecode(bool value) const {
    StoreNonPointer(&raw_ptr()->binary_declaration_,
                    FunctionLayout::IsDeclaredInBytecode::update(
                        value, raw_ptr()->binary_declaration_));
  }
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

  void InheritBinaryDeclarationFrom(const Function& src) const;
  void InheritBinaryDeclarationFrom(const Field& src) const;

  static const intptr_t kMaxInstructionCount = (1 << 16) - 1;

  void SetOptimizedInstructionCountClamped(uintptr_t value) const {
    if (value > kMaxInstructionCount) value = kMaxInstructionCount;
    set_optimized_instruction_count(value);
  }

  void SetOptimizedCallSiteCountClamped(uintptr_t value) const {
    if (value > kMaxInstructionCount) value = kMaxInstructionCount;
    set_optimized_call_site_count(value);
  }

  void SetKernelDataAndScript(const Script& script,
                              const ExternalTypedData& data,
                              intptr_t offset) const;

  intptr_t KernelDataProgramOffset() const;

  ExternalTypedDataPtr KernelData() const;

  bool IsOptimizable() const;
  void SetIsOptimizable(bool value) const;

  // Whether this function must be optimized immediately and cannot be compiled
  // with the unoptimizing compiler. Such a function must be sure to not
  // deoptimize, since we won't generate deoptimization info or register
  // dependencies. It will be compiled into optimized code immediately when it's
  // run.
  bool ForceOptimize() const {
    return IsFfiFromAddress() || IsFfiGetAddress() || IsFfiLoad() ||
           IsFfiStore() || IsFfiTrampoline() || IsTypedDataViewFactory() ||
           IsUtf8Scan();
  }

  bool CanBeInlined() const;

  MethodRecognizer::Kind recognized_kind() const {
    return RecognizedBits::decode(raw_ptr()->kind_tag_);
  }
  void set_recognized_kind(MethodRecognizer::Kind value) const;

  bool IsRecognized() const {
    return recognized_kind() != MethodRecognizer::kUnknown;
  }

  bool HasOptimizedCode() const;

  // Whether the function is ready for compiler optimizations.
  bool ShouldCompilerOptimize() const;

  // Returns true if the argument counts are valid for calling this function.
  // Otherwise, it returns false and the reason (if error_message is not NULL).
  bool AreValidArgumentCounts(intptr_t num_type_arguments,
                              intptr_t num_arguments,
                              intptr_t num_named_arguments,
                              String* error_message) const;

  // Returns a TypeError if the provided arguments don't match the function
  // parameter types, null otherwise. Assumes AreValidArguments is called first.
  //
  // If the function has a non-null receiver in the arguments, the instantiator
  // type arguments are retrieved from the receiver, otherwise the null type
  // arguments vector is used.
  //
  // If the function is generic, the appropriate function type arguments are
  // retrieved either from the arguments array or the receiver (if a closure).
  // If no function type arguments are available in either location, the bounds
  // of the function type parameters are instantiated and used as the function
  // type arguments.
  //
  // The local function type arguments (_not_ parent function type arguments)
  // are also checked against the bounds of the corresponding parameters to
  // ensure they are appropriate subtypes if the function is generic.
  ObjectPtr DoArgumentTypesMatch(const Array& args,
                                 const ArgumentsDescriptor& arg_names) const;

  // Returns a TypeError if the provided arguments don't match the function
  // parameter types, null otherwise. Assumes AreValidArguments is called first.
  //
  // If the function is generic, the appropriate function type arguments are
  // retrieved either from the arguments array or the receiver (if a closure).
  // If no function type arguments are available in either location, the bounds
  // of the function type parameters are instantiated and used as the function
  // type arguments.
  //
  // The local function type arguments (_not_ parent function type arguments)
  // are also checked against the bounds of the corresponding parameters to
  // ensure they are appropriate subtypes if the function is generic.
  ObjectPtr DoArgumentTypesMatch(
      const Array& args,
      const ArgumentsDescriptor& arg_names,
      const TypeArguments& instantiator_type_args) const;

  // Returns a TypeError if the provided arguments don't match the function
  // parameter types, null otherwise. Assumes AreValidArguments is called first.
  //
  // The local function type arguments (_not_ parent function type arguments)
  // are also checked against the bounds of the corresponding parameters to
  // ensure they are appropriate subtypes if the function is generic.
  ObjectPtr DoArgumentTypesMatch(const Array& args,
                                 const ArgumentsDescriptor& arg_names,
                                 const TypeArguments& instantiator_type_args,
                                 const TypeArguments& function_type_args) const;

  // Returns true if the type argument count, total argument count and the names
  // of optional arguments are valid for calling this function.
  // Otherwise, it returns false and the reason (if error_message is not NULL).
  bool AreValidArguments(intptr_t num_type_arguments,
                         intptr_t num_arguments,
                         const Array& argument_names,
                         String* error_message) const;
  bool AreValidArguments(const ArgumentsDescriptor& args_desc,
                         String* error_message) const;

  // Fully qualified name uniquely identifying the function under gdb and during
  // ast printing. The special ':' character, if present, is replaced by '_'.
  const char* ToFullyQualifiedCString() const;

  const char* ToLibNamePrefixedQualifiedCString() const;

  const char* ToQualifiedCString() const;

  static constexpr intptr_t maximum_unboxed_parameter_count() {
    // Subtracts one that represents the return value
    return FunctionLayout::UnboxedParameterBitmap::kCapacity - 1;
  }

  void reset_unboxed_parameters_and_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    StoreNonPointer(&raw_ptr()->unboxed_parameters_info_,
                    FunctionLayout::UnboxedParameterBitmap());
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  void set_unboxed_integer_parameter_at(intptr_t index) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(index >= 0 && index < maximum_unboxed_parameter_count());
    index++;  // position 0 is reserved for the return value
    const_cast<FunctionLayout::UnboxedParameterBitmap*>(
        &raw_ptr()->unboxed_parameters_info_)
        ->SetUnboxedInteger(index);
#else
    UNREACHABLE();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  void set_unboxed_double_parameter_at(intptr_t index) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(index >= 0 && index < maximum_unboxed_parameter_count());
    index++;  // position 0 is reserved for the return value
    const_cast<FunctionLayout::UnboxedParameterBitmap*>(
        &raw_ptr()->unboxed_parameters_info_)
        ->SetUnboxedDouble(index);

#else
    UNREACHABLE();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  void set_unboxed_integer_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    const_cast<FunctionLayout::UnboxedParameterBitmap*>(
        &raw_ptr()->unboxed_parameters_info_)
        ->SetUnboxedInteger(0);
#else
    UNREACHABLE();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  void set_unboxed_double_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    const_cast<FunctionLayout::UnboxedParameterBitmap*>(
        &raw_ptr()->unboxed_parameters_info_)
        ->SetUnboxedDouble(0);

#else
    UNREACHABLE();
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool is_unboxed_parameter_at(intptr_t index) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(index >= 0);
    index++;  // position 0 is reserved for the return value
    return raw_ptr()->unboxed_parameters_info_.IsUnboxed(index);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool is_unboxed_integer_parameter_at(intptr_t index) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(index >= 0);
    index++;  // position 0 is reserved for the return value
    return raw_ptr()->unboxed_parameters_info_.IsUnboxedInteger(index);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool is_unboxed_double_parameter_at(intptr_t index) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    ASSERT(index >= 0);
    index++;  // position 0 is reserved for the return value
    return raw_ptr()->unboxed_parameters_info_.IsUnboxedDouble(index);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool has_unboxed_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return raw_ptr()->unboxed_parameters_info_.IsUnboxed(0);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool has_unboxed_integer_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return raw_ptr()->unboxed_parameters_info_.IsUnboxedInteger(0);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

  bool has_unboxed_double_return() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
    return raw_ptr()->unboxed_parameters_info_.IsUnboxedDouble(0);
#else
    return false;
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)
  }

#if !defined(DART_PRECOMPILED_RUNTIME)
  bool HasUnboxedParameters() const {
    return raw_ptr()->unboxed_parameters_info_.HasUnboxedParameters();
  }
  bool HasUnboxedReturnValue() const {
    return raw_ptr()->unboxed_parameters_info_.HasUnboxedReturnValue();
  }
#endif  //  !defined(DART_PRECOMPILED_RUNTIME)

  // Returns true if the type of this function is a subtype of the type of
  // the other function.
  bool IsSubtypeOf(const Function& other, Heap::Space space) const;

  bool IsDispatcherOrImplicitAccessor() const {
    switch (kind()) {
      case FunctionLayout::kImplicitGetter:
      case FunctionLayout::kImplicitSetter:
      case FunctionLayout::kImplicitStaticGetter:
      case FunctionLayout::kNoSuchMethodDispatcher:
      case FunctionLayout::kInvokeFieldDispatcher:
      case FunctionLayout::kDynamicInvocationForwarder:
        return true;
      default:
        return false;
    }
  }

  // Returns true if this function represents an explicit getter function.
  bool IsGetterFunction() const {
    return kind() == FunctionLayout::kGetterFunction;
  }

  // Returns true if this function represents an implicit getter function.
  bool IsImplicitGetterFunction() const {
    return kind() == FunctionLayout::kImplicitGetter;
  }

  // Returns true if this function represents an implicit static getter
  // function.
  bool IsImplicitStaticGetterFunction() const {
    return kind() == FunctionLayout::kImplicitStaticGetter;
  }

  // Returns true if this function represents an explicit setter function.
  bool IsSetterFunction() const {
    return kind() == FunctionLayout::kSetterFunction;
  }

  // Returns true if this function represents an implicit setter function.
  bool IsImplicitSetterFunction() const {
    return kind() == FunctionLayout::kImplicitSetter;
  }

  // Returns true if this function represents an initializer for a static or
  // instance field. The function returns the initial value and the caller is
  // responsible for setting the field.
  bool IsFieldInitializer() const {
    return kind() == FunctionLayout::kFieldInitializer;
  }

  // Returns true if this function represents a (possibly implicit) closure
  // function.
  bool IsClosureFunction() const {
    FunctionLayout::Kind k = kind();
    return (k == FunctionLayout::kClosureFunction) ||
           (k == FunctionLayout::kImplicitClosureFunction);
  }

  // Returns true if this function represents a generated irregexp function.
  bool IsIrregexpFunction() const {
    return kind() == FunctionLayout::kIrregexpFunction;
  }

  // Returns true if this function represents an implicit closure function.
  bool IsImplicitClosureFunction() const {
    return kind() == FunctionLayout::kImplicitClosureFunction;
  }

  // Returns true if this function represents a non implicit closure function.
  bool IsNonImplicitClosureFunction() const {
    return IsClosureFunction() && !IsImplicitClosureFunction();
  }

  // Returns true if this function represents an implicit static closure
  // function.
  bool IsImplicitStaticClosureFunction() const {
    return IsImplicitClosureFunction() && is_static();
  }
  static bool IsImplicitStaticClosureFunction(FunctionPtr func);

  // Returns true if this function represents an implicit instance closure
  // function.
  bool IsImplicitInstanceClosureFunction() const {
    return IsImplicitClosureFunction() && !is_static();
  }

  // Returns true if this function represents a local function.
  bool IsLocalFunction() const { return parent_function() != Function::null(); }

  // Returns true if this function represents a signature function without code.
  bool IsSignatureFunction() const {
    return kind() == FunctionLayout::kSignatureFunction;
  }
  static bool IsSignatureFunction(FunctionPtr function) {
    NoSafepointScope no_safepoint;
    return KindBits::decode(function->ptr()->kind_tag_) ==
           FunctionLayout::kSignatureFunction;
  }

  // Returns true if this function represents an ffi trampoline.
  bool IsFfiTrampoline() const {
    return kind() == FunctionLayout::kFfiTrampoline;
  }
  static bool IsFfiTrampoline(FunctionPtr function) {
    NoSafepointScope no_safepoint;
    return KindBits::decode(function->ptr()->kind_tag_) ==
           FunctionLayout::kFfiTrampoline;
  }

  bool IsFfiLoad() const {
    const auto kind = recognized_kind();
    return MethodRecognizer::kFfiLoadInt8 <= kind &&
           kind <= MethodRecognizer::kFfiLoadPointer;
  }

  bool IsFfiStore() const {
    const auto kind = recognized_kind();
    return MethodRecognizer::kFfiStoreInt8 <= kind &&
           kind <= MethodRecognizer::kFfiStorePointer;
  }

  bool IsFfiFromAddress() const {
    const auto kind = recognized_kind();
    return kind == MethodRecognizer::kFfiFromAddress;
  }

  bool IsFfiGetAddress() const {
    const auto kind = recognized_kind();
    return kind == MethodRecognizer::kFfiGetAddress;
  }

  bool IsUtf8Scan() const {
    const auto kind = recognized_kind();
    return kind == MethodRecognizer::kUtf8DecoderScan;
  }

  bool IsAsyncFunction() const { return modifier() == FunctionLayout::kAsync; }

  bool IsAsyncClosure() const {
    return is_generated_body() &&
           Function::Handle(parent_function()).IsAsyncFunction();
  }

  bool IsGenerator() const {
    return (modifier() & FunctionLayout::kGeneratorBit) != 0;
  }

  bool IsSyncGenerator() const {
    return modifier() == FunctionLayout::kSyncGen;
  }

  bool IsSyncGenClosure() const {
    return is_generated_body() &&
           Function::Handle(parent_function()).IsSyncGenerator();
  }

  bool IsGeneratorClosure() const {
    return is_generated_body() &&
           Function::Handle(parent_function()).IsGenerator();
  }

  bool IsAsyncGenerator() const {
    return modifier() == FunctionLayout::kAsyncGen;
  }

  bool IsAsyncGenClosure() const {
    return is_generated_body() &&
           Function::Handle(parent_function()).IsAsyncGenerator();
  }

  bool IsAsyncOrGenerator() const {
    return modifier() != FunctionLayout::kNoModifier;
  }

  // Recognise synthetic sync-yielding functions like the inner-most:
  //   user_func /* was sync* */ {
  //     :sync_op_gen() {
  //        :sync_op() yielding {
  //          // ...
  //        }
  //      }
  //   }
  bool IsSyncYielding() const {
    return (parent_function() != Function::null())
               ? Function::Handle(parent_function()).IsSyncGenClosure()
               : false;
  }

  bool IsTypedDataViewFactory() const {
    if (is_native() && kind() == FunctionLayout::kConstructor) {
      // This is a native factory constructor.
      const Class& klass = Class::Handle(Owner());
      return IsTypedDataViewClassId(klass.id());
    }
    return false;
  }

  DART_WARN_UNUSED_RESULT
  ErrorPtr VerifyCallEntryPoint() const;

  DART_WARN_UNUSED_RESULT
  ErrorPtr VerifyClosurizedEntryPoint() const;

  static intptr_t InstanceSize() {
    return RoundedAllocationSize(sizeof(FunctionLayout));
  }

  static FunctionPtr New(const String& name,
                         FunctionLayout::Kind kind,
                         bool is_static,
                         bool is_const,
                         bool is_abstract,
                         bool is_external,
                         bool is_native,
                         const Object& owner,
                         TokenPosition token_pos,
                         Heap::Space space = Heap::kOld);

  // Allocates a new Function object representing a closure function
  // with given kind - kClosureFunction or kImplicitClosureFunction.
  static FunctionPtr NewClosureFunctionWithKind(FunctionLayout::Kind kind,
                                                const String& name,
                                                const Function& parent,
                                                TokenPosition token_pos,
                                                const Object& owner);

  // Allocates a new Function object representing a closure function.
  static FunctionPtr NewClosureFunction(const String& name,
                                        const Function& parent,
                                        TokenPosition token_pos);

  // Allocates a new Function object representing an implicit closure function.
  static FunctionPtr NewImplicitClosureFunction(const String& name,
                                                const Function& parent,
                                                TokenPosition token_pos);

  // Allocates a new Function object representing a signature function.
  // The owner is the scope class of the function type.
  // The parent is the enclosing function or null if none.
  static FunctionPtr NewSignatureFunction(const Object& owner,
                                          const Function& parent,
                                          TokenPosition token_pos,
                                          Heap::Space space = Heap::kOld);

  static FunctionPtr NewEvalFunction(const Class& owner,
                                     const Script& script,
                                     bool is_static);

  FunctionPtr CreateMethodExtractor(const String& getter_name) const;
  FunctionPtr GetMethodExtractor(const String& getter_name) const;

  static bool IsDynamicInvocationForwarderName(const String& name);
  static bool IsDynamicInvocationForwarderName(StringPtr name);

  static StringPtr DemangleDynamicInvocationForwarderName(const String& name);

  static StringPtr CreateDynamicInvocationForwarderName(const String& name);

#if !defined(DART_PRECOMPILED_RUNTIME)
  FunctionPtr CreateDynamicInvocationForwarder(
      const String& mangled_name) const;

  FunctionPtr GetDynamicInvocationForwarder(const String& mangled_name,
                                            bool allow_add = true) const;
#endif

  // Slow function, use in asserts to track changes in important library
  // functions.
  int32_t SourceFingerprint() const;

  // Return false and report an error if the fingerprint does not match.
  bool CheckSourceFingerprint(int32_t fp) const;

  // Works with map [deopt-id] -> ICData.
  void SaveICDataMap(
      const ZoneGrowableArray<const ICData*>& deopt_id_to_ic_data,
      const Array& edge_counters_array) const;
  // Uses 'ic_data_array' to populate the table 'deopt_id_to_ic_data'. Clone
  // ic_data (array and descriptor) if 'clone_ic_data' is true.
  void RestoreICDataMap(ZoneGrowableArray<const ICData*>* deopt_id_to_ic_data,
                        bool clone_ic_data) const;

  ArrayPtr ic_data_array() const;
  void ClearICDataArray() const;
  ICDataPtr FindICData(intptr_t deopt_id) const;

  // Sets deopt reason in all ICData-s with given deopt_id.
  void SetDeoptReasonForAll(intptr_t deopt_id, ICData::DeoptReasonId reason);

  void set_modifier(FunctionLayout::AsyncModifier value) const;

// 'WasCompiled' is true if the function was compiled once in this
// VM instantiation. It is independent from presence of type feedback
// (ic_data_array) and code, which may be loaded from a snapshot.
// 'WasExecuted' is true if the usage counter has ever been positive.
// 'ProhibitsHoistingCheckClass' is true if this function deoptimized before on
// a hoisted check class instruction.
// 'ProhibitsBoundsCheckGeneralization' is true if this function deoptimized
// before on a generalized bounds check.
#define STATE_BITS_LIST(V)                                                     \
  V(WasCompiled)                                                               \
  V(WasExecutedBit)                                                            \
  V(ProhibitsHoistingCheckClass)                                               \
  V(ProhibitsBoundsCheckGeneralization)

  enum StateBits {
#define DECLARE_FLAG_POS(Name) k##Name##Pos,
    STATE_BITS_LIST(DECLARE_FLAG_POS)
#undef DECLARE_FLAG_POS
  };
#define DEFINE_FLAG_BIT(Name)                                                  \
  class Name##Bit : public BitField<uint8_t, bool, k##Name##Pos, 1> {};
  STATE_BITS_LIST(DEFINE_FLAG_BIT)
#undef DEFINE_FLAG_BIT

#define DEFINE_FLAG_ACCESSORS(Name)                                            \
  void Set##Name(bool value) const {                                           \
    set_state_bits(Name##Bit::update(value, state_bits()));                    \
  }                                                                            \
  bool Name() const { return Name##Bit::decode(state_bits()); }
  STATE_BITS_LIST(DEFINE_FLAG_ACCESSORS)
#undef DEFINE_FLAG_ACCESSORS

  void SetUsageCounter(intptr_t value) const {
    if (usage_counter() > 0) {
      SetWasExecuted(true);
    }
    set_usage_counter(value);
  }

  bool WasExecuted() const { return (usage_counter() > 0) || WasExecutedBit(); }

  void SetWasExecuted(bool value) const { SetWasExecutedBit(value); }

  // static: Considered during class-side or top-level resolution rather than
  //         instance-side resolution.
  // const: Valid target of a const constructor call.
  // abstract: Skipped during instance-side resolution.
  // reflectable: Enumerated by mirrors, invocable by mirrors. False for private
  //              functions of dart: libraries.
  // debuggable: Valid location of a breakpoint. Synthetic code is not
  //             debuggable.
  // visible: Frame is included in stack traces. Synthetic code such as
  //          dispatchers is not visible. Synthetic code that can trigger
  //          exceptions such as the outer async functions that create Futures
  //          is visible.
  // instrinsic: Has a hand-written assembly prologue.
  // inlinable: Candidate for inlining. False for functions with features we
  //            don't support during inlining (e.g., optional parameters),
  //            functions which are too big, etc.
  // native: Bridge to C/C++ code.
  // redirecting: Redirecting generative or factory constructor.
  // external: Just a declaration that expects to be defined in another patch
  //           file.
  // generated_body: Has a generated body.
  // polymorphic_target: A polymorphic method.
  // has_pragma: Has a @pragma decoration.
  // no_such_method_forwarder: A stub method that just calls noSuchMethod.

#define FOR_EACH_FUNCTION_KIND_BIT(V)                                          \
  V(Static, is_static)                                                         \
  V(Const, is_const)                                                           \
  V(Abstract, is_abstract)                                                     \
  V(Reflectable, is_reflectable)                                               \
  V(Visible, is_visible)                                                       \
  V(Debuggable, is_debuggable)                                                 \
  V(Inlinable, is_inlinable)                                                   \
  V(Intrinsic, is_intrinsic)                                                   \
  V(Native, is_native)                                                         \
  V(Redirecting, is_redirecting)                                               \
  V(External, is_external)                                                     \
  V(GeneratedBody, is_generated_body)                                          \
  V(PolymorphicTarget, is_polymorphic_target)                                  \
  V(HasPragma, has_pragma)                                                     \
  V(IsSynthetic, is_synthetic)                                                 \
  V(IsExtensionMember, is_extension_member)

#define DEFINE_ACCESSORS(name, accessor_name)                                  \
  void set_##accessor_name(bool value) const {                                 \
    set_kind_tag(name##Bit::update(value, raw_ptr()->kind_tag_));              \
  }                                                                            \
  bool accessor_name() const { return name##Bit::decode(raw_ptr()->kind_tag_); }
  FOR_EACH_FUNCTION_KIND_BIT(DEFINE_ACCESSORS)
#undef DEFINE_ACCESSORS

  // optimizable: Candidate for going through the optimizing compiler. False for
  //              some functions known to be execute infrequently and functions
  //              which have been de-optimized too many times.
  bool is_optimizable() const {
    return FunctionLayout::OptimizableBit::decode(raw_ptr()->packed_fields_);
  }
  void set_is_optimizable(bool value) const {
    set_packed_fields(FunctionLayout::OptimizableBit::update(
        value, raw_ptr()->packed_fields_));
  }

  // Indicates whether this function can be optimized on the background compiler
  // thread.
  bool is_background_optimizable() const {
    return FunctionLayout::BackgroundOptimizableBit::decode(
        raw_ptr()->packed_fields_);
  }

  void set_is_background_optimizable(bool value) const {
    set_packed_fields(FunctionLayout::BackgroundOptimizableBit::update(
        value, raw_ptr()->packed_fields_));
  }

 private:
  void set_parameter_names(const Array& value) const;
  void set_ic_data_array(const Array& value) const;
  void SetInstructionsSafe(const Code& value) const;

  enum KindTagBits {
    kKindTagPos = 0,
    kKindTagSize = 5,
    kRecognizedTagPos = kKindTagPos + kKindTagSize,
    kRecognizedTagSize = 9,
    kModifierPos = kRecognizedTagPos + kRecognizedTagSize,
    kModifierSize = 2,
    kLastModifierBitPos = kModifierPos + (kModifierSize - 1),
// Single bit sized fields start here.
#define DECLARE_BIT(name, _) k##name##Bit,
    FOR_EACH_FUNCTION_KIND_BIT(DECLARE_BIT)
#undef DECLARE_BIT
        kNumTagBits
  };

  COMPILE_ASSERT(MethodRecognizer::kNumRecognizedMethods <
                 (1 << kRecognizedTagSize));
  COMPILE_ASSERT(kNumTagBits <=
                 (kBitsPerByte *
                  sizeof(static_cast<FunctionLayout*>(nullptr)->kind_tag_)));

  class KindBits : public BitField<uint32_t,
                                   FunctionLayout::Kind,
                                   kKindTagPos,
                                   kKindTagSize> {};

  class RecognizedBits : public BitField<uint32_t,
                                         MethodRecognizer::Kind,
                                         kRecognizedTagPos,
                                         kRecognizedTagSize> {};
  class ModifierBits : public BitField<uint32_t,
                                       FunctionLayout::AsyncModifier,
                                       kModifierPos,
                                       kModifierSize> {};

#define DEFINE_BIT(name, _)                                                    \
  class name##Bit : public BitField<uint32_t, bool, k##name##Bit, 1> {};
  FOR_EACH_FUNCTION_KIND_BIT(DEFINE_BIT)
#undef DEFINE_BIT

  void set_name(const String& value) const;
  void set_kind(FunctionLayout::Kind value) const;
  void set_parent_function(const Function& value) const;
  FunctionPtr implicit_closure_function() const;
  void set_implicit_closure_function(const Function& value) const;
  InstancePtr implicit_static_closure() const;
  void set_implicit_static_closure(const Instance& closure) const;
  ScriptPtr eval_script() const;
  void set_eval_script(const Script& value) const;
  void set_num_optional_parameters(intptr_t value) const;  // Encoded value.
  void set_kind_tag(uint32_t value) const;
  void set_data(const Object& value) const;
  static FunctionPtr New(Heap::Space space = Heap::kOld);

  void PrintSignatureParameters(Thread* thread,
                                Zone* zone,
                                NameVisibility name_visibility,
                                BaseTextBuffer* printer) const;

  // Returns true if the type of the formal parameter at the given position in
  // this function is contravariant with the type of the other formal parameter
  // at the given position in the other function.
  bool IsContravariantParameter(intptr_t parameter_position,
                                const Function& other,
                                intptr_t other_parameter_position,
                                Heap::Space space) const;

  // Returns the index in the parameter names array of the corresponding flag
  // for the given parametere index. Also returns (via flag_mask) the
  // corresponding mask within the flag.
  intptr_t GetRequiredFlagIndex(intptr_t index, intptr_t* flag_mask) const;

  FINAL_HEAP_OBJECT_IMPLEMENTATION(Function, Object);
  friend class Class;
  friend class SnapshotWriter;
  friend class Parser;  // For set_eval_script.
  friend class ProgramVisitor;  // For set_parameter_names.
  // FunctionLayout::VisitFunctionPointers accesses the private constructor of
  // Function.
  friend class FunctionLayout;
  friend class ClassFinalizer;  // To reset parent_function.
  friend class Type;            // To adjust parent_function.
};
```



## object.cc

```c++
intptr_t Function::Hash() const {
  return String::HashRawSymbol(name());
}

bool Function::HasBreakpoint() const {
#if defined(PRODUCT)
  return false;
#else
  Thread* thread = Thread::Current();
  return thread->isolate()->debugger()->HasBreakpoint(*this, thread->zone());
#endif
}

void Function::InstallOptimizedCode(const Code& code) const {
  DEBUG_ASSERT(IsMutatorOrAtSafepoint());
  // We may not have previous code if FLAG_precompile is set.
  // Hot-reload may have already disabled the current code.
  if (HasCode() && !Code::Handle(CurrentCode()).IsDisabled()) {
    Code::Handle(CurrentCode()).DisableDartCode();
  }
  AttachCode(code);
}

void Function::SetInstructions(const Code& value) const {
  DEBUG_ASSERT(IsMutatorOrAtSafepoint());
  SetInstructionsSafe(value);
}

void Function::SetInstructionsSafe(const Code& value) const {
  StorePointer(&raw_ptr()->code_, value.raw());
  StoreNonPointer(&raw_ptr()->entry_point_, value.EntryPoint());
  StoreNonPointer(&raw_ptr()->unchecked_entry_point_,
                  value.UncheckedEntryPoint());
}

void Function::AttachCode(const Code& value) const {
  DEBUG_ASSERT(IsMutatorOrAtSafepoint());
  // Finish setting up code before activating it.
  value.set_owner(*this);
  SetInstructions(value);
  ASSERT(Function::Handle(value.function()).IsNull() ||
         (value.function() == this->raw()));
}

bool Function::HasCode() const {
  NoSafepointScope no_safepoint;
  ASSERT(raw_ptr()->code_ != Code::null());
#if defined(DART_PRECOMPILED_RUNTIME)
  return raw_ptr()->code_ != StubCode::LazyCompile().raw();
#else
  return raw_ptr()->code_ != StubCode::LazyCompile().raw() &&
         raw_ptr()->code_ != StubCode::InterpretCall().raw();
#endif  // defined(DART_PRECOMPILED_RUNTIME)
}

#if !defined(DART_PRECOMPILED_RUNTIME)
bool Function::IsBytecodeAllowed(Zone* zone) const {
  if (FLAG_intrinsify) {
    // Bigint intrinsics should not be interpreted, because their Dart version
    // is only to be used when intrinsics are disabled. Mixing an interpreted
    // Dart version with a compiled intrinsified version results in a mismatch
    // in the number of digits processed by each call.
    switch (recognized_kind()) {
      case MethodRecognizer::kBigint_lsh:
      case MethodRecognizer::kBigint_rsh:
      case MethodRecognizer::kBigint_absAdd:
      case MethodRecognizer::kBigint_absSub:
      case MethodRecognizer::kBigint_mulAdd:
      case MethodRecognizer::kBigint_sqrAdd:
      case MethodRecognizer::kBigint_estimateQuotientDigit:
      case MethodRecognizer::kMontgomery_mulMod:
        return false;
      default:
        break;
    }
  }
  switch (kind()) {
    case FunctionLayout::kDynamicInvocationForwarder:
      return is_declared_in_bytecode();
    case FunctionLayout::kImplicitClosureFunction:
    case FunctionLayout::kIrregexpFunction:
    case FunctionLayout::kFfiTrampoline:
      return false;
    default:
      return true;
  }
}

void Function::AttachBytecode(const Bytecode& value) const {
  DEBUG_ASSERT(IsMutatorOrAtSafepoint());
  ASSERT(!value.IsNull());
  // Finish setting up code before activating it.
  if (!value.InVMIsolateHeap()) {
    value.set_function(*this);
  }
  StorePointer(&raw_ptr()->bytecode_, value.raw());

  // We should not have loaded the bytecode if the function had code.
  // However, we may load the bytecode to access source positions (see
  // ProcessBytecodeTokenPositionsEntry in kernel.cc).
  // In that case, do not install InterpretCall stub below.
  if (FLAG_enable_interpreter && !HasCode()) {
    // Set the code entry_point to InterpretCall stub.
    SetInstructions(StubCode::InterpretCall());
  }
}
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

bool Function::HasCode(FunctionPtr function) {
  NoSafepointScope no_safepoint;
  ASSERT(function->ptr()->code_ != Code::null());
#if defined(DART_PRECOMPILED_RUNTIME)
  return function->ptr()->code_ != StubCode::LazyCompile().raw();
#else
  return function->ptr()->code_ != StubCode::LazyCompile().raw() &&
         function->ptr()->code_ != StubCode::InterpretCall().raw();
#endif  // !defined(DART_PRECOMPILED_RUNTIME)
}

void Function::ClearCode() const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  ASSERT(Thread::Current()->IsMutatorThread());

  StorePointer(&raw_ptr()->unoptimized_code_, Code::null());

  if (FLAG_enable_interpreter && HasBytecode()) {
    SetInstructions(StubCode::InterpretCall());
  } else {
    SetInstructions(StubCode::LazyCompile());
  }
#endif  // defined(DART_PRECOMPILED_RUNTIME)
}

void Function::ClearBytecode() const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  StorePointer(&raw_ptr()->bytecode_, Bytecode::null());
#endif  // defined(DART_PRECOMPILED_RUNTIME)
}

void Function::EnsureHasCompiledUnoptimizedCode() const {
  ASSERT(!ForceOptimize());
  Thread* thread = Thread::Current();
  ASSERT(thread->IsMutatorThread());
  DEBUG_ASSERT(thread->TopErrorHandlerIsExitFrame());
  Zone* zone = thread->zone();

  const Error& error =
      Error::Handle(zone, Compiler::EnsureUnoptimizedCode(thread, *this));
  if (!error.IsNull()) {
    Exceptions::PropagateError(error);
  }
}

void Function::SwitchToUnoptimizedCode() const {
  ASSERT(HasOptimizedCode());
  Thread* thread = Thread::Current();
  Isolate* isolate = thread->isolate();
  Zone* zone = thread->zone();
  ASSERT(thread->IsMutatorThread());
  // TODO(35224): DEBUG_ASSERT(thread->TopErrorHandlerIsExitFrame());
  const Code& current_code = Code::Handle(zone, CurrentCode());

  if (FLAG_trace_deoptimization_verbose) {
    THR_Print("Disabling optimized code: '%s' entry: %#" Px "\n",
              ToFullyQualifiedCString(), current_code.EntryPoint());
  }
  current_code.DisableDartCode();
  const Error& error =
      Error::Handle(zone, Compiler::EnsureUnoptimizedCode(thread, *this));
  if (!error.IsNull()) {
    Exceptions::PropagateError(error);
  }
  const Code& unopt_code = Code::Handle(zone, unoptimized_code());
  unopt_code.Enable();
  AttachCode(unopt_code);
  isolate->TrackDeoptimizedCode(current_code);
}

void Function::SwitchToLazyCompiledUnoptimizedCode() const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  if (!HasOptimizedCode()) {
    return;
  }

  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  ASSERT(thread->IsMutatorThread());

  const Code& current_code = Code::Handle(zone, CurrentCode());
  TIR_Print("Disabling optimized code for %s\n", ToCString());
  current_code.DisableDartCode();

  const Code& unopt_code = Code::Handle(zone, unoptimized_code());
  if (unopt_code.IsNull()) {
    // Set the lazy compile or interpreter call stub code.
    if (FLAG_enable_interpreter && HasBytecode()) {
      TIR_Print("Switched to interpreter call stub for %s\n", ToCString());
      SetInstructions(StubCode::InterpretCall());
    } else {
      TIR_Print("Switched to lazy compile stub for %s\n", ToCString());
      SetInstructions(StubCode::LazyCompile());
    }
    return;
  }

  TIR_Print("Switched to unoptimized code for %s\n", ToCString());

  AttachCode(unopt_code);
  unopt_code.Enable();
#endif
}

void Function::set_unoptimized_code(const Code& value) const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  DEBUG_ASSERT(IsMutatorOrAtSafepoint());
  ASSERT(value.IsNull() || !value.is_optimized());
  StorePointer(&raw_ptr()->unoptimized_code_, value.raw());
#endif
}

ContextScopePtr Function::context_scope() const {
  if (IsClosureFunction()) {
    const Object& obj = Object::Handle(raw_ptr()->data_);
    ASSERT(!obj.IsNull());
    return ClosureData::Cast(obj).context_scope();
  }
  return ContextScope::null();
}

void Function::set_context_scope(const ContextScope& value) const {
  if (IsClosureFunction()) {
    const Object& obj = Object::Handle(raw_ptr()->data_);
    ASSERT(!obj.IsNull());
    ClosureData::Cast(obj).set_context_scope(value);
    return;
  }
  UNREACHABLE();
}

InstancePtr Function::implicit_static_closure() const {
  if (IsImplicitStaticClosureFunction()) {
    const Object& obj = Object::Handle(raw_ptr()->data_);
    ASSERT(!obj.IsNull());
    return ClosureData::Cast(obj).implicit_static_closure();
  }
  return Instance::null();
}

void Function::set_implicit_static_closure(const Instance& closure) const {
  if (IsImplicitStaticClosureFunction()) {
    const Object& obj = Object::Handle(raw_ptr()->data_);
    ASSERT(!obj.IsNull());
    ClosureData::Cast(obj).set_implicit_static_closure(closure);
    return;
  }
  UNREACHABLE();
}

ScriptPtr Function::eval_script() const {
  const Object& obj = Object::Handle(raw_ptr()->data_);
  if (obj.IsScript()) {
    return Script::Cast(obj).raw();
  }
  return Script::null();
}

void Function::set_eval_script(const Script& script) const {
  ASSERT(token_pos() == TokenPosition::kMinSource);
  ASSERT(raw_ptr()->data_ == Object::null());
  set_data(script);
}

FunctionPtr Function::extracted_method_closure() const {
  ASSERT(kind() == FunctionLayout::kMethodExtractor);
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(obj.IsFunction());
  return Function::Cast(obj).raw();
}

void Function::set_extracted_method_closure(const Function& value) const {
  ASSERT(kind() == FunctionLayout::kMethodExtractor);
  ASSERT(raw_ptr()->data_ == Object::null());
  set_data(value);
}

ArrayPtr Function::saved_args_desc() const {
  ASSERT(kind() == FunctionLayout::kNoSuchMethodDispatcher ||
         kind() == FunctionLayout::kInvokeFieldDispatcher);
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(obj.IsArray());
  return Array::Cast(obj).raw();
}

void Function::set_saved_args_desc(const Array& value) const {
  ASSERT(kind() == FunctionLayout::kNoSuchMethodDispatcher ||
         kind() == FunctionLayout::kInvokeFieldDispatcher);
  ASSERT(raw_ptr()->data_ == Object::null());
  set_data(value);
}

FieldPtr Function::accessor_field() const {
  ASSERT(kind() == FunctionLayout::kImplicitGetter ||
         kind() == FunctionLayout::kImplicitSetter ||
         kind() == FunctionLayout::kImplicitStaticGetter ||
         kind() == FunctionLayout::kFieldInitializer);
  return Field::RawCast(raw_ptr()->data_);
}

void Function::set_accessor_field(const Field& value) const {
  ASSERT(kind() == FunctionLayout::kImplicitGetter ||
         kind() == FunctionLayout::kImplicitSetter ||
         kind() == FunctionLayout::kImplicitStaticGetter ||
         kind() == FunctionLayout::kFieldInitializer);
  // Top level classes may be finalized multiple times.
  ASSERT(raw_ptr()->data_ == Object::null() || raw_ptr()->data_ == value.raw());
  set_data(value);
}

FunctionPtr Function::parent_function() const {
  if (IsClosureFunction() || IsSignatureFunction()) {
    const Object& obj = Object::Handle(raw_ptr()->data_);
    ASSERT(!obj.IsNull());
    if (IsClosureFunction()) {
      return ClosureData::Cast(obj).parent_function();
    } else {
      return SignatureData::Cast(obj).parent_function();
    }
  }
  return Function::null();
}

void Function::set_parent_function(const Function& value) const {
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  if (IsClosureFunction()) {
    ClosureData::Cast(obj).set_parent_function(value);
  } else {
    ASSERT(IsSignatureFunction());
    SignatureData::Cast(obj).set_parent_function(value);
  }
}

FunctionPtr Function::GetGeneratedClosure() const {
  const auto& closure_functions = GrowableObjectArray::Handle(
      Isolate::Current()->object_store()->closure_functions());
  auto& entry = Object::Handle();

  for (auto i = (closure_functions.Length() - 1); i >= 0; i--) {
    entry = closure_functions.At(i);

    ASSERT(entry.IsFunction());

    const auto& closure_function = Function::Cast(entry);
    if (closure_function.parent_function() == raw() &&
        closure_function.is_generated_body()) {
      return closure_function.raw();
    }
  }

  return Function::null();
}

// Enclosing outermost function of this local function.
FunctionPtr Function::GetOutermostFunction() const {
  FunctionPtr parent = parent_function();
  if (parent == Object::null()) {
    return raw();
  }
  Function& function = Function::Handle();
  do {
    function = parent;
    parent = function.parent_function();
  } while (parent != Object::null());
  return function.raw();
}

bool Function::HasGenericParent() const {
  if (IsImplicitClosureFunction()) {
    // The parent function of an implicit closure function is not the enclosing
    // function we are asking about here.
    return false;
  }
  Function& parent = Function::Handle(parent_function());
  while (!parent.IsNull()) {
    if (parent.IsGeneric()) {
      return true;
    }
    parent = parent.parent_function();
  }
  return false;
}

FunctionPtr Function::implicit_closure_function() const {
  if (IsClosureFunction() || IsSignatureFunction() || IsFactory() ||
      IsDispatcherOrImplicitAccessor() || IsFieldInitializer()) {
    return Function::null();
  }
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(obj.IsNull() || obj.IsScript() || obj.IsFunction() || obj.IsArray());
  if (obj.IsNull() || obj.IsScript()) {
    return Function::null();
  }
  if (obj.IsFunction()) {
    return Function::Cast(obj).raw();
  }
  ASSERT(is_native());
  ASSERT(obj.IsArray());
  const Object& res = Object::Handle(Array::Cast(obj).At(1));
  return res.IsNull() ? Function::null() : Function::Cast(res).raw();
}

void Function::set_implicit_closure_function(const Function& value) const {
  ASSERT(!IsClosureFunction() && !IsSignatureFunction());
  const Object& old_data = Object::Handle(raw_ptr()->data_);
  if (is_native()) {
    ASSERT(old_data.IsArray());
    ASSERT((Array::Cast(old_data).At(1) == Object::null()) || value.IsNull());
    Array::Cast(old_data).SetAt(1, value);
  } else {
    // Maybe this function will turn into a native later on :-/
    if (old_data.IsArray()) {
      ASSERT((Array::Cast(old_data).At(1) == Object::null()) || value.IsNull());
      Array::Cast(old_data).SetAt(1, value);
    } else {
      ASSERT(old_data.IsNull() || value.IsNull());
      set_data(value);
    }
  }
}

TypePtr Function::ExistingSignatureType() const {
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  if (IsSignatureFunction()) {
    return SignatureData::Cast(obj).signature_type();
  } else if (IsClosureFunction()) {
    return ClosureData::Cast(obj).signature_type();
  } else {
    ASSERT(IsFfiTrampoline());
    return FfiTrampolineData::Cast(obj).signature_type();
  }
}

void Function::SetFfiCSignature(const Function& sig) const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  FfiTrampolineData::Cast(obj).set_c_signature(sig);
}

FunctionPtr Function::FfiCSignature() const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return FfiTrampolineData::Cast(obj).c_signature();
}

bool Function::FfiCSignatureContainsHandles() const {
  ASSERT(IsFfiTrampoline());
  const Function& c_signature = Function::Handle(FfiCSignature());
  const intptr_t num_params = c_signature.num_fixed_parameters();
  for (intptr_t i = 0; i < num_params; i++) {
    const bool is_handle =
        AbstractType::Handle(c_signature.ParameterTypeAt(i)).type_class_id() ==
        kFfiHandleCid;
    if (is_handle) {
      return true;
    }
  }
  return AbstractType::Handle(c_signature.result_type()).type_class_id() ==
         kFfiHandleCid;
}

int32_t Function::FfiCallbackId() const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return FfiTrampolineData::Cast(obj).callback_id();
}

void Function::SetFfiCallbackId(int32_t value) const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  FfiTrampolineData::Cast(obj).set_callback_id(value);
}

FunctionPtr Function::FfiCallbackTarget() const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return FfiTrampolineData::Cast(obj).callback_target();
}

void Function::SetFfiCallbackTarget(const Function& target) const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  FfiTrampolineData::Cast(obj).set_callback_target(target);
}

InstancePtr Function::FfiCallbackExceptionalReturn() const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return FfiTrampolineData::Cast(obj).callback_exceptional_return();
}

void Function::SetFfiCallbackExceptionalReturn(const Instance& value) const {
  ASSERT(IsFfiTrampoline());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  FfiTrampolineData::Cast(obj).set_callback_exceptional_return(value);
}

TypePtr Function::SignatureType(Nullability nullability) const {
  Type& type = Type::Handle(ExistingSignatureType());
  if (type.IsNull()) {
    // The function type of this function is not yet cached and needs to be
    // constructed and cached here.
    // A function type is type parameterized in the same way as the owner class
    // of its non-static signature function.
    // It is not type parameterized if its signature function is static, or if
    // none of its result type or formal parameter types are type parameterized.
    // Unless the function type is a generic typedef, the type arguments of the
    // function type are not explicitly stored in the function type as a vector
    // of type arguments.
    // The type class of a non-typedef function type is always the non-generic
    // _Closure class, whether the type is generic or not.
    // The type class of a typedef function type is always the typedef class,
    // which may be generic, in which case the type stores type arguments.
    // With the introduction of generic functions, we may reach here before the
    // function type parameters have been resolved. Therefore, we cannot yet
    // check whether the function type has an instantiated signature.
    // We can do it only when the signature has been resolved.
    // We only set the type class of the function type to the typedef class
    // if the signature of the function type is the signature of the typedef.
    // Note that a function type can have a typedef class as owner without
    // representing the typedef, as in the following example:
    // typedef F(f(int x)); where the type of f is a function type with F as
    // owner, without representing the function type of F.
    Class& scope_class = Class::Handle(Owner());
    if (!scope_class.IsTypedefClass() ||
        (scope_class.signature_function() != raw())) {
      scope_class = Isolate::Current()->object_store()->closure_class();
    }
    const TypeArguments& signature_type_arguments =
        TypeArguments::Handle(scope_class.type_parameters());
    // Return the still unfinalized signature type.
    type = Type::New(scope_class, signature_type_arguments, token_pos(),
                     nullability);
    type.set_signature(*this);
    SetSignatureType(type);
  }
  return type.ToNullability(nullability, Heap::kOld);
}

void Function::SetSignatureType(const Type& value) const {
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  if (IsSignatureFunction()) {
    SignatureData::Cast(obj).set_signature_type(value);
    ASSERT(!value.IsCanonical() || (value.signature() == this->raw()));
  } else if (IsClosureFunction()) {
    ClosureData::Cast(obj).set_signature_type(value);
  } else {
    ASSERT(IsFfiTrampoline());
    FfiTrampolineData::Cast(obj).set_signature_type(value);
  }
}

bool Function::IsRedirectingFactory() const {
  if (!IsFactory() || !is_redirecting()) {
    return false;
  }
  ASSERT(!IsClosureFunction());  // A factory cannot also be a closure.
  return true;
}

TypePtr Function::RedirectionType() const {
  ASSERT(IsRedirectingFactory());
  ASSERT(!is_native());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return RedirectionData::Cast(obj).type();
}

const char* Function::KindToCString(FunctionLayout::Kind kind) {
  return FunctionLayout::KindToCString(kind);
}

void Function::SetRedirectionType(const Type& type) const {
  ASSERT(IsFactory());
  Object& obj = Object::Handle(raw_ptr()->data_);
  if (obj.IsNull()) {
    obj = RedirectionData::New();
    set_data(obj);
  }
  RedirectionData::Cast(obj).set_type(type);
}

StringPtr Function::RedirectionIdentifier() const {
  ASSERT(IsRedirectingFactory());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return RedirectionData::Cast(obj).identifier();
}

void Function::SetRedirectionIdentifier(const String& identifier) const {
  ASSERT(IsFactory());
  Object& obj = Object::Handle(raw_ptr()->data_);
  if (obj.IsNull()) {
    obj = RedirectionData::New();
    set_data(obj);
  }
  RedirectionData::Cast(obj).set_identifier(identifier);
}

FunctionPtr Function::RedirectionTarget() const {
  ASSERT(IsRedirectingFactory());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(!obj.IsNull());
  return RedirectionData::Cast(obj).target();
}

void Function::SetRedirectionTarget(const Function& target) const {
  ASSERT(IsFactory());
  Object& obj = Object::Handle(raw_ptr()->data_);
  if (obj.IsNull()) {
    obj = RedirectionData::New();
    set_data(obj);
  }
  RedirectionData::Cast(obj).set_target(target);
}

FunctionPtr Function::ForwardingTarget() const {
  ASSERT(kind() == FunctionLayout::kDynamicInvocationForwarder);
  Array& checks = Array::Handle();
  checks ^= raw_ptr()->data_;
  return Function::RawCast(checks.At(0));
}

void Function::SetForwardingChecks(const Array& checks) const {
  ASSERT(kind() == FunctionLayout::kDynamicInvocationForwarder);
  ASSERT(checks.Length() >= 1);
  ASSERT(Object::Handle(checks.At(0)).IsFunction());
  set_data(checks);
}

// This field is heavily overloaded:
//   eval function:           Script expression source
//   kernel eval function:    Array[0] = Script
//                            Array[1] = Kernel data
//                            Array[2] = Kernel offset of enclosing library
//   signature function:      SignatureData
//   method extractor:        Function extracted closure function
//   implicit getter:         Field
//   implicit setter:         Field
//   impl. static final gttr: Field
//   field initializer:       Field
//   noSuchMethod dispatcher: Array arguments descriptor
//   invoke-field dispatcher: Array arguments descriptor
//   redirecting constructor: RedirectionData
//   closure function:        ClosureData
//   irregexp function:       Array[0] = RegExp
//                            Array[1] = Smi string specialization cid
//   native function:         Array[0] = String native name
//                            Array[1] = Function implicit closure function
//   regular function:        Function for implicit closure function
//   ffi trampoline function: FfiTrampolineData  (Dart->C)
//   dyn inv forwarder:       Array[0] = Function target
//                            Array[1] = TypeArguments default type args
//                            Array[i] = ParameterTypeCheck
void Function::set_data(const Object& value) const {
  StorePointer(&raw_ptr()->data_, value.raw());
}

bool Function::IsInFactoryScope() const {
  if (!IsLocalFunction()) {
    return IsFactory();
  }
  Function& outer_function = Function::Handle(parent_function());
  while (outer_function.IsLocalFunction()) {
    outer_function = outer_function.parent_function();
  }
  return outer_function.IsFactory();
}

void Function::set_name(const String& value) const {
  ASSERT(value.IsSymbol());
  StorePointer(&raw_ptr()->name_, value.raw());
}

void Function::set_owner(const Object& value) const {
  ASSERT(!value.IsNull() || IsSignatureFunction());
  StorePointer(&raw_ptr()->owner_, value.raw());
}

RegExpPtr Function::regexp() const {
  ASSERT(kind() == FunctionLayout::kIrregexpFunction);
  const Array& pair = Array::Cast(Object::Handle(raw_ptr()->data_));
  return RegExp::RawCast(pair.At(0));
}

class StickySpecialization : public BitField<intptr_t, bool, 0, 1> {};
class StringSpecializationCid
    : public BitField<intptr_t, intptr_t, 1, ObjectLayout::kClassIdTagSize> {};

intptr_t Function::string_specialization_cid() const {
  ASSERT(kind() == FunctionLayout::kIrregexpFunction);
  const Array& pair = Array::Cast(Object::Handle(raw_ptr()->data_));
  return StringSpecializationCid::decode(Smi::Value(Smi::RawCast(pair.At(1))));
}

bool Function::is_sticky_specialization() const {
  ASSERT(kind() == FunctionLayout::kIrregexpFunction);
  const Array& pair = Array::Cast(Object::Handle(raw_ptr()->data_));
  return StickySpecialization::decode(Smi::Value(Smi::RawCast(pair.At(1))));
}

void Function::SetRegExpData(const RegExp& regexp,
                             intptr_t string_specialization_cid,
                             bool sticky) const {
  ASSERT(kind() == FunctionLayout::kIrregexpFunction);
  ASSERT(IsStringClassId(string_specialization_cid));
  ASSERT(raw_ptr()->data_ == Object::null());
  const Array& pair = Array::Handle(Array::New(2, Heap::kOld));
  pair.SetAt(0, regexp);
  pair.SetAt(1, Smi::Handle(Smi::New(StickySpecialization::encode(sticky) |
                                     StringSpecializationCid::encode(
                                         string_specialization_cid))));
  set_data(pair);
}

StringPtr Function::native_name() const {
  ASSERT(is_native());
  const Object& obj = Object::Handle(raw_ptr()->data_);
  ASSERT(obj.IsArray());
  return String::RawCast(Array::Cast(obj).At(0));
}

void Function::set_native_name(const String& value) const {
  Zone* zone = Thread::Current()->zone();
  ASSERT(is_native());

  // Due to the fact that kernel needs to read in the constant table before the
  // annotation data is available, we don't know at function creation time
  // whether the function is a native or not.
  //
  // Reading the constant table can cause a static function to get an implicit
  // closure function.
  //
  // We therefore handle both cases.
  const Object& old_data = Object::Handle(zone, raw_ptr()->data_);
  ASSERT(old_data.IsNull() ||
         (old_data.IsFunction() &&
          Function::Handle(zone, Function::RawCast(old_data.raw()))
              .IsImplicitClosureFunction()));

  const Array& pair = Array::Handle(zone, Array::New(2, Heap::kOld));
  pair.SetAt(0, value);
  pair.SetAt(1, old_data);  // will be the implicit closure function if needed.
  set_data(pair);
}

void Function::set_result_type(const AbstractType& value) const {
  ASSERT(!value.IsNull());
  StorePointer(&raw_ptr()->result_type_, value.raw());
}

AbstractTypePtr Function::ParameterTypeAt(intptr_t index) const {
  const Array& parameter_types = Array::Handle(raw_ptr()->parameter_types_);
  return AbstractType::RawCast(parameter_types.At(index));
}

void Function::SetParameterTypeAt(intptr_t index,
                                  const AbstractType& value) const {
  ASSERT(!value.IsNull());
  // Method extractor parameters are shared and are in the VM heap.
  ASSERT(kind() != FunctionLayout::kMethodExtractor);
  const Array& parameter_types = Array::Handle(raw_ptr()->parameter_types_);
  parameter_types.SetAt(index, value);
}

void Function::set_parameter_types(const Array& value) const {
  StorePointer(&raw_ptr()->parameter_types_, value.raw());
}

StringPtr Function::ParameterNameAt(intptr_t index) const {
  const Array& parameter_names = Array::Handle(raw_ptr()->parameter_names_);
  return String::RawCast(parameter_names.At(index));
}

void Function::SetParameterNameAt(intptr_t index, const String& value) const {
  ASSERT(!value.IsNull() && value.IsSymbol());
  const Array& parameter_names = Array::Handle(raw_ptr()->parameter_names_);
  parameter_names.SetAt(index, value);
}

void Function::set_parameter_names(const Array& value) const {
  StorePointer(&raw_ptr()->parameter_names_, value.raw());
}

void Function::CreateNameArrayIncludingFlags(Heap::Space space) const {
  // Currently, we only store flags for named parameters that are required.
  const intptr_t num_parameters = NumParameters();
  intptr_t num_total_slots = num_parameters;
  if (HasOptionalNamedParameters()) {
    const intptr_t last_index = (NumOptionalNamedParameters() - 1) /
                                compiler::target::kNumParameterFlagsPerElement;
    const intptr_t num_flag_slots = last_index + 1;
    num_total_slots += num_flag_slots;
  }
  auto& array = Array::Handle(Array::New(num_total_slots, space));
  if (num_total_slots > num_parameters) {
    // Set flag slots to Smi 0 before handing off.
    auto& empty_flags_smi = Smi::Handle(Smi::New(0));
    for (intptr_t i = num_parameters; i < num_total_slots; i++) {
      array.SetAt(i, empty_flags_smi);
    }
  }
  set_parameter_names(array);
}

intptr_t Function::GetRequiredFlagIndex(intptr_t index,
                                        intptr_t* flag_mask) const {
  // If these calculations change, also change
  // FlowGraphBuilder::BuildClosureCallHasRequiredNamedArgumentsCheck.
  ASSERT(flag_mask != nullptr);
  ASSERT(index >= num_fixed_parameters());
  index -= num_fixed_parameters();
  *flag_mask = (1 << compiler::target::kRequiredNamedParameterFlag)
               << ((static_cast<uintptr_t>(index) %
                    compiler::target::kNumParameterFlagsPerElement) *
                   compiler::target::kNumParameterFlags);
  return NumParameters() +
         index / compiler::target::kNumParameterFlagsPerElement;
}

bool Function::IsRequiredAt(intptr_t index) const {
  if (index < num_fixed_parameters() + NumOptionalPositionalParameters()) {
    return false;
  }
  intptr_t flag_mask;
  const intptr_t flag_index = GetRequiredFlagIndex(index, &flag_mask);
  const Array& parameter_names = Array::Handle(raw_ptr()->parameter_names_);
  if (flag_index >= parameter_names.Length()) {
    return false;
  }
  const intptr_t flags =
      Smi::Value(Smi::RawCast(parameter_names.At(flag_index)));
  return (flags & flag_mask) != 0;
}

void Function::SetIsRequiredAt(intptr_t index) const {
  intptr_t flag_mask;
  const intptr_t flag_index = GetRequiredFlagIndex(index, &flag_mask);
  const Array& parameter_names = Array::Handle(raw_ptr()->parameter_names_);
  ASSERT(flag_index < parameter_names.Length());
  const intptr_t flags =
      Smi::Value(Smi::RawCast(parameter_names.At(flag_index)));
  parameter_names.SetAt(flag_index, Smi::Handle(Smi::New(flags | flag_mask)));
}

void Function::TruncateUnusedParameterFlags() const {
  const Array& parameter_names = Array::Handle(raw_ptr()->parameter_names_);
  const intptr_t num_params = NumParameters();
  if (parameter_names.Length() == num_params) {
    // No flag slots to truncate.
    return;
  }
  // Truncate the parameter names array to remove unused flags from the end.
  intptr_t last_used = parameter_names.Length() - 1;
  for (; last_used >= num_params; --last_used) {
    if (Smi::Value(Smi::RawCast(parameter_names.At(last_used))) != 0) {
      break;
    }
  }
  parameter_names.Truncate(last_used + 1);
}

void Function::set_type_parameters(const TypeArguments& value) const {
  StorePointer(&raw_ptr()->type_parameters_, value.raw());
}

intptr_t Function::NumTypeParameters(Thread* thread) const {
  if (type_parameters() == TypeArguments::null()) {
    return 0;
  }
  REUSABLE_TYPE_ARGUMENTS_HANDLESCOPE(thread);
  TypeArguments& type_params = thread->TypeArgumentsHandle();
  type_params = type_parameters();
  // We require null to represent a non-generic function.
  ASSERT(type_params.Length() != 0);
  return type_params.Length();
}

intptr_t Function::NumParentTypeParameters() const {
  if (IsImplicitClosureFunction()) {
    return 0;
  }
  Thread* thread = Thread::Current();
  Function& parent = Function::Handle(parent_function());
  intptr_t num_parent_type_params = 0;
  while (!parent.IsNull()) {
    num_parent_type_params += parent.NumTypeParameters(thread);
    if (parent.IsImplicitClosureFunction()) break;
    parent = parent.parent_function();
  }
  return num_parent_type_params;
}

void Function::PrintSignatureTypes() const {
  Function& sig_fun = Function::Handle(raw());
  Type& sig_type = Type::Handle();
  while (!sig_fun.IsNull()) {
    sig_type = sig_fun.SignatureType();
    THR_Print("%s%s\n",
              sig_fun.IsImplicitClosureFunction() ? "implicit closure: " : "",
              sig_type.ToCString());
    sig_fun = sig_fun.parent_function();
  }
}

TypeParameterPtr Function::LookupTypeParameter(const String& type_name,
                                               intptr_t* function_level) const {
  ASSERT(!type_name.IsNull());
  Thread* thread = Thread::Current();
  REUSABLE_TYPE_ARGUMENTS_HANDLESCOPE(thread);
  REUSABLE_TYPE_PARAMETER_HANDLESCOPE(thread);
  REUSABLE_STRING_HANDLESCOPE(thread);
  REUSABLE_FUNCTION_HANDLESCOPE(thread);
  TypeArguments& type_params = thread->TypeArgumentsHandle();
  TypeParameter& type_param = thread->TypeParameterHandle();
  String& type_param_name = thread->StringHandle();
  Function& function = thread->FunctionHandle();

  function = this->raw();
  while (!function.IsNull()) {
    type_params = function.type_parameters();
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
    if (function.IsImplicitClosureFunction()) {
      // The parent function is not the enclosing function, but the closurized
      // function with identical type parameters.
      break;
    }
    function = function.parent_function();
    if (function_level != NULL) {
      (*function_level)--;
    }
  }
  return TypeParameter::null();
}

void Function::set_kind(FunctionLayout::Kind value) const {
  set_kind_tag(KindBits::update(value, raw_ptr()->kind_tag_));
}

void Function::set_modifier(FunctionLayout::AsyncModifier value) const {
  set_kind_tag(ModifierBits::update(value, raw_ptr()->kind_tag_));
}

void Function::set_recognized_kind(MethodRecognizer::Kind value) const {
  // Prevent multiple settings of kind.
  ASSERT((value == MethodRecognizer::kUnknown) || !IsRecognized());
  set_kind_tag(RecognizedBits::update(value, raw_ptr()->kind_tag_));
}

void Function::set_token_pos(TokenPosition token_pos) const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  ASSERT(!token_pos.IsClassifying() || IsMethodExtractor());
  StoreNonPointer(&raw_ptr()->token_pos_, token_pos);
#endif
}

void Function::set_kind_tag(uint32_t value) const {
  StoreNonPointer(&raw_ptr()->kind_tag_, static_cast<uint32_t>(value));
}

void Function::set_packed_fields(uint32_t packed_fields) const {
  StoreNonPointer(&raw_ptr()->packed_fields_, packed_fields);
}

void Function::set_num_fixed_parameters(intptr_t value) const {
  ASSERT(value >= 0);
  ASSERT(Utils::IsUint(FunctionLayout::kMaxFixedParametersBits, value));
  const uint32_t* original = &raw_ptr()->packed_fields_;
  StoreNonPointer(original, FunctionLayout::PackedNumFixedParameters::update(
                                value, *original));
}

void Function::SetNumOptionalParameters(intptr_t value,
                                        bool are_optional_positional) const {
  ASSERT(Utils::IsUint(FunctionLayout::kMaxOptionalParametersBits, value));
  uint32_t packed_fields = raw_ptr()->packed_fields_;
  packed_fields = FunctionLayout::PackedHasNamedOptionalParameters::update(
      !are_optional_positional, packed_fields);
  packed_fields =
      FunctionLayout::PackedNumOptionalParameters::update(value, packed_fields);
  set_packed_fields(packed_fields);
}

bool Function::IsOptimizable() const {
  if (FLAG_precompiled_mode) {
    return true;
  }
  if (ForceOptimize()) return true;
  if (is_native()) {
    // Native methods don't need to be optimized.
    return false;
  }
  const intptr_t function_length = end_token_pos().Pos() - token_pos().Pos();
  if (is_optimizable() && (script() != Script::null()) &&
      (function_length < FLAG_huge_method_cutoff_in_tokens)) {
    // Additional check needed for implicit getters.
    return (unoptimized_code() == Object::null()) ||
           (Code::Handle(unoptimized_code()).Size() <
            FLAG_huge_method_cutoff_in_code_size);
  }
  return false;
}

void Function::SetIsOptimizable(bool value) const {
  ASSERT(!is_native());
  set_is_optimizable(value);
  if (!value) {
    set_is_inlinable(false);
    set_usage_counter(INT32_MIN);
  }
}

#if !defined(DART_PRECOMPILED_RUNTIME)
bool Function::CanBeInlined() const {
  // Our force-optimized functions cannot deoptimize to an unoptimized frame.
  // If the instructions of the force-optimized function body get moved via
  // code motion, we might attempt do deoptimize a frame where the force-
  // optimized function has only partially finished. Since force-optimized
  // functions cannot deoptimize to unoptimized frames we prevent them from
  // being inlined (for now).
  if (ForceOptimize()) {
    if (IsFfiTrampoline()) {
      // The CallSiteInliner::InlineCall asserts in PrepareGraphs that
      // GraphEntryInstr::SuccessorCount() == 1, but FFI trampoline has two
      // entries (a normal and a catch entry).
      return false;
    }
    return CompilerState::Current().is_aot();
  }

#if !defined(PRODUCT)
  Thread* thread = Thread::Current();
  if (thread->isolate()->debugger()->HasBreakpoint(*this, thread->zone())) {
    return false;
  }
#endif  // !defined(PRODUCT)

  return is_inlinable() && !is_external() && !is_generated_body();
}
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

intptr_t Function::NumParameters() const {
  return num_fixed_parameters() + NumOptionalParameters();
}

intptr_t Function::NumImplicitParameters() const {
  const FunctionLayout::Kind k = kind();
  if (k == FunctionLayout::kConstructor) {
    // Type arguments for factory; instance for generative constructor.
    return 1;
  }
  if ((k == FunctionLayout::kClosureFunction) ||
      (k == FunctionLayout::kImplicitClosureFunction) ||
      (k == FunctionLayout::kSignatureFunction) ||
      (k == FunctionLayout::kFfiTrampoline)) {
    return 1;  // Closure object.
  }
  if (!is_static()) {
    // Closure functions defined inside instance (i.e. non-static) functions are
    // marked as non-static, but they do not have a receiver.
    // Closures are handled above.
    ASSERT((k != FunctionLayout::kClosureFunction) &&
           (k != FunctionLayout::kImplicitClosureFunction) &&
           (k != FunctionLayout::kSignatureFunction));
    return 1;  // Receiver.
  }
  return 0;  // No implicit parameters.
}

bool Function::AreValidArgumentCounts(intptr_t num_type_arguments,
                                      intptr_t num_arguments,
                                      intptr_t num_named_arguments,
                                      String* error_message) const {
  if ((num_type_arguments != 0) &&
      (num_type_arguments != NumTypeParameters())) {
    if (error_message != NULL) {
      const intptr_t kMessageBufferSize = 64;
      char message_buffer[kMessageBufferSize];
      Utils::SNPrint(message_buffer, kMessageBufferSize,
                     "%" Pd " type arguments passed, but %" Pd " expected",
                     num_type_arguments, NumTypeParameters());
      // Allocate in old space because it can be invoked in background
      // optimizing compilation.
      *error_message = String::New(message_buffer, Heap::kOld);
    }
    return false;  // Too many type arguments.
  }
  if (num_named_arguments > NumOptionalNamedParameters()) {
    if (error_message != NULL) {
      const intptr_t kMessageBufferSize = 64;
      char message_buffer[kMessageBufferSize];
      Utils::SNPrint(message_buffer, kMessageBufferSize,
                     "%" Pd " named passed, at most %" Pd " expected",
                     num_named_arguments, NumOptionalNamedParameters());
      // Allocate in old space because it can be invoked in background
      // optimizing compilation.
      *error_message = String::New(message_buffer, Heap::kOld);
    }
    return false;  // Too many named arguments.
  }
  const intptr_t num_pos_args = num_arguments - num_named_arguments;
  const intptr_t num_opt_pos_params = NumOptionalPositionalParameters();
  const intptr_t num_pos_params = num_fixed_parameters() + num_opt_pos_params;
  if (num_pos_args > num_pos_params) {
    if (error_message != NULL) {
      const intptr_t kMessageBufferSize = 64;
      char message_buffer[kMessageBufferSize];
      // Hide implicit parameters to the user.
      const intptr_t num_hidden_params = NumImplicitParameters();
      Utils::SNPrint(message_buffer, kMessageBufferSize,
                     "%" Pd "%s passed, %s%" Pd " expected",
                     num_pos_args - num_hidden_params,
                     num_opt_pos_params > 0 ? " positional" : "",
                     num_opt_pos_params > 0 ? "at most " : "",
                     num_pos_params - num_hidden_params);
      // Allocate in old space because it can be invoked in background
      // optimizing compilation.
      *error_message = String::New(message_buffer, Heap::kOld);
    }
    return false;  // Too many fixed and/or positional arguments.
  }
  if (num_pos_args < num_fixed_parameters()) {
    if (error_message != NULL) {
      const intptr_t kMessageBufferSize = 64;
      char message_buffer[kMessageBufferSize];
      // Hide implicit parameters to the user.
      const intptr_t num_hidden_params = NumImplicitParameters();
      Utils::SNPrint(message_buffer, kMessageBufferSize,
                     "%" Pd "%s passed, %s%" Pd " expected",
                     num_pos_args - num_hidden_params,
                     num_opt_pos_params > 0 ? " positional" : "",
                     num_opt_pos_params > 0 ? "at least " : "",
                     num_fixed_parameters() - num_hidden_params);
      // Allocate in old space because it can be invoked in background
      // optimizing compilation.
      *error_message = String::New(message_buffer, Heap::kOld);
    }
    return false;  // Too few fixed and/or positional arguments.
  }
  return true;
}

bool Function::AreValidArguments(intptr_t num_type_arguments,
                                 intptr_t num_arguments,
                                 const Array& argument_names,
                                 String* error_message) const {
  const Array& args_desc_array = Array::Handle(ArgumentsDescriptor::NewBoxed(
      num_type_arguments, num_arguments, argument_names, Heap::kNew));
  ArgumentsDescriptor args_desc(args_desc_array);
  return AreValidArguments(args_desc, error_message);
}

bool Function::AreValidArguments(const ArgumentsDescriptor& args_desc,
                                 String* error_message) const {
  const intptr_t num_type_arguments = args_desc.TypeArgsLen();
  const intptr_t num_arguments = args_desc.Count();
  const intptr_t num_named_arguments = args_desc.NamedCount();

  if (!AreValidArgumentCounts(num_type_arguments, num_arguments,
                              num_named_arguments, error_message)) {
    return false;
  }
  // Verify that all argument names are valid parameter names.
  Thread* thread = Thread::Current();
  Isolate* isolate = thread->isolate();
  Zone* zone = thread->zone();
  String& argument_name = String::Handle(zone);
  String& parameter_name = String::Handle(zone);
  const intptr_t num_positional_args = num_arguments - num_named_arguments;
  const intptr_t num_parameters = NumParameters();
  for (intptr_t i = 0; i < num_named_arguments; i++) {
    argument_name = args_desc.NameAt(i);
    ASSERT(argument_name.IsSymbol());
    bool found = false;
    for (intptr_t j = num_positional_args; j < num_parameters; j++) {
      parameter_name = ParameterNameAt(j);
      ASSERT(parameter_name.IsSymbol());
      if (argument_name.Equals(parameter_name)) {
        found = true;
        break;
      }
    }
    if (!found) {
      if (error_message != nullptr) {
        const intptr_t kMessageBufferSize = 64;
        char message_buffer[kMessageBufferSize];
        Utils::SNPrint(message_buffer, kMessageBufferSize,
                       "no optional formal parameter named '%s'",
                       argument_name.ToCString());
        *error_message = String::New(message_buffer);
      }
      return false;
    }
  }
  if (isolate->null_safety()) {
    // Verify that all required named parameters are filled.
    for (intptr_t j = num_parameters - NumOptionalNamedParameters();
         j < num_parameters; j++) {
      if (IsRequiredAt(j)) {
        parameter_name = ParameterNameAt(j);
        ASSERT(parameter_name.IsSymbol());
        bool found = false;
        for (intptr_t i = 0; i < num_named_arguments; i++) {
          argument_name = args_desc.NameAt(i);
          ASSERT(argument_name.IsSymbol());
          if (argument_name.Equals(parameter_name)) {
            found = true;
            break;
          }
        }
        if (!found) {
          if (error_message != nullptr) {
            const intptr_t kMessageBufferSize = 64;
            char message_buffer[kMessageBufferSize];
            Utils::SNPrint(message_buffer, kMessageBufferSize,
                           "missing required named parameter '%s'",
                           parameter_name.ToCString());
            *error_message = String::New(message_buffer);
          }
          return false;
        }
      }
    }
  }
  return true;
}

// Checks each supplied function type argument is a subtype of the corresponding
// bound. Also takes the number of type arguments to skip over because they
// belong to parent functions and are not included in the type parameters.
// Returns null if all checks succeed, otherwise returns a non-null Error for
// one of the failures.
static ObjectPtr TypeArgumentsAreBoundSubtypes(
    Zone* zone,
    const TokenPosition& token_pos,
    const TypeArguments& type_parameters,
    intptr_t num_parent_type_args,
    const TypeArguments& instantiator_type_arguments,
    const TypeArguments& function_type_arguments) {
  ASSERT(!type_parameters.IsNull());
  ASSERT(!function_type_arguments.IsNull());
  const intptr_t kNumTypeArgs = function_type_arguments.Length();
  ASSERT_EQUAL(num_parent_type_args + type_parameters.Length(), kNumTypeArgs);

  // Don't bother allocating handles, there's nothing to check.
  if (kNumTypeArgs - num_parent_type_args == 0) return Error::null();

  auto& type = AbstractType::Handle(zone);
  auto& bound = AbstractType::Handle(zone);
  auto& name = String::Handle(zone);
  for (intptr_t i = num_parent_type_args; i < kNumTypeArgs; i++) {
    type = type_parameters.TypeAt(i - num_parent_type_args);
    ASSERT(type.IsTypeParameter());
    const auto& parameter = TypeParameter::Cast(type);
    bound = parameter.bound();
    name = parameter.name();
    // Only perform non-covariant checks where the bound is not the top type.
    if (parameter.IsGenericCovariantImpl() || bound.IsTopTypeForSubtyping()) {
      continue;
    }
    if (!AbstractType::InstantiateAndTestSubtype(&type, &bound,
                                                 instantiator_type_arguments,
                                                 function_type_arguments)) {
      return Error::RawCast(ThrowTypeError(token_pos, type, bound, name));
    }
  }

  return Error::null();
}

// Returns a TypeArguments object where, for each type parameter local to this
// function, the entry in the TypeArguments is an instantiated version of its
// bound. In the instantiated bound, any local function type parameter
// references are replaced with the corresponding bound if that bound can be
// fully instantiated without local function type parameters, otherwise dynamic.
static TypeArgumentsPtr InstantiateTypeParametersToBounds(
    Zone* zone,
    const TokenPosition& token_pos,
    const TypeArguments& type_parameters,
    const TypeArguments& instantiator_type_args,
    intptr_t num_parent_type_args,
    const TypeArguments& parent_type_args) {
  ASSERT(!type_parameters.IsNull());
  const intptr_t kNumCurrentTypeArgs = type_parameters.Length();
  const intptr_t kNumTypeArgs = kNumCurrentTypeArgs + num_parent_type_args;
  auto& function_type_args = TypeArguments::Handle(zone);

  bool all_bounds_instantiated = true;

  // Create a type argument vector large enough for the parents' and current
  // type arguments.
  function_type_args = TypeArguments::New(kNumTypeArgs);
  auto& type = AbstractType::Handle(zone);
  auto& bound = AbstractType::Handle(zone);
  // First copy over the parent type args (or the dynamic type if null).
  for (intptr_t i = 0; i < num_parent_type_args; i++) {
    type = parent_type_args.IsNull() ? Type::DynamicType()
                                     : parent_type_args.TypeAt(i);
    function_type_args.SetTypeAt(i, type);
  }
  // Now try fully instantiating the bounds of each parameter using only
  // the instantiator and parent function type arguments. If possible, keep the
  // instantiated bound as the entry. Otherwise, just set that entry to dynamic.
  for (intptr_t i = num_parent_type_args; i < kNumTypeArgs; i++) {
    type = type_parameters.TypeAt(i - num_parent_type_args);
    const auto& param = TypeParameter::Cast(type);
    bound = param.bound();
    // Only instantiate up to the parent type parameters.
    if (!bound.IsInstantiated(kAny, num_parent_type_args)) {
      bound = bound.InstantiateFrom(instantiator_type_args, function_type_args,
                                    num_parent_type_args, Heap::kNew);
    }
    if (!bound.IsInstantiated()) {
      // There are local type variables used in this bound.
      bound = Type::DynamicType();
      all_bounds_instantiated = false;
    }
    function_type_args.SetTypeAt(i, bound);
  }

  // If all the bounds were instantiated in the first pass, then there can't
  // be any self or mutual recursion, so skip the bounds subtype check.
  if (all_bounds_instantiated) return function_type_args.raw();

  // Do another pass, using the set of TypeArguments we just created. If a given
  // bound was instantiated in the last pass, just copy it over. (We don't need
  // to iterate to a fixed point, since there should be no self or mutual
  // recursion in the bounds.)
  const auto& first_round =
      TypeArguments::Handle(zone, function_type_args.raw());
  function_type_args = TypeArguments::New(kNumTypeArgs);
  // Again, copy over the parent type arguments first.
  for (intptr_t i = 0; i < num_parent_type_args; i++) {
    type = first_round.TypeAt(i);
    function_type_args.SetTypeAt(i, type);
  }
  for (intptr_t i = num_parent_type_args; i < kNumTypeArgs; i++) {
    type = type_parameters.TypeAt(i - num_parent_type_args);
    const auto& param = TypeParameter::Cast(type);
    bound = first_round.TypeAt(i);
    // The dynamic type is never a bound, even when implicit, so it also marks
    // bounds that were not already fully instantiated.
    if (bound.raw() == Type::DynamicType()) {
      bound = param.bound();
      bound = bound.InstantiateFrom(instantiator_type_args, first_round,
                                    kAllFree, Heap::kNew);
    }
    function_type_args.SetTypeAt(i, bound);
  }

  return function_type_args.raw();
}

// Retrieves the function type arguments, if any. This could be explicitly
// passed type from the arguments array, delayed type arguments in closures,
// or instantiated bounds for the type parameters if no other source for
// function type arguments are found.
static TypeArgumentsPtr RetrieveFunctionTypeArguments(
    Thread* thread,
    Zone* zone,
    const Function& function,
    const Instance& receiver,
    const TypeArguments& instantiator_type_args,
    const TypeArguments& type_params,
    const Array& args,
    const ArgumentsDescriptor& args_desc) {
  ASSERT(!function.IsNull());

  const intptr_t kNumCurrentTypeArgs = function.NumTypeParameters(thread);
  const intptr_t kNumParentTypeArgs = function.NumParentTypeParameters();
  const intptr_t kNumTypeArgs = kNumCurrentTypeArgs + kNumParentTypeArgs;
  // Non-generic functions don't receive type arguments.
  if (kNumTypeArgs == 0) return Object::empty_type_arguments().raw();
  // Closure functions require that the receiver be provided (and is a closure).
  ASSERT(!function.IsClosureFunction() || receiver.IsClosure());

  // Only closure functions should have possibly generic parents.
  ASSERT(function.IsClosureFunction() || kNumParentTypeArgs == 0);
  const auto& parent_type_args =
      function.IsClosureFunction()
          ? TypeArguments::Handle(
                zone, Closure::Cast(receiver).function_type_arguments())
          : Object::null_type_arguments();
  // We don't try to instantiate the parent type parameters to their bounds
  // if not provided or check any closed-over type arguments against the parent
  // type parameter bounds (since they have been type checked already).
  if (kNumCurrentTypeArgs == 0) return parent_type_args.raw();

  auto& function_type_args = TypeArguments::Handle(zone);
  if (function.IsClosureFunction()) {
    const auto& closure = Closure::Cast(receiver);
    function_type_args = closure.delayed_type_arguments();
    if (function_type_args.raw() == Object::empty_type_arguments().raw()) {
      // There are no delayed type arguments, so set back to null.
      function_type_args = TypeArguments::null();
    } else {
      // We should never end up here when the receiver is a closure with delayed
      // type arguments unless this dynamically called closure function was
      // retrieved directly from the closure instead of going through
      // DartEntry::ResolveCallable, which appropriately checks for this case.
      ASSERT(args_desc.TypeArgsLen() == 0);
    }
  }

  if (args_desc.TypeArgsLen() > 0) {
    function_type_args ^= args.At(0);
  }

  if (function_type_args.IsNull()) {
    // We have no explicitly provided function type arguments, so generate
    // some by instantiating the parameters to bounds.
    return InstantiateTypeParametersToBounds(
        zone, function.token_pos(), type_params, instantiator_type_args,
        kNumParentTypeArgs, parent_type_args);
  }

  if (kNumParentTypeArgs > 0) {
    function_type_args = function_type_args.Prepend(
        zone, parent_type_args, kNumParentTypeArgs, kNumTypeArgs);
  }

  return function_type_args.raw();
}

// Retrieves the instantiator type arguments, if any, from the receiver.
static TypeArgumentsPtr RetrieveInstantiatorTypeArguments(
    Zone* zone,
    const Function& function,
    const Instance& receiver) {
  if (function.IsClosureFunction()) {
    ASSERT(receiver.IsClosure());
    const auto& closure = Closure::Cast(receiver);
    return closure.instantiator_type_arguments();
  }
  if (!receiver.IsNull()) {
    const auto& cls = Class::Handle(zone, receiver.clazz());
    if (cls.NumTypeArguments() > 0) {
      return receiver.GetTypeArguments();
    }
  }
  return Object::empty_type_arguments().raw();
}

ObjectPtr Function::DoArgumentTypesMatch(
    const Array& args,
    const ArgumentsDescriptor& args_desc) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  auto& receiver = Instance::Handle(zone);
  if (IsClosureFunction() || HasThisParameter()) {
    receiver ^= args.At(args_desc.FirstArgIndex());
  }
  const auto& instantiator_type_arguments = TypeArguments::Handle(
      zone, RetrieveInstantiatorTypeArguments(zone, *this, receiver));
  return Function::DoArgumentTypesMatch(args, args_desc,
                                        instantiator_type_arguments);
}

ObjectPtr Function::DoArgumentTypesMatch(
    const Array& args,
    const ArgumentsDescriptor& args_desc,
    const TypeArguments& instantiator_type_arguments) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  auto& receiver = Instance::Handle(zone);
  if (IsClosureFunction() || HasThisParameter()) {
    receiver ^= args.At(args_desc.FirstArgIndex());
  }

  const auto& params = TypeArguments::Handle(zone, type_parameters());
  const auto& function_type_arguments = TypeArguments::Handle(
      zone, RetrieveFunctionTypeArguments(thread, zone, *this, receiver,
                                          instantiator_type_arguments, params,
                                          args, args_desc));
  return Function::DoArgumentTypesMatch(
      args, args_desc, instantiator_type_arguments, function_type_arguments);
}

ObjectPtr Function::DoArgumentTypesMatch(
    const Array& args,
    const ArgumentsDescriptor& args_desc,
    const TypeArguments& instantiator_type_arguments,
    const TypeArguments& function_type_arguments) const {
  // We need a concrete (possibly empty) type arguments vector, not the
  // implicitly filled with dynamic one.
  ASSERT(!function_type_arguments.IsNull());

  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  // Perform any non-covariant bounds checks on the provided function type
  // arguments to make sure they are appropriate subtypes of the bounds.
  const intptr_t kNumLocalTypeArgs = NumTypeParameters(thread);
  if (kNumLocalTypeArgs > 0) {
    const intptr_t kNumParentTypeArgs = NumParentTypeParameters();
    ASSERT_EQUAL(kNumLocalTypeArgs + kNumParentTypeArgs,
                 function_type_arguments.Length());
    const auto& params = TypeArguments::Handle(zone, type_parameters());
    const auto& result = Object::Handle(
        zone, TypeArgumentsAreBoundSubtypes(
                  zone, token_pos(), params, kNumParentTypeArgs,
                  instantiator_type_arguments, function_type_arguments));
    if (result.IsError()) {
      return result.raw();
    }
  } else {
    ASSERT_EQUAL(NumParentTypeParameters(), function_type_arguments.Length());
  }

  AbstractType& type = AbstractType::Handle(zone);
  Instance& argument = Instance::Handle(zone);

  auto check_argument = [](const Instance& argument, const AbstractType& type,
                           const TypeArguments& instantiator_type_args,
                           const TypeArguments& function_type_args) -> bool {
    // If the argument type is the top type, no need to check.
    if (type.IsTopTypeForSubtyping()) return true;
    if (argument.IsNull()) {
      return Instance::NullIsAssignableTo(type);
    }
    return argument.IsAssignableTo(type, instantiator_type_args,
                                   function_type_args);
  };

  // Check types of the provided arguments against the expected parameter types.
  const intptr_t arg_offset = args_desc.FirstArgIndex();
  // Only check explicit arguments.
  const intptr_t arg_start = arg_offset + NumImplicitParameters();
  const intptr_t num_positional_args = args_desc.PositionalCount();
  for (intptr_t arg_index = arg_start; arg_index < num_positional_args;
       ++arg_index) {
    argument ^= args.At(arg_index);
    // Adjust for type arguments when they're present.
    const intptr_t param_index = arg_index - arg_offset;
    type = ParameterTypeAt(param_index);

    if (!check_argument(argument, type, instantiator_type_arguments,
                        function_type_arguments)) {
      auto& name = String::Handle(zone, ParameterNameAt(param_index));
      return ThrowTypeError(token_pos(), argument, type, name);
    }
  }

  const intptr_t num_named_arguments = args_desc.NamedCount();
  if (num_named_arguments == 0) {
    return Error::null();
  }

  const int num_parameters = NumParameters();
  const int num_fixed_params = num_fixed_parameters();

  String& argument_name = String::Handle(zone);
  String& parameter_name = String::Handle(zone);

  // Check types of named arguments against expected parameter type.
  for (intptr_t named_index = 0; named_index < num_named_arguments;
       named_index++) {
    argument_name = args_desc.NameAt(named_index);
    ASSERT(argument_name.IsSymbol());
    argument ^= args.At(args_desc.PositionAt(named_index));

    // Try to find the named parameter that matches the provided argument.
    // Even when annotated with @required, named parameters are still stored
    // as if they were optional and so come after the fixed parameters.
    // Currently O(n^2) as there's no guarantee from either the CFE or the
    // VM that named parameters and named arguments are sorted in the same way.
    intptr_t param_index = num_fixed_params;
    for (; param_index < num_parameters; param_index++) {
      parameter_name = ParameterNameAt(param_index);
      ASSERT(parameter_name.IsSymbol());

      if (!parameter_name.Equals(argument_name)) continue;

      type = ParameterTypeAt(param_index);
      if (!check_argument(argument, type, instantiator_type_arguments,
                          function_type_arguments)) {
        auto& name = String::Handle(zone, ParameterNameAt(param_index));
        return ThrowTypeError(token_pos(), argument, type, name);
      }
      break;
    }
    // Only should fail if AreValidArguments returns a false positive.
    ASSERT(param_index < num_parameters);
  }
  return Error::null();
}

// Helper allocating a C string buffer in the zone, printing the fully qualified
// name of a function in it, and replacing ':' by '_' to make sure the
// constructed name is a valid C++ identifier for debugging purpose.
// Set 'chars' to allocated buffer and return number of written characters.

enum QualifiedFunctionLibKind {
  kQualifiedFunctionLibKindLibUrl,
  kQualifiedFunctionLibKindLibName
};

static intptr_t ConstructFunctionFullyQualifiedCString(
    const Function& function,
    char** chars,
    intptr_t reserve_len,
    bool with_lib,
    QualifiedFunctionLibKind lib_kind) {
  Zone* zone = Thread::Current()->zone();
  const char* name = String::Handle(zone, function.name()).ToCString();
  const char* function_format = (reserve_len == 0) ? "%s" : "%s_";
  reserve_len += Utils::SNPrint(NULL, 0, function_format, name);
  const Function& parent = Function::Handle(zone, function.parent_function());
  intptr_t written = 0;
  if (parent.IsNull()) {
    const Class& function_class = Class::Handle(zone, function.Owner());
    ASSERT(!function_class.IsNull());
    const char* class_name =
        String::Handle(zone, function_class.Name()).ToCString();
    ASSERT(class_name != NULL);
    const char* library_name = NULL;
    const char* lib_class_format = NULL;
    if (with_lib) {
      const Library& library = Library::Handle(zone, function_class.library());
      ASSERT(!library.IsNull());
      switch (lib_kind) {
        case kQualifiedFunctionLibKindLibUrl:
          library_name = String::Handle(zone, library.url()).ToCString();
          break;
        case kQualifiedFunctionLibKindLibName:
          library_name = String::Handle(zone, library.name()).ToCString();
          break;
        default:
          UNREACHABLE();
      }
      ASSERT(library_name != NULL);
      lib_class_format = (library_name[0] == '\0') ? "%s%s_" : "%s_%s_";
    } else {
      library_name = "";
      lib_class_format = "%s%s.";
    }
    reserve_len +=
        Utils::SNPrint(NULL, 0, lib_class_format, library_name, class_name);
    ASSERT(chars != NULL);
    *chars = zone->Alloc<char>(reserve_len + 1);
    written = Utils::SNPrint(*chars, reserve_len + 1, lib_class_format,
                             library_name, class_name);
  } else {
    written = ConstructFunctionFullyQualifiedCString(parent, chars, reserve_len,
                                                     with_lib, lib_kind);
  }
  ASSERT(*chars != NULL);
  char* next = *chars + written;
  written += Utils::SNPrint(next, reserve_len + 1, function_format, name);
  // Replace ":" with "_".
  while (true) {
    next = strchr(next, ':');
    if (next == NULL) break;
    *next = '_';
  }
  return written;
}

const char* Function::ToFullyQualifiedCString() const {
  char* chars = NULL;
  ConstructFunctionFullyQualifiedCString(*this, &chars, 0, true,
                                         kQualifiedFunctionLibKindLibUrl);
  return chars;
}

const char* Function::ToLibNamePrefixedQualifiedCString() const {
  char* chars = NULL;
  ConstructFunctionFullyQualifiedCString(*this, &chars, 0, true,
                                         kQualifiedFunctionLibKindLibName);
  return chars;
}

const char* Function::ToQualifiedCString() const {
  char* chars = NULL;
  ConstructFunctionFullyQualifiedCString(*this, &chars, 0, false,
                                         kQualifiedFunctionLibKindLibUrl);
  return chars;
}

FunctionPtr Function::InstantiateSignatureFrom(
    const TypeArguments& instantiator_type_arguments,
    const TypeArguments& function_type_arguments,
    intptr_t num_free_fun_type_params,
    Heap::Space space) const {
  Zone* zone = Thread::Current()->zone();
  const Object& owner = Object::Handle(zone, RawOwner());
  // Note that parent pointers in newly instantiated signatures still points to
  // the original uninstantiated parent signatures. That is not a problem.
  const Function& parent = Function::Handle(zone, parent_function());

  // See the comment on kCurrentAndEnclosingFree to understand why we don't
  // adjust 'num_free_fun_type_params' downward in this case.
  bool delete_type_parameters = false;
  if (num_free_fun_type_params == kCurrentAndEnclosingFree) {
    num_free_fun_type_params = kAllFree;
    delete_type_parameters = true;
  } else {
    ASSERT(!HasInstantiatedSignature(kAny, num_free_fun_type_params));

    // A generic typedef may declare a non-generic function type and get
    // instantiated with unrelated function type parameters. In that case, its
    // signature is still uninstantiated, because these type parameters are
    // free (they are not declared by the typedef).
    // For that reason, we only adjust num_free_fun_type_params if this
    // signature is generic or has a generic parent.
    if (IsGeneric() || HasGenericParent()) {
      // We only consider the function type parameters declared by the parents
      // of this signature function as free.
      const int num_parent_type_params = NumParentTypeParameters();
      if (num_parent_type_params < num_free_fun_type_params) {
        num_free_fun_type_params = num_parent_type_params;
      }
    }
  }

  Function& sig = Function::Handle(Function::NewSignatureFunction(
      owner, parent, TokenPosition::kNoSource, space));
  AbstractType& type = AbstractType::Handle(zone);

  // Copy the type parameters and instantiate their bounds (if necessary).
  if (!delete_type_parameters) {
    const TypeArguments& type_params =
        TypeArguments::Handle(zone, type_parameters());
    if (!type_params.IsNull()) {
      TypeArguments& instantiated_type_params = TypeArguments::Handle(zone);
      TypeParameter& type_param = TypeParameter::Handle(zone);
      Class& cls = Class::Handle(zone);
      String& param_name = String::Handle(zone);
      for (intptr_t i = 0; i < type_params.Length(); ++i) {
        type_param ^= type_params.TypeAt(i);
        type = type_param.bound();
        if (!type.IsInstantiated(kAny, num_free_fun_type_params)) {
          type = type.InstantiateFrom(instantiator_type_arguments,
                                      function_type_arguments,
                                      num_free_fun_type_params, space);
          // A returned null type indicates a failed instantiation in dead code
          // that must be propagated up to the caller, the optimizing compiler.
          if (type.IsNull()) {
            return Function::null();
          }
          cls = type_param.parameterized_class();
          param_name = type_param.name();
          ASSERT(type_param.IsFinalized());
          ASSERT(type_param.IsCanonical());
          type_param = TypeParameter::New(
              cls, sig, type_param.index(), param_name, type,
              type_param.IsGenericCovariantImpl(), type_param.nullability(),
              type_param.token_pos());
          type_param.SetIsFinalized();
          type_param.SetCanonical();
          type_param.SetDeclaration(true);
          if (instantiated_type_params.IsNull()) {
            instantiated_type_params = TypeArguments::New(type_params.Length());
            for (intptr_t j = 0; j < i; ++j) {
              type = type_params.TypeAt(j);
              instantiated_type_params.SetTypeAt(j, type);
            }
          }
          instantiated_type_params.SetTypeAt(i, type_param);
        } else if (!instantiated_type_params.IsNull()) {
          instantiated_type_params.SetTypeAt(i, type_param);
        }
      }
      sig.set_type_parameters(instantiated_type_params.IsNull()
                                  ? type_params
                                  : instantiated_type_params);
    }
  }

  type = result_type();
  if (!type.IsInstantiated(kAny, num_free_fun_type_params)) {
    type = type.InstantiateFrom(instantiator_type_arguments,
                                function_type_arguments,
                                num_free_fun_type_params, space);
    // A returned null type indicates a failed instantiation in dead code that
    // must be propagated up to the caller, the optimizing compiler.
    if (type.IsNull()) {
      return Function::null();
    }
  }
  sig.set_result_type(type);
  const intptr_t num_params = NumParameters();
  sig.set_num_fixed_parameters(num_fixed_parameters());
  sig.SetNumOptionalParameters(NumOptionalParameters(),
                               HasOptionalPositionalParameters());
  sig.set_parameter_types(Array::Handle(Array::New(num_params, space)));
  for (intptr_t i = 0; i < num_params; i++) {
    type = ParameterTypeAt(i);
    if (!type.IsInstantiated(kAny, num_free_fun_type_params)) {
      type = type.InstantiateFrom(instantiator_type_arguments,
                                  function_type_arguments,
                                  num_free_fun_type_params, space);
      // A returned null type indicates a failed instantiation in dead code that
      // must be propagated up to the caller, the optimizing compiler.
      if (type.IsNull()) {
        return Function::null();
      }
    }
    sig.SetParameterTypeAt(i, type);
  }
  sig.set_parameter_names(Array::Handle(zone, parameter_names()));

  if (delete_type_parameters) {
    ASSERT(sig.HasInstantiatedSignature(kFunctions));
  }
  return sig.raw();
}

// Checks if the type of the specified parameter of this function is a supertype
// of the type of the specified parameter of the other function (i.e. check
// parameter contravariance).
// Note that types marked as covariant are already dealt with in the front-end.
bool Function::IsContravariantParameter(intptr_t parameter_position,
                                        const Function& other,
                                        intptr_t other_parameter_position,
                                        Heap::Space space) const {
  const AbstractType& param_type =
      AbstractType::Handle(ParameterTypeAt(parameter_position));
  if (param_type.IsTopTypeForSubtyping()) {
    return true;
  }
  const AbstractType& other_param_type =
      AbstractType::Handle(other.ParameterTypeAt(other_parameter_position));
  return other_param_type.IsSubtypeOf(param_type, space);
}

bool Function::HasSameTypeParametersAndBounds(const Function& other,
                                              TypeEquality kind) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();

  const intptr_t num_type_params = NumTypeParameters(thread);
  if (num_type_params != other.NumTypeParameters(thread)) {
    return false;
  }
  if (num_type_params > 0) {
    const TypeArguments& type_params =
        TypeArguments::Handle(zone, type_parameters());
    ASSERT(!type_params.IsNull());
    const TypeArguments& other_type_params =
        TypeArguments::Handle(zone, other.type_parameters());
    ASSERT(!other_type_params.IsNull());
    TypeParameter& type_param = TypeParameter::Handle(zone);
    TypeParameter& other_type_param = TypeParameter::Handle(zone);
    AbstractType& bound = AbstractType::Handle(zone);
    AbstractType& other_bound = AbstractType::Handle(zone);
    for (intptr_t i = 0; i < num_type_params; i++) {
      type_param ^= type_params.TypeAt(i);
      other_type_param ^= other_type_params.TypeAt(i);
      bound = type_param.bound();
      ASSERT(bound.IsFinalized());
      other_bound = other_type_param.bound();
      ASSERT(other_bound.IsFinalized());
      if (kind == TypeEquality::kInSubtypeTest) {
        // Bounds that are mutual subtypes are considered equal.
        if (!bound.IsSubtypeOf(other_bound, Heap::kOld) ||
            !other_bound.IsSubtypeOf(bound, Heap::kOld)) {
          return false;
        }
      } else {
        if (!bound.IsEquivalent(other_bound, kind)) {
          return false;
        }
      }
    }
  }
  return true;
}

bool Function::IsSubtypeOf(const Function& other, Heap::Space space) const {
  const intptr_t num_fixed_params = num_fixed_parameters();
  const intptr_t num_opt_pos_params = NumOptionalPositionalParameters();
  const intptr_t num_opt_named_params = NumOptionalNamedParameters();
  const intptr_t other_num_fixed_params = other.num_fixed_parameters();
  const intptr_t other_num_opt_pos_params =
      other.NumOptionalPositionalParameters();
  const intptr_t other_num_opt_named_params =
      other.NumOptionalNamedParameters();
  // This function requires the same arguments or less and accepts the same
  // arguments or more. We can ignore implicit parameters.
  const intptr_t num_ignored_params = NumImplicitParameters();
  const intptr_t other_num_ignored_params = other.NumImplicitParameters();
  if (((num_fixed_params - num_ignored_params) >
       (other_num_fixed_params - other_num_ignored_params)) ||
      ((num_fixed_params - num_ignored_params + num_opt_pos_params) <
       (other_num_fixed_params - other_num_ignored_params +
        other_num_opt_pos_params)) ||
      (num_opt_named_params < other_num_opt_named_params)) {
    return false;
  }
  // Check the type parameters and bounds of generic functions.
  if (!HasSameTypeParametersAndBounds(other, TypeEquality::kInSubtypeTest)) {
    return false;
  }
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  Isolate* isolate = thread->isolate();
  // Check the result type.
  const AbstractType& other_res_type =
      AbstractType::Handle(zone, other.result_type());
  // 'void Function()' is a subtype of 'Object Function()'.
  if (!other_res_type.IsTopTypeForSubtyping()) {
    const AbstractType& res_type = AbstractType::Handle(zone, result_type());
    if (!res_type.IsSubtypeOf(other_res_type, space)) {
      return false;
    }
  }
  // Check the types of fixed and optional positional parameters.
  for (intptr_t i = 0; i < (other_num_fixed_params - other_num_ignored_params +
                            other_num_opt_pos_params);
       i++) {
    if (!IsContravariantParameter(i + num_ignored_params, other,
                                  i + other_num_ignored_params, space)) {
      return false;
    }
  }
  // Check that for each optional named parameter of type T of the other
  // function type, there exists an optional named parameter of this function
  // type with an identical name and with a type S that is a supertype of T.
  // Note that SetParameterNameAt() guarantees that names are symbols, so we
  // can compare their raw pointers.
  const int num_params = num_fixed_params + num_opt_named_params;
  const int other_num_params =
      other_num_fixed_params + other_num_opt_named_params;
  bool found_param_name;
  String& other_param_name = String::Handle(zone);
  for (intptr_t i = other_num_fixed_params; i < other_num_params; i++) {
    other_param_name = other.ParameterNameAt(i);
    ASSERT(other_param_name.IsSymbol());
    found_param_name = false;
    for (intptr_t j = num_fixed_params; j < num_params; j++) {
      ASSERT(String::Handle(zone, ParameterNameAt(j)).IsSymbol());
      if (ParameterNameAt(j) == other_param_name.raw()) {
        found_param_name = true;
        if (!IsContravariantParameter(j, other, i, space)) {
          return false;
        }
        break;
      }
    }
    if (!found_param_name) {
      return false;
    }
  }
  if (isolate->null_safety()) {
    // Check that for each required named parameter in this function, there's a
    // corresponding required named parameter in the other function.
    String& param_name = other_param_name;
    for (intptr_t j = num_params - num_opt_named_params; j < num_params; j++) {
      if (IsRequiredAt(j)) {
        param_name = ParameterNameAt(j);
        ASSERT(param_name.IsSymbol());
        bool found = false;
        for (intptr_t i = other_num_fixed_params; i < other_num_params; i++) {
          ASSERT(String::Handle(zone, other.ParameterNameAt(i)).IsSymbol());
          if (other.ParameterNameAt(i) == param_name.raw()) {
            found = true;
            if (!other.IsRequiredAt(i)) {
              return false;
            }
          }
        }
        if (!found) {
          return false;
        }
      }
    }
  }
  return true;
}

// The compiler generates an implicit constructor if a class definition
// does not contain an explicit constructor or factory. The implicit
// constructor has the same token position as the owner class.
bool Function::IsImplicitConstructor() const {
  return IsGenerativeConstructor() && (token_pos() == end_token_pos());
}

bool Function::IsImplicitStaticClosureFunction(FunctionPtr func) {
  NoSafepointScope no_safepoint;
  uint32_t kind_tag = func->ptr()->kind_tag_;
  return (KindBits::decode(kind_tag) ==
          FunctionLayout::kImplicitClosureFunction) &&
         StaticBit::decode(kind_tag);
}

FunctionPtr Function::New(Heap::Space space) {
  ASSERT(Object::function_class() != Class::null());
  ObjectPtr raw =
      Object::Allocate(Function::kClassId, Function::InstanceSize(), space);
  return static_cast<FunctionPtr>(raw);
}

FunctionPtr Function::New(const String& name,
                          FunctionLayout::Kind kind,
                          bool is_static,
                          bool is_const,
                          bool is_abstract,
                          bool is_external,
                          bool is_native,
                          const Object& owner,
                          TokenPosition token_pos,
                          Heap::Space space) {
  ASSERT(!owner.IsNull() || (kind == FunctionLayout::kSignatureFunction));
  const Function& result = Function::Handle(Function::New(space));
  result.set_kind_tag(0);
  result.set_parameter_types(Object::empty_array());
  result.set_parameter_names(Object::empty_array());
  result.set_name(name);
  result.set_kind_tag(0);  // Ensure determinism of uninitialized bits.
  result.set_kind(kind);
  result.set_recognized_kind(MethodRecognizer::kUnknown);
  result.set_modifier(FunctionLayout::kNoModifier);
  result.set_is_static(is_static);
  result.set_is_const(is_const);
  result.set_is_abstract(is_abstract);
  result.set_is_external(is_external);
  result.set_is_native(is_native);
  result.set_is_reflectable(true);  // Will be computed later.
  result.set_is_visible(true);      // Will be computed later.
  result.set_is_debuggable(true);   // Will be computed later.
  result.set_is_intrinsic(false);
  result.set_is_redirecting(false);
  result.set_is_generated_body(false);
  result.set_has_pragma(false);
  result.set_is_polymorphic_target(false);
  result.set_is_synthetic(false);
  NOT_IN_PRECOMPILED(result.set_state_bits(0));
  result.set_owner(owner);
  NOT_IN_PRECOMPILED(result.set_token_pos(token_pos));
  NOT_IN_PRECOMPILED(result.set_end_token_pos(token_pos));
  result.set_num_fixed_parameters(0);
  result.SetNumOptionalParameters(0, false);
  NOT_IN_PRECOMPILED(result.set_usage_counter(0));
  NOT_IN_PRECOMPILED(result.set_deoptimization_counter(0));
  NOT_IN_PRECOMPILED(result.set_optimized_instruction_count(0));
  NOT_IN_PRECOMPILED(result.set_optimized_call_site_count(0));
  NOT_IN_PRECOMPILED(result.set_inlining_depth(0));
  NOT_IN_PRECOMPILED(result.set_is_declared_in_bytecode(false));
  NOT_IN_PRECOMPILED(result.set_binary_declaration_offset(0));
  result.set_is_optimizable(is_native ? false : true);
  result.set_is_background_optimizable(is_native ? false : true);
  result.set_is_inlinable(true);
  result.reset_unboxed_parameters_and_return();
  result.SetInstructionsSafe(StubCode::LazyCompile());
  if (kind == FunctionLayout::kClosureFunction ||
      kind == FunctionLayout::kImplicitClosureFunction) {
    ASSERT(space == Heap::kOld);
    const ClosureData& data = ClosureData::Handle(ClosureData::New());
    result.set_data(data);
  } else if (kind == FunctionLayout::kSignatureFunction) {
    const SignatureData& data =
        SignatureData::Handle(SignatureData::New(space));
    result.set_data(data);
  } else if (kind == FunctionLayout::kFfiTrampoline) {
    const FfiTrampolineData& data =
        FfiTrampolineData::Handle(FfiTrampolineData::New());
    result.set_data(data);
  } else {
    // Functions other than signature functions have no reason to be allocated
    // in new space.
    ASSERT(space == Heap::kOld);
  }

  // Force-optimized functions are not debuggable because they cannot
  // deoptimize.
  if (result.ForceOptimize()) {
    result.set_is_debuggable(false);
  }

  return result.raw();
}

FunctionPtr Function::NewClosureFunctionWithKind(FunctionLayout::Kind kind,
                                                 const String& name,
                                                 const Function& parent,
                                                 TokenPosition token_pos,
                                                 const Object& owner) {
  ASSERT((kind == FunctionLayout::kClosureFunction) ||
         (kind == FunctionLayout::kImplicitClosureFunction));
  ASSERT(!parent.IsNull());
  ASSERT(!owner.IsNull());
  const Function& result = Function::Handle(
      Function::New(name, kind,
                    /* is_static = */ parent.is_static(),
                    /* is_const = */ false,
                    /* is_abstract = */ false,
                    /* is_external = */ false,
                    /* is_native = */ false, owner, token_pos));
  result.set_parent_function(parent);
  return result.raw();
}

FunctionPtr Function::NewClosureFunction(const String& name,
                                         const Function& parent,
                                         TokenPosition token_pos) {
  // Use the owner defining the parent function and not the class containing it.
  const Object& parent_owner = Object::Handle(parent.RawOwner());
  return NewClosureFunctionWithKind(FunctionLayout::kClosureFunction, name,
                                    parent, token_pos, parent_owner);
}

FunctionPtr Function::NewImplicitClosureFunction(const String& name,
                                                 const Function& parent,
                                                 TokenPosition token_pos) {
  // Use the owner defining the parent function and not the class containing it.
  const Object& parent_owner = Object::Handle(parent.RawOwner());
  return NewClosureFunctionWithKind(FunctionLayout::kImplicitClosureFunction,
                                    name, parent, token_pos, parent_owner);
}

FunctionPtr Function::NewSignatureFunction(const Object& owner,
                                           const Function& parent,
                                           TokenPosition token_pos,
                                           Heap::Space space) {
  const Function& result = Function::Handle(Function::New(
      Symbols::AnonymousSignature(), FunctionLayout::kSignatureFunction,
      /* is_static = */ false,
      /* is_const = */ false,
      /* is_abstract = */ false,
      /* is_external = */ false,
      /* is_native = */ false,
      owner,  // Same as function type scope class.
      token_pos, space));
  result.set_parent_function(parent);
  result.set_is_reflectable(false);
  result.set_is_visible(false);
  result.set_is_debuggable(false);
  return result.raw();
}

FunctionPtr Function::NewEvalFunction(const Class& owner,
                                      const Script& script,
                                      bool is_static) {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  const Function& result = Function::Handle(
      zone,
      Function::New(String::Handle(Symbols::New(thread, ":Eval")),
                    FunctionLayout::kRegularFunction, is_static,
                    /* is_const = */ false,
                    /* is_abstract = */ false,
                    /* is_external = */ false,
                    /* is_native = */ false, owner, TokenPosition::kMinSource));
  ASSERT(!script.IsNull());
  result.set_is_debuggable(false);
  result.set_is_visible(true);
  result.set_eval_script(script);
  return result.raw();
}

bool Function::SafeToClosurize() const {
#if defined(DART_PRECOMPILED_RUNTIME)
  return HasImplicitClosureFunction();
#else
  return true;
#endif
}

bool Function::IsDynamicClosureCallDispatcher(Thread* thread) const {
  if (!IsInvokeFieldDispatcher()) return false;
  if (thread->isolate()->object_store()->closure_class() != Owner()) {
    return false;
  }
  const auto& handle = String::Handle(thread->zone(), name());
  return handle.Equals(Symbols::DynamicCall());
}

FunctionPtr Function::ImplicitClosureFunction() const {
  // Return the existing implicit closure function if any.
  if (implicit_closure_function() != Function::null()) {
    return implicit_closure_function();
  }
#if defined(DART_PRECOMPILED_RUNTIME)
  // In AOT mode all implicit closures are pre-created.
  FATAL("Cannot create implicit closure in AOT!");
  return Function::null();
#else
  ASSERT(!IsSignatureFunction() && !IsClosureFunction());
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  // Create closure function.
  const String& closure_name = String::Handle(zone, name());
  const Function& closure_function = Function::Handle(
      zone, NewImplicitClosureFunction(closure_name, *this, token_pos()));

  // Set closure function's context scope.
  if (is_static()) {
    closure_function.set_context_scope(Object::empty_context_scope());
  } else {
    const ContextScope& context_scope = ContextScope::Handle(
        zone, LocalScope::CreateImplicitClosureScope(*this));
    closure_function.set_context_scope(context_scope);
  }

  // Set closure function's type parameters.
  closure_function.set_type_parameters(
      TypeArguments::Handle(zone, type_parameters()));

  // Set closure function's result type to this result type.
  closure_function.set_result_type(AbstractType::Handle(zone, result_type()));

  // Set closure function's end token to this end token.
  closure_function.set_end_token_pos(end_token_pos());

  // The closurized method stub just calls into the original method and should
  // therefore be skipped by the debugger and in stack traces.
  closure_function.set_is_debuggable(false);
  closure_function.set_is_visible(false);

  // Set closure function's formal parameters to this formal parameters,
  // removing the receiver if this is an instance method and adding the closure
  // object as first parameter.
  const int kClosure = 1;
  const int has_receiver = is_static() ? 0 : 1;
  const int num_fixed_params = kClosure - has_receiver + num_fixed_parameters();
  const int num_opt_params = NumOptionalParameters();
  const bool has_opt_pos_params = HasOptionalPositionalParameters();
  const int num_params = num_fixed_params + num_opt_params;
  closure_function.set_num_fixed_parameters(num_fixed_params);
  closure_function.SetNumOptionalParameters(num_opt_params, has_opt_pos_params);
  closure_function.set_parameter_types(
      Array::Handle(zone, Array::New(num_params, Heap::kOld)));
  closure_function.CreateNameArrayIncludingFlags(Heap::kOld);
  AbstractType& param_type = AbstractType::Handle(zone);
  String& param_name = String::Handle(zone);
  // Add implicit closure object parameter.
  param_type = Type::DynamicType();
  closure_function.SetParameterTypeAt(0, param_type);
  closure_function.SetParameterNameAt(0, Symbols::ClosureParameter());
  for (int i = kClosure; i < num_params; i++) {
    param_type = ParameterTypeAt(has_receiver - kClosure + i);
    closure_function.SetParameterTypeAt(i, param_type);
    param_name = ParameterNameAt(has_receiver - kClosure + i);
    closure_function.SetParameterNameAt(i, param_name);
    if (IsRequiredAt(has_receiver - kClosure + i)) {
      closure_function.SetIsRequiredAt(i);
    }
  }
  closure_function.TruncateUnusedParameterFlags();
  closure_function.InheritBinaryDeclarationFrom(*this);

  // Change covariant parameter types to either Object? for an opted-in implicit
  // closure or to Object* for a legacy implicit closure.
  if (!is_static()) {
    BitVector is_covariant(zone, NumParameters());
    BitVector is_generic_covariant_impl(zone, NumParameters());
    kernel::ReadParameterCovariance(*this, &is_covariant,
                                    &is_generic_covariant_impl);

    Type& object_type = Type::Handle(zone, Type::ObjectType());
    ObjectStore* object_store = Isolate::Current()->object_store();
    object_type = nnbd_mode() == NNBDMode::kOptedInLib
                      ? object_store->nullable_object_type()
                      : object_store->legacy_object_type();
    for (intptr_t i = kClosure; i < num_params; ++i) {
      const intptr_t original_param_index = has_receiver - kClosure + i;
      if (is_covariant.Contains(original_param_index) ||
          is_generic_covariant_impl.Contains(original_param_index)) {
        closure_function.SetParameterTypeAt(i, object_type);
      }
    }
  }
  const Type& signature_type =
      Type::Handle(zone, closure_function.SignatureType());
  if (!signature_type.IsFinalized()) {
    ClassFinalizer::FinalizeType(Class::Handle(zone, Owner()), signature_type);
  }
  set_implicit_closure_function(closure_function);
  ASSERT(closure_function.IsImplicitClosureFunction());
  return closure_function.raw();
#endif  // defined(DART_PRECOMPILED_RUNTIME)
}

void Function::DropUncompiledImplicitClosureFunction() const {
  if (implicit_closure_function() != Function::null()) {
    const Function& func = Function::Handle(implicit_closure_function());
    if (!func.HasCode()) {
      set_implicit_closure_function(Function::Handle());
    }
  }
}

StringPtr Function::Signature() const {
  Thread* thread = Thread::Current();
  ZoneTextBuffer printer(thread->zone());
  PrintSignature(kInternalName, &printer);
  return Symbols::New(thread, printer.buffer());
}

StringPtr Function::UserVisibleSignature() const {
  Thread* thread = Thread::Current();
  ZoneTextBuffer printer(thread->zone());
  PrintSignature(kUserVisibleName, &printer);
  return Symbols::New(thread, printer.buffer());
}

void Function::PrintSignatureParameters(Thread* thread,
                                        Zone* zone,
                                        NameVisibility name_visibility,
                                        BaseTextBuffer* printer) const {
  AbstractType& param_type = AbstractType::Handle(zone);
  const intptr_t num_params = NumParameters();
  const intptr_t num_fixed_params = num_fixed_parameters();
  const intptr_t num_opt_pos_params = NumOptionalPositionalParameters();
  const intptr_t num_opt_named_params = NumOptionalNamedParameters();
  const intptr_t num_opt_params = num_opt_pos_params + num_opt_named_params;
  ASSERT((num_fixed_params + num_opt_params) == num_params);
  intptr_t i = 0;
  if (name_visibility == kUserVisibleName) {
    // Hide implicit parameters.
    i = NumImplicitParameters();
  }
  String& name = String::Handle(zone);
  while (i < num_fixed_params) {
    param_type = ParameterTypeAt(i);
    ASSERT(!param_type.IsNull());
    param_type.PrintName(name_visibility, printer);
    if (i != (num_params - 1)) {
      printer->AddString(", ");
    }
    i++;
  }
  if (num_opt_params > 0) {
    if (num_opt_pos_params > 0) {
      printer->AddString("[");
    } else {
      printer->AddString("{");
    }
    for (intptr_t i = num_fixed_params; i < num_params; i++) {
      if (num_opt_named_params > 0 && IsRequiredAt(i)) {
        printer->AddString("required ");
      }
      param_type = ParameterTypeAt(i);
      ASSERT(!param_type.IsNull());
      param_type.PrintName(name_visibility, printer);
      // The parameter name of an optional positional parameter does not need
      // to be part of the signature, since it is not used.
      if (num_opt_named_params > 0) {
        name = ParameterNameAt(i);
        printer->AddString(" ");
        printer->AddString(name.ToCString());
      }
      if (i != (num_params - 1)) {
        printer->AddString(", ");
      }
    }
    if (num_opt_pos_params > 0) {
      printer->AddString("]");
    } else {
      printer->AddString("}");
    }
  }
}

InstancePtr Function::ImplicitStaticClosure() const {
  ASSERT(IsImplicitStaticClosureFunction());
  if (implicit_static_closure() == Instance::null()) {
    Zone* zone = Thread::Current()->zone();
    const Context& context = Context::Handle(zone);
    Instance& closure =
        Instance::Handle(zone, Closure::New(Object::null_type_arguments(),
                                            Object::null_type_arguments(),
                                            *this, context, Heap::kOld));
    set_implicit_static_closure(closure);
  }
  return implicit_static_closure();
}

InstancePtr Function::ImplicitInstanceClosure(const Instance& receiver) const {
  ASSERT(IsImplicitClosureFunction());
  Zone* zone = Thread::Current()->zone();
  const Context& context = Context::Handle(zone, Context::New(1));
  context.SetAt(0, receiver);
  TypeArguments& instantiator_type_arguments = TypeArguments::Handle(zone);
  if (!HasInstantiatedSignature(kCurrentClass)) {
    instantiator_type_arguments = receiver.GetTypeArguments();
  }
  ASSERT(HasInstantiatedSignature(kFunctions));  // No generic parent function.
  return Closure::New(instantiator_type_arguments,
                      Object::null_type_arguments(), *this, context);
}

FunctionPtr Function::ImplicitClosureTarget(Zone* zone) const {
  const auto& parent = Function::Handle(zone, parent_function());
  const auto& func_name = String::Handle(zone, parent.name());
  const auto& owner = Class::Handle(zone, parent.Owner());
  const auto& error = owner.EnsureIsFinalized(Thread::Current());
  ASSERT(error == Error::null());
  auto& target = Function::Handle(zone, owner.LookupFunction(func_name));

  if (!target.IsNull() && (target.raw() != parent.raw())) {
    DEBUG_ASSERT(Isolate::Current()->HasAttemptedReload());
    if ((target.is_static() != parent.is_static()) ||
        (target.kind() != parent.kind())) {
      target = Function::null();
    }
  }

  return target.raw();
}

intptr_t Function::ComputeClosureHash() const {
  ASSERT(IsClosureFunction());
  const Class& cls = Class::Handle(Owner());
  uintptr_t result = String::Handle(name()).Hash();
  result += String::Handle(Signature()).Hash();
  result += String::Handle(cls.Name()).Hash();
  return result;
}

void Function::PrintSignature(NameVisibility name_visibility,
                              BaseTextBuffer* printer) const {
  Thread* thread = Thread::Current();
  Zone* zone = thread->zone();
  Isolate* isolate = thread->isolate();
  String& name = String::Handle(zone);
  const TypeArguments& type_params =
      TypeArguments::Handle(zone, type_parameters());
  if (!type_params.IsNull()) {
    const intptr_t num_type_params = type_params.Length();
    ASSERT(num_type_params > 0);
    TypeParameter& type_param = TypeParameter::Handle(zone);
    AbstractType& bound = AbstractType::Handle(zone);
    printer->AddString("<");
    for (intptr_t i = 0; i < num_type_params; i++) {
      type_param ^= type_params.TypeAt(i);
      name = type_param.name();
      printer->AddString(name.ToCString());
      bound = type_param.bound();
      // Do not print default bound or non-nullable Object bound in weak mode.
      if (!bound.IsNull() &&
          (!bound.IsObjectType() ||
           (isolate->null_safety() && bound.IsNonNullable()))) {
        printer->AddString(" extends ");
        bound.PrintName(name_visibility, printer);
      }
      if (i < num_type_params - 1) {
        printer->AddString(", ");
      }
    }
    printer->AddString(">");
  }
  printer->AddString("(");
  PrintSignatureParameters(thread, zone, name_visibility, printer);
  printer->AddString(") => ");
  const AbstractType& res_type = AbstractType::Handle(zone, result_type());
  res_type.PrintName(name_visibility, printer);
}

bool Function::HasInstantiatedSignature(Genericity genericity,
                                        intptr_t num_free_fun_type_params,
                                        TrailPtr trail) const {
  if (num_free_fun_type_params == kCurrentAndEnclosingFree) {
    num_free_fun_type_params = kAllFree;
  } else if (genericity != kCurrentClass) {
    // A generic typedef may declare a non-generic function type and get
    // instantiated with unrelated function type parameters. In that case, its
    // signature is still uninstantiated, because these type parameters are
    // free (they are not declared by the typedef).
    // For that reason, we only adjust num_free_fun_type_params if this
    // signature is generic or has a generic parent.
    if (IsGeneric() || HasGenericParent()) {
      // We only consider the function type parameters declared by the parents
      // of this signature function as free.
      const int num_parent_type_params = NumParentTypeParameters();
      if (num_parent_type_params < num_free_fun_type_params) {
        num_free_fun_type_params = num_parent_type_params;
      }
    }
  }
  AbstractType& type = AbstractType::Handle(result_type());
  if (!type.IsInstantiated(genericity, num_free_fun_type_params, trail)) {
    return false;
  }
  const intptr_t num_parameters = NumParameters();
  for (intptr_t i = 0; i < num_parameters; i++) {
    type = ParameterTypeAt(i);
    if (!type.IsInstantiated(genericity, num_free_fun_type_params, trail)) {
      return false;
    }
  }
  TypeArguments& type_params = TypeArguments::Handle(type_parameters());
  TypeParameter& type_param = TypeParameter::Handle();
  for (intptr_t i = 0; i < type_params.Length(); ++i) {
    type_param ^= type_params.TypeAt(i);
    type = type_param.bound();
    if (!type.IsInstantiated(genericity, num_free_fun_type_params, trail)) {
      return false;
    }
  }
  return true;
}

ClassPtr Function::Owner() const {
  if (raw_ptr()->owner_ == Object::null()) {
    ASSERT(IsSignatureFunction());
    return Class::null();
  }
  if (raw_ptr()->owner_->IsClass()) {
    return Class::RawCast(raw_ptr()->owner_);
  }
  const Object& obj = Object::Handle(raw_ptr()->owner_);
  ASSERT(obj.IsPatchClass());
  return PatchClass::Cast(obj).patched_class();
}

ClassPtr Function::origin() const {
  if (raw_ptr()->owner_ == Object::null()) {
    ASSERT(IsSignatureFunction());
    return Class::null();
  }
  if (raw_ptr()->owner_->IsClass()) {
    return Class::RawCast(raw_ptr()->owner_);
  }
  const Object& obj = Object::Handle(raw_ptr()->owner_);
  ASSERT(obj.IsPatchClass());
  return PatchClass::Cast(obj).origin_class();
}

void Function::InheritBinaryDeclarationFrom(const Function& src) const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  StoreNonPointer(&raw_ptr()->binary_declaration_,
                  src.raw_ptr()->binary_declaration_);
#endif
}

void Function::InheritBinaryDeclarationFrom(const Field& src) const {
#if defined(DART_PRECOMPILED_RUNTIME)
  UNREACHABLE();
#else
  if (src.is_declared_in_bytecode()) {
    set_is_declared_in_bytecode(true);
    set_bytecode_offset(src.bytecode_offset());
  } else {
    set_kernel_offset(src.kernel_offset());
  }
#endif
}

void Function::SetKernelDataAndScript(const Script& script,
                                      const ExternalTypedData& data,
                                      intptr_t offset) const {
  Array& data_field = Array::Handle(Array::New(3));
  data_field.SetAt(0, script);
  data_field.SetAt(1, data);
  data_field.SetAt(2, Smi::Handle(Smi::New(offset)));
  set_data(data_field);
}

ScriptPtr Function::script() const {
  // NOTE(turnidge): If you update this function, you probably want to
  // update Class::PatchFieldsAndFunctions() at the same time.
  const Object& data = Object::Handle(raw_ptr()->data_);
  if (IsDynamicInvocationForwarder()) {
    const auto& forwarding_target = Function::Handle(ForwardingTarget());
    return forwarding_target.script();
  }
  if (IsImplicitGetterOrSetter()) {
    const auto& field = Field::Handle(accessor_field());
    return field.Script();
  }
  if (data.IsArray()) {
    Object& script = Object::Handle(Array::Cast(data).At(0));
    if (script.IsScript()) {
      return Script::Cast(script).raw();
    }
  }
  if (token_pos() == TokenPosition::kMinSource) {
    // Testing for position 0 is an optimization that relies on temporary
    // eval functions having token position 0.
    const Script& script = Script::Handle(eval_script());
    if (!script.IsNull()) {
      return script.raw();
    }
  }
  const Object& obj = Object::Handle(raw_ptr()->owner_);
  if (obj.IsPatchClass()) {
    return PatchClass::Cast(obj).script();
  }
  if (IsClosureFunction()) {
    return Function::Handle(parent_function()).script();
  }
  if (obj.IsNull()) {
    ASSERT(IsSignatureFunction());
    return Script::null();
  }
  ASSERT(obj.IsClass());
  return Class::Cast(obj).script();
}

ExternalTypedDataPtr Function::KernelData() const {
  Object& data = Object::Handle(raw_ptr()->data_);
  if (data.IsArray()) {
    Object& script = Object::Handle(Array::Cast(data).At(0));
    if (script.IsScript()) {
      return ExternalTypedData::RawCast(Array::Cast(data).At(1));
    }
  }
  if (IsClosureFunction()) {
    Function& parent = Function::Handle(parent_function());
    ASSERT(!parent.IsNull());
    return parent.KernelData();
  }

  const Object& obj = Object::Handle(raw_ptr()->owner_);
  if (obj.IsClass()) {
    Library& lib = Library::Handle(Class::Cast(obj).library());
    return lib.kernel_data();
  }
  ASSERT(obj.IsPatchClass());
  return PatchClass::Cast(obj).library_kernel_data();
}

intptr_t Function::KernelDataProgramOffset() const {
  ASSERT(!is_declared_in_bytecode());
  if (IsNoSuchMethodDispatcher() || IsInvokeFieldDispatcher() ||
      IsFfiTrampoline()) {
    return 0;
  }
  Object& data = Object::Handle(raw_ptr()->data_);
  if (data.IsArray()) {
    Object& script = Object::Handle(Array::Cast(data).At(0));
    if (script.IsScript()) {
      return Smi::Value(Smi::RawCast(Array::Cast(data).At(2)));
    }
  }
  if (IsClosureFunction()) {
    Function& parent = Function::Handle(parent_function());
    ASSERT(!parent.IsNull());
    return parent.KernelDataProgramOffset();
  }

  const Object& obj = Object::Handle(raw_ptr()->owner_);
  if (obj.IsClass()) {
    Library& lib = Library::Handle(Class::Cast(obj).library());
    ASSERT(!lib.is_declared_in_bytecode());
    return lib.kernel_offset();
  }
  ASSERT(obj.IsPatchClass());
  return PatchClass::Cast(obj).library_kernel_offset();
}

bool Function::HasOptimizedCode() const {
  return HasCode() && Code::Handle(CurrentCode()).is_optimized();
}

bool Function::ShouldCompilerOptimize() const {
  return !FLAG_enable_interpreter ||
         ((unoptimized_code() != Object::null()) && WasCompiled()) ||
         ForceOptimize();
}

const char* Function::NameCString(NameVisibility name_visibility) const {
  switch (name_visibility) {
    case kInternalName:
      return String::Handle(name()).ToCString();
    case kScrubbedName:
    case kUserVisibleName:
      return UserVisibleNameCString();
  }
  UNREACHABLE();
  return nullptr;
}

const char* Function::UserVisibleNameCString() const {
  if (FLAG_show_internal_names) {
    return String::Handle(name()).ToCString();
  }
  return String::ScrubName(String::Handle(name()), is_extension_member());
}

StringPtr Function::UserVisibleName() const {
  if (FLAG_show_internal_names) {
    return name();
  }
  return Symbols::New(
      Thread::Current(),
      String::ScrubName(String::Handle(name()), is_extension_member()));
}

StringPtr Function::QualifiedScrubbedName() const {
  Thread* thread = Thread::Current();
  ZoneTextBuffer printer(thread->zone());
  PrintName(NameFormattingParams(kScrubbedName), &printer);
  return Symbols::New(thread, printer.buffer());
}

StringPtr Function::QualifiedUserVisibleName() const {
  Thread* thread = Thread::Current();
  ZoneTextBuffer printer(thread->zone());
  PrintName(NameFormattingParams(kUserVisibleName), &printer);
  return Symbols::New(thread, printer.buffer());
}

void Function::PrintName(const NameFormattingParams& params,
                         BaseTextBuffer* printer) const {
  // If |this| is the generated asynchronous body closure, use the
  // name of the parent function.
  Function& fun = Function::Handle(raw());

  if (params.disambiguate_names) {
    if (fun.IsInvokeFieldDispatcher()) {
      printer->AddString("[invoke-field] ");
    }
    if (fun.IsImplicitClosureFunction()) {
      printer->AddString("[tear-off] ");
    }
    if (fun.IsMethodExtractor()) {
      printer->AddString("[tear-off-extractor] ");
    }
  }

  if (fun.IsNonImplicitClosureFunction()) {
    // Sniff the parent function.
    fun = fun.parent_function();
    ASSERT(!fun.IsNull());
    if (!fun.IsAsyncGenerator() && !fun.IsAsyncFunction() &&
        !fun.IsSyncGenerator()) {
      // Parent function is not the generator of an asynchronous body closure,
      // start at |this|.
      fun = raw();
    }
  }
  if (IsClosureFunction()) {
    if (fun.IsLocalFunction() && !fun.IsImplicitClosureFunction()) {
      Function& parent = Function::Handle(fun.parent_function());
      if (parent.IsAsyncClosure() || parent.IsSyncGenClosure() ||
          parent.IsAsyncGenClosure()) {
        // Skip the closure and use the real function name found in
        // the parent.
        parent = parent.parent_function();
      }
      if (params.include_parent_name) {
        parent.PrintName(params, printer);
        // A function's scrubbed name and its user visible name are identical.
        printer->AddString(".");
      }
      if (params.disambiguate_names &&
          fun.name() == Symbols::AnonymousClosure().raw()) {
        printer->Printf("<anonymous closure @%" Pd ">", fun.token_pos().Pos());
      } else {
        printer->AddString(fun.NameCString(params.name_visibility));
      }
      // If we skipped rewritten async/async*/sync* body then append a suffix
      // to the end of the name.
      if (fun.raw() != raw() && params.disambiguate_names) {
        printer->AddString("{body}");
      }
      return;
    }
  }

  if (fun.kind() == FunctionLayout::kConstructor) {
    printer->AddString("new ");
  } else if (params.include_class_name) {
    const Class& cls = Class::Handle(Owner());
    if (!cls.IsTopLevel()) {
      const Class& mixin = Class::Handle(cls.Mixin());
      printer->AddString(params.name_visibility == kUserVisibleName
                             ? mixin.UserVisibleNameCString()
                             : cls.NameCString(params.name_visibility));
      printer->AddString(".");
    }
  }

  printer->AddString(fun.NameCString(params.name_visibility));

  // If we skipped rewritten async/async*/sync* body then append a suffix
  // to the end of the name.
  if (fun.raw() != raw() && params.disambiguate_names) {
    printer->AddString("{body}");
  }

  // Field dispatchers are specialized for an argument descriptor so there
  // might be multiples of them with the same name but different argument
  // descriptors. Add a suffix to disambiguate.
  if (params.disambiguate_names && fun.IsInvokeFieldDispatcher()) {
    printer->AddString(" ");
    if (NumTypeParameters() != 0) {
      printer->Printf("<%" Pd ">", fun.NumTypeParameters());
    }
    printer->AddString("(");
    printer->Printf("%" Pd "", fun.num_fixed_parameters());
    if (fun.NumOptionalPositionalParameters() != 0) {
      printer->Printf(" [%" Pd "]", fun.NumOptionalPositionalParameters());
    }
    if (fun.NumOptionalNamedParameters() != 0) {
      printer->AddString(" {");
      String& name = String::Handle();
      for (intptr_t i = 0; i < fun.NumOptionalNamedParameters(); i++) {
        name = fun.ParameterNameAt(fun.num_fixed_parameters() + i);
        printer->Printf("%s%s", i > 0 ? ", " : "", name.ToCString());
      }
      printer->AddString("}");
    }
    printer->AddString(")");
  }
}

StringPtr Function::GetSource() const {
  if (IsImplicitConstructor() || IsSignatureFunction() || is_synthetic()) {
    // We may need to handle more cases when the restrictions on mixins are
    // relaxed. In particular we might start associating some source with the
    // forwarding constructors when it becomes possible to specify a particular
    // constructor from the mixin to use.
    return String::null();
  }
  Zone* zone = Thread::Current()->zone();
  const Script& func_script = Script::Handle(zone, script());

  intptr_t from_line;
  intptr_t from_col;
  intptr_t to_line;
  intptr_t to_col;
  intptr_t to_length;
  func_script.GetTokenLocation(token_pos(), &from_line, &from_col);
  func_script.GetTokenLocation(end_token_pos(), &to_line, &to_col, &to_length);

  if (to_length == 1) {
    // Handle special cases for end tokens of closures (where we exclude the
    // last token):
    // (1) "foo(() => null, bar);": End token is `,', but we don't print it.
    // (2) "foo(() => null);": End token is ')`, but we don't print it.
    // (3) "var foo = () => null;": End token is `;', but in this case the
    // token semicolon belongs to the assignment so we skip it.
    const String& src = String::Handle(func_script.Source());
    if (src.IsNull() || src.Length() == 0) {
      return Symbols::OptimizedOut().raw();
    }
    uint16_t end_char = src.CharAt(end_token_pos().value());
    if ((end_char == ',') ||  // Case 1.
        (end_char == ')') ||  // Case 2.
        (end_char == ';' && String::Handle(zone, name())
                                .Equals("<anonymous closure>"))) {  // Case 3.
      to_length = 0;
    }
  }

  return func_script.GetSnippet(from_line, from_col, to_line,
                                to_col + to_length);
}

// Construct fingerprint from token stream. The token stream contains also
// arguments.
int32_t Function::SourceFingerprint() const {
#if !defined(DART_PRECOMPILED_RUNTIME)
  if (is_declared_in_bytecode()) {
    return kernel::BytecodeFingerprintHelper::CalculateFunctionFingerprint(
        *this);
  }
  return kernel::KernelSourceFingerprintHelper::CalculateFunctionFingerprint(
      *this);
#else
  return 0;
#endif  // !defined(DART_PRECOMPILED_RUNTIME)
}

void Function::SaveICDataMap(
    const ZoneGrowableArray<const ICData*>& deopt_id_to_ic_data,
    const Array& edge_counters_array) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
  // Compute number of ICData objects to save.
  // Store edge counter array in the first slot.
  intptr_t count = 1;
  for (intptr_t i = 0; i < deopt_id_to_ic_data.length(); i++) {
    if (deopt_id_to_ic_data[i] != NULL) {
      count++;
    }
  }
  const Array& array = Array::Handle(Array::New(count, Heap::kOld));
  count = 1;
  for (intptr_t i = 0; i < deopt_id_to_ic_data.length(); i++) {
    if (deopt_id_to_ic_data[i] != NULL) {
      ASSERT(i == deopt_id_to_ic_data[i]->deopt_id());
      array.SetAt(count++, *deopt_id_to_ic_data[i]);
    }
  }
  array.SetAt(0, edge_counters_array);
  set_ic_data_array(array);
#else   // DART_PRECOMPILED_RUNTIME
  UNREACHABLE();
#endif  // DART_PRECOMPILED_RUNTIME
}

void Function::RestoreICDataMap(
    ZoneGrowableArray<const ICData*>* deopt_id_to_ic_data,
    bool clone_ic_data) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
  if (FLAG_force_clone_compiler_objects) {
    clone_ic_data = true;
  }
  ASSERT(deopt_id_to_ic_data->is_empty());
  Zone* zone = Thread::Current()->zone();
  const Array& saved_ic_data = Array::Handle(zone, ic_data_array());
  if (saved_ic_data.IsNull()) {
    // Could happen with deferred loading.
    return;
  }
  const intptr_t saved_length = saved_ic_data.Length();
  ASSERT(saved_length > 0);
  if (saved_length > 1) {
    const intptr_t restored_length =
        ICData::Cast(Object::Handle(zone, saved_ic_data.At(saved_length - 1)))
            .deopt_id() +
        1;
    deopt_id_to_ic_data->SetLength(restored_length);
    for (intptr_t i = 0; i < restored_length; i++) {
      (*deopt_id_to_ic_data)[i] = NULL;
    }
    for (intptr_t i = 1; i < saved_length; i++) {
      ICData& ic_data = ICData::ZoneHandle(zone);
      ic_data ^= saved_ic_data.At(i);
      if (clone_ic_data) {
        const ICData& original_ic_data = ICData::Handle(zone, ic_data.raw());
        ic_data = ICData::Clone(ic_data);
        ic_data.SetOriginal(original_ic_data);
      }
      ASSERT(deopt_id_to_ic_data->At(ic_data.deopt_id()) == nullptr);
      (*deopt_id_to_ic_data)[ic_data.deopt_id()] = &ic_data;
    }
  }
#else   // DART_PRECOMPILED_RUNTIME
  UNREACHABLE();
#endif  // DART_PRECOMPILED_RUNTIME
}

void Function::set_ic_data_array(const Array& value) const {
  StorePointer<ArrayPtr, std::memory_order_release>(&raw_ptr()->ic_data_array_,
                                                    value.raw());
}

ArrayPtr Function::ic_data_array() const {
  return LoadPointer<ArrayPtr, std::memory_order_acquire>(
      &raw_ptr()->ic_data_array_);
}

void Function::ClearICDataArray() const {
  set_ic_data_array(Array::null_array());
}

ICDataPtr Function::FindICData(intptr_t deopt_id) const {
  const Array& array = Array::Handle(ic_data_array());
  ICData& ic_data = ICData::Handle();
  for (intptr_t i = 1; i < array.Length(); i++) {
    ic_data ^= array.At(i);
    if (ic_data.deopt_id() == deopt_id) {
      return ic_data.raw();
    }
  }
  return ICData::null();
}

void Function::SetDeoptReasonForAll(intptr_t deopt_id,
                                    ICData::DeoptReasonId reason) {
  const Array& array = Array::Handle(ic_data_array());
  ICData& ic_data = ICData::Handle();
  for (intptr_t i = 1; i < array.Length(); i++) {
    ic_data ^= array.At(i);
    if (ic_data.deopt_id() == deopt_id) {
      ic_data.AddDeoptReason(reason);
    }
  }
}

bool Function::CheckSourceFingerprint(int32_t fp) const {
#if !defined(DEBUG)
  return true;  // Only check on debug.
#endif

  if (Isolate::Current()->obfuscate() || FLAG_precompiled_mode ||
      (Dart::vm_snapshot_kind() != Snapshot::kNone)) {
    return true;  // The kernel structure has been altered, skip checking.
  }

  if (is_declared_in_bytecode()) {
    // AST and bytecode compute different fingerprints, and we only track one
    // fingerprint set.
    return true;
  }

  if (SourceFingerprint() != fp) {
    // This output can be copied into a file, then used with sed
    // to replace the old values.
    // sed -i.bak -f /tmp/newkeys \
    //    runtime/vm/compiler/recognized_methods_list.h
    THR_Print("s/0x%08x/0x%08x/\n", fp, SourceFingerprint());
    return false;
  }
  return true;
}

CodePtr Function::EnsureHasCode() const {
  if (HasCode()) return CurrentCode();
  Thread* thread = Thread::Current();
  ASSERT(thread->IsMutatorThread());
  DEBUG_ASSERT(thread->TopErrorHandlerIsExitFrame());
  Zone* zone = thread->zone();
  const Object& result =
      Object::Handle(zone, Compiler::CompileFunction(thread, *this));
  if (result.IsError()) {
    if (result.IsLanguageError()) {
      Exceptions::ThrowCompileTimeError(LanguageError::Cast(result));
      UNREACHABLE();
    }
    Exceptions::PropagateError(Error::Cast(result));
    UNREACHABLE();
  }
  // Compiling in unoptimized mode should never fail if there are no errors.
  ASSERT(HasCode());
  ASSERT(ForceOptimize() || unoptimized_code() == result.raw());
  return CurrentCode();
}

bool Function::NeedsMonomorphicCheckedEntry(Zone* zone) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
  if (!IsDynamicFunction()) {
    return false;
  }

  // For functions which need an args descriptor the switchable call sites will
  // transition directly to calling via a stub (and therefore never call the
  // monomorphic entry).
  //
  // See runtime_entry.cc:DEFINE_RUNTIME_ENTRY(UnlinkedCall)
  if (PrologueNeedsArgumentsDescriptor()) {
    return false;
  }

  // All dyn:* forwarders are called via SwitchableCalls and all except the ones
  // with `PrologueNeedsArgumentsDescriptor()` transition into monomorphic
  // state.
  if (Function::IsDynamicInvocationForwarderName(name())) {
    return true;
  }

  // If table dispatch is disabled, all instance calls use switchable calls.
  if (!(FLAG_precompiled_mode && FLAG_use_bare_instructions &&
        FLAG_use_table_dispatch)) {
    return true;
  }

  // Only if there are dynamic callers and if we didn't create a dyn:* forwarder
  // for it do we need the monomorphic checked entry.
  return HasDynamicCallers(zone) &&
         !kernel::NeedsDynamicInvocationForwarder(*this);
#else
  UNREACHABLE();
  return true;
#endif
}

bool Function::HasDynamicCallers(Zone* zone) const {
#if !defined(DART_PRECOMPILED_RUNTIME)
  // Issue(dartbug.com/42719):
  // Right now the metadata of _Closure.call says there are no dynamic callers -
  // even though there can be. To be conservative we return true.
  if ((name() == Symbols::GetCall().raw() || name() == Symbols::Call().raw()) &&
      Class::IsClosureClass(Owner())) {
    return true;
  }

  // Use the results of TFA to determine whether this function is ever
  // called dynamically, i.e. using switchable calls.
  kernel::ProcedureAttributesMetadata metadata;
  metadata = kernel::ProcedureAttributesOf(*this, zone);
  if (IsGetterFunction() || IsImplicitGetterFunction() || IsMethodExtractor()) {
    return metadata.getter_called_dynamically;
  } else {
    return metadata.method_or_setter_called_dynamically;
  }
#else
  UNREACHABLE();
  return true;
#endif
}

bool Function::PrologueNeedsArgumentsDescriptor() const {
  // The prologue of those functions need to examine the arg descriptor for
  // various purposes.
  return IsGeneric() || HasOptionalParameters();
}

bool Function::MayHaveUncheckedEntryPoint() const {
  return FLAG_enable_multiple_entrypoints &&
         (NeedsArgumentTypeChecks() || IsImplicitClosureFunction());
}

const char* Function::ToCString() const {
  if (IsNull()) {
    return "Function: null";
  }
  Zone* zone = Thread::Current()->zone();
  ZoneTextBuffer buffer(zone);
  buffer.Printf("Function '%s':", String::Handle(zone, name()).ToCString());
  if (is_static()) {
    buffer.AddString(" static");
  }
  if (is_abstract()) {
    buffer.AddString(" abstract");
  }
  switch (kind()) {
    case FunctionLayout::kRegularFunction:
    case FunctionLayout::kClosureFunction:
    case FunctionLayout::kImplicitClosureFunction:
    case FunctionLayout::kGetterFunction:
    case FunctionLayout::kSetterFunction:
      break;
    case FunctionLayout::kSignatureFunction:
      buffer.AddString(" signature");
      break;
    case FunctionLayout::kConstructor:
      buffer.AddString(is_static() ? " factory" : " constructor");
      break;
    case FunctionLayout::kImplicitGetter:
      buffer.AddString(" getter");
      break;
    case FunctionLayout::kImplicitSetter:
      buffer.AddString(" setter");
      break;
    case FunctionLayout::kImplicitStaticGetter:
      buffer.AddString(" static-getter");
      break;
    case FunctionLayout::kFieldInitializer:
      buffer.AddString(" field-initializer");
      break;
    case FunctionLayout::kMethodExtractor:
      buffer.AddString(" method-extractor");
      break;
    case FunctionLayout::kNoSuchMethodDispatcher:
      buffer.AddString(" no-such-method-dispatcher");
      break;
    case FunctionLayout::kDynamicInvocationForwarder:
      buffer.AddString(" dynamic-invocation-forwarder");
      break;
    case FunctionLayout::kInvokeFieldDispatcher:
      buffer.AddString(" invoke-field-dispatcher");
      break;
    case FunctionLayout::kIrregexpFunction:
      buffer.AddString(" irregexp-function");
      break;
    case FunctionLayout::kFfiTrampoline:
      buffer.AddString(" ffi-trampoline-function");
      break;
    default:
      UNREACHABLE();
  }
  if (IsNoSuchMethodDispatcher() || IsInvokeFieldDispatcher()) {
    const auto& args_desc_array = Array::Handle(zone, saved_args_desc());
    const ArgumentsDescriptor args_desc(args_desc_array);
    buffer.AddChar('[');
    args_desc.PrintTo(&buffer);
    buffer.AddChar(']');
  }
  if (is_const()) {
    buffer.AddString(" const");
  }
  buffer.AddChar('.');
  return buffer.buffer();
}
```

