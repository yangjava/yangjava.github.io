---
layout: post
categories: Synchronized
description: none
keywords: Synchronized
---
# 死磕Synchronized底层实现
本系列文章将对HotSpot的synchronized锁实现进行全面分析，内容包括偏向锁、轻量级锁、重量级锁的加锁、解锁、锁升级流程的原理及源码分析，希望给在研究synchronized路上的同学一些帮助。

本篇文章将对synchronized机制做个大致的介绍，包括用以承载锁状态的对象头、锁的几种形式、各种形式锁的加锁和解锁流程、什么时候会发生锁升级。需要注意的是本文旨在介绍背景和概念，在讲述一些流程的时候，只提到了主要case，对于实现细节、运行时的不同分支都在后面的文章中详细分析。

## synchronized简介

Java中提供了两种实现同步的基础语义：synchronized方法和synchronized块， 我们来看个demo：

```java
public class SyncTest {
    public void syncBlock(){
        synchronized (this){
            System.out.println("hello block");
        }
    }
    public synchronized void syncMethod(){
        System.out.println("hello method");
    }
}
```
当SyncTest.java被编译成class文件的时候，synchronized关键字和synchronized方法的字节码略有不同，我们可以用javap -v 命令查看class文件对应的JVM字节码信息，部分信息如下：
```java
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit						  // monitorexit指令退出同步块
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 

  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
 
}
```
从上面的中文注释处可以看到，对于synchronized关键字而言，javac在编译时，会生成对应的monitorenter和monitorexit指令分别对应synchronized同步块的进入和退出，有两个monitorexit指令的原因是：为了保证抛异常的情况下也能释放锁，所以javac为同步代码块添加了一个隐式的try-finally，在finally中会调用monitorexit命令释放锁。而对于synchronized方法而言，javac为其生成了一个ACC_SYNCHRONIZED关键字，在JVM进行方法调用时，发现调用的方法被ACC_SYNCHRONIZED修饰，则会先尝试获得锁。

在JVM底层，对于这两种synchronized语义的实现大致相同，在后文中会选择一种进行详细分析。

因为本文旨在分析synchronized的实现原理，因此对于其使用的一些问题就不赘述了，不了解的朋友可以看看这篇文章。

## 锁的几种形式

传统的锁（也就是下文要说的重量级锁）依赖于系统的同步函数，在linux上使用mutex互斥锁，最底层实现依赖于futex，关于futex可以看我之前的文章，这些同步函数都涉及到用户态和内核态的切换、进程的上下文切换，成本较高。对于加了synchronized关键字但运行时并没有多线程竞争，或两个线程接近于交替执行的情况，使用传统锁机制无疑效率是会比较低的。

在JDK 1.6之前,synchronized只有传统的锁机制，因此给开发者留下了synchronized关键字相比于其他同步机制性能不好的印象。

在JDK 1.6引入了两种新型锁机制：偏向锁和轻量级锁，它们的引入是为了解决在没有多线程竞争或基本没有竞争的场景下因使用传统锁机制带来的性能开销问题。

在看这几种锁机制的实现前，我们先来了解下对象头，它是实现多种锁机制的基础。

## 对象头
因为在Java中任意对象都可以用作锁，因此必定要有一个映射关系，存储该对象以及其对应的锁信息（比如当前哪个线程持有锁，哪些线程在等待）。一种很直观的方法是，用一个全局map，来存储这个映射关系，但这样会有一些问题：需要对map做线程安全保障，不同的synchronized之间会相互影响，性能差；另外当同步对象较多时，该map可能会占用比较多的内存。

所以最好的办法是将这个映射关系存储在对象头中，因为对象头本身也有一些hashcode、GC相关的数据，所以如果能将锁信息与这些信息共存在对象头中就好了。

在JVM中，对象在内存中除了本身的数据外还会有个对象头，对于普通对象而言，其对象头中有两类信息：mark word和类型指针。另外对于数组而言还会有一份记录数组长度的数据。

类型指针是指向该对象所属类对象的指针，mark word用于存储对象的HashCode、GC分代年龄、锁状态等信息。在32位系统上mark word长度为32bit，64位系统上长度为64bit。为了能在有限的空间里存储下更多的数据，其存储格式是不固定的

可以看到锁信息也是存在于对象的mark word中的。当对象状态为偏向锁（biasable）时，mark word存储的是偏向的线程ID；当状态为轻量级锁（lightweight locked）时，mark word存储的是指向线程栈中Lock Record的指针；当状态为重量级锁（inflated）时，为指向堆中的monitor对象的指针。

## 重量级锁

重量级锁是我们常说的传统意义上的锁，其利用操作系统底层的同步机制去实现Java中的线程同步。

重量级锁的状态下，对象的mark word为指向一个堆中monitor对象的指针。

一个monitor对象包括这么几个关键字段：cxq（下图中的ContentionList），EntryList ，WaitSet，owner。

其中cxq ，EntryList ，WaitSet都是由ObjectWaiter的链表结构，owner指向持有锁的线程。

当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个ObjectWaiter对象插入到cxq的队列尾部，然后暂停当前线程。当持有锁的线程释放锁前，会将cxq中的所有元素移动到EntryList中去，并唤醒EntryList的队首线程。

如果一个线程在同步块中调用了Object#wait方法，会将该线程对应的ObjectWaiter从EntryList移除并加入到WaitSet中，然后释放锁。当wait的线程被notify之后，会将对应的ObjectWaiter从WaitSet移动到EntryList中。

以上只是对重量级锁流程的一个简述，其中涉及到的很多细节，比如ObjectMonitor对象从哪来？释放锁时是将cxq中的元素移动到EntryList的尾部还是头部？notfiy时，是将ObjectWaiter移动到EntryList的尾部还是头部？

关于具体的细节，会在重量级锁的文章中分析。

## 轻量级锁

JVM的开发者发现在很多情况下，在Java程序运行时，同步块中的代码都是不存在竞争的，不同的线程交替的执行同步块中的代码。这种情况下，用重量级锁是没必要的。因此JVM引入了轻量级锁的概念。

线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个Lock Record，其包括一个用于存储对象头中的 mark word（官方称之为Displaced Mark Word）以及一个指向对象的指针。下图右边的部分就是一个Lock Record。

加锁过程
1.在线程栈中创建一个Lock Record，将其obj（即上图的Object reference）字段指向锁对象。

2.直接通过CAS指令将Lock Record的地址存储在对象头的mark word中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，进入到步骤3。

3.如果是当前线程已经持有该锁了，代表这是一次锁重入。设置Lock Record第一部分（Displaced Mark Word）为null，起到了一个重入计数器的作用。然后结束。

4.走到这一步说明发生了竞争，需要膨胀为重量级锁。

解锁过程
1.遍历线程栈,找到所有obj字段等于当前锁对象的Lock Record。

2.如果Lock Record的Displaced Mark Word为null，代表这是一次重入，将obj设置为null后continue。

3.如果Lock Record的Displaced Mark Word不为null，则利用CAS指令将对象头的mark word恢复成为Displaced Mark Word。如果成功，则continue，否则膨胀为重量级锁。


## 偏向锁

Java是支持多线程的语言，因此在很多二方包、基础库中为了保证代码在多线程的情况下也能正常运行，也就是我们常说的线程安全，都会加入如synchronized这样的同步语义。但是在应用在实际运行时，很可能只有一个线程会调用相关同步方法。比如下面这个demo：
```java
import java.util.ArrayList;
import java.util.List;

public class SyncDemo1 {

    public static void main(String[] args) {
        SyncDemo1 syncDemo1 = new SyncDemo1();
        for (int i = 0; i < 100; i++) {
            syncDemo1.addString("test:" + i);
        }
    }

    private List<String> list = new ArrayList<>();

    public synchronized void addString(String s) {
        list.add(s);
    }

}
```

在这个demo中为了保证对list操纵时线程安全，对addString方法加了synchronized的修饰，但实际使用时却只有一个线程调用到该方法，对于轻量级锁而言，每次调用addString时，加锁解锁都有一个CAS操作；对于重量级锁而言，加锁也会有一个或多个CAS操作（这里的’一个‘、’多个‘数量词只是针对该demo，并不适用于所有场景）。

在JDK1.6中为了提高一个对象在一段很长的时间内都只被一个线程用做锁对象场景下的性能，引入了偏向锁，在第一次获得锁时，会有一个CAS操作，之后该线程再获取锁，只会执行几个简单的命令，而不是开销相对较大的CAS命令。我们来看看偏向锁是如何做的。


对象创建
当JVM启用了偏向锁模式（1.6以上默认开启），当新创建一个对象的时候，如果该对象所属的class没有关闭偏向锁模式（什么时候会关闭一个class的偏向模式下文会说，默认所有class的偏向模式都是是开启的），那新创建对象的mark word将是可偏向状态，此时mark word中的thread id（参见上文偏向状态下的mark word格式）为0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。

加锁过程
case 1：当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，则会用CAS指令，将mark word中的thread id由0改成当前线程Id。如果成功，则代表获得了偏向锁，继续执行同步块中的代码。否则，将偏向锁撤销，升级为轻量级锁。

case 2：当被偏向的线程再次进入同步块时，发现锁对象偏向的就是当前线程，在通过一些额外的检查后（细节见后面的文章），会往当前线程的栈中添加一条Displaced Mark Word为空的Lock Record中，然后继续执行同步块的代码，因为操纵的是线程私有的栈，因此不需要用到CAS指令；由此可见偏向锁模式下，当被偏向的线程再次尝试获得锁时，仅仅进行几个简单的操作就可以了，在这种情况下，synchronized关键字带来的性能开销基本可以忽略。

