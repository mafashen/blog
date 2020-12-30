#### invoke 
```java
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        //此对象是否覆盖语言级访问检查
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = getCallerClass();
                //检查调用的类是否有对此方法的访问权限
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }
    
    //检查class文件方法上的public位是否为1
    public static boolean quickCheckMemberAccess(Class<?> var0, int var1) {
        return Modifier.isPublic(getClassAccessFlags(var0) & var1);
    }
    
    public static boolean isPublic(int mod) {
        return (mod & PUBLIC) != 0;
    }
```
#### MethodAccessor
```java
    public MethodAccessor newMethodAccessor(Method method) {
        checkInitted();

        if (Reflection.isCallerSensitive(method)) {
            Method altMethod = findMethodForReflection(method);
            if (altMethod != null) {
                method = altMethod;
            }
        }

        //读取 sun.reflect.noInflation 配置（-Dsun.reflect.noInflation=true），如果配置了将直接生成 Java实现
        if (noInflation && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            return new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        } 
        //否则使用Native实现
        else {
            NativeMethodAccessorImpl acc =
                new NativeMethodAccessorImpl(method);
            DelegatingMethodAccessorImpl res =
                new DelegatingMethodAccessorImpl(acc);
            acc.setParent(res);
            return res;
        }
    }
```
加入DelegatingMethodAccessorImpl，使用委派模式，方便进行实现的替换
```java
    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        return delegate.invoke(obj, args);
    }
```
Java实现的版本在初始化时需要较多时间，但长久来说性能较好；native版本正好相反，启动时相对较快，但运行时间长了之后速度就比不过Java版了。这是HotSpot的优化方式带来的性能特性，同时也是许多虚拟机的共同点：跨越native边界会对优化有阻碍作用，它就像个黑箱一样让虚拟机难以分析也将其内联，于是运行时间长了之后反而是托管版本的代码更快些。

当该反射调用成为热点时，它甚至可以被内联到靠近Method.invoke()的一侧，大大降低了反射调用的开销。而native版的反射调用则无法被有效内联，因而调用开销无法随程序的运行而降低。 
```java
    //sun.reflect.NativeMethodAccessorImpl
     public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        // 调用次数大于numInvocations（默认15）且不是匿名内部类，生成Java实现，并替换methodAccessor，之后的调用将使用Java实现的版本
        // 匿名内部类不能通过名称引用，因此不能从生成的字节码中找到
        if (++numInvocations > ReflectionFactory.inflationThreshold()
                && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            //使用字节码技术生成Java实现
            MethodAccessorImpl acc = (MethodAccessorImpl)
                new MethodAccessorGenerator().
                    generateMethod(method.getDeclaringClass(),
                                   method.getName(),
                                   method.getParameterTypes(),
                                   method.getReturnType(),
                                   method.getExceptionTypes(),
                                   method.getModifiers());
            parent.setDelegate(acc);
        }

        return invoke0(method, obj, args);
    }
```

#### JVM实现
NativeMethodAccessorImpl_invoke0方法的对应JVM实现在 NativeAccessors.c中
```
    JNIEXPORT jobject JNICALL Java_jdk_internal_reflect_NativeMethodAccessorImpl_invoke0
(JNIEnv *env, jclass unused, jobject m, jobject obj, jobjectArray args)
{
    return JVM_InvokeMethod(env, m, obj, args);
}
```
JVM_InvokeMethod 定义在 jvm.h 中
```
    JVM_ENTRY(jobject, JVM_InvokeMethod(JNIEnv *env, jobject method, jobject obj, jobjectArray args0))
  JVMWrapper("JVM_InvokeMethod");
  Handle method_handle;
  //检测当前线程的堆栈地址
  if (thread->stack_available((address) &method_handle) >= JVMInvokeMethodSlack) {
    method_handle = Handle(THREAD, JNIHandles::resolve(method));
    Handle receiver(THREAD, JNIHandles::resolve(obj));
    objArrayHandle args(THREAD, objArrayOop(JNIHandles::resolve(args0)));
    //调用方法
    oop result = Reflection::invoke_method(method_handle(), receiver, args, CHECK_NULL);
    jobject res = JNIHandles::make_local(env, result);
    if (JvmtiExport::should_post_vm_object_alloc()) {
      oop ret_type = java_lang_reflect_Method::return_type(method_handle());
      assert(ret_type != NULL, "sanity check: ret_type oop must not be NULL!");
      if (java_lang_Class::is_primitive(ret_type)) {
        // Only for primitive type vm allocates memory for java object.
        // See box() method.
        JvmtiExport::post_vm_object_alloc(JavaThread::current(), result);
      }
    }
    return res;
  } 
  //堆栈溢出
  else {
    THROW_0(vmSymbols::java_lang_StackOverflowError());
  }
JVM_END

    inline size_t JavaThread::stack_available(address cur_sp) {
      // This code assumes java stacks grow down
      address low_addr; // Limit on the address for deepest stack depth
      if (_stack_guard_state == stack_guard_unused) {
        low_addr = stack_end();
      } else {
        low_addr = stack_reserved_zone_base();
      }
      return cur_sp > low_addr ? cur_sp - low_addr : 0;
    }
```

