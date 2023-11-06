---
layout: post
categories: [JVM]
description: none
keywords: JVM
---
# 堆初始化GenCollectedHeap

## 堆的初始化
接下来分析特定堆的初始化过程，这里以GenCollectedHeap和MarkSweepPolicy为例： 
```
#endif // INCLUDE_ALL_GCS
    // 若没有进行配置，虚拟机将默认使用MarkSweepPolicy策略
    } else { // default old generation
      gc_policy = new MarkSweepPolicy();
    }
    gc_policy->initialize_all();
    // 否则，虚拟机将使用GenCollectedHeap(分代收集堆)
    Universe::_collectedHeap = new GenCollectedHeap(gc_policy);
  }
  // 接下来是相应实现的堆的初始化
  jint status = Universe::heap()->initialize();
  if (status != JNI_OK) {
    return status;
  }
```
垃圾回收策略
```
// GenCollectorPolicy继承自CollectorPolicy，表示分代内存使用的CollectorPolicy，同样定义在collectorPolicy.hpp中
class GenCollectorPolicy : public CollectorPolicy {
 protected:
  // _min_gen0_size：gen0的内存最小值
  size_t _min_gen0_size;
  // _initial_gen0_size：gen0的内存初始值
  size_t _initial_gen0_size;
  // _max_gen0_size：gen0的内存最大值
  size_t _max_gen0_size;

  // _gen_alignment and _space_alignment will have the same value most of the
  // time. When using large pages they can differ.
  // _gen_alignment：分代内存分配粒度，_gen_alignment必须被_space_alignment整除，_heap_alignment被_gen_alignment整除
  size_t _gen_alignment;
  
  // _generations一种特殊的Generation实现
  GenerationSpec **_generations;

  // Return true if an allocation should be attempted in the older
  // generation if it fails in the younger generation.  Return
  // false, otherwise.
  virtual bool should_try_older_generation_allocation(size_t word_size) const;

  void initialize_flags();
  void initialize_size_info();

  DEBUG_ONLY(void assert_flags();)
  DEBUG_ONLY(void assert_size_info();)

  // Try to allocate space by expanding the heap.
  virtual HeapWord* expand_heap_and_allocate(size_t size, bool is_tlab);

  // Compute max heap alignment
  size_t compute_max_alignment();

 // Scale the base_size by NewRatio according to
 //     result = base_size / (NewRatio + 1)
 // and align by min_alignment()
 size_t scale_by_NewRatio_aligned(size_t base_size);

 // Bound the value by the given maximum minus the min_alignment
 size_t bound_minus_alignment(size_t desired_size, size_t maximum_size);

 public:
  GenCollectorPolicy();

  // Accessors
  size_t min_gen0_size()     { return _min_gen0_size; }
  size_t initial_gen0_size() { return _initial_gen0_size; }
  size_t max_gen0_size()     { return _max_gen0_size; }
  size_t gen_alignment()     { return _gen_alignment; }

  virtual int number_of_generations() = 0;

  virtual GenerationSpec **generations() {
    assert(_generations != NULL, "Sanity check");
    return _generations;
  }

  virtual GenCollectorPolicy* as_generation_policy() { return this; }

  virtual void initialize_generations() { };

  virtual void initialize_all() {
    CollectorPolicy::initialize_all();
    // initialize_generations()根据用户参数，配置各内存代的管理器。
    initialize_generations();
  }

```