case 3.当其他线程进入同步块时，发现已经有偏向的线程了，则会进入到撤销偏向锁的逻辑里，一般来说，会在safepoint中去查看偏向的线程是否还存活，如果存活且还在同步块中则将锁升级为轻量级锁，原偏向的线程继续拥有锁，当前线程则走入到锁升级的逻辑里；如果偏向的线程已经不存活或者不在同步块中，则将对象头的mark word改为无锁状态（unlocked），之后再升级为轻量级锁。

由此可见，偏向锁升级的时机为：当锁已经发生偏向后，只要有另一个线程尝试获得偏向锁，则该偏向锁就会升级成轻量级锁。当然这个说法不绝对，因为还有批量重偏向这一机制。

解锁过程
当有其他线程尝试获得锁时，是根据遍历偏向线程的lock record来确定该线程是否还在执行同步块中的代码。因此偏向锁的解锁很简单，仅仅将栈中的最近一条lock record的obj字段设置为null。需要注意的是，偏向锁的解锁步骤中并不会修改对象头中的thread id。

下图展示了锁状态的转换流程：

img

另外，偏向锁默认不是立即就启动的，在程序启动后，通常有几秒的延迟，可以通过命令 -XX:BiasedLockingStartupDelay=0来关闭延迟。

批量重偏向与撤销
从上文偏向锁的加锁解锁过程中可以看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时将偏向锁撤销为无锁状态或升级为轻量级/重量级锁。safe point这个词我们在GC中经常会提到，其代表了一个状态，在该状态下所有线程都是暂停的（大概这么个意思），详细可以看这篇文章。总之，偏向锁的撤销是有一定成本的，如果说运行时的场景本身存在多线程竞争的，那偏向锁的存在不仅不能提高性能，而且会导致性能下降。因此，JVM中增加了一种批量重偏向/撤销的机制。

存在如下两种情况：（见官方论文第4小节）:

1.一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁进行之后的操作。这种case下，会导致大量的偏向锁撤销操作。

2.存在明显多线程竞争的场景下使用偏向锁是不合适的，例如生产者/消费者队列。

批量重偏向（bulk rebias）机制是为了解决第一种场景。批量撤销（bulk revoke）则是为了解决第二种场景。

其做法是：以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。每个class对象会有一个对应的epoch字段，每个处于偏向锁状态对象的mark word中也有该字段，其初始值为创建该对象时，class中的epoch的值。每次发生批量重偏向时，就将该值+1，同时遍历JVM中所有线程的栈，找到该class所有正处于加锁状态的偏向锁，将其epoch字段改为新值。下次获得锁时，发现当前对象的epoch值和class的epoch不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其mark word的Thread Id 改成当前线程Id。

当达到重偏向阈值后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40），JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。

End
Java中的synchronized有偏向锁、轻量级锁、重量级锁三种形式，分别对应了锁只被一个线程持有、不同线程交替持有锁、多线程竞争锁三种情况。当条件不满足时，锁会按偏向锁->轻量级锁->重量级锁 的顺序升级。JVM种的锁也是能降级的，只不过条件很苛刻，不在我们讨论范围之内。该篇文章主要是对Java的synchronized做个基本介绍，后文会有更详细的分析。


# 死磕Synchronized底层实现--偏向锁
偏向锁入口
要找锁的入口，肯定是要在源码中找到对monitorenter指令解析的地方。在HotSpot的中有两处地方对monitorenter指令进行解析：一个是在bytecodeInterpreter.cpp#1816 ，另一个是在templateTable_x86_64.cpp#3667。
偏向锁获取流程

下面开始偏向锁获取流程分析，代码在bytecodeInterpreter.cpp#1816。

```c++
CASE(_monitorenter): {
  // lockee 就是锁对象
  oop lockee = STACK_OBJECT(-1);
  // derefing's lockee ought to provoke implicit null check
  CHECK_NULL(lockee);
  // code 1：找到一个空闲的Lock Record
  BasicObjectLock* limit = istate->monitor_base();
  BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
  BasicObjectLock* entry = NULL;
  while (most_recent != limit ) {
    if (most_recent->obj() == NULL) entry = most_recent;
    else if (most_recent->obj() == lockee) break;
    most_recent++;
  }
  //entry不为null，代表还有空闲的Lock Record
  if (entry != NULL) {
    // code 2：将Lock Record的obj指针指向锁对象
    entry->set_obj(lockee);
    int success = false;
    uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;
	// markoop即对象头的mark word
    markOop mark = lockee->mark();
    intptr_t hash = (intptr_t) markOopDesc::no_hash;
    // code 3：如果锁对象的mark word的状态是偏向模式
    if (mark->has_bias_pattern()) {
      uintptr_t thread_ident;
      uintptr_t anticipated_bias_locking_value;
      thread_ident = (uintptr_t)istate->thread();
     // code 4：这里有几步操作，下文分析
      anticipated_bias_locking_value =
        (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) &
        ~((uintptr_t) markOopDesc::age_mask_in_place);
	 // code 5：如果偏向的线程是自己且epoch等于class的epoch
      if  (anticipated_bias_locking_value == 0) {
        // already biased towards this thread, nothing to do
        if (PrintBiasedLockingStatistics) {
          (* BiasedLocking::biased_lock_entry_count_addr())++;
        }
        success = true;
      }
       // code 6：如果偏向模式关闭，则尝试撤销偏向锁
      else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
        markOop header = lockee->klass()->prototype_header();
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        // 利用CAS操作将mark word替换为class中的mark word
        if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (*BiasedLocking::revoked_lock_entry_count_addr())++;
        }
      }
         // code 7：如果epoch不等于class中的epoch，则尝试重偏向
      else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
        // 构造一个偏向当前线程的mark word
        markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
        if (hash != markOopDesc::no_hash) {
          new_header = new_header->copy_set_hash(hash);
        }
        // CAS替换对象头的mark word  
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (* BiasedLocking::rebiased_lock_entry_count_addr())++;
        }
        else {
          // 重偏向失败，代表存在多线程竞争，则调用monitorenter方法进行锁升级
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
        success = true;
      }
      else {
         // 走到这里说明当前要么偏向别的线程，要么是匿名偏向（即没有偏向任何线程）
       	// code 8：下面构建一个匿名偏向的mark word，尝试用CAS指令替换掉锁对象的mark word
        markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |(uintptr_t)markOopDesc::age_mask_in_place |epoch_mask_in_place));
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
        // debugging hint
        DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
           // CAS修改成功
          if (PrintBiasedLockingStatistics)
            (* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
        }
        else {
          // 如果修改失败说明存在多线程竞争，所以进入monitorenter方法
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
        success = true;
      }
    }

    // 如果偏向线程不是当前线程或没有开启偏向模式等原因都会导致success==false
    if (!success) {
      // 轻量级锁的逻辑
      //code 9: 构造一个无锁状态的Displaced Mark Word，并将Lock Record的lock指向它
      markOop displaced = lockee->mark()->set_unlocked();
      entry->lock()->set_displaced_header(displaced);
      //如果指定了-XX:+UseHeavyMonitors，则call_vm=true，代表禁用偏向锁和轻量级锁
      bool call_vm = UseHeavyMonitors;
      // 利用CAS将对象头的mark word替换为指向Lock Record的指针
      if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
        // 判断是不是锁重入
        if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {		//code 10: 如果是锁重入，则直接将Displaced Mark Word设置为null
          entry->lock()->set_displaced_header(NULL);
        } else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
      }
    }
    UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
  } else {
    // lock record不够，重新执行
    istate->set_msg(more_monitors);
    UPDATE_PC_AND_RETURN(0); // Re-execute
  }
}
```
JVM中的每个类也有一个类似mark word的prototype_header，用来标记该class的epoch和偏向开关等信息。上面的代码中lockee->klass()->prototype_header()即获取class的prototype_header。

code 1，从当前线程的栈中找到一个空闲的Lock Record（即代码中的BasicObjectLock，下文都用Lock Record代指），判断Lock Record是否空闲的依据是其obj字段 是否为null。注意这里是按内存地址从低往高找到最后一个可用的Lock Record，换而言之，就是找到内存地址最高的可用Lock Record。

code 2，获取到Lock Record后，首先要做的就是为其obj字段赋值。

code 3，判断锁对象的mark word是否是偏向模式，即低3位是否为101。

code 4，这里有几步位运算的操作  anticipated_bias_locking_value = (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) & ​        ~((uintptr_t) markOopDesc::age_mask_in_place); 这个位运算可以分为3个部分。

第一部分((uintptr_t)lockee->klass()->prototype_header() | thread_ident) 将当前线程id和类的prototype_header相或，这样得到的值为（当前线程id + prototype_header中的（epoch + 分代年龄 + 偏向锁标志 + 锁标志位）），注意prototype_header的分代年龄那4个字节为0

第二部分 ^ (uintptr_t)mark 将上面计算得到的结果与锁对象的markOop进行异或，相等的位全部被置为0，只剩下不相等的位。

第三部分  & ~((uintptr_t) markOopDesc::age_mask_in_place) markOopDesc::age_mask_in_place为...0001111000,取反后，变成了...1110000111,除了分代年龄那4位，其他位全为1；将取反后的结果再与上面的结果相与，将上面异或得到的结果中分代年龄给忽略掉。

code 5，anticipated_bias_locking_value==0代表偏向的线程是当前线程且mark word的epoch等于class的epoch，这种情况下什么都不用做。

