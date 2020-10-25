# Visitor

## visitor.h

```c++
// ...

namespace dart {

// Forward declarations.
class Isolate;
class IsolateGroup;

// 对象指针访问器接口
class ObjectPointerVisitor {
 public:
  explicit ObjectPointerVisitor(IsolateGroup* isolate_group);
  virtual ~ObjectPointerVisitor() {}

  IsolateGroup* isolate_group() const { return isolate_group_; }

  // Visit pointers inside the given typed data [view].
  //
  // Range of pointers to visit 'first' <= pointer <= 'last'.
  // 访问给定 first~last 范围之内的所有对象指针
  virtual void VisitTypedDataViewPointers(TypedDataViewPtr view,
                                          ObjectPtr* first,
                                          ObjectPtr* last) {
    VisitPointers(first, last);
  }

  // Range of pointers to visit 'first' <= pointer <= 'last'.
  virtual void VisitPointers(ObjectPtr* first, ObjectPtr* last) = 0;

  // len argument is the number of pointers to visit starting from 'p'.
  void VisitPointers(ObjectPtr* p, intptr_t len) {
    VisitPointers(p, (p + len - 1));
  }

  void VisitPointer(ObjectPtr* p) { VisitPointers(p, p); }

  const char* gc_root_type() const { return gc_root_type_; }
  void set_gc_root_type(const char* gc_root_type) {
    gc_root_type_ = gc_root_type;
  }

  void clear_gc_root_type() { gc_root_type_ = "unknown"; }

  virtual bool visit_weak_persistent_handles() const { return false; }

  // 当访问对象以维护对象引用路径的时候，时候使用对象的域来访问。
  // 否则，使用 Isolate 的 field_table 来访问
  virtual bool trace_values_through_fields() const { return false; }

  const SharedClassTable* shared_class_table() const {
    return shared_class_table_;
  }

 private:
  IsolateGroup* isolate_group_;
  const char* gc_root_type_;
  SharedClassTable* shared_class_table_;

  DISALLOW_IMPLICIT_CONSTRUCTORS(ObjectPointerVisitor);
};

// 对象访问器接口
class ObjectVisitor {
 public:
  ObjectVisitor() {}

  virtual ~ObjectVisitor() {}

  // Invoked for each object.
  virtual void VisitObject(ObjectPtr obj) = 0;

 private:
  DISALLOW_COPY_AND_ASSIGN(ObjectVisitor);
};

// 对象查找器接口
class FindObjectVisitor {
 public:
  FindObjectVisitor() {}
  virtual ~FindObjectVisitor() {}

  // Allow to specify a address filter.
  virtual uword filter_addr() const { return 0; }
  bool VisitRange(uword begin_addr, uword end_addr) const {
    uword addr = filter_addr();
    return (addr == 0) || ((begin_addr <= addr) && (addr < end_addr));
  }

  // 查找对象
  virtual bool FindObject(ObjectPtr obj) const = 0;

 private:
  DISALLOW_COPY_AND_ASSIGN(FindObjectVisitor);
};

}  // namespace dart

#endif  // RUNTIME_VM_VISITOR_H_
```



## visitor.cc

```c++
// ...

namespace dart {

ObjectPointerVisitor::ObjectPointerVisitor(IsolateGroup* isolate_group)
    : isolate_group_(isolate_group),
      gc_root_type_("unknown"),
      // 拷贝 IsolateGroup 的 shared_class_table
      shared_class_table_(isolate_group->shared_class_table()) {}

}  // namespace dart
```