```
// CollectorPolicy的定义在hotspot/src/share/vm/memory/collectorPolicy.hpp中，该类及其子类用于定义垃圾回收器使用的全局属性，并初始化分代内存及其他共享资源。
class CollectorPolicy : public CHeapObj<mtGC> {
 protected:
 // _gc_policy_counters：跟踪分代内存的性能的计数器
  GCPolicyCounters* _gc_policy_counters;

  virtual void initialize_alignments() = 0;
  virtual void initialize_flags();
  virtual void initialize_size_info();

  DEBUG_ONLY(virtual void assert_flags();)
  DEBUG_ONLY(virtual void assert_size_info();)

  // _initial_heap_byte_size：初始堆内存 
  size_t _initial_heap_byte_size;
  // _max_heap_byte_size：最大堆内存
  size_t _max_heap_byte_size;
  // _min_heap_byte_size：最小堆内存
  size_t _min_heap_byte_size;

  // _space_alignment：space分配粒度
  size_t _space_alignment;
  // heap分配粒度，_heap_alignment必须大于_space_alignment，且是_space_alignment的整数倍
  size_t _heap_alignment;

  // Needed to keep information if MaxHeapSize was set on the command line
  // when the flag value is aligned etc by ergonomics
  // 是否通过命令行参数设置了最大堆内存
  bool _max_heap_size_cmdline;

  // The sizing of the heap are controlled by a sizing policy.
  // 用来自适应调整堆内存大小的策略实现
  AdaptiveSizePolicy* _size_policy;

  // Set to true when policy wants soft refs cleared.
  // Reset to false by gc after it clears all soft refs.
  // 是否需要清除所有的软引用，当软引用清除结束，垃圾回收器会将其置为false
  bool _should_clear_all_soft_refs;

  // Set to true by the GC if the just-completed gc cleared all
  // softrefs.  This is set to true whenever a gc clears all softrefs, and
  // set to false each time gc returns to the mutator.  For example, in the
  // ParallelScavengeHeap case the latter would be done toward the end of
  // mem_allocate() where it returns op.result()
  // 当GC刚清除完所有的软引用时会设置该属性为true，当返回mutator时被设置成false
  bool _all_soft_refs_clear;

  CollectorPolicy();

 public:
  // 构造方法调用后就会调用initialize_all方法
  virtual void initialize_all() {
    // initialize_alignments用来初始化分代内存及内存分配相关属性的，没有默认实现
    initialize_alignments();
    // initialize_flags()初始化了永久代的一些大小配置参数
    initialize_flags();
    // initialize_size_info()设置了Java堆大小的相关参数
    initialize_size_info();
  }
```

GenCollectedHeap的构造函数中使用传入的策略作为_gen_policy(代策略)。以MarkSweepPolicy为例，看看其构造函数：
```
//定义在/hotspot/src/share/vm/memory/collectorPolicy.cpp中
MarkSweepPolicy::MarkSweepPolicy() {
  initialize_all();
}
```
MarkSweepPolicy的构造函数调用了initialize_all()来完成策略的初始化，initialize_all()是父类GenCollectorPolicy()的虚函数，它调用了三个子初始化虚函数，这三个子初始化过程由GenCollectorPolicy的子类实现。其中initialize_flags()初始化了永久代的一些大小配置参数，initialize_size_info()设置了Java堆大小的相关参数，initialize_generations()根据用户参数，配置各内存代的管理器。
```
//hotspot/src/share/vm/memory/collectorPolicy.hpp中
virtual void initialize_all() {
    initialize_flags();
    initialize_size_info();
    initialize_generations();
  }
```
下面通过initialize_generations()来看看各代有哪些实现方式： 

- 若配置了UseParNewGC，并且并行GC线程数大于1，那么新生代就会使用ParNew实现
```
//永久代初始化
  _generations = new GenerationSpecPtr[number_of_generations()];
  //...

  if (UseParNewGC && ParallelGCThreads > 0) {
    _generations[0] = new GenerationSpec(Generation::ParNew, _initial_gen0_size, _max_gen0_size);
  }
```
- 默认新生代使用DefNew实现
```
else {
    _generations[0] = new GenerationSpec(Generation::DefNew, _initial_gen0_size, _max_gen0_size);
  }
```
- 老年代固定使用MarkSweepCompact实现
```
_generations[1] = new GenerationSpec(Generation::MarkSweepCompact, _initial_gen1_size, _max_gen1_size);
```
(其中DefNew、ParNew、MarkSweepCompact等均为Generation的枚举集合Name的成员，描述了可能实现的各种代实现类型)

