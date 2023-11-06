---
layout: post
categories: [JVM]
description: none
keywords: JVM
---
# 垃圾回收过程DefNewGeneration

## 分代实现
由于虚拟机的分代实现，虚拟机不会考虑各个内存代如何实现垃圾回收，具体的工作(对象内存的分配也是一样)由各内存代根据垃圾回收策略自行实现。

DefNewGeneration的使用复制算法进行回收。复制算法的思想是将eden和from区活跃的对象复制到to区，并清空eden区和from区，如果to区满了，那么部分对象将会被晋升移动到老年代，随后交换from和to区，即原来的to区存放了存活的对象作为新的from区存在，而from区被清空后当做新的to区而存在，移动次数超过一定阈值的对象也会被移动到老年代。

此外，在分析DefNewGeneration的垃圾回收之前，可以了解一下，在垃圾回收过程中，对对象的遍历处理定义一个抽象基类OopClosure(对象表)，并使用其不同的实现类来完成对对象的不同处理。
其中使用FastScanClosure来处理所有的根对象，FastEvacuateFollowersClosure处理所有的递归引用对象等。

在前文分析中，会调用各个内存代的collect()来完成垃圾回收。

DefNewGeneration的GC基本过程

DefNewGeneration::collect()定义在/hotspot/src/share/vm/memory/defNewGeneration.cpp中 。

当Survivor区空间不足，从Eden区移动过来的对象将会晋升到老年代，然而当老年代空间不足时，那么垃圾回收就是不安全的，将直接返回。
```
if (!collection_attempt_is_safe()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: Collection attempt not safe :: ");
    }
    gch->set_incremental_collection_failed(); // Slight lie: we did not even attempt one
    return;
  }
```

collection_attempt_is_safe()判断垃圾回收是否安全有以下判定条件：
```
bool DefNewGeneration::collection_attempt_is_safe() {
  if (!to()->is_empty()) {
    return false;
  }
  if (_next_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _next_gen = gch->next_gen(this);
    assert(_next_gen != NULL,
           "This must be the youngest gen, and not the only gen");
  }
  return _next_gen->promotion_attempt_is_safe(used());
}
```
- To区非空，则可能有不够充足的转移空间 
- 调用下一个内存代的promotion_attempt_is_safe()进行判断，是否有充足的空间容纳新生代的所有对象

一些准备工作
统计堆的使用空间大小(仅留作输出，可以不管)
准备IsAliveClosure、ScanWeakRefClosure。
```
IsAliveClosure is_alive(this);
ScanWeakRefClosure scan_weak_ref(this);
```

清空ageTable和to区。
```
age_table()->clear();
to()->clear(SpaceDecorator::Mangle);
```

在初始化堆的过程，会创建一个覆盖整个空间的数组GenRemSet，数组每个字节对应于堆的512字节，用于遍历新生代和老年代空间，这里对GenRemSet进行初始化准备。
```
 gch->rem_set()->prepare_for_younger_refs_iterate(false);

```

准备FastEvacuateFollowersClosure。
```
 FastScanClosure fsc_with_no_gc_barrier(this, false);
  FastScanClosure fsc_with_gc_barrier(this, true);

  set_promo_failure_scan_stack_closure(&fsc_with_no_gc_barrier);
  FastEvacuateFollowersClosure evacuate_followers(gch, _level, this,
                                                  &fsc_with_no_gc_barrier,
                                                  &fsc_with_gc_barrier);
```

调用GenCollectedHeap的gen_process_strong_roots()将当前代上的根对象复制到转移空间中。
```
 gch->gen_process_strong_roots(_level,
                                true,  // Process younger gens, if any,
                                       // as strong roots.
                                true,  // activate StrongRootsScope
                                false, // not collecting perm generation.
                                SharedHeap::SO_AllClasses,
                                &fsc_with_no_gc_barrier,
                                true,   // walk *all* scavengable nmethods
                                &fsc_with_gc_barrier);
```
递归处理根集对象的引用对象。
```
 // "evacuate followers".
  evacuate_followers.do_void();
```

处理发现的引用。
```
FastKeepAliveClosure keep_alive(this, &scan_weak_ref);
  ReferenceProcessor* rp = ref_processor();
  rp->setup_policy(clear_all_soft_refs);
  rp->process_discovered_references(&is_alive, &keep_alive, &evacuate_followers,
                                    NULL);
```