code 6，(anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0代表class的prototype_header或对象的mark word中偏向模式是关闭的，又因为能走到这已经通过了mark->has_bias_pattern()判断，即对象的mark word中偏向模式是开启的，那也就是说class的prototype_header不是偏向模式。

然后利用CAS指令Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark撤销偏向锁，我们知道CAS会有几个参数，1是预期的原值，2是预期修改后的值 ，3是要修改的对象，与之对应，cmpxchg_ptr方法第一个参数是预期修改后的值，第2个参数是修改的对象，第3个参数是预期原值，方法返回实际原值，如果等于预期原值则说明修改成功。

code 7，如果epoch已过期，则需要重偏向，利用CAS指令将锁对象的mark word替换为一个偏向当前线程且epoch为类的epoch的新的mark word。

code 8，CAS将偏向线程改为当前线程，如果当前是匿名偏向则能修改成功，否则进入锁升级的逻辑。

code 9，这一步已经是轻量级锁的逻辑了。从上图的mark word的格式可以看到，轻量级锁中mark word存的是指向Lock Record的指针。这里构造一个无锁状态的mark word，然后存储到Lock Record（Lock Record的格式可以看第一篇文章）。设置mark word是无锁状态的原因是：轻量级锁解锁时是将对象头的mark word设置为Lock Record中的Displaced Mark Word，所以创建时设置为无锁状态，解锁时直接用CAS替换就好了。

code 10， 如果是锁重入，则将Lock Record的Displaced Mark Word设置为null，起到一个锁重入计数的作用。

以上是偏向锁加锁的流程（包括部分轻量级锁的加锁流程），如果当前锁已偏向其他线程||epoch值过期||偏向模式关闭||获取偏向锁的过程中存在并发冲突，都会进入到InterpreterRuntime::monitorenter方法， 在该方法中会对偏向锁撤销和升级。

## 偏向锁的撤销
这里说的撤销是指在获取偏向锁的过程因为不满足条件导致要将锁对象改为非偏向锁状态；释放是指退出同步块时的过程，释放锁的逻辑会在下一小节阐述。请读者注意本文中撤销与释放的区别。

如果获取偏向锁失败会进入到InterpreterRuntime::monitorenter方法

```C++
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  ...
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  ...
IRT_END
```
可以看到如果开启了JVM偏向锁，那会进入到ObjectSynchronizer::fast_enter方法中。
```C++
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }

 slow_enter (obj, lock, THREAD) ;
}
```
如果是正常的Java线程，会走上面的逻辑进入到BiasedLocking::revoke_and_rebias方法，如果是VM线程则会走到下面的BiasedLocking::revoke_at_safepoint。我们主要看BiasedLocking::revoke_and_rebias方法。这个方法的主要作用像它的方法名：撤销或者重偏向，第一个参数封装了锁对象和当前线程，第二个参数代表是否允许重偏向，这里是true。
```C++
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
    
  markOop mark = obj->mark();
  if (mark->is_biased_anonymously() && !attempt_rebias) {
     //如果是匿名偏向且attempt_rebias==false会走到这里，如锁对象的hashcode方法被调用会出现这种情况，需要撤销偏向锁。
    markOop biased_value       = mark;
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
    markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
    if (res_mark == biased_value) {
      return BIAS_REVOKED;
    }
  } else if (mark->has_bias_pattern()) {
    // 锁对象开启了偏向模式会走到这里
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    //code 1： 如果对应class关闭了偏向模式
    if (!prototype_header->has_bias_pattern()) {
      markOop biased_value       = mark;
      markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
      assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;
    //code2： 如果epoch过期
    } else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
      if (attempt_rebias) {
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED;
        }
      } else {
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED;
        }
      }
    }
  }
  //code 3：批量重偏向与批量撤销的逻辑
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  if (heuristics == HR_NOT_BIASED) {
    return NOT_BIASED;
  } else if (heuristics == HR_SINGLE_REVOKE) {
    //code 4：撤销单个线程
    Klass *k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {
      // 走到这里说明需要撤销的是偏向当前线程的锁，当调用Object#hashcode方法时会走到这一步
      // 因为只要遍历当前线程的栈就好了，所以不需要等到safepoint再撤销。
      ResourceMark rm;
      if (TraceBiasedLocking) {
        tty->print_cr("Revoking bias by walking my own stack:");
      }
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD);
      ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
      assert(cond == BIAS_REVOKED, "why not?");
      return cond;
    } else {
      // 下面代码最终会在VM线程中的safepoint调用revoke_bias方法
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      return revoke.status_code();
    }
  }
	
  assert((heuristics == HR_BULK_REVOKE) ||
         (heuristics == HR_BULK_REBIAS), "?");
   //code5：批量撤销、批量重偏向的逻辑
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  return bulk_revoke.status_code();
}
```
会走到该方法的逻辑有很多，我们只分析最常见的情况：假设锁已经偏向线程A，这时B线程尝试获得锁。

上面的code 1，code 2B线程都不会走到，最终会走到code 4处，如果要撤销的锁偏向的是当前线程则直接调用revoke_bias撤销偏向锁，否则会将该操作push到VM Thread中等到safepoint的时候再执行。

关于VM Thread这里介绍下：在JVM中有个专门的VM Thread，该线程会源源不断的从VMOperationQueue中取出请求，比如GC请求。对于需要safepoint的操作（VM_Operationevaluate_at_safepoint返回true）必须要等到所有的Java线程进入到safepoint才开始执行。 关于safepoint可以参考下这篇文章。

接下来我们着重分析下revoke_bias方法。第一个参数为锁对象，第2、3个参数为都为false

```C++
static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread) {
  markOop mark = obj->mark();
  // 如果没有开启偏向模式，则直接返回NOT_BIASED
  if (!mark->has_bias_pattern()) {
    ...
    return BiasedLocking::NOT_BIASED;
  }

  uint age = mark->age();
  // 构建两个mark word，一个是匿名偏向模式（101），一个是无锁模式（001）
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);

  ...

  JavaThread* biased_thread = mark->biased_locker();
  if (biased_thread == NULL) {
     // 匿名偏向。当调用锁对象的hashcode()方法可能会导致走到这个逻辑
     // 如果不允许重偏向，则将对象的mark word设置为无锁模式
    if (!allow_rebias) {
      obj->set_mark(unbiased_prototype);
    }
    ...
    return BiasedLocking::BIAS_REVOKED;
  }

  // code 1：判断偏向线程是否还存活
  bool thread_is_alive = false;
  // 如果当前线程就是偏向线程 
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  } else {
     // 遍历当前jvm的所有线程，如果能找到，则说明偏向的线程还存活
    for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
      if (cur_thread == biased_thread) {
        thread_is_alive = true;
        break;
      }
    }
  }
  // 如果偏向的线程已经不存活了
  if (!thread_is_alive) {
    // 允许重偏向则将对象mark word设置为匿名偏向状态，否则设置为无锁状态
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      obj->set_mark(unbiased_prototype);
    }
    ...
    return BiasedLocking::BIAS_REVOKED;
  }

  // 线程还存活则遍历线程栈中所有的Lock Record
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    // 如果能找到对应的Lock Record说明偏向的线程还在执行同步代码块中的代码
    if (mon_info->owner() == obj) {
      ...
      // 需要升级为轻量级锁，直接修改偏向线程栈中的Lock Record。为了处理锁重入的case，在这里将Lock Record的Displaced Mark Word设置为null，第一个Lock Record会在下面的代码中再处理
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header(mark);
    } else {
      ...
    }
  }
  if (highest_lock != NULL) {
    // 修改第一个Lock Record为无锁状态，然后将obj的mark word设置为指向该Lock Record的指针
    highest_lock->set_displaced_header(unbiased_prototype);
    obj->release_set_mark(markOopDesc::encode(highest_lock));
    ...
  } else {
    // 走到这里说明偏向线程已经不在同步块中了
    ...
    if (allow_rebias) {
       //设置为匿名偏向状态
      obj->set_mark(biased_prototype);
    } else {
      // 将mark word设置为无锁状态
      obj->set_mark(unbiased_prototype);
    }
  }

  return BiasedLocking::BIAS_REVOKED;
}

```
需要注意下，当调用锁对象的Object#hash或System.identityHashCode()方法会导致该对象的偏向锁或轻量级锁升级。这是因为在Java中一个对象的hashcode是在调用这两个方法时才生成的，如果是无锁状态则存放在mark word中，如果是重量级锁则存放在对应的monitor中，而偏向锁是没有地方能存放该信息的，所以必须升级。具体可以看这篇文章的hashcode()方法对偏向锁的影响小节（注意：该文中对于偏向锁的加锁描述有些错误），另外我也向该文章作者请教过一些问题，他很热心的回答了我，在此感谢一下！

言归正传，revoke_bias方法逻辑：

查看偏向的线程是否存活，如果已经不存活了，则直接撤销偏向锁。JVM维护了一个集合存放所有存活的线程，通过遍历该集合判断某个线程是否存活。
偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁。我们回顾一下偏向锁的加锁流程：每次进入同步块（即执行monitorenter）的时候都会以从高往低的顺序在栈中找到第一个可用的Lock Record，将其obj字段指向锁对象。每次解锁（即执行monitorexit）的时候都会将最低的一个相关Lock Record移除掉。所以可以通过遍历线程栈中的Lock Record来判断线程是否还在同步块中。
将偏向线程所有相关Lock Record的Displaced Mark Word设置为null，然后将最高位的Lock Record的Displaced Mark Word 设置为无锁状态，最高位的Lock Record也就是第一次获得锁时的Lock Record（这里的第一次是指重入获取锁时的第一次），然后将对象头指向最高位的Lock Record，这里不需要用CAS指令，因为是在safepoint。 执行完后，就升级成了轻量级锁。原偏向线程的所有Lock Record都已经变成轻量级锁的状态。这里如果看不明白，请回顾上篇文章的轻量级锁加锁过程。


## 偏向锁的释放

偏向锁的释放入口在bytecodeInterpreter.cpp#1923
```C++
CASE(_monitorexit): {
  oop lockee = STACK_OBJECT(-1);
  CHECK_NULL(lockee);
  // derefing's lockee ought to provoke implicit null check
  // find our monitor slot
  BasicObjectLock* limit = istate->monitor_base();
  BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
  // 从低往高遍历栈的Lock Record
  while (most_recent != limit ) {
    // 如果Lock Record关联的是该锁对象
    if ((most_recent)->obj() == lockee) {
      BasicLock* lock = most_recent->lock();
      markOop header = lock->displaced_header();
      // 释放Lock Record
      most_recent->set_obj(NULL);
      // 如果是偏向模式，仅仅释放Lock Record就好了。否则要走轻量级锁or重量级锁的释放流程
      if (!lockee->mark()->has_bias_pattern()) {
        bool call_vm = UseHeavyMonitors;
        // header!=NULL说明不是重入，则需要将Displaced Mark Word CAS到对象头的Mark Word
        if (header != NULL || call_vm) {
          if (call_vm || Atomic::cmpxchg_ptr(header, lockee->mark_addr(), lock) != lock) {
            // CAS失败或者是重量级锁则会走到这里，先将obj还原，然后调用monitorexit方法
            most_recent->set_obj(lockee);
            CALL_VM(InterpreterRuntime::monitorexit(THREAD, most_recent), handle_exception);
          }
        }
      }
      //执行下一条命令
      UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
    }
    //处理下一条Lock Record
    most_recent++;
  }
  // Need to throw illegal monitor state exception
  CALL_VM(InterpreterRuntime::throw_illegal_monitor_state_exception(THREAD), handle_exception);
  ShouldNotReachHere();
}
```
上面的代码结合注释理解起来应该不难，偏向锁的释放很简单，只要将对应Lock Record释放就好了，而轻量级锁则需要将Displaced Mark Word替换到对象头的mark word中。如果CAS失败或者是重量级锁则进入到InterpreterRuntime::monitorexit方法中。该方法会在轻量级与重量级锁的文章中讲解。

## 批量重偏向和批量撤销
批量重偏向和批量撤销的背景可以看上篇文章，相关实现在BiasedLocking::revoke_and_rebias中：
```C++
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  ...
  //code 1：重偏向的逻辑
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  // 非重偏向的逻辑
  ...
      
  assert((heuristics == HR_BULK_REVOKE) ||
         (heuristics == HR_BULK_REBIAS), "?");	
   //code 2：批量撤销、批量重偏向的逻辑
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  return bulk_revoke.status_code();
}
在每次撤销偏向锁的时候都通过update_heuristics方法记录下来，以类为单位，当某个类的对象撤销偏向次数达到一定阈值的时候JVM就认为该类不适合偏向模式或者需要重新偏向另一个对象，update_heuristics就会返回HR_BULK_REVOKE或HR_BULK_REBIAS。进行批量撤销或批量重偏向。

先看update_heuristics方法。

static HeuristicsResult update_heuristics(oop o, bool allow_rebias) {
  markOop mark = o->mark();
  //如果不是偏向模式直接返回
  if (!mark->has_bias_pattern()) {
    return HR_NOT_BIASED;
  }
 
  // 锁对象的类
  Klass* k = o->klass();
  // 当前时间
  jlong cur_time = os::javaTimeMillis();
  // 该类上一次批量撤销的时间
  jlong last_bulk_revocation_time = k->last_biased_lock_bulk_revocation_time();
  // 该类偏向锁撤销的次数
  int revocation_count = k->biased_lock_revocation_count();
  // BiasedLockingBulkRebiasThreshold是重偏向阈值（默认20），BiasedLockingBulkRevokeThreshold是批量撤销阈值（默认40），BiasedLockingDecayTime是开启一次新的批量重偏向距离上次批量重偏向的后的延迟时间，默认25000。也就是开启批量重偏向后，经过了一段较长的时间（>=BiasedLockingDecayTime），撤销计数器才超过阈值，那我们会重置计数器。
  if ((revocation_count >= BiasedLockingBulkRebiasThreshold) &&
      (revocation_count <  BiasedLockingBulkRevokeThreshold) &&
      (last_bulk_revocation_time != 0) &&
      (cur_time - last_bulk_revocation_time >= BiasedLockingDecayTime)) {
    // This is the first revocation we've seen in a while of an
    // object of this type since the last time we performed a bulk
    // rebiasing operation. The application is allocating objects in
    // bulk which are biased toward a thread and then handing them
    // off to another thread. We can cope with this allocation
    // pattern via the bulk rebiasing mechanism so we reset the
    // klass's revocation count rather than allow it to increase
    // monotonically. If we see the need to perform another bulk
    // rebias operation later, we will, and if subsequently we see
    // many more revocation operations in a short period of time we
    // will completely disable biasing for this type.
    k->set_biased_lock_revocation_count(0);
    revocation_count = 0;
  }

  // 自增撤销计数器
  if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
    revocation_count = k->atomic_incr_biased_lock_revocation_count();
  }
  // 如果达到批量撤销阈值则返回HR_BULK_REVOKE
  if (revocation_count == BiasedLockingBulkRevokeThreshold) {
    return HR_BULK_REVOKE;
  }
  // 如果达到批量重偏向阈值则返回HR_BULK_REBIAS
  if (revocation_count == BiasedLockingBulkRebiasThreshold) {
    return HR_BULK_REBIAS;
  }
  // 没有达到阈值则撤销单个对象的锁
  return HR_SINGLE_REVOKE;
}
```
当达到阈值的时候就会通过VM 线程在safepoint调用bulk_revoke_or_rebias_at_safepoint, 参数bulk_rebias如果是true代表是批量重偏向否则为批量撤销。attempt_rebias_of_object代表对操作的锁对象o是否运行重偏向，这里是true。
```C++
static BiasedLocking::Condition bulk_revoke_or_rebias_at_safepoint(oop o,
                                                                   bool bulk_rebias,
                                                                   bool attempt_rebias_of_object,
                                                                   JavaThread* requesting_thread) {
  ...
  jlong cur_time = os::javaTimeMillis();
  o->klass()->set_last_biased_lock_bulk_revocation_time(cur_time);


  Klass* k_o = o->klass();
  Klass* klass = k_o;

  if (bulk_rebias) {
    // 批量重偏向的逻辑
    if (klass->prototype_header()->has_bias_pattern()) {
      // 自增前类中的的epoch
      int prev_epoch = klass->prototype_header()->bias_epoch();
      // code 1：类中的epoch自增
      klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
      int cur_epoch = klass->prototype_header()->bias_epoch();

      // code 2：遍历所有线程的栈，更新类型为该klass的所有锁实例的epoch
      for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
        GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
        for (int i = 0; i < cached_monitor_info->length(); i++) {
          MonitorInfo* mon_info = cached_monitor_info->at(i);
          oop owner = mon_info->owner();
          markOop mark = owner->mark();
          if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
            // We might have encountered this object already in the case of recursive locking
            assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
            owner->set_mark(mark->set_bias_epoch(cur_epoch));
          }
        }
      }
    }

    // 接下来对当前锁对象进行重偏向
    revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread);
  } else {
    ...

    // code 3：批量撤销的逻辑，将类中的偏向标记关闭，markOopDesc::prototype()返回的是一个关闭偏向模式的prototype
    klass->set_prototype_header(markOopDesc::prototype());

    // code 4：遍历所有线程的栈，撤销该类所有锁的偏向
    for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
      GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
      for (int i = 0; i < cached_monitor_info->length(); i++) {
        MonitorInfo* mon_info = cached_monitor_info->at(i);
        oop owner = mon_info->owner();
        markOop mark = owner->mark();
        if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
          revoke_bias(owner, false, true, requesting_thread);
        }
      }
    }

    // 撤销当前锁对象的偏向模式
    revoke_bias(o, false, true, requesting_thread);
  }

  ...
  
  BiasedLocking::Condition status_code = BiasedLocking::BIAS_REVOKED;

  if (attempt_rebias_of_object &&
      o->mark()->has_bias_pattern() &&
      klass->prototype_header()->has_bias_pattern()) {
    // 构造一个偏向请求线程的mark word
    markOop new_mark = markOopDesc::encode(requesting_thread, o->mark()->age(),
                                           klass->prototype_header()->bias_epoch());
    // 更新当前锁对象的mark word
    o->set_mark(new_mark);
    status_code = BiasedLocking::BIAS_REVOKED_AND_REBIASED;
    ...
  }

  ...

  return status_code;
}
```
该方法分为两个逻辑：批量重偏向和批量撤销。

先看批量重偏向，分为两步：

code 1 将类中的撤销计数器自增1，之后当该类已存在的实例获得锁时，就会尝试重偏向，相关逻辑在偏向锁获取流程小节中。

code 2 处理当前正在被使用的锁对象，通过遍历所有存活线程的栈，找到所有正在使用的偏向锁对象，然后更新它们的epoch值。也就是说不会重偏向正在使用的锁，否则会破坏锁的线程安全性。

批量撤销逻辑如下：

code 3将类的偏向标记关闭，之后当该类已存在的实例获得锁时，就会升级为轻量级锁；该类新分配的对象的mark word则是无锁模式。

code 4处理当前正在被使用的锁对象，通过遍历所有存活线程的栈，找到所有正在使用的偏向锁对象，然后撤销偏向锁。



# 死磕Synchronized底层实现--轻量级锁

轻量级锁并不复杂，其中很多内容在偏向锁一文中已提及过，与本文内容会有部分重叠。

另外轻量级锁的背景和基本流程在概论中已有讲解。强烈建议在看过两篇文章的基础下阅读本文。

本系列文章将对HotSpot的synchronized锁实现进行全面分析，内容包括偏向锁、轻量级锁、重量级锁的加锁、解锁、锁升级流程的原理及源码分析，希望给在研究synchronized路上的同学一些帮助。主要包括以下几篇文章：

死磕Synchronized底层实现--概论

死磕Synchronized底层实现--偏向锁

死磕Synchronized底层实现--轻量级锁

死磕Synchronized底层实现--重量级锁

更多文章见个人博客：https://github.com/farmerjohngit/myblog

本文分为两个部分：

1.轻量级锁获取流程

2.轻量级锁释放流程

本人看的JVM版本是jdk8u，具体版本号以及代码可以在这里看到。

轻量级锁获取流程
下面开始轻量级锁获取流程分析，代码在bytecodeInterpreter.cpp#1816。

CASE(_monitorenter): {
oop lockee = STACK_OBJECT(-1);
...
if (entry != NULL) {
...
// 上面省略的代码中如果CAS操作失败也会调用到InterpreterRuntime::monitorenter

    // traditional lightweight locking
    if (!success) {
      // 构建一个无锁状态的Displaced Mark Word
      markOop displaced = lockee->mark()->set_unlocked();
      // 设置到Lock Record中去
      entry->lock()->set_displaced_header(displaced);
      bool call_vm = UseHeavyMonitors;
      if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
        // 如果CAS替换不成功，代表锁对象不是无锁状态，这时候判断下是不是锁重入
        // Is it simple recursive case?
        if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
          entry->lock()->set_displaced_header(NULL);
        } else {
          // CAS操作失败则调用monitorenter
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
      }
    }
    UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
} else {
istate->set_msg(more_monitors);
UPDATE_PC_AND_RETURN(0); // Re-execute
}
}
如果锁对象不是偏向模式或已经偏向其他线程，则success为false。这时候会构建一个无锁状态的mark word设置到Lock Record中去，我们称Lock Record中存储对象mark word的字段叫Displaced Mark Word。

