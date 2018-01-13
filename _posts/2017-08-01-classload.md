---
layout: post
title:  "ART 类加载"
date:   2017-08-01 15:42:40
categories: Android
tags: System
---


有两个方法，分别是ClassLoader的findClass和loadClass，区别是loadclass会尝试去寻找已经加载的类，若没有，则尝试去父classloader加载类，若还是没有，则调用findclass。
//android 5+以上无法用findClass找到android.app.Application类，未验证

findclass由BaseDexClassLoader实现：

```java
@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);}
            throw cnfe;}
        return c;}
```
可以看见调用了pathLIst（DexPathList类型）的findclass方法：

```java
    public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;}}}
        if (dexElementsSuppressedExceptions != null) {
suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;}
```

DexPathList的Element数组，存放了加载的DexFile，一次调用各DexFile的loadClassBinaryName：

```java
public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, suppressed);
    }

    private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                     List<Throwable> suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie);
        } catch (NoClassDefFoundError e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        } catch (ClassNotFoundException e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        }
        return result;
    }
```
最后调用了defineClassNative，对应art/runtime/native/dalvik_system_DexFile.cc中的DexFile_defineClassNative方法：

```cpp
static jclass DexFile_defineClassNative(JNIEnv* env, jclass, jstring javaName, jobject javaLoader,
                                        jobject cookie) {
  std::unique_ptr<std::vector<const DexFile*>> dex_files = ConvertJavaArrayToNative(env, cookie);
  if (dex_files.get() == nullptr) {
    VLOG(class_linker) << "Failed to find dex_file";
    DCHECK(env->ExceptionCheck());
    return nullptr;
  }

  ScopedUtfChars class_name(env, javaName);
  if (class_name.c_str() == nullptr) {
    VLOG(class_linker) << "Failed to find class_name";
    return nullptr;
  }
  const std::string descriptor(DotToDescriptor(class_name.c_str()));
  const size_t hash(ComputeModifiedUtf8Hash(descriptor.c_str()));
  for (auto& dex_file : *dex_files) {
    const DexFile::ClassDef* dex_class_def = dex_file->FindClassDef(descriptor.c_str(), hash);
    if (dex_class_def != nullptr) {
      ScopedObjectAccess soa(env);
      ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
      class_linker->RegisterDexFile(*dex_file);
      StackHandleScope<1> hs(soa.Self());
      Handle<mirror::ClassLoader> class_loader(
          hs.NewHandle(soa.Decode<mirror::ClassLoader*>(javaLoader)));
      mirror::Class* result = class_linker->DefineClass(soa.Self(), descriptor.c_str(), hash,
                                                        class_loader, *dex_file, *dex_class_def);
      if (result != nullptr) {
        VLOG(class_linker) << "DexFile_defineClassNative returning " << result
                           << " for " << class_name.c_str();
        return soa.AddLocalReference<jclass>(result);
      }
    }
  }
  VLOG(class_linker) << "Failed to find dex_class_def " << class_name.c_str();
  return nullptr;
}
```
调用了class_linker->DefineClass，这个函数内容很多，抓重点：

```cpp
// Add the newly loaded class to the loaded classes table.
  mirror::Class* existing = InsertClass(descriptor, klass.Get(), hash);
  if (existing != nullptr) {
    // We failed to insert because we raced with another thread. Calling EnsureResolved may cause
    // this thread to block.
    return EnsureResolved(self, descriptor, existing);
  }

  // Load the fields and other things after we are inserted in the table. This is so that we don't
  // end up allocating unfree-able linear alloc resources and then lose the race condition. The
  // other reason is that the field roots are only visited from the class table. So we need to be
  // inserted before we allocate / fill in these fields.
  LoadClass(self, dex_file, dex_class_def, klass);
  if (self->IsExceptionPending()) {
    // An exception occured during load, set status to erroneous while holding klass' lock in case
    // notification is necessary.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, mirror::Class::kStatusError, self);
    }
    return nullptr;
  }

  // Finish loading (if necessary) by finding parents
  CHECK(!klass->IsLoaded());
  if (!LoadSuperAndInterfaces(klass, dex_file)) {
    // Loading failed.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, mirror::Class::kStatusError, self);
    }
    return nullptr;
  }
  CHECK(klass->IsLoaded());
  // Link the class (if necessary)
  CHECK(!klass->IsResolved());
  // TODO: Use fast jobjects?
  auto interfaces = hs.NewHandle<mirror::ObjectArray<mirror::Class>>(nullptr);

  MutableHandle<mirror::Class> h_new_class = hs.NewHandle<mirror::Class>(nullptr);
  if (!LinkClass(self, descriptor, klass, interfaces, &h_new_class)) {
    // Linking failed.
    if (!klass->IsErroneous()) {
      mirror::Class::SetStatus(klass, mirror::Class::kStatusError, self);
    }
    return nullptr;
  }
```
先调用InsertClass将新类添加到已加载类的列表中。再调用LoadClass和LinkClass来加载和链接类。

