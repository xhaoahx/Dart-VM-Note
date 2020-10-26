# Thread

## thrad.h

```c++
// ...

namespace dart {

class AbstractType;
class ApiLocalScope;
class Array;
class CompilerState;
class Class;
class Code;
class Bytecode;
class Error;
class ExceptionHandlers;
class Field;
class FieldTable;
class Function;
class GrowableObjectArray;
class HandleScope;
class Heap;
class HierarchyInfo;
class Instance;
class Interpreter;
class Isolate;
class IsolateGroup;
class Library;
class Object;
class OSThread;
class JSONObject;
class PcDescriptors;
class RuntimeEntry;
class Smi;
class StackResource;
class StackTrace;
class String;
class TimelineStream;
class TypeArguments;
class TypeParameter;
class TypeUsageInfo;
class Zone;

namespace compiler {
namespace target {
class Thread;
}  // namespace target
}  // namespace compiler

/// 可重用 Handle
#define REUSABLE_HANDLE_LIST(V)                                                \
  V(AbstractType)                                                              \
  V(Array)                                                                     \
  V(Class)                                                                     \
  V(Code)                                                                      \
  V(Bytecode)                                                                  \
  V(Error)                                                                     \
  V(ExceptionHandlers)                                                         \
  V(Field)                                                                     \
  V(Function)                                                                  \
  V(GrowableObjectArray)                                                       \
  V(Instance)                                                                  \
  V(Library)                                                                   \
  V(Object)                                                                    \
  V(PcDescriptors)                                                             \
  V(Smi)                                                                       \
  V(String)                                                                    \
  V(TypeArguments)                                                             \
  V(TypeParameter)

// 缓存 VM 存根
#define CACHED_VM_STUBS_LIST(V)                                                \
  V(CodePtr, write_barrier_code_, StubCode::WriteBarrier().raw(), nullptr)     \
  V(CodePtr, array_write_barrier_code_, StubCode::ArrayWriteBarrier().raw(),   \
    nullptr)                                                                   \
  V(CodePtr, fix_callers_target_code_, StubCode::FixCallersTarget().raw(),     \
    nullptr)                                                                   \
  V(CodePtr, fix_allocation_stub_code_,                                        \
    StubCode::FixAllocationStubTarget().raw(), nullptr)                        \
  V(CodePtr, invoke_dart_code_stub_, StubCode::InvokeDartCode().raw(),         \
    nullptr)                                                                   \
  V(CodePtr, invoke_dart_code_from_bytecode_stub_,                             \
    StubCode::InvokeDartCodeFromBytecode().raw(), nullptr)                     \
  V(CodePtr, call_to_runtime_stub_, StubCode::CallToRuntime().raw(), nullptr)  \
  V(CodePtr, null_error_shared_without_fpu_regs_stub_,                         \
    StubCode::NullErrorSharedWithoutFPURegs().raw(), nullptr)                  \
  V(CodePtr, null_error_shared_with_fpu_regs_stub_,                            \
    StubCode::NullErrorSharedWithFPURegs().raw(), nullptr)                     \
  V(CodePtr, null_arg_error_shared_without_fpu_regs_stub_,                     \
    StubCode::NullArgErrorSharedWithoutFPURegs().raw(), nullptr)               \
  V(CodePtr, null_arg_error_shared_with_fpu_regs_stub_,                        \
    StubCode::NullArgErrorSharedWithFPURegs().raw(), nullptr)                  \
  V(CodePtr, null_cast_error_shared_without_fpu_regs_stub_,                    \
    StubCode::NullCastErrorSharedWithoutFPURegs().raw(), nullptr)              \
  V(CodePtr, null_cast_error_shared_with_fpu_regs_stub_,                       \
    StubCode::NullCastErrorSharedWithFPURegs().raw(), nullptr)                 \
  V(CodePtr, range_error_shared_without_fpu_regs_stub_,                        \
    StubCode::RangeErrorSharedWithoutFPURegs().raw(), nullptr)                 \
  V(CodePtr, range_error_shared_with_fpu_regs_stub_,                           \
    StubCode::RangeErrorSharedWithFPURegs().raw(), nullptr)                    \
  V(CodePtr, allocate_mint_with_fpu_regs_stub_,                                \
    StubCode::AllocateMintSharedWithFPURegs().raw(), nullptr)                  \
  V(CodePtr, allocate_mint_without_fpu_regs_stub_,                             \
    StubCode::AllocateMintSharedWithoutFPURegs().raw(), nullptr)               \
  V(CodePtr, allocate_object_stub_, StubCode::AllocateObject().raw(), nullptr) \
  V(CodePtr, allocate_object_parameterized_stub_,                              \
    StubCode::AllocateObjectParameterized().raw(), nullptr)                    \
  V(CodePtr, allocate_object_slow_stub_, StubCode::AllocateObjectSlow().raw(), \
    nullptr)                                                                   \
  V(CodePtr, stack_overflow_shared_without_fpu_regs_stub_,                     \
    StubCode::StackOverflowSharedWithoutFPURegs().raw(), nullptr)              \
  V(CodePtr, stack_overflow_shared_with_fpu_regs_stub_,                        \
    StubCode::StackOverflowSharedWithFPURegs().raw(), nullptr)                 \
  V(CodePtr, switchable_call_miss_stub_, StubCode::SwitchableCallMiss().raw(), \
    nullptr)                                                                   \
  V(CodePtr, throw_stub_, StubCode::Throw().raw(), nullptr)                    \
  V(CodePtr, re_throw_stub_, StubCode::Throw().raw(), nullptr)                 \
  V(CodePtr, assert_boolean_stub_, StubCode::AssertBoolean().raw(), nullptr)   \
  V(CodePtr, optimize_stub_, StubCode::OptimizeFunction().raw(), nullptr)      \
  V(CodePtr, deoptimize_stub_, StubCode::Deoptimize().raw(), nullptr)          \
  V(CodePtr, lazy_deopt_from_return_stub_,                                     \
    StubCode::DeoptimizeLazyFromReturn().raw(), nullptr)                       \
  V(CodePtr, lazy_deopt_from_throw_stub_,                                      \
    StubCode::DeoptimizeLazyFromThrow().raw(), nullptr)                        \
  V(CodePtr, slow_type_test_stub_, StubCode::SlowTypeTest().raw(), nullptr)    \
  V(CodePtr, lazy_specialize_type_test_stub_,                                  \
    StubCode::LazySpecializeTypeTest().raw(), nullptr)                         \
  V(CodePtr, enter_safepoint_stub_, StubCode::EnterSafepoint().raw(), nullptr) \
  V(CodePtr, exit_safepoint_stub_, StubCode::ExitSafepoint().raw(), nullptr)   \
  V(CodePtr, call_native_through_safepoint_stub_,                              \
    StubCode::CallNativeThroughSafepoint().raw(), nullptr)

// 缓存非 VM 存根
#define CACHED_NON_VM_STUB_LIST(V)                                             \
  // 空指针 
  V(ObjectPtr, object_null_, Object::null(), nullptr)                          \
  // 常量 true 
  V(BoolPtr, bool_true_, Object::bool_true().raw(), nullptr)                   \
  // 常量 false
  V(BoolPtr, bool_false_, Object::bool_false().raw(), nullptr)

// VM 全局对象、地址在每一个线程之中的缓存
// 注意：产量 false 必须直接跟在常量 true 之后
// List of VM-global objects/addresses cached in each Thread object.
// Important: constant false must immediately follow constant true.
#define CACHED_VM_OBJECTS_LIST(V)                                              \
  CACHED_NON_VM_STUB_LIST(V)                                                   \
  CACHED_VM_STUBS_LIST(V)

// This assertion marks places which assume that boolean false immediate
// follows bool true in the CACHED_VM_OBJECTS_LIST
#define ASSERT_BOOL_FALSE_FOLLOWS_BOOL_TRUE()                                  \
  ASSERT((Thread::bool_true_offset() + kWordSize) ==                           \
         Thread::bool_false_offset());

// 缓存 VM 存根地址
#define CACHED_VM_STUBS_ADDRESSES_LIST(V)                                      \
  V(uword, write_barrier_entry_point_, StubCode::WriteBarrier().EntryPoint(),  \
    0)                                                                         \
  V(uword, array_write_barrier_entry_point_,                                   \
    StubCode::ArrayWriteBarrier().EntryPoint(), 0)                             \
  V(uword, call_to_runtime_entry_point_,                                       \
    StubCode::CallToRuntime().EntryPoint(), 0)                                 \
  V(uword, allocate_mint_with_fpu_regs_entry_point_,                           \
    StubCode::AllocateMintSharedWithFPURegs().EntryPoint(), 0)                 \
  V(uword, allocate_mint_without_fpu_regs_entry_point_,                        \
    StubCode::AllocateMintSharedWithoutFPURegs().EntryPoint(), 0)              \
  V(uword, allocate_object_entry_point_,                                       \
    StubCode::AllocateObject().EntryPoint(), 0)                                \
  V(uword, allocate_object_parameterized_entry_point_,                         \
    StubCode::AllocateObjectParameterized().EntryPoint(), 0)                   \
  V(uword, allocate_object_slow_entry_point_,                                  \
    StubCode::AllocateObjectSlow().EntryPoint(), 0)                            \
  V(uword, stack_overflow_shared_without_fpu_regs_entry_point_,                \
    StubCode::StackOverflowSharedWithoutFPURegs().EntryPoint(), 0)             \
  V(uword, stack_overflow_shared_with_fpu_regs_entry_point_,                   \
    StubCode::StackOverflowSharedWithFPURegs().EntryPoint(), 0)                \
  V(uword, megamorphic_call_checked_entry_,                                    \
    StubCode::MegamorphicCall().EntryPoint(), 0)                               \
  V(uword, switchable_call_miss_entry_,                                        \
    StubCode::SwitchableCallMiss().EntryPoint(), 0)                            \
  V(uword, optimize_entry_, StubCode::OptimizeFunction().EntryPoint(), 0)      \
  V(uword, deoptimize_entry_, StubCode::Deoptimize().EntryPoint(), 0)          \
  V(uword, call_native_through_safepoint_entry_point_,                         \
    StubCode::CallNativeThroughSafepoint().EntryPoint(), 0)                    \
  V(uword, slow_type_test_entry_point_, StubCode::SlowTypeTest().EntryPoint(), \
    0)

// 缓存地址
#define CACHED_ADDRESSES_LIST(V)                                               \
  CACHED_VM_STUBS_ADDRESSES_LIST(V)                                            \
  V(uword, bootstrap_native_wrapper_entry_point_,                              \
    NativeEntry::BootstrapNativeCallWrapperEntry(), 0)                         \
  V(uword, no_scope_native_wrapper_entry_point_,                               \
    NativeEntry::NoScopeNativeCallWrapperEntry(), 0)                           \
  V(uword, auto_scope_native_wrapper_entry_point_,                             \
    NativeEntry::AutoScopeNativeCallWrapperEntry(), 0)                         \
  V(uword, interpret_call_entry_point_, RuntimeEntry::InterpretCallEntry(), 0) \
  V(StringPtr*, predefined_symbols_address_, Symbols::PredefinedAddress(),     \
    NULL)                                                                      \
  V(uword, double_nan_address_, reinterpret_cast<uword>(&double_nan_constant), \
    0)                                                                         \
  V(uword, double_negate_address_,                                             \
    reinterpret_cast<uword>(&double_negate_constant), 0)                       \
  V(uword, double_abs_address_, reinterpret_cast<uword>(&double_abs_constant), \
    0)                                                                         \
  V(uword, float_not_address_, reinterpret_cast<uword>(&float_not_constant),   \
    0)                                                                         \
  V(uword, float_negate_address_,                                              \
    reinterpret_cast<uword>(&float_negate_constant), 0)                        \
  V(uword, float_absolute_address_,                                            \
    reinterpret_cast<uword>(&float_absolute_constant), 0)                      \
  V(uword, float_zerow_address_,                                               \
    reinterpret_cast<uword>(&float_zerow_constant), 0)

// 缓存常量表
#define CACHED_CONSTANTS_LIST(V)                                               \
  CACHED_VM_OBJECTS_LIST(V)                                                    \
  CACHED_ADDRESSES_LIST(V)

enum class ValidationPolicy {
  kValidateFrames = 0,
  kDontValidateFrames = 1,
};

// VM 线程。
// 可能用于执行 Dart 代码或者执行 helper（例如垃圾回收或者编译）任务。此类与关联的线程是在进入一个 Isolate 之前， 
// 通过 EnsureInit 函数类分配的，并且在底层操作系统线程退出的时候会自动地销毁。
// 注意：在 Windows 平台，CleanUp 必须手动调用
class Thread : public ThreadState {
 public:
  // 此线程执行的任务
  enum TaskKind {
    kUnknownTask = 0x0,
    // 赋值器，执行代码
    kMutatorTask = 0x1,
    // 编译
    kCompilerTask = 0x2,
    // 同步标记
    kMarkerTask = 0x4,
    // 清理
    kSweeperTask = 0x8,
    // 整理
    kCompactorTask = 0x10,
    // 新生代垃圾回收
    kScavengerTask = 0x20,
  };
  // Converts a TaskKind to its corresponding C-String name.
  static const char* TaskKindToCString(TaskKind kind);

  ~Thread();

  // 当前正在执行的线程，如果没有，返回 NULL
  static Thread* Current() {
#if defined(HAS_C11_THREAD_LOCAL)
    return static_cast<Thread*>(OSThread::CurrentVMThread());
#else
    BaseThread* thread = OSThread::GetCurrentTLS();
    if (thread == NULL || thread->is_os_thread()) {
      return NULL;
    }
    return static_cast<Thread*>(thread);
#endif
  }

  // 使当前线程进入 Isolate
  static bool EnterIsolate(Isolate* isolate);
  // 使当前线程退出 Isolate
  static void ExitIsolate();

  // 不同于 main mutator 线程的其他线程可以作为 helper 进入某个 Isolate 来获得对此 Isolate 的有限并发访问。
  // 例如：SweeperTask（使用 class table，它是 写时拷贝的）
  /// bypass_safepoint 用于无视安全点
  static bool EnterIsolateAsHelper(Isolate* isolate,
                                   TaskKind kind,
                                   bool bypass_safepoint = false);
  static void ExitIsolateAsHelper(bool bypass_safepoint = false);

  static bool EnterIsolateGroupAsHelper(IsolateGroup* isolate_group,
                                        TaskKind kind,
                                        bool bypass_safepoint);
  static void ExitIsolateGroupAsHelper(bool bypass_safepoint);

  // 释放存储块归还给 Isolate
  void ReleaseStoreBuffer();
  void AcquireMarkingStack();
  void ReleaseMarkingStack();

  // 设置栈极限
  void SetStackLimit(uword value);
  // 清空栈极限
  void ClearStackLimit();

  // 访问当前的栈极限。
  // Access to the current stack limit for generated code. Either the true OS
  // thread's stack limit minus some headroom, or a special value to trigger
  // interrupts.
  uword stack_limit_address() const {
    return reinterpret_cast<uword>(&stack_limit_);
  }
  static intptr_t stack_limit_offset() {
    return OFFSET_OF(Thread, stack_limit_);
  }

  // 操作系统线程的栈极限
  static intptr_t saved_stack_limit_offset() {
    return OFFSET_OF(Thread, saved_stack_limit_);
  }
  uword saved_stack_limit() const { return saved_stack_limit_; }

#if defined(USING_SAFE_STACK)
  uword saved_safestack_limit() const { return saved_safestack_limit_; }
  void set_saved_safestack_limit(uword limit) {
    saved_safestack_limit_ = limit;
  }
#endif
  static uword saved_shadow_call_stack_offset() {
    return OFFSET_OF(Thread, saved_shadow_call_stack_);
  }

  // Stack overflow flags
  enum {
    kOsrRequest = 0x1,  // Current stack overflow caused by OSR request.
  };

  // 写屏障 mask
  uword write_barrier_mask() const { return write_barrier_mask_; }

  static intptr_t write_barrier_mask_offset() {
    return OFFSET_OF(Thread, write_barrier_mask_);
  }
  static intptr_t stack_overflow_flags_offset() {
    return OFFSET_OF(Thread, stack_overflow_flags_);
  }

  int32_t IncrementAndGetStackOverflowCount() {
    return ++stack_overflow_count_;
  }

  static uword stack_overflow_shared_stub_entry_point_offset(bool fpu_regs) {
    return fpu_regs
               ? stack_overflow_shared_with_fpu_regs_entry_point_offset()
               : stack_overflow_shared_without_fpu_regs_entry_point_offset();
  }

  static intptr_t safepoint_state_offset() {
    return OFFSET_OF(Thread, safepoint_state_);
  }

  static intptr_t callback_code_offset() {
    return OFFSET_OF(Thread, ffi_callback_code_);
  }

  // Tag state is maintained on transitions.
  enum {
    // Always true in generated state.
    kDidNotExit = 0,
    // The VM did exit the generated state through FFI.
    // This can be true in both native and VM state.
    kExitThroughFfi = 1,
    // The VM exited the generated state through FFI.
    // This can be true in both native and VM state.
    kExitThroughRuntimeCall = 2,
  };

  static intptr_t exit_through_ffi_offset() {
    return OFFSET_OF(Thread, exit_through_ffi_);
  }

  // 任务类型
  TaskKind task_kind() const { return task_kind_; }

  // Retrieves and clears the stack overflow flags.  These are set by
  // the generated code before the slow path runtime routine for a
  // stack overflow is called.
  uword GetAndClearStackOverflowFlags();

  // 中断标记
  enum {
    // VM 内部 检查：安全点，储存缓冲等
    kVMInterrupt = 0x1,  // Internal VM checks: safepoints, store buffers, etc.
    // 处理带外消息的中断
    kMessageInterrupt = 0x2,  // An interrupt to process an out of band message.
    
    kInterruptsMask = (kVMInterrupt | kMessageInterrupt),
  };

  void ScheduleInterrupts(uword interrupt_bits);
  void ScheduleInterruptsLocked(uword interrupt_bits);
  ErrorPtr HandleInterrupts();
  uword GetAndClearInterrupts();
    
  // 是否处于中断状态
  bool HasScheduledInterrupts() const {
    return (stack_limit_ & kInterruptsMask) != 0;
  }

  // Monitor corresponding to this thread.
  Monitor* thread_lock() const { return &thread_lock_; }

  // The reusable api local scope for this thread.
  ApiLocalScope* api_reusable_scope() const { return api_reusable_scope_; }
  void set_api_reusable_scope(ApiLocalScope* value) {
    ASSERT(value == NULL || api_reusable_scope_ == NULL);
    api_reusable_scope_ = value;
  }

  // The api local scope for this thread, this where all local handles
  // are allocated.
  ApiLocalScope* api_top_scope() const { return api_top_scope_; }
  void set_api_top_scope(ApiLocalScope* value) { api_top_scope_ = value; }
  static intptr_t api_top_scope_offset() {
    return OFFSET_OF(Thread, api_top_scope_);
  }

  void EnterApiScope();
  void ExitApiScope();

  // The isolate that this thread is operating on, or nullptr if none.
  Isolate* isolate() const { return isolate_; }
  static intptr_t isolate_offset() { return OFFSET_OF(Thread, isolate_); }

  // The isolate group that this thread is operating on, or nullptr if none.
  IsolateGroup* isolate_group() const { return isolate_group_; }

  static intptr_t field_table_values_offset() {
    return OFFSET_OF(Thread, field_table_values_);
  }

  bool IsMutatorThread() const { return is_mutator_thread_; }

  bool CanCollectGarbage() const;

  // Offset of Dart TimelineStream object.
  static intptr_t dart_stream_offset() {
    return OFFSET_OF(Thread, dart_stream_);
  }

  // 此线程执行 Dart 代码？
  bool IsExecutingDartCode() const;

  // 此线程退出 Dart 代码？
  bool HasExitedDartCode() const;

  // 编译器状态
  CompilerState& compiler_state() {
    ASSERT(compiler_state_ != nullptr);
    return *compiler_state_;
  }

  HierarchyInfo* hierarchy_info() const {
    ASSERT(isolate_ != NULL);
    return hierarchy_info_;
  }

  void set_hierarchy_info(HierarchyInfo* value) {
    ASSERT(isolate_ != NULL);
    ASSERT((hierarchy_info_ == NULL && value != NULL) ||
           (hierarchy_info_ != NULL && value == NULL));
    hierarchy_info_ = value;
  }

  TypeUsageInfo* type_usage_info() const {
    ASSERT(isolate_ != NULL);
    return type_usage_info_;
  }

  void set_type_usage_info(TypeUsageInfo* value) {
    ASSERT(isolate_ != NULL);
    ASSERT((type_usage_info_ == NULL && value != NULL) ||
           (type_usage_info_ != NULL && value == NULL));
    type_usage_info_ = value;
  }

#if !defined(DART_PRECOMPILED_RUNTIME)
  Interpreter* interpreter() const { return interpreter_; }
  void set_interpreter(Interpreter* value) { interpreter_ = value; }
#endif

  int32_t no_callback_scope_depth() const { return no_callback_scope_depth_; }

  void IncrementNoCallbackScopeDepth() {
    ASSERT(no_callback_scope_depth_ < INT_MAX);
    no_callback_scope_depth_ += 1;
  }

  void DecrementNoCallbackScopeDepth() {
    ASSERT(no_callback_scope_depth_ > 0);
    no_callback_scope_depth_ -= 1;
  }

  void StoreBufferAddObject(ObjectPtr obj);
  void StoreBufferAddObjectGC(ObjectPtr obj);
#if defined(TESTING)
  bool StoreBufferContains(ObjectPtr obj) const {
    return store_buffer_block_->Contains(obj);
  }
#endif
  void StoreBufferBlockProcess(StoreBuffer::ThresholdPolicy policy);
  static intptr_t store_buffer_block_offset() {
    return OFFSET_OF(Thread, store_buffer_block_);
  }

  bool is_marking() const { return marking_stack_block_ != NULL; }
  void MarkingStackAddObject(ObjectPtr obj);
  void DeferredMarkingStackAddObject(ObjectPtr obj);
  void MarkingStackBlockProcess();
  void DeferredMarkingStackBlockProcess();
  static intptr_t marking_stack_block_offset() {
    return OFFSET_OF(Thread, marking_stack_block_);
  }

  uword top_exit_frame_info() const { return top_exit_frame_info_; }
  void set_top_exit_frame_info(uword top_exit_frame_info) {
    top_exit_frame_info_ = top_exit_frame_info;
  }
  static intptr_t top_exit_frame_info_offset() {
    return OFFSET_OF(Thread, top_exit_frame_info_);
  }

  // 此线程运行的 Isolate 持有的堆
  Heap* heap() const { return heap_; }
  static intptr_t heap_offset() { return OFFSET_OF(Thread, heap_); }

  /// 线程栈栈顶
  uword top() const { return top_; }
  /// 线程栈栈底
  uword end() const { return end_; }
  void set_top(uword top) { top_ = top; }
  void set_end(uword end) { end_ = end; }
  static intptr_t top_offset() { return OFFSET_OF(Thread, top_); }
  static intptr_t end_offset() { return OFFSET_OF(Thread, end_); }

  int32_t no_safepoint_scope_depth() const {
#if defined(DEBUG)
    return no_safepoint_scope_depth_;
#else
    return 0;
#endif
  }

  void IncrementNoSafepointScopeDepth() {
#if defined(DEBUG)
    ASSERT(no_safepoint_scope_depth_ < INT_MAX);
    no_safepoint_scope_depth_ += 1;
#endif
  }

  void DecrementNoSafepointScopeDepth() {
#if defined(DEBUG)
    ASSERT(no_safepoint_scope_depth_ > 0);
    no_safepoint_scope_depth_ -= 1;
#endif
  }

#define DEFINE_OFFSET_METHOD(type_name, member_name, expr, default_init_value) \
  static intptr_t member_name##offset() {                                      \
    return OFFSET_OF(Thread, member_name);                                     \
  }
  CACHED_CONSTANTS_LIST(DEFINE_OFFSET_METHOD)
#undef DEFINE_OFFSET_METHOD

#if defined(TARGET_ARCH_ARM) || defined(TARGET_ARCH_ARM64) ||                  \
    defined(TARGET_ARCH_X64)
  static intptr_t write_barrier_wrappers_thread_offset(Register reg) {
    ASSERT((kDartAvailableCpuRegs & (1 << reg)) != 0);
    intptr_t index = 0;
    for (intptr_t i = 0; i < kNumberOfCpuRegisters; ++i) {
      if ((kDartAvailableCpuRegs & (1 << i)) == 0) continue;
      if (i == reg) break;
      ++index;
    }
    return OFFSET_OF(Thread, write_barrier_wrappers_entry_points_) +
           index * sizeof(uword);
  }

  static intptr_t WriteBarrierWrappersOffsetForRegister(Register reg) {
    intptr_t index = 0;
    for (intptr_t i = 0; i < kNumberOfCpuRegisters; ++i) {
      if ((kDartAvailableCpuRegs & (1 << i)) == 0) continue;
      if (i == reg) {
        return index * kStoreBufferWrapperSize;
      }
      ++index;
    }
    UNREACHABLE();
    return 0;
  }
#endif

#define DEFINE_OFFSET_METHOD(name)                                             \
  static intptr_t name##_entry_point_offset() {                                \
    return OFFSET_OF(Thread, name##_entry_point_);                             \
  }
  RUNTIME_ENTRY_LIST(DEFINE_OFFSET_METHOD)
#undef DEFINE_OFFSET_METHOD

#define DEFINE_OFFSET_METHOD(returntype, name, ...)                            \
  static intptr_t name##_entry_point_offset() {                                \
    return OFFSET_OF(Thread, name##_entry_point_);                             \
  }
  LEAF_RUNTIME_ENTRY_LIST(DEFINE_OFFSET_METHOD)
#undef DEFINE_OFFSET_METHOD

  // 全局对象池
  ObjectPoolPtr global_object_pool() const { return global_object_pool_; }
  void set_global_object_pool(ObjectPoolPtr raw_value) {
    global_object_pool_ = raw_value;
  }

  const uword* dispatch_table_array() const { return dispatch_table_array_; }
  void set_dispatch_table_array(const uword* array) {
    dispatch_table_array_ = array;
  }

  static bool CanLoadFromThread(const Object& object);
  static intptr_t OffsetFromThread(const Object& object);
  static bool ObjectAtOffset(intptr_t offset, Object* object);
  static intptr_t OffsetFromThread(const RuntimeEntry* runtime_entry);

  // VM 标志
  uword vm_tag() const { return vm_tag_; }
  void set_vm_tag(uword tag) { vm_tag_ = tag; }
  static intptr_t vm_tag_offset() { return OFFSET_OF(Thread, vm_tag_); }

  int64_t unboxed_int64_runtime_arg() const {
    return unboxed_int64_runtime_arg_;
  }
  void set_unboxed_int64_runtime_arg(int64_t value) {
    unboxed_int64_runtime_arg_ = value;
  }
  static intptr_t unboxed_int64_runtime_arg_offset() {
    return OFFSET_OF(Thread, unboxed_int64_runtime_arg_);
  }

  GrowableObjectArrayPtr pending_functions();
  void clear_pending_functions();

  static intptr_t global_object_pool_offset() {
    return OFFSET_OF(Thread, global_object_pool_);
  }

  static intptr_t dispatch_table_array_offset() {
    return OFFSET_OF(Thread, dispatch_table_array_);
  }

  ObjectPtr active_exception() const { return active_exception_; }
  void set_active_exception(const Object& value);
  static intptr_t active_exception_offset() {
    return OFFSET_OF(Thread, active_exception_);
  }

  ObjectPtr active_stacktrace() const { return active_stacktrace_; }
  void set_active_stacktrace(const Object& value);
  static intptr_t active_stacktrace_offset() {
    return OFFSET_OF(Thread, active_stacktrace_);
  }

  uword resume_pc() const { return resume_pc_; }
  void set_resume_pc(uword value) { resume_pc_ = value; }
  static uword resume_pc_offset() { return OFFSET_OF(Thread, resume_pc_); }

  ErrorPtr sticky_error() const;
  void set_sticky_error(const Error& value);
  void ClearStickyError();
  DART_WARN_UNUSED_RESULT ErrorPtr StealStickyError();

  StackTracePtr async_stack_trace() const;
  void set_async_stack_trace(const StackTrace& stack_trace);
  void set_raw_async_stack_trace(StackTracePtr raw_stack_trace);
  void clear_async_stack_trace();
  static intptr_t async_stack_trace_offset() {
    return OFFSET_OF(Thread, async_stack_trace_);
  }

  /// 清除可重用 Handle
  void ClearReusableHandles();

#define REUSABLE_HANDLE(object)                                                \
  object& object##Handle() const { return *object##_handle_; }
  REUSABLE_HANDLE_LIST(REUSABLE_HANDLE)
#undef REUSABLE_HANDLE

  /*
   * 用于支持线程的安全点操作
   *
   * - safepoint_state_ 的 0 Bit 被用于表示此线程是否处于安全点
   * - safepoint_state_ 的 1 Bit 被用于表示此线程是否请求安全点操作
   * - safepoint_state_ 的 0 Bit 被用于表示此线程处于阻塞状态以等待安全点操作完成
   *
   * 安全点执行状态（如上所述）储存在 execution_state_ 域中
   * 线程可能的潜在的执行状态有：
   *   kThreadInGenerated - 此线程正在执行 Dart 代码
   *   kThreadInVM - 此线程正在运行 VM
   *   kThreadInNative - 此线程正在执行 NATIVE 代码
   *   kThreadInBlockedState - 此线程处于阻塞状态并且等待某个资源
   */
  static bool IsAtSafepoint(uword state) {
    return AtSafepointField::decode(state);
  }
  bool IsAtSafepoint() const {
    return AtSafepointField::decode(safepoint_state_);
  }
  static uword SetAtSafepoint(bool value, uword state) {
    return AtSafepointField::update(value, state);
  }
  void SetAtSafepoint(bool value) {
    ASSERT(thread_lock()->IsOwnedByCurrentThread());
    safepoint_state_ = AtSafepointField::update(value, safepoint_state_);
  }
  bool IsSafepointRequested() const {
    return SafepointRequestedField::decode(safepoint_state_);
  }
  static uword SetSafepointRequested(bool value, uword state) {
    return SafepointRequestedField::update(value, state);
  }
  uword SetSafepointRequested(bool value) {
    ASSERT(thread_lock()->IsOwnedByCurrentThread());
    if (value) {
      // acquire pulls from the release in TryEnterSafepoint.
      return safepoint_state_.fetch_or(SafepointRequestedField::encode(true),
                                       std::memory_order_acquire);
    } else {
      // release pushes to the acquire in TryExitSafepoint.
      return safepoint_state_.fetch_and(~SafepointRequestedField::encode(true),
                                        std::memory_order_release);
    }
  }
  static bool IsBlockedForSafepoint(uword state) {
    return BlockedForSafepointField::decode(state);
  }
  bool IsBlockedForSafepoint() const {
    return BlockedForSafepointField::decode(safepoint_state_);
  }
  void SetBlockedForSafepoint(bool value) {
    ASSERT(thread_lock()->IsOwnedByCurrentThread());
    safepoint_state_ =
        BlockedForSafepointField::update(value, safepoint_state_);
  }
  bool BypassSafepoints() const {
    return BypassSafepointsField::decode(safepoint_state_);
  }
  static uword SetBypassSafepoints(bool value, uword state) {
    return BypassSafepointsField::update(value, state);
  }

  /// 执行状态
  enum ExecutionState {
    kThreadInVM = 0,
    kThreadInGenerated,
    kThreadInNative,
    kThreadInBlockedState
  };

  ExecutionState execution_state() const {
    return static_cast<ExecutionState>(execution_state_);
  }
  // Normally execution state is only accessed for the current thread.
  NO_SANITIZE_THREAD
  ExecutionState execution_state_cross_thread_for_testing() const {
    return static_cast<ExecutionState>(execution_state_);
  }
  void set_execution_state(ExecutionState state) {
    execution_state_ = static_cast<uword>(state);
  }
  static intptr_t execution_state_offset() {
    return OFFSET_OF(Thread, execution_state_);
  }

  virtual bool MayAllocateHandles() {
    return (execution_state() == kThreadInVM) ||
           (execution_state() == kThreadInGenerated);
  }

  static uword safepoint_state_unacquired() { return SetAtSafepoint(false, 0); }
  static uword safepoint_state_acquired() { return SetAtSafepoint(true, 0); }

  bool TryEnterSafepoint() {
    uword old_state = 0;
    uword new_state = SetAtSafepoint(true, 0);
    return safepoint_state_.compare_exchange_strong(old_state, new_state,
                                                    std::memory_order_release);
  }

  void EnterSafepoint() {
    ASSERT(no_safepoint_scope_depth() == 0);
    // First try a fast update of the thread state to indicate it is at a
    // safepoint.
    if (!TryEnterSafepoint()) {
      // Fast update failed which means we could potentially be in the middle
      // of a safepoint operation.
      EnterSafepointUsingLock();
    }
  }

  bool TryExitSafepoint() {
    uword old_state = SetAtSafepoint(true, 0);
    uword new_state = 0;
    return safepoint_state_.compare_exchange_strong(old_state, new_state,
                                                    std::memory_order_acquire);
  }

  void ExitSafepoint() {
    // First try a fast update of the thread state to indicate it is not at a
    // safepoint anymore.
    if (!TryExitSafepoint()) {
      // Fast update failed which means we could potentially be in the middle
      // of a safepoint operation.
      ExitSafepointUsingLock();
    }
  }

  void CheckForSafepoint() {
    ASSERT(no_safepoint_scope_depth() == 0);
    if (IsSafepointRequested()) {
      BlockForSafepoint();
    }
  }

  int32_t AllocateFfiCallbackId();

  // Store 'code' for the native callback identified by 'callback_id'.
  //
  // Expands the callback code array as necessary to accomodate the callback ID.
  void SetFfiCallbackCode(int32_t callback_id, const Code& code);

  // Ensure that 'callback_id' refers to a valid callback in this isolate.
  //
  // If "entry != 0", additionally checks that entry is inside the instructions
  // of this callback.
  //
  // Aborts if any of these conditions fails.
  void VerifyCallbackIsolate(int32_t callback_id, uword entry);

  Thread* next() const { return next_; }

  // Visit all object pointers.
  void VisitObjectPointers(ObjectPointerVisitor* visitor,
                           ValidationPolicy validate_frames);
  void RememberLiveTemporaries();
  void DeferredMarkLiveTemporaries();

  bool IsValidHandle(Dart_Handle object) const;
  bool IsValidLocalHandle(Dart_Handle object) const;
  intptr_t CountLocalHandles() const;
  int ZoneSizeInBytes() const;
  void UnwindScopes(uword stack_marker);

  void InitVMConstants();

  uint64_t GetRandomUInt64() { return thread_random_.NextUInt64(); }

  uint64_t* GetFfiMarshalledArguments(intptr_t size) {
    if (ffi_marshalled_arguments_size_ < size) {
      if (ffi_marshalled_arguments_size_ > 0) {
        free(ffi_marshalled_arguments_);
      }
      ffi_marshalled_arguments_ =
          reinterpret_cast<uint64_t*>(malloc(size * sizeof(uint64_t)));
    }
    return ffi_marshalled_arguments_;
  }

#ifndef PRODUCT
  void PrintJSON(JSONStream* stream) const;
#endif

 private:
  template <class T>
  T* AllocateReusableHandle();

  enum class RestoreWriteBarrierInvariantOp {
    kAddToRememberedSet,
    kAddToDeferredMarkingStack
  };
  friend class RestoreWriteBarrierInvariantVisitor;
  void RestoreWriteBarrierInvariant(RestoreWriteBarrierInvariantOp op);

  // Set the current compiler state and return the previous compiler state.
  CompilerState* SetCompilerState(CompilerState* state) {
    CompilerState* previous = compiler_state_;
    compiler_state_ = state;
    return previous;
  }

  // Accessed from generated code.
  // ** This block of fields must come first! **
  // For AOT cross-compilation, we rely on these members having the same offsets
  // in SIMARM(IA32) and ARM, and the same offsets in SIMARM64(X64) and ARM64.
  // We use only word-sized fields to avoid differences in struct packing on the
  // different architectures. See also CheckOffsets in dart.cc.
  RelaxedAtomic<uword> stack_limit_;
  uword write_barrier_mask_;
  Isolate* isolate_;
  const uword* dispatch_table_array_;
  uword top_ = 0;
  uword end_ = 0;
  // Offsets up to this point can all fit in a byte on X64. All of the above
  // fields are very abundantly accessed from code. Thus, keeping them first
  // is important for code size (although code size on X64 is not a priority).
  uword saved_stack_limit_;
  uword stack_overflow_flags_;
  InstancePtr* field_table_values_;
  Heap* heap_;
  uword volatile top_exit_frame_info_;
  StoreBufferBlock* store_buffer_block_;
  MarkingStackBlock* marking_stack_block_;
  MarkingStackBlock* deferred_marking_stack_block_;
  uword volatile vm_tag_;
  StackTracePtr async_stack_trace_;
  // Memory location dedicated for passing unboxed int64 values from
  // generated code to runtime.
  // TODO(dartbug.com/33549): Clean this up when unboxed values
  // could be passed as arguments.
  ALIGN8 int64_t unboxed_int64_runtime_arg_;

// State that is cached in the TLS for fast access in generated code.
#define DECLARE_MEMBERS(type_name, member_name, expr, default_init_value)      \
  type_name member_name;
  CACHED_CONSTANTS_LIST(DECLARE_MEMBERS)
#undef DECLARE_MEMBERS

#define DECLARE_MEMBERS(name) uword name##_entry_point_;
  RUNTIME_ENTRY_LIST(DECLARE_MEMBERS)
#undef DECLARE_MEMBERS

#define DECLARE_MEMBERS(returntype, name, ...) uword name##_entry_point_;
  LEAF_RUNTIME_ENTRY_LIST(DECLARE_MEMBERS)
#undef DECLARE_MEMBERS

#if defined(TARGET_ARCH_ARM) || defined(TARGET_ARCH_ARM64) ||                  \
    defined(TARGET_ARCH_X64)
  uword write_barrier_wrappers_entry_points_[kNumberOfDartAvailableCpuRegs];
#endif

  // JumpToExceptionHandler state:
  ObjectPtr active_exception_;
  ObjectPtr active_stacktrace_;
  ObjectPoolPtr global_object_pool_;
  uword resume_pc_;
  uword saved_shadow_call_stack_ = 0;
  uword execution_state_;
  std::atomic<uword> safepoint_state_;
  GrowableObjectArrayPtr ffi_callback_code_;
  uword exit_through_ffi_ = 0;
  ApiLocalScope* api_top_scope_;

  // ---- End accessed from generated code. ----

  // The layout of Thread object up to this point should not depend
  // on DART_PRECOMPILED_RUNTIME, as it is accessed from generated code.
  // The code is generated without DART_PRECOMPILED_RUNTIME, but used with
  // DART_PRECOMPILED_RUNTIME.

  TaskKind task_kind_;
  TimelineStream* dart_stream_;
  IsolateGroup* isolate_group_ = nullptr;
  mutable Monitor thread_lock_;
  ApiLocalScope* api_reusable_scope_;
  int32_t no_callback_scope_depth_;
#if defined(DEBUG)
  int32_t no_safepoint_scope_depth_;
#endif
  VMHandles reusable_handles_;
  intptr_t defer_oob_messages_count_;
  uint16_t deferred_interrupts_mask_;
  uint16_t deferred_interrupts_;
  int32_t stack_overflow_count_;

  // Compiler state:
  CompilerState* compiler_state_ = nullptr;
  HierarchyInfo* hierarchy_info_;
  TypeUsageInfo* type_usage_info_;
  GrowableObjectArrayPtr pending_functions_;

  ErrorPtr sticky_error_;

  Random thread_random_;

  intptr_t ffi_marshalled_arguments_size_ = 0;
  uint64_t* ffi_marshalled_arguments_;

  InstancePtr* field_table_values() const { return field_table_values_; }

// Reusable handles support.
#define REUSABLE_HANDLE_FIELDS(object) object* object##_handle_;
  REUSABLE_HANDLE_LIST(REUSABLE_HANDLE_FIELDS)
#undef REUSABLE_HANDLE_FIELDS

#if defined(DEBUG)
#define REUSABLE_HANDLE_SCOPE_VARIABLE(object)                                 \
  bool reusable_##object##_handle_scope_active_;
  REUSABLE_HANDLE_LIST(REUSABLE_HANDLE_SCOPE_VARIABLE);
#undef REUSABLE_HANDLE_SCOPE_VARIABLE
#endif  // defined(DEBUG)

  // Generated code assumes that AtSafepointField is the LSB.
  class AtSafepointField : public BitField<uword, bool, 0, 1> {};
  class SafepointRequestedField : public BitField<uword, bool, 1, 1> {};
  class BlockedForSafepointField : public BitField<uword, bool, 2, 1> {};
  class BypassSafepointsField : public BitField<uword, bool, 3, 1> {};

#if defined(USING_SAFE_STACK)
  uword saved_safestack_limit_;
#endif

#if !defined(DART_PRECOMPILED_RUNTIME)
  Interpreter* interpreter_;
#endif

  Thread* next_;  // Used to chain the thread structures in an isolate.
  bool is_mutator_thread_ = false;

  explicit Thread(bool is_vm_isolate);

  void StoreBufferRelease(
      StoreBuffer::ThresholdPolicy policy = StoreBuffer::kCheckThreshold);
  void StoreBufferAcquire();

  void MarkingStackRelease();
  void MarkingStackAcquire();
  void DeferredMarkingStackRelease();
  void DeferredMarkingStackAcquire();

  void set_safepoint_state(uint32_t value) { safepoint_state_ = value; }
  void EnterSafepointUsingLock();
  void ExitSafepointUsingLock();
  void BlockForSafepoint();

  void FinishEntering(TaskKind kind);
  void PrepareLeaving();

  static void SetCurrent(Thread* current) { OSThread::SetCurrentTLS(current); }

  void DeferOOBMessageInterrupts();
  void RestoreOOBMessageInterrupts();

#define REUSABLE_FRIEND_DECLARATION(name)                                      \
  friend class Reusable##name##HandleScope;
  REUSABLE_HANDLE_LIST(REUSABLE_FRIEND_DECLARATION)
#undef REUSABLE_FRIEND_DECLARATION

  friend class ApiZone;
  friend class Interpreter;
  friend class InterruptChecker;
  friend class Isolate;
  friend class IsolateGroup;
  friend class IsolateTestHelper;
  friend class NoOOBMessageScope;
  friend class Simulator;
  friend class StackZone;
  friend class ThreadRegistry;
  friend class NoActiveIsolateScope;
  friend class CompilerState;
  friend class compiler::target::Thread;
  friend class FieldTable;
  friend Isolate* CreateWithinExistingIsolateGroup(IsolateGroup*,
                                                   const char*,
                                                   char**);
  DISALLOW_COPY_AND_ASSIGN(Thread);
};

#if defined(HOST_OS_WINDOWS)
// Clears the state of the current thread and frees the allocation.
void WindowsThreadCleanUp();
#endif

// Disable thread interrupts.
class DisableThreadInterruptsScope : public StackResource {
 public:
  explicit DisableThreadInterruptsScope(Thread* thread);
  ~DisableThreadInterruptsScope();
};

// Within a NoSafepointScope, the thread must not reach any safepoint. Used
// around code that manipulates raw object pointers directly without handles.
#if defined(DEBUG)
class NoSafepointScope : public ThreadStackResource {
 public:
  explicit NoSafepointScope(Thread* thread = nullptr)
      : ThreadStackResource(thread != nullptr ? thread : Thread::Current()) {
    this->thread()->IncrementNoSafepointScopeDepth();
  }
  ~NoSafepointScope() { thread()->DecrementNoSafepointScopeDepth(); }

 private:
  DISALLOW_COPY_AND_ASSIGN(NoSafepointScope);
};
#else   // defined(DEBUG)
class NoSafepointScope : public ValueObject {
 public:
  explicit NoSafepointScope(Thread* thread = nullptr) {}

 private:
  DISALLOW_COPY_AND_ASSIGN(NoSafepointScope);
};
#endif  // defined(DEBUG)

}  // namespace dart

#endif  // RUNTIME_VM_THREAD_H_

```

 