如果当前锁的状态不是无锁状态，则CAS失败。如果这是一次锁重入，那直接将Lock Record的 Displaced Mark Word设置为null。

我们看个demo，在该demo中重复3次获得锁，

synchronized(obj){
synchronized(obj){
synchronized(obj){
}
}
}
假设锁的状态是轻量级锁，下图反应了mark word和线程栈中Lock Record的状态，可以看到右边线程栈中包含3个指向当前锁对象的Lock Record。其中栈中最高位的Lock Record为第一次获取锁时分配的。其Displaced Mark word的值为锁对象的加锁前的mark word，之后的锁重入会在线程栈中分配一个Displaced Mark word为null的Lock Record。



为什么JVM选择在线程栈中添加Displaced Mark word为null的Lock Record来表示重入计数呢？首先锁重入次数是一定要记录下来的，因为每次解锁都需要对应一次加锁，解锁次数等于加锁次数时，该锁才真正的被释放，也就是在解锁时需要用到说锁重入次数的。一个简单的方案是将锁重入次数记录在对象头的mark word中，但mark word的大小是有限的，已经存放不下该信息了。另一个方案是只创建一个Lock Record并在其中记录重入次数，Hotspot没有这样做的原因我猜是考虑到效率有影响：每次重入获得锁都需要遍历该线程的栈找到对应的Lock Record，然后修改它的值。