MarkSweepPolicy、ConcurrentMarkSweepPolicy、ASConcurrentMarkSweepPolicy对各代的实现综合如下表所示： 

## 堆的初始化：堆内存空间分配
分析完了构造函数，回到Universe模块中堆的initialize()。 
以GenCollectedHeap为例： 
根据构造函数传入的gc_policy(分代策略)来初始化分代数
```
//定义在/hotspot/src/share/vm/memory/genCollectedHeap.cpp中
jint GenCollectedHeap::initialize() {
  //...
  _n_gens = gen_policy()->number_of_generations();
```
根据GenCollectedHeap的定义可以看到，GenCollectedHeap最多支持10个分代
```
 enum SomeConstants {
    max_gens = 10
  };

//...
 private:
  int _n_gens;
  Generation* _gens[max_gens];
```
其实并不需要这么多分代，MarkSweepPolicy、ConcurrentMarkSweepPolicy、ASConcurrentMarkSweepPolicy(ConcurrentMarkSweepPolicy的子类)均有着共同的祖先类TwoGenerationCollectorPolicy，其分代只有2代，即新生代和老年代。 

每代的大小是基于GenGrain大小对齐的
```
// The heap must be at least as aligned as generations.
  size_t alignment = Generation::GenGrain;
```
GenGrain定义在/hotspot/src/share/vm/memory/generation.h中，在非ARM平台中是2^16字节，即64KB大小 

获取各分代的管理器指针数组和永久代的管理器指针，并对齐各代的大小到64KB
```
PermanentGenerationSpec *perm_gen_spec =
                                collector_policy()->permanent_generation();

  // Make sure the sizes are all aligned.
  for (i = 0; i < _n_gens; i++) {
    _gen_specs[i]->align(alignment);
  }
  perm_gen_spec->align(alignment);
```
GenerationSpec的align()定义在/hotspot/src/share/vm/memory/generationSpec.h，使初始和最大大小值向上对齐至64KB的倍数
```
// Alignment
  void align(size_t alignment) {
    set_init_size(align_size_up(init_size(), alignment));
    set_max_size(align_size_up(max_size(), alignment));
  }
```
调用allocate()为堆分配空间，其起始地址为heap_address
```
char* heap_address;
  size_t total_reserved = 0;
  int n_covered_regions = 0;
  ReservedSpace heap_rs(0);

  heap_address = allocate(alignment, perm_gen_spec, &total_reserved,
                          &n_covered_regions, &heap_rs);
```

初始分配所得的空间将被封装在_reserved(CollectedHeap的MemRegion成员)中
```
_reserved = MemRegion((HeapWord*)heap_rs.base(),
                        (HeapWord*)(heap_rs.base() + heap_rs.size()));
```
调整实际的堆大小为去掉永久代的misc_data和misc_code的空间，并创建一个覆盖整个空间的数组，数组每个字节对应于堆的512字节，用于遍历新生代和老年代空间
```
 _reserved.set_word_size(0);
  _reserved.set_start((HeapWord*)heap_rs.base());
  size_t actual_heap_size = heap_rs.size() - perm_gen_spec->misc_data_size()
                                           - perm_gen_spec->misc_code_size();
  _reserved.set_end((HeapWord*)(heap_rs.base() + actual_heap_size));

  _rem_set = collector_policy()->create_rem_set(_reserved, n_covered_regions);
  set_barrier_set(rem_set()->bs());
```
调用heap_rs的的first_part()，依次为新生代和老年代分配空间并调用各代管理器的init()将其初始化为Generation空间，最后为永久代分配空间和进行初始化
```
_gch = this;

  for (i = 0; i < _n_gens; i++) {
    ReservedSpace this_rs = heap_rs.first_part(_gen_specs[i]->max_size(),
                                              UseSharedSpaces, UseSharedSpaces);
    _gens[i] = _gen_specs[i]->init(this_rs, i, rem_set());
    heap_rs = heap_rs.last_part(_gen_specs[i]->max_size());
  }
  _perm_gen = perm_gen_spec->init(heap_rs, PermSize, rem_set());
```

