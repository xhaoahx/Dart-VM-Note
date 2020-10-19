# ClassId

## 非对象、字符串、数组

```c++
#define CLASS_LIST_NO_OBJECT_NOR_STRING_NOR_ARRAY(V)                           \
  V(Class)                                                                     \
  V(PatchClass)                                                                \
  V(Function)                                                                  \
  V(ClosureData)                                                               \
  V(SignatureData)                                                             \
  V(RedirectionData)                                                           \
  V(FfiTrampolineData)                                                         \
  V(Field)                                                                     \
  V(Script)                                                                    \
  V(Library)                                                                   \
  V(Namespace)                                                                 \
  V(KernelProgramInfo)                                                         \
  V(Code)                                                                      \
  V(Bytecode)                                                                  \
  V(Instructions)                                                              \
  V(InstructionsSection)                                                       \
  V(ObjectPool)                                                                \
  V(PcDescriptors)                                                             \
  V(CodeSourceMap)                                                             \
  V(CompressedStackMaps)                                                       \
  V(LocalVarDescriptors)                                                       \
  V(ExceptionHandlers)                                                         \
  V(Context)                                                                   \
  V(ContextScope)                                                              \
  V(ParameterTypeCheck)                                                        \
  V(SingleTargetCache)                                                         \
  V(UnlinkedCall)                                                              \
  V(MonomorphicSmiableCall)                                                    \
  V(CallSiteData)                                                              \
  V(ICData)                                                                    \
  V(MegamorphicCache)                                                          \
  V(SubtypeTestCache)                                                          \
  V(LoadingUnit)                                                               \
  V(Error)                                                                     \
  V(ApiError)                                                                  \
  V(LanguageError)                                                             \
  V(UnhandledException)                                                        \
  V(UnwindError)                                                               \
  V(Instance)                                                                  \
  V(LibraryPrefix)                                                             \
  V(TypeArguments)                                                             \
  V(AbstractType)                                                              \
  V(Type)                                                                      \
  V(TypeRef)                                                                   \
  V(TypeParameter)                                                             \
  V(Closure)                                                                   \
  V(Number)                                                                    \
  V(Integer)                                                                   \
  V(Smi)                                                                       \
  V(Mint)                                                                      \
  V(Double)                                                                    \
  V(Bool)                                                                      \
  V(GrowableObjectArray)                                                       \
  V(Float32x4)                                                                 \
  V(Int32x4)                                                                   \
  V(Float64x2)                                                                 \
  V(TypedDataBase)                                                             \
  V(TypedData)                                                                 \
  V(ExternalTypedData)                                                         \
  V(TypedDataView)                                                             \
  V(Pointer)                                                                   \
  V(DynamicLibrary)                                                            \
  V(Capability)                                                                \
  V(ReceivePort)                                                               \
  V(SendPort)                                                                  \
  V(StackTrace)                                                                \
  V(RegExp)                                                                    \
  V(WeakProperty)                                                              \
  V(MirrorReference)                                                           \
  V(LinkedHashMap)                                                             \
  V(FutureOr)                                                                  \
  V(UserTag)                                                                   \
  V(TransferableTypedData)                                                     \
  V(WeakSerializationReference)                                                \
  V(ImageHeader)
```

## 数组

```c++
#define CLASS_LIST_ARRAYS(V)                                                   \
  V(Array)                                                                     \
  V(ImmutableArray)
```

## 字符串

```c++
#define CLASS_LIST_STRINGS(V)                                                  \
  V(String)                                                                    \
  V(OneByteString)                                                             \
  V(TwoByteString)                                                             \
  V(ExternalOneByteString)                                                     \
  V(ExternalTwoByteString)
```

## 类型数组

```dart
#define CLASS_LIST_TYPED_DATA(V)                                               \
  V(Int8Array)                                                                 \
  V(Uint8Array)                                                                \
  V(Uint8ClampedArray)                                                         \
  V(Int16Array)                                                                \
  V(Uint16Array)                                                               \
  V(Int32Array)                                                                \
  V(Uint32Array)                                                               \
  V(Int64Array)                                                                \
  V(Uint64Array)                                                               \
  V(Float32Array)                                                              \
  V(Float64Array)                                                              \
  V(Float32x4Array)                                                            \
  V(Int32x4Array)                                                              \
  V(Float64x2Array)
```

## FFI类型

```c++
#define CLASS_LIST_FFI(V)                                                      \
  V(Int8)                                                                      \
  V(Int16)                                                                     \
  V(Int32)                                                                     \
  V(Int64)                                                                     \
  V(Uint8)                                                                     \
  V(Uint16)                                                                    \
  V(Uint32)                                                                    \
  V(Uint64)                                                                    \
  V(IntPtr)                                                                    \
  V(Float)                                                                     \
  V(Double)                                                                    \
  V(NativeType)                                                                \
  V(DynamicLibrary)                                                            \
  V(Struct)                                                                    \
  V(Pointer)                                                                   \
  V(NativeFunction)                                                            \
  V(Void)                                                                      \
  V(Handle)
```