所以最终Hotspot选择每次获得锁都添加一个Lock Record来表示锁的重入。

接下来看看InterpreterRuntime::monitorenter方法

IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
...
Handle h_obj(thread, elem->obj());
assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
"must be NULL or an object");
if (UseBiasedLocking) {
// Retry fast entry if bias is revoked to avoid unnecessary inflation
ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
} else {
ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
}
...
IRT_END
fast_enter的流程在偏向锁一文已经分析过，如果当前是偏向模式且偏向的线程还在使用锁，那会将锁的mark word改为轻量级锁的状态，同时会将偏向的线程栈中的Lock Record修改为轻量级锁对应的形式。代码位置在biasedLocking.cpp#212。

// 线程还存活则遍历线程栈中所有的Lock Record
GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
BasicLock* highest_lock = NULL;
for (int i = 0; i < cached_monitor_info->length(); i++) {
MonitorInfo* mon_info = cached_monitor_info->at(i);
// 如果能找到对应的Lock Record说明偏向的线程还在执行同步代码块中的代码
if (mon_info->owner() == obj) {
...
// 需要升级为轻量级锁，直接修改偏向线程栈中的Lock Record。为了处理锁重入的case，在这里将Lock Record的Displaced Mark Word设置为null，第一个Lock Record会在下面的代码中再处理
markOop mark = markOopDesc::encode((BasicLock*) NULL);
highest_lock = mon_info->lock();
highest_lock->set_displaced_header(mark);
} else {
...
}
}
if (highest_lock != NULL) {
// 修改第一个Lock Record为无锁状态，然后将obj的mark word设置为指向该Lock Record的指针
highest_lock->set_displaced_header(unbiased_prototype);
obj->release_set_mark(markOopDesc::encode(highest_lock));
...
} else {
...
}
我们看slow_enter的流程。

void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
markOop mark = obj->mark();
assert(!mark->has_bias_pattern(), "should not see bias pattern here");
// 如果是无锁状态
if (mark->is_neutral()) {
//设置Displaced Mark Word并替换对象头的mark word
lock->set_displaced_header(mark);
if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
TEVENT (slow_enter: release stacklock) ;
return ;
}
} else
if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
assert(lock != mark->locker(), "must not re-lock the same lock");
assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
// 如果是重入，则设置Displaced Mark Word为null
lock->set_displaced_header(NULL);
return;
}

...
// 走到这一步说明已经是存在多个线程竞争锁了 需要膨胀为重量级锁
lock->set_displaced_header(markOopDesc::unused_mark());
ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
轻量级锁释放流程
CASE(_monitorexit): {
oop lockee = STACK_OBJECT(-1);
CHECK_NULL(lockee);
// derefing's lockee ought to provoke implicit null check
// find our monitor slot
BasicObjectLock* limit = istate->monitor_base();
BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
// 从低往高遍历栈的Lock Record
while (most_recent != limit ) {
// 如果Lock Record关联的是该锁对象
if ((most_recent)->obj() == lockee) {
BasicLock* lock = most_recent->lock();
markOop header = lock->displaced_header();
// 释放Lock Record
most_recent->set_obj(NULL);
// 如果是偏向模式，仅仅释放Lock Record就好了。否则要走轻量级锁or重量级锁的释放流程
if (!lockee->mark()->has_bias_pattern()) {
bool call_vm = UseHeavyMonitors;
// header!=NULL说明不是重入，则需要将Displaced Mark Word CAS到对象头的Mark Word
if (header != NULL || call_vm) {
if (call_vm || Atomic::cmpxchg_ptr(header, lockee->mark_addr(), lock) != lock) {
// CAS失败或者是重量级锁则会走到这里，先将obj还原，然后调用monitorexit方法
most_recent->set_obj(lockee);
CALL_VM(InterpreterRuntime::monitorexit(THREAD, most_recent), handle_exception);
}
}
}
//执行下一条命令
UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
}
//处理下一条Lock Record
most_recent++;
}
// Need to throw illegal monitor state exception
CALL_VM(InterpreterRuntime::throw_illegal_monitor_state_exception(THREAD), handle_exception);
ShouldNotReachHere();
}
轻量级锁释放时需要将Displaced Mark Word替换到对象头的mark word中。如果CAS失败或者是重量级锁则进入到InterpreterRuntime::monitorexit方法中。

//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))

Handle h_obj(thread, elem->obj());
...
ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
// Free entry. This must be done here, since a pending exception might be installed on
//释放Lock Record
elem->set_obj(NULL);
...
IRT_END
monitorexit调用完slow_exit方法后,就释放Lock Record。

void ObjectSynchronizer::slow_exit(oop object, BasicLock* lock, TRAPS) {
fast_exit (object, lock, THREAD) ;
}
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
...
markOop dhw = lock->displaced_header();
markOop mark ;
if (dhw == NULL) {
// 重入锁，什么也不做
...
return ;
}

mark = object->mark() ;

// 如果是mark word==Displaced Mark Word即轻量级锁，CAS替换对象头的mark word
if (mark == (markOop) lock) {
assert (dhw->is_neutral(), "invariant") ;
if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
TEVENT (fast_exit: release stacklock) ;
return;
}
}
//走到这里说明是重量级锁或者解锁时发生了竞争，膨胀后调用重量级锁的exit方法。
ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;
}
该方法中先判断是不是轻量级锁，如果是轻量级锁则将替换mark word，否则膨胀为重量级锁并调用exit方法，相关逻辑将在重量级锁的文章中讲解。

# 死磕Synchronized底层实现--偏向锁 

本文将分为几块内容：

1.偏向锁的入口

2.偏向锁的获取流程

3.偏向锁的撤销流程

4.偏向锁的释放流程

5.偏向锁的批量重偏向和批量撤销

本文分析的JVM版本是JVM8，具体版本号以及代码可以在这里看到。

偏向锁入口
目前网上的很多文章，关于偏向锁源码入口都找错地方了，导致我之前对于偏向锁的很多逻辑一直想不通，走了很多弯路。

synchronized分为synchronized代码块和synchronized方法，其底层获取锁的逻辑都是一样的，本文讲解的是synchronized代码块的实现。上篇文章也说过，synchronized代码块是由monitorenter和monitorexit两个指令实现的。

关于HotSpot虚拟机中获取锁的入口，网上很多文章要么给出的方法入口为interpreterRuntime.cpp#monitorenter，要么给出的入口为bytecodeInterpreter.cpp#1816。包括占小狼的这篇文章关于锁入口的位置说法也是有问题的（当然文章还是很好的，在我刚开始研究synchronized的时候，小狼哥的这篇文章给了我很多帮助）。