## 内存空间申请实现
那么GenCollectedHeap是如何向系统申请内存空间的呢？ 
答案就在allocate()函数中 
1.在申请之前，当然要对内存空间的大小和分块数进行计算 
(1).内存页的大小将根据虚拟机是否配置UseLargePages而不同，large_page_size在不同平台上表现不同，x86使用2/4M(物理地址扩展模式)的页大小，AMD64使用2M，否则，Linux默认内存页大小只有4KB，接下来会以各代所配置的最大大小进行计算，若最大值设置为负数，那么jvm将报错退出，默认的新生代和老年代的分块数为1，而永久代的分块数为2
```
char* GenCollectedHeap::allocate(size_t alignment,
                                 PermanentGenerationSpec* perm_gen_spec,
                                 size_t* _total_reserved,
                                 int* _n_covered_regions,
                                 ReservedSpace* heap_rs){
  //...
  // Now figure out the total size.
  size_t total_reserved = 0;
  int n_covered_regions = 0;
  const size_t pageSize = UseLargePages ?
      os::large_page_size() : os::vm_page_size();

  for (int i = 0; i < _n_gens; i++) {
    total_reserved += _gen_specs[i]->max_size();
    if (total_reserved < _gen_specs[i]->max_size()) {
      vm_exit_during_initialization(overflow_msg);
    }
    n_covered_regions += _gen_specs[i]->n_covered_regions();
  }
```
加上永久代空间的大小和块数
```
total_reserved += perm_gen_spec->max_size();
if (total_reserved < perm_gen_spec->max_size()) {
    vm_exit_during_initialization(overflow_msg);
  }
  n_covered_regions += perm_gen_spec->n_covered_regions();
```
加上永久代的misc_data和misc_code的空间大小(数据区和代码区)，但其实并不是堆的一部分
```
size_t s = perm_gen_spec->misc_data_size() + perm_gen_spec->misc_code_size();

  total_reserved += s;
```
如果配置了UseLargePages，那么将向上将申请的内存空间大小对齐至页
```
if (UseLargePages) {
    assert(total_reserved != 0, "total_reserved cannot be 0");
    total_reserved = round_to(total_reserved, os::large_page_size());
    if (total_reserved < os::large_page_size()) {
      vm_exit_during_initialization(overflow_msg);
    }
  }
```

对象地址压缩的内容
根据UnscaledNarrowOop(直接使用压缩指针)选取合适的堆起始地址，并尝试在该地址上分配内存
```
if (UseCompressedOops) {
      heap_address = Universe::preferred_heap_base(total_reserved, Universe::UnscaledNarrowOop);
      *_total_reserved = total_reserved;
      *_n_covered_regions = n_covered_regions;
      *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                                   UseLargePages, heap_address);
```

若不能再该地址进行分配内存，则尝试使用ZereBasedNarrowOop(零基压缩)尝试在更高的地址空间上进行分配
```
if (heap_address != NULL && !heap_rs->is_reserved()) {
        // Failed to reserve at specified address - the requested memory
        // region is taken already, for example, by 'java' launcher.
        // Try again to reserver heap higher.
        heap_address = Universe::preferred_heap_base(total_reserved, Universe::ZeroBasedNarrowOop);
        *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                                     UseLargePages, heap_address);
```
若仍然失败，则使用普通的指针压缩技术在其他地址上进行分配
```
 if (heap_address != NULL && !heap_rs->is_reserved()) {
          // Failed to reserve at specified address again - give up.
          heap_address = Universe::preferred_heap_base(total_reserved, Universe::HeapBasedNarrowOop);
          assert(heap_address == NULL, "");
          *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                                       UseLargePages, heap_address);
        }
      }
```
调用ReservedHeapSpace的构造函数进行内存空间的申请
```
  *_total_reserved = total_reserved;
  *_n_covered_regions = n_covered_regions;
  *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                               UseLargePages, heap_address);

  return heap_address;
```
在构造函数中并没有发现对内存空间进行申请，那么继续看父类ReservedSpace的构造函数
```
ReservedSpace::ReservedSpace(size_t size, size_t alignment,  
                             bool large,  
                             char* requested_address,  
                             const size_t noaccess_prefix) {  
  initialize(size+noaccess_prefix, alignment, large, requested_address,  noaccess_prefix, false);  
} 
```