```cpp
void ClassLinker::LoadClass(Thread* self, const DexFile& dex_file,
                            const DexFile::ClassDef& dex_class_def,
                            Handle<mirror::Class> klass) {
  const uint8_t* class_data = dex_file.GetClassData(dex_class_def);
  if (class_data == nullptr) {
    return;  // no fields or methods - for example a marker interface
  }
  bool has_oat_class = false;
  if (Runtime::Current()->IsStarted() && !Runtime::Current()->IsAotCompiler()) {
    OatFile::OatClass oat_class = FindOatClass(dex_file, klass->GetDexClassDefIndex(),
                                               &has_oat_class);
    if (has_oat_class) {
      LoadClassMembers(self, dex_file, class_data, klass, &oat_class);
    }
  }
  if (!has_oat_class) {
    LoadClassMembers(self, dex_file, class_data, klass, nullptr);
  }
}
```
这里分为，有oat和没有oat，最后都是调用 LoadClassMembers：

```cpp
void ClassLinker::LoadClassMembers(Thread* self, const DexFile& dex_file,
                                   const uint8_t* class_data,
                                   Handle<mirror::Class> klass,
                                   const OatFile::OatClass* oat_class) {
  {
    // Note: We cannot have thread suspension until the field and method arrays are setup or else
    // Class::VisitFieldRoots may miss some fields or methods.
    ScopedAssertNoThreadSuspension nts(self, __FUNCTION__);
    // Load static fields.
    ClassDataItemIterator it(dex_file, class_data);
    const size_t num_sfields = it.NumStaticFields();
    ArtField* sfields = num_sfields != 0 ? AllocArtFieldArray(self, num_sfields) : nullptr;
    for (size_t i = 0; it.HasNextStaticField(); i++, it.Next()) {
      CHECK_LT(i, num_sfields);
      LoadField(it, klass, &sfields[i]);
    }
    klass->SetSFields(sfields);
    klass->SetNumStaticFields(num_sfields);
    DCHECK_EQ(klass->NumStaticFields(), num_sfields);
    // Load instance fields.
    const size_t num_ifields = it.NumInstanceFields();
    ArtField* ifields = num_ifields != 0 ? AllocArtFieldArray(self, num_ifields) : nullptr;
    for (size_t i = 0; it.HasNextInstanceField(); i++, it.Next()) {
      CHECK_LT(i, num_ifields);
      LoadField(it, klass, &ifields[i]);
    }
    klass->SetIFields(ifields);
    klass->SetNumInstanceFields(num_ifields);
    DCHECK_EQ(klass->NumInstanceFields(), num_ifields);
    ArtMethod* const direct_methods = (it.NumDirectMethods() != 0)
        ? AllocArtMethodArray(self, it.NumDirectMethods())
        : nullptr;
    ArtMethod* const virtual_methods = (it.NumVirtualMethods() != 0)
        ? AllocArtMethodArray(self, it.NumVirtualMethods())
        : nullptr;
    {
      // Used to get exclusion between with VisitNativeRoots so that no thread sees a length for
      // one array with a pointer for a different array.
      WriterMutexLock mu(self, *Locks::classlinker_classes_lock_);
      // Load methods.
      klass->SetDirectMethodsPtr(direct_methods);
      klass->SetNumDirectMethods(it.NumDirectMethods());
      klass->SetVirtualMethodsPtr(virtual_methods);
      klass->SetNumVirtualMethods(it.NumVirtualMethods());
    }
    size_t class_def_method_index = 0;
    uint32_t last_dex_method_index = DexFile::kDexNoIndex;
    size_t last_class_def_method_index = 0;
    for (size_t i = 0; it.HasNextDirectMethod(); i++, it.Next()) {
      ArtMethod* method = klass->GetDirectMethodUnchecked(i, image_pointer_size_);
      LoadMethod(self, dex_file, it, klass, method);
      LinkCode(method, oat_class, class_def_method_index);
      uint32_t it_method_index = it.GetMemberIndex();
      if (last_dex_method_index == it_method_index) {
        // duplicate case
        method->SetMethodIndex(last_class_def_method_index);
      } else {
        method->SetMethodIndex(class_def_method_index);
        last_dex_method_index = it_method_index;
        last_class_def_method_index = class_def_method_index;
      }
      class_def_method_index++;
    }
    for (size_t i = 0; it.HasNextVirtualMethod(); i++, it.Next()) {
      ArtMethod* method = klass->GetVirtualMethodUnchecked(i, image_pointer_size_);
      LoadMethod(self, dex_file, it, klass, method);
      DCHECK_EQ(class_def_method_index, it.NumDirectMethods() + i);
      LinkCode(method, oat_class, class_def_method_index);
      class_def_method_index++;
    }
    DCHECK(!it.HasNext());
  }
  self->AllowThreadSuspension();
}
```
代码很长，大体就是前半部分加载class的静态域和实例域，后半部分先加载direct方法，再加载virtual方法。先调用LoadMethod方法初始化ArtMethod对象。