若没有发生晋升失败： 

那么此刻eden区和from区的对象应该已经全部转移了，将调用clear()情况这两片内存区域　。
```
if (!promotion_failed()) {
    // Swap the survivor spaces.
    eden()->clear(SpaceDecorator::Mangle);
    from()->clear(SpaceDecorator::Mangle);
```
交换from和to区域，为下次gc做准备。
```
swap_spaces();
```
swap_spaces只是交换了_from_space和_to_space的起始地址，并设置eden的下一片需要进行压缩的区域为现在的from区(与TenuredGeneration的标记-压缩-清理垃圾回收相关，用来标志各内存区的压缩顺序)，即原来的to区，而新的from区的下一片需要进行压缩的区域为为NULL。
```
void DefNewGeneration::swap_spaces() {
  ContiguousSpace* s = from();
  _from_space        = to();
  _to_space          = s;
  eden()->set_next_compaction_space(from());
  // The to-space is normally empty before a compaction so need
  // not be considered.  The exception is during promotion
  // failure handling when to-space can contain live objects.
  from()->set_next_compaction_space(NULL);

  //...
}
```
计算新的survior区域的对象进入老年代的经历的MinorGC次数阈值。
```
 // Set the desired survivor size to half the real survivor space
    _tenuring_threshold =
      age_table()->compute_tenuring_threshold(to()->capacity()/HeapWordSize);
```

当gc成功，会重新计算gc超时的时间计数。
```
AdaptiveSizePolicy* size_policy = gch->gen_policy()->size_policy();
    size_policy->reset_gc_overhead_limit_count();
```
若发生了晋升失败，即老年代没有足够的内存空间用以存放新生代所晋升的对象： 

恢复晋升失败对象的markOop(被标记的活跃对象的markword内容为转发指针，指向经过复制后对象的新地址)。
```
remove_forwarding_pointers();
```

remove_forwarding_pointers()会调用RemoveForwardPointerClosure对eden和from区内的对象进行遍历，RemoveForwardPointerClosure将调用其do_object()初始化eden和from区所有对象的对象头部分。
```
void DefNewGeneration::remove_forwarding_pointers() {
  RemoveForwardPointerClosure rspc;
  eden()->object_iterate(&rspc);
  from()->object_iterate(&rspc);
  //...assert
  while (!_objs_with_preserved_marks.is_empty()) {
    oop obj   = _objs_with_preserved_marks.pop();
    markOop m = _preserved_marks_of_objs.pop();
    obj->set_mark(m);
  }
  _objs_with_preserved_marks.clear(true);
  _preserved_marks_of_objs.clear(true);
```
在晋升失败处理的handle_promotion_failure()中，会将晋升失败对象以<oop, markOop>作为一对分别保存在_objs_with_preserved_marks和_preserved_marks_of_objs栈中，这里就会恢复晋升失败对象的对象头，并清除这两个栈。

仍然需要交换from和to区域，设置from的下一片需要进行压缩的区域为to
```
swap_spaces();
 from()->set_next_compaction_space(to());
```
当没有晋升失败是，gc成功，会清空eden和from区、交换from和to区、survivor区对象成熟阈值调整等，以准备下次gc；而当晋升失败时，虽然会在后面交换from和to区，但是并不会清空eden和from区，而是会清空eden和from区所有对象的对象头，而只恢复晋升失败部分的对象头(加上to区的部分就是全部活跃对象了)，这样，在随后触发的FullGC中能够对From和To区进行压缩处理。 

设置堆的MinorGC失败标记，并通知老年代(更高的内存代)晋升失败，比如在ConcurrentMarkSweepGeneration会根据配置进行dump输出以供JVM问题诊断
```
gch->set_incremental_collection_failed();

    // Inform the next generation that a promotion failure occurred.
    _next_gen->promotion_failure_occurred();
```
设置from和to区域的并发遍历指针的安全值为碰撞指针所在位置，并更新堆的最后一次gc的时间
```
  from()->set_concurrent_iteration_safe_limit(from()->top());
  to()->set_concurrent_iteration_safe_limit(to()->top());
  SpecializationStats::print();
  update_time_of_last_gc(os::javaTimeMillis());
```