Reflection.cpp
```
    oop Reflection::invoke_method(oop method_mirror, Handle receiver, objArrayHandle args, TRAPS) {
    //使用Java Class 对象中的属性
  oop mirror             = java_lang_reflect_Method::clazz(method_mirror);
  int slot               = java_lang_reflect_Method::slot(method_mirror);
  bool override          = java_lang_reflect_Method::override(method_mirror) != 0;
  //参数类型
  objArrayHandle ptypes(THREAD, objArrayOop(java_lang_reflect_Method::parameter_types(method_mirror)));
  //返回类型
  oop return_type_mirror = java_lang_reflect_Method::return_type(method_mirror);
  BasicType rtype;
  //返回类型是原始数据类型
  if (java_lang_Class::is_primitive(return_type_mirror)) {
    rtype = basic_type_mirror_to_basic_type(return_type_mirror, CHECK_NULL);
  } else {
    //返回数据类型是对象
    rtype = T_OBJECT;
  }

  instanceKlassHandle klass(THREAD, java_lang_Class::as_Klass(mirror));
  Method* m = klass->method_with_idnum(slot);
  if (m == NULL) {
    THROW_MSG_0(vmSymbols::java_lang_InternalError(), "invoke");
  }
  methodHandle method(THREAD, m);

  return invoke(klass, method, receiver, override, ptypes, rtype, args, true, THREAD);
}

// Method call (shared by invoke_method and invoke_constructor)
static oop invoke(instanceKlassHandle klass,
                  methodHandle reflected_method,
                  Handle receiver,
                  bool override,
                  objArrayHandle ptypes,
                  BasicType rtype,
                  objArrayHandle args,
                  bool is_method_invoke,
                  TRAPS) {

  ResourceMark rm(THREAD);

  methodHandle method;      // 实际调用的方法句柄
  KlassHandle target_klass; // target klass, receiver's klass for non-static

  // 确保 klass 已经初始化过了
  klass->initialize(CHECK_NULL);

  //是否是静态方法
  bool is_static = reflected_method->is_static();
  if (is_static) {
    // ignore receiver argument
    method = reflected_method;
    target_klass = klass;
  } else {
    // 目标对象为null
    if (receiver.is_null()) {
      THROW_0(vmSymbols::java_lang_NullPointerException());
    }
    // 调用对象必须是定义方法的类的实例
    if (!receiver->is_a(klass())) {
      THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(), "object is not an instance of declaring class");
    }
    // target klass is receiver's klass
    target_klass = KlassHandle(THREAD, receiver->klass());
    // no need to resolve if method is private or <init>
    if (reflected_method->is_private() || reflected_method->name() == vmSymbols::object_initializer_name()) {
      method = reflected_method;
    } else {
      // 定位真正应该调用的方法
      // 接口方法
      if (reflected_method->method_holder()->is_interface()) {
        // resolve interface call
        //
        // Match resolution errors with those thrown due to reflection inlining
        // Linktime resolution & IllegalAccessCheck already done by Class.getMethod()
        method = resolve_interface_call(klass, reflected_method, target_klass, receiver, THREAD);
        if (HAS_PENDING_EXCEPTION) {
          // Method resolution threw an exception; wrap it in an InvocationTargetException
          oop resolution_exception = PENDING_EXCEPTION;
          CLEAR_PENDING_EXCEPTION;
          // JVMTI has already reported the pending exception
          // JVMTI internal flag reset is needed in order to report InvocationTargetException
          if (THREAD->is_Java_thread()) {
            JvmtiExport::clear_detected_exception((JavaThread*)THREAD);
          }
          JavaCallArguments args(Handle(THREAD, resolution_exception));
          THROW_ARG_0(vmSymbols::java_lang_reflect_InvocationTargetException(),
                      vmSymbols::throwable_void_signature(),
                      &args);
        }
      }  else {
        // 如果方法可以被重写，我们将使用vtable索引进行解析
        assert(!reflected_method->has_itable_index(), "");
        int index = reflected_method->vtable_index();
        method = reflected_method;
        if (index != Method::nonvirtual_vtable_index) {
          method = methodHandle(THREAD, target_klass->method_at_vtable(index));
        }
        if (!method.is_null()) {
          // 抽象方法
          if (method->is_abstract()) {
            // new default: 6531596
            ResourceMark rm(THREAD);
            Handle h_origexception = Exceptions::new_exception(THREAD,
              vmSymbols::java_lang_AbstractMethodError(),
              Method::name_and_sig_as_C_string(target_klass(),
              method->name(),
              method->signature()));
            JavaCallArguments args(h_origexception);
            THROW_ARG_0(vmSymbols::java_lang_reflect_InvocationTargetException(),
              vmSymbols::throwable_void_signature(),
              &args);
          }
        }
      }
    }
  }

  // I believe this is a ShouldNotGetHere case which requires
  // an internal vtable bug. If you ever get this please let Karen know.
  if (method.is_null()) {
    ResourceMark rm(THREAD);
    THROW_MSG_0(vmSymbols::java_lang_NoSuchMethodError(),
                Method::name_and_sig_as_C_string(klass(),
                reflected_method->name(),
                reflected_method->signature()));
  }

  assert(ptypes->is_objArray(), "just checking");
  int args_len = args.is_null() ? 0 : args->length();
  // Check number of arguments
  if (ptypes->length() != args_len) {
    THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(),
                "wrong number of arguments");
  }
  // 创建对象来包含JavaCall的参数
  JavaCallArguments java_args(method->size_of_parameters());

  //调用实例方法，将方法参数压入堆栈
  if (!is_static) {
    java_args.push_oop(receiver);
  }

  for (int i = 0; i < args_len; i++) {
    oop type_mirror = ptypes->obj_at(i);
    oop arg = args->obj_at(i);
    if (java_lang_Class::is_primitive(type_mirror)) {
      jvalue value;
      BasicType ptype = basic_type_mirror_to_basic_type(type_mirror, CHECK_NULL);
      BasicType atype = Reflection::unbox_for_primitive(arg, &value, CHECK_NULL);
      if (ptype != atype) {
        Reflection::widen(&value, atype, ptype, CHECK_NULL);
      }
      switch (ptype) {
        case T_BOOLEAN:     java_args.push_int(value.z);    break;
        case T_CHAR:        java_args.push_int(value.c);    break;
        case T_BYTE:        java_args.push_int(value.b);    break;
        case T_SHORT:       java_args.push_int(value.s);    break;
        case T_INT:         java_args.push_int(value.i);    break;
        case T_LONG:        java_args.push_long(value.j);   break;
        case T_FLOAT:       java_args.push_float(value.f);  break;
        case T_DOUBLE:      java_args.push_double(value.d); break;
        default:
          THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(), "argument type mismatch");
      }
    } else {
      if (arg != NULL) {
        Klass* k = java_lang_Class::as_Klass(type_mirror);
        if (!arg->is_a(k)) {
          THROW_MSG_0(vmSymbols::java_lang_IllegalArgumentException(),
                      "argument type mismatch");
        }
      }
      Handle arg_handle(THREAD, arg);         // Create handle for argument
      java_args.push_oop(arg_handle); // Push handle
    }
  }

  assert(java_args.size_of_parameters() == method->size_of_parameters(),
    "just checking");

  // All oops (including receiver) is passed in as Handles. An potential oop is returned as an
  // oop (i.e., NOT as an handle)
  JavaValue result(rtype);
  //调用方法
  JavaCalls::call(&result, method, &java_args, THREAD);

  if (HAS_PENDING_EXCEPTION) {
    // Method threw an exception; wrap it in an InvocationTargetException
    oop target_exception = PENDING_EXCEPTION;
    CLEAR_PENDING_EXCEPTION;
    // JVMTI has already reported the pending exception
    // JVMTI internal flag reset is needed in order to report InvocationTargetException
    if (THREAD->is_Java_thread()) {
      JvmtiExport::clear_detected_exception((JavaThread*)THREAD);
    }

    JavaCallArguments args(Handle(THREAD, target_exception));
    THROW_ARG_0(vmSymbols::java_lang_reflect_InvocationTargetException(),
                vmSymbols::throwable_void_signature(),
                &args);
  } else {
    if (rtype == T_BOOLEAN || rtype == T_BYTE || rtype == T_CHAR || rtype == T_SHORT) {
      narrow((jvalue*)result.get_value_addr(), rtype, CHECK_NULL);
    }
    //返回结果自动装箱
    return Reflection::box((jvalue*)result.get_value_addr(), rtype, THREAD);
  }
}
```