要找锁的入口，肯定是要在源码中找到对monitorenter指令解析的地方。在HotSpot的中有两处地方对monitorenter指令进行解析：一个是在bytecodeInterpreter.cpp#1816 ，另一个是在templateTable_x86_64.cpp#3667。

前者是JVM中的字节码解释器(bytecodeInterpreter)，用C++实现了每条JVM指令（如monitorenter、invokevirtual等），其优点是实现相对简单且容易理解，缺点是执行慢。后者是模板解释器(templateInterpreter)，其对每个指令都写了一段对应的汇编代码，启动时将每个指令与对应汇编代码入口绑定，可以说是效率做到了极致。模板解释器的实现可以看这篇文章，在研究的过程中也请教过文章作者‘汪先生’一些问题，这里感谢一下。

在HotSpot中，只用到了模板解释器，字节码解释器根本就没用到，R大的读书笔记中说的很清楚了，大家可以看看，这里不再赘述。

所以montorenter的解析入口在模板解释器中，其代码位于templateTable_x86_64.cpp#3667。通过调用路径：templateTable_x86_64#monitorenter->interp_masm_x86_64#lock_object进入到偏向锁入口macroAssembler_x86#biased_locking_enter，在这里大家可以看到会生成对应的汇编代码。需要注意的是，不是说每次解析monitorenter指令都会调用biased_locking_enter,而是只会在JVM启动的时候调用该方法生成汇编代码，之后对指令的解析是通过直接执行汇编代码。

其实bytecodeInterpreter的逻辑和templateInterpreter的逻辑是大同小异的，因为templateInterpreter中都是汇编代码，比较晦涩，所以看bytecodeInterpreter的实现会便于理解一点。但这里有个坑，在jdk8u之前，bytecodeInterpreter并没有实现偏向锁的逻辑。我之前看的JDK8-87ee5ee27509这个版本就没有实现偏向锁的逻辑，导致我看了很久都没看懂。在这个commit中对bytecodeInterpreter加入了偏向锁的支持，我大致了看了下和templateInterpreter对比除了栈结构不同外，其他逻辑大致相同，所以下文就按bytecodeInterpreter中的代码对偏向锁逻辑进行讲解。templateInterpreter的汇编代码讲解可以看这篇文章，其实汇编源码中都有英文注释，了解了汇编几个基本指令的作用再结合注释理解起来也不是很难。

偏向锁获取流程
下面开始偏向锁获取流程分析，代码在bytecodeInterpreter.cpp#1816。注意本文代码都有所删减。

CASE(_monitorenter): {
// lockee 就是锁对象
oop lockee = STACK_OBJECT(-1);
// derefing's lockee ought to provoke implicit null check
CHECK_NULL(lockee);
// code 1：找到一个空闲的Lock Record
BasicObjectLock* limit = istate->monitor_base();
BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
BasicObjectLock* entry = NULL;
while (most_recent != limit ) {
if (most_recent->obj() == NULL) entry = most_recent;
else if (most_recent->obj() == lockee) break;
most_recent++;
}
//entry不为null，代表还有空闲的Lock Record
if (entry != NULL) {
// code 2：将Lock Record的obj指针指向锁对象
entry->set_obj(lockee);
int success = false;
uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;
// markoop即对象头的mark word
markOop mark = lockee->mark();
intptr_t hash = (intptr_t) markOopDesc::no_hash;
// code 3：如果锁对象的mark word的状态是偏向模式
if (mark->has_bias_pattern()) {
uintptr_t thread_ident;
uintptr_t anticipated_bias_locking_value;
thread_ident = (uintptr_t)istate->thread();
// code 4：这里有几步操作，下文分析
anticipated_bias_locking_value =
(((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) &
~((uintptr_t) markOopDesc::age_mask_in_place);
// code 5：如果偏向的线程是自己且epoch等于class的epoch
if  (anticipated_bias_locking_value == 0) {
// already biased towards this thread, nothing to do
if (PrintBiasedLockingStatistics) {
(* BiasedLocking::biased_lock_entry_count_addr())++;
}
success = true;
}
// code 6：如果偏向模式关闭，则尝试撤销偏向锁
else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
markOop header = lockee->klass()->prototype_header();
if (hash != markOopDesc::no_hash) {
header = header->copy_set_hash(hash);
}
// 利用CAS操作将mark word替换为class中的mark word
if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
if (PrintBiasedLockingStatistics)
(*BiasedLocking::revoked_lock_entry_count_addr())++;
}
}
// code 7：如果epoch不等于class中的epoch，则尝试重偏向
else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
// 构造一个偏向当前线程的mark word
markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
if (hash != markOopDesc::no_hash) {
new_header = new_header->copy_set_hash(hash);
}
// CAS替换对象头的mark word  
if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
if (PrintBiasedLockingStatistics)
(* BiasedLocking::rebiased_lock_entry_count_addr())++;
}
else {
// 重偏向失败，代表存在多线程竞争，则调用monitorenter方法进行锁升级
CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
}
success = true;
}
else {
// 走到这里说明当前要么偏向别的线程，要么是匿名偏向（即没有偏向任何线程）
// code 8：下面构建一个匿名偏向的mark word，尝试用CAS指令替换掉锁对象的mark word
markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |(uintptr_t)markOopDesc::age_mask_in_place |epoch_mask_in_place));
if (hash != markOopDesc::no_hash) {
header = header->copy_set_hash(hash);
}
markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
// debugging hint
DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
// CAS修改成功
if (PrintBiasedLockingStatistics)
(* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
}
else {
// 如果修改失败说明存在多线程竞争，所以进入monitorenter方法
CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
}
success = true;
}
}

    // 如果偏向线程不是当前线程或没有开启偏向模式等原因都会导致success==false
    if (!success) {
      // 轻量级锁的逻辑
      //code 9: 构造一个无锁状态的Displaced Mark Word，并将Lock Record的lock指向它
      markOop displaced = lockee->mark()->set_unlocked();
      entry->lock()->set_displaced_header(displaced);
      //如果指定了-XX:+UseHeavyMonitors，则call_vm=true，代表禁用偏向锁和轻量级锁
      bool call_vm = UseHeavyMonitors;
      // 利用CAS将对象头的mark word替换为指向Lock Record的指针
      if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
        // 判断是不是锁重入
        if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {		//code 10: 如果是锁重入，则直接将Displaced Mark Word设置为null
          entry->lock()->set_displaced_header(NULL);
        } else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
      }
    }
    UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
} else {
// lock record不够，重新执行
istate->set_msg(more_monitors);
UPDATE_PC_AND_RETURN(0); // Re-execute
}
}
再回顾下对象头中mark word的格式：image

JVM中的每个类也有一个类似mark word的prototype_header，用来标记该class的epoch和偏向开关等信息。上面的代码中lockee->klass()->prototype_header()即获取class的prototype_header。

code 1，从当前线程的栈中找到一个空闲的Lock Record（即代码中的BasicObjectLock，下文都用Lock Record代指），判断Lock Record是否空闲的依据是其obj字段 是否为null。注意这里是按内存地址从低往高找到最后一个可用的Lock Record，换而言之，就是找到内存地址最高的可用Lock Record。

code 2，获取到Lock Record后，首先要做的就是为其obj字段赋值。

code 3，判断锁对象的mark word是否是偏向模式，即低3位是否为101。

code 4，这里有几步位运算的操作  anticipated_bias_locking_value = (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) & ​        ~((uintptr_t) markOopDesc::age_mask_in_place); 这个位运算可以分为3个部分。

第一部分((uintptr_t)lockee->klass()->prototype_header() | thread_ident) 将当前线程id和类的prototype_header相或，这样得到的值为（当前线程id + prototype_header中的（epoch + 分代年龄 + 偏向锁标志 + 锁标志位）），注意prototype_header的分代年龄那4个字节为0

第二部分 ^ (uintptr_t)mark 将上面计算得到的结果与锁对象的markOop进行异或，相等的位全部被置为0，只剩下不相等的位。

第三部分  & ~((uintptr_t) markOopDesc::age_mask_in_place) markOopDesc::age_mask_in_place为...0001111000,取反后，变成了...1110000111,除了分代年龄那4位，其他位全为1；将取反后的结果再与上面的结果相与，将上面异或得到的结果中分代年龄给忽略掉。

code 5，anticipated_bias_locking_value==0代表偏向的线程是当前线程且mark word的epoch等于class的epoch，这种情况下什么都不用做。