下面将分别对根集对象标记、活跃对象标记、引用处理进行分析：

在分析gen_process_strong_roots()之前，首先看下处理函数会做哪些工作： 

处理函数封装在之前构造的FastScanClosure中，而FastScanClosure的do_oop()调用了的工作函数do_oop_work()。让我们看看do_oop_work()究竟做了什么。 

这里使用模板函数来解决压缩指针的不同类型(实际的oop和压缩指针narrowOop)问题，并当对象非空时，获取该oop/narrowOop对象(narrowOop需要进行指针解压)
```
 T heap_oop = oopDesc::load_heap_oop(p);
  // Should we copy the obj?
  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
```

若该对象在遍历区域内(_boudary是在FastScanClosure初始化的时候，为初始化时指定代的结束地址，与当前遍历代的起始地址_gen_boundary共同作为对象的访问边界，故新生代DefNewGeneration会将其自身内存代和更低的内存代的活跃对象都标记复制到to区域中)，若该对象没有被标记过，即其标记状态不为marked_value，就会将该对象复制到to区域内，随后根据是否使用指针压缩将新的对象地址进行压缩
```
if ((HeapWord*)obj < _boundary) {
      assert(!_g->to()->is_in_reserved(obj), "Scanning field twice?");
      oop new_obj = obj->is_forwarded() ? obj->forwardee()
                                        : _g->copy_to_survivor_space(obj);
      oopDesc::encode_store_heap_oop_not_null(p, new_obj);
    }
```
copy_to_survivor_space()的过程如下： 

当该对象占用空间小于应当直接移动到老年代的阈值时，就会将其分配到to区
```
size_t s = old->size();
  oop obj = NULL;

  // Try allocating obj in to-space (unless too old)
  if (old->age() < tenuring_threshold()) {
    obj = (oop) to()->allocate(s);
  }
```
否则会尝试将该对象晋升，若晋升失败，则调用handle_promotion_failure()处理
```
 if (obj == NULL) {
    obj = _next_gen->promote(old, s);
    if (obj == NULL) {
      handle_promotion_failure(old);
      return old;
    }
  }
```
将原对象的数据内容复制到to区域新分配的对象上，并增加该对象的复制计数和更新ageTable (Prefetch使用的是目标架构的prefetch指令，用于将指定地址和长度的内存预取到cache，用于提升存取性能)
```
else {
    // Prefetch beyond obj
    const intx interval = PrefetchCopyIntervalInBytes;
    Prefetch::write(obj, interval);

    // Copy obj
    Copy::aligned_disjoint_words((HeapWord*)old, (HeapWord*)obj, s);

    // Increment age if obj still in new generation
    obj->incr_age();
    age_table()->add(obj, s);
  }
```
最后调用forward_to()设置原对象的对象头为转发指针(表示该对象已被复制，并指明该对象已经被复制到什么位置)
```
  // Done, insert forward pointer to obj in this header
  old->forward_to(obj);
```
接下来分析gen_process_strong_roots()： 

现在考虑一个问题：我们知道，被根对象所触及的所有对象都是活跃对象，那么如何确定一个内存代中的活跃对象呢？或者换个思路，内存代中哪些对象是不可触及的垃圾对象呢？如果其他内存代没有指向该对象的引用并且该对象也没有被内存代内其他对象引用，那么该对象就是一个垃圾对象。据此，把内存代内活跃对象的处理分为两步：第一步，将内存代内正常的根对象和其他内存代内直接引用的内存代内的对象移动到To区域，这些对象作为活跃对象(虽然其他内存代的对象可能在下次Full GC成为垃圾对象，但显然Minor GC显然不能将这些对象当做垃圾对象)，这样，活跃对象的引用判断范围就缩小到了当前内存代，内存代内剩下的对象只要不是被这些活跃对象所引用，那么就必然是垃圾对象了；第二步，递归遍历这些对象，将其所引用的在该内存代的对象移动到To区域。最终，剩下的对象就是垃圾对象了。 