initialize()的实现如下： 
如果目标操作系统不支持large_page_memory，那么将进行特殊处理，此外，对指针压缩处理还需要对请求分配的内存空间大小进行调整
```
if (requested_address != 0) {  
    requested_address -= noaccess_prefix; // adjust requested address  
    assert(requested_address != NULL, "huge noaccess prefix?");  
  } 
```
对于上述特殊情况，会调用reserve_memory_special()进行内存空间的申请，并若申请成功会进行空间大小的对齐验证
```
if (special) {  

    //向操作系统申请指定大小的内存，并映射到用户指定的内存空间中  
    base = os::reserve_memory_special(size, requested_address, executable);  

    if (base != NULL) {  
      if (failed_to_reserve_as_requested(base, requested_address, size, true)) {  
        // OS ignored requested address. Try different address.  
        return;  
      }  
      // Check alignment constraints  
      assert((uintptr_t) base % alignment == 0, "Large pages returned a non-aligned address");  
      _special = true; 
```
若配置了UseSharedSpace或UseCompressedOops，那么堆将在指定地址进行申请，就会调用attempt_reserve_memory_at()进行申请，否则，调用reserve_memory()进行申请
```
if (requested_address != 0) {  
      base = os::attempt_reserve_memory_at(size, requested_address);  

      if (failed_to_reserve_as_requested(base, requested_address, size, false)) {  
        // OS ignored requested address. Try different address.  
        base = NULL;  
      }  
    } else {  
      base = os::reserve_memory(size, NULL, alignment);  
    }  
```
若分配成功，还需要对分配的起始地址进行对齐验证。若没有对齐，则会进行手工调整。调整的方法为尝试申请一块size+alignment大小的空间，若成功则向上对齐所得的内存空间的起始地址(失败则无法对齐，直接返回)，并以此为起始地址重新申请一块size大小的空间，这块size大小的空间必然包含于size+alignment大小的空间内，以此达到对齐地址的目的。
```
// Check alignment constraints  
    if ((((size_t)base + noaccess_prefix) & (alignment - 1)) != 0) {  
      // Base not aligned, retry  
      if (!os::release_memory(base, size)) fatal("os::release_memory failed");  
      // Reserve size large enough to do manual alignment and  
      // increase size to a multiple of the desired alignment  
      size = align_size_up(size, alignment);  
      size_t extra_size = size + alignment;  
      do {  
        char* extra_base = os::reserve_memory(extra_size, NULL, alignment);  
        if (extra_base == NULL) return;  

        // Do manual alignement  
        base = (char*) align_size_up((uintptr_t) extra_base, alignment);  
        assert(base >= extra_base, "just checking");  
        // Re-reserve the region at the aligned base address.  
        os::release_memory(extra_base, extra_size);  
        base = os::reserve_memory(size, base);  
      } while (base == NULL); 
```
最后，在地址空间均已分配完毕，GenCollectedHeap的initialize()中为各代划分了各自的内存空间范围，就会调用各代的GenerationSpec的init()函数完成各代的初始化。
```
switch (name()) {
    case PermGen::MarkSweepCompact:
      return new CompactingPermGen(perm_rs, shared_rs, init_size, remset, this);

#ifndef SERIALGC
    case PermGen::MarkSweep:
      guarantee(false, "NYI");
      return NULL;

    case PermGen::ConcurrentMarkSweep: {
      assert(UseConcMarkSweepGC, "UseConcMarkSweepGC should be set");
      CardTableRS* ctrs = remset->as_CardTableRS();
      if (ctrs == NULL) {
        vm_exit_during_initialization("RemSet/generation incompatibility.");
      }
      // XXXPERM
      return new CMSPermGen(perm_rs, init_size, ctrs,
                   (FreeBlockDictionary::DictionaryChoice)CMSDictionaryChoice);
    }
#endif // SERIALGC
    default:
      guarantee(false, "unrecognized GenerationName");
      return NULL;
  }
```