```cpp
void ClassLinker::LoadMethod(Thread* self, const DexFile& dex_file, const ClassDataItemIterator& it,
2392                             Handle<mirror::Class> klass, ArtMethod* dst) {
2393  uint32_t dex_method_idx = it.GetMemberIndex();
2394  const DexFile::MethodId& method_id = dex_file.GetMethodId(dex_method_idx);
  const char* method_name = dex_file.StringDataByIdx(method_id.name_idx_);

  ScopedAssertNoThreadSuspension ants(self, "LoadMethod");
  dst->SetDexMethodIndex(dex_method_idx);
  dst->SetDeclaringClass(klass.Get());
  dst->SetCodeItemOffset(it.GetMethodCodeItemOffset());

  dst->SetDexCacheResolvedMethods(klass->GetDexCache()->GetResolvedMethods());
  dst->SetDexCacheResolvedTypes(klass->GetDexCache()->GetResolvedTypes());

  uint32_t access_flags = it.GetMethodAccessFlags();

  if (UNLIKELY(strcmp("finalize", method_name) == 0)) {
    // Set finalizable flag on declaring class.
    if (strcmp("V", dex_file.GetShorty(method_id.proto_idx_)) == 0) {
      // Void return type.
      if (klass->GetClassLoader() != nullptr) {  // All non-boot finalizer methods are flagged.
        klass->SetFinalizable();
      } else {
        std::string temp;
        const char* klass_descriptor = klass->GetDescriptor(&tem       // The Enum class declares a "final" finalize() method to prevent subclasses from
        // introducing a finalizer. We don't want to set the finalizable flag for Enum or its
        // subclasses, so we exclude it here.
        // We also want to avoid setting the flag on Object, where we know that finalize() is
        // empty.
        if (strcmp(klass_descriptor, "Ljava/lang/Object;") != 0 &&
            strcmp(klass_descriptor, "Ljava/lang/Enum;") != 0) {
          klass->SetFinalizable();
        }
      }
    }
  } else if (method_name[0] == '<') {
    // Fix broken access flags for initializers. Bug 11157540.
    bool is_init = (strcmp("<init>", method_name) == 0);
    bool is_clinit = !is_init && (strcmp("<clinit>", method_name) == 0);
    if (UNLIKELY(!is_init && !is_clinit)) {
      LOG(WARNING) << "Unexpected '<' at start of method name " << method_name;
    } else {
      if (UNLIKELY((access_flags & kAccConstructor) == 0)) {
        LOG(WARNING) << method_name << " didn't have expected constructor access flag in class "
            << PrettyDescriptor(klass.Get()) << " in dex file " << dex_file.GetLocation();
        access_flags |= kAccConstructor;
      }
    }
  }
  dst->SetAccessFlags(access_flags);
}
```