调用SharedHeap的process_strong_roots()处理根集对象，在当前内存代(新生代的eden和from区)的根集对象将会被复制到to区
```
if (!do_code_roots) {
    SharedHeap::process_strong_roots(activate_scope, collecting_perm_gen, so,
                                     not_older_gens, NULL, older_gens);
  } else {
    bool do_code_marking = (activate_scope || nmethod::oops_do_marking_is_active());
    CodeBlobToOopClosure code_roots(not_older_gens, /*do_marking=*/ do_code_marking);
    SharedHeap::process_strong_roots(activate_scope, collecting_perm_gen, so,
                                     not_older_gens, &code_roots, older_gens);
  }
```
结合FastScanClosure可知，process_strong_roots()主要将当前内存代上的正常根对象复制到To区域。 

处理更低的内存代
```
if (younger_gens_as_roots) {
    if (!_gen_process_strong_tasks->is_task_claimed(GCH_PS_younger_gens)) {
      for (int i = 0; i < level; i++) {
        not_older_gens->set_generation(_gens[i]);
        _gens[i]->oop_iterate(not_older_gens);
      }
      not_older_gens->reset_generation();
    }
  }
```

内存代的oop_iterate()是调用space_iterate()对该内存代的内存空间进行遍历
```
//定义在/hotspot/src/share/vm/memory/generation.cpp中
void Generation::oop_iterate(OopClosure* cl) {
  GenerationOopIterateClosure blk(cl, _reserved);
  space_iterate(&blk);
}
```

space_iterate()由Generation的实现类重写，以OneContigSpaceCardGeneration为例(后面会处理更高的内存代，这里DefNewGeneration并没有更低的内存代)，将遍历代上的内存空间。
```
void OneContigSpaceCardGeneration::space_iterate(SpaceClosure* blk,
                                                 bool usedOnly) {
  blk->do_space(_the_space);
}
```

GenerationOopIterateClosure的do_space()如下：
```
virtual void do_space(Space* s) {
    s->object_iterate(_cl);
  }
```

space的oop_iterate()根据Eden和from/to的实现如下： 
```
void ContiguousSpace::oop_iterate(MemRegion mr, OopClosure* blk) {
 //...
  HeapWord* obj_addr = block_start(mr.start());
  HeapWord* t = mr.end();

  // Handle first object specially.
  oop obj = oop(obj_addr);
  SpaceMemRegionOopsIterClosure smr_blk(blk, mr);
  obj_addr += obj->oop_iterate(&smr_blk);
  while (obj_addr < t) {
    oop obj = oop(obj_addr);
    assert(obj->is_oop(), "expected an oop");
    obj_addr += obj->size();
    // If "obj_addr" is not greater than top, then the
    // entire object "obj" is within the region.
    if (obj_addr <= t) {
      obj->oop_iterate(blk);
    } else {
      // "obj" extends beyond end of region
      obj->oop_iterate(&smr_blk);
      break;
    }
  };
}
```
该函数的作用是遍历该区域的起始地址到空闲分配指针之间的所有对象，并调用对象的oop_iterate()进行处理。

oop是在堆上的对象的基类型，其oop_iterate()调用了Klass的oop_oop_iterate##nv_suffix()
```
inline int oopDesc::oop_iterate(OopClosureType* blk) {                     \
  SpecializationStats::record_call();                                      \
  return blueprint()->oop_oop_iterate##nv_suffix(this, blk);               \
}   
```

oop_oop_iterate##nv_suffix()由具体的Klass子类(如对象在堆上的实现instanceKlass)实现，以访问和处理其所包含的引用对象
```
#define InstanceKlass_OOP_OOP_ITERATE_DEFN(OopClosureType, nv_suffix)        \
                                                                             \
int instanceKlass::oop_oop_iterate##nv_suffix(oop obj, OopClosureType* closure) { \
  SpecializationStats::record_iterate_call##nv_suffix(SpecializationStats::ik);\
  /* header */                                                          \
  if (closure->do_header()) {                                           \
    obj->oop_iterate_header(closure);                                   \
  }                                                                     \
  InstanceKlass_OOP_MAP_ITERATE(                                        \
    obj,                                                                \
    SpecializationStats::                                               \
      record_do_oop_call##nv_suffix(SpecializationStats::ik);           \
    (closure)->do_oop##nv_suffix(p),                                    \
    assert_is_in_closed_subset)                                         \
  return size_helper();                                                 \
}
```