## 堆的分配实现
mem_allocate将由堆的实现类型定义，以GenCollectedHeap为例：
```
HeapWord* GenCollectedHeap::mem_allocate(size_t size, 
                                         bool is_large_noref, 
                                         bool is_tlab, 
                                         bool* gc_overhead_limit_was_exceeded) { 
  return collector_policy()->mem_allocate_work(size, 
                                               is_tlab, 
                                               gc_overhead_limit_was_exceeded); 
} 
```
由之前分析，GenCollectedHeap根据用户配置有着不同的GC策略(默认的和配置UseSerialGC的 MarkSweepPolicy、配置UseComcMarkSweepGC和UseAdaptiveSizePolicy的 ASConcurrentMarkSweepPolicy、只配置UseComcMarkSweepGC的 ConcurrentMarkSweepPolicy)，但这里，对象内存空间的基本结构和分配的思想是一致的，所以统一由 GenCollectorPolicy实现进行分代层级的对象分配操作，但具体的工作将交由各代的实现者来完成。

GenCollectedPolicy的mem_allocate_work()函数如下：
gch指向GenCollectedHeap堆，内存分配请求将循环不断地进行尝试，直到分配成功或GC后分配失败
```
HeapWord* GenCollectorPolicy::mem_allocate_work(size_t size, 
                                        bool is_tlab, 
                                        bool* gc_overhead_limit_was_exceeded) { 
  GenCollectedHeap *gch = GenCollectedHeap::heap(); 
  //... 
  // Loop until the allocation is satisified, 
  // or unsatisfied after GC. 
  for (int try_count = 1; /* return or throw */; try_count += 1) { 
```
对于占用空间比较大的对象，如果经常放在新生代，那么剩余的内存空间就会非常紧张，将可能会导致新生代内存垃圾回收的频繁触发。故若对象的大小超过一定值，那么就不应该分配在新生代。
```
//...紧接上面部分 
dleMark hm; // discard any handles allocated in each iteration 
 
 // First allocation attempt is lock-free. 
 Generation *gen0 = gch->get_gen(0); 
 
 if (gen0->should_allocate(size, is_tlab)) { 
   result = gen0->par_allocate(size, is_tlab); 
   if (result != NULL) { 
     assert(gch->is_in_reserved(result), "result not in heap"); return result; 
   } 
 } 
```
若对象应该在新生代上分配，就会调用新生代的par_allocate()进行分配，注意在新生代普遍是采用复制收集器的，而内存的分配对应采用了无锁式的指针碰撞技术。