code 6，(anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0代表class的prototype_header或对象的mark word中偏向模式是关闭的，又因为能走到这已经通过了mark->has_bias_pattern()判断，即对象的mark word中偏向模式是开启的，那也就是说class的prototype_header不是偏向模式。

然后利用CAS指令Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark撤销偏向锁，我们知道CAS会有几个参数，1是预期的原值，2是预期修改后的值 ，3是要修改的对象，与之对应，cmpxchg_ptr方法第一个参数是预期修改后的值，第2个参数是修改的对象，第3个参数是预期原值，方法返回实际原值，如果等于预期原值则说明修改成功。

code 7，如果epoch已过期，则需要重偏向，利用CAS指令将锁对象的mark word替换为一个偏向当前线程且epoch为类的epoch的新的mark word。

code 8，CAS将偏向线程改为当前线程，如果当前是匿名偏向则能修改成功，否则进入锁升级的逻辑。

code 9，这一步已经是轻量级锁的逻辑了。从上图的mark word的格式可以看到，轻量级锁中mark word存的是指向Lock Record的指针。这里构造一个无锁状态的mark word，然后存储到Lock Record（Lock Record的格式可以看第一篇文章）。设置mark word是无锁状态的原因是：轻量级锁解锁时是将对象头的mark word设置为Lock Record中的Displaced Mark Word，所以创建时设置为无锁状态，解锁时直接用CAS替换就好了。

code 10， 如果是锁重入，则将Lock Record的Displaced Mark Word设置为null，起到一个锁重入计数的作用。

以上是偏向锁加锁的流程（包括部分轻量级锁的加锁流程），如果当前锁已偏向其他线程||epoch值过期||偏向模式关闭||获取偏向锁的过程中存在并发冲突，都会进入到InterpreterRuntime::monitorenter方法， 在该方法中会对偏向锁撤销和升级。

偏向锁的撤销
这里说的撤销是指在获取偏向锁的过程因为不满足条件导致要将锁对象改为非偏向锁状态；释放是指退出同步块时的过程，释放锁的逻辑会在下一小节阐述。请读者注意本文中撤销与释放的区别。

如果获取偏向锁失败会进入到InterpreterRuntime::monitorenter方法

IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
...
Handle h_obj(thread, elem->obj());
assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
"must be NULL or an object");
if (UseBiasedLocking) {
// Retry fast entry if bias is revoked to avoid unnecessary inflation
ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
} else {
ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
}
...
IRT_END
可以看到如果开启了JVM偏向锁，那会进入到ObjectSynchronizer::fast_enter方法中。

void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
if (UseBiasedLocking) {
if (!SafepointSynchronize::is_at_safepoint()) {
BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
return;
}
} else {
assert(!attempt_rebias, "can not rebias toward VM thread");
BiasedLocking::revoke_at_safepoint(obj);
}
assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
}

slow_enter (obj, lock, THREAD) ;
}
如果是正常的Java线程，会走上面的逻辑进入到BiasedLocking::revoke_and_rebias方法，如果是VM线程则会走到下面的BiasedLocking::revoke_at_safepoint。我们主要看BiasedLocking::revoke_and_rebias方法。这个方法的主要作用像它的方法名：撤销或者重偏向，第一个参数封装了锁对象和当前线程，第二个参数代表是否允许重偏向，这里是true。

BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");

markOop mark = obj->mark();
if (mark->is_biased_anonymously() && !attempt_rebias) {
//如果是匿名偏向且attempt_rebias==false会走到这里，如锁对象的hashcode方法被调用会出现这种情况，需要撤销偏向锁。
markOop biased_value       = mark;
markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
if (res_mark == biased_value) {
return BIAS_REVOKED;
}
} else if (mark->has_bias_pattern()) {
// 锁对象开启了偏向模式会走到这里
Klass* k = obj->klass();
markOop prototype_header = k->prototype_header();
//code 1： 如果对应class关闭了偏向模式
if (!prototype_header->has_bias_pattern()) {
markOop biased_value       = mark;
markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
return BIAS_REVOKED;
//code2： 如果epoch过期
} else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
if (attempt_rebias) {
assert(THREAD->is_Java_thread(), "");
markOop biased_value       = mark;
markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
if (res_mark == biased_value) {
return BIAS_REVOKED_AND_REBIASED;
}
} else {
markOop biased_value       = mark;
markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
if (res_mark == biased_value) {
return BIAS_REVOKED;
}
}
}
}
//code 3：批量重偏向与批量撤销的逻辑
HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
if (heuristics == HR_NOT_BIASED) {
return NOT_BIASED;
} else if (heuristics == HR_SINGLE_REVOKE) {
//code 4：撤销单个线程
Klass *k = obj->klass();
markOop prototype_header = k->prototype_header();
if (mark->biased_locker() == THREAD &&
prototype_header->bias_epoch() == mark->bias_epoch()) {
// 走到这里说明需要撤销的是偏向当前线程的锁，当调用Object#hashcode方法时会走到这一步
// 因为只要遍历当前线程的栈就好了，所以不需要等到safepoint再撤销。
ResourceMark rm;
if (TraceBiasedLocking) {
tty->print_cr("Revoking bias by walking my own stack:");
}
BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD);
((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
assert(cond == BIAS_REVOKED, "why not?");
return cond;
} else {
// 下面代码最终会在VM线程中的safepoint调用revoke_bias方法
VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
VMThread::execute(&revoke);
return revoke.status_code();
}
}

assert((heuristics == HR_BULK_REVOKE) ||
(heuristics == HR_BULK_REBIAS), "?");
//code5：批量撤销、批量重偏向的逻辑
VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
(heuristics == HR_BULK_REBIAS),
attempt_rebias);
VMThread::execute(&bulk_revoke);
return bulk_revoke.status_code();
}
会走到该方法的逻辑有很多，我们只分析最常见的情况：假设锁已经偏向线程A，这时B线程尝试获得锁。

上面的code 1，code 2B线程都不会走到，最终会走到code 4处，如果要撤销的锁偏向的是当前线程则直接调用revoke_bias撤销偏向锁，否则会将该操作push到VM Thread中等到safepoint的时候再执行。

关于VM Thread这里介绍下：在JVM中有个专门的VM Thread，该线程会源源不断的从VMOperationQueue中取出请求，比如GC请求。对于需要safepoint的操作（VM_Operationevaluate_at_safepoint返回true）必须要等到所有的Java线程进入到safepoint才开始执行。 关于safepoint可以参考下这篇文章。

接下来我们着重分析下revoke_bias方法。第一个参数为锁对象，第2、3个参数为都为false

static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread) {
markOop mark = obj->mark();
// 如果没有开启偏向模式，则直接返回NOT_BIASED
if (!mark->has_bias_pattern()) {
...
return BiasedLocking::NOT_BIASED;
}

uint age = mark->age();
// 构建两个mark word，一个是匿名偏向模式（101），一个是无锁模式（001）
markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);

...

JavaThread* biased_thread = mark->biased_locker();
if (biased_thread == NULL) {
// 匿名偏向。当调用锁对象的hashcode()方法可能会导致走到这个逻辑
// 如果不允许重偏向，则将对象的mark word设置为无锁模式
if (!allow_rebias) {
obj->set_mark(unbiased_prototype);
}
...
return BiasedLocking::BIAS_REVOKED;
}

// code 1：判断偏向线程是否还存活
bool thread_is_alive = false;
// 如果当前线程就是偏向线程
if (requesting_thread == biased_thread) {
thread_is_alive = true;
} else {
// 遍历当前jvm的所有线程，如果能找到，则说明偏向的线程还存活
for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
if (cur_thread == biased_thread) {
thread_is_alive = true;
break;
}
}
}
// 如果偏向的线程已经不存活了
if (!thread_is_alive) {
// 允许重偏向则将对象mark word设置为匿名偏向状态，否则设置为无锁状态
if (allow_rebias) {
obj->set_mark(biased_prototype);
} else {
obj->set_mark(unbiased_prototype);
}
...
return BiasedLocking::BIAS_REVOKED;
}

// 线程还存活则遍历线程栈中所有的Lock Record
GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
BasicLock* highest_lock = NULL;
for (int i = 0; i < cached_monitor_info->length(); i++) {
MonitorInfo* mon_info = cached_monitor_info->at(i);
// 如果能找到对应的Lock Record说明偏向的线程还在执行同步代码块中的代码
if (mon_info->owner() == obj) {
...
// 需要升级为轻量级锁，直接修改偏向线程栈中的Lock Record。为了处理锁重入的case，在这里将Lock Record的Displaced Mark Word设置为null，第一个Lock Record会在下面的代码中再处理
markOop mark = markOopDesc::encode((BasicLock*) NULL);
highest_lock = mon_info->lock();
highest_lock->set_displaced_header(mark);
} else {
...
}
}
if (highest_lock != NULL) {
// 修改第一个Lock Record为无锁状态，然后将obj的mark word设置为指向该Lock Record的指针
highest_lock->set_displaced_header(unbiased_prototype);
obj->release_set_mark(markOopDesc::encode(highest_lock));
...
} else {
// 走到这里说明偏向线程已经不在同步块中了
...
if (allow_rebias) {
//设置为匿名偏向状态
obj->set_mark(biased_prototype);
} else {
// 将mark word设置为无锁状态
obj->set_mark(unbiased_prototype);
}
}

return BiasedLocking::BIAS_REVOKED;
}
需要注意下，当调用锁对象的Object#hash或System.identityHashCode()方法会导致该对象的偏向锁或轻量级锁升级。这是因为在Java中一个对象的hashcode是在调用这两个方法时才生成的，如果是无锁状态则存放在mark word中，如果是重量级锁则存放在对应的monitor中，而偏向锁是没有地方能存放该信息的，所以必须升级。具体可以看这篇文章的hashcode()方法对偏向锁的影响小节（注意：该文中对于偏向锁的加锁描述有些错误），另外我也向该文章作者请教过一些问题，他很热心的回答了我，在此感谢一下！

言归正传，revoke_bias方法逻辑：

查看偏向的线程是否存活，如果已经不存活了，则直接撤销偏向锁。JVM维护了一个集合存放所有存活的线程，通过遍历该集合判断某个线程是否存活。
偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁。我们回顾一下偏向锁的加锁流程：每次进入同步块（即执行monitorenter）的时候都会以从高往低的顺序在栈中找到第一个可用的Lock Record，将其obj字段指向锁对象。每次解锁（即执行monitorexit）的时候都会将最低的一个相关Lock Record移除掉。所以可以通过遍历线程栈中的Lock Record来判断线程是否还在同步块中。
将偏向线程所有相关Lock Record的Displaced Mark Word设置为null，然后将最高位的Lock Record的Displaced Mark Word 设置为无锁状态，最高位的Lock Record也就是第一次获得锁时的Lock Record（这里的第一次是指重入获取锁时的第一次），然后将对象头指向最高位的Lock Record，这里不需要用CAS指令，因为是在safepoint。 执行完后，就升级成了轻量级锁。原偏向线程的所有Lock Record都已经变成轻量级锁的状态。这里如果看不明白，请回顾上篇文章的轻量级锁加锁过程。
偏向锁的释放
偏向锁的释放入口在bytecodeInterpreter.cpp#1923