instanceKlass的OopMapBlock描述了在实例对象空间中一连串引用类型域的起始位置和数量，而InstanceKlass_OOP_MAP_ITERATE(是一个语句块)会遍历OopMapBlock的所有块
```
#define InstanceKlass_OOP_MAP_ITERATE(obj, do_oop, assert_fn)            \
{                                                                        \
  /* Compute oopmap block range. The common case                         \
     is nonstatic_oop_map_size == 1. */                                  \
  OopMapBlock* map           = start_of_nonstatic_oop_maps();            \
  OopMapBlock* const end_map = map + nonstatic_oop_map_count();          \
  if (UseCompressedOops) {                                               \
    while (map < end_map) {                                              \
      InstanceKlass_SPECIALIZED_OOP_ITERATE(narrowOop,                   \
        obj->obj_field_addr<narrowOop>(map->offset()), map->count(),     \
        do_oop, assert_fn)                                               \
      ++map;                                                             \
    }                                                                    \
  } else {                                                               \
    while (map < end_map) {                                              \
      InstanceKlass_SPECIALIZED_OOP_ITERATE(oop,                         \
        obj->obj_field_addr<oop>(map->offset()), map->count(),           \
        do_oop, assert_fn)                                               \
      ++map;                                                             \
    }                                                                    \
  }                                                                      \
}
```
InstanceKlass_SPECIALIZED_OOP_ITERATE如下：
```
#define InstanceKlass_SPECIALIZED_OOP_ITERATE( \
  T, start_p, count, do_oop,                \
  assert_fn)                                \
{                                           \
  T* p         = (T*)(start_p);             \
  T* const end = p + (count);               \
  while (p < end) {                         \
    (assert_fn)(p);                         \
    do_oop;                                 \
    ++p;                                    \
  }                                         \
}
```
其中T为所要处理的对象的指针或压缩指针，start_p为OopMapBlock中引用域的起始地址，count为OopMapBlock中引用的数量，do_oop为引用的处理，assert_fn为断言，该宏所定义的语句块就是将对象引用域的引用调用FastScanClosure的do_oop_nv进行处理。 
　　所以，对更低内存代的遍历和处理就是把更低内存代的对象在DefNewGeneration内存代所引用的对象移动到To区域。

处理更高的内存代
```
for (int i = level+1; i < _n_gens; i++) {
    older_gens->set_generation(_gens[i]);
    rem_set()->younger_refs_iterate(_gens[i], older_gens);
    older_gens->reset_generation();
  }
```
类似地，把更高内存代的对象在DefNewGeneration内存代所引用的对象移动到To区域。这样就完成了第一步，将回收范围限定在DefNewGeneration内存代内。

## DefNewGeneration的存活对象的递归标记过程
在分析递归标记活跃对象的过程之前，不妨先了解一下递归标记所使用的cheney算法。 

在广优先遍历扫描活跃对象的过程中，对于所需的遍历队列，将复用to的从空闲指针开始的一段空间作为隐式队列。在之前，根集对象已经被拷贝到to区域的空闲空间，而scanned指针仍然停留在没有复制根集对象时空闲指针的位置，即scanned指针到当前空闲分配指针(to()->top())的这段空间保存着已经标记的根集对象，所以只需要继续遍历这段空间的根集对象，将发现的引用对象复制到to区域后，让scanned指针更新到这段空间的结束位置，而若还有未标记的对象的话，那么，空间指针必然又前进了一段距离，继续遍历这段新的未处理空间的对象，直至scanned指针追上空闲分配指针即可

FastEvacuateFollowersClosure的do_void()将完成递归标记工作：

当各分代的空闲分配指针不在变化时，说明所有可触及对象都已经递归标记完成，否则，将调用oop_since_save_marks_iterate()进行遍历标记。　
```
void DefNewGeneration::FastEvacuateFollowersClosure::do_void() {
  do {
    _gch->oop_since_save_marks_iterate(_level, _scan_cur_or_nonheap,
                                       _scan_older);
  } while (!_gch->no_allocs_since_save_marks(_level));
  guarantee(_gen->promo_failure_scan_is_complete(), "Failed to finish scan");
}
```
循环条件oop_since_save_marks_iterate()是对当前代、更高的内存代以及永久代检查其scanned指针_saved_mark_word是否与当前空闲分配指针位置相同，即检查scanned指针是否追上空闲分配指针
```
bool GenCollectedHeap::no_allocs_since_save_marks(int level) {
  for (int i = level; i < _n_gens; i++) {
    if (!_gens[i]->no_allocs_since_save_marks()) return false;
  }
  return perm_gen()->no_allocs_since_save_marks();
}
```
在DefNewGeneration中，eden和from区的分配指针不应当有所变化，只需要检查to区的空闲分配指针位置是否变化即可
```
bool DefNewGeneration::no_allocs_since_save_marks() {
  assert(eden()->saved_mark_at_top(), "Violated spec - alloc in eden");
  assert(from()->saved_mark_at_top(), "Violated spec - alloc in from");
  return to()->saved_mark_at_top();
}
```
循环处理oop_since_save_marks_iterate()： 

