在进行MinorGC的时候，新生代一般使用的复制算法，在标记完存活对象后，需要将仍然存活的对象复制到survivor区，下面是hotspot中将对象复制到survivor区的实现代码
```c
oop DefNewGeneration::copy_to_survivor_space(oop old) {
  assert(is_in_reserved(old) && !old->is_forwarded(),
         "shouldn't be scavenging this oop");
  size_t s = old->size();
  oop obj = NULL;

  // Try allocating obj in to-space (unless too old)
  if (old->age() < tenuring_threshold()) {
    obj = (oop) to()->allocate(s); // to区域分配
  }

  // Otherwise try allocating obj tenured
  // 这里可能是对象年龄大于晋升年龄，也可能是survivor区的空间不足，分配失败引起的
  if (obj == NULL) {
    obj = _next_gen->promote(old, s); // 老年代分配
    if (obj == NULL) {
      handle_promotion_failure(old);
      return old;
    }
  } else {
    // Prefetch beyond obj
    const intx interval = PrefetchCopyIntervalInBytes;
    Prefetch::write(obj, interval);

    // Copy obj 拷贝对象
    Copy::aligned_disjoint_words((HeapWord*)old, (HeapWord*)obj, s);

    // Increment age if obj still in new generation
    // 如果对象仍然在新生代，对象年龄+1
    obj->incr_age();
    // 统计相同年龄对象大小
    age_table()->add(obj, s);
  }

  // Done, insert forward pointer to obj in this header
  old->forward_to(obj);

  return obj;
}
```
可以看到，会对当前复制的对象进行年龄上的判断，如果小于 **tenuring_threshold** 则在to区域分配内存，否则在老年代分配内存；而 **tenuring_threshold** 的计算正是对象动态晋升的机制，除了和指定的最大晋升年龄 **MaxTenuringThreshold** 参数有关，还和另外一个参数 **TargetSurvivorRatio** 有关。
```c
void DefNewGeneration::adjust_desired_tenuring_threshold() {
  // Set the desired survivor size to half the real survivor space
  _tenuring_threshold =
    age_table()->compute_tenuring_threshold(to()->capacity()/HeapWordSize);
}
```
*compute_tenuring_threshold* 方法会动态计算对象的晋升年龄，其中size数组保存的是年龄从小到大的对象总大小，计算方法则是从小到大累加每个年龄的对象，当总大小超过survivor区大小 * TargetSurvivorRatio 的大小时，晋升的年龄就是最后的累加的这个年龄，大于或等于这个年龄的对象都会晋升到老年代。
```c
uint ageTable::compute_tenuring_threshold(size_t survivor_capacity) {
  size_t desired_survivor_size = (size_t)((((double) survivor_capacity)*TargetSurvivorRatio)/100);
  size_t total = 0;
  uint age = 1;
  assert(sizes[0] == 0, "no objects with age zero should be recorded");
  while (age < table_size) {
    total += sizes[age];
    // check if including objects of age 'age' made us pass the desired
    // size, if so 'age' is the new threshold
    if (total > desired_survivor_size) break;
    age++;
  }
  uint result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;
  return result;
}
```
size数组的计算，对于每个标记存活的对象，都会调用ageTable的add方法，统计相同年龄对象的总大小
```
void add(oop p, size_t oop_size) {
    uint age = p->age();
    assert(age > 0 && age < table_size, "invalid age of object");
    sizes[age] += oop_size;
}
```