在新生代上尝试无锁式的分配失败，那么就获取堆的互斥锁，并尝试在各代空间内进行内存分配
```
unsigned int gc_count_before;  // read inside the Heap_lock locked region 
    { 
      MutexLocker ml(Heap_lock); 
     //... 
      bool first_only = ! should_try_older_generation_allocation(size); 
 
      result = gch->attempt_allocation(size, is_tlab, first_only); 
      if (result != NULL) { 
        assert(gch->is_in_reserved(result), "result not in heap"); return result; 
      } 
```
其中should_try_older_generation_allocation()如下：
```
bool GenCollectorPolicy::should_try_older_generation_allocation( 
        size_t word_size) const { 
  GenCollectedHeap* gch = GenCollectedHeap::heap(); 
  size_t gen0_capacity = gch->get_gen(0)->capacity_before_gc(); 
  return    (word_size > heap_word_size(gen0_capacity)) 
         || GC_locker::is_active_and_needs_gc() 
         || gch->incremental_collection_failed(); 
} 
```
当进行gc前，新生代的空闲空间大小不足以分配对象，或者有线程触发了gc，或前一次的FullGC是由MinorGC触发的情况，都应该不再尝试再更高的内存代上进行分配，以保证新分配的对象尽可能在新生代空间上。 

attempt_allocation()实现如下：
```
HeapWord* GenCollectedHeap::attempt_allocation(size_t size, 
                                               bool is_tlab, 
                                               bool first_only) { 
  HeapWord* res; 
  for (int i = 0; i < _n_gens; i++) { 
    if (_gens[i]->should_allocate(size, is_tlab)) { 
      res = _gens[i]->allocate(size, is_tlab); 
      if (res != NULL) return res; 
      else if (first_only) break; 
    } 
  } 
  // Otherwise... 
  return NULL; 
} 
```
即由低内存代向高内存代尝试分配内存 

从各个代空间都找不到可用的空闲内存(或不应该在更高的内存代上分配时)，如果已经有线程触发了gc，那么当各代空间还有virtual space可扩展空间可用时，将会尝试扩展代空间并再次尝试进行内存分配，有点在gc前想尽一切办法获得内存的意思。

```
if (GC_locker::is_active_and_needs_gc()) { 
        if (is_tlab) { 
          return NULL;  // Caller will retry allocating individual object 
        } 
        if (!gch->is_maximal_no_gc()) { 
          // Try and expand heap to satisfy request 
          result = expand_heap_and_allocate(size, is_tlab); 
          // result could be null if we are out of space 
          if (result != NULL) { 
            return result; 
          } 
        } 
```
否则各代已经没有可用的可扩展空间时，当当前线程没有位于jni的临界区时，将释放堆的互斥锁，以使得请求gc的线程可以进行gc操作，等待所有本地线程退出临界区和gc完成后，将继续循环尝试进行对象的内存分配
```
JavaThread* jthr = JavaThread::current(); 
        if (!jthr->in_critical()) { 
          MutexUnlocker mul(Heap_lock); 
          // Wait for JNI critical section to be exited 
          GC_locker::stall_until_clear(); 
          continue; 
        } 
```
若各代无法分配对象的内存，并且没有gc被触发，那么当前请求内存分配的线程将发起一次gc，这里将提交给VM一个 GenCollectForAllocation操作以触发gc，当操作执行成功并返回时，若gc锁已被获得，那么说明已经由其他线程触发了gc，将继续 循环以等待gc完成
```
VM_GenCollectForAllocation op(size, 
                                  is_tlab, 
                                  gc_count_before); 
    VMThread::execute(&op); 
    if (op.prologue_succeeded()) { 
      result = op.result(); 
      if (op.gc_locked()) { 
         assert(result == NULL, "must be NULL if gc_locked() is true"); continue; // retry and/or stall as necessary 
      } 
```
否则将等待gc完成，若gc超时则会将gc_overhead_limit_was_exceeded设置为true返回给调用者，并重置超时状态，并对分配的对象进行填充处理
```
const bool limit_exceeded = size_policy()->gc_overhead_limit_exceeded(); 
  const bool softrefs_clear = all_soft_refs_clear(); 
  assert(!limit_exceeded || softrefs_clear, "Should have been cleared"); if (limit_exceeded && softrefs_clear) { *gc_overhead_limit_was_exceeded = true; size_policy()->set_gc_overhead_limit_exceeded(false); if (op.result() != NULL) { CollectedHeap::fill_with_object(op.result(), size); } return NULL; 
  } 
```