```cpp
void ClassLinker::LinkCode(ArtMethod* method, const OatFile::OatClass* oat_class,
                           uint32_t class_def_method_index) {
  Runtime* const runtime = Runtime::Current();
  if (runtime->IsAotCompiler()) {
    // The following code only applies to a non-compiler runtime.
    return;
  }
  // Method shouldn't have already been linked.
  DCHECK(method->GetEntryPointFromQuickCompiledCode() == nullptr);
  if (oat_class != nullptr) {
    // Every kind of method should at least get an invoke stub from the oat_method.
    // non-abstract methods also get their code pointers.
    const OatFile::OatMethod oat_method = oat_class->GetOatMethod(class_def_method_index);
    oat_method.LinkMethod(method);
  }

  // Install entry point from interpreter.
  bool enter_interpreter = NeedsInterpreter(method, method->GetEntryPointFromQuickCompiledCode());
  if (enter_interpreter && !method->IsNative()) {
    method->SetEntryPointFromInterpreter(artInterpreterToInterpreterBridge);
  } else {
    method->SetEntryPointFromInterpreter(artInterpreterToCompiledCodeBridge);
  }

  if (method->IsAbstract()) {
    method->SetEntryPointFromQuickCompiledCode(GetQuickToInterpreterBridge());
    return;
  }

  if (method->IsStatic() && !method->IsConstructor()) {
    // For static methods excluding the class initializer, install the trampoline.
    // It will be replaced by the proper entry point by ClassLinker::FixupStaticTrampolines
    // after initializing class (see ClassLinker::InitializeClass method).
    method->SetEntryPointFromQuickCompiledCode(GetQuickResolutionStub());
  } else if (enter_interpreter) {
    if (!method->IsNative()) {
      // Set entry point from compiled code if there's no code or in interpreter only mode.
      method->SetEntryPointFromQuickCompiledCode(GetQuickToInterpreterBridge());
    } else {
      method->SetEntryPointFromQuickCompiledCode(GetQuickGenericJniStub());
    }
  }

  if (method->IsNative()) {
    // Unregistering restores the dlsym lookup stub.
    method->UnregisterNative();

    if (enter_interpreter) {
      // We have a native method here without code. Then it should have either the generic JNI
      // trampoline as entrypoint (non-static), or the resolution trampoline (static).
      // TODO: this doesn't handle all the cases where trampolines may be installed.
      const void* entry_point = method->GetEntryPointFromQuickCompiledCode();
      DCHECK(IsQuickGenericJniStub(entry_point) || IsQuickResolutionStub(entry_point));
    }
  }
}
```
前半部分新设置是解释执行还是本地代码执行：

```cpp
static bool NeedsInterpreter(ArtMethod* method, const void* quick_code)
    SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
  if (quick_code == nullptr) {
    // No code: need interpreter.
    // May return true for native code, in the case of generic JNI
    // DCHECK(!method->IsNative());
    return true;
  }
  // If interpreter mode is enabled, every method (except native and proxy) must
  // be run with interpreter.
  return Runtime::Current()->GetInstrumentation()->InterpretOnly() &&
         !method->IsNative() && !method->IsProxyMethod();
}
```
由上可知，如果没有oat的话，肯定是解释执行，如果由oat则需要看是否设置了强制解释执行。回到LinkCode后半部分，主要是根据各种情况设置method的解释执行入口点，或者执行编译代码的入口点。
说一下ArtMethod这个结构：

```cpp
class ArtMethod {
  // Field order required by test "ValidateFieldOrderOfJavaCppUnionClasses". 
  // The class we are a part of. 
  GcRoot<mirror::Class> declaring_class_;

  // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access. 
  GcRoot<mirror::PointerArray> dex_cache_resolved_methods_;

  // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access. 
  GcRoot<mirror::ObjectArray<mirror::Class>> dex_cache_resolved_types_;

  // Access flags; low 16 bits are defined by spec. 
  uint32_t access_flags_;

  /* Dex file fields. The defining dex file is available via declaring_class_->dex_cache_ */

  // Offset to the CodeItem. 
  uint32_t dex_code_item_offset_;

  // Index into method_ids of the dex file associated with this method. 
  uint32_t dex_method_index_;

  /* End of dex file fields. */

  // Entry within a dispatch table for this method. For static/direct methods the index is into 
  // the declaringClass.directMethods, for virtual methods the vtable and for interface methods the 
  // ifTable. 
  uint32_t method_index_;

  // Fake padding field gets inserted here. 
  // Must be the last fields in the method. 
  // PACKED(4) is necessary for the correctness of 
  // RoundUp(OFFSETOF_MEMBER(ArtMethod, ptr_sized_fields_), pointer_size). 
  struct PACKED(4) PtrSizedFields {

    // Method dispatch from the interpreter invokes this pointer which may cause a bridge into 
    // compiled code. 
    void* entry_point_from_interpreter_;

    // Pointer to JNI function registered to this method, or a function to resolve the JNI function. 
    void* entry_point_from_jni_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into 
    // the interpreter. 
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
}
```
`dex_cache_resolved_methods_`是一个指针数组，指向已经解析的过的ArtMethod，假如一个函数被调用时没有被解析过，由数组dex_cache_resolved_methods_获取并执行的是resolution_method_。待解析完成，得到callee的实际ArtMethod后，再去执行实际的代码；此外，还会将解析得到的ArtMethod填充到数组dex_cache_resolved_methods_的相应位置。这样，之后callee再被调用时，便无需再次进行方法解析。这跟plt和got的原理时一样的。