CASE(_monitorexit): {
oop lockee = STACK_OBJECT(-1);
CHECK_NULL(lockee);
// derefing's lockee ought to provoke implicit null check
// find our monitor slot
BasicObjectLock* limit = istate->monitor_base();
BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
// 从低往高遍历栈的Lock Record
while (most_recent != limit ) {
// 如果Lock Record关联的是该锁对象
if ((most_recent)->obj() == lockee) {
BasicLock* lock = most_recent->lock();
markOop header = lock->displaced_header();
// 释放Lock Record
most_recent->set_obj(NULL);
// 如果是偏向模式，仅仅释放Lock Record就好了。否则要走轻量级锁or重量级锁的释放流程
if (!lockee->mark()->has_bias_pattern()) {
bool call_vm = UseHeavyMonitors;
// header!=NULL说明不是重入，则需要将Displaced Mark Word CAS到对象头的Mark Word
if (header != NULL || call_vm) {
if (call_vm || Atomic::cmpxchg_ptr(header, lockee->mark_addr(), lock) != lock) {
// CAS失败或者是重量级锁则会走到这里，先将obj还原，然后调用monitorexit方法
most_recent->set_obj(lockee);
CALL_VM(InterpreterRuntime::monitorexit(THREAD, most_recent), handle_exception);
}
}
}
//执行下一条命令
UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
}
//处理下一条Lock Record
most_recent++;
}
// Need to throw illegal monitor state exception
CALL_VM(InterpreterRuntime::throw_illegal_monitor_state_exception(THREAD), handle_exception);
ShouldNotReachHere();
}
上面的代码结合注释理解起来应该不难，偏向锁的释放很简单，只要将对应Lock Record释放就好了，而轻量级锁则需要将Displaced Mark Word替换到对象头的mark word中。如果CAS失败或者是重量级锁则进入到InterpreterRuntime::monitorexit方法中。该方法会在轻量级与重量级锁的文章中讲解。

批量重偏向和批量撤销
批量重偏向和批量撤销的背景可以看上篇文章，相关实现在BiasedLocking::revoke_and_rebias中：

BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
...
//code 1：重偏向的逻辑
HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
// 非重偏向的逻辑
...

assert((heuristics == HR_BULK_REVOKE) ||
(heuristics == HR_BULK_REBIAS), "?");
//code 2：批量撤销、批量重偏向的逻辑
VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
(heuristics == HR_BULK_REBIAS),
attempt_rebias);
VMThread::execute(&bulk_revoke);
return bulk_revoke.status_code();
}
在每次撤销偏向锁的时候都通过update_heuristics方法记录下来，以类为单位，当某个类的对象撤销偏向次数达到一定阈值的时候JVM就认为该类不适合偏向模式或者需要重新偏向另一个对象，update_heuristics就会返回HR_BULK_REVOKE或HR_BULK_REBIAS。进行批量撤销或批量重偏向。

先看update_heuristics方法。

static HeuristicsResult update_heuristics(oop o, bool allow_rebias) {
markOop mark = o->mark();
//如果不是偏向模式直接返回
if (!mark->has_bias_pattern()) {
return HR_NOT_BIASED;
}

// 锁对象的类
Klass* k = o->klass();
// 当前时间
jlong cur_time = os::javaTimeMillis();
// 该类上一次批量撤销的时间
jlong last_bulk_revocation_time = k->last_biased_lock_bulk_revocation_time();
// 该类偏向锁撤销的次数
int revocation_count = k->biased_lock_revocation_count();
// BiasedLockingBulkRebiasThreshold是重偏向阈值（默认20），BiasedLockingBulkRevokeThreshold是批量撤销阈值（默认40），BiasedLockingDecayTime是开启一次新的批量重偏向距离上次批量重偏向的后的延迟时间，默认25000。也就是开启批量重偏向后，经过了一段较长的时间（>=BiasedLockingDecayTime），撤销计数器才超过阈值，那我们会重置计数器。
if ((revocation_count >= BiasedLockingBulkRebiasThreshold) &&
(revocation_count <  BiasedLockingBulkRevokeThreshold) &&
(last_bulk_revocation_time != 0) &&
(cur_time - last_bulk_revocation_time >= BiasedLockingDecayTime)) {
// This is the first revocation we've seen in a while of an
// object of this type since the last time we performed a bulk
// rebiasing operation. The application is allocating objects in
// bulk which are biased toward a thread and then handing them
// off to another thread. We can cope with this allocation
// pattern via the bulk rebiasing mechanism so we reset the
// klass's revocation count rather than allow it to increase
// monotonically. If we see the need to perform another bulk
// rebias operation later, we will, and if subsequently we see
// many more revocation operations in a short period of time we
// will completely disable biasing for this type.
k->set_biased_lock_revocation_count(0);
revocation_count = 0;
}

// 自增撤销计数器
if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
revocation_count = k->atomic_incr_biased_lock_revocation_count();
}
// 如果达到批量撤销阈值则返回HR_BULK_REVOKE
if (revocation_count == BiasedLockingBulkRevokeThreshold) {
return HR_BULK_REVOKE;
}
// 如果达到批量重偏向阈值则返回HR_BULK_REBIAS
if (revocation_count == BiasedLockingBulkRebiasThreshold) {
return HR_BULK_REBIAS;
}
// 没有达到阈值则撤销单个对象的锁
return HR_SINGLE_REVOKE;
}
当达到阈值的时候就会通过VM 线程在safepoint调用bulk_revoke_or_rebias_at_safepoint, 参数bulk_rebias如果是true代表是批量重偏向否则为批量撤销。attempt_rebias_of_object代表对操作的锁对象o是否运行重偏向，这里是true。

static BiasedLocking::Condition bulk_revoke_or_rebias_at_safepoint(oop o,
bool bulk_rebias,
bool attempt_rebias_of_object,
JavaThread* requesting_thread) {
...
jlong cur_time = os::javaTimeMillis();
o->klass()->set_last_biased_lock_bulk_revocation_time(cur_time);


Klass* k_o = o->klass();
Klass* klass = k_o;

if (bulk_rebias) {
// 批量重偏向的逻辑
if (klass->prototype_header()->has_bias_pattern()) {
// 自增前类中的的epoch
int prev_epoch = klass->prototype_header()->bias_epoch();
// code 1：类中的epoch自增
klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
int cur_epoch = klass->prototype_header()->bias_epoch();

      // code 2：遍历所有线程的栈，更新类型为该klass的所有锁实例的epoch
      for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
        GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
        for (int i = 0; i < cached_monitor_info->length(); i++) {
          MonitorInfo* mon_info = cached_monitor_info->at(i);
          oop owner = mon_info->owner();
          markOop mark = owner->mark();
          if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
            // We might have encountered this object already in the case of recursive locking
            assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
            owner->set_mark(mark->set_bias_epoch(cur_epoch));
          }
        }
      }
    }

    // 接下来对当前锁对象进行重偏向
    revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread);
} else {
...

    // code 3：批量撤销的逻辑，将类中的偏向标记关闭，markOopDesc::prototype()返回的是一个关闭偏向模式的prototype
    klass->set_prototype_header(markOopDesc::prototype());

    // code 4：遍历所有线程的栈，撤销该类所有锁的偏向
    for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
      GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
      for (int i = 0; i < cached_monitor_info->length(); i++) {
        MonitorInfo* mon_info = cached_monitor_info->at(i);
        oop owner = mon_info->owner();
        markOop mark = owner->mark();
        if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
          revoke_bias(owner, false, true, requesting_thread);
        }
      }
    }

    // 撤销当前锁对象的偏向模式
    revoke_bias(o, false, true, requesting_thread);
}

...

BiasedLocking::Condition status_code = BiasedLocking::BIAS_REVOKED;

if (attempt_rebias_of_object &&
o->mark()->has_bias_pattern() &&
klass->prototype_header()->has_bias_pattern()) {
// 构造一个偏向请求线程的mark word
markOop new_mark = markOopDesc::encode(requesting_thread, o->mark()->age(),
klass->prototype_header()->bias_epoch());
// 更新当前锁对象的mark word
o->set_mark(new_mark);
status_code = BiasedLocking::BIAS_REVOKED_AND_REBIASED;
...
}

...

return status_code;
}
该方法分为两个逻辑：批量重偏向和批量撤销。

先看批量重偏向，分为两步：

code 1 将类中的撤销计数器自增1，之后当该类已存在的实例获得锁时，就会尝试重偏向，相关逻辑在偏向锁获取流程小节中。

code 2 处理当前正在被使用的锁对象，通过遍历所有存活线程的栈，找到所有正在使用的偏向锁对象，然后更新它们的epoch值。也就是说不会重偏向正在使用的锁，否则会破坏锁的线程安全性。

批量撤销逻辑如下：

code 3将类的偏向标记关闭，之后当该类已存在的实例获得锁时，就会升级为轻量级锁；该类新分配的对象的mark word则是无锁模式。

code 4处理当前正在被使用的锁对象，通过遍历所有存活线程的栈，找到所有正在使用的偏向锁对象，然后撤销偏向锁。



# 参考资料

jdk8u源码 [http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683)
死磕Synchronized底层实现 [https://github.com/farmerjohngit/myblog/issues/12](https://github.com/farmerjohngit/myblog/issues/12)