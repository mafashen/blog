**AtomicInteger** 是Java提供的原子操作类，其内部通过 **UnSafe** 工具类，使用 ==CAS(compare and set)== 的方式保证更新操作的原子性；==*CAS*== 可以看成是一种乐观锁的实现方式，每次更新前会检查当前的值是否符合预期，符合才进行更新；其依赖于底层硬件的指令支持，不同的体系结构的处理器不一样。因为其锁的粒度低，所以在并发下能够提供更优秀的吞吐量表现。

---
下面一起来看下 **AtomicInteger** 的具体实现
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            //jni native
            valueOffset = U.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }

    private volatile int value;
}
```
在静态代码块中，通过本地方法 *objectFieldOffset()* 获取到 *value* 字段的偏移地址 
```
jint find_field_offset(jobject field, int must_be_static, TRAPS) {
  if (field == NULL) {
    THROW_0(vmSymbols::java_lang_NullPointerException());
  }

  oop reflected   = JNIHandles::resolve_non_null(field);
  //获取指定字段的class类型
  oop mirror      = java_lang_reflect_Field::clazz(reflected);
  //转换为JVM中的 Klass 类型
  Klass* k      = java_lang_Class::as_Klass(mirror);
  //获取字段的槽位
  int slot        = java_lang_reflect_Field::slot(reflected);
  //获取字段上的修饰符
  int modifiers   = java_lang_reflect_Field::modifiers(reflected);
  //静态字段
  if (must_be_static >= 0) {
    int really_is_static = ((modifiers & JVM_ACC_STATIC) != 0);
    if (must_be_static != really_is_static) {
      THROW_0(vmSymbols::java_lang_IllegalArgumentException());
    }
  }

  int offset = InstanceKlass::cast(k)->field_offset(slot);
  return field_offset_from_byte_offset(offset);
}
```
```
//instanceKlass.hpp
FieldInfo* field(int index) const { return FieldInfo::from_field_array(_fields, index); }

//fieldinfo.hpp
static FieldInfo* from_field_array(Array<u2>* fields, int index) {
    return ((FieldInfo*)fields->adr_at(index * field_slots));
  }
```
其实现方式就是通过JVM内部使用的Klass定位到传入的字段在对象字段表中的偏移量

---
**AtomicInteger** 实现 ==*CAS*== 的方式主要是通过 **UnSafe** 内部的 *compareAndSwapInt()* 方法
```
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```
此处传入的valueOffset即在静态代码块中获取到的 *value* 字段相对于具体对象的逻辑偏移量
```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  //获取到要处理的int变量的指针
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```
```c++
inline void* index_oop_from_field_offset_long(oop p, jlong field_offset) {
  jlong byte_offset = field_offset_to_byte_offset(field_offset);
#ifdef ASSERT
  if (p != NULL) {
    assert(byte_offset >= 0 && byte_offset <= (jlong)MAX_OBJECT_SIZE, "sane offset");
    if (byte_offset == (jint)byte_offset) {
      void* ptr_plus_disp = (address)p + byte_offset;
      assert((void*)p->obj_field_addr<oop>((jint)byte_offset) == ptr_plus_disp,
             "raw [ptr+disp] must be consistent with oop::field_base");
    }
    jlong p_size = HeapWordSize * (jlong)(p->size());
    assert(byte_offset < p_size, err_msg("Unsafe access: offset " INT64_FORMAT " > object's size " INT64_FORMAT, byte_offset, p_size));
  }
#endif
  if (sizeof(char*) == sizeof(jint))    // (this constant folds!)
    return (address)p + (jint) byte_offset;
  else
    return (address)p +        byte_offset;
}

inline jlong field_offset_from_byte_offset(jlong byte_offset) {
  return byte_offset;
}
```

```
jbyte Atomic::cmpxchg(jbyte exchange_value, volatile jbyte* dest, jbyte compare_value) {
  assert(sizeof(jbyte) == 1, "assumption.");
  uintptr_t dest_addr = (uintptr_t)dest;
  uintptr_t offset = dest_addr % sizeof(jint);
  volatile jint* dest_int = (volatile jint*)(dest_addr - offset);
  //获取到当前值
  jint cur = *dest_int;
  //转为byte指针
  jbyte* cur_as_bytes = (jbyte*)(&cur);
  jint new_val = cur;
  jbyte* new_val_as_bytes = (jbyte*)(&new_val);
  new_val_as_bytes[offset] = exchange_value;
  while (cur_as_bytes[offset] == compare_value) {
    jint res = cmpxchg(new_val, dest_int, cur);
    if (res == cur) break;
    cur = res;
    new_val = cur;
    new_val_as_bytes[offset] = exchange_value;
  }
  return cur_as_bytes[offset];
}

unsigned Atomic::cmpxchg(unsigned int exchange_value,
                         volatile unsigned int* dest, unsigned int compare_value) {
  assert(sizeof(unsigned int) == sizeof(jint), "more work to do");
  return (unsigned int)Atomic::cmpxchg((jint)exchange_value, (volatile jint*)dest,
                                       (jint)compare_value);
}
```
atomic_linux_x86.cpp
```
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```
可以看到 Java 提供的**UnSafe** 中 ==*CAS*== 操作，在JVM中，主要通过 Atomic.cpp 来实现的，一些具体的操作，又根据不同的处理器体系结构，定义了不同的汇编代码来操作底层硬件指令。