oop_since_save_marks_iterate()是对当前代、更高的内存代以及永久代的对象遍历处理
```
#define GCH_SINCE_SAVE_MARKS_ITERATE_DEFN(OopClosureType, nv_suffix)    \
void GenCollectedHeap::                                                 \
oop_since_save_marks_iterate(int level,                                 \
                             OopClosureType* cur,                       \
                             OopClosureType* older) {                   \
  _gens[level]->oop_since_save_marks_iterate##nv_suffix(cur);           \
  for (int i = level+1; i < n_gens(); i++) {                            \
    _gens[i]->oop_since_save_marks_iterate##nv_suffix(older);           \
  }                                                                     \
  perm_gen()->oop_since_save_marks_iterate##nv_suffix(older);           \
}
```
那么为什么要处理更高的内存代对象？因为在复制过程中，有对象通过晋升移动到了更高的内存代。 

不过为什么老年代TenuredGeneration不像ConcurrentMarkSweepGeneration一样维护一个晋升对象的链表PromotionInfo来加快晋升对象的处理呢？ 　

oop_since_save_marks_iterate##nv_suffix()在DefNewGeneration中的定义如下，实际上是调用eden、to、from区的同名函数进行处理，并更新各区的空闲分配指针。
```
#define DefNew_SINCE_SAVE_MARKS_DEFN(OopClosureType, nv_suffix) \
                                                                \
void DefNewGeneration::                                         \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* cl) {   \
  cl->set_generation(this);                                     \
  eden()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  to()->oop_since_save_marks_iterate##nv_suffix(cl);            \
  from()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  cl->reset_generation();                                       \
  save_marks();                                                 \
}
```
之前说到，在空间分配指针到scanned指针之间的区域就是已分配但未扫描的对象，所以在这里将对这片区域内的对象调用遍历函数进行处理，以标记遍历的对象所引用的对象，并保存新的scanned指针。
```
#define ContigSpace_OOP_SINCE_SAVE_MARKS_DEFN(OopClosureType, nv_suffix)  \
                                                                          \
void ContiguousSpace::                                                    \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* blk) {            \
  HeapWord* t;                                                            \
  HeapWord* p = saved_mark_word();                                        \
  assert(p != NULL, "expected saved mark");                               \
                                                                          \
  const intx interval = PrefetchScanIntervalInBytes;                      \
  do {                                                                    \
    t = top();                                                            \
    while (p < t) {                                                       \
      Prefetch::write(p, interval);                                       \
      debug_only(HeapWord* prev = p);                                     \
      oop m = oop(p);                                                     \
      p += m->oop_iterate(blk);                                           \
    }                                                                     \
  } while (t < top());                                                    \
                                                                          \
  set_saved_mark_word(p);                                                 \
```

## DefNewGeneration的引用处理：
处理_discoveredSoftRefs数组中的软引用
```
  // Soft references
  {
    TraceTime tt("SoftReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();
```

处理_discoveredWeakRefs数组中的弱引用
```
// Weak references
  {
    TraceTime tt("WeakReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }
```
处理_discoveredFinalRefs数组中的Final引用
```
// Final references
  {
    TraceTime tt("FinalReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

```
处理_discoveredPhantomRefs列表中的影子引用
```
// Phantom references
  {
    TraceTime tt("PhantomReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                               is_alive, keep_alive, complete_gc, task_executor);
  }
```
处理JNI弱全局引用
```
{
    TraceTime tt("JNI Weak Reference", trace_time, false, gclog_or_tty);
    if (task_executor != NULL) {
      task_executor->set_single_threaded_mode();
    }
    process_phaseJNI(is_alive, keep_alive, complete_gc);
  }
```