## thread.cc

```c++
// Copyright (c) 2015, the Dart project authors.  Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

#include "vm/thread.h"

#include "vm/dart_api_state.h"
#include "vm/growable_array.h"
#include "vm/heap/safepoint.h"
#include "vm/isolate.h"
#include "vm/json_stream.h"
#include "vm/lockers.h"
#include "vm/log.h"
#include "vm/message_handler.h"
#include "vm/native_entry.h"
#include "vm/object.h"
#include "vm/os_thread.h"
#include "vm/profiler.h"
#include "vm/runtime_entry.h"
#include "vm/stub_code.h"
#include "vm/symbols.h"
#include "vm/thread_interrupter.h"
#include "vm/thread_registry.h"
#include "vm/timeline.h"
#include "vm/zone.h"

#if !defined(DART_PRECOMPILED_RUNTIME)
#include "vm/ffi_callback_trampolines.h"
#endif  // !defined(DART_PRECOMPILED_RUNTIME)

namespace dart {

#if !defined(PRODUCT)
DECLARE_FLAG(bool, trace_service);
DECLARE_FLAG(bool, trace_service_verbose);
#endif  // !defined(PRODUCT)

Thread::~Thread() {
  // We should cleanly exit any isolate before destruction.
  ASSERT(isolate_ == NULL);
  ASSERT(store_buffer_block_ == NULL);
  ASSERT(marking_stack_block_ == NULL);
#if !defined(DART_PRECOMPILED_RUNTIME)
  delete interpreter_;
  interpreter_ = nullptr;
#endif
  // There should be no top api scopes at this point.
  ASSERT(api_top_scope() == NULL);
  // Delete the resusable api scope if there is one.
  if (api_reusable_scope_ != nullptr) {
    delete api_reusable_scope_;
    api_reusable_scope_ = NULL;
  }
}

#if defined(DEBUG)
#define REUSABLE_HANDLE_SCOPE_INIT(object)                                     \
  reusable_##object##_handle_scope_active_(false),
#else
#define REUSABLE_HANDLE_SCOPE_INIT(object)
#endif  // defined(DEBUG)

#define REUSABLE_HANDLE_INITIALIZERS(object) object##_handle_(NULL),

Thread::Thread(bool is_vm_isolate)
    : ThreadState(false),
      stack_limit_(0),
      write_barrier_mask_(ObjectLayout::kGenerationalBarrierMask),
      isolate_(NULL),
      dispatch_table_array_(NULL),
      saved_stack_limit_(0),
      stack_overflow_flags_(0),
      heap_(NULL),
      top_exit_frame_info_(0),
      store_buffer_block_(NULL),
      marking_stack_block_(NULL),
      vm_tag_(0),
      async_stack_trace_(StackTrace::null()),
      unboxed_int64_runtime_arg_(0),
      active_exception_(Object::null()),
      active_stacktrace_(Object::null()),
      global_object_pool_(ObjectPool::null()),
      resume_pc_(0),
      execution_state_(kThreadInNative),
      safepoint_state_(0),
      ffi_callback_code_(GrowableObjectArray::null()),
      api_top_scope_(NULL),
      task_kind_(kUnknownTask),
      dart_stream_(NULL),
      thread_lock_(),
      api_reusable_scope_(NULL),
      no_callback_scope_depth_(0),
#if defined(DEBUG)
      no_safepoint_scope_depth_(0),
#endif
      reusable_handles_(),
      defer_oob_messages_count_(0),
      deferred_interrupts_mask_(0),
      deferred_interrupts_(0),
      stack_overflow_count_(0),
      hierarchy_info_(NULL),
      type_usage_info_(NULL),
      pending_functions_(GrowableObjectArray::null()),
      sticky_error_(Error::null()),
      REUSABLE_HANDLE_LIST(REUSABLE_HANDLE_INITIALIZERS)
          REUSABLE_HANDLE_LIST(REUSABLE_HANDLE_SCOPE_INIT)
#if defined(USING_SAFE_STACK)
              saved_safestack_limit_(0),
#endif
#if !defined(DART_PRECOMPILED_RUNTIME)
      interpreter_(nullptr),
#endif
      next_(NULL) {
#if defined(SUPPORT_TIMELINE)
  dart_stream_ = Timeline::GetDartStream();
  ASSERT(dart_stream_ != NULL);
#endif
#define DEFAULT_INIT(type_name, member_name, init_expr, default_init_value)    \
  member_name = default_init_value;
  CACHED_CONSTANTS_LIST(DEFAULT_INIT)
#undef DEFAULT_INIT

#if defined(TARGET_ARCH_ARM) || defined(TARGET_ARCH_ARM64) ||                  \
    defined(TARGET_ARCH_X64)
  for (intptr_t i = 0; i < kNumberOfDartAvailableCpuRegs; ++i) {
    write_barrier_wrappers_entry_points_[i] = 0;
  }
#endif

#define DEFAULT_INIT(name) name##_entry_point_ = 0;
  RUNTIME_ENTRY_LIST(DEFAULT_INIT)
#undef DEFAULT_INIT

#define DEFAULT_INIT(returntype, name, ...) name##_entry_point_ = 0;
  LEAF_RUNTIME_ENTRY_LIST(DEFAULT_INIT)
#undef DEFAULT_INIT

  // We cannot initialize the VM constants here for the vm isolate thread
  // due to boot strapping issues.
  if (!is_vm_isolate) {
    InitVMConstants();
  }
}

static const double double_nan_constant = NAN;

static const struct ALIGN16 {
  uint64_t a;
  uint64_t b;
} double_negate_constant = {0x8000000000000000ULL, 0x8000000000000000ULL};

static const struct ALIGN16 {
  uint64_t a;
  uint64_t b;
} double_abs_constant = {0x7FFFFFFFFFFFFFFFULL, 0x7FFFFFFFFFFFFFFFULL};

static const struct ALIGN16 {
  uint32_t a;
  uint32_t b;
  uint32_t c;
  uint32_t d;
} float_not_constant = {0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF};

static const struct ALIGN16 {
  uint32_t a;
  uint32_t b;
  uint32_t c;
  uint32_t d;
} float_negate_constant = {0x80000000, 0x80000000, 0x80000000, 0x80000000};

static const struct ALIGN16 {
  uint32_t a;
  uint32_t b;
  uint32_t c;
  uint32_t d;
} float_absolute_constant = {0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF, 0x7FFFFFFF};

static const struct ALIGN16 {
  uint32_t a;
  uint32_t b;
  uint32_t c;
  uint32_t d;
} float_zerow_constant = {0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF, 0x00000000};

void Thread::InitVMConstants() {
#define ASSERT_VM_HEAP(type_name, member_name, init_expr, default_init_value)  \
  ASSERT((init_expr)->IsOldObject());
  CACHED_VM_OBJECTS_LIST(ASSERT_VM_HEAP)
#undef ASSERT_VM_HEAP

#define INIT_VALUE(type_name, member_name, init_expr, default_init_value)      \
  ASSERT(member_name == default_init_value);                                   \
  member_name = (init_expr);
  CACHED_CONSTANTS_LIST(INIT_VALUE)
#undef INIT_VALUE

#if defined(TARGET_ARCH_ARM) || defined(TARGET_ARCH_ARM64) ||                  \
    defined(TARGET_ARCH_X64)
  for (intptr_t i = 0; i < kNumberOfDartAvailableCpuRegs; ++i) {
    write_barrier_wrappers_entry_points_[i] =
        StubCode::WriteBarrierWrappers().EntryPoint() +
        i * kStoreBufferWrapperSize;
  }
#endif

#define INIT_VALUE(name)                                                       \
  ASSERT(name##_entry_point_ == 0);                                            \
  name##_entry_point_ = k##name##RuntimeEntry.GetEntryPoint();
  RUNTIME_ENTRY_LIST(INIT_VALUE)
#undef INIT_VALUE

#define INIT_VALUE(returntype, name, ...)                                      \
  ASSERT(name##_entry_point_ == 0);                                            \
  name##_entry_point_ = k##name##RuntimeEntry.GetEntryPoint();
  LEAF_RUNTIME_ENTRY_LIST(INIT_VALUE)
#undef INIT_VALUE

// Setup the thread specific reusable handles.
#define REUSABLE_HANDLE_ALLOCATION(object)                                     \
  this->object##_handle_ = this->AllocateReusableHandle<object>();
  REUSABLE_HANDLE_LIST(REUSABLE_HANDLE_ALLOCATION)
#undef REUSABLE_HANDLE_ALLOCATION
}

#ifndef PRODUCT
// Collect information about each individual zone associated with this thread.
void Thread::PrintJSON(JSONStream* stream) const {
  JSONObject jsobj(stream);
  jsobj.AddProperty("type", "_Thread");
  jsobj.AddPropertyF("id", "threads/%" Pd "",
                     OSThread::ThreadIdToIntPtr(os_thread()->trace_id()));
  jsobj.AddProperty("kind", TaskKindToCString(task_kind()));
  jsobj.AddPropertyF("_zoneHighWatermark", "%" Pu "", zone_high_watermark());
  jsobj.AddPropertyF("_zoneCapacity", "%" Pu "", current_zone_capacity());
}
#endif

GrowableObjectArrayPtr Thread::pending_functions() {
  if (pending_functions_ == GrowableObjectArray::null()) {
    pending_functions_ = GrowableObjectArray::New(Heap::kOld);
  }
  return pending_functions_;
}

void Thread::clear_pending_functions() {
  pending_functions_ = GrowableObjectArray::null();
}

void Thread::set_active_exception(const Object& value) {
  active_exception_ = value.raw();
}

void Thread::set_active_stacktrace(const Object& value) {
  active_stacktrace_ = value.raw();
}

ErrorPtr Thread::sticky_error() const {
  return sticky_error_;
}

void Thread::set_sticky_error(const Error& value) {
  ASSERT(!value.IsNull());
  sticky_error_ = value.raw();
}

void Thread::ClearStickyError() {
  sticky_error_ = Error::null();
}

ErrorPtr Thread::StealStickyError() {
  NoSafepointScope no_safepoint;
  ErrorPtr return_value = sticky_error_;
  sticky_error_ = Error::null();
  return return_value;
}

const char* Thread::TaskKindToCString(TaskKind kind) {
  switch (kind) {
    case kUnknownTask:
      return "kUnknownTask";
    case kMutatorTask:
      return "kMutatorTask";
    case kCompilerTask:
      return "kCompilerTask";
    case kSweeperTask:
      return "kSweeperTask";
    case kMarkerTask:
      return "kMarkerTask";
    default:
      UNREACHABLE();
      return "";
  }
}

StackTracePtr Thread::async_stack_trace() const {
  return async_stack_trace_;
}

void Thread::set_async_stack_trace(const StackTrace& stack_trace) {
  ASSERT(!stack_trace.IsNull());
  async_stack_trace_ = stack_trace.raw();
}

void Thread::set_raw_async_stack_trace(StackTracePtr raw_stack_trace) {
  async_stack_trace_ = raw_stack_trace;
}

void Thread::clear_async_stack_trace() {
  async_stack_trace_ = StackTrace::null();
}

bool Thread::EnterIsolate(Isolate* isolate) {
  const bool kIsMutatorThread = true;
  Thread* thread = isolate->ScheduleThread(kIsMutatorThread);
  if (thread != NULL) {
    ASSERT(thread->store_buffer_block_ == NULL);
    ASSERT(thread->isolate() == isolate);
    ASSERT(thread->isolate_group() == isolate->group());
    thread->FinishEntering(kMutatorTask);
    return true;
  }
  return false;
}

void Thread::ExitIsolate() {
  Thread* thread = Thread::Current();
  ASSERT(thread != nullptr);
  ASSERT(thread->IsMutatorThread());
  ASSERT(thread->isolate() != nullptr);
  ASSERT(thread->isolate_group() != nullptr);
  DEBUG_ASSERT(!thread->IsAnyReusableHandleScopeActive());

  thread->PrepareLeaving();

  Isolate* isolate = thread->isolate();
  thread->set_vm_tag(isolate->is_runnable() ? VMTag::kIdleTagId
                                            : VMTag::kLoadWaitTagId);
  const bool kIsMutatorThread = true;
  isolate->UnscheduleThread(thread, kIsMutatorThread);
}

bool Thread::EnterIsolateAsHelper(Isolate* isolate,
                                  TaskKind kind,
                                  bool bypass_safepoint) {
  ASSERT(kind != kMutatorTask);
  const bool kIsMutatorThread = false;
  Thread* thread = isolate->ScheduleThread(kIsMutatorThread, bypass_safepoint);
  if (thread != NULL) {
    ASSERT(!thread->IsMutatorThread());
    ASSERT(thread->isolate() == isolate);
    ASSERT(thread->isolate_group() == isolate->group());
    thread->FinishEntering(kind);
    return true;
  }
  return false;
}

void Thread::ExitIsolateAsHelper(bool bypass_safepoint) {
  Thread* thread = Thread::Current();
  ASSERT(thread != nullptr);
  ASSERT(!thread->IsMutatorThread());
  ASSERT(thread->isolate() != nullptr);
  ASSERT(thread->isolate_group() != nullptr);

  thread->PrepareLeaving();

  Isolate* isolate = thread->isolate();
  ASSERT(isolate != NULL);
  const bool kIsMutatorThread = false;
  isolate->UnscheduleThread(thread, kIsMutatorThread, bypass_safepoint);
}

bool Thread::EnterIsolateGroupAsHelper(IsolateGroup* isolate_group,
                                       TaskKind kind,
                                       bool bypass_safepoint) {
  ASSERT(kind != kMutatorTask);
  Thread* thread = isolate_group->ScheduleThread(bypass_safepoint);
  if (thread != NULL) {
    ASSERT(!thread->IsMutatorThread());
    ASSERT(thread->isolate() == nullptr);
    ASSERT(thread->isolate_group() == isolate_group);
    thread->FinishEntering(kind);
    return true;
  }
  return false;
}

void Thread::ExitIsolateGroupAsHelper(bool bypass_safepoint) {
  Thread* thread = Thread::Current();
  ASSERT(thread != nullptr);
  ASSERT(!thread->IsMutatorThread());
  ASSERT(thread->isolate() == nullptr);
  ASSERT(thread->isolate_group() != nullptr);

  thread->PrepareLeaving();

  const bool kIsMutatorThread = false;
  thread->isolate_group()->UnscheduleThread(thread, kIsMutatorThread,
                                            bypass_safepoint);
}

void Thread::ReleaseStoreBuffer() {
  ASSERT(IsAtSafepoint());
  // Prevent scheduling another GC by ignoring the threshold.
  ASSERT(store_buffer_block_ != NULL);
  StoreBufferRelease(StoreBuffer::kIgnoreThreshold);
  // Make sure to get an *empty* block; the isolate needs all entries
  // at GC time.
  // TODO(koda): Replace with an epilogue (PrepareAfterGC) that acquires.
  store_buffer_block_ = isolate_group()->store_buffer()->PopEmptyBlock();
}

void Thread::SetStackLimit(uword limit) {
  // The thread setting the stack limit is not necessarily the thread which
  // the stack limit is being set on.
  MonitorLocker ml(&thread_lock_);
  if (!HasScheduledInterrupts()) {
    // No interrupt pending, set stack_limit_ too.
    stack_limit_ = limit;
  }
  saved_stack_limit_ = limit;
}

void Thread::ClearStackLimit() {
  SetStackLimit(~static_cast<uword>(0));
}

void Thread::ScheduleInterrupts(uword interrupt_bits) {
  MonitorLocker ml(&thread_lock_);
  ScheduleInterruptsLocked(interrupt_bits);
}

void Thread::ScheduleInterruptsLocked(uword interrupt_bits) {
  ASSERT(thread_lock_.IsOwnedByCurrentThread());
  ASSERT((interrupt_bits & ~kInterruptsMask) == 0);  // Must fit in mask.

  // Check to see if any of the requested interrupts should be deferred.
  uword defer_bits = interrupt_bits & deferred_interrupts_mask_;
  if (defer_bits != 0) {
    deferred_interrupts_ |= defer_bits;
    interrupt_bits &= ~deferred_interrupts_mask_;
    if (interrupt_bits == 0) {
      return;
    }
  }

  if (stack_limit_ == saved_stack_limit_) {
    stack_limit_ = (kInterruptStackLimit & ~kInterruptsMask) | interrupt_bits;
  } else {
    stack_limit_ = stack_limit_ | interrupt_bits;
  }
}

uword Thread::GetAndClearInterrupts() {
  MonitorLocker ml(&thread_lock_);
  if (stack_limit_ == saved_stack_limit_) {
    return 0;  // No interrupt was requested.
  }
  uword interrupt_bits = stack_limit_ & kInterruptsMask;
  stack_limit_ = saved_stack_limit_;
  return interrupt_bits;
}

void Thread::DeferOOBMessageInterrupts() {
  MonitorLocker ml(&thread_lock_);
  defer_oob_messages_count_++;
  if (defer_oob_messages_count_ > 1) {
    // OOB message interrupts are already deferred.
    return;
  }
  ASSERT(deferred_interrupts_mask_ == 0);
  deferred_interrupts_mask_ = kMessageInterrupt;

  if (stack_limit_ != saved_stack_limit_) {
    // Defer any interrupts which are currently pending.
    deferred_interrupts_ = stack_limit_ & deferred_interrupts_mask_;

    // Clear deferrable interrupts, if present.
    stack_limit_ = stack_limit_ & ~deferred_interrupts_mask_;

    if ((stack_limit_ & kInterruptsMask) == 0) {
      // No other pending interrupts.  Restore normal stack limit.
      stack_limit_ = saved_stack_limit_;
    }
  }
#if !defined(PRODUCT)
  if (FLAG_trace_service && FLAG_trace_service_verbose) {
    OS::PrintErr("[+%" Pd64 "ms] Isolate %s deferring OOB interrupts\n",
                 Dart::UptimeMillis(), isolate()->name());
  }
#endif  // !defined(PRODUCT)
}

void Thread::RestoreOOBMessageInterrupts() {
  MonitorLocker ml(&thread_lock_);
  defer_oob_messages_count_--;
  if (defer_oob_messages_count_ > 0) {
    return;
  }
  ASSERT(defer_oob_messages_count_ == 0);
  ASSERT(deferred_interrupts_mask_ == kMessageInterrupt);
  deferred_interrupts_mask_ = 0;
  if (deferred_interrupts_ != 0) {
    if (stack_limit_ == saved_stack_limit_) {
      stack_limit_ = kInterruptStackLimit & ~kInterruptsMask;
    }
    stack_limit_ = stack_limit_ | deferred_interrupts_;
    deferred_interrupts_ = 0;
  }
#if !defined(PRODUCT)
  if (FLAG_trace_service && FLAG_trace_service_verbose) {
    OS::PrintErr("[+%" Pd64 "ms] Isolate %s restoring OOB interrupts\n",
                 Dart::UptimeMillis(), isolate()->name());
  }
#endif  // !defined(PRODUCT)
}

ErrorPtr Thread::HandleInterrupts() {
  uword interrupt_bits = GetAndClearInterrupts();
  if ((interrupt_bits & kVMInterrupt) != 0) {
    CheckForSafepoint();
    if (isolate_group()->store_buffer()->Overflowed()) {
      if (FLAG_verbose_gc) {
        OS::PrintErr("Scavenge scheduled by store buffer overflow.\n");
      }
      heap()->CollectGarbage(Heap::kNew);
    }
  }
  if ((interrupt_bits & kMessageInterrupt) != 0) {
    MessageHandler::MessageStatus status =
        isolate()->message_handler()->HandleOOBMessages();
    if (status != MessageHandler::kOK) {
      // False result from HandleOOBMessages signals that the isolate should
      // be terminating.
      if (FLAG_trace_isolates) {
        OS::PrintErr(
            "[!] Terminating isolate due to OOB message:\n"
            "\tisolate:    %s\n",
            isolate()->name());
      }
      NoSafepointScope no_safepoint;
      ErrorPtr error = Thread::Current()->StealStickyError();
      ASSERT(error->IsUnwindError());
      return error;
    }
  }
  return Error::null();
}

uword Thread::GetAndClearStackOverflowFlags() {
  uword stack_overflow_flags = stack_overflow_flags_;
  stack_overflow_flags_ = 0;
  return stack_overflow_flags;
}

void Thread::StoreBufferBlockProcess(StoreBuffer::ThresholdPolicy policy) {
  StoreBufferRelease(policy);
  StoreBufferAcquire();
}

void Thread::StoreBufferAddObject(ObjectPtr obj) {
  ASSERT(this == Thread::Current());
  store_buffer_block_->Push(obj);
  if (store_buffer_block_->IsFull()) {
    StoreBufferBlockProcess(StoreBuffer::kCheckThreshold);
  }
}

void Thread::StoreBufferAddObjectGC(ObjectPtr obj) {
  store_buffer_block_->Push(obj);
  if (store_buffer_block_->IsFull()) {
    StoreBufferBlockProcess(StoreBuffer::kIgnoreThreshold);
  }
}

void Thread::StoreBufferRelease(StoreBuffer::ThresholdPolicy policy) {
  StoreBufferBlock* block = store_buffer_block_;
  store_buffer_block_ = NULL;
  isolate_group()->store_buffer()->PushBlock(block, policy);
}

void Thread::StoreBufferAcquire() {
  store_buffer_block_ = isolate_group()->store_buffer()->PopNonFullBlock();
}

void Thread::MarkingStackBlockProcess() {
  MarkingStackRelease();
  MarkingStackAcquire();
}

void Thread::DeferredMarkingStackBlockProcess() {
  DeferredMarkingStackRelease();
  DeferredMarkingStackAcquire();
}

void Thread::MarkingStackAddObject(ObjectPtr obj) {
  marking_stack_block_->Push(obj);
  if (marking_stack_block_->IsFull()) {
    MarkingStackBlockProcess();
  }
}

void Thread::DeferredMarkingStackAddObject(ObjectPtr obj) {
  deferred_marking_stack_block_->Push(obj);
  if (deferred_marking_stack_block_->IsFull()) {
    DeferredMarkingStackBlockProcess();
  }
}

void Thread::MarkingStackRelease() {
  MarkingStackBlock* block = marking_stack_block_;
  marking_stack_block_ = NULL;
  write_barrier_mask_ = ObjectLayout::kGenerationalBarrierMask;
  isolate_group()->marking_stack()->PushBlock(block);
}

void Thread::MarkingStackAcquire() {
  marking_stack_block_ = isolate_group()->marking_stack()->PopEmptyBlock();
  write_barrier_mask_ = ObjectLayout::kGenerationalBarrierMask |
                        ObjectLayout::kIncrementalBarrierMask;
}

void Thread::DeferredMarkingStackRelease() {
  MarkingStackBlock* block = deferred_marking_stack_block_;
  deferred_marking_stack_block_ = NULL;
  isolate_group()->deferred_marking_stack()->PushBlock(block);
}

void Thread::DeferredMarkingStackAcquire() {
  deferred_marking_stack_block_ =
      isolate_group()->deferred_marking_stack()->PopEmptyBlock();
}

bool Thread::CanCollectGarbage() const {
  // We grow the heap instead of triggering a garbage collection when a
  // thread is at a safepoint in the following situations :
  //   - background compiler thread finalizing and installing code
  //   - disassembly of the generated code is done after compilation
  // So essentially we state that garbage collection is possible only
  // when we are not at a safepoint.
  return !IsAtSafepoint();
}

bool Thread::IsExecutingDartCode() const {
  return (top_exit_frame_info() == 0) && VMTag::IsDartTag(vm_tag());
}

bool Thread::HasExitedDartCode() const {
  return (top_exit_frame_info() != 0) && !VMTag::IsDartTag(vm_tag());
}

template <class C>
C* Thread::AllocateReusableHandle() {
  C* handle = reinterpret_cast<C*>(reusable_handles_.AllocateScopedHandle());
  C::initializeHandle(handle, C::null());
  return handle;
}

void Thread::ClearReusableHandles() {
#define CLEAR_REUSABLE_HANDLE(object) *object##_handle_ = object::null();
  REUSABLE_HANDLE_LIST(CLEAR_REUSABLE_HANDLE)
#undef CLEAR_REUSABLE_HANDLE
}

void Thread::VisitObjectPointers(ObjectPointerVisitor* visitor,
                                 ValidationPolicy validation_policy) {
  ASSERT(visitor != NULL);

  if (zone() != NULL) {
    zone()->VisitObjectPointers(visitor);
  }

  // Visit objects in thread specific handles area.
  reusable_handles_.VisitObjectPointers(visitor);

  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&pending_functions_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&global_object_pool_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&active_exception_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&active_stacktrace_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&sticky_error_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&async_stack_trace_));
  visitor->VisitPointer(reinterpret_cast<ObjectPtr*>(&ffi_callback_code_));

#if !defined(DART_PRECOMPILED_RUNTIME)
  if (interpreter() != NULL) {
    interpreter()->VisitObjectPointers(visitor);
  }
#endif

  // Visit the api local scope as it has all the api local handles.
  ApiLocalScope* scope = api_top_scope_;
  while (scope != NULL) {
    scope->local_handles()->VisitObjectPointers(visitor);
    scope = scope->previous();
  }

  // Only the mutator thread can run Dart code.
  if (IsMutatorThread()) {
    // The MarkTask, which calls this method, can run on a different thread.  We
    // therefore assume the mutator is at a safepoint and we can iterate its
    // stack.
    // TODO(vm-team): It would be beneficial to be able to ask the mutator
    // thread whether it is in fact blocked at the moment (at a "safepoint") so
    // we can safely iterate its stack.
    //
    // Unfortunately we cannot use `this->IsAtSafepoint()` here because that
    // will return `false` even though the mutator thread is waiting for mark
    // tasks (which iterate its stack) to finish.
    const StackFrameIterator::CrossThreadPolicy cross_thread_policy =
        StackFrameIterator::kAllowCrossThreadIteration;

    // Iterate over all the stack frames and visit objects on the stack.
    StackFrameIterator frames_iterator(top_exit_frame_info(), validation_policy,
                                       this, cross_thread_policy);
    StackFrame* frame = frames_iterator.NextFrame();
    while (frame != NULL) {
      frame->VisitObjectPointers(visitor);
      frame = frames_iterator.NextFrame();
    }
  } else {
    // We are not on the mutator thread.
    RELEASE_ASSERT(top_exit_frame_info() == 0);
  }
}

class RestoreWriteBarrierInvariantVisitor : public ObjectPointerVisitor {
 public:
  RestoreWriteBarrierInvariantVisitor(IsolateGroup* group,
                                      Thread* thread,
                                      Thread::RestoreWriteBarrierInvariantOp op)
      : ObjectPointerVisitor(group),
        thread_(thread),
        current_(Thread::Current()),
        op_(op) {}

  void VisitPointers(ObjectPtr* first, ObjectPtr* last) {
    for (; first != last + 1; first++) {
      ObjectPtr obj = *first;
      // Stores into new-space objects don't need a write barrier.
      if (obj->IsSmiOrNewObject()) continue;

      // To avoid adding too much work into the remembered set, skip
      // arrays. Write barrier elimination will not remove the barrier
      // if we can trigger GC between array allocation and store.
      if (obj->GetClassId() == kArrayCid) continue;

      // Dart code won't store into VM-internal objects except Contexts and
      // UnhandledExceptions. This assumption is checked by an assertion in
      // WriteBarrierElimination::UpdateVectorForBlock.
      if (!obj->IsDartInstance() && !obj->IsContext() &&
          !obj->IsUnhandledException())
        continue;

      // Dart code won't store into canonical instances.
      if (obj->ptr()->IsCanonical()) continue;

      // Objects in the VM isolate heap are immutable and won't be
      // stored into. Check this condition last because there's no bit
      // in the header for it.
      if (obj->ptr()->InVMIsolateHeap()) continue;

      switch (op_) {
        case Thread::RestoreWriteBarrierInvariantOp::kAddToRememberedSet:
          if (!obj->ptr()->IsRemembered()) {
            obj->ptr()->AddToRememberedSet(current_);
          }
          if (current_->is_marking()) {
            current_->DeferredMarkingStackAddObject(obj);
          }
          break;
        case Thread::RestoreWriteBarrierInvariantOp::kAddToDeferredMarkingStack:
          // Re-scan obj when finalizing marking.
          current_->DeferredMarkingStackAddObject(obj);
          break;
      }
    }
  }

 private:
  Thread* const thread_;
  Thread* const current_;
  Thread::RestoreWriteBarrierInvariantOp op_;
};

// Write barrier elimination assumes that all live temporaries will be
// in the remembered set after a scavenge triggered by a non-Dart-call
// instruction (see Instruction::CanCallDart()), and additionally they will be
// in the deferred marking stack if concurrent marking started. Specifically,
// this includes any instruction which will always create an exit frame
// below the current frame before any other Dart frames.
//
// Therefore, to support this assumption, we scan the stack after a scavenge
// or when concurrent marking begins and add all live temporaries in
// Dart frames preceeding an exit frame to the store buffer or deferred
// marking stack.
void Thread::RestoreWriteBarrierInvariant(RestoreWriteBarrierInvariantOp op) {
  ASSERT(IsAtSafepoint());
  ASSERT(IsMutatorThread());

  const StackFrameIterator::CrossThreadPolicy cross_thread_policy =
      StackFrameIterator::kAllowCrossThreadIteration;
  StackFrameIterator frames_iterator(top_exit_frame_info(),
                                     ValidationPolicy::kDontValidateFrames,
                                     this, cross_thread_policy);
  RestoreWriteBarrierInvariantVisitor visitor(isolate_group(), this, op);
  bool scan_next_dart_frame = false;
  for (StackFrame* frame = frames_iterator.NextFrame(); frame != NULL;
       frame = frames_iterator.NextFrame()) {
    if (frame->IsExitFrame()) {
      scan_next_dart_frame = true;
    } else if (frame->IsDartFrame(/*validate=*/false)) {
      if (scan_next_dart_frame) {
        frame->VisitObjectPointers(&visitor);
      }
      scan_next_dart_frame = false;
    }
  }
}

void Thread::DeferredMarkLiveTemporaries() {
  RestoreWriteBarrierInvariant(
      RestoreWriteBarrierInvariantOp::kAddToDeferredMarkingStack);
}

void Thread::RememberLiveTemporaries() {
  RestoreWriteBarrierInvariant(
      RestoreWriteBarrierInvariantOp::kAddToRememberedSet);
}

bool Thread::CanLoadFromThread(const Object& object) {
  // In order to allow us to use assembler helper routines with non-[Code]
  // objects *before* stubs are initialized, we only loop ver the stubs if the
  // [object] is in fact a [Code] object.
  if (object.IsCode()) {
#define CHECK_OBJECT(type_name, member_name, expr, default_init_value)         \
  if (object.raw() == expr) {                                                  \
    return true;                                                               \
  }
    CACHED_VM_STUBS_LIST(CHECK_OBJECT)
#undef CHECK_OBJECT
  }

  // For non [Code] objects we check if the object equals to any of the cached
  // non-stub entries.
#define CHECK_OBJECT(type_name, member_name, expr, default_init_value)         \
  if (object.raw() == expr) {                                                  \
    return true;                                                               \
  }
  CACHED_NON_VM_STUB_LIST(CHECK_OBJECT)
#undef CHECK_OBJECT
  return false;
}

intptr_t Thread::OffsetFromThread(const Object& object) {
  // In order to allow us to use assembler helper routines with non-[Code]
  // objects *before* stubs are initialized, we only loop ver the stubs if the
  // [object] is in fact a [Code] object.
  if (object.IsCode()) {
#define COMPUTE_OFFSET(type_name, member_name, expr, default_init_value)       \
  ASSERT((expr)->ptr()->InVMIsolateHeap());                                    \
  if (object.raw() == expr) {                                                  \
    return Thread::member_name##offset();                                      \
  }
    CACHED_VM_STUBS_LIST(COMPUTE_OFFSET)
#undef COMPUTE_OFFSET
  }

  // For non [Code] objects we check if the object equals to any of the cached
  // non-stub entries.
#define COMPUTE_OFFSET(type_name, member_name, expr, default_init_value)       \
  if (object.raw() == expr) {                                                  \
    return Thread::member_name##offset();                                      \
  }
  CACHED_NON_VM_STUB_LIST(COMPUTE_OFFSET)
#undef COMPUTE_OFFSET

  UNREACHABLE();
  return -1;
}

bool Thread::ObjectAtOffset(intptr_t offset, Object* object) {
  if (Isolate::Current() == Dart::vm_isolate()) {
    // --disassemble-stubs runs before all the references through
    // thread have targets
    return false;
  }

#define COMPUTE_OFFSET(type_name, member_name, expr, default_init_value)       \
  if (Thread::member_name##offset() == offset) {                               \
    *object = expr;                                                            \
    return true;                                                               \
  }
  CACHED_VM_OBJECTS_LIST(COMPUTE_OFFSET)
#undef COMPUTE_OFFSET
  return false;
}

intptr_t Thread::OffsetFromThread(const RuntimeEntry* runtime_entry) {
#define COMPUTE_OFFSET(name)                                                   \
  if (runtime_entry->function() == k##name##RuntimeEntry.function()) {         \
    return Thread::name##_entry_point_offset();                                \
  }
  RUNTIME_ENTRY_LIST(COMPUTE_OFFSET)
#undef COMPUTE_OFFSET

#define COMPUTE_OFFSET(returntype, name, ...)                                  \
  if (runtime_entry->function() == k##name##RuntimeEntry.function()) {         \
    return Thread::name##_entry_point_offset();                                \
  }
  LEAF_RUNTIME_ENTRY_LIST(COMPUTE_OFFSET)
#undef COMPUTE_OFFSET

  UNREACHABLE();
  return -1;
}

#if defined(DEBUG)
bool Thread::TopErrorHandlerIsSetJump() const {
  if (long_jump_base() == nullptr) return false;
  if (top_exit_frame_info_ == 0) return true;
#if defined(USING_SIMULATOR) || defined(USING_SAFE_STACK)
  // False positives: simulator stack and native stack are unordered.
  return true;
#else
#if !defined(DART_PRECOMPILED_RUNTIME)
  // False positives: interpreter stack and native stack are unordered.
  if ((interpreter_ != nullptr) && interpreter_->HasFrame(top_exit_frame_info_))
    return true;
#endif
  return reinterpret_cast<uword>(long_jump_base()) < top_exit_frame_info_;
#endif
}

bool Thread::TopErrorHandlerIsExitFrame() const {
  if (top_exit_frame_info_ == 0) return false;
  if (long_jump_base() == nullptr) return true;
#if defined(USING_SIMULATOR) || defined(USING_SAFE_STACK)
  // False positives: simulator stack and native stack are unordered.
  return true;
#else
#if !defined(DART_PRECOMPILED_RUNTIME)
  // False positives: interpreter stack and native stack are unordered.
  if ((interpreter_ != nullptr) && interpreter_->HasFrame(top_exit_frame_info_))
    return true;
#endif
  return top_exit_frame_info_ < reinterpret_cast<uword>(long_jump_base());
#endif
}
#endif  // defined(DEBUG)

bool Thread::IsValidHandle(Dart_Handle object) const {
  return IsValidLocalHandle(object) || IsValidZoneHandle(object) ||
         IsValidScopedHandle(object);
}

bool Thread::IsValidLocalHandle(Dart_Handle object) const {
  ApiLocalScope* scope = api_top_scope_;
  while (scope != NULL) {
    if (scope->local_handles()->IsValidHandle(object)) {
      return true;
    }
    scope = scope->previous();
  }
  return false;
}

intptr_t Thread::CountLocalHandles() const {
  intptr_t total = 0;
  ApiLocalScope* scope = api_top_scope_;
  while (scope != NULL) {
    total += scope->local_handles()->CountHandles();
    scope = scope->previous();
  }
  return total;
}

int Thread::ZoneSizeInBytes() const {
  int total = 0;
  ApiLocalScope* scope = api_top_scope_;
  while (scope != NULL) {
    total += scope->zone()->SizeInBytes();
    scope = scope->previous();
  }
  return total;
}

void Thread::EnterApiScope() {
  ASSERT(MayAllocateHandles());
  ApiLocalScope* new_scope = api_reusable_scope();
  if (new_scope == NULL) {
    new_scope = new ApiLocalScope(api_top_scope(), top_exit_frame_info());
    ASSERT(new_scope != NULL);
  } else {
    new_scope->Reinit(this, api_top_scope(), top_exit_frame_info());
    set_api_reusable_scope(NULL);
  }
  set_api_top_scope(new_scope);  // New scope is now the top scope.
}

void Thread::ExitApiScope() {
  ASSERT(MayAllocateHandles());
  ApiLocalScope* scope = api_top_scope();
  ApiLocalScope* reusable_scope = api_reusable_scope();
  set_api_top_scope(scope->previous());  // Reset top scope to previous.
  if (reusable_scope == NULL) {
    scope->Reset(this);  // Reset the old scope which we just exited.
    set_api_reusable_scope(scope);
  } else {
    ASSERT(reusable_scope != scope);
    delete scope;
  }
}

void Thread::UnwindScopes(uword stack_marker) {
  // Unwind all scopes using the same stack_marker, i.e. all scopes allocated
  // under the same top_exit_frame_info.
  ApiLocalScope* scope = api_top_scope_;
  while (scope != NULL && scope->stack_marker() != 0 &&
         scope->stack_marker() == stack_marker) {
    api_top_scope_ = scope->previous();
    delete scope;
    scope = api_top_scope_;
  }
}

void Thread::EnterSafepointUsingLock() {
  isolate_group()->safepoint_handler()->EnterSafepointUsingLock(this);
}

void Thread::ExitSafepointUsingLock() {
  isolate_group()->safepoint_handler()->ExitSafepointUsingLock(this);
}

void Thread::BlockForSafepoint() {
  isolate_group()->safepoint_handler()->BlockForSafepoint(this);
}

void Thread::FinishEntering(TaskKind kind) {
  ASSERT(store_buffer_block_ == nullptr);

  task_kind_ = kind;
  if (isolate_group()->marking_stack() != NULL) {
    // Concurrent mark in progress. Enable barrier for this thread.
    MarkingStackAcquire();
    DeferredMarkingStackAcquire();
  }

  // TODO(koda): Use StoreBufferAcquire once we properly flush
  // before Scavenge.
  if (kind == kMutatorTask) {
    StoreBufferAcquire();
  } else {
    store_buffer_block_ = isolate_group()->store_buffer()->PopEmptyBlock();
  }
}

void Thread::PrepareLeaving() {
  ASSERT(store_buffer_block_ != nullptr);
  ASSERT(execution_state() == Thread::kThreadInVM);

  task_kind_ = kUnknownTask;
  if (is_marking()) {
    MarkingStackRelease();
    DeferredMarkingStackRelease();
  }
  StoreBufferRelease();
}

DisableThreadInterruptsScope::DisableThreadInterruptsScope(Thread* thread)
    : StackResource(thread) {
  if (thread != NULL) {
    OSThread* os_thread = thread->os_thread();
    ASSERT(os_thread != NULL);
    os_thread->DisableThreadInterrupts();
  }
}

DisableThreadInterruptsScope::~DisableThreadInterruptsScope() {
  if (thread() != NULL) {
    OSThread* os_thread = thread()->os_thread();
    ASSERT(os_thread != NULL);
    os_thread->EnableThreadInterrupts();
  }
}

const intptr_t kInitialCallbackIdsReserved = 1024;
int32_t Thread::AllocateFfiCallbackId() {
  Zone* Z = isolate()->current_zone();
  if (ffi_callback_code_ == GrowableObjectArray::null()) {
    ffi_callback_code_ = GrowableObjectArray::New(kInitialCallbackIdsReserved);
  }
  const auto& array = GrowableObjectArray::Handle(Z, ffi_callback_code_);
  array.Add(Code::Handle(Z, Code::null()));
  const int32_t id = array.Length() - 1;

  // Allocate a native callback trampoline if necessary.
#if !defined(DART_PRECOMPILED_RUNTIME)
  if (NativeCallbackTrampolines::Enabled()) {
    auto* const tramps = isolate()->native_callback_trampolines();
    ASSERT(tramps->next_callback_id() == id);
    tramps->AllocateTrampoline();
  }
#endif

  return id;
}

void Thread::SetFfiCallbackCode(int32_t callback_id, const Code& code) {
  Zone* Z = isolate()->current_zone();

  /// In AOT the callback ID might have been allocated during compilation but
  /// 'ffi_callback_code_' is initialized to empty again when the program
  /// starts. Therefore we may need to initialize or expand it to accomodate
  /// the callback ID.

  if (ffi_callback_code_ == GrowableObjectArray::null()) {
    ffi_callback_code_ = GrowableObjectArray::New(kInitialCallbackIdsReserved);
  }

  const auto& array = GrowableObjectArray::Handle(Z, ffi_callback_code_);

  if (callback_id >= array.Length()) {
    if (callback_id >= array.Capacity()) {
      array.Grow(callback_id + 1);
    }
    array.SetLength(callback_id + 1);
  }

  array.SetAt(callback_id, code);
}

void Thread::VerifyCallbackIsolate(int32_t callback_id, uword entry) {
  NoSafepointScope _;

  const GrowableObjectArrayPtr array = ffi_callback_code_;
  if (array == GrowableObjectArray::null()) {
    FATAL("Cannot invoke callback on incorrect isolate.");
  }

  const SmiPtr length_smi = GrowableObjectArray::NoSafepointLength(array);
  const intptr_t length = Smi::Value(length_smi);

  if (callback_id < 0 || callback_id >= length) {
    FATAL("Cannot invoke callback on incorrect isolate.");
  }

  if (entry != 0) {
    ObjectPtr* const code_array =
        Array::DataOf(GrowableObjectArray::NoSafepointData(array));
    // RawCast allocates handles in ASSERTs.
    const CodePtr code = static_cast<CodePtr>(code_array[callback_id]);
    if (!Code::ContainsInstructionAt(code, entry)) {
      FATAL("Cannot invoke callback on incorrect isolate.");
    }
  }
}

}  // namespace dart
```

