# JVM

#  JVM 内存模型

## 概述

运行一个 Java 应用程序，必须要先安装 JDK 或者 JRE 包。因为 Java 应用在编译后会变成字节码，通过字节码运行在 JVM 中，而 JVM 是 JRE 的核心组成部分。JVM 不仅承担了 Java 字节码的分析和执行，同时也内置了自动内存分配管理机制。这个机制可以大大降低手动分配回收机制可能带来的内存泄露和内存溢出风险，使 Java 开发人员不需要关注每个对象的内存分配以及回收，从而更专注于业务本身。

## 运行时数据区(Runtime Data Area)

Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间，有的区域随着虚拟机进程的启动而存在，有些区域则依赖用户线程的启动和结束而建立和销毁。根据《Java虚拟机规范（Java SE 7版）》的规定，Java虚拟机所管理的内存将会包括以下几个运行时数据区域。在 Java 中，JVM 内存模型主要分为堆、方法区、程序计数器、虚拟机栈和本地方法栈。其中，堆和方法区被所有线程共享，虚拟机栈、本地方法栈、程序计数器是线程私有的。

![JVM架构](png/JVM架构.png)

### 程序计数器(Program Counter Register)

程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过一些更高效的方式去实现），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是Native方法，这个计数器值则为空（Undefined）。此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

程序计数器（Program Counter Register）也叫PC寄存器。程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。JVM支持多个线程同时运行，每个线程都有自己的程序计数器。倘若当前执行的是 JVM 的方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native 方法，则PC寄存器中为空(undefined)。

- 当前线程私有
- 当前线程所执行的字节码的行号指示器
- 不会出现OutOfMemoryError情况
- 以一种数据结构的形式放置于内存中

**注意**：程序计数器是唯一一个不会出现 `OutOfMemoryError` 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。

### Java虚拟机栈(Java Virtual Machine Stacks)

与程序计数器一样，Java虚拟机栈（Java Virtual Machine Stacks）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于**存储局部变量表、操作数栈、动态链接、方法出口**等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

经常有人把Java内存区分为堆内存（Heap）和栈内存（Stack），这种分法比较粗糙，Java内存区域的划分实际上远比这复杂。这种划分方式的流行只能说明大多数程序员最关注的、与对象内存分配关系最密切的内存区域是这两块。其中所指的“堆”笔者在后面会专门讲述，而所指的“栈”就是现在讲的虚拟机栈，或者说是虚拟机栈中局部变量表部分。

**局部变量表**存放了编译期可知的各种**基本数据类型**（boolean、byte、char、short、int、float、long、double）、**对象引用**（reference类型，它不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

Slot

我们还听到很多博客讲述这个区域的时候有用到 slot这个概念，那个这个slot是什么呢？

我们刚刚说到栈帧的数据结构，他存储的重要信息，其中有一个是局部变量表，而这个局部变量表也是由一个个小的数据结构组成，他就叫局部变量空间（slot）。你也可以把它当成是一个空间大小的度量单位，就像字节一样，因为存储的数据的大小也是用这个衡量的。

局部变量表存放了编译期可知的各种基本数据类型(boolean、byte、char、short、int、float、long、double)、对象引用(reference类型)和returnAddress类型。

我们看到 int，short等只占用了一个 slot，但是double占用了两个slot，（long没有画出，但他也占用两个slot）在java虚拟机中确实是这样，double和long类型比较特殊，在其他方面，存储long和double的时候，别的数据一般使用一个单位的空间就行，他们就需要两个。

其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

在Java虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），如果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常。

![JVM-栈](F:\work\openGuide\JVM\png\JVM-栈.png)

JAVA虚拟机栈（Java Virtual Machine Stacks）是每个线程有一个私有的栈，随着线程的创建而创建，其生命周期与线程同进同退。栈里面存着的是一种叫“栈帧”的东西，每个Java方法在被调用的时候都会创建一个栈帧，一旦完成调用，则出栈。所有的的栈帧都出栈后，线程也就完成了使命。栈帧中存放了局部变量表（基本数据类型和对象引用）、操作数栈、动态链接(指向当前方法所属的类的运行时常量池的引用等)、方法出口(方法返回地址)、和一些额外的附加信息。栈的大小可以固定也可以动态扩展。当栈调用深度大于JVM所允许的范围，会抛出StackOverflowError的错误，不过这个深度范围不是一个恒定的值。

- 线程私有，生命周期与线程相同
- java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧，存储局部变量表(基本类型、对象引用)、操作数栈、动态链接、方法出口等信息
- `StackOverflowError`：当线程请求的栈深度大于虚拟机所允许的深度
- `OutOfMemoryError`：如果栈的扩展时无法申请到足够的内存

可以通过 -Xss 这个虚拟机参数来指定每个线程的 Java 虚拟机栈内存大小，在 JDK 1.4 中默认为 256K，而在 JDK 1.5+ 默认为 1M：

```java
java -Xss2M HackTheJava
```

> **相关参数：**
>
> -Xss：设置方法栈的最大值

### 本地方法栈(Native Method Stack)

本地方法栈（Native Method Stack）与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。在虚拟机规范中对本地方法栈中方法使用的语言、使用方式与数据结构并没有强制规定，因此具体的虚拟机可以自由实现它。甚至有的虚拟机（譬如Sun HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。

本地方法栈（Native Method Stacks）与Java栈的作用和原理非常相似。区别只不过是Java栈是为执行Java方法服务的，而本地方法栈则是为执行本地方法（Native Method）服务的。在JVM规范中，并没有对本地方法栈的具体实现方法以及数据结构作强制规定，虚拟机可以自由实现它。在HotSopt虚拟机中直接就把本地方法栈和Java栈合二为一。

### Java堆(Java Heap)

对于大多数应用来说，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。这一点在Java虚拟机规范中的描述是：所有的对象实例以及数组都要在堆上分配，但是随着JIT编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生，所有的对象都分配在堆上也渐渐变得不是那么“绝对”了。

Java堆是垃圾收集器管理的主要区域，因此很多时候也被称做“GC堆”（Garbage Collected Heap，幸好国内没翻译成“垃圾堆”）。从内存回收的角度来看，由于现在收集器基本都采用分代收集算法，所以Java堆中还可以细分为：新生代和老年代；再细致一点的有Eden空间、From Survivor空间、To Survivor空间等。从内存分配的角度来看，线程共享的Java堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer,TLAB）。不过无论如何划分，都与存放内容无关，无论哪个区域，存储的都仍然是对象实例，进一步划分的目的是为了更好地回收内存，或者更快地分配内存。在本章中，我们仅仅针对内存区域的作用进行讨论，Java堆中的上述各个区域的分配、回收等细节将是第3章的主题。

根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

堆内存（JAVA Heap）。是被线程共享的一块内存区域，创建的对象和数组都保存在 Java 堆内存中，也是垃圾收集器进行垃圾收集的最重要的内存区域。由于现代 VM 采用**分代收集算法**, 因此 Java 堆从 GC 的角度还可以细分为: **新生代**(Eden区、From Survivor 区和 To Survivor 区)和老年代。

- **线程共享**
- **主要用于存储JAVA实例或对象**
- **GC发生的主要区域**
- **是Java虚拟机所管理的内存中最大的一块**
- **当堆中没有内存能完成实例分配，且堆也无法再扩展，则会抛出OutOfMemoryError异常**

> **相关参数：**
>
> -Xms：设置堆内存初始大小
>
> -Xmx：设置堆内存最大值
>
> -XX:MaxTenuringThreshold：设置对象在新生代中存活的次数
>
> -XX:PretenureSizeThreshold：设置超过指定大小的大对象直接分配在旧生代中
>
> **新生代相关参数**（注意：当新生代设置得太小时，也可能引发大对象直接分配到旧生代）：
>
> -Xmn：设置新生代内存大小
>
> -XX:SurvivorRatio：设置Eden与Survivor空间的大小比例

### 方法区(Method Area)

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储**已被虚拟机加载的类信息、常量、静态变量、方法**、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。

对于习惯在HotSpot虚拟机上开发、部署程序的开发者来说，很多人都更愿意把方法区称为“永久代”（Permanent Generation），本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队选择把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已，这样HotSpot的垃圾收集器可以像管理Java堆一样管理这部分内存，能够省去专门为方法区编写内存管理代码的工作。对于其他虚拟机（如BEA JRockit、IBM J9等）来说是不存在永久代的概念的。原则上，如何实现方法区属于虚拟机实现细节，不受虚拟机规范约束，但使用永久代来实现方法区，现在看来并不是一个好主意，因为这样更容易遇到内存溢出问题（永久代有-XX：MaxPermSize的上限，J9和JRockit只要没有触碰到进程可用内存的上限，例如32位系统中的4GB，就不会出现问题），而且有极少数方法（例如String.intern()）会因这个原因导致不同虚拟机下有不同的表现。因此，对于HotSpot虚拟机，根据官方发布的路线图信息，现在也有放弃永久代并逐步改为采用Native Memory来实现方法区的规划了，在目前已经发布的JDK 1.7的HotSpot中，已经把原本放在永久代的字符串常量池移出。

Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入了方法区就如永久代的名字一样“永久”存在了。这区域的内存回收目标主要是针对常量池的回收和对类型的卸载，一般来说，这个区域的回收“成绩”比较难以令人满意，尤其是类型的卸载，条件相当苛刻，但是这部分区域的回收确实是必要的。在Sun公司的BUG列表中，曾出现过的若干个严重的BUG就是由于低版本的HotSpot虚拟机对此区域未完全回收而导致内存泄漏。根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

方法区（Method Area）用于存放**虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码**等数据。

- **又称之为：非堆（Non-Heap）或 永久区**
- **线程共享**
- **主要存储：类的类型信息、常量池（Runtime Constant Pool）、字段信息、方法信息、类变量和Class类的引用等**
- **Java虚拟机规范规定：当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常**

> **相关参数：**
>
> -XX:PermSize：设置Perm区的初始大小
>
> -XX:MaxPermSize：设置Perm区的最大值

注意:

+ 在Java8中已经使用元空间来代替永久代，也就是在Java8中已经没有永久代了。类似-XX:MaxPermSize这些设置永久代内存大小的参数均已失效了。
  - 为什么元空间:`类及相关的元数据的生命周期与类加载器的一致`
    - 每个加载器有专门的存储空间
    - 只进行线性分配
    - 不会单独回收某个类
    - 省掉了GC扫描及压缩的时间
    - 元空间里的对象的位置是固定的
    - 如果GC发现某个类加载器不再存活了，会把相关的空间整个回收掉  
+ JDK 1.7 的HotSpot中，`将原本放在永久代的字符串常量池移出了`。

元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中，而是使用本地内存**。因此，默认情况下，元空间的大小仅受本地内存限制，但可以通过以下参数来指定元空间的大小

- -XX:MetaspaceSize，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值
- -XX:MaxMetaspaceSize，最大空间，默认是没有限制的。
- -XX:MinMetaspaceFreeRatio，在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集
- -XX:MaxMetaspaceFreeRatio，在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集

### 运行时常量池(Runtime Constant Pool)

运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。

Java虚拟机对Class文件每一部分（自然也包括常量池）的格式都有严格规定，每一个字节用于存储哪种数据都必须符合规范上的要求才会被虚拟机认可、装载和执行，但对于运行时常量池，Java虚拟机规范没有做任何细节的要求，不同的提供商实现的虚拟机可以按照自己的需要来实现这个内存区域。不过，一般来说，除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法。

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

**JVM常量池**

JVM常量池主要分为**Class文件常量池、运行时常量池、全局字符串常量池、以及基本类型包装类对象常量池**。

- **Class文件常量池**：class文件是一组以字节为单位的二进制数据流，在java代码的编译期间，我们编写的java文件就被编译为.class文件格式的二进制数据存放在磁盘中，其中就包括class文件常量池。
- **运行时常量池**：运行时常量池相对于class常量池一大特征就是具有动态性，java规范并不要求常量只能在运行时才产生，也就是说运行时常量池的内容并不全部来自class常量池，在运行时可以通过代码生成常量并将其放入运行时常量池中，这种特性被用的最多的就是String.intern()。
- **全局字符串常量池**：字符串常量池是JVM所维护的一个字符串实例的引用表，在HotSpot VM中，它是一个叫做StringTable的全局表。在字符串常量池中维护的是字符串实例的引用，底层C++实现就是一个Hashtable。这些被维护的引用所指的字符串实例，被称作”被驻留的字符串”或”interned string”或通常所说的”进入了字符串常量池的字符串”。
- **基本类型包装类对象常量池**：java中基本类型的包装类的大部分都实现了常量池技术，这些类是Byte,Short,Integer,Long,Character,Boolean,另外两种浮点数类型的包装类则没有实现。另外上面这5种整型的包装类也只是在对应值小于等于127时才可使用对象池，也即对象不负责创建和管理大于127的这些类的对象。

### 直接内存(Direct Memory)

直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现，所以我们放到这里一起讲解。

在JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，肯定还是会受到本机总内存（包括RAM以及SWAP区或者分页文件）大小以及处理器寻址空间的限制。服务器管理员在配置虚拟机参数时，会根据实际内存设置-Xmx等参数信息，但经常忽略直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常。

### 总结

![JVM内存结构（JDK1.6）](png\JVM内存结构（JDK1.6）.png)



![JVM内存结构（JDK1.7）](png\JVM内存结构（JDK1.7）.png)

![JVM内存结构（JDK1.8）](png\JVM内存结构（JDK1.8）.png)



**线程隔离数据区：**

- **程序计数器：** 一块较小的内存空间，存储当前线程所执行的字节码行号指示器
- **虚拟机栈：** 里面的元素叫栈帧，存储 **局部变量表、操作栈、动态链接、方法返回地址** 等
- **本地方法栈：** 为虚拟机使用到的本地Native方法服务时的栈帧，和虚拟机栈类似

**线程共享数据区：**

- **方法区：** 存储已被虚拟机加载的**类信息、常量、静态变量、即时编译器编译后的代码**等数据
- **堆：** 唯一目的就是存放**对象的实例**，是垃圾回收管理器的主要区域
- **元数据区**：常量池、方法元信息、



## HotSpot虚拟机对象探秘

介绍完Java虚拟机的运行时数据区之后，我们大致知道了虚拟机内存的概况，读者了解了内存中放了些什么后，也许就会想更进一步了解这些虚拟机内存中的数据的其他细节，譬如它们是如何创建、如何布局以及如何访问的。对于这样涉及细节的问题，必须把讨论范围限定在具体的虚拟机和集中在某一个内存区域上才有意义。基于实用优先的原则，笔者以常用的虚拟机HotSpot和常用的内存区域Java堆为例，深入探讨HotSpot虚拟机在Java堆中对象分配、布局和访问的全过程。

### 对象的创建

Java是一门面向对象的编程语言，在Java程序运行过程中无时无刻都有对象被创建出来。在语言层面上，创建对象（例如克隆、反序列化）通常仅仅是一个new关键字而已，而在虚拟机中，对象（文中讨论的对象限于普通Java对象，不包括数组和Class对象等）的创建又是怎样一个过程呢？

虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程，后续将探讨这部分内容的细节。

在类加载检查通过后，接下来**虚拟机将为新生对象分配内存**。对象所需内存的大小在类加载完成后便可完全确定（如何确定将在后续中介绍），为对象分配空间的任务等同于把一块确定大小的内存从Java堆中划分出来。假设Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”（Bump the Pointer）。如果Java堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”（Free List）。选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。因此，在使用Serial、ParNew等带Compact过程的收集器时，系统采用的分配算法是指针碰撞，而使用CMS这种基于Mark-Sweep算法的收集器时，通常采用空闲列表。

除如何划分可用空间之外，还有另外一个需要考虑的问题是对象创建在虚拟机中是非常频繁的行为，即使是仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的情况。解决这个问题有两种方案，一种是对分配内存空间的动作进行同步处理——实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性；另一种是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer,TLAB）。哪个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新的TLAB时，才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX：+/-UseTLAB参数来设定。

内存分配完成后，虚拟机需要将分配到的**内存空间都初始化为零值**（不包括对象头），如果使用TLAB，这一工作过程也可以提前至TLAB分配时进行。这一步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

接下来，虚拟机要对对象进行必要的设置，**例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头**（Object Header）之中。根据虚拟机当前的运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。关于对象头的具体内容，稍后再做详细介绍。

在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从Java程序的视角来看，对象创建才刚刚开始——＜init＞方法还没有执行，所有的字段都还为零。所以，一般来说（由字节码中是否跟随invokespecial指令所决定），执行new指令之后会接着执行＜init＞方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

下面的代码清单是HotSpot虚拟机bytecodeInterpreter.cpp中的代码片段（这个解释器实现很少有机会实际使用，因为大部分平台上都使用模板解释器；当代码通过JIT编译器执行时差异就更大了。不过，这段代码用于了解HotSpot的运作过程是没有什么问题的）。

```
//确保常量池中存放的是已解释的类
if(!constants->tag_at(index).is_unresolved_klass()){
//断言确保是klassOop和instanceKlassOop（这部分下一节介绍）
oop entry=（klassOop）*constants-＞obj_at_addr（index）;
assert（entry-＞is_klass（），"Should be resolved klass"）;
klassOop k_entry=（klassOop）entry;
assert（k_entry-＞klass_part（）-＞oop_is_instance（），"Should be instanceKlass"）;
instanceKlass * ik=（instanceKlass*）k_entry-＞klass_part（）;
//确保对象所属类型已经经过初始化阶段
if(ik->is_initialized()&&ik->can_be_fastpath_allocated())
{
//取对象长度
size_t obj_size=ik->size_helper();
oop result=NULL；
//记录是否需要将对象所有字段置零值
bool need_zero=!ZeroTLAB；
//是否在TLAB中分配对象
if(UseTLAB){
result=（oop）THREAD->tlab().allocate(obj_size);
}
if（result==NULL）{
need_zero=true；
//直接在eden中分配对象
retry：
HeapWord * compare_to=*Universe：heap（）-＞top_addr（）；
HeapWord * new_top=compare_to+obj_size；
/*cmpxchg是x86中的CAS指令，这里是一个C++方法，通过CAS方式分配空间，如果并发失败，
转到retry中重试，直至成功分配为止*/
if（new_top＜=*Universe：heap（）-＞end_addr（））{
if（Atomic：cmpxchg_ptr（new_top,Universe：heap（）-＞top_addr（），compare_to）！=compare_to）{
goto retry；
}
result=（oop）compare_to；
}
}
if（result！=NULL）{
//如果需要，则为对象初始化零值
if（need_zero）{
HeapWord * to_zero=（HeapWord*）result+sizeof（oopDesc）/oopSize；
obj_size-=sizeof（oopDesc）/oopSize；
if（obj_size＞0）{
memset（to_zero，0，obj_size * HeapWordSize）；
}
}
//根据是否启用偏向锁来设置对象头信息
if（UseBiasedLocking）{
result-＞set_mark（ik-＞prototype_header（））；
}else{
result-＞set_mark（markOopDesc：prototype（））；
}r
esult-＞set_klass_gap（0）；
result-＞set_klass（k_entry）；
//将对象引用入栈，继续执行下一条指令
SET_STACK_OBJECT（result，0）；
UPDATE_PC_AND_TOS_AND_CONTINUE（3，1）；
}
}
}
```

#### **请简单阐述一下对象的创建过程？**

```
    public static void main(String[] args) {
        Object object=new Object();
        String string=new String();
    }
```

main方法中创建了两个对象执行过程在右边字节码中展示完全一致new、dup、invokespecial、astore四个步骤　　

　　1、new，虚拟机指令为对象分配内存并在栈顶压入了指向这段内存的地址供后续操作来调用

　　2、dup，其实就是一个复制操作，其作用是把栈顶的内容复制一份再压入栈。jvm为什么要这么做呢

　　　　这完全是jvm自己编译优化的做法，再后续操作之前虚拟机自己会调用一次。我们都知道对象都有一个this的关键字指向对象本身，this是什么时候赋值的呢，就是这个时候

　　　　至于另一个引用当然是赋值给方法中的变量了

　　3、invokespecial，该过程是对实例对象进行初始化，第一步分配内存后对象内的实例变量都是初始值，在该步骤才会初始化对象内的实例变量

　　4、astore，方法内的变量指向内存中的对象

### 对象的内存布局

#### 对象内存构成

Java 中通过 new 关键字创建一个类的实例对象，对象存于内存的堆中并给其分配一个内存地址，那么是否想过如下这些问题：

- **这个实例对象是以怎样的形态存在内存中的?**
- **一个Object对象在内存中占用多大?**
- **对象中的属性是如何在内存中分配的?**

在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域：

- **对象头（object header）**：包括了关于堆对象的布局、类型、GC状态、同步状态和标识哈希码的基本信息。Java对象和vm内部对象都有一个共同的对象头格式。
- **实例数据（Instance Data）**：主要是存放类的数据信息，父类的信息，对象字段属性信息。
- **对齐填充（Padding）**：为了字节对齐，填充的数据，不是必须的。

#### 对象头

参考网址：http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html

```
object header
Common structure at the beginning of every GC-managed heap object. (Every oop points to an object header.) Includes fundamental information about the heap object's layout, type, GC state, synchronization state, and identity hash code. Consists of two words. In arrays it is immediately followed by a length field. Note that both Java objects and VM-internal objects have a common object header format.
```

我们可以在Hotspot官方文档中找到它的描述。从中可以发现，它是Java对象和虚拟机内部对象都有的共同格式，由两个字(计算机术语)组成。另外，如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。

它里面提到了对象头由两个**字**组成，这两个**字**是什么呢？我们还是在上面的那个Hotspot官方文档中往上看，可以发现还有另外两个名词的定义解释，分别是 **mark word** 和 **klass pointer**。

```
klass pointer
The second word of every object header. Points to another object (a metaobject) which describes the layout and behavior of the original object. For Java objects, the "klass" contains a C++ style "vtable".
mark word
The first word of every object header. Usually a set of bitfields including synchronization state and identity hash code. May also be a pointer (with characteristic low bit encoding) to synchronization related information. During GC, may contain GC state bits.
```

##### Mark Word

用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等。

这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和64bit，官方称它为“Mark Word”。对象需要存储的运行时数据很多，其实已经超出了32位、64位Bitmap结构所能记录的限度，但是对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。

我们打开[openjdk的源码包](https://download.java.net/openjdk/jdk8/promoted/b132/openjdk-8-src-b132-03_mar_2014.zip)，对应路径`/openjdk/hotspot/src/share/vm/oops`，Mark Word对应到C++的代码`markOop.hpp`，可以从注释中看到它们的组成。代码为HotSpot虚拟机markOop.cpp中的代码（注释）片段，它描述了32bit下MarkWord的存储状态。

```
//Bit-format of an object header(most significant first,big endian layout below):
//32 bits:
//--------
//hash：25------------>|age：4 biased_lock：1 lock：2（normal object）
//JavaThread*：23 epoch：2 age：4 biased_lock：1 lock：2（biased object）
//size：32------------------------------------------>|（CMS free block）
//PromotedObject*：29---------->|promo_bits：3----->|（CMS promoted object）
```

Mark Word在不同的锁状态下存储的内容不同，在32位JVM中是这么存的

![JVMMarkWord32](png\JVMMarkWord32.png)

在64位JVM中是这么存的:

![JVMMarkWord64](png\JVMMarkWord64.png)

虽然它们在不同位数的JVM中长度不一样，但是基本组成内容是一致的。

- **锁标志位（lock）**：区分锁状态，11时表示对象待GC回收状态, 只有最后2位锁标识(11)有效。
- **biased_lock**：是否偏向锁，由于无锁和偏向锁的锁标识都是 01，没办法区分，这里引入一位的偏向锁标识位。
- **分代年龄（age）**：表示对象被GC的次数，当该次数到达阈值的时候，对象就会转移到老年代。
- **对象的hashcode（hash）**：运行期间调用System.identityHashCode()来计算，延迟计算，并把结果赋值到这里。当对象加锁后，计算的结果31位不够表示，在偏向锁，轻量锁，重量锁，hashcode会被转移到Monitor中。
- **偏向锁的线程ID（JavaThread）**：偏向模式的时候，当某个线程持有对象的时候，对象这里就会被置为该线程的ID。 在后面的操作中，就无需再进行尝试获取锁的动作。
- **epoch**：偏向锁在CAS锁操作过程中，偏向性标识，表示对象更偏向哪个锁。
- **ptr_to_lock_record**：轻量级锁状态下，指向栈中锁记录的指针。当锁获取是无竞争的时，JVM使用原子操作而不是OS互斥。这种技术称为轻量级锁定。在轻量级锁定的情况下，JVM通过CAS操作在对象的标题字中设置指向锁记录的指针。
- **ptr_to_heavyweight_monitor**：重量级锁状态下，指向对象监视器Monitor的指针。如果两个不同的线程同时在同一个对象上竞争，则必须将轻量级锁定升级到Monitor以管理等待的线程。在重量级锁定的情况下，JVM在对象的ptr_to_heavyweight_monitor设置指向Monitor的指针。

##### Klass Pointer

即类型指针，是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。对象头的另外一部分是类型指针，即**对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例**。并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说，查找对象的元数据信息并不一定要经过对象本身

#### 实例数据

如果对象有属性字段，则这里会有数据信息。如果对象无属性字段，则这里就不会有数据。根据字段类型的不同占不同的字节，例如boolean类型占1个字节，int类型占4个字节等等；实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都需要记录起来。这部分的存储顺序会受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在Java源码中定义顺序的影响。HotSpot虚拟机默认的分配策略为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers），从分配策略中可以看出，相同宽度的字段总是被分配到一起。在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。如果CompactFields参数值为true（默认为true），那么子类之中较窄的变量也可能会插入到父类变量的空隙之中。

#### 对齐数据

对象可以有对齐数据也可以没有。默认情况下，Java虚拟机堆中对象的起始地址需要对齐至8的倍数。如果一个对象用不到8N个字节则需要对其填充，以此来补齐对象头和实例数据占用内存之后剩余的空间大小。如果对象头和实例数据已经占满了JVM所分配的内存空间，那么就不用再进行对齐填充了。

所有的对象分配的字节总SIZE需要是8的倍数，如果前面的对象头和实例数据占用的总SIZE不满足要求，则通过对齐数据来填满。

**为什么要对齐数据**？字段内存对齐的其中一个原因，是让字段只出现在同一CPU的缓存行中。如果字段不是对齐的，那么就有可能出现跨缓存行的字段。也就是说，该字段的读取可能需要替换两个缓存行，而该字段的存储也会同时污染两个缓存行。这两种情况对程序的执行效率而言都是不利的。其实对其填充的最终目的是为了计算机高效寻址。

分对齐填充并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或者2倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

### 对象内存实战

概念的东西是抽象的，你说它是这样组成的，就真的是吗？学习是需要持怀疑的态度的，任何理论和概念只有自己证实和实践之后才能接受它。还好 openjdk 给我们提供了一个工具包，可以用来获取对象的信息和虚拟机的信息，我们只需引入 jol-core 依赖，如下

```
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.8</version>
</dependency>
```

jol-core 常用的三个方法

- ClassLayout.parseInstance(object).toPrintable()：查看对象内部信息.
- GraphLayout.parseInstance(object).toPrintable()：查看对象外部信息，包括引用的对象.
- GraphLayout.parseInstance(object).totalSize()：查看对象总大小.

#### 普通对象

为了简单化，我们不用复杂的对象，自己创建一个类 D，先看无属性字段的时候

```java
public class D {
}
```

通过 jol-core 的 api，我们将对象的内部信息打印出来

```java
public static void main(String[] args) {
    D d = new D();
    System.out.println(ClassLayout.parseInstance(d).toPrintable());
}
```

最后的打印结果为

```
com.demo.D object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)  //Mark                          01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)   //Kclass                        05 c4 00 20 (00000101 11000100 00000000 00100000) (536921093)
     12     4        (loss due to the next object alignment) //padding
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

可以看到有 OFFSET、SIZE、TYPE DESCRIPTION、VALUE 这几个名词头，它们的含义分别是

- OFFSET：偏移地址，单位字节；
- SIZE：占用的内存大小，单位为字节；
- TYPE DESCRIPTION：类型描述，其中object header为对象头；
- VALUE：对应内存中当前存储的值，二进制32位；

可以看到，d对象实例共占据16byte，对象头（object header）占据12byte（96bit），其中 mark word占8byte（64bit），klass pointe 占4byte，另外剩余4byte是填充对齐的。

网址：https://docs.oracle.com/javase/7/docs/technotes/guides/vm/performance-enhancements-7.html

这里由于默认开启了**指针压缩** ，所以对象头占了12byte，具体的指针压缩的概念这里就不再阐述了，感兴趣的读者可以自己查阅下[官方文档](https://docs.oracle.com/javase/7/docs/technotes/guides/vm/performance-enhancements-7.html)。jdk8版本是默认开启指针压缩的，可以通过配置vm参数开启关闭指针压缩，`-XX:-UseCompressedOops`。

如果关闭指针压缩重新打印对象的内存布局，可以发现总SIZE变大了，从下图中可以看到，对象头所占用的内存大小变为16byte（128bit），其中 mark word占8byte，klass pointe 占8byte，无对齐填充。

开启指针压缩可以减少对象的内存使用。从两次打印的D对象布局信息来看，关闭指针压缩时，对象头的SIZE增加了4byte，这里由于D对象是无属性的，读者可以试试增加几个属性字段来看下，这样会明显的发现SIZE增长。因此开启指针压缩，理论上来讲，大约能节省百分之五十的内存。jdk8及以后版本已经默认开启指针压缩，无需配置。

#### 数组对象

上面使用的是普通对象，我们来看下数组对象的内存布局，比较下有什么异同

```java
public static void main(String[] args) {
    int[] a = {1};
    System.out.println(ClassLayout.parseInstance(a).toPrintable());
}
```

打印的内存布局信息，如下

```
[I object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)   //Mark                        00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           6d 01 00 20 (01101101 00000001 00000000 00100000) (536871277) //Kclass
     12     4        (object header)    //array length                       01 00 00 00 (00000001 00000000 00000000 00000000) (1)
     16     4    int [I.<elements>        //实例数据                     N/A
     20     4        (loss due to the next object alignment)  //对齐填充
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

可以看到这时总SIZE为共24byte，对象头占16byte，其中Mark Work占8byte，Klass Point 占4byte，array length 占4byte，因为里面只有一个int 类型的1，所以数组对象的实例数据占据4byte，剩余对齐填充占据4byte。

#### 结尾

经过以上的内容我们了解了对象在内存中的布局，了解对象的内存布局和对象头的概念，特别是对象头的Mark Word的内容，在我们后续分析 synchronize 锁优化 和 JVM 垃圾回收年龄代的时候会有很大作用。

JVM中大家是否还记得对象在Suvivor中每熬过一次MinorGC，年龄就增加1，当它的年龄增加到一定程度后就会被晋升到老年代中，这个次数默认是15岁，有想过为什么是15吗？在Mark Word中可以发现标记对象分代年龄的分配的空间是4bit，而4bit能表示的最大数就是2^4-1 = 15。

### 对象的访问定位

建立对象是为了使用对象，我们的Java程序需要通过栈上的reference数据来操作堆上的具体对象。由于reference类型在Java虚拟机规范中只规定了一个指向对象的引用，并没有定义这个引用应该通过何种方式去定位、访问堆中的对象的具体位置，所以对象访问方式也是取决于虚拟机实现而定的。目前主流的访问方式有使用句柄和直接指针两种。

如果使用句柄访问的话，那么Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息，如图。 

![JVM-对象的访问定位](png/JVM-对象的访问定位.png)

如果使用直接指针访问，那么Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址，如图所示。 
![JVM-对象访问位置1](png\JVM-对象访问位置1.png)

这两种对象访问方式各有优势，使用句柄来访问的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要修改。

使用直接指针访问方式的最大好处就是速度更快，它节省了一次指针定位的时间开销，由于对象的访问在Java中非常频繁，因此这类开销积少成多后也是一项非常可观的执行成本。就本书讨论的主要虚拟机Sun HotSpot而言，它是使用第二种方式进行对象访问的，但从整个软件开发的范围来看，各种语言和框架使用句柄来访问的情况也十分常见。

### JVM运行时内存

JVM运行时内存又称堆内存(Heap)。Java 堆从 GC 的角度还可以细分为: 新生代(Eden 区、From Survivor 区和 To Survivor 区)和老年代。

![RuntimeDataArea](png\RuntimeDataArea.png)

![JVM堆内存划分](png/JVM堆内存划分.png)

当代主流虚拟机（Hotspot VM）的垃圾回收都采用“分代回收”的算法。“分代回收”是基于这样一个事实：对象的生命周期不同，所以针对不同生命周期的对象可以采取不同的回收方式，以便提高回收效率。Hotspot VM将内存划分为不同的物理区，就是“分代”思想的体现。



**一个对象从出生到消亡**

![JVM对象申请空间流程](png/JVM对象申请空间流程.png)

一个对象产生之后首先进行栈上分配，栈上如果分配不下会进入伊甸区，伊甸区经过一次垃圾回收之后进入surivivor区，survivor区在经过一次垃圾回收之后又进入另外一个survivor，与此同时伊甸区的某些对象也跟着进入另外一个survivot，什么时候年龄够了就会进入old区，这是整个对象的一个逻辑上的移动过程。



#### 新生代（Young Generation）

**主要是用来存放新生的对象**。一般占据堆的1/3空间。由于频繁创建对象，所以新生代会频繁触发MinorGC进行垃圾回收。新生代又分为 Eden区、ServivorFrom、ServivorTo三个区。

- **Eden区**：Java新对象的出生地（如果新创建的对象占用内存很大，则直接分配到老年代）。当Eden区内存不够的时候就会触发MinorGC，对新生代区进行一次垃圾回收
- **ServivorTo**：保留了一次MinorGC过程中的幸存者
- **ServivorFrom**：上一次GC的幸存者，作为这一次GC的被扫描者



**MinorGC流程**

- **MinorGC采用复制算法**
- 首先把Eden和ServivorFrom区域中存活的对象复制到ServicorTo区域（如果有对象的年龄以及达到了老年的标准，则复制到老年代区），同时把这些对象的年龄+1（如果ServicorTo不够位置了就放到老年区）
- 然后清空Eden和ServicorFrom中的对象
- 最后ServicorTo和ServicorFrom互换，原ServicorTo成为下一次GC时的ServicorFrom区



**为什么 Survivor 分区不能是 0 个？**

如果 Survivor 是 0 的话，也就是说新生代只有一个 Eden 分区，每次垃圾回收之后，存活的对象都会进入老生代，这样老生代的内存空间很快就被占满了，从而触发最耗时的 Full GC ，显然这样的收集器的效率是我们完全不能接受的。

**为什么 Survivor 分区不能是 1 个？**

如果 Survivor 分区是 1 个的话，假设我们把两个区域分为 1:1，那么任何时候都有一半的内存空间是闲置的，显然空间利用率太低不是最佳的方案。

但如果设置内存空间的比例是 8:2 ，只是看起来似乎“很好”，假设新生代的内存为 100 MB（ Survivor 大小为 20 MB ），现在有 70 MB 对象进行垃圾回收之后，剩余活跃的对象为 15 MB 进入 Survivor 区，这个时候新生代可用的内存空间只剩了 5 MB，这样很快又要进行垃圾回收操作，显然这种垃圾回收器最大的问题就在于，需要频繁进行垃圾回收。

**为什么 Survivor 分区是 2 个？**

如果Survivor分区有2个分区，我们就可以把 Eden、From Survivor、To Survivor 分区内存比例设置为 8:1:1 ，那么任何时候新生代内存的利用率都 90% ，这样空间利用率基本是符合预期的。再者就是虚拟机的大部分对象都符合“朝生夕死”的特性，所以每次新对象的产生都在空间占比比较大的Eden区，垃圾回收之后再把存活的对象方法存入Survivor区，如果是 Survivor区存活的对象，那么“年龄”就+1，当年龄增长到15（可通过 -XX:+MaxTenuringThreshold 设定）对象就升级到老生代。



**总结**

根据上面的分析可以得知，当新生代的 Survivor 分区为 2 个的时候，不论是空间利用率还是程序运行的效率都是最优的，所以这也是为什么 Survivor 分区是 2 个的原因了。



#### 老年代（Old Generation）

**主要存放应用程序中生命周期长的内存对象**。老年代的对象比较稳定，所以MajorGC不会频繁执行。在进行MajorGC前一般都先进行了一次MinorGC，使得有新生代的对象晋身入老年代，导致空间不够用时才触发。当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次MajorGC进行垃圾回收腾出空间。



**MajorGC流程**

MajorGC采用标记—清除算法。首先扫描一次所有老年代，标记出存活的对象，然后回收没有标记的对象。MajorGC的耗时比较长，因为要扫描再回收。MajorGC会产生内存碎片，为了减少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。当老年代也满了装不下的时候，就会抛出OOM（Out of Memory）异常。



#### 永久区（Perm Generation）

指内存的永久保存区域，**主要存放元数据**，例如Class、Method的元信息，与垃圾回收要回收的Java对象关系不大。相对于新生代和年老代来说，该区域的划分对垃圾回收影响比较小。GC不会在主程序运行期对永久区域进行清理，所以这也导致了永久代的区域会随着加载的Class的增多而胀满，最终抛出OOM异常。



**JAVA8与元数据**

在Java8中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中，而是使用本地内存**。因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入Native Memory，字符串池和类的静态变量放入java堆中，这样可以加载多少类的元数据就不再由MaxPermSize控制，而由系统的实际可用空间来控制。

### 内存分配策略

堆内存常见的分配测试如下：

- 对象优先在Eden区分配
- 大对象直接进入老年代
- 长期存活的对象将进入老年代

| **参数**                        | **说明信息**                                                 |
| :------------------------------ | ------------------------------------------------------------ |
| -Xms                            | 初始堆大小。如：-Xms256m                                     |
| -Xmx                            | 最大堆大小。如：-Xmx512m                                     |
| -Xmn                            | 新生代大小。通常为Xmx的1/3或1/4。新生代=Eden+2个Survivor空间。实际可用空间为=Eden+1个Survivor，即 90% |
| -Xss                            | JDK1.5+每个线程堆栈大小为 1M，一般来说如果栈不是很深的话， 1M 是绝对够用了的 |
| -XX:NewRatio                    | 新生代与老年代的比例。如–XX:NewRatio=2，则新生代占整个堆空间的1/3，老年代占2/3 |
| -XX:SurvivorRatio               | 新生代中Eden与Survivor的比值。默认值为 8，即Eden占新生代空间的8/10，另外两个Survivor各占1/10 |
| -XX:PermSize                    | 永久代（方法区）的初始大小                                   |
| -XX:MaxPermSize                 | 永久代（方法区）的最大值                                     |
| -XX:+PrintGCDetails             | 打印GC信息                                                   |
| -XX:+HeapDumpOnOutOfMemoryError | 让虚拟机在发生内存溢出时Dump出当前的内存堆转储快照，以便分析用 |



**参数基本策略**

各分区的大小对GC的性能影响很大。如何将各分区调整到合适的大小，分析活跃数据的大小是很好的切入点。

**活跃数据的大小**：指应用程序稳定运行时长期存活对象在堆中占用的空间大小，即Full GC后堆中老年代占用空间的大小。

可以通过GC日志中Full GC之后老年代数据大小得出，比较准确的方法是在程序稳定后，多次获取GC数据，通过取平均值的方式计算活跃数据的大小。活跃数据和各分区之间的比例关系如下：

| 空间   | 倍数                                    |
| ------ | --------------------------------------- |
| 总大小 | **3-4** 倍活跃数据的大小                |
| 新生代 | **1-1.5** 活跃数据的大小                |
| 老年代 | **2-3** 倍活跃数据的大小                |
| 永久代 | **1.2-1.5** 倍Full GC后的永久代空间占用 |

例如，根据GC日志获得老年代的活跃数据大小为300M，那么各分区大小可以设为：

> 总堆：1200MB = 300MB × 4
>
> 新生代：450MB = 300MB × 1.5
>
> 老年代： 750MB = 1200MB - 450MB

这部分设置仅仅是堆大小的初始值，后面的优化中，可能会调整这些值，具体情况取决于应用程序的特性和需求。

## 实战：OutOfMemoryError异常

在Java虚拟机规范的描述中，除了程序计数器外，虚拟机内存的其他几个运行时区域都有发生OutOfMemoryError（下文称OOM）异常的可能，本节将通过若干实例来验证异常发生的场景（代码清单2-3～代码清单2-9的几段简单代码），并且会初步介绍几个与内存相关的 
最基本的虚拟机参数。

本节内容的目的有两个：第一，通过代码验证Java虚拟机规范中描述的各个运行时区域存储的内容；第二，希望读者在工作中遇到实际的内存溢出异常时，能根据异常的信息快速判断是哪个区域的内存溢出，知道什么样的代码可能会导致这些区域内存溢出，以及出现这些异常后该如何处理。

下文代码的开头都注释了执行时所需要设置的虚拟机启动参数（注释中“VM Args”后面跟着的参数），这些参数对实验的结果有直接影响，读者调试代码的时候千万不要忽略。如果读者使用控制台命令来执行程序，那直接跟在Java命令之后书写就可以。如果读者使用Eclipse IDE，则可以参考图在Debug/Run页签中的设置。 

下文的代码都是基于Sun公司的HotSpot虚拟机运行的，对于不同公司的不同版本的虚拟机，参数和程序运行的结果可能会有所差别。

### Java堆溢出

Java堆用于存储对象实例，只要不断地创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，那么在对象数量到达最大堆的容量限制后就会产生内存溢出异常。

代码限制Java堆的大小为20MB，不可扩展（将堆的最小值-Xms参数与最大值-Xmx参数设置为一样即可避免堆自动扩展），通过参数-XX：+HeapDumpOnOutOfMemoryError可以让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照以便事后进行分析。

Java堆内存溢出异常测试

```
/**
 * VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    static class OOMObject {
    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();

        while (true) {
            list.add(new OOMObject());
        }
    }
}
```

运行结果：

```
java.lang.OutOfMemoryError :Java heap space
Dumping heap to java_pid3404.hprof.
Heap dump file created[22045981 bytes in 0.663 secs]
```

Java堆内存的OOM异常是实际应用中常见的内存溢出异常情况。当出现Java堆内存溢出时，异常堆栈信息“java.lang.OutOfMemoryError”会跟着进一步提示“Java heap space”。

要解决这个区域的异常，一般的手段是先通过内存映像分析工具（如Eclipse Memory Analyzer）对Dump出来的堆转储快照进行分析，重点是确认内存中的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）。下图显示了使用Eclipse Memory Analyzer打开的堆转储快照文件。 

如果是内存泄露，可进一步通过工具查看泄露对象到GC Roots的引用链。于是就能找到泄露对象是通过怎样的路径与GC Roots相关联并导致垃圾收集器无法自动回收它们的。掌握了泄露对象的类型信息及GC Roots引用链的信息，就可以比较准确地定位出泄露代码的位置。

如果不存在泄露，换句话说，就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数（-Xmx与-Xms），与机器物理内存对比看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗。

以上是处理Java堆内存问题的简单思路，处理这些问题所需要的知识、工具与经验是后面3章的主题。

### 虚拟机栈和本地方法栈溢出

由于在HotSpot虚拟机中并不区分虚拟机栈和本地方法栈，因此，对于HotSpot来说，虽然-Xoss参数（设置本地方法栈大小）存在，但实际上是无效的，栈容量只由-Xss参数设定。

关于虚拟机栈和本地方法栈，在Java虚拟机规范中描述了两种异常：

- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。
- 如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。

这里把异常分成两种情况，看似更加严谨，但却存在着一些互相重叠的地方：当栈空间无法继续分配时，到底是内存太小，还是已使用的栈空间太大，其本质上只是对同一件事情的两种描述而已。

在笔者的实验中，将实验范围限制于单线程中的操作，尝试了下面两种方法均无法让虚拟机产生OutOfMemoryError异常，尝试的结果都是获得StackOverflowError异常，测试代码如代码清单2-4所示。

- 使用-Xss参数减少栈内存容量。结果：抛出StackOverflowError异常，异常出现时输出的堆栈深度相应缩小。
- 定义了大量的本地变量，增大此方法帧中本地变量表的长度。结果：抛出StackOverflowError异常时输出的堆栈深度相应缩小。

虚拟机栈和本地方法栈OOM测试（仅作为第1点测试程序）

```
/**
 * VM Args：-Xss128k
 */
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

运行结果：

```
stack length :2402
Exception in thread"main"java.lang.StackOverflowError
at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.java :20 ) at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.java :21 ) at org.fenixsoft.oom.VMStackSOF.leak (WIStackSOF.iava :21 ) 
.....后续异常堆栈信息省略
```

实验结果表明：在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。

如果测试时不限于单线程，通过不断地建立线程的方式倒是可以产生内存溢出异常，如代码清单2-5所示。但是这样产生的内存溢出异常与栈空间是否足够大并不存在任何联系，或者准确地说，在这种情况下，为每个线程的栈分配的内存越大，反而越容易产生内存溢出异常。

其实原因不难理解，操作系统分配给每个进程的内存是有限制的，譬如32位的Windows限制为2GB。虚拟机提供了参数来控制Java堆和方法区的这两部分内存的最大值。剩余的内存为2GB（操作系统限制）减去Xmx（最大堆容量），再减去MaxPermSize（最大方法区容量），程序计数器消耗内存很小，可以忽略掉。如果虚拟机进程本身耗费的内存不计算在内，剩下的内存就由虚拟机栈和本地方法栈“瓜分”了。每个线程分配到的栈容量越大，可以 
建立的线程数量自然就越少，建立线程时就越容易把剩下的内存耗尽。

这一点读者需要在开发多线程的应用时特别注意，出现StackOverflowError异常时有错误堆栈可以阅读，相对来说，比较容易找到问题的所在。而且，如果使用虚拟机默认参数，栈深度在大多数情况下（因为每个方法压入栈的帧大小并不是一样的，所以只能说在大多数情况下）达到1000～2000完全没有问题，对于正常的方法调用（包括递归），这个深度应该完全够用了。但是，如果是建立过多线程导致的内存溢出，在不能减少线程数或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少栈容量来换取更多的线程。如果没有这方面的处理经验，这种通过“减少内存”的手段来解决内存溢出的方式会比较难以想到。 
创建线程导致内存溢出异常

```
/**
 * VM Args：-Xss2M （这时候不妨设大些）
 */
public class JavaVMStackOOM {

       private void dontStop() {
              while (true) {
              }
       }

       public void stackLeakByThread() {
              while (true) {
                     Thread thread = new Thread(new Runnable() {
                            @Override
                            public void run() {
                                   dontStop();
                            }
                     });
                     thread.start();
              }
       }

       public static void main(String[] args) throws Throwable {
              JavaVMStackOOM oom = new JavaVMStackOOM();
              oom.stackLeakByThread();
       }
}
```

注意，特别提示一下，如果读者要尝试运行上面这段代码，记得要先保存当前的工作。由于在Windows平台的虚拟机中，Java的线程是映射到操作系统的内核线程上的，因此上述代码执行时有较大的风险，可能会导致操作系统假死。 
运行结果：

```
Exception in thread"main"java.lang.OutOfMemoryError :unable to create new native thread
```

### 方法区和运行时常量池溢出

由于运行时常量池是方法区的一部分，因此这两个区域的溢出测试就放在一起进行。前面提到JDK 1.7开始逐步“去永久代”的事情，在此就以测试代码观察一下这件事对程序的实际影响。

String.intern（）是一个Native方法，它的作用是：如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。在JDK 1.6及之前的版本中，由于常量池分配在永久代内，我们可以通过-XX：PermSize和-XX：MaxPermSize限制方法区大小，从而间接限制其中常量池的容量，如代码清单2-6所示。

运行时常量池导致的内存溢出异常

```
/**
 * VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class RuntimeConstantPoolOOM {

    public static void main(String[] args) {
        // 使用List保持着常量池引用，避免Full GC回收常量池行为
        List<String> list = new ArrayList<String>();
        // 10MB的PermSize在integer范围内足够产生OOM了
        int i = 0; 
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
}
```

运行结果：

```
Exception in thread"main"java.lang.OutOfMemoryError :PermGen space
at java.lang.String, intern (Native Method )
at org.fenixsoft.oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:18)
```

从运行结果中可以看到，运行时常量池溢出，在OutOfMemoryError后面跟随的提示信息是“PermGen space”，说明运行时常量池属于方法区（HotSpot虚拟机中的永久代）的一部分。

而使用JDK 1.7运行这段程序就不会得到相同的结果，while循环将一直进行下去。关于这个字符串常量池的实现问题，还可以引申出一个更有意思的影响，如代码清单2-7所示。

String.intern（）返回引用的测试

```
public class RuntimeConstantPoolOOM {

    public static void main(String[] args) {
        public static void main(String[] args) {
        String str1 = new StringBuilder("中国").append("钓鱼岛").toString();
        System.out.println(str1.intern() == str1);

        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2);
    }   }
}
```

这段代码在JDK 1.6中运行，会得到两个false，而在JDK 1.7中运行，会得到一个true和一个false。产生差异的原因是：在JDK 1.6中，intern（）方法会把首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个字符串实例的引用，而由StringBuilder创建的字符串实例在Java堆上，所以必然不是同一个引用，将返回false。而JDK 1.7（以及部分其他虚拟机，例如JRockit）的intern（）实现不会再复制实例，只是在常量池中记录首次出现的实例引用，因此intern（）返回的引用和由StringBuilder创建的那个字符串实例是同一个。对str2比较返回false是因为“java”这个字符串在执行StringBuilder.toString（）之前已经出现过，字符串常量池中已经有它的引用了，不符合“首次出现”的原则，而“计算机软件”这个字符串则是首次出现的，因此返回true。

方法区用于存放Class的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。对于这些区域的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出。虽然直接使用Java SE API也可以动态产生类（如反射时的GeneratedConstructorAccessor和动态代理等），但在本次实验中操作起来比较麻烦。在代码清单2-8中，笔者借助CGLib直接操作字节码运行时生成了大量的动态类。

值得特别注意的是，我们在这个例子中模拟的场景并非纯粹是一个实验，这样的应用经常会出现在实际应用中：当前的很多主流框架，如Spring、Hibernate，在对类进行增强时，都会使用到CGLib这类字节码技术，增强的类越多，就需要越大的方法区来保证动态生成的Class可以加载入内存。另外，JVM上的动态语言（例如Groovy等）通常都会持续创建类来实现语言的动态性，随着这类语言的流行，也越来越容易遇到与代码清单2-8相似的溢出场景。

借助CGLib使方法区出现内存溢出异常

```
/**
 * VM Args： -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodAreaOOM {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }

    static class OOMObject {

    }
}
```

运行结果：

```
Caused by :java.lang.OutOfMemoryError :PermGen space
at java.lang.ClassLoader.defineClassl (Native Method)
at java.lang.ClassLoader.defineClassCond (ClassLoader. java :632 ) at java.lang.ClassLoader.defineClass (ClassLoader.java :616 )
— 8 more
```

方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾收集器回收掉，判定条件是比较苛刻的。在经常动态生成大量Class的应用中，需要特别注意类的回收状况。这类场景除了上面提到的程序使用了CGLib字节码增强和动态语言之外，常见的还有：大量JSP或动态产生JSP文件的应用（JSP第一次运行时需要编译为Java类）、基于OSGi的应用（即使是同一个类文件，被不同的加载器加载也会视为不同的类）等。

### 本机直接内存溢出

DirectMemory容量可通过-XX：MaxDirectMemorySize指定，如果不指定，则默认与Java堆最大值（-Xmx指定）一样，代码越过了DirectByteBuffer类，直接通过反射获取Unsafe实例进行内存分配（Unsafe类的getUnsafe（）方法限制了只有引导类加载器才会返回实例，也就是设计者希望只有rt.jar中的类才能使用Unsafe的功能）。因为，虽然使用DirectByteBuffer分配内存也会抛出内存溢出异常，但它抛出异常时并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配，于是手动抛出异常，真正申请分配内存的方法是unsafe.allocateMemory（）。

使用unsafe分配本机内存

```
/**
 * VM Args：-Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```

运行结果：

```
Exception in thread"main"java.lang.OutOfMemoryError at sun.misc.Unsafe .allocateMemory (Native Method ) at org. fenixsoft. oom.DMOOM.main (DMOOM.java :20 )
```

由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果读者发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。

## OOM

JVM发生OOM的九种场景如下：

**场景一：Java heap space**

> 当堆内存（Heap Space）没有足够空间存放新创建的对象时，就会抛出 `java.lang.OutOfMemoryError:Javaheap space` 错误（根据实际生产经验，可以对程序日志中的 OutOfMemoryError 配置关键字告警，一经发现，立即处理）。
>
> **原因分析**
>
> `Javaheap space` 错误产生的常见原因可以分为以下几类：
>
> - 请求创建一个超大对象，通常是一个大数组
> - 超出预期的访问量/数据量，通常是上游系统请求流量飙升，常见于各类促销/秒杀活动，可以结合业务流量指标排查是否有尖状峰值
> - 过度使用终结器（Finalizer），该对象没有立即被 GC
> - 内存泄漏（Memory Leak），大量对象引用没有释放，JVM 无法对其自动回收，常见于使用了 File 等资源没有回收
>
> **解决方案**
>
> 针对大部分情况，通常只需通过 `-Xmx` 参数调高 JVM 堆内存空间即可。如果仍然没有解决，可参考以下情况做进一步处理：
>
> - 如果是超大对象，可以检查其合理性，比如是否一次性查询了数据库全部结果，而没有做结果数限制
> - 如果是业务峰值压力，可以考虑添加机器资源，或者做限流降级
> - 如果是内存泄漏，需要找到持有的对象，修改代码设计，比如关闭没有释放的连接

**场景二：GC overhead limit exceeded**

> 当 Java 进程花费 98% 以上的时间执行 GC，但只恢复了不到 2% 的内存，且该动作连续重复了 5 次，就会抛出 `java.lang.OutOfMemoryError:GC overhead limit exceeded` 错误。简单地说，就是应用程序已经基本耗尽了所有可用内存， GC 也无法回收。
>
> 此类问题的原因与解决方案跟 `Javaheap space` 非常类似，可以参考上文。

**场景三：Permgen space**

> 该错误表示永久代（Permanent Generation）已用满，通常是因为加载的 class 数目太多或体积太大。
>
> **原因分析**
>
> 永久代存储对象主要包括以下几类：
>
> - 加载/缓存到内存中的 class 定义，包括类的名称，字段，方法和字节码
> - 常量池
> - 对象数组/类型数组所关联的 class
> - JIT 编译器优化后的 class 信息
>
> PermGen 的使用量与加载到内存的 class 的数量/大小正相关。
>
> **解决方案**
>
> 根据 Permgen space 报错的时机，可以采用不同的解决方案，如下所示：
>
> - 程序启动报错，修改 `-XX:MaxPermSize` 启动参数，调大永久代空间
> - 应用重新部署时报错，很可能是没有应用没有重启，导致加载了多份 class 信息，只需重启 JVM 即可解决
> - 运行时报错，应用程序可能会动态创建大量 class，而这些 class 的生命周期很短暂，但是 JVM 默认不会卸载 class，可以设置 `-XX:+CMSClassUnloadingEnabled` 和 `-XX:+UseConcMarkSweepGC` 这两个参数允许 JVM 卸载 class。
>
> 如果上述方法无法解决，可以通过 jmap 命令 dump 内存对象 `jmap-dump:format=b,file=dump.hprof<process-id>` ，然后利用 Eclipse MAT https://www.eclipse.org/mat 功能逐一分析开销最大的 classloader 和重复 class。

**场景四：Metaspace**

> JDK 1.8 使用 Metaspace 替换了永久代（Permanent Generation），该错误表示 Metaspace 已被用满，通常是因为加载的 class 数目太多或体积太大。
>
> 此类问题的原因与解决方法跟 `Permgenspace` 非常类似，可以参考上文。需要特别注意的是调整 Metaspace 空间大小的启动参数为 `-XX:MaxMetaspaceSize`。

**场景五：Unable to create new native thread**

> 每个 Java 线程都需要占用一定的内存空间，当 JVM 向底层操作系统请求创建一个新的 native 线程时，如果没有足够的资源分配就会报此类错误。
>
> **原因分析**
>
> JVM 向 OS 请求创建 native 线程失败，就会抛出 `Unableto createnewnativethread`，常见的原因包括以下几类：
>
> - 线程数超过操作系统最大线程数 ulimit 限制
> - 线程数超过 kernel.pid_max（只能重启）
> - native 内存不足
>
> 该问题发生的常见过程主要包括以下几步：
>
> - JVM 内部的应用程序请求创建一个新的 Java 线程
> - JVM native 方法代理了该次请求，并向操作系统请求创建一个 native 线程
> - 操作系统尝试创建一个新的 native 线程，并为其分配内存
> - 如果操作系统的虚拟内存已耗尽，或是受到 32 位进程的地址空间限制，操作系统就会拒绝本次 native 内存分配
> - JVM 将抛出 `java.lang.OutOfMemoryError:Unableto createnewnativethread`错误
>
> **解决方案**
>
> - 升级配置，为机器提供更多的内存
> - 降低 Java Heap Space 大小
> - 修复应用程序的线程泄漏问题
> - 限制线程池大小
> - 使用 -Xss 参数减少线程栈的大小
> - 调高 OS 层面的线程最大数：执行 `ulimia-a` 查看最大线程数限制，使用 `ulimit-u xxx` 调整最大线程数限制

**场景六：Out of swap space？**

> 该错误表示所有可用的虚拟内存已被耗尽。虚拟内存（Virtual Memory）由物理内存（Physical Memory）和交换空间（Swap Space）两部分组成。当运行时程序请求的虚拟内存溢出时就会报 `Outof swap space?` 错误。
>
> **原因分析**
>
> 该错误出现的常见原因包括以下几类：
>
> - 地址空间不足
> - 物理内存已耗光
> - 应用程序的本地内存泄漏（native leak），例如不断申请本地内存，却不释放
> - 执行 `jmap-histo:live<pid>` 命令，强制执行 Full GC；如果几次执行后内存明显下降，则基本确认为 Direct ByteBuffer 问题
>
> **解决方案**
>
> 根据错误原因可以采取如下解决方案：
>
> - 升级地址空间为 64 bit
> - 使用 Arthas 检查是否为 Inflater/Deflater 解压缩问题，如果是，则显式调用 end 方法
> - Direct ByteBuffer 问题可以通过启动参数 `-XX:MaxDirectMemorySize` 调低阈值
> - 升级服务器配置/隔离部署，避免争用

**场景七：Kill process or sacrifice child**

> 有一种内核作业（Kernel Job）名为 Out of Memory Killer，它会在可用内存极低的情况下“杀死”（kill）某些进程。OOM Killer 会对所有进程进行打分，然后将评分较低的进程“杀死”，具体的评分规则可以参考 Surviving the Linux OOM Killer。不同于其它OOM错误， `Killprocessorsacrifice child` 错误不是由 JVM 层面触发的，而是由操作系统层面触发的。
>
> **原因分析**
>
> 默认情况下，Linux 内核允许进程申请的内存总量大于系统可用内存，通过这种“错峰复用”的方式可以更有效的利用系统资源。然而，这种方式也会无可避免地带来一定的“超卖”风险。例如某些进程持续占用系统内存，然后导致其他进程没有可用内存。此时，系统将自动激活 OOM Killer，寻找评分低的进程，并将其“杀死”，释放内存资源。
>
> **解决方案**
>
> - 升级服务器配置/隔离部署，避免争用
> - OOM Killer 调优

**场景八：Requested array size exceeds VM limit**

> JVM 限制了数组的最大长度，该错误表示程序请求创建的数组超过最大长度限制。JVM 在为数组分配内存前，会检查要分配的数据结构在系统中是否可寻址，通常为 `Integer.MAX_VALUE-2`。
>
> 此类问题比较罕见，通常需要检查代码，确认业务是否需要创建如此大的数组，是否可以拆分为多个块，分批执行。

**场景九：Direct buffer memory**

> Java 允许应用程序通过 Direct ByteBuffer 直接访问堆外内存，许多高性能程序通过 Direct ByteBuffer 结合内存映射文件（Memory Mapped File）实现高速 IO。
>
> **原因分析**
>
> Direct ByteBuffer 的默认大小为 64 MB，一旦使用超出限制，就会抛出 `Directbuffer memory` 错误。
>
> **解决方案**
>
> - Java 只能通过 ByteBuffer.allocateDirect 方法使用 Direct ByteBuffer，因此，可以通过 Arthas 等在线诊断工具拦截该方法进行排查
> - 检查是否直接或间接使用了 NIO，如 netty，jetty 等
> - 通过启动参数 `-XX:MaxDirectMemorySize` 调整 Direct ByteBuffer 的上限值
> - 检查 JVM 参数是否有 `-XX:+DisableExplicitGC` 选项，如果有就去掉，因为该参数会使 `System.gc()` 失效
> - 检查堆外内存使用代码，确认是否存在内存泄漏；或者通过反射调用 `sun.misc.Cleaner` 的 `clean()` 方法来主动释放被 Direct ByteBuffer 持有的内存空间
> - 内存容量确实不足，升级配置

**最佳实践**

> ① OOM发生时输出堆dump：
>
> `-XX:+HeapDumpOnOutOfMemoryError` `-XX:HeapDumpPath=$CATALINA_HOME/logs`
>
> ② OOM发生后的执行动作：
>
> `-XX:OnOutOfMemoryError=$CATALINA_HOME/bin/stop.sh` 
>
> `-XX:OnOutOfMemoryError=$CATALINA_HOME/bin/restart.sh`
>
> OOM之后除了保留堆dump外，根据管理策略选择合适的运行脚本。

## 本章小结

通过本章的学习，我们明白了虚拟机中的内存是如何划分的，哪部分区域、什么样的代码和操作可能导致内存溢出异常。虽然Java有垃圾收集机制，但内存溢出异常离我们仍然并不遥远，本章只是讲解了各个区域出现内存溢出异常的原因，第3章将详细讲解Java垃圾收集机制为了避免内存溢出异常的出现都做了哪些努力。

# 垃圾收集

## 概述

说起垃圾收集（Garbage Collection,GC），大部分人都把这项技术当做Java语言的伴生产物。事实上，GC的历史比Java久远，1960年诞生于MIT的Lisp是第一门真正使用内存动态分配和垃圾收集技术的语言。当Lisp还在胚胎时期时，人们就在思考GC需要完成的3件事情：

- 哪些内存需要回收？
- 什么时候回收？
- 如何回收？

经过半个多世纪的发展，目前内存的动态分配与内存回收技术已经相当成熟，一切看起来都进入了“自动化”时代，那为什么我们还要去了解GC和内存分配呢？答案很简单：当需要排查各种内存溢出、内存泄漏问题时，当垃圾收集成为系统达到更高并发量的瓶颈时，我们就需要对这些“自动化”的技术实施必要的监控和调节。

把时间从半个多世纪以前拨回到现在，回到我们熟悉的Java语言。Java内存运行时区域的各个部分，其中程序计数器、虚拟机栈、本地方法栈3个区域随线程而生，随线程而灭；栈中的栈帧随着方法的进入和退出而有条不紊地执行着出栈和入栈操作。每一个栈帧中分配多少内存基本上是在类结构确定下来时就已知的（尽管在运行期会由JIT编译器进行一些优化，但在本章基于概念模型的讨论中，大体上可以认为是编译期可知的），因此这几个区域的内存分配和回收都具备确定性，在这几个区域内就不需要过多考虑回收的问题，因为方法结束或者线程结束时，内存自然就跟随着回收了。而Java堆和方法区则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在程序处于运行期间时才能知道会创建哪些对象，这部分内存的分配和回收都是动态的，垃圾收集器所关注的是这部分内存，本章后续讨论中的“内存”分配与回收也仅指这一部分内存。

## 对象已死吗

在堆里面存放着Java世界中几乎所有的对象实例，垃圾收集器在对堆进行回收前，第一件事情就是要确定这些对象之中哪些还“存活”着，哪些已经“死去”（即不可能再被任何途径使用的对象）。

### 引用计数算法

很多教科书判断对象是否存活的算法是这样的：给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。作者面试过很多的应届生和一些有多年工作经验的开发人员，他们对于这个问题给予的都是这个答案。

![引用计数法](png\引用计数法.png)

客观地说，引用计数算法（Reference Counting）的实现简单，判定效率也很高，在大部分情况下它都是一个不错的算法，也有一些比较著名的应用案例，例如微软公司的COM（Component Object Model）技术、使用ActionScript 3的FlashPlayer、Python语言和在游戏脚本领域被广泛应用的Squirrel中都使用了引用计数算法进行内存管理。但是，至少主流的Java虚拟机里面没有选用引用计数算法来管理内存，其中最主要的原因是它很难解决对象之间相互循环引用的问题。

当图中的数值变成0时，这个时候使用引用计数算法就可以判定它是垃圾了，但是引用计数法不能解决一个问题，就是当对象是循环引用的时候，计数器值都不为0，这个时候引用计数器无法通知GC收集器来回收他们，如下图所示：

![引用计数法-问题](F:\work\openGuide\JVM\png\引用计数法-问题.png)

举个简单的例子，请看代码清单3-1中的testGC（）方法：对象objA和objB都有字段instance，赋值令objA.instance=objB及objB.instance=objA，除此之外，这两个对象再无任何引用，实际上这两个对象已经不可能再被访问，但是它们因为互相引用着对方，导致它们的引用计数都不为0，于是引用计数算法无法通知GC收集器回收它们。

引用计数算法的缺陷

```
/**
 * testGC()方法执行后，objA和objB会不会被GC呢？ 
 */
public class ReferenceCountingGC {

    public Object instance = null;

    private static final int _1MB = 1024 * 1024;

    /**
     * 这个成员属性的唯一意义就是占点内存，以便在能在GC日志中看清楚是否有回收过
     */
    private byte[] bigSize = new byte[2 * _1MB];

    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;

        objA = null;
        objB = null;

        // 假设在这行发生GC，objA和objB是否能被回收？
        System.gc();
    }
}
```

运行结果：

```
[F u l l G C（S y s t e m）[T e n u r e d：0 K-＞2 1 0 K（1 0 2 4 0 K），0.0 1 4 9 1 4 2 s e c s]4603K-＞210K（19456K），[Perm：2999K-＞2999K（21248K）]，0.0150007 secs][Times：user=0.01 sys=0.00，real=0.02 secs]
Heap
def new generation total 9216K,used 82K[0x00000000055e0000，0x0000000005fe0000，0x0000000005fe0000）
Eden space 8192K，1%used[0x00000000055e00000x00000000055f4850，0x0000000005de0000）
from space 1024K，0%used[0x0000000005de0000，0x0000000005de0000，0x0000000005ee0000）
to space 1024K，0%used[0x0000000005ee0000，0x0000000005ee0000，0x0000000005fe0000）
tenured generation total 10240K,used 210K[0x0000000005fe0000，0x00000000069e0000，0x00000000069e0000）
the space 10240K，2%used[0x0000000005fe0000，0x0000000006014a18，0x0000000006014c00，0x00000000069e0000）
compacting perm gen total 21248K,used 3016K[0x00000000069e0000，0x0000000007ea0000，0x000000000bde0000）
the space 21248K，14%used[0x00000000069e0000，0x0000000006cd2398，0x0000000006cd2400，0x0000000007ea0000）
No shared spaces configured.
```

从运行结果中可以清楚看到，GC日志中包含“4603K-＞210K”，意味着虚拟机并没有因为这两个对象互相引用就不回收它们，这也从侧面说明虚拟机并不是通过引用计数算法来判断对象是否存活的。

### 可达性分析算法

在主流的商用程序语言（Java、C#，甚至包括前面提到的古老的Lisp）的主流实现中，都是称通过可达性分析（Reachability Analysis）来判定对象是否存活的。这个算法的基本思路就是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。如图3-1所示，对象object 5、object 6、object 7虽然互相有关联，但是它们到GC Roots是不可达的，所以它们将会被判定为是可回收的对象。

![这里写图片描述](https://img-blog.csdn.net/20161221131715693)

在Java语言中，可作为GC Roots的对象包括下面几种：

- **虚拟机栈（栈帧中的本地变量表）中引用的对象。**
- **方法区中类静态属性引用的对象。**
- **方法区中常量引用的对象。**
- 本地方法栈中JNI（即一般说的Native方法）引用的对象。

根可达算法（Root Searching）的意思是说从根上开始搜索，当一个程序启动后，马上需要的那些个对象就叫做根对象，所谓的根可达算法就是首先找到根对象，然后跟着这根线一直往外找到那些有用的。常见的GC roots如下：

- **线程栈变量：** 线程里面会有线程栈和main栈帧，从这个main() 里面开始的这些对象都是我们的根对象

- **静态变量：** 一个class 它有一个静态的变量，load到内存之后马上就得对静态变量进行初始化，所以静态变量到的对象这个叫做根对象

- **常量池：** 如果你这个class会用到其他的class的那些个类的对象，这些就是根对象

- **JNI：** 如果我们调用了 C和C++ 写的那些本地方法所用到的那些个类或者对象

![根可达算法](png/根可达算法.png)

图中的 object5 和object6 虽然他们之间互相引用了，但是从根找不到它，所以就是垃圾，而object8没有任何引用自然而然也是垃圾，其他的Object对象都有可以从根找到的，所以是有用的，不会被垃圾回收掉。

**GC Root**

GC Roots 是一组必须活跃的引用。用通俗的话来说，就是程序接下来通过直接引用或者间接引用，能够访问到的潜在被使用的对象。GC Roots 包括：

- **Java 线程中，当前所有正在被调用的方法的引用类型参数、局部变量、临时值等。也就是与我们栈帧相关的各种引用**
- **所有当前被加载的 Java 类**
- **Java 类的引用类型静态变量**
- **运行时常量池里的引用类型常量（String 或 Class 类型）**
- **JVM 内部数据结构的一些引用，比如 sun.jvm.hotspot.memory.Universe 类**
- **用于同步的监控对象，比如调用了对象的 wait() 方法**
- **JNI handles，包括 global handles 和 local handles**



GC Roots 大体可以分为三大类：

- **活动线程相关的各种引用**
- **类的静态变量的引用**
- **JNI 引用**

### 再谈引用

无论是通过引用计数算法判断对象的引用数量，还是通过可达性分析算法判断对象的引用链是否可达，判定对象是否存活都与“引用”有关。在JDK 1.2以前，Java中的引用的定义很传统：**如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用**。这种定义很纯粹，但是太过狭隘，一个对象在这种定义下只有被引用或者没有被引用两种状态，对于如何描述一些“食之无味，弃之可惜”的对象就显得无能为力。我们希望能描述这样一类对象：当内存空间还足够时，则能保留在内存之中；如果内存空间在进行垃圾收集后还是非常紧张，则可以抛弃这些对象。很多系统的缓存功能都符合这样的应用场景。

在JDK 1.2之后，Java对引用的概念进行了扩充，将引用分为强引用（Strong 
Reference）、软引用（Soft Reference）、弱引用（Weak Reference）、虚引用（PhantomReference）4种，这4种引用强度依次逐渐减弱。

强引用就是指在程序代码之中普遍存在的，类似“Object obj=new Object（）”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象。

软引用是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK 1.2之后，提供了SoftReference类来实现软引用。

弱引用也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK 1.2之后，提供了WeakReference类来实现弱引用。

虚引用也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。在JDK 1.2之后，提供了PhantomReference类来实现虚引用。

## 引用级别

Java中4种引用的级别和强度由高到低依次为：**强引用→软引用→弱引用→虚引用**

当**垃圾回收器**回收时，某些对象会被回收，某些不会被回收。垃圾回收器会从**根对象**`Object`来**标记**存活的对象，然后将某些不可达的对象和一些引用的对象进行回收。如下所示：

| 引用类型 | 被垃圾回收时间 | 用途               | 生存时间          |
| -------- | -------------- | ------------------ | ----------------- |
| 强引用   | 从来不会       | 对象的一般状态     | JVM停止运行时终止 |
| 软引用   | 当内存不足时   | 对象缓存           | 内存不足时终止    |
| 弱引用   | 正常垃圾回收时 | 对象缓存           | 垃圾回收后终止    |
| 虚引用   | 正常垃圾回收时 | 跟踪对象的垃圾回收 | 垃圾回收后终止    |

![引用级别](png/引用级别.png)



### 强引用（StrongReference）

强引用是我们最常见的对象，它属于不可回收资源，垃圾回收器（后面简称GC）绝对不会回收它，即使是内存不足，JVM宁愿抛出 OutOfMemoryError 异常，使程序终止，也不会来回收强引用对象。如果一个对象具有强引用，那**垃圾回收器**绝不会回收它。如下：

```java
Object strongReference = new Object();
```

当**内存空间不足**时，`Java`虚拟机宁愿抛出`OutOfMemoryError`错误，使程序**异常终止**，也不会靠随意**回收**具有**强引用**的**对象**来解决内存不足的问题。 如果强引用对象**不使用时**，需要弱化从而使`GC`能够回收，如下：

```java
strongReference = null;
```

显式地设置`strongReference`对象为`null`，或让其**超出**对象的**生命周期**范围，则`gc`认为该对象**不存在引用**，这时就可以回收这个对象。具体什么时候收集这要取决于`GC`算法。

```java
public void test() {
	Object strongReference = new Object();
	// 省略其他操作
}
```

在一个**方法的内部**有一个**强引用**，这个引用保存在`Java`**栈**中，而真正的引用内容(`Object`)保存在`Java`**堆**中。 当这个**方法运行完成**后，就会退出**方法栈**，则引用对象的**引用数**为`0`，这个对象会被回收。但是如果这个`strongReference`是**全局变量**时，就需要在不用这个对象时赋值为`null`，因为**强引用**不会被垃圾回收。



### 软引用（SoftReference）

如果一个对象只具有**软引用**，则**内存空间充足**时，**垃圾回收器**就**不会**回收它；如果**内存空间不足**了，就会**回收**这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。

```java
// 强引用
String strongReference = new String("abc");
// 软引用
String str = new String("abc");
SoftReference<String> softReference = new SoftReference<String>(str);
```

**软引用**可以和一个**引用队列**(`ReferenceQueue`)联合使用。如果**软引用**所引用对象被**垃圾回收**，`JAVA`虚拟机就会把这个**软引用**加入到与之关联的**引用队列**中。

```java
ReferenceQueue<String> referenceQueue = new ReferenceQueue<>();
String str = new String("abc");
SoftReference<String> softReference = new SoftReference<>(str, referenceQueue);
```

**注意**：软引用对象是在jvm内存不够时才会被回收，我们调用System.gc()方法只是起通知作用，JVM什么时候扫描回收对象是JVM自己的状态决定的。就算扫描到软引用对象也不一定会回收它，只有内存不够的时候才会回收。

**垃圾收集线程**会在虚拟机抛出`OutOfMemoryError`之前回**收软引用对象**，而**虚拟机**会尽可能优先回收**长时间闲置不用**的**软引用对象**。对那些**刚构建**的或刚使用过的**"较新的"**软对象会被虚拟机尽可能**保留**，这就是引入**引用队列**`ReferenceQueue`的原因。



### 弱引用（WeakReference）

弱引用对象相对软引用对象具有更短暂的生命周期，只要 GC 发现它仅有弱引用，不管内存空间是否充足，都会回收它，不过 GC 是一个优先级很低的线程，因此不一定会很快发现那些仅有弱引用的对象。

**弱引用**与**软引用**的区别在于：只具有**弱引用**的对象拥有**更短暂**的**生命周期**。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有**弱引用**的对象，不管当前**内存空间足够与否**，都会**回收**它的内存。不过，由于垃圾回收器是一个**优先级很低的线程**，因此**不一定**会**很快**发现那些只具有**弱引用**的对象。

```java
String str = new String("abc");
WeakReference<String> weakReference = new WeakReference<>(str);
str = null;
```

`JVM`首先将**软引用**中的**对象**引用置为`null`，然后通知**垃圾回收器**进行回收：

```java
str = null;
System.gc();
```

**注意**：如果一个对象是偶尔(很少)的使用，并且希望在使用时随时就能获取到，但又不想影响此对象的垃圾收集，那么你应该用Weak Reference来记住此对象。

下面的代码会让一个**弱引用**再次变为一个**强引用**：

```java
String str = new String("abc");
WeakReference<String> weakReference = new WeakReference<>(str);
// 弱引用转强引用
String strongReference = weakReference.get();
```

同样，**弱引用**可以和一个**引用队列**(`ReferenceQueue`)联合使用，如果**弱引用**所引用的**对象**被**垃圾回收**，`Java`虚拟机就会把这个**弱引用**加入到与之关联的**引用队列**中。



### 虚引用（PhantomReference）

**虚引用**顾名思义，就是**形同虚设**。与其他几种引用都不同，**虚引用**并**不会**决定对象的**生命周期**。如果一个对象**仅持有虚引用**，那么它就和**没有任何引用**一样，在任何时候都可能被垃圾回收器回收。

**应用场景：**

**虚引用**主要用来**跟踪对象**被垃圾回收器**回收**的活动。 **虚引用**与**软引用**和**弱引用**的一个区别在于：

> 虚引用必须和引用队列(ReferenceQueue)联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

```java
String str = new String("abc");
ReferenceQueue queue = new ReferenceQueue();
// 创建虚引用，要求必须与一个引用队列关联
PhantomReference pr = new PhantomReference(str, queue);
```

程序可以通过判断引用**队列**中是否已经加入了**虚引用**，来了解被引用的对象是否将要进行**垃圾回收**。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的**内存被回收之前**采取必要的行动。

### 生存还是死亡

即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为有必要执行finalize()方法，那么这个对象将会放置在一个叫做F-Queue的队列之中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，这样做的原因是，如果一个对象在finalize()方法中执行缓慢，或者发生了死循环（更极端的情况），将很可能会导致F-Queue队列中其他对象永久处于等待，甚至导致整个内存回收系统崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移除出“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的被回收了。从代码清单3-2中我们可以看到一个对象的finalize()被 
执行，但是它仍然可以存活。

代码清单3-2　一次对象自我拯救的演示

```
/**
 * 此代码演示了两点： 
 * 1.对象可以在被GC时自我拯救。 
 * 2.这种自救的机会只有一次，因为一个对象的finalize()方法最多只会被系统自动调用一次
 * @author zzm
 */
public class FinalizeEscapeGC {

    public static FinalizeEscapeGC SAVE_HOOK = null;

    public void isAlive() {
        System.out.println("yes, i am still alive :)");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize mehtod executed!");
        FinalizeEscapeGC.SAVE_HOOK = this;
    }

    public static void main(String[] args) throws Throwable {
        SAVE_HOOK = new FinalizeEscapeGC();

        //对象第一次成功拯救自己
        SAVE_HOOK = null;
        System.gc();
        // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }

        // 下面这段代码与上面的完全相同，但是这次自救却失败了
        SAVE_HOOK = null;
        System.gc();
        // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }
    }
}
```

运行结果：

```
finalize mehtod executed ! 
yes,i am still alive : ) 
no,i am dead : (
```

从代码清单3-2的运行结果可以看出，SAVE_HOOK对象的finalize()方法确实被GC收集器触发过，并且在被收集前成功逃脱了。

另外一个值得注意的地方是，代码中有两段完全一样的代码片段，执行结果却是一次逃脱成功，一次失败，这是因为任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，它的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了。

需要特别说明的是，上面关于对象死亡时finalize()方法的描述可能带有悲情的艺术色彩，笔者并不鼓励大家使用这种方法来拯救对象。相反，笔者建议大家尽量避免使用它，因为它不是C/C++中的析构函数，而是Java刚诞生时为了使C/C++程序员更容易接受它所做出的一个妥协。它的运行代价高昂，不确定性大，无法保证各个对象的调用顺序。有些教材中描述它适合做“关闭外部资源”之类的工作，这完全是对这个方法用途的一种自我安慰。finalize()能做的所有工作，使用try-finally或者其他方式都可以做得更好、更及时，所以笔者建议大家完全可以忘掉Java语言中有这个方法的存在。



### 回收方法区

很多人认为方法区（或者HotSpot虚拟机中的永久代）是没有垃圾收集的，Java虚拟机规范中确实说过可以不要求虚拟机在方法区实现垃圾收集，而且在方法区中进行垃圾收集的“性价比”一般比较低：在堆中，尤其是在新生代中，常规应用进行一次垃圾收集一般可以回收70%～95%的空间，而永久代的垃圾收集效率远低于此。

永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。回收废弃常量与回收Java堆中的对象非常类似。以常量池中字面量的回收为例，假如一个字符串“abc”已经进入了常量池中，但是当前系统没有任何一个String对象是叫做“abc”的，换句话说，就是没有任何String对象引用常量池中的“abc”常量，也没有其他地方引用了这个字面量，如果这时发生内存回收，而且必要的话，这个“abc”常量就会被系统清理出常量池。常量池中的其他类（接口）、方法、字段的符号引用也与此类似。

判定一个常量是否是“废弃常量”比较简单，而要判定一个类是否是“无用的类”的条件则相对苛刻许多。类需要同时满足下面3个条件才能算是“无用的类”：

- 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。
- 加载该类的ClassLoader已经被回收。
- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

虚拟机可以对满足上述3个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样，不使用了就必然会回收。是否对类进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制，还可以使用-verbose：class以及-XX：+TraceClassLoading、-XX：+TraceClassUnLoading查看类加载和卸载信息，其中-verbose：class和-XX：+TraceClassLoading可以在Product版的虚拟机中使用，-XX：+TraceClassUnLoading参数需要FastDebug版的虚拟机支持。

### JDK7

+ 回收的主要内容
  - 废弃常量
  - 无用类
+ 如何判定类是无用的
  - 该类的所有实例都被回收
  - 加载该类的ClassLoader已经被回收
  - 该类对应的java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法
+ `类并不与对象一样，不使用了就必然会回收。`是否对类进行回收，HotSpot提供了一系列的参数：
  - -Xnoclassgc  关闭虚拟机对class的垃圾回收功能。
  - -verbose:class 及 -XX:+TraceClassLoading 、 -XX:+TraceClassUnLoading  查看类加载卸载信息 

### JDK8

Jdk8中使用元空间替代了永久代，故类的信息回收方式取决于元空间的回收方式.即：

   + `类及相关的元数据的生命周期与类加载器的一致`

## 垃圾收集算法(GarbageCollectionAlgorithm)

清理垃圾算法又叫内存回收算法。垃圾收集算法的思想及其发展过程如下：

### 标记（Mark）

垃圾回收的第一步，就是找出活跃的对象。根据 GC Roots 遍历所有的可达对象，这个过程，就叫作标记。

![标记（Mark）](png/JVM/GC算法-标记（Mark）.png)

如图所示，圆圈代表的是对象。绿色的代表 GC Roots，红色的代表可以追溯到的对象。可以看到标记之后，仍然有多个灰色的圆圈，它们都是被回收的对象。

### 清除（Sweep）

清除阶段就是把未被标记的对象回收掉。

![清除（Sweep）](png/JVM/GC算法-清除（Sweep）.png)

但是这种简单的清除方式，有一个明显的弊端，那就是碎片问题。标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。在很久之前使用 Windows 系统时，有一个非常有用的功能，就是内存整理和磁盘整理，运行之后有可能会显著提高系统性能。这个出发点是一样的。

### 复制（Copying）

![GC算法-复制(Copying)算法](png\JVM\GC算法-复制(Copying)算法.png)

为了解决效率问题，一种称为“复制”（Copying）的收集算法出现了，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。只是这种算法的代价是将内存缩小为了原来的一半，未免太高了一点。

### 整理（Compact）

其实，不用分配一个对等的额外空间，也是可以完成内存的整理工作。可以把内存想象成一个非常大的数组，根据随机的 index 删除了一些数据。那么对整个数组的清理，其实是不需要另外一个数组来进行支持的，使用程序就可以实现。它的主要思路，就是移动所有存活的对象，且按照内存地址顺序依次排列，然后将末端内存地址以后的内存全部回收。

![整理（Compact）](F:/lemon-guide-main/images/JVM/整理（Compact）.png)

但是需要注意，这只是一个理想状态。对象的引用关系一般都是非常复杂的，我们这里不对具体的算法进行描述。你只需要了解，从效率上来说，一般整理算法是要低于复制算法的。



### 扩展回收算法

https://juejin.cn/post/6896035896916148237

目前JVM的垃圾回收器都是对几种朴素算法的发扬光大（没有最优的算法，只有最合适的算法）：

- **复制算法（Copying）**：复制算法是所有算法里面效率最高的，缺点是会造成一定的空间浪费
- **标记-清除（Mark-Sweep）**：效率一般，缺点是会造成内存碎片问题
- **标记-整理（Mark-Compact）**：效率比前两者要差，但没有空间浪费，也消除了内存碎片问题

![收集算法](png/JVM/GC算法-收集算法.png)

#### 标记清除（Mark-Sweep）

![GC算法-标记清除(Mark-Sweep)算法](png\JVM\GC算法-标记清除(Mark-Sweep)算法.png)

最基础的收集算法是“标记-清除”（Mark-Sweep）算法，算法分为“标记”和“清除”两个阶段：首先从 GC Root 开始遍历对象图，并标记（Mark）所遇到的每个对象，标识出所有要回收的对象。然后回收器检查堆中每一个对象，并将所有未被标记的对象进行回收。

**缺点**

- 标记、清除的效率都不高
- 清除后产生大量的内存碎片，空间碎片太多会导致在分配大对象时无法找到足够大的连续内存，从而不得不触发另一次垃圾回收动作

#### 标记整理（Mark-Compact）

![标记整理算法](png/JVM/GC算法-标记整理(Mark-Compact)算法.png)

“标记-整理”（Mark-Compact）算法与标记清除算法类似，但不是在标记完成后对可回收对象进行清理，而是将所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

**优点**

- 消除了标记清除导致的内存分散问题，也消除了复制算法中内存减半的高额代价

**不足之处**

- 效率低下，需要标记所有存活对象，还要标记所有存活对象的引用地址。效率上低于复制算法

#### 标记复制（Mark-Coping）

标记-复制算法将内存分为两块相同大小的区域（比如新生代的Survivor区），每次在其中一块区域分配元素，当这块区域内存占满时，就会将存活下来的元素复制到另一块内存区域并清空当前内存区域。

- 缺点：浪费一半的内存空间
- 优点：简单高效

**JVM在Eden区保存新对象，在GC时，将Eden和Survivor中存活对象复制到Survivor的另一个分区。这是JVM对复制算法的一个优化。只浪费了1/10的内存空间【JVM的Eden区和Survivor区的比例为 8:2】**

 ![标记复制](png/JVM/GC算法-标记复制.jpg)



#### 分代收集（Generational Collection）

分代收集就是根据对象的存活周期将内存分为新生代和老年代。

- **新生代**对象“朝生夕死”，每次收集都有大量对象（99%）死去，所以可以选择**标记-复制算法**，只需要付出少量对象的复制成本就可以完成每次垃圾收集
- **老年代**对象生存几率比较高，存活对象比较多，如果选择复制算法需要付出较高的IO成本，而且没用额外的空间可以用于复制，此时选择**标记-清除**或者**标记-整理**就比较合理



研究表明大部分对象可以分为两类：

- 大部分对象的生命周期都很短
- 其他对象则很可能会存活很长时间

根据对象存活周期的不同将内存划分为几块。对不同周期的对象采取不同的收集算法：

- **新生代**：每次垃圾收集会有大批对象回收，所以采取复制算法
- **老年代**：对象存活率高，采取标记清理或者标记整理算法



**① 年轻代（Young Generation）**

年轻代使用的垃圾回收算法是复制算法。因为年轻代发生 GC 后，只会有非常少的对象存活，复制这部分对象是非常高效的。但复制算法会造成一定的空间浪费，所以年轻代中间也会分很多区域。

![年轻代](png/JVM/GC算法-年轻代.jpg)

如图所示，年轻代分为：**1个伊甸园空间（Eden ）**，**2个幸存者空间（Survivor ）**。当年轻代中的 Eden 区分配满的时候，就会触发年轻代的 GC（Minor GC）。具体过程如下：

- 在 Eden 区执行了第一次 GC 之后，存活的对象会被移动到其中一个 Survivor 分区（以下简称from）
- Eden 区再次 GC，这时会采用复制算法，将 Eden 和 from 区一起清理。存活的对象会被复制到 to 区，然后只需要清空 from 区就可以了

在这个过程中，总会有1个 Survivor 分区是空置的。Eden、from、to 的默认比例是 8:1:1，所以只会造成 10% 的空间浪费。这个比例，是由参数 **-XX:SurvivorRatio** 进行配置的（默认为 8）。



**② 老年代（Old/Tenured Generation）**

老年代一般使用“**标记-清除**”、“**标记-整理**”算法，因为老年代的对象存活率一般是比较高的，空间又比较大，拷贝起来并不划算，还不如采取就地收集的方式。对象进入老年代的途径如下：

- **提升（Promotion）**

  如果对象够老，会通过“提升”进入老年代

- **分配担保**

  年轻代回收后存活的对象大于10%时，因Survivor空间不够存储，对象就会直接在老年代上分配

- **大对象直接在老年代分配**

  超出某个大小的对象将直接在老年代分配

- **动态对象年龄判定**

  有的垃圾回收算法，并不要求 age 必须达到 15 才能晋升到老年代，它会使用一些动态的计算方法。

  比如，如果幸存区中相同年龄对象大小的和，大于幸存区的一半，大于或等于 age 的对象将会直接进入老年代。

#### 分区收集

## HotSpot的算法实现

### 枚举根节点

从可达性分析中从GC Roots节点找引用链这个操作为例，可作为GC Roots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中，现在很多应用仅仅方法区就有数百兆，如果要逐个检查这里面的引用，那么必然会消耗很多时间。

另外，可达性分析对执行时间的敏感还体现在GC停顿上，因为这项分析工作必须在一个能确保一致性的快照中进行——这里“一致性”的意思是指在整个分析期间整个执行系统看起来就像被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不断变化的情况，该点不满足的话分析结果准确性就无法得到保证。这点是导致GC进行时必须停顿所有Java执行线程（Sun将这件事情称为“Stop The World”）的其中一个重要原因，即使是在号称（几乎）不会发生停顿的CMS收集器中，枚举根节点时也是必须要停顿的。

由于目前的主流Java虚拟机使用的都是准确式GC（这个概念在第1章介绍Exact VM对Classic VM的改进时讲过），所以当执行系统停顿下来后，并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机应当是有办法直接得知哪些地方存放着对象引用。在HotSpot的实现中，是使用一组称为OopMap的数据结构来达到这个目的的，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT编译过程中，也会在特定的位置记录下栈和寄存器中哪些位置是引用。这样，GC在扫描时就可以直接得知这些信息了。下面的代码清单3-3是HotSpot Client VM生成的一段String.hashCode（）方法的本地代码，可以看到在0x026eb7a9处的call指令有OopMap记录，它指明了EBX寄存器和栈中偏移量为16的内存区域中各有一个普通对象指针（Ordinary Object Pointer）的引用，有效范围为从call指令开始直到0x026eb730（指令流的起始位置）+142（OopMap记录的偏移量）=0x026eb7be，即hlt指令为止。

代码清单3-3　String.hashCode（）方法编译后的本地代码

```
[Verified Entry Point]
0x026eb730：mov%eax，-0x8000（%esp）
…… ；ImplicitNullCheckStub slow case
0x026eb7a9：call 0x026e83e0 ；OopMap{ebx=Oop[16]=Oop off=142} ；*caload ；-java.lang.String：hashCode@48（line 1489）
；{runtime_call}
0x026eb7ae：push$0x83c5c18 ；{external_word}
0x026eb7b3：call 0x026eb7b8
0x026eb7b8：pusha
0x026eb7b9：call 0x0822bec0；{runtime_call}
0x026eb7be：hlt
```

### 安全点

在OopMap的协助下，HotSpot可以快速且准确地完成GC Roots枚举，但一个很现实的问题随之而来：可能导致引用关系变化，或者说OopMap内容变化的指令非常多，如果为每一条指令都生成对应的OopMap，那将会需要大量的额外空间，这样GC的空间成本将会变得很高。

实际上，HotSpot也的确没有为每条指令都生成OopMap，前面已经提到，只是在“特定的位置”记录了这些信息，这些位置称为安全点（Safepoint），即程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。Safepoint的选定既不能太少以致于让GC等待时间太长，也不能过于频繁以致于过分增大运行时的负荷。所以，安全点的选定基 
本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生Safepoint。

对于Sefepoint，另一个需要考虑的问题是如何在GC发生时让所有线程（这里不包括执行JNI调用的线程）都“跑”到最近的安全点上再停顿下来。这里有两种方案可供选择：抢先式中断（Preemptive Suspension）和主动式中断（Voluntary Suspension），其中抢先式中断不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点上，就恢复线程，让它“跑”到安全点上。现在几乎没有虚拟机实现采用抢先式中断来暂停线程从而响应GC事件。

而主动式中断的思想是当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。下面代码清单3-4中的test指令是HotSpot生成的轮询指令，当需要暂停线程时，虚拟机把0x160100的内存页设置为不可读，线程执行到test指令时就会产生一个自陷异常信号，在预先注册的异常处理器中暂停线程实现等待，这样一条汇编指令便完成安全点轮询和触发线程中断。

代码清单3-4　轮询指令

```
0x01b6d627：call 0x01b2b210；OopMap{[60]=Oop off=460} ；*invokeinterface size ；-Client1：main@113（line 23）
；{virtual_call}
0x01b6d62c：nop ；OopMap{[60]=Oop off=461} ；*if_icmplt ；-Client1：main@118（line 23）
0x01b6d62d：test%eax，0x160100；{poll}
0x01b6d633：mov 0x50（%esp），%esi
0x01b6d637：cmp%eax，%esi
```

### 安全区域

使用Safepoint似乎已经完美地解决了如何进入GC的问题，但实际情况却并不一定。Safepoint机制保证了程序执行时，在不太长的时间内就会遇到可进入GC的Safepoint。但是，程序“不执行”的时候呢？所谓的程序不执行就是没有分配CPU时间，典型的例子就是线程处于Sleep状态或者Blocked状态，这时候线程无法响应JVM的中断请求，“走”到安全的地方去中断挂起，JVM也显然不太可能等待线程重新被分配CPU时间。对于这种情况，就需要安全区域（Safe Region）来解决。

安全区域是指在一段代码片段之中，引用关系不会发生变化。在这个区域中的任意地方开始GC都是安全的。我们也可以把Safe Region看做是被扩展了的Safepoint。

在线程执行到Safe Region中的代码时，首先标识自己已经进入了Safe Region，那样，当在这段时间里JVM要发起GC时，就不用管标识自己为Safe Region状态的线程了。在线程要离开Safe Region时，它要检查系统是否已经完成了根节点枚举（或者是整个GC过程），如果完成了，那线程就继续执行，否则它就必须等待直到收到可以安全离开Safe Region的信号为止。

到此，笔者简要地介绍了HotSpot虚拟机如何去发起内存回收的问题，但是虚拟机如何具体地进行内存回收动作仍然未涉及，因为内存回收如何进行是由虚拟机所采用的GC收集器决定的，而通常虚拟机中往往不止有一种GC收集器。下面继续来看HotSpot中有哪些GC收集器。

## 垃圾收集器(GarbageCollector)

如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。虚拟机的收集器如下：

![GC_收集器](png\JVM\GC_收集器.png) 
GC垃圾收集器的JVM配置参数：

- **-XX:+UseSerialGC**：年轻代和老年代都用串行收集器
- **-XX:+UseParNewGC**：年轻代使用 ParNew，老年代使用 Serial Old
- **-XX:+UseParallelGC**：年轻代使用 ParallerGC，老年代使用 Serial Old
- **-XX:+UseParallelOldGC**：新生代和老年代都使用并行收集器
- **-XX:+UseConcMarkSweepGC**：表示年轻代使用 ParNew，老年代的用 CMS
- **-XX:+UseG1GC**：使用 G1垃圾回收器
- **-XX:+UseZGC**：使用 ZGC 垃圾回收器

### Serial收集器

![GC_SerialOld收集器](png\JVM\GC_SerialOld收集器.jpg)

Serial收集器是最基本、发展历史最悠久的收集器，曾经（在JDK 1.3.1之前）是虚拟机新生代收集的唯一选择。处理GC的只有一条线程，并且在垃圾回收的过程中暂停一切用户线程（Stop The World），直到它收集结束。最简单的垃圾回收器，因为简单，所以高效，它通常用在客户端应用上。限定单个CPU时，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程收集效率。因为客户端应用不会频繁创建很多对象，用户也不会感觉出明显的卡顿。相反，它使用的资源更少，也更轻量级。

### ParNew收集器

![GC_ParNew收集器](png\JVM\GC_ParNew收集器.jpg)

ParNew是Serial的多线程版本。由多条GC线程并行地进行垃圾清理。清理过程依然要停止用户线程。ParNew 追求“低停顿时间”，与 Serial 唯一区别就是使用了多线程进行垃圾收集，在多 CPU 环境下性能比 Serial 会有一定程度的提升；但线程切换需要额外的开销，因此在单 CPU 环境中表现不如 Serial。是许多运行在Server模式下的虚拟机中首选的新生代收集器，其中有一个与性能无关但很重要的原因是，除了Serial收集器外，目前只有它能与CMS收集器配合工作。

**并发和并行**在垃圾收集器中可以解释如下。

- 并行（Parallel）：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- 并发（Concurrent）：指用户线程与垃圾收集线程同时执行（但不一定是并行的，可能会交替执行），用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。

### Parallel Scavenge收集器

另一个多线程版本的垃圾回收器。Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量（Throughput）。所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间），虚拟机总共运行了100分钟，其中垃圾收集花掉1分钟，那吞吐量就是99%。Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的-XX：MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX：GCTimeRatio参数。Parallel Scavenge收集器也经常称为“吞吐量优先”收集器。

它与ParNew的主要区别是：

- **Parallel Scavenge**：追求CPU吞吐量，能够在较短时间完成任务，适合没有交互的后台计算。弱交互强计算
- **ParNew**：追求降低用户停顿时间，适合交互式应用。强交互弱计算

MaxGCPauseMillis参数允许的值是一个大于0的毫秒数，收集器将尽可能地保证内存回收花费的时间不超过设定值。不过大家不要认为如果把这个参数的值设置得稍小一点就能使得系统的垃圾收集速度变得更快，GC停顿时间缩短是以牺牲吞吐量和新生代空间来换取的：系统把新生代调小一些，收集300MB新生代肯定比收集500MB快吧，这也直接导致垃圾收集发生得更频繁一些，原来10秒收集一次、每次停顿100毫秒，现在变成5秒收集一次、每次停顿70毫秒。停顿时间的确在下降，但吞吐量也降下来了。

GCTimeRatio参数的值应当是一个大于0且小于100的整数，也就是垃圾收集时间占总时间的比率，相当于是吞吐量的倒数。如果把此参数设置为19，那允许的最大GC时间就占总时间的5%（即1/（1+19）），默认值为99，就是允许最大1%（即1/（1+99））的垃圾收集时间。

Parallel Scavenge收集器还有一个参数-XX：+UseAdaptiveSizePolicy值得关注。这是一个开关参数，当这个参数打开之后，就不需要手工指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX：SurvivorRatio）、晋升老年代对象年龄（-XX：PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种调节方式称为GC自适应的调节策略（GC Ergonomics）。如果读者对于收集器运作原来不太了解，手工优化存在困难的时候，使用Parallel Scavenge收集器配合自适应调节策略，把内存管理的调优任务交给虚拟机去完成将是一个不错的选择。只需要把基本的内存数据设置好（如-Xmx设置最大堆），然后使用MaxGCPauseMillis参数（更关注最大停顿时间）或GCTimeRatio（更关注吞吐量）参数给虚拟机设立一个优化目标，那具体细节参数的调节工作就由虚拟机完成了。自适应调节策略也是Parallel Scavenge收集器与ParNew收集器的一个重要区别。

### Serial Old收集器

![GC_SerialOld收集器](png\JVM\GC_SerialOld收集器.jpg)

与年轻代的 Serial 垃圾收集器对应，都是单线程版本，同样适合客户端使用。年轻代的 Serial，使用复制算法。老年代的 Old Serial，使用标记-整理算法。这个收集器的主要意义也是在于给Client模式下的虚拟机使用。

### Parallel Old收集器

![GC_ParallelOld收集器](png\JVM\GC_ParallelOld收集器.jpg)

Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。追求 CPU 吞吐量。在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器。

### CMS收集器

![GC_CMS收集器](png\JVM\GC_CMS收集器.png)

**CMS（Concurrent Mark Sweep）收集器(并发标记清除收集器)**是一款致力于获取最短停顿时间的收集器，使用多个线程来扫描堆内存并标记可被清除的对象，然后清除标记的对象。在下面两种情形下会暂停工作线程：

- 在老年代中标记引用对象的时候

- 在做垃圾回收的过程中堆内存中有变化发生

**主要流程如下**：

- **初始标记(CMS initial mark)**：仅标记出GC Roots能直接关联到的对象。需要Stop-the-world
- **并发标记(CMS concurrenr mark)**：进行GC Roots遍历的过程，寻找出所有可达对象
- **重新标记(CMS remark)**：修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录。需要Stop-the-world
- **并发清除(CMS concurrent sweep)**：清出垃圾

其中，初始标记、重新标记这两个步骤仍然需要“Stop The World”。从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。主要优点在名字上已经体现出来了：并发收集、低停顿，Sun公司的一些官方文档中也称之为并发低停顿收集器（Concurrent Low Pause Collector）。

**CMS触发机制**：当老年代的使用率达到80%时，就会触发一次CMS GC。

- `-XX:CMSInitiatingOccupancyFraction=80`
- `-XX:+UseCMSInitiatingOccupancyOnly`

对比与并行垃圾回收器，CMS回收器使用更多的CPU来保证更高的吞吐量。如果我们可以有更多的CPU用来提升性能，那么CMS垃圾回收器是比并行回收器更好的选择。使用 `-XX:+UseParNewGCJVM` 参数来开启使用CMS垃圾回收器。

**缺点**

- **并发回收导致CPU资源紧张**

  在并发阶段，它虽然不会导致用户线程停顿，但却会因为占用了一部分线程而导致应用程序变慢，降低程序总吞吐量。CMS默认启动的回收线程数是：（CPU核数 + 3）/ 4，当CPU核数不足四个时，CMS对用户程序的影响就可能变得很大。

- **无法清理浮动垃圾**

  在CMS的并发标记和并发清理阶段，用户线程还在继续运行，就还会伴随有新的垃圾对象不断产生，但这一部分垃圾对象是出现在标记过程结束以后，CMS无法在当次收集中处理掉它们，只好留到下一次垃圾收集时再清理掉。这一部分垃圾称为“浮动垃圾”。

- **并发失败（Concurrent Mode Failure）**

  由于在垃圾回收阶段用户线程还在并发运行，那就还需要预留足够的内存空间提供给用户线程使用，因此CMS不能像其他回收器那样等到老年代几乎完全被填满了再进行回收，必须预留一部分空间供并发回收时的程序运行使用。默认情况下，当老年代使用了 80% 的空间后就会触发 CMS 垃圾回收，这个值可以通过 -XX**:** CMSInitiatingOccupancyFraction 参数来设置。

  这里会有一个风险：要是CMS运行期间预留的内存无法满足程序分配新对象的需要，就会出现一次“并发失败”（Concurrent Mode Failure），这时候虚拟机将不得不启动后备预案：Stop The World，临时启用 Serial Old 来重新进行老年代的垃圾回收，这样一来停顿时间就很长了。

- **内存碎片问题**

  CMS是一款基于“标记-清除”算法实现的回收器，这意味着回收结束时会有内存碎片产生。内存碎片过多时，将会给大对象分配带来麻烦，往往会出现老年代还有很多剩余空间，但就是无法找到足够大的连续空间来分配当前对象，而不得不提前触发一次 Full GC 的情况。

  为了解决这个问题，CMS收集器提供了一个 -XX:+UseCMSCompactAtFullCollection 开关参数（默认开启），用于在 Full GC 时开启内存碎片的合并整理过程，由于这个内存整理必须移动存活对象，是无法并发的，这样停顿时间就会变长。还有另外一个参数 -XX:CMSFullGCsBeforeCompaction，这个参数的作用是要求CMS在执行过若干次不整理空间的 Full GC 之后，下一次进入 Full GC 前会先进行碎片整理（默认值为0，表示每次进入 Full GC 时都进行碎片整理）。

**作用内存区域**：老年代

**适用场景**：对停顿时间敏感的场合

**算法类型**：标记-清除

### G1收集器

![GC_G1收集器](png\JVM\GC_G1收集器.jpg)

G1（Garbage First）回收器采用面向局部收集的设计思路和基于Region的内存布局形式，是一款主要面向服务端应用的垃圾回收器。G1设计初衷就是替换 CMS，成为一种全功能收集器。G1 在JDK9 之后成为服务端模式下的默认垃圾回收器，取代了 Parallel Scavenge 加 Parallel Old 的默认组合，而 CMS 被声明为不推荐使用的垃圾回收器。G1从整体来看是基于 标记-整理 算法实现的回收器，但从局部（两个Region之间）上看又是基于标记-复制 算法实现的。**G1 回收过程**，G1 回收器的运作过程大致可分为四个步骤：

- **初始标记（Initial Marking）**：（会STW）仅仅只是标记一下 GC Roots 能直接关联到的对象，并且修改TAMS（Next Top at Mark Start）指针的值，让下一阶段用户线程并发运行时，能正确地在可用的Region中分配新对象。这个阶段需要停顿线程，但耗时很短，而且是借用进行Minor GC的时候同步完成的，所以G1收集器在这个阶段实际并没有额外的停顿。
- **并发标记（Concurrent Marking）**：从 GC Roots 开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行。当对象图扫描完成以后，还要重新处理在并发时有引用变动的对象。
- **最终标记（Final Marking）**：（会STW）对用户线程做短暂的暂停，处理并发阶段结束后仍有引用变动的对象。
- **清理阶段（Live Data Counting and Evacuation）**：（会STW）更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集，然后把决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间。这里的操作涉及存活对象的移动，必须暂停用户线程，由多条回收器线程并行完成的。

G1收集器中的堆内存被划分为多个大小相等的内存块（Region），每个Region是逻辑连续的一段内存，结构如下：

![G1-Region](png/JVM/GC_G1-Region.png)

每个Region被标记了E、S、O和H，说明每个Region在运行时都充当了一种角色，其中H是以往算法中没有的，它代表Humongous（巨大的），这表示这些Region存储的是巨型对象（humongous object，H-obj），当新建对象大小超过Region大小一半时，直接在新的一个或多个连续Region中分配，并标记为H。

**Region**

堆内存中一个Region的大小可以通过 `-XX:G1HeapRegionSize`参数指定，大小区间只能是1M、2M、4M、8M、16M和32M，总之是2的幂次方。如果G1HeapRegionSize为默认值，则在堆初始化时计算Region的实际大小，默认把堆内存按照2048份均分，最后得到一个合理的大小。

**GC模式**

- **young gc**

  发生在年轻代的GC算法，一般对象（除了巨型对象）都是在eden region中分配内存，当所有eden region被耗尽无法申请内存时，就会触发一次young gc，这种触发机制和之前的young gc差不多，执行完一次young gc，活跃对象会被拷贝到survivor region或者晋升到old region中，空闲的region会被放入空闲列表中，等待下次被使用。

  - `-XX:MaxGCPauseMillis`：设置G1收集过程目标时间，默认值`200ms`
  - `-XX:G1NewSizePercent`：新生代最小值，默认值`5%`
  - `-XX:G1MaxNewSizePercent`：新生代最大值，默认值`60%`

- **mixed gc**

  当越来越多的对象晋升到老年代old region时，为了避免堆内存被耗尽，虚拟机会触发一个混合的垃圾收集器，即mixed gc，该算法并不是一个old gc，除了回收整个young region，还会回收一部分的old region，这里需要注意：**是一部分老年代，而不是全部老年代**，可以选择哪些old region进行收集，从而可以对垃圾回收的耗时时间进行控制

- **full gc**

  - 如果对象内存分配速度过快，mixed gc来不及回收，导致老年代被填满，就会触发一次full gc，G1的full gc算法就是单线程执行的serial old gc，会导致异常长时间的暂停时间，需要进行不断的调优，尽可能的避免full gc

**G1垃圾回收器** 应用于大的堆内存空间。它将堆内存空间划分为不同的区域，对各个区域并行地做回收工作。G1在回收内存空间后还立即对空闲空间做整合工作以减少碎片。CMS却是在全部停止(stop the world,STW)时执行内存整合工作。对于不同的区域G1根据垃圾的数量决定优先级。使用 `-XX:UseG1GCJVM` 参数来开启使用G1垃圾回收器。`-XX:InitiatingHeapOccupancyPercent`：当老年代大小占整个堆大小百分比达到该阈值时，会触发一次mixed gc。

**优点**：

- 并行与并发，充分发挥多核优势
- 分代收集，所以不需要与其它收集器配合即可工作
- 空间整合，整体来看基于”标记-整理算法“，局部采用”复制算法“都不会产生内存碎片
- 可以指定GC最大停顿时长

**缺点**：

- 需要记忆集来记录新生代和老年代之间的引用关系
- 需要占用大量的内存，可能达到整个堆内存容量的20%甚至更多

**作用内存区域**：跨代

**适用场景**：作为关注停顿时间的场景的收集器备选方案

**算法类型**：整体来看基于”标记-整理算法“，局部采用"复制算法"

### ZGC收集器

![ZGC收集器](png/JVM/GC_ZGC收集器.jpg)

Z Garbage Collector，简称 ZGC，是 JDK 11 中新加入的尚在实验阶段的低延迟垃圾收集器。它和 Shenandoah 同属于超低延迟的垃圾收集器，但在吞吐量上比 Shenandoah 有更优秀的表现，甚至超过了 G1，接近了“吞吐量优先”的 Parallel 收集器组合，可以说近乎实现了“鱼与熊掌兼得”。

与CMS中的ParNew和G1类似，ZGC也采用标记-复制算法，不过ZGC对该算法做了重大改进：ZGC在标记、转移和重定位阶段几乎都是并发的，这是ZGC实现停顿时间小于10ms目标的最关键原因。

ZGC只有三个STW阶段：**初始标记**，**再标记**，**初始转移**。其中，初始标记和初始转移分别都只需要扫描所有GC Roots，其处理时间和GC Roots的数量成正比，一般情况耗时非常短；再标记阶段STW时间很短，最多1ms，超过1ms则再次进入并发标记阶段。即，ZGC几乎所有暂停都只依赖于GC Roots集合大小，停顿时间不会随着堆的大小或者活跃对象的大小而增加。与ZGC对比，G1的转移阶段完全STW的，且停顿时间随存活对象的大小增加而增加。

**ZGC 的内存布局**

与 Shenandoah 和 G1 一样，ZGC 也采用基于 Region 的堆内存布局，但与它们不同的是， ZGC 的 Region 具有动态性，也就是可以动态创建和销毁，容量大小也是动态的，有大、中、小三类容量:

![ZGC内存布局](png/JVM/GC_ZGC内存布局.jpg)

- 小型 Region (Small Region)：容量固定为 2MB，用于放置小于 256KB 的小对象
- 中型 Region (M edium Region)：容量固定为 32MB，用于放置大于等于 256KB 但小于 4MB 的对象
- 大型 Region (Large Region)：容量不固定，可以动态变化，但必须为 2MB 的整数倍，用于放置 4MB 或以上的大对象。每个大型 Region 中只会存放一个大对象，这也预示着虽然名字叫作“大型 Region”，但它的实际容量完全有可能小于中型 Region，最小容量可低至 4MB

在 JDK 11 及以上版本，可以通过以下参数开启 ZGC：`-XX:+UnlockExperimentalVMOptions -XX:+UseZGC` 。

### Shenandoah收集器

Shenandoah 与 G1 有很多相似之处，比如都是基于 Region 的内存布局，都有用于存放大对象的 Humongous Region，默认回收策略也是优先处理回收价值最大的 Region。不过也有三个重大的区别：

- Shenandoah支持并发的整理算法，G1整理阶段虽是多线程并行，但无法与用户程序并发执行
- 默认不使用分代收集理论
- 使用连接矩阵 (Connection Matrix)记录跨Region的引用关系，替换掉了G1中的记忆级(Remembered Set)，内存和计算成本更低

Shenandoah 收集器的工作原理相比 G1 要复杂不少，其运行流程示意图如下：

![Shenandoah收集器运行流程](png/JVM/GC_Shenandoah收集器运行流程.jpg)

可见Shenandoah的并发程度明显比G1更高，只需要在初始标记、最终标记、初始引用更新和最终引用更新这几个阶段进行短暂的“Stop The World”，其他阶段皆可与用户程序并发执行，其中最重要的并发标记、并发回收和并发引用更新详情如下：

- **并发标记( Concurrent Marking)**
- **并发回收( Concurrent Evacuation)**
- **并发引用更新( Concurrent Update Reference)**

Shenandoah 的高并发度让它实现了超低的停顿时间，但是更高的复杂度也伴随着更高的系统开销，这在一定程度上会影响吞吐量，下图是 Shenandoah 与之前各种收集器在停顿时间维度和系统开销维度上的对比：

![收集器停顿时间和系统开销对比](png/JVM/GC_收集器停顿时间和系统开销对比.png)

OracleJDK 并不支持 Shenandoah，如果你用的是 OpenJDK 12 或某些支持 Shenandoah 移植版的 JDK 的话，可以通过以下参数开启 Shenandoah：`-XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC` 。

## GC实战

### 理解GC日志

**ParallelGC YoungGC日志**

![GC_ParallelGCYoungGC日志](png\JVM\GC_ParallelGCYoungGC日志.jpg)

**ParallelGC FullGC日志**

![GC_ParallelGCFullGC日志](png\JVM\GC_ParallelGCFullGC日志.jpg)

每个收集器的日志格式都可以不一样。但虚拟机设计者为了方便用户阅读，将各个收集器的日志都维持一定的共性，例如以下两段典型的GC日志：

```
33.125：[GC[DefNew：3324K-＞152K（3712K），0.0025925 secs]3324K-＞152K（11904K），0.0031680 secs]
100.667：[FullGC[Tenured：0K-＞210K（10240K），0.0 149142secs]4603K-＞210K（19456K），[Perm：2999K-＞
2999K（21248K）]，0.0150007 secs][Times：user=0.01 sys=0.00，real=0.02 secs] 
```

最前面的数字“33.125：”和“100.667：”代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数。

GC日志开头的“[GC”和“[Full GC”说明了这次垃圾收集的停顿类型，而不是用来区分新生代GC还是老年代GC的。如果有“Full”，说明这次GC是发生了Stop-The-World的，例如下面这段新生代收集器ParNew的日志也会出现“[Full GC”（这一般是因为出现了分配担保失败之类的问题，所以才导致STW）。如果是调用System.gc（）方法所触发的收集，那么在这里将显示“[Full GC（System）”。

```
[Full GC 283.736：[ParNew：261599K-＞261599K（261952K），0.0000288 secs]
```

接下来的“[DefNew”、“[Tenured”、“[Perm”表示GC发生的区域，这里显示的区域名称与使用的GC收集器是密切相关的，例如上面样例所使用的Serial收集器中的新生代名为“Default New Generation”，所以显示的是“[DefNew”。如果是ParNew收集器，新生代名称就会变为“[ParNew”，意为“Parallel New Generation”。如果采用Parallel Scavenge收集器，那它配套的新生代称为“PSYoungGen”，老年代和永久代同理，名称也是由收集器决定的。

后面方括号内部的“3324K-＞152K（3712K）”含义是“GC前该内存区域已使用容量-＞GC后该内存区域已使用容量（该内存区域总容量）”。而在方括号之外的“3324K-＞152K（11904K）”表示“GC前Java堆已使用容量-＞GC后Java堆已使用容量（Java堆总容量）”。

再往后，“0.0025925 secs”表示该内存区域GC所占用的时间，单位是秒。有的收集器会给出更具体的时间数据，如“[Times：user=0.01 sys=0.00，real=0.02 secs]”，这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。

### 最佳实践

在不同的 JVM 的不垃圾回收器上，看参数默认是什么，不要轻信别人的建议，命令行示例如下：

```shell
java -XX:+PrintFlagsFinal -XX:+UseG1GC  2>&1 | grep UseAdaptiveSizePolicy
```

PrintCommandLineFlags：通过它，你能够查看当前所使用的垃圾回收器和一些默认的值。

```shell
# java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=127905216 -XX:MaxHeapSize=2046483456 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
openjdk version "1.8.0_41"
OpenJDK Runtime Environment (build 1.8.0_41-b04)
OpenJDK 64-Bit Server VM (build 25.40-b25, mixed mode)
```



G1垃圾收集器JVM参数最佳实践：

```shell
# 1.基本参数
-server                  # 服务器模式
-Xmx12g                  # 初始堆大小
-Xms12g                  # 最大堆大小
-Xss256k                 # 每个线程的栈内存大小
-XX:+UseG1GC             # 使用 G1 (Garbage First) 垃圾收集器   
-XX:MetaspaceSize=256m   # 元空间初始大小
-XX:MaxMetaspaceSize=1g  # 元空间最大大小
-XX:MaxGCPauseMillis=200 # 每次YGC / MixedGC 的最多停顿时间 (期望最长停顿时间)

# 2.必备参数
-XX:+PrintGCDetails            # 输出详细GC日志
-XX:+PrintGCDateStamps         # 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintTenuringDistribution # 打印对象分布：为了分析GC时的晋升情况和晋升导致的高暂停，看对象年龄分布日志
-XX:+PrintHeapAtGC                 # 在进行GC的前后打印出堆的信息
-XX:+PrintReferenceGC              # 打印Reference处理信息:强引用/弱引用/软引用/虚引用/finalize方法万一有问题
-XX:+PrintGCApplicationStoppedTime # 打印STW时间
-XX:+PrintGCApplicationConCurrentTime # 打印GC间隔的服务运行时长

# 3.日志分割参数
-XX:+UseGCLogFileRotation   # 开启日志文件分割
-XX:NumberOfGCLogFiles=14   # 最多分割几个文件，超过之后从头文件开始写
-XX:GCLogFileSize=32M       # 每个文件上限大小，超过就触发分割
-Xloggc:/path/to/gc-%t.log  # GC日志输出的文件路径,使用%t作为日志文件名,即gc-2021-03-29_20-41-47.log
```

CMS垃圾收集器JVM参数最佳实践：

```shell
# 1.基本参数
-server   # 服务器模式
-Xmx4g    # JVM最大允许分配的堆内存，按需分配
-Xms4g    # JVM初始分配的堆内存，一般和Xmx配置成一样以避免每次gc后JVM重新分配内存
-Xmn256m  # 年轻代内存大小，整个JVM内存=年轻代 + 年老代 + 持久代
-Xss512k  # 设置每个线程的堆栈大小
-XX:+DisableExplicitGC                # 忽略手动调用GC, System.gc()的调用就会变成一个空调用，完全不触发GC
-XX:+UseConcMarkSweepGC               # 使用 CMS 垃圾收集器
-XX:+CMSParallelRemarkEnabled         # 降低标记停顿
-XX:+UseCMSCompactAtFullCollection    # 在FULL GC的时候对年老代的压缩
-XX:+UseFastAccessorMethods           # 原始类型的快速优化
-XX:+UseCMSInitiatingOccupancyOnly    # 使用手动定义初始化定义开始CMS收集
-XX:LargePageSizeInBytes=128m         # 内存页的大小
-XX:CMSInitiatingOccupancyFraction=70 # 使用cms作为垃圾回收使用70％后开始CMS收集

# 2.必备参数
-XX:+PrintGCDetails                # 输出详细GC日志
-XX:+PrintGCDateStamps             # 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintTenuringDistribution     # 打印对象分布：为分析GC时的晋升情况和晋升导致的高暂停，看对象年龄分布
-XX:+PrintHeapAtGC                 # 在进行GC的前后打印出堆的信息
-XX:+PrintReferenceGC              # 打印Reference处理信息:强引用/弱引用/软引用/虚引用/finalize方法万一有问题
-XX:+PrintGCApplicationStoppedTime # 打印STW时间
-XX:+PrintGCApplicationConCurrentTime # 打印GC间隔的服务运行时长

# 3.日志分割参数
-XX:+UseGCLogFileRotation   # 开启日志文件分割
-XX:NumberOfGCLogFiles=14   # 最多分割几个文件，超过之后从头文件开始写
-XX:GCLogFileSize=32M       # 每个文件上限大小，超过就触发分割
-Xloggc:/path/to/gc-%t.log  # GC日志输出的文件路径,使用%t作为日志文件名,即gc-2021-03-29_20-41-47.log
```



**test、stage 环境jvm使用CMS 参数配置（jdk8）**

```shell
-server -Xms256M -Xmx256M -Xss512k -Xmn96M -XX:MetaspaceSize=128M -XX:MaxMetaspaceSize=128M -XX:InitialHeapSize=256M -XX:MaxHeapSize=256M  -XX:+PrintCommandLineFlags -XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=2 -XX:+CMSParallelInitialMarkEnabled -XX:+CMSParallelRemarkEnabled -XX:+UnlockDiagnosticVMOptions -XX:+ParallelRefProcEnabled -XX:+AlwaysPreTouch -XX:MaxTenuringThreshold=8  -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+PrintGC -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC  -XX:+PrintTenuringDistribution  -XX:SurvivorRatio=8 -Xloggc:../logs/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../dump
```

**online 环境jvm使用CMS参数配置（jdk8）**

```shell
-server -Xms4G -Xmx4G -Xss512k  -Xmn1536M -XX:MetaspaceSize=128M -XX:MaxMetaspaceSize=128M -XX:InitialHeapSize=4G -XX:MaxHeapSize=4G  -XX:+PrintCommandLineFlags -XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=2 -XX:+CMSParallelInitialMarkEnabled -XX:+CMSParallelRemarkEnabled -XX:+UnlockDiagnosticVMOptions -XX:+ParallelRefProcEnabled -XX:+AlwaysPreTouch -XX:MaxTenuringThreshold=10  -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+PrintGC -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC  -XX:+PrintTenuringDistribution  -XX:SurvivorRatio=8 -Xloggc:../logs/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../dump
```











JDK 1.7中的各种垃圾收集器到此已全部介绍完毕，在描述过程中提到了很多虚拟机非稳定的运行参数，在表3-2中整理了这些参数供读者实践时参考。 
![这里写图片描述](https://img-blog.csdn.net/20161223112439417?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](https://img-blog.csdn.net/20161223112514808?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 内存分配与回收策略

Java技术体系中所提倡的自动内存管理最终可以归结为自动化地解决了两个问题：给对象分配内存以及回收分配给对象的内存。关于回收内存这一点，我们已经使用了大量篇幅去介绍虚拟机中的垃圾收集器体系以及运作原理，现在我们再一起来探讨一下给对象分配内存的那点事儿。

对象的内存分配，往大方向讲，就是在堆上分配（但也可能经过JIT编译后被拆散为标量类型并间接地栈上分配），对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。少数情况下也可能会直接分配在老年代中，分配的规则并不是百分之百固定的，其细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数的设置。

接下来我们将会讲解几条最普遍的内存分配规则，并通过代码去验证这些规则。本节下面的代码在测试时使用Client模式虚拟机运行，没有手工指定收集器组合，换句话说，验证的是在使用Serial/Serial Old收集器下（ParNew/Serial Old收集器组合的规则也基本一致）的内存分配和回收的策略。读者不妨根据自己项目中使用的收集器写一些程序去验证一下使用其他几种收集器的内存分配策略。



### 对象优先在Eden分配

大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

虚拟机提供了-XX：+PrintGCDetails这个收集器日志参数，告诉虚拟机在发生垃圾收集行为时打印内存回收日志，并且在进程退出的时候输出当前的内存各区域分配情况。在实际应用中，内存回收日志一般是打印到文件后通过日志工具进行分析，不过本实验的日志并不多，直接阅读就能看得很清楚。

代码清单3-5的testAllocation（）方法中，尝试分配3个2MB大小和1个4MB大小的对象，在运行时通过-Xms20M、-Xmx20M、-Xmn10M这3个参数限制了Java堆大小为20MB，不可扩展，其中10MB分配给新生代，剩下的10MB分配给老年代。-XX：SurvivorRatio=8决定了新生代中Eden区与一个Survivor区的空间比例是8:1，从输出的结果也可以清晰地看到“eden space 8192K、from space 1024K、to space 1024K”的信息，新生代总可用空间为9216KB（Eden区+1个Survivor区的总容量）。

执行testAllocation（）中分配allocation4对象的语句时会发生一次Minor GC，这次GC的结果是新生代6651KB变为148KB，而总内存占用量则几乎没有减少（因为allocation1、allocation2、allocation3三个对象都是存活的，虚拟机几乎没有找到可回收的对象）。这次GC发生的原因是给allocation4分配内存的时候，发现Eden已经被占用了6MB，剩余空间已不足以分配allocation4所需的4MB内存，因此发生Minor GC。GC期间虚拟机又发现已有的3个2MB大小的对象全部无法放入Survivor空间（Survivor空间只有1MB大小），所以只好通过分配担保机制提前转移到老年代去。

这次GC结束后，4MB的allocation4对象顺利分配在Eden中，因此程序执行完的结果是Eden占用4MB（被allocation4占用），Survivor空闲，老年代被占用6MB（被allocation1、allocation2、allocation3占用）。通过GC日志可以证实这一点。

注意　作者多次提到的Minor GC和Full GC有什么不一样吗？

- 新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。
- 老年代GC（Major GC/Full GC）：指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC的策略选择过程）。Major 的速度一般会比Minor GC慢10倍以上。

代码清单3-5　新生代Minor GC

```
private static final int _1MB = 1024 * 1024;

/**
 * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
  */
public static void testAllocation() {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[2 * _1MB];
    allocation2 = new byte[2 * _1MB];
    allocation3 = new byte[2 * _1MB];
    allocation4 = new byte[4 * _1MB];  // 出现一次Minor GC
 }
```

 

运行结果：

```
[GC[DefMew:6651K->148K(9216K),0.0070106 secs]6651K->6292K(19456K), 0.0070426 secs] [Times :user=0.00 sys=0.00,real=0.00 secs]
Heap
def new generation total 9216K,used 4326K[0x029d0000 ,0x033d0000 ,0x033d0000 ) eden space 8192K ,5Uused[0x029d0000 ,0x02de4828 ,0x031d0000 )
from space 1024K ,14Sused[0x032d0000 ,0x032f5370 ,0x033d0000 )
to space 1024K ,0%used[0x03ldO000 ,0x031d0000 ,0x032d0000 )
tenured generation total 1024OK,used 6144K[0x033d0000 ,0x03dd0000 ,0x03dd0000 ) 
the space 1024OK,60lused[0x033d0000,0x039d0030,0x039d0200,0x03dd0000) 
compacting perm gen total 12288K,used 2114K[0x03dd0000 ,0x049d0000 ,0x07dd0000 ) 
the space 12288K ,17lused[0x03dd0000 ,0x03fe0998 ,0x03fe0a00 ,0x049d0000 )
Mo shared spaces configured.
```

 



### 大对象直接进入老年代

所谓的大对象是指，需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组（笔者列出的例子中的byte[]数组就是典型的大对象）。大对象对虚拟机的内存分配来说就是一个坏消息（替Java虚拟机抱怨一句，比遇到一个大对象更加坏的消息就是遇到一群“朝生夕灭”的“短命大对象”，写程序的时候应当避免），经常出现大对象容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来“安置”它们。

虚拟机提供了一个-XX：PretenureSizeThreshold参数，令大于这个设置值的对象直接在老年代分配。这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制（复习一下：新生代采用复制算法收集内存）。

执行代码清单3-6中的testPretenureSizeThreshold（）方法后，我们看到Eden空间几乎没有被使用，而老年代的10MB空间被使用了40%，也就是4MB的allocation对象直接就分配在老年代中，这是因为PretenureSizeThreshold被设置为3MB（就是3145728，这个参数不能像-Xmx之类的参数一样直接写3MB），因此超过3MB的对象都会直接在老年代进行分配。注意PretenureSizeThreshold参数只对Serial和ParNew两款收集器有效，Parallel Scavenge收集器不认识这个参数，Parallel Scavenge收集器一般并不需要设置。如果遇到必须使用此参数的场合，可以考虑ParNew加CMS的收集器组合。

代码清单3-6　大对象直接进入老年代

```
private static final int _1MB = 1024 * 1024;

/**
 * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
 * -XX:PretenureSizeThreshold=3145728
 */
public static void testPretenureSizeThreshold() {
    byte[] allocation;
    allocation = new byte[4 * _1MB];  //直接分配在老年代中
}
```

 

运行结果：

```
Heap
def new generation total 9216K,used 671K[0x029d0000，0x033d0000，0x033d0000）
eden space 8192K，8%used[0x029d0000，0x02a77e98，0x031d0000）
from space 1024K，0%used[0x031d0000，0x031d0000，0x032d0000）
to space 1024K，0%used[0x032d0000，0x032d0000，0x033d0000）
tenured generation total 10240K,used 4096K[0x033d0000，0x03dd0000，0x03dd0000）
the space 10240K，40%used[0x033d0000，0x037d0010，0x037d0200，0x03dd0000）
compacting perm gen total 12288K,used 2107K[0x03dd0000，0x049d0000，0x07dd0000）
the space 12288K，17%used[0x03dd0000，0x03fdefd0，0x03fdf000，0x049d0000）
No shared spaces configured.
```



 



### 长期存活的对象将进入老年代

既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到这点，虚拟机给每个对象定义了一个对象年龄（Age）计数器。如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。对象在Survivor区中每“熬过”一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁），就将会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数-XX：MaxTenuringThreshold设置。

读者可以试试分别以-XX：MaxTenuringThreshold=1和-XX：MaxTenuringThreshold=15两种设置来执行代码清单3-7中的testTenuringThreshold（）方法，此方法中的allocation1对象需要256KB内存，Survivor空间可以容纳。当MaxTenuringThreshold=1时，allocation1对象在第二次GC发生时进入老年代，新生代已使用的内存GC后非常干净地变成0KB。而MaxTenuringThreshold=15时，第二次GC发生后，allocation1对象则还留在新生代Survivor空间，这时新生代仍然有404KB被占用。

代码清单3-7　长期存活的对象进入老年代

```
private static final int _1MB = 1024 * 1024;

/**
 * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1
 * -XX:+PrintTenuringDistribution
 */
@SuppressWarnings("unused")
public static void testTenuringThreshold() {
    byte[] allocation1, allocation2, allocation3;
    allocation1 = new byte[_1MB / 4];  // 什么时候进入老年代决定于XX:MaxTenuringThreshold设置
    allocation2 = new byte[4 * _1MB];
    allocation3 = new byte[4 * _1MB];
    allocation3 = null;
    allocation3 = new byte[4 * _1MB];
}
```



 

以MaxTenuringThreshold=1参数来运行的结果：

```
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 1（max 1）
-age 1：414664 bytes，414664 total ：4859K-＞404K（9216K），0.0065012 secs]4859K-＞4500K（19456K），0.0065283 secs][Times：user=0.02 sys=0.00，real=0.02 secs]
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 1（max 1）
：4500K-＞0K（9216K），0.0009253 secs]8596K-＞4500K（19456K），0.0009458 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
Heap
def new generation total 9216K,used 4178K[0x029d0000，0x033d0000，0x033d0000）
eden space 8192K，51%used[0x029d0000，0x02de4828，0x031d0000）
from space 1024K，0%used[0x031d0000，0x031d0000，0x032d0000）
to space 1024K，0%used[0x032d0000，0x032d0000，0x033d0000）
tenured generation total 10240K,used 4500K[0x033d0000，0x03dd0000，0x03dd0000）
the space 10240K，43%used[0x033d0000，0x03835348，0x03835400，0x03dd0000）
compacting perm gen total 12288K,used 2114K[0x03dd0000，0x049d0000，0x07dd0000）
the space 12288K，17%used[0x03dd0000，0x03fe0998，0x03fe0a00，0x049d0000）
No shared spaces configured. 
```

以MaxTenuringThreshold=15参数来运行的结果：

```
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 15（max 15）
-age 1：414664 bytes，414664 total ：4859K-＞404K（9216K），0.0049637 secs]4859K-＞4500K（19456K），0.0049932 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 15（max 15）
-age 2：414520 bytes，414520 total ：4500K-＞404K（9216K），0.0008091 secs]8596K-＞4500K（19456K），0.0008305 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
Heap
def new generation total 9216K,used 4582K[0x029d0000，0x033d0000，0x033d0000）
eden space 8192K，51%used[0x029d0000，0x02de4828，0x031d0000）
from space 1024K，39%used[0x031d0000，0x03235338，0x032d0000）
to space 1024K，0%used[0x032d0000，0x032d0000，0x033d0000）
tenured generation total 10240K,used 4096K[0x033d0000，0x03dd0000，0x03dd0000）
the space 10240K，40%used[0x033d0000，0x037d0010，0x037d0200，0x03dd0000）
compacting perm gen total 12288K,used 2114K[0x03dd0000，0x049d0000，0x07dd0000）
the space 12288K，17%used[0x03dd0000，0x03fe0998，0x03fe0a00，0x049d0000）
No shared spaces configured.
```

### 动态对象年龄判定

为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

执行代码清单3-8中的testTenuringThreshold2（）方法，并设置-XX： 
MaxTenuringThreshold=15，会发现运行结果中Survivor的空间占用仍然为0%，而老年代比预期增加了6%，也就是说，allocation1、allocation2对象都直接进入了老年代，而没有等到15岁的临界年龄。因为这两个对象加起来已经到达了512KB，并且它们是同年的，满足同年对象达到Survivor空间的一半规则。我们只要注释掉其中一个对象new操作，就会发现另外一个就不会晋升到老年代中去了。

代码清单3-8　动态对象年龄判定

```
private static final int _1MB = 1024 * 1024;

/**
 * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15
 * -XX:+PrintTenuringDistribution
 */
@SuppressWarnings("unused")
public static void testTenuringThreshold2() {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[_1MB / 4];   // allocation1+allocation2大于survivo空间一半
    allocation2 = new byte[_1MB / 4];  
    allocation3 = new byte[4 * _1MB];
    allocation4 = new byte[4 * _1MB];
    allocation4 = null;
    allocation4 = new byte[4 * _1MB];
}
```

运行结果：

```
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 1（max 15）
-age 1：676824 bytes，676824 total ：5115K-＞660K（9216K），0.0050136 secs]5115K-＞4756K（19456K），0.0050443 secs][Times：user=0.00 sys=0.01，real=0.01 secs]
[GC[DefNew
Desired Survivor size 524288 bytes,new threshold 15（max 15）
：4756K-＞0K（9216K），0.0010571 secs]8852K-＞4756K（19456K），0.0011009 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
Heap
def new generation total 9216K,used 4178K[0x029d0000，0x033d0000，0x033d0000）
eden space 8192K，51%used[0x029d0000，0x02de4828，0x031d0000）
from space 1024K，0%used[0x031d0000，0x031d0000，0x032d0000）
to space 1024K，0%used[0x032d0000，0x032d0000，0x033d0000）
tenured generation total 10240K,used 4756K[0x033d0000，0x03dd0000，0x03dd0000）
the space 10240K，46%used[0x033d0000，0x038753e8，0x03875400，0x03dd0000）
compacting perm gen total 12288K,used 2114K[0x03dd0000，0x049d0000，0x07dd0000）
the space 12288K，17%used[0x03dd0000，0x03fe09a0，0x03fe0a00，0x049d0000）
No shared spaces configured.
```

### 空间分配担保

在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC。

下面解释一下“冒险”是冒了什么风险，前面提到过，新生代使用复制收集算法，但为了内存利用率，只使用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况（最极端的情况就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把Survivor无法容纳的对象直接进入老年代。与生活中的贷款担保类似，老年代要进行这样的担保，前提是老年代本身还有容纳这些对象的剩余空间，一共有多少对象会活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代对象容量的平均大小值作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间。

取平均值进行比较其实仍然是一种动态概率的手段，也就是说，如果某次Minor GC存活后的对象突增，远远高于平均值的话，依然会导致担保失败（Handle Promotion Failure）。如果出现了HandlePromotionFailure失败，那就只好在失败后重新发起一次Full GC。虽然担保失败时绕的圈子是最大的，但大部分情况下都还是会将HandlePromotionFailure开关打开，避免Full GC过于频繁，参见代码清单3-9，请读者在JDK 6 Update 24之前的版本中运行测试。

代码清单3-9　空间分配担保

```
private static final int _1MB = 1024 * 1024;

/**
 * VM参数：-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:-HandlePromotionFailure
 */
@SuppressWarnings("unused")
public static void testHandlePromotion() {
    byte[] allocation1, allocation2, allocation3, allocation4, allocation5, allocation6, allocation7;
    allocation1 = new byte[2 * _1MB];
    allocation2 = new byte[2 * _1MB];
    allocation3 = new byte[2 * _1MB];
    allocation1 = null;
    allocation4 = new byte[2 * _1MB];
    allocation5 = new byte[2 * _1MB];
    allocation6 = new byte[2 * _1MB];
    allocation4 = null;
    allocation5 = null;
    allocation6 = null;
    allocation7 = new byte[2 * _1MB];
}
```

以HandlePromotionFailure=false参数来运行的结果：

```
[GC[DefNew：6651K-＞148K（9216K），0.0078936 secs]6651K-＞4244K（19456K），0.0079192 secs][Times：user=0.00 sys=0.02，real=0.02 secs]
[G C[D e f N e w：6 3 7 8 K-＞6 3 7 8 K（9 2 1 6 K），0.0 0 0 0 2 0 6 s e c s][T e n u r e d：4096K-＞4244K（10240K），0.0042901 secs]10474K-＞
4244K（19456K），[Perm：2104K-＞2104K（12288K）]，0.0043613 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
```

以HandlePromotionFailure=true参数来运行的结果：

```
[GC[DefNew：6651K-＞148K（9216K），0.0054913 secs]6651K-＞4244K（19456K），0.0055327 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
[GC[DefNew：6378K-＞148K（9216K），0.0006584 secs]10474K-＞4244K（19456K），0.0006857 secs][Times：user=0.00 sys=0.00，real=0.00 secs]
```

在JDK 6 Update 24之后，这个测试结果会有差异，HandlePromotionFailure参数不会再影响到虚拟机的空间分配担保策略，观察OpenJDK中的源码变化（见代码清单3-10），虽然源码中还定义了HandlePromotionFailure参数，但是在代码中已经不会再使用它。JDK 6 Update 24之后的规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC，否则将进行Full GC。

代码清单3-10　HotSpot中空间分配检查的代码片段

```
bool TenuredGeneration：promotion_attempt_is_safe（size_tmax_promotion_in_bytes）const{
    //老年代最大可用的连续空间
    size_t available=max_contiguous_available（）；
    //每次晋升到老年代的平均大小
    size_t av_promo=（size_t）gc_stats（）-＞avg_promoted（）-＞padded_average（）；
    //老年代可用空间是否大于平均晋升大小，或者老年代可用空间是否大于当此GC时新生代所有对象容量
    bool res=（available＞=av_promo）||（available＞=
    max_promotion_in_bytes）；
    return res；
}
```

### Full GC场景

**场景一：System.gc()方法的调用**

此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定,但很多情况下它会触发 Full GC,从而增加Full GC的频率,也即增加了间歇性停顿的次数。强烈影响系建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过 `-XX:+ DisableExplicitGC` 来禁止RMI调用System.gc()。



**场景二：老年代代空间不足**

- 原因分析：新生代对象转入老年代、创建大对象或数组时，执行FullGC后仍空间不足
- 抛出错误：`Java.lang.OutOfMemoryError: Java heap space`
- 解决办法：
  - 尽量让对象在YoungGC时被回收
  - 让对象在新生代多存活一段时间
  - 不要创建过大的对象或数组



**场景三：永生区空间不足**

- 原因分析：JVM方法区因系统中要加载的类、反射的类和调用的方法较多而可能会被占满
- 抛出错误：`java.lang.OutOfMemoryError: PermGen space`
- 解决办法：
  - 增大老年代空间大小
  - 使用CMS GC



**场景四：CMS GC时出现promotion failed和concurrent mode failure**

- 原因分析：
  - `promotion failed`：是在进行Minor GC时，survivor space放不下、对象只能放入老年代，而此时老年代也放不下造成
   - `concurrent mode failure`：是在执行CMS GC的过程中同时有对象要放入老年代，而此时老年代空间不足造成的
- 抛出错误：GC日志中存在`promotion failed`和`concurrent mode`
- 解决办法：增大幸存区或老年代



**场景五：堆中分配很大的对象**

- 原因分析：创建大对象或长数据时，此对象直接进入老年代，而老年代虽有很大剩余空间，但没有足够的连续空间来存储
- 抛出错误：触发FullGC
- 解决办法：配置-XX:+UseCMSCompactAtFullCollection开关参数，用于享受用完FullGC后额外免费赠送的碎片整理过程，但同时停顿时间不得不变长。可以使用-XX:CMSFullGCsBeforeCompaction参数来指定执行多少次不压缩的FullGC后才执行一次压缩



### CMS GC场景

**场景一：动态扩容引起的空间震荡**

- **现象**

  服务**刚刚启动时 GC 次数较多**，最大空间剩余很多但是依然发生 GC，这种情况我们可以通过观察 GC 日志或者通过监控工具来观察堆的空间变化情况即可。GC Cause 一般为 Allocation Failure，且在 GC 日志中会观察到经历一次 GC ，堆内各个空间的大小会被调整，如下图所示：

  ![动态扩容引起的空间震荡](F:/lemon-guide-main/images/JVM/动态扩容引起的空间震荡.png)

- **原因**

  在 JVM 的参数中 `-Xms` 和 `-Xmx` 设置的不一致，在初始化时只会初始 `-Xms` 大小的空间存储信息，每当空间不够用时再向操作系统申请，这样的话必然要进行一次 GC。另外，如果空间剩余很多时也会进行缩容操作，JVM 通过 `-XX:MinHeapFreeRatio` 和 `-XX:MaxHeapFreeRatio` 来控制扩容和缩容的比例，调节这两个值也可以控制伸缩的时机。整个伸缩的模型理解可以看这个图，当 committed 的空间大小超过了低水位/高水位的大小，capacity 也会随之调整：

  ![JVM内存伸缩模型](F:/lemon-guide-main/images/JVM/JVM内存伸缩模型.png)

- **策略** 

  观察 CMS GC 触发时间点 Old/MetaSpace 区的 committed 占比是不是一个固定的值，或者像上文提到的观察总的内存使用率也可以。尽量 **将成对出现的空间大小配置参数设置成固定的** ，如 `-Xms` 和 `-Xmx`，`-XX:MaxNewSize` 和 `-XX:NewSize`，`-XX:MetaSpaceSize` 和 `-XX:MaxMetaSpaceSize` 等。



**场景二：显式GC的去与留**

- **现象**

  除了扩容缩容会触发 CMS GC 之外，还有 Old 区达到回收阈值、MetaSpace 空间不足、Young 区晋升失败、大对象担保失败等几种触发条件，如果这些情况都没有发生却触发了 GC ？这种情况有可能是代码中手动调用了 System.gc 方法，此时可以找到 GC 日志中的 GC Cause 确认下。

- **原因**

  **保留 System.gc**：CMS中使用 Foreground Collector 时将会带来非常长的 STW，在应用程序中 System.gc 被频繁调用，那就非常危险。增加 `-XX:+DisableExplicitGC` 参数则可以禁用。**去掉 System.gc**：禁用掉后会带来另一个内存泄漏的问题，为 DirectByteBuffer 分配空间过程中会显式调用 System.gc ，希望通过 Full GC 来强迫已经无用的 DirectByteBuffer 对象释放掉它们关联的 Native Memory，如Netty等。

- **策略**

  无论是保留还是去掉都会有一定的风险点，不过目前互联网中的 RPC 通信会大量使用 NIO，所以建议保留。此外 JVM 还提供了 `-XX:+ExplicitGCInvokesConcurrent` 和 `-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses` 参数来将 System.gc 的触发类型从 Foreground 改为 Background，同时 Background 也会做 Reference Processing，这样的话就能大幅降低了 STW 开销，同时也不会发生 NIO Direct Memory OOM。



**场景三：MetaSpace区OOM**

- **现象**

  JVM 在启动后或者某个时间点开始， **MetaSpace 的已使用大小在持续增长，同时每次 GC 也无法释放，调大 MetaSpace 空间也无法彻底解决** 。

- **原因**

  在讨论为什么会 OOM 之前，我们先来看一下这个区里面会存什么数据，Java 7 之前字符串常量池被放到了 Perm 区，所有被 intern 的 String 都会被存在这里，由于 String.intern 是不受控的，所以 `-XX:MaxPermSize` 的值也不太好设置，经常会出现 `java.lang.OutOfMemoryError: PermGen space` 异常，所以在 Java 7 之后常量池等字面量（Literal）、类静态变量（Class Static）、符号引用（Symbols Reference）等几项被移到 Heap 中。而 Java 8 之后 PermGen 也被移除，取而代之的是 MetaSpace。由场景一可知，为了避免弹性伸缩带来的额外 GC 消耗，我们会将 `-XX:MetaSpaceSize` 和 `-XX:MaxMetaSpaceSize` 两个值设置为固定的，但这样也会导致在空间不够的时候无法扩容，然后频繁地触发 GC，最终 OOM。所以关键原因是 **ClassLoader不停地在内存中load了新的Class ，一般这种问题都发生在动态类加载等情况上。**

- **策略**

  可以 dump 快照之后通过 JProfiler 或 MAT 观察 Classes 的 Histogram（直方图） 即可，或者直接通过命令即可定位， jcmd 打几次 Histogram 的图，看一下具体是哪个包下的 Class 增加较多就可以定位。如果无法从整体的角度定位，可以添加 `-XX:+TraceClassLoading` 和 `-XX:+TraceClassUnLoading` 参数观察详细的类加载和卸载信息。



**场景四：过早晋升**

- **现象**

  这种场景主要发生在分代的收集器上面，专业的术语称为“Premature Promotion”。90% 的对象朝生夕死，只有在 Young 区经历过几次 GC 的洗礼后才会晋升到 Old 区，每经历一次 GC 对象的 GC Age 就会增长 1，最大通过 `-XX:MaxTenuringThreshold` 来控制。过早晋升一般不会直接影响 GC，总会伴随着浮动垃圾、大对象担保失败等问题，但这些问题不是立刻发生的，我们可以观察以下几种现象来判断是否发生了过早晋升：

  - **分配速率接近于晋升速率** ，对象晋升年龄较小。

    GC 日志中出现“Desired survivor size 107347968 bytes, **new threshold 1(max 6)** ”等信息，说明此时经历过一次 GC 就会放到 Old 区。

  - **Full GC 比较频繁** ，且经历过一次 GC 之后 Old 区的 **变化比例非常大** 。

    如Old区触发回收阈值是80%，经历一次GC之后下降到了10%，这说明Old区70%的对象存活时间其实很短。

    ![FullGC变化比例大](F:/lemon-guide-main/images/JVM/FullGC变化比例大.png)

  过早晋升的危害：

  - Young GC 频繁，总的吞吐量下降
  - Full GC 频繁，可能会有较大停顿

- **原因**

  主要的原因有以下两点：

  - **Young/Eden 区过小**： 过小的直接后果就是 Eden 被装满的时间变短，本应该回收的对象参与了 GC 并晋升，Young GC 采用的是复制算法，由基础篇我们知道 copying 耗时远大于 mark，也就是 Young GC 耗时本质上就是 copy 的时间（CMS 扫描 Card Table 或 G1 扫描 Remember Set 出问题的情况另说），没来及回收的对象增大了回收的代价，所以 Young GC 时间增加，同时又无法快速释放空间，Young GC 次数也跟着增加
  - **分配速率过大**： 可以观察出问题前后 Mutator 的分配速率，如果有明显波动可以尝试观察网卡流量、存储类中间件慢查询日志等信息，看是否有大量数据被加载到内存中

- **策略**

  - 如果是 **Young/Eden 区过小** ，可以在总的 Heap 内存不变的情况下适当增大Young区。一般情况下Old的大小应当为活跃对象的2~3倍左右，考虑到浮动垃圾问题最好在3倍左右，剩下的都可以分给Young区

  - 过早晋升优化来看，原配置为Young 1.2G+Old 2.8G，通过观察CMS GC的情况找到存活对象大概为 300~400M，于是调整Old 1.5G左右，剩下2.5G分给Young 区。仅仅调了一个Young区大小参数（`-Xmn`），整个 JVM 一分钟Young GC从26次降低到了11次，单次时间也没有增加，总的GC时间从1100ms降低到了500ms，CMS GC次数也从40分钟左右一次降低到了7小时30分钟一次：

    ![过早晋升优化GC](F:/lemon-guide-main/images/JVM/过早晋升优化GC.png)

    ![过早晋升优化Oldgen](F:/lemon-guide-main/images/JVM/过早晋升优化Oldgen.png)

    如果是分配速率过大：

    - **偶发较大** ：通过内存分析工具找到问题代码，从业务逻辑上做一些优化
    - **一直较大** ：当前的 Collector 已经不满足 Mutator 的期望了，这种情况要么扩容 Mutator 的 VM，要么调整 GC 收集器类型或加大空间

- **小结**

  过早晋升问题一般不会特别明显，但日积月累之后可能会爆发一波收集器退化之类的问题，所以我们还是要提前避免掉的，可以看看自己系统里面是否有这些现象，如果比较匹配的话，可以尝试优化一下。一行代码优化的 ROI 还是很高的。如果在观察 Old 区前后比例变化的过程中，发现可以回收的比例非常小，如从 80% 只回收到了 60%，说明我们大部分对象都是存活的，Old 区的空间可以适当调大些。



**场景五：CMS Old GC频繁**

- **现象**

  Old 区频繁的做 CMS GC，但是每次耗时不是特别长，整体最大 STW 也在可接受范围内，但由于 GC 太频繁导致吞吐下降比较多。

- **原因**

  这种情况比较常见，基本都是一次 Young GC 完成后，负责处理 CMS GC 的一个后台线程 concurrentMarkSweepThread 会不断地轮询，使用 `shouldConcurrentCollect()` 方法做一次检测，判断是否达到了回收条件。如果达到条件，使用 `collect_in_background()` 启动一次 Background 模式 GC。轮询的判断是使用 `sleepBeforeNextCycle()` 方法，间隔周期为 `-XX:CMSWaitDuration` 决定，默认为 2s。

- **策略**

  处理这种常规内存泄漏问题基本是一个思路，主要步骤如下：

  ![CMSOldGC频繁](F:/lemon-guide-main/images/JVM/CMSOldGC频繁.png)

  Dump Diff 和 Leak Suspects 比较直观，这里说下其它几个关键点：

  - **内存 Dump**： 使用 jmap、arthas 等 dump 堆进行快照时记得摘掉流量，同时 **分别在 CMS GC 的发生前后分别 dump 一次** 
  - **分析 Top Component**： 要记得按照对象、类、类加载器、包等多个维度观察Histogram，同时使用 outgoing和incoming分析关联的对象，其次Soft Reference和Weak Reference、Finalizer 等也要看一下
  - **分析 Unreachable**： 重点看一下这个，关注下 Shallow 和 Retained 的大小。如下图所示的一次 GC 优化，就根据 Unreachable Objects 发现了 Hystrix 的滑动窗口问题。

  ![分析Unreachable](F:/lemon-guide-main/images/JVM/分析Unreachable.png)

  

**场景六：单次CMS Old GC耗时长**

- **现象**

  CMS GC 单次 STW 最大超过 1000ms，不会频繁发生，如下图所示最长达到了 8000ms。某些场景下会引起“雪崩效应”，这种场景非常危险，我们应该尽量避免出现。

  ![CMSGC单次STW长](F:/lemon-guide-main/images/JVM/CMSGC单次STW长.png)

- **原因**

  CMS在回收的过程中，STW的阶段主要是 Init Mark 和 Final Remark 这两个阶段，也是导致CMS Old GC 最多的原因，另外有些情况就是在STW前等待Mutator的线程到达SafePoint也会导致时间过长，但这种情况较少。

- **策略**

  知道了两个 STW 过程执行流程，我们分析解决就比较简单了，由于大部分问题都出在 Final Remark 过程，这里我们也拿这个场景来举例，主要步骤：

  - **【方向】** 观察详细 GC 日志，找到出问题时 Final Remark 日志，分析下 Reference 处理和元数据处理 real 耗时是否正常，详细信息需要通过 `-XX:+PrintReferenceGC` 参数开启。 **基本在日志里面就能定位到大概是哪个方向出了问题，耗时超过 10% 的就需要关注** 。

  ```shell
  2019-02-27T19:55:37.920+0800: 516952.915: [GC (CMS Final Remark) 516952.915: [ParNew516952.939: [SoftReference, 0 refs, 0.0003857 secs]516952.939: [WeakReference, 1362 refs, 0.0002415 secs]516952.940: [FinalReference, 146 refs, 0.0001233 secs]516952.940: [PhantomReference, 0 refs, 57 refs, 0.0002369 secs]516952.940: [JNI Weak Reference, 0.0000662 secs]
  [class unloading, 0.1770490 secs]516953.329: [scrub symbol table, 0.0442567 secs]516953.373: [scrub string table, 0.0036072 secs][1 CMS-remark: 1638504K(2048000K)] 1667558K(4352000K), 0.5269311 secs] [Times: user=1.20 sys=0.03, real=0.53 secs]
  ```

  - **【根因】** 有了具体的方向我们就可以进行深入的分析，一般来说最容易出问题的地方就是 Reference 中的 FinalReference 和元数据信息处理中的 scrub symbol table 两个阶段，想要找到具体问题代码就需要内存分析工具 MAT 或 JProfiler 了，注意要 dump 即将开始 CMS GC 的堆。在用 MAT 等工具前也可以先用命令行看下对象 Histogram，有可能直接就能定位问题。
    - 对 FinalReference 的分析主要观察 `java.lang.ref.Finalizer` 对象的 dominator tree，找到泄漏的来源。经常会出现问题的几个点有 Socket 的 `SocksSocketImpl` 、Jersey 的 `ClientRuntime`、MySQL 的 `ConnectionImpl` 等等
    - scrub symbol table 表示清理元数据符号引用耗时，符号引用是 Java 代码被编译成字节码时，方法在 JVM 中的表现形式，生命周期一般与 Class 一致，当 `_should_unload_classes` 被设置为 true 时在 `CMSCollector::refProcessingWork()` 中与 Class Unload、String Table 一起被处理

  - **【策略】** 知道 GC 耗时的根因就比较好处理了，这种问题不会大面积同时爆发，不过有很多时候单台 STW 的时间会比较长，如果业务影响比较大，及时摘掉流量，具体后续优化策略如下：
    - FinalReference：找到内存来源后通过优化代码的方式来解决，如果短时间无法定位可以增加 `-XX:+ParallelRefProcEnabled` 对 Reference 进行并行处理
    - symbol table：观察 MetaSpace 区的历史使用峰值，以及每次 GC 前后的回收情况，一般没有使用动态类加载或者 DSL 处理等，MetaSpace 的使用率上不会有什么变化，这种情况可以通过 `-XX:-CMSClassUnloadingEnabled` 来避免 MetaSpace 的处理，JDK8 会默认开启 CMSClassUnloadingEnabled，这会使得 CMS 在 CMS-Remark 阶段尝试进行类的卸载

- **小结**

  正常情况进行的 Background CMS GC，出现问题基本都集中在 Reference 和 Class 等元数据处理上，在 Reference 类的问题处理方面，不管是 FinalReference，还是 SoftReference、WeakReference 核心的手段就是找准时机 dump快照，然后用内存分析工具来分析。Class处理方面目前除了关闭类卸载开关，没有太好的方法。在 G1 中同样有 Reference 的问题，可以观察日志中的 Ref Proc，处理方法与 CMS 类似。



**场景七：内存碎片&收集器退化**

- **现象**

  并发的 CMS GC 算法，退化为 Foreground 单线程串行 GC 模式，STW 时间超长，有时会长达十几秒。其中 CMS 收集器退化后单线程串行 GC 算法有两种：

  - 带压缩动作的算法，称为 MSC，上面我们介绍过，使用标记-清理-压缩，单线程全暂停的方式，对整个堆进行垃圾收集，也就是真正意义上的 Full GC，暂停时间要长于普通 CMS
  - 不带压缩动作的算法，收集 Old 区，和普通的 CMS 算法比较相似，暂停时间相对 MSC 算法短一些

- **原因**

  CMS 发生收集器退化主要有以下几种情况：

  - **晋升失败（Promotion Failed）**

  - **增量收集担保失败**

  - **显式 GC**

  - **并发模式失败（Concurrent Mode Failure）**

- **策略**

  分析到具体原因后，我们就可以针对性解决了，具体思路还是从根因出发，具体解决策略：

  - **内存碎片**： 通过配置 `-XX:UseCMSCompactAtFullCollection=true` 来控制 Full GC 的过程中是否进行空间的整理（默认开启，注意是 Full GC，不是普通 CMS GC），以及 `-XX: CMSFullGCsBeforeCompaction=n` 来控制多少次 Full GC 后进行一次压缩
  - **增量收集**： 降低触发 CMS GC 的阈值，即参数 `-XX:CMSInitiatingOccupancyFraction` 的值，让 CMS GC 尽早执行，以保证有足够的连续空间，也减少 Old 区空间的使用大小，另外需要使用 `-XX:+UseCMSInitiatingOccupancyOnly` 来配合使用，不然 JVM 仅在第一次使用设定值，后续则自动调整
  - **浮动垃圾**： 视情况控制每次晋升对象的大小，或者缩短每次 CMS GC 的时间，必要时可调节 NewRatio 的值。另外使用 `-XX:+CMSScavengeBeforeRemark` 在过程中提前触发一次Young GC，防止后续晋升过多对象

- **小结**

  正常情况下触发并发模式的 CMS GC，停顿非常短，对业务影响很小，但 CMS GC 退化后，影响会非常大，建议发现一次后就彻底根治。只要能定位到内存碎片、浮动垃圾、增量收集相关等具体产生原因，还是比较好解决的，关于内存碎片这块，如果 `-XX:CMSFullGCsBeforeCompaction` 的值不好选取的话，可以使用 `-XX:PrintFLSStatistics` 来观察内存碎片率情况，然后再设置具体的值。最后就是在编码的时候也要避免需要连续地址空间的大对象的产生，如过长的字符串，用于存放附件、序列化或反序列化的 byte 数组等，还有就是过早晋升问题尽量在爆发问题前就避免掉。



**场景八：堆外内存OOM**

- **现象**

  内存使用率不断上升，甚至开始使用 SWAP 内存，同时可能出现 GC 时间飙升，线程被 Block 等现象， **通过 top 命令发现 Java 进程的 RES 甚至超过了** `**-Xmx**` **的大小** 。出现这些现象时，基本可确定是出现堆外内存泄漏。

- **原因**

  JVM 的堆外内存泄漏，主要有两种的原因：

  - 通过 `UnSafe#allocateMemory`，`ByteBuffer#allocateDirect` 主动申请了堆外内存而没有释放，常见于 NIO、Netty 等相关组件
  - 代码中有通过 JNI 调用 Native Code 申请的内存没有释放

- **策略**

  **原因一：主动申请未释放**

  **原因二：通过 JNI 调用的 Native Code 申请的内存未释放**



**场景九：JNI引发的GC问题**

- **现象**

  在 GC 日志中，出现 GC Cause 为 GCLocker Initiated GC。

  ```shell
  2020-09-23T16:49:09.727+0800: 504426.742: [GC (GCLocker Initiated GC) 504426.742: [ParNew (promotion failed): 209716K->6042K(1887488K), 0.0843330 secs] 1449487K->1347626K(3984640K), 0.0848963 secs] [Times: user=0.19 sys=0.00, real=0.09 secs]2020-09-23T16:49:09.812+0800: 504426.827: [Full GC (GCLocker Initiated GC) 504426.827: [CMS: 1341583K->419699K(2097152K), 1.8482275 secs] 1347626K->419699K(3984640K), [Metaspace: 297780K->297780K(1329152K)], 1.8490564 secs] [Times: user=1.62 sys=0.20, real=1.85 secs]
  ```

- **原因**

  JNI（Java Native Interface）意为 Java 本地调用，它允许 Java 代码和其他语言写的 Native 代码进行交互。JNI 如果需要获取 JVM 中的 String 或者数组，有两种方式：

  - 拷贝传递
  - 共享引用（指针），性能更高

  由于 Native 代码直接使用了 JVM 堆区的指针，如果这时发生 GC，就会导致数据错误。因此，在发生此类 JNI 调用时，禁止 GC 的发生，同时阻止其他线程进入 JNI 临界区，直到最后一个线程退出临界区时触发一次 GC。

- **策略**

  - 添加 `-XX+PrintJNIGCStalls` 参数，可以打印出发生 JNI 调用时的线程，进一步分析，找到引发问题的 JNI 调用
  - JNI 调用需要谨慎，不一定可以提升性能，反而可能造成 GC 问题
  - 升级 JDK 版本到 14，避免 [JDK-8048556](https://bugs.openjdk.java.net/browse/JDK-8048556) 导致的重复 GC



## GC调优(命令篇)

运用jvm自带的命令可以方便的在生产监控和打印堆栈的日志信息帮忙我们来定位问题！Sun JDK监控和故障处理命令如下：

| 命令   | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| jps    | JVM Process Status Tool显示指定系统内所有的HotSpot虚拟机进程。 |
| jstat  | JVM statistics Monitoring用于监视虚拟机运行时状态信息的命令  |
| jinfo  | JVM Configuration info这个命令作用是实时查看和调整虚拟机运行参数。 |
| jmap   |                                                              |
| jhat   |                                                              |
| jstack |                                                              |



![img](https://img-blog.csdn.net/20151006115425079)
![img](https://img-blog.csdn.net/20151006115455143)



### jps : 虚拟机进程状况工具

jps ( JVM Process Status Tool ) 的功能:可以列出正在运行的虚拟机进程,并显示虛拟机执行主类(Mam Class,main ( ) 函数所在的类)名称以及这些进程的本地虛拟机唯一ID ( Local Virtual Machine Identifier,LVMID ) 。

jsp命令格式:

```
jps[options][hostid]
```

**option参数**

> - -l : 输出主类全名或jar路径
> - -q : 只输出LVMID
> - -m : 输出JVM启动时传递给main()的参数
> - -v : 输出JVM启动时显示指定的JVM参数

其中[option]、[hostid]参数也可以不写。

**示例**

```dos
$ jps -l -m
  28920 org.apache.catalina.startup.Bootstrap start
  11589 org.apache.catalina.startup.Bootstrap start
  25816 sun.tools.jps.Jps -l -m
```

### jstat :虚拟机统计信息监视工具

jstat( JVM Statistics Monitoring Tool )是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据是运行期定位虚拟机性能问题的首选工具。

jstat命令格式为:

```
jstat[option vmid[interval[s|ms][count]]]
```

对于命令格式中的VMID与LVMID需要特别说明一下:如果是本地虚拟机进程,VMID与 LVMID是一致的,如果是远程虚拟机进程,那VMID的格式应当是:

```
[protocol:][//]lvmid[@hostname[:port]/servername]
```

参数interval和count代表查询间隔和次数,如果省略这两个参数,说明只查询一次。假设需要每250毫秒查询一次进程2764垃圾收集状况,一共查询20次,那命令应当是:

```
jstat -gc 2764 250 20
```

 参数

> - [option] : 操作参数
> - LVMID : 本地虚拟机进程ID
> - [interval] : 连续输出的时间间隔
> - [count] : 连续输出的次数

option **参数总览**

| Option           | Displays…                                                    |
| ---------------- | ------------------------------------------------------------ |
| class            | class loader的行为统计。Statistics on the behavior of the class loader. |
| compiler         | HotSpt JIT编译器行为统计。Statistics of the behavior of the HotSpot Just-in-Time compiler. |
| gc               | 垃圾回收堆的行为统计。Statistics of the behavior of the garbage collected heap. |
| gccapacity       | 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计。Statistics of the capacities of the generations and their corresponding spaces. |
| gcutil           | 垃圾回收统计概述。Summary of garbage collection statistics.  |
| gccause          | 垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因。Summary of garbage collection statistics (same as -gcutil), with the cause of the last and |
| gcnew            | 新生代行为统计。Statistics of the behavior of the new generation. |
| gcnewcapacity    | 新生代与其相应的内存空间的统计。Statistics of the sizes of the new generations and its corresponding spaces. |
| gcold            | 年老代和永生代行为统计。Statistics of the behavior of the old and permanent generations. |
| gcoldcapacity    | 年老代行为统计。Statistics of the sizes of the old generation. |
| gcpermcapacity   | 永生代行为统计。Statistics of the sizes of the permanent generation. |
| printcompilation | HotSpot编译方法统计。HotSpot compilation method statistics.  |

#### option 参数详解

##### -class

监视类装载、卸载数量、总空间以及耗费的时间

```ruby
$ jstat -class 11589
 Loaded  Bytes  Unloaded  Bytes     Time   
  7035  14506.3     0     0.0       3.67
```

> - Loaded : 加载class的数量
> - Bytes : class字节大小
> - Unloaded : 未加载class的数量
> - Bytes : 未加载class的字节大小
> - Time : 加载时间

##### -compiler

输出JIT编译过的方法数量耗时等

```yaml
$ jstat -compiler 1262
Compiled Failed Invalid   Time   FailedType FailedMethod
    2573      1       0    47.60          1 org/apache/catalina/loader/WebappClassLoader findResourceInternal  
```

> - Compiled : 编译数量
> - Failed : 编译失败数量
> - Invalid : 无效数量
> - Time : 编译耗时
> - FailedType : 失败类型
> - FailedMethod : 失败方法的全限定名

##### -gc

垃圾回收堆的行为统计，常用命令

```shell
$ jstat -gc 1262
 S0C    S1C     S0U     S1U   EC       EU        OC         OU        PC       PU         YGC    YGCT    FGC    FGCT     GCT   
26112.0 24064.0 6562.5  0.0   564224.0 76274.5   434176.0   388518.3  524288.0 42724.7    320    6.417   1      0.398    6.815
```

C即Capacity 总容量，U即Used 已使用的容量

> - S0C : survivor0区的总容量
> - S1C : survivor1区的总容量
> - S0U : survivor0区已使用的容量
> - S1U : survivor1区已使用的容量
> - EC : Eden区的总容量
> - EU : Eden区已使用的容量
> - OC : Old区的总容量
> - OU : Old区已使用的容量
> - PC 当前perm的容量 (KB)
> - PU perm的使用 (KB)
> - YGC : 新生代垃圾回收次数
> - YGCT : 新生代垃圾回收时间
> - FGC : 老年代垃圾回收次数
> - FGCT : 老年代垃圾回收时间
> - GCT : 垃圾回收总消耗时间

```shell
$ jstat -gc 1262 2000 20
```

这个命令意思就是每隔2000ms输出1262的gc情况，一共输出20次

##### -gccapacity

同-gc，不过还会输出Java堆各区域使用到的最大、最小空间

```shell
$ jstat -gccapacity 1262
 NGCMN    NGCMX     NGC    S0C   S1C       EC         OGCMN      OGCMX      OGC        OC       PGCMN    PGCMX     PGC      PC         YGC    FGC 
614400.0 614400.0 614400.0 26112.0 24064.0 564224.0   434176.0   434176.0   434176.0   434176.0 524288.0 1048576.0 524288.0 524288.0    320     1  
```

> - NGCMN : 新生代占用的最小空间
> - NGCMX : 新生代占用的最大空间
> - OGCMN : 老年代占用的最小空间
> - OGCMX : 老年代占用的最大空间
> - OGC：当前年老代的容量 (KB)
> - OC：当前年老代的空间 (KB)
> - PGCMN : perm占用的最小空间
> - PGCMX : perm占用的最大空间

##### -gcutil

同-gc，不过输出的是已使用空间占总空间的百分比

```shell
$ jstat -gcutil 28920
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
 12.45   0.00  33.85   0.00   4.44  4       0.242     0    0.000    0.242
```

##### -gccause

垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因

```shell
$ jstat -gccause 28920
  S0     S1     E      O      P       YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
 12.45   0.00  33.85   0.00   4.44      4    0.242     0    0.000    0.242   Allocation Failure   No GC  
```

> - LGCC：最近垃圾回收的原因
> - GCC：当前垃圾回收的原因

##### -gcnew

统计新生代的行为

```shell
$ jstat -gcnew 28920
 S0C      S1C      S0U        S1U  TT  MTT  DSS      EC        EU         YGC     YGCT  
 419392.0 419392.0 52231.8    0.0  6   6    209696.0 3355520.0 1172246.0  4       0.242
```

> - TT：Tenuring threshold(提升阈值)
> - MTT：最大的tenuring threshold
> - DSS：survivor区域大小 (KB)

##### -gcnewcapacity

新生代与其相应的内存空间的统计

```shell
$ jstat -gcnewcapacity 28920
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC        YGC   FGC 
 4194304.0  4194304.0  4194304.0 419392.0 419392.0 419392.0 419392.0  3355520.0  3355520.0     4     0
```

> - NGC:当前年轻代的容量 (KB)
> - S0CMX:最大的S0空间 (KB)
> - S0C:当前S0空间 (KB)
> - ECMX:最大eden空间 (KB)
> - EC:当前eden空间 (KB)

##### -gcold

统计旧生代的行为

```shell
$ jstat -gcold 28920
   PC       PU        OC           OU       YGC    FGC    FGCT     GCT   
1048576.0  46561.7   6291456.0     0.0      4      0      0.000    0.242
```

##### -gcoldcapacity

统计旧生代的大小和空间

```shell
$ jstat -gcoldcapacity 28920
   OGCMN       OGCMX        OGC         OC         YGC   FGC    FGCT     GCT   
  6291456.0   6291456.0   6291456.0   6291456.0     4     0    0.000    0.242
```

##### -gcpermcapacity

永生代行为统计

```shell
$ jstat -gcpermcapacity 28920
    PGCMN      PGCMX       PGC         PC      YGC   FGC    FGCT     GCT   
 1048576.0  2097152.0  1048576.0  1048576.0     4     0    0.000    0.242
```

##### -printcompilation

hotspot编译方法统计

```shell
$ jstat -printcompilation 28920
    Compiled  Size  Type Method
    1291      78     1    java/util/ArrayList indexOf
```

> - Compiled：被执行的编译任务的数量
> - Size：方法字节码的字节数
> - Type：编译类型
> - Method：编译方法的类名和方法名。类名使用”/” 代替 “.” 作为空间分隔符. 方法名是给出类的方法名. 格式是一致于HotSpot - XX:+PrintComplation 选项

### jinfo : Java配置信息工具

jinfo ( Configuration Info for Java ) 的作用是实时地查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表,但如果想知道未被显式指定的参数的系统默认值,除了去找资料外,就只能使用jinfo的-flag选项进行查询了(如果只限于 JDK 1.6或以上版本的话,使用java-XX : +PrintFlagsFinal查看参数默认值也是一个很好的选择 ),jinfo还可以使用-sysprops选项把虚拟机进程的System.getProperties() 的内容打印出来。这个命令在JDK 1.5时期已经随着Linux版的JDK发 布 ,当时只提供了信息查询的功能 ,JDK 1.6之后,jinfo在Windows和Linux平台都有提供,并且加入了运行期修改参数的能力 ,可以使用-flag[+|-jname或者-flag name=value修改一部分运行期可写的虚拟机参数值。 JDK 1.6中,jinfo对手Windows平台功能仍然有较大限制,只提供了最基本的-flag选项。

jinfo命令格式:

```
jinfo[option]pid
```

 

执行样例:查询CMSInitiatingOccupancyFraction参数值。

```
C:\>jinfo -flag CMSInitiatingOccupancyFraction 1444 
-XX :CMSInitiatingOccupancyFraction=85
```

 



### jmap : Java内存映像工具

jmap ( Memory Map for Java ) 命令用于生成堆转储快照(一般称为heapdump或dump文件 )。如果不使用jmap命令,要想获取Java堆转储快照,还有一些比较“暴力”的手段:譬如-XX : +HeapDumpOnOutOfMemoryError参数,可以让虚拟机在OOM异常出现之后自动生成dump文件,通过-XX : +HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break] 键让虚拟机生成dump文件 ,又或者在Linux系统下通过Kill -3命令发送进程退出信号“吓唬”下虚拟机,也能拿到dump文件。

jmap的作用并不仅仅是为了获取dump文件,它还可以查询finalize执行队列、Java堆和永久代的详细信息,如空间使用率、当前用的是哪种收集器等。

和jinfo命令一样,jmap有不少功能在Windows平台下都是受限的,除了生成dump文件的-dump选项和用于查看每个类的实例、空间占用统计的-histo选项在所有操作系统都提供之外 ,其余选项都只能在Linux/Solaris下使用。

jmap命令格式:

```
jmap[option]vmid
```

 

option 选项的合法值与具体含义见表4-4。

![img](https://img-blog.csdn.net/20151006130957401)

代码清单4-2是使用jmap生成一个正在运行的Eclipse的dump快照文件的例子,例子中的3500是通过jps命令查询到的LVMID。

代码清单4-2 使用jmap生成dump文件

```
￼C:\Users\IcyFenix>jmap-dump:format=b,file=eclipse.bin 3500 
Dumping heap to C :\Users\IcyFenix\eclipse.bin.
Heap dump file created
```

 



### jhat :虚拟机堆转储快照分析工具

Sun JDK提供jhat(JVM Heap Analysis Tool)命令与jmap搭配使用,来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/HTML服务器 ,生成dump文件的分析结果后,可以在浏览器中查看。不过[实事求是](https://www.baidu.com/s?wd=实事求是&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)地说,在实际工作中,除非笔者手上真的没有别的工具可用, 否则一般都不会去直接使用jhat命令来分析dump文件 ,主要原因有二:一是一般不会在部署应用程序的服务器上直接分析dump文 件 ,即使可以这样做,也会尽量将dump文件复制到其他机器上进行分析,因为分析工作是一个耗时而且消耗硬件资源的过程,既然都要在其他机器进行,就没有必要受到命令行工具的限制了;另一个原因是jhat的分析功能相对来说比较简陋,后文将会介绍到的VisualVM , 以及专业用于分析dump文件的Eclipse Memory Analyzer、 IBM HeapAnalyzer等工具,都能实现比jhat更强大更专业的分析功能。代码清单4-3演示了使用jhat分析4.2.4节中采用jmap生成的Eclipse IDE的内存快照文件。

代码清单4-3 使用jhat分析dump文件

```
C:\Users\IcyFenix>jhat eclipse.bin
Reading from eclipse.bin.
Dump file created Fri Nov 19 22 :07 :21 CST 2010 Snapshot read,resolving.
Resolving 1225951 objects.
Chasing references,expect 245 dots...... Eliminating duplicate references
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

屏幕显不“Server is ready.”的提示后,用户在浏览器中键入http://localhost:7000/就可以 看到分析结果,如图4-3所示。

![img](https://img-blog.csdn.net/20151006132127307)

分析结果默认是以包为单位进行分组显示,分析内存泄漏问题主要会使用到其中 的“Heap Histogram” (与jmap-histo功能一样)与OQL页签的功能,前者可以找到内存中总容量最大的对象,后者是标准的对象查询语言,使用类似SQL的语法对内存中的对象进行查询统计。



### jstack : Java堆栈跟踪工具

jstack(Stack Trace for Java)命令用于生成虚拟机当前时刻的线程快照(一般称为 threaddump或者javacore文件 )。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合 ,生成线程快照的主要目的是定位线程出现长时间停顿的原因,如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿的时候通过jstack来查看各个线程的调用堆栈,就可以知道没有响应的线程到底在后台做些什么事情,或者等待着什么资源。

jstack命令格式:

```
jstack [option] vmid
```

 

option选项的合法值与具体含义见表4-5。

![img](https://img-blog.csdn.net/20151006142939347)

代码清单4-4是使用jstack查看Eclipse线程堆栈的例子,例子中的3500是通过jps命令查询到的LVMID。

代码清单4 - 4 使用jstack查看线程堆栈(部分结果)

![img](https://img-blog.csdn.net/20151006143152943)

在JDK 1.5中 ,java.lang.Thread类新增了一个getAllStackTraces()用于获取虚拟机中所有线程的StackTraceElement对象。使用这个方法可以通过简单的几行代码就完成jstack的大部分功能,在实际项目中不妨调用这个方法做个管理员页面,可以随时使用浏览器来查看线程堆栈,如代码清单4-5所示,这是笔者的一个小经验。

代码清单4 - 5 查看线程状况的JSP页面

```
<%@ page import="java.util.Map"%>

<html>
<head>
<title>服务器线程信息</title>
</head>
<body>
<pre>
<%
    for (Map.Entry<Thread, StackTraceElement[]> stackTrace : Thread.getAllStackTraces().entrySet()) {
        Thread thread = (Thread) stackTrace.getKey();
        StackTraceElement[] stack = (StackTraceElement[]) stackTrace.getValue();
        if (thread.equals(Thread.currentThread())) {
            continue;
        }
        out.print("\n线程：" + thread.getName() + "\n");
        for (StackTraceElement element : stack) {
            out.print("\t"+element+"\n");
        }
    }
%>
</pre>
</body>
</html>
```



### HSDIS : JIT生成代码反汇编

在Java虚拟机规范中,详细描述了虚拟机指令集中每条指令的执行过程、执行前后对操作数栈、局部变量表的影响等细节。这些细节描述与Sun的早期虚拟机( Sun Classic VM)高度吻合 ,但随着技术的发展,高性能虚拟机真正的细节实现方式已经渐渐与虚拟机规范所描述的内容产生了越来越大的差距,虚拟机规范中的描述逐渐成了虚拟机实现的“概念模型”— 即实现只能保证规范描述等效。基于这个原因,我们分析程序的执行语义问题(虚拟机做了什么)时 ,在字节码层面上分析完全可行,但分析程序的执行行为问题(虚拟机是怎样做的、性能如何)时 ,在字节码层面上分析就没有什么意义了,需要通过其他方式解决。

分析程序如何执行,通过软件调试工具(GDB、Windbg等 )来断点调试是最常见的手段 ,但是这样的调试方式在Java虚拟机中会遇到很大困难,因为大量执行代码是通过JIT编译器动态生成到CodeBuffer中的 ,没有很简单的手段来处理这种混合模式的调试(不过相信虚拟机开发团队内部肯定是有内部工具的)。因此,不得不通过一些特别的手段来解决问题, 基于这种背景,本节的主角——HSDIS插件就正式登场了。

HSDIS是一个Sun官方推荐的HotSpot虚拟机JIT编译代码的反汇编插件,它包含在HotSpot虚拟机的源码之中,但没有提供编译后的程序。在Project Kerni的网站也可以下载到单独的源码。它的作用是让HotSpot的-XX : +PrintAssembly指令调用它来把动态生成的本地代码还原为汇编代码输出,同时还生成了大量非常有价值的注释,这样我们就可以通过输出的代码来分析问题。读者可以根据自己的操作系统和CPU类型从Project Kenai的网站上下载编译好的插件,直接放到JDK_HOME/jre/bin/client和JDK_HOME/jre/bin/server目录中即可。如果没 有找到所需操作系统(譬如Windows的就没有 )的成品 ,那就得自己使用源码编译一下。

还需要注意的是,如果读者使用的是Debug或者FastDebug版的HotSpot ,那可以直接通过-XX : +PrintAssembly指令使用插件;如果使用的是Product版的HotSpot , 那还要额外加入一个-XX : +UnlockDiagnosticVMOptions参数。笔者以代码清单4-6中的简单测试代码为例演示一下这个插件的使用。

代码清单4 - 6 测试代码

```
public class Bar {
    int a = 1;
    static int b = 2;

    public int sum(int c) {
        return a + b + c;
    }

    public static void main(String[] args) {
        new Bar().sum(3);
    }
}
```

编译这段代码,并使用以下命令执行。

```
java -XX:+PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*Bar.sum -XX:CompileCommand=compileonly,*Bar.sum test.Bar
```

 

其中 ,参数-Xcomp是让虚拟机以编译模式执行代码,这样代码可以“偷懒”,不需要执行足够次数来预热就能触发JIT编译。两个-XX : CompileCommand意思是让编译器不要内联sum()并且只编译sum() , -XX : +PrintAssembly就是输出反汇编内容。如果一也顺利的话 ,那么屏幕上会出现类似下面代码清单4-7所示的内容。

代码清单4 - 7 测试代码

![img](https://img-blog.csdn.net/20151006145254167)

![img](https://img-blog.csdn.net/20151006145427402)



## GC调优(工具篇)

理论总是作为指导实践的工具,能把这些知识应用到实际工作中才是 我们的最终目的。

给一个系统定位问题的时候,知识、经验是关键基础,数据是依据,工具是运用知识处理数据的手段。这里说的数据包括:运行日志、异常堆栈、GC日志、线程快照( threaddump/javacore文件)、堆转储快照(heapdump/hprof文件)等。

### JMX API:  JVM内部运行时状态信息标准API

从 JVM 运行时获取GC行为数据, 最简单的办法是使用标准 JMX API 接口. JMX是获取 JVM内部运行时状态信息 的标准API. 可以编写程序代码, 通过 JMX API 来访问本程序所在的JVM，也可以通过JMX客户端执行(远程)访问。

### JConsole: Java监视与管理控制台

JConsole ( Java Monitoring and Management Console ) 是—种基于JMX的可视化监视管理工具。它管理部分的功能是针对JMX MBean进行管理,由于MBean可以使用代码、中间件服务器的管理控制台或者所有符合JMX规范的软件进行访问,所以本节将会着重介绍JConsole监视部分的功能。

### VisualVM:多合一故障处理工具

VisualVM(All-in-One Java Troubleshooting Tool)是到目前为止随JDK发布的功能最强大的运行监视和故障处理程序,并且可以预见在未来一段时间内都是官方主力发展的虚拟机故障处理工具。

通过插件扩展支持,VisualVM可以做到:

- 显示虚拟机进程以及进程的配置、环境信息(jps、 jinfo)。
- 监视应用程序的CPU、GC、堆、方法区以及线程的信息(jstat、jstack)。
- dump以及分析堆转储快照(jmap、jhat)。
- 方法级的程序运行性能分析,找出被调用最多、运行时间最长的方法。
- 离线程序快照:收集程序的运行时配置、线程dump、内存dump等信息建立个快照, 可以将快照发送开发者处进行Bug反馈。
- 其他plugins的无限的可能性……

### BTrace:动态日志跟踪

BTrace是一个很“有趣”的VisualVM插件 ,本身也是可以独立运行的程序。它的作用是在不停止目标程序运行的前提下,通过HotSpot虚拟机的HotSwap技术动态加入原本并不存在的调试代码。这项功能对实际生产中的程序很有意义:经常遇到程序出现问题,但排查错误的一些必要信息,譬如方法参数、返回值等,在开发时并没有打印到日志之中,以至于不得不停掉服务,通过调试增量来加入日志代码以解决问题。当遇到生产环境服务无法随便停止时 ,缺一两句日志导致排错进行不下去是一件非常郁闷的事情。

### GCViewerGC:日志分析工具

### Profilers:分析器

### AProf

## GC调优(实战篇)

### 磁盘不足排查

其实，磁盘不足排查算是系统、程序层面的问题排查，并不算是JVM，但是另一方面考虑过来就是，系统磁盘的不足，也会导致JVM的运行异常，所以也把磁盘不足算进来了。并且排查磁盘不足，是比较简单，就是几个命令，然后就是逐层的排查，首先第一个命令就是**df -h**，查询磁盘的状态： 

从上面的显示中其中第一行使用的2.8G最大，然后是挂载在 **/** 目录下，我们直接**cd /**。然后通过执行：

```shell
du -sh *
```

查看各个文件的大小，找到其中最大的，或者说存储量级差不多的并且都非常大的文件，把那些没用的大文件删除就好。然后，就是直接cd到对应的目录也是执行：du -sh *，就这样一层一层的执行，找到对应的没用的，然后文件又比较大的，可以直接删除。

### CPU过高排查

**排查过程**

- 使用`top`查找进程id
- 使用`top -Hp <pid>`查找进程中耗cpu比较高的线程id
- 使用`printf %x <pid>`将线程id十进制转十六进制
- 使用` jstack -pid | grep -A 20 <pid>`过滤出线程id锁关联的栈信息
- 根据栈信息中的调用链定位业务代码

案例代码如下：

```java
public class CPUSoaring {
        public static void main(String[] args) {

                Thread thread1 = new Thread(new Runnable(){
                        @Override
                        public void run() {
                                for (;;){
                                      System.out.println("I am children-thread1");
                                }
                        }
                },"children-thread1");
                
                 Thread thread2 = new Thread(new Runnable(){
                        @Override
                        public void run() {
                                for (;;){
                                      System.out.println("I am children-thread2");
                                }
                        }
                },"children-thread2");
                
                thread1.start();
                thread2.start();
                System.err.println("I am is main thread!!!!!!!!");
        }
}
```

- 第一步：首先通过**top**命令可以查看到id为**3806**的进程所占的CPU最高：

- 第二步：然后通过**top -Hp pid**命令，找到占用CPU最高的线程：

- 第三步：接着通过：**printf '%x\n' tid**命令将线程的tid转换为十六进制：xid：

- 第四步：最后通过：**jstack pid|grep xid -A 30**命令就是输出线程的堆栈信息，线程所在的位置：

- 第五步：还可以通过**jstack -l pid > 文件名称.txt** 命令将线程堆栈信息输出到文件，线下查看。

  这就是一个CPU飙高的排查过程，目的就是要**找到占用CPU最高的线程所在的位置**，然后就是**review**你的代码，定位到问题的所在。使用Arthas的工具排查也是一样的，首先要使用top命令找到占用CPU最高的Java进程，然后使用Arthas进入该进程内，**使用dashboard命令排查占用CPU最高的线程。**，最后通过**thread**命令线程的信息。

### 内存打满排查

**排查过程**

- 查找进程id：`top -d 2 -c`
- 查看JVM堆内存分配情况：`jmap -heap pid`
- 查看占用内存比较多的对象：`jmap -histo pid | head -n 100`
- 查看占用内存比较多的存活对象：`jmap -histo:live pid | head -n 100`

**示例**

- 第一步：top -d 2 -c 

- 第二步：jmap -heap 8338

- 第三步：定位占用内存比较多的对象

  这里就能看到对象个数以及对象大小……

  这里看到一个自定义的类，这样我们就定位到具体对象，看看这个对象在那些地方有使用、为何会有大量的对象存在。

### OOM异常排查

OOM的异常排查也比较简单，首先服务上线的时候，要先设置这两个参数：

```shell
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${目录}
```

指定项目出现OOM异常的时候自动导出堆转储文件，然后通过内存分析工具（**Visual VM**）来进行线下的分析。

首先我们来聊一聊，哪些原因会导致OOM异常，站在JVM的分区的角度：

- **Java堆**
- **方法区**
- **虚拟机栈**
- **本地方法栈**
- **程序计数器**
- **直接内存**

只有**程序计数器**区域不会出现OOM，在Java 8及以上的**元空间**（本地内存）都会出现OOM。

而站在程序代码的角度来看，总结了大概有以下几点原因会导致OOM异常：

- **内存泄露**
- **对象过大、过多**
- **方法过长**
- **过度使用代理框架，生成大量的类信息**

接下来我们屋来看看OOM的排查，出现OOM异常后dump出了堆转储文件，然后打开jdk自带的Visual VM工具，导入堆转储文件，首先我使用的OOM异常代码如下：

```java
import java.util.ArrayList;
import java.util.List;

class OOM {

        static class User{
                private String name;
                private int age;

                public User(String name, int age){
                        this.name = name;
                        this.age = age;
                }
        }

        public static void main(String[] args) throws InterruptedException {
                List<User> list = new ArrayList<>();
                for (int i = 0; i < Integer.MAX_VALUE; i++) {
                        Thread.sleep(1000);
                        User user = new User("zhangsan"+i,i);
                        list.add(user);
                }
        }

}
```

代码很简单，就是往集合里面不断地add对象，带入堆转储文件后，在类和实例那栏就可以看到实例最多的类：

这样就找到导致OOM异常的类，还可以通过下面的方法查看导致OOM异常的线程堆栈信息，找到对应异常的代码段。

上面的方法是排查已经出现了OOM异常的方法，肯定是防线的最后一步，那么在此之前怎么防止出现OOM异常呢？

一般大厂都会有自己的监控平台，能够实施的**监控测试环境、预览环境、线上实施的服务健康状况（CPU、内存）** 等信息，对于频繁GC，并且GC后内存的回收率很差的，就要引起我们的注意了。

因为一般方法的长度合理，95%以上的对象都是朝生夕死，在**Minor GC**后只剩少量的存活对象，所以在代码层面上应该避免**方法过长、大对象**的现象。

每次自己写完代码，自己检查后，都可以提交给比自己高级别的工程师**review**自己的代码，就能及时的发现代码的问题，基本上代码没问题，百分之九十以上的问题都能避免，这也是大厂注重代码质量，并且时刻**review**代码的习惯。



## Jvisualvm

项目频繁YGC 、FGC问题排查

### 内存问题

对象内存占用、实例个数监控

![img](F:/lemon-guide-main/images/JVM/20200119164751943.png)


对象内存占用、年龄值监控

![img](F:/lemon-guide-main/images/JVM/2020011916475242.png)


通过上面两张图发现这些对象占用内存比较大而且存活时间也是比较常，所以survivor 中的空间被这些对象占用，而如果缓存再次刷新则会创建同样大小对象来替换老数据，这时发现eden内存空间不足，就会触发yonggc 如果yonggc 结束后发现eden空间还是不够则会直接放到老年代，所以这样就产生了大对象的提前晋升，导致fgc增加……

**优化办法**：优化两个缓存对象，将缓存对象大小减小。优化一下两个对象，缓存关键信息！



### CPU耗时问题排查

Cpu使用耗时监控：

![img](F:/lemon-guide-main/images/JVM/20200119164752266.png)

耗时、调用次数监控：

![img](F:/lemon-guide-main/images/JVM/20200119164749513.png)

从上面监控图可以看到主要耗时还是在网络请求，没有看到具体业务代码消耗过错cpu……



## 调优堆内存分配

**初始堆空间大小设置**

- 使用系统默认配置在系统稳定运行一段时间后查看记录内存使用情况：Eden、survivor0 、survivor1 、old、metaspace
- 按照通用法则通过gc信息分配调整大小，整个堆大小是Full GC后老年代空间占用大小的3-4倍
- 老年代大小为Full GC后老年代空间占用大小的2-3倍
- 新生代大小为Full GC后老年代空间占用大小的1-1.5倍 
- 元数据空间大小为Full GC后元数据空间占用大小的1.2-1.5倍

活跃数大小是应用程序运行在稳定态时，长期存活的对象在java堆中占用的空间大小。也就是在应用趋于稳太时FullGC之后Java堆中存活对象占用空间大小。（注意在jdk8中将jdk7中的永久代改为元数据区，metaspace 使用的物理内存，不占用堆内存）



**堆大小调整的着手点、分析点**

- 统计Minor GC 持续时间
- 统计Minor GC 的次数
- 统计Full GC的最长持续时间
- 统计最差情况下Full GC频率
- 统计GC持续时间和频率对优化堆的大小是主要着手点，我们按照业务系统对延迟和吞吐量的需求，在按照这些分析我们可以进行各个区大小的调整



## 年轻代调优

**年轻代调优规则**

- 老年代空间大小不应该小于活跃数大小1.5倍。老年代空间大小应为老年代活跃数2-3倍
- 新生代空间至少为java堆内存大小的10% 。新生代空间大小应为1-1.5倍的老年代活跃数
- 在调小年轻代空间时应保持老年代空间不变



MinorGC是收集eden+from survivor 区域的，当业务系统匀速生成对象的时候如果年轻带分配内存偏小会发生频繁的MinorGC，如果分配内存过大则会导致MinorGC停顿时间过长，无法满足业务延迟性要求。所以按照堆分配空间分配之后分析gc日志，看看MinorGC的频率和停顿时间是否满足业务要求。

- **MinorGC频繁原因**
  MinorGC 比较频繁说明eden内存分配过小，在恒定的对象产出的情况下很快无空闲空间来存放新对象所以产生了MinorGC,所以eden区间需要调大。

- **年轻代大小调整**
  Eden调整到多大那，我们可以查看GC日志查看业务多长时间填满了eden空间，然后按照业务能承受的收集频率来计算eden空间大小。比如eden空间大小为128M每5秒收集一次，则我们为了达到10秒收集一次则可以调大eden空间为256M这样能降低收集频率。年轻代调大的同时相应的也要调大老年代，否则有可能会出现频繁的concurrent model  failed  从而导致Full GC 。

- **MinorGC停顿时间过长**
  MinorGC 收集过程是要产生STW的。如果年轻代空间太大，则gc收集时耗时比较大，所以我们按业务对停顿的要求减少内存，比如现在一次MinorGC 耗时12.8毫秒，eden内存大小192M ,我们为了减少MinorGC 耗时我们要减少内存。比如我们MinorGC 耗时标准为10毫秒，这样耗时减少16.6% 同样年轻代内存也要减少16.6% 即192*0.1661 = 31.89M 。年轻代内存为192-31.89=160.11M,在减少年轻代大小，而要保持老年代大小不变则要减少堆内存大小至512-31.89=480.11M



堆内存：512M 年轻代: 192M  收集11次耗时141毫秒 12.82毫秒/次

![img](F:/lemon-guide-main/images/JVM/20200119164911457.png)

堆内存：512M 年轻代：192M 收集12次耗时151毫秒   12.85毫秒/次

![img](F:/lemon-guide-main/images/JVM/20200119164911559.png)

**按照上面计算调优**
堆内存： 480M 年轻带： 160M 收集14次 耗时154毫秒  11毫秒/次 相比之前的 12.82毫秒/次 停顿时间减少1.82毫秒

![img](F:/lemon-guide-main/images/JVM/20200119164911817.png)

**但是还没达到10毫秒的要求，继续按照这样的逻辑进行  11-10=1 ；1/11= 0.909 即 0.09 所以耗时还要降低9%。**
年轻代减少：160*0.09 = 14.545=14.55 M;  160-14.55 =145.45=145M 
堆大小： 480-14.55 = 465.45=465M
但是在这样调整后使用jmap -heap 查看的时候年轻代大小和实际配置数据有出入（年轻代大小为150M大于配置的145M），这是因为-XX:NewRatio 默认2 即年轻代和老年代1：2的关系，所以这里将-XX:NewRatio 设置为3 即年轻代、老年大小比为1：3 ，最终堆内存大小为：

![img](F:/lemon-guide-main/images/JVM/20200119164913228.png)

MinorGC耗时 159/16=9.93毫秒

![img](F:/lemon-guide-main/images/JVM/20200119164912847.png)

MinorGC耗时 185/18=10.277=10.28毫秒

![img](F:/lemon-guide-main/images/JVM/2020011916490666.png)

MinorGC耗时 205/20=10.25毫秒

![img](F:/lemon-guide-main/images/JVM/20200119164912355.png)


Ok 这样MinorGC停顿时间过长问题解决，MinorGC要么比较频繁要么停顿时间比较长，解决这个问题就是调整年轻代大小，但是调整的时候还是要遵守这些规则。



## 老年代调优

按照同样的思路对老年代进行调优，同样分析FullGC 频率和停顿时间，按照优化设定的目标进行老年代大小调整。

**老年代调优流程**

- 分析每次MinorGC 之后老年代空间占用变化，计算每次MinorGC之后晋升到老年代的对象大小
- 按照MinorGC频率和晋升老年代对象大小计算提升率即每秒钟能有多少对象晋升到老年代
- FullGC之后统计老年代空间被占用大小计算老年带空闲空间，再按照第2部计算的晋升率计算该老年代空闲空间多久会被填满而再次发生FullGC，同样观察FullGC 日志信息，计算FullGC频率，如果频率过高则可以增大老年代空间大小老解决，增大老年代空间大小应保持年轻代空间大小不变
- 如果在FullGC 频率满足优化目标而停顿时间比较长的情况下可以考虑使用CMS、G1收集器。并发收集减少停顿时间

## 栈溢出

栈溢出异常的排查（包括**虚拟机栈、本地方法栈**）基本和OOM的一场排查是一样的，导出异常的堆栈信息，然后使用mat或者Visual VM工具进行线下分析，找到出现异常的代码或者方法。

当线程请求的栈深度大于虚拟机栈所允许的大小时，就会出现**StackOverflowError**异常，二从代码的角度来看，导致线程请求的深度过大的原因可能有：**方法栈中对象过大，或者过多，方法过长从而导致局部变量表过大，超过了-Xss参数的设置**。



## 死锁排查

死锁的案例演示的代码如下：

```java
public class DeadLock {

	public static Object lock1 = new Object();
	public static Object lock2 = new Object();

	public static void main(String[] args){
		Thread a = new Thread(new Lock1(),"DeadLock1");
		Thread b = new Thread(new Lock2(),"DeadLock2");
		a.start();
		b.start();
	}
}
class Lock1 implements Runnable{
	@Override
	public void run(){
		try{
			while(true){
				synchronized(DeadLock.lock1){
					System.out.println("Waiting for lock2");
					Thread.sleep(3000);
					synchronized(DeadLock.lock2){
						System.out.println("Lock1 acquired lock1 and lock2 ");
					}
				}
			}
		}catch(Exception e){
			e.printStackTrace();
		}
	}
}
class Lock2 implements Runnable{
	@Override
	public void run(){
		try{
			while(true){
				synchronized(DeadLock.lock2){
					System.out.println("Waiting for lock1");
					Thread.sleep(3000);
					synchronized(DeadLock.lock1){
						System.out.println("Lock2 acquired lock1 and lock2");
					}
				}
			}
		}catch(Exception e){
			e.printStackTrace();
		}
	}
}
```

上面的代码非常的简单，就是两个类的实例作为锁资源，然后分别开启两个线程，不同顺序的对锁资源资源进行加锁，并且获取一个锁资源后，等待三秒，是为了让另一个线程有足够的时间获取另一个锁对象。

运行上面的代码后，就会陷入死锁的僵局：

![死锁排查-死锁运行结果示例](F:/lemon-guide-main/images/JVM/死锁排查-死锁运行结果示例.jpg)

对于死锁的排查，若是在测试环境或者本地，直接就可以使用Visual VM连接到该进程，如下界面就会自动检测到死锁的存在

![死锁排查-检测死锁](F:/lemon-guide-main/images/JVM/死锁排查-检测死锁.jpg)

并且查看线程的堆栈信息。就能看到具体的死锁的线程：

![死锁排查-查看线程堆栈信息](F:/lemon-guide-main/images/JVM/死锁排查-查看线程堆栈信息.jpg)

线上的话可以上用Arthas也可以使用原始的命令进行排查，原始命令可以先使用**jps**查看具体的Java进程的ID，然后再通过**jstack ID**查看进程的线程堆栈信息，他也会自动给你提示有死锁的存在：

![死锁排查-jstack查看线程堆栈](F:/lemon-guide-main/images/JVM/死锁排查-jstack查看线程堆栈.jpg)

Arthas工具可以使用**thread**命令排查死锁，要关注的是**BLOCKED**状态的线程，如下图所示：

![死锁排查-Arthas查看死锁](F:/lemon-guide-main/images/JVM/死锁排查-Arthas查看死锁.jpg)

具体thread的详细参数可以参考如下图所示：

![死锁排查-Thread详细参数](F:/lemon-guide-main/images/JVM/死锁排查-Thread详细参数.jpg)



**如何避免死锁**

上面我们聊了如何排查死锁，下面我们来聊一聊如何避免死锁的发生，从上面的案例中可以发现，死锁的发生两个线程同时都持有对方不释放的资源进入僵局。所以，在代码层面，要避免死锁的发生，主要可以从下面的四个方面进行入手：

- **首先避免线程的对于资源的加锁顺序要保持一致**

- **并且要避免同一个线程对多个资源进行资源的争抢**

- **另外的话，对于已经获取到的锁资源，尽量设置失效时间，避免异常，没有释放锁资源，可以使用acquire() 方法加锁时可指定 timeout 参数**

- **最后，就是使用第三方的工具检测死锁，预防线上死锁的发生**

死锁的排查已经说完了，上面的基本就是问题的排查，也可以算是调优的一部分吧，但是对于JVM调优来说，重头戏应该是在**Java堆**，这部分的调优才是重中之重。



## 调优实战

上面说完了调优的目的和调优的指标，那么我们就来实战调优，首先准备我的案例代码，如下：

```java
import java.util.ArrayList;
import java.util.List;

class OOM {

	static class User{
		private String name;
		private int age;

		public User(String name, int age){
			this.name = name;
			this.age = age;
		}

	}

	public static void main(String[] args) throws InterruptedException {
		List<User> list = new ArrayList<>();
		for (int i = 0; i < Integer.MAX_VALUE; i++) {
		     Tread.sleep(1000);
			System.err.println(Thread.currentThread().getName());
			User user = new User("zhangsan"+i,i);
			list.add(user);
		}
	}
}
```

案例代码很简单，就是不断的往一个集合里里面添加对象，首先初次我们启动的命令为：

```shell
java   -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:+PrintHeapAtGC -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=50M -Xloggc:./logs/emps-gc-%t.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs/emps-heap.dump OOM
```

就是纯粹的设置了一些GC的打印日志，然后通过Visual VM来看GC的显示如下：

![调优实战-VisualVM查看GC显示](F:/lemon-guide-main/images/JVM/调优实战-VisualVM查看GC显示.jpg)

可以看到一段时间后出现4次Minor GC，使用的时间是29.648ms，发生一次Full GC使用的时间是41.944ms。

Minor GC非常频繁，Full GC也是，在短时间内就发生了几次，观察输出的日志发现以及Visual VM的显示来看，都是因为内存没有设置，太小，导致Minor GC频繁。

因此，我们第二次适当的增大Java堆的大小，调优设置的参数为：

```shell
java -Xmx2048m -Xms2048m -Xmn1024m -Xss256k  -XX:+UseConcMarkSweepGC  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:+PrintHeapAtGC -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=50M -Xloggc:./logs/emps-gc-%t.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs/emps-heap.dump OOM
```

观察一段时间后，结果如下图所示：

![调优实战-一段时间后VisualVM查看GC显示](F:/lemon-guide-main/images/JVM/调优实战-一段时间后VisualVM查看GC显示.jpg)

可以发现Minor GC次数明显下降，但是还是发生了Full GC，根据打印的日志来看，是因为元空间的内存不足，看了上面的Visual VM元空间的内存图，也是一样，基本都到顶了：

![调优实战-元空间不足](F:/lemon-guide-main/images/JVM/调优实战-元空间不足.jpg)

因此第三次对于元空间的区域设置大一些，并且将GC回收器换成是CMS的，设置的参数如下：

```shell
java -Xmx2048m -Xms2048m -Xmn1024m -Xss256k -XX:MetaspaceSize=100m -XX:MaxMetaspaceSize=100m -XX:+UseConcMarkSweepGC  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:+PrintHeapAtGC -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=50M -Xloggc:./logs/emps-gc-%t.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs/emps-heap.dump OOM
```

观察相同的时间后，Visual VM的显示图如下：

![调优实战-元空间调后后VisualVM查看GC显示](F:/lemon-guide-main/images/JVM/调优实战-元空间调后后VisualVM查看GC显示.jpg)

同样的时间，一次Minor GC和Full GC都没有发生，所以这样我觉得也算是已经调优了。

但是调优并不是一味的调大内存，是要在各个区域之间取得平衡，可以适当的调大内存，以及更换GC种类，举个例子，当把上面的案例代码的Thread.sleep(1000)给去掉。

然后再来看Visual VM的图，如下：

![调优实战-去掉线程休眠VisualVM显示](F:/lemon-guide-main/images/JVM/调优实战-去掉线程休眠VisualVM显示.jpg)

可以看到Minor GC也是非常频繁的，因为这段代码本身就是不断的增大内存，直到OOM异常，真正的实际并不会这样，可能当内存增大到一定两级后，就会在一段范围平衡。

当我们将上面的情况，再适当的增大内存，JVM参数如下：

```shell
java -Xmx4048m -Xms4048m -Xmn2024m -XX:SurvivorRatio=7  -Xss256k -XX:MetaspaceSize=300m -XX:MaxMetaspaceSize=100m -XX:+UseConcMarkSweepGC  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:+PrintHeapAtGC -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=50M -Xloggc:./logs/emps-gc-%t.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs/emps-heap.dump OOM
```

可以看到相同时间内，确实Minor GC减少了，但是时间增大了，因为复制算法，基本都是存活的，复制需要耗费大量的性能和时间：

![调优实战-减少MinorGC后VisualVM显示](F:/lemon-guide-main/images/JVM/调优实战-减少MinorGC后VisualVM显示.jpg)

所以，调优要有取舍，取得一个平衡点，性能、状态达到佳就OK了，并没最佳的状态，这就是调优的基本法则，而且调优也是一个细活，所谓慢工出细活，需要耗费大量的时间，慢慢调，不断的做对比。



## 调优参数

### 堆

- -Xms1024m 设置堆的初始大小
- -Xmx1024m 设置堆的最大大小
- -XX:NewSize=1024m 设置年轻代的初始大小
- -XX:MaxNewSize=1024m 设置年轻代的最大值
- -XX:SurvivorRatio=8 Eden和S区的比例
- -XX:NewRatio=4 设置老年代和新生代的比例
- -XX:MaxTenuringThreshold=10 设置晋升老年代的年龄条件



### 栈

- -Xss128k



### 元空间

- -XX:MetasapceSize=200m 设置初始元空间大小
- -XX:MaxMatespaceSize=200m 设置最大元空间大小 默认无限制



### 直接内存

- -XX:MaxDirectMemorySize 设置直接内存的容量，默认与堆最大值一样



### 日志

- -Xloggc:/opt/app/ard-user/ard-user-gc-%t.log   设置日志目录和日志名称
- -XX:+UseGCLogFileRotation           开启滚动生成日志
- -XX:NumberOfGCLogFiles=5            滚动GC日志文件数，默认0，不滚动
- -XX:GCLogFileSize=20M               GC文件滚动大小，需开 UseGCLogFileRotation
- -XX:+PrintGCDetails      开启记录GC日志详细信息（包括GC类型、各个操作使用的时间）,并且在程序运行结束打印出JVM的内存占用情况
- -XX:+ PrintGCDateStamps             记录系统的GC时间
- -XX:+PrintGCCause                   产生GC的原因(默认开启)



### GC

#### Serial垃圾收集器（新生代）

开启

- -XX:+UseSerialGC

关闭：

- -XX:-UseSerialGC //新生代使用Serial  老年代则使用SerialOld



#### Parallel Scavenge收集器（新生代）开启

- -XX:+UseParallelOldGC 关闭
- -XX:-UseParallelOldGC  新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器



#### ParallelOl垃圾收集器（老年代）开启

- -XX:+UseParallelGC 关闭
- -XX:-UseParallelGC 新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器



#### ParNew垃圾收集器（新生代）开启

- -XX:+UseParNewGC 关闭
- -XX:-UseParNewGC //新生代使用功能ParNew 老年代则使用功能CMS



#### CMS垃圾收集器（老年代）开启

- -XX:+UseConcMarkSweepGC 关闭
- -XX:-UseConcMarkSweepGC
- -XX:MaxGCPauseMillis  GC停顿时间，垃圾收集器会尝试用各种手段达到这个时间，比如减小年轻代
- -XX:+UseCMSCompactAtFullCollection 用于在CMS收集器不得不进行FullGC时开启内存碎片的合并整理过程，由于这个内存整理必须移动存活对象，（在Shenandoah和ZGC出现前）是无法并发的
- -XX：CMSFullGCsBefore-Compaction 多少次FullGC之后压缩一次，默认值为0，表示每次进入FullGC时都进行碎片整理）
- -XX:CMSInitiatingOccupancyFraction 当老年代使用达到该比例时会触发FullGC，默认是92
- -XX:+UseCMSInitiatingOccupancyOnly 这个参数搭配上面那个用，表示是不是要一直使用上面的比例触发FullGC，如果设置则只会在第一次FullGC的时候使用-XX:CMSInitiatingOccupancyFraction的值，之后会进行自动调整
- -XX:+CMSScavengeBeforeRemark 在FullGC前启动一次MinorGC，目的在于减少老年代对年轻代的引用，降低CMSGC的标记阶段时的开销，一般CMS的GC耗时80%都在标记阶段
- -XX:+CMSParallellnitialMarkEnabled 默认情况下初始标记是单线程的，这个参数可以让他多线程执行，可以减少STW
- -XX:+CMSParallelRemarkEnabled 使用多线程进行重新标记，目的也是为了减少STW



#### G1垃圾收集器开启

- -XX:+UseG1GC 关闭
- -XX:-UseG1GC
- -XX：G1HeapRegionSize 设置每个Region的大小，取值范围为1MB～32MB
- -XX：MaxGCPauseMillis 设置垃圾收集器的停顿时间，默认值是200毫秒，通常把期望停顿时间设置为一两百毫秒或者两三百毫秒会是比较合理的

# JVM之Class文件结构

代码编译的结果从本地机器码转变为字节码。

对于Java虚拟机来说，Class文件是虚拟机的一个重要接口。无论使用何种语言开发，只要能将源文件编译成正确的Class文件，那么这种语言就可以在Java虚拟机上运行。Class文件是Java虚拟机的基石。

## 概述

### 跨平台

我们所编写的每一行代码，要在机器上运行最终都需要编译成二进制的机器码 CPU 才能识别。但是由于虚拟机的存在，屏蔽了操作系统与 CPU 指令集的差异性，类似于 Java 这种建立在虚拟机之上的编程语言通常会编译成一种中间格式的文件Class文件来进行存储。Java 具有跨平台性，其实现基础就是虚拟机和字节码（ByteCode）存储格式。

### 语言无关性

Java 虚拟机的设计者在设计之初就考虑并实现了其它语言在 Java 虚拟机上运行的可能性。所以并不是只有 Java 语言能够跑在 Java 虚拟机上，时至今日诸如 Kotlin、Groovy、Jython、JRuby 等一大批 JVM 语言都能够在 Java 虚拟机上运行。它们和 Java 语言一样都会被编译器编译成字节码文件，然后由虚拟机来执行。所以说类文件（字节码文件）具有语言无关性。Java 虚拟机不与 Java 语言绑定，只与 Class 文件所关联。Java 虚拟机作为一个通用的、与机器无关的执行平台，任何语言都可以将 Java 虚拟机作为它们的运行基础，以 Class 文件作为它们产品的交付媒介。

## Class文件结构

Java文件经过编译后生产Class字节码文件。JVM时通过字节码来执行。熟悉Class文件的数据结构很重要。随着虚拟机的发展，不可避免会对Class文件进行调整，但是其基本结构和框架是非常稳定的。

Class文件是一组以8位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在Class文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在。当遇到需要占用8位字节以上空间的数据项时，则会按照高位在前的方式分割成若干个8位字节进行存储。

根据Java虚拟机规范的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：无符号数和表，后面的解析都要以这两种数据类型为基础，所以这里要先介绍这两个概念。

- 无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。
- 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，所有表都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上就是一张表，它由下面的数据项构成。

无论是无符号数还是表，当需要描述同一类型但数量不定的多个数据时，经常会使用一个前置的容量计数器加若干个连续的数据项的形式，这时称这一系列连续的某一类型的数据为某一类型的集合。

Class文件结构定义

```
ClassFile
 {
       u4             magic;
       u2             minor_version;
       u2             major_version;
       u2             constant_pool_count;
       cp_info        constant_pool[constant_pool_count-1];
       u2             access_flags;
       u2             this_class;
       u2             super_class;
       u2             interfaces_count;
       u2             interfaces[interfaces_count];
       u2             fields_count;
       field_info     fields[fields_count];
       u2             methods_count;
       method_info    methods[methods_count];
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}

```

![JVM-Class结构](png\JVM-Class结构.png)

**Class 文件格式**

| 类型           | 名称                | 数量                    |
| :------------- | ------------------- | ----------------------- |
| u4             | magic               | 1                       |
| u2             | minor_version       | 1                       |
| u2             | major_version       | 1                       |
| u2             | constant_pool_count | 1                       |
| cp_info        | constant_pool       | constant_pool_count - 1 |
| u2             | access_flags        | 1                       |
| u2             | this_class          | 1                       |
| u2             | super_class         | 1                       |
| u2             | interfaces_count    | 1                       |
| u2             | interfaces          | interfaces_count        |
| u2             | fields_count        | 1                       |
| field_info     | fields              | fields_count            |
| u2             | methods_count       | 1                       |
| method_info    | methods             | methods_count           |
| u2             | attributes_count    | 1                       |
| attribute_info | attributes          | attributes_count        |

Class由于它没有任何分隔符号，所以在表中的数据项，无论是顺序还是数量，甚至于数据存储的字节序（Byte Ordering,Class文件中字节序为Big-Endian）这样的细节，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，都不允许改变。接下来我们将一起看看这个表中各个数据项的具体含义。

### 魔数(MagicNumber)

每个Class文件的头4个字节称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件。

Class 文件的头 4 个字节被称为魔数（Magic Number），它的唯一作用是确定该 Class 文件是否能被虚拟机接受，其值为“**0xCAFEBABE**” （谐音咖啡宝贝）。

### **版本号**(MinorVersion&MajorVersion)

魔数后面的4个字节存储的是 Class 文件的版本号，分为两类：

- 次版本号（Minor Version）：第5、6个字节
- 主版本号（Major Version）：第7、8个字节

JDK1.0 主版本号为45， 1.1 为46， 依次类推，到JDK8的版本号为52， 16进制为0x33.

 一个 JVM实例只能支持特定范围内的主版本号 （Mi 至Mj） 和 0 至特定范围内 （0 至 m） 的副版 本号。假设一个 Class 文件的格式版本号为 V， 仅当Mi.0 ≤ v ≤ Mj.m成立时，这个 Class 文件 才可以被此 Java 虚拟机支持。不同版本的 Java 虚拟机实现支持的版本号也不同，高版本号的 Java 虚拟机实现可以支持低版本号的 Class 文件，反之则不成立。 JVM在加载class文件的时候，会读取出主版本号，然后比较这个class文件的主版本号和JVM本身的版 本号，如果JVM本身的版本号 < class文件的版本号，JVM会认为加载不了这个class文件，会抛出我 们经常见到的" java.lang.UnsupportedClassVersionError: Bad version number in .class file " Error 错误；反之，JVM会认为可以加载此class文件，继续加载此class文件。

### 常量池(**ConstantPool**)

**主版本号之后是常量池入口，常量池可以理解为 Class 文件之中的资源仓库**，它是 Class 文件结构中与其他项目关联最多的数据类型，也是占用 Class 文件空间最大的数据项目之一，同是它还是 Class 文件中第一个出现的表类型数据项目。

**因为常量池中常量的数量是不固定的，所以在常量池入口需要放置一个 u2 类型的数据来表示常量池的容量「constant_pool_count」**，和计算机科学中计数的方法不一样，这个`容量是从 1 开始而不是从 0 开始计数`。之所以将第 0 项常量空出来是为了满足后面某些指向常量池的索引值的数据在特定情况下需要表达「不引用任何一个常量池项目」的含义，这种情况可以把索引值置为 0 来表示。具体用来干什么，如果你仔细观察Object类的Class文件，你会发现Object这个顶级类的父类索引指向的是这个0的槽位。

> Class 文件结构中只有常量池的容量计数是从 1 开始的，其它集合类型，包括接口索引集合、字段表集合、方法表集合等容量计数都是从 0 开始。

![JVM-Class结构-常量池](png\JVM-Class结构-常量池.png)

常量池中主要存放两大类常量：**字面量（Literal）**和**符号引用（Symbolic References）**。

- **字面量**比较接近 Java 语言层面的常量概念，如字符串、声明为 final 的常量值等。
- 符号引用属于编译原理方面的概念，包括了以下三类常量：
  - 类和接口的全限定名（Fully Qualified Name）
  - 字段的名称和描述符（Descriptor）
  - 方法的名称和描述符

经过javac编译后的Class文件不会保存方法、字段最终在内存中的布局信息，而是保存其具体地址的符号引用。当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析翻译到具体的内存地址中。

**引用替换的时机**

符号引用替换为直接引用的操作发生在类加载过程(加载 -> 连接(验证、准备、解析) -> 初始化)中的解析阶段，会将符号引用转换(替换)为对应的直接引用，放入运行时常量池中。

常量池中每一项常量都是一个表，在JDK 1.7之前共有11种结构各不相同的表结构数据，在JDK 1.7中为了更好地支持动态语言调用，又额外增加了4种（CONSTANT_MethodHandle_info、CONSTANT_MethodType_info、CONSTANT_Dynamic_info和CONSTANT_InvokeDynamic_info）。后来为了支持 Java 模块化，又加入了 2 个常量，所以截止 JDK13，常量表中有 17 种不同类型的常量。这种表都有一个共同的特点，就是表开始的第一位是一个u1类型的标志位（tag，取值标志列），代表当前这个常量属于哪种常量类型。

截至 JDK 13，常量表中分别有17种不同数据类型的常量如下表。

| 常量池元素名称                   | tag标识 | 含义                               |
| -------------------------------- | ------- | ---------------------------------- |
| CONSTANT_Utf8_info               | 1       | UTF-8编码的字符串                  |
| CONSTANT_Integer_info            | 3       | 整型字面量                         |
| CONSTANT_Float_info              | 4       | 浮点型字面量                       |
| CONSTANT_Long_info               | 5       | 长整型字面量                       |
| CONSTANT_Double_info             | 6       | 双精度浮点型字面量                 |
| CONSTANT_Class_info              | 7       | 类或接口的符号引用                 |
| CONSTANT_String_info             | 8       | 字符串类型的字面量                 |
| CONSTANT_Fieldref_info           | 9       | 字段的符号引用                     |
| CONSTANT_Methodref_info          | 10      | 类中方法的符号引用                 |
| CONSTANT_InterfaceMethodref_info | 11      | 接口中方法的符号引用               |
| CONSTANT_NameAndType_info        | 12      | 字段和方法的名称以及类型的符号引用 |
| CONSTANT_MethodHandler_info      | 15      | 表示方法句柄                       |
| CONSTANT_MethodType_info         | 16      | 标识方法类型                       |
| CONSTANT_Dynamic_info            | 17      | 表示一个动态计算常量               |
| CONSTANT_InvokeDynamic_info      | 18      | 表示一个动态方法调用点             |
| CONSTANT_Module_info             | 19      | 表示一个模块                       |
| CONSTANT_Package_info            | 20      | 表示一个模块中开放或者导出的包     |

#### 常量项结构表

```
`CONSTANT_Class_info` {
    u1 tag;  // 值为1
    u2 name_index;  // 指向权限定名常量项的索引
}

`CONSTANT_Fieldref_info` {
    u1 tag;  // 值为9
    u2 class_index;  // 指向声明字段的类或接口描述符CONSTANT_Class_info的索引项
    u2 name_and_type_index;  // 指向字段描述符CONSTANT_NameAndType的索引项
}

`CONSTANT_Methodref_info` {
    u1 tag;  // 值为10
    u2 class_index;  // 指向声明方法的类描述符CONSTANT_Class_info的索引项
    u2 name_and_type_index;  // 指向名称及类型描述符CONSTANT_NameAndType的索引项
}

`CONSTANT_InterfaceMethodref_info` {
    u1 tag;  // 值为11
    u2 class_index;  // 指向声明方法的接口描述符CONSTANT_Class_info的索引项
    u2 name_and_type_index;  // 指向名称及类型描述符CONSTANT_NameAndType的索引项
}
`CONSTANT_String_info` {
    u1 tag;  // 值为8
    u2 string_index;  // 指向字符串字面量的索引
}
`CONSTANT_Integer_info` {
    u1 tag;  // 值为3
    u4 bytes;  // 按照高位在前存储的int值
}

`CONSTANT_Float_info` {
    u1 tag;  // 值为4
    u4 bytes;  // 按照高位在前存储的float值
}
`CONSTANT_Long_info` {
    u1 tag;  // 值为5
    u4 high_bytes;  // 按照高位在前存储的long值
    u4 low_bytes;  //
}

`CONSTANT_Double_info` {
    u1 tag;  // 值为6 
    u4 high_bytes;  // 按照高位在前存储的double值
    u4 low_bytes;  //
}
`CONSTANT_NameAndType_info` {
    u1 tag;  // 值为12
    u2 name_index;  // 指向该字段或方法名称常量项的索引
    u2 descriptor_index;  // 指向该字段或方法描述符常量项的索引
}
`CONSTANT_Utf8_info` {
    u1 tag;  // 值为1
    u2 length;  // UTF8编码的字符串占用的字节数
    u1 bytes[length];  // 长度为length的UTF8编码的字符串
}
`CONSTANT_MethodHandle_info` {
    u1 tag;  // 值为15
    u1 reference_kind;  // 值必须在[1~9]之间，它决定了方法句柄的类型。
    					//方法句柄类型的值表示方法句柄的字节码行为
    u2 reference_index;  // 值必须是对常量池的有效索引
}
`CONSTANT_MethodType_info` {
    u1 tag;  // 值为16
    u2 descriptor_index;  // 值必须是对常量池的有效索引，
    			//常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符
}
`CONSTANT_InvokeDynamic_info` {
    u1 tag;  // 值为18
    u2 bootstrap_method_attr_index;  // 值必须是对当前Class文件中引导方法表的
    								//bootstrap_methods[]数组的有效索引
    u2 name_and_type_index;  // 值必须是对当前常量池的有效索引，常量池在该索引处的项必须
					    	// 是CONSTANT_NameAndType_info结构，表示方法名和方法描述符
}
`CONSTANT_Module_info` {
    u1 tag;  //  
    u2 name_index;  //
}
`CONSTANT_Package_info` {
    u1 tag;  //
    u2 name_index;  //
}

```

**17种数据类型的常量结构**

常量池中的17种数据类型的结构如下：

1）CONSTANT_Utf8_info

UTF-8 编码的字符串。

| 结构   | 类型 | 描述                                |
| ------ | ---- | ----------------------------------- |
| tag    | u1   | 值为1                               |
| length | u2   | UTF-8 编码的字符串占用的字节数      |
| bytes  | u1   | 长度为 length 的 UTF-8 编码的字符串 |

2）CONSTANT_Integer_info

整型字面量。

| 结构  | 类型 | 描述                      |
| ----- | ---- | ------------------------- |
| tag   | u1   | 值为3                     |
| bytes | u4   | 按照高位在前存储的 int 值 |

3）CONSTANT_Float_info

浮点型字面量。

| 结构  | 类型 | 描述                        |
| ----- | ---- | --------------------------- |
| tag   | u1   | 值为4                       |
| bytes | u4   | 按照高位在前存储的 float 值 |

4）CONSTANT_Long_info

长整型字面量。

| 结构  | 类型 | 描述                       |
| ----- | ---- | -------------------------- |
| tag   | u1   | 值为5                      |
| bytes | u8   | 按照高位在前存储的 long 值 |

5）CONSTANT_Double_info

双精度浮点型字面量。

| 结构  | 类型 | 描述                         |
| ----- | ---- | ---------------------------- |
| tag   | u1   | 值为6                        |
| bytes | u8   | 按照高位在前存储的 double 值 |

6）CONSTANT_Class_info

类或接口的符号引用。

| 结构  | 类型 | 描述                     |
| ----- | ---- | ------------------------ |
| tag   | u1   | 值为7                    |
| index | u2   | 指向全限定名常量项的索引 |

7）CONSTANT_String_info

字符串类型字面量。

| 结构  | 类型 | 描述                   |
| ----- | ---- | ---------------------- |
| tag   | u1   | 值为8                  |
| index | u2   | 指向字符串字面量的索引 |

8）CONSTANT_Fieldref_info

字段的符号引用。

| 结构  | 类型 | 描述                                                        |
| ----- | ---- | ----------------------------------------------------------- |
| tag   | u1   | 值为9                                                       |
| index | u2   | 指向声明字段的类或者接口描述符 CONSTANT_Class_info 的索引项 |
| index | u2   | 指向字段描述符 CONSTANT_NameAndType 的索引项                |

9）CONSTANT_Methodref_info

类中方法的符号引用。

| 结构  | 类型 | 描述                                                |
| ----- | ---- | --------------------------------------------------- |
| tag   | u1   | 值为10                                              |
| index | u2   | 指向声明方法的类描述符 CONSTANT_Class_info 的索引项 |
| index | u2   | 指向名称及类型描述符 CONSTANT_NameAndType 的索引项  |

10）CONSTANT_InterfaceMethodref_info

  接口中方法的符号引用。

| 结构  | 类型 | 描述                                                  |
| ----- | ---- | ----------------------------------------------------- |
| tag   | u1   | 值为11                                                |
| index | u2   | 指向声明方法的接口描述符 CONSTANT_Class_info 的索引项 |
| index | u2   | 指向名称及类型描述符 CONSTANT_NameAndType 的索引项    |

11）CONSTANT_NameAndType_info

字段或方法的部分符号引用。

| 结构  | 类型 | 描述                               |
| ----- | ---- | ---------------------------------- |
| tag   | u1   | 值为12                             |
| index | u2   | 指向该字段或方法名称常量项的索引   |
| index | u2   | 指向该字段或方法描述符常量项的索引 |

12）CONSTANT_MethodHandle_info

表示方法句柄。

| 结构            | 类型 | 描述                                                         |
| --------------- | ---- | ------------------------------------------------------------ |
| tag             | u1   | 值为15                                                       |
| reference_kind  | u1   | 值必须在1至9之间（包括1和9），它决定方法句柄的类型。方法句柄类型的值表示方法句柄的字节码行为 |
| reference_index | u2   | 值必须是对常量池的有效索引                                   |

13）CONSTANT_MethodType_info

表示方法类型。

| 结构             | 类型 | 描述                                                         |
| ---------------- | ---- | ------------------------------------------------------------ |
| tag              | u1   | 值为16                                                       |
| descriptor_index | u2   | 值必须是对常量池的有效索引，常量池在该索引处的项必须是 CONSTANT_Utf8_info 结构，表示方法的描述符 |

14）CONSTANT_Dynamic_info

表示一个动态计算常量。

| 结构                        | 类型 | 描述                                                         |
| --------------------------- | ---- | ------------------------------------------------------------ |
| tag                         | u1   | 值为17                                                       |
| bootstrap_method_attr_index | u2   | 值必须是对当前 Class 文件中引导方法表的 bootstrap_methods[] 数组的有效索引 |
| name_and_type_index         | u2   | 值必须是对当前常量池的有效索引，常量池在该索引处的项必须是 CONSTANT_NameAdnType_info 结构，表示方法名和方法描述符 |

15）CONSTANT_InvokeDynamic_info

表示一个动态方法调用点。

| 结构                        | 类型 | 描述                                                         |
| --------------------------- | ---- | ------------------------------------------------------------ |
| tag                         | u1   | 值为18                                                       |
| bootstrap_method_attr_index | u2   | 值必须是对当前 Class 文件中引导方法表的 bootstrap_methods[] 数组的有效索引 |
| name_and_type_index         | u2   | 值必须是对当前常量池的有效索引，常量池在该索引处的项必须是 CONSTANT_NameAdnType_info 结构，表示方法名和方法描述符 |

16）CONSTANT_Module_info

表示一个模块。

| 结构       | 类型 | 描述                                                         |
| ---------- | ---- | ------------------------------------------------------------ |
| tag        | u1   | 值为19                                                       |
| name_index | u2   | 值必须是对当前常量池的有效索引，常量池在该索引处的项必须是 CONSTANT_Utf8_info 结构，表示模块名字 |

17）CONSTANT_Package_info

表示一个模块中开放或者导出的包。

| 结构       | 类型 | 描述                                                         |
| ---------- | ---- | ------------------------------------------------------------ |
| tag        | u1   | 值为20                                                       |
| name_index | u2   | 值必须是对当前常量池的有效索引，常量池在该索引处的项必须是 CONSTANT_Utf8_info 结构，表示包名称 |

### Class文件结构实战

#### Test类源码解读Class文件结构

```
public class Test {

    private int m;
	
	public int inc(){
		return m + 1;
	}
}
```

#### 查看二进制信息

对Test.java使用javac编译后，使用vim查看Test.class文件，此时显示文件为乱码信息，输入`:%!xxd` 即可显示二进制信息

```
#           魔数  次版本号 主版本号        1    2
0000000: cafe babe 0000 0034 0013 0a00 0400 0f09  .......4........
#                  3       4        5 
0000010: 0003 0010 0700 1107 0012 0100 016d 0100  .............m..
0000020: 0149 0100 063c 696e 6974 3e01 0003 2829  .I...<init>...()
0000030: 5601 0004 436f 6465 0100 0f4c 696e 654e  V...Code...LineN
0000040: 756d 6265 7254 6162 6c65 0100 0369 6e63  umberTable...inc
0000050: 0100 0328 2949 0100 0a53 6f75 7263 6546  ...()I...SourceF
0000060: 696c 6501 000e 5465 7374 436c 6173 732e  ile...TestClass.
0000070: 6a61 7661 0c00 0700 080c 0005 0006 0100  java............
#                                 18 length    后面的字符到0x21之前表示java/lang/Object
0000080: 0954 6573 7443 6c61 7373 0100 106a 6176  .TestClass...jav
#                                          访问标志
0000090: 612f 6c61 6e67 2f4f 626a 6563 7400 2100  a/lang/Object.!.
00000a0: 0300 0400 0000 0100 0200 0500 0600 0000  ................
00000b0: 0200 0100 0700 0800 0100 0900 0000 1d00  ................
00000c0: 0100 0100 0000 052a b700 01b1 0000 0001  .......*........
00000d0: 000a 0000 0006 0001 0000 0001 0001 000b  ................
00000e0: 000c 0001 0009 0000 001f 0002 0001 0000  ................
00000f0: 0007 2ab4 0002 0460 ac00 0000 0100 0a00  ..*....`........
0000100: 0000 0600 0100 0000 0600 0100 0d00 0000  ................
0000110: 0200 0e0a  

```

根据以上信息可以获得主版本号信息 0x0034 的十进制52，52对应的JDK版本是JDK8，可以向下兼容45-51的JDK版本。JDK版本是从45开始的，JDK1.0-1.1 使用了45.0-45.3的版本号。

次版本号在JDK12之前都没有使用过，全为0。

常量池容量是十六进制0x0013，十进制为19，因此常量池中有18项常量，索引范围在1-18。

常量池容量大小0x0013之后有18个常量，每个常量所占占用的字节大小都不相同，以第一个常量 0x0a(十进制为10)为例，**查常量池项目类型表可知（常量类型表的结构一般为 tag（u1） 和 index（u2））**，**10为类中方法的符号引用CONSTANT_Methodref_info**，该引用 有一个tag （u1类型占用一个字节 = 0x0a = 10）、两个index（u2类型占用两个字节 0x0004和0x000f）, 第一个`index = 4` 指向声明方法描述符CONSTANT_Class_info的索引项，第二个`index = 15`指向名称及类型描述符CONSTANT_NameAndType的索引项。根据第一个**index索引值0x0004 代表第四个常量**，第四个常量的 tag 是0x07为CONSTANT_Class_info，结构为tag和index（指向全限定名常量项的索引），则**tag = 7(0x07)，index = 18（0x0012）**索引为18表示为最后一个常量，查常量池知，**第18个常量为 0x01是CONSTANT_Utf8_info常量，其长度为0x0010 = 16个字节** ，由上面的`Constant pool`知，**这16个字节表示 java/lang/Object**。依次类推即可查询所有的常量信息

#### **查看字节码信息**javap

javap是jdk自带的反解析工具。它的作用就是根据class字节码文件，反解析出当前类对应的code区 （汇编指令）、本地变量表、异常表和代码行偏移量映射表、常量池等等信息。

javap的用法格式：javap <option> <class>

options如下：

```
-help --help -? 输出此用法消息
-version 版本信息，其实是当前javap所在jdk的版本信息，不是class在哪个jdk下生成的。
-v -verbose 输出附加信息（包括行号、本地变量表，反汇编等详细信息）
-l 输出行号和本地变量表
-public 仅显示公共类和成员
-protected 显示受保护的/公共类和成员
-package 显示程序包/受保护的/公共类 和成员 (默认)
-p -private 显示所有类和成员
-c 对代码进行反汇编
-s 输出内部类型签名
-sysinfo 显示正在处理的类的系统信息 (路径， 大小， 日期， MD5 散
列)
-constants 显示静态最终常量
-classpath <path> 指定查找用户类文件的位置
-bootclasspath <path> 覆盖引导类文件的位置
```

一般常用的是 -v -l -c三个选项。

javap -v classxx，不仅会输出行号、本地变量表信息、反编译汇编代码，还会输出当前类用 到的常量池等信息。

javap -l 会输出行号和本地变量表信息。

javap -c 会对当前class字节码进行反编译生成汇编代码。

使用javap -verbose工具可以参看当前Class文件的字节码信息

```
Classfile /F:/Test.class
  Last modified 2020-8-24; size 265 bytes
  MD5 checksum 0d5efc4b65ae7eb6d64f84136ce58ff9
  Compiled from "Test.java"
public class Test
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#16         // Test.m:I
   #3 = Class              #17            // Test
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               inc
  #12 = Utf8               ()I
  #13 = Utf8               SourceFile
  #14 = Utf8               Test.java
  #15 = NameAndType        #7:#8          // "<init>":()V
  #16 = NameAndType        #5:#6          // m:I
  #17 = Utf8               Test
  #18 = Utf8               java/lang/Object
{
  public Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 6: 0
}
SourceFile: "Test.java"
```

### 常量池实战

#### **int 和 float在class 文件的存储结构**

int类型和float 类型的数据类型占用 4 个字节的空间。

```
public class Constant {
    private final int a = 10;
    private final int b = 10;
    private float c = 11f;
    private float d = 11f;
    private float e = 11f;    
}
```

用javap -v Constant 或字节码工具查看，如下图。常量池里有只有一个float和int常量

```
Classfile /F:/Constant.class
  Last modified 2022-4-9; size 400 bytes
  MD5 checksum 508b4808a5a820bd54923cb8a4d0796e
  Compiled from "Constant.java"
public class Constant
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#25         // java/lang/Object."<init>":()V
   #2 = Fieldref           #8.#26         // Constant.a:I
   #3 = Fieldref           #8.#27         // Constant.b:I
   #4 = Float              11.0f
   #5 = Fieldref           #8.#28         // Constant.c:F
   #6 = Fieldref           #8.#29         // Constant.d:F
   #7 = Fieldref           #8.#30         // Constant.e:F
   #8 = Class              #31            // Constant
   #9 = Class              #32            // java/lang/Object
  #10 = Utf8               a
  #11 = Utf8               I
  #12 = Utf8               ConstantValue
  #13 = Integer            10
  #14 = Utf8               b
  #15 = Utf8               c
  #16 = Utf8               F
  #17 = Utf8               d
  #18 = Utf8               e
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               SourceFile
  #24 = Utf8               Constant.java
  #25 = NameAndType        #19:#20        // "<init>":()V
  #26 = NameAndType        #10:#11        // a:I
  #27 = NameAndType        #14:#11        // b:I
  #28 = NameAndType        #15:#16        // c:F
  #29 = NameAndType        #17:#16        // d:F
  #30 = NameAndType        #18:#16        // e:F
  #31 = Utf8               Constant
  #32 = Utf8               java/lang/Object
{
  public Constant();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: bipush        10
         7: putfield      #2                  // Field a:I
        10: aload_0
        11: bipush        10
        13: putfield      #3                  // Field b:I
        16: aload_0
        17: ldc           #4                  // float 11.0f
        19: putfield      #5                  // Field c:F
        22: aload_0
        23: ldc           #4                  // float 11.0f
        25: putfield      #6                  // Field d:F
        28: aload_0
        29: ldc           #4                  // float 11.0f
        31: putfield      #7                  // Field e:F
        34: return
      LineNumberTable:
        line 1: 0
        line 2: 4
        line 3: 10
        line 4: 16
        line 5: 22
        line 6: 28
}
SourceFile: "Constant.java"
```

代码中所有用到 int 类型 10 的地方，会使用指向常量池的指针值#13个常量池项 (cp_info)，即值为 10的结构体CONSTANT_Integer_info，而用到float类型的11f时，也会指向常量池的指针值#4来定位到第#4个常量池项(cp_info) 即值为11f的结构体CONSTANT_Float_info。

#### **long和 double数据类型的常量在常量池中是怎样表示和存储 的？**

Java语言规范规定了 long 类型和 double类型的数据类型占用8 个字节的空间。那么存在于class 字节码文件中的该类型的常量是如何存储的呢？

```
public class Constant {
    private long k = -6076574518398440533L;
    private long m = -6076574518398440533L;
    private long n = -6076574518398440533L;
    private double o = 10.1234567890D;
    private double p = 10.1234567890D;
    private double q = 10.1234567890D;
}
```

用字节码工具查看，发现#2，#3表示一个Long类型的常量， #7，#8表示一个Double类型的常量。

```
Classfile /F:/Constant.class
  Last modified 2022-4-9; size 414 bytes
  MD5 checksum bbb6a887cae025ee13f6e15e56a0234b
  Compiled from "Constant.java"
public class Constant
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #13.#28        // java/lang/Object."<init>":()V
   #2 = Long               -6076574518398440533l
   #4 = Fieldref           #12.#29        // Constant.k:J
   #5 = Fieldref           #12.#30        // Constant.m:J
   #6 = Fieldref           #12.#31        // Constant.n:J
   #7 = Double             10.123456789d
   #9 = Fieldref           #12.#32        // Constant.o:D
  #10 = Fieldref           #12.#33        // Constant.p:D
  #11 = Fieldref           #12.#34        // Constant.q:D
  #12 = Class              #35            // Constant
  #13 = Class              #36            // java/lang/Object
  #14 = Utf8               k
  #15 = Utf8               J
  #16 = Utf8               m
  #17 = Utf8               n
  #18 = Utf8               o
  #19 = Utf8               D
  #20 = Utf8               p
  #21 = Utf8               q
  #22 = Utf8               <init>
  #23 = Utf8               ()V
  #24 = Utf8               Code
  #25 = Utf8               LineNumberTable
  #26 = Utf8               SourceFile
  #27 = Utf8               Constant.java
  #28 = NameAndType        #22:#23        // "<init>":()V
  #29 = NameAndType        #14:#15        // k:J
  #30 = NameAndType        #16:#15        // m:J
  #31 = NameAndType        #17:#15        // n:J
  #32 = NameAndType        #18:#19        // o:D
  #33 = NameAndType        #20:#19        // p:D
  #34 = NameAndType        #21:#19        // q:D
  #35 = Utf8               Constant
  #36 = Utf8               java/lang/Object
{
  public Constant();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc2_w        #2                  // long -6076574518398440533l
         8: putfield      #4                  // Field k:J
        11: aload_0
        12: ldc2_w        #2                  // long -6076574518398440533l
        15: putfield      #5                  // Field m:J
        18: aload_0
        19: ldc2_w        #2                  // long -6076574518398440533l
        22: putfield      #6                  // Field n:J
        25: aload_0
        26: ldc2_w        #7                  // double 10.123456789d
        29: putfield      #9                  // Field o:D
        32: aload_0
        33: ldc2_w        #7                  // double 10.123456789d
        36: putfield      #10                 // Field p:D
        39: aload_0
        40: ldc2_w        #7                  // double 10.123456789d
        43: putfield      #11                 // Field q:D
        46: return
      LineNumberTable:
        line 1: 0
        line 2: 4
        line 3: 11
        line 4: 18
        line 5: 25
        line 6: 32
        line 7: 39
}
SourceFile: "Constant.java"
```

#### **String类型的字符串常量在常量池中是怎样表示和存储的？**

对于字符串而言，JVM会将字符串类型的字面量以UTF-8 编码格式存储到在class字节码文件中。这么 说可能有点摸不着北，我们先从直观的Java源码中中出现的用双引号"" 括起来的字符串来看，在编译器编译的时候，都会将这些字符串转换成CONSTANT_String_info结构体，然后放置于常量池中。

源码如下

```
public class Constant {
  private String s="Java JVM";
}
```

CONSTANT_String_info 指向了地址#15，而#15是CONSTANT_Utf8_info存储的真正的字符串常量

```
Classfile /F:/Constant.class
  Last modified 2022-4-9; size 257 bytes
  MD5 checksum ea6aee7f91756d274022da413e1991d4
  Compiled from "Constant.java"
public class Constant
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #5.#14         // java/lang/Object."<init>":()V
   #2 = String             #15            // Java JVM
   #3 = Fieldref           #4.#16         // Constant.s:Ljava/lang/String;
   #4 = Class              #17            // Constant
   #5 = Class              #18            // java/lang/Object
   #6 = Utf8               s
   #7 = Utf8               Ljava/lang/String;
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               SourceFile
  #13 = Utf8               Constant.java
  #14 = NameAndType        #8:#9          // "<init>":()V
  #15 = Utf8               Java JVM
  #16 = NameAndType        #6:#7          // s:Ljava/lang/String;
  #17 = Utf8               Constant
  #18 = Utf8               java/lang/Object
{
  public Constant();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String Java JVM
         7: putfield      #3                  // Field s:Ljava/lang/String;
        10: return
      LineNumberTable:
        line 1: 0
        line 2: 4
}
SourceFile: "Constant.java"
```

#### **类文件中定义的类名和类中使用到的类在常量池中是怎样被组织和存储的？**

JVM会将某个Java 类中所有使用到了的类的完全限定名以二进制形式的完全限定名封装成 CONSTANT_Class_info结构体中，然后将其放置到常量池里。CONSTANT_Class_info 的tag值为 7。

Note ：类的完全限定名和二进制形式的完全限定名在某个Java源码中，我们会使用很多个类，比如我们定义了一个 TestClass的类，并把它放到 test包下，则 TestClass类的完全限定名为test.ClassTest，将JVM编译 器将类编译成class文件后，此完全限定名在class文件中，是以二进制形式的完全限定名存储的，即它会把完全限定符的"."换成"/" ，即在class文件中存储的 TestClass类的完全限定名称是"test/ClassTest"。因为这种形式的完全限定名是放在了class二进制形式的字节码文件中，所以就称之为二进制形式的完全限定名。

```
package Test;

import java.util.Date;
public class TestClass {
    private Date date = new Date();
}
```

javap -v Test.TestClass， 可以发现常量池里有3个CONSTANT_Class_info结构体，一个是Test/TestClass, 一个是java/lang/Object，还有一个是java/util/Date. 他们分别指向了地址#21，#22和#19的CONSTANT_Utf8_info字符串。

为什么有3个类呢？首先Test/TestClass是当前类，在常量池出现毋庸置疑。JVM规定所有类都是Object的子类，所以JVM在编译后都会把java/lang/Object加上。至于java/util/Date是因为程序里引进了此类，并且使用此类创建了对象，所以会出现在常量池。

注意点: 对于某个类而言，其class文件中至少要有两个CONSTANT_Class_info常量池项，用来表示自己的类 信息和其父类信息。(除了java.lang.Object类除外，其他的任何类都会默认继承自 java.lang.Object）如果类声明实现了某些接口，那么接口的信息也会生成对应的 CONSTANT_Class_info常量池项。除此之外，如果在类中使用到了其他的类，只有真正使用到了相应的类，JDK编译器才会将类的信息组 成CONSTANT_Class_info常量池项放置到常量池中。

如果把代码修改为

```
import java.util.Date;
public class TestClass {
    private Date date;
}
```

javap 后java/util/Date类就不见了。因为Date类只是申明了变量，没有真正实例化和使用，将类信息放置到常量池中的目的，是为了在后续的代码中有可能会反复用到它。很显然，JDK在编译TestClass类的时候，解析到Date类有没有用到，发现该类在代码中就没有用到过，所以就认为没有必要将它的信息放置到常量 池中了。

**总结：**

**1. 对于某个类或接口而言，其自身、父类和继承或实现的接口的信息会被直接组装成 CONSTANT_Class_info常量池项放置到常量池中；**

**2. 类中或接口中使用到了其他的类，只有在类中实际使用到了该类时，该类的信息才会在常量池中有 对应的CONSTANT_Class_info常量池项；**

**3. 类中或接口中仅仅定义某种类型的变量，JDK只会将变量的类型描述信息以UTF-8字符串组成 CONSTANT_Utf8_info常量池项放置到常量池中，上面在类中的private Date date;JDK编译器 只会将表示date的数据类型的“Ljava/util/Date”字符串放置到常量池中。**

#### **哪些字面量会进入常量池中？**

1. final类型的8种基本类型的值会进入常量池。

2. 非final类型（包括static的）的8种基本类型的值，只有double、float、long的值会进入常量池。

3. 常量池中包含的字符串类型字面量（双引号引起来的字符串值）。

```
public class TestConstant {
    private int int_num = 110;
    private char char_num = 'a';
    private short short_num = 120;
    private float float_num = 130.0f;
    private double double_num = 140.0;
    private byte byte_num = 111;
    private long long_num = 3333L;
    private long long_delay_num;
    private boolean boolean_flage = true;
    public void init() {
    this.long_delay_num = 5555L;
}
```

### 访问标志(AccessFlags)

在常量池结束之后，紧接着的两个字节代表访问标志（access_flags），这个标志用于识别一些类或者接口层次的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话，是否被声明为final等。具体的标志位以及标志的含义如下。 

| 标志名称       | 标志值 | 含义                                                 |
| -------------- | ------ | ---------------------------------------------------- |
| ACC_PUBLIC     | 0x0001 | 是否为public类型                                     |
| ACC_FINAL      | 0x0010 | 是否为final，只有类可以设置                          |
| ACC_SUPER      | 0x0020 | 是否允许使用invokespecial字节码指令，1.2版本以后为真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                     |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口和抽象类为真             |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类型并非由用户代码产生                       |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                     |
| ACC_ENUM       | 0x4000 | 标识这是一个枚举                                     |

access_flags中一共有16个标志位可以使用，当前只定义了其中8个，没有使用到的标志位要求一律为0。

#### 实战

以Test.Class为例，他是一个普通Java类，不是接口、枚举、注解等，被public修饰但没有被声明为final或abstract，使用了JDK1.2之后的编译器进行编译 ，因此他的ACC_SUPER和ACC_PUBLIC为真，其余为假。因此`access_flag的值为：0x0001|0x0020 = 0x0021`

```
#                                          访问标志
0000090: 612f 6c61 6e67 2f4f 626a 6563 74    0021    00
```

### 类索引(ThisClass)、父类索引(SuperClass)与接口索引集合(Interfaces)

类索引（this_class）、父类索引（super_class）和接口索引（interfaces）都按顺序排列在访问标志之后，类索引和父类索引用两个 u2 类型的索引值表示，而接口索引是一组 u2 类型的数据的集合。

类索引用于确定该类的全限定名，父类索引确定该类的父类的全限定名，由于 Java 不允许多继承，因此父类索引只有一个，Object 类的父类索引为 0。接口索引集合就用来描述这个类实现了哪些接口，这些被实现的接口将按implements语句（如果这个类本身是一个接口，则应当是extends语句）后的接口顺序从左到右排列在接口索引集合中。类索引和父类索引各自指向一个 CONSTANT_Class_info 的类描述符常量，通过这个索引值可以找到定义在 CONSTANT_Utf8_info 类型的常量中的全限定名字符串。

对于接口索引集合，入口的第一项 u2 类型的数据为接口计数器（interfaces_count），表示索引表的容量，如果该类没有实现任何接口，则该计数器的值为 0，后面接口的索引表不再占用任何字节。

#### 实战

**类索引（this_class）和父类索引（super_class）都是一个 u2 类型的数据，而接口索引集合（interfaces）是一组 u2 类型的数据集合，Class 文件中由这三项数据来确定这个类的继承关系**。

- 类索引用于确定这个类的全限定名
- 父类索引用于确定这个类的父类的全限定名
- 接口索引集合用于描述这个类实现了哪些接口

类索引、父类索引、接口索引都排在访问标志之后。由于所有的类都是java.lang.Object类的子类，因此除了Object类之外所有类的父类索引都不为0。

类索引和父类索引各自指向CONSTANT_Class_info的类描述常量，通过CONSTANT_Class_info的类型常量中的索引可以找到

CONSTANT_Utf8_info类型的常量中的全限定名字符串。从而获取到该类的全限定名

```shell
0000090: 612f 6c61 6e67 2f4f 626a 6563 7400 2100  a/lang/Object.!.    
#        this_class
00000a0:   0300     0400 0000 0100 0200 0500 0600 0000  ................                            
```

根据上述字节码文件：0x0021之后就是**类索引（this_class）0x0003**，即常量池索引为3的第三个常量0x07（CONSTANT_Class_info类型常量），后面的0x0011指向全限定名常量项的索引，即第17个常量，该常量一定是一个CONSTANT_Utf8_info类型的常量，该常量除了tag索引值外，后面的u2为length表示UTF-8编码的字符串长度（以字节为单位），length后面的16进制字节码就是相应的字符串。

```shell
flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #3 = Class              #17            // Test
  #17 = Utf8               Test
```

**类索引之后为父类索引（super_class） = 0x0004，十进制表示第4个常量**

```shell
flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #4 = Class              #18            // java/lang/Object
  #18 = Utf8               java/lang/Object
```

**父类索引之后为接口索引（interfaces） = 0x0000**，因为没有实现任何接口，因此为全0.

### 字段表集合(FieldInfo)

字段表集合（field_info）用于描述接口或类中声明的变量，包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。字段包含待信息有字段的作用域（public、private、protected）、是实例变量还是类变量（static）、可变性（final）等等。这些信息要么有，要么没有，很适合用标志位来表示，而字段叫什么，被定义为什么数据类型，这些都无法固定，只能用常量池中的常量来描述。

字段表fields结构组成格式如下。 

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attrubute_info | attributes       | attributes_count |

- access_flags:标识变量的访问表示，该值可选由JVM规范规定。在实际情况中，ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED三个标志最多只能选择其一，ACC_FINAL、ACC_VOLATILE不能同时选择。接口之中的字段必须有ACC_PUBLIC、ACC_STATIC、ACC_FINAL标志，这些都是由Java本身的语言规则所决定的。
- name_index：变量的简单名称引用，指向常量池索引。简单名称是指没有类型和参数修饰的方法或者字段名称
- descriptor_index：变量的类型信息，引用字段和方法的描述符，指向常量池索引。描述符的作用是用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值。根据描述符规则，基本数据类型（byte、char、double、float、int、long、short、boolean）以及代表无返回值的void类型都用一个大写字符来表示，而对象类型则用字符L加对象的全限定名来表示。

字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型，access_flags可选项如下： 

| 标志名称      | 标志值 | 含义                 |
| ------------- | ------ | -------------------- |
| ACC_PUBLIC    | 0x0001 | 是否为public类型     |
| ACC_PRIVATE   | 0x0002 | 是否为private类型    |
| ACC_PROTECTED | 0x0004 | 是否为protected类型  |
| ACC_STATIC    | 0x0008 | 是否为static类型     |
| ACC_FINAL     | 0x0010 | 是否为final          |
| ACC_VOLATILE  | 0x0040 | 是否为volatile       |
| ACC_TRANSIENT | 0x0080 | 是否为transient      |
| ACC_SYNTHETIC | 0x1000 | 是否为编译器自动产生 |
| ACC_ENUM      | 0x4000 | 是否为enum           |

方法和字段的描述符

| 标识符 | 含义                        |
| ------ | --------------------------- |
| B      | 基本类型byte                |
| C      | 基本类型char                |
| D      | 基本类型double              |
| F      | 基本类型float               |
| I      | 基本类型int                 |
| J      | 基本类型long                |
| S      | 基本类型short               |
| Z      | 基本类型boolean             |
| V      | 特殊类型void                |
| L      | 对象类型如Ljava/lang/Object |

对于数组类型，每一维度将使用一个前置的“[”字符来描述，如一个定义为“java.lang.String[][]”类型的二维数组，将被记录为：“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录为“[I”。

用描述符来描述方法时，按照先参数列表，后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“（）”之内。如方法void inc（）的描述符为“（）V”，方法java.lang.String toString（）的描述符为“（）Ljava/lang/String；”，方法int indexOf（char[]source,int sourceOffset,int sourceCount,char[]target,int targetOffset,int targetCount,int fromIndex）的描述符为“（[CII[CIII）I”。

#### 实战

Test.java类中声明的变量有 `private int m；`查看下列字节码标注和字段修饰符表知`access_flag = 0x0002= ACC_PRIVATE`

```shell
#                     fields_count  access_flag   name_index  descriptor_index   attributes_count
00000a0: 03 0004 0000     0001        0002          0005           0006             0000   
```

name_index = 0x0005 = 5 ，查看常量池表第五个常量是CONSTANT_Utf8_info，其值为 m。而 name_index = 0x0006=6，也是CONSTANT_Utf8_info常量，值为 I

**常量池信息：**

```shell
 #5 = Utf8               m
 #6 = Utf8               I
```

`attributes_count`表示属性表计数器为0，也就是没有需要额外描述的信息。但是如果将 字段 m 的声明改为 `private int m = 123`，则会存在一项名称为ConstantVaule的属性，其值指向常量123。

### 方法表集合(MethodInfo)

Class文件存储格式中对方法的描述与对字段的描述几乎采用了完全一致的方式，方法表的结构如同字段表一样，依次包括了访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项。这些数据项目的含义也非常类似，仅在访问标志和属性表集合的可选项中有所区别。

 methods结构组成格式

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attrubute_info | attributes       | attributes_count |

因为volatile关键字和transient关键字不能修饰方法，所以方法表的访问标志中没有了ACC_VOLATILE标志和ACC_TRANSIENT标志。与之相对的，synchronized、native、strictfp和abstract关键字可以修饰方法，所以方法表的访问标志中增加了ACC_SYNCHRONIZED、ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志。对于方法表，所有标志位及其取值如下。

access_flags可选项

| 标志名称         | 标志值 | 含义                       |
| ---------------- | ------ | -------------------------- |
| ACC_PUBLIC       | 0x0001 | 是否为public类型           |
| ACC_PRIVATE      | 0x0002 | 是否为private类型          |
| ACC_PROTECTED    | 0x0004 | 是否为protected类型        |
| ACC_STATIC       | 0x0008 | 是否为static类型           |
| ACC_FINAL        | 0x0010 | 是否为final                |
| ACC_SYNCHRONIZED | 0x0020 | 是否为SYNCHRONIZED         |
| ACC_BRIDGE       | 0x0040 | 是否为编译器产生的桥接方法 |
| ACC_VARARGS      | 0x0080 | 是否接收不定参数           |
| ACC_NATIVE       | 0x0100 | 是否为native               |
| ACC_ABSTRACT     | 0x0400 | 是否为abstract             |
| ACC_STRICTFP     | 0x0800 | 是否为strictfp             |
| ACC_SYNTHETIC    | 0x1000 | 是否为编译器自动产生       |

 对于方法里的代码，经过编译器编译成字节码指令后，存放在方法属性表中一个名为「Code」的属性里面。

#### 实战

根据`methods_count = 2`知Test.java类中有两个方法，除了inc（）方法外，还有一个编译器添加的实例构造方法<init>。

第一个方法的访问标志值为access_flag = 0x0001即ACC_PUBLIC方法，name_index = 0x0007 查常量池知名为<init>()的方法，描述索引值descriptor_index = 0x0008，查字节码常量池知 代表“（）V”的常量。属性表计数器attributes_count = 0x0001表名此方法的属性表集合有1项属性，属性名称的索引值为0x0009，对应的常量为“Code”，说明此属性是方法的字节码描述。

```shell
#     methods_count  access_flag  name_index   descriptor_index  attributes_count
00000b0: 0002         0001            0007         0008              0001     
#    attributes_name_index
         0009                00 00 00 1d 00 
```

**常量池信息：**

```bash
Constant pool:
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
```

如果父类方法没有被子类重写，方法表集合中就不会出现来自父类的方法信息。否则编译器会自动添加方法，最常见的添加的方法便是类构造器<clinit>（）和实例构造器<init>（）。

### 属性表集合(AttributeInfo)

在 Class 文件、字段表、方法表中都可以携带自己的属性表（attribute_info）集合，用于描述某些场景专有的信息。

属性表集合不像 Class 文件中的其它数据项要求这么严格，不强制要求各属性表的顺序，并且只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java 虚拟机在运行时会略掉它不认识的属性。

虚拟机中预定义的属性:

| 属性名称                            | 使用位置           | 含义                                                         |
| ----------------------------------- | ------------------ | ------------------------------------------------------------ |
| Code                                | 方法表             | Java代码编译成的字节码指令                                   |
| ConstantValue                       | 字段表             | final关键字定义的常量池                                      |
| Deprecated                          | 类，方法，字段表   | 被声明为deprecated的方法和字段                               |
| Exceptions                          | 方法表             | 方法抛出的异常                                               |
| EnclosingMethod                     | 类文件             | 仅当一个类为局部类或者匿名类是才能拥有这个属性，这个属性用于标识这个类所在的外围方法 |
| InnerClass                          | 类文件             | 内部类列表                                                   |
| LineNumberTable                     | Code属性           | Java源码的行号与字节码指令的对应关系                         |
| LocalVariableTable                  | Code属性           | 方法的局部变量描述                                           |
| StackMapTable                       | Code属性           | JDK1.6中新增的属性，供新的类型检查检验器检查和处理目标方法的局部变量和操作数有所需要的类是否匹配 |
| Signature                           | 类，方法表，字段表 | 用于支持泛型情况下的方法签名                                 |
| SourceFile                          | 类文件             | 记录源文件名称                                               |
| SourceDebugExtension                | 类文件             | 用于存储额外的调试信息                                       |
| Synthetic                           | 类，方法表，字段表 | 标志方法或字段为编译器自动生成的                             |
| LocalVariableTypeTable              | 类                 | 使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加 |
| RuntimeVisibleAnnotations           | 类，方法表，字段表 | 为动态注解提供支持                                           |
| RuntimeInvisibleAnnotations         | 表，方法表，字段表 | 用于指明哪些注解是运行时不可见的                             |
| RuntimeVisibleParameterAnnotation   | 方法表             | 作用与RuntimeVisibleAnnotations属性类似，只不过作用对象为方法 |
| RuntimeInvisibleParameterAnnotation | 方法表             | 作用与RuntimeInvisibleAnnotations属性类似，作用对象哪个为方法参数 |
| AnnotationDefault                   | 方法表             | 用于记录注解类元素的默认值                                   |
| BootstrapMethods                    | 类文件             | 用于保存invokeddynamic指令引用的引导方式限定符               |

 对于每个属性，它的名称需要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。一个符合规则的属性表应该满足表6-14中所定义的结构。 

对于每个属性，它的名称需要从常量池中引用一个CONSTANT_utf8_info类型的常量类表示，而属性值的结构则是完全自定义的，只需要通过一个u4的长度属性区说明属性值所占用的位数即可。

属性表定义的结构：

| 类型 | 名称                 | 数量             |
| ---- | -------------------- | ---------------- |
| u2   | attribute_name_index | 1                |
| u2   | attribute_length     | 1                |
| u1   | info                 | attribute_length |

#### Code属性

Java程序方法体中的代码经过Javac编译器处理后，最终变为字节码指令存储在Code属性内。Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，譬如接口或者抽象类中的方法就不存在Code属性，如果方法表有Code属性存在，那么它的结构如下。

Code属性

| 类型           | 名称                   | 数量                   |
| -------------- | ---------------------- | ---------------------- |
| u2             | attribute_name_index   | 1                      |
| u4             | attribute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_bale         | exception_table_length |
| u2             | attributes_count       | 1                      |
| attribute_info | attributes             | attributes_count       |

- attribute_name_index：是一项指向CONSTANT_Utf8_info型常量的索引，常量值固定为“Code”，它代表了该属性的属性名称
- attribute_length：指示了属性值的长度，由于属性名称索引与属性长度一共为6字节，所以属性值的长度固定为整个属性表长度减去6个字节。
- max_stack：代表了操作数栈（Operand Stacks）深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个深度。虚拟机运行的时候需要根据这个值来分配栈帧（Stack Frame）中的操作栈深度。
- max_locals:代表了局部变量表所需的存储空间。在这里，max_locals的单位是Slot,Slot是虚拟机为局部变量分配内存所使用的最小单位。对于byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用1个Slot，而double和long这两种64位的数据类型则需要两个Slot来存放。方法参数（包括实例方法中的隐藏参数“this”）、显式异常处理器的参数（Exception Handler Parameter，就是try-catch语句中catch块所定义的异常）、方法体中定义的局部变量都需要使用局部变量表来存放。另外，并不是在方法中用到了多少个局部变量，就把这些局部变量所占Slot之和作为max_locals的值，原因是局部变量表中的Slot可以重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占的Slot可以被其他局部变量所使用，Javac编译器会根据变量的作用域来分配Slot给各个变量使用，然后计算出max_locals的大小。
- code_length和code：用来存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code是用于存储字节码指令的一系列字节流。既然叫字节码指令，那么每个指令就是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应找出这个字节码代表的是什么指令，并且可以知道这条指令后面是否需要跟随参数，以及参数应当如何理解。我们知道一个u1数据类型的取值范围为0x00～0xFF，对应十进制的0～255，也就是一共可以表达256条指令，目前，Java虚拟机规范已经定义了其中约200条编码值对应的指令含义

关于code_length，有一件值得注意的事情，虽然它是一个u4类型的长度值，理论上最大值可以达到2（32次方）-1，但是虚拟机规范中明确限制了一个方法不允许超过65535条字节码指令，即它实际只使用了u2的长度，如果超过这个限制，Javac编译器也会拒绝编译。一般来讲，编写Java代码时只要不是刻意去编写一个超长的方法来为难编译器，是不太可能超过这个最大值的限制。但是，某些特殊情况，例如在编译一个很复杂的JSP文件时，某些JSP编译器会把JSP内容和页面输出的信息归并于一个方法之中，就可能因为方法生成字节码超长的原因而导致编译失败。

#### 实战

Code属性是Class文件中最重要的一个属性，如果把一个Java程序中的信息分为代码（Code，方法体里面的Java代码）和元数据（Metadata，包括类、字段、方法定义及其他信息）两部分，那么在整个Class文件中，Code属性用于描述代码，所有的其他数据项目都用于描述元数据。了解Code属性是学习后面关于字节码执行引擎内容的必要基础，能直接阅读字节码也是工作中分析Java代码语义问题的必要工具和基本技能，因此笔者准备了一个比较详细的实例来讲解虚拟机是如何使用这个属性的。

继续以TestClass.class文件为例，实例构造器“＜init＞”方法的Code属性。

```
  public Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
```

它的操作数栈的最大深度和本地变量表的容量都为0x0001，字节码区域所占空间的长度为0x0005。虚拟机读取到字节码区域的长度后，按照顺序依次读入紧随的5个字节，并根据字节码指令表翻译出所对应的字节码指令。翻译“2A B7 00 0A B1”的过程为： 
1）读入2A，查表得0x2A对应的指令为aload_0，这个指令的含义是将第0个Slot中为reference类型的本地变量推送到操作数栈顶。 
2）读入B7，查表得0xB7对应的指令为invokespecial，这条指令的作用是以栈顶的reference类型的数据所指向的对象作为方法接收者，调用此对象的实例构造器方法、private方法或者它的父类的方法。这个方法有一个u2类型的参数说明具体调用哪一个方法，它指向常量池中的一个CONSTANT_Methodref_info类型常量，即此方法的方法符号引用。 
3）读入00 0A，这是invokespecial的参数，查常量池得0x000A对应的常量为实例构造器“＜init＞”方法的符号引用。 
4）读入B1，查表得0xB1对应的指令为return，含义是返回此方法，并且返回值为void。这条指令执行后，当前方法结束。  

这段字节码虽然很短，但是至少可以看出它的执行过程中的数据交换、方法调用等操作都是基于栈（操作栈）的。我们可以初步猜测：Java虚拟机执行字节码是基于栈的体系结构。但是与一般基于堆栈的零字节指令又不太一样，某些指令（如invokespecial)后面还会带有参数。

我们使用javap命令把此Class文件中的另外一个方法的字节码指令也计算出来。

```
{
  public Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 6: 0
}
```

如果大家注意到javap中输出的“Args_size”的值，可能会有疑问：这个类有两个方法——实例构造器＜init＞（）和inc（），这两个方法很明显都是没有参数的，为什么Args_size会为1？而且无论是在参数列表里还是方法体内，都没有定义任何局部变量，那Locals又为什么会等于1？如果有这样的疑问，大家可能是忽略了一点：在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象。这个访问机制对Java程序的编写很重要，而它的实现却非常简单，仅仅是通过Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数而已。因此在实例方法的局部变量表中至少会存在一个指向当前对象实例的局部变量，局部变量表中也会预留出第一个Slot位来存放对象实例的引用，方法参数值从1开始计算。这个处理只对实例方法有效，如果代码清单6-1中的inc（）方法声明为static，那Args_size就不会等于1而是等于0了。

在字节码指令之后的是这个方法的显式异常处理表（下文简称异常表）集合，异常表对于Code属性来说并不是必须存在的，如代码清单6-4中就没有异常表生成。

异常表的格式如表6-16所示，它包含4个字段，这些字段的含义为：如果当字节码在第start_pc行到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向一个CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转向到handler_pc处进行处理。 
![这里写图片描述](https://img-blog.csdn.net/20170802100210187?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

异常表实际上是Java代码的一部分，编译器使用异常表而不是简单的跳转命令来实现Java异常及finally处理机制。

代码清单6-5是一段演示异常表如何运作的例子，这段代码主要演示了在字节码层面中try-catch-finally是如何实现的。在阅读字节码之前，大家不妨先看看下面的Java源码，想一下这段代码的返回值在出现异常和不出现异常的情况下分别应该是多少？

代码清单6-5　异常表运作演示

```
//Java源码
public int inc（）{
int x；
try{
x=1；
return x；
}catch（Exception e）{
x=2；
return x；
}finally{
x=3；
}}
//编译后的ByteCode字节码及异常表
public int inc（）；
Code：
Stack=1，Locals=5，Args_size=1
0：iconst_1//try块中的x=1
1：istore_1
2：iload_1//保存x到returnValue中，此时x=1
3：istore 4
5：iconst_3//finaly块中的x=3
6：istore_1
7：iload 4//将returnValue中的值放到栈顶，准备给ireturn返回
9：ireturn
10：astore_2//给catch中定义的Exception e赋值，存储在Slot 2中
11：iconst_2//catch块中的x=2
12：istore_1
13：iload_1//保存x到returnValue中，此时x=2
14：istore 4
16：iconst_3//finaly块中的x=3
17：istore_1
18：iload 4//将returnValue中的值放到栈顶，准备给ireturn返回
20：ireturn
21：astore_3//如果出现了不属于java.lang.Exception及其子类的异常才会走到这里
22：iconst_3//finaly块中的x=3
23：istore_1
24：aload_3//将异常放置到栈顶，并抛出
25：athrow
Exception table：
from to target type
0 5 10 Class java/lang/Exception
0 5 21 any
10 16 21 any
```

编译器为这段Java源码生成了3条异常表记录，对应3条可能出现的代码执行路径。从Java代码的语义上讲，这3条执行路径分别为：

- 如果try语句块中出现属于Exception或其子类的异常，则转到catch语句块处理。
- 如果try语句块中出现不属于Exception或其子类的异常，则转到finally语句块处理。
- 如果catch语句块中出现任何异常，则转到finally语句块处理。

返回到我们上面提出的问题，这段代码的返回值应该是多少？对Java语言熟悉的读者应该很容易说出答案：如果没有出现异常，返回值是1；如果出现了Exception异常，返回值是2；如果出现了Exception以外的异常，方法非正常退出，没有返回值。我们一起来分析一下字节码的执行过程，从字节码的层面上看看为何会有这样的返回结果。

字节码中第0～4行所做的操作就是将整数1赋值给变量x，并且将此时x的值复制一份副本到最后一个本地变量表的Slot中（这个Slot里面的值在ireturn指令执行前将会被重新读到操作栈顶，作为方法返回值使用。为了讲解方便，笔者给这个Slot起了个名字：returnValue）。如果这时没有出现异常，则会继续走到第5～9行，将变量x赋值为3，然后将之前保存在returnValue中的整数1读入到操作栈顶，最后ireturn指令会以int形式返回操作栈顶中的值，方法结束。如果出现了异常，PC寄存器指针转到第10行，第10～20行所做的事情是将2赋值给变量x，然后将变量x此时的值赋给returnValue，最后再将变量x的值改为3。方法返回前同样将returnValue中保留的整数2读到了操作栈顶。从第21行开始的代码，作用是变量x的值赋为3，并将栈顶的异常抛出，方法结束。

尽管大家都知道这段代码出现异常的概率非常小，但并不影响它为我们演示异常表的作用。如果大家到这里仍然对字节码的运作过程比较模糊，其实也不要紧，关于虚拟机执行字节码的过程，本书第8章中将会有更详细的讲解。

#### Exceptions属性

这里的Exceptions属性是在方法表中与Code属性平级的一项属性，读者不要与前面刚刚讲解完的异常表产生混淆。Exceptions属性的作用是列举出方法中可能抛出的受查异常（Checked Excepitons），也就是方法描述时在throws关键字后面列举的异常。如下。

| 类型 | 名称                  | 数量                 |
| ---- | --------------------- | -------------------- |
| u2   | attribute_name_index  | 1                    |
| u4   | attribute_length      | 1                    |
| u2   | number_of_exceptions  | 1                    |
| u2   | exception_index_table | number_of_exceptions |


 Exceptions属性中的number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常，每一种受查异常使用一个exception_index_table项表示，exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引，代表了该受查异常的类型。

#### LineNumberTable属性

LineNumberTable属性用于描述Java源码行号与字节码行号（字节码的偏移量）之间的对应关系。它并不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中分别使用-g：none或-g：lines选项来取消或要求生成这项信息。如果选择不生成LineNumberTable属性，对程序运行产生的最主要的影响就是当抛出异常时，堆栈中将不会显示出错的行号，并且在调试程序的时候，也无法按照源码行来设置断点。LineNumberTable属性的结构如下。

| 类型                   | 名称                     | 数量                     |
| ---------------------- | ------------------------ | ------------------------ |
| u2                     | attribute_name_index     | 1                        |
| u4                     | attribute_length         | 1                        |
| u2                     | line_number_table_length | 1                        |
| line_number_table_info | line_number_table        | line_number_table_length |

 line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合，line_number_info表包括了start_pc和line_number两个u2类型的数据项，前者是字节码行号，后者是Java源码行号。

line_number_table_info属性结构表

| 类型 | 名称        | 数量 | 备注       |
| ---- | ----------- | ---- | ---------- |
| u2   | start_pc    | 1    | 字节码行号 |
| u4   | line_number | 1    | 源码行号   |

#### LocalVariableTable属性

LocalVariableTable属性用于描述栈帧中局部变量表中的变量与Java源码中定义的变量之间的关系，它也不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中分别使用-g：none或-g：vars选项来取消或要求生成这项信息。如果没有生成这项属性，最大的影响就是当其他人引用这个方法时，所有的参数名称都将会丢失，IDE将会使用诸如arg0、arg1之类的占位符代替原有的参数名，这对程序运行没有影响，但是会对代码编写带来较大不便，而且在调试期间无法根据参数名称从上下文中获得参数值。LocalVariableTable属性的结构如下。

| 类型                      | 名称                        | 数量                        |
| ------------------------- | --------------------------- | --------------------------- |
| u2                        | attribute_name_index        | 1                           |
| u4                        | attribute_length            | 1                           |
| u2                        | local_variable_table_length | 1                           |
| local_variable_table_info | local_variable_table        | local_variable_table_length |

 其中，local_variable_info项目代表了一个栈帧与源码中的局部变量的关联，结构见表6-20。 

local_variable_info结构表

| 类型 | 名称              | 数量 | 备注                                                         |
| ---- | ----------------- | ---- | ------------------------------------------------------------ |
| u2   | start_pc          | 1    | 局部变量的生命周期开始的字节码偏移量                         |
| u2   | length            | 1    | 局部变量作用范围覆盖长度                                     |
| u2   | name_index        | 1    | 局部变量名称索引                                             |
| u2   | description_index | 1    | 局部变量描述符                                               |
| u2   | index             | 1    | 局部变量在栈帧局部变量表中Slot位置，如果为64位，会占用Slot的index和index+1位置 |

start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围。

name_index和descriptor_index都是指向常量池中CONSTANT_Utf8_info型常量的索引，分别代表了局部变量的名称以及这个局部变量的描述符。

index是这个局部变量在栈帧局部变量表中Slot的位置。当这个变量数据类型是64位类型时（double和long），它占用的Slot为index和index+1两个。

顺便提一下，在JDK 1.5引入泛型之后，LocalVariableTable属性增加了一个“姐妹属性”：LocalVariableTypeTable，这个新增的属性结构与LocalVariableTable非常相似，仅仅是把记录的字段描述符的descriptor_index替换成了字段的特征签名（Signature），对于非泛型类型来说，描述符和特征签名能描述的信息是基本一致的，但是泛型引入之后，由于描述符中泛型的参数化类型被擦除掉，描述符就不能准确地描述泛型类型了，因此出现了LocalVariableTypeTable。

#### SourceFile属性

SourceFile属性用于记录生成这个Class文件的源码文件名称。这个属性也是可选的，可以分别使用Javac的-g：none或-g：source选项来关闭或要求生成这项信息。在Java中，对于大多数的类来说，类名和文件名是一致的，但是有一些特殊情况（如内部类）例外。如果不生成这项属性，当抛出异常时，堆栈中将不会显示出错代码所属的文件名。这个属性是一个定长的属性。

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | sourcefile_index     | 1    |


sourcefile_index数据项是指向常量池中CONSTANT_Utf8_info型常量的索引，常量值是源码文件的文件名。

#### ConstantValue属性

ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰的变量（类变量）才可以使用这项属性。类似“int x=123”和“static int x=123”这样的变量定义在Java程序中是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。对于非static类型的变量（也就是实例变量）的赋值是在实例构造器＜init＞方法中进行的；而对于类变量，则有两种方式可以选择：在类构造器＜clinit＞方法中或者使用ConstantValue属性。目前Sun Javac编译器的选择是：如果同时使用final和static来修饰一个变量（按照习惯，这里称“常量”更贴切），并且这个变量的数据类型是基本类型或者java.lang.String的话，就生成ConstantValue属性来进行初始化，如果这个变量没有被final修饰，或者并非基本类型及字符串，则将会选择在＜clinit＞方法中进行初始化。

虽然有final关键字才更符合“ConstantValue”的语义，但虚拟机规范中并没有强制要求字段必须设置了ACC_FINAL标志，只要求了有ConstantValue属性的字段必须设置ACC_STATIC标志而已，对final关键字的要求是Javac编译器自己加入的限制。而对ConstantValue的属性值只能限于基本类型和String，不过笔者不认为这是什么限制，因为此属性的属性值只是一个 
常量池的索引号，由于Class文件格式的常量类型中只有与基本属性和字符串相对应的字面量，所以就算ConstantValue属性想支持别的类型也无能为力。ConstantValue属性如下： 

| 类型 | 名称                 | 数量 |
| ---- | -------------------- | ---- |
| u2   | attribute_name_index | 1    |
| u4   | attribute_length     | 1    |
| u2   | constantvalue_index  | 1    |

从数据结构中可以看出，ConstantValue属性是一个定长属性，它的attribute_length数据项值必须固定为2。constantvalue_index数据项代表了常量池中一个字面量常量的引用，根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info、CONSTANT_String_info常量中的一种。

#### InnerClasses属性

InnerClasses属性用于记录内部类与宿主类之间的关联。如果一个类中定义了内部类，那编译器将会为它以及它所包含的内部类生成InnerClasses属性。该属性的结构如下。 

| 类型 | 名称                 | 数量              |
| ---- | -------------------- | ----------------- |
| u2   | attribute_name_index | 1                 |
| u4   | attribute_length     | 1                 |
| u2   | number_of_classes    | 1                 |
| u2   | inner_classes        | number_of_classes |

数据项number_of_classes代表需要记录多少个内部类信息，每一个内部类的信息都由一个inner_classes_info表进行描述。inner_classes_info表的结构如下：

inner_class_info表结构

| 类型 | 名称                    | 数量 | 备注                                                     |
| ---- | ----------------------- | ---- | -------------------------------------------------------- |
| u2   | inner_class_info_index  | 1    | 指向常量池Class索引                                      |
| u2   | outer_class_info_index  | 1    | 指向常量池Class索引                                      |
| u2   | inner_class_name_index  | 1    | 指向常量池utf-8类型索引，为内部类名称，如果为匿名类则为0 |
| u2   | inner_name_access_flags | 1    |                                                          |

 inner_class_info_index和outer_class_info_index都是指向常量池中CONSTANT_Class_info型常量的索引，分别代表了内部类和宿主类的符号引用。

inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引，代表这个内部类的名称，如果是匿名内部类，那么这项值为0。

inner_class_access_flags是内部类的访问标志，类似于类的access_flags，可选项如下。

| 标志名称       | 标志值 | 含义                           |
| -------------- | ------ | ------------------------------ |
| ACC_PUBLIC     | 0x0001 | 内部类是否为public类型         |
| ACC_PRIVATE    | 0x0002 | 内部类是否为private类型        |
| ACC_PROTECTED  | 0x0004 | 内部类是否为protected类型      |
| ACC_STATIC     | 0x0008 | 内部类是否为static类型         |
| ACC_FINAL      | 0x0010 | 内部类是否为final              |
| ACC_INTERFACE  | 0x0020 | 内部类是否为synchronized       |
| ACC_ABSTRACT   | 0x0400 | 内部类是否为abstract           |
| ACC_SYNTHETIC  | 0x1000 | 内部类是否并非由用户代码产生的 |
| ACC_ANNOTATION | 0x2000 | 内部类是否是一个注解           |
| ACC_ENUM       | 0x4000 | 内部类是否是一个枚举           |

#### Deprecated及Synthetic属性

Deprecated和Synthetic两个属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。

Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为不再推荐使用，它可以通过在代码中使用@deprecated注释进行设置。

Synthetic属性代表此字段或者方法并不是由Java源码直接产生的，而是由编译器自行添加的，在JDK 1.5之后，标识一个类、字段或者方法是编译器自动产生的，也可以设置它们访问标志中的ACC_SYNTHETIC标志位，其中最典型的例子就是Bridge Method。所有由非用户代码产生的类、方法及字段都应当至少设置Synthetic属性和ACC_SYNTHETIC标志位中的一项，唯一的例外是实例构造器“＜init＞”方法和类构造器“＜clinit＞”方法。

Deprecated和Synthetic属性的结构非常简单。如下: 

| 类型 | 名称                 | 数量 | 备注                                                         |
| ---- | -------------------- | ---- | ------------------------------------------------------------ |
| u2   | attribute_name_index | 1    |                                                              |
| u4   | attribute_length     | 1    | 0x00000000
其中attribute_length数据项的值必须为0x00000000，因为没有任何属性值需要设置。 |

#### StackMapTable属性

StackMapTable属性在JDK 1.6发布后增加到了Class文件规范中，它是一个复杂的变长属性，位于Code属性的属性表中。这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用（见7.3.2节），目的在于代替以前比较消耗性能的基于数据流分析的类型推导验证器。

这个类型检查验证器最初来源于Sheng Liang（听名字似乎是虚拟机团队中的华裔成员）为Java ME CLDC实现的字节码验证器。新的验证器在同样能保证Class文件合法性的前提下，省略了在运行期通过数据流分析去确认字节码的行为逻辑合法性的步骤，而是在编译阶段将一系列的验证类型（Verification Types）直接记录在Class文件之中，通过检查这些验证类型代替了类型推导过程，从而大幅提升了字节码验证的性能。这个验证器在JDK 1.6中首次提供，并在JDK 1.7中强制代替原本基于类型推断的字节码验证器。关于这个验证器的工作原理，《Java虚拟机规范（Java SE 7版）》花费了整整120页的篇幅来讲解描述，并且分析证明新验证方法的严谨性，笔者在此不再赘述。

StackMapTable属性中包含零至多个栈映射帧（Stack Map Frames），每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示该执行到该字节码时局部变量表和操作数栈的验证类型。类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。StackMapTable属性的结构见表6-27。 
![这里写图片描述](https://img-blog.csdn.net/20170802105030571?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

《Java虚拟机规范（Java SE 7版）》明确规定：在版本号大于或等于50.0的Class文件中，如果方法的Code属性中没有附带StackMapTable属性，那就意味着它带有一个隐式的StackMap属性。这个StackMap属性的作用等同于number_of_entries值为0的StackMapTable属性。一个方法的Code属性最多只能有一个StackMapTable属性，否则将抛出ClassFormatError异常。

#### Signature属性

Signature属性在JDK 1.5发布后增加到了Class文件规范之中，它是一个可选的定长属性，可以出现于类、属性表和方法表结构的属性表中。在JDK 1.5中大幅增强了Java语言的语法，在此之后，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variables）或参数化类型（Parameterized Types），则Signature属性会为它记录泛型签名信息。之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是擦除法实现的伪泛型，在字节码（Code属性）中，泛型信息编译（类型变量、参数化类型）之后都通通被擦除掉。使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。但坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同 
等对待，例如运行期做反射时无法获得到泛型信息。Signature属性就是为了弥补这个缺陷而增设的，现在Java的反射API能够获取泛型类型，最终的数据来源也就是这个属性。关于Java泛型、Signature属性和类型擦除，在第10章介绍编译器优化的时候会通过一个具体的例子来讲解。Signature属性的结构见表6-28。 
![这里写图片描述](https://img-blog.csdn.net/20170802105150721?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 
其中signature_index项的值必须是一个对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示类签名、方法类型签名或字段类型签名。如果当前的Signature属性是类文件的属性，则这个结构表示类签名，如果当前的Signature属性是方法表的属性，则这个结构表示方法类型签名，如果当前Signature属性是字段表的属性，则这个结构表示字段类型签名。

#### BootstrapMethods属性

BootstrapMethods属性在JDK 1.7发布后增加到了Class文件规范之中，它是一个复杂的变长属性，位于类文件的属性表中。这个属性用于保存invokedynamic指令引用的引导方法限定符。《Java虚拟机规范（Java SE 7版）》规定，如果某个类文件结构的常量池中曾经出现过CONSTANT_InvokeDynamic_info类型的常量，那么这个类文件的属性表中必须存在一个明确的BootstrapMethods属性，另外，即使CONSTANT_InvokeDynamic_info类型的常量在常量池中出现过多次，类文件的属性表中最多也只能有一个BootstrapMethods属性。BootstrapMethods属性与JSR-292中的InvokeDynamic指令和java.lang.Invoke包关系非常密切，要介绍这个属性的作用，必须先弄清楚InovkeDynamic指令的运作原理，笔者将在第8章专门用1节篇幅去介绍它们，在此先暂时略过。

目前的Javac暂时无法生成InvokeDynamic指令和BootstrapMethods属性，必须通过一些非常规的手段才能使用到它们，也许在不久的将来，等JSR-292更加成熟一些，这种状况就会改变。BootstrapMethods属性的结构见表6-29。 
![这里写图片描述](https://img-blog.csdn.net/20170802105308634?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

其中引用到的bootstrap_method结构见表6-30。 
![这里写图片描述](https://img-blog.csdn.net/20170802105343776?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHVheHVuNjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

BootstrapMethods属性中，num_bootstrap_methods项的值给出了bootstrap_methods[]数组中的引导方法限定符的数量。而bootstrap_methods[]数组的每个成员包含了一个指向常量池CONSTANT_MethodHandle结构的索引值，它代表了一个引导方法，还包含了这个引导方法静态参数的序列（可能为空）。bootstrap_methods[]数组中的每个成员必须包含以下3项内容。

bootstrap_method_ref：bootstrap_method_ref项的值必须是一个对常量池的有效索引。常量池在该索引处的值必须是一个CONSTANT_MethodHandle_info结构。

num_bootstrap_arguments：num_bootstrap_arguments项的值给出了bootstrap_arguments[]数组成员的数量。

bootstrap_arguments[]：bootstrap_arguments[]数组的每个成员必须是一个对常量池的有效索引。常量池在该索引处必须是下列结构之一：CONSTANT_String_info、CONSTANT_Class_info、CONSTANT_Integer_info、CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_MethodHandle_info或CONSTANT_MethodType_info。

#### 运行时注解相关属性

JDK 5时期，Java语言的语法进行了多项增强，其中之一是提供了对注解（Annotation）的支持。为了存储源码中注解信息，Class文件同步增加了RuntimeVisibleAnnotations、RuntimeInvisibleAnnotations、RuntimeVisibleParameterAnnotations和RuntimeInvisibleParameter-Annotations四个属性。JDK 8，进一步加强了Java语言的注解使用范围，又新增类型注解（JSR 308），所以Class文件中也同步增加了RuntimeVisibleTypeAnnotations和RuntimeInvisibleTypeAnnotations两个属性。这六个属性不论结构还是功能都比较雷同，以RuntimeVisibleAnnotations为例：

RuntimeVisibleAnnotations是一个变长属性，它记录了类、字段或方法的声明上记录运行时可见注解，当我们使用反射API来获取类、字段或方法上的注解时，返回值就是通过这个属性来取到的。

## JVM之类加载机制

### 概述

我们的源代码经过编译器编译成字节码之后，最终都需要加载到虚拟机之后才能运行。虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是**虚拟机的类加载机制**。

与那些在编译时需要进行连接工作的语言不同，在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略虽然会令类加载时稍微增加一些性能开销，但是会为Java应用程序提供高度的灵活性，Java里天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。

例如，一个面向接口的应用程序，可以等到运行时再指定实际的实现类；用户可以通过 Java 预定义的和自定义的类加载器，让一个本地的应用程序运行从网络上或其它地方加载一个二进制流作为程序代码的一部分。

### 类加载的条件

类从被虚拟机从加载到卸载，整个生命周期包含：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）、卸载（Unloading）7 个阶段。其中验证、准备、解析 3 个部分统称为连接（Linking）。如下图

![JVM-类加载](png\JVM-类加载.png)

 


加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。注意，这里笔者写的是按部就班地“开始”，而不是按部就班地“进行”或“完成”，强调这点是因为这些阶段通常都是互相交叉地混合式进行的，通常会在一个阶段执行的过程中调用、激活另外一个阶段。

什么情况下需要开始类加载过程的第一个阶段：加载？Java虚拟机规范中并没有进行强制约束，这点可以交给虚拟机的具体实现来自由把握。但是对于初始化阶段，虚拟机规范则是严格规定了有且只有5种情况必须立即对类进行“初始化”（而加载、验证、准备自然需要在此之前开始）：

- 1）遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
- 2）使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
- 3）当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 4）当虚拟机启动时，用户需要指定一个要执行的主类（包含main（）方法的那个类），虚拟机会先初始化这个主类。
- 5）当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

对于这5种会触发类进行初始化的场景，虚拟机规范中使用了一个很强烈的限定语：“有且只有”，这5种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。下面举3个例子来说明何为被动引用，分别见代码清单7-1～代码清单7-3。 

```
package org.fenixsoft.classloading;

/**
 * 被动使用类字段演示一：
 * 通过子类引用父类的静态字段，不会导致子类初始化
 **/
public class SuperClass {

    static {
        System.out.println("SuperClass init!");
    }

    public static int value = 123;
}

public class SubClass extends SuperClass {

    static {
        System.out.println("SubClass init!");
    }
}

/**
 * 非主动使用类字段演示
 **/
public class NotInitialization {

    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }

}
```


上述代码运行之后，只会输出“SuperClass init！”，而不会输出“SubClass init！”。对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定，这点取决于虚拟机的具体实现。对于Sun HotSpot虚拟机来说，可通过-XX：+TraceClassLoading参数观察到此操作会导致子类的加载。 

```
package org.fenixsoft.classloading;

/**
 * 被动使用类字段演示二：
 * 通过数组定义来引用类，不会触发此类的初始化
 **/
public class NotInitialization {

    public static void main(String[] args) {
        SuperClass[] sca = new SuperClass[10];
    }

}
```


为了节省版面，这段代码复用了代码清单7-1中的SuperClass，运行之后发现没有输出“SuperClass init！”，说明并没有触发类org.fenixsoft.classloading.SuperClass的初始化阶段。但是这段代码里面触发了另外一个名为“[Lorg.fenixsoft.classloading.SuperClass”的类的初始化阶段，对于用户代码来说，这并不是一个合法的类名称，它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建动作由字节码指令newarray触发。

这个类代表了一个元素类型为org.fenixsoft.classloading.SuperClass的一维数组，数组中应有的属性和方法（用户可直接使用的只有被修饰为public的length属性和clone（）方法）都实现在这个类里。Java语言中对数组的访问比C/C++相对安全是因为这个类封装了数组元素的访问方法，而C/C++直接翻译为对数组指针的移动。在Java语言中，当检查到发生数组越界时会抛出java.lang.ArrayIndexOutOfBoundsException异常。 

```
package org.fenixsoft.classloading;

/**
 * 被动使用类字段演示三：
 * 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
 **/
public class ConstClass {

    static {
        System.out.println("ConstClass init!");
    }

    public static final String HELLOWORLD = "hello world";
}

/**
 * 非主动使用类字段演示
 **/
public class NotInitialization {

    public static void main(String[] args) {
        System.out.println(ConstClass.HELLOWORLD);
    }
}
```


上述代码运行之后，也没有输出“ConstClass init！”，这是因为虽然在Java源码中引用了ConstClass类中的常量HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“hello world”存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口，这两个类在编译成Class之后就不存在任何联系了。

接口的加载过程与类加载过程稍有一些不同，针对接口需要做一些特殊说明：接口也有初始化过程，这点与类是一致的，上面的代码都是用静态语句块“static{}”来输出初始化信息的，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成“＜clinit＞（）”类构造器，用于初始化接口中所定义的成员变量。接口与类真正有所区别的是前面讲述的5种“有且仅有”需要开始初始化场景中的第3种：当一个类在初始化时，要求其父类全部都已经初始化过了，但是一个接口在初始化时，并不要求其父接口全部都完成了初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。

### 类加载的过程

详细讲解一下Java虚拟机中类加载的全过程，也就是加载、验证、准备、解析和初始化这5个阶段所执行的具体动作。

#### 加载（Loading）

**加载**是**类加载**（Class Loading）过程的一个阶段。在加载阶段，虚拟机需要完成以下3件事情：

- 通过一个类的全限定名来获取定义此类的二进制字节流。
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

对于类的二进制流，虚拟机可以通过多种途径产生或者获得。

- 从ZIP包中读取，这很常见，最终成为日后JAR、EAR、WAR格式的基础。
- 从网络中获取，这种场景最典型的应用就是Applet。
- 运行时计算生成，这种场景使用得最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流。
- 由其他文件生成，典型场景是JSP应用，即由JSP文件生成对应的Class类。
- 从数据库中读取，这种场景相对少见些，例如有些中间件服务器（如SAP Netweaver）可以选择把程序安装到数据库中来完成程序代码在集群间的分发。 

**对于数组而言，**数组类本身不用过类加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但数组类的元素类型最终还是需要依靠类加载器来完成加载。一个数组类（以C代之）创建过程遵循下列规则：

- 如果数组的组件类型是引用型，则使用上述加载规则，递归加载这个组件类型。**数组C将被标识在加载该组件的类加载器的类名空间上**
- 如果数组的组件类型不是引用型（如int[]数组的组件类型int），**Java虚拟机将会把数组C标记为与引导类加载器关联**
- 数组类的可访问性与它的组件类型的可访问性一致。如果组件类型不是引用型，则可访问性默认是public

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构。然后在内存中实例化一个java.lang.Class类的对象（并没有明确规定是在Java堆中，对于HotSpot虚拟机而言，Class对象比较特殊，它虽然是对象，但是存放在方法区里面），这个对象将作为程序访问方法区中的这些类型数据的外部接口。

#### 验证（Verification）

验证的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

验证阶段大致上会完成下面4个阶段的检验动作：文件格式验证、元数据验证、字节码验证、符号引用验证。

##### 文件格式验证

第一阶段要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理。验证点如下：

- 是否以魔数0xCAFEBABE开头。
- 主、次版本号是否在当前虚拟机处理范围之内。
- 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）。
- 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
- CONSTANT_Utf8_info型的常量中是否有不符合UTF8编码的数据。
- Class文件中各个部分及文件本身是否有被删除的或附加的其他信息。

……

该验证阶段的主要目的是保证输入的字节流能正确地解析并存储于方法区之内，格式上符合描述一个Java类型信息的要求。这阶段的验证是基于二进制字节流进行的，只有通过了这个阶段的验证后，字节流才会进入内存的方法区中进行存储，所以后面的3个验证阶段全部是基于方法区的存储结构进行的，不会再直接操作字节流。

##### 元数据验证

第二阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求。验证点如下：

- 这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）。
- 这个类的父类是否继承了不允许被继承的类（被final修饰的类）。
- 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
- 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。

……

第二阶段的主要目的是对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。

##### 字节码验证

第三阶段是整个验证过程中最复杂的一个阶段，主要目的是通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。验证点如下：

- 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似这样的情况：在操作栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中。
- 保证跳转指令不会跳转到方法体以外的字节码指令上。
- 保证方法体中的类型转换是有效的，例如可以把一个子类对象赋值给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。

……

如果一个类方法体的字节码没有通过字节码验证，那肯定是有问题的；但如果一个方法体通过了字节码验证，也不能说明其一定就是安全的。即使字节码验证之中进行了大量的检查，也不能保证这一点。这里涉及了离散数学中一个很著名的问题“Halting Problem”：通俗一点的说法就是，通过程序去校验程序逻辑是无法做到绝对准确的——不能通过程序准确地检查出程序是否能在有限的时间之内结束运行。

由于数据流验证的高复杂性，虚拟机设计团队为了避免过多的时间消耗在字节码验证阶段，在JDK 1.6之后的Javac编译器和Java虚拟机中进行了一项优化，给方法体的Code属性的属性表中增加了一项名为“StackMapTable”的属性，这项属性描述了方法体中所有的基本块（Basic Block，按照控制流拆分的代码块）开始时本地变量表和操作栈应有的状态，在字节码验证期间，就不需要根据程序推导这些状态的合法性，只需要检查StackMapTable属性中的记录是否合法即可。这样将字节码验证的类型推导转变为类型检查从而节省一些时间。

理论上StackMapTable属性也存在错误或被篡改的可能，所以是否有可能在恶意篡改了Code属性的同时，也生成相应的StackMapTable属性来骗过虚拟机的类型校验则是虚拟机设计者值得思考的问题。

在JDK 1.6的HotSpot虚拟机中提供了-XX：-UseSplitVerifier选项来关闭这项优化，或者使用参数-XX：+FailOverToOldVerifier要求在类型校验失败的时候退回到旧的类型推导方式进行校验。而在JDK 1.7之后，对于主版本号大于50的Class文件，使用类型检查来完成数据流分析校验则是唯一的选择，不允许再退回到类型推导的校验方式。

##### 符号引用验证

最后一个阶段的校验发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三阶段——解析阶段中发生。符号引用验证可以看做是对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验，验证点如下：

- 符号引用中通过字符串描述的全限定名是否能找到对应的类。
- 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
- 符号引用中的类、字段、方法的访问性（private、protected、public、default）是否可被当前类访问。 
  ……

符号引用验证的目的是确保解析动作能正常执行，如果无法通过符号引用验证，那么将会抛出一个java.lang.IncompatibleClassChangeError异常的子类，如java.lang.IllegalAccessError、java.lang.NoSuchFieldError、java.lang.NoSuchMethodError等。

对于虚拟机的类加载机制来说，验证阶段是一个非常重要的、但不是一定必要（因为对程序运行期没有影响）的阶段。如果所运行的全部代码（包括自己编写的及第三方包中的代码）都已经被反复使用和验证过，那么在实施阶段就可以考虑使用-Xverify：none参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。



### 准备（Preparation）

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。这个阶段中有两个容易产生混淆的概念需要强调一下，首先，这时候进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。其次，这里所说的初始值“通常情况”下是数据类型的零值，假设一个类变量的定义为：

```
public static int value=123;
```

那变量value在准备阶段过后的初始值为0而不是123，因为这时候尚未开始执行任何Java方法，而把value赋值为123的putstatic指令是程序被编译后，存放于类构造器＜clinit＞（）方法之中，所以把value赋值为123的动作将在初始化阶段才会执行。Java中所有基本数据类型的零值如下：

| 数据类型  |   零值   |
| :-------: | :------: |
|    int    |    0     |
|   long    |    0L    |
|   short   | (short)0 |
|   char    | '\u0000' |
|   byte    | (byte)0  |
|  boolean  |  false   |
|   float   |   0.0f   |
|  double   |   0.0d   |
| reference |   null   |


上面提到，在“通常情况”下初始值是零值，那相对的会有一些“特殊情况”：如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量value就会被初始化为ConstantValue属性所指定的值，假设上面类变量value的定义变为：

```
public static final int value=123;
```

编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为123。

### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，符号引用在Class文件中它以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现，那解析阶段中所说的直接引用与符号引用又有什么关联呢？

符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到内存中。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。

直接引用（Direct References）：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在内存中存在。

虚拟机规范之中并未规定解析阶段发生的具体时间，只要求了在执行anewarray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invokespecial、invokestatic、invokevirtual、ldc、ldc_w、multianewarray、new、putfield和putstatic这16个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析。所以虚拟机实现可以根据需要来判断到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用将要被使用前才去解析它。

对同一个符号引用进行多次解析请求是很常见的事情，除invokedynamic指令以外，虚拟机实现可以对第一次解析的结果进行缓存（在运行时常量池中记录直接引用，并把常量标识为已解析状态）从而避免解析动作重复进行。无论是否真正执行了多次解析动作，虚拟机需要保证的是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直成功；同样的，如果第一次解析失败了，那么其他指令对这个符号的解析请求也应该收到相同的异常。

对于invokedynamic指令，上面规则则不成立。当碰到某个前面已经由invokedynamic指令触发过解析的符号引用时，并不意味着这个解析结果对于其他invokedynamic指令也同样生效。因为invokedynamic指令的目的本来就是用于动态语言支持（目前仅使用Java语言不会生成这条字节码指令），它所对应的引用称为“动态调用点限定符”（Dynamic Call Site Specifier），这里“动态”的含义就是必须等到程序实际运行到这条指令的时候，解析动作才能进行。相对的，其余可触发解析的指令都是“静态”的，可以在刚刚完成加载阶段，还没有开始执行代码时就进行解析。

解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行，分别对应于常量池的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info、CONSTANT_InterfaceMethodref_info、CONSTANT_MethodType_info、CONSTANT_MethodHandle_info和CONSTANT_InvokeDynamic_info 7种常量类型。下面将讲解前面4种引用的解析过程，对于后面3种，与JDK 1.7新增的动态语言支持息息相关，由于Java语言是一门静态类型语言，因此在没有介绍invokedynamic指令的语义之前，没有办法将它们和现在的Java语言对应上，笔者将在介绍动态语言调用时一起分析讲解。

1.类或接口的解析

假设当前代码所处的类为D，如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用，那虚拟机完成整个解析的过程需要以下3个步骤： 
1）如果C不是一个数组类型，那虚拟机将会把代表N的全限定名传递给D的类加载器去加载这个类C。在加载过程中，由于元数据验证、字节码验证的需要，又可能触发其他相关类的加载动作，例如加载这个类的父类或实现的接口。一旦这个加载过程出现了任何异常，解析过程就宣告失败。 
2）如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似“[Ljava/lang/Integer”的形式，那将会按照第1点的规则加载数组元素类型。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表此数组维度和元素的数组对象。 
3）如果上面的步骤没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成之前还要进行符号引用验证，确认D是否具备对C的访问权限。如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

2.字段解析

要解析一个未被解析过的字段符号引用，首先将会对字段表内class_index[2]项中索引的CONSTANT_Class_info符号引用进行解析，也就是字段所属的类或接口的符号引用。如果在解析这个类或接口符号引用的过程中出现了任何异常，都会导致字段符号引用解析的失败。如果解析成功完成，那将这个字段所属的类或接口用C表示，虚拟机规范要求按照如下步骤对C进行后续字段的搜索。 
1）如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。 
2）否则，如果在C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。 
3）否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。 
4）否则，查找失败，抛出java.lang.NoSuchFieldError异常。

如果查找过程成功返回了引用，将会对这个字段进行权限验证，如果发现不具备对字段的访问权限，将抛出java.lang.IllegalAccessError异常。

在实际应用中，虚拟机的编译器实现可能会比上述规范要求得更加严格一些，如果有一个同名字段同时出现在C的接口和父类中，或者同时在自己或父类的多个接口中出现，那编译器将可能拒绝编译。在代码清单7-4中，如果注释了Sub类中的“public static int A=4；”，接口与父类同时存在字段A，那编译器将提示“The field Sub.A is ambiguous”，并且拒绝编译这段代码。 

```
package org.fenixsoft.classloading;

public class FieldResolution {

    interface Interface0 {
        int A = 0;
    }

    interface Interface1 extends Interface0 {
        int A = 1;
    }

    interface Interface2 {
        int A = 2;
    }

    static class Parent implements Interface1 {
        public static int A = 3;
    }

    static class Sub extends Parent implements Interface2 {
        public static int A = 4;
    }

    public static void main(String[] args) {
        System.out.println(Sub.A);
    }
}
```

3.类方法解析

类方法解析的第一个步骤与字段解析一样，也需要先解析出类方法表的class_index项中索引的方法所属的类或接口的符号引用，如果解析成功，我们依然用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的类方法搜索。 
1）类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中发现class_index中索引的C是个接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。 
2）如果通过了第1步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。 
3）否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。 
4）否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时查找结束，抛出java.lang.AbstractMethodError异常。 
5）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。 
最后，如果查找过程成功返回了直接引用，将会对这个方法进行权限验证，如果发现不具备对此方法的访问权限，将抛出java.lang.IllegalAccessError异常。

4.接口方法解析

接口方法也需要先解析出接口方法表的class_index[4]项中索引的方法所属的类或接口的符号引用，如果解析成功，依然用C表示这个接口，接下来虚拟机将会按照如下步骤进行后续的接口方法搜索。 
1）与类方法解析不同，如果在接口方法表中发现class_index中的索引C是个类而不是接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。 
2）否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。 
3）否则，在接口C的父接口中递归查找，直到java.lang.Object类（查找范围会包括Object类）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。 
4）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError异常。

由于接口中的所有方法默认都是public的，所以不存在访问权限的问题，因此接口方法的符号解析应当不会抛出java.lang.IllegalAccessError异常。



### 初始化

类初始化阶段是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。

在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，或者可以从另外一个角度来表达：初始化阶段是执行类构造器＜clinit＞（）方法的过程。我们在下文会讲解＜clinit＞（）方法是怎么生成的，在这里，我们先看一下＜clinit＞（）方法执行过程中一些可能会影响程序运行行为的特点和细节，这部分相对更贴近于普通的程序开发人员。

＜clinit＞（）方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问，如代码清单7-5中的例子所示。 



```
public class Test {
    static {
        i = 0;  //  给变量复制可以正常编译通过
        System.out.print(i);  // 这句编译器会提示“非法向前引用”  
    }
    static int i = 1;
}
```




＜clinit＞（）方法与类的构造函数（或者说实例构造器＜init＞（）方法）不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的＜clinit＞（）方法执行之前，父类的＜clinit＞（）方法已经执行完毕。因此在虚拟机中第一个被执行的＜clinit＞（）方法的类肯定是java.lang.Object。

由于父类的＜clinit＞（）方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作，如在代码清单7-6中，字段B的值将会是2而不是1。 



```
static class Parent {
        public static int A = 1;
        static {
            A = 2;
        }
}

static class Sub extends Parent {
        public static int B = A;
}

public static void main(String[] args) {
        System.out.println(Sub.B);
}
```

[


＜clinit＞（）方法对于类或接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成＜clinit＞（）方法。

接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成＜clinit＞（）方法。但接口与类不同的是，执行接口的＜clinit＞（）方法不需要先执行父接口的＜clinit＞（）方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类在初始化时也一样不会执行接口的＜clinit＞（）方法。

虚拟机会保证一个类的＜clinit＞（）方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的＜clinit＞（）方法，其他线程都需要阻塞等待，直到活动线程执行＜clinit＞（）方法完毕。如果在一个类的＜clinit＞（）方法中有耗时很长的操作，就可能造成多个进程阻塞[2]，在实际应用中这种阻塞往往是很隐蔽的。代码清单7-7演示了这种场景。 



```
static class DeadLoopClass {
    static {
        // 如果不加上这个if语句，编译器将提示“Initializer does not complete normally”并拒绝编译
        if (true) {
            System.out.println(Thread.currentThread() + "init DeadLoopClass");
            while (true) {
            }
        }
    }
}

public static void main(String[] args) {
    Runnable script = new Runnable() {
        public void run() {
            System.out.println(Thread.currentThread() + "start");
            DeadLoopClass dlc = new DeadLoopClass();
            System.out.println(Thread.currentThread() + " run over");
        }
    };

    Thread thread1 = new Thread(script);
    Thread thread2 = new Thread(script);
    thread1.start();
    thread2.start();
}
```




运行结果如下，即一条线程在死循环以模拟长时间操作，另外一条线程在阻塞等待。

```
Thread[Thread-0 ,5 ,main]start 
Thread[Thread-1 ,5 ,main]start 
Thread[Thread-0 ,5 ,main]init DeadLoopClass
 
```

类加载过程如下：

- 加载。加载分为三步：
  - 通过类的全限定性类名获取该类的二进制流
  - 将该二进制流的静态存储结构转为方法区的运行时数据结构
  - 在堆中为该类生成一个class对象
- 验证：验证该class文件中的字节流信息复合虚拟机的要求，不会威胁到jvm的安全
- 准备：为class对象的静态变量分配内存，初始化其初始值
- 解析：该阶段主要完成符号引用转化成直接引用
- 初始化：到了初始化阶段，才开始执行类中定义的java代码；初始化阶段是调用类构造器的过程

# 类加载器

虚拟机设计团队把类加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。

类加载器可以说是Java语言的一项创新，也是Java语言流行的重要原因之一，它最初是为了满足Java Applet的需求而开发出来的。虽然目前Java Applet技术基本上已经“死掉”，但类加载器却在类层次划分、OSGi、热部署、代码加密等领域大放异彩，成为了Java技术体系中一块重要的基石，可谓是失之桑榆，收之东隅。

虚拟机设计团队把加载动作放到 JVM 外部实现，以便让应用程序决定如何获取所需的类，JVM 提供了 3 种类加载器：

![Classloader](F:/lemon-guide-main/images/JVM/Classloader.png)

- **启动类加载器(Bootstrap ClassLoader)**

  **用来加载java核心类库，无法被java程序直接引用**。负责加载 JAVA_HOME\lib 目录中的，或通过-Xbootclasspath 参数指定路径中的，且被虚拟机认可（按文件名识别，如 rt.jar）的类。

- **扩展类加载器(Extension ClassLoader)**

  **用来加载java的扩展库，java的虚拟机实现会提供一个扩展库目录，该类加载器在扩展库目录里面查找并加载java类**。负责加载 JAVA_HOME\lib\ext 目录中的，或通过 java.ext.dirs 系统变量指定路径中的类库。

- **应用程序类加载器(Application ClassLoader)**

  **它根据java的类路径来加载类，一般来说，java应用的类都是通过它来加载的**。负责加载用户路径（classpath）上的类库。JVM 通过双亲委派模型进行类的加载，当然我们也可以通过继承java.lang.ClassLoader 实现自定义的类加载器。

- **自定义类加载器(User ClassLoader)**

  由JAVA语言实现，继承自ClassLoader。



此外我们比较需要知道的几点：

- 一个类是由 jvm 加载是通过类加载器+全限定类名确定唯一性的
- 双亲委派，众所周知，子加载器会尽量委托给父加载器进行加载，父加载器找不到再自己加载
- 线程上下文类加载，为了满足 spi 等需求突破双亲委派机制，当高层类加载器想加载底层类时通过 Thread.contextClassLoader 来获取当前线程的类加载器(往往是底层类加载器)去加载类

## 类与类加载器

类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类加载阶段。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

这里所指的“相等”，包括代表类的Class对象的equals（）方法、isAssignableFrom（）方法、isInstance（）方法的返回结果，也包括使用instanceof关键字做对象所属关系判定等情况。如果没有注意到类加载器的影响，在某些情况下可能会产生具有迷惑性的结果，代码清单7-8中演示了不同的类加载器对instanceof关键字运算的结果的影响。



```
/**
 * 类加载器与instanceof关键字演示
 * 
 */
public class ClassLoaderTest {

    public static void main(String[] args) throws Exception {

        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };

        Object obj = myLoader.loadClass("org.fenixsoft.classloading.ClassLoaderTest").newInstance();

        System.out.println(obj.getClass());
        System.out.println(obj instanceof org.fenixsoft.classloading.ClassLoaderTest);
    }
}
```


运行结果：

```
class org.fenixsoft.classloading.ClassLoaderTest 
false
```

代码清单7-8中构造了一个简单的类加载器，尽管很简单，但是对于这个演示来说还是够用了。它可以加载与自己在同一路径下的Class文件。我们使用这个类加载器去加载了一个名为“org.fenixsoft.classloading.ClassLoaderTest”的类，并实例化了这个类的对象。两行输出结果中，从第一句可以看出，这个对象确实是类org.fenixsoft.classloading.ClassLoaderTest实例化出来的对象，但从第二句可以发现，这个对象与类org.fenixsoft.classloading.ClassLoaderTest做所属类型检查的时候却返回了false，这是因为虚拟机中存在了两个ClassLoaderTest类，一个是由系统应用程序类加载器加载的，另外一个是由我们自定义的类加载器加载的，虽然都来自同一个Class文件，但依然是两个独立的类，做对象所属类型检查时结果自然为false。



## 双亲委派模型

从Java虚拟机的角度来讲，只存在两种不同的类加载器：一种是启动类加载器 
（Bootstrap ClassLoader），这个类加载器使用C++语言实现，是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。

从Java开发人员的角度来看，类加载器还可以划分得更细致一些，绝大部分Java程序都会使用到以下3种系统提供的类加载器。

启动类加载器（Bootstrap ClassLoader）：前面已经介绍过，这个类将器负责将存放在＜JAVA_HOME＞\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null代替即可，如代码清单 
7-9所示为java.lang.ClassLoader.getClassLoader（）方法的代码片段。

![img](https://img2018.cnblogs.com/blog/1168971/201903/1168971-20190322173525021-374793122.png)

扩展类加载器（Extension ClassLoader）：这个加载器由sun.misc.Launcher 
$ExtClassLoader实现，它负责加载＜JAVA_HOME＞\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

应用程序类加载器（Application ClassLoader）：这个类加载器由sun.misc.Launcher $App-ClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader（）方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

我们的应用程序都是由这3种类加载器互相配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器之间的关系一般如图7-2所示。 
![img](https://img2018.cnblogs.com/blog/1168971/201903/1168971-20190322173542868-1522797052.png)

 

图7-2中展示的类加载器之间的这种层次关系，称为类加载器的双亲委派模型（Parents Delegation Model）。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码。

类加载器的双亲委派模型在JDK 1.2期间被引入并被广泛应用于之后几乎所有的Java程序中，但它并不是一个强制性的约束模型，而是Java设计者推荐给开发者的一种类加载器实现方式。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，**因此所有的加载请求最终都应该传送到顶层的启动类加载器中**，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，**无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载**，因此Object类在程序的各种类加载器环境中都是同一个类。**相反，如果没有使用双亲委派模型，由各个类加载器自行去加载的话，如果用户自己编写了一个称为java.lang.Object的类，并放在程序的ClassPath中，那系统中将会出现多个不同的Object类，Java类型体系中最基础的行为也就无法保证，应用程序也将会变得一片混乱**。如果读者有兴趣的话，可以尝试去编写一个与rt.jar类库中已有类重名的Java类，将会发现可以正常编译，但永远无法被加载运行。

双亲委派模型对于保证Java程序的稳定运作很重要，但它的实现却非常简单，实现双亲委派的代码都集中在java.lang.ClassLoader的loadClass（）方法之中，代码如下：先检查是否已经被加载过，若没有加载则调用父加载器的loadClass（）方法，若父加载器为空则默认使用启动类加载器作为父加载器。如果父类加载失败，抛出ClassNotFoundException异常后，再调用自己的findClass（）方法进行加载。

java.lang.ClassLoader 的 loadClass() 方法：

```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException{
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // 如果父类加载器抛出 ClassNotFoundException
                    // 说明父类加载器无法完成加载请求
                }

                if (c == null) {
                    // 在父类加载器无法加载时
                    // 再调用本身的 findClass 方法来进行类加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // 这是定义类加载器，记录统计数据
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```


双亲委派

当一个类加载器收到一个类加载的请求，他首先不会尝试自己去加载，而是将这个请求委派给父类加载器去加载，只有父类加载器在自己的搜索范围类查找不到给类时，子加载器才会尝试自己去加载该类。

所有的加载请求都会传送到根加载器去加载，只有当父加载器无法加载时，子类加载器才会去加载：

![双亲委派](png/双亲委派.png)

**为什么需要双亲委派模型？**

为了防止内存中出现多个相同的字节码。因为如果没有双亲委派的话，用户就可以自己定义一个java.lang.String类，那么就无法保证类的唯一性。

**那怎么打破双亲委派模型？**

自定义类加载器，继承ClassLoader类，重写loadClass方法和findClass方法。

**双亲委派模型的作用**

- 避免类的重复加载
- 保证Java核心类库的安全

## 破坏双亲委派模型

上文提到过双亲委派模型并不是一个强制性的约束模型，而是Java设计者推荐给开发者的类加载器实现方式。在Java的世界中大部分的类加载器都遵循这个模型，但也有例外，到目前为止，双亲委派模型主要出现过3较大规模的“被破坏”情况。

双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK 1.2发布之前。由于双亲委派模型在JDK 1.2之后才被引入，而类加载器和抽象类java.lang.ClassLoader则在JDK 1.0时代就已经存在，面对已经存在的用户自定义类加载器的实现代码，Java设计者引入双亲委派模型时不得不做出一些妥协。为了向前兼容，JDK 1.2之后的java.lang.ClassLoader添加了一个新的protected方法findClass（），在此之前，用户去继承java.lang.ClassLoader的唯一目的就是为了重写loadClass（）方法，因为虚拟机在进行类加载的时候会调用加载器的私有方法loadClassInternal（），而这个方法的唯一逻辑就是去调用自己的loadClass（）。

上一节我们已经看过loadClass（）方法的代码，双亲委派的具体逻辑就实现在这个方法之中，JDK 1.2之后已不提倡用户再去覆盖loadClass（）方法，而应当把自己的类加载逻辑写到findClass（）方法中，在loadClass（）方法的逻辑里如果父类加载失败，则会调用自己的findClass（）方法来完成加载，这样就可以保证新写出来的类加载器是符合双亲委派规则的。

双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷所导致的，双亲委派很好地解决了各个类加载器的基础类的统一问题（越基础的类由越上层的加载器进行加载），基础类之所以称为“基础”，是因为它们总是作为被用户代码调用的API，但世事往往没有绝对的完美，如果基础类又要调用回用户的代码，那该怎么办？

这并非是不可能的事情，一个典型的例子便是JNDI服务，JNDI现在已经是Java的标准服务，它的代码由启动类加载器去加载（在JDK 1.3时放进去的rt.jar），但JNDI的目的就是对资源进行集中管理和查找，它需要调用由独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者（SPI,Service Provider Interface）的代码，但启动类加载器不可能“认识”这些代码啊！那该怎么办？

为了解决这个问题，Java设计团队只好引入了一个不太优雅的设计：线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过java.lang.Thread类的setContextClassLoaser（）方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。

有了线程上下文类加载器，就可以做一些“舞弊”的事情了，JNDI服务使用这个线程上下文类加载器去加载所需要的SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作，这种行为实际上就是打通了双亲委派模型的层次结构来逆向使用类加载器，实际上已经违背了双亲委派模型的一般性原则，但这也是无可奈何的事情。Java中所有涉及SPI的加载动作基本上都采用这种方式，例如JNDI、JDBC、JCE、JAXB和JBI等。

双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的，这里所说的“动态性”指的是当前一些非常“热门”的名词：代码热替换（HotSwap）、模块热部署（Hot Deployment）等，说白了就是希望应用程序能像我们的计算机外设那样，接上鼠标、U盘，不用重启机器就能立即使用，鼠标有问题或要升级就换个鼠标，不用停机也不用重启。对于个人计算机来说，重启一次其实没有什么大不了的，但对于一些生产系统来说，关机重启一次可能就要被列为生产事故，这种情况下热部署就对软件开发者，尤其是企业级软件开发者具有很大的吸引力。

Sun公司所提出的JSR-294[1]、JSR-277[2]规范在与JCP组织的模块化规范之争中落败给JSR-291（即OSGi R4.2），虽然Sun不甘失去Java模块化的主导权，独立在发展Jigsaw项目，但目前OSGi已经成为了业界“事实上”的Java模块化标准[3]，而OSGi实现模块化热部署的关键则是它自定义的类加载器机制的实现。每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换。

在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载请求时，OSGi将按照下面的顺序进行类搜索： 
1）将以java.*开头的类委派给父类加载器加载。 
2）否则，将委派列表名单内的类委派给父类加载器加载。 
3）否则，将Import列表中的类委派给Export这个类的Bundle的类加载器加载。 
4）否则，查找当前Bundle的ClassPath，使用自己的类加载器加载。 
5）否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载。 
6）否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载。 
7）否则，类查找失败。 
上面的查找顺序中只有开头两点仍然符合双亲委派规则，其余的类查找都是在平级的类加载器中进行的。

笔者虽然使用了“被破坏”这个词来形容上述不符合双亲委派模型原则的行为，但这里“被破坏”并不带有贬义的感情色彩。只要有足够意义和理由，突破已有的原则就可认为是一种创新。正如OSGi中的类加载器并不符合传统的双亲委派的类加载器，并且业界对其为了实现热部署而带来的额外的高复杂度还存在不少争议，但在Java程序员中基本有一个共识：OSGi中对类加载器的使用是很值得学习的，弄懂了OSGi的实现，就可以算是掌握了类加载器的精髓。

# 本章小结

本章介绍了类加载过程的“加载”、“验证”、“准备”、“解析”和“初始化”5个阶段中虚拟机进行了哪些动作，还介绍了类加载器的工作原理及其对虚拟机的意义。

经过第6和第7两章的讲解，相信读者已经对如何在Class文件中定义类，如何将类加载到虚拟机中这两个问题有了比较系统的了解，第8章我们将一起来看看虚拟机如何执行定义在Class文件里的字节码。

# 虚拟机字节码执行引擎

## 概述

执行引擎是Java虚拟机最核心的组成部分之一。“虚拟机”是一个相对于“物理机”的概念 ,这两种机器都有代码执行能力,其区别是物理机的执行引擎是直接建立在处理器、硬件、指令集和操作系统层面上的,而虚拟机的执行引擎则是由自己实现的,因此可以自行制定指令集与执行引擎的结构体系,并且能够执行那些不被硬件直接支持的指令集格式。

在Java虚拟机规范中制定了虚拟机字节码执行引擎的概念模型,这个概念模型成为各种虚拟机执行引擎的统一外观(Facade )。在不同的虚拟机实现里面,执行引擎在执行Java代码的时候可能会有解释执行(通过解释器执行)和编译执行(通过即时编译器产生本地代码执行)两种选择 , 也可能两者兼备,甚至还可能会包含几个不同级别的编译器执行引擎。 但从外观上看起来,所有的Java虚拟机的执行引擎都是一致的:输入的是字节码文件,处理过程是字节码解析的等效过程,输出的是执行结果,本章将主要从概念模型的角度来讲解虚拟机的方法调用和字节码执行。

有一些虚拟机(如Sun Classic VM ) 的内部只存在解释器,只能解释执行,而另外一些虚拟机(如BEA JRockit) 的内部只存在即时编译器,只能编译执行。



## 运行时栈帧结构

栈帧( Stack Frame ) 是用于支持虚拟机进行方法调用和方法执行的数据结构,它是虚拟机运行时数据区中的虚拟机栈( Virtual Machine Stack ) 的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态连接和方法[返回地址](https://www.baidu.com/s?wd=返回地址&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)等信息。每一个方法从调用开始至执行完成的过程 ,都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

每一个栈帧都包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。在编译程序代码的时候,栈帧中需要多大的局部变量表,多深的操作数栈都已经完全确定了,并且写入到方法表的Code属性之中,因此一个栈帧需要分配多少内存,不会受到程序运行期变量数据的影响,而仅仅取决于具体的虚拟机实现。

一个线程中的方法调用链可能会很长,很多方法都同时处于执行状态。对于执行引擎来说,在活动线程中,只有位于栈顶的栈帧才是有效的,称为当前栈帧( Current Stack Frame ) , 与这个栈帧相关联的方法称为当前方法( Current Method )。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作,在概念模型上,典型的栈帧结构如图8-1所示。

![img](https://img-blog.csdn.net/20151025144817491)

接下来详细讲解一下栈帧的局部变量表、操作数栈、动态连接、方法返回地址等各个部分的作用和数据结构。



### 局部变量表

局部变量表(Local Variable Table)是一组变量值存储空间,**用于存放方法参数和方法内部定义的局部变量**。在Java程序编译为Class文件时 ,就在方法的Code属性的max_locals数据项中确定了该方法所需要分配的局部变量表的最大容量。

局部变量表的容量以变量槽( Variable Slot,下称Slot) 为最小单位,虚拟机规范中并没有明确指明一个Slot应占用的内存空间大小,只是很有导向性地说到每个Slot都应该能存放一 个boolean、byte、char、short、int、float、reference或returnAddress类型的数据,这8种数据类型,都可以使用32位或更小的物理内存来存放,但这种描述与明确指出“每个Slot占用32位长度的内存空间”是有一些差别的,它允许Slot的长度可以随着处理器、操作系统或虚拟机的不同而发生变化。只要保证即使在64位虚拟机中使用了64位的物理内存空间去实现一个Slot, 虚拟机仍要使用对齐和补白的手段让Slot在外观上看起来与32位虚拟机中的一致。

既然前面提到了Java虚拟机的数据类型,在此再简单介绍一下它们。一个Slot可以存放一个32位以内的数据类型,Java中占用32位以内的数据类型有boolean、byte、char、short、int、float、reference和returnAddress 8种类型。前面6种不需要多加解释,读者可以按照Java 语言中对应数据类型的概念去理解它们(仅是这样理解而已,Java语言与Java虚拟机中的基本数据类型是存在本质差别的),而第7种reference类型表示对一个对象实例的引用,虚拟机规范既没有说明它的长度,也没有明确指出这种引用应有怎样的结构。但一般来说,虚拟机实现至少都应当能通过这个引用做到两点,一是从此引用中直接或间接地查找到对象在Java 堆中的数据存放的起始地址索引,二是此引用中直接或间接地查找到对象所属数据类型在方法区中的存储的类型信息,否则无法实现Java语言规范中定义的语法约束约束 。第8种即returnAddress类型目前已经很少见了,它是为字节码指令jsr、 jsr_w和ret服务的,指向了一条字节码指令的地址,很古老的Java虚拟机曾经使用这几条指令来实现异常处理,现在已经由异常表代替。

对于64位的数据类型,虚拟机会以高位对齐的方式为其分配两个连续的Slot空间。 Java语言中明确的(reference类型则可能是32位也可能是64位 )64位的数据类型只有long和double两种。值得一提的是 ,这里把long和double数据矣型分割存储的做法与“long和double的非原子性协定” 中把一次long和double数据类型读写分割为两次32位读写的做法有些类似,读者阅读到Java内存模型时可以互相对比一下。不过,由于局部变量表建立在线程的堆栈上,是线程私有的数据,无论读写两个连续的Slot是否为原子操作,都不会引起数据安全问题。

虚拟机通过索引定位的方式使用局部变量表,索引值的范围是从0开始至局部变量表最大的Slot数量。如果访问的是32位数据类型的变量,索引n就代表了使用第n个Slot,如果是64 位数据类型的变量,则说明会同时使用n和n+1两个Slot。对于两个相邻的共同存放一个64位数据的两个Slot,不允许采用任何方式单独访问其中的某一个,Java虚拟机规范中明确要求了如果遇到进行这种操作的字节码序列,虚拟机应该在类加载的校验阶段拋出异常。

在方法执行时,虚拟机是使用局部变量表完成参数值到参数变量列表的传递过程的,如果执行的是实例方法(非static的方法),那局部变量表中第0位索引的Slot默认是用于传递方法所属对象实例的引用,在方法中可以通过关键字“this”来访问到这个隐含的参数。其余参数则按照参数表顺序排列，占用从1开始的局部变量Slot,参数表分配完毕后,再根据方法体内部定义的变量顺序和作用域分配其余的Slot。

为了尽可能节省栈帧空间,局部变量表中的Slot是可以重用的,方法体中定义的变量, 其作用域并不一定会覆盖整个方法体,如果当前字节码PC计数器的值已经超出了某个变量的作用域 ,那这个变量对应的Slot就可以交给其他变量使用。不过 ,这样的设计除了节省栈帧空间以外,还会伴随一些额外的副作用,例如 ,在某些情况下,Slot的复用会直接影响到系统的垃圾收集行为,请看代码清单8-1〜代码清单8-3的3个演示。

代码清单8 - 1 局部变量表Slot复用对垃圾收集的影响之一

```csharp
public static void main(String[] args)() {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    System.gc();
}
```

代码清单8-1中的代码很简单,即向内存填充了64MB的数据 ,然后通知虚拟机进行垃圾收集。我们在虚拟机运行参数中加上“-verbose : gc”来看看垃圾收集的过程,发现在 System.gc() 运行后并没有回收这64MB的内存,下面是运行的结果:

```css
[GC 66846K->65824K (125632K ) ,0.0032678 secs] [Full GC 65824K-> 65746K (125632K) ,0.0064131 secs]
```

没有回收placeholder所占的内存能说得过去,因为在执行Systemgc() 时 ,变量 placeholder还处于作用域之内,虚拟机自然不敢回收placeholder的内存。那我们把代码修改一下 ,变成代码清单8-2中的样子。

代码清单8 - 2 局部变量表Slot复用对垃圾收集的影响之二

```csharp
public static void main(String[] args)() {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    System.gc();
}
```

加入了花括号之后,placeholder的作用域被限制在花括号之内,从代码逻辑上讲,在执行System.gc() 的时候,placeholder已经不可能再被访问了,但执行一下这段程序,会发现运行结果如下,还是有64MB的内存没有被回收,这又是为什么呢?

在解释为什么之前,我们先对这段代码进行第二次修改,在调用System.gc() 之前加入 —行“int a=0;” , 变成代码清单8-3的样子。

代码清单8 - 3 局部变量表Slot复用对垃圾收集的影响之三

```csharp
public static void main(String[] args)() {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    int a = 0;
    System.gc();
}
```

这个修改看起来[莫名其妙](https://www.baidu.com/s?wd=莫名其妙&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)，但运行一下程序，却发现这次内存真的被正确回收了。

```css
[GC 66401K-> 65778K (125632K ) ,0.0035471 secs] [Full GC 65778K->218K (125632K) ,0.0140596 secs]
```

在代码清单8-1〜代码清单8-3中 ,placeholder能否被回收的根本原因是:局部变量表中的Slot是否还存有关于placeholder数组对象的引用。第一次修改中,代码虽然已经离开了placeholder的作用域,但在此之后,没有任何对局部变量表的读写操作,placeholder原本所占用的Slot还没有被其他变量所复用,所以作为GC Roots —部分的局部变量表仍然保持着对它的关联。这种关联没有被及时打断,在绝大部分情况下影响都很轻微。但如果遇到一个方法 ,其后面的代码有一些耗时很长的操作,而前面又定义了占用了大量内存、实际上已经不会再使用的变量,手动将其设置为null值(用来代替那句int a=0, 把变量对应的局部变量表 Slot清 空 )便不见得是一个绝对无意义的操作,这种操作可以作为一种在极特殊情形(对象占用内存大、此方法的栈帧长时间不能被回收、方法调用次数达不到JIT的编译条件)下的“奇技”来使用。Java语言的一本非常著名的书籍《 Practical Java》中把“不使用的对象应手动赋值为null”作为一条推荐的编码规则,但是并没有解释具体的原因,很长时间之内都有读者对这条规则感到疑惑。

虽然代码清单8-1〜代码清单8-3的代码示例说明了赋null值的操作在某些情况下确实是有用的 ,但笔者的观点是不应当对赋null值的操作有过多的依赖,更没有必要把它当做一个普遍的编码规则来推广。原因有两点,从编码角度讲,以恰当的变量作用域来控制变量回收时间才是最优雅的解决方法,如代码清单8-3那样的场景并不多见。更关键的是,从执行角度讲 ,使用赋null值的操作来优化内存回收是建立在对字节码执行引擎概念模型的理解之上的 ,在第6章介绍完字节码后,笔者专门增加了一个6.5节“公有设计、私有实现”来强调概念模型与实际执行过程是外部看起来等效,内部看上去则可以完全不同。在虚拟机使用解释器执行时 ,通常与概念模型还比较接近,但经过JIT编译器后 ,才是虚拟机执行代码的主要方式 ,赋null值的操作在经过JIT编译优化后就会被消除掉,这时候将变量设置为null就是没有意义的。字节码被编译为本地代码后,对GC Roots的枚举也与解释执行时期有巨大差别,以前面例子来看,代码清单8-2在经过JIT编译后, System.gc() 执行时就可以正确地回收掉内存 ,无须写成代码清单8-3的样子。

关于局部变量表,还有一点可能会对实际开发产生影响,就是局部变量不像前面介绍的类变量那样存在“准备阶段”。通过第7章的讲解,我们已经知道类变量有两次赋初始值的过程 ,一次在准备阶段,赋予系统初始值;另外一次在初始化阶段,赋予程序员定义的初始值。因此 ,即使在初始化阶段程序员没有为类变量赋值也没有关系,类变量仍然具有一个确定的初始值。但局部变量就不一样,如果一个局部变量定义了但没有赋初始值是不能使用的,不要认为Java中任何情况下都存在诸如整型变量默认为0 ,布尔型变量默认为false等这样的默认值。如代码清单8-4所 示 ,这段代码其实并不能运行,还好编译器能在编译期间就检查到并提示这一点,即便编译能通过或者手动生成字节码的方式制造出下面代码的效果,字节码校验的时候也会被虛拟机发现而导致类加载失败。

代码清单8 - 4 未赋值的局部变量

```csharp
public static void main(String[] args) {
    int a;
    System.out.println(a);
}
```

注：Java虚拟机规范中没有明确规定reference类型的长度,它的长度与实际使用32还是64位虚 拟机有关,如果是64位虚拟机,还与是否开启某些对象指针压缩的优化有关,这里暂且只取32位虚拟机的reference长度。



### 操作数栈

操作数栈( Operand Stack ) 也常称为操作栈,它是一个后入先出( Last In First Out,LIFO )栈。同局部变量表一样,操作数栈的最大深度也在编译的时候写入到Code属性的max_Stacks数据项中。操作数栈的每一个元素可以是任意的Java数据类型,包括long和double。32位数据类型所占的栈容量为1 ,64位数据类型所占的栈容量为2。在方法执行的任何时候 ,操作数栈的深度都不会超过在max_Stacks数据项中设定的最大值。

当一个方法刚刚开始执行的时候,这个方法的操作数栈是空的,在方法的执行过程中, 会有各种字节码指令往操作数栈中写入和提取内容,也就是出栈/ 入栈操作。例如 ,**在做算术运算的时候是通过操作数栈来进行的,又或者在调用其他方法的时候是通过操作数栈来进行参数传递**。

举个例子,**整数加法的字节码指令iadd在运行的时候操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值,当执行这个指令时,会将这两个int值出栈并相加,然后将相加的结果入栈**。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器要严格保证这一点，在类校验阶段的数据流分析中还要再次验证这一点。再以上面的iadd指令为例，这个指令用于整型数加法，它在执行时，最接近栈顶的两个元素的数据类型必须为int型，不能出现一个long和一个float使用iadd命令相加的情况。

另外，在概念模型中，两个栈帧作为虚拟机栈的元素，是完全相互独立的。但大多虚拟机的实现里都会做一些优化处理，**令两个栈帧出现一部分重叠。让下面栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起，这样在进行方法调用时就可以共用一部分数据，无须进行额外的参数复制传递**，重叠的过程如图8-2所示。

![img](https://img-blog.csdn.net/20151025144226697)

Java虚拟机的解释执行引擎称为“基于栈的执行引擎”,其中所指的“栈”就是操作数栈。



### 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用,持有这个引用是为了支持方法调用过程中的动态连接(Dynamic Linking)。通过第6章的讲解,我们知道Class文件的常量池中存有大量的符号引用,字节码中的方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就转化为直接引用 ,这种转化称为静态解析。另外一部分将在每一次运行期间转化为直接引用,这部分称为动态连接。关于这两个转化过程的详细信息,将在8.3节中详细讲解。



### 方法返回地址

当一个方法开始执行后,只有两种方式可以退出这个方法。第一种方式是执行引擎遇到任意一个方法返回的字节码指令,这时候可能会有返回值传递给上层的方法调用者(调用当前方法的方法称为调用者),是否有返回值和返回值的类型将根据遇到何种方法返回指令来决定,这种退出方法的方式称为正常完成出口(Normal Method Invocation Completion ) 。

另外一种退出方式是,在方法执行过程中遇到了异常,并且这个异常没有在方法体内得到处理,无论是Java虚拟机内部产生的异常,还是代码中使用athrow字节码指令产生的异 常 ,只要在本方法的异常表中没有搜索到匹配的异常处理器,就会导致方法退出,这种退出方法的方式称为异常完成出口( Abrupt Method Invocation Completion)。一个方法使用异常完成出口的方式退出,是不会给它的上层调用者产生任何返回值的。

无论采用何种退出方式,在方法退出之后,都需要返回到方法被调用的位置,程序才能继续执行,方法返回时可能需要在栈帧中保存一些信息,用来帮助恢复它的上层方法的执行状态。一般来说,方法正常退出时,调用者的PC计数器的值可以作为返回地址,栈帧中很可能会保存这个计数器值。而方法异常退出时,返回地址是要通过异常处理器表来确定的,栈帧中一般不会保存这部分信息。

方法退出的过程实际上就等同于把当前栈帧出栈,因此退出时可能执行的操作有:恢复上层方法的局部变量表和操作数栈,把返回值(如果有的话)压入调用者栈帧的操作数栈中 ,调整PC计数器的值以指向方法调用指令后面的一条指令等。



### 附加信息

虚拟机规范允许具体的虚拟机实现增加一些规范里没有描述的信息到栈帧之中,例如与调试相关的信息,这部分信息完全取决于具体的虚拟机实现,这里不再详述。在实际开发中 ,一般会把动态连接、方法返回地址与其他附加信息全部归为一类,称为栈帧信息。



## 方法调用

方法调用并不等同于方法执行,方法调用阶段唯一的任务就是确定被调用方法的版本(即调用哪一个方法),暂时还不涉及方法内部的具体运行过程。在程序运行时,进行方法调用是最普遍、最频繁的操作,但前面已经讲过,Class文件的编译过程中不包含传统编译中的连接步骤,一 切方法调用在Class文件里面存储的都只是符号引用,而不是方法在实际运行时内存布局中的入口地址(相当于之前说的直接引用)。这个特性给Java带来了更强大的动态扩展能力,但也使得Java方法调用过程变得相对复杂起来,需要在类加载期间,甚至到运行期间才能确定目标方法的直接引用。



### 解析

继续前面关于方法调用的话题，所有方法调用中的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能成立的前提是：方法在程序真正运行之前就有一个确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。换句话说，调用目标在程序代码写好、编译器进行编译时就必须确定下来。这类方法的调用称为解析（Resolution）。

在Java语言中符合“编译器可知，运行期不可变”这个要求的方法，主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各个的特点决定了它们都不可能通过继承或别的方式重写其他版本，因此它们都适合在类加载阶段进行解析。

与之相对应的是,在Java虚拟机里面提供了5条方法调用字节码指令,分别如下。

- invokestatic :调用静态方法。
- invokespecial :调用实例构造器<init>方法、私有方法和父类方法。
- invokevirtual :调用所有的虚方法。
- invokeinterface :调用接口方法,会在运行时再确定一个实现此接口的对象。
- invokedynamic :先在运行时动态解析出调用点限定符所引用的方法,然后再执行该方法 ,在此之前的4条调用指令,分派逻辑是固化在Java虚拟机内部的,而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。

只要能被invokestatic和invokespecial指令调用的方法,都可以在解析阶段中确定唯一的调用版本,符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类 ,它们在类加载的时候就会把符号引用解析为该方法的直接引用。这些方法可以称为非虚方法,与之相反 ,其他方法称为虚方法(除去final方法 ,后文会提到)。代码清单8-5演示了一个最常见的解析调用的例子,此样例中,静态方法sayHello() 只可能属于类型StaticResolution , 没有任何手段可以覆盖或隐藏这个方法。

代码清单8 - 5 方法静态解析演示



```
/**
 * 方法静态解析演示
 * 
 * @author zzm
 */
public class StaticResolution {

    public static void sayHello() {
        System.out.println("hello world");
    }

    public static void main(String[] args) {
        StaticResolution.sayHello();
    }

}
```



使用javap命令查看这段程序的字节码,会发现的确是通过invokestatic命令来调用sayHello()方法的。

![img](https://img-blog.csdn.net/20151025154902266)

Java中的非虛方法除了使用invokestatic、invokespecial调用的方法之外还有一种,就是被final修饰的方法。虽然final方法是使用invokevirtual指令来调用的,但是由于它无法被覆盖, 没有其他版本,所以也无须对方法接收者进行多态选择,又或者说多态选择的结果肯定是唯一的。在Java语言规范中明确说明了final方法是一种非虚方法。

解析调用一定是个静态的过程,在编译期间就完全确定,在类装载的解析阶段就会把涉及的符号引用全部转变为可确定的直接引用,不会延迟到运行期再去完成。而分派(Dispatch)调用则可能是静态的也可能是动态的,根据分派依据的宗量数可分为单分派和多分派。这两类分派方式的两两组合就构成了静态单分派、静态多分派、动态单分派、动态多分派4种分派组合情况,下面我们再看看虚拟机中的方法分派是如何进行的。



### 分派

众所周知,Java是一门面向对象的程序语言,因为Java具备面向对象的3个基本特征:继承、封装和多态。本节讲解的分派调用过程将会揭示多态性特征的一些最基本的体现, 如“重载”和“重写”在Java虚拟机之中是如何实现的,这里的实现当然不是语法上该如何写, 我们关心的依然是虚拟机如何确定正确的目标方法。

#### 1.静态分派

在开始讲解静态分派前 ,笔者准备了一段经常出现在面试题中的程序代码,读者不妨先看一遍,想一下程序的输出结果是什么。后面我们的话题将围绕这个类的方法来重载(Overload)代码,以分析虚拟机和编译器确定方法版本的过程。方法静态分派如代码清单8-6所示。

代码清单8 - 6 方法静态分派演示

```
package org.fenixsoft.polymorphic;

/**
 * 方法静态分派演示
 * @author zzm
 */
public class StaticDispatch {

    static abstract class Human {
    }

    static class Man extends Human {
    }

    static class Woman extends Human {
    }

    public void sayHello(Human guy) {
        System.out.println("hello,guy!");
    }

    public void sayHello(Man guy) {
        System.out.println("hello,gentleman!");
    }

    public void sayHello(Woman guy) {
        System.out.println("hello,lady!");
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
```



运行结果:

```
hello,guy!
hello,guy!
```

代码清单8-6中的代码实际上是在考验阅读者对重载的理解程度,相信对Java编程稍有经验的程序员看完程序后都能得出正确的运行结果,但为什么会选择执行参数类型为Human的重载呢?在解决这个问题之前,我们先按如下代码定义两个重要的概念。

```
Human man=new Man();
```

我们把上面代码中的“Human”称为变量的静态类型( Static Type ) , 或者叫做的外观类型 ( Apparent Type ) , 后面的“Man”则称为变量的实际类型( Actual Type ), 静态类型和实际类型在程序中都可以发生一些变化,区别是静态类型的变化仅仅在使用时发生,变量本身的静态类型不会被改变,并且最终的静态类型是在编译期可知的;而实际类型变化的结果在运行期才可确定,编译器在编译程序的时候并不知道一个对象的实际类型是什么。例如下面的代码:



```
// 实际类型变化
Human man=new Man(); 
man=new Woman();
// 静态类型变化
sr.sayHello((Man)man);
sr.sayHello((Woman)man);
```



解释了这两个概念,再回到代码清单8-6的样例代码中。main()里面的两次sayHello() 方法调用,在方法接收者已经确定是对象“sr”的前提下,使用哪个重载版本,就完全取决于传入参数的数量和数据类型。代码中刻意地定义了两个静态类型相同但实际类型不同的变量,但虚拟机(准确地说是编译器)在重载时是通过参数的静态类型而不是实际类型作为判定依据的。并且静态类型是编译期可知的,因此 ,在编译阶段,Javac编译器会根据参数的静态类型决定使用哪个重载版本,所以选择了sayHello(Human) 作为调用目标, 并把这个方法的符号引用写到main() 方法里的两条invokevirtual指令的中 。

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派的典型应用是方法重载。静态分派发生在编译阶段,因此确定静态分派的动作实际上不是由虚拟机来执行的。另外 ,编译器虽然能确定出方法的重载版本,但在很多情况下这个重载版本并不 是“唯一的” ,往往只能确定一个“更加合适的”版本。这种模糊的结论在由0和1构成的计算机世界中算是比较“稀罕” 的事情 ,产生这种模糊结论的主要原因是字面量不需要定义,所以字面量没有显式的静态类型,它的静态类型只能通过语言上的规则去理解和推断。代码清单8- 7演示了何为“更加合适的”版本。

代码清单8 - 7 重载方法匹配优先级



```
package org.fenixsoft.polymorphic;

public class Overload {

    public static void sayHello(Object arg) {
        System.out.println("hello Object");
    }

    public static void sayHello(int arg) {
        System.out.println("hello int");
    }

    public static void sayHello(long arg) {
        System.out.println("hello long");
    }

    public static void sayHello(Character arg) {
        System.out.println("hello Character");
    }

    public static void sayHello(char arg) {
        System.out.println("hello char");
    }

    public static void sayHello(char... arg) {
        System.out.println("hello char ...");
    }

    public static void sayHello(Serializable arg) {
        System.out.println("hello Serializable");
    }

    public static void main(String[] args) {
        sayHello('a');
    }
}
```



上面的代码运行后会输出:

```
hello char
```

这很好理解,‘a’是一个char类型的数据,自然会寻找参数类型为char的重载方法,如果注释掉sayHello(char arg) 方法,那输出会变为:

```
hello int
```

这时发生了一次自动类型转换,’a’除了可以代表一个字符串,还可以代表数字97 (字符,a,的Unicode数值为十进制数字97 ) , 因此参数类型为int的重载也是合适的。我们继续注释掉sayHello(int arg)方法,那输出会变为:

```
hello long
```

这时发生了两次自动类型转换,’a’转型为整数97之后 ,进一步转型为长整数97L ,匹配了参数类型为long的重载。笔者在代码中没有写其他的类型如float、double等的重载,不过实际上自动转型还能继续发生多次,按照char->int-> long-> float-> double的顺序转型进行匹配。但不会匹配到byte和short类型的重载,因为char到byte或short的转型是不安全的。我们继续注释掉sayHello(long arg)方法,那输会变为:

```
hello Character
```

这时发生了一次自动装箱,’a’被包装为它的封装类型java.lang.Character ,所以匹配到了参数类型为Character的重载,继续注释掉sayHello(Character arg) 方法,那输出会变为:

```
hello Serializable
```

这个输出可能会让人感觉摸不着头脑,一个字符或数字与序列化有什么关系?出现hello Serializable,是因为java.lang.Serializable是java.lang.Character类实现的一个接口,当自动装箱之后发现还是找不到装箱类,但是找到了装箱类实现了的接口类型,所以紧接着又发生一次自动转型。char可以转型成int,但是Character是绝对不会转型为Integer的 ,它只能安全地转型为它实现的接口或父类。Character还实现了另外一个接口java.lang.Comparable<Character> , 如果同时出现两个参数分别为Serializable和Comparable<Character>的重载方法,那它们在此时的优先级是一样的。编译器无法确定要自动转型为哪种类型,会提示类型模糊,拒绝编译。程序必须在调用时显式地指定字面量的静态类型,如 : sayHello((Comparable<Character>)’a’) , 才能编译通过。下面继续注释掉sayHello(Serializable arg)方法 ,输出会变为:

```
hello Object
```

这时是char装箱后转型为父类了,如果有多个父类,那将在继承关系中从下往上开始搜索 ,越接近上层的优先级越低。即使方法调用传入的参数值为null时 ,这个规则仍然适用。 我们把sayHello(Object arg) 也注释掉,输出将会变为:

```
hello char ...
```

7个重载方法已经被注释得只剩一个了,可见变长参数的重载优先级是最低的,这时候字符’a’被当做了一个数组元素。笔者使用的是char类型的变长参数,读者在验证时还可以选择int类型、Character类型、Object类型等的变长参数重载来把上面的过程重新演示一遍。但要注意的是,有一些在单个参数中能成立的自动转型,如char转型为int ,在变长参数中是不成立的。

代码清单8-7演示了编译期间选择静态分派目标的过程,这个过程也是Java语言实现方法重载的本质。演示所用的这段程序属于很极端的例子,除了用做面试题为难求职者以外,在 实际工作中几乎不可能有实际用途。笔者拿来做演示仅仅是用于讲解重载时目标方法选择的过程 ,大部分情况下进行这样极端的重载都可算是真正的“关于茴香豆的茴有几种写法的研究”。无论对重载的认识有多么深刻,一个合格的程序员都不应该在实际应用中写出如此极端的重载代码。

另外还有一点读者可能比较容易混淆:笔者讲述的解析与分派这两者之间的关系并不是二选一的排他关系,它们是在不同层次上去筛选、确定目标方法的过程。例如,前面说过, 静态方法会在类加载期就进行解析,而静态方法显然也是可以拥有重载版本的,选择重载版本的过程也是通过静态分派完成的。

#### 2.动态分派

了解了静态分派,我们接下来看一下动态分派的过程,它和多态性的另外一个重要体现——-重写(Override)有着很密切的关联。我们还是用前面的Man和Woman一起sayHello的例子来讲解动态分派,请看代码清单8-8中所示的代码。

代码清单8 - 8 方法动态分派演示



```
package org.fenixsoft.polymorphic;

/**
 * 方法动态分派演示
 * @author zzm
 */
public class DynamicDispatch {

    static abstract class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}
```



运行结果:

```
man say hello 
woman say hello 
woman say hello
```

这个运行结果相信不会出乎任何人的意料,对于习惯了面向对象思维的Java程序员会觉得这是完全理所当然的。现在的问题还是和前面的一样,虚拟机是如何知道要调用哪个方法的?

显然这里不可能再根据静态类型来决定,因为静态类型同样都是Human的两个变量man和woman在调用sayHello()方法时执行了不同的行为,并且变量man在两次调用中执行了不同的方法。导致这个现象的原因很明显,是这两个变量的实际类型不同,Java虚拟机是如何根据实际类型来分派方法执行版本的呢?我们使用javap命令输出这段代码的字节码,尝试从中寻找答案,输出结果如代码清单8-9所示。

代码清单8-9 main() 方法的字节码

![img](https://img-blog.csdn.net/20151025202334322)
![img](https://img-blog.csdn.net/20151025202357546)

0 〜15行的字节码是准备动作,作用是建立man和woman的内存空间、调用Man和Woman 类型的实例构造器,将这两个实例的引用存放在第1、2个局部变量表Slot之中 ,这个动作也就对应了代码中的这两句:

```
Human man=new Man(); 
Human woman=new Woman();
 
```

接下来的16〜21句是关键部分,16、20两句分别把刚刚创建的两个对象的引用压到栈顶 ,这两个对象是将要执行的sayHello()方法的所有者,称为接收者( Receiver ) ; 17和21句是方法调用指令,这两条调用指令单从字节码角度来看,无论是指令(都是invokevirtual) 还是参数(都是常量池中第22项的常量,注释显示了这个常量是Human.sayHello()的符号引用)完全一样的,但是这两句指令最终执行的目标方法并不相同。原因就需要从invokevirtual指令的多态查找过程开始说起,invokevirtual指令的运行时解析过程大致分为以下几个步骤:

- 1 ) 找到操作数栈顶的第一个元素所指向的对象的实际类型,记作C。
- 2 ) 如果在类型C中找到与常量中的描述符和简单名称都相符的方法,则进行访问权限校验 ,如果通过则返回这个方法的直接引用,查找过程结束;如果不通过,则返回 java.lang.IllegalAccessError异常。
- 3 ) 否则,按照继承关系从下往上依次对C的各个父类进行第2步的搜索和验证过程。
- 4 ) 如果始终没有找到合适的方法,则拋出java.lang.AbstractMethodError异常。

由于invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型,所以两次调用中的invokevirtual指令把常量池中的类方法符号引用解析到了不同的直接引用上,这个过程就是Java语言中方法重写的本质。我们把这种在运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

#### 3.单分派与多分派

方法的接收者与方法的参数统称为方法的宗量,这个定义最早应该来源于《.丨ava与模 式》一书。根据分派基于多少种宗量,可以将分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。

单分派和多分派的定义读起来拗口,从字面上看也比较抽象,不过对照着实例看就不难理解了。代码清单8-10中列举了一个Father和Son—起来做出“一个艰难的决定”的例子。

代码清单8 - 10 单分派和多分派



```
/**
 * 单分派、多分派演示
* @author zzm
 */
public class Dispatch {

    static class QQ {}

    static class _360 {}

    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("father choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}
```



运行结果:

```
father choose 360 
son choose qq
```

在main函数中调用了两次hardChoice() 方法 ,这两次hardChoice() 方法的选择结果在程序输出中已经显示得很清楚了。

我们来看看编译阶段编译器的选择过程,也就是静态分派的过程。这时选择目标方法的依据有两点: 一是静态类型是Father还是Son,二是方法参数是QQ还是360。这次选择结果的最终产物是产生了两条invokevirtual指令 ,两条指令的参数分别为常量池中指向 Father.hardChoice ( 360 ) 及Father.hardChoice ( QQ ) 方法的符号引用。因为是根据两个宗量进行选择,所以Java语言的静态分派属于多分派类型。

再看看运行阶段虚拟机的选择,也就是动态分派的过程。在执行“son.hardChoice ( new QQ ( ) ) ”这句代码时,更准确地说,是在执行这句代码所对应的invokevirtual指令时,由于编译期已经决定目标方法的签名必须为hardChoice ( QQ ) , 虛拟机此时不会关心传递过来的参数“QQ”到底是“腾讯QQ”还是“奇瑞QQ” ,因为这时参数的静态类型、实际类型都对方法的选择不会构成任何影响,唯一可以影响虚拟机选择的因素只有此方法的接受者的实际类型是Father还是Son。因为只有一个宗量作为选择依据,所以Java语言的动态分派属于单分派类型。

根据上述论证的结果,我们可以总结一句:今天(直至还未发布的Java1.8 )的Java语言是一门静态多分派、动态单分派的语言。强调“今天的Java语言”是因为这个结论未必会恒久不变 ,C#在3.0及之前的版本与Java—样是动态单分派语言,但在C#4.0中引入了dynamic类型后 ,就可以很方便地实现动态多分派。

按照目前Java语言的发展趋势，它并没有直接变为动态语言的迹象,而是通过内置动态语言(如JavaScript)执行引擎的方式来满足动态性的需求。但是Java虚拟机层面上则不是如此 ,在JDK 1.7中实现的JSR-292里面就已经开始提供对动态语言的支持了, JDK 1.7中新增的invokedymmic指令也成为了最复杂的一条方法调用的字节码指令,稍后笔者将专门讲解这个JDK 1.7的新特性。

#### 4.虚拟机动态分派的实现

前面介绍的分派过程,作为对虚拟机概念模型的解析基本上已经足够了,它已经解决了虚拟机在分派中“会做什么”这个问题。但是虚拟机“具体是如何做到的”,可能各种虚拟机的实现都会有些差别。

由于动态分派是非常频繁的动作,而且动态分派的方法版本选择过程需要运行时在类的方法元数据中搜索合适的目标方法,因此在虚拟机的实际实现中基于性能的考虑,大部分实现都不会真正地进行如此频繁的搜索。面对这种情况,最常用的“稳定优化”手段就是为类在方法区中建立一个虚方法表(Vritual Method Table,也称为vtable,与此对应的,在invokeinterface执行时也会用到接口方法表——Inteface Method Table,简称itable ) ,被用虚方法表索引来代替元数据查找以提高性能。我们先看看代码清单8-10所对应的虚方法表结构示例 ,如图8-3所示。

![img](https://img-blog.csdn.net/20151025224817212)

虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写,那子类的虚方法表里面的地址入口和父类相同方法的地址入口是一致的,都指向父类的实现入口。如果子类中重写了这个方法,子类方法表中的地址将会替换为指向子类实现版本的入口地址。图8-3中 ,Son重写了来自Father的全部方法,因此Son的方法表没有指向Father类型数据的箭头。但是Son和Father都没有重写来自Object的方法 ,所以它们的方法表中所有从Object继承来的方法都指向了 Object的数据类型。

为了程序实现上的方便,具有相同签名的方法,在父类、子类的虚方法表中都应当具有一样的索引序号,这样当类型变换时,仅需要变更查找的方法表,就可以从不同的虚方法表中按索引转换出所需的入口地址。

方法表一般在类加载的连接阶段进行初始化,准备了类的变量初始值后,虚拟机会把该类的方法表也初始化完毕。

上文中笔者说方法表是分派调用的“稳定优化”手段 ,虚拟机除了使用方法表之外,在条件允许的情况下,还会使用内联缓存( Inline Cache )和基于“类型继承关系分析” ( Class Hierarchy Analysis,CHA ) 技术的守护内联( Guarded Mining ) 两种非稳定的“激进优化”手段来获得更高的性能,关于这两种优化技术的原理和运作过程,读者可以参考本书第11章中的相关内容。



### 动态类型语言支持

Java虚拟机的字节码指令集的数量从Sun公司的第一款Java虚拟机问世至JDK 7来临之前的十余年时间里,一直没有发生任何变化。随着JDK 7的发布,字节码指令集终于迎来了第一位新成员—— invokedynamic指令。这条新增加的指令是JDK 7实现“动态类型语言” (Dynamically Typed Language ) 支持而进行的改进之一,也是为JDK 8可以顺利实现Lambda表达式做技术准备。在本节中,我们将详细讲解JDK 7这项新特性出现的前因后果和它的深远意义。

#### 1.动态类型语言

在介绍Java虚拟机的动态类型语言支持之前,我们要先弄明白动态类型语言是什么?它与Java语言、Java虚拟机有什么关系?了解JDK 1.7提供动态类型语言支持的技术背景,对理解这个语言特性是很有必要的。

什么是动态类型语言? 动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期,满足这个特征的语言有很多,常用的包括:APL、Clojure、Erlang、 Groovy、JavaScript、Jython、Lisp、Lua、PHP、Prolog、Python、Ruby、Smalltalk和Tcl等。相对的 ,在编译期就进行类型检查过程的语言(如C++和Java等 )就是最常用的静态类型语言。

觉得上面定义过于概念化?那我们不妨通过两个例子以最浅显的方式来说明什么是“在编译期/运行期进行”和什么是“类型检查”。首先看下面这段简单的Java代码 ,它是否能正常编译和运行?

```
public static void main (String[]args ){ 
    int[][][] array=new int[1][0][-1];
}
```

这段代码能够正常编译,但运行的时候会报NegativeArraySizeException异常。在Java虚拟机规范中明确规定了NegativeArraySizeException是运行时异常 ,通俗一点来说,运行时异常就是只要代码不运行到这一行就不会有问题。与运行时异常相对应的是连接时异常, 例如很常见的NoClassDefFoundError便属于连接时异常,即使会导致连接时异常的代码放在一条无法执行到的分支路径上,类加载时(Java的连接过程不在编译阶段,而在类加载阶段)也照样会拋出异常。

不过,在C语言中,含义相同的代码会在编译期报错:

```
int main (void ) {
    int i[1][0][-1] ;//GCC拒绝编译,报'size of array is negative'
    return 0;
}
```

由此看来,一门语言的哪一种检查行为要在运行期进行,哪一种检查要在编译期进行并没有必然的因果逻辑关系,关键是语言规范中人为规定的。再举一个例子来解释“类型检查” ,例如下面这一句非常简单的代码:

```
obj.println("hello world");
```

虽然每个人都能看懂这行代码要做什么，但对于计算机来说,这一行代码“没头没尾”是无法执行的,它需要一个具体的上下文才有讨论的意义。

现在假设这行代码是在Java语言中,并且变量obj的静态类型为java.io.PrintStream ,那变量obj的实际类型就必须是PrintStream的子类(实现了PrintStream接口的类)才是合法的。否则 ,哪怕obj属于一个确实有用println(String)方法,但与PrintStream接口没有继承关系,代码依然不可能运行— 因为类型检查不合法。

但是相同的代码在ECMAScript (JavaScript)中情况则不一样,无论obj具体是何种类型 ,只要这种类型的定义中确实包含有println ( String ) 方 法 ,那方法调用使可成功。

这种差别产生的原因是Java语言在编译期间已将pnntln( String )方法完整的符号引用(本例中为一个CONSTANT_InterfaceMethodref_info常量)生成出来,作为方法调用指令的参数存储到Class文件中,例如下面这段代码:

```
invokevirtual#4 ;//Method java/io/PrintStream.println:(Ljava/lang/String ;)V
```

这个符号引用包含了此方法定义在哪个具体类型之中、方法的名字以及参数顺序、参数类型和方法返回值等信息,通过这个符号引用,虚拟机可以翻译出这个方法的直接引用。而在ECMAScript等动态类型语言中,变量obj本身是没有类型的,变量obj的值才具有类型,编译时最多只能确定方法名称、参数、返回植这些信息,而不会去确定古法所在的具体类型(即方法接收者不固定)。“变量无类型而变量值才有类型”这个特点也是动态类型语言的一个重要特征。

了解了动态和静态类型语言的区别后,也许读者的下一个问题就是动态、静态类型语言两者谁更好,或者谁更加先进?这种比较不会有确切答案,因为它们都有自己的优点,选择哪种语言是需要经过权衡的。静态类型语言在编译期确定类型,最显著的好处是编译器可以提供严谨的类型检查,这样与类型相关的问题能在编码的时候就及时发现,利于稳定性及代码达到更大规模。而动态类型语言在运行期确定类型,这可以为开发人员提供更大的灵活性 ,某些在静态类型语言中需用大量“臃肿”代码来实现的功能,由动态类型语言来实现可能 会更加清晰和简洁,清晰和简洁通常也就意味着开发效率的提升。

#### 2.JDK1.7与动态类型

回到本节的主题,来看看Java语言、虚拟机与动态类型语言之间有什么关系。Java虚拟机毫无疑问是Java语言的运行平台,但它的使命并不仅限于此,早在1997年出版的[《Java虚拟机规范》](https://www.baidu.com/s?wd=《Java虚拟机规范》&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)中就规划了这样一个愿景:“在未来,我们会对Java虚拟机进行适当的扩展,以便更好地支持其他语言运行于Java虚拟机之上”。而目前确实已经有许多动态类型语言运行于Java虚拟机之上了,如Clojure、Groovy、Jython和JRuby等 ,能够在同一个虚拟机上可以达到静态类型语言的严谨性与动态类型语言的灵活性,这是一件很美妙的事情。

但遗憾的是,Java虚拟机层面对动态类型语言的支持一直都有所欠缺,主要表现在方法调用方面:JDK 1.7以前的字节码指令集中,4条方法调用指令(invokevirtual、 invokespecial、invokestatic、 invokeinterface ) 的第一个参数都是被调用的方法的符号引用( CONSTANT_Methodref_info或者CONSTANT_InterfaceMethodref_info常量),前面已经提到过 ,方法的符号引用在编译时产生,而动态类型语言只有在运行时才能确定接收者类型。这样,在Java虚拟机上实现的动态类型语言就不得不使用其他方式(如编译时留个占位符类型 ,运行时动态生成字节码实现具体类型到占位符类型的适配)来实现 ,这样势必让动态类型语言实现的复杂度增加,也可能带来额外的性能或者内存开销。尽管可以利用一些办法(如 Call Site Caching )让这些开销尽量变小,但这种底层问题终归是应当在虛拟机层次上去解决才最合适,因此在Java虚拟机层面上提供动态类型的直接支持就成为了Java平台的发展趋势之一 ,这就是JDK 1.7 ( JSR-292 ) 中invokedynamic指令以及java.lang.invoke包出现的技术背景。

#### 3 .java.lang.invoke包

JDK1 .7实现了JSR-292，新加入的java.lang.invoke包就是JSR-292的一个重要组成部分 , 这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这种方式以外,提供一种新的动态确定目标方法的机制,称为MethodHandle。这种表达方式也许不太好懂?那不妨把MethodHandle与C/C++中的Function Pointer,或者C#里面的Delegate类比一下。举个例子, 如果我们要实现一个带谓词的排序函数,在C/C++中常用的做法是把谓词定义为函数,用函数指针把谓词传递到排序方法,如下 :

```
void sort(int list[],const int size,int(*compare)(int,int))
```

但Java语言做不到这一点,即没有办法单独地把一个函数作为参数进行传递。普遍的做法是设计一个带有compare()方法的Comparator接口 ,以实现了这个接口的对象作为参数, 例如Collections.sort() 就是这样定义的:

```
void sort (List list,Comparator c)
```

不过,在拥有Method Handle之后,Java语言也可以拥有类似于函数指针或者委托的方法别名的工具了。代码清单8-11演示了MethodHandle的基本用途,无论obj是何种类型(临时定义的ClassA抑或是实现PrintStream接口的实现类System.out) ,都可以正确地调用到println()方法。

代码清单8-11 MethodHandle演示

```
import static java.lang.invoke.MethodHandles.lookup;

import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;

/**
 * JSR 292 MethodHandle基础用法演示
 * @author zzm
 */
public class MethodHandleTest {

    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
        // 无论obj最终是哪个实现类，下面这句都能正确调用到println方法。
        getPrintlnMH(obj).invokeExact("icyfenix");
    }

    private static MethodHandle getPrintlnMH(Object reveiver) throws Throwable {
        // MethodType：代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）和具体参数（methodType()第二个及以后的参数）。
        MethodType mt = MethodType.methodType(void.class, String.class);
        // lookup()方法来自于MethodHandles.lookup，这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
        // 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即是this指向的对象，这个参数以前是放在参数列表中进行传递，现在提供了bindTo()方法来完成这件事情。
        return lookup().findVirtual(reveiver.getClass(), "println", mt).bindTo(reveiver);
    }
}
```



实际上,方法getPrintlnMH()中模拟了invokevirtual指令的执行过程,只不过它的分派逻辑并非固化在Class文件的字节码上，而是通过一个具体方法来实现。而这个方法本身的返回值(MethodHandle对象),可以视为对最终调用方法的一个“引用”。以此为基础,有了MethodHandle就可以写出类似于下面这样的函数声明:

```
void sort (List list,MethodHandle compare)
```

从上面的例子可以看出,使用MethodHandle并没有什么困难,不过看完它的用法之后, 读者大概就会产生疑问,相同的事情,用反射不是早就可以实现了吗?

确实 ,仅站在Java语言的角度来看,MethodHandle的使用方法和效果与Reflection有众多相似之处,不过,它们还是有以下这些区别:

从本质上讲,Reflection和MethodHandle机制都是在模拟方法调用,但Reflection是在模拟Java代码层次的方法调用,而MethodHandle是在模拟字节码层次的方法调用。在 MethodHandles.lookup中的3个方法——findStatic ( ) 、 fmdVirtual ( ) 、 fmdSpecial ( ) 正是为了对应于invokestatic、 invokevirtual 、invokeinterface和invokespecial这几条字节码指令的执行权限校验行为,而这些底层细节在使用Reflection API时是不需要关心的。

Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的 java.lang.invoke.MethodHandle对象所包含的信息多。前者是方法在Java一端的全面映像,包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式,还包含执行权限等的运行期信息。而后者仅仅包含与执行该方法相关的信息。用通俗的话来讲,Reflection是重量级 ,而MethodHandle是轻量级。

由于MethodHandle是对字节码的方法指令调用的模拟,所以理论上虚拟机在这方面做的各种优化(如方法内联),在MethodHandle上也应当可以采用类似思路去支持(但目前实现还不完善)。而通过反射去调用方法则不行。

MethodHandle与Reflection除了上面列举的区别外,最关键的一点还在于去掉前面讨论施加的前提“仅站在Java语言的角度来看” : Reflection API的设计目标是只为Java语言服务的, 而MethodHandle则设计成可服务于所有Java虚拟机之上的语言,其中也包括Java语言。

#### 4.invokedynamic指令

本节一开始就提到了JDK 1.7为了更好地支持动态类型语言,引入了第5条方法调用的字节码指令invokedynamic,之后一直没有再提到它,甚至把代码清单8-11中使用MethodHandle的示例代码反编译后也不会看见invokedynamic的身影,它的应用之处在哪里呢?

在某种程度上,invokedynamic指令与MethodHandle机制的作用是一样的,都是为了解决原有4条“invoke*”指令方法分派规则固化在虚拟机之中的问题,把如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之中,让用户(包含其他语言的设计者)有更高的自由度。 而且 ,它们两者的思路也是可类比的,可以把它们想象成为了达成同一个目的,一个采用上层Java代码和API来实现,另一个用字节码和Class中其他属性、常量来完成。因此,如果理解了前面的MethodHandle例 子 ,那么理解invokedynamic指令也并不困难。

每一处含有invokedynamic指令的位置都称做“动态调用点” ( Dynamic Call Site ) , 这条指令的第一个参数不再是代表方法符号引用的CONSTANT_Methodref_info常量 ,而变为JDK 1.7新加入的CONSTANT_InvokeDynamic_info常量,从这个新常量中可以得到3项信息:引导 
方法(Bootstrap Method,此方法存放在新增的BootstrapMethods属性中)、方法类型 ( MethodType ) 和名称。引导方法是有固定的参数,并且返回A是java.langinvoke.CallSite对象 ,这个代表真正要执行的目标方法调用。根据CONSTANT_InvokeDynamic_info常量中提供的信息,虚拟机可以找到并且执行引导方法,从而获得一个CallSite对象,最终调用要执行的目标方法。我们还是举一个实际的例子来解释这个过程,如代码清单8-12所示。

代码清单8-12 invokedynamic指令演示



```
import static java.lang.invoke.MethodHandles.lookup;

import java.lang.invoke.CallSite;
import java.lang.invoke.ConstantCallSite;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

public class InvokeDynamicTest {

    public static void main(String[] args) throws Throwable {
        INDY_BootstrapMethod().invokeExact("icyfenix");
    }

    public static void testMethod(String s) {
        System.out.println("hello String:" + s);
    }

    public static CallSite BootstrapMethod(MethodHandles.Lookup lookup, String name, MethodType mt) throws Throwable {
        return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
    }

    private static MethodType MT_BootstrapMethod() {
        return MethodType
                .fromMethodDescriptorString(
                        "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",
                        null);
    }

    private static MethodHandle MH_BootstrapMethod() throws Throwable {
        return lookup().findStatic(InvokeDynamicTest.class, "BootstrapMethod", MT_BootstrapMethod());
    }

    private static MethodHandle INDY_BootstrapMethod() throws Throwable {
        CallSite cs = (CallSite) MH_BootstrapMethod().invokeWithArguments(lookup(), "testMethod",
                MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
        return cs.dynamicInvoker();
    }
}
```



这段代码与前面MethodHandleTest的作用基本上是一样的,虽然笔者没有加以注释,但是阅读起来应当不困难。本书前面提到过,由于invokedynamic指令所面向的使用者并非Java语言 ,而是其他Java虚拟机之上的动态语言,因此仅依靠Java语言的编译器Javac没有办法生成带有invokedynamic指令的字节码(曾经有一个java.dyn.InvokeDymmic的语法糖可以实现, 但后来被取消了),所以要使用Java语言来演示invokedynamic指令只能用一些变通的办法。John Rose (Da Vinci Machine Project的Leader)编写了一个把程序的字节码转换为使用 invokedynamic的简单工具INDY来完成这件事情,我们要使用这个工具来产生最终要的字节码 ,因此这个示例代码中的方法名称不能随意改动,更不能把几个方法合并到一起写,因为它们是要被INDY工具读取的。

把上面代码编译、再使用INDY转换后重新生成的字节码如代码清单8-13所示 (结果使用javap输出 ,因版面原因,精简了许多无关的内容)。

![img](https://img-blog.csdn.net/20151026000803359)
![img](https://img-blog.csdn.net/20151026000846066)

从main()方法的字节码可见,原本的方法调用指令已经替换为invokedynamic,它的参数为第123项常量(第二个值为0的参数在HotSpot中用不到,与invokeinterface指令那个值为0的参数一样都是占位的)。

```
2 :invokedynamic#123 ,0//InvokeDynamic#0 :testMethod :(Ljava/lang/String ; )V
```

从常量池中可见,第123项常量显示“#123=InvokeDynamic#0 : #121”说明它是一项 CONSTANT_InvokeDynamic_info类型常量,常量值中前面的“#0”代表引导方法取BootstrapMethods属性表的第0项 (javap没备列出属性表的具体内容,不过示例中仅有一个引导方法,即BootstrapMethod() ) , 而后面的“#121”代表引用第121项类型为 CONSTANT_NameAndType_info的常量,从这个常量中可以获取方法名称和描述符,即后面输出的“testMethod : ( Ljava/lang/String ; ) V’。

再看一下BootstrapMethod() ,这个方法Java源码中没有,是INDY产生的,但是它的字节码很容易读懂,所奏逻辑就是调用MethodHandles $Lookup的findStatic ( )方 法 ,产生testMethod ( ) 方法的MethodHandle,然后用它创建一个ConstantCallSite对象。最后,这个对象返回给invokedynamic指令实现对testMethod ( ) 方法的调用,invokedynamic指令的调用过程到此就宣告完成了。

#### 5.掌控方法分派规则

invokedynamic指令与前面4条“invoke*”指令的最大差别就是它的分派逻辑不是由虚拟机决定的 ,而是由程序员决定。在介绍Java虚拟机动态语言支持的最后一个小结中,笔者通过一个简单例子(如代码清单8-14所 示 ),帮助读者理解程序员在可以掌控方法分派规则之后 ,能做什么以前无法做到的事情。

代码清单8 - 14 方法调用问题



```
class GrandFather {
    void thinking() {
        System.out.println("i am grandfather");
    }
}

class Father extends GrandFather {
    void thinking() {
        System.out.println("i am father");
    }
}

class Son extends Father {
    void thinking() {
       // 请读者在这里填入适当的代码（不能修改其他地方的代码）
       // 实现调用祖父类的thinking()方法，打印"i am grandfather"
   }
}
```



在Java程序中 ,可以通过“super”关键字很方便地调用到父类中的方法,但如果要访问祖类的方法呢?读者在阅读本书下面提供的解决方案之前,不妨自己思考一下,在JDK 1.7之 前有没有办法解决这个问题。

在JDK 1.7之前,使用纯粹的Java语言很难处理这个问题(直接生成字节码就很简单,如使用ASM等字节码工具),原因是在Son类的thinking() 方法中无法获取一个实际类型是GrandFather的对象引用,而invokevirtual指令的分派逻辑就是按照方法接收者的实际类型进行分配，这个逻辑是固化在虚拟机中的，程序员无法改变。在JDK 1.7中,可以使用代码清单8- 15中的程序来解决这个问题。

代码清单8-15 使用MethodHandle来解决相关问题



```
import static java.lang.invoke.MethodHandles.lookup;

import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;

class Test {

class GrandFather {
    void thinking() {
        System.out.println("i am grandfather");
    }
}

class Father extends GrandFather {
    void thinking() {
        System.out.println("i am father");
    }
}

class Son extends Father {
     void thinking() {
          try {
                MethodType mt = MethodType.methodType(void.class);
                MethodHandle mh = lookup().findSpecial(GrandFather.class, 
"thinking", mt, getClass());
                mh.invoke(this);
            } catch (Throwable e) {
            }
        }
    }

    public static void main(String[] args) {
        (new Test().new Son()).thinking();
    }
}
```



运行结果:

```
 i'm father
```

## 基于栈的字节码解释执行引擎

许多Java虚拟机的执行引擎在执行Java代码的时候都有解释执行(通过解释器执行)和编译执行(通过即时编译器产生本地代码执行)两种选择,在本章中,我们先来探讨一下在解释执行时,虚拟机执行引擎是如何工作的。

### 解释执行

[Java语言](https://www.baidu.com/s?wd=Java语言&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)经常被人们定位为“解释执行”的语言,在Java初生的JDK 1.0时代 ,这种定义还算是比较准确的,但当主流的虚拟机中都包含了即时编译器后,Class文件中的代码到底会被解释执行还是编译执行,就成了只有虚拟机自己才能准确判断的事情。再后来 ,Java也发展出了可以直接生成本地代码的编译器[如GCJ」(GNU Compiler for the Java )],而C/C++语言也出现了通过解释器执行的版本(如CINT) ,这时候再笼统地说“解释执行”,对于整个 Java语言来说就成了几乎是没有意义的概念,只有确定了谈论对象是某种具体的Java实现版本和执行引擎运行模式时,谈解释执行还是编译执行才会比较确切。

不论是解释还是编译,也不论是物理机还是虚拟机,对于应用程序,机器都不可能如人那样阅读、理解 ,然后就获得了执行能力。大部分的程序代码到物理机的目标代码或虚拟机能执行的指令集之前,都需要经过图8-4中的各个步骤。如果读者对编译原理的相关课程还有印象的话,很容易就会发现图8-4中下面那条分支,就是传统编译原理中程序代码到目标机器代码的生成过程,而中间的那条分支,自然就是解释执行的过程。

如今,基于物理机、Java虚拟机,或者非Java的其他高级语言虚拟机(HLLVM )的语 言 ,大多都会遵循这种基于现代经典编译原理的思路,在执行前先对程序源码进行词法分析和语法分析处理,把源码转化为抽象语法树( Abstract Syntax Tree,AST)。对于一门具体语言的实现来说,词法分析、语法分析以至后面的优化器和目标代码生成器都可以选择独立于执行引擎,形成一个完整意义的编译器去实现,这类代表是C/C++语言。也可以选择把其中一部分步骤(如生成抽象语法树之前的步骤)实现为一个半独立的编译器,这类代表是Java 语言。又或者把这些步骤和执行引擎全部集中封装在一个封闭的黑匣子之中,如大多数的JavaScript执行器。

图8-4 编译过程

![img](https://img-blog.csdn.net/20151026002926434)

Java语言中 ,Javac编译器完成了程序代码经过词法分析、语法分析到抽象语法树,再遍历语法树生成线性的字节码指令流的过程。因为这一部分动作是在Java虚拟机之外进行的, 而解释器在虚拟机的内部,所以Java程序的编译就是半独立的实现。



### 基于栈的指令集与基于寄存器的指令集

Java编译器输出的指令流，基本上是一种基于栈的[指令集架构](https://www.baidu.com/s?wd=指令集架构&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)( Instruction Set Architecture,ISA ) , 指令流中的指令大部分都是零地址指令,它们包赖操作数栈进行工作。与之相对的另外一套常用的指令集架构是基于寄存器的指令集,最典型的就是x86的二地址指令集 ,说得通俗一些,就是现在我们主流PC机中直接支持的指令集架构,这些指令依赖寄存器进行工作。那么 ,基于栈的指令集与基于寄存器的指令集这两者之间有什么不同呢?

举个最简单的例子,分别使用这两种指令集计算“ 1+1”的结果,基于栈的指令集会是这样子的:

```
iconst_1
iconst_1
iadd
istore_0
```

两条iconst_1指令连续把两个常量1压入栈后,iadd指令把栈顶的两个值出栈、相 加 ,然后把结果放回栈顶 ,最后istore_0把栈顶的值放到局部变量表的第0个Slot中。

如果基于寄存器,那程序可能会是这个样子:

```
mov eax ,1 
add eax ,1
```

mov指令把EAX寄存器的值设为1 ,然后add指令再把这个值加1 ,结果就保存在EAX寄存器里面。

了解了基于栈的指令集与基于寄存器的指令集的区别后,读者可能会有进一步的疑问, 这两套指令集谁更好一些呢?

应该这么说,既然两套指令集会同时并存和发展,那肯定是各有优势的,如果有一套指令集全面优于另外一套的话,就不会存在选择的问题了。

基于栈的指令集主要的优点就是可移植,寄存器由硬件直接提供,程序直接依赖这些硬件寄存器则[不可避免](https://www.baidu.com/s?wd=不可避免&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)地要受到硬件的约束。例如 ,现在32位80x86体系的处理器中提供了8 个32位的寄存器,而ARM体系的CPU ( 在当前的手机、PDA中相当流行的一种处理器)则提供了16个32位的通用寄存器。如果使用栈架构的指令集,用户程序不会直接使用这些寄存器 ,就可以由虚拟机实现来自行决定把一些访问最频繁的数据(程序计数器、栈顶缓存等) 放到寄存器中以获取尽量好的性能,这样实现起来也更加简单一些。栈架构的指令集还有一 些其他的优点,如代码相对更加紧凑(字节码中每个字节就对应一条指令,而多地址指令集中还需要存放参数)、编译器实现更加简单(不需要考虑空间分配的问题,所需空间都在栈上操作 ) 等。

栈架构指令集的主要缺点是执行速度相对来说会稍慢一些。所有主流物理机的指令集都是寄存器架构也从侧面印证了这一点。

虽然栈架构指令集的代码非常紧凑,但是完成相同功能所需的指令数量一般会比寄存器架构多,因为出栈、入栈操作本身就产生了相当多的指令数量。更重要的是 ,栈实现在内存之中 ,频繁的栈访问也就意味着频繁的内存访问,相对于处理器来说,内存始终是执行速度的瓶颈。尽管虚拟机可以采取栈顶缓存的手段,把最常用的操作映射到寄存器中避免直接内存访问 ,但这也只能是优化措施而不是解决本质问题的方法。 由于指令数量和内存访问的原因 ,所以导致了栈架构指令集的执行速度会相对较慢。

注： 
部分字节码指令会带有参数,而纯粹基于栈的指令集架构中应当全部都是零地址指令,也就是都不存在显式的参数。Java这样实现主要是考虑了代码的可校验性。

这里说的是物理机器上的寄存器,也有基于寄存器的虚拟机,如Google Android平台的 Dalvik VM。即使是基于寄存器的虚拟机,也希望把虚拟机寄存器尽量映射到物理寄存器上以获取尽可能高的性能。



### 基于栈的解释器执行过程

初步的理论知识已经讲解过了,本节准备了一段Java代码 ,看看在虚拟机中实际是如何执行的。前面曾经举过一个计算“ 1+1”的例子,这样的算术题目显然太过简单了,笔者准备了四则运算的例子,请看代码清单8-16。

![img](https://img-blog.csdn.net/20151026004443937)

从Java语言的角度来看,这段代码没有任何解释的必要,可以直接使用javap命令看看它的字节码指令,如代码清单8-17所示。

![img](https://img-blog.csdn.net/20151026004546804)

javap提示这段代码需要深度为2的操作数栈和4个Slot的局部变量空间,笔者根据这些信息画了图8-5〜图8-11共7张图,用它们来描述代码清单8-17执行过程中的代码、操作数栈和局部变量表的变化情况。

![img](https://img-blog.csdn.net/20151026004639110)

![img](https://img-blog.csdn.net/20151026004717404)

![img](https://img-blog.csdn.net/20151026004751395)

![img](https://img-blog.csdn.net/20151026004818423)

![img](https://img-blog.csdn.net/20151026004854235)

![img](https://img-blog.csdn.net/20151026004922476)

![img](https://img-blog.csdn.net/20151026004951772)

上面的执行过程仅仅是一种概念模型,虚拟机最终会对执行过程做一些优化来提高性能 ,实际的运作过程不一定完全符合概念模型的描述……更准确地说,实际情况会和上面描述的概念模型差距非常大,这种差距产生的原因是虚拟机中解析器和即时编译器都会对输入的字节码进行优化,例如 ,在HotSpot虚拟机中,有很多以“fast_”开头的非标准字节码指令用于合并、替换输入的字节码以提升解释执行性能,而即时编译的优化手段更加花样繁多。

不过 ,我们从这段程序的执行中也可以看出栈结构指令集的一般运行过程,整个运算过程的中间变量都以操作数栈的出栈、入栈为信息交换途径,符合我们在前面分析的特点。

# 类加载及执行子系统的案例与实战

# 概述

在Class文件格式与执行引擎这部分中,用户的程序能直接影响的内容并不太多, Class文件以何种格式存储,类型何时加载、如何连接,以及虚拟机如何执行字节码指令等都是由虚拟机直接控制的行为,用户程序无法对其进行改变。能通过程序进行操作的,主要是字节码生成与类加载器这两部分的功能,但仅仅在如何处理这两点上,就已经出现了许多值得欣赏和借鉴的思路,这些思路后来成为了许多常用功能和程序实现的基础。

# 案例分析

## Tomcat:正统的类加载器架构

主流的Java Web服务器 ,如Tomcat、Jetty、WebLogic、WebSphere或其他笔者没有列举的服务器,都实现了自己定义的类加载器(一般都不止一个)。因为一个功能健全的Web服务器 ,要解决如下几个问题:

- 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以实现相互隔离。这是最基本的需求,两个不同的应用程序可能会依赖同一个第三方类库的不同版本,不能要求一个类库在一个服务器中只有一份,服务器应当保证两个应用程序的类库可以互相独立使用。
- 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以互相共享。这个需求也很常见,例如,用户可能有10个使用Spring组织的应用程序部署在同一台服务器上,如果把10份Spring分别存放在各个应用程序的隔离目录中,将会是很大的资源浪费——这主要倒不是浪费磁盘空间的问题,而是指类库在使用时都要被加载到服务器内存,如果类库不能共享 ,虚拟机的方法区就会很容易出现过度膨胀的风险。
- 服务器需要尽可能地保证自身的安全不受部署的Web应用程序影响。目前,有许多主流的Java Web服务器自身也是使用Java语言来实现的。因此 ,服务器本身也有类库依赖的问题,一般来说,基于安全考虑,服务器所使用的类库应该与应用程序的类库互相独立。
- 支持JSP应用的Web服务器 ,大多数都需要支持HotSwap功能。我们知道,JSP文件最终要编译成Java Class才能由虚拟机执行,但JSP文件由于其纯文本存储的特性,运行时修改的概率远远大于第三方类库或程序自身的Class文件。而且ASP、PHP和JSP这些网页应用也把修改后无须重启作为一个很大的“优势”来看待,因此“主流”的Web服务器都会支持JSP生成类的热替换,当然也有“非主流”的 ,如运行在生产模式( Production Mode ) 下的WebLogic服务器默认就不会处理JSP文件的变化。

由于存在上述问题,在部署Web应用时 ,单独的一个ClassPath就无法满足需求了,所以各种Web服务器都“不约而同”地提供了好几个ClassPath路径供用户存放第三方类库,这些路径一般都以“lib”或“classes”命名。被放置到不同路径中的类库,具备不同的访问范围和服务对象,通常,每一个目录都会有一个相应的自定义类加载器去加载放置在里面的Java类库。 现在 ,笔者就以Tomcat服务器为例,看一看Tomcat具体是如何规划用户类库结构和类加载器的。

在Tomcat目录结构中,有3组目录(“/common/*”、“/server/*”和“/shared/*”)可以存放Java类库,另外还可以加上Web应用程序自身的目录“/WEB-INF/*” ,一共4组 ,把Java类库放 
置在这些目录中的含义分别如下。

- 放置在/common目录中:类库可被Tomcat和所有的Web应用程序共同使用。
- 放置在/server目录中：类库可被Tomcat使用，对所有的Web应用程序都不可见。
- 放置在/shared目录中:类库可被所有的Web应用程序共同使用,但对Tomcat自己不可见。
- 放置在/WebApp/WEB-INF目录中:类库仅仅可以被此Web应用程序使用,对Tomcat和其他Web应用程序都不可见。

为了支持这套目录结构,并对目录里面的类库进行加载和隔离,Tomcat自定义了多个类加载器,这些类加载器按照经典的双亲委派模型来实现,其关系如图9-1所示。

![img](https://img-blog.csdn.net/20151101162452934)

灰色背景的3个类加载器是JDK默认提供的类加载器,这3个加载器的作用在第7章中已经介绍过了。而CommonClassLoader、CatalinaClassLoader、SharedClassLoader和WebappClassLoader则是Tomcat自己定义的类加载器,它们分别加载/common/*、/server/*、 /shared/*和/WebApp/WEB-INF/*中的Java类库。其中WebApp类加载器和Jsp类加载器通常会存在多个实例,每一个Web应用程序对应一个WebApp类加载器,每一个JSP文件对应一个Jsp类加载器。

从图9-1的委派关系可以看出，CommonClassLoader能加载的类都可以被Catalina ClassLoader和SharedClassLoader使用 ,而CatalinaClassLoader和SharedClassLoader自己能加载的类则与对方相互隔离。WebAppClassLoader可以使用SharedClassLoader加载到的类,但各个WebAppClassLoader实例之间相互隔离。而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个Class , 它出现的目的就是为了被丢弃:当服务器检测到JSP文件被修改 
时 ,会替掉目前的JasperLoader的实例,并通过再建立一个新的Jsp类加载器来实现JSP文件的HotSwap功能。

对于Tomcat的6.x版本 ,只有指定了tomcat/conf/catalina.properties配置文件的server.loader和share.loader项后才会真正建立CatalinaClassLoader和SharedClassLoader的实例,否则会用到这两个类加载器的地方会用CommonClassLoader的实例代替,而默认的配置文件中没有设置这两个loader项 ,所以Tomcat 6.x[顺理成章](https://www.baidu.com/s?wd=顺理成章&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)地把/common、/server和/shared三个目录默认合并到一起变成一个/lib目录,这个目录里的类库相当于以前/common目录中类库的作用。这是Tomcat设计团队为了简化大多数的部署场景所做的一项改进,如果默认设置不能满足需要, 用户可以通过修改配置文件指定server.loader和share.loader的方式重新启用Tomcat 5.x的加载器架构。

Tomcat加载器的实现清晰易懂,并且采用了官方推荐的“正统”的使用类加载器的方式。 如果读者阅读完上面的案例后,能完全理解Tomcat设计团队这样布置加载器架构的用意,那说明已经大致掌握了类加载器“主流”的使用方式,那么笔者不妨再提一个问题让读者思考一下 :前面曾经提到过一个场景,如果有10个Web应用程序都是用Spring来进行组织和管理的话 ,可以把Spring放到Common或Shared目录下让这些程序共享。Spring要对用户程序的类进行管理 ,自然要能访问到用户程序的类,而用户的程序显然是放在/WebApp/WEB-INF目录中的 ,那么被CommonClassLoader或SharedClassLoader加载的Spring如何访问弃不在其加载范围内的用户程序呢?如果读过本书第7章的相关内容,相信读者可以很容易地回答这个问题。

Tomcat是Apache基金会中的一款开源的Java Web服务器 ,主页地址为: [http://tomcat.apache.org](http://tomcat.apache.org/)。本案例中选用的是Tomcat 5.x服务器的目录和类加载器结构,在Tomcat6.x的默认配置下,/common、/server和/shared三个目录已经合并到一起了。

## OSGi :灵活的类加载器架构

Java程序社区中流传着这么一个观点:“学习JEE规 范 ,去看JBoss源 码 ;学习类加载器, 就去看OSGi源码”。尽管“JEE规范”和“类加载器的知识”并不是一个对等的概念,不 过 ,既然 这个观点能在程序员中流传开来,也从侧面说明了OSGi对类加载器的运用确实有其独到之 处。

Java程序社区中流传着这么一个观点:“学习JEE规范 ,去看JBoss源码;学习类加载器, 就去看OSGi源码”。尽管“JEE规范”和“类加载器的知识”并不是一个对等的概念,不过 ,既然这个观点能在程序员中流传开来,也从侧面说明了OSGi对类加载器的运用确实有其独到之处。

OSGi ( Open Service Gateway Initiative ) 是OSGi联盟 ( OSGi Alliance ) 制定的一个基于Java语言的动态模块化规范,这个规范最初由Sun、IBM、爰立信等公司联合发起,目的是使服务提供商通过住宅网关为各种家用智能设备提供各种服务,后来这个规范在Java的其他技术领域也有相当不错的发展,现在已经成为Java世界中“事实上”的模块化标准,并且已经有了Equinox、Felix等成熟的实现。OSGi在Java程序员中最著名的应用案例就是Eclipse IDE,另 外还有许多大型的软件平台和中间件服务器都基于或声明将会基于OSGi规范来实现,如IBM Jazz平台、GlassFish服务器、jBossOSGi等。

OSGi中的每个模块(称为Bundle)与普通的Java类库区别并不太大,两者一般都以JAR格式进行封装,并且内部存储的都是Java Package和Class。但是一个Bundle可以声明它所依赖的Java Package(通过Import-Packagel描述),也可以声明它允许导出发布的Java Package(通过Export-Package描述)。在OSGi里面,Bundle之间的依赖关系从传统的上层模块依赖底层模块转变为平级模块之间的依赖(至少外观上如此),而且类库的可见性能得到非常精确的控制,一个模块里只有被Export过的Package才可能由外界访问,其他的Package和Class将会隐藏起来。除了更精确的模块划分和可见性控制外,**引入OSGi的另外一个重要理 
由是**,基于OSGi的程序很可能(只是很可能,并不是一定会)可以实现模块级的热插拔功能 ,当程序升级更新或调试除错时,可以只停用、重新安装然后启用程序的其中一部分,这对企业级程序开发来说是一个非常有诱惑力的特性。

OSGi之所以能有上述“诱人”的特点,要归功于它灵活的类加载器架构。OSGi的Bundle类加载器之间只有规则,没有固定的委派关系。例如 ,某个Bundle声明了一个它依赖的Package,如果有其他Bundle声明发布了这个Package,那么所有对这个Package的类加载动作都会委派给发布它的Bundle类加载器去完成。不涉及某个具体的Package时 ,各个Bundle加载器都是平级关系,只有具体使用某个Package和Class的时候,才会根据Package导入导出定义来构造Bundle间的委派和依赖。

另外,一个Bundle类加载器为其他Bundle提供服务时,会根据Export-Package列表严格控制访问范围。如果一个类存在于Bundle的类库中但是没有被Export,那么这个Bundle的类加载器能找到这个类,但不会提供给其他Bundle使用 ,而且OSGi平台也不会把其他Bundle的类加载请求分配给这个Bundle来办理。

我们可以举一个更具体一些的简单例子,假设存在Bundle A、 Bundle B、 Bundle C三个模 块 ,并且这三个Bundle定义的依赖关系如下。

- Bundle A : 声明发布了packageA,依赖了java.*的包。
- Bundle B : 声明依赖了packageA和packageC,同时也依赖了java.*的包。
- Bundle C : 声明发布了packageC , 依赖了packageA。

那么 ,这三个Bundle之间的类加载器及父类加载器之间的关系如图9-2所示。

![img](https://img-blog.csdn.net/20151101165141828)

由于没有牵扯到具体的OSGi实现 ,所以图9-2中的类加载器都没有指明具体的加载器实现,只是一个体现了加载器之间关系的概念模型,并且只是体现了OSGi中最简单的加载器委派关系。一般来说,在OSGi中,加载一个类可能发生的查找行为和委派关系会比图9-2中 显示的复杂得多,类加载时可能进行的查找规则如下:

- 以java.*开头的类,委派给父类加载器加载。
- 否则 ,委派列表名单内的类,委派给父类加载器加载。
- 否则,Import列表中的类,委派给Export这个类的Bundle的类加载器加载。
- 否则,查找当前Bundle的Classpath,使用自己的类加载器加载。
- 否则,查找是否在自己的Fragment Bundle中,如果是,则委派给Fragment Bundle的类加载器加载。
- 否则,查找Dynamic Import列表的Bundle, 委派给对应Bundle的类加载器加载。
- 否则 ,类查找失败。

从图9-2中还可以看出,在OSGi里面,加载器之间的关系不再是双亲委派模型的树形结构 ,而是已经进一步发展成了一种更为复杂的、运行时才能确定的网状结构。这种网状的类加载器架构在带来更好的灵活性的同时,也可能会产生许多新的隐患。笔者曾经参与过将一个非OSGi的大型系统向Equinox OSGi平台迁移的项目,由于历史原因,代码模块之间的依赖关系错综复杂,勉强分离出各个模块的Bundle后 ,发现在高并发环境下经常出现死锁。我们很容易地找到了死锁的原因:如果出现了Bundle A依赖Bundle B的Package B , 而Bundle B又依赖了Bundle A的Package A , 这两个Bundle进行类加载时就很容易发生死锁。具体情况是当 Bundle A加载Package B的类时,首先需要锁定当前类加载器的实例对象 
(java.lang.ClassLoader.loadClass()是一个synchronized方法),然后把请求委派给Bundle B的加载器处理，如果这时候Bundle B也正好想加载Package A的类,它也先锁定自己的加载器再去请求Bundle A的加载器处理,这样 ,两个加载器都在等待对方处理自己的请求,而对方处理完之前自己又一直处于同步锁定的状态,因此它们就互相死锁,永远无法完成加载请求了。Equinox的Bug List中有关于这类问题的Bug, 也提供了一个以牺牲性能为代价的解决方案—–用户可以用osgi.classloader.singleThreadLoads参数来按单线程串行化的方式强制进行类加载动作。在JDK 1.7中 ,为非树状继承关系下的类加载器架构进行了一次专门的升级,目的是从底层避免这类死锁出现的可能。

总体来说,OSGi描绘了 一个很美好的模块化开发的目标,而且定义了实现这个目标所需要的各种服务,同时也有成熟框架对其提供实现支持。对于单个虚拟机下的应用,从开发初期就建立在OSGi上是一个很不错的选择,这样便于约束依赖。但并非所有的应用都适合采用OSGi作为基础架构,OSGi在提供强大功能的同时,也引入了额外的复杂度,带来了线程死锁和内存泄漏的风险。

## 字节码生成技术与动态代理的实现

“字节码生成”并不是什么高深的技术,读者在看到“字节码生成”这个标题时也先不必去想诸如Javassist、CGLib、ASM之类的字节码类库,因为JDK里面的javac命令就是字节码生成技术的“老祖宗” ,并且javac也是一个由Java语言写成的程序,它的代码存放在OpenJDK的langtools/src/share/classes/com/sun/tools/javac目录中。要深入了解字节码生成,阅读javac的源码是个很好的途径,不过javac对于我们这个例子来说太过庞大了。在Java里面除了javac和字节码类库外,使用字节码生成的例子还有很多,如Web服务器中的JSP编译器 ,编译时植入的AOP框架 ,还有很常用的动态代理技术,甚至在使用反射的时候虚拟机都有可能会在运行时生成字节码来提高执行速度。我们选择其中相对简单的动态代理来看看字节码生成技术是如何影响程序运作的。

相信许多Java开发人员都使用过动态代理,即使没有直接使用过Java.lang.reflect.Proxy或实现过java.lang.reflect.InvocationHandler接口 ,应该也用过Spring来做过Bean的组织管理。如果使用过Spring , 那大多数情况都会用过动态代理,因为如果Bean是面向接口编程,那么在 Spring内部都是通过动态代理的方式来对Bean进行增强的。动态代理中所谓的“动态”,是针对使用Java代码实际编写了代理类的“静态”代理而言的,它的优势不在于省去了编写代理类那一点工作量,而是实现了可以在原始类和接口还未知的时候,就确定代理类的代理行为, 当代理类与原始类脱离直接联系后,就可以很灵活地重用于不同的应用场景之中。

代码清单9-1演示了一个最简单的动态代理的用法,原始的逻辑是打印一句“hello world” ,代理类的逻辑是在原始类方法执行前打印一句“welcome”。我们先看一下代码,然后再分析JDK是如何做到的。

代码清单9-1 动态代理的简单示例

```
public class DynamicProxyTest {

    interface IHello {
        void sayHello();
    }

    static class Hello implements IHello {
        @Override
        public void sayHello() {
            System.out.println("hello world");
        }
    }

    static class DynamicProxy implements InvocationHandler {

        Object originalObj;

        Object bind(Object originalObj) {
            this.originalObj = originalObj;
            return Proxy.newProxyInstance(originalObj.getClass().getClassLoader(), originalObj.getClass().getInterfaces(), this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("welcome");
            return method.invoke(originalObj, args);
        }
    }

    public static void main(String[] args) {
        IHello hello = (IHello) new DynamicProxy().bind(new Hello());
        hello.sayHello();
    }
}
```

运行结果如下:

```
welcome 
hello world 
```

上述代码里,唯一的“黑厘子”就是Proxy.newProxyInstance()方法,除此之外再没有任何特殊之处。这个方法返回一个实现了IHello的接口,并且代理了new Hello()实例行为的对象。跟踪这个方法的源码,可以看到程序进行了验证、优化、缓存、同步、生成字节码、显式类加载等操作，前面的步骤并不是我们关注的重点,而最后它调用了sun.misc.ProxyGenerator.generateProxyClass()方法来完成生成字节码的动作,这个方法可以在运行时产生一个描述代理类的字节码byte []数组。如果想看一看这个在运行时产生的代理类中写了些什么 ,可以在main()方法中加入下面这句:

```
System.getProperties().put ("sun.misc.ProxyGenerator.saveGeneratedFiles" ,"true") 
```

加入这句代码后再次运行程序,磁盘中将会产生一个名为“$Proxy0.class”的代理类Class文件 ,反编译后可以看见如代码清单9-2所示的内容。

加入这句代码后再次运行程序,磁盘中将会产生一个名为“$Proxy0.class”的代理类Class文件 ,反编译后可以看见如代码清单9-2所示的内容。

代码清单9-2 反编译的动态代理类的代码

```


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy
  implements DynamicProxyTest.IHello
{
  private static Method m3;
  private static Method m1;
  private static Method m0;
  private static Method m2;

  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final void sayHello()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  // 此处由于版面原因，省略equals()、hashCode()、toString()三个方法的代码
  // 这3个方法的内容与sayHello()非常相似。

  static
  {
    try
    {
      m3 = Class.forName("org.fenixsoft.bytecode.DynamicProxyTest$IHello").getMethod("sayHello", new Class[0]);
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
} 
```

这个代理类的实现代码也很简单,它为传入接口中的每一个方法,以及从 java.lang.Object中继承来的equals()、hashCode()、toString()方法都生成了对应的实现 ,并且统一调用了InvocationHandler对象的invoke()方法(代码中的“this.h”就是父类Proxy中保存的InvocationHandler实例变量)来实现这些方法的内容,各个方法的区别不过是传入的参数和Method对象有所不同而已,所以无论调用动态代理的哪一个方法,实际上都是在执行InvocationHandler.invoke()中的代理逻辑。

这个例子中并没有讲到generateProxyClass()方法具体是如何产生代理 
类“$Proxy0.class”的字节码的,大致的生成过程其实就是根据Class文件的格式规范去拼装字节码 ,但在实际开发中,以byte为单位直接拼装出字节码的应用场合很少见,这种生成方式也只能产生一些高度模板化的代码。对于用户的程序代码来说,如果有要大量操作字节码的需求，还是使用封装好的字节码类库比较合适。如果读者对动态代理的字节码拼装过程很感兴趣 ,可以在OpenJDK的jdk/src/share/classes/sun/misc目录下找到sun.misc.ProxyGenerator的源 
码。

## Retrotranslator : 跨越JDK版本

一般来说,以“做项目”为主的软件公司比较容易更新技术,在下一个项目中换一个技术框架、升级到最新的JDK版本 ,甚至把Java换成C#、C++来开发程序都是有可能的。但当公司发展壮大,技术有所积累,逐渐成为以“做产品”为主的软件公司后,自主选择技术的权利就会丧失掉,因为之前所积累的代码和技术都是用真金白银换来的,一个稳健的团队也不会随意地改变底层的技术。然而在飞速发展的程序设计领域,新技术总是日新月异、层出不穷 ,偏偏这些新技术又如鲜花之于蜜蜂一样,对程序员散发着天然的吸引力。

在Java世界里,每一次JDK大版本的发布,都伴随着一场大规模的技术革新,而对Java程序编写习惯改变最大的,无疑是JDK 1.5的发布。自动装箱、泛型、动态注解、枚举、变长参数、遍历循环(foreach循环)……事实上,在没有这些语法特性的年代,Java程序也照样能写,但是现在看来,上述每一种语法的改进几乎都是“必不可少”的。就如同习惯了24寸 液晶显示器的程序员,很难习惯在15寸纯平显示器上编写代码。但假如“不幸”因为要保护现有投资、维持程序结构稳定等,必须使用1.5以前版本的JDK呢 ?我们没有办法把15寸显示器变成24寸的,但却可以跨越JDK版本之间的沟壑,把JDK 1.5中编写的代码放到JDK 1.4或1.3 的环境中去部署使用。为了解决这个问题,一种名为“Java逆向移植”的工具( Java Backporting Tools ) 应运而生,Retrotranslator是这类工具中较出色的一个。

Retrotranslator的作用是将JDK 1.5编译出来的Class文件转变为可以在JDK 1.4或1.3上部署的版本 ,它可以很好地支持自动装箱、泛型、动态注解、枚举、变长参数、遍历循环、静态导入这些语法特性,甚至还可以支持JDK 1.5中新增的集合改进、并发包以及对泛型、注解等的反射操作。了解了Retrotranslator这种逆向移植工具可以做什么以后,现在关心的是它是怎样做到的?

要想知道Retrotranslator如何在旧版本JDK中模拟新版本JDK的功能 ,首先要弄清楚JDK 升级中会提供哪些新的功能。JDK每次升级新增的功能大致可以分为以下4类 :

- 在编译器层面做的改进。如自动装箱拆箱,实际上就是编译器在程序中使用到包装对象的地方自动插入了很多Integer.valueOf() 、Float.valueOf() 之类的代码;变长参数在编译之后就自动转化成了一个数组来完成参数传递;泛型的信息则在编译阶段就已经擦除掉了(但是在元数据中还保留着),相应的地方被编译器自动插入了类型转换代码。
- 对Java API的代码增强。譬如JDK 1.2时代引入的java.util.Collections等一系列集合类,在JDK 1.5时代引入的java.util.concurrent并发包等。
- 需要在字节码中进行支持的改动。如JDK 1.7里面新加入的语法特性:动态语言支持, 就需要在虚拟机中新增一条invokedynamic字节码指令来实现相关的调用功能。不过字节码指令集一直处于相对比较稳定的状态,这种需要在字节码层面直接进行的改动是比较少见的。
- 虚拟机内部的改进。如JDK 1.5中实现的JSR-133规范重新定义的Java内存模型(Java Memory Model,JMM)、CMS收集器之类的改动,这类改动对于程序员编写代码基本是透明的 ,但会对程序运行时产生影响。

上述4类新功能中,Retrotranslator只能模拟前两类,对于后面两类直接在虚拟机内部实现的改进,一般所有的逆向移植工具都是无能为力的,至少不能完整地或者在可接受的效率上完成全部模拟，否则虚拟机设计团队也没有必要[舍近求远](https://www.baidu.com/s?wd=舍近求远&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)地改动处于JDK底层的虚拟机。 在可以模拟的两类功能中,第二类模拟相对更容易实现一些,如JDK 1.5引入的 java.util.concurrent包 ,实际是由多线程大师Doug Lea开发的一套并发包,在JDK 1.5出现之前就已经存在(那时候名字叫做dl.util.concurrent, 引入JDK时由作者和JDK开发团队共同做了一些改进),所以要在旧的JDK中支持这部分功能,以独立类库的方式便可实现。 Retrotranslator中附带了一个名叫“backport-util-concurrent.jar”的类库来代替JDK 1.5的并发包。

至于JDK在编译阶段进行处理的那些改进,Retrotranslator则是使用ASM框架直接对字节码进行处理。由于组成Class文件的字节码指令数量并没有改变,所以无论是JDK 1.3、 JDK 1.4还是JDK 1.5,能用字节码表达的语义范围应该是一致的。当然,肯定不可能简单地把Class的文件版本号从49.0改回48.0就能解决问题了,虽然字节码指令的数量没有变化,但是元数据信息和一些语法支持的内容还是要做相应的修改。以枚举为例,在JDK 1.5中增加了enum关键字 ,但是Class文件常量池的CONSTANT_Class_info类型常量并没有发生任何语义变化 ,仍然是代表一个类或接口的符号引用,没有加入枚举,也没有增加 过“CONSTANT_Enum_info”之类的“枚举符号引用”常量。所以使用enum关键字定义常量,虽然从Java语法上看起来与使用class关键字定义类、使用interface关键字定义接口是同一层次的 ,但实际上这是由Javac编译器做出来的假象,从字节码的角度来看,枚举仅仅是一个继承于java.lang.Enum、自动生成了values()和valueOf()方法的普通类而已。

Retrotranslator对枚举所做的主要处理就是把枚举类的父类从“java.lang.Enum”替换为它运行时类库中包含的“net.sf.retrotranslator.runtime.java.lang.Enum_”,然后再在类和字段的访问标志中抹去ACC_ENUM标志位。当然 ,这只是处理的总体思路,具体的实现要比上面说的复杂得多。可以想象既然两个父类实现都不一样, values ( ) 和valueOf( )的方法自然需要重写 ,常量池需要引入大量新的来自父类的符号引用,这些都是实现细节。图9-3是一个使用JDK 1.5编译的枚举类与被Retrotranslator转换处理后的字节码的对比图。

![img](https://img-blog.csdn.net/20151101191436012)

# 实战:自己动手实现远程执行功能

不知道读者在做程序维护的时候是否遇到过这类情形:排查问题的过程中,想查看内存中的一些参数值,却又没有方法把这些值输出到界面或日志中,又或者定位到某个缓存数据有问题 ,但缺少缓存的统一管理界面,不得不重启服务才能清理这个缓存。类似的需求有一个共同的特点,那就是只要在服务中执行一段程序代码,就可以定位或排除问题,但就是偏偏找不到可以让服务器执行临时代码的途径,这时候就会希望Java服务器中也有提供类似Groovy Console的功能。

JDK 1.6之后提供了Compiler API , 可以动态地编译Java程序,虽然这样达不到动态语言的灵活度,但让服务器执行临时代码的需求就可以得到解决了。在JDK 1.6之前 ,也可以通过其他方式来做到,譬如写一个JSP文件上传到服务器,然后在浏览器中运行它,或者在服务端程序中加入一个BeanShell Script、JavaScript等的执行引擎(如Mozilla Rhino)去执行动态脚本。在本章的实战部分,我们将使用前面学到的关于类加载及虚拟机执行子系统的知识去实现在服务端执行临时代码的功能。

## 目标

首先 ,在实现“在服务端执行临时代码”这个需求之前,先来明确一下本次实战的具体目标 ,我们希望最终的产品是这样的:

- 不依赖JDK版本,能在目前还普遍使用的JDK中部署,也就是使用JDK1.4〜JDK1.7都可以运行。
- 不改变原有服务端程序的部署,不依赖任何第三方类库。
- 不侵入原有程序,即无须改动原程序的任何代码,也不会对原有程序的运行带来任何影响。
- 考到BeanShell Script或JavaScript等脚本编写起来不太方便,“临时代码”需要直接支持Java语言。
- “临时代码”应当具备足够的自由度,不需要依赖特定的类或实现特定的接口。这里写的 是“不需要”而不是“不可以” ,当“临时代码”需要引用其他类库时也没有限制,只要服务端程序能使用的,临时代码应当都能直接引用。
- “临时代码” 的执行结果能返回到客户端,执行结果可以包括程序中输出的信息及拋出的异常等。

看完上面列出的目标,你觉得完成这个需求需要做多少工作呢?也许答案比大多数人所想的都要简单一些:5个类 ,250行代码(含注释),大约一个半小时左右的开发时间就可以了 ,现在就开始编写程序吧!

## 思路

在程序实现的过程中,我们需要解决以下3个问题:

- 如何编译提交到服务器的Java代码?
- 如何执行编译之后的Java代码?
- 如何收集Java代码的执行结果?

对于第一个问题,我们有两种思路可以选择,一种是使用tools.jar包 (在SunJDK/lib目录下)中的com.sun.tools.javac.Main类来编译Java文件,这其实和使用Javac命令编译是一样的。 这种思路的缺点是引入了额外的JAR包 ,而且把程序“綁死”在Sun的JDK上了,要部署到其他公司的JDK中还得把tools.jar带上(虽然JRockit和J9虚拟机也有这个JAR包,但它总不是标准所规定必须存在的)。另外一种思路是直接在客户端编译好,把字节码而不是Java代码传到服务端,这听起来好像有点[投机取巧](https://www.baidu.com/s?wd=投机取巧&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd),一般来说确实不应该假定客户端一定具有编译代码的能力,但是既然程序员会写Java代码去给服务端排查问题,那么很难想象他的机器上会连编译Java程序的环境都没有。

对于第二个问题,简单地一想:要执行编译后的Java代码,让类加载器加载这个类生成一个Class对象 ,然后反射调用一下某个方法就可以了(因为不实现任何接口,我们可以借用一下Java中人人皆知的“main() ”方法)。但我们还应该考虑得更周全些:一段程序往往不是编写、运行一次就能达到效果,同一个类可能要反复地修改、提交、执行。另外,提交上去的类要能访问服务端的其他类库才行。还有 ,既然提交的是临时代码,那提交的Java类在执行完后就应当能卸载和回收。

最后的一个问题,我们想把程序往标准输出(Systemout)和标准错误输出( Systemerr ) 中打印的信息收集起来,但标准输出设备是整个虚拟机进程全局共享的资源,如桌使用System.setOut()/System.setErr()方法把输出流重定向到自己定义的PrintStream对象上固然可以收集输出信息,但也会对原有程序产生影响:会把其他线程向标准输出中打印的信息也收集了。虽然这些并不是不能解决的问题,不过为了达到完全不影响原程序的目的 ,我们可以采用另外一种办法,即直接在执行的类中把对System.out的符号引用替换为我们准备的PnntStream的符号引用,依赖前面学习的知识,做到这一点并不困难。

## 实现

在程序实现部分,我们主要看一下代码及其注释。首先看看实现过程中需要用到的4个支持类。第一个类用于实现“同一个类的代码可以被多次加载”这个需求,即用于解决【目标】中列举的第2个问题的HotSwapClassLoader,具体程序如代码清单9-3所示。

代码清单9-3 HotSwapClassLoader的实现

```
/**
 * 为了多次载入执行类而加入的加载器<br>
 * 把defineClass方法开放出来，只有外部显式调用的时候才会使用到loadByte方法
 * 由虚拟机调用时，仍然按照原有的双亲委派规则使用loadClass方法进行类加载
 *
 */
public class HotSwapClassLoader extends ClassLoader {

    public HotSwapClassLoader() {
        super(HotSwapClassLoader.class.getClassLoader());
    }

    public Class loadByte(byte[] classByte) {
        return defineClass(null, classByte, 0, classByte.length);
    }

} 
```

HotSwapClassLoader所做的事情仅仅是公开父类(即java.lang.ClassLoader ) 中的protected方法defineClass() ,我们将会使用这个方法把提交执行的Java类的byte[]数组转变为Class对象。HotSwapClassLoader中并没有重写loadClass() 或findClass() 方法 ,因此如果不算外部手工调用loadByte() 方法的话,这个类加载器的类查找范围与它的父类加载器是完全一致的,在被虚拟机调用时,它会按照双亲委派模型交给父类加载。构造函数中指定为加载HotSwapClassLoader类的类加载器也为父类加载器,这一步是实现提交的执行代码可以访问服务端引用类库的关键,下面我们来看看代码清单9-3。

第二个类是实现将java.lang.System替换为我们自己定义的HackSystem类的过程,它直接修改符合Class文件格式的byte[]数组中的常量池部分,将常量池中指定内容的 CONSTANT_UtfB_info常量替换为新的字符串,具体代码如代码清单9-4所示。 ClassModifier中涉及对byte[]数组操作的部分,主要是将byte[]与int和String互相转换,以及把对byte[]数据的替换操作封装在代码清单9-5所示的ByteUtils中。

代码清单9-4 ClassModifier的实现

```
/**
 * 修改Class文件，暂时只提供修改常量池常量的功能
 * @author zzm 
 */
public class ClassModifier {

    /**
     * Class文件中常量池的起始偏移
     */
    private static final int CONSTANT_POOL_COUNT_INDEX = 8;

    /**
     * CONSTANT_Utf8_info常量的tag标志
     */
    private static final int CONSTANT_Utf8_info = 1;

    /**
     * 常量池中11种常量所占的长度，CONSTANT_Utf8_info型常量除外，因为它不是定长的
     */
    private static final int[] CONSTANT_ITEM_LENGTH = { -1, -1, -1, 5, 5, 9, 9, 3, 3, 5, 5, 5, 5 };

    private static final int u1 = 1;
    private static final int u2 = 2;

    private byte[] classByte;

    public ClassModifier(byte[] classByte) {
        this.classByte = classByte;
    }

    /**
     * 修改常量池中CONSTANT_Utf8_info常量的内容
     * @param oldStr 修改前的字符串
     * @param newStr 修改后的字符串
     * @return 修改结果
     */
    public byte[] modifyUTF8Constant(String oldStr, String newStr) {
        int cpc = getConstantPoolCount();
        int offset = CONSTANT_POOL_COUNT_INDEX + u2;
        for (int i = 0; i < cpc; i++) {
            int tag = ByteUtils.bytes2Int(classByte, offset, u1);
            if (tag == CONSTANT_Utf8_info) {
                int len = ByteUtils.bytes2Int(classByte, offset + u1, u2);
                offset += (u1 + u2);
                String str = ByteUtils.bytes2String(classByte, offset, len);
                if (str.equalsIgnoreCase(oldStr)) {
                    byte[] strBytes = ByteUtils.string2Bytes(newStr);
                    byte[] strLen = ByteUtils.int2Bytes(newStr.length(), u2);
                    classByte = ByteUtils.bytesReplace(classByte, offset - u2, u2, strLen);
                    classByte = ByteUtils.bytesReplace(classByte, offset, len, strBytes);
                    return classByte;
                } else {
                    offset += len;
                }
            } else {
                offset += CONSTANT_ITEM_LENGTH[tag];
            }
        }
        return classByte;
    }

    /**
     * 获取常量池中常量的数量
     * @return 常量池数量
     */
    public int getConstantPoolCount() {
        return ByteUtils.bytes2Int(classByte, CONSTANT_POOL_COUNT_INDEX, u2);
    }
} 
```

代码清单9-5 ByteUtils的实现

```
/**
 * Bytes数组处理工具
 * @author
 */
public class ByteUtils {

    public static int bytes2Int(byte[] b, int start, int len) {
        int sum = 0;
        int end = start + len;
        for (int i = start; i < end; i++) {
            int n = ((int) b[i]) & 0xff;
            n <<= (--len) * 8;
            sum = n + sum;
        }
        return sum;
    }

    public static byte[] int2Bytes(int value, int len) {
        byte[] b = new byte[len];
        for (int i = 0; i < len; i++) {
            b[len - i - 1] = (byte) ((value >> 8 * i) & 0xff);
        }
        return b;
    }

    public static String bytes2String(byte[] b, int start, int len) {
        return new String(b, start, len);
    }

    public static byte[] string2Bytes(String str) {
        return str.getBytes();
    }

    public static byte[] bytesReplace(byte[] originalBytes, int offset, int len, byte[] replaceBytes) {
        byte[] newBytes = new byte[originalBytes.length + (replaceBytes.length - len)];
        System.arraycopy(originalBytes, 0, newBytes, 0, offset);
        System.arraycopy(replaceBytes, 0, newBytes, offset, replaceBytes.length);
        System.arraycopy(originalBytes, offset + len, newBytes, offset + replaceBytes.length, originalBytes.length - offset - len);
        return newBytes;
    }
}
```

经过ClassModifier处理后的byte[]数组才会传给HotSwapClassLoader.loadByte()方法进行类加载,byte[]数组在这里替换符号引用之后,与客户端直接在Java代码中引用HackSystem类再编译生成的Class是完全一样的。这样的实现既避免了客户端编写临时执行代码时要依赖特定的类(不然无法引入HackSystem) ,又避免了服务端修改标准输出后影响到其他程序的 输出。下面我们来看看代码清单9-4和代码清单9-5。

```
/**
 * 为JavaClass劫持java.lang.System提供支持
 * 除了out和err外，其余的都直接转发给System处理
 * 
 * @author zzm
 */
public class HackSystem {

    public final static InputStream in = System.in;

    private static ByteArrayOutputStream buffer = new ByteArrayOutputStream();

    public final static PrintStream out = new PrintStream(buffer);

    public final static PrintStream err = out;

    public static String getBufferString() {
        return buffer.toString();
    }

    public static void clearBuffer() {
        buffer.reset();
    }

    public static void setSecurityManager(final SecurityManager s) {
        System.setSecurityManager(s);
    }

    public static SecurityManager getSecurityManager() {
        return System.getSecurityManager();
    }

    public static long currentTimeMillis() {
        return System.currentTimeMillis();
    }

    public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length) {
        System.arraycopy(src, srcPos, dest, destPos, length);
    }

    public static int identityHashCode(Object x) {
        return System.identityHashCode(x);
    }

    // 下面所有的方法都与java.lang.System的名称一样
    // 实现都是字节转调System的对应方法
    // 因版面原因，省略了其他方法
} 
```



至此, 4个支持类已经讲解完毕,我们来看看最后一个类JavaClassExecuter , 它是提供给外部调用的入口,调用前面几个支持类组装逻辑,完成类加载工作。JavaClassExecuter只有一个execute()方法,用输入的符合Class文件格式的byte[]数组替换java.lang.System的符号引用后,使用HotSwapClassLoader加载生成一个Class对象,由于每次执行execute()方法都会生成一个新的类加载器实例,因此同一个类可以实现重复加载。然后,反射调用这个Class对象的main()方法,如果期间出现任何异常,将异常信息打印到HackSystemout中,最后把缓冲区中的信息、作为方法的结果返回。JavaClassExecuter的实现代码如代运清单9- 7所示。

代码清单9-7 JavaClassExecuter的实现



```
/**
 * JavaClass执行工具
 *
 * @author zzm
 */
public class JavaClassExecuter {

    /**
     * 执行外部传过来的代表一个Java类的Byte数组<br>
     * 将输入类的byte数组中代表java.lang.System的CONSTANT_Utf8_info常量修改为劫持后的HackSystem类
     * 执行方法为该类的static main(String[] args)方法，输出结果为该类向System.out/err输出的信息
     * @param classByte 代表一个Java类的Byte数组
     * @return 执行结果
     */
    public static String execute(byte[] classByte) {
        HackSystem.clearBuffer();
        ClassModifier cm = new ClassModifier(classByte);
        byte[] modiBytes = cm.modifyUTF8Constant("java/lang/System", "org/fenixsoft/classloading/execute/HackSystem");
        HotSwapClassLoader loader = new HotSwapClassLoader();
        Class clazz = loader.loadByte(modiBytes);
        try {
            Method method = clazz.getMethod("main", new Class[] { String[].class });
            method.invoke(null, new String[] { null });
        } catch (Throwable e) {
            e.printStackTrace(HackSystem.out);
        }
        return HackSystem.getBufferString();
    }
} 
```



## 验证

远程执行功能的编码到此就完成了,接下来就要检验一下我们的劳动成果了。如果只是测试的话,那么可以任意写一个Java类 ,内容无所谓,只要向System.out输出信息即可,取名为TestClass, 同时放到服务器C盘的根目录中。然后,建立一个JSP文件并加入如代码清单9- 8所示的内容,就可以在浏览器中看到这个类的运行结果了。

```
<%@ page import="java.lang.*" %>
<%@ page import="java.io.*" %>
<%@ page import="org.fenixsoft.classloading.execute.*" %>
<%
    InputStream is = new FileInputStream("c:/TestClass.class");
    byte[] b = new byte[is.available()];
    is.read(b);
    is.close();

    out.println("<textarea style='width:1000;height=800'>");
    out.println(JavaClassExecuter.execute(b));
    out.println("</textarea>"); 
%> 
```

当然 ,上面的做法只是用于测试和演示,实际使用这个JavaExecuter执行器的时候,如果还要手工复制一个Class文件到服务器上就没有什么意义了。笔者给这个执行器写了一个“外壳”,是一个Eclipse插件 ,可以把Java文件编译后传输到服务器中,然后把执行器的返回结果输到Eclipse的Console窗口里,这样就可以在有灵感的时候随时写几行调试代码, 放到测试环境的服务器上立即运行了。虽然实现简单,但效果很不错,对调试问题也非常有用 ,如图9-4所示。

![img](https://img-blog.csdn.net/20151101235258909)

# Java编译器

## 概述

Java编译器分为**前端编译器、JIT 编译器和AOT编译器**。

| 编译器                            |                  |                                                    |
| --------------------------------- | ---------------- | -------------------------------------------------- |
| 前端编译器                        | 源代码到字节码   | Sun的Javac、 Eclipse JDT中的增量式编译器( ECJ )    |
| JIT编译器(Just In Time Compiler)  | 从字节码到机器码 | HotSpotVM的C1、C2编译器。                          |
| AOT编译器(Ahead Of Time Compiler) | 源代码到机器码   | GNU Compiler for the Java ( GCJ ) 、 Excelsior JET |

### 前端编译器

在 JDK 的安装目录里有一个 javac 工具，就是它将 Java 代码翻译成字节码，这个工具我们叫做编译器。相对于后面要讲的其他编译器，其因为处于编译的前期，因此又被成为前端编译器。

通过 javac 编译器，我们可以很方便地将 java 源文件翻译成字节码文件。就拿 Hello World 作为例子：

```
public class Demo{
   public static void main(String args[]){
		System.out.println("Hello World!");
   }
}
```

我们使用 javac 命令编译上面这个类，便会生成一个 Demo.class 文件

```
javac Demo.java
```

打开 Demo.class 文件，我们会发现是一连串的 16 进制数据流。

运行 javac 命令的过程，其实就是 javac 编译器解析 Java 源代码，并生成字节码文件的过程。

### JIT 编译器

当源代码转化为字节码之后，其实要运行程序，有两种选择。一种是使用 Java 解释器解释执行字节码，另一种则是使用 JIT 编译器将字节码转化为本地机器代码。

当 JIT 编译器完成第一次编译后，其会将字节码对应的机器码保存下来，下次可以直接使用。

在 HotSpot 虚拟机内置了两个即时编译器，分别称为 Client Compiler 和Server Compiler。这两种不同的编译器衍生出两种不同的编译模式，我们分别称之为：C1 编译模式，C2 编译模式。

C1 编译模式会将字节码编译为本地代码，进行简单、可靠的优化，如有必要将加入性能监控的逻辑。而 C2 编译模式，也是将字节码编译为本地代码，但是会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化。

实际上对于 HotSpot 虚拟机来说，其一共有三种运行模式可选，分别是：

- 混合模式（Mixed Mode） 。即 C1 和 C2 两种模式混合起来使用，这是默认的运行模式。如果你想单独使用 C1 模式或 C2 模式，使用 `-client` 或 `-server` 打开即可。
- 解释模式（Interpreted Mode）。即所有代码都解释执行，使用 `-Xint` 参数可以打开这个模式。
- 编译模式（Compiled Mode）。 此模式优先采用编译，但是无法编译时也会解释执行，使用 `-Xcomp` 打开这种模式。

在命令行中输入 `java -version` 可以看到，我机器上的虚拟机使用 Mixed Mode 运行模式。

```
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
```

### AOT 编译器

AOT 编译器的基本思想是：在程序执行前生成 Java 方法的本地代码，以便在程序运行时直接使用本地代码。

但是 Java 语言本身的动态特性带来了额外的复杂性，影响了 Java 程序静态编译代码的质量。例如 Java 语言中的动态类加载，因为 AOT 是在程序运行前编译的，所以无法获知这一信息，所以会导致一些问题的产生。

### 总结

前端编译器，最常见的就是我们的 javac 编译器，其将 Java 源代码编译为 Java 字节码文件。JIT 即时编译器，最常见的是 HotSpot 虚拟机中的 Client Compiler 和 Server Compiler，其将 Java 字节码编译为本地机器代码。而 AOT 编译器则能将源代码直接编译为本地机器码。这三种编译器的编译速度和编译质量如下：

- 编译速度上，解释执行 > AOT 编译器 > JIT 编译器。
- 编译质量上，JIT 编译器 > AOT 编译器 > 解释执行。

而在 JVM 中，通过这几种不同方式的配合，使得 JVM 的编译质量和运行速度达到最优的状态。

## Javac编译器

官网编译流程：http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html

javac目录说明:http://openjdk.java.net/groups/compiler/doc/package-overview/index.html

编译过程类名：http://openjdk.java.net/groups/compiler/doc/hhgtjavac/index.html

参考资料：https://www.zhihu.com/column/c_1326646811726356480

书籍：深入解析Java编译器：源码剖析与实例详解-马智

### maven依赖

```
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.8.0</version>
    <scope>system</scope>
    <systemPath>${JAVA_HOME}/lib/tools.jar</systemPath>
    // <systemPath>${env.JAVA_HOME}/lib/tools.jar</systemPath>
</dependency>

```

### 运行编译方法

运行**com.sun.tools.javac.Main**的main()方法来执行编译与命令行中使用Javac的命令没有什么区别

```
public class ComplierTest {

    public static void main(String[] args) throws Exception {
        com.sun.tools.javac.Main.main(args);
    }
}
```

![JVM-javac-flow](png\JVM\JVM-javac-flow.png)

javac编译一般可以分为三个阶段。这个过程由`com.sun.tools.javac.main.JavaCompiler` 类处理。

- 解析与填充符号表的过程。
- 插入式注解处理器的注解处理过程。
- 分析与字节码生成过程。

### javac源码

#### 包概述

![JVM_javac_package](png\JVM\JVM_javac_package.png)

#### 主要方法概述

```
com.sun.tools.javac.main.JavaCompiler#initProcessAnnotations 准备过程：初始化插入式注解处理器
com.sun.tools.javac.main.JavaCompiler#parseFiles  词法分析 语法分析
com.sun.tools.javac.main.JavaCompiler#enterTrees 输出到符号表
com.sun.tools.javac.main.JavaCompiler#processAnnotations 执行注解处理
com.sun.tools.javac.main.JavaCompiler#compile2  分析以及字节码生成
com.sun.tools.javac.main.JavaCompiler#attribute 标注
com.sun.tools.javac.main.JavaCompiler#flow 数据流处理
com.sun.tools.javac.main.JavaCompiler#desugar 解语法糖
com.sun.tools.javac.main.JavaCompiler#generate 生成字节码
```

### 解析与填充符号表

解析步骤由parseFiles()方法完成,解析步骤包括了经典程序编译原理中的词法分析和语法分析两个过程。

#### 词法、语法分析

**词法分析**是将源代码的字符流转变为标记(Token)集合。关键字、变量名、字面量、运算符都可以成为标记。

词法分析过程由com.sun.tools.javac.parser.Scanner类来实现。

如“int a=b+2”这句代码包含了6个标记,分别是int、a、=、b、+、2 ,虽然关键字int由 3个字符构成,但是它只是一个Token,不可再拆分。

**语法分析**是根据Token序列构造抽象语法树的过程。抽象语法树( Abstract Syntax Tree,AST ) 是一种用来描述程序代码语法结构的树形表示方式,语法树的每一个节点都代表着程序代码中的一个语法结构( Construct ) ,例如包、类型、修饰符、运算符、接口、返回值甚至代码注释等都可以是一个语法结构。

语法分析过程由 com.sun.tools.javac.parser.Parser类实现,使用TreeMaker. 语法树是从JCTree实现 com.sun.source.Tree的子类型及其子类型构建的。

经过这个步骤之后,编译器就基本不会再对源码文件进行操作了,后续的操作都建立在抽象语法树之上。

#### 填充符号表

填充符号表enterTrees()方法所做的事情。

**符号表（Symbol Table）**是由一组符号地址和符号信息构成的表格，读者可以把它想象成哈希表中K-V值对的形式（实际上符号表不一定是哈希表实现，可以是有序符号表、树状符号表、栈结构符号表等）。符号表中所登记的信息在编译的不同阶段都要用到。在语义分析中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的说明是否一致）和产生中间代码。在目标生成阶段，当对符号名进行地址分配时，符号表是地址分配的依据。

在Javac源代码中,填充符号表的过程由com.sun.tools.javac.comp.Enter类实现,此过程的出口是一个待处理列表( To Do List ) ,包含了每一个编译单元的抽象语法树的顶级节点, 以及package-info.java ( 如果存在的话）的顶级节点。

### 注解处理器

在JDK 1.5之后,Java语言提供了对注解(Annotation ) 的支持,这些注解与普通的Java代码一样,是在运行期间发挥作用的。在JDK 1.6中实现了JSR-269规范 ,提供了一组插入式注解处理器的标准API在编译期间对注解进行处理,我们可以把它看做是一组编译器的插件 ,在这些插件里面,可以读取、修改、添加抽象语法树中的任意元素。如果这些插件在处理注解期间对语法树进行了修改,编译器将回到解析及填充符号表的过程重新处理,直到所有插入式注解处理器都没有再对语法树进行修改为止,每一次循环称为一个Round回环过程。

在Javac源码中,插入式注解处理器的初始化过程是在initPorcessAnnotations() 方法中完成的,而它的执行过程则是在processAnnotations() 方法中完成的,这个方法判断是否还有新的注解处理器需要执行,如果有的话,通过com.sun.tools.javac.processing.JavacProcessingEnvironment类的doProcessing() 方法生成一个新的JavaCompiler对象对编译的后续步骤进行处理。

### 语义分析与字节码生成

语法分析之后,编译器获得了程序代码的抽象语法树表示,语法树能表示一个结构正确的源程序的抽象,但无法保证源程序是符合逻辑的。而语义分析的主要任务是对结构上正确的源程序进行上下文有关性质的审查,如进行类型审查。举个例子,假设有如下的3个变量定义语句:

```
int a=1;
boolean b=false; 
char c=2;
```

后续可能出现的賦值运算:

```
int d=a+c;
int d=b+c;
char d=a+c;
```

后续代码中如果出现了如上3种賦值运算的话,那它们都能构成结构正确的语法树,但是只有第1种的写法在语义上是没有问题的,能够通过编译,其余两种在Java语言中是不合逻辑的 ,无法编译 (是否合乎语义逻辑必须限定在具体的语言与具体的上下文环境之中才有意义。如在C语言中 ,a 、b 、c 的上下文定义不变,第2 、3种写法都是可以正确编译)。

#### 标注检查

Javac的编译过程中,语义分析过程分为标注检查以及数据及控制流分析两个步骤,分别由attribute() 和flow() 方法完成。

标注检查步骤检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配等。在标注检查步骤中,还有一个重要的动作称为常量折叠,如果我们在代码中写了如下定义:

```
int a=1+2;
```

那么在语法树上仍然能看到字面量“ 1”、“2”以及操作符“+”,但是在经过常量折叠之后 ,它们将会被折叠为字面量“3”。 由于编译期间进行了常量折叠 ,所以在代码里面定义“a=1+2”比起直接定义“a=3” , 并不会增加程序运行期哪怕仅仅一个 CPU指令的运算量。

标注检查步骤实现类是com.sun.tools.javac.comp.Attr类和 com.sun.tools.javac.comp.Check类。

### 数据及控制流分析

数据及控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。编译时期的数据及控制流分析与类加载时的数据及控制流分析的目的基本上是一致的，但校验范围有所区别，有一些校验项只有在编译期或运行期才能进行。下面举一个关于final修饰符的数据及控制流分析的例子。

final语义校验

```
// 方法一带有final修饰
public void foo(final int arg) {
    final int var = 0;
    // do something
}

// 方法二没有final修饰
public void foo(int arg) {
    int var = 0;
    // do something
}
```

在这两个foo() 方法中,第一种方法的参数和局部变量定义使用了final修饰符,而第二种方法则没有,在代码编写时程序肯定会受到final修饰符的影响,不能再改变arg和var变量的值 ,但是这两段代码编译出来的Class文件是没有任何一点区别的,局部变量与字段(实例变量、类变量)是有区别的,它在常量池中没有 CONSTANT_Fieldref_info的符号引用,自然就没有访问标志(Access_Flags ) 的信息,甚至可能连名称都不会保留下来(取决于编译时的选项),自然在Class文件中不可能知道一个局部变量是不是声明为final了。因此,将局部变量声明为final,对运行期是没有影响的,变量的不变性仅仅由编译器在编译期间保障。在Javac的源码中,数据及控制流分析的入口是flow() 方法具体操作由com.sun.tools.javac.comp.Flow类来完成。

#### 解语法糖

语法糖( Syntactic Sugar ) , 也称糖衣语法,是由英国计算机科学家彼得•约翰•兰达 ( Peter J.Landin)发明的一个术语,指在计算机语言中添加的某种语法,这种语法对语言的功能并没有影响,但是更方便程序员使用。通常来说,使用语法糖能够增加程序的可读性, 从而减少程序代码出错的机会。

Java在现代编程语言之中属于“低糖语言” (相对于C#及许多其他JVM语言来说),尤其是JDK 1.5之前的版本,“低糖”语法也是Java语言被怀疑已经“落后”的一个表面理由。Java中最常用的语法糖主要是前面提到过的泛型(泛型并不一定都是语法糖实现,如C#的泛型就是直接由CLR支持的 )、变长参数、自动装箱/拆箱等,虚拟机运行时不支持这些语法 ,它们在编译阶段还原回简单的基础语法结构,这个过程称为解语法糖。

在Javac的源码中,解语法糖的过程由desugar() 方法触发,在 com.sun.tools.javac.comp.TransTypes类和com.sun.tools.javac.comp.Lower类中完成。

#### 字节码生成

字节码生成是Javac编译过程的最后一个阶段,在Javac源码里面由com.sun.tools.javac.jvm.Gen类来完成。字节码生成阶段不仅仅是把前面各个步骤所生成的信息 (语法树、符号表)转化成字节码写到磁盘中,编译器还进行了少量的代码添加和转换工作。

例如实例构造器<init>() 方法和类构造器<clinit> ()方法就是在这个阶段添加到语法树之中的( 注意 ,这里的实例构造器并不是指默认构造函数, 如果用户代码中没有提供任何构造函数,那编译器将会添加一个没有参数的、访问性( public、 protected或private ) 与当前类一致的默认构造函数,这个工作在填充符号表阶段就已经完成 ),这两个构造器的产生过程实际上是一个代码收敛的过程,编译器会把语句块( 对于实例构造器而言是“{}”块 ,对于类构造器而言是“static{}”块 )、变量初始化(实例变量和类变量)、调用父类的实例构造器 ( 仅仅是实例构造器,<clinit>()方法中无须调用父类的<clinit>() 方法,虚拟机会自动保证父类构造器的执行,但在<clinit>() 方法中经常会生成调用java.lang.Object的<init>() 方法的代码 ) 等操作收敛到<init>() 和<clinit>() 方法之中,并且保证一定是按先执行父类的实例构造器,然后初始化变量,最后执行语句块的顺序进行,上面所述的动作由Gen.normalizeDefs() 方法来实现。除了生成构造器以外,还有其他的一些代码替换工作用于优化程序的实现逻辑,如把字符串的加操作替换为StringBuffer或StringBuilder ( 取决于目标代码的版本是否大于或等于JDK 1.5 )的append() 操作等。

完成了对语法树的遍历和调整之后,就会把填充了所有所需信息的符号表交给 com.sun.tools.javac.jvm.ClassWriter类 ,由这个类的writeClass()方法输出字节码,生成最终的Class文件 ,到此为止整个编译过程宣告结束。

## Java语法糖

语法糖可以看做是编译器实现的一些“小把戏” ,这些“小把戏”可能会使得效率“大提升” ,但我们也应该去了解这些“小把戏”背后的真实世界,那样才能利用好它们,而不是被它们所迷惑。

### 泛型与类型擦除

泛型是JDK 1.5的一项新增特性,它的本质是参数化类型( Parametersized Type )的应用 ,也就是说所操作的数据类型被指定为一个参数。这种参数类型可以用在类、接口和方法的创建中,分别称为泛型类、泛型接口和泛型方法。

泛型思想在Java语言处于还没有出现泛型的版本时,只能通过Object是所有类型的父类和类型强制转换两个特点的配合来实现类型泛化。例如,在哈希表的取中, JDK 1.5之前使用HashMap的get() 方法,返回值就是一个Object对象,由于Java语言里面所有的类型都继承于java.lang.Object,所以Object转型成任何对都是有可能的。但是也因为有无限的可能性,就只有程序员和运行期的虚拟机才知道这个Object到底是个什么类型的对象。在编译期间,编译器无法检查这个Object的强制转型是否成功,如果仅仅依赖程序员去保障这项操作的正确性,许多ClassCastException的风险就会转嫁到程序运行期之中。

泛型技术在C#和Java之中的使用方式看似相同,但实现上却有着根本性的分歧,C#里面泛型无论在程序源码中、编译后的IL中( Intermediate Language,中间语言,这时候泛型是一个占位符),或是运行期的CLR中 ,都是切实存在的,List<int>与List<String>就是两个不同的类型,它们在系统运行期生成,有自己的虚方法表和类型数据,这种实现称为类型膨胀 ,基于这种方法实现的泛型称为真实泛型。

Java语言中的泛型则不一样,它只在程序源码中存在,在编译后的字节码文件中,就已 经替换为原来的原生类型( Raw Type,也称为裸类型 )了,并且在相应的地方插入了强制转型代码,因此,对于运行期的Java语言来说,ArrayList<int>与ArrayList<String>就是同一个类,所以泛型技术实际上是Java语言的一颗语法糖,Java语言中的泛型实现方法称为类型擦除 ,基于这种方法实现的泛型称为伪泛型。

一段简单的Java泛型的例子,我们可以看一下它编译后的结果是怎样的。

泛型擦除前的例子

```
public static void main(String[] args) {
    Map<String, String> map = new HashMap<String, String>();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println(map.get("hello"));
    System.out.println(map.get("how are you?"));
}
```

把这段Java代码编译成Class文件 ,然后再用字节码反编译工具进行反编译后,将会发现泛型都不见了，程序又变回了Java泛型出现之前的写法,泛型类型都变回了原生类型。

泛型擦除后的例子

```
public static void main(String[] args) {
    Map map = new HashMap();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println((String) map.get("hello"));
    System.out.println((String) map.get("how are you?"));
}
```

通过擦除法来实现泛型丧失了一些泛型思想应有的优雅。

当泛型遇见重载1

```
public class GenericTypes {

    public static void method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
    }

    public static void method(List<Integer> list) {
        System.out.println("invoke method(List<Integer> list)");
    }
}
```

请想一想,上面这段代码是否正确,能否编译执行?也许你已经有了答案,这段代码是 
不能被编译的,因为参数List<Integer>和List<String>编译之后都被擦除了,变成了一样的原生类型List<E> ,擦除动作导致这两种方法的特征签名变得一模一样。初步看来,无法重载的原因已经找到了,但真的就是如此吗?只能说 ,泛型擦除成相同的原生类型只是无法重载的其中一部分原因。

当泛型遇见重载2

```
public class GenericTypes {

    public static String method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
        return "";
    }

    public static int method(List<Integer> list) {
        System.out.println("invoke method(List<Integer> list)");
        return 1;
    }

    public static void main(String[] args) {
        method(new ArrayList<String>());
        method(new ArrayList<Integer>());
    }
}
```

执行结果：

```
invoke method(List<String> list)
invoke method(List<Integer> list)
```

差别是两个method方法添加了不同的返回值,由于这两个返回值的加入,方法重载居然成功了,即这段代码可以被编译和执行了。这是对Java语言中返回值不参与重载选择的基本认知的挑战吗?

重载当然不是根据返回值来确定的,之所以这次能编译和执行成功, 是因为两个method() 方法加入了不同的返回值后才能共存在一个Class文件之中。Class文件方法表( methodjnfo ) 的数据结构时曾经提到过,方法重载要求方法具备不同的特征签名,返回值并不包含在方法的特征签名之中,所以返回值不参与重载选择,但是在Class文件格式之中,只要描述符不是完全一致的两个方法就可以共存。也就是说,两个方法如果有相同的名称和特征签名,但返回值不同,那它们也是可以合法地共存于一个Class文件中的。

由于Java泛型的引入,各种场景(虚拟机解析、反射等)下的方法调用都可能对原有的基础产生影响和新的需求,如在泛型类中如何获取传入的参数化类型等。因此 ,JCP组织对虚拟机规范做出了相应的修改,引入了诸如Signature、LocalVariableTypeTable等新的属性用于解决伴随泛型而来的参数类型的识别问题,Signature是其中最重要的一项属性,它的作用就是存储一个方法在字节码层面的特征签名,这个属性中保存的参数类型并不是原生类型 ,而是包括了参数化类型的信息。修改后的虚拟机规范要求所有能识别49.0以上版本的 Class文件的虚拟机都要能正确地识别Signature参数。

从上面的例子可以看到擦除法对实际编码带来的影响,由于List<String> 和List<Integer> 擦除后是同一个类型,我们只能添加两个并不需要实际使用到的返回值才能完成重载,这是一种毫无优雅和美感可言的解决方案,并且存在一定语意上的混乱,譬如上面脚注中提到的,必须用SunJDK 1.6的Javac才能编译成功,其他版本或者ECJ编译器都可能拒绝编译。

另外 ,从Signature属性的出现我们还可以得出结论,擦除法所谓的擦除,仅仅是对方法的Code属性中的字节码进行擦除,实际上元数据中还是保留了泛型信息,这也是我们能通过反射手段取得参数化类型的根本依据。

### 自动装箱、拆箱与遍历循环

从纯技术的角度来讲,自动装箱、自动拆箱与遍历循环(Foreach循环 )这些语法糖,无论是实现上还是思想上都不能和上文介绍的泛型相比,两者的难度和深度都有很大差距。

自动装箱、拆箱与遍历循环

```
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4);
    // 如果在JDK 1.7中，还有另外一颗语法糖 ，
    // 能让上面这句代码进一步简写成List<Integer> list = [1, 2, 3, 4];
    int sum = 0;
    for (int i : list) {
        sum += i;
    }
    System.out.println(sum);
}
```

自动装箱、拆箱与遍历循环编译之后

```
public static void main(String[] args) {
    List list = Arrays.asList( new Integer[] {
         Integer.valueOf(1),
         Integer.valueOf(2),
         Integer.valueOf(3),
         Integer.valueOf(4) });

    int sum = 0;
    for (Iterator localIterator = list.iterator(); localIterator.hasNext(); ) {
        int i = ((Integer)localIterator.next()).intValue();
        sum += i;
    }
    System.out.println(sum);
}
```

泛型、自动装箱、自动拆箱、遍历循环与变长参数5种语法糖 以及编译后的变化。泛型就不必说了,自动装箱、拆箱在编译之后被转化成了对应的包装和还原方法,如本例中的Integer.valueOf() 与Integer.intValue() 方法,而遍历循环则把代码还原成了迭代器的实现,这也是为何遍历循环需要被遍历的类实现Iterable接口的原因。最后再看看变长参数,它在调用的时候变成了一个数组类型的参数,在变长参数出现之前,程序员就是使用数组来完成类似功能的。

自动装箱的陷阱

```
public static void main(String[] args) {
    Integer a = 1;
    Integer b = 2;
    Integer c = 3;
    Integer d = 3;
    Integer e = 321;
    Integer f = 321;
    Long g = 3L;
    System.out.println(c == d);
    System.out.println(e == f);
    System.out.println(c == (a + b));
    System.out.println(c.equals(a + b));
    System.out.println(g == (a + b));
    System.out.println(g.equals(a + b));
}
```

无论读者的回答是否正确,鉴于包装类的“= ”运算在不遇到算术运算的情况下不会自动拆箱,以及它们equals()方法不处理数据转型的关系,建议在实际编码中尽量避免这样使用自动装箱与拆箱。

### 条件编译

许多程序设计语言都提供了条件编译的途径,如C、C++中使用预处理器指示符 (#ifdef)来完成条件编译。C、C++的预处理器最初的任务是解决编译时的代码依赖关系( 如非常常用的#include预处理命令),而在Java语言之中并没有使用预处理器,因为Java语言天然的编译方式(编译器并非一个个地编译Java文件 ,而是将所有编译单元的语法树顶级节点输入到待处理列表后再进行编译,因此各个文件之间能够互相提供符号信息)无须使用预处理器。那Java语言是否有办法实现条件编译呢?

Java语言当然也可以进行条件编译,方法就是使用条件为常量的if语句。此代码中的if语句不同于其他Java代码 ,它在编译阶段就会被“运行”,生成的字节码之中只包括“System.out.println ( “block 1” ) ; ”一条语句,并不会包含if语句及另外一个分子中的“System.out.println ( “block 2”) ; ”

Java语言的条件编译

```
public static void main(String[] args) {
    if (true) {
        System.out.println("block 1");
    } else {
        System.out.println("block 2");
    }
}
```

上述代码编译后Class文件的反编译结果:

```
public static void main(String[] args) {
    System.out.println("block 1");
}
```

只能使用条件为常量的if语句才能达到上述效果,如果使用常量与其他带有条件判断能力的语句搭配,则可能在控制流分析中提示错误,被拒绝编译代。

不能使用其他条件语句来完成条件编译

```
public static void main(String[] args) {
    // 编译器将会提示“Unreachable code”
    while (false) {
        System.out.println("");
    }
}
```

Java语言中条件编译的实现,也是Java语言的一颗语法糖,根据布尔常量值的真假,编译器将会把分支中不成立的代码消除掉 ,这一工作将在编译器解除语法糖阶段 ( com.sun.tools.javac.comp.Lower类中)完成。由于这种条件编译的实现方式使用了if语句,所以它必须遵循最基本的Java语法 ,只能写在方法体内部,因此它只能实现语句基本块 (Block)级别的条件编译,而没有办法实现根据条件调整整个Java类的结构。

除了本节中介绍的泛型、自动装箱、自动拆箱、遍历循环、变长参数和条件编译之外 ,Java语言还有不少其他的语法糖,如内部类、枚举类、断言语句、对枚举和字符串(在 JDK 1.7中支持)的switch支持、try语句中定义和关闭资源(在JDK 1.7中支持)等 ,读者可以通过跟踪Javac源码、反编译Class文件等方式了解它们的本质实现。

## 实战：插入式注解处理器

阅读Javac编译器的源码，我们知道编译器在把Java程序源码编译为字节码的时候，会对Java程序源码做各方面的检查校验。我们将会使用注解处理器API来编写一款拥有自己编码风格的校验工具：NameCheckProcessor。

NameCheckProcessor的目标也仅定为对Java程序命名进行检查

### 源码

要通过注解处理器API实现一个编译器插件，首先需要了解这组API的一些基本知识。我们实现注解处理器的代码需要继承抽象类javax.annotation.processing.AbstractProcessor，这个抽象类中只有一个必须覆盖的abstract方法：“process()”，它是Javac编译器在执行注解处理器代码时要调用的过程，我们可以从这个方法的第一个参数“annotations”中获取到此注解处理器所要处理的注解集合，从第二个参数“roundEnv”中访问到当前这个Round中的语法树节点，每个语法树节点在这里表示为一个Element。在JDK 1.6新增的javax.lang.model包中定义了16类Element，包括了Java代码中最常用的元素，如：“包（PACKAGE）、枚举（ENUM）、类（CLASS）、注解（ANNOTATION_TYPE）、接口（INTERFACE）、枚举值（ENUM_CONSTANT）、字段（FIELD）、参数（PARAMETER）、本地变量（LOCAL_VARIABLE）、异常（EXCEPTION_PARAMETER）、方法（METHOD）、构造函数（CONSTRUCTOR）、静态语句块（STATIC_INIT，即static{}块）、实例语句块（INSTANCE_INIT，即{}块）、参数化类型（TYPE_PARAMETER，既泛型尖括号内的类型）和未定义的其他语法树节点（OTHER）”。除了process（）方法的传入参数之外，还有一个很常用的实例变量“processingEnv”，它是AbstractProcessor中的一个protected变量，在注解处理器初始化的时候（init（）方法执行的时候）创建，继承了AbstractProcessor的注解处理器代码可以直接访问到它。它代表了注解处理器框架提供的一个上下文环境，要创建新的代码、向编译器输出信息、获取其他工具类等都需要用到这个实例变量。

注解处理器除了process（）方法及其参数之外，还有两个可以配合使用的Annotations：@SupportedAnnotationTypes和@SupportedSourceVersion，前者代表了这个注解处理器对哪些注解感兴趣，可以使用星号“*”作为通配符代表对所有的注解都感兴趣，后者指出这个注解处理器可以处理哪些版本的Java代码。

每一个注解处理器在运行的时候都是单例的，如果不需要改变或生成语法树的内容，process（）方法就可以返回一个值为false的布尔值，通知编译器这个Round中的代码未发生变化，无须构造新的JavaCompiler实例，在这次实战的注解处理器中只对程序命名进行检查，不需要改变语法树的内容，因此process（）方法的返回值都是false。关于注解处理器的API，笔者就简单介绍这些，对这个领域有兴趣的读者可以阅读相关的帮助文档。下面来看看注解处理器NameCheckProcessor的具体代码。

注解处理器NameCheckProcessor

```
// 可以用"*"表示支持所有Annotations
@SupportedAnnotationTypes("*")
// 只支持JDK 1.6的Java代码
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class NameCheckProcessor extends AbstractProcessor {

    private NameChecker nameChecker;

    /**
     * 初始化名称检查插件
     */
    @Override
    public void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        nameChecker = new NameChecker(processingEnv);
    }

    /**
     * 对输入的语法树的各个节点进行进行名称检查
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (!roundEnv.processingOver()) {
            for (Element element : roundEnv.getRootElements())
                nameChecker.checkNames(element);
        }
        return false;
    }

}
```

从上面代码可以看出，NameCheckProcessor能处理基于JDK 1.6的源码，它不限于特定的注解，对任何代码都“感兴趣”，而在process（）方法中是把当前Round中的每一个RootElement传递到一个名为NameChecker的检查器中执行名称检查逻辑，NameChecker的代码如代码清单10-12所示。

命名检查器NameChecker

```
/**
 * 程序名称规范的编译器插件：<br>
 * 如果程序命名不合规范，将会输出一个编译器的WARNING信息
 */
public class NameChecker {
    private final Messager messager;

    NameCheckScanner nameCheckScanner = new NameCheckScanner();

    NameChecker(ProcessingEnvironment processsingEnv) {
        this.messager = processsingEnv.getMessager();
    }

    /**
     * 对Java程序命名进行检查，根据《Java语言规范》第三版第6.8节的要求，Java程序命名应当符合下列格式：
     * 
     * <ul>
     * <li>类或接口：符合驼式命名法，首字母大写。
     * <li>方法：符合驼式命名法，首字母小写。
     * <li>字段：
     * <ul>
     * <li>类、实例变量: 符合驼式命名法，首字母小写。
     * <li>常量: 要求全部大写。
     * </ul>
     * </ul>
     */
    public void checkNames(Element element) {
        nameCheckScanner.scan(element);
    }

    /**
     * 名称检查器实现类，继承了JDK 1.6中新提供的ElementScanner6<br>
     * 将会以Visitor模式访问抽象语法树中的元素
     */
    private class NameCheckScanner extends ElementScanner6<Void, Void> {

        /**
         * 此方法用于检查Java类
         */
        @Override
        public Void visitType(TypeElement e, Void p) {
            scan(e.getTypeParameters(), p);
            checkCamelCase(e, true);
            super.visitType(e, p);
            return null;
        }

        /**
         * 检查方法命名是否合法
         */
        @Override
        public Void visitExecutable(ExecutableElement e, Void p) {
            if (e.getKind() == METHOD) {
                Name name = e.getSimpleName();
                if (name.contentEquals(e.getEnclosingElement().getSimpleName()))
                    messager.printMessage(WARNING, "一个普通方法 “" + name + "”不应当与类名重复，避免与构造函数产生混淆", e);
                checkCamelCase(e, false);
            }
            super.visitExecutable(e, p);
            return null;
        }

        /**
         * 检查变量命名是否合法
         */
        @Override
        public Void visitVariable(VariableElement e, Void p) {
            // 如果这个Variable是枚举或常量，则按大写命名检查，否则按照驼式命名法规则检查
            if (e.getKind() == ENUM_CONSTANT || e.getConstantValue() != null || heuristicallyConstant(e))
                checkAllCaps(e);
            else
                checkCamelCase(e, false);
            return null;
        }

        /**
         * 判断一个变量是否是常量
         */
        private boolean heuristicallyConstant(VariableElement e) {
            if (e.getEnclosingElement().getKind() == INTERFACE)
                return true;
            else if (e.getKind() == FIELD && e.getModifiers().containsAll(EnumSet.of(PUBLIC, STATIC, FINAL)))
                return true;
            else {
                return false;
            }
        }

        /**
         * 检查传入的Element是否符合驼式命名法，如果不符合，则输出警告信息
         */
        private void checkCamelCase(Element e, boolean initialCaps) {
            String name = e.getSimpleName().toString();
            boolean previousUpper = false;
            boolean conventional = true;
            int firstCodePoint = name.codePointAt(0);

            if (Character.isUpperCase(firstCodePoint)) {
                previousUpper = true;
                if (!initialCaps) {
                    messager.printMessage(WARNING, "名称“" + name + "”应当以小写字母开头", e);
                    return;
                }
            } else if (Character.isLowerCase(firstCodePoint)) {
                if (initialCaps) {
                    messager.printMessage(WARNING, "名称“" + name + "”应当以大写字母开头", e);
                    return;
                }
            } else
                conventional = false;

            if (conventional) {
                int cp = firstCodePoint;
                for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                    cp = name.codePointAt(i);
                    if (Character.isUpperCase(cp)) {
                        if (previousUpper) {
                            conventional = false;
                            break;
                        }
                        previousUpper = true;
                    } else
                        previousUpper = false;
                }
            }

            if (!conventional)
                messager.printMessage(WARNING, "名称“" + name + "”应当符合驼式命名法（Camel Case Names）", e);
        }

        /**
         * 大写命名检查，要求第一个字母必须是大写的英文字母，其余部分可以是下划线或大写字母
         */
        private void checkAllCaps(Element e) {
            String name = e.getSimpleName().toString();

            boolean conventional = true;
            int firstCodePoint = name.codePointAt(0);

            if (!Character.isUpperCase(firstCodePoint))
                conventional = false;
            else {
                boolean previousUnderscore = false;
                int cp = firstCodePoint;
                for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                    cp = name.codePointAt(i);
                    if (cp == (int) '_') {
                        if (previousUnderscore) {
                            conventional = false;
                            break;
                        }
                        previousUnderscore = true;
                    } else {
                        previousUnderscore = false;
                        if (!Character.isUpperCase(cp) && !Character.isDigit(cp)) {
                            conventional = false;
                            break;
                        }
                    }
                }
            }

            if (!conventional)
                messager.printMessage(WARNING, "常量“" + name + "”应当全部以大写字母或下划线命名，并且以字母开头", e);
        }
    }
}
```

NameChecker的代码看起来有点长，但实际上注释占了很大一部分，其实即使算上注释也不到190行。它通过一个继承于javax.lang.model.util.ElementScanner6的NameCheckScanner类，以Visitor模式来完成对语法树的遍历，分别执行visitType（）、visitVariable（）和visitExecutable（）方法来访问类、字段和方法，这3个visit方法对各自的命名规则做相应的检查，checkCamelCase（）与checkAllCaps（）方法则用于实现驼式命名法和全大写命名规则的检查。

整个注解处理器只需NameCheckProcessor和NameChecker两个类就可以全部完成，为了验证我们的实战成果，代码清单10-13中提供了一段命名规范的“反面教材”代码，其中的每一个类、方法及字段的命名都存在问题，但是使用普通的Javac编译这段代码时不会提示任何一个Warning信息。

代码清单10-13　包含了多处不规范命名的代码样例

```
public class BADLY_NAMED_CODE {

    enum colors {
        red, blue, green;
    }

    static final int _FORTY_TWO = 42;

    public static int NOT_A_CONSTANT = _FORTY_TWO;

    protected void BADLY_NAMED_CODE() {
        return;
    }

    public void NOTcamelCASEmethodNAME() {
        return;
    }
}
```

### 运行与测试

我们可以通过Javac命令的“-processor”参数来执行编译时需要附带的注解处理器，如果有多个注解处理器的话，用逗号分隔。还可以使用-XprintRounds和-XprintProcessorInfo参数来查看注解处理器运作的详细信息，本次实战中的NameCheckProcessor的编译及执行过程如代码清单10-14所示。

注解处理器的运行过程

```
D：\src＞javac org/fenixsoft/compile/NameChecker.java
D：\src＞javac org/fenixsoft/compile/NameCheckProcessor.java
D：\src＞javac-processor org.fenixsoft.compile.NameCheckProcessor org/fenixsoft/compile/BADLY_NAMED_CODE.java
org\fenixsoft\compile\BADLY_NAMED_CODE.java：3：警告：名称"BADLY_NAMED_CODE"应当符合驼式命名法（Camel Case Names）
public class BADLY_NAMED_CODE{
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：5：警告：名称"colors"应当以大写字母开头
enum colors{
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：6：警告：常量"red"应当全部以大写字母或下划线命名，并且以字母开头
red,blue,green；
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：6：警告：常量"blue"应当全部以大写字母或下划线命名，并且以字母开头
red,blue,green；
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：6：警告：常量"green"应当全部以大写字母或下划线命名，并且以字母开头
red,blue,green；
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：9：警告：常量"_FORTY_TWO"应当全部以大写字母或下划线命名，并且以字母开头
static final int_FORTY_TWO=42；
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：11：警告：名称"NOT_A_CONSTANT"应当以小写字母开头
public static int NOT_A_CONSTANT=_FORTY_TWO；
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：13：警告：名称"Test"应当以小写字母开头
protected void Test（）{
^
org\fenixsoft\compile\BADLY_NAMED_CODE.java：17：警告：名称"NOTcamelCASEmethodNAME"应当以小写字母开头
public void NOTcamelCASEmethodNAME（）{
^
```

### 其他应用案例

NameCheckProcessor的实战例子只演示了JSR-269嵌入式注解处理器API中的一部分功能，基于这组API支持的项目还有用于校验Hibernate标签使用正确性的Hibernate Validator Annotation Processor（本质上与NameCheckProcessor所做的事情差不多）、自动为字段生成getter和setter方法的Project Lombok（根据已有元素生成新的语法树元素）等。



# Java内存模型与线程

## 概述

多任务处理在现代计算机操作系统中几乎已是一项必备的功能了。在许多情况下，让计算机同时去做几件事情，不仅是因为计算机的运算能力强大了，还有一个很重要的原因是计算机的运算速度与它的存储和通信子系统速度的差距太大，大量的时间都花费在磁盘I/O、网络通信或者数据库访问上。如果不希望处理器在大部分时间里都处于等待其他资源的状态，就必须使用一些手段去把处理器的运算能力“压榨”出来，否则就会造成很大的浪费，而让计算机同时处理几项任务则是最容易想到、也被证明是非常有效的“压榨”手段。

除了充分利用计算机处理器的能力外，一个服务端同时对多个客户端提供服务则是另一个更具体的并发应用场景。衡量一个服务性能的高低好坏，每秒事务处理数（Transactions Per Second,TPS）是最重要的指标之一，它代表着一秒内服务端平均能响应的请求总数，而TPS值与程序的并发能力又有非常密切的关系。对于计算量相同的任务，程序线程并发协调得越有条不紊，效率自然就会越高；反之，线程之间频繁阻塞甚至死锁，将会大大降低程序的并发能力。

服务端是Java语言最擅长的领域之一，这个领域的应用占了Java应用中最大的一块份额，不过如何写好并发应用程序却又是服务端程序开发的难点之一，处理好并发方面的问题通常需要更多的编码经验来支持。幸好Java语言和虚拟机提供了许多工具，把并发编程的门槛降低了不少。并且各种中间件服务器、各类框架都努力地替程序员处理尽可能多的线程并发细节，使得程序员在编码时能更关注业务逻辑，而不是花费大部分时间去关注此服务会同时被多少人调用、如何协调硬件资源。无论语言、中间件和框架如何先进，开发人员都不能期望它们能独立完成所有并发处理的事情，了解并发的内幕也是成为一个高级程序员不可缺少的课程。

Amdahl定律通过系统中并行化与串行化的比重来描述多处理器系统能获得的运算加速能力，摩尔定律则用于描述处理器晶体管数量与运行效率之间的发展关系。这两个定律的更替代表了近年来硬件发展从追求处理器频率到追求多核心并行处理的发展过程。

## 硬件的效率与一致性

在正式讲解Java虚拟机并发相关的知识之前，我们先花费一点时间去了解一下物理计算机中的并发问题，物理机遇到的并发问题与虚拟机中的情况有不少相似之处，物理机对并发的处理方案对于虚拟机的实现也有相当大的参考意义。

“让计算机并发执行若干个运算任务”与“更充分地利用计算机处理器的效能”之间的因果关系，看起来顺理成章，实际上它们之间的关系并没有想象中的那么简单，其中一个重要的复杂性来源是绝大多数的运算任务都不可能只靠处理器“计算”就能完成，处理器至少要与内存交互，如读取运算数据、存储运算结果等，这个I/O操作是很难消除的（无法仅靠寄存器来完成所有运算任务）。由于计算机的存储设备与处理器的运算速度有几个数量级的差距，所以现代计算机系统都不得不加入一层读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为内存与处理器之间的缓冲：将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了。

基于高速缓存的存储交互很好地解决了处理器与内存的速度矛盾，但是也为计算机系统带来更高的复杂度，因为它引入了一个新的问题：缓存一致性（Cache Coherence）。在多处理器系统中，每个处理器都有自己的高速缓存，而它们又共享同一主内存（Main Memory），如图12-1所示。当多个处理器的运算任务都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致，如果真的发生这种情况，那同步回到主内存时以谁的缓存数据为准呢？为了解决一致性的问题，需要各个处理器访问缓存时都遵循一些协议，在读写时要根据协议来进行操作，这类协议有MSI、MESI（Illinois Protocol）、MOSI、Synapse、Firefly及Dragon Protocol等。在本章中将会多次提到的“内存模型”一词，可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象。不同架构的物理机器可以拥有不一样的内存模型，而Java虚拟机也有自己的内存模型，并且这里介绍的内存访问操作与硬件的缓存访问操作具有很高的可比性。

![img](https://img-blog.csdn.net/20151107223057063)

图　12-1　处理器、高速缓存、主内存间的交互关系

除了增加高速缓存之外，为了使得处理器内部的运算单元能尽量被充分利用，处理器可能会对输入代码进行乱序执行（Out-Of-Order Execution）优化，处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致，因此，如果存在一个计算任务依赖另外一个计算任务的中间结果，那么其顺序性并不能靠代码的先后顺序来保证。与处理器的乱序执行优化类似，Java虚拟机的即时编译器中也有类似的指令重排序（Instruction Reorder）优化。

## Java内存模型

Java虚拟机规范中试图定义一种Java内存模型（Java Memory Model,JMM）来屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。在此之前，主流程序语言（如C/C++等）直接使用物理硬件和操作系统的内存模型，因此，会由于不同平台上内存模型的差异，有可能导致程序在一套平台上并发完全正常，而在另外一套平台上并发访问却经常出错，因此在某些场景就必须针对不同的平台来编写程序。

定义Java内存模型并非一件容易的事情，这个模型必须定义得足够严谨，才能让Java的并发内存访问操作不会产生歧义；但是，也必须定义得足够宽松，使得虚拟机的实现有足够的自由空间去利用硬件的各种特性（寄存器、高速缓存和指令集中某些特有的指令）来获取更好的执行速度。经过长时间的验证和修补，在JDK 1.5（实现了JSR-133[2]）发布后，Java内存模型已经成熟和完善起来了。

## 主内存与工作内存

Java内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。此处的变量（Variables）与Java编程中所说的变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。为了获得较好的执行效能，Java内存模型并没有限制执行引擎使用处理器的特定寄存器或缓存来和主内存进行交互，也没有限制即时编译器进行调整代码执行顺序这类优化措施。

Java内存模型规定了所有的变量都存储在主内存（Main Memory）中（此处的主内存与介绍物理硬件时的主内存名字一样，两者也可以互相类比，但此处仅是虚拟机内存的一部分）。每条线程还有自己的工作内存（Working Memory，可与前面讲的处理器高速缓存类比），线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如图12-2所示。

![img](https://img-blog.csdn.net/20151107223240673)

图　12-2　线程、主内存、工作内存三者的交互关系（请与图12-1对比）

注：

- 如果局部变量是一个reference类型，它引用的对象在Java堆中可被各个线程共享，但是reference本身在Java栈的局部变量表中，它是线程私有的。
- “拷贝副本”，如“假设线程中访问一个10MB的对象，也会把这10MB的内存复制一份拷贝出来吗？”，事实上并不会如此，这个对象的引用、对象中某个在线程访问到的字段是有可能存在拷贝的，但不会有虚拟机实现成把整个对象拷贝A一次。
- volatile变量依然有工作内存的拷贝，但是由于它特殊的操作顺序性规定，所以看起来如同直接在主内存中读写访问一般，因此这里的描述对于volatile也并不存在例外。
- 除了实例数据，Java堆还保存了对象的其他信息，对于HotSpot虚拟机来讲，有Mark Word（存储对象哈希码、GC标志、GC年龄、同步锁等信息）、Klass Point（指向存储类型元数据的指针）及一些用于字节对齐补白的填充数据（如果实例数据刚好满足8字节对齐的话，则可以不存在补白）。

## 内存间交互操作

关于主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存之类的实现细节，Java内存模型中定义了以下8种操作来完成，虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的（对于double和long类型的变量来说，load、store、read和write操作在某些平台上允许有例外，这个问题后文会讲）。

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用。
- load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的write操作使用。
- write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

如果要把一个变量从主内存复制到工作内存，那就要顺序地执行read和load操作，如果要把变量从工作内存同步回主内存，就要顺序地执行store和write操作。注意，Java内存模型只要求上述两个操作必须按顺序执行，而没有保证是连续执行。也就是说，read与load之间、store与write之间是可插入其他指令的，如对主内存中的变量a、b进行访问时，一种可能出现顺序是read a、read b、load b、load a。除此之外，Java内存模型还规定了在执行上述8种基本操作时必须满足如下规则：

- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者从工作内存发起回写了但主内存不接受的情况出现。
- 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。
- 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说，就是对一个变量实施use、store操作之前，必须先执行过了assign和load操作。
- 一个变量在同一个时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。
- 如果对一个变量执行lock操作，那将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
- 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定住的变量。
- 对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）。

这8种内存访问操作以及上述规则限定，再加上稍后介绍的对volatile的一些特殊规定，就已经完全确定了Java程序中哪些内存访问操作在并发下是安全的。由于这种定义相当严谨但又十分烦琐，实践起来很麻烦，所以在后文将介绍这种定义的一个等效判断原则——先行发生原则，用来确定一个访问在并发环境下是否安全。

注： 
基于理解难度和严谨性考虑，最新的JSR-133文档中，已经放弃采用这8种操作去定义Java内存模型的访问协议了（仅是描述方式改变了，Java内存模型并没有改变）。

## 对于volatile型变量的特殊规则

关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制，但是它并不容易完全被正确、完整地理解，以至于许多程序员都习惯不去使用它，遇到需要处理多线程数据竞争问题的时候一律使用synchronized来进行同步。了解volatile变量的语义对后面了解多线程操作的其他特性很有意义，在本节中我们将多花费一些时间去弄清楚volatile的语义到底是什么。

Java内存模型对volatile专门定义了一些特殊的访问规则，在介绍这些比较拗口的规则定义之前，先用不那么正式但[通俗易懂](https://www.baidu.com/s?wd=通俗易懂&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)的语言来介绍一下这个关键字的作用。

当一个变量定义为volatile之后，它将具备两种特性，第一是保证此变量对所有线程的可见性，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点，普通变量的值在线程间传递均需要通过主内存来完成，例如，线程A修改一个普通变量的值，然后向主内存进行回写，另外一条线程B在线程A回写完成了之后再从主内存进行读取操作，新变量值才会对线程B可见。

关于volatile变量的可见性，经常会被开发人员误解，认为以下描述成立：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反应到其他线程之中，换句话说，volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是安全的”。这句话的论据部分并没有错，但是其论据并不能得出“基于volatile变量的运算在并发下是安全的”这个结论。volatile变量在各个线程的工作内存中不存在一致性问题（在各个线程的工作内存中，volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性问题），但是Java里面的运算并非原子操作，导致volatile变量的运算在并发下一样是不安全的，我们可以通过一段简单的演示来说明原因，请看代码清单12-1中演示的例子。

代码清单12-1　volatile的运算

```
/**
 * volatile变量自增运算测试
 * 
 */
public class VolatileTest {

    public static volatile int race = 0;

    public static void increase() {
        race++;
    }

    private static final int THREADS_COUNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++) {
                        increase();
                    }
                }
            });
            threads[i].start();
        }

        // 等待所有累加线程都结束
        while (Thread.activeCount() > 1)
            Thread.yield();

        System.out.println(race);
    }
}
```

 

这段代码发起了20个线程，每个线程对race变量进行10000次自增操作，如果这段代码能够正确并发的话，最后输出的结果应该是200000。读者运行完这段代码之后，并不会获得期望的结果，而且会发现每次运行程序，输出的结果都不一样，都是一个小于200000的数字，这是为什么呢？

问题就出现在自增运算“race++”之中，我们用Javap反编译这段代码后会得到代码清单12-2，发现只有一行代码的increase（）方法在Class文件中是由4条字节码指令构成的（return指令不是由race++产生的，这条指令可以不计算），从字节码层面上很容易就分析出并发失败的原因了：当getstatic指令把race的值取到操作栈顶时，volatile关键字保证了race的值在此时是正确的，但是在执行iconst_1、iadd这些指令的时候，其他线程可能已经把race的值加大了，而在操作栈顶的值就变成了过期的数据，所以putstatic指令执行后就可能把较小的race值同步回主内存之中。

代码清单12-2　VolatileTest的字节码

```
public static void increase（）；
Code：
Stack=2，Locals=0，Args_size=0
0：getstatic#13；//Field race：I
3：iconst_1
4：iadd
5：putstatic#13；//Field race：I
8：return
LineNumberTable：
line 14：0
line 15：8 
```



客观地说，笔者在此使用字节码来分析并发问题，仍然是不严谨的，因为即使编译出来只有一条字节码指令，也并不意味执行这条指令就是一个原子操作。一条字节码指令在解释执行时，解释器将要运行许多行代码才能实现它的语义，如果是编译执行，一条字节码指令也可能转化成若干条本地机器码指令，此处使用-XX：+PrintAssembly参数输出反汇编来分析会更加严谨一些，但考虑到读者阅读的方便，并且字节码已经能说明问题，所以此处使用字节码来分析。

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用synchronized或java.util.concurrent中的原子类）来保证原子性。

- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 变量不需要与其他的状态变量共同参与不变约束。

而在像如下的代码清单12-3所示的这类场景就很适合使用volatile变量来控制并发，当shutdown（）方法被调用时，能保证所有线程中执行的doWork（）方法都立即停下来。

代码清单12-3　volatile的使用场景



```
volatile boolean shutdownRequested；
public void shutdown（）{
    shutdownRequested=true；
}
public void doWork（）{
    while（！shutdownRequested）{
        //do stuff
    }
} 
```

使用volatile变量的第二个语义是禁止指令重排序优化，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。因为在一个线程的方法执行过程中无法感知到这点，这也就是Java内存模型中描述的所谓的“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics）。

上面的描述仍然不太容易理解，我们还是继续通过一个例子来看看为何指令重排序会干扰程序的并发执行，演示程序如代码清单12-4所示。

代码清单12-4　指令重排序



```
Map configOptions；
char[]configText；
//此变量必须定义为volatile
volatile boolean initialized=false；
//假设以下代码在线程A中执行
//模拟读取配置信息，当读取完成后将initialized设置为true以通知其他线程配置可用
configOptions=new HashMap（）；
configText=readConfigFile（fileName）；
processConfigOptions（configText,configOptions）；
initialized=true；
//假设以下代码在线程B中执行
//等待initialized为true，代表线程A已经把配置信息初始化完成
while（！initialized）{
    sleep（）；
}
//使用线程A中初始化好的配置信息
doSomethingWithConfig（）； 
```



代码清单12-4中的程序是一段伪代码，其中描述的场景十分常见，只是我们在处理配置文件时一般不会出现并发而已。如果定义initialized变量时没有使用volatile修饰，就可能会由于指令重排序的优化，导致位于线程A中最后一句的代码“initialized=true”被提前执行（这里虽然使用Java作为伪代码，但所指的重排序优化是机器级的优化操作，提前执行是指这句话对应的汇编代码被提前执行），这样在线程B中使用配置信息的代码就可能出现错误，而volatile关键字则可以避免此类情况的发生。

指令重排序是并发编程中最容易让开发人员产生疑惑的地方，除了上面伪代码的例子之外，笔者再举一个可以实际操作运行的例子来分析volatile关键字是如何禁止指令重排序优化的。代码清单12-5是一段标准的DCL单例代码，可以观察加入volatile和未加入volatile关键字时所生成汇编代码的差别（如何获得JIT的汇编代码，请参考4.2.7节）。

代码清单12-5　DCL单例模式



```
public class Singleton {

    private volatile static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args) {
            Singleton.getInstance();
    }
} 
```



编译后，这段代码对instance变量赋值部分如代码清单12-6所示。

代码清单12-6



```
0x01a3de0f：mov$0x3375cdb0，%esi         ；……beb0cd75 33
                                        ；{oop（'Singleton'）}
0x01a3de14：mov%eax，0x150（%esi）      ；……89865001 0000
0x01a3de1a：shr$0x9，%esi                ；……c1ee09
0x01a3de1d：movb$0x0，0x1104800（%esi）    ；……c6860048 100100
0x01a3de24：lock addl$0x0，（%esp）        ；……f0830424 00
                                        ；*putstatic instance
                                        ；-
Singleton：getInstance@24 
```



通过对比就会发现，关键变化在于有volatile修饰的变量，赋值后（前面mov%eax，0x150（%esi）这句便是赋值操作）多执行了一个“lock addl ＄0x0，（%esp）”操作，这个操作相当于一个内存屏障（Memory Barrier或Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置），只有一个CPU访问内存时，并不需要内存屏障；但如果有两个或更多CPU访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了。这句指令中的“addl ＄0x0，（%esp）”（把ESP寄存器的值加0）显然是一个空操作（采用这个空操作而不是空操作指令nop是因为IA32手册规定lock前缀不允许配合nop指令使用），关键在于lock前缀，查询IA32手册，它的作用是使得本CPU的Cache写入了内存，该写入动作也会引起别的CPU或者别的内核无效化（Invalidate）其Cache，这种操作相当于对Cache中的变量做了一次前面介绍Java内存模式中所说的“store和write”操作。所以通过这样一个空操作，可让前面volatile变量的修改对其他CPU立即可见。

那为何说它禁止指令重排序呢？从硬件架构上讲，指令重排序是指CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理。但并不是说指令任意重排，CPU需要能正确处理指令依赖情况以保障程序能得出正确的执行结果。譬如指令1把地址A中的值加10，指令2把地址A中的值乘以2，指令3把地址B中的值减去3，这时指令1和指令2是有依赖的，它们之间的顺序不能重排——（A+10）*2与A*2+10显然不相等，但指令3可以重排到指令1、2之前或者中间，只要保证CPU执行后面依赖到A、B值的操作时能获取到正确的A和B值即可。所以在本内CPU中，重排序看起来依然是有序的。因此，lock addl＄0x0，（%esp）指令把修改同步到内存时，意味着所有之前的操作都已经执行完成，这样便形成了“指令重排序无法越过内存屏障”的效果。

解决了volatile的语义问题，再来看看在众多保障并发安全的工具中选用volatile的意义——它能让我们的代码比使用其他的同步工具更快吗？在某些情况下，volatile的同步机制的性能确实要优于锁（使用synchronized关键字或java.util.concurrent包里面的锁），但是由于虚拟机对锁实行的许多消除和优化，使得我们很难量化地认为volatile就会比synchronized快多少。如果让volatile自己与自己比较，那可以确定一个原则：volatile变量读操作的性能消耗与普通变量几乎没有什么差别，但是写操作则可能会慢一些，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。不过即便如此，大多数场景下volatile的总开销仍然要比锁低，我们在volatile与锁之中选择的唯一依据仅仅是volatile的语义能否满足使用场景的需求。

最后，我们回头看一下Java内存模型中对volatile变量定义的特殊规则。假定T表示一个线程，V和W分别表示两个volatile型变量，那么在进行read、load、use、assign、store和write操作时需要满足如下规则：

- 只有当线程T对变量V执行的前一个动作是load的时候，线程T才能对变量V执行use动作；并且，只有当线程T对变量V执行的后一个动作是use的时候，线程T才能对变量V执行load动作。线程T对变量V的use动作可以认为是和线程T对变量V的load、read动作相关联，必须连续一起出现（这条规则要求在工作内存中，每次使用V前都必须先从主内存刷新最新的值，用于保证能看见其他线程对变量V所做的修改后的值）。
- 只有当线程T对变量V执行的前一个动作是assign的时候，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行的后一个动作是store的时候，线程T才能对变量V执行assign动作。线程T对变量V的assign动作可以认为是和线程T对变量V的store、write动作相关联，必须连续一起出现（这条规则要求在工作内存中，每次修改V后都必须立刻同步回主内存中，用于保证其他线程可以看到自己对变量V所做的修改）。

注： 
volatile屏蔽指令重排序的语义在JDK 1.5中才被完全修复，此前的JDK中即使将变量声明为volatile也仍然不能完全避免重排序所导致的问题（主要是volatile变量前后的代码仍然存在重排序问题），这点也是在JDK 1.5之前的Java中无法安全地使用DCL（双锁检测）来实现单例模式的原因。 
Doug Lea列出了各种处理器架构下的内存屏障指令：http://g.oswego.edu/dl/jmm/cookbook.html。

## 对于long和double型变量的特殊规则

Java内存模型要求lock、unlock、read、load、assign、use、store、write这8个操作都具有原子性，但是对于64位的数据类型（long和double），在模型中特别定义了一条相对宽松的规定：允许虚拟机将没有被volatile修饰的64位数据的读写操作划分为两次32位的操作来进行，即允许虚拟机实现选择可以不保证64位数据类型的load、store、read和write这4个操作的原子性，这点就是所谓的long和double的非原子性协定（Nonatomic Treatment of double and long Variables）。

如果有多个线程共享一个并未声明为volatile的long或double类型的变量，并且同时对它们进行读取和修改操作，那么某些线程可能会读取到一个既非原值，也不是其他线程修改值的代表了“半个变量”的数值。

不过这种读取到“半个变量”的情况非常罕见（在目前商用Java虚拟机中不会出现），因为Java内存模型虽然允许虚拟机不把long和double变量的读写实现成原子操作，但允许虚拟机选择把这些操作实现为具有原子性的操作，而且还“强烈建议”虚拟机这样实现。在实际开发中，目前各种平台下的商用虚拟机几乎都选择把64位数据的读写操作作为原子操作来对待，因此我们在编写代码时一般不需要把用到的long和double变量专门声明为volatile。



## 原子性、可见性与有序性

介绍完Java内存模型的相关操作和规则，我们再整体回顾一下这个模型的特征。Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这3个特征来建立的，我们逐个来看一下哪些操作实现了这3个特性。

原子性（Atomicity）：由Java内存模型来直接保证的原子性变量操作包括read、load、assign、use、store和write，我们大致可以认为基本数据类型的访问读写是具备原子性的（例外就是long和double的非原子性协定，读者只要知道这件事情就可以了，无须太过在意这些几乎不会发生的例外情况）。

如果应用场景需要一个更大范围的原子性保证（经常会遇到），Java内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐式地使用这两个操作，这两个字节码指令反映到Java代码中就是同步块——synchronized关键字，因此在synchronized块之间的操作也具备原子性。

可见性（Visibility）：可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。上文在讲解volatile变量的时候我们已详细讨论过这一点。Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此，普通变量与volatile变量的区别是，volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。因此，可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。

除了volatile之外，Java还有两个关键字能实现可见性，即synchronized和final。同步块的可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）”这条规则获得的，而final关键字的可见性是指：被final修饰的字段在构造器中一旦初始化完成，并且构造器没有把“this”的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到“初始化了一半”的对象），那在其他线程中就能看见final字段的值。如代码清单12-7所示，变量i与j都具备可见性，它们无须同步就能被其他线程正确访问。

代码清单12-7　final与可见性



```
public static final int i；
public final int j；
static{
    i=0；
    //do something
}
{
    //也可以选择在构造函数中初始化
    j=0；
    //do something
} 
```



有序性（Ordering）：Java内存模型的有序性在前面讲解volatile时也详细地讨论过了，Java程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程，所有的操作都是无序的。前半句是指“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics），后半句是指“指令重排序”现象和“工作内存与主内存同步延迟”现象。

[Java语言](https://www.baidu.com/s?wd=Java语言&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这条规则决定了持有同一个锁的两个同步块只能串行地进入。

介绍完并发中3种重要的特性后，有没有发现synchronized关键字在需要这3种特性的时候都可以作为其中一种的解决方案？看起来很“万能”吧。的确，大部分的并发控制操作都能使用synchronized来完成。synchronized的“万能”也间接造就了它被程序员滥用的局面，越“万能”的并发控制，通常会伴随着越大的性能影响，这点我们将在讲解虚拟机锁优化时再介绍。



## 先行发生原则

如果Java内存模型中所有的有序性都仅仅靠volatile和synchronized来完成，那么有一些操作将会变得很烦琐，但是我们在编写Java并发代码的时候并没有感觉到这一点，这是因为Java语言中有一个“先行发生”（happens-before）的原则。这个原则非常重要，它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们可以通过几条规则一揽子地解决并发环境下两个操作之间是否可能存在冲突的所有问题。

现在就来看看“先行发生”原则指的是什么。先行发生是Java内存模型中定义的两项操作之间的偏序关系，如果说操作A先行发生于操作B，其实就是说在发生操作B之前，操作A产生的影响能被操作B观察到，“影响”包括修改了内存中共享变量的值、发送了消息、调用了方法等。这句话不难理解，但它意味着什么呢？我们可以举个例子来说明一下，如代码清单12-8中所示的这3句伪代码。

代码清单12-8　先行发生原则示例1



```
//以下操作在线程A中执行
i=1；
//以下操作在线程B中执行
j=i；
//以下操作在线程C中执行
i=2； 
```



假设线程A中的操作“i=1”先行发生于线程B的操作“j=i”，那么可以确定在线程B的操作执行后，变量j的值一定等于1，得出这个结论的依据有两个：一是根据先行发生原则，“i=1”的结果可以被观察到；二是线程C还没“登场”，线程A操作结束之后没有其他线程会修改变量i的值。现在再来考虑线程C，我们依然保持线程A和线程B之间的先行发生关系，而线程C出现在线程A和线程B的操作之间，但是线程C与线程B没有先行发生关系，那j的值会是多少呢？答案是不确定！1和2都有可能，因为线程C对变量i的影响可能会被线程B观察到，也可能不会，这时候线程B就存在读取到过期数据的风险，不具备多线程安全性。

下面是Java内存模型下一些“天然的”先行发生关系，这些先行发生关系无须任何同步器协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从下列规则推导出来的话，它们就没有顺序性保障，虚拟机可以对它们随意地进行重排序。

- 程序次序规则（Program Order Rule）：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- 管程锁定规则（Monitor Lock Rule）：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序。
- volatile变量规则（Volatile Variable Rule）：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序。
- 线程启动规则（Thread Start Rule）：Thread对象的start（）方法先行发生于此线程的每一个动作。
- 线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join（）方法结束、Thread.isAlive（）的返回值等手段检测到线程已经终止执行。
- 线程中断规则（Thread Interruption Rule）：对线程interrupt（）方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted（）方法检测到是否有中断发生。
- 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize（）方法的开始。
- 传递性（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。

Java语言无须任何同步手段保障就能成立的先行发生规则就只有上面这些了，演示一下如何使用这些规则去判定操作间是否具备顺序性，对于读写共享变量的操作来说，就是线程是否安全，读者还可以从下面这个例子中感受一下“时间上的先后顺序”与“先行发生”之间有什么不同。演示例子如代码清单12-9所示。

代码清单12-9　先行发生原则示例2



```
private int value=0；
pubilc void setValue（int value）{
    this.value=value；
}
public int getValue（）{
    return value；
} 
```



代码清单12-9中显示的是一组再普通不过的getter/setter方法，假设存在线程A和B，线程A先（时间上的先后）调用了“setValue（1）”，然后线程B调用了同一个对象的“getValue（）”，那么线程B收到的返回值是什么？

我们依次分析一下先行发生原则中的各项规则，由于两个方法分别由线程A和线程B调用，不在一个线程中，所以程序次序规则在这里不适用；由于没有同步块，自然就不会发生lock和unlock操作，所以管程锁定规则不适用；由于value变量没有被volatile关键字修饰，所以volatile变量规则不适用；后面的线程启动、终止、中断规则和对象终结规则也和这里完全没有关系。因为没有一个适用的先行发生规则，所以最后一条传递性也无从谈起，因此我们可以判定尽管线程A在操作时间上先于线程B，但是无法确定线程B中“getValue（）”方法的返回结果，换句话说，这里面的操作不是线程安全的。

那怎么修复这个问题呢？我们至少有两种比较简单的方案可以选择：要么把getter/setter方法都定义为synchronized方法，这样就可以套用管程锁定规则；要么把value定义为volatile变量，由于setter方法对value的修改不依赖value的原值，满足volatile关键字使用场景，这样就可以套用volatile变量规则来实现先行发生关系。

通过上面的例子，我们可以得出结论：一个操作“时间上的先发生”不代表这个操作会是“先行发生”，那如果一个操作“先行发生”是否就能推导出这个操作必定是“时间上的先发生”呢？很遗憾，这个推论也是不成立的，一个典型的例子就是多次提到的“指令重排序”，演示例子如代码清单12-10所示。

代码清单12-10　先行发生原则示例3

```
//以下操作在同一个线程中执行
int i=1；
int j=2； 
```

代码清单12-10的两条赋值语句在同一个线程之中，根据程序次序规则，“int i=1”的操作先行发生于“int j=2”，但是“int j=2”的代码完全可能先被处理器执行，这并不影响先行发生原则的正确性，因为我们在这条线程之中没有办法感知到这点。

上面两个例子综合起来证明了一个结论：时间先后顺序与先行发生原则之间基本没有太大的关系，所以我们衡量并发安全问题的时候不要受到时间顺序的干扰，一切必须以先行发生原则为准。



## Java与线程

并发不一定要依赖多线程（如PHP中很常见的多进程并发），但是在Java里面谈论并发，大多数都与线程脱不开关系。既然我们这本书探讨的话题是Java虚拟机的特性，那讲到Java线程，我们就从Java线程在虚拟机中的实现开始讲起。



## 线程的实现

我们知道，线程是比进程更轻量级的调度执行单位，线程的引入，可以把一个进程的资源分配和执行调度分开，各个线程既可以共享进程资源（内存地址、文件I/O等），又可以独立调度（线程是CPU调度的基本单位）。

主流的操作系统都提供了线程实现，Java语言则提供了在不同硬件和操作系统平台下对线程操作的统一处理，每个已经执行start（）且还未结束的java.lang.Thread类的实例就代表了一个线程。我们注意到Thread类与大部分的Java API有显著的差别，它的所有关键方法都是声明为Native的。在Java API中，一个Native方法往往意味着这个方法没有使用或无法使用平台无关的手段来实现（当然也可能是为了执行效率而使用Native方法，不过，通常最高效率的手段也就是平台相关的手段）。正因为如此，作者把本节的标题定为“线程的实现”而不是“Java线程的实现”。

实现线程主要有3种方式：使用内核线程实现、使用用户线程实现和使用用户线程加轻量级进程混合实现。



## 1.使用内核线程实现

内核线程（Kernel-Level Thread,KLT）就是直接由操作系统内核（Kernel，下称内核）支持的线程，这种线程由内核来完成线程切换，内核通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身，这样操作系统就有能力同时处理多件事情，支持多线程的内核就叫做多线程内核（Multi-Threads Kernel）。

程序一般不会直接去使用内核线程，而是去使用内核线程的一种高级接口——轻量级进程（Light Weight Process,LWP），轻量级进程就是我们通常意义上所讲的线程，由于每个轻量级进程都由一个内核线程支持，因此只有先支持内核线程，才能有轻量级进程。这种轻量级进程与内核线程之间1:1的关系称为一对一的线程模型，如图12-3所示。

![img](https://img-blog.csdn.net/20151107225658672)

图　12-3　轻量级进程与内核线程之间1:1的关系

由于内核线程的支持，每个轻量级进程都成为一个独立的调度单元，即使有一个轻量级进程在系统调用中阻塞了，也不会影响整个进程继续工作，但是轻量级进程具有它的局限性：首先，由于是基于内核线程实现的，所以各种线程操作，如创建、析构及同步，都需要进行系统调用。而系统调用的代价相对较高，需要在用户态（User Mode）和内核态（Kernel Mode）中来回切换。其次，每个轻量级进程都需要有一个内核线程的支持，因此轻量级进程要消耗一定的内核资源（如内核线程的栈空间），因此一个系统支持轻量级进程的数量是有限的。



## 2.使用用户线程实现

从广义上来讲，一个线程只要不是内核线程，就可以认为是用户线程（User Thread,UT），因此，从这个定义上来讲，轻量级进程也属于用户线程，但轻量级进程的实现始终是建立在内核之上的，许多操作都要进行系统调用，效率会受到限制。

而狭义上的用户线程指的是完全建立在用户空间的线程库上，系统内核不能感知线程存在的实现。用户线程的建立、同步、销毁和调度完全在用户态中完成，不需要内核的帮助。如果程序实现得当，这种线程不需要切换到内核态，因此操作可以是非常快速且低消耗的，也可以支持规模更大的线程数量，部分高性能数据库中的多线程就是由用户线程实现的。这种进程与用户线程之间1：N的关系称为一对多的线程模型，如图12-4所示。

![img](https://img-blog.csdn.net/20151107230820259)

图　12-4　进程与用户线程之间1：N的关系

使用用户线程的优势在于不需要系统内核支援，劣势也在于没有系统内核的支援，所有的线程操作都需要用户程序自己处理。线程的创建、切换和调度都是需要考虑的问题，而且由于操作系统只把处理器资源分配到进程，那诸如“阻塞如何处理”、“多处理器系统中如何将线程映射到其他处理器上”这类问题解决起来将会异常困难，甚至不可能完成。因而使用用户线程实现的程序一般都比较复杂，除了以前在不支持多线程的操作系统中（如DOS）的多线程程序与少数有特殊需求的程序外，现在使用用户线程的程序越来越少了，Java、Ruby等语言都曾经使用过用户线程，最终又都放弃使用它。



## 3.使用用户线程加轻量级进程混合实现

线程除了依赖内核线程实现和完全由用户程序自己实现之外，还有一种将内核线程与用户线程一起使用的实现方式。在这种混合实现下，既存在用户线程，也存在轻量级进程。用户线程还是完全建立在用户空间中，因此用户线程的创建、切换、析构等操作依然廉价，并且可以支持大规模的用户线程并发。而操作系统提供支持的轻量级进程则作为用户线程和内核线程之间的桥梁，这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级线程来完成，大大降低了整个进程被完全阻塞的风险。在这种混合模式中，用户线程与轻量级进程的数量比是不定的，即为N：M的关系，如图12-5所示，这种就是多对多的线程模型。

许多UNIX系列的操作系统，如Solaris、HP-UX等都提供了N：M的线程模型实现。

![img](https://img-blog.csdn.net/20151107231335494)

图　12-5　用户线程与轻量级进程之间N：M的关系



## 4.Java线程的实现

Java线程在JDK 1.2之前，是基于称为“绿色线程”（Green Threads）的用户线程实现的，而在JDK 1.2中，线程模型替换为基于操作系统原生线程模型来实现。因此，在目前的JDK版本中，操作系统支持怎样的线程模型，在很大程度上决定了Java虚拟机的线程是怎样映射的，这点在不同的平台上没有办法达成一致，虚拟机规范中也并未限定Java线程需要使用哪种线程模型来实现。线程模型只对线程的并发规模和操作成本产生影响，对Java程序的编码和运行过程来说，这些差异都是透明的。

对于Sun JDK来说，它的Windows版与Linux版都是使用一对一的线程模型实现的，一条Java线程就映射到一条轻量级进程之中，因为Windows和Linux系统提供的线程模型就是一对一的。

而在Solaris平台中，由于操作系统的线程特性可以同时支持一对一（通过Bound Threads或Alternate Libthread实现）及多对多（通过LWP/Thread Based Synchronization实现）的线程模型，因此在Solaris版的JDK中也对应提供了两个平台专有的虚拟机参数：-XX：+UseLWPSynchronization（默认值）和-XX：+UseBoundThreads来明确指定虚拟机使用哪种线程模型。

Windows下有纤程包（Fiber Package），Linux下也有NGPT（在2.4内核的年代）来实现N：M模型，但是它们都没有成为主流。



## Java线程调度

线程调度是指系统为线程分配处理器使用权的过程，主要调度方式有两种，分别是协同式线程调度（Cooperative Threads-Scheduling）和抢占式线程调度（Preemptive Threads-Scheduling）。

如果使用协同式调度的多线程系统，线程的执行时间由线程本身来控制，线程把自己的工作执行完了之后，要主动通知系统切换到另外一个线程上。协同式多线程的最大好处是实现简单，而且由于线程要把自己的事情干完后才会进行线程切换，切换操作对线程自己是可知的，所以没有什么线程同步的问题。Lua语言中的“协同例程”就是这类实现。它的坏处也很明显：线程执行时间不可控制，甚至如果一个线程编写有问题，一直不告知系统进行线程切换，那么程序就会一直阻塞在那里。很久以前的Windows 3.x系统就是使用协同式来实现多进程多任务的，相当不稳定，一个进程坚持不让出CPU执行时间就可能会导致整个系统崩溃。

如果使用抢占式调度的多线程系统，那么每个线程将由系统来分配执行时间，线程的切换不由线程本身来决定（在Java中，Thread.yield（）可以让出执行时间，但是要获取执行时间的话，线程本身是没有什么办法的）。在这种实现线程调度的方式下，线程的执行时间是系统可控的，也不会有一个线程导致整个进程阻塞的问题，Java使用的线程调度方式就是抢占式调度。与前面所说的Windows 3.x的例子相对，在Windows 9x/NT内核中就是使用抢占式来实现多进程的，当一个进程出了问题，我们还可以使用任务管理器把这个进程“杀掉”，而不至于导致系统崩溃。

虽然Java线程调度是系统自动完成的，但是我们还是可以“建议”系统给某些线程多分配一点执行时间，另外的一些线程则可以少分配一点——这项操作可以通过设置线程优先级来完成。Java语言一共设置了10个级别的线程优先级（Thread.MIN_PRIORITY至Thread.MAX_PRIORITY），在两个线程同时处于Ready状态时，优先级越高的线程越容易被系统选择执行。

不过，线程优先级并不是太靠谱，原因是Java的线程是通过映射到系统的原生线程上来实现的，所以线程调度最终还是取决于操作系统，虽然现在很多操作系统都提供线程优先级的概念，但是并不见得能与Java线程的优先级一一对应，如Solaris中有2147483648（232）种优先级，但Windows中就只有7种，比Java线程优先级多的系统还好说，中间留下一点空位就可以了，但比Java线程优先级少的系统，就不得不出现几个优先级相同的情况了，表12-1显示了Java线程优先级与Windows线程优先级之间的对应关系，Windows平台的JDK中使用了除THREAD_PRIORITY_IDLE之外的其余6种线程优先级。

![img](https://img-blog.csdn.net/20151108010346726)

上文说到“线程优先级并不是太靠谱”，不仅仅是说在一些平台上不同的优先级实际会变得相同这一点，还有其他情况让我们不能太依赖优先级：优先级可能会被系统自行改变。例如，在Windows系统中存在一个称为“优先级推进器”（Priority Boosting，当然它可以被关闭掉）的功能，它的大致作用就是当系统发现一个线程执行得特别“勤奋努力”的话，可能会越过线程优先级去为它分配执行时间。因此，我们不能在程序中通过优先级来完全准确地判断一组状态都为Ready的线程将会先执行哪一个。



## 状态转换

Java语言定义了5种线程状态，在任意一个时间点，一个线程只能有且只有其中的一种状态，这5种状态分别如下。

- 新建（New）：创建后尚未启动的线程处于这种状态。
- 运行（Runable）：Runable包括了操作系统线程状态中的Running和Ready，也就是处于此状态的线程有可能正在执行，也有可能正在等待着CPU为它分配执行时间。
- 无限期等待（Waiting）：处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显式地唤醒。以下方法会让线程陷入无限期的等待状态： 
  - 没有设置Timeout参数的Object.wait（）方法。
  - 没有设置Timeout参数的Thread.join（）方法。
  - LockSupport.park（）方法。
- 限期等待（Timed Waiting）：处于这种状态的线程也不会被分配CPU执行时间，不过无须等待被其他线程显式地唤醒，在一定时间之后它们会由系统自动唤醒。以下方法会让线程进入限期等待状态： 
  - Thread.sleep（）方法。
  - 设置了Timeout参数的Object.wait（）方法。
  - 设置了Timeout参数的Thread.join（）方法。
  - LockSupport.parkNanos（）方法。
  - LockSupport.parkUntil（）方法。
- 阻塞（Blocked）：线程被阻塞了，“阻塞状态”与“等待状态”的区别是：“阻塞状态”在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生；而“等待状态”则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这种状态。
- 结束（Terminated）：已终止线程的线程状态，线程已经结束执行。

上述5种状态在遇到特定事件发生的时候将会互相转换，它们的转换关系如图12-6所示。

![img](https://img-blog.csdn.net/20151108010839537)

图　12-6　线程状态转换关系

# 线程安全与锁优化

# 概述

在软件业发展的初期，程序编写都是以算法为核心的，程序员会把数据和过程分别作为独立的部分来考虑，数据代表问题空间中的客体，程序代码则用于处理这些数据，这种思维方式直接站在计算机的角度去抽象问题和解决问题，称为面向过程的编程思想。与此相对的是，面向对象的编程思想是站在现实世界的角度去抽象和解决问题，它把数据和行为都看做是对象的一部分，这样可以让程序员能以符合现实世界的思维方式来编写和组织程序。

面向过程的编程思想极大地提升了现代软件开发的生产效率和软件可以达到的规模，但是现实世界与计算机世界之间不可避免地存在一些差异。例如，人们很难想象现实中的对象在一项工作进行期间，会被不停地中断和切换，对象的属性（数据）可能会在中断期间被修改和变“脏”，而这些事件在计算机世界中则是很正常的事情。有时候，良好的设计原则不得不向现实做出一些让步，我们必须让程序在计算机中正确无误地运行，然后再考虑如何将代码组织得更好，让程序运行得更快。对于这部分的主题“高效并发”来讲，首先需要保证并发的正确性，然后在此基础上实现高效。本章先从如何保证并发的正确性和如何实现线程安全讲起。

# 线程安全

“线程安全”这个名称，相信稍有经验的程序员都会听说过，甚至在代码编写和走查的时候可能还会经常挂在嘴边，但是如何找到一个不太拗口的概念来定义线程安全却不是一件容易的事情，在Google中搜索它的概念，找到的是类似于“如果一个对象可以安全地被多个线程同时使用，那它就是线程安全的”这样的定义——并不能说它不正确，但是人们无法从中获取到任何有用的信息。

《Java Concurrency In Practice》的作者Brian Goetz对“线程安全”有一个比较恰当的定义：“当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的”。

这个定义比较严谨，它要求线程安全的代码都必须具备一个特征：代码本身封装了所有必要的正确性保障手段（如互斥同步等），令调用者无须关心多线程的问题，更无须自己采取任何措施来保证多线程的正确调用。这点听起来简单，但其实并不容易做到，在大多数场景中，我们都会将这个定义弱化一些，如果把“调用这个对象的行为”限定为“单次调用”，这个定义的其他描述也能够成立的话，我们就可以称它是线程安全了，为什么要弱化这个定义，现在暂且放下，稍后再详细探讨。



## Java语言中的线程安全

我们已经有了线程安全的一个抽象定义，那接下来就讨论一下在Java语言中，线程安全具体是如何体现的？有哪些操作是线程安全的？我们这里讨论的线程安全，就限定于多个线程之间存在共享数据访问这个前提，因为如果一段代码根本不会与其他线程共享数据，那么从线程安全的角度来看，程序是串行执行还是多线程执行对它来说是完全没有区别的。

为了更加深入地理解线程安全，在这里我们可以不把线程安全当做一个非真即假的二元排他选项来看待，按照线程安全的“安全程度”由强至弱来排序，我们可以将Java语言中各种操作共享的数据分为以下5类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。



## 1.不可变

在Java语言中（特指JDK 1.5以后，即Java内存模型被修正之后的Java语言），不可变（Immutable）的对象一定是线程安全的，无论是对象的方法实现还是方法的调用者，都不需要再采取任何的线程安全保障措施，只要一个不可变的对象被正确地构建出来（没有发生this引用逃逸的情况），那其外部的可见状态永远也不会改变，永远也不会看到它在多个线程之中处于不一致的状态。“不可变”带来的安全性是最简单和最纯粹的。

Java语言中，如果共享数据是一个基本数据类型，那么只要在定义时使用final关键字修饰它就可以保证它是不可变的。如果共享数据是一个对象，那就需要保证对象的行为不会对其状态产生任何影响才行，不妨想一想java.lang.String类的对象，它是一个典型的不可变对象，我们调用它的substring（）、replace（）和concat（）这些方法都不会影响它原来的值，只会返回一个新构造的字符串对象。

保证对象行为不影响自己状态的途径有很多种，其中最简单的就是把对象中带有状态的变量都声明为final，这样在构造函数结束之后，它就是不可变的，例如代码清单13-1中java.lang.Integer构造函数所示的，它通过将内部状态变量value定义为final来保障状态不变。

代码清单13-1　JDK中Integer类的构造函数



```
/**
*The value of the＜code＞Integer＜/code＞.
*@serial
*/
private final int value；
/**
*Constructs a newly allocated＜code＞Integer＜/code＞object that
*represents the specified＜code＞int＜/code＞value.
*
*@param value the value to be represented by the
*＜code＞Integer＜/code＞object.
*/
public Integer（int value）{
    this.value=value；
}
```



在Java API中符合不可变要求的类型，除了上面提到的String之外，常用的还有枚举类型，以及java.lang.Number的部分子类，如Long和Double等数值包装类型，BigInteger和BigDecimal等大数据类型；但同为Number的子类型的原子类AtomicInteger和AtomicLong则并非不可变的，读者不妨看看这两个原子类的源码，想一想为什么。



## 2.绝对线程安全

绝对的线程安全完全满足Brian Goetz给出的线程安全的定义，这个定义其实是很严格的，一个类要达到“不管运行时环境如何，调用者都不需要任何额外的同步措施”通常需要付出很大的，甚至有时候是不切实际的代价。在Java API中标注自己是线程安全的类，大多数都不是绝对的线程安全。我们可以通过Java API中一个不是“绝对线程安全”的线程安全类来看看这里的“绝对”是什么意思。

如果说java.util.Vector是一个线程安全的容器，相信所有的Java程序员对此都不会有异议，因为它的add（）、get（）和size（）这类方法都是被synchronized修饰的，尽管这样效率很低，但确实是安全的。但是，即使它所有的方法都被修饰成同步，也不意味着调用它的时候永远都不再需要同步手段了，请看一下代码清单13-2中的测试代码。

代码清单13-2　对Vector线程安全的测试



```
private static Vector<Integer> vector = new Vector<Integer>();

    public static void main(String[] args) {
        while (true) {
            for (int i = 0; i < 10; i++) {
                vector.add(i);
            }

            Thread removeThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < vector.size(); i++) {
                        vector.remove(i);
                    }
                }
            });

            Thread printThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < vector.size(); i++) {
                        System.out.println((vector.get(i)));
                    }
                }
            });

            removeThread.start();
            printThread.start();

            //不要同时产生过多的线程，否则会导致操作系统假死
            while (Thread.activeCount() > 20);
        }
    }
```



运行结果如下：



```
Exception in thread"Thread-132"java.lang.ArrayIndexOutOfBoundsException：
Array index out of range：17
at java.util.Vector.remove（Vector.java：777）
at org.fenixsoft.mulithread.VectorTest$1.run（VectorTest.java：21）
at java.lang.Thread.run（Thread.java：662）
```



很明显，尽管这里使用到的Vector的get（）、remove（）和size（）方法都是同步的，但是在多线程的环境中，如果不在方法调用端做额外的同步措施的话，使用这段代码仍然是不安全的，因为如果另一个线程恰好在错误的时间里删除了一个元素，导致序号i已经不再可用的话，再用i访问数组就会抛出一个ArrayIndexOutOfBoundsException。如果要保证这段代码能正确执行下去，我们不得不把removeThread和printThread的定义改成如代码清单13-3所示的样子。

代码清单13-3　必须加入同步以保证Vector访问的线程安全性



```
Thread removeThread = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (vector) {
                for (int i = 0; i < vector.size(); i++) {
                    vector.remove(i);
                }
            }
        }
    });

    Thread printThread = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (vector) {
                for (int i = 0; i < vector.size(); i++) {
                    System.out.println((vector.get(i)));
                }
            }
        }
    });
```



## 3.相对线程安全

相对的线程安全就是我们通常意义上所讲的线程安全，它需要保证对这个对象单独的操作是线程安全的，我们在调用的时候不需要做额外的保障措施，但是对于一些特定顺序的连续调用，就可能需要在调用端使用额外的同步手段来保证调用的正确性。上面代码清单13-2和代码清单13-3就是相对线程安全的明显的案例。

在Java语言中，大部分的线程安全类都属于这种类型，例如Vector、HashTable、Collections的synchronizedCollection（）方法包装的集合等。



## 4.线程兼容

线程兼容是指对象本身并不是线程安全的，但是可以通过在调用端正确地使用同步手段来保证对象在并发环境中可以安全地使用，我们平常说一个类不是线程安全的，绝大多数时候指的是这一种情况。Java API中大部分的类都是属于线程兼容的，如与前面的Vector和HashTable相对应的集合类ArrayList和HashMap等。



## 5.线程对立

线程对立是指无论调用端是否采取了同步措施，都无法在多线程环境中并发使用的代码。由于Java语言天生就具备多线程特性，线程对立这种排斥多线程的代码是很少出现的，而且通常都是有害的，应当尽量避免。

一个线程对立的例子是Thread类的suspend（）和resume（）方法，如果有两个线程同时持有一个线程对象，一个尝试去中断线程，另一个尝试去恢复线程，如果并发进行的话，无论调用时是否进行了同步，目标线程都是存在死锁风险的，如果suspend（）中断的线程就是即将要执行resume（）的那个线程，那就肯定要产生死锁了。也正是由于这个原因，suspend（）和resume（）方法已经被JDK声明废弃（@Deprecated）了。常见的线程对立的操作还有System.setIn（）、Sytem.setOut（）和System.runFinalizersOnExit（）等。



## 线程安全的实现方法

了解了什么是线程安全之后，紧接着的一个问题就是我们应该如何实现线程安全，这听起来似乎是一件由代码如何编写来决定的事情，确实，如何实现线程安全与代码编写有很大的关系，但虚拟机提供的同步和锁机制也起到了非常重要的作用。本节中，代码编写如何实现线程安全和虚拟机如何实现同步与锁这两者都会有所涉及，相对而言更偏重后者一些，只要读者了解了虚拟机线程安全手段的运作过程，自己去思考代码如何编写并不是一件困难的事情。



## 1.互斥同步

互斥同步（Mutual Exclusion＆Synchronization）是常见的一种并发正确性保障手段。同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一个（或者是一些，使用信号量的时候）线程使用。而互斥是实现同步的一种手段，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式。因此，在这4个字里面，互斥是因，同步是果；互斥是方法，同步是目的。

在Java中，最基本的互斥同步手段就是synchronized关键字，synchronized关键字经过编译之后，会在同步块的前后分别形成monitorenter和monitorexit这两个字节码指令，这两个字节码都需要一个reference类型的参数来指明要锁定和解锁的对象。如果Java程序中的synchronized明确指定了对象参数，那就是这个对象的reference；如果没有明确指定，那就根据synchronized修饰的是实例方法还是类方法，去取对应的对象实例或Class对象来作为锁对象。

根据虚拟机规范的要求，在执行monitorenter指令时，首先要尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加1，相应的，在执行monitorexit指令时会将锁计数器减1，当计数器为0时，锁就被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到对象锁被另外一个线程释放为止。

在虚拟机规范对monitorenter和monitorexit的行为描述中，有两点是需要特别注意的。首先，synchronized同步块对同一条线程来说是可重入的，不会出现自己把自己锁死的问题。其次，同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入。Java的线程是映射到操作系统的原生线程之上的，如果要阻塞或唤醒一个线程，都需要操作系统来帮忙完成，这就需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间。对于代码简单的同步块（如被synchronized修饰的getter（）或setter（）方法），状态转换消耗的时间有可能比用户代码执行的时间还要长。所以synchronized是Java语言中一个重量级（Heavyweight）的操作，有经验的程序员都会在确实必要的情况下才使用这种操作。而虚拟机本身也会进行一些优化，譬如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁地切入到核心态之中。

除了synchronized之外，我们还可以使用java.util.concurrent（下文称J.U.C）包中的重入锁（ReentrantLock）来实现同步，在基本用法上，ReentrantLock与synchronized很相似，他们都具备一样的线程重入特性，只是代码写法上有点区别，一个表现为API层面的互斥锁（lock（）和unlock（）方法配合try/finally语句块来完成），另一个表现为原生语法层面的互斥锁。不过，相比synchronized,ReentrantLock增加了一些高级功能，主要有以下3项：等待可中断、可实现公平锁，以及锁可以绑定多个条件。

等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。

公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized中的锁是非公平的，ReentrantLock默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。

锁绑定多个条件是指一个ReentrantLock对象可以同时绑定多个Condition对象，而在synchronized中，锁对象的wait（）和notify（）或notifyAll（）方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无须这样做，只需要多次调用newCondition（）方法即可。

如果需要使用上述功能，选用ReentrantLock是一个很好的选择，那如果是基于性能考虑呢？关于synchronized和ReentrantLock的性能问题，Brian Goetz对这两种锁在JDK 1.5与单核处理器，以及JDK 1.5与双Xeon处理器环境下做了一组吞吐量对比的实验，实验结果如图13-1和图13-2所示。

![img](https://img-blog.csdn.net/20151108215105086)

图　13-1　JDK 1.5、单核处理器下两种锁的吞吐量对比

![img](https://img-blog.csdn.net/20151108215136363)

图　13-2　JDK 1.5、双Xeon处理器下两种锁的吞吐量对比

从图13-1和图13-2可以看出，多线程环境下synchronized的吞吐量下降得非常严重，而ReentrantLock则能基本保持在同一个比较稳定的水平上。与其说ReentrantLock性能好，还不如说synchronized还有非常大的优化余地。后续的技术发展也证明了这一点，JDK 1.6中加入了很多针对锁的优化措施（13.3节我们就会讲解这些优化措施），JDK 1.6发布之后，人们就发现synchronized与ReentrantLock的性能基本上是完全持平了。因此，如果读者的程序是使用JDK 1.6或以上部署的话，性能因素就不再是选择ReentrantLock的理由了，虚拟机在未来的性能改进中肯定也会更加偏向于原生的synchronized，所以还是提倡在synchronized能实现需求的情况下，优先考虑使用synchronized来进行同步。



## 2.非阻塞同步

互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题，因此这种同步也称为阻塞同步（Blocking Synchronization）。从处理问题的方式上说，互斥同步属于一种悲观的并发策略，总是认为只要不去做正确的同步措施（例如加锁），那就肯定会出现问题，无论共享数据是否真的会出现竞争，它都要进行加锁（这里讨论的是概念模型，实际上虚拟机会优化掉很大一部分不必要的加锁）、用户态核心态转换、维护锁计数器和检查是否有被阻塞的线程需要唤醒等操作。随着硬件指令集的发展，我们有了另外一个选择：基于冲突检测的乐观并发策略，通俗地说，就是先进行操作，如果没有其他线程争用共享数据，那操作就成功了；如果共享数据有争用，产生了冲突，那就再采取其他的补偿措施（最常见的补偿措施就是不断地重试，直到成功为止），这种乐观的并发策略的许多实现都不需要把线程挂起，因此这种同步操作称为非阻塞同步（Non-Blocking Synchronization）。

为什么笔者说使用乐观并发策略需要“硬件指令集的发展”才能进行呢？因为我们需要操作和冲突检测这两个步骤具备原子性，靠什么来保证呢？如果这里再使用互斥同步来保证就失去意义了，所以我们只能靠硬件来完成这件事情，硬件保证一个从语义上看起来需要多次操作的行为只通过一条处理器指令就能完成，这类指令常用的有：

- 测试并设置（Test-and-Set）。
- 获取并增加（Fetch-and-Increment）。
- 交换（Swap）。
- 比较并交换（Compare-and-Swap，下文称CAS）。
- 加载链接/条件存储（Load-Linked/Store-Conditional，下文称LL/SC）。

其中，前面的3条是20世纪就已经存在于大多数指令集之中的处理器指令，后面的两条是现代处理器新增的，而且这两条指令的目的和功能是类似的。在IA64、x86指令集中有cmpxchg指令完成CAS功能，在sparc-TSO也有casa指令实现，而在ARM和PowerPC架构下，则需要使用一对ldrex/strex指令来完成LL/SC的功能。

CAS指令需要有3个操作数，分别是内存位置（在Java中可以简单理解为变量的内存地址，用V表示）、旧的预期值（用A表示）和新值（用B表示）。CAS指令执行时，当且仅当V符合旧预期值A时，处理器用新值B更新V的值，否则它就不执行更新，但是无论是否更新了V的值，都会返回V的旧值，上述的处理过程是一个原子操作。

在JDK 1.5之后，Java程序中才可以使用CAS操作，该操作由sun.misc.Unsafe类里面的compareAndSwapInt（）和compareAndSwapLong（）等几个方法包装提供，虚拟机在内部对这些方法做了特殊处理，即时编译出来的结果就是一条平台相关的处理器CAS指令，没有方法调用的过程，或者可以认为是无条件内联进去了。

由于Unsafe类不是提供给用户程序调用的类（Unsafe.getUnsafe（）的代码中限制了只有启动类加载器（Bootstrap ClassLoader）加载的Class才能访问它），因此，如果不采用反射手段，我们只能通过其他的Java API来间接使用它，如J.U.C包里面的整数原子类，其中的compareAndSet（）和getAndIncrement（）等方法都使用了Unsafe类的CAS操作。

我们不妨拿一段在第12章中没有解决的问题代码来看看如何使用CAS操作来避免阻塞同步，代码如代码清单12-1所示。我们曾经通过这段20个线程自增10000次的代码来证明volatile变量不具备原子性，那么如何才能让它具备原子性呢？把“race++”操作或increase（）方法用同步块包裹起来当然是一个办法，但是如果改成如代码清单13-4所示的代码，那效率将会提高许多。

代码清单13-4　Atomic的原子自增运算



```
/**
 * Atomic变量自增运算测试
 * 
 * @author zzm
*/
public class AtomicTest {

    public static AtomicInteger race = new AtomicInteger(0);

    public static void increase() {
        race.incrementAndGet();
    }

    private static final int THREADS_COUNT = 20;

    public static void main(String[] args) throws Exception {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++) {
                        increase();
                    }
                }
            });
            threads[i].start();
        }

        while (Thread.activeCount() > 1)
            Thread.yield();

        System.out.println(race);
    }
}
```



运行结果如下：

```
200000
```

使用AtomicInteger代替int后，程序输出了正确的结果，一切都要归功于incrementAndGet（）方法的原子性。它的实现其实非常简单，如代码清单13-5所示。

代码清单13-5　incrementAndGet（）方法的JDK源码



```
    /**
     * Atomically increment by one the current value.
     * @return the updated value
     */
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }
```



incrementAndGet（）方法在一个无限循环中，不断尝试将一个比当前值大1的新值赋给自己。如果失败了，那说明在执行“获取-设置”操作的时候值已经有了修改，于是再次循环进行下一次操作，直到设置成功为止。

尽管CAS看起来很美，但显然这种操作无法涵盖互斥同步的所有使用场景，并且CAS从语义上来说并不是完美的，存在这样的一个逻辑漏洞：如果一个变量V初次读取的时候是A值，并且在准备赋值的时候检查到它仍然为A值，那我们就能说它的值没有被其他线程改变过了吗？如果在这段期间它的值曾经被改成了B，后来又被改回为A，那CAS操作就会误认为它从来没有被改变过。这个漏洞称为CAS操作的“ABA”问题。J.U.C包为了解决这个问题，提供了一个带有标记的原子引用类“AtomicStampedReference”，它可以通过控制变量值的版本来保证CAS的正确性。不过目前来说这个类比较“鸡肋”，大部分情况下ABA问题不会影响程序并发的正确性，如果需要解决ABA问题，改用传统的互斥同步可能会比原子类更高效。



## 3.无同步方案

要保证线程安全，并不是一定就要进行同步，两者没有因果关系。同步只是保证共享数据争用时的正确性的手段，如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性，因此会有一些代码天生就是线程安全的，笔者简单地介绍其中的两类。

可重入代码（Reentrant Code）：这种代码也叫做纯代码（Pure Code），可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误。相对线程安全来说，可重入性是更基本的特性，它可以保证线程安全，即所有的可重入的代码都是线程安全的，但是并非所有的线程安全的代码都是可重入的。

可重入代码有一些共同的特征，例如不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数中传入、不调用非可重入的方法等。我们可以通过一个简单的原则来判断代码是否具备可重入性：如果一个方法，它的返回结果是可以预测的，只要输入了相同的数据，就都能返回相同的结果，那它就满足可重入性的要求，当然也就是线程安全的。

线程本地存储（Thread Local Storage）：如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行？如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。

符合这种特点的应用并不少见，大部分使用消费队列的架构模式（如“生产者-消费者”模式）都会将产品的消费过程尽量在一个线程中消费完，其中最重要的一个应用实例就是经典Web交互模型中的“一个请求对应一个服务器线程”（Thread-per-Request）的处理方式，这种处理方式的广泛应用使得很多Web服务端应用都可以使用线程本地存储来解决线程安全问题。

Java语言中，如果一个变量要被多线程访问，可以使用volatile关键字声明它为“易变的”；如果一个变量要被某个线程独享，Java中就没有类似C++中__declspec（thread）这样的关键字，不过还是可以通过java.lang.ThreadLocal类来实现线程本地存储的功能。每一个线程的Thread对象中都有一个ThreadLocalMap对象，这个对象存储了一组以ThreadLocal.threadLocalHashCode为键，以本地线程变量为值的K-V值对，ThreadLocal对象就是当前线程的ThreadLocalMap的访问入口，每一个ThreadLocal对象都包含了一个独一无二的threadLocalHashCode值，使用这个值就可以在线程K-V值对中找回对应的本地线程变量。



## 锁优化

高效并发是从JDK 1.5到JDK 1.6的一个重要改进，HotSpot虚拟机开发团队在这个版本上花费了大量的精力去实现各种锁优化技术，如适应性自旋（Adaptive Spinning）、锁消除（Lock Elimination）、锁粗化（Lock Coarsening）、轻量级锁（Lightweight Locking）和偏向锁（Biased Locking）等，这些技术都是为了在线程之间更高效地共享数据，以及解决竞争问题，从而提高程序的执行效率。



## 自旋锁与自适应自旋

前面我们讨论互斥同步的时候，提到了互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给系统的并发性能带来了很大的压力。同时，虚拟机的开发团队也注意到在许多应用上，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得。如果物理机器有一个以上的处理器，能让两个或以上的线程同时并行执行，我们就可以让后面请求锁的那个线程“稍等一下”，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。为了让线程等待，我们只需让线程执行一个忙循环（自旋），这项技术就是所谓的自旋锁。

自旋锁在JDK 1.4.2中就已经引入，只不过默认是关闭的，可以使用-XX：+UseSpinning参数来开启，在JDK 1.6中就已经改为默认开启了。自旋等待不能代替阻塞，且先不说对处理器数量的要求，自旋等待本身虽然避免了线程切换的开销，但它是要占用处理器时间的，因此，如果锁被占用的时间很短，自旋等待的效果就会非常好，反之，如果锁被占用的时间很长，那么自旋的线程只会白白消耗处理器资源，而不会做任何有用的工作，反而会带来性能上的浪费。因此，自旋等待的时间必须要有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程了。自旋次数的默认值是10次，用户可以使用参数-XX：PreBlockSpin来更改。

在JDK 1.6中引入了自适应的自旋锁。自适应意味着自旋的时间不再固定了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将允许自旋等待持续相对更长的时间，比如100个循环。另外，如果对于某个锁，自旋很少成功获得过，那在以后要获取这个锁时将可能省略掉自旋过程，以避免浪费处理器资源。有了自适应自旋，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测就会越来越准确，虚拟机就会变得越来越“聪明”了。



## 锁消除

锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析的数据支持，如果判断在一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。

变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是程序员自己应该是很清楚的，怎么会在明知道不存在数据争用的情况下要求同步呢？答案是有许多同步措施并不是程序员自己加入的，同步的代码在Java程序中的普遍程度也许超过了大部分读者的想象。我们来看看代码清单13-6中的例子，这段非常简单的代码仅仅是输出3个字符串相加的结果，无论是源码字面上还是程序语义上都没有同步。

代码清单13-6　一段看起来没有同步的代码

```
public String concatString（String s1，String s2，String s3）{
    return s1+s2+s3；
}
```

我们也知道，由于String是一个不可变的类，对字符串的连接操作总是通过生成新的String对象来进行的，因此Javac编译器会对String连接做自动优化。在JDK 1.5之前，会转化为StringBuffer对象的连续append（）操作，在JDK 1.5及以后的版本中，会转化为StringBuilder对象的连续append（）操作，即代码清单13-6中的代码可能会变成代码清单13-7的样子。

代码清单13-7　Javac转化后的字符串连接操作



```
public String concatString（String s1，String s2，String s3）{
    StringBuffer sb=new StringBuffer（）；
    sb.append（s1）；
    sb.append（s2）；
    sb.append（s3）；
    return sb.toString（）；
} 
```



现在大家还认为这段代码没有涉及同步吗？每个StringBuffer.append（）方法中都有一个同步块，锁就是sb对象。虚拟机观察变量sb，很快就会发现它的动态作用域被限制在concatString（）方法内部。也就是说，sb的所有引用永远不会“逃逸”到concatString（）方法之外，其他线程无法访问到它，因此，虽然这里有锁，但是可以被安全地消除掉，在即时编译之后，这段代码就会忽略掉所有的同步而直接执行了。

客观地说，既然谈到锁消除与逃逸分析，那虚拟机就不可能是JDK 1.5之前的版本，实际上会转化为非线程安全的StringBuilder来完成字符串拼接，并不会加锁，但这也不影响笔者用这个例子证明Java对象中同步的普遍性。



## 锁粗化

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小——只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待锁的线程也能尽快拿到锁。

大部分情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加[锁和](https://www.baidu.com/s?wd=锁和&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。

代码清单13-7中连续的append（）方法就属于这类情况。如果虚拟机探测到有这样一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展（粗化）到整个操作序列的外部，以代码清单13-7为例，就是扩展到第一个append（）操作之前直至最后一个append（）操作之后，这样只需要加锁一次就可以了。



## 轻量级锁

轻量级锁是JDK 1.6之中加入的新型锁机制，它名字中的“轻量级”是相对于使用操作系统互斥量来实现的传统锁而言的，因此传统的锁机制就称为“重量级”锁。首先需要强调一点的是，轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

要理解轻量级锁，以及后面会讲到的偏向锁的原理和运作过程，必须从HotSpot虚拟机的对象（对象头部分）的内存布局开始介绍。HotSpot虚拟机的对象头（Object Header）分为两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄（Generational GC Age）等，这部分数据的长度在32位和[64位](https://www.baidu.com/s?wd=64位&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)的虚拟机中分别为32bit和64bit，官方称它为“Mark Word”，它是实现轻量级锁和偏向锁的关键。另外一部分用于存储指向方法区对象类型数据的指针，如果是数组对象的话，还会有一个额外的部分用于存储数组长度。

对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。例如，在32位的HotSpot虚拟机中对象未被锁定的状态下，Mark Word的32bit空间中的25bit用于存储对象哈希码（HashCode），4bit用于存储对象分代年龄，2bit用于存储锁标志位，1bit固定为0，在其他状态（轻量级锁定、重量级锁定、GC标记、可偏向）下对象的存储内容见表13-1。

![img](https://img-blog.csdn.net/20151108232606441)

简单地介绍了对象的内存布局后，我们把话题返回到轻量级锁的执行过程上。在代码进入同步块的时候，如果此同步对象没有被锁定（锁标志位为“01”状态），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word），这时候线程堆栈与对象头的状态如图13-3所示。

![img](https://img-blog.csdn.net/20151108232915022)

图　13-3　轻量级锁CAS操作之前堆栈与对象的状态

然后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针。如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位（Mark Word的最后2bit）将转变为“00”，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如图13-4所示。

![img](https://img-blog.csdn.net/20151108233202466)

图　13-4　轻量级锁CAS操作之后堆栈与对象的状态

如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果只说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。

上面描述的是轻量级锁的加锁过程，它的解锁过程也是通过CAS操作来进行的，如果对象的Mark Word仍然指向着线程的锁记录，那就用CAS操作把对象当前的Mark Word和线程中复制的Displaced Mark Word替换回来，如果替换成功，整个同步过程就完成了。如果替换失败，说明有其他线程尝试过获取该锁，那就要在释放锁的同时，唤醒被挂起的线程。

轻量级锁能提升程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用CAS操作避免了使用互斥量的开销，但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。



## 偏向锁

偏向锁也是JDK 1.6中引入的一项锁优化，它的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，连CAS操作都不做了。

偏向锁的“偏”，就是偏心的“偏”、偏袒的“偏”，它的意思是这个锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。

如果读懂了前面轻量级锁中关于对象头Mark Word与线程之间的操作过程，那偏向锁的原理理解起来就会很简单。假设当前虚拟机启用了偏向锁（启用参数-XX：+UseBiasedLocking，这是JDK 1.6的默认值），那么，当锁对象第一次被线程获取的时候，虚拟机将会把对象头中的标志位设为“01”，即偏向模式。同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中，如果CAS操作成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作（例如Locking、Unlocking及对Mark Word的Update等）。

当有另外一个线程去尝试获取这个锁时，偏向模式就宣告结束。根据锁对象目前是否处于被锁定的状态，撤销偏向（Revoke Bias）后恢复到未锁定（标志位为“01”）或轻量级锁定（标志位为“00”）的状态，后续的同步操作就如上面介绍的轻量级锁那样执行。偏向锁、轻量级锁的状态转化及对象Mark Word的关系如图13-5所示。

![img](https://img-blog.csdn.net/20151108234251254)

图　13-5　偏向锁、轻量级锁的状态转化及对象Mark Word的关系

偏向锁可以提高带有同步但无竞争的程序性能。它同样是一个带有效益权衡（Trade Off）性质的优化，也就是说，它并不一定总是对程序运行有利，如果程序中大多数的锁总是被多个不同的线程访问，那偏向模式就是多余的。在具体问题具体分析的前提下，有时候使用参数-XX：-UseBiasedLocking来禁止偏向锁优化反而可以提升性能。

# GC 算法(实现篇)

学习了GC算法的相关概念之后, 我们将介绍在JVM中这些算法的具体实现。首先要记住的是, 大多数JVM都需要使用两种不同的GC算法 —— 一种用来清理年轻代, 另一种用来清理老年代。

我们可以选择JVM内置的各种算法。如果不通过参数明确指定垃圾收集算法, 则会使用宿主平台的默认实现。本章会详细介绍各种算法的实现原理。

下面是关于Java 8中各种组合的垃圾收集器概要列表,对于之前的Java版本来说,可用组合会有一些不同:

| Young               | Tenured      | JVM options                              |
| ------------------- | ------------ | ---------------------------------------- |
| Incremental(增量GC) | Incremental  | -Xincgc                                  |
| Serial              | Serial       | -XX:+UseSerialGC                         |
| Parallel Scavenge   | Serial       | -XX:+UseParallelGC -XX:-UseParallelOldGC |
| Parallel New        | Serial       | N/A                                      |
| Serial              | Parallel Old | N/A                                      |
| Parallel Scavenge   | Parallel Old | -XX:+UseParallelGC -XX:+UseParallelOldGC |
| Parallel New        | Parallel Old | N/A                                      |
| Serial              | CMS          | -XX:-UseParNewGC -XX:+UseConcMarkSweepGC |
| Parallel Scavenge   | CMS          | N/A                                      |
| Parallel New        | CMS          | -XX:+UseParNewGC -XX:+UseConcMarkSweepGC |
| G1                  | -XX:+UseG1GC |                                          |

看起来有些复杂, 不用担心。主要使用的是上表中黑体字表示的这四种组合。其余的要么是被废弃(deprecated), 要么是不支持或者是不太适用于生产环境。所以,接下来我们只介绍下面这些组合及其工作原理:

- 年轻代和老年代的串行GC(Serial GC)
- 年轻代和老年代的并行GC(Parallel GC)
- 年轻代的并行GC(Parallel New) + 老年代的CMS(Concurrent Mark and Sweep)
- G1, 负责回收年轻代和老年代



## Serial GC(串行GC)

Serial GC 对年轻代使用 mark-copy(标记-复制) 算法, 对老年代使用 mark-sweep-compact(标记-清除-整理)算法. 顾名思义, 两者都是单线程的垃圾收集器,不能进行并行处理。两者都会触发全线暂停(STW),停止所有的应用线程。

因此这种GC算法不能充分利用多核CPU。不管有多少CPU内核, JVM 在垃圾收集时都只能使用单个核心。

要启用此款收集器, 只需要指定一个JVM启动参数即可,同时对年轻代和老年代生效:

```avrasm
java -XX:+UseSerialGC com.mypackages.MyExecutableClass
```

该选项只适合几百MB堆内存的JVM,而且是单核CPU时比较有用。 对于服务器端来说, 因为一般是多个CPU内核, 并不推荐使用, 除非确实需要限制JVM所使用的资源。大多数服务器端应用部署在多核平台上, 选择 Serial GC 就表示人为的限制系统资源的使用。 导致的就是资源闲置, 多的CPU资源也不能用来降低延迟,也不能用来增加吞吐量。

下面让我们看看Serial GC的垃圾收集日志, 并从中提取什么有用的信息。为了打开GC日志记录, 我们使用下面的JVM启动参数:

```diff
-XX:+PrintGCDetails -XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
```

产生的GC日志输出类似这样(为了排版,已手工折行):

```yaml
2015-05-26T14:45:37.987-0200:
        151.126: [GC (Allocation Failure)
        151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs]
        1619346K->1273247K(2027264K), 0.0585007 secs]
    [Times: user=0.06 sys=0.00, real=0.06 secs]
2015-05-26T14:45:59.690-0200:
        172.829: [GC (Allocation Failure)
        172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]
        172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs]
        1832479K->755802K(2027264K),
        [Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs]
    [Times: user=0.18 sys=0.00, real=0.18 secs]
```

此GC日志片段展示了在JVM中发生的很多事情。 实际上,在这段日志中产生了两个GC事件, 其中一次清理的是年轻代,另一次清理的是整个堆内存。让我们先来分析前一次GC,其在年轻代中产生。



### Minor GC(小型GC)

以下代码片段展示了清理年轻代内存的GC事件:

```
2015-05-26T14:45:37.987-02001 : 151.12622 : [ GC3 (Allocation Failure4 151.126: 

[DefNew5 : 629119K->69888K6 (629120K)7 , 0.0584157 secs] 1619346K->1273247K8 

(2027264K)9, 0.0585007 secs10] [Times: user=0.06 sys=0.00, real=0.06 secs]11

2015-05-26T14:45:37.987-0200 – GC事件开始的时间. 其中-0200表示西二时区,而中国所在的东8区为 +0800。
151.126 – GC事件开始时,相对于JVM启动时的间隔时间,单位是秒。
GC – 用来区分 Minor GC 还是 Full GC 的标志。GC表明这是一次小型GC(Minor GC)
Allocation Failure – 触发 GC 的原因。本次GC事件, 是由于年轻代中没有空间来存放新的数据结构引起的。
DefNew – 垃圾收集器的名称。这个神秘的名字表示的是在年轻代中使用的: 单线程, 标记-复制(mark-copy), 全线暂停(STW) 垃圾收集器。
629119K->69888K – 在垃圾收集之前和之后年轻代的使用量。
(629120K) – 年轻代总的空间大小。
1619346K->1273247K – 在垃圾收集之前和之后整个堆内存的使用情况。
(2027264K) – 可用堆的总空间大小。
0.0585007 secs – GC事件持续的时间,以秒为单位。
[Times: user=0.06 sys=0.00, real=0.06 secs] – GC事件的持续时间, 通过三个部分来衡量:
user – 在此次垃圾回收过程中, 所有 GC线程所消耗的CPU时间之和。
sys – GC过程中中操作系统调用和系统等待事件所消耗的时间。
real – 应用程序暂停的时间。因为串行垃圾收集器(Serial Garbage Collector)只使用单线程, 因此 real time 等于 user 和 system 时间的总和。
```

可以从上面的日志片段了解到, 在GC事件中,JVM 的内存使用情况发生了怎样的变化。此次垃圾收集之前, 堆内存总的使用量为 1,619,346K。其中,年轻代使用了 629,119K。可以算出,老年代使用量为 990,227K。

更重要的信息蕴含在下一批数字中, 垃圾收集之后, 年轻代的使用量减少了 559,231K, 但堆内存的总体使用量只下降了 346,099K。 从中可以算出,有 213,132K 的对象从年轻代提升到了老年代。

此次GC事件也可以用下面的示意图来说明, 显示的是GC开始之前, 以及刚刚结束之后, 这两个时间点内存使用情况的快照:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225001.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225001.png)



### Full GC(完全GC)

理解第一次 minor GC 事件后,让我们看看日志中的第二次GC事件:

> `2015-05-26T14:45:59.690-0200`1 : `172.829`2 : [GC (Allocation Failure 172.829: 
>
> `[DefNew: 629120K->629120K(629120K), 0.0000372 secs`3] 172.829:[`Tenured`4: 
>
> `1203359K->755802K`5 `(1398144K)`6, `0.1855567 secs`7 ] `1832479K->755802K`8 
>
> `(2027264K)`9, `[Metaspace: 6741K->6741K(1056768K)]`10 
>
> `[Times: user=0.18 sys=0.00, real=0.18 secs]`11

\>

> 1. `2015-05-26T14:45:59.690-0200` – GC事件开始的时间. 其中`-0200`表示西二时区,而中国所在的东8区为 `+0800`。
> 2. `172.829` – GC事件开始时,相对于JVM启动时的间隔时间,单位是秒。
> 3. `[DefNew: 629120K->629120K(629120K), 0.0000372 secs` – 与上面的示例类似, 因为内存分配失败,在年轻代中发生了一次 minor GC。此次GC同样使用的是 DefNew 收集器, 让年轻代的使用量从 629120K 降为 0。注意,JVM对此次GC的报告有些问题,误将年轻代认为是完全填满的。此次垃圾收集消耗了 0.0000372秒。
> 4. `Tenured` – 用于清理老年代空间的垃圾收集器名称。Tenured 表明使用的是单线程的全线暂停垃圾收集器, 收集算法为 标记-清除-整理(mark-sweep-compact )。
> 5. `1203359K->755802K` – 在垃圾收集之前和之后老年代的使用量。
> 6. `(1398144K)` – 老年代的总空间大小。
> 7. `0.1855567 secs` – 清理老年代所花的时间。
> 8. `1832479K->755802K` – 在垃圾收集之前和之后,整个堆内存的使用情况。
> 9. `(2027264K)` – 可用堆的总空间大小。
> 10. `[Metaspace: 6741K->6741K(1056768K)]` – 关于 Metaspace 空间, 同样的信息。可以看出, 此次GC过程中 Metaspace 中没有收集到任何垃圾。
> 11. `[Times: user=0.18 sys=0.00, real=0.18 secs]` – GC事件的持续时间, 通过三个部分来衡量: 
>
> \* user – 在此次垃圾回收过程中, 所有 GC线程所消耗的CPU时间之和。
> \* sys – GC过程中中操作系统调用和系统等待事件所消耗的时间。
> \* real – 应用程序暂停的时间。因为串行垃圾收集器(Serial Garbage Collector)只使用单线程, 因此 real time 等于 user 和 system 时间的总和。

和 Minor GC 相比,最明显的区别是 —— 在此次GC事件中, 除了年轻代, 还清理了老年代和Metaspace. 在GC事件开始之前, 以及刚刚结束之后的内存布局,可以用下面的示意图来说明:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225002.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225002.png)



## Parallel GC(并行GC)

并行垃圾收集器这一类组合, 在年轻代使用 标记-复制(mark-copy)算法, 在老年代使用 标记-清除-整理(mark-sweep-compact)算法。年轻代和老年代的垃圾回收都会触发STW事件,暂停所有的应用线程来执行垃圾收集。两者在执行 标记和 复制/整理阶段时都使用多个线程, 因此得名“(Parallel)”。通过并行执行, 使得GC时间大幅减少。

通过命令行参数 `-XX:ParallelGCThreads=NNN` 来指定 GC 线程数。 其默认值为CPU内核数。

可以通过下面的任意一组命令行参数来指定并行GC:

```avrasm
java -XX:+UseParallelGC com.mypackages.MyExecutableClass
java -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
java -XX:+UseParallelGC -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
```

并行垃圾收集器适用于多核服务器,主要目标是增加吞吐量。因为对系统资源的有效使用,能达到更高的吞吐量:

- 在GC期间, 所有 CPU 内核都在并行清理垃圾, 所以暂停时间更短
- 在两次GC周期的间隔期, 没有GC线程在运行,不会消耗任何系统资源

另一方面, 因为此GC的所有阶段都不能中断, 所以并行GC很容易出现长时间的停顿. 如果延迟是系统的主要目标, 那么就应该选择其他垃圾收集器组合。

> 译者注: 长时间卡顿的意思是，此GC启动之后，属于一次性完成所有操作, 于是单次 pause 的时间会较长。

让我们看看并行垃圾收集器的GC日志长什么样, 从中我们可以得到哪些有用信息。下面的GC日志中显示了一次 minor GC 暂停 和一次 major GC 暂停:

```yaml
2015-05-26T14:27:40.915-0200: 116.115: [GC (Allocation Failure)
        [PSYoungGen: 2694440K->1305132K(2796544K)]
    9556775K->8438926K(11185152K)
    , 0.2406675 secs]
    [Times: user=1.77 sys=0.01, real=0.24 secs]
2015-05-26T14:27:41.155-0200: 116.356: [Full GC (Ergonomics)
        [PSYoungGen: 1305132K->0K(2796544K)]
        [ParOldGen: 7133794K->6597672K(8388608K)] 8438926K->6597672K(11185152K),
        [Metaspace: 6745K->6745K(1056768K)]
    , 0.9158801 secs]
    [Times: user=4.49 sys=0.64, real=0.92 secs]
```



### Minor GC(小型GC)

第一次GC事件表示发生在年轻代的垃圾收集:

> `2015-05-26T14:27:40.915-0200`1: `116.115`2: `[ GC`3 (`Allocation Failure`4)
>
> [`PSYoungGen`5: `2694440K->1305132K`6 `(2796544K)`7] `9556775K->8438926K`8
>
> `(11185152K)`9, `0.2406675 secs`10]
>
> `[Times: user=1.77 sys=0.01, real=0.24 secs]`11

\>

> 1. `2015-05-26T14:27:40.915-0200` – GC事件开始的时间. 其中`-0200`表示西二时区,而中国所在的东8区为 `+0800`。
> 2. `116.115` – GC事件开始时,相对于JVM启动时的间隔时间,单位是秒。
> 3. `GC` – 用来区分 Minor GC 还是 Full GC 的标志。`GC`表明这是一次小型GC(Minor GC)
> 4. `Allocation Failure` – 触发垃圾收集的原因。本次GC事件, 是由于年轻代中没有适当的空间存放新的数据结构引起的。
> 5. `PSYoungGen` – 垃圾收集器的名称。这个名字表示的是在年轻代中使用的: 并行的 标记-复制(mark-copy), 全线暂停(STW) 垃圾收集器。
> 6. `2694440K->1305132K` – 在垃圾收集之前和之后的年轻代使用量。
> 7. `(2796544K)` – 年轻代的总大小。
> 8. `9556775K->8438926K` – 在垃圾收集之前和之后整个堆内存的使用量。
> 9. `(11185152K)` – 可用堆的总大小。
> 10. `0.2406675 secs` – GC事件持续的时间,以秒为单位。
> 11. `[Times: user=1.77 sys=0.01, real=0.24 secs]` – GC事件的持续时间, 通过三个部分来衡量: 
>
> \* user – 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间。
> \* sys – GC过程中中操作系统调用和系统等待事件所消耗的时间。
> \* real – 应用程序暂停的时间。在 Parallel GC 中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用了8个线程。 请注意,总有一定比例的处理过程是不能并行进行的。

所以,可以简单地算出, 在垃圾收集之前, 堆内存总使用量为 9,556,775K。 其中年轻代为 2,694,440K。同时算出老年代使用量为 6,862,335K. 在垃圾收集之后, 年轻代使用量减少为 1,389,308K, 但总的堆内存使用量只减少了 `1,117,849K`。这表示有大小为 271,459K 的对象从年轻代提升到老年代。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225003.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225003.png)



### Full GC(完全GC)

学习了并行GC如何清理年轻代之后, 下面介绍清理整个堆内存的GC日志以及如何进行分析:

> `2015-05-26T14:27:41.155-0200` : `116.356` : [`Full GC` (`Ergonomics`)
>
> ```
> [PSYoungGen: 1305132K->0K(2796544K)]` [`ParOldGen` :`7133794K->6597672K
> ```
>
> `(8388608K)`] `8438926K->6597672K` `(11185152K)`, 
>
> `[Metaspace: 6745K->6745K(1056768K)]`, `0.9158801 secs`,
>
> ```
> [Times: user=4.49 sys=0.64, real=0.92 secs]
> ```
>
> 1. `2015-05-26T14:27:41.155-0200` – GC事件开始的时间. 其中`-0200`表示西二时区,而中国所在的东8区为 `+0800`。
> 2. `116.356` – GC事件开始时,相对于JVM启动时的间隔时间,单位是秒。 我们可以看到, 此次事件在前一次 MinorGC完成之后立刻就开始了。
> 3. `Full GC` – 用来表示此次是 Full GC 的标志。`Full GC`表明本次清理的是年轻代和老年代。
> 4. `Ergonomics` – 触发垃圾收集的原因。`Ergonomics` 表示JVM内部环境认为此时可以进行一次垃圾收集。
> 5. `[PSYoungGen: 1305132K->0K(2796544K)]` – 和上面的示例一样, 清理年轻代的垃圾收集器是名为 “PSYoungGen” 的STW收集器, 采用标记-复制(mark-copy)算法。 年轻代使用量从 1305132K 变为 `0`, 一般 Full GC 的结果都是这样。
> 6. `ParOldGen` – 用于清理老年代空间的垃圾收集器类型。在这里使用的是名为 ParOldGen 的垃圾收集器, 这是一款并行 STW垃圾收集器, 算法为 标记-清除-整理(mark-sweep-compact)。
> 7. `7133794K->6597672K` – 在垃圾收集之前和之后老年代内存的使用情况。
> 8. `(8388608K)` – 老年代的总空间大小。
> 9. `8438926K->6597672K` – 在垃圾收集之前和之后堆内存的使用情况。
> 10. `(11185152K)` – 可用堆内存的总容量。
> 11. `[Metaspace: 6745K->6745K(1056768K)]` – 类似的信息,关于 Metaspace 空间的。可以看出, 在GC事件中 Metaspace 里面没有回收任何对象。
> 12. `0.9158801 secs` – GC事件持续的时间,以秒为单位。
> 13. `[Times: user=4.49 sys=0.64, real=0.92 secs]` – GC事件的持续时间, 通过三个部分来衡量: 
>
> \* user – 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间。
> \* sys – GC过程中中操作系统调用和系统等待事件所消耗的时间。
> \* real – 应用程序暂停的时间。在 Parallel GC 中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用了8个线程。 请注意,总有一定比例的处理过程是不能并行进行的。

同样,和 Minor GC 的区别是很明显的 —— 在此次GC事件中, 除了年轻代, 还清理了老年代和 Metaspace. 在GC事件前后的内存布局如下图所示:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225004.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225004.png)



## Concurrent Mark and Sweep(并发标记-清除)

CMS的官方名称为 “Mostly Concurrent Mark and Sweep Garbage Collector”(主要并发-标记-清除-垃圾收集器). 其对年轻代采用并行 STW方式的 [mark-copy (标记-复制)算法](https://plumbr.eu/handbook/garbage-collection-algorithms/removing-unused-objects/copy), 对老年代主要使用并发 [mark-sweep (标记-清除)算法](https://plumbr.eu/handbook/garbage-collection-algorithms/removing-unused-objects/sweep)。

CMS的设计目标是避免在老年代垃圾收集时出现长时间的卡顿。主要通过两种手段来达成此目标。

- 第一, 不对老年代进行整理, 而是使用空闲列表(free-lists)来管理内存空间的回收。
- 第二, 在 mark-and-sweep (标记-清除) 阶段的大部分工作和应用线程一起并发执行。

也就是说, 在这些阶段并没有明显的应用线程暂停。但值得注意的是, 它仍然和应用线程争抢CPU时间。默认情况下, CMS 使用的并发线程数等于CPU内核数的 `1/4`。

通过以下选项来指定CMS垃圾收集器:

```avrasm
java -XX:+UseConcMarkSweepGC com.mypackages.MyExecutableClass
```

如果服务器是多核CPU，并且主要调优目标是降低延迟, 那么使用CMS是个很明智的选择. 减少每一次GC停顿的时间,会直接影响到终端用户对系统的体验, 用户会认为系统非常灵敏。 因为多数时候都有部分CPU资源被GC消耗, 所以在CPU资源受限的情况下,CMS会比并行GC的吞吐量差一些。

和前面的GC算法一样, 我们先来看看CMS算法在实际应用中的GC日志, 其中包括一次 minor GC, 以及一次 major GC 停顿:

```sql
2015-05-26T16:23:07.219-0200: 64.322: [GC (Allocation Failure) 64.322:
            [ParNew: 613404K->68068K(613440K), 0.1020465 secs]
            10885349K->10880154K(12514816K), 0.1021309 secs]
        [Times: user=0.78 sys=0.01, real=0.11 secs]
2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark)
            [1 CMS-initial-mark: 10812086K(11901376K)]
            10887844K(12514816K), 0.0001997 secs]
        [Times: user=0.00 sys=0.00, real=0.00 secs]
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs]
        [Times: user=0.07 sys=0.00, real=0.03 secs]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean:0.016/0.016 secs][Times: user=0.02 sys=0.00, real=0.02 secs]2015-05-26T16:23:07.373-0200:64.476:[CMS-concurrent-abortable-preclean-start]2015-05-26T16:23:08.446-0200:65.550:[CMS-concurrent-abortable-preclean:0.167/1.074 secs][Times: user=0.20 sys=0.00, real=1.07 secs]2015-05-26T16:23:08.447-0200:65.550:[GC (CMS FinalRemark)[YG occupancy:387920 K (613440 K)]65.550:[Rescan(parallel),0.0085125 secs]65.559:[weak refs processing,0.0000243 secs]65.559:[class unloading,0.0013120 secs]65.560:[scrub symbol table,0.0008345 secs]65.561:[scrub string table,0.0001759 secs][1 CMS-remark:10812086K(11901376K)]11200006K(12514816K),0.0110730 secs][Times: user=0.06 sys=0.00, real=0.01 secs]2015-05-26T16:23:08.458-0200:65.561:[CMS-concurrent-sweep-start]2015-05-26T16:23:08.485-0200:65.588:[CMS-concurrent-sweep:0.027/0.027 secs][Times: user=0.03 sys=0.00, real=0.03 secs]2015-05-26T16:23:08.485-0200:65.589:[CMS-concurrent-reset-start]2015-05-26T16:23:08.497-0200:65.601:[CMS-concurrent-reset:0.012/0.012 secs][Times: user=0.01 sys=0.00, real=0.01 secs]
```



### Minor GC(小型GC)

日志中的第一次GC事件是清理年轻代的小型GC(Minor GC)。让我们来分析 CMS 垃圾收集器的行为:

> `2015-05-26T16:23:07.219-0200`: `64.322`:[`GC`(`Allocation Failure`) 64.322: 
>
> [`ParNew`: `613404K->68068K``(613440K)`, `0.1020465 secs`]
>
> `10885349K->10880154K``(12514816K)`, `0.1021309 secs`]
>
> ```
> [Times: user=0.78 sys=0.01, real=0.11 secs]
> ```

\>

> 1. `2015-05-26T16:23:07.219-0200` – GC事件开始的时间. 其中`-0200`表示西二时区,而中国所在的东8区为 `+0800`。
> 2. `64.322` – GC事件开始时,相对于JVM启动时的间隔时间,单位是秒。
> 3. `GC` – 用来区分 Minor GC 还是 Full GC 的标志。`GC`表明这是一次小型GC(Minor GC)
> 4. `Allocation Failure` – 触发垃圾收集的原因。本次GC事件, 是由于年轻代中没有适当的空间存放新的数据结构引起的。
> 5. `ParNew` – 垃圾收集器的名称。这个名字表示的是在年轻代中使用的: 并行的 标记-复制(mark-copy), 全线暂停(STW)垃圾收集器, 专门设计了用来配合老年代使用的 Concurrent Mark & Sweep 垃圾收集器。
> 6. `613404K->68068K` – 在垃圾收集之前和之后的年轻代使用量。
> 7. `(613440K)` – 年轻代的总大小。
> 8. `0.1020465 secs` – 垃圾收集器在 w/o final cleanup 阶段消耗的时间
> 9. `10885349K->10880154K` – 在垃圾收集之前和之后堆内存的使用情况。
> 10. `(12514816K)` – 可用堆的总大小。
> 11. `0.1021309 secs` – 垃圾收集器在标记和复制年轻代存活对象时所消耗的时间。包括和ConcurrentMarkSweep收集器的通信开销, 提升存活时间达标的对象到老年代,以及垃圾收集后期的一些最终清理。
> 12. `[Times: user=0.78 sys=0.01, real=0.11 secs]` – GC事件的持续时间, 通过三个部分来衡量: 
>
> \* user – 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间。
> \* sys – GC过程中中操作系统调用和系统等待事件所消耗的时间。
> \* real – 应用程序暂停的时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有固定比例的处理过程是不能并行化的。

从上面的日志可以看出,在GC之前总的堆内存使用量为 10,885,349K, 年轻代的使用量为 613,404K。这意味着老年代使用量等于 10,271,945K。GC之后,年轻代的使用量减少了 545,336K, 而总的堆内存使用只下降了 5,195K。可以算出有 540,141K 的对象从年轻代提升到老年代。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225005.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225005.png)



### Full GC(完全GC)

现在, 我们已经熟悉了如何解读GC日志, 接下来将介绍一种完全不同的日志格式。下面这一段很长很长的日志, 就是CMS对老年代进行垃圾收集时输出的各阶段日志。为了简洁,我们对这些阶段逐个介绍。 首先来看CMS收集器整个GC事件的日志:

```sql
2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark)
        [1 CMS-initial-mark: 10812086K(11901376K)]
    10887844K(12514816K), 0.0001997 secs]
    [Times: user=0.00 sys=0.00, real=0.00 secs]
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs]
    [Times: user=0.07 sys=0.00, real=0.03 secs]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs]
    [Times: user=0.02 sys=0.00, real=0.02 secs]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean:0.167/1.074 secs][Times: user=0.20 sys=0.00, real=1.07 secs]2015-05-26T16:23:08.447-0200:65.550:[GC (CMS FinalRemark)[YG occupancy:387920 K (613440 K)]65.550:[Rescan(parallel),0.0085125 secs]65.559:[weak refs processing,0.0000243 secs]65.559:[class unloading,0.0013120 secs]65.560:[scrub symbol table,0.0008345 secs]65.561:[scrub string table,0.0001759 secs][1 CMS-remark:10812086K(11901376K)]11200006K(12514816K),0.0110730 secs][Times: user=0.06 sys=0.00, real=0.01 secs]2015-05-26T16:23:08.458-0200:65.561:[CMS-concurrent-sweep-start]2015-05-26T16:23:08.485-0200:65.588:[CMS-concurrent-sweep:0.027/0.027 secs][Times: user=0.03 sys=0.00, real=0.03 secs]2015-05-26T16:23:08.485-0200:65.589:[CMS-concurrent-reset-start]2015-05-26T16:23:08.497-0200:65.601:[CMS-concurrent-reset:0.012/0.012 secs][Times: user=0.01 sys=0.00, real=0.01 secs]
```

只是要记住 —— 在实际情况下, 进行老年代的并发回收时, 可能会伴随着多次年轻代的小型GC. 在这种情况下, 大型GC的日志中就会掺杂着多次小型GC事件, 像前面所介绍的一样。

阶段 1: Initial Mark(初始标记). 这是第一次STW事件。 此阶段的目标是标记老年代中所有存活的对象, 包括 GC ROOR 的直接引用, 以及由年轻代中存活对象所引用的对象。 后者也非常重要, 因为老年代是独立进行回收的。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225006.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225006.png)

> `2015-05-26T16:23:07.321-0200: 64.42`1: [GC (`CMS Initial Mark`1
>
> [1 CMS-initial-mark: `10812086K`1`(11901376K)`1] `10887844K`1`(12514816K)`1,
>
> `0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]`1
>
> 1. `2015-05-26T16:23:07.321-0200: 64.42` – GC事件开始的时间. 其中 `-0200` 是时区,而中国所在的东8区为 +0800。 而 64.42 是相对于JVM启动的时间。 下面的其他阶段也是一样,所以就不再重复介绍。
> 2. `CMS Initial Mark` – 垃圾回收的阶段名称为 “Initial Mark”。 标记所有的 GC Root。
> 3. `10812086K` – 老年代的当前使用量。
> 4. `(11901376K)` – 老年代中可用内存总量。
> 5. `10887844K` – 当前堆内存的使用量。
> 6. `(12514816K)` – 可用堆的总大小。
> 7. `0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]` – 此次暂停的持续时间, 以 user, system 和 real time 3个部分进行衡量。

阶段 2: Concurrent Mark(并发标记). 在此阶段, 垃圾收集器遍历老年代, 标记所有的存活对象, 从前一阶段 “Initial Mark” 找到的 root 根开始算起。 顾名思义, “并发标记”阶段, 就是与应用程序同时运行,不用暂停的阶段。 请注意, 并非所有老年代中存活的对象都在此阶段被标记, 因为在标记过程中对象的引用关系还在发生变化。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225007.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225007.png)

在上面的示意图中, “Current object” 旁边的一个引用被标记线程并发删除了。

> 2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
>
> 2015-05-26T16:23:07.357-0200: 64.460: [`CMS-concurrent-mark`1: `035/0.035 secs`1]
>
> `[Times: user=0.07 sys=0.00, real=0.03 secs]`1
>
> 1. `CMS-concurrent-mark` – 并发标记(“Concurrent Mark”) 是CMS垃圾收集中的一个阶段, 遍历老年代并标记所有的存活对象。
> 2. `035/0.035 secs` – 此阶段的持续时间, 分别是运行时间和相应的实际时间。
> 3. `[Times: user=0.07 sys=0.00, real=0.03 secs]` – `Times` 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅并发标记在运行,程序也在运行

阶段 3: Concurrent Preclean(并发预清理). 此阶段同样是与应用线程并行执行的, 不需要停止应用线程。 因为前一阶段是与程序并发进行的,可能有一些引用已经改变。如果在并发标记过程中发生了引用关系变化,JVM会(通过“Card”)将发生了改变的区域标记为“脏”区(这就是所谓的[卡片标记,Card Marking](http://psy-lob-saw.blogspot.com.ee/2014/10/the-jvm-write-barrier-card-marking.html))。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225008.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225008.png)

在预清理阶段,这些脏对象会被统计出来,从他们可达的对象也被标记下来。此阶段完成后, 用以标记的 card 也就被清空了。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225009.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812225009.png)

此外, 本阶段也会执行一些必要的细节处理, 并为 Final Remark 阶段做一些准备工作。

> 2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
>
> 2015-05-26T16:23:07.373-0200: 64.476: [`CMS-concurrent-preclean`: `0.016/0.016 secs`] `[Times: user=0.02 sys=0.00, real=0.02 secs]`
>
> 1. `CMS-concurrent-preclean` – 并发预清理阶段, 统计此前的标记阶段中发生了改变的对象。
> 2. `0.016/0.016 secs` – 此阶段的持续时间, 分别是运行时间和对应的实际时间。
> 3. `[Times: user=0.02 sys=0.00, real=0.02 secs]` – Times 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅GC的并发标记在运行,程序也在运行。

阶段 4: Concurrent Abortable Preclean(并发可取消的预清理). 此阶段也不停止应用线程. 本阶段尝试在 STW 的 Final Remark 之前尽可能地多做一些工作。本阶段的具体时间取决于多种因素, 因为它循环做同样的事情,直到满足某个退出条件( 如迭代次数, 有用工作量, 消耗的系统时间,等等)。

> 2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
>
> 2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean1: 0.167/1.074 secs2][Times: user=0.20 sys=0.00, real=1.07 secs]3
>
> 1. `CMS-concurrent-abortable-preclean` – 此阶段的名称: “Concurrent Abortable Preclean”。
> 2. `0.167/1.074 secs` – 此阶段的持续时间, 运行时间和对应的实际时间。有趣的是, 用户时间明显比时钟时间要小很多。通常情况下我们看到的都是时钟时间小于用户时间, 这意味着因为有一些并行工作, 所以运行时间才会小于使用的CPU时间。这里只进行了少量的工作 — 0.167秒的CPU时间,GC线程经历了很多系统等待。从本质上讲,GC线程试图在必须执行 STW暂停之前等待尽可能长的时间。默认条件下,此阶段可以持续最多5秒钟。
> 3. `[Times: user=0.20 sys=0.00, real=1.07 secs]` – “Times” 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅GC的并发标记线程在运行,程序也在运行

此阶段可能显著影响STW停顿的持续时间, 并且有许多重要的[配置选项](https://blogs.oracle.com/jonthecollector/entry/did_you_know)和失败模式。

阶段 5: Final Remark(最终标记). 这是此次GC事件中第二次(也是最后一次)STW阶段。本阶段的目标是完成老年代中所有存活对象的标记. 因为之前的 preclean 阶段是并发的, 有可能无法跟上应用程序的变化速度。所以需要 STW暂停来处理复杂情况。

通常CMS会尝试在年轻代尽可能空的情况运行 final remark 阶段, 以免接连多次发生 STW 事件。

看起来稍微比之前的阶段要复杂一些:

> ```
> 2015-05-26T16:23:08.447-0200: 65.550`: [GC (`CMS Final Remark`) [`YG occupancy: 387920 K (613440 K)`] 
> 65.550: `[Rescan (parallel) , 0.0085125 secs]` 65.559: [`weak refs processing, 0.0000243 secs]65.559`: [`class unloading, 0.0013120 secs]65.560`: [`scrub string table, 0.0001759 secs`] 
> [1 CMS-remark: `10812086K(11901376K)`] `11200006K(12514816K)`,`0.0110730 secs`] `[Times: user=0.06 sys=0.00, real=0.01 secs]
> ```
>
> 1. `2015-05-26T16:23:08.447-0200: 65.550` – GC事件开始的时间. 包括时钟时间,以及相对于JVM启动的时间. 其中`-0200`表示西二时区,而中国所在的东8区为 `+0800`。
> 2. `CMS Final Remark` – 此阶段的名称为 “Final Remark”, 标记老年代中所有存活的对象，包括在此前的并发标记过程中创建/修改的引用。
> 3. `YG occupancy: 387920 K (613440 K)` – 当前年轻代的使用量和总容量。
> 4. `[Rescan (parallel) , 0.0085125 secs]` – 在程序暂停时重新进行扫描(Rescan),以完成存活对象的标记。此时 rescan 是并行执行的,消耗的时间为 0.0085125秒。
> 5. `weak refs processing, 0.0000243 secs]65.559` – 处理弱引用的第一个子阶段(sub-phases)。 显示的是持续时间和开始时间戳。
> 6. `class unloading, 0.0013120 secs]65.560` – 第二个子阶段, 卸载不使用的类。 显示的是持续时间和开始的时间戳。
> 7. `scrub string table, 0.0001759 secs` – 最后一个子阶段, 清理持有class级别 metadata 的符号表(symbol tables),以及内部化字符串对应的 string tables。当然也显示了暂停的时钟时间。
> 8. `10812086K(11901376K)` – 此阶段完成后老年代的使用量和总容量
> 9. `11200006K(12514816K)` – 此阶段完成后整个堆内存的使用量和总容量
> 10. `0.0110730 secs` – 此阶段的持续时间。
> 11. `[Times: user=0.06 sys=0.00, real=0.01 secs]` – GC事件的持续时间, 通过不同的类别来衡量: user, system and real time。

在5个标记阶段完成之后, 老年代中所有的存活对象都被标记了, 现在GC将清除所有不使用的对象来回收老年代空间:

阶段 6: Concurrent Sweep(并发清除). 此阶段与应用程序并发执行,不需要STW停顿。目的是删除未使用的对象,并收回他们占用的空间。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250010.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250010.png)

\>

> 2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start] 2015-05-26T16:23:08.485-0200: 65.588: [`CMS-concurrent-sweep`: `0.027/0.027 secs`] `[Times: user=0.03 sys=0.00, real=0.03 secs]`

\>

> 1. `CMS-concurrent-sweep` – 此阶段的名称, “Concurrent Sweep”, 清除未被标记、不再使用的对象以释放内存空间。
> 2. `0.027/0.027 secs` – 此阶段的持续时间, 分别是运行时间和实际时间
> 3. `[Times: user=0.03 sys=0.00, real=0.03 secs]` – “Times”部分对并发阶段来说没有多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅是并发标记在运行,程序也在运行。

阶段 7: Concurrent Reset(并发重置). 此阶段与应用程序并发执行,重置CMS算法相关的内部数据, 为下一次GC循环做准备。

\>

> 2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start] 2015-05-26T16:23:08.497-0200: 65.601: [`CMS-concurrent-reset`: `0.012/0.012 secs`] `[Times: user=0.01 sys=0.00, real=0.01 secs]`

\>

> 1. `CMS-concurrent-reset` – 此阶段的名称, “Concurrent Reset”, 重置CMS算法的内部数据结构, 为下一次GC循环做准备。
> 2. `0.012/0.012 secs` – 此阶段的持续时间, 分别是运行时间和对应的实际时间
> 3. `[Times: user=0.01 sys=0.00, real=0.01 secs]` – “Times”部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅GC线程在运行,程序也在运行。

总之, CMS垃圾收集器在减少停顿时间上做了很多给力的工作, 大量的并发线程执行的工作并不需要暂停应用线程。 当然, CMS也有一些缺点,其中最大的问题就是老年代内存碎片问题, 在某些情况下GC会造成不可预测的暂停时间, 特别是堆内存较大的情况下。



## G1 – Garbage First(垃圾优先算法)

G1最主要的设计目标是: 将STW停顿的时间和分布变成可预期以及可配置的。事实上, G1是一款软实时垃圾收集器, 也就是说可以为其设置某项特定的性能指标. 可以指定: 在任意 `xx` 毫秒的时间范围内, STW停顿不得超过 `x` 毫秒。 如: 任意1秒暂停时间不得超过5毫秒. Garbage-First GC 会尽力达成这个目标(有很大的概率会满足, 但并不完全确定,具体是多少将是硬实时的[hard real-time])。

为了达成这项指标, G1 有一些独特的实现。首先, 堆不再分成连续的年轻代和老年代空间。而是划分为多个(通常是2048个)可以存放对象的 小堆区(smaller heap regions)。每个小堆区都可能是 Eden区, Survivor区或者Old区. 在逻辑上, 所有的Eden区和Survivor区合起来就是年轻代, 所有的Old区拼在一起那就是老年代:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250011.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250011.png)

这样的划分使得 GC不必每次都去收集整个堆空间, 而是以增量的方式来处理: 每次只处理一部分小堆区,称为此次的回收集(collection set). 每次暂停都会收集所有年轻代的小堆区, 但可能只包含一部分老年代小堆区:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250012.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250012.png)

G1的另一项创新, 是在并发阶段估算每个小堆区存活对象的总数。用来构建回收集(collection set)的原则是: 垃圾最多的小堆区会被优先收集。这也是G1名称的由来: garbage-first。

要启用G1收集器, 使用的命令行参数为:

```avrasm
java -XX:+UseG1GC com.mypackages.MyExecutableClass
```



### Evacuation Pause: Fully Young(转移暂停:纯年轻代模式)

在应用程序刚启动时, G1还未执行过(not-yet-executed)并发阶段, 也就没有获得任何额外的信息, 处于初始的 fully-young 模式. 在年轻代空间用满之后, 应用线程被暂停, 年轻代堆区中的存活对象被复制到存活区, 如果还没有存活区,则选择任意一部分空闲的小堆区用作存活区。

复制的过程称为转移(Evacuation), 这和前面讲过的年轻代收集器基本上是一样的工作原理。转移暂停的日志信息很长,为简单起见, 我们去除了一些不重要的信息. 在并发阶段之后我们会进行详细的讲解。此外, 由于日志记录很多, 所以并行阶段和“其他”阶段的日志将拆分为多个部分来进行讲解:

> ```
> 0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]
> [Parallel Time: 13.9 ms, GC Workers: 8]
> …
> [Code Root Fixup: 0.0 ms]
> [Code Root Purge: 0.0 ms]
> ```
>
> [Clear CT: 0.1 ms] 
>
> ```
> [Other: 0.4 ms]` 
> `…
> [Eden: 24.0M(24.0M)->0.0B(13.0M)` `Survivors: 0.0B->3072.0K` `Heap: 24.0M(256.0M)->21.9M(256.0M)]` 
> `[Times: user=0.04 sys=0.04, real=0.02 secs]
> ```
>
> 1. `0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]` – G1转移暂停,只清理年轻代空间。暂停在JVM启动之后 134 ms 开始, 持续的系统时间为 0.0144秒 。
> 2. `[Parallel Time: 13.9 ms, GC Workers: 8]` – 表明后面的活动由8个 Worker 线程并行执行, 消耗时间为13.9毫秒(real time)。
> 3. `…` – 为阅读方便, 省略了部分内容,请参考后文。
> 4. `[Code Root Fixup: 0.0 ms]` – 释放用于管理并行活动的内部数据。一般都接近于零。这是串行执行的过程。
> 5. `[Code Root Purge: 0.0 ms]` – 清理其他部分数据, 也是非常快的, 但如非必要则几乎等于零。这是串行执行的过程。
> 6. `[Other: 0.4 ms]` – 其他活动消耗的时间, 其中有很多是并行执行的。
> 7. `…` – 请参考后文。
> 8. `[Eden: 24.0M(24.0M)->0.0B(13.0M)` – 暂停之前和暂停之后, Eden 区的使用量/总容量。
> 9. `Survivors: 0.0B->3072.0K` – 暂停之前和暂停之后, 存活区的使用量。
> 10. `Heap: 24.0M(256.0M)->21.9M(256.0M)]` – 暂停之前和暂停之后, 整个堆内存的使用量与总容量。
> 11. `[Times: user=0.04 sys=0.04, real=0.02 secs]` – GC事件的持续时间, 通过三个部分来衡量: 
>
> \* user – 在此次垃圾回收过程中, 由GC线程所消耗的总的CPU时间。
> \* sys – GC过程中, 系统调用和系统等待事件所消耗的时间。
> \* real – 应用程序暂停的时间。在并行GC(Parallel GC)中, 这个数字约等于: (user time + system time)/GC线程数。 这里使用的是8个线程。 请注意,总是有一定比例的处理过程是不能并行化的。
>
> 说明: 系统时间(wall clock time, elapsed time), 是指一段程序从运行到终止，系统时钟走过的时间。一般来说，系统时间都是要大于CPU时间

最繁重的GC任务由多个专用的 worker 线程来执行。下面的日志描述了他们的行为:

> ```
> [Parallel Time: 13.9 ms, GC Workers: 8]
> ```
>
> `[GC Worker Start (ms)`: Min: 134.0, Avg: 134.1, Max: 134.1, Diff: 0.1] 
>
> `[Ext Root Scanning (ms)`: Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.2]
> [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] 
>
> [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0] 
>
> [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] 
>
> `[Code Root Scanning (ms)`: Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2] 
>
> `[Object Copy (ms)`: Min: 10.8, Avg: 12.1, Max: 12.6, Diff: 1.9, Sum: 96.5]
>
> `[Termination (ms)`: Min: 0.8, Avg: 1.5, Max: 2.8, Diff: 1.9, Sum: 12.2] 
>
> `[Termination Attempts`: Min: 173, Avg: 293.2, Max: 362, Diff: 189, Sum: 2346] 
>
> `[GC Worker Other (ms)`: Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1] 
>
> `GC Worker Total (ms)`: Min: 13.7, Avg: 13.8, Max: 13.8, Diff: 0.1, Sum: 110.2] 
>
> `[GC Worker End (ms)`: Min: 147.8, Avg: 147.8, Max: 147.8, Diff: 0.0]
>
> 1. `[Parallel Time: 13.9 ms, GC Workers: 8]` – 表明下列活动由8个线程并行执行,消耗的时间为13.9毫秒(real time)。
> 2. `[GC Worker Start (ms)` – GC的worker线程开始启动时,相对于 pause 开始的时间戳。如果 `Min` 和 `Max` 差别很大,则表明本机其他进程所使用的线程数量过多, 挤占了GC的CPU时间。
> 3. `[Ext Root Scanning (ms)` – 用了多长时间来扫描堆外(non-heap)的root, 如 classloaders, JNI引用, JVM的系统root等。后面显示了运行时间, “Sum” 指的是CPU时间。
> 4. `[Code Root Scanning (ms)` – 用了多长时间来扫描实际代码中的 root: 例如局部变量等等(local vars)。
> 5. `[Object Copy (ms)` – 用了多长时间来拷贝收集区内的存活对象。
> 6. `[Termination (ms)` – GC的worker线程用了多长时间来确保自身可以安全地停止, 这段时间什么也不用做, stop 之后该线程就终止运行了。
> 7. `[Termination Attempts` – GC的worker 线程尝试多少次 try 和 teminate。如果worker发现还有一些任务没处理完,则这一次尝试就是失败的, 暂时还不能终止。
> 8. `[GC Worker Other (ms)` – 一些琐碎的小活动,在GC日志中不值得单独列出来。
> 9. `GC Worker Total (ms)` – GC的worker 线程的工作时间总计。
> 10. `[GC Worker End (ms)` – GC的worker 线程完成作业的时间戳。通常来说这部分数字应该大致相等, 否则就说明有太多的线程被挂起, 很可能是因为[坏邻居效应(noisy neighbor)](https://github.com/cncounter/translation/blob/master/tiemao_2016/45_noisy_neighbors/noisy_neighbor_cloud _performance.md) 所导致的。

此外,在转移暂停期间,还有一些琐碎执行的小活动。这里我们只介绍其中的一部分, 其余的会在后面进行讨论。

> ```
> [Other: 0.4 ms]
> ```
>
> [Choose CSet: 0.0 ms] 
>
> ```
> [Ref Proc: 0.2 ms]
> [Ref Enq: 0.0 ms]
> ```
>
> [Redirty Cards: 0.1 ms] 
>
> [Humongous Register: 0.0 ms] 
>
> [Humongous Reclaim: 0.0 ms] 
>
> ```
> [Free CSet: 0.0 ms]
> ```

\>

> 1. `[Other: 0.4 ms]` – 其他活动消耗的时间, 其中有很多也是并行执行的。
> 2. `[Ref Proc: 0.2 ms]` – 处理非强引用(non-strong)的时间: 进行清理或者决定是否需要清理。
> 3. `[Ref Enq: 0.0 ms]` – 用来将剩下的 non-strong 引用排列到合适的 ReferenceQueue 中。
> 4. `[Free CSet: 0.0 ms]` – 将回收集中被释放的小堆归还所消耗的时间, 以便他们能用来分配新的对象。



### Concurrent Marking(并发标记)

G1收集器的很多概念建立在CMS的基础上,所以下面的内容需要你对CMS有一定的理解. 虽然也有很多地方不同, 但并发标记的目标基本上是一样的. G1的并发标记通过 Snapshot-At-The-Beginning(开始时快照) 的方式, 在标记阶段开始时记下所有的存活对象。即使在标记的同时又有一些变成了垃圾. 通过对象是存活信息, 可以构建出每个小堆区的存活状态, 以便回收集能高效地进行选择。

这些信息在接下来的阶段会用来执行老年代区域的垃圾收集。在两种情况下是完全地并发执行的： 一、如果在标记阶段确定某个小堆区只包含垃圾; 二、在STW转移暂停期间, 同时包含垃圾和存活对象的老年代小堆区。

当堆内存的总体使用比例达到一定数值时,就会触发并发标记。默认值为 `45%`, 但也可以通过JVM参数 `InitiatingHeapOccupancyPercent` 来设置。和CMS一样, G1的并发标记也是由多个阶段组成, 其中一些是完全并发的, 还有一些阶段需要暂停应用线程。

阶段 1: Initial Mark(初始标记)。 此阶段标记所有从GC root 直接可达的对象。在CMS中需要一次STW暂停, 但G1里面通常是在转移暂停的同时处理这些事情, 所以它的开销是很小的. 可以在 Evacuation Pause 日志中的第一行看到(initial-mark)暂停:

```dos
1.631: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0062656 secs]
```

阶段 2: Root Region Scan(Root区扫描). 此阶段标记所有从 “根区域” 可达的存活对象。 根区域包括: 非空的区域, 以及在标记过程中不得不收集的区域。因为在并发标记的过程中迁移对象会造成很多麻烦, 所以此阶段必须在下一次转移暂停之前完成。如果必须启动转移暂停, 则会先要求根区域扫描中止, 等它完成才能继续扫描. 在当前版本的实现中, 根区域是存活的小堆区: y包括下一次转移暂停中肯定会被清理的那部分年轻代小堆区。

```less
1.362: [GC concurrent-root-region-scan-start]
1.364: [GC concurrent-root-region-scan-end, 0.0028513 secs]
```

阶段 3: Concurrent Mark(并发标记). 此阶段非常类似于CMS: 它只是遍历对象图, 并在一个特殊的位图中标记能访问到的对象. 为了确保标记开始时的快照准确性, 所有应用线程并发对对象图执行的引用更新,G1 要求放弃前面阶段为了标记目的而引用的过时引用。

这是通过使用 `Pre-Write` 屏障来实现的,(不要和之后介绍的 `Post-Write` 混淆, 也不要和多线程开发中的内存屏障(memory barriers)相混淆)。Pre-Write屏障的作用是: G1在进行并发标记时, 如果程序将对象的某个属性做了变更, 就会在 log buffers 中存储之前的引用。 由并发标记线程负责处理。

```less
1.364: [GC concurrent-mark-start]
1.645: [GC co ncurrent-mark-end, 0.2803470 secs]
```

阶段 4: Remark(再次标记). 和CMS类似,这也是一次STW停顿,以完成标记过程。对于G1,它短暂地停止应用线程, 停止并发更新日志的写入, 处理其中的少量信息, 并标记所有在并发标记开始时未被标记的存活对象。这一阶段也执行某些额外的清理, 如引用处理(参见 Evacuation Pause log) 或者类卸载(class unloading)。

```yaml
1.645: [GC remark 1.645: [Finalize Marking, 0.0009461 secs]
1.646: [GC ref-proc, 0.0000417 secs] 1.646:
    [Unloading, 0.0011301 secs], 0.0074056 secs]
[Times: user=0.01 sys=0.00, real=0.01 secs]
```

阶段 5: Cleanup(清理). 最后这个小阶段为即将到来的转移阶段做准备, 统计小堆区中所有存活的对象, 并将小堆区进行排序, 以提升GC的效率. 此阶段也为下一次标记执行所有必需的整理工作(house-keeping activities): 维护并发标记的内部状态。

最后要提醒的是, 所有不包含存活对象的小堆区在此阶段都被回收了。有一部分是并发的: 例如空堆区的回收,还有大部分的存活率计算, 此阶段也需要一个短暂的STW暂停, 以不受应用线程的影响来完成作业. 这种STW停顿的日志如下:

```less
1.652: [GC cleanup 1213M->1213M(1885M), 0.0030492 secs]
[Times: user=0.01 sys=0.00, real=0.00 secs]
```

如果发现某些小堆区中只包含垃圾, 则日志格式可能会有点不同, 如: 
​

```less
1.872: [GC cleanup 1357M->173M(1996M), 0.0015664 secs] 
[Times: user=0.01 sys=0.00, real=0.01 secs] 
1.874: [GC concurrent-cleanup-start] 
1.876: [GC concurrent-cleanup-end, 0.0014846 secs]
```



### Evacuation Pause: Mixed (转移暂停: 混合模式)

能并发清理老年代中整个整个的小堆区是一种最优情形, 但有时候并不是这样。并发标记完成之后, G1将执行一次混合收集(mixed collection), 不只清理年轻代, 还将一部分老年代区域也加入到 collection set 中。

混合模式的转移暂停(Evacuation pause)不一定紧跟着并发标记阶段。有很多规则和历史数据会影响混合模式的启动时机。比如, 假若在老年代中可以并发地腾出很多的小堆区,就没有必要启动混合模式。

因此, 在并发标记与混合转移暂停之间, 很可能会存在多次 fully-young 转移暂停。

添加到回收集的老年代小堆区的具体数字及其顺序, 也是基于许多规则来判定的。 其中包括指定的软实时性能指标, 存活性,以及在并发标记期间收集的GC效率等数据, 外加一些可配置的JVM选项. 混合收集的过程, 很大程度上和前面的 fully-young gc 是一样的, 但这里我们还要介绍一个概念: remembered sets(历史记忆集)。

Remembered sets (历史记忆集)是用来支持不同的小堆区进行独立回收的。例如,在收集A、B、C区时, 我们必须要知道是否有从D区或者E区指向其中的引用, 以确定他们的存活性. 但是遍历整个堆需要相当长的时间, 这就违背了增量收集的初衷, 因此必须采取某种优化手段. 其他GC算法有独立的 Card Table 来支持年轻代的垃圾收集一样, 而G1中使用的是 Remembered Sets。

如下图所示, 每个小堆区都有一个 remembered set, 列出了从外部指向本区的所有引用。这些引用将被视为附加的 GC root. 注意,在并发标记过程中,老年代中被确定为垃圾的对象会被忽略, 即使有外部引用指向他们: 因为在这种情况下引用者也是垃圾。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250013.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250013.png)

接下来的行为,和其他垃圾收集器一样: 多个GC线程并行地找出哪些是存活对象,确定哪些是垃圾:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250014.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250014.png)

最后, 存活对象被转移到存活区(survivor regions), 在必要时会创建新的小堆区。现在,空的小堆区被释放, 可用于存放新的对象了。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250015.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/2018122250015.png)

为了维护 remembered set, 在程序运行的过程中, 只要写入某个字段,就会产生一个 Post-Write 屏障。如果生成的引用是跨区域的(cross-region),即从一个区指向另一个区, 就会在目标区的Remembered Set中,出现一个对应的条目。为了减少 Write Barrier 造成的开销, 将卡片放入Remembered Set 的过程是异步的, 而且经过了很多的优化. 总体上是这样: Write Barrier 把脏卡信息存放到本地缓冲区(local buffer), 有专门的GC线程负责收集, 并将相关信息传给被引用区的 remembered set。

混合模式下的日志, 和纯年轻代模式相比, 可以发现一些有趣的地方:

> [`[Update RS (ms)`: Min: 0.7, Avg: 0.8, Max: 0.9, Diff: 0.2, Sum: 6.1] 
>
> `[Processed Buffers`: Min: 0, Avg: 2.2, Max: 5, Diff: 5, Sum: 18]
>
> `[Scan RS (ms)`: Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.8]
>
> ```
> [Clear CT: 0.2 ms]
> [Redirty Cards: 0.1 ms]
> ```
>
> \1. `[Update RS (ms)` – 因为 Remembered Sets 是并发处理的,必须确保在实际的垃圾收集之前, 缓冲区中的 card 得到处理。如果card数量很多, 则GC并发线程的负载可能就会很高。可能的原因是, 修改的字段过多, 或者CPU资源受限。
>
> 1. `[Processed Buffers` – 每个 worker 线程处理了多少个本地缓冲区(local buffer)。
> 2. `[Scan RS (ms)` – 用了多长时间扫描来自RSet的引用。
> 3. `[Clear CT: 0.2 ms]` – 清理 card table 中 cards 的时间。清理工作只是简单地删除“脏”状态, 此状态用来标识一个字段是否被更新的, 供Remembered Sets使用。
> 4. `[Redirty Cards: 0.1 ms]` – 将 card table 中适当的位置标记为 dirty 所花费的时间。”适当的位置”是由GC本身执行的堆内存改变所决定的, 例如引用排队等。



### 总结

通过本节内容的学习, 你应该对G1垃圾收集器有了一定了解。当然, 为了简洁, 我们省略了很多实现细节， 例如如何处理[巨无霸对象(humongous objects)](https://plumbr.eu/handbook/gc-tuning-in-practice#humongous-allocations)。 综合来看, G1是HotSpot中最先进的 准产品级(production-ready) 垃圾收集器。重要的是, HotSpot 工程师的主要精力都放在不断改进G1上面, 在新的java版本中,将会带来新的功能和优化。

可以看到, G1 解决了 CMS 中的各种疑难问题, 包括暂停时间的可预测性, 并终结了堆内存的碎片化。对单业务延迟非常敏感的系统来说, 如果CPU资源不受限制,那么G1可以说是 HotSpot 中最好的选择, 特别是在最新版本的Java虚拟机中。当然,这种降低延迟的优化也不是没有代价的: 由于额外的写屏障(write barriers)和更积极的守护线程, G1的开销会更大。所以, 如果系统属于吞吐量优先型的, 又或者CPU持续占用100%, 而又不在乎单次GC的暂停时间, 那么CMS是更好的选择。

> 总之: G1适合大内存,需要低延迟的场景。

选择正确的GC算法,唯一可行的方式就是去尝试,并找出不对劲的地方, 在下一章我们将给出一般指导原则。

注意,G1可能会成为Java 9的默认GC: http://openjdk.java.net/jeps/248

## ZGC

ZGC（Z Garbage Collector） 是一款性能比 G1 更加优秀的垃圾收集器。ZGC 第一次出现是在 JDK 11 中以实验性的特性引入，这也是 JDK 11 中最大的亮点。在 JDK 15 中 ZGC 不再是实验功能，可以正式投入生产使用了，使用 –XX:+UseZGC 可以启用 ZGC。





ZGC 有 3 个重要特性：





- 暂停时间不会超过 10 ms。





> JDK 16 发布后，GC 暂停时间已经缩小到 1 ms 以内，并且时间复杂度是 o(1)，这也就是说 GC 停顿时间是一个固定值了，并不会受堆内存大小影响。
>
> 下面图片来自:https://malloc.se/blog/zgc-jdk16



![img](https://pics7.baidu.com/feed/c2fdfc039245d688de953940b6bf8917d01b24c1.jpeg?token=bf8ea1bdd916f3b84e4a99ef816ceb6e)



- 最大支持 16TB 的大堆，最小支持 8MB 的小堆。
- 跟 G1 相比，对应用程序吞吐量的影响小于 15 %。





### 1 内存多重映射





内存多重映射，就是使用 mmap 把不同的虚拟内存地址映射到同一个物理内存地址上。如下图：



![img](https://pics5.baidu.com/feed/b03533fa828ba61e93f9916a57496303314e59e2.jpeg?token=189e6829e49981468977b87f6f3e99b7)



ZGC 为了更灵活高效地管理内存，使用了内存多重映射，把同一块儿物理内存映射为 Marked0、Marked1 和 Remapped 三个虚拟内存。





当应用程序创建对象时，会在堆上申请一个虚拟地址，这时 ZGC 会为这个对象在 Marked0、Marked1 和 Remapped 这三个视图空间分别申请一个虚拟地址，这三个虚拟地址映射到同一个物理地址。





Marked0、Marked1 和 Remapped 这三个虚拟内存作为 ZGC 的三个视图空间，在同一个时间点内只能有一个有效。ZGC 就是通过这三个视图空间的切换，来完成并发的垃圾回收。





### 2 染色指针





### 2.1 三色标记回顾





我们知道 G1 垃圾收集器使用了三色标记，这里先做一个回顾。下面是一个三色标记过程中的对象引用示例图：



![img](https://pics5.baidu.com/feed/8b82b9014a90f603480e6f35296f4712b151ed89.jpeg?token=34df826e6597a553fb37fe61529a7c58)



总共有三种颜色，说明如下：





- 白色：本对象还没有被标记线程访问过。
- 灰色：本对象已经被访问过，但是本对象引用的其他对象还没有被全部访问。
- 黑色：本对象已经被访问过，并且本对象引用的其他对象也都被访问过了。





三色标记的过程如下：





1. 初始阶段，所有对象都是白色。
2. 将 GC Roots 直接引用的对象标记为灰色。
3. 处理灰色对象，把当前灰色对象引用的所有对象都变成灰色，之后将当前灰色对象变成黑色。
4. 重复步骤 3，直到不存在灰色对象为止。





三色标记结束后，白色对象就是没有被引用的对象（比如上图中的 H 和 G），可以被回收了。





### 2.2 染色指针





ZGC 出现之前， GC 信息保存在对象头的 Mark Word 中。比如 64 位的 JVM，对象头的 Mark Word 中保存的信息如下图：



![img](https://pics5.baidu.com/feed/55e736d12f2eb93866893413c21f713ce7dd6ff6.jpeg?token=de54155dd0b97b93b951f2781dbd5935)



前 62位保存了 GC 信息，最后两位保存了锁标志。





ZGC 的一大创举是将 GC 信息保存在了染色指针上。**染色指针是一种将少量信息直接存储在指针上的技术**。在 64 位 JVM 中，对象指针是 64 位，如下图：



![img](https://pics2.baidu.com/feed/4b90f603738da9778c38fe18a12c0c108618e371.jpeg?token=ebd07d292fbc46625c6dd5c88f32e784)



在这个 64 位的指针上，高 16 位都是 0，暂时不用来寻址。剩下的 48 位支持的内存可以达到 256 TB（2 ^48）,这可以满足多数大型服务器的需要了。不过 ZGC 并没有把 48 位都用来保存对象信息，而是用高 4 位保存了四个标志位，这样 ZGC 可以管理的最大内存可以达到 16 TB（2 ^ 44）。





通过这四个标志位，JVM 可以从指针上直接看到对象的三色标记状态（Marked0、Marked1）、是否进入了重分配集（Remapped）、是否需要通过 finalize 方法来访问到（Finalizable）。





无需进行对象访问就可以获得 GC 信息，这大大提高了 GC 效率。





### 3 内存布局





首先我们回顾一下 G1 垃圾收集器的内存布局。G1把整个堆分成了大小相同的 region，每个堆大约可以有 2048 个region，每个 region 大小为 1~32 MB （必须是 2 的次方）。如下图：



![img](https://pics7.baidu.com/feed/5ab5c9ea15ce36d3219de81e298ece8ee850b18e.jpeg?token=af7c963b6282e0bde0b9364ce35dda64)



跟 G1 类似，ZGC 的堆内存也是基于 Region 来分布，不过 ZGC 是不区分新生代老年代的。不同的是，ZGC 的 Region 支持动态地创建和销毁，并且 Region 的大小不是固定的，包括三种类型的 Region ：





- Small Region：2MB，主要用于放置小于 256 KB 的小对象。
- Medium Region：32MB，主要用于放置大于等于 256 KB 小于 4 MB 的对象。
- Large Region：N * 2MB。这个类型的 Region 是可以动态变化的，不过必须是 2MB 的整数倍，最小支持 4 MB。每个 Large Region 只放置一个大对象，并且是不会被重分配的。





### 4 读屏障





读屏障类似于 Spring AOP 的前置增强，是 JVM 向应用代码中插入一小段代码，当应用线程从堆中读取对象的引用时，会先执行这段代码。**注意：只有从堆内存中读取对象的引用时，才会执行这个代码**。下面代码只有第一行需要加入读屏障。





```
Object o = obj.FieldA
Object p = o //不是从堆中读取引用
o.dosomething() //不是从堆中读取引用
int i =  obj.FieldB //不是引用类型
```





读屏障在解释执行时通过 load 相关的字节码指令加载数据。作用是在对象标记和转移过程中，判断对象的引用地址是否满足条件，并作出相应动作。如下图：



![img](https://pics5.baidu.com/feed/42a98226cffc1e176d44ad8b58ed020a728de91c.jpeg?token=60c59d84c8caa4e04dfcd933a7853b3b)



标记、转移和重定位这些过程请看下一节。





> 读屏障会对应用程序的性能有一定影响，据测试，对性能的最高影响达到 4%，但提高了 GC 并发能力，降低了 STW。





### 5 GC 过程





前面已经讲过，ZGC 使用内存多重映射技术，把物理内存映射为 Marked0、Marked1 和 Remapped 三个地址视图，利用地址视图的切换，ZGC 实现了高效的并发收集。





ZGC 的垃圾收集过程包括标记、转移和重定位三个阶段。如下图：



![img](https://pics2.baidu.com/feed/58ee3d6d55fbb2fb182ebf715a37d4ad4723dcd7.jpeg?token=2f8e035d09d14d7035b1744dfb2dcc28)



ZGC 初始化后，整个内存空间的地址视图被设置为 Remapped。





### 5.1 初始标记





从 GC Roots 出发，找出 GC Roots 直接引用的对象，放入活跃对象集合，这个过程需要 STW，不过 **STW 的时间跟 GC Roots 数量成正比**，耗时比较短。





### 5.2 并发标记





并发标记过程中，GC 线程和 Java 应用线程会并行运行。这个过程需要注意下面几点：





- GC 标记线程访问对象时，如果对象地址视图是 Remapped，就把对象地址视图切换到 Marked0，如果对象地址视图已经是 Marked0，说明已经被其他标记线程访问过了，跳过不处理。
- 标记过程中Java 应用线程新创建的对象会直接进入 Marked0 视图。
- 标记过程中Java 应用线程访问对象时，如果对象的地址视图是 Remapped，就把对象地址视图切换到 Marked0，可以参考前面讲的读屏障。
- 标记结束后，如果对象地址视图是 Marked0，那就是活跃的，如果对象地址视图是 Remapped，那就是不活跃的。





**标记阶段的活跃视图也可能是 Marked1，为什么会采用两个视图呢？**





这里采用两个视图是为了区分前一次标记和这一次标记。如果这次标记的视图是 Marked0，那下一次并发标记就会把视图切换到 Marked1。这样做可以配合 ZGC 按照页回收垃圾的做法。如下图：



![img](https://pics2.baidu.com/feed/0bd162d9f2d3572c51844bb89b6e972e63d0c3f3.jpeg?token=4272286f45cc22b8f5d9c1359ee74829)



第二次标记的时候，如果还是切换到 Marked0，那么 2 这个对象区分不出是活跃的还是上次标记过的。如果第二次标记切换到 Marked1，就可以区分出了。





这时 Marked0 这个视图的对象就是上次标记过程被标记过活跃，转移的时候没有被转移，但这次标记没有被标记为活跃的对象。Marked1 视图的对象是这次标记被标记为活跃的对象。Remapped 视图的对象是上次垃圾回收发生转移或者是被 Java 应用线程访问过，本次垃圾回收中被标记为不活跃的对象。





### 5.3 再标记





并发标记阶段 GC 线程和 Java 应用线程并发执行，标记过程中可能会有引用关系发生变化而导致的漏标记问题。再标记阶段重新标记**并发标记阶段**发生变化的对象，还会对非强引用（软应用，虚引用等）进行并行标记。





这个阶段需要 STW，但是需要标记的对象少，耗时很短。





### 5.4 初始转移





**转移就是把活跃对象复制到新的内存，之前的内存空间可以被回收。**





初始转移需要扫描 GC Roots 直接引用的对象并进行转移，这个过程需要 STW，STW 时间跟 GC Roots 成正比。





### 5.5 并发转移





并发转移过程 GC 线程和 Java 线程是并发进行的。上面已经讲过，转移过程中对象视图会被切回 Remapped 。转移过程需要注意以下几点：





- 如果 GC 线程访问对象的视图是 Marked0，则转移对象，并把对象视图设置成 Remapped。
- 如果 GC 线程访问对象的视图是 Remapped，说明被其他 GC 线程处理过，跳过不再处理。
- 并发转移过程中 Java 应用线程创建的新对象地址视图是 Remapped。
- 如果 Java 应用线程访问的对象被标记为活跃并且对象视图是 Marked0，则转移对象，并把对象视图设置成 Remapped。





### 5.6 重定位





转移过程对象的地址发生了变化，在这个阶段，把所有指向对象旧地址的指针调整到对象的新地址上。





### 6 垃圾收集算法





ZGC 采用标记 - 整理算法，算法的思想是把所有存活对象移动到堆的一侧，移动完成后回收掉边界以外的对象。如下图：



![img](https://pics5.baidu.com/feed/8b82b9014a90f603a63c81712d6f4712b151edfb.jpeg?token=c667a4dcdbadf57b681fac3e5919c928)



### 4.1 JDK 16 之前





在 JDK 16 之前，ZGC 会预留（Reserve）一块儿堆内存，这个预留内存不能用于 Java 线程的内存分配。即使从 Java 线程的角度看堆内存已经满了也不能使用 Reserve，只有 GC 过程中搬移存活对象的时候才可以使用。如下图：



![img](https://pics0.baidu.com/feed/242dd42a2834349bc40b2404d397e1c737d3bead.jpeg?token=702b8e94ed7eb043b99e59209ca5d6cb)



这样做的好处是算法简单，非常适合并行收集。但这样做有几个问题：





- 因为有预留内存，能给 Java 线程分配的堆内存小于 JVM 声明的堆内存。
- Reserve 仅仅用于存放 GC 过程中搬移的对象，有点内存浪费。
- 因为 Reserve 不能给 GC 过程中搬移对象的 Java 线程使用，搬移线程可能会因为申请不到足够内存而不能完成对象搬移，这返回过来又会导致应用程序的 OOM。





### 4.2 JDK 16 改进





JDK 16 发布后，ZGC 支持就地搬移对象（G1 在 Full GC 的时候也是就地搬移）。这样做的好处是不用预留空闲内存了。如下图：



![img](https://pics1.baidu.com/feed/0b7b02087bf40ad1bfb3ca244c51e5d6a8ecce2d.jpeg?token=0dd9ea4a4ba22d628303016e9cb49762)



不过就地搬移也有一定的挑战。比如：必须考虑搬移对象的顺序，否则可能会覆盖尚未移动的对象。这就需要 GC 线程之间更好的进行协作，不利于并发收集，同时也会导致搬移对象的 Java 线程需要考虑什么可以做什么不可以做。





为了获得更好的 GC 表现，**JDK 16 在支持就地搬移的同时，也支持预留（Reserve）堆内存的方式**，并且 ZGC 不需要真的预留空闲的堆内存。默认情况下，只要有空闲的 region，ZGC 就会使用预留堆内存的方式，如果没有空闲的 region，否则 ZGC 就会启用就地搬移。如果有了空闲的 region， ZGC 又会切换到预留堆内存的搬移方式。





### 7 总结





内存多重映射和染色指针的引入，使 ZGC 的并发性能大幅度提升。





**ZGC 只有 3 个需要 STW 的阶段**，其中初始标记和初始转移只需要扫描所有 GC Roots，STW 时间 GC Roots 的数量成正比，不会耗费太多时间。再标记过程主要处理并发标记引用地址发生变化的对象，这些对象数量比较少，耗时非常短。可见整个 ZGC 的 STW 时间几乎只跟 GC Roots 数量有关系，不会随着堆大小和对象数量的变化而变化。





ZGC 也有一个缺点，就是浮动垃圾。因为 ZGC 没有分代概念，虽然 ZGC 的 STW 时间在 1ms 以内，但是 ZGC 的整个执行过程耗时还是挺长的。在这个过程中 Java 线程可能会创建大量的新对象，这些对象会成为浮动垃圾，只能等下次 GC 的时候进行回收。

## Shenandoah 的性能

> 译注: Shenandoah: 谢南多厄河; 情人渡,水手谣; –> 此款GC暂时没有标准的中文译名; 翻译为大水手垃圾收集器?

我们列出了HotSpot中可用的所有 “准生产级” 算法。还有一种还在实验室中的算法, 称为 超低延迟垃圾收集器(Ultra-Low-Pause-Time Garbage Collector) . 它的设计目标是管理大型的多核服务器上,超大型的堆内存: 管理 100GB 及以上的堆容量, GC暂停时间小于 10ms。 当然,也是需要和吞吐量进行权衡的: 没有GC暂停的时候,算法的实现对吞吐量的性能损失不能超过10%

在新算法作为准产品级进行发布之前, 我们不准备去讨论具体的实现细节, 但它也构建在前面所提到的很多算法的基础上, 例如并发标记和增量收集。但其中有很多东西是不同的。它不再将堆内存划分成多个代, 而是只采用单个空间. 没错, Shenandoah 并不是一款分代垃圾收集器。这也就不再需要 card tables 和 remembered sets. 它还使用转发指针(forwarding pointers), 以及Brooks 风格的读屏障(Brooks style read barrier), 以允许对存活对象的并发复制, 从而减少GC暂停的次数和时间。

# GC 调优(工具篇)

JVM 在程序执行的过程中, 提供了GC行为的原生数据。那么, 我们就可以利用这些原生数据来生成各种报告。原生数据(*raw data*) 包括:

- 各个内存池的当前使用情况,
- 各个内存池的总容量,
- 每次GC暂停的持续时间,
- GC暂停在各个阶段的持续时间。

可以通过这些数据算出各种指标, 例如: 程序的内存分配率, 提升率等等。本章主要介绍如何获取原生数据。 后续的章节将对重要的派生指标(derived metrics)展开讨论, 并引入GC性能相关的话题。

## JMX API

从 JVM 运行时获取GC行为数据, 最简单的办法是使用标准 [JMX API 接口](https://docs.oracle.com/javase/tutorial/jmx/index.html). JMX是获取 JVM内部运行时状态信息 的标准API. 可以编写程序代码, 通过 JMX API 来访问本程序所在的JVM，也可以通过JMX客户端执行(远程)访问。

最常见的 JMX客户端是 [JConsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html) 和 [JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) (可以安装各种插件,十分强大)。两个工具都是标准JDK的一部分, 而且很容易使用. 如果使用的是 JDK 7u40 及更高版本, 还可以使用另一个工具: [Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)( 大致翻译为 Java控制中心, `jmc.exe`)。

> JVisualVM安装MBeans插件的步骤: 通过 工具(T) – 插件(G) – 可用插件 – 勾选VisualVM-MBeans – 安装 – 下一步 – 等待安装完成…… 其他插件的安装过程基本一致。

所有 JMX客户端都是独立的程序,可以连接到目标JVM上。目标JVM可以在本机, 也可能是远端JVM. 如果要连接远端JVM, 则目标JVM启动时必须指定特定的环境变量,以开启远程JMX连接/以及端口号。 示例如下:

```mipsasm
java -Dcom.sun.management.jmxremote.port=5432 com.yourcompany.YourApp
```

在此处, JVM 打开端口`5432`以支持JMX连接。

通过 JVisualVM 连接到某个JVM以后, 切换到 MBeans 标签, 展开 “java.lang/GarbageCollector” . 就可以看到GC行为信息, 下图是 JVisualVM 中的截图:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227001.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227001.png)

下图是Java Mission Control 中的截图:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227002.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227002.png)

从以上截图中可以看到两款垃圾收集器。其中一款负责清理年轻代(**PS Scavenge**)，另一款负责清理老年代(**PS MarkSweep**); 列表中显示的就是垃圾收集器的名称。可以看到 , jmc 的功能和展示数据的方式更强大。

对所有的垃圾收集器, 通过 JMX API 获取的信息包括:

- **CollectionCount** : 垃圾收集器执行的GC总次数,
- **CollectionTime**: 收集器运行时间的累计。这个值等于所有GC事件持续时间的总和,
- **LastGcInfo**: 最近一次GC事件的详细信息。包括 GC事件的持续时间(duration), 开始时间(startTime) 和 结束时间(endTime), 以及各个内存池在最近一次GC之前和之后的使用情况,
- **MemoryPoolNames**: 各个内存池的名称,
- **Name**: 垃圾收集器的名称
- **ObjectName**: 由JMX规范定义的 MBean的名字,,
- **Valid**: 此收集器是否有效。本人只见过 “`true`“的情况

根据经验, 这些信息对GC的性能来说,不能得出什么结论. 只有编写程序, 获取GC相关的 JMX 信息来进行统计和分析。 在下文可以看到, 一般也不怎么关注 MBean , 但 MBean 对于理解GC的原理倒是挺有用的。



## JVisualVM

[JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) 工具的 “[VisualGC](http://www.oracle.com/technetwork/java/visualgc-136680.html)” 插件提供了基本的 JMX客户端功能, 还实时显示出 GC事件以及各个内存空间的使用情况。

Visual GC 插件常用来监控本机运行的Java程序, 比如开发者和性能调优专家经常会使用此插件, 以快速获取程序运行时的GC信息。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227003.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227003.png)

左侧的图表展示了各个内存池的使用情况: Metaspace/永久代, 老年代, Eden区以及两个存活区。

在右边, 顶部的两个图表与 GC无关, 显示的是 JIT编译时间 和 类加载时间。下面的6个图显示的是内存池的历史记录, 每个内存池的GC次数,GC总时间, 以及最大值，峰值, 当前使用情况。

再下面是 HistoGram, 显示了年轻代对象的年龄分布。至于对象的年龄监控(objects tenuring monitoring), 本章不进行讲解。

与纯粹的JMX工具相比, VisualGC 插件提供了更友好的界面, 如果没有其他趁手的工具, 请选择VisualGC. 本章接下来会介绍其他工具, 这些工具可以提供更多的信息, 以及更好的视角. 当然, 在“Profilers(分析器)”一节中，也会介绍 JVisualVM 的适用场景 —— 如: 分配分析(allocation profiling), 所以我们绝不会贬低哪一款工具, 关键还得看实际情况。



## jstat

[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html) 也是标准JDK提供的一款监控工具(Java Virtual Machine statistics monitoring tool),可以统计各种指标。既可以连接到本地JVM,也可以连到远程JVM. 查看支持的指标和对应选项可以执行 “`jstat -options`” 。例如:

```smalltalk
+-----------------+---------------------------------------------------------------+
|     Option      |                          Displays...                          |
+-----------------+---------------------------------------------------------------+
|class            | Statistics on the behavior of the class loader                |
|compiler         | Statistics  on  the behavior of the HotSpot Just-In-Time com- |
|                 | piler                                                         |
|gc               | Statistics on the behavior of the garbage collected heap      |
|gccapacity       | Statistics of the capacities of  the  generations  and  their |
|                 | corresponding spaces.                                         |
|gccause          | Summary  of  garbage collection statistics (same as -gcutil), |
|                 | with the cause  of  the  last  and  current  (if  applicable) |
|                 | garbage collection events.                                    |
|gcnew            | Statistics of the behavior of the new generation.             |
|gcnewcapacity    | Statistics of the sizes of the new generations and its corre- |
|                 | sponding spaces.                                              |
|gcold            | Statistics of the behavior of the old and  permanent  genera- |
|                 | tions.                                                        |
|gcoldcapacity    | Statistics of the sizes of the old generation.                |
|gcpermcapacity   | Statistics of the sizes of the permanent generation.          |
|gcutil           | Summary of garbage collection statistics.                     |
|printcompilation | Summary of garbage collection statistics.                     |
+-----------------+---------------------------------------------------------------+
```

jstat 对于快速确定GC行为是否健康非常有用。启动方式为: “`jstat -gc -t PID 1s`” , 其中,PID 就是要监视的Java进程ID。可以通过 `jps` 命令查看正在运行的Java进程列表。

```mipsasm
jps

jstat -gc -t 2428 1s
```

以上命令的结果, 是 jstat 每秒向标准输出输出一行新内容, 比如:

```yaml
Timestamp  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
200.0    8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169344.0  21248.0 20534.3 3072.0 2807.7     34    0.720  658   133.684  134.404
201.0    8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169343.2  21248.0 20534.3 3072.0 2807.7     34    0.720  662   134.712  135.432
202.0    8448.0 8448.0 8102.5  0.0   67712.0  67598.5   169344.0   169343.6  21248.0 20534.3 3072.0 2807.7     34    0.720  667   135.840  136.559
203.0    8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
204.0    8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
205.0    8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
206.0    8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
207.08448.08448.08154.80.067712.067712.0169344.0169343.521248.020547.23072.02807.7340.720673136.289137.009208.08448.08448.08154.80.067712.067712.0169344.0169343.521248.020547.23072.02807.7340.720673136.289137.009
```

稍微解释一下上面的内容。参考 [jstat manpage](http://www.manpagez.com/man/1/jstat/) , 我们可以知道:

- jstat 连接到 JVM 的时间, 是JVM启动后的 200秒。此信息从第一行的 “**Timestamp**” 列得知。继续看下一行, jstat 每秒钟从JVM 接收一次信息, 也就是命令行参数中 “`1s`” 的含义。
- 从第一行的 “**YGC**” 列得知年轻代共执行了34次GC, 由 “**FGC**” 列得知整个堆内存已经执行了 658次 full GC。
- 年轻代的GC耗时总共为 `0.720 秒`, 显示在“**YGCT**” 这一列。
- Full GC 的总计耗时为 `133.684 秒`, 由“**FGCT**”列得知。 这立马就吸引了我们的目光, 总的JVM 运行时间只有 200 秒, **但其中有 66% 的部分被 Full GC 消耗了**。

再看下一行, 问题就更明显了。

- 在接下来的一秒内共执行了 4 次 Full GC。参见 “**FGC**” 列.
- 这4次 Full GC 暂停占用了差不多 1秒的时间(根据 **FGCT** 列的差得知)。与第一行相比, Full GC 耗费了`928 毫秒`, 即 `92.8%` 的时间。
- 根据 “**OC** 和 “**OU**” 列得知, **整个老年代的空间** 为 `169,344.0 KB` (“OC“), 在 4 次 Full GC 后依然占用了 `169,344.2 KB` (“OU“)。用了 `928ms` 的时间却只释放了 800 字节的内存, 怎么看都觉得很不正常。

只看这两行的内容, 就知道程序出了很严重的问题。继续分析下一行, 可以确定问题依然存在,而且变得更糟。

JVM几乎完全卡住了(stalled), 因为GC占用了90%以上的计算资源。GC之后, 所有的老代空间仍然还在占用。事实上, 程序在一分钟以后就挂了, 抛出了 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误。

可以看到, 通过 jstat 能很快发现对JVM健康极为不利的GC行为。一般来说, 只看 jstat 的输出就能快速发现以下问题:

- 最后一列 “**GCT**”, 与JVM的总运行时间 “**Timestamp**” 的比值, 就是GC 的开销。如果每一秒内, “**GCT**” 的值都会明显增大, 与总运行时间相比, 就暴露出GC开销过大的问题. 不同系统对GC开销有不同的容忍度, 由性能需求决定, 一般来讲, 超过 `10%` 的GC开销都是有问题的。
- “**YGC**” 和 “**FGC**” 列的快速变化往往也是有问题的征兆。频繁的GC暂停会累积,并导致更多的线程停顿(stop-the-world pauses), 进而影响吞吐量。
- 如果看到 “**OU**” 列中,老年代的使用量约等于老年代的最大容量(**OC**), 并且不降低的话, 就表示虽然执行了老年代GC, 但基本上属于无效GC。



## GC日志(GC logs)

通过日志内容也可以得到GC相关的信息。因为GC日志模块内置于JVM中, 所以日志中包含了对GC活动最全面的描述。 这就是事实上的标准, 可作为GC性能评估和优化的最真实数据来源。

GC日志一般输出到文件之中, 是纯 text 格式的, 当然也可以打印到控制台。有多个可以控制GC日志的JVM参数。例如,可以打印每次GC的持续时间, 以及程序暂停时间(`-XX:+PrintGCApplicationStoppedTime`), 还有GC清理了多少引用类型(`-XX:+PrintReferenceGC`)。

要打印GC日志, 需要在启动脚本中指定以下参数:

```diff
-XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:<filename>
```

以上参数指示JVM: 将所有GC事件打印到日志文件中, 输出每次GC的日期和时间戳。不同GC算法输出的内容略有不同. ParallelGC 输出的日志类似这样:

```css
199.879: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1473386 secs] [Times: user=0.43 sys=0.01, real=0.15 secs]
200.027: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1567794 secs] [Times: user=0.41 sys=0.00, real=0.16 secs]
200.184: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1621946 secs] [Times: user=0.43 sys=0.00, real=0.16 secs]
200.346: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)],0.1547695 secs][Times: user=0.41 sys=0.00, real=0.15 secs]200.502:[Full GC (Ergonomics)[PSYoungGen:64000K->63999K(74240K)][ParOldGen:169318K->169318K(169472K)]233318K->233317K(243712K),[Metaspace:20427K->20427K(1067008K)],0.1563071 secs][Times: user=0.42 sys=0.01, real=0.16 secs]200.659:[Full GC (Ergonomics)[PSYoungGen:64000K->63999K(74240K)][ParOldGen:169318K->169318K(169472K)]233318K->233317K(243712K),[Metaspace:20427K->20427K(1067008K)],0.1538778 secs][Times: user=0.42 sys=0.00, real=0.16 secs]
```

在 “[04. GC算法:实现篇](http://blog.csdn.net/renfufei/article/details/54885190)” 中详细介绍了这些格式, 如果对此不了解, 可以先阅读该章节。

分析以上日志内容, 可以得知:

- 这部分日志截取自JVM启动后200秒左右。
- 日志片段中显示, 在`780毫秒`以内, 因为垃圾回收 导致了5次 Full GC 暂停(去掉第六次暂停,这样更精确一些)。
- 这些暂停事件的总持续时间是 `777毫秒`, 占总运行时间的 **99.6%**。
- 在GC完成之后, 几乎所有的老年代空间(`169,472 KB`)依然被占用(`169,318 KB`)。

通过日志信息可以确定, 该应用的GC情况非常糟糕。JVM几乎完全停滞, 因为GC占用了超过`99%`的CPU时间。 而GC的结果是, 老年代空间仍然被占满, 这进一步肯定了我们的结论。 示例程序和jstat 小节中的是同一个, 几分钟之后系统就挂了, 抛出 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误, 不用说, 问题是很严重的.

从此示例可以看出, GC日志对监控GC行为和JVM是否处于健康状态非常有用。一般情况下, 查看 GC 日志就可以快速确定以下症状:

- GC开销太大。如果GC暂停的总时间很长, 就会损害系统的吞吐量。不同的系统允许不同比例的GC开销, 但一般认为, 正常范围在 `10%` 以内。
- 极个别的GC事件暂停时间过长。当某次GC暂停时间太长, 就会影响系统的延迟指标. 如果延迟指标规定交易必须在 `1,000 ms`内完成, 那就不能容忍任何超过 `1000毫秒`的GC暂停。
- 老年代的使用量超过限制。如果老年代空间在 Full GC 之后仍然接近全满, 那么GC就成为了性能瓶颈, 可能是内存太小, 也可能是存在内存泄漏。这种症状会让GC的开销暴增。

可以看到,GC日志中的信息非常详细。但除了这些简单的小程序, 生产系统一般都会生成大量的GC日志, 纯靠人工是很难阅读和进行解析的。



## GCViewer

我们可以自己编写解析器, 来将庞大的GC日志解析为直观易读的图形信息。 但很多时候自己写程序也不是个好办法, 因为各种GC算法的复杂性, 导致日志信息格式互相之间不太兼容。那么神器来了: [GCViewer](https://github.com/chewiebug/GCViewer)。

[GCViewer](https://github.com/chewiebug/GCViewer) 是一款开源的GC日志分析工具。项目的 GitHub 主页对各项指标进行了完整的描述. 下面我们介绍最常用的一些指标。

第一步是获取GC日志文件。这些日志文件要能够反映系统在性能调优时的具体场景. 假若运营部门(operational department)反馈: 每周五下午,系统就运行缓慢, 不管GC是不是主要原因, 分析周一早晨的日志是没有多少意义的。

获取到日志文件之后, 就可以用 GCViewer 进行分析, 大致会看到类似下面的图形界面:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227004.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227004.png)

使用的命令行大致如下:

```mipsasm
java -jar gcviewer_1.3.4.jar gc.log
```

当然, 如果不想打开程序界面,也可以在后面加上其他参数,直接将分析结果输出到文件。

命令大致如下:

```mipsasm
java -jar gcviewer_1.3.4.jar gc.log summary.csv chart.png
```

以上命令将信息汇总到当前目录下的 Excel 文件 `summary.csv` 之中, 将图形信息保存为 `chart.png` 文件。

点击下载: [gcviewer的jar包及使用示例](http://download.csdn.net/detail/renfufei/9753654) 。

上图中, Chart 区域是对GC事件的图形化展示。包括各个内存池的大小和GC事件。上图中, 只有两个可视化指标: 蓝色线条表示堆内存的使用情况, 黑色的Bar则表示每次GC暂停时间的长短。

从图中可以看到, 内存使用量增长很快。一分钟左右就达到了堆内存的最大值. 堆内存几乎全部被消耗, 不能顺利分配新对象, 并引发频繁的 Full GC 事件. 这说明程序可能存在内存泄露, 或者启动时指定的内存空间不足。

从图中还可以看到 GC暂停的频率和持续时间。`30秒`之后, GC几乎不间断地运行,最长的暂停时间超过`1.4秒`。

在右边有三个选项卡。“**`Summary`**(摘要)” 中比较有用的是 “`Throughput`”(吞吐量百分比) 和 “`Number of GC pauses`”(GC暂停的次数), 以及“`Number of full GC pauses`”(Full GC 暂停的次数). 吞吐量显示了有效工作的时间比例, 剩下的部分就是GC的消耗。

以上示例中的吞吐量为 **`6.28%`**。这意味着有 **`93.72%`** 的CPU时间用在了GC上面. 很明显系统所面临的情况很糟糕 —— 宝贵的CPU时间没有用于执行实际工作, 而是在试图清理垃圾。

下一个有意思的地方是“**Pause**”(暂停)选项卡:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227005.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227005.png)

“`Pause`” 展示了GC暂停的总时间,平均值,最小值和最大值, 并且将 total 与minor/major 暂停分开统计。如果要优化程序的延迟指标, 这些统计可以很快判断出暂停时间是否过长。另外, 我们可以得出明确的信息: 累计暂停时间为 `634.59 秒`, GC暂停的总次数为 `3,938 次`, 这在`11分钟/660秒`的总运行时间里那不是一般的高。

更详细的GC暂停汇总信息, 请查看主界面中的 “**Event details**” 标签:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227006.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227006.png)

从“**Event details**” 标签中, 可以看到日志中所有重要的GC事件汇总: `普通GC停顿` 和 `Full GC 停顿次数`, 以及`并发执行数`, `非 stop-the-world 事件`等。此示例中, 可以看到一个明显的地方, Full GC 暂停严重影响了吞吐量和延迟, 依据是: `3,928 次 Full GC`, 暂停了`634秒`。

可以看到, GCViewer 能用图形界面快速展现异常的GC行为。一般来说, 图像化信息能迅速揭示以下症状:

- 低吞吐量。当应用的吞吐量下降到不能容忍的地步时, 有用工作的总时间就大量减少. 具体有多大的 “容忍度”(tolerable) 取决于具体场景。按照经验, 低于 90% 的有效时间就值得警惕了, 可能需要好好优化下GC。
- 单次GC的暂停时间过长。只要有一次GC停顿时间过长,就会影响程序的延迟指标. 例如, 延迟需求规定必须在 1000 ms以内完成交易, 那就不能容忍任何一次GC暂停超过1000毫秒。
- 堆内存使用率过高。如果老年代空间在 Full GC 之后仍然接近全满, 程序性能就会大幅降低, 可能是资源不足或者内存泄漏。这种症状会对吞吐量产生严重影响。

业界良心 —— 图形化展示的GC日志信息绝对是我们重磅推荐的。不用去阅读冗长而又复杂的GC日志,通过容易理解的图形, 也可以得到同样的信息。



## 分析器(Profilers)

下面介绍分析器([profilers](http://zeroturnaround.com/rebellabs/developer-productivity-report-2015-java-performance-survey-results/3/), Oracle官方翻译是:`抽样器`)。相对于前面的工具, 分析器只关心GC中的一部分领域. 本节我们也只关注分析器相关的GC功能。

首先警告 —— 不要认为分析器适用于所有的场景。分析器有时确实作用很大, 比如检测代码中的CPU热点时。但某些情况使用分析器不一定是个好方案。

对GC调优来说也是一样的。要检测是否因为GC而引起延迟或吞吐量问题时, 不需要使用分析器. 前面提到的工具( `jstat` 或 原生/可视化GC日志)就能更好更快地检测出是否存在GC问题. 特别是从生产环境中收集性能数据时, 最好不要使用分析器, 因为性能开销非常大。

如果确实需要对GC进行优化, 那么分析器就可以派上用场了, 可以对 Object 的创建信息一目了然. 换个角度看, 如果GC暂停的原因不在某个内存池中, 那就只会是因为创建对象太多了。 所有分析器都能够跟踪对象分配(via allocation profiling), 根据内存分配的轨迹, 让你知道 **实际驻留在内存中的是哪些对象**。

分配分析能定位到在哪个地方创建了大量的对象. 使用分析器辅助进行GC调优的好处是, 能确定哪种类型的对象最占用内存, 以及哪些线程创建了最多的对象。

下面我们通过实例介绍3种分配分析器: **`hprof`**, **`JVisualV`M** 和 **`AProf`**。实际上还有很多分析器可供选择, 有商业产品,也有免费工具, 但其功能和应用基本上都是类似的。



### hprof

[hprof 分析器](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html)内置于JDK之中。 在各种环境下都可以使用, 一般优先使用这款工具。

要让 `hprof` 和程序一起运行, 需要修改启动脚本, 类似这样:

```avrasm
java -agentlib:hprof=heap=sites com.yourcompany.YourApplication
```

在程序退出时,会将分配信息dump(转储)到工作目录下的 `java.hprof.txt` 文件中。使用文本编辑器打开, 并搜索 “**SITES BEGIN**” 关键字, 可以看到:

```mipsasm
SITES BEGIN (ordered by live bytes) Tue Dec  8 11:16:15 2015
          percent          live          alloc'ed  stack class
 rank   self  accum     bytes objs     bytes  objs trace name
    1  64.43% 4.43%   8370336 20121  27513408 66138 302116 int[]
    2  3.26% 88.49%    482976 20124   1587696 66154 302104 java.util.ArrayList
    3  1.76% 88.74%    241704 20121   1587312 66138 302115 eu.plumbr.demo.largeheap.ClonableClass0006
    ... 部分省略 ...

SITES END
```

从以上片段可以看到, allocations 是根据每次创建的对象数量来排序的。第一行显示所有对象中有 **`64.43%`** 的对象是整型数组(`int[]`), 在标识为 `302116` 的位置创建。搜索 “**TRACE 302116**” 可以看到:

```css
TRACE 302116:   
    eu.plumbr.demo.largeheap.ClonableClass0006.<init>(GeneratorClass.java:11)
    sun.reflect.GeneratedConstructorAccessor7.newInstance(<Unknown Source>:Unknown line)
    sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    java.lang.reflect.Constructor.newInstance(Constructor.java:422)
```

现在, 知道有 `64.43%` 的对象是整数数组, 在 `ClonableClass0006` 类的构造函数中, 第11行的位置, 接下来就可以优化代码, 以减少GC的压力。



### Java VisualVM

本章前面的第一部分, 在监控 JVM 的GC行为工具时介绍了 JVisualVM , 本节介绍其在分配分析上的应用。

JVisualVM 通过GUI的方式连接到正在运行的JVM。 连接上目标JVM之后 :

1. 打开 “工具” –> “选项” 菜单, 点击 **性能分析(Profiler)** 标签, 新增配置, 选择 Profiler 内存, 确保勾选了 “Record allocations stack traces”(记录分配栈跟踪)。
2. 勾选 “Settings”(设置) 复选框, 在内存设置标签下,修改预设配置。
3. 点击 “Memory”(内存) 按钮开始进行内存分析。
4. 让程序运行一段时间,以收集关于对象分配的足够信息。
5. 单击下方的 “Snapshot”(快照) 按钮。可以获取收集到的快照信息。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227007.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227007.png)

完成上面的步骤后, 可以得到类似这样的信息:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227008.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227008.png)

上图按照每个类被创建的对象数量多少来排序。看第一行可以知道, 创建的最多的对象是 `int[]` 数组. 鼠标右键单击这行, 就可以看到这些对象都在哪些地方创建的:

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227009.png)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812227009.png)

与 `hprof` 相比, JVisualVM 更加容易使用 —— 比如上面的截图中, 在一个地方就可以看到所有`int[]` 的分配信息, 所以多次在同一处代码进行分配的情况就很容易发现。



### AProf

最重要的一款分析器,是由 Devexperts 开发的 **[AProf](https://code.devexperts.com/display/AProf/About+Aprof)**。 内存分配分析器 AProf 也被打包为 Java agent 的形式。

用 AProf 分析应用程序, 需要修改 JVM 启动脚本,类似这样:

```javascript
java -javaagent:/path-to/aprof.jar com.yourcompany.YourApplication
```

重启应用之后, 工作目录下会生成一个 `aprof.txt` 文件。此文件每分钟更新一次, 包含这样的信息:

```mipsasm
========================================================================================================================
TOTAL allocation dump for 91,289 ms (0h01m31s)
Allocated 1,769,670,584 bytes in 24,868,088 objects of 425 classes in 2,127 locations
========================================================================================================================

Top allocation-inducing locations with the data types allocated from them
------------------------------------------------------------------------------------------------------------------------
eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 1,423,675,776 (80.44%) bytes in 17,113,721 (68.81%) objects (avg size 83 bytes)
    int[]: 711,322,976 (40.19%) bytes in 1,709,911 (6.87%) objects (avg size 416 bytes)
    char[]: 369,550,816 (20.88%) bytes in 5,132,759 (20.63%) objects (avg size 72 bytes)
    java.lang.reflect.Constructor: 136,800,000 (7.73%) bytes in 1,710,000 (6.87%) objects (avg size 80 bytes)
    java.lang.Object[]: 41,079,872 (2.32%) bytes in 1,710,712 (6.87%) objects (avg size 24 bytes)
    java.lang.String: 41,063,496 (2.32%) bytes in 1,710,979 (6.88%) objects (avg size 24 bytes)
    java.util.ArrayList:41,050,680(2.31%) bytes in1,710,445(6.87%) objects (avg size 24 bytes)... cut for brevity ...
```

上面的输出是按照 `size` 进行排序的。可以看出, `80.44%` 的 bytes 和 `68.81%` 的 objects 是在 `ManyTargetsGarbageProducer.newRandomClassObject()` 方法中分配的。 其中, **int[]** 数组占用了 `40.19%` 的内存, 是最大的一个。

继续往下看, 会发现 `allocation traces`(分配痕迹)相关的内容, 也是以 allocation size 排序的:

```mipsasm
Top allocated data types with reverse location traces
------------------------------------------------------------------------------------------------------------------------
int[]: 725,306,304 (40.98%) bytes in 1,954,234 (7.85%) objects (avg size 371 bytes)
    eu.plumbr.demo.largeheap.ClonableClass0006.: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
        java.lang.reflect.Constructor.newInstance: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
            eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 38,357,280 (2.16%) bytes in 92,205 (0.37%) objects (avg size 416 bytes)
            java.lang.reflect.Constructor.newInstance: 416 (0.00%) bytes in 1 (0.00%) objects (avg size 416 bytes)
... cut for brevity ...
```

可以看到, `int[]` 数组的分配, 在 `ClonableClass0006` 构造函数中继续增大。

和其他工具一样, `AProf` 揭露了 分配的大小以及位置信息(`allocation size and locations`), 从而能够快速找到最耗内存的部分。在我们看来, **AProf** 是最有用的分配分析器, 因为它只专注于内存分配, 所以做得最好。 当然, 这款工具是开源免费的, 资源开销也最小。

# GC 调优(基础篇)

GC调优(Tuning Garbage Collection)和其他性能调优是同样的原理。初学者可能会被 200 多个 GC参数弄得一头雾水, 然后随便调整几个来试试结果,又或者修改几行代码来测试。其实只要参照下面的步骤，就能保证你的调优方向正确:

1. 列出性能调优指标(State your performance goals)
2. 执行测试(Run tests)
3. 检查结果(Measure the results)
4. 与目标进行对比(Compare the results with the goals)
5. 如果达不到指标, 修改配置参数, 然后继续测试(go back to running tests)

第一步, 我们需要做的事情就是: 制定明确的GC性能指标。对所有性能监控和管理来说, 有三个维度是通用的:

- Latency(延迟)
- Throughput(吞吐量)
- Capacity(系统容量)

我们先讲解基本概念,然后再演示如何使用这些指标。如果您对 延迟、吞吐量和系统容量等概念很熟悉, 可以跳过这一小节。

## 核心概念(Core Concepts)

我们先来看一家工厂的装配流水线。工人在流水线将现成的组件按顺序拼接,组装成自行车。通过实地观测, 我们发现从组件进入生产线，到另一端组装成自行车需要4小时。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812226001.jpg)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812226001.jpg)

继续观察,我们还发现,此后每分钟就有1辆自行车完成组装, 每天24小时,一直如此。将这个模型简化, 并忽略维护窗口期后得出结论： 这条流水线每小时可以组装60辆自行车。

> 说明: 时间窗口/窗口期，请类比车站卖票的窗口，是一段规定/限定做某件事的时间段。

通过这两种测量方法, 就知道了生产线的相关性能信息： 延迟与吞吐量:

- 生产线的延迟: 4小时
- 生产线的吞吐量: 60辆/小时

请注意, 衡量延迟的时间单位根据具体需要而确定 —— 从纳秒(nanosecond)到几千年(millennia)都有可能。系统的吞吐量是每个单位时间内完成的操作。操作(Operations)一般是特定系统相关的东西。在本例中,选择的时间单位是小时, 操作就是对自行车的组装。

掌握了延迟和吞吐量两个概念之后, 让我们对这个工厂来进行实际的调优。自行车的需求在一段时间内都很稳定, 生产线组装自行车有四个小时延迟, 而吞吐量在几个月以来都很稳定: 60辆/小时。假设某个销售团队突然业绩暴涨, 对自行车的需求增加了1倍。客户每天需要的自行车不再是 60 * 24 = 1440辆, 而是 2*1440 = 2880辆/天。老板对工厂的产能不满意，想要做些调整以提升产能。

看起来总经理很容易得出正确的判断, 系统的延迟没法子进行处理 —— 他关注的是每天的自行车生产总量。得出这个结论以后, 假若工厂资金充足, 那么应该立即采取措施, 改善吞吐量以增加产能。

我们很快会看到, 这家工厂有两条相同的生产线。每条生产线一分钟可以组装一辆成品自行车。 可以想象，每天生产的自行车数量会增加一倍。达到 2880辆/天。要注意的是, 不需要减少自行车的装配时间 —— 从开始到结束依然需要 4 小时。

[![img](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812226002.jpg)](https://gitee.com/chenssy/blog-home/raw/master/image/201812/gc-xnty/201812226002.jpg)

巧合的是，这样进行的性能优化,同时增加了吞吐量和产能。一般来说，我们会先测量当前的系统性能, 再设定新目标, 只优化系统的某个方面来满足性能指标。

在这里做了一个很重要的决定 —— 要增加吞吐量,而不是减小延迟。在增加吞吐量的同时, 也需要增加系统容量。比起原来的情况, 现在需要两条流水线来生产出所需的自行车。在这种情况下, 增加系统的吞吐量并不是免费的, 需要水平扩展, 以满足增加的吞吐量需求。

在处理性能问题时, 应该考虑到还有另一种看似不相关的解决办法。假如生产线的延迟从1分钟降低为30秒,那么吞吐量同样可以增长 1 倍。

或者是降低延迟, 或者是客户非常有钱。软件工程里有一种相似的说法 —— 每个性能问题背后,总有两种不同的解决办法。 可以用更多的机器, 或者是花精力来改善性能低下的代码。



### Latency(延迟)

GC的延迟指标由一般的延迟需求决定。延迟指标通常如下所述:

- 所有交易必须在10秒内得到响应
- 90%的订单付款操作必须在3秒以内处理完成
- 推荐商品必须在 100 ms 内展示到用户面前

面对这类性能指标时, 需要确保在交易过程中, GC暂停不能占用太多时间，否则就满足不了指标。“不能占用太多” 的意思需要视具体情况而定, 还要考虑到其他因素, 比如外部数据源的交互时间(round-trips), 锁竞争(lock contention), 以及其他的安全点等等。

假设性能需求为: `90%`的交易要在 `1000ms` 以内完成, 每次交易最长不能超过 `10秒`。 根据经验, 假设GC暂停时间比例不能超过10%。 也就是说, 90%的GC暂停必须在 `100ms` 内结束, 也不能有超过 `1000ms` 的GC暂停。为简单起见, 我们忽略在同一次交易过程中发生多次GC停顿的可能性。

有了正式的需求,下一步就是检查暂停时间。在本节中我们通过查看GC日志, 检查一下GC暂停的时间。相关的信息散落在不同的日志片段中, 看下面的数据:

```yaml
2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
        [PSYoungGen: 93677K->70109K(254976K)]
        [ParOldGen: 499597K->511230K(761856K)]
        593275K->581339K(1016832K),
        [Metaspace: 2936K->2936K(1056768K)]
    , 0.0713174 secs]
    [Times: user=0.21 sys=0.02, real=0.07 secs
```

这表示一次GC暂停, 在 `2015-06-04T13:34:16` 这个时刻触发. 对应于JVM启动之后的 `2,578 ms`。

此事件将应用线程暂停了 `0.0713174` 秒。虽然花费的总时间为 210 ms, 但因为是多核CPU机器, 所以最重要的数字是应用线程被暂停的总时间, 这里使用的是并行GC, 所以暂停时间大约为 `70ms` 。 这次GC的暂停时间小于 `100ms` 的阈值，满足需求。

继续分析, 从所有GC日志中提取出暂停相关的数据, 汇总之后就可以得知是否满足需求。



### Throughput(吞吐量)

吞吐量和延迟指标有很大区别。当然两者都是根据一般吞吐量需求而得出的。一般吞吐量需求(Generic requirements for throughput) 类似这样:

- 解决方案每天必须处理 100万个订单
- 解决方案必须支持1000个登录用户,同时在5-10秒内执行某个操作: A、B或C
- 每周对所有客户进行统计, 时间不能超过6小时，时间窗口为每周日晚12点到次日6点之间。

可以看出,吞吐量需求不是针对单个操作的, 而是在给定的时间内, 系统必须完成多少个操作。和延迟需求类似, GC调优也需要确定GC行为所消耗的总时间。每个系统能接受的时间不同, 一般来说, GC占用的总时间比不能超过 `10%`。

现在假设需求为: 每分钟处理 1000 笔交易。同时, 每分钟GC暂停的总时间不能超过6秒(即10%)。

有了正式的需求, 下一步就是获取相关的信息。依然是从GC日志中提取数据, 可以看到类似这样的信息:

```yaml
2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
        [PSYoungGen: 93677K->70109K(254976K)]
        [ParOldGen: 499597K->511230K(761856K)]
        593275K->581339K(1016832K),
        [Metaspace: 2936K->2936K(1056768K)],
     0.0713174 secs]
     [Times: user=0.21 sys=0.02, real=0.07 secs
```

此时我们对 用户耗时(user)和系统耗时(sys)感兴趣, 而不关心实际耗时(real)。在这里, 我们关心的时间为 `0.23s`(user + sys = 0.21 + 0.02 s), 这段时间内, GC暂停占用了 cpu 资源。 重要的是, 系统运行在多核机器上, 转换为实际的停顿时间(stop-the-world)为 `0.0713174秒`, 下面的计算会用到这个数字。

提取出有用的信息后, 剩下要做的就是统计每分钟内GC暂停的总时间。看看是否满足需求: 每分钟内总的暂停时间不得超过6000毫秒(6秒)。



### Capacity(系统容量)

系统容量(Capacity)需求,是在达成吞吐量和延迟指标的情况下,对硬件环境的额外约束。这类需求大多是来源于计算资源或者预算方面的原因。例如:

- 系统必须能部署到小于512 MB内存的Android设备上
- 系统必须部署在Amazon EC2实例上, 配置不得超过 c3.xlarge(4核8GB)。
- 每月的 Amazon EC2 账单不得超过 `$12,000`

因此, 在满足延迟和吞吐量需求的基础上必须考虑系统容量。可以说, 假若有无限的计算资源可供挥霍, 那么任何 延迟和吞吐量指标 都不成问题, 但现实情况是, 预算(budget)和其他约束限制了可用的资源。



## 相关示例

介绍完性能调优的三个维度后, 我们来进行实际的操作以达成GC性能指标。

请看下面的代码:

```java
//imports skipped for brevity
public class Producer implements Runnable {

  private static ScheduledExecutorService executorService
         = Executors.newScheduledThreadPool(2);

  private Deque<byte[]> deque;
  private int objectSize;
  private int queueSize;

  public Producer(int objectSize, int ttl) {
    this.deque = new ArrayDeque<byte[]>();
    this.objectSize = objectSize;
    this.queueSize = ttl * 1000;
  }

  @Override
  public void run() {
    for (int i = 0; i < 100; i++) {
        deque.add(new byte[objectSize]);
        if (deque.size() > queueSize) {
            deque.poll();
        }
    }
  }

  public static void main(String[] args)
        throws InterruptedException {
    executorService.scheduleAtFixedRate(
        new Producer(200 * 1024 * 1024 / 1000, 5),
        0, 100, TimeUnit.MILLISECONDS
    );
    executorService.scheduleAtFixedRate(
        new Producer(50 * 1024 * 1024 / 1000, 120),
        0, 100, TimeUnit.MILLISECONDS);
    TimeUnit.MINUTES.sleep(10);
    executorService.shutdownNow();}}
```

这段程序代码, 每 100毫秒 提交两个作业(job)来。每个作业都模拟特定的生命周期: 创建对象, 然后在预定的时间释放, 接着就不管了, 由GC来自动回收占用的内存。

在运行这个示例程序时，通过以下JVM参数打开GC日志记录:

```diff
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps
```

还应该加上JVM参数 `-Xloggc`以指定GC日志的存储位置,类似这样:

```diff
-Xloggc:C:\\Producer_gc.log
```

在日志文件中可以看到GC的行为, 类似下面这样:

```yaml
2015-06-04T13:34:16.119-0200: 1.723: [GC (Allocation Failure)
        [PSYoungGen: 114016K->73191K(234496K)]
    421540K->421269K(745984K),
    0.0858176 secs]
    [Times: user=0.04 sys=0.06, real=0.09 secs]

2015-06-04T13:34:16.738-0200: 2.342: [GC (Allocation Failure)
        [PSYoungGen: 234462K->93677K(254976K)]
    582540K->593275K(766464K),
    0.2357086 secs]
    [Times: user=0.11 sys=0.14, real=0.24 secs]

2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
        [PSYoungGen: 93677K->70109K(254976K)]
        [ParOldGen: 499597K->511230K(761856K)]
    593275K->581339K(1016832K),
        [Metaspace: 2936K->2936K(1056768K)],
    0.0713174 secs]
    [Times: user=0.21 sys=0.02, real=0.07 secs]
```

基于日志中的信息, 可以通过三个优化目标来提升性能:

1. 确保最坏情况下,GC暂停时间不超过预定阀值
2. 确保线程暂停的总时间不超过预定阀值
3. 在确保达到延迟和吞吐量指标的情况下, 降低硬件配置以及成本。

为此, 用三种不同的配置, 将代码运行10分钟, 得到了三种不同的结果, 汇总如下:

| 堆内存大小(Heap) | GC算法(GC Algorithm)    | 有效时间比(Useful work) | 最长停顿时间(Longest pause) |
| ---------------- | ----------------------- | ----------------------- | --------------------------- |
| -Xmx12g          | -XX:+UseConcMarkSweepGC | 89.8%                   | 560 ms                      |
| -Xmx12g          | -XX:+UseParallelGC      | 91.5%                   | 1,104 ms                    |
| -Xmx8g           | -XX:+UseConcMarkSweepGC | 66.3%                   | 1,610 ms                    |

使用不同的GC算法,和不同的内存配置,运行相同的代码, 以测量GC暂停时间与 延迟、吞吐量的关系。实验的细节和结果在后面章节详细介绍。

注意, 为了尽量简单, 示例中只改变了很少的输入参数, 此实验也没有在不同CPU数量或者不同的堆布局下进行测试。



### Tuning for Latency(调优延迟指标)

假设有一个需求, 每次作业必须在 1000ms 内处理完成。我们知道, 实际的作业处理只需要100 ms，简化后， 两者相减就可以算出对 GC暂停的延迟要求。现在需求变成: GC暂停不能超过900ms。这个问题很容易找到答案, 只需要解析GC日志文件, 并找出GC暂停中最大的那个暂停时间即可。

再来看测试所用的三个配置:

| 堆内存大小(Heap) | GC算法(GC Algorithm)    | 有效时间比(Useful work) | 最长停顿时间(Longest pause) |
| ---------------- | ----------------------- | ----------------------- | --------------------------- |
| -Xmx12g          | -XX:+UseConcMarkSweepGC | 89.8%                   | 560 ms                      |
| -Xmx12g          | -XX:+UseParallelGC      | 91.5%                   | 1,104 ms                    |
| -Xmx8g           | -XX:+UseConcMarkSweepGC | 66.3%                   | 1,610 ms                    |

可以看到,其中有一个配置达到了要求。运行的参数为:

```mipsasm
java -Xmx12g -XX:+UseConcMarkSweepGC Producer
```

对应的GC日志中,暂停时间最大为 `560 ms`, 这达到了延迟指标 `900 ms` 的要求。如果还满足吞吐量和系统容量需求的话,就可以说成功达成了GC调优目标, 调优结束。



### Tuning for Throughput(吞吐量调优)

假定吞吐量指标为: 每小时完成 1300万次操作处理。同样是上面的配置, 其中有一种配置满足了需求:

| 堆内存大小(Heap) | GC算法(GC Algorithm)    | 有效时间比(Useful work) | 最长停顿时间(Longest pause) |
| ---------------- | ----------------------- | ----------------------- | --------------------------- |
| -Xmx12g          | -XX:+UseConcMarkSweepGC | 89.8%                   | 560 ms                      |
| -Xmx12g          | -XX:+UseParallelGC      | 91.5%                   | 1,104 ms                    |
| -Xmx8g           | -XX:+UseConcMarkSweepGC | 66.3%                   | 1,610 ms                    |

此配置对应的命令行参数为:

```mipsasm
java -Xmx12g -XX:+UseParallelGC Producer
```

可以看到,GC占用了 8.5%的CPU时间,剩下的 `91.5%` 是有效的计算时间。为简单起见, 忽略示例中的其他安全点。现在需要考虑:

1. 每个CPU核心处理一次作业需要耗时 `100ms`
2. 因此, 一分钟内每个核心可以执行 60,000 次操作(每个job完成100次操作)
3. 一小时内, 一个核心可以执行 360万次操作
4. 有四个CPU内核, 则每小时可以执行: 4 x 3.6M = 1440万次操作

理论上，通过简单的计算就可以得出结论, 每小时可以执行的操作数为: `14.4 M * 91.5% = 13,176,000` 次, 满足需求。

值得一提的是, 假若还要满足延迟指标, 那就有问题了, 最坏情况下, GC暂停时间为 `1,104 ms`, 最大延迟时间是前一种配置的两倍。



### Tuning for Capacity(调优系统容量)

假设需要将软件部署到服务器上(commodity-class hardware), 配置为 `4核10G`。这样的话, 系统容量的要求就变成: 最大的堆内存空间不能超过 `8GB`。有了这个需求, 我们需要调整为第三套配置进行测试:

| 堆内存大小(Heap) | GC算法(GC Algorithm)    | 有效时间比(Useful work) | 最长停顿时间(Longest pause) |
| ---------------- | ----------------------- | ----------------------- | --------------------------- |
| -Xmx12g          | -XX:+UseConcMarkSweepGC | 89.8%                   | 560 ms                      |
| -Xmx12g          | -XX:+UseParallelGC      | 91.5%                   | 1,104 ms                    |
| -Xmx8g           | -XX:+UseConcMarkSweepGC | 66.3%                   | 1,610 ms                    |

程序可以通过如下参数执行:

```mipsasm
java -Xmx8g -XX:+UseConcMarkSweepGC Producer
```

测试结果是延迟大幅增长, 吞吐量同样大幅降低:

- 现在,GC占用了更多的CPU资源, 这个配置只有 `66.3%` 的有效CPU时间。因此,这个配置让吞吐量从最好的情况 13,176,000 操作/小时 下降到 不足 9,547,200次操作/小时.
- 最坏情况下的延迟变成了 1,610 ms, 而不再是 560ms。

通过对这三个维度的介绍, 你应该了解, 不是简单的进行“性能(performance)”优化, 而是需要从三种不同的维度来进行考虑, 测量, 并调优延迟和吞吐量, 此外还需要考虑系统容量的约束。

# GC 调优(命令篇)

运用jvm自带的命令可以方便的在生产监控和打印堆栈的日志信息帮忙我们来定位问题！虽然jvm调优成熟的工具已经有很多：jconsole、大名鼎鼎的VisualVM，IBM的Memory Analyzer等等，但是在生产环境出现问题的时候，一方面工具的使用会有所限制，另一方面喜欢装X的我们，总喜欢在出现问题的时候在终端输入一些命令来解决。所有的工具几乎都是依赖于jdk的接口和底层的这些命令，研究这些命令的使用也让我们更能了解jvm构成和特性。

Sun JDK监控和故障处理命令有jps jstat jmap jhat jstack jinfo下面做一一介绍



## jps

JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。



### 命令格式

```less
jps [options] [hostid]
```



### option参数

> - -l : 输出主类全名或jar路径
> - -q : 只输出LVMID
> - -m : 输出JVM启动时传递给main()的参数
> - -v : 输出JVM启动时显示指定的JVM参数

其中[option]、[hostid]参数也可以不写。



### 示例

```dos
$ jps -l -m
  28920 org.apache.catalina.startup.Bootstrap start
  11589 org.apache.catalina.startup.Bootstrap start
  25816 sun.tools.jps.Jps -l -m
```

## jstat

jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。



### 命令格式

```less
jstat [option] LVMID [interval] [count]
```



### 参数

> - [option] : 操作参数
> - LVMID : 本地虚拟机进程ID
> - [interval] : 连续输出的时间间隔
> - [count] : 连续输出的次数

#### option 参数总览

| Option           | Displays…                                                    |
| ---------------- | ------------------------------------------------------------ |
| class            | class loader的行为统计。Statistics on the behavior of the class loader. |
| compiler         | HotSpt JIT编译器行为统计。Statistics of the behavior of the HotSpot Just-in-Time compiler. |
| gc               | 垃圾回收堆的行为统计。Statistics of the behavior of the garbage collected heap. |
| gccapacity       | 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计。Statistics of the capacities of the generations and their corresponding spaces. |
| gcutil           | 垃圾回收统计概述。Summary of garbage collection statistics.  |
| gccause          | 垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因。Summary of garbage collection statistics (same as -gcutil), with the cause of the last and |
| gcnew            | 新生代行为统计。Statistics of the behavior of the new generation. |
| gcnewcapacity    | 新生代与其相应的内存空间的统计。Statistics of the sizes of the new generations and its corresponding spaces. |
| gcold            | 年老代和永生代行为统计。Statistics of the behavior of the old and permanent generations. |
| gcoldcapacity    | 年老代行为统计。Statistics of the sizes of the old generation. |
| gcpermcapacity   | 永生代行为统计。Statistics of the sizes of the permanent generation. |
| printcompilation | HotSpot编译方法统计。HotSpot compilation method statistics.  |

#### option 参数详解

##### -class

监视类装载、卸载数量、总空间以及耗费的时间

```ruby
$ jstat -class 11589
 Loaded  Bytes  Unloaded  Bytes     Time   
  7035  14506.3     0     0.0       3.67
```

> - Loaded : 加载class的数量
> - Bytes : class字节大小
> - Unloaded : 未加载class的数量
> - Bytes : 未加载class的字节大小
> - Time : 加载时间

##### -compiler

输出JIT编译过的方法数量耗时等

```yaml
$ jstat -compiler 1262
Compiled Failed Invalid   Time   FailedType FailedMethod
    2573      1       0    47.60          1 org/apache/catalina/loader/WebappClassLoader findResourceInternal  
```

> - Compiled : 编译数量
> - Failed : 编译失败数量
> - Invalid : 无效数量
> - Time : 编译耗时
> - FailedType : 失败类型
> - FailedMethod : 失败方法的全限定名

##### -gc

垃圾回收堆的行为统计，常用命令

```shell
$ jstat -gc 1262
 S0C    S1C     S0U     S1U   EC       EU        OC         OU        PC       PU         YGC    YGCT    FGC    FGCT     GCT   
26112.0 24064.0 6562.5  0.0   564224.0 76274.5   434176.0   388518.3  524288.0 42724.7    320    6.417   1      0.398    6.815
```

C即Capacity 总容量，U即Used 已使用的容量

> - S0C : survivor0区的总容量
> - S1C : survivor1区的总容量
> - S0U : survivor0区已使用的容量
> - S1U : survivor1区已使用的容量
> - EC : Eden区的总容量
> - EU : Eden区已使用的容量
> - OC : Old区的总容量
> - OU : Old区已使用的容量
> - PC 当前perm的容量 (KB)
> - PU perm的使用 (KB)
> - YGC : 新生代垃圾回收次数
> - YGCT : 新生代垃圾回收时间
> - FGC : 老年代垃圾回收次数
> - FGCT : 老年代垃圾回收时间
> - GCT : 垃圾回收总消耗时间

```shell
$ jstat -gc 1262 2000 20
```

这个命令意思就是每隔2000ms输出1262的gc情况，一共输出20次

##### -gccapacity

同-gc，不过还会输出Java堆各区域使用到的最大、最小空间

```shell
$ jstat -gccapacity 1262
 NGCMN    NGCMX     NGC    S0C   S1C       EC         OGCMN      OGCMX      OGC        OC       PGCMN    PGCMX     PGC      PC         YGC    FGC 
614400.0 614400.0 614400.0 26112.0 24064.0 564224.0   434176.0   434176.0   434176.0   434176.0 524288.0 1048576.0 524288.0 524288.0    320     1  
```

> - NGCMN : 新生代占用的最小空间
> - NGCMX : 新生代占用的最大空间
> - OGCMN : 老年代占用的最小空间
> - OGCMX : 老年代占用的最大空间
> - OGC：当前年老代的容量 (KB)
> - OC：当前年老代的空间 (KB)
> - PGCMN : perm占用的最小空间
> - PGCMX : perm占用的最大空间

##### -gcutil

同-gc，不过输出的是已使用空间占总空间的百分比

```shell
$ jstat -gcutil 28920
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
 12.45   0.00  33.85   0.00   4.44  4       0.242     0    0.000    0.242
```

##### -gccause

垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因

```shell
$ jstat -gccause 28920
  S0     S1     E      O      P       YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
 12.45   0.00  33.85   0.00   4.44      4    0.242     0    0.000    0.242   Allocation Failure   No GC  
```

> - LGCC：最近垃圾回收的原因
> - GCC：当前垃圾回收的原因

##### -gcnew

统计新生代的行为

```shell
$ jstat -gcnew 28920
 S0C      S1C      S0U        S1U  TT  MTT  DSS      EC        EU         YGC     YGCT  
 419392.0 419392.0 52231.8    0.0  6   6    209696.0 3355520.0 1172246.0  4       0.242
```

> - TT：Tenuring threshold(提升阈值)
> - MTT：最大的tenuring threshold
> - DSS：survivor区域大小 (KB)

##### -gcnewcapacity

新生代与其相应的内存空间的统计

```shell
$ jstat -gcnewcapacity 28920
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC        YGC   FGC 
 4194304.0  4194304.0  4194304.0 419392.0 419392.0 419392.0 419392.0  3355520.0  3355520.0     4     0
```

> - NGC:当前年轻代的容量 (KB)
> - S0CMX:最大的S0空间 (KB)
> - S0C:当前S0空间 (KB)
> - ECMX:最大eden空间 (KB)
> - EC:当前eden空间 (KB)

##### -gcold

统计旧生代的行为

```shell
$ jstat -gcold 28920
   PC       PU        OC           OU       YGC    FGC    FGCT     GCT   
1048576.0  46561.7   6291456.0     0.0      4      0      0.000    0.242
```

##### -gcoldcapacity

统计旧生代的大小和空间

```shell
$ jstat -gcoldcapacity 28920
   OGCMN       OGCMX        OGC         OC         YGC   FGC    FGCT     GCT   
  6291456.0   6291456.0   6291456.0   6291456.0     4     0    0.000    0.242
```

##### -gcpermcapacity

永生代行为统计

```shell
$ jstat -gcpermcapacity 28920
    PGCMN      PGCMX       PGC         PC      YGC   FGC    FGCT     GCT   
 1048576.0  2097152.0  1048576.0  1048576.0     4     0    0.000    0.242
```

##### -printcompilation

hotspot编译方法统计

```shell
$ jstat -printcompilation 28920
    Compiled  Size  Type Method
    1291      78     1    java/util/ArrayList indexOf
```

> - Compiled：被执行的编译任务的数量
> - Size：方法字节码的字节数
> - Type：编译类型
> - Method：编译方法的类名和方法名。类名使用”/” 代替 “.” 作为空间分隔符. 方法名是给出类的方法名. 格式是一致于HotSpot - XX:+PrintComplation 选项



## jmap

jmap(JVM Memory Map)命令用于生成heap dump文件，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候·自动生成dump文件。 jmap不仅能生成dump文件，还阔以查询finalize执行队列、Java堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。



### 命令格式

```less
jmap [option] LVMID
```



### option参数

> - dump : 生成堆转储快照
> - finalizerinfo : 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
> - heap : 显示Java堆详细信息
> - histo : 显示堆中对象的统计信息
> - permstat : to print permanent generation statistics
> - F : 当-dump没有响应时，强制生成dump快照



### 示例

##### -dump

常用格式

```lua
-dump::live,format=b,file=<filename> pid 
```

dump堆到文件,format指定输出格式，live指明是活着的对象,file指定文件名

```lua
$ jmap -dump:live,format=b,file=dump.hprof 28920
  Dumping heap to /home/xxx/dump.hprof ...
  Heap dump file created
```

dump.hprof这个后缀是为了后续可以直接用MAT(Memory Anlysis Tool)打开。

##### -finalizerinfo

打印等待回收对象的信息

```vhdl
$ jmap -finalizerinfo 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01
  Number of objects pending for finalization: 0
```

可以看到当前F-QUEUE队列中并没有等待Finalizer线程执行finalizer方法的对象。

##### -heap

打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况,可以用此来判断内存目前的使用情况以及垃圾回收情况

```cpp
$ jmap -heap 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01  

  using thread-local object allocation.
  Parallel GC with 4 thread(s)//GC 方式  

  Heap Configuration: //堆内存初始化配置
     MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
     MaxHeapFreeRatio = 100 //对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
     MaxHeapSize      = 2082471936 (1986.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
     NewSize          = 1310720 (1.25MB)//对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
     MaxNewSize       = 17592186044415 MB//对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
     OldSize          = 5439488 (5.1875MB)//对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
     NewRatio         = 2 //对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
     SurvivorRatio    = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值 
     PermSize         = 21757952 (20.75MB)  //对应jvm启动参数-XX:PermSize=<value>:设置JVM堆的‘永生代’的初始大小
     MaxPermSize      = 85983232 (82.0MB)//对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小
     G1HeapRegionSize = 0 (0.0MB)  

  Heap Usage://堆内存使用情况
  PS Young Generation
  Eden Space://Eden区内存分布
     capacity = 33030144 (31.5MB)//Eden区总容量
     used     = 1524040 (1.4534378051757812MB)  //Eden区已使用
     free     = 31506104 (30.04656219482422MB)  //Eden区剩余容量
     4.614088270399305% used //Eden区使用比率
  From Space:  //其中一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  To Space:  //另一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  PS Old Generation //当前的Old区内存分布
     capacity = 86507520 (82.5MB)
     used     = 0 (0.0MB)
     free     = 86507520 (82.5MB)
     0.0% used
  PS Perm Generation//当前的 “永生代” 内存分布
     capacity = 22020096 (21.0MB)
     used     = 2496528 (2.3808746337890625MB)
     free     = 19523568 (18.619125366210938MB)
     11.337498256138392% used  

  670 interned Strings occupying 43720 bytes.
```

可以很清楚的看到Java堆中各个区域目前的情况。

##### -histo

打印堆的对象统计，包括对象数、内存大小等等 （因为在dump:live前会进行full gc，如果带上live则只统计活对象，因此不加live的堆大小要大于加live堆的大小 ）

```xml
$ jmap -histo:live 28920 | more
 num     #instances         #bytes  class name
----------------------------------------------
   1:         83613       12012248  <constMethodKlass>
   2:         23868       11450280  [B
   3:         83613       10716064  <methodKlass>
   4:         76287       10412128  [C
   5:          8227        9021176  <constantPoolKlass>
   6:          8227        5830256  <instanceKlassKlass>
   7:          7031        5156480  <constantPoolCacheKlass>
   8:         73627        1767048  java.lang.String
   9:          2260        1348848  <methodDataKlass>
  10:          8856         849296  java.lang.Class
  ....
```

仅仅打印了前10行

`xml class name`是对象类型，说明如下：

```java
B  byte
C  char
D  double
F  float
I  int
J  long
Z  boolean
[  数组，如[I表示int[]
[L+类名 其他对象
```

##### -permstat

打印Java堆内存的永久保存区域的类加载器的智能统计信息。对于每个类加载器而言，它的名称、活跃度、地址、父类加载器、它所加载的类的数量和大小都会被打印。此外，包含的字符串数量和大小也会被打印。

```fsharp
$ jmap -permstat 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01
  finding class loader instances ..done.
  computing per loader stat ..done.
  please wait.. computing liveness.liveness analysis may be inaccurate ...
  
  class_loader            classes bytes   parent_loader           alive?  type  
  <bootstrap>             3111    18154296          null          live    <internal>
  0x0000000600905cf8      1       1888    0x0000000600087f08      dead    sun/reflect/DelegatingClassLoader@0x00000007800500a0
  0x00000006008fcb48      1       1888    0x0000000600087f08      dead    sun/reflect/DelegatingClassLoader@0x00000007800500a0
  0x00000006016db798      0       0       0x00000006008d3fc0      dead    java/util/ResourceBundle$RBClassLoader@0x0000000780626ec0
  0x00000006008d6810      1       3056      null          dead    sun/reflect/DelegatingClassLoader@0x00000007800500a0
```

##### -F

强制模式。如果指定的pid没有响应，请使用jmap -dump或jmap -histo选项。此模式下，不支持live子选项。



## jhat

jhat(JVM Heap Analysis Tool)命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看。在此要注意，一般不会直接在服务器上进行分析，因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。



### 命令格式

```less
jhat [dumpfile]
```



### 参数

> - -stack false|true 关闭对象分配调用栈跟踪(tracking object allocation call stack)。 如果分配位置信息在堆转储中不可用. 则必须将此标志设置为 false. 默认值为 true.>
> - -refs false|true 关闭对象引用跟踪(tracking of references to objects)。 默认值为 true. 默认情况下, 返回的指针是指向其他特定对象的对象,如反向链接或输入引用(referrers or incoming references), 会统计/计算堆中的所有对象。>
> - -port port-number 设置 jhat HTTP server 的端口号. 默认值 7000.>
> - -exclude exclude-file 指定对象查询时需要排除的数据成员列表文件(a file that lists data members that should be excluded from the reachable objects query)。 例如, 如果文件列列出了 java.lang.String.value , 那么当从某个特定对象 Object o 计算可达的对象列表时, 引用路径涉及 java.lang.String.value 的都会被排除。>
> - -baseline exclude-file 指定一个基准堆转储(baseline heap dump)。 在两个 heap dumps 中有相同 object ID 的对象会被标记为不是新的(marked as not being new). 其他对象被标记为新的(new). 在比较两个不同的堆转储时很有用.>
> - -debug int 设置 debug 级别. 0 表示不输出调试信息。 值越大则表示输出更详细的 debug 信息.>
> - -version 启动后只显示版本信息就退出>
> - -J< flag > 因为 jhat 命令实际上会启动一个JVM来执行, 通过 -J 可以在启动JVM时传入一些启动参数. 例如, -J-Xmx512m 则指定运行 jhat 的Java虚拟机使用的最大堆内存为 512 MB. 如果需要使用多个JVM启动参数,则传入多个 -Jxxxxxx.



### 示例

```erlang
$ jhat -J-Xmx512m dump.hprof
  eading from dump.hprof...
  Dump file created Fri Mar 11 17:13:42 CST 2016
  Snapshot read, resolving...
  Resolving 271678 objects...
  Chasing references, expect 54 dots......................................................
  Eliminating duplicate references......................................................
  Snapshot resolved.
  Started HTTP server on port 7000
  Server is ready.
```

中间的-J-Xmx512m是在dump快照很大的情况下分配512M内存去启动HTTP服务器，运行完之后就可在浏览器打开Http://localhost:7000进行快照分析 堆快照分析主要在最后面的Heap Histogram里，里面根据class列出了dump的时候所有存活对象。

分析同样一个dump快照，MAT需要的额外内存比jhat要小的多的多，所以建议使用MAT来进行分析，当然也看个人偏好。



### 分析

打开浏览器Http://localhost:7000，该页面提供了几个查询功能可供使用：

```sql
All classes including platform
Show all members of the rootset
Show instance counts for all classes (including platform)
Show instance counts for all classes (excluding platform)
Show heap histogram
Show finalizer summary
Execute Object Query Language (OQL) query
```

一般查看堆异常情况主要看这个两个部分： Show instance counts for all classes (excluding platform)，平台外的所有对象信息。如下图：![img](http://ityouknow.com/assets/images/2016/jvm-jhat-excluding-paltform.png)
Show heap histogram 以树状图形式展示堆情况。如下图：![img](http://ityouknow.com/assets/images/2016/jvm-jhat-heap-histogram.png)
具体排查时需要结合代码，观察是否大量应该被回收的对象在一直被引用或者是否有占用内存特别大的对象无法被回收。
一般情况，会down到客户端用工具来分析



## jstack

jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。



### 命令格式

```less
jstack [option] LVMID
```



### option参数

> - -F : 当正常输出请求不被响应时，强制输出线程堆栈
> - -l : 除堆栈外，显示关于锁的附加信息
> - -m : 如果调用到本地方法的话，可以显示C/C++的堆栈

Jstack -l PID >> 123.txt 在发生死锁时可以用jstack -l pid来观察锁持有情况 >>123.txt dump到指定文件



### 示例

```x86asm
$ jstack -l 11494|more
2016-07-28 13:40:04
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.71-b01 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007febb0002000 nid=0x6b6f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
        - None

"http-bio-8005-exec-2" daemon prio=10 tid=0x00007feb94028000 nid=0x7b8c waiting on condition [0x00007fea8f56e000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000cae09b80> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:104)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:32)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
        - None
      .....
```

https://www.hollischuang.com/archives/110

### 分析

这里有一篇文章解释的很好 [分析打印出的文件内容](http://www.hollischuang.com/archives/110)

[https://www.hollischuang.com/archives/110](https://www.cnblogs.com/java-chen-hao/p/10653916.html#_labelTop)

## jinfo

jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数。 之前的jps -v口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo口令



### 命令格式

```less
jinfo [option] [args] LVMID
```



### option参数

> - -flag : 输出指定args参数的值
> - -flags : 不需要args参数，输出所有JVM参数的值
> - -sysprops : 输出系统属性，等同于System.getProperties()



### 示例

# Java开发必须掌握的线上问题排查命令

**正文**

作为一个合格的开发人员，不仅要能写得一手还代码，还有一项很重要的技能就是排查问题。这里提到的排查问题不仅仅是在coding的过程中debug等，还包括的就是线上问题的排查。由于在生产环境中，一般没办法debug（其实有些问题，debug也白扯。。。）,所以我们需要借助一些常用命令来查看运行时的具体情况，这些运行时信息包括但不限于运行日志、异常堆栈、堆使用情况、GC情况、JVM参数情况、线程情况等。

给一个系统定位问题的时候，知识、经验是关键，数据是依据，工具是运用知识处理数据的手段。为了便于我们排查和解决问题，Sun公司为我们提供了一些常用命令。这些命令一般都是jdk/lib/tools.jar中类库的一层薄包装。随着JVM的安装一起被安装到机器中，在bin目录中。下面就来认识一下这些命令以及具体使用方式。文中涉及到的所有命令的详细信息可以参考 [Java命令学习系列文章](http://www.hollischuang.com/archives/tag/java命令学习系列)



## jps



### 功能

显示当前所有java进程pid的命令。



### 常用指令

`jps`：显示当前用户的所有java进程的PID

`jps -v 3331`：显示虚拟机参数

`jps -m 3331`：显示传递给main()函数的参数

`jps -l 3331`：显示主类的全路径



### [详细介绍](http://www.hollischuang.com/archives/105)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## jinfo



### 功能

实时查看和调整虚拟机参数，可以显示未被显示指定的参数的默认值（`jps -v 则不能`）。

> jdk8中已经不支持该命令。



### 常用指令

`jinfo -flag CMSIniniatingOccupancyFration 1444`：查询CMSIniniatingOccupancyFration参数值



### [详细介绍](http://www.hollischuang.com/archives/1094)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## jstat



### 功能

显示进程中的类装载、内存、垃圾收集、JIT编译等运行数据。



### 常用指令

`jstat -gc 3331 250 20` ：查询进程2764的垃圾收集情况，每250毫秒查询一次，一共查询20次。

`jstat -gccause`：额外输出上次GC原因

`jstat -calss`：件事类装载、类卸载、总空间以及所消耗的时间



### [详细介绍](http://www.hollischuang.com/archives/481)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## jmap



### 功能

生成堆转储快照（heapdump）



### 常用指令

`jmap -heap 3331`：查看java 堆（heap）使用情况

`jmap -histo 3331`：查看堆内存(histogram)中的对象数量及大小

`jmap -histo:live 3331`：JVM会先触发gc，然后再统计信息

`jmap -dump:format=b,file=heapDump 3331`：将内存使用的详细情况输出到文件，之后一般使用其他工具进行分析。



### [详细介绍](http://www.hollischuang.com/archives/303)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## jhat



### 功能

一般与jmap搭配使用，用来分析jmap生成的堆转储文件。

> 由于有很多可视化工具（Eclipse Memory Analyzer 、IBM HeapAnalyzer）可以替代，所以很少用。不过在没有可视化工具的机器上也是可用的。



### 常用指令

`jmap -dump:format=b,file=heapDump 3331` + `jhat heapDump`：解析Java堆转储文件,并启动一个 web server



### [详细介绍](http://www.hollischuang.com/archives/1047)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## jstack



### 功能

生成当前时刻的线程快照。



### 常用指令

`jstack 3331`：查看线程情况

`jstack -F 3331`：正常输出不被响应时，使用该指令

`jstack -l 3331`：除堆栈外，显示关于锁的附件信息

Jstack -l PID >> 123.txt 在发生死锁时可以用jstack -l pid来观察锁持有情况 >>123.txt dump到指定文件



### [详细介绍](http://www.hollischuang.com/archives/110)

------

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## 常见问题定位过程



### 频繁GC问题或内存溢出问题

一、使用`jps`查看线程ID

二、使用`jstat -gc 3331 250 20` 查看gc情况，一般比较关注PERM区的情况，查看GC的增长情况。

三、使用`jstat -gccause`：额外输出上次GC原因

四、使用`jmap -dump:format=b,file=heapDump 3331`生成堆转储文件

五、使用jhat或者可视化工具（Eclipse Memory Analyzer 、IBM HeapAnalyzer）分析堆情况。

六、结合代码解决内存溢出或泄露问题。

------



### 死锁问题

一、使用`jps`查看线程ID

二、使用`jstack 3331`：查看线程情况

[回到顶部](https://www.cnblogs.com/java-chen-hao/p/10654071.html#_labelTop)

## 结语

经常使用适当的虚拟机监控和分析工具可以加快我们分析数据、定位解决问题的速度，但也要知道，工具永远都是知识技能的一层包装，没有什么工具是包治百病的。

# Java GC性能优化实战

## GC优化是必要的吗？

或者更准确地说，GC优化对Java基础服务来说是必要的吗？答案是否定的，事实上GC优化对Java基础服务来说在有些场合是可以省去的，但前提是这些正在运行的Java系统，必须包含以下参数或行为： 
\+ 内存大小已经通过-Xms和-Xmx参数指定过 
\+ 运行在server模式下（使用-server参数） 
\+ 系统中没有残留超时日志之类的错误日志

换句话说，如果你在运行时没有手动设置内存大小并且打印出了过多的超时日志，那你就需要对系统进行GC优化。

不过你需要时刻谨记一句话：GC tuning is the last task to be done.

现在来想一想GC优化的最根本原因，垃圾收集器的工作就是清除Java创建的对象，垃圾收集器需要清理的对象数量以及要执行的GC数量均取决于已创建的对象数量。因此，为了使你的系统在GC上表现良好，首先需要减少创建对象的数量。

俗话说“冰冻三尺非一日之寒”，我们在编码时要首先要把下面这些小细节做好，否则一些琐碎的不良代码累积起来将让GC的工作变得繁重而难于管理：

- 使用`StringBuilder`或`StringBuffer`来代替`String`
- 尽量少输出日志

尽管如此，仍然会有我们束手无策的情况。XML和JSON解析过程往往占用了最多的内存，即使我们已经尽可能地少用String、少输出日志，仍然会有大量的临时内存（大约10-100MB）被用来解析XML或JSON文件，但我们又很难弃用XML和JSON。在此，你只需要知道这一过程会占据大量内存即可。

如果在经过几次重复的优化后应用程序的内存用量情况有所改善，那么久可以启动GC优化了。

笔者总结了GC优化的两个目的： 
\1. 将进入老年代的对象数量降到最低 
\2. 减少Full GC的执行时间



## 将进入老年代的对象数量降到最低

除了可以在JDK 7及更高版本中使用的G1收集器以外，其他分代GC都是由Oracle JVM提供的。关于分代GC，就是对象在Eden区被创建，随后被转移到Survivor区，在此之后剩余的对象会被转入老年代。也有一些对象由于占用内存过大，在Eden区被创建后会直接被传入老年代。老年代GC相对来说会比新生代GC更耗时，因此，减少进入老年代的对象数量可以显著降低Full GC的频率。你可能会以为减少进入老年代的对象数量意味着把它们留在新生代，事实正好相反，新生代内存的大小是可以调节的。



## 降低Full GC的时间

Full GC的执行时间比Minor GC要长很多，因此，如果在Full GC上花费过多的时间（超过1s），将可能出现超时错误。 
\+ 如果通过减小老年代内存来减少Full GC时间，可能会引起`OutOfMemoryError`或者导致Full GC的频率升高。 
\+ 另外，如果通过增加老年代内存来降低Full GC的频率，Full GC的时间可能因此增加。

因此，你需要把老年代的大小设置成一个“合适”的值。



## 影响GC性能的参数

不要幻想着“如果有人用他设置的GC参数获取了不错的性能，我们为什么不复制他的参数设置呢？”，因为对于不用的Web服务，它们创建的对象大小和生命周期都不相同。

举一个简单的例子，如果一个任务的执行条件是A，B，C，D和E，另一个完全相同的任务执行条件只有A和B，那么哪一个任务执行速度更快呢？作为常识来讲，答案很明显是后者。

Java GC参数的设置也是这个道理，设置好几个参数并不会提升GC执行的速度，反而会使它变得更慢。GC优化的基本原则是将不同的GC参数应用到两个及以上的服务器上然后比较它们的性能，然后将那些被证明可以提高性能或减少GC执行时间的参数应用于最终的工作服务器上。

下面这张表展示了与内存大小相关且会影响GC性能的GC参数

 

表1：GC优化需要考虑的JVM参数

 

| 类型           | 参数                | 描述                       |
| -------------- | ------------------- | -------------------------- |
| 堆内存大小     | `-Xms`              | 启动JVM时堆内存的大小      |
|                | `-Xmx`              | 堆内存最大限制             |
| 新生代空间大小 | `-XX:NewRatio`      | 新生代和老年代的内存比     |
|                | `-XX:NewSize`       | 新生代内存大小             |
|                | `-XX:SurvivorRatio` | Eden区和Survivor区的内存比 |

笔者在进行GC优化时最常用的参数是`-Xms`,`-Xmx`和`-XX:NewRatio`。`-Xms`和`-Xmx`参数通常是必须的，所以`NewRatio`的值将对GC性能产生重要的影响。

有些人可能会问如何设置永久代内存大小，你可以用`-XX:PermSize`和`-XX:MaxPermSize`参数来进行设置，但是要记住，只有当出现`OutOfMemoryError`错误时你才需要去设置永久代内存。

还有一个会影响GC性能的因素是[垃圾收集器的类型](https://crowhawk.github.io/2017/08/15/jvm_3/),下表展示了关于GC类型的可选参数（基于JDK 6.0）：

 

表2：GC类型可选参数

 

| GC类型                 | 参数                                                         | 备注                            |
| ---------------------- | ------------------------------------------------------------ | ------------------------------- |
| Serial GC              | -XX:+UseSerialGC                                             |                                 |
| Parallel GC            | -XX:+UseParallelGC-XX:ParallelGCThreads=value                |                                 |
| Parallel Compacting GC | -XX:+UseParallelOldGC                                        |                                 |
| CMS GC                 | -XX:+UseConcMarkSweepGC-XX:+UseParNewGC-XX:+CMSParallelRemarkEnabled-XX:CMSInitiatingOccupancyFraction=value-XX:+UseCMSInitiatingOccupancyOnly |                                 |
| G1                     | -XX:+UnlockExperimentalVMOptions-XX:+UseG1GC                 | 在JDK 6中这两个参数必须配合使用 |

除了G1收集器外，可以通过设置上表中每种类型第一行的参数来切换GC类型，最常见的非侵入式GC就是Serial GC，它针对客户端系统进行了特别的优化。

会影响GC性能的参数还有很多，但是上述的参数会带来最显著的效果，请切记，设置太多的参数并不一定会提升GC的性能。



## GC优化的过程

GC优化的过程和大多数常见的提升性能的过程相似，下面是笔者使用的流程：



### 1.监控GC状态

你需要监控GC从而检查系统中运行的GC的各种状态



### 2.分析监控结果后决定是否需要优化GC

在检查GC状态后，你需要分析监控结构并决定是否需要进行GC优化。如果分析结果显示运行GC的时间只有0.1-0.3秒，那么就不需要把时间浪费在GC优化上，但如果运行GC的时间达到1-3秒，甚至大于10秒，那么GC优化将是很有必要的。

但是，如果你已经分配了大约10GB内存给Java，并且这些内存无法省下，那么就无法进行GC优化了。在进行GC优化之前，你需要考虑为什么你需要分配这么大的内存空间，如果你分配了1GB或2GB大小的内存并且出现了`OutOfMemoryError`，那你就应该执行堆转储（heap dump）来消除导致异常的原因。

> 注意：
>
> 堆转储（heap dump）是一个用来检查Java内存中的对象和数据的内存文件。该文件可以通过执行JDK中的`jmap`命令来创建。在创建文件的过程中，所有Java程序都将暂停，因此，不要再系统执行过程中创建该文件。
>
> 你可以在互联网上搜索heap dump的详细说明。



### 3.设置GC类型/内存大小

如果你决定要进行GC优化，那么你需要选择一个GC类型并且为它设置内存大小。此时如果你有多个服务器，请如上文提到的那样，在每台机器上设置不同的GC参数并分析它们的区别。



### 4.分析结果

在设置完GC参数后就可以开始收集数据，请在收集至少24小时后再进行结果分析。如果你足够幸运，你可能会找到系统的最佳GC参数。如若不然，你还需要分析输出日志并检查分配的内存，然后需要通过不断调整GC类型/内存大小来找到系统的最佳参数。



### 5.如果结果令人满意，将参数应用到所有服务器上并结束GC优化

如果GC优化的结果令人满意，就可以将参数应用到所有服务器上，并停止GC优化。

在下面的章节中，你将会看到上述每一步所做的具体工作。

## 监控GC状态并分析结果

在运行中的Web应用服务器（Web Application Server,WAS）上查看GC状态的最佳方式就是使用`jstat`命令。

下面的例子展示了某个还没有执行GC优化的JVM的状态（虽然它并不是运行服务器）。

```mipsasm
$ jstat -gcutil 21719 1s
S0    S1    E    O    P    YGC    YGCT    FGC    FGCT GCT
48.66 0.00 48.10 49.70 77.45 3428 172.623 3 59.050 231.673
48.66 0.00 48.10 49.70 77.45 3428 172.623 3 59.050 231.673
```

我们先看一下YGC（从应用程序启动到采样时发生 Young GC 的次数）和YGCT（从应用程序启动到采样时 Young GC 所用的时间(秒)），计算YGCT/YGC会得出，平均每次新生代的GC耗时50ms，这是一个很小的数字，通过这个结果可以看出，我们大可不必关注新生代GC对GC性能的影响。

现在来看一下FGC（ 从应用程序启动到采样时发生 Full GC 的次数）和FGCT（从应用程序启动到采样时 Full GC 所用的时间(秒)），计算FGCT/FGC会得出，平均每次老年代的GC耗时19.68s。有可能是执行了三次Full GC，每次耗时19.68s，也有可能是有两次只花了1s,另一次花了58s。不管是哪一种情况，GC优化都是很有必要的。

使用`jstat`命令可以很容易地查看GC状态，但是分析GC的最佳方式是加上`-verbosegc`参数来生成日志。在之前的文章中笔者已经解释了如何分析这些日志。HPJMeter是笔者最喜欢的用于分析`-verbosegc`生成的日志的工具，它简单易用，使用HPJmeter可以很容易地查看GC执行时间以及GC发生频率。

此外，如果GC执行时间满足下列所有条件，就没有必要进行GC优化了： 
\+ Minor GC执行非常迅速（50ms以内） 
\+ Minor GC没有频繁执行（大约10s执行一次） 
\+ Full GC执行非常迅速（1s以内） 
\+ Full GC没有频繁执行（大约10min执行一次）

括号中的数字并不是绝对的，它们也随着服务的状态而变化。有些服务可能要求一次Full GC在0.9s以内，而有些则会放得更宽一些。因此，对于不同的服务，需要按照不同的标准考虑是否需要执行GC优化。

当检查GC状态时，不能只查看Minor GC和Full GC的时间，还必须要关注GC执行的次数。如果新生代空间太小，Minor GC将会非常频繁地执行（有时每秒会执行一次，甚至更多）。此外，传入老年代的对象数目会上升，从而导致Full GC的频率升高。因此，在执行`jstat`命令时，请使用`-gccapacity`参数来查看具体占用了多少空间。



## 设置GC类型/内存大小



### 设置GC类型

Oracle JVM有5种垃圾收集器，但是在JDK 7以前的版本中，你只能在Parallel GC, Parallel Compacting GC 和CMS GC之中选择，至于具体选择哪个，则没有具体的原则和规则。

既然这样的话，我们如何来选择GC呢？最好的方法是把三种都用上，但是有一点必须明确——CMS GC通常比其他并行（Parallel）GC都要快（这是因为CMS GC是并发的GC），如果确实如此，那只选择CMS GC就可以了，不过CMS GC也不总是更快，当出现concurrent mode failure时，CMS GC就会比并行GC更慢了。

Concurrent mode failure

现在让我们来深入地了解一下concurrent mode failure。

并行GC和CMS GC的最大区别是并行GC采用“标记-整理”(Mark-Compact)算法而CMS GC采用“标记-清除”(Mark-Sweep)算法,compact步骤就是通过移动内存来消除内存碎片，从而消除分配的内存之间的空白区域。

对于并行GC来说，无论何时执行Full GC，都会进行compact工作，这消耗了太多的时间。不过在执行完Full GC后，下次内存分配将会变得更快（因为直接顺序分配相邻的内存）。

相反，CMS GC没有compact的过程，因此CMS GC运行的速度更快。但是也是由于没有整理内存，在进行磁盘清理之前，内存中会有很多零碎的空白区域，这也导致没有足够的空间分配给大对象。例如，在老年代还有300MB可用空间，但是连一个10MB的对象都没有办法被顺序存储在老年代中，在这种情况下，会报出“concurrent mode failure”的warning，然后系统执行compact操作。但是CMS GC在这种情况下执行的compact操作耗时要比并行GC高很多，并且这还会导致另一个问题，关于“concurrent mode failure”的详细说明。

综上所述，你需要根据你的系统情况为其选择一个最适合的GC类型。

每个系统都有最适合它的GC类型等着你去寻找，如果你有6台服务器，我建议你每两个服务器设置相同的参数，然后加上`-verbosegc`参数再分析结果。



### 设置内存大小

下面展示了内存大小、GC运行次数和GC运行时间之间的关系：

大内存空间 
\+ 减少了GC的次数 
\+ 提高了GC的运行时间

小内存空间 
\+ 增多了GC的次数 
\+ 降低了GC的运行时间

关于如何设置内存的大小，没有一个标准答案，如果服务器资源充足并且Full GC能在1s内完成，把内存设为10GB也是可以的，但是大部分服务器并不处在这种状态中，当内存设为10GB时，Full GC会耗时10-30s,具体的时间自然与对象的大小有关。

既然如此，我们该如何设置内存大小呢？通常我推荐设为500MB，这不是说你要通过`-Xms500m`和`-Xmx500m`参数来设置WAS内存。根据GC优化之前的状态，如果Full GC后还剩余300MB的空间，那么把内存设为1GB是一个不错的选择（300MB（默认程序占用）+ 500MB（老年代最小空间）+200MB（空闲内存））。这意味着你需要为老年代设置至少500MB空间，因此如果你有三个运行服务器，可以把它们的内存分别设置为1GB，1.5GB，2GB，然后检查结果。

理论上来说，GC执行速度应该遵循1GB> 1.5GB> 2GB，1GB内存时GC执行速度最快。然而，理论上的1GB内存Full GC消耗1s、2GB内存Full GC消耗2 s在现实里是无法保证的，实际的运行时间还依赖于服务器的性能和对象大小。因此，最好的方法是创建尽可能多的测量数据并监控它们。

在设置内存空间大小时，你还需要设置一个参数：`NewRatio`。`NewRatio`的值是新生代和老年代空间大小的比例。如果`XX:NewRatio=1`，则新生代空间:老年代空间=1:1，如果堆内存为1GB，则新生代:老年代=500MB:500MB。如果`NewRatio`等于2，则新生代:老年代=1:2，因此，`NewRatio`的值设置得越大，则老年代空间越大，新生代空间越小。

你可能会认为把`NewRatio`设为1会是最好的选择，然而事实并非如此，根据笔者的经验，当`NewRatio`设为2或3时，整个GC的状态表现得更好。

完成GC优化最快地方法是什么？答案是比较性能测试的结果。为了给每台服务器设置不同的参数并监控它们，最好查看的是一或两天后的数据。当通过性能测试来进行GC优化时，你需要在不同的测试时保证它们有相同的负载和运行环境。然而，即使是专业的性能测试人员，想精确地控制负载也很困难，并且需要大量的时间准备。因此，更加方便容易的方式是直接设置参数来运行，然后等待运行的结果（即使这需要消耗更多的时间）。



## 分析GC优化的结果

在设置了GC参数和`-verbosegc`参数后，可以使用tail命令确保日志被正确地生成。如果参数设置得不正确或日志未生成，那你的时间就被白白浪费了。如果日志收集没有问题的话，在收集一或两天数据后再检查结果。最简单的方法是把日志从服务器移到你的本地PC上，然后用HPJMeter分析数据。

在分析结果时，请关注下列几点（这个优先级是笔者根据自己的经验拟定的，我认为选取GC参数时应考虑的最重要的因素是Full GC的运行时间。）： 
\+ 单次Full GC运行时间 
\+ 单次Minor GC运行时间 
\+ Full GC运行间隔 
\+ Minor GC运行间隔 
\+ 整个Full GC的时间 
\+ 整个Minor GC的运行时间 
\+ 整个GC的运行时间 
\+ Full GC的执行次数 
\+ Minor GC的执行次数

找到最佳的GC参数是件非常幸运的，然而在大多数时候，我们并不会如此幸运，在进行GC优化时一定要小心谨慎，因为当你试图一次完成所有的优化工作时，可能会出现`OutOfMemoryError`错误。



## 优化案例

到目前为止，我们一直在从理论上介绍GC优化，现在是时候将这些理论付诸实践了，我们将通过几个例子来更深入地理解GC优化。



### 示例1

下面这个例子是针对Service S的优化，对于最近刚开发出来的Service S，执行Full GC需要消耗过多的时间。

现在看一下执行`jstat -gcutil`的结果

```mipsasm
S0 S1 E O P YGC YGCT FGC FGCT GCT
12.16 0.00 5.18 63.78 20.32 54 2.047 5 6.946 8.993
```

左边的Perm区的值对于最初的GC优化并不重要，而YGC参数的值更加对于这次优化更为重要。

平均执行一次Minor GC和Full GC消耗的时间如下表所示：

 

表3：Service S的Minor GC 和Full GC的平均执行时间

 

| GC类型   | GC执行次数 | GC执行时间 | 平均值 |
| -------- | ---------- | ---------- | ------ |
| Minor GC | 54         | 2.047s     | 37ms   |
| Full GC  | 5          | 6.946s     | 1.389s |

37ms对于Minor GC来说还不赖，但1.389s对于Full GC来说意味着当GC发生在数据库Timeout设置为1s的系统中时，可能会频繁出现超时现象。

首先，你需要检查开始GC优化前内存的使用情况。使用`jstat -gccapacity`命令可以检查内存用量情况。在笔者的服务器上查看到的结果如下：

```armasm
NGCMN NGCMX NGC S0C S1C EC OGCMN OGCMX OGC OC PGCMN PGCMX PGC PC YGC FGC
212992.0 212992.0 212992.0 21248.0 21248.0 170496.0 1884160.0 1884160.0 1884160.0 1884160.0 262144.0 262144.0 262144.0 262144.0 54 5
```

其中的关键值如下： 
\+ 新生代内存用量：212,992 KB 
\+ 老年代内存用量：1,884,160 KB

因此，除了永久代以外，被分配的内存空间加起来有2GB，并且新生代：老年代=1：9，为了得到比使用`jstat`更细致的结果，还需加上`-verbosegc`参数获取日志，并把三台服务器按照如下方式设置（除此以外没有使用任何其他参数）： 
\+ NewRatio=2 
\+ NewRatio=3 
\+ NewRatio=4

一天后我得到了系统的GC log，幸运的是，在设置完NewRatio后系统没有发生任何Full GC。

这是为什么呢？这是因为大部分对象在创建后很快就被回收了，所有这些对象没有被传入老年代，而是在新生代就被销毁回收了。

在这样的情况下，就没有必要去改变其他的参数值了，只要选择一个最合适的`NewRatio`值即可。那么，如何确定最佳的NewRatio值呢？为此，我们分析一下每种`NewRatio`值下Minor GC的平均响应时间。

在每种参数下Minor GC的平均响应时间如下： 
\+ NewRatio=2：45ms 
\+ NewRatio=3：34ms 
\+ NewRatio=4：30ms

我们可以根据GC时间的长短得出NewRatio=4是最佳的参数值（尽管NewRatio=4时新生代空间是最小的）。在设置完GC参数后，服务器没有发生Full GC。

为了说明这个问题，下面是服务执行一段时间后执行`jstat –gcutil`的结果:

```mipsasm
S0 S1 E O P YGC YGCT FGC FGCT GCT
8.61 0.00 30.67 24.62 22.38 2424 30.219 0 0.000 30.219
```

你可能会认为是服务器接收的请求少才使得GC发生的频率较低，实际上，虽然Full GC没有执行过，但Minor GC被执行了2424次。



### 示例2

这是一个Service A的例子。我们通过公司内部的应用性能管理系统（APM）发现JVM暂停了相当长的时间（超过8秒），因此我们进行了GC优化。我们努力寻找JVM暂停的原因，后来发现是因为Full GC执行时间过长，因此我们决定进行GC优化。

在GC优化的开始阶段，我们加上了`-verbosegc`参数，结果如下图所示：

 

![img](https://pic.yupoo.com/crowhawk/ebb4b181/a24f4e9b.png)

 

 

图1：进行GC优化之前STW的时间

 

上图是由HPJMeter生成的图片之一。横坐标表示JVM执行的时间，纵坐标表示每次GC的时间。CMS为绿点，表示Full GC的结果，而Parallel Scavenge为蓝点，表示Minor GC的结果。

之前我说过CMS GC是最快的GC，但是上面的结果显示在一些时候CMS耗时达到了15s。是什么导致了这一结果？请记住我之前说的：CMS在执行compact（整理）操作时会显著变慢。此外，服务的内存通过`-Xms1g`和`=Xmx4g`设置了，而分配的内存只有4GB。

因此笔者将GC类型从CMS GC改为了Parallel GC，把内存大小设为2GB，并把`NewRatio`设为3。在执行`jstat -gcutil`几小时后的结果如下：

```mipsasm
S0 S1 E O P YGC YGCT FGC FGCT GCT
0.00 30.48 3.31 26.54 37.01 226 11.131 4 11.758 22.890
```

Full GC的时间缩短了，变成了每次3s，跟15s比有了显著提升。但是3s依然不够快，为此笔者创建了以下6种情况： 
\+ Case 1: `-XX:+UseParallelGC -Xms1536m -Xmx1536m -XX:NewRatio=2` 
\+ Case 2: `-XX:+UseParallelGC -Xms1536m -Xmx1536m -XX:NewRatio=3` 
\+ Case 3: `-XX:+UseParallelGC -Xms1g -Xmx1g -XX:NewRatio=3` 
\+ Case 4: `-XX:+UseParallelOldGC -Xms1536m -Xmx1536m -XX:NewRatio=2` 
\+ Case 5: `-XX:+UseParallelOldGC -Xms1536m -Xmx1536m -XX:NewRatio=3` 
\+ Case 6: `-XX:+UseParallelOldGC -Xms1g -Xmx1g -XX:NewRatio=3`

上面哪一种情况最快？结果显示，内存空间越小，运行结果最少。下图展示了性能最好的Case 6的结果图，它的最慢响应时间只有1.7s，并且响应时间的平均值已经被控制到了1s以内。

 

![img](https://pic.yupoo.com/crowhawk/026cb5ec/dd3bdbb9.png)

 

 

图2：Case 6的持续时间图

 

基于上图的结果，按照Case 6调整了GC参数，但这却导致每晚都会发生`OutOfMemoryError`。很难解释发生异常的具体原因，简单地说，应该是批处理程序导致了内存泄漏，我们正在解决相关的问题。

如果只对GC日志做一些短时间的分析就将相关参数部署到所有服务器上来执行GC优化，这将是非常危险的。切记，只有当你同时仔细分析服务的执行情况和GC日志后，才能保证GC优化没有错误地执行。

在上文中，我们通过两个GC优化的例子来说明了GC优化是怎样执行的。正如上文中提到的，例子中设置的GC参数可以设置在相同的服务器之上，但前提是他们具有相同的CPU、操作系统、JDK版本并且运行着相同的服务。此外，不要把我使用的参数照搬到你的应用上，它们可能在你的机器上并不能起到同样良好的效果。

## 总结

笔者没有执行heap dump并分析内存的详细内容，而是通过自己的经验进行GC优化。精确地分析内存可以得到更好的优化效果，不过这种分析一般只适用于内存使用量相对固定的场景。如果服务严重过载并占有了大量的内存，则建议你根据之前的经验进行GC优化。

笔者已经在一些服务上设置了G1 GC参数并进行了性能测试，但还没有应用于正式的生产环境。G1 GC的速度快于任何其他的GC类型，但是你必须要升级到JDK 7。此外，暂时还无法保证它的稳定性，没有人知道运行时是否会出现致命的错误，因此G1 
GC暂时还不适合投入应用。

等未来JDK 7真正稳定了（这并不是说它现在不稳定），并且WAS针对JDK 7进行优化后，G1 GC最终能按照预期的那样来工作，等到那一天我们可能就不再需要GC优化了。

# GC调优工具实战

## jps

用于显示当前用户下的所有java进程信息：

```shell
# jps [options] [hostid] 
# q:仅输出VM标识符, m: 输出main method的参数,l:输出完全的包名, v:输出jvm参数
[root@localhost ~]# jps -l
28729 sun.tools.jps.Jps
23789 cn.ms.test.DemoMain
23651 cn.ms.test.TestMain
```



## jstat

用于监视虚拟机运行时状态信息（类装载、内存、垃圾收集、JIT编译等运行数据）：

**-gc**：垃圾回收统计（大小）

```shell
# 每隔2000ms输出<pid>进程的gc情况，一共输出2次
[root@localhost ~]# jstat -gc <pid> 2000 2
# 每隔2s输出<pid>进程的gc情况，每个3条记录就打印隐藏列标题
[root@localhost ~]# jstat -gc -t -h3 <pid> 2s
Timestamp        S0C    S1C    S0U    S1U    ... YGC     YGCT    FGC    FGCT     GCT   
         1021.6 1024.0 1024.0  0.0   1024.0  ...  1    0.012   0      0.000    0.012
         1023.7 1024.0 1024.0  0.0   1024.0  ...  1    0.012   0      0.000    0.012
         1025.7 1024.0 1024.0  0.0   1024.0  ...  1    0.012   0      0.000    0.012
Timestamp        S0C    S1C    S0U    S1U    ... YGC     YGCT    FGC    FGCT     GCT   
         1027.7 1024.0 1024.0  0.0   1024.0  ...  1    0.012   0      0.000    0.012
         1029.7 1024.0 1024.0  0.0   1024.0  ...  1    0.012   0      0.000    0.012
# 结果说明: C即Capacity 总容量，U即Used 已使用的容量
##########################
# S0C：年轻代中第一个survivor（幸存区）的容量 (kb)
# S1C：年轻代中第二个survivor（幸存区）的容量 (kb)
# S0U：年轻代中第一个survivor（幸存区）目前已使用空间 (kb)
# S1U：年轻代中第二个survivor（幸存区）目前已使用空间 (kb)
# EC：年轻代中Eden（伊甸园）的容量 (kb)
# EU：年轻代中Eden（伊甸园）目前已使用空间 (kb)
# OC：Old代的容量 (kb)
# OU：Old代目前已使用空间 (kb)
# PC：Perm(持久代)的容量 (kb)
# PU：Perm(持久代)目前已使用空间 (kb)
# YGC：从应用程序启动到采样时年轻代中gc次数
# YGCT：从应用程序启动到采样时年轻代中gc所用时间(s)
# FGC：从应用程序启动到采样时old代(全gc)gc次数
# FGCT：从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT：从应用程序启动到采样时gc用的总时间(s)
```

**-gcutil**：垃圾回收统计（百分比）

```shell
[root@localhost bin]# jstat -gcutil <pid>
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  99.80  16.21  26.18  93.34  90.74      9    0.056     2    0.045    0.102
# 结果说明
##########################
# S0：年轻代中第一个survivor（幸存区）已使用的占当前容量百分比
# S1：年轻代中第二个survivor（幸存区）已使用的占当前容量百分比
# E：年轻代中Eden（伊甸园）已使用的占当前容量百分比
# O：old代已使用的占当前容量百分比
# P：perm代已使用的占当前容量百分比
# YGC：从应用程序启动到采样时年轻代中gc次数
# YGCT：从应用程序启动到采样时年轻代中gc所用时间(s)
# FGC：从应用程序启动到采样时old代(全gc)gc次数
# FGCT：从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT：从应用程序启动到采样时gc用的总时间(s)
```

**-gccapacity**：堆内存统计

```shell
[root@localhost ~]# jstat -gccapacity <pid>
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC      PGCMN    PGCMX     PGC       PC     YGC    FGC 
 84480.0 1349632.0 913408.0 54272.0 51200.0 502784.0   168448.0  2699264.0   168448.0   168448.0  21504.0  83968.0  51712.0  51712.0      9     0
# 结果说明
##########################
# NGCMN：年轻代(young)中初始化(最小)的大小 (kb)
# NGCMX：年轻代(young)的最大容量 (kb)
# NGC：年轻代(young)中当前的容量 (kb)
# S0C：年轻代中第一个survivor（幸存区）的容量 (kb)
# S1C：年轻代中第二个survivor（幸存区）的容量 (kb)
# EC：年轻代中Eden（伊甸园）的容量 (kb)
# OGCMN：old代中初始化(最小)的大小 (kb)
# OGCMX：old代的最大容量 (kb)
# OGC：old代当前新生成的容量 (kb)
# OC：Old代的容量 (kb)
# PGCMN：perm代中初始化(最小)的大小 (kb)
# PGCMX：perm代的最大容量 (kb)
# PGC：perm代当前新生成的容量 (kb)
# PC：Perm(持久代)的容量 (kb)
# YGC：从应用程序启动到采样时年轻代中gc次数
# GCT：从应用程序启动到采样时gc用的总时间(s)
```

**-gccause**：垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因

```shell
[root@localhost ~]# jstat -gccause <pid>
  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC       
  0.00  79.23  39.37  39.92  99.74      9    0.198     0    0.000    0.198 Allocation Failure   No GC
# 结果说明
##########################
# LGCC：最近垃圾回收的原因
# GCC：当前垃圾回收的原因
```



## jstack

jstack(Java Stack Trace)主要用于打印线程的堆栈信息，是JDK自带的很强大的线程分析工具，可以帮助我们排查程序运行时的线程状态、死锁状态等。

```shell
# dump出进程<pid>的线程堆栈快照至/data/1.log文件中
jstack -l <pid> >/data/1.log

# 参数说明：
# -F：如果正常执行jstack命令没有响应（比如进程hung住了），可以加上此参数强制执行thread dump
# -m：除了打印Java的方法调用栈之外，还会输出native方法的栈帧
# -l：打印与锁有关的附加信息。使用此参数会导致JVM停止时间变长，在生产环境需慎用
```

jstack dump文件中值得关注的线程状态有：

- **死锁（Deadlock） —— 重点关注**
- 执行中（Runnable）  
- **等待资源（Waiting on condition） —— 重点关注**
  - 等待某个资源或条件发生来唤醒自己。具体需结合jstacktrace来分析，如线程正在sleep，网络读写繁忙而等待
  - 如果大量线程在“waiting on condition”，并且在等待网络资源，可能是网络瓶颈的征兆
- **等待获取监视器（Waiting on monitor entry） —— 重点关注**
  - 如果大量线程在“waiting for monitor entry”，可能是一个全局锁阻塞住了大量线程
- 暂停（Suspended）
- 对象等待中（Object.wait() 或 TIMED_WAITING）
- **阻塞（Blocked） —— 重点关注**
- 停止（Parked）



**注意**：如果某个相同的call stack经常出现， 我们有80%的以上的理由确定这个代码存在性能问题（读网络的部分除外）。



**场景一：分析BLOCKED问题**

```shell
"RMI TCP Connection(267865)-172.16.5.25" daemon prio=10 tid=0x00007fd508371000 nid=0x55ae waiting for monitor entry [0x00007fd4f8684000]
   java.lang.Thread.State: BLOCKED (on object monitor)
at org.apache.log4j.Category.callAppenders(Category.java:201)
- waiting to lock <0x00000000acf4d0c0> (a org.apache.log4j.Logger)
at org.apache.log4j.Category.forcedLog(Category.java:388)
at org.apache.log4j.Category.log(Category.java:853)
at org.apache.commons.logging.impl.Log4JLogger.warn(Log4JLogger.java:234)
at com.tuan.core.common.lang.cache.remote.SpyMemcachedClient.get(SpyMemcachedClient.java:110)
```

- 线程状态是 Blocked，阻塞状态。说明线程等待资源超时
- “ waiting to lock <0x00000000acf4d0c0>”指，线程在等待给这个 0x00000000acf4d0c0 地址上锁（英文可描述为：trying to obtain  0x00000000acf4d0c0 lock）
- 在 dump 日志里查找字符串 0x00000000acf4d0c0，发现有大量线程都在等待给这个地址上锁。如果能在日志里找到谁获得了这个锁（如locked < 0x00000000acf4d0c0 >），就可以顺藤摸瓜了
- “waiting for monitor entry”说明此线程通过 synchronized(obj) {……} 申请进入了临界区，从而进入了下图1中的“Entry Set”队列，但该 obj 对应的 monitor 被其他线程拥有，所以本线程在 Entry Set 队列中等待
- 第一行里，"RMI TCP Connection(267865)-172.16.5.25"是 Thread Name 。tid指Java Thread id。nid指native线程的id。prio是线程优先级。[0x00007fd4f8684000]是线程栈起始地址。



**场景二：分析CPU过高问题**

1.top命令找出最高占用的进程（Shift+P）

2.查看高负载进程下的高负载线程（top -Hp <PID>或ps -mp <PID> -o THREAD,tid,time）

3.找出最高占用的线程并记录thread_id，把线程号进行换算成16进制编号（printf "%X\n" thread_id）

4.（可选）执行查看高负载的线程名称（jstack 16143 | grep 3fb6）

5.导出进程的堆栈日志，找到3fb6 这个线程号（jstack 16143 >/home/16143.log）

6.根据找到的堆栈信息关联到代码进行定位分析即可



## jmap

jmap(Java Memory Map)主要用于打印内存映射。常用命令：

`jmap -dump:live,format=b,file=xxx.hprof <pid> `

**查看JVM堆栈的使用情况**

```powershell
[root@localhost ~]# jmap -heap 7243
Attaching to process ID 27900, please wait...
Debugger attached successfully.
Client compiler detected.
JVM version is 20.45-b01
using thread-local object allocation.
Mark Sweep Compact GC
Heap Configuration: #堆内存初始化配置
   MinHeapFreeRatio = 40     #-XX:MinHeapFreeRatio设置JVM堆最小空闲比率  
   MaxHeapFreeRatio = 70   #-XX:MaxHeapFreeRatio设置JVM堆最大空闲比率  
   MaxHeapSize = 100663296 (96.0MB)   #-XX:MaxHeapSize=设置JVM堆的最大大小
   NewSize = 1048576 (1.0MB)     #-XX:NewSize=设置JVM堆的‘新生代’的默认大小
   MaxNewSize = 4294901760 (4095.9375MB) #-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
   OldSize = 4194304 (4.0MB)  #-XX:OldSize=设置JVM堆的‘老生代’的大小
   NewRatio = 2    #-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
   SurvivorRatio = 8  #-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值
   PermSize = 12582912 (12.0MB) #-XX:PermSize=<value>:设置JVM堆的‘持久代’的初始大小  
   MaxPermSize = 67108864 (64.0MB) #-XX:MaxPermSize=<value>:设置JVM堆的‘持久代’的最大大小  
Heap Usage:
New Generation (Eden + 1 Survivor Space): #新生代区内存分布，包含伊甸园区+1个Survivor区
   capacity = 30212096 (28.8125MB)
   used = 27103784 (25.848182678222656MB)
   free = 3108312 (2.9643173217773438MB)
   89.71169693092462% used
Eden Space: #Eden区内存分布
   capacity = 26869760 (25.625MB)
   used = 26869760 (25.625MB)
   free = 0 (0.0MB)
   100.0% used
From Space: #其中一个Survivor区的内存分布
   capacity = 3342336 (3.1875MB)
   used = 234024 (0.22318267822265625MB)
   free = 3108312 (2.9643173217773438MB)
   7.001809512867647% used
To Space: #另一个Survivor区的内存分布
   capacity = 3342336 (3.1875MB)
   used = 0 (0.0MB)
   free = 3342336 (3.1875MB)
   0.0% used
PS Old Generation: #当前的Old区内存分布
   capacity = 67108864 (64.0MB)
   used = 67108816 (63.99995422363281MB)
   free = 48 (4.57763671875E-5MB)
   99.99992847442627% used
PS Perm Generation: #当前的 “持久代” 内存分布
   capacity = 14417920 (13.75MB)
   used = 14339216 (13.674942016601562MB)
   free = 78704 (0.0750579833984375MB)
   99.45412375710227% used
```

新生代内存回收就是采用空间换时间方式；如果from区使用率一直是100% 说明程序创建大量的短生命周期的实例，使用jstat统计jvm在内存回收中发生的频率耗时以及是否有full gc，使用这个数据来评估一内存配置参数、gc参数是否合理。

**统计一【jmap -histo】**：统计所有类的实例数量和所占用的内存容量

```powershell
[root@localhost ~]# jmap -histo 7243
 num     #instances         #bytes  class name
----------------------------------------------
   1:          8969       19781168  [B
   2:          1835        2296720  [I
   3:         19735        2050688  [C
   4:          3448         385608  java.lang.Class
   5:          3829         371456  [Ljava.lang.Object;
   6:         14634         351216  java.lang.String
   7:          6695         214240  java.util.concurrent.ConcurrentHashMap$Node
   8:          6257         100112  java.lang.Object
   9:          2155          68960  java.util.HashMap$Node
  10:           723          63624  java.lang.reflect.Method
  11:            49          56368  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  12:           830          46480  java.util.zip.ZipFile$ZipFileInputStream
  13:          1146          45840  java.lang.ref.Finalizer
  ......
```

**统计二【jmap -histo】**：查看对象数最多的对象，并过滤Map关键词，然后按降序排序输出

```shell
[root@localhost ~]# jmap -histo 7243 |grep Map|sort -k 2 -g -r|less
Total         96237       26875560
   7:          6695         214240  java.util.concurrent.ConcurrentHashMap$Node
   9:          2155          68960  java.util.HashMap$Node
  18:           563          27024  java.util.HashMap
  21:           505          20200  java.util.LinkedHashMap$Entry
  16:           337          34880  [Ljava.util.HashMap$Node;
  27:           336          16128  gnu.trove.THashMap
  56:           163           6520  java.util.WeakHashMap$Entry
  60:           127           6096  java.util.WeakHashMap
  38:           127          10144  [Ljava.util.WeakHashMap$Entry;
  53:           126           7056  java.util.LinkedHashMap
......
```



**统计三【jmap -histo】**：统计实例数量最多的前10个类

```shell
[root@localhost ~]# jmap -histo 7243 | sort -n -r -k 2 | head -10
 num     #instances         #bytes  class name
----------------------------------------------
Total         96237       26875560
   3:         19735        2050688  [C
   6:         14634         351216  java.lang.String
   1:          8969       19781168  [B
   7:          6695         214240  java.util.concurrent.ConcurrentHashMap$Node
   8:          6257         100112  java.lang.Object
   5:          3829         371456  [Ljava.lang.Object;
   4:          3448         385608  java.lang.Class
   9:          2155          68960  java.util.HashMap$Node
   2:          1835        2296720  [I
```



**统计四【jmap -histo】**：统计合计容量最多的前10个类

```shell
[root@localhost ~]# jmap -histo 7243 | sort -n -r -k 3 | head -10
 num     #instances         #bytes  class name
----------------------------------------------
Total         96237       26875560
   1:          8969       19781168  [B
   2:          1835        2296720  [I
   3:         19735        2050688  [C
   4:          3448         385608  java.lang.Class
   5:          3829         371456  [Ljava.lang.Object;
   6:         14634         351216  java.lang.String
   7:          6695         214240  java.util.concurrent.ConcurrentHashMap$Node
   8:          6257         100112  java.lang.Object
   9:          2155          68960  java.util.HashMap$Node
```

**dump注意事项**

- 在应用快要发生FGC的时候把堆数据导出来

  ​	老年代或新生代used接近100%时，就表示即将发生GC，也可以再JVM参数中指定触发GC的阈值。

  - 查看快要发生FGC使用命令：jmap -heap < pid >
  - 数据导出：jmap -dump:format=b,file=heap.bin < pid >

- 通过命令查看大对象：jmap -histo < pid >|less



**使用总结**

- 如果程序内存不足或者频繁GC，很有可能存在内存泄露情况，这时候就要借助Java堆Dump查看对象的情况
- 要制作堆Dump可以直接使用jvm自带的jmap命令
- 可以先使用`jmap -heap`命令查看堆的使用情况，看一下各个堆空间的占用情况
- 使用`jmap -histo:[live]`查看堆内存中的对象的情况。如果有大量对象在持续被引用，并没有被释放掉，那就产生了内存泄露，就要结合代码，把不用的对象释放掉
- 也可以使用 `jmap -dump:format=b,file=<fileName>`命令将堆信息保存到一个文件中，再借助jhat命令查看详细内容
- 在内存出现泄露、溢出或者其它前提条件下，建议多dump几次内存，把内存文件进行编号归档，便于后续内存整理分析
- 在用cms gc的情况下，执行jmap -heap有些时候会导致进程变T，因此强烈建议别执行这个命令，如果想获取内存目前每个区域的使用状况，可通过jstat -gc或jstat -gccapacity来拿到





## jhat

jhat（JVM Heap Analysis Tool）命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看。在此要注意，一般不会直接在服务器上进行分析，因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。

```powershell
# 解析Java堆转储文件,并启动一个 web server
jhat heapDump.dump
```



## jconsole

jconsole(Java Monitoring and Management Console)是一个Java GUI监视工具，可以以图表化的形式显示各种数据，并可通过远程连接监视远程的服务器VM。用java写的GUI程序，用来监控VM，并可监控远程的VM，非常易用，而且功能非常强。命令行里打jconsole，选则进程就可以了。

**第一步**：在远程机的tomcat的catalina.sh中加入配置：

```powershell
JAVA_OPTS="$JAVA_OPTS -Djava.rmi.server.hostname=192.168.202.121 -Dcom.sun.management.jmxremote"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.port=12345"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.authenticate=true"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.pwd.file=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.101-3.b13.el7_2.x86_64/jre/lib/management/jmxremote.password"
```



**第二步**：配置权限文件

```powershell
[root@localhost bin]# cd /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.101-3.b13.el7_2.x86_64/jre/lib/management/
[root@localhost management]# cp jmxremote.password.template jmxremote.password
[root@localhost management]# vi jmxremote.password
```

monitorRole QED
controlRole chenqimiao



**第三步**：配置权限文件为600

```powershell
[root@localhost management]# chmod 600 jmxremote.password jmxremote.access
```

这样基本配置就结束了，下面说两个坑，第一个就是防火墙的问题，要开放指定端口的防火墙，我这里配置的是12345端口，第二个是hostname的问题：

![jconsole-ip2](F:/lemon-guide-main/images/JVM/jconsole-ip2.png)

请将127.0.0.1修改为本地真实的IP,我的服务器IP是192.168.202.121：

![jconsole-ip](F:/lemon-guide-main/images/JVM/jconsole-ip.png)

 **第四步**：查看JConsole

![JConsole-新建连接](F:/lemon-guide-main/images/JVM/JConsole-新建连接.png)

![JConsole-Console](F:/lemon-guide-main/images/JVM/JConsole-Console.png)



## jvisualvm

jvisualvm(JVM Monitoring/Troubleshooting/Profiling Tool)同jconsole都是一个基于图形化界面的、可以查看本地及远程的JAVA GUI监控工具，Jvisualvm同jconsole的使用方式一样，直接在命令行打入Jvisualvm即可启动，不过Jvisualvm相比，界面更美观一些，数据更实时。 jvisualvm的使用VisualVM进行远程连接的配置和JConsole是一摸一样的，最终效果图如下

![jvisualvm](F:/lemon-guide-main/images/JVM/jvisualvm.png)



**Visual GC(监控垃圾回收器)**

Java VisualVM默认没有安装Visual GC插件，需要手动安装，JDK的安装目录的bin目露下双击 jvisualvm.sh，即可打开Java VisualVM，点击菜单栏： **工具->插件** 安装Visual GC，最终效果如下图所示：

![Visual-GC](F:/lemon-guide-main/images/JVM/Visual-GC.png)



**大dump文件**

从服务器dump堆内存后文件比较大（5.5G左右），加载文件、查看实例对象都很慢，还提示配置xmx大小。表明给VisualVM分配的堆内存不够，找到$JAVA_HOME/lib/visualvm}/etc/visualvm.conf这个文件，修改：

```shell
default_options="-J-Xms24m -J-Xmx192m"
```

再重启VisualVM就行了。



## jmc

jmc（Java Mission Control）是JDK自带的一个图形界面监控工具，监控信息非常全面。JMC打开性能日志后，主要包括**一般信息、内存、代码、线程、I/O、系统、事件** 功能。

![jmc-main](F:/lemon-guide-main/images/JVM/jmc-main.jpg)

JMC的最主要的特征就是JFR（Java Flight Recorder），是基于JAVA的飞行记录器，JFR的数据是一些列JVM事件的历史纪录，可以用来诊断JVM的性能和操作，收集后的数据可以使用JMC来分析。



### 启动JFR

在商业版本上面，JFR默认是关闭的，可以通过在启动时增加参数 `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder` 来启动应用。启动之后，也只是开启了JFR特性，但是还没有开始进行事件记录。这就要通过GUI和命令行了。

- **通过Java Mission Control启动JFR**

  打开Java Mission Control点击对应的JVM启动即可，事件记录有两种模式（如果选择第2种模式，那么JVM会使用一个循环使用的缓存来存放事件数据）：

  - 记录固定一段时间的事件（比如：1分钟）
  - 持续进行记录

- **通过命令行启动JFR**

  通过在启动的时候，增加参数：`-XX:+FlightRecorderOptions=string` 来启动真正地事件记录，这里的 `string` 可以是以下值（下列参数都可以使用jcmd命令，在JVM运行的时候进行动态调整，[参考地址](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)）：

  - `name=name`：标识recording的名字（一个进程可以有多个recording存在，它们使用名字来进行区分）
  - `defaultrecording=<ture|false>`：是否启动recording，默认是false，我们要分析，必须要设置为true
  - `setting=paths`：包含JFR配置的文件名字
  - `delay=time`：启动之后，经过多长时间（比如：30s，1h）开始进行recording
  - `duration=time`：做多长时间的recording
  - `filename=path`：recordding记录到那个文件里面
  - `compress=<ture|false>`：是否对recording进行压缩（gzip）,默认为false
  - `maxage=time`：在循环使用的缓存中，事件数据保存的最大时长
  - `maxsize=size`：事件数据缓存的最大大小（比如：1024k，1M）



### 常用JFR命令

- 启动recording

  命令格式：`jcmd process_id JFR.start [options_list]`，其中options_list就是上述的参数值。

- dump出循环缓存中的数据

  命令格式：`jcmd process_id JFR.dump [options_list]`，其中options_list参数的可选值如下：

  - `name=name`：recording的名字
  - `recording=n`：JFR recording的数字（一个标识recording的随机数）
  - `filename=path`：dump文件的保存路径

- 查看进程中所有recording

  命令格式：` jcmd process_id JFR.check [verbose]`，不同recording使用名字进行区分，同时JVM还为它分配一个随机数。

- 停止recording

  命令格式：` jcmd process_id JFR.stop [options_list]`，其中options_list参数的可选值如下：

  - `name=name`：要停止的recording名字
  - `recording=n`：要停止的recording的标识数字
  - `discard=boolean`：如果为true，数据被丢弃，而不是写入下面指定的文件当中
  - `filename=path`：写入数据的文件名称



### 命令启动JFR案例

- **第一步**：创建一个包含了你自己配置的JFR模板文件（`template.jfc`）。运行jmc, 然后Window->Flight Recording Template Manage菜单。准备好档案后，就可以导出文件，并移动到要排查问题的环境中

  ![jmc-jfc](F:/lemon-guide-main/images/JVM/jmc-jfc.png)

- **第二步**：由于JFR需要JDK的商业证书，这一步需要解锁jdk的商业特性

  ```powershell
  [root@localhost bin]# jcmd 12234 VM.unlock_commercial_features
  12234: Commercial Features already unlocked.
  ```

- **第三步**：最后你就可以启动JFR，命令格式如下：

  ```powershell
  jcmd <PID> JFR.start name=test duration=60s [settings=template.jfc] filename=output.jfr
  ```

  ​	上述命令立即启动JFR并开始使用 `template.jfc`（在 `$JAVA_HOME/jre/lib/jfr` 下有 `default.jfc` 和 `profile.jfc` 模板）的配置收集 `60s` 的JVM信息，输出到 `output.jfr` 中。一旦记录完成之后，就可以复制.jfr文件到你的工作环境使用jmc GUI来分析。它几乎包含了排查jvm问题需要的所有信息，包括堆dump时的异常信息。使用案例如下：

  ```powershell
  [root@localhost bin]# jcmd 12234 JFR.start name=test duration=60s filename=output.jfr
  12234: Started recording 6. The result will be written to: /root/zookeeper-3.4.12/bin/output.jfr
  [root@localhost bin]# ls -l
  -rw-r--r-- 1 root root 298585 6月  29 11:09 output.jfr
  ```



**JFR（Java Flight Recorder）**

- Java Mission Control的最主要的特征就是Java Flight Recorder。正如它的名字所示，JFR的数据是一些列JVM事件的历史纪录，可以用来诊断JVM的性能和操作
- JFR的基本操作就是开启哪些事件（比如：线程由于等待锁而阻塞的事件）。当开启的事件发生了，事件相关的数据会记录到内存或磁盘文件上。记录事件数据的缓存是循环使用的，只有最近发生的事件才能够从缓存中找到，之前的都因为缓存的限制被删除了。Java Mission Control能够对这些事件在界面上进行展示（从JVM的内存中读取或从事件数据文件中读取），我们可以通过这些事件来对JVM的性能进行诊断
- 事件的类型、缓存的大小、事件数据的存储方式等等都是通过JVM参数、Java Mission Control的GUI界面、jcmd命令来控制的。JFR默认是编译进程序的，因为它的开销很小，一般来说对应用的影响小于1%。不过，如果我们增加了事件的数目、修改了记录事件的阈值，都有可能增加JFR的开销



### JFR概况

​	下面对GlassFish web服务器进行JFR记录的例子，在这个服务器上面运行着在第2章介绍的股票servlet。Java Mission Control加载完JFR获取的事件之后，大概是下面这个样子：

![jfr-main](F:/lemon-guide-main/images/JVM/jfr-main.jpg)

我们可以看到，通过上图可以看到：CPU使用率，Heap使用率，JVM信息，System Properties，JFR的记录情况等等。



### JFR内存视图

Java Mission Control 可以看到非常多的信息，下图只显示了一个标签的内容。下图显示了JVM 的内存波动非常频繁，因为新生代经常被清除（有意思的是，head的大小并没有增长）。下面左边的面板显示了最近一段时间的垃圾回收情况，包括：GC的时长和垃圾回收的类型。如果我们点击一个事件，右边的面板会展示这个事件的具体情况，包括：垃圾垃圾回收的各个阶段及其统计信息。从面板的标签可以看到，还有很多其它信息，比如：有多少对象被清除了，花了多长时间；GC算法的配置；分配的对象信息等等。在第5章和第6章中，我们会详细介绍。

![jfr-memory](F:/lemon-guide-main/images/JVM/jfr-memory.jpg)



### JFR 代码视图

这张图也有很多tab，可以看到各个包的使用频率和类的使用情况、异常、编译、代码缓存、类加载情况等等：

![jfr-code](F:/lemon-guide-main/images/JVM/jfr-code.jpg)



### JFR事件视图

下图显示了事件的概述视图：

![jfr-event](F:/lemon-guide-main/images/JVM/jfr-event.jpg)



## EclipseMAT

虽然Java虚拟机可以帮我们对内存进行回收，但是其回收的是Java虚拟机不再引用的对象。很多时候我们使用系统的IO流、Cursor、Receiver如果不及时释放，就会导致内存泄漏（OOM）。但是，很多时候内存泄漏的现象不是很明显，比如内部类、Handler相关的使用导致的内存泄漏，或者你使用了第三方library的一些引用，比较消耗资源，但又不是像系统资源那样会引起你足够的注意去手动释放它们。以下通过内存泄漏分析、集合使用率、Hash性能分析和OQL快读定位空集合来使用MAT。



**GC Roots**

JAVA虚拟机通过可达性（Reachability)来判断对象是否存活，基本思想：`以”GC Roots”的对象作为起始点向下搜索，搜索形成的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（即不可达的），则该对象被判定为可以被回收的对象，反之不能被回收`。GC Roots可以是以下任意对象

- 一个在current thread（当前线程）的call stack（调用栈）上的对象（如方法参数和局部变量）
- 线程自身或者system class loader（系统类加载器）加载的类
- native code（本地代码）保留的活动对象



**内存泄漏**

当对象无用了，但仍然可达（未释放），垃圾回收器无法回收。



**Java四种引用类型**

- Strong References（强引用）

  普通的java引用，我们通常new的对象就是：`StringBuffer buffer = new StringBuffer();` 如果一个对象通过一串强引用链可达，那么它就不会被垃圾回收。你肯定不希望自己正在使用的引用被垃圾回收器回收吧。但对于集合中的对象，应在不使用的时候移除掉，否则会占用更多的内存，导致内存泄漏。

- Soft Reference（软引用）

  当对象是Soft Reference可达时，gc会向系统申请更多内存，而不是直接回收它，当内存不足的时候才回收它。因此Soft Reference适合用于构建一些缓存系统，比如图片缓存。

- Weak Reference（弱引用）

  WeakReference不会强制对象保存在内存中。它拥有比较短暂的生命周期，允许你使用垃圾回收器的能力去权衡一个对象的可达性。在垃圾回收器扫描它所管辖的内存区域过程中，一旦gc发现对象是Weak Reference可达，就会把它放到 `Reference Queue` 中，等下次gc时回收它。

  系统为我们提供了WeakHashMap，和HashMap类似，只是其key使用了weak reference。如果WeakHashMap的某个key被垃圾回收器回收，那么entity也会自动被remove。由于WeakReference被GC回收的可能性较大，因此，在使用它之前，你需要通过weakObj.get()去判断目的对象引用是否已经被回收。一旦WeakReference.get()返回null，它指向的对象就会被垃圾回收，那么WeakReference对象就没有用了，意味着你应该进行一些清理。比如在WeakHashMap中要把回收过的key从Map中删除掉，避免无用的的weakReference不断增长。

  ReferenceQueue可以让你很容易地跟踪dead references。WeakReference类的构造函数有一个ReferenceQueue参数，当指向的对象被垃圾回收时，会把WeakReference对象放到ReferenceQueue中。这样，遍历ReferenceQueue可以得到所有回收过的WeakReference。

- Phantom Reference（虚引用）

  其余Soft/Weak Reference区别较大是它的get()方法总是返回null。这意味着你只能用Phantom Reference本身，而得不到它指向的对象。当Weak Reference指向的对象变得弱可达(weakly reachable）时会立即被放到ReferenceQueue中，这在finalization、garbage collection之前发生。理论上，你可以在finalize()方法中使对象“复活”（使一个强引用指向它就行了，gc不会回收它）。但没法复活PhantomReference指向的对象。而PhantomReference是在garbage collection之后被放到ReferenceQueue中的，没法复活。



**MAT视图与概念**

- **Shallow Heap**

  Shallow Size就是对象本身占用内存的大小，不包含其引用的对象内存，实际分析中作用不大。 常规对象（非数组）的Shallow Size由其成员变量的数量和类型决定。数组的Shallow Size有数组元素的类型（对象类型、基本类型）和数组长度决定。案例如下：

```java
public class String {
    public final class String {8 Bytes header
    private char value[]; 4 Bytes
    private int offset; 4 Bytes
    private int count; 4 Bytes
    private int hash = 0; 4 Bytes
	// ......
}
// "Shallow size“ of a String ==24 Bytes12345678
```

Java的对象成员都是些引用。真正的内存都在堆上，看起来是一堆原生的byte[]、char[]、int[]，对象本身的内存都很小。所以我们可以看到以Shallow Heap进行排序的Histogram图中，排在第一位第二位的是byte和char。



- **Retained Heap**

  Retained Heap值的计算方式是将Retained Set中的所有对象大小叠加。或者说，由于X被释放，导致其它所有被释放对象（包括被递归释放的）所占的Heap大小。当X被回收时哪些将被GC回收的对象集合。比如: 

  一个ArrayList持有100000个对象，每一个占用16 bytes，移除这些ArrayList可以释放16×100000+X，X代表ArrayList的Shallow大小。相对于Shallow Heap，Retained Heap可以更精确的反映一个对象实际占用的大小（因为如果该对象释放，Retained Heap都可以被释放）。



- **Histogram**

  可列出每一个类的实例数。支持正则表达式查找，也可以计算出该类所有对象的Retained Size。

![mat-histogram](F:/lemon-guide-main/images/JVM/mat-histogram.jpg)



- **Dominator Tree**

  对象之间dominator关系树。如果从GC Root到达Y的的所有path都经过X，那么我们称X dominates Y，或者X是Y的Dominator Dominator Tree由系统中复杂的对象图计算而来。从MAT的dominator tree中可以看到占用内存最大的对象以及每个对象的dominator。 我们也可以右键选择Immediate Dominator”来查看某个对象的dominator。

![mat-dominator-tree](F:/lemon-guide-main/images/JVM/mat-dominator-tree.jpg)



- **Path to GC Roots**

  ​	查看一个对象到RC Roots的引用链通常在排查内存泄漏的时候，我们会选择exclude all phantom/weak/soft etc.references, 
  意思是查看排除虚引用/弱引用/软引用等的引用链，因为被虚引用/弱引用/软引用的对象可以直接被GC给回收，我们要看的就是某个对象否还存在Strong 引用链（在导出HeapDump之前要手动出发GC来保证），如果有，则说明存在内存泄漏，然后再去排查具体引用。

  ![mat-path-to-gc-roots](F:/lemon-guide-main/images/JVM/mat-path-to-gc-roots.jpg)

  查看当前Object所有引用,被引用的对象：

  - List objects with （以Dominator Tree的方式查看）
    - incoming references 引用到该对象的对象
    - outcoming references 被该对象引用的对象
  - Show objects by class （以class的方式查看）
    - incoming references 引用到该对象的对象
    - outcoming references 被该对象引用的对象



- **OQL(Object Query Language)**

类似SQL查询语言：Classes：Table、Objects：Rows、Fileds：Cols

```mysql
select * from com.example.mat.Listener
# 查找size＝0并且未使用过的ArrayList
select * from java.util.ArrayList where size=0 and modCount=01
# 查找所有的Activity 
select * from instanceof android.app.Activity
```



- **内存快照对比**

方式一：Compare To Another Heap Dump（直接进行比较）

![mat-compare-to-another-heap-dump-1](F:/lemon-guide-main/images/JVM/mat-compare-to-another-heap-dump-1.jpg)

![mat-compare-to-another-heap-dump-2](F:/lemon-guide-main/images/JVM/mat-compare-to-another-heap-dump-2.jpg)

![mat-compare-to-another-heap-dump-3](F:/lemon-guide-main/images/JVM/mat-compare-to-another-heap-dump-3.jpg)



方式二：Compare Baseket（更全面，可以直接给出百分比）

![mat-compare-baseket-1](F:/lemon-guide-main/images/JVM/mat-compare-baseket-1.jpg)

![mat-compare-baseket-2](F:/lemon-guide-main/images/JVM/mat-compare-baseket-2.jpg)

![mat-compare-baseket-3](F:/lemon-guide-main/images/JVM/mat-compare-baseket-3.jpg)

![mat-compare-baseket-4](F:/lemon-guide-main/images/JVM/mat-compare-baseket-4.jpg)



**MAT内存分析实战**

- **实战一：内存泄漏分析**

  查找导致内存泄漏的类。既然环境已经搭好，heap dump也成功倒入，接下来就去分析问题。

  - 查找目标类 
    如果在开发过程中，你的目标很明确，比如就是查找自己负责的服务，那么通过包名或者Class筛选，OQL搜索都可以快速定位到。点击OQL图标，在窗口输入，并按Ctrl + F5或者!按钮执行：

    `select * from instanceof android.app.Activity`

  - Paths to GC Roots：exclude all phantom/weak/soft etc.references 

    查看一个对象到RC Roots是否存在引用链。要将虚引用/弱引用/软引用等排除，因为被虚引用/弱引用/软引用的对象可以直接被GC给回收

  - 分析具体的引用为何没有被释放，并进行修复



**小技巧：**

- 当目的不明确时，可以直接定位到RetainedHeap最大的Object，Select incoming references，查看引用链，定位到可疑的对象然后Path to GC Roots进行引用链分析
- 如果大对象筛选看不出区别，可以试试按照class分组，再寻找可疑对象进行GC引用链分析
- 直接按照包名直接查看GC引用链，可以一次性筛选多个类，但是如下图所示，选项是 Merge Shortest Path to GCRoots，这个选项具体不是很明白，不过也能筛选出存在GC引用链的类，这种方式的准确性还待验证

![mat-实践一](F:/lemon-guide-main/images/JVM/mat-实践一.jpg)

所以有时候进行MAT分析还是需要一些经验，能够帮你更快更准确的定位。



- **实战二：集合使用率分析**

  集合在开发中会经常使用到，如何选择合适的数据结构的集合，初始容量是多少（太小，可能导致频繁扩容），太大，又会开销跟多内存。当这些问题不是很明确时或者想查看集合的使用情况时，可以通过MAT来进行分析。

  - **筛选目标对象**

    ![mat-实践二-1](F:/lemon-guide-main/images/JVM/mat-实践二-1.jpg)

  - **Show Retained Set（查找当X被回收时那些将被GC回收的对象集合）**

    ![mat-实践二-2](F:/lemon-guide-main/images/JVM/mat-实践二-2.jpg)

  - **筛选指定的Object（Hash Map，ArrayList）并按照大小进行分组**

    ![mat-实践二-3](F:/lemon-guide-main/images/JVM/mat-实践二-3.jpg)

  - **查看指定类的Immediate dominators**

    ![mat-实践二-4](F:/lemon-guide-main/images/JVM/mat-实践二-4.jpg)



**Collections fill ratio**

这种方式只能查看那些具有预分配内存能力的集合，比如HashMap，ArrayList。计算方式：”size / capacity”

![mat-实践二-5](F:/lemon-guide-main/images/JVM/mat-实践二-5.jpg)

![mat-实践二-6](F:/lemon-guide-main/images/JVM/mat-实践二-6.jpg)



- **实战三：Hash相关性能分析**

  当Hash集合中过多的对象返回相同Hash值的时候，会严重影响性能（Hash算法原理自行搜索），这里来查找导致Hash集合的碰撞率较高的罪魁祸首。

  - **Map Collision Ratio**

    检测每一个HashMap或者HashTable实例并按照碰撞率排序：**碰撞率 = 碰撞的实体/Hash表中所有实体**

    ![mat-实践三-1](F:/lemon-guide-main/images/JVM/mat-实践三-1.jpg)

  - **查看Immediate dominators**

    ![mat-实践三-2](F:/lemon-guide-main/images/JVM/mat-实践三-2.jpg)

    ![mat-实践三-3](F:/lemon-guide-main/images/JVM/mat-实践三-3.jpg)

  - **通过HashEntries查看key value**

    ![mat-实践三-4](F:/lemon-guide-main/images/JVM/mat-实践三-4.jpg)

  - **Array等其它集合分析方法类似**



- **实战四：通过OQL快速定位未使用的集合**

  - **通过OQL查询empty并且未修改过的集合：**

    ```mysql
    select * from java.util.ArrayList where size=0 and modCount=01
    select * from java.util.HashMap where size=0 and modCount=0
    select * from java.util.Hashtable where count=0 and modCount=012
    ```

    ![mat-实践四-1](F:/lemon-guide-main/images/JVM/mat-实践四-1.jpg)

  - **Immediate dominators(查看引用者)**

    ![mat-实践四-2](F:/lemon-guide-main/images/JVM/mat-实践四-2.jpg)

  - **计算空集合的Retained Size值，查看浪费了多少内存**

    ![mat-实践四-3](F:/lemon-guide-main/images/JVM/mat-实践四-3.jpg)



## 🔥火焰图

火焰图是用来分析程序运行瓶颈的工具。火焰图也可以用来分析 Java 应用。

### 环境安装

确认你的机器已经安装了**git、jdk、perl、c++编译器**。

#### 安装Perl

```shell
wget http://www.cpan.org/src/5.0/perl-5.26.1.tar.gz
tar zxvf perl-5.26.1.tar.gz
cd perl-5.26.1
./Configure -de
make
make test
make install
```

wget后面的路径可以按需更改。安装过程比较耗时间，安装完成后可通过**perl -version**查看是否安装成功。



#### C++编译器

```shell
apt-get install g++
```

一般用于编译c++程序，缺少这个编译器进行make编译c++代码时，会报“g++: not found”的错误。



#### clone相关项目

下载下来所需要的两个项目（这里建议放到data目录下）：

```shell
git clone https://github.com/jvm-profiling-tools/async-profiler
git clone https://github.com/brendangregg/FlameGraph
```



#### 编译项目

下载好以后，需要打开async-profiler文件，输入make指令进行编译：

```shell
cd async-profiler
make
```



### 生成文件

#### 生成火焰图数据

可以从 github 上下载 [async-profiler](https://github.com/jvm-profiling-tools/async-profiler) 的压缩包进行相关操作。进入async-profiler项目的目录下，然后输入如下指令：

```shell
./profiler.sh -d 60 -o collapsed -f /tmp/test_01.txt ${pid}
```

上面的-d表示的是持续时长，后面60代表持续采集时间60s，-o表示的是采集规范，这里用的是collapsed，-f后面的路径，表示的是数据采集后生成的数据存放的文件路径（这里放在了/tmp/test_01.txt），${pid}表示的是采集目标进程的pid，回车运行，运行期间阻塞，知道约定时间完成。运行完成后去tmp下看看有没有对应文件。



#### 生成svg文件

上一步产生的文件里的内容，肉眼是很难看懂的，所以现在[FlameGraph](https://github.com/brendangregg/FlameGraph)的作用就体现出来了，它可以读取该文件并生成直观的火焰图，现在进入该项目目录下面，执行如下语句：

```shell
perl flamegraph.pl --colors=java /tmp/test_01.txt > test_01.svg
```

因为是perl文件，这里使用perl指令运行该文件，后面--colors表示着色风格，这里是java，后面的是数据文件的路径，这里是刚刚上面生成的那个文件/tmp/test_01.txt，最后的test_01.svg就是最终生成的火焰图文件存放的路径和文件命名，这里是命名为test_01.svg并保存在当前路径，运行后看到该文件已经存在于当前目录下。



#### 展示火焰图

现在下载下来该文件，使用浏览器打开，效果如下：

![火焰图案例](F:/lemon-guide-main/images/JVM/火焰图案例.svg)



### 火焰图案例

生成的[火焰图案例](images/DevOps/火焰图案例.svg)如下：

![火焰图案例](F:/lemon-guide-main/images/JVM/火焰图案例.jpg)

#### 瓶颈点1

CoohuaAnalytics$KafkaConsumer:::send方法中Gzip压缩占比较高。已经定位到方法级别，再看代码就快速很多，直接找到具体位置，找到第一个消耗大户：**Gzip压缩**

![火焰图-瓶颈点1](F:/lemon-guide-main/images/JVM/火焰图-瓶颈点1.jpg)



#### 瓶颈点2

展开2这个波峰，查看到这个getOurStackTrace方法占用了大比例的CPU，怀疑代码里面频繁用丢异常的方式获取当前代码栈：

![火焰图-瓶颈点2](F:/lemon-guide-main/images/JVM/火焰图-瓶颈点2.jpg)

直接看代码：

![火焰图-瓶颈点2-代码](F:/lemon-guide-main/images/JVM/火焰图-瓶颈点2-代码.jpg)

果然如推断，找到第二个CPU消耗大户：new Exception().getStackTrace()。



#### 瓶颈点3

展开波峰3，可以看到是这个Gzip解压缩：

![火焰图-瓶颈点3](F:/lemon-guide-main/images/JVM/火焰图-瓶颈点3.jpg)

定位到具体的代码，可以看到对每个请求的参数进行了gzip解压缩：

![火焰图-瓶颈点3-代码](F:/lemon-guide-main/images/JVM/火焰图-瓶颈点3-代码.jpg)

# Java执行引擎工作原理：方法调用

- 方法调用如何实现
- 函数指针和指针函数
- CallStub源码详解

## 1 方法调用如何实现

> 计算机核心三大功能：方法调用、取指、运算

### 1.1 真实机器如何实现方法调用

- 参数入栈。有几个参数就把几个参数入栈，此时入的是调用者自己的栈
- 代码指针（eip）入栈。以便物理机器执行完调用函数之后返回继续执行原指令
- 调用函数的栈基址入栈，为物理机器从被调用者函数返回做准备
- 为调用方法分配栈空间，每个函数都有自己的栈空间。

```
// 一个进行求和运算的汇编程序
main:
    //保存调用者栈基地址，并为main()函数分配新栈空间
    pushl   %ebp
    movl    %esp, %ebp
    subl    $32,  %esp          //分配新栈空间，一共32字节

    //初始化两个操作数，一个是5，一个是3
    movl    $5,   20(%esp)
    movl    $3,   24(%esp)

    //将5和3压栈（参数入栈）
    movl    $24(%esp), %eax
    movl    %eax, 4(%esp)
    movl    20(%esp),  %eax
    movl    %eax, (%esp)
    
    //调用add函数
    calladd
    movl    %eax, 28(%esp)      //得到add函数返回结果

    //返回
    movl    $0,   %eax
    leave
    ret

add:
    //保存调用者栈基地址，并为add()函数分配新栈空间
    pushl   %ebp
    mov     %esp, %ebp
    subl    $16,  %esp

    //获取入参
    movl    12(%ebp), %(eax)
    movl    8(%ebp),  %(edx)

    //执行运算
    addl    %edx,  %eax
    movl    %eax,  -4(%ebp)

    //返回
    movl    -4(%ebp), %eax
    leave
    ret
```

> 我们先来了解一下栈空间的分配，在Linux平台上，栈是向下增长的，也就是从内存的高地址向低地址增长，所以每次调用一个新的函数时，新函数的栈顶相对于调用者函数的栈顶，内存地址一定是低方位的。

栈模型.png

#### 完成add参数压栈后的main()函数堆栈布局

```
//初始化两个操作数，一个是5，一个是3
    movl    $5,   20(%esp)
    movl    $3,   24(%esp)

    //将5和3压栈（参数入栈）
    movl    $24(%esp), %eax
    movl    %eax, 4(%esp)
    movl    20(%esp),  %eax
    movl    %eax, (%esp)
```

完成add参数复制栈布局.png

- 返回值约定保存在eax寄存器中
- ebp：栈基地址
- esp：栈顶地址
- 相对寻址：28(%esp)相对于栈顶向上偏移28字节
- 方法内的局部变量分配在靠近栈底位置，而传递的参数分配在靠近栈顶的位置

#### 调用add函数时的函数堆栈布局

调用add的堆栈布局.png

- 在调用函数之前，会自动向栈顶压如eip，以便调用结束后可正常执行原程序
- 执行函数调用时，需要手动将ebp入栈

#### 物理机器执行函数调用的主要步骤

- 保存调用者栈基址，当前IP寄存器入栈
- 调用函数时，在x86平台上，参数从右到左依次入栈
- 一个方法所分配的栈空间大小，取决于该方法内部的局部变量空间、为被调用者所传递的入参大小
- 被调用者在接收入参时，从8(%ebp)处开始，往上逐个获得每一个入参参数
- 被调用者将返回的结果保存到eax寄存器中，调用者从该寄存器中获取返回值。


### 1.2 C语言函数调用


```
//一个简单的带参数求和函数调用
#include<stdio.h>
int add(int a, int b);
int main() {
	int a = 5;
	int b = 3;
	int c = add(a, b);
	return 0;
}
int add(int a,int b) {
	int z = 1 + 2;
	return z;
}
```


```
//main函数反汇编的代码
int main() {
//参数压栈、分配空间
002D1760  push        ebp       
002D1761  mov         ebp,esp  
002D1763  sub         esp,0E4h 
//以下部分代码不需要注意
002D1769  push        ebx  
002D176A  push        esi  
002D176B  push        edi  
002D176C  lea         edi,[ebp-0E4h]  
002D1772  mov         ecx,39h  
002D1777  mov         eax,0CCCCCCCCh  
002D177C  rep stos    dword ptr es:[edi]  
002D177E  mov         ecx,offset _F08B5E04_JVM1@cpp (02DC003h)  
002D1783  call        @__CheckForDebuggerJustMyCode@4 (02D120Dh)  

//main函数正式代码部分
	int a = 5;
002D1788  mov         dword ptr [a],5  
	int b = 3;
002D178F  mov         dword ptr [b],3  
	int c = add(a, b);
002D1796  mov         eax,dword ptr [b]  
	int c = add(a, b);
002D1799  push        eax  
002D179A  mov         ecx,dword ptr [a]  
002D179D  push        ecx  
002D179E  call        add (02D1172h)  
002D17A3  add         esp,8  
002D17A6  mov         dword ptr [c],eax  
	return 0;
002D17A9  xor         eax,eax  
}
//add函数汇编代码
int add(int a,int b) {
//参数压栈、分配空间
002D16F0  push        ebp  
002D16F1  mov         ebp,esp  
002D16F3  sub         esp,0CCh  
//一下部分代码不需要注意
002D16F9  push        ebx  
002D16FA  push        esi  
002D16FB  push        edi  
002D16FC  lea         edi,[ebp-0CCh]  
002D1702  mov         ecx,33h  
002D1707  mov         eax,0CCCCCCCCh  
002D170C  rep stos    dword ptr es:[edi]  
002D170E  mov         ecx,offset _F08B5E04_JVM1@cpp (02DC003h)  
002D1713  call        @__CheckForDebuggerJustMyCode@4 (02D120Dh)  

//add函数正式代码部分
	int z = 1 + 2;
002D1718  mov         dword ptr [z],3  
	return z;
002D171F  mov         eax,dword ptr [z]  
}
```

- 其实这就是物理机器函数调用是的步骤

#### 有参数传递场景下的C程序函数调用机制

- 压栈
- 参数传递顺序
- 读取入参

### JVM函数调用机制

```
//一个简单的求和函数
package cn.leishida;

public class Test {
	public static void main(String[] args) {
		add(5,8);
	}
	public static int add(int a,int b) {
		int c = a+b;
		int d = c + 9;
		return d;
	}
}
```

```
//编译成字节码的内容
Classfile /G:/workspace/JVM/src/cn/leishida/Test.class
  Compiled from "Test.java"
public class cn.leishida.Test
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#16         // cn/leishida/Test.add:(II)I
   #3 = Class              #17            // cn/leishida/Test
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               add
  #12 = Utf8               (II)I
  #13 = Utf8               SourceFile
  #14 = Utf8               Test.java
  #15 = NameAndType        #5:#6          // "<init>":()V
  #16 = NameAndType        #11:#12        // add:(II)I
  #17 = Utf8               cn/leishida/Test
  #18 = Utf8               java/lang/Object
{
  public cn.leishida.Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iconst_5
         1: bipush        8
         3: invokestatic  #2                  // Method add:(II)I
         6: pop
         7: return
      LineNumberTable:
        line 5: 0
        line 6: 7

  public static int add(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: iload_2
         5: bipush        9
         7: iadd
         8: istore_3
         9: iload_3
        10: ireturn
      LineNumberTable:
        line 8: 0
        line 9: 4
        line 10: 9
}
SourceFile: "Test.java"
```

> 现在我们还不用了解这些字节码的具体含义和作用，但我们知道机器只认识机器码的，所以JVM一定存在一个边界，边界的一边是C，一边是机器码。换言之，C可以直接调用汇编命令。

#### 指针函数与函数指针

##### 函数指针：指向一个函数的首地址（指针符号连同函数名一起被括住）

void (*fun)(int a, int b);  //声明一个函数指针
void add(int x,int y);      //声明一个函数原型
fun = add                   //将add函数首地址赋值给fun


```
#include<stdio.h>
int (*addPointer)(int a, int b);
int add(int a, int b);

int main(){
	int a = 5;
	int b = 3;
	addPointer = add;//初始化函数指针，使其指向add首地址
	int c = addPointer(a, b);
	printf("c = %d\n", c);
	return 0;
}
int add(int a, int b) {
	int c = a + b;
	return c;
}
```


##### 指针函数：返回值是一个指针类型(指针符号没有被括号括住)

int *fun(int a,int b);

```
#include<stdio.h>
int* add(int a, int b);
int main() {
	int a = 5;
	int b = 3;
	int* c = add(a, b);
	printf("c = %p\n", c);
	return 0;
}

int* add(int a, int b) {
	int c = a + b;
	return &c;
}
```

### 1.3 JVM内部分水岭call_stub

> call_stub在JVM内部具有举足轻重的意义，这个函数指针正式JVM内部C程序与机器指令的分水岭。JVM在调用这个函数之前，主要执行C程序，而JVM通过这个函数指针调用目标函数之后，就直接进入了机器指令的领域。


```
//call_stub指针原型：
#define CAST_TO_FN_PTR(func_type,value) ((func_type),(castable_address(value)))
static CallStub call_stub()                              
{ return CAST_TO_FN_PTR(CallStub, _call_stub_entry); }
```

宏替换后：

```
static CallStub call_stub()                              
{ return (CallStub))(castable_address(_call_stub_entry)); }
```

> 在执行函数对应的第一条字节码指令之前，必须经过call_stub函数指针进入对应例程，然后在目标例程中出发对Java主函数第一条字节码指令的调用。

这段代码之前的(CallStub)实际上就是转型，我们先来看一下CallStub的定义

```
// Calls to Java
  typedef void (*CallStub)(
    address   link,
    intptr_t* result,
    BasicType result_type,
    methodOopDesc* method,
    address   entry_point,
    intptr_t* parameters,
    int       size_of_parameters,
    TRAPS
  );
```

由这段定义我们可以知道，CallStub定义了一个函数指针，他指向一个返回类型是void的，有8个入参的函数。
那么这个函数又是怎么调用的呢？

```
void JavaCalls::call(JavaValue* result, methodHandle method, JavaCallArguments* args, TRAPS) {
//省略部分代码
// do call
  { JavaCallWrapper link(method, receiver, result, CHECK);
    { HandleMark hm(thread);  // HandleMark used by HandleMarkCleaner

      StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );

      result = link.result();  // circumvent MS C++ 5.0 compiler bug (result is clobbered across call)
      // Preserve oop return value across possible gc points
      if (oop_result_flag) {
        thread->set_vm_result((oop) result->get_jobject());
      }
    }
  } // Exit JavaCallWrapper (can block - potential return oop must be preserved)
    //省略部分代码...
}
```

可以看到，JVM直接调用了call_stub并传入了8个入参，可是我们再看call_stub定义的时候是没有入参的，这是为什么呢。

```
static CallStub call_stub()                              
{ return (CallStub))(castable_address(_call_stub_entry)); }
//我们将这一行展开
{
    //声明一个指针变量
    CallStub functionPointer;
    //调用castable_address
    address returnAddress = castable_address(_call_stub_entry);
    functionPointer = (CallStub)returnAddress;
    
    //返回CallStub类型的变量
    return functionPointer;
}
```

实际上，JVM调用的是call_stub()函数所返回的函数指针变量，而CallStub在定义时就规定了这种函数指针类型有8个入参，所以JVM在调用这种类型的函数指针前也必须传入8个入参。

```
typedef uintptr_t     address_word; //unsigned integer which will hold a //pointer， 
//except for some implementations of a C++，linkage pointer to function. 
//Should never need one of those to be placed in thisype anyway.
inline address_word  castable_address(address x)              
{ return address_word(x) ; }
```

而uintptr_t实际上就是无符号整数，我们接着分析call_stub函数，实际上就是下面

```
static CallStub call_stub()                              
{ return (CallStub))(unsigned int(_call_stub_entry)); }
```

所以这个函数的逻辑总体包含两部

- 将_call_stub_entry转化为Unsigned int类型
- 将_call_stub_entry转换后的类型在转换为CallStub这一函数指针类型。

> 可是这个函数指针究竟指向哪里呢？其实在JVM初始化过程中就将_call_stub_entry这一变量指向了某个内存地址，在x86 32位Linux平台上，JVM初始化过程存在这样一条链路，这条链路从JVM的main函数开始，调用到init_global()这个全局数据初始化模块，最后在调用到StubRoutines这个例程生成模块，最终在stubGenerator_x86_32.cpp:generate_initial()函数中对_call_stub_entry变量初始化。如下图


```
StubRoutines::_call_stub_entry=generate_call_stub(StubRoutines::_call_stub_return_address);
```

这一句可以说是JVM最核心的功能，不过在分析之前我们先要弄清楚call_stub的入参：

```
StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );
```

| 参数名                     | 含义                                                         |
| -------------------------- | ------------------------------------------------------------ |
| link                       | 这是一个连接器                                               |
| result_val_address         | 函数返回值地址                                               |
| result_type                | 函数返回值类型                                               |
| method()                   | JVM内部所表示的Java对象                                      |
| entry_point                | JVM调用Java方法例程入口，经过这段例程才能跳转到Java字节码对应的机器指令运行 |
| args->parameters()         | Java方法的入参集合                                           |
| args->size_of_parameters() | Java方法的入参数量                                           |
| CHECK                      | 当前线程对象                                                 |

- 连接器link：

> 连接器link的所属类型是JavaCallWrapper，实际上是在Java函数的调用者与被调用者之间打起了一座桥梁，通过这个桥梁，我们可以实现堆栈追踪，得到整个方法的调用链路。在Java函数调用时，link指针江北保存到当前方法的堆栈中。

- method（）

> 是当前Java方法在JVM内部的表示对象，每一个Java方法在被JVM加载时，都会在内部为这个Java方法建立函数模型，保存一份Java方法的全部原始描述信息。包括：
>
>   * Java函数的名称，所属类
>   * Java函数的入参信息，包括入参类型、入参参数名、入参数量、入参顺序等
>   * Java函数编译后的字节码信息，包括对应的字节码指令、所占用的总字节数等
>   * Java函数的注解信息
>   * Java函数的继承信息
>   * Java函数的返回信息

- entry_point：

>是继_call_stub_entry例程之后的又一个最主要的例程入口
>JVM调用Java函数时通过_call_stub_entry所指向的函数地址最终调用到Java函数。而通过_call_stub_entry所指向的函数调用Java函数之前，必须要先经过entry_point例程。在这个例程里面会真正从method（）对象上拿到Java函数编译后的字节码，通过entry_point可以得到Java函数所对应的第一个字节码指令，然后开始调用。

- parameters：

> 描述Java函数的入参信息，在JVM真正调用Java函数的第一个字节码指令之前，JVM会在CallStub（）函数中解析Java函数的入参，然后为Java函数分配堆栈，并将Java函数的入参逐个入栈

- size_of_parameters:

> Java函数的入参个数。JVM根据这个值计算Java堆栈空间大小。

### _call_stub_entry例程

_call_stub_entry的值是generate_call_stub()函数赋值的，现在我们来分析一下。

```
  address generate_call_stub(address& return_address) {
    StubCodeMark mark(this, "StubRoutines", "call_stub");
    address start = __ pc();

    // stub code parameters / addresses
    assert(frame::entry_frame_call_wrapper_offset == 2, "adjust this code");
    bool  sse_save = false;
    const Address rsp_after_call(rbp, -4 * wordSize); // same as in generate_catch_exception()!
    const int     locals_count_in_bytes  (4*wordSize);
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
    const Address result        (rbp,  3 * wordSize);
    const Address result_type   (rbp,  4 * wordSize);
    const Address method        (rbp,  5 * wordSize);
    const Address entry_point   (rbp,  6 * wordSize);
    const Address parameters    (rbp,  7 * wordSize);
    const Address parameter_size(rbp,  8 * wordSize);
    const Address thread        (rbp,  9 * wordSize); // same as in generate_catch_exception()!
    sse_save =  UseSSE > 0;

    // stub code
    __ enter();
    __ movptr(rcx, parameter_size);              // parameter counter
    __ shlptr(rcx, Interpreter::logStackElementSize); // convert parameter count to bytes
    __ addptr(rcx, locals_count_in_bytes);       // reserve space for register saves
    __ subptr(rsp, rcx);
    __ andptr(rsp, -(StackAlignmentInBytes));    // Align stack

    // save rdi, rsi, & rbx, according to C calling conventions
    __ movptr(saved_rdi, rdi);
    __ movptr(saved_rsi, rsi);
    __ movptr(saved_rbx, rbx);
    // save and initialize %mxcsr
    if (sse_save) {
      Label skip_ldmx;
      __ stmxcsr(mxcsr_save);
      __ movl(rax, mxcsr_save);
      __ andl(rax, MXCSR_MASK);    // Only check control and mask bits
      ExternalAddress mxcsr_std(StubRoutines::addr_mxcsr_std());
      __ cmp32(rax, mxcsr_std);
      __ jcc(Assembler::equal, skip_ldmx);
      __ ldmxcsr(mxcsr_std);
      __ bind(skip_ldmx);
    }

    // make sure the control word is correct.
    __ fldcw(ExternalAddress(StubRoutines::addr_fpu_cntrl_wrd_std()));

#ifdef ASSERT
    // make sure we have no pending exceptions
    { Label L;
      __ movptr(rcx, thread);
      __ cmpptr(Address(rcx, Thread::pending_exception_offset()), (int32_t)NULL_WORD);
      __ jcc(Assembler::equal, L);
      __ stop("StubRoutines::call_stub: entered with pending exception");
      __ bind(L);
    }
#endif

    // pass parameters if any
    BLOCK_COMMENT("pass parameters if any");
    Label parameters_done;
    __ movl(rcx, parameter_size);  // parameter counter
    __ testl(rcx, rcx);
    __ jcc(Assembler::zero, parameters_done);

    // parameter passing loop

    Label loop;
    // Copy Java parameters in reverse order (receiver last)
    // Note that the argument order is inverted in the process
    // source is rdx[rcx: N-1..0]
    // dest   is rsp[rbx: 0..N-1]

    __ movptr(rdx, parameters);          // parameter pointer
    __ xorptr(rbx, rbx);

    __ BIND(loop);

    // get parameter
    __ movptr(rax, Address(rdx, rcx, Interpreter::stackElementScale(), -wordSize));
    __ movptr(Address(rsp, rbx, Interpreter::stackElementScale(),
                    Interpreter::expr_offset_in_bytes(0)), rax);          // store parameter
    __ increment(rbx);
    __ decrement(rcx);
    __ jcc(Assembler::notZero, loop);

    // call Java function
    __ BIND(parameters_done);
    __ movptr(rbx, method);           // get methodOop
    __ movptr(rax, entry_point);      // get entry_point
    __ mov(rsi, rsp);                 // set sender sp
    BLOCK_COMMENT("call Java function");
    __ call(rax);

    BLOCK_COMMENT("call_stub_return_address:");
    return_address = __ pc();

#ifdef COMPILER2
    {
      Label L_skip;
      if (UseSSE >= 2) {
        __ verify_FPU(0, "call_stub_return");
      } else {
        for (int i = 1; i < 8; i++) {
          __ ffree(i);
        }

        // UseSSE <= 1 so double result should be left on TOS
        __ movl(rsi, result_type);
        __ cmpl(rsi, T_DOUBLE);
        __ jcc(Assembler::equal, L_skip);
        if (UseSSE == 0) {
          // UseSSE == 0 so float result should be left on TOS
          __ cmpl(rsi, T_FLOAT);
          __ jcc(Assembler::equal, L_skip);
        }
        __ ffree(0);
      }
      __ BIND(L_skip);
    }
#endif // COMPILER2

    // store result depending on type
    // (everything that is not T_LONG, T_FLOAT or T_DOUBLE is treated as T_INT)
    __ movptr(rdi, result);
    Label is_long, is_float, is_double, exit;
    __ movl(rsi, result_type);
    __ cmpl(rsi, T_LONG);
    __ jcc(Assembler::equal, is_long);
    __ cmpl(rsi, T_FLOAT);
    __ jcc(Assembler::equal, is_float);
    __ cmpl(rsi, T_DOUBLE);
    __ jcc(Assembler::equal, is_double);

    // handle T_INT case
    __ movl(Address(rdi, 0), rax);
    __ BIND(exit);

    // check that FPU stack is empty
    __ verify_FPU(0, "generate_call_stub");

    // pop parameters
    __ lea(rsp, rsp_after_call);

    // restore %mxcsr
    if (sse_save) {
      __ ldmxcsr(mxcsr_save);
    }

    // restore rdi, rsi and rbx,
    __ movptr(rbx, saved_rbx);
    __ movptr(rsi, saved_rsi);
    __ movptr(rdi, saved_rdi);
    __ addptr(rsp, 4*wordSize);

    // return
    __ pop(rbp);
    __ ret(0);

    // handle return types different from T_INT
    __ BIND(is_long);
    __ movl(Address(rdi, 0 * wordSize), rax);
    __ movl(Address(rdi, 1 * wordSize), rdx);
    __ jmp(exit);

    __ BIND(is_float);
    // interpreter uses xmm0 for return values
    if (UseSSE >= 1) {
      __ movflt(Address(rdi, 0), xmm0);
    } else {
      __ fstp_s(Address(rdi, 0));
    }
    __ jmp(exit);

    __ BIND(is_double);
    // interpreter uses xmm0 for return values
    if (UseSSE >= 2) {
      __ movdbl(Address(rdi, 0), xmm0);
    } else {
      __ fstp_d(Address(rdi, 0));
    }
    __ jmp(exit);

    return start;
  }
```

> 这段代码的主要作用就是生成机器码，使用C语言动态生成，弄懂这段指令的逻辑，对理解JVM的字节码执行引擎至关重要。

- 1 pc()函数

```
 address start = __ pc();
```

pc()函数定义：

```
address pc() const{
    return _code_pos;
}
```

> JVM启动过程中，会生成许多例程，例如函数调用、字节码例程、异常处理、函数返回等。每一个例程的开始都是address start = __pc();（JVM的例程都会写入JVM堆内存）。偏移量_code_pos指向上一个例程的最后字节的位置，新的例程开始时会先保存这个起始位置，以便JVM通过CallStub这个函数指针执行这段动态生成的机器指令。

- 2 定义入参

```
// stub code parameters / addresses
    assert(frame::entry_frame_call_wrapper_offset == 2, "adjust this code");
    bool  sse_save = false;
    const Address rsp_after_call(rbp, -4 * wordSize); // same as in generate_catch_exception()!
    const int     locals_count_in_bytes  (4*wordSize);
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
    const Address result        (rbp,  3 * wordSize);
    const Address result_type   (rbp,  4 * wordSize);
    const Address method        (rbp,  5 * wordSize);
    const Address entry_point   (rbp,  6 * wordSize);
    const Address parameters    (rbp,  7 * wordSize);
    const Address parameter_size(rbp,  8 * wordSize);
    const Address thread        (rbp,  9 * wordSize); // same as in generate_catch_exception()!
    sse_save =  UseSSE > 0;
```

> 一个函数的堆栈空间大体上可以分为3部分：
>
> - 堆栈变量区：
>   保存方法的局部变量，或者对数据的地址引用（指针），如果没有局部变量，则不分配本区
> - 入参区域
>   如果当前方法调用了其他方法，并且传了参数，那么这些入参会保存在调用者堆栈中，就是所谓的压栈
> - ip和bp区：一个是代码段寄存器，一个是堆栈栈基急促安琪，一个用于恢复调用者方法的代码位置，一个用于恢复调用方法的堆栈位置。这部分前面有提到过

| 入参                                    | 位置     |
| --------------------------------------- | -------- |
| (address)&link:连接器                   | 8(%ebp)  |
| result_val_address:返回地址             | 12(%ebp) |
| result_type:返回类型                    | 16(%ebp) |
| method():方法内部对象                   | 20(%ebp) |
| entry_point:Java方法调用入口例程        | 24(%ebp) |
| parameters():Java方法的入参             | 28(%ebp) |
| size_of_parametres():Java方法的入参数量 | 32(%ebp) |
| CHECK:当前线程                          | 36(%ebp) |

如果按N=4字节来寻址，那我们可以这样写

| 入参                                    | 位置     | C++类标记             |
| --------------------------------------- | -------- | --------------------- |
| (address)&link:连接器                   | 2N(%ebp) | Address link(rbp, 2N) |
| result_val_address:返回地址             | 3N(%ebp) | Address link(rbp, 3N) |
| result_type:返回类型                    | 4N(%ebp) | Address link(rbp, 4N) |
| method():方法内部对象                   | 5N(%ebp) | Address link(rbp, 5N) |
| entry_point:Java方法调用入口例程        | 6N(%ebp) | Address link(rbp, 6N) |
| parameters():Java方法的入参             | 7N(%ebp) | Address link(rbp, 7N) |
| size_of_parametres():Java方法的入参数量 | 8N(%ebp) | Address link(rbp, 8N) |
| CHECK:当前线程                          | 9N(%ebp) | Address link(rbp, 9N) |

如此一来，generate_call_stub函数前面十几行的类声明就不难理解了

```
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
```

这四个相对于rbp偏移量为负，说明这四个参数位置在CalStub（）函数的堆栈内部，而这四个变量用于保存调用者的信息，后面会详细分析。

- 3 CallStub：保存调用者堆栈

这部分逻辑从这行代码开始

```
 // stub code
    __ enter();
```

> 这行代码在不同的硬件平台对应不同的机器指令。

enter()的定义：

```
push(rbp);
mov(rbp,rsp);
```

很明显是把rbp压栈的操作，前面已经讲过了。

- 4 CallStub：动态分配堆栈

> JVM为了能够调用Java函数，需要在运行期指导一个Java函数的入参大小，然后动态计算出所需要的堆栈空间。由于物理机器不能识别Java程序，因此JVM必然要通过自己作为中间桥梁连接到Java程序，并让Java被调用的函数的堆栈能够计生在JVM的CallStub（）函数堆栈。
> 想要实现寄生在CallStub的函数堆栈中，我们就要对它的空间进行扩展，物理机器为扩展堆栈提供了简单指令：

```
sub operand,%esp
```

> 在JVM内部，一切Java对象实例及其成员变量和成员方法的访问，最终皆通过指针得以寻址，同理，JVM在传递Java函数参数时所传递的也只是Java入参对象实例的指针，而指针的宽度在特定的硬件平台上是一样的。


```
    __ movptr(rcx, parameter_size);              // parameter counter
    __ shlptr(rcx, Interpreter::logStackElementSize); // convert parameter count to bytes，将参数长度转化为字节，保存到rcx
    
    __ addptr(rcx, locals_count_in_bytes);       // reserve space for register saves，保存rdi,rsi,rbx,mxcsr四个寄存器的值
    __ subptr(rsp, rcx);    //扩展堆栈空间，rcx是之前计算出的参数的字节
    __ andptr(rsp, -(StackAlignmentInBytes));    // Align stack内存对齐
```

至此，JVM完成了动态堆栈内存分配！！

- 5 CallStub：调用者保存

> esi、edi、ebx属于调用者函数的私有数据，在发生函数调用之前，调用者函数必须将这些数据保存起来，以便被调用函数执行完毕从新回到调用者函数中时能够正常运行。JVM直接将其保存到了被调用者函数的堆栈中。

```
// save rdi, rsi, & rbx, according to C calling conventions
    __ movptr(saved_rdi, rdi);
    __ movptr(saved_rsi, rsi);
    __ movptr(saved_rbx, rbx);
    //...这部分是保存mxcsr的，属于Intel的SSE技术
```

- 6 CallStub：参数压栈
  接下来JVM要做的事将即将被调用的Java函数入参复制到剩余的堆栈空间。
  采取基址+变址

```
 // pass parameters if any
    BLOCK_COMMENT("pass parameters if any");
    Label parameters_done;
    __ movl(rcx, parameter_size);  // parameter counter参数数量，控制循环次数
    __ testl(rcx, rcx);          //判断参数数量是否为0，如果是，直接跳过参数处理
    __ jcc(Assembler::zero, parameters_done);

    // parameter passing loop

    Label loop;
    // Copy Java parameters in reverse order (receiver last)
    // Note that the argument order is inverted in the process
    // source is rdx[rcx: N-1..0]
    // dest   is rsp[rbx: 0..N-1]

    __ movptr(rdx, parameters);          // parameter pointer，第一个入参地址
    __ xorptr(rbx, rbx);

    __ BIND(loop);
    
```

此时物理寄存器状态：

| 寄存器名称                     | 指向             |
| ------------------------------ | ---------------- |
| edx                            | parameters首地址 |
| ecx                            | Java函数入参数量 |
| 现在开始循环将Java函数参数压栈 |                  |

```
// get parameter
    __ movptr(rax, Address(rdx, rcx, Interpreter::stackElementScale(), -wordSize));
    __ movptr(Address(rsp, rbx, Interpreter::stackElementScale(),
                    Interpreter::expr_offset_in_bytes(0)), rax);          // store parameter
    __ increment(rbx);
    __ decrement(rcx);
    __ jcc(Assembler::notZero, loop);
```

至此，Java函数的3个参数全部被压栈，离函数调用越来越近

### 7 CallStub：调用entry_point例程

> 前面经过调用者框架栈帧保存（栈基），堆栈动态扩展、现场保存、Java函数参数压栈这一系列处理，JVM终于为Java函数的调用演完前奏，而标志开始的则是entry_point例程。


```
  // call Java function
    __ BIND(parameters_done);
    __ movptr(rbx, method);           // get methodOop
    __ movptr(rax, entry_point);      // get entry_point
    __ mov(rsi, rsp);                 // set sender sp
    BLOCK_COMMENT("call Java function");
    __ call(rax);
```

到目前为止各个参数的位置

| 参数               | 保存位置           |
| ------------------ | ------------------ |
| link               | 仍在堆栈中8(%ebp)  |
| result_val_address | 仍在堆栈中12(%ebp) |
| result_type        | 仍在堆栈中16(%ebp) |
| method             | ebx寄存器          |
| entry_point        | eax寄存器          |
| parameters         | edx寄存器          |
| size_of_parameters | ecx寄存器          |
| CHECK              | 仍在堆栈中32(%ebp) |

物理寄存器保存的重要信息

| 寄存器名 | 指向                                         |
| -------- | -------------------------------------------- |
| edx      | parameters首地址                             |
| ecx      | Java函数入参数量                             |
| ebx      | 指向Java函数，即Java函数所对应的的method对象 |
| esi      | CallStub栈顶                                 |
| eax      | entry_point例程入口                          |

> 此时eax已经指向了entry_point例程入口，只需要call一下就可以赚到entry_point例程，去执行entry_point例程

 - 8 获取返回值

 > 调用完entry_point例程之后会有返回值，CallStub会获取返回值并继续处理。

```
// store result depending on type
    // (everything that is not T_LONG, T_FLOAT or T_DOUBLE is treated as T_INT)
    __ movptr(rdi, result);
    Label is_long, is_float, is_double, exit;
    __ movl(rsi, result_type);
```

> 上面的代码存储了结果和结果类型，JVM会将这两个值分别存进edi和esi这两个寄存器，这是基于约定的技术实现方式。

call_stub例程的讲解至此告一段落，置于JVM内部如何从C/C++程序完成Java函数的调用，会在之后讲解完entry_point例程后揭开真相。

# 常量池解析

- Java字节码常量池的内存分配链路
- oop-klass模型
- 常量池的解析原理

> 在字节码文件中，常量池的字节码流所在的块区紧跟在魔数和版本号之后，因此JVM在解析完魔数与版本号后就开始解析常量池。JVM解析Java类字节码文件的接口：ClassFileParser::parseClassFile(),总体步骤如下：解析魔数-->解析版本号-->解析常量池-->解析父类-->解析接口-->解析类变量-->解析类方法-->构建类结构。JVM解析常量池信息主要链路:ClassFileParser::parseClassFile()-->ClassFileParser::parse_constant_pool()-->oopFactory::new_constantPool()（分配常量池内存）；ClassFileParser::parse_constant_pool_entries()(解析常量池信息)。

## 常量池内存分配

### 常量池内存分配总体链路

- ClassFileParser::parse_constant_pool()函数中，通过以下代码实现常量池内存分配：

```
onstantPoolOop constant_pool = oopFactory::new——constantPool(length,oopDesc::IsSafeConc,CHECK_(nullHandle));
```

- length:代表当前字节码文件的常量池中一共包含多少个常量池元素，由Java编译器在编译期间通过计算得出，保存在字节码文件中。

oopFactory::new_constantPool()的链路比较长，下图展示了总体调用路径：
常量池内存分配链路.png

- oopFactory：oop的工厂类，JVM内部，查了那个量尺、字段、符号、方法等一切都被对象包装起来，在内存中通过oop指针进行跟踪，而这些对象的内存分配、对象创建与初始化工作都通过oopFactory这个入口实现。
- constantPoolKlass：常量池类型对象。JVM内部预留的一段描述常量池结构信息的内存。
- collectedHeap：JVM内部的堆内存区域，可被垃圾收集器回收并反复利用。其代表JVM内部广义的堆内存区域。除了堆栈变量之外的一切内存分配都需要经过本区域。
- psPermGen：表示perm区内存，JDK1.6版本产物，Java类的字节码信息会保存到这块区域，JDK1.8后，perm去概念被metaSpace概念取代。

> perm指内存的永久保存区域，这一部分用于存放Class和Meta的信息，Class在被加载的时候放入permGen space区域，如果加载了太多类就很可能出现PermGen space错误。而在metaSpace时代，这块区域是属于“本地内存”区域，也就是操作系统，相对于JVM虚拟机内存而言的，JVM直接向操作系统申请内存存储Java类的袁茜茜，如此一来可以解决为permGen space指定的内存太小而导致perm区内存耗尽的问题。

**从宏观层面上看，常量池内存分配大体上可以分为以下三步：**

- 在堆区分配内存空间
- 初始化对象
- 初始化oop

### 内存分配

> 从oopFactory::new_constantPool()调用开始，到mutableSpace:allocate()的过程可认为是第一阶段，即为constantPool申请内存。

-JVM为constantPool申请内存链路

- oopFactory.cpp(ck->allocate(length,is_conc_safe,CHECK_NULL))
- constantPoolKlass.cpp(CollectedHeap::permanent_obj_allocate(klass, size, CHECK_NULL))
- collectedHeap.inline.hpp(permanent_obj_allocate_no_klass_install(klass,size,CHECK_NULL))
- collectedHeap.inline.hpp(common_permanent_men_allocate_init(size,CHECK_NULL))
- collectedHeap.inline.hpp(common_permanent_men_allocate_noinit(size,CHECK_NULL))
- collectedHeap.inline.hpp(Universe::heap()->permanent_men_allocate(size))
- parallelScavengeHeap.cpp(perm_gen()->allocate_permanent(size))
- psPermGen.cpp(allocate_noexpand(size,false))
- paOldGen.hpp(objec_space()->allocate(word_size))

内存申请最终通过objec_space()->allocate(word_size)实现，定义如下：

```
HeapWord* MutableSpace::allocate(size_t size) {
  assert(Heap_lock->owned_by_self() ||
         (SafepointSynchronize::is_at_safepoint() &&
          Thread::current()->is_VM_thread()),
         "not locked");
  HeapWord* obj = top();
  if (pointer_delta(end(), obj) >= size) {
    HeapWord* new_top = obj + size;
    set_top(new_top);
    assert(is_object_aligned((intptr_t)obj) && is_object_aligned((intptr_t)new_top),
           "checking alignment");
    return obj;
  } else {
    return NULL;
  }
}
```

- 通过HeapWord* new_top = obj + size;将permSpace内存区域的top指针往高地址方向移动了size大小的字节数，完成了内存分配（仅仅从JVM申请的堆内存中划拨了一块空间用于存储常量池结构信息）
- 先执行HeapWord* obj = top();然后执行HeapWord* new_top = obj + size;返回的仍是原堆顶obj指针，可以通过指针还原从原堆顶到当前堆顶之间的内存空间，将其强转型为常量池对象。
- JVM在启动过程中完成permSpace的内存申请和初始化，每次新写入一个对象，该对象的内存首地址便是写入之前top指针所指位置。

#### constantPool的内存有多大

> 为一个Java类创建其对应的常量池，需要在JVM堆区为常量池先申请一块连续空间，所申请的空间大小取决于一个Java类在编译时所确定的常量池大小（size），在常量池初始化链路中调用constantPoolKlass::allocate()方法，这个方法会调用constantPoolOopDesc::object_size(length)方法获取常量池大小，方法原型：


```
 static int header_size()             { return sizeof(constantPoolOopDesc)/HeapWordSize; }
 static int object_size(int length)   { return align_object_size(header_size() + length); }
```

> object_size()的逻辑十分简单，就是将header_size和length相加后进行内存对齐。header_size（）返回的是constantPoolOopDesc类型的大小。align_object_size()是实现内存对齐，便于GC进行工作时更加高效。在32位操作系统上，HeapWordSize大小为4，是一个指针变量的长度，在32位平台上sizeof(constantPoolOopDesc)返回40.

##### 关于sizeof

> 当计算C++类时，返回的是其所有变量的大小加上虚函数指针大小，若在类中定义了普通函数，都不会计算其大小。


```
#include<stdio.h>

class A {
private:
	char* a;
public:
	int getA() {
		return 1;
	}
};

int main() {
	printf("sizeof(A)=%d\n", sizeof(A));
	return 0;
}
```

**这一段程序在32位系统输出是4,64位系统输出为8，因为他只包含一个char型指针变量。如此一来我们可以推算sizeof(constantPoolOopDesc)的返回值大小**
constantPoolOopDesc本身包含8个字段

- typeArrayOop           _tags
- constantPoolCacheOop   _cache
- klassOop               _pool_holder
- typeArrayOop           _operands
- int                    _flags
- int                    _length
- volatile bool          _is_conc_safe
- int                    _orig_length

而由于constantPoolOopDesc继承自oopDesc类，因此还会包含父类的成员变量：

```
volatile markOop _mark;     //指针
union _metadata{            //指针
    wideKlassOop _klass;
    narrowOop _compressed_klass;
}_metadata;
```

- 所以，constantPoolOopDesc最终包含10个字段，在32位平台上占40字节。
  我们接下来在看HeapWordSize的定义：

```
const int HeapWordSize        = sizeof(HeapWord);
class HeapWord {
  friend class VMStructs;
 private:
  char* i;
#ifndef PRODUCT
 public:
  char* value() { return i; }
#endif
};
```

> 可以看到，HeapWordSize就是一个char*类型的大小，在32位平台是4,64位平台是8，所以return sizeof(constantPoolOopDesc)/HeapWordSize;返回的就是constantPoolOopDesc类型本身所占内存的总字节数。所以最终constantPoolKlass::allocate()从JVM堆中所申请的内存空间大小包含：constantPoolOopDesc大小和Java类常量池元素数量。

我们来举一个例子：

```
public class Iphone6S {
    int length = 138;
    int width = 67;
    int height = 7;
    int weight = 142;
    int ram = 2;
    int rom = 16;
    int pixel = 1200;
}
```

字节码中显示常量池大小为0x0023，对应十进制35，因此常量池一共包含38个元素，再看javap反编译的信息：

```
Constant pool:
   #1 = Methodref          #10.#25        // java/lang/Object."<init>":()V
   #2 = Fieldref           #9.#26         // JVM/Iphone6S.length:I
   #3 = Fieldref           #9.#27         // JVM/Iphone6S.width:I
   #4 = Fieldref           #9.#28         // JVM/Iphone6S.height:I
   #5 = Fieldref           #9.#29         // JVM/Iphone6S.weight:I
   #6 = Fieldref           #9.#30         // JVM/Iphone6S.ram:I
   #7 = Fieldref           #9.#31         // JVM/Iphone6S.rom:I
   #8 = Fieldref           #9.#32         // JVM/Iphone6S.pixel:I
   #9 = Class              #33            // JVM/Iphone6S
  #10 = Class              #34            // java/lang/Object
  #11 = Utf8               length
  #12 = Utf8               I
  #13 = Utf8               width
  #14 = Utf8               height
  #15 = Utf8               weight
  #16 = Utf8               ram
  #17 = Utf8               rom
  #18 = Utf8               pixel
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               SourceFile
  #24 = Utf8               Iphone6S.java
  #25 = NameAndType        #19:#20        // "<init>":()V
  #26 = NameAndType        #11:#12        // length:I
  #27 = NameAndType        #13:#12        // width:I
  #28 = NameAndType        #14:#12        // height:I
  #29 = NameAndType        #15:#12        // weight:I
  #30 = NameAndType        #16:#12        // ram:I
  #31 = NameAndType        #17:#12        // rom:I
  #32 = NameAndType        #18:#12        // pixel:I
  #33 = Utf8               JVM/Iphone6S
  #34 = Utf8               java/lang/Object
```

JVM会保留0号常量池位置，所以只有34个元素。按照上面分析，JVM会从permSpace中划分（40+35）*4字节的内存大小（32位平台）。

#### 内存空间布局

> JVM为常量池对象申请的内存位于perm区，perm区本事是一片连续的内存区域，而JVM为常量池申请内存时也是正片区域连续划分，因此每一个constantPoolOop对象实例在perm区中都是连续分布的，不会存在碎片化。

> 最终申请好的空间布局如图ConstantPoolOop内存布局.png所示（假设运行于Linux32位平台，指针宽度为4字节，并且常量池分配方向从低地址到高地址。JVM内部为对象分配内存时，先分配对象头，然后分配对象的实例数据，不管字段对象还是方法对象亦或是数组，都是如此。

#### 初始化内存

> JVM为Java类所对应的常量分配好内存空间后，接着需要对这段内存空间进行初始化（实际上是清零操作）。在Linux32位平台上最终调用pd_fill_to_words()函数，声明如下：


```
static void pd_fill_to_words(HeapWord* tohw, size_t count, juint value) {
#ifdef AMD64
  julong* to = (julong*) tohw;
  julong  v  = ((julong) value << 32) | value;
  while (count-- > 0) {
    *to++ = v;
  }
#else
  juint* to = (juint*)tohw;
  count *= HeapWordSize / BytesPerInt;
  while (count-- > 0) {
    *to++ = value;
  }
#endif // AMD64
}
```

> 可以看到这个函数有3个入参，会将指定内存区的内存数据全部清空为value值，在CollectedHeap::init_obj()中调用了Copy::fill_to_aligned_words(obj+hs,size-hs)函数，而Copy::fill_to_aligned_words()函数有三个入参，声明如下：

```
  static void fill_to_aligned_words(HeapWord* to, size_t count, juint value = 0) {
    assert_params_aligned(to);
    pd_fill_to_aligned_words(to, count, value);
  }
```

**该函数第三个参数默认为0，所以在执行pd_fill_to_aligned_words（）函数时，指定的内存区会全部清零。

### oop-klass模型

#### 两模型三维度

> JVM内部基于oop-klass模型描述一个Java类，将一个Java类一拆为二，分别描述，第一个模型是oop，第二个模型是klass。所谓oop，实际上是指ordinary object pointer（普通对象指针），用来表示对象的实例信息，看起来像个指针，实际上对象实例数据都藏在指针所指向的内存首地址后面的一片内存区域中。而klass则包含元数据和方法信息，用来描述Java类或者JVM内部自带的C++类型信息，**Java类的继承信息、成员变量、静态变量、成员方法、构造函数等信息都在klass中保存。JVM由此便可以在运行期反射出Java类的全部结构信息。**

- oop模型：侧重于描述Java类的实例数据，为Java类生成一张“实例数据视图”，从数据维度描述一个Java类实例对象中各属性在运行期的值
- klass模型
  * Java类的“元信息视图”，为JVM在运行期呈现Java类的全息数据结构信息，**是JVM在运行期的移动台反射出类信息的基础。**
  * 虚函数列表（方法分发规则）

> tip:Java类的所有函数都视为是“virtual”的，这样Java类的每个方法都可以直接被其子类覆盖而不需要添加任何关键字作为修饰符，因此，**Java类中的每个方法都可以晚绑定**，而也正是因为所有函数都视为虚函数，所以在JVM内部的C++就必须维护一套函数分发表。

#### 体系总览

oop klass handle的三角关系.png

> handle是对oop的行为的封装，在访问Java类时，一定是通过handle内部指针得到oop示例的，在通过oop就能拿到klass，如此handle最终便能操纵oop的行为了。
> Handle类的基本结构：

```
class Handle VALUE_OBJ_CLASS_SPEC {
 private:
  oop* _handle;
//省略部分代码
};
```

> 可以看到Handle内部只有一个成员变量_handle指向oop类型，因此该变量指向的是一个oop首地址。oop一般由对象头、对象专有属性和数据体三部分构成，结构如图
> oop模型.png
> JVM内部定义了若干oop类型，每一种oop类型都有自己特有的数据结构，oop的专有属性区便是用于存放各个oop所特有的数据结构的地方。

#### oop体系

究竟什么是普通对象指针？

- Hotspot里的oop指什么

> Hotspot里的oop其实就是GC所托管的指针，所有oopDesc及其子类（除了markOopDesc外）的实例都是由GC所管理

- 对象指针前为何冠以“普通”二字

> 在HotSpot里面，oop就是指一个真正的指针，而markOop则是一个看起来像指针，实际上是藏在指针里面的对象（数据），并没有指向内存的功能，这也正是它不受GC托管的原因，只要除了函数作用域，指针变量就会直接从堆栈上释放，不需要垃圾回收了。

oop的继承体系：

```
typedef class oopDesc*                            oop;                  //所有oop顶级父类
typedef class   instanceOopDesc*            instanceOop;                //表示Java类实例
typedef class   methodOopDesc*                    methodOop;            //表示Java方法
typedef class   constMethodOopDesc*            constMethodOop;          //表示Java方法中的只读信息（字节码指令）
typedef class   methodDataOopDesc*            methodDataOop;            //表示性能统计的相关数据
typedef class   arrayOopDesc*                    arrayOop;              //表示数组对象
typedef class     objArrayOopDesc*            objArrayOop;              //表示引用类型数组对象
typedef class     typeArrayOopDesc*            typeArrayOop;            //表示基本类型数组对象
typedef class   constantPoolOopDesc*            constantPoolOop;        //表示Java字节码文件中的常量池
typedef class   constantPoolCacheOopDesc*   constantPoolCacheOop;       //与constantPoolOop伴生，是后者的缓存对象
typedef class   klassOopDesc*                    klassOop;              //指向JVM内部的klass实例的对象
typedef class   markOopDesc*                    markOop;                //oop的标记对象
typedef class   compiledICHolderOopDesc*    compiledICHolderOop;
```

#### klass体系

- klass提供一个与Java类对等的C++类型描述
- klass提供虚拟机内部的函数分发机制

klass的继承体系：

```
class Klass;                                //klass家族基类
class   instanceKlass;                      //虚拟机层面上与Java类对等的数据结构
class     instanceMirrorKlass;              //描述java.lang.Class的实例
class     instanceRefKlass;                 //描述java.lang.ref.Reference的子类
class   methodKlass;                        //表示Java类的方法
class   constMethodKlass;                   //描述Java类方法所对应的字节码指令信息的固有属性
class   methodDataKlass;                    
class   klassKlass;                         //klass链路末端，在jdk8中已经不存在
class     instanceKlassKlass;               
class     arrayKlassKlass;
class       objArrayKlassKlass;
class       typeArrayKlassKlass;
class   arrayKlass;                         //描述Java数组的信息，是抽象基类
class     objArrayKlass;                    //描述Java引用类型数组的数据结构
class     typeArrayKlass;                   //描述Java中基本类型数组的数据结构
class   constantPoolKlass;                  //描述Java字节码文件中的常量池的数据结构
class   constantPoolCacheKlass;
class   compiledICHolderKlass;
```

基类klass定义：

```
class Klass : public Klass_vtbl {
  friend class VMStructs;
 protected:
 
  enum { _primary_super_limit = 8 };
  jint        _layout_helper;
  juint       _super_check_offset;
  Symbol*     _name;

 public:
  oop* oop_block_beg() const { return adr_secondary_super_cache(); }
  oop* oop_block_end() const { return adr_next_sibling() + 1; }

 protected:
  klassOop    _secondary_super_cache;
  objArrayOop _secondary_supers;
  klassOop    _primary_supers[_primary_super_limit];
  oop       _java_mirror;
  klassOop  _super;
  klassOop _subklass;
  klassOop _next_sibling;

  //
  // End of the oop block.
  //
  jint        _modifier_flags;  // Processed access flags, for use by Class.getModifiers.
  AccessFlags _access_flags;    // Access flags. The class/interface distinction is stored here.
  juint    _alloc_count;        // allocation profiling support - update klass_size_in_bytes() if moved/deleted

  // Biased locking implementation and statistics
  // (the 64-bit chunk goes first, to avoid some fragmentation)
  jlong    _last_biased_lock_bulk_revocation_time;
  markOop  _prototype_header;   // Used when biased locking is both enabled and disabled for this type
  jint     _biased_lock_revocation_count;
  //省略部分代码
};
```

| 字段名          | 含义                     |
| --------------- | ------------------------ |
| _layout_helper  | 对象布局的综合描述符     |
| _name           | 类名，如java/lang/String |
| _java_mirror    | 类的镜像类               |
| _super          | 父类                     |
| _subklass       | 指向第一个子类           |
| _next_sibling   | 指向下一个兄弟结点       |
| _modifier_flags | 修饰符标识，如static     |
| _access_flags   | 访问权限标识，如public   |

- 如果一个KLASS既不是instance也不是array，则_layout_helper被设置为0，若是instance，则为正数，若是数组，则为负数。

#### handle体系

> handle通过oop拿到klass，可以说是对普通对象的一种间接引用，这完全是为GC考虑

- 通过handle能够让GC知道其内部代码有哪些地方持有GC所管理的对象的引用，只需要扫码handle对应的table，JVM无需关注其内部到底哪些地方持有对普通对象的引用
- 在GC过程中，如果发生了对象移动（如从新生代移到老年代）那么JVM内部引用无需跟着更改为被移动对象的新地址，只需要修改handle table的对应指针。

**在C++领域，类的继承和多态性最终通过vptr（虚函数表）来实现，在klass内部记录了每一个类的vptr信息**

- vptr虚函数表

> 存放Java类中非静态和非private方法入口，jVM调用Java类的方法（非static和非private）时，最终访问vtable找到对应入口

- itable接口函数表

> 存放java类所实现的接口类方法，JVM调用接口方法时，最终访问itable找到对应入口。

**vtable和itable并不存放Java类方法和接口方法的直接入口，而是指向了Method对象入口。**

handle体系家族：

```
// Specific Handles for different oop types oop家族对应的handle宏定义
#define DEF_HANDLE(type, is_a)                   \
  class type##Handle;                            \
  class type##Handle: public Handle {            \
   protected:                                    \
    type##Oop    obj() const                     { return (type##Oop)Handle::obj(); } \
    type##Oop    non_null_obj() const            { return (type##Oop)Handle::non_null_obj(); } \
                                                 \
   public:                                       \
    /* Constructors */                           \
    type##Handle ()                              : Handle()                 {} \
    type##Handle (type##Oop obj) : Handle((oop)obj) {                         \
      assert(SharedSkipVerify || is_null() || ((oop)obj)->is_a(),             \
             "illegal type");                                                 \
    }                                                                         \
    type##Handle (Thread* thread, type##Oop obj) : Handle(thread, (oop)obj) { \
      assert(SharedSkipVerify || is_null() || ((oop)obj)->is_a(), "illegal type");  \
    }                                                                         \
    \
    /* Special constructor, use sparingly */ \
    type##Handle (type##Oop *handle, bool dummy) : Handle((oop*)handle, dummy) {} \
                                                 \
    /* Operators for ease of use */              \
    type##Oop    operator () () const            { return obj(); } \
    type##Oop    operator -> () const            { return non_null_obj(); } \
  };


DEF_HANDLE(instance         , is_instance         )
DEF_HANDLE(method           , is_method           )
DEF_HANDLE(constMethod      , is_constMethod      )
DEF_HANDLE(methodData       , is_methodData       )
DEF_HANDLE(array            , is_array            )
DEF_HANDLE(constantPool     , is_constantPool     )
DEF_HANDLE(constantPoolCache, is_constantPoolCache)
DEF_HANDLE(objArray         , is_objArray         )
DEF_HANDLE(typeArray        , is_typeArray        )
```

```
// Specific KlassHandles for different Klass types  klass家族对应的handle宏定义

#define DEF_KLASS_HANDLE(type, is_a)             \
  class type##Handle : public KlassHandle {      \
   public:                                       \
    /* Constructors */                           \
    type##Handle ()                              : KlassHandle()           {} \
    type##Handle (klassOop obj) : KlassHandle(obj) {                          \
      assert(SharedSkipVerify || is_null() || obj->klass_part()->is_a(),      \
             "illegal type");                                                 \
    }                                                                         \
    type##Handle (Thread* thread, klassOop obj) : KlassHandle(thread, obj) {  \
      assert(SharedSkipVerify || is_null() || obj->klass_part()->is_a(),      \
             "illegal type");                                                 \
    }                                                                         \
                                                 \
    /* Access to klass part */                   \
    type*        operator -> () const            { return (type*)obj()->klass_part(); } \
                                                 \
    static type##Handle cast(KlassHandle h)      { return type##Handle(h()); } \
                                                 \
  };


DEF_KLASS_HANDLE(instanceKlass         , oop_is_instance_slow )
DEF_KLASS_HANDLE(methodKlass           , oop_is_method        )
DEF_KLASS_HANDLE(constMethodKlass      , oop_is_constMethod   )
DEF_KLASS_HANDLE(klassKlass            , oop_is_klass         )
DEF_KLASS_HANDLE(arrayKlassKlass       , oop_is_arrayKlass    )
DEF_KLASS_HANDLE(objArrayKlassKlass    , oop_is_objArrayKlass )
DEF_KLASS_HANDLE(typeArrayKlassKlass   , oop_is_typeArrayKlass)
DEF_KLASS_HANDLE(arrayKlass            , oop_is_array         )
DEF_KLASS_HANDLE(typeArrayKlass        , oop_is_typeArray_slow)
DEF_KLASS_HANDLE(objArrayKlass         , oop_is_objArray_slow )
DEF_KLASS_HANDLE(constantPoolKlass     , oop_is_constantPool  )
DEF_KLASS_HANDLE(constantPoolCacheKlass, oop_is_constantPool  )
```

> JVM内部，为了让GC便于回收（既能回收一个类实例所对应的实例数据oop，也能回收其对应的元数据和需方法表klass[一个是堆区的垃圾回收，一个是永久区的垃圾回收]），将oop本身鞥装成了oop，而klass也被封装成了oop。

#### oop、klass、handle的转换

##### 1 从oop和klass到handle

> handle主要用于封装组昂oop和klass，因此往往在声明handle类实例的时候，直接将oop或者klass传递进去完成封装。在JVM执行Java类方法时，最终也是通过handle拿到对应的oop和klass，为了高效快速调用，JVM重载了类方法访问操作符->。

Handle的构造函数：

```
inline Handle::Handle(oop obj) {
  if (obj == NULL) {
    _handle = NULL;
  } else {
    _handle = Thread::current()->handle_area()->allocate_handle(obj);
  }
}
```

> 这个构造函数接收oop类型参数，并将其保存到当前线程在堆区申请的handleArea表中，由于oop仅是一种指针，所以表中存储的实际上也是指针。

默认构造函数：

```
  // Constructors
  Handle()                                       { _handle = NULL; }
```

Handle重载操作符->:

```
  oop     non_null_obj() const                   { assert(_handle != NULL, "resolving NULL handle"); return *_handle; }
  oop     operator -> () const                   { return non_null_obj(); }
```

> 治理定义了oop operate ->()操作符重载函数，返回non_null_obj()，而后者直接返回oop类型的*_handle类型，因此，如果JVM想要调用oop的某个函数，可以直接通过handle。

> klass体系的Handle类全部继承与KlassHandle这个基类，与oop体系类似，KlassHandle中也定义了多种构造函数用来实现对klass或oop的封装，并且重载了operate ->()函数实现从handle直接调用klass的函数。JVM创建Java类所对应的类模型时便使用了这种方式。


```
instanceKlassHandle ClassFileParser::parseClassFile(Symbol* name,
                                                    Handle class_loader,
                                                    Handle protection_domain,
                                                    KlassHandle host_klass,
                                                    GrowableArray<Handle>* cp_patches,
                                                    TempNewSymbol& parsed_name,
                                                    bool verify,
                                                    TRAPS){
                                                       // We can now create the basic klassOop for this klass
    klassOop ik = oopFactory::new_instanceKlass(name, vtable_size, itable_size,
                                                static_field_size,
                                                total_oop_map_count,
                                                rt, CHECK_(nullHandle));
    instanceKlassHandle this_klass (THREAD, ik);  
     // Fill in information already parsed
    this_klass->set_access_flags(access_flags);
    this_klass->set_should_verify_class(verify);
    jint lh = Klass::instance_layout_helper(instance_size, false);
    this_klass->set_layout_helper(lh);
    assert(this_klass->oop_is_instance(), "layout is correct");
    assert(this_klass->size_helper() == instance_size, "correct size_helper");
    // Not yet: supers are done below to support the new subtype-checking fields
    //this_klass->set_super(super_klass());
    this_klass->set_class_loader(class_loader());
    this_klass->set_nonstatic_field_size(nonstatic_field_size);
    this_klass->set_has_nonstatic_fields(has_nonstatic_fields);
                                                    }
```

> 在该函数中，先通过klassOop ik = oopFactory::new_instanceKlass();创建了一个klassOopDesc实例，接着通过instanceKlassHandle this_klass (THREAD, ik);  将oop封装到了instanceKlassHandle中，接下来便通过this_klass指针来直接调用instanceKlass中的各种函数。

##### 2 klass和oop的相互转换

> 为了便于GC回收，每一种klass实例最终都要被封装成对应的oop，具体操作是先分配对应的oop实例，然后将klass实例分配到oop对象头后面，从而实现oop+klass这种内存布局结构。对于任何一种给定的oop和其对应的klass，oop对象首地址到其对应的klass对象首地址的距离是固定的，因此只要得到一个便能根据相对寻址找出另一个。对于每一种oop，都提供了klass_part()函数，可以直接由oop得到对应的klass实例。

klass_part()原型：

```
Klass* klass_part() const                      { return (Klass*)((address)this + sizeof(klassOopDesc)); }
```

同样，将klass转换为oop只需要对klass首地址做减法

假设在某个Java方法中连续实例化了三个Student类，那么JVM内存中会出现如下图的布局：
Student类演示Java内存.png

### 常量池klass模型

> 到目前为止，JVM为constantPoolOop所分配的内存区域还是空的，但是，在ClassFileParser:parse_constant_pool()函数中执行完oopFactory::new_constantPool()函数时，已经为constantPoolOop初始化好了_metadata所指向的实例。

```
constantPoolOop oopFactory::new_constantPool(int length,
                                             bool is_conc_safe,
                                             TRAPS) {
  constantPoolKlass* ck = constantPoolKlass::cast(Universe::constantPoolKlassObj());
  return ck->allocate(length, is_conc_safe, CHECK_NULL);
}
```

> constantPoolKlass* ck = constantPoolKlass::cast(Universe::constantPoolKlassObj());这行代码调用全局对象Universe的静态函数constantPoolKlassObj（）来获取constantPoolKlass实例指针，逻辑如下：

```
static klassOop constantPoolKlassObj(){
    return _constatntPoolKlassObj;
}
```

这个函数直接返回了全局静态变量_constantPoolKlassObj，该变量在JVM启动过程中被实例化，JVM实例化过程会调用如下逻辑：


```
klassOop constantPoolKlass::create_klass(TRAPS) {
  constantPoolKlass o;
  KlassHandle h_this_klass(THREAD, Universe::klassKlassObj());
  KlassHandle k = base_create_klass(h_this_klass, header_size(), o.vtbl_value(), CHECK_NULL);
  // Make sure size calculation is right
  assert(k()->size() == align_object_size(header_size()), "wrong size for object");
  java_lang_Class::create_mirror(k, CHECK_NULL); // Allocate mirror
  return k();
}
```

这个函数获取了klassKlass实例，JVM初始化过程中会执行一下逻辑：

```
klassOop klassKlass::create_klass(TRAPS) {
  KlassHandle h_this_klass;
  klassKlass o;
  // for bootstrapping, handles may not be available yet.
  klassOop k = base_create_klass_oop(h_this_klass, header_size(), o.vtbl_value(), CHECK_NULL);
  k->set_klass(k); // point to thyself
  // Do not try to allocate mirror, java.lang.Class not loaded at this point.
  // See Universe::fixup_mirrors()
  return k;
}
```

这个逻辑中最终调用了base_create_klass_oop（）函数创建klassKlass实例，该实例是全局性的，可以通过Universe::klassKlassObj()获取

> 至此，JVM启动过程中，先创建了klassKlass实例，在根据该实例创建了常量池对应的Klass类——constantPoolKlass。所以我们先要分析klassKlass实例的创建。

#### klassKlass实例的构建总链路

klassKlass创建总链路.png
这部分总体分为六个步骤：

- 为klassOop申请内存
- klassOop内存清零
- 初始化klassOop_metadata
- 初始化klass
- 自指

##### 1.为klassOop申请内存

> 从进入klassKlass.cpp::create_klass()之后就一路开始调用层层封装的函数，直到CollectedHeap::permanent_obj_allocate_no_klass_install()才结束，这里开始在永久区为obj分配对象内存，这个函数的逻辑如下：

```
HeapWord* CollectedHeap::common_mem_allocate_init(size_t size, TRAPS) {
  HeapWord* obj = common_mem_allocate_noinit(size, CHECK_NULL);
  init_obj(obj, size);
  return obj;
}
```

当这个函数执行完毕后，JVM的永久区便多了一个对象内存布局，该对象是klassKlassOop，由oop对象头（_mark和_metadata)和klassKlass实例区组成，而现在_metadata指向哪里呢？

##### 2.klassOop内存清零

> 为klassOop分配完内存后，接下来便是将这段内存清零，之所以要清零，是因为JVM永久区是一个可以重复使用的内存区域，会被GC反复回收，因此刚刚申请的内存区域可能还保存着原来已经被清理的对象的数据。

##### 3.初始化mark

> 经过上面两个步骤，接下来就要向内存填充数据了，oop一共包含两个成员变量：_mark和_metadata，这里先填充mark变量，主要通过调用post_allocate_setup_no_klass_install函数实现，主要调用了obj->set_mark(markOopDesc::prototype())函数。声明如下：

```
static markOop prototype() {
    return markOop( no_hash_in_place | no_lock_in_place );
  }
```

在C语言中，有一种快速赋初值的写法，例如

```
int x = 3;
//可以写成
int x(3);
//同样对于指针数据：
int x = 3;
int *p = &x;
//可以写成：
int x = 3;
int *p(&x);
```

> markOop的实际类型是markOopDesc*类型，同样支持上面的写法，所以这个函数最终将返回一个指针，存储了一个整型数字，由于oop._mark成员变量用于存储JVM内部对象的哈希值，线程锁等信息，而此时返回的mark表示则标记当前oop尚未建立唯一哈希值。JVM内部每次读取oop的mark标识时会调用markOopDesc的value函数，定义如下：

```
uintptr_t value() const { return (uintptr_t) this; }
```

> 可以看到，内部其实是将this指针当做值使用，并没有指向markOopDesc实例，它仅仅用于存储JVM内部对象的哈希值、锁状态标识等信息。

##### 4 初始化_metadata

> 初始化玩mark标记后，开始初始化_metadata成员，这是一个联合体，最终是NULL

##### 5.初始化klass

> 现在已经完成了oop对象头初始化，接着开始初始化klassKlass类型实例Klass::base_create_klass_oop，经过这一步后大部分属性值都被赋空。

##### 6.自指

> 执行完上面步骤后，返回到了klassKlass::create_klass()函数中，开始执行k->set_klass(k)这个逻辑，这里的k是klassOop，是klassOopDesc*类型指针，这一步其实是在设置klassOopDesc的_metadata变量，在之前的步骤中，_metadata被设置为NULL，而这一次则是将它指向了自己。

### 常量池klass模型（2）

#### constantPoolKlass模型构建

constantPoolKlass模型构建的过程与上文相同：

- 内存申请
- 内存清零
- 初始化_mark
- 初始化-metadata
- 初始化klass
- 返回包装好的klassOop

> 虽然常量池模型构建与klassKlass构建的基本逻辑一致，但是入参不同，而最后_metadata所指也不同，constantPoolKlass对应的oop的_metadata最终指向前面创建的klass，最终内存布局如下图：

constantPoolKlass内存布局.png

#### constantPoolOop与klass

> 其实constantPoolKlass正式constantPoolOop的类元信息，即constantPoolOop的_metadata指针应该指向constantPoolKlass对象实例，而JVM内部确实也这么做了：

```
constantPoolOop oopFactory::new_constantPool(int length,
                                             bool is_conc_safe,
                                             TRAPS) {
  constantPoolKlass* ck = constantPoolKlass::cast(Universe::constantPoolKlassObj());
  return ck->allocate(length, is_conc_safe, CHECK_NULL);
}
```

> 在这段逻辑中先是获取到了随JVM启动创建的constantPoolKlass实例指针ck，然后调用了ck->allocate()方法，ck指针最终被传递到post_allocation_install_obj_klass()中，因此JVM在执行obj->set_klass()方法时，constantPoolOop的_metadata指针便在此处指向了constantPoolKlass实例对象，最终构建的constantPoolOop内存布局如下图：

constantPoolOop最终布局.png

> 至此constantPoolOopDesc实例的构建便结束了，其他oop实例对象的构建大同小异。

#### klassKlass终结符

> JVM在构建constantPoolOop过程中，由于它本身大小不固定，所以需要使用constantPoolKlass来描述不固定信息，而由于constantPoolKlass也是不固定的，因此会引用klassKlass来描述自己，但如果klassKlass也引用一个klass来描述自身，那么klass链路将一直持续，因此JVM将klassKlass作为整个引用链的终结符，并让他自己指向自己。

## 常量池解析

### constantPoolOop域初始化

在构建constantPoolOop的过程中，会执行constantPoolKlass::allocate()函数，该函数干了三件事：

- 构建constantPoolOop对象实例
- 初始化constantPoolOop实例域变量
- 初始化tag


```
constantPoolOop constantPoolKlass::allocate(int length, bool is_conc_safe, TRAPS) {
  //省略部分代码

  pool->set_length(length);
  pool->set_tags(NULL);
  pool->set_cache(NULL);
  pool->set_operands(NULL);
  pool->set_pool_holder(NULL);
  pool->set_flags(0);
  // only set to non-zero if constant pool is merged by RedefineClasses
  pool->set_orig_length(0);
  // if constant pool may change during RedefineClasses, it is created
  // unsafe for GC concurrent processing.
  pool->set_is_conc_safe(is_conc_safe);
  // all fields are initialized; needed for GC

  //省略部分代码
}
```

### 初始化tag

> 在创建constantPoolOop过程中，会为其_tags域申请内存空间：

```
// initialize tag array
  typeArrayOop t_oop = oopFactory::new_permanent_byteArray(length, CHECK_NULL);
  typeArrayHandle tags (THREAD, t_oop);
  for (int index = 0; index < length; index++) {
    tags()->byte_at_put(index, JVM_CONSTANT_Invalid);
  }
  pool->set_tags(tags());
```

> 可以看到JVM在永久区为tags开辟了与constantPoolOop实例数据长度一致的空间，接下来JVM开始解析字节码文件的常量池元素，并逐个填充到这块区域。

### 解析常量池元素

> 现在开始解析常量池元素，为此专门定义了一个函数：parse_constant_pool_entries()：

```
void ClassFileParser::parse_constant_pool_entries(constantPoolHandle cp, int length, TRAPS) {
  // Use a local copy of ClassFileStream. It helps the C++ compiler to optimize
  // this function (_current can be allocated in a register, with scalar
  // replacement of aggregates). The _current pointer is copied back to
  // stream() when this function returns. DON'T call another method within
  // this method that uses stream().
  ClassFileStream* cfs0 = stream();
  ClassFileStream cfs1 = *cfs0;
  ClassFileStream* cfs = &cfs1;
#ifdef ASSERT
  assert(cfs->allocated_on_stack(),"should be local");
  u1* old_current = cfs0->current();
#endif

  // Used for batching symbol allocations.
  const char* names[SymbolTable::symbol_alloc_batch_size];
  int lengths[SymbolTable::symbol_alloc_batch_size];
  int indices[SymbolTable::symbol_alloc_batch_size];
  unsigned int hashValues[SymbolTable::symbol_alloc_batch_size];
  int names_count = 0;

  // parsing  Index 0 is unused
  for (int index = 1; index < length; index++) {
    // Each of the following case guarantees one more byte in the stream
    // for the following tag or the access_flags following constant pool,
    // so we don't need bounds-check for reading tag.
    u1 tag = cfs->get_u1_fast();                //从字节码文件中读取占1字节宽度的字节流，因为每一个常量池的起始元素的第一个字节用于描述常量池元素类型
    switch (tag) {                              //对不同类型的常量池元素进行处理
      case JVM_CONSTANT_Class :
        {
          cfs->guarantee_more(3, CHECK);  // name_index, tag/access_flags
          u2 name_index = cfs->get_u2_fast();   //获取类的名称索引
          cp->klass_index_at_put(index, name_index);//将当前常量池元素的类型和名称索引分别保存到constantPoolOop的tag数组和数据区
        }
        break;
      case JVM_CONSTANT_Fieldref :
        {
          cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
          u2 class_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          cp->field_at_put(index, class_index, name_and_type_index);
        }
        break;
      case JVM_CONSTANT_Methodref :
        {
          cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
          u2 class_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          cp->method_at_put(index, class_index, name_and_type_index);
        }
        break;
      //省略部分代码
  }
```

>  cp->klass_index_at_put(index, name_index);是将当前常量池元素保存到constantPoolOop中，其实现方式如下：

```
void klass_index_at_put(int which, int name_index) {
    tag_at_put(which, JVM_CONSTANT_ClassIndex);     //将当前常量池元素的类型保存到constantPoolOop所指向的tag对应位置的数组中
    *int_at_addr(which) = name_index;               //将当前常量池元素的名称索引保存到constantPoolOop的数据区中对应的位置
  }
 void tag_at_put(int which, jbyte t)          { tags()->byte_at_put(which, t); }
```

> 当前常量池元素本身在字节码文件常量区位置索引将决定该元素最终在tag数组中的位置，也决定它的索引值最终在constantPoolOop的数据区的位置。

#### 方法元素解析

```
Constant pool:
   #1 = Methodref          #10.#25        // java/lang/Object."<init>":()V
   #2 = Fieldref           #9.#26         // JVM/Iphone6S.length:I
   #3 = Fieldref           #9.#27         // JVM/Iphone6S.width:I
   #4 = Fieldref           #9.#28         // JVM/Iphone6S.height:I
   #5 = Fieldref           #9.#29         // JVM/Iphone6S.weight:I
   #6 = Fieldref           #9.#30         // JVM/Iphone6S.ram:I
   #7 = Fieldref           #9.#31         // JVM/Iphone6S.rom:I
   #8 = Fieldref           #9.#32         // JVM/Iphone6S.pixel:I
   #9 = Class              #33            // JVM/Iphone6S
  #10 = Class              #34            // java/lang/Object
  #11 = Utf8               length
  #12 = Utf8               I
  #13 = Utf8               width
  #14 = Utf8               height
  #15 = Utf8               weight
  #16 = Utf8               ram
  #17 = Utf8               rom
  #18 = Utf8               pixel
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               SourceFile
  #24 = Utf8               Iphone6S.java
  #25 = NameAndType        #19:#20        // "<init>":()V
  #26 = NameAndType        #11:#12        // length:I
  #27 = NameAndType        #13:#12        // width:I
  #28 = NameAndType        #14:#12        // height:I
  #29 = NameAndType        #15:#12        // weight:I
  #30 = NameAndType        #16:#12        // ram:I
  #31 = NameAndType        #17:#12        // rom:I
  #32 = NameAndType        #18:#12        // pixel:I
  #33 = Utf8               JVM/Iphone6S
  #34 = Utf8               java/lang/Object
```

```
enum {
    JVM_CONSTANT_Utf8 = 1,
    JVM_CONSTANT_Unicode,               /* unused */
    JVM_CONSTANT_Integer,
    JVM_CONSTANT_Float,
    JVM_CONSTANT_Long,
    JVM_CONSTANT_Double,
    JVM_CONSTANT_Class,
    JVM_CONSTANT_String,
    JVM_CONSTANT_Fieldref,
    JVM_CONSTANT_Methodref,
    JVM_CONSTANT_InterfaceMethodref,
    JVM_CONSTANT_NameAndType,
    JVM_CONSTANT_MethodHandle           = 15,  // JSR 292
    JVM_CONSTANT_MethodType             = 16,  // JSR 292
    //JVM_CONSTANT_(unused)             = 17,  // JSR 292 early drafts only
    JVM_CONSTANT_InvokeDynamic          = 18,  // JSR 292
    JVM_CONSTANT_ExternalMax            = 18   // Last tag found in classfiles
};
```

> 我们仍然拿上面的字节码作为例子，第一个常量池元素为Methodref，因此tags的#1位置存的值是10，而constantPoolOop的数据区的#1位置有两个索引怎么存储呢？JVM对两个索引值进行了拼接，变成一个值然后存储：


```
  void interface_method_at_put(int which, int class_index, int name_and_type_index) {
    tag_at_put(which, JVM_CONSTANT_InterfaceMethodref);
    *int_at_addr(which) = ((jint) name_and_type_index<<16) | class_index;  // Not so nice
  }
```

> 因此，#10.#25在constantPoolOop的数据区的#1位置实际存的是：0x0A19

#### 字符串元素解析

> 无论是tag还是constantPoolOop的数据区，一个存储位置只能存放一个指针宽度的数据，而字符串往往很大，因此JVM专门设计了一个符号表的内存区，tag和constantPoolOop数据区内仅存指针指向符号区：

```
case JVM_CONSTANT_Utf8 :
        {
          cfs->guarantee_more(2, CHECK);  // utf8_length
          u2  utf8_length = cfs->get_u2_fast();
          u1* utf8_buffer = cfs->get_u1_buffer();
          assert(utf8_buffer != NULL, "null utf8 buffer");
          // Got utf8 string, guarantee utf8_length+1 bytes, set stream position forward.
          cfs->guarantee_more(utf8_length+1, CHECK);  // utf8 string, tag/access_flags
          cfs->skip_u1_fast(utf8_length);

          // Before storing the symbol, make sure it's legal
          if (_need_verify) {
            verify_legal_utf8((unsigned char*)utf8_buffer, utf8_length, CHECK);
          }

          if (EnableInvokeDynamic && has_cp_patch_at(index)) {
            Handle patch = clear_cp_patch_at(index);
            guarantee_property(java_lang_String::is_instance(patch()),
                               "Illegal utf8 patch at %d in class file %s",
                               index, CHECK);
            char* str = java_lang_String::as_utf8_string(patch());
            // (could use java_lang_String::as_symbol instead, but might as well batch them)
            utf8_buffer = (u1*) str;
            utf8_length = (int) strlen(str);
          }

          unsigned int hash;
          Symbol* result = SymbolTable::lookup_only((char*)utf8_buffer, utf8_length, hash);
          if (result == NULL) {
            names[names_count] = (char*)utf8_buffer;
            lengths[names_count] = utf8_length;
            indices[names_count] = index;
            hashValues[names_count++] = hash;
            if (names_count == SymbolTable::symbol_alloc_batch_size) {
              SymbolTable::new_symbols(cp, names_count, names, lengths, indices, hashValues, CHECK);
              names_count = 0;
            }
          } else {
            cp->symbol_at_put(index, result);
          }
        }
        break;
```

> 为了节省内存JVM会先判断符号表中是否存在相同的字符串，如果已经存在则不会分配内存，这就是你在一个类中定义两个字符串但是这两个字符串的值相同，最终都会同时指向常量池中同一个位置的原因。

# 类变量解析

- Java类变量解析的原理
- 计算机基础——偏移量与内存对齐
- Java类与字段的对齐与补白
- Java字段的继承机制
- 使用HSDB查看运行时的Java类结构

## 类变量解析

> 在ClassFileParser::parseClassFile()函数中，解析完常量池、父类和接口后，接着编调用parse_fields()函数解析类变量信息:


```
// Fields (offsets are filled in later)
    FieldAllocationCount fac;
    objArrayHandle fields_annotations;
    typeArrayHandle fields = parse_fields(class_name, cp, access_flags.is_interface(), &fac, &fields_annotations,
                                          &java_fields_count,
                                          CHECK_(nullHandle));
```

FieldAllocationCount的声明如下：

```
class FieldAllocationCount: public ResourceObj {
 public:
  u2 count[MAX_FIELD_ALLOCATION_TYPE];

  FieldAllocationCount() {
    for (int i = 0; i < MAX_FIELD_ALLOCATION_TYPE; i++) {
      count[i] = 0;
    }
  }

  FieldAllocationType update(bool is_static, BasicType type) {
    FieldAllocationType atype = basic_type_to_atype(is_static, type);
    // Make sure there is no overflow with injected fields.
    assert(count[atype] < 0xFFFF, "More than 65535 fields");
    count[atype]++;
    return atype;
  }
};
```

> FieldAllocationCount主要记录了静态类型变量的数量和非静态类型的变量数量，后面JVM为Java类分配内存空间时，会根据这些变量的数量计算所占内存大小。下面我们看JVM对Java类域变量的具体解析逻辑：

```
typeArrayHandle ClassFileParser::parse_fields(Symbol* class_name,
                                              constantPoolHandle cp, bool is_interface,
                                              FieldAllocationCount *fac,
                                              objArrayHandle* fields_annotations,
                                              u2* java_fields_count_ptr, TRAPS) {
  ClassFileStream* cfs = stream();
  typeArrayHandle nullHandle;
  cfs->guarantee_more(2, CHECK_(nullHandle));  // length，获取Java类域变量的数量
  u2 length = cfs->get_u2_fast();
  *java_fields_count_ptr = length;

  int num_injected = 0;
  InjectedField* injected = JavaClasses::get_injected(class_name, &num_injected);

  // Tuples of shorts [access, name index, sig index, initial value index, byte offset, generic signature index]
  typeArrayOop new_fields = oopFactory::new_permanent_shortArray((length + num_injected) * FieldInfo::field_slots, CHECK_(nullHandle));
  typeArrayHandle fields(THREAD, new_fields);

  typeArrayHandle field_annotations;
  for (int n = 0; n < length; n++) {
    cfs->guarantee_more(8, CHECK_(nullHandle));  // access_flags, name_index, descriptor_index, attributes_count

    //读取变量访问表示，如private|public等
    AccessFlags access_flags;
    jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_FIELD_MODIFIERS;
    verify_legal_field_modifiers(flags, is_interface, CHECK_(nullHandle));
    access_flags.set_flags(flags);
    //读取变量名称索引
    u2 name_index = cfs->get_u2_fast();
    int cp_size = cp->length();
    check_property(
      valid_cp_range(name_index, cp_size) && cp->tag_at(name_index).is_utf8(),
      "Invalid constant pool index %u for field name in class file %s",
      name_index, CHECK_(nullHandle));
    Symbol*  name = cp->symbol_at(name_index);
    verify_legal_field_name(name, CHECK_(nullHandle));
    //读取类型索引
    u2 signature_index = cfs->get_u2_fast();
    check_property(
      valid_cp_range(signature_index, cp_size) &&
        cp->tag_at(signature_index).is_utf8(),
      "Invalid constant pool index %u for field signature in class file %s",
      signature_index, CHECK_(nullHandle));
    Symbol*  sig = cp->symbol_at(signature_index);
    verify_legal_field_signature(name, sig, CHECK_(nullHandle));

    u2 constantvalue_index = 0;
    bool is_synthetic = false;
    u2 generic_signature_index = 0;
    bool is_static = access_flags.is_static();
    //读取变量属性
    u2 attributes_count = cfs->get_u2_fast();
    if (attributes_count > 0) {
      parse_field_attributes(cp, attributes_count, is_static, signature_index,
                             &constantvalue_index, &is_synthetic,
                             &generic_signature_index, &field_annotations,
                             CHECK_(nullHandle));
      if (field_annotations.not_null()) {
        if (fields_annotations->is_null()) {
          objArrayOop md = oopFactory::new_system_objArray(length, CHECK_(nullHandle));
          *fields_annotations = objArrayHandle(THREAD, md);
        }
        (*fields_annotations)->obj_at_put(n, field_annotations());
      }
      if (is_synthetic) {
        access_flags.set_is_synthetic();
      }
    }

    FieldInfo* field = FieldInfo::from_field_array(fields(), n);
    field->initialize(access_flags.as_short(),
                      name_index,
                      signature_index,
                      constantvalue_index,
                      generic_signature_index,
                      0);
    //判断变量属性
    BasicType type = cp->basic_type_for_signature_at(signature_index);

    // Remember how many oops we encountered and compute allocation type
    FieldAllocationType atype = fac->update(is_static, type);

    // The correct offset is computed later (all oop fields will be located together)
    // We temporarily store the allocation type in the offset field
    field->set_offset(atype);
  }

  if (num_injected != 0) {
    int index = length;
    for (int n = 0; n < num_injected; n++) {
      // Check for duplicates
      if (injected[n].may_be_java) {
        Symbol* name      = injected[n].name();
        Symbol* signature = injected[n].signature();
        bool duplicate = false;
        for (int i = 0; i < length; i++) {
          FieldInfo* f = FieldInfo::from_field_array(fields(), i);
          if (name      == cp->symbol_at(f->name_index()) &&
              signature == cp->symbol_at(f->signature_index())) {
            // Symbol is desclared in Java so skip this one
            duplicate = true;
            break;
          }
        }
        if (duplicate) {
          // These will be removed from the field array at the end
          continue;
        }
      }

      // Injected field
      FieldInfo* field = FieldInfo::from_field_array(fields(), index);
      field->initialize(JVM_ACC_FIELD_INTERNAL,
                        injected[n].name_index,
                        injected[n].signature_index,
                        0,
                        0,
                        0);

      BasicType type = FieldType::basic_type(injected[n].signature());

      // Remember how many oops we encountered and compute allocation type
      FieldAllocationType atype = fac->update(false, type);

      // The correct offset is computed later (all oop fields will be located together)
      // We temporarily store the allocation type in the offset field
      field->set_offset(atype);
      index++;
    }

    if (index < length + num_injected) {
      // sometimes injected fields already exist in the Java source so
      // the fields array could be too long.  In that case trim the
      // fields array.
      new_fields = oopFactory::new_permanent_shortArray(index * FieldInfo::field_slots, CHECK_(nullHandle));
      for (int i = 0; i < index * FieldInfo::field_slots; i++) {
        new_fields->short_at_put(i, fields->short_at(i));
      }
      fields = new_fields;
    }
  }

  if (_need_verify && length > 1) {
    // Check duplicated fields
    ResourceMark rm(THREAD);
    NameSigHash** names_and_sigs = NEW_RESOURCE_ARRAY_IN_THREAD(
      THREAD, NameSigHash*, HASH_ROW_SIZE);
    initialize_hashtable(names_and_sigs);
    bool dup = false;
    {
      debug_only(No_Safepoint_Verifier nsv;)
      for (AllFieldStream fs(fields, cp); !fs.done(); fs.next()) {
        Symbol* name = fs.name();
        Symbol* sig = fs.signature();
        // If no duplicates, add name/signature in hashtable names_and_sigs.
        if (!put_after_lookup(name, sig, names_and_sigs)) {
          dup = true;
          break;
        }
      }
    }
    if (dup) {
      classfile_parse_error("Duplicate field name&signature in class file %s",
                            CHECK_(nullHandle));
    }
  }

  return fields;
}
```

从上面代码中可以看到JVM解析Java类变量的逻辑：

- 获取Java类中的变量数量
- 读取变量的访问标识
- 读取变量名称索引
- 读取变量类型索引
- 读取变量属性
- 判断变量类型
- 统计各类型数量

> 在Java类所对应的字节码文件中，有专门的一块区域保存Java类的变量信息，字节码文件会一次描述各个变量的访问标识、名称索引、类型索引、和属性，而且都占用2字节，所以只需要依次调用cfs->get_u2_fast()即可。解析完一个变量的属性后，调用fields->short_at_put()，fields是一开始就申请的内存：

```
typeArrayOop new_fields = oopFactory::new_permanent_shortArray((length + num_injected) * FieldInfo::field_slots, CHECK_(nullHandle));
```

## 偏移量

> 解析完字节码文件中Java类的全部域变量信息后，JVM计算出五种数据类型各自的数量，并据此计算各个变量的偏移量。

### 静态变量偏移量

```
instanceKlassHandle ClassFileParser::parseClassFile(Symbol* name,
                                                    Handle class_loader,
                                                    Handle protection_domain,
                                                    KlassHandle host_klass,
                                                    GrowableArray<Handle>* cp_patches,
                                                    TempNewSymbol& parsed_name,
                                                    bool verify,
                                                    TRAPS) {
    //省略部分代码
 // Field size and offset computation
    int nonstatic_field_size = super_klass() == NULL ? 0 : super_klass->nonstatic_field_size();
#ifndef PRODUCT
    int orig_nonstatic_field_size = 0;
#endif
    int static_field_size = 0;
    int next_static_oop_offset;
    int next_static_double_offset;
    int next_static_word_offset;
    int next_static_short_offset;
    int next_static_byte_offset;
    int next_static_type_offset;
    int next_nonstatic_oop_offset;
    int next_nonstatic_double_offset;
    int next_nonstatic_word_offset;
    int next_nonstatic_short_offset;
    int next_nonstatic_byte_offset;
    int next_nonstatic_type_offset;
    int first_nonstatic_oop_offset;
    int first_nonstatic_field_offset;
    int next_nonstatic_field_offset;

    // Calculate the starting byte offsets
    next_static_oop_offset      = instanceMirrorKlass::offset_of_static_fields();
    next_static_double_offset   = next_static_oop_offset +
                                  (fac.count[STATIC_OOP] * heapOopSize);
    if ( fac.count[STATIC_DOUBLE] &&
         (Universe::field_type_should_be_aligned(T_DOUBLE) ||
          Universe::field_type_should_be_aligned(T_LONG)) ) {
      next_static_double_offset = align_size_up(next_static_double_offset, BytesPerLong);
    }

    next_static_word_offset     = next_static_double_offset +
                                  (fac.count[STATIC_DOUBLE] * BytesPerLong);
    next_static_short_offset    = next_static_word_offset +
                                  (fac.count[STATIC_WORD] * BytesPerInt);
    next_static_byte_offset     = next_static_short_offset +
                                  (fac.count[STATIC_SHORT] * BytesPerShort);
    next_static_type_offset     = align_size_up((next_static_byte_offset +
                                  fac.count[STATIC_BYTE] ), wordSize );
    static_field_size           = (next_static_type_offset -
                                  next_static_oop_offset) / wordSize;

    first_nonstatic_field_offset = instanceOopDesc::base_offset_in_bytes() +
                                   nonstatic_field_size * heapOopSize;
    next_nonstatic_field_offset = first_nonstatic_field_offset;
    
    //省略部分代码                                                    
                                                    }
```

> 对于JDK1.6而言，静态变量存储在JVM中所对应的镜像类Mirror中，当Java代码访问静态变量时，最终JVM也是通过设置偏移量进行访问。

### 非静态变量偏移量

非静态类型变量的偏移量计算逻辑可以分为五个步骤：

- 计算非静态变量起始偏移量
- 计算nonstatic_double_offset和nonestatic_oop_offset这2中非静态类型变量的起始偏移
- 计算剩余三种类型变量的起始偏移量
- 计算对齐补白空间
- 计算补白后非静态变量所需要的内存空间总大小

#### 计算非静态变量起始偏移量

> 在JVM内部，使用oop-klass模型描述对象，常量池本身就是这种模型，对于Java类，JVM内部也是这样描述，对应的oop类是instanceOopDesc，对应的klass是instanceKlass。
> **oop模型主要是存储对象实例的实际数据，而klass模型则主要存储对象结构信息和虚函数方法表，就是类的结构和行为。当JVM加载一个Java类时，会首先构建对应的instanceKlass对象，而当new一个Java对象实例时，就会构建出对应的instanceOop对象。**

instanceOop对象主要由Java类的成员变量组成，而这些成员变量在instanceOop中的排列，便由各种变量类型的偏移量决定。在Hotspot内部，oop对象都包含对象头，因此实际上非静态变量的偏移量要从对象头末尾开始计算。

```
 first_nonstatic_field_offset = instanceOopDesc::base_offset_in_bytes() +
                                   nonstatic_field_size * heapOopSize;
```

instanceOopDesc::base_offset_in_bytes()声明如下：

```
static int base_offset_in_bytes() {
    return UseCompressedOops ?
             klass_gap_offset_in_bytes() :
             sizeof(instanceOopDesc);
  }
```

##### 压缩策略

> 如果没有启用压缩策略，在64位平台，oopDesc的值是16，如果启用了压缩策略则是12，因为oop._mark是不会被压缩的，任何时候都占8字节，而oop._klass则会受影响，若开启了压缩策略则它仅会占用4字节，所以oop对象头总共占用12字节内存空间。

##### 继承

> Java类是面向对象的，除了Object类外，其他类都显示或隐式的继承了某个父类，字段继承和方法继承则构成了继承的全部内容，子类必须将分类中所定义的非静态字段信息全部复制一遍，才能实现字段继承的目标，因此计算子类的非静态字段的起始偏移量时必须将父类可悲继承的字段的内存大小考虑在内。Hotspot将父类可悲继承的字段的内存空间安排在子类所对应的oop对象头后，因此最终一个Java类中非静态字段的起始偏移地址在父类被继承的字段域的末尾。

#### 内存对齐与字段重排

- 什么是内存对齐

> 内存对齐与数据在内存中的位置有关，如果一个变量的起始地址正好等于其长度的整数倍，则这种内存分配就称作自然对齐。例如一个int型变量的内存地址是0x00000008，则该变量是自然对齐的。

32位平台的内存对齐规则

| 变量类型 | 长度  | 对齐规则    |
| -------- | ----- | ----------- |
| char     | 1字节 | 按1字节对齐 |
| short    | 2字节 | 按2字节对齐 |
| int      | 4字节 | 按4字节对齐 |

- 为什么要字节对齐

1. 一些平台对某些特定类型的数据只能从某些特定地址开始存取，例如有些架构的CPU在访问一个没有进行对齐的变量时会发生错误。
2. 如果不按要求对数据存放进行对齐，在存取效率上会有损失

> 为了实现内存对齐，JVM不仅使用了补白，还使用了字段重排，将相同类型的字段组合在一起，按照double->word->short->byte-oop的顺序依次分配。

- 计算变量偏移量

> Hotspot将相同类型的字段存储在一起，那么便可以先计算出其内部的5大类型字段的起始偏移量，基于该类型的起始偏移量，便可逐个计算出该类型所对应的每一个具体的Java类字段的偏移量：


```
    //Java类字段的起始偏移量
    first_nonstatic_field_offset = instanceOopDesc::base_offset_in_bytes() +
                                   nonstatic_field_size * heapOopSize;
    next_nonstatic_field_offset = first_nonstatic_field_offset;
    
    //分别获取五种类型字段的数量
    unsigned int nonstatic_double_count = fac.count[NONSTATIC_DOUBLE];
    unsigned int nonstatic_word_count   = fac.count[NONSTATIC_WORD];
    unsigned int nonstatic_short_count  = fac.count[NONSTATIC_SHORT];
    unsigned int nonstatic_byte_count   = fac.count[NONSTATIC_BYTE];
    unsigned int nonstatic_oop_count    = fac.count[NONSTATIC_OOP];
    
    //根据不同的顺序策略，计算起始偏移量
     if( allocation_style < 0 || allocation_style > 2 ) { // Out of range?
      assert(false, "0 <= FieldsAllocationStyle <= 2");
      allocation_style = 1; // Optimistic
    }

    // The next classes have predefined hard-coded fields offsets
    // (see in JavaClasses::compute_hard_coded_offsets()).
    // Use default fields allocation order for them.
    if( (allocation_style != 0 || compact_fields ) && class_loader.is_null() &&
        (class_name == vmSymbols::java_lang_AssertionStatusDirectives() ||
         class_name == vmSymbols::java_lang_Class() ||
         class_name == vmSymbols::java_lang_ClassLoader() ||
         class_name == vmSymbols::java_lang_ref_Reference() ||
         class_name == vmSymbols::java_lang_ref_SoftReference() ||
         class_name == vmSymbols::java_lang_StackTraceElement() ||
         class_name == vmSymbols::java_lang_String() ||
         class_name == vmSymbols::java_lang_Throwable() ||
         class_name == vmSymbols::java_lang_Boolean() ||
         class_name == vmSymbols::java_lang_Character() ||
         class_name == vmSymbols::java_lang_Float() ||
         class_name == vmSymbols::java_lang_Double() ||
         class_name == vmSymbols::java_lang_Byte() ||
         class_name == vmSymbols::java_lang_Short() ||
         class_name == vmSymbols::java_lang_Integer() ||
         class_name == vmSymbols::java_lang_Long())) {
      allocation_style = 0;     // Allocate oops first
      compact_fields   = false; // Don't compact fields
    }

    if( allocation_style == 0 ) {
      // Fields order: oops, longs/doubles, ints, shorts/chars, bytes
      next_nonstatic_oop_offset    = next_nonstatic_field_offset;
      next_nonstatic_double_offset = next_nonstatic_oop_offset +
                                      (nonstatic_oop_count * heapOopSize);
    } else if( allocation_style == 1 ) {
      // Fields order: longs/doubles, ints, shorts/chars, bytes, oops
      next_nonstatic_double_offset = next_nonstatic_field_offset;
    } else if( allocation_style == 2 ) {
      // Fields allocation: oops fields in super and sub classes are together.
      if( nonstatic_field_size > 0 && super_klass() != NULL &&
          super_klass->nonstatic_oop_map_size() > 0 ) {
        int map_count = super_klass->nonstatic_oop_map_count();
        OopMapBlock* first_map = super_klass->start_of_nonstatic_oop_maps();
        OopMapBlock* last_map = first_map + map_count - 1;
        int next_offset = last_map->offset() + (last_map->count() * heapOopSize);
        if (next_offset == next_nonstatic_field_offset) {
          allocation_style = 0;   // allocate oops first
          next_nonstatic_oop_offset    = next_nonstatic_field_offset;
          next_nonstatic_double_offset = next_nonstatic_oop_offset +
                                         (nonstatic_oop_count * heapOopSize);
        }
      }
      if( allocation_style == 2 ) {
        allocation_style = 1;     // allocate oops last
        next_nonstatic_double_offset = next_nonstatic_field_offset;
      }
    } else {
      ShouldNotReachHere();
    }
    
    //计算出实际起始偏移量
    next_nonstatic_word_offset  = next_nonstatic_double_offset +
                                  (nonstatic_double_count * BytesPerLong);
    next_nonstatic_short_offset = next_nonstatic_word_offset +
                                  (nonstatic_word_count * BytesPerInt);
    next_nonstatic_byte_offset  = next_nonstatic_short_offset +
                                  (nonstatic_short_count * BytesPerShort);
```

计算出实际起始偏移量后就开始计算每种类型所对应的Java类中的字段的具体偏移量了。

```
for (AllFieldStream fs(fields, cp); !fs.done(); fs.next()) {
      int real_offset;
      FieldAllocationType atype = (FieldAllocationType) fs.offset();
      switch (atype) {
        case STATIC_OOP:
          real_offset = next_static_oop_offset;
          next_static_oop_offset += heapOopSize;
          break;
        case STATIC_BYTE:
          real_offset = next_static_byte_offset;
          next_static_byte_offset += 1;
          break;
        case STATIC_SHORT:
          real_offset = next_static_short_offset;
          next_static_short_offset += BytesPerShort;
          break;
        case STATIC_WORD:
          real_offset = next_static_word_offset;
          next_static_word_offset += BytesPerInt;
          break;
        case STATIC_DOUBLE:
          real_offset = next_static_double_offset;
          next_static_double_offset += BytesPerLong;
          break;
        case NONSTATIC_OOP:
          if( nonstatic_oop_space_count > 0 ) {
            real_offset = nonstatic_oop_space_offset;
            nonstatic_oop_space_offset += heapOopSize;
            nonstatic_oop_space_count  -= 1;
          } else {
            real_offset = next_nonstatic_oop_offset;
            next_nonstatic_oop_offset += heapOopSize;
          }
          // Update oop maps
          if( nonstatic_oop_map_count > 0 &&
              nonstatic_oop_offsets[nonstatic_oop_map_count - 1] ==
              real_offset -
              int(nonstatic_oop_counts[nonstatic_oop_map_count - 1]) *
              heapOopSize ) {
            // Extend current oop map
            nonstatic_oop_counts[nonstatic_oop_map_count - 1] += 1;
          } else {
            // Create new oop map
            nonstatic_oop_offsets[nonstatic_oop_map_count] = real_offset;
            nonstatic_oop_counts [nonstatic_oop_map_count] = 1;
            nonstatic_oop_map_count += 1;
            if( first_nonstatic_oop_offset == 0 ) { // Undefined
              first_nonstatic_oop_offset = real_offset;
            }
          }
          break;
        case NONSTATIC_BYTE:
          if( nonstatic_byte_space_count > 0 ) {
            real_offset = nonstatic_byte_space_offset;
            nonstatic_byte_space_offset += 1;
            nonstatic_byte_space_count  -= 1;
          } else {
            real_offset = next_nonstatic_byte_offset;
            next_nonstatic_byte_offset += 1;
          }
          break;
        case NONSTATIC_SHORT:
          if( nonstatic_short_space_count > 0 ) {
            real_offset = nonstatic_short_space_offset;
            nonstatic_short_space_offset += BytesPerShort;
            nonstatic_short_space_count  -= 1;
          } else {
            real_offset = next_nonstatic_short_offset;
            next_nonstatic_short_offset += BytesPerShort;
          }
          break;
        case NONSTATIC_WORD:
          if( nonstatic_word_space_count > 0 ) {
            real_offset = nonstatic_word_space_offset;
            nonstatic_word_space_offset += BytesPerInt;
            nonstatic_word_space_count  -= 1;
          } else {
            real_offset = next_nonstatic_word_offset;
            next_nonstatic_word_offset += BytesPerInt;
          }
          break;
        case NONSTATIC_DOUBLE:
          real_offset = next_nonstatic_double_offset;
          next_nonstatic_double_offset += BytesPerLong;
          break;
        default:
          ShouldNotReachHere();
      }
      fs.set_offset(real_offset);
    }
```

#### gap填充

> 在64位平台上，如果开启了指针压缩策略，则对象头仅会占用12个内存单元，这就带来一个问题：它影响了后面的Java类字段的自然对齐效果，为了解决这一问题，JVM将int、short、byte等宽度小于等于4字节的字段往这个内存间隙插入。

```
 if( nonstatic_double_count > 0 ) {
    //填充gap间隙，如果对象头+父类非静态字段的末尾不是8字节对齐，在中间填充Java中非long/double类型的字段
    //gap填充的顺序：int、short、byte、oopmap
      int offset = next_nonstatic_double_offset;
      next_nonstatic_double_offset = align_size_up(offset, BytesPerLong);
      if( compact_fields && offset != next_nonstatic_double_offset ) {
        // Allocate available fields into the gap before double field.
        int length = next_nonstatic_double_offset - offset;
        assert(length == BytesPerInt, "");
        
        //先将int型字段填充进gap
        nonstatic_word_space_offset = offset;
        if( nonstatic_word_count > 0 ) {
          nonstatic_word_count      -= 1;
          nonstatic_word_space_count = 1; // Only one will fit
          length -= BytesPerInt;
          offset += BytesPerInt;
        }
        
        //接着填充short
        nonstatic_short_space_offset = offset;
        while( length >= BytesPerShort && nonstatic_short_count > 0 ) {
          nonstatic_short_count       -= 1;
          nonstatic_short_space_count += 1;
          length -= BytesPerShort;
          offset += BytesPerShort;
        }
        //继续填充byte
        nonstatic_byte_space_offset = offset;
        while( length > 0 && nonstatic_byte_count > 0 ) {
          nonstatic_byte_count       -= 1;
          nonstatic_byte_space_count += 1;
          length -= 1;
        }
        //继续填充oopmap
        // Allocate oop field in the gap if there are no other fields for that.
        nonstatic_oop_space_offset = offset;
        if( length >= heapOopSize && nonstatic_oop_count > 0 &&
            allocation_style != 0 ) { // when oop fields not first
          nonstatic_oop_count      -= 1;
          nonstatic_oop_space_count = 1; // Only one will fit
          length -= heapOopSize;
          offset += heapOopSize;
        }
      }
    }
```

### Java字段内存分配总结

- 任何对象都是以8字节为粒度进行对齐的
- 类属性按照以下优先级排列：long/double-->int/float-->char/short-->byte/bool-->oop
- 不同继承关系中的成员不能混合排列，首先按照规则2处理父类中的成员
- 当父类最后一个属性和子类的第一个属性之间间隔不足4字节时，必须扩展到4字节的基本单位
- 如果子类的哥成员是一个long或double，并且父类没有用完8字节，JVM会破会规则2，按int->short->byte->referenct的顺序向未填满的空间填充

## 从源码看字段继承

### private字段可以被继承吗

- 父类的私有成员变量的确是被子类继承的
- 子类虽然能够继承父类的私有变量，但是却没有权利直接支配，除非通过父类开放的接口

### 使用HSDB验证字段分配与继承

示例代码：

```
public abstract class MyClass {
    private Integer i = 1;
    protected long plong = 12L;
    protected final short s = 6;
    public char c = 'A';
}
public class Test extends MyClass {
    private long l;
    private Integer i = 3;
    private long plong = 18L;
    public char c = 'B';
    public void add(int a, int b){
        Test test = this;
        int z = a+b;
        int x = 3;
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.add(2,3);
        System.exit(0);
    }
}
```

启动HSDB：

```
java -cp .;"C:\Program Files\Java\jdk1.8.0_181\lib/sa-jdi.jar" sun.jvm.hotspot.HSDB
```

universe查看内存分配情况

```
Heap Parameters:
Gen 0:   eden [0x0000000012000000,0x00000000120fed40,0x00000000122b0000) space capacity = 2818048, 37.03897165697674 used
  from [0x00000000122b0000,0x00000000122b0000,0x0000000012300000) space capacity = 327680, 0.0 used
  to   [0x0000000012300000,0x0000000012300000,0x0000000012350000) space capacity = 327680, 0.0 usedInvocations: 0

Gen 1:   old  [0x0000000012350000,0x0000000012350000,0x0000000012a00000) space capacity = 7012352, 0.0 usedInvocations: 0
```

scanoops 0x0000000012000000 0x0000000012a00000 Test在内存中搜索Test类实例

```
0x00000000120f6108 Test
```

inspect 0x00000000120f6108查看这个地址的oop全部数据：

```
instance of Oop for Test @ 0x00000000120f6108 @ 0x00000000120f6108 (size = 72)
_mark: 1
_metadata._klass: InstanceKlass for Test
i: Oop for java/lang/Integer @ 0x0000000012094768 Oop for java/lang/Integer @ 0x0000000012094768
plong: 12
s: 6
c: 'A'
l: 0
i: Oop for java/lang/Integer @ 0x0000000012094798 Oop for java/lang/Integer @ 0x0000000012094798
plong: 18
c: 'B'
```

instanceKlassOop.png
父类的四个字段分别是i,plong,s,c，子类的四个字段分别是I,i,plong,c
classBrowser查看Test类中各个字段偏移量：
Test变量偏移量.png
父类MyClass中各个字段偏移量：
MyClass类成员偏移量.png

> 可以看到，偏移量最小的是plong，偏移量是16正好位于Test类oop对象头之后，然后是s，c，空白填充4字节，然后是Test中的字段域。

> 如果子类中定义了与父类同名同类的字段，无论父类中访问权限是什么，也无论是否被final修饰，子类不会覆盖父类的字段，JVM会在内存中同时为父类和子类的相同字段各分配一段内存空间。

## 总结

Hotspot解析Java类非静态字段和分配堆内存空间的主要逻辑总结为如下几部

- 计息常量池，统计Java类中非静态字段的总数量，按照5大类型分别统计
- 计算Java类字段的起始偏移量，起始偏移位置从父类继承的字段域的末尾开始
- 按照分配策略，计算五大类型中的每一个类型的起始偏移量
- 以五大类型各个类型的起始偏移量为基准，计算每一个大类型下各个具体字段的偏移量
- 计算Java类在堆内存中所需要的内存空间

**当全部解析玩Java类之后，Java类的全部字段信息以及其偏移量会保存到Hotspot所构建出来的instanceKlass中，当Java程序中使用new关键字创建Java类的实例对象时，Hotspot会从instanceKlass中读取Java类所需要的堆内存大小并分配对应的内存空间。

# Java栈帧

- entry_point例程
- 局部变量表创建的机制
- 堆栈与栈帧的概念
- JVM栈帧创建的详细过程
- slot大小到底是多大
- slot复用
- 操作数栈复用与深度

## entry_point例程生成

entry_point创建链路.png
TemplateInterpreter::initialize()函数：

```
void TemplateInterpreter::initialize() {
  if (_code != NULL) return;
  // assertions
  assert((int)Bytecodes::number_of_codes <= (int)DispatchTable::length,
         "dispatch table too small");

  AbstractInterpreter::initialize();

  TemplateTable::initialize();

  // generate interpreter
  { ResourceMark rm;
    TraceTime timer("Interpreter generation", TraceStartupTime);
    int code_size = InterpreterCodeSize;
    NOT_PRODUCT(code_size *= 4;)  // debug uses extra interpreter code space
    _code = new StubQueue(new InterpreterCodeletInterface, code_size, NULL,
                          "Interpreter");
    InterpreterGenerator g(_code);
    if (PrintInterpreter) print();
  }

  // initialize dispatch table
  _active_table = _normal_table;
}
```

> InterpreterGenerator g(_code);创建解释器生成器的实例，在Hotspot内部有三种解释器，分别是字节码解释器、C++解释器和模板解释器。

- 字节码解释器：逐条解释翻译字节码指令
- 模板解释器：将字节码指令直接翻译成机器指令，比较高效

InterpreterGenerator的构造函数：

```
InterpreterGenerator::InterpreterGenerator(StubQueue* code)
 : TemplateInterpreterGenerator(code) {
   generate_all(); // down here so it can be "virtual"
}
```

> generate_all是产生一块解释器运行时所需要的各种例程及入口，对于模板解释器而言就是生成好的机器指令。generate_all中就包含普通Java函数调用所对应的entry_point的入口：

```
void TemplateInterpreterGenerator::generate_all() {
  //省略部分代码

  { CodeletMark cm(_masm, "return entry points");
    for (int i = 0; i < Interpreter::number_of_return_entries; i++) {
      Interpreter::_return_entry[i] =
        EntryPoint(
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(atos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(ltos, i),
          generate_return_entry_for(ftos, i),
          generate_return_entry_for(dtos, i),
          generate_return_entry_for(vtos, i)
        );
    }
  }

  { CodeletMark cm(_masm, "earlyret entry points");
    Interpreter::_earlyret_entry =
      EntryPoint(
        generate_earlyret_entry_for(btos),
        generate_earlyret_entry_for(ztos),
        generate_earlyret_entry_for(ctos),
        generate_earlyret_entry_for(stos),
        generate_earlyret_entry_for(atos),
        generate_earlyret_entry_for(itos),
        generate_earlyret_entry_for(ltos),
        generate_earlyret_entry_for(ftos),
        generate_earlyret_entry_for(dtos),
        generate_earlyret_entry_for(vtos)
      );
  }

  //省略部分代码

#define method_entry(kind)                                                                    \
  { CodeletMark cm(_masm, "method entry point (kind = " #kind ")");                    \
    Interpreter::_entry_table[Interpreter::kind] = generate_method_entry(Interpreter::kind);  \
  }

  // all non-native method kinds
  method_entry(zerolocals)
  method_entry(zerolocals_synchronized)
  method_entry(empty)
  method_entry(accessor)
  method_entry(abstract)
  method_entry(method_handle)
  method_entry(java_lang_math_sin  )
  method_entry(java_lang_math_cos  )
  method_entry(java_lang_math_tan  )
  method_entry(java_lang_math_abs  )
  method_entry(java_lang_math_sqrt )
  method_entry(java_lang_math_log  )
  method_entry(java_lang_math_log10)
  method_entry(java_lang_ref_reference_get)

  // all native method kinds (must be one contiguous block)
  Interpreter::_native_entry_begin = Interpreter::code()->code_end();
  method_entry(native)
  method_entry(native_synchronized)
  Interpreter::_native_entry_end = Interpreter::code()->code_end();

#undef method_entry

  // Bytecodes
  set_entry_points_for_all_bytes();
  set_safepoints_for_all_bytes();
}
```

> 这个方法将生成模板解释器对应的各种模板例程的机器指令，并保存入口地址，它的上半段定义了一些重要的逻辑入口，同时会生成其对应的机器指令，而从#define method_entry()开始则定义了一系列方法入口。

> 当JVM调用Java函数时，例如Java类的构造函数、类成员方法、静态方法、虚方法等，就会从不同的入口进去，在CallStub例程中进入不同的函数入口。对于正常的java方法调用（包括主函数），其所对应的entry_point一般都是zerolocals或zerolocals_synchronized，如果方法加了同步关键字synchronize，则其entry_point是zerolocals_synchronized。调用method_entry(zerolocals)就相当于执行了一下逻辑：

```
Interpreter::_entry_table[Interpreter::zerolocals] = generate_method_entry(Interpreter::zerolocals);
```

> 这个逻辑执行完后，JVM会为zerolocals生成本地机器指令，同时将这串机器指令的首地址保存到Interpreter::_entry_table数组中。为zerolocals方法入口生成机器指令的函数如下：

```
address AbstractInterpreterGenerator::generate_method_entry(AbstractInterpreter::MethodKind kind) {
  // determine code generation flags
  bool synchronized = false;
  address entry_point = NULL;

  switch (kind) {
    case Interpreter::zerolocals             :                                                                             break;
    case Interpreter::zerolocals_synchronized: synchronized = true;                                                        break;
    case Interpreter::native                 : entry_point = ((InterpreterGenerator*)this)->generate_native_entry(false);  break;
    case Interpreter::native_synchronized    : entry_point = ((InterpreterGenerator*)this)->generate_native_entry(true);   break;
    case Interpreter::empty                  : entry_point = ((InterpreterGenerator*)this)->generate_empty_entry();        break;
    case Interpreter::accessor               : entry_point = ((InterpreterGenerator*)this)->generate_accessor_entry();     break;
    case Interpreter::abstract               : entry_point = ((InterpreterGenerator*)this)->generate_abstract_entry();     break;
    case Interpreter::method_handle          : entry_point = ((InterpreterGenerator*)this)->generate_method_handle_entry(); break;

    case Interpreter::java_lang_math_sin     : // fall thru
    case Interpreter::java_lang_math_cos     : // fall thru
    case Interpreter::java_lang_math_tan     : // fall thru
    case Interpreter::java_lang_math_abs     : // fall thru
    case Interpreter::java_lang_math_log     : // fall thru
    case Interpreter::java_lang_math_log10   : // fall thru
    case Interpreter::java_lang_math_sqrt    : entry_point = ((InterpreterGenerator*)this)->generate_math_entry(kind);     break;
    case Interpreter::java_lang_ref_reference_get
                                             : entry_point = ((InterpreterGenerator*)this)->generate_Reference_get_entry(); break;
    default                                  : ShouldNotReachHere();                                                       break;
  }

  if (entry_point) return entry_point;

  return ((InterpreterGenerator*)this)->generate_normal_entry(synchronized);

}
```

当入参类型是zerolocals时直接执行最后一句，generate_normal_entry就开始为zerolocals生成本地机器指令：

```
address InterpreterGenerator::generate_normal_entry(bool synchronized) {
  // determine code generation flags
  bool inc_counter  = UseCompiler || CountCompiledCalls;

  // rbx,: methodOop
  // rsi: sender sp
  address entry_point = __ pc();

  //定义寄存器变量
  const Address size_of_parameters(rbx, methodOopDesc::size_of_parameters_offset());
  const Address size_of_locals    (rbx, methodOopDesc::size_of_locals_offset());
  const Address invocation_counter(rbx, methodOopDesc::invocation_counter_offset() + InvocationCounter::counter_offset());
  const Address access_flags      (rbx, methodOopDesc::access_flags_offset());

  // get parameter size (always needed)入参数量
  __ load_unsigned_short(rcx, size_of_parameters);

  // rbx,: methodOop
  // rcx: size of parameters

  // rsi: sender_sp (could differ from sp+wordSize if we were called via c2i )
    
  __ load_unsigned_short(rdx, size_of_locals);       // get size of locals in words
  __ subl(rdx, rcx);                                // rdx = no. of additional locals

  // see if we've got enough room on the stack for locals plus overhead.
  generate_stack_overflow_check();

  // get return address获取返回地址
  __ pop(rax);

  // compute beginning of parameters (rdi)计算第一个入参在堆栈中的地址
  __ lea(rdi, Address(rsp, rcx, Interpreter::stackElementScale(), -wordSize));

  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals 为局部变量slot（不包含入参）分配堆栈空间，初始化为0
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }

  if (inc_counter) __ movl(rcx, invocation_counter);  // (pre-)fetch invocation count
  // initialize fixed part of activation frame 创建栈帧
  generate_fixed_frame(false);

  // make sure method is not native & not abstract
#ifdef ASSERT
  __ movl(rax, access_flags);
  {
    Label L;
    __ testl(rax, JVM_ACC_NATIVE);
    __ jcc(Assembler::zero, L);
    __ stop("tried to execute native method as non-native");
    __ bind(L);
  }
  { Label L;
    __ testl(rax, JVM_ACC_ABSTRACT);
    __ jcc(Assembler::zero, L);
    __ stop("tried to execute abstract method in interpreter");
    __ bind(L);
  }
#endif

  // Since at this point in the method invocation the exception handler
  // would try to exit the monitor of synchronized methods which hasn't
  // been entered yet, we set the thread local variable
  // _do_not_unlock_if_synchronized to true. The remove_activation will
  // check this flag.

  __ get_thread(rax);
  const Address do_not_unlock_if_synchronized(rax,
        in_bytes(JavaThread::do_not_unlock_if_synchronized_offset()));
  __ movbool(do_not_unlock_if_synchronized, true);

  // increment invocation count & check for overflow 引用计数
  Label invocation_counter_overflow;
  Label profile_method;
  Label profile_method_continue;
  if (inc_counter) {
    generate_counter_incr(&invocation_counter_overflow, &profile_method, &profile_method_continue);
    if (ProfileInterpreter) {
      __ bind(profile_method_continue);
    }
  }
  Label continue_after_compile;
  __ bind(continue_after_compile);

  bang_stack_shadow_pages(false);

  // reset the _do_not_unlock_if_synchronized flag
  __ get_thread(rax);
  __ movbool(do_not_unlock_if_synchronized, false);

  // check for synchronized methods
  // Must happen AFTER invocation_counter check and stack overflow check,
  // so method is not locked if overflows.
  //
  if (synchronized) {
    // Allocate monitor and lock method
    lock_method();
  } else {
    // no synchronization necessary
#ifdef ASSERT       //开始执行Java方法第一条字节码
      { Label L;
        __ movl(rax, access_flags);
        __ testl(rax, JVM_ACC_SYNCHRONIZED);
        __ jcc(Assembler::zero, L);
        __ stop("method needs synchronization");
        __ bind(L);
      }
#endif
  }

  // start execution
#ifdef ASSERT
  { Label L;
     const Address monitor_block_top (rbp,
                 frame::interpreter_frame_monitor_block_top_offset * wordSize);
    __ movptr(rax, monitor_block_top);
    __ cmpptr(rax, rsp);
    __ jcc(Assembler::equal, L);
    __ stop("broken stack frame setup in interpreter");
    __ bind(L);
  }
#endif

  // jvmti support
  __ notify_method_entry();
    //进入Java方法第一条字节码
  __ dispatch_next(vtos);

  // invocation counter overflow
  if (inc_counter) {
    if (ProfileInterpreter) {
      // We have decided to profile this method in the interpreter
      __ bind(profile_method);
      __ call_VM(noreg, CAST_FROM_FN_PTR(address, InterpreterRuntime::profile_method));
      __ set_method_data_pointer_for_bcp();
      __ get_method(rbx);
      __ jmp(profile_method_continue);
    }
    // Handle overflow of counter and compile method
    __ bind(invocation_counter_overflow);
    generate_counter_overflow(&continue_after_compile);
  }

  return entry_point;
}
```

> 这个函数会在JVM启动过程中调用，执行完成后，会向JVM的代码缓存区写入对应的本地机器指令，当JVM调用一个特定的Java方法时，会根据Java方法所对应的类型找到对应的函数入口，并执行这段预先生成好的机器指令。

## 局部变量表创建

### constMethod的内存布局

对于JDK6，JVM内部通过偏移量为Java函数定义了以下规则：

- method对象的constMethod指针紧跟在methodOop对象头后面，也即constMethod的偏移量固定
- constMethod内部存储Java函数所对应的字节码指令的位置相对于constMethod起始位的偏移量固定
- method对象内部存储Java函数的参数数量、局部变量数量的参数的偏移量是固定的。

JDK6的method机器相关属性的偏移量：
methodOopDesc内存布局.png

> max_locals与size_of_parameters这两个参数相对于methodOop对象头的偏移量分别是38与36，JVM将会基于此计算局部变量表的大小。JDK8中，由于这两个参数恒定不变，因此将其当做只读属性，移到了constMethod对象中。

### 局部变量表空间计算

> 局部变量表包含Java方法的所有入参和方法内部所声明的全部局部变量。
> 举例说明

```
public class A {
    public void add(int x,int y){
        int z = x + y;
    }
}
```

使用javap -verbose分析：

```
  public void add(int, int);
    descriptor: (II)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: istore_3
         4: return
      LineNumberTable:
        line 5: 0
        line 6: 4
```

> 可以看到add方法的局部变量表的最大容量是4，入参数量是3，这是因为add方法是类的成员方法，因此会有一个隐藏的第一个入参this。3个入参加上内部定义的局部变量z构成了局部变量表。


```
0: iload_1
1: iload_2
这两条字节码指令分别将局部变量表的第一个槽位x和第二个槽位y的数据推送至表达式栈栈顶，
（槽位标号从0开始，很显然第0号槽位是this）
istore_3
这条字节码指令将计算结果板寸到局部变量表的第三个槽位，正是定义的局部变量z
```

> 对于本例，x和y这两个入参在调用方法调用add时便已经分配完毕，因此JVM只需为z分配堆栈空间，而z所需要的堆栈空间大小就是编译期间计算出的局部变量表的大小减去入参数量。entry_point隔出了这种方法：

```
 const Address size_of_parameters(rbx, methodOopDesc::size_of_parameters_offset());
  const Address size_of_locals    (rbx, methodOopDesc::size_of_locals_offset());
  const Address invocation_counter(rbx, methodOopDesc::invocation_counter_offset() + InvocationCounter::counter_offset());
  const Address access_flags      (rbx, methodOopDesc::access_flags_offset());

  // get parameter size (always needed)
  __ load_unsigned_short(rcx, size_of_parameters);

  // rbx,: methodOop
  // rcx: size of parameters

  // rsi: sender_sp (could differ from sp+wordSize if we were called via c2i )

  __ load_unsigned_short(rdx, size_of_locals);       // get size of locals in words
  __ subl(rdx, rcx);                                // rdx = no. of additional locals
```

这段代码最终生成的机器指令是：

```
movl 0x26(%ebx),%ecx
movl 0x24(%ebx),%edx
sub %ecx,%edx
```

### 初始化局部变量区

在CallStub执行call %eax指令前，物理寄存器中所保存的重要信息如下：

| 寄存器名 | 指向                                         |
| -------- | -------------------------------------------- |
| edx      | parameters首地址                             |
| ecx      | Java函数入参数量                             |
| ebx      | 指向Java函数，即Java函数所对应的的method对象 |
| esi      | CallStub栈顶                                 |

在entry_point历程中，分配堆栈空间时进行push操作：

```
// get return address获取返回地址
  __ pop(rax);

  // compute beginning of parameters (rdi)计算第一个入参在堆栈中的地址
  __ lea(rdi, Address(rsp, rcx, Interpreter::stackElementScale(), -wordSize)); 
  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals 为局部变量slot（不包含入参）分配堆栈空间，初始化为0
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }
```

这段逻辑执行了三件事：

- 将栈顶的返回值暂存到rax寄存器
- 获取Java函数第一个入参在堆栈的位置
- 为局部变量表分配堆栈空间

#### 1 暂存返回地址

> JVM为了实现操作数栈与局部变量表的复用，要将接下来被调用的Java方法分配局部变量的堆栈空间与被调用的Java方法的入参区域连城一块，而中间又间隔了return address，所以要将他暂存起来（pop %eax）

#### 2 获取Java函数第一个入参在堆栈的位置

> 这一步将第一个入参的位置保存在edi寄存器中，因为JVM是基于局部变量表的起始位置做偏移的。

#### 3 为局部变量表分配堆栈空间

```
{
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }
```

> 这一段指令的逻辑是，先测试edx是否为0，如果为0则没有定义局部变量，直接跳过分配堆栈步骤，否则的话执行循环，通过__ push((int32_t)NULL_WORD);                      // initialize local variables向栈顶压入一个0，然后edx-1，一直进行到edx为0。这么做的原因是可以在分配堆栈空间的同时，将堆栈清零。

## 堆栈与栈帧

> 为局部变量分配完堆栈空间后，接着就要构建Java方法所对应的栈帧中的固定部分（fixed frame）。在JVM内部，每个Java方法对应一个栈帧，这个栈帧包括局部变量表、操作数栈、常量池缓存指针、返回地址等。对于一个Java方法栈帧，其首位分别是局部变量表和操作数栈，而中间的部分则是其他重要数据，正是这部分的数据结构是固定不变的。

### 栈帧是什么

> 一个方法的栈帧就是这个方法对应的堆栈。

- 堆栈是什么

> 函数内不会定义局部变量，这些变量最终要在内存中占用一定的空间。由于这些变量都位于同一个函数，自然地就要将这些变量合起来当做一个整体，将其内存空间分配到一起，这样有利于变量空间的整体内存申请和释放。所以“栈帧”就是一个容器，而堆栈就是栈帧的容器。

- 堆栈有什么用

> 可以节省空间，并有利于CPU的寻址，如果不采用线性布局的堆栈空间，被调用函数的堆栈空间地址保存在内存中，而访问内存的效率要比寄存器直接传递数据的效率低得多。

# Java数据结构与面向对象

- 数据结构的基本含义
- Java数据结构的实现机制
- Java数据结构的字节码格式分析
- 大端与小端

### 数据结构与算法

> 编程就是使用合适的算法处理特定的数据结构，程序就是算法与数据结构的有机结合。Java程序的算法由Java字节码指令所驱动，而数据结构往往会作为算法的输入输出和中间产出，及时输出的是一个简单的数字也是一种数据结构。

- 在Java中不仅将程序算法“字节码”化，连同数据结构一起被“字节码”化。
- Java的数据结构接触了对物理机器的依赖，但是C中定义的结构体被转换成机器指令后，他的类型信息会被彻底抹去，变得不可理解。

### Java类型识别

> Java类在编译器生成的字节码有其特定的组织规律，Java虚拟机在加载类是，对于其生成的字节码信息按照固定的格式进行解析，可以解析出字节码中所存储的类型的结构信息，从而在运行期完全还原出原始的Java类的全部结构。

### Java字节码概述

- class文件的十个组成结构
  * MagicNumber
  * Version
  * Constant-pool
  * Access_flag
  * This_class
  * Super_class
  * Interfaces
  * Fileds
  * Methods
  * Attributes

简单说明

- MagicNumber：用来标识class文件，位于每一个Java class文件的最前面四个字节，值固定为0xCAFEBABE，虚拟机加载class文件时会先检查这四个字节，如果不是这个值则拒绝加载
- Version：由2个长度为2字节的字段组成，分别为MagicVersion和Minor Version个，代表当前class文件的主版本号和次版本号。高版本的JVM可以兼容低版本的
- 常量池：从第九个字节开始，首先是2字节的长度字段constant_pool_count说明常量池包含多少常量，接下来的二进制信息描述这些常量，常量池里放的是字面常量和符号引用

> 字面常量主要包含文本串以及被声明为final的常量，符号引用包好类的接口和全局限定名、字段的名称和描述符、方法的名称和描述符，因为Java语言在编译时没有连接这一步，所有的引用都是运行时动态加载的，所以要把这些引用存在class文件里。符号引用保存的是引用的全局限定名，所以保存的是字符串。
> 字面常量分为字符串、整形、长整形、浮点型、双精度浮点型

- Access_flag:保存当前类的访问权限
- This_class：保存当前类的全局限定名在常量池的索引
- Super_class: 保存当前类的父类的全局限定名在常量池的索引
- Interfaces：主要保存当前类实现的接口列表，包含interfaces_count和interfaces[interfaces_count]
- Fields:主要保存当前类的成员列表：包含fields_count和fields[files_count]
- Methods：保存当前类的方法列表，包含methods_count和method[methods_count]
- Attributes：主要保存当前类attributes列表，包含attributes_count和attributes[attributes_count]

#### JVM内部的int类型

- 1字节：uint8_t,JVM:u1
- 2字节: uint16_t,JVM:u2
- 4字节：uint32_t,JVM:u4
- 8字节：uint64_t,JVM:u8

#### 常量池与JVM内部对象模型

> 常量池是Java字节码文件中比较重要的概念，是整个Java类的核心所在，他记录了一个Java类的所有成员变量，成员方法和静态变量与静态方法、构造函数等全部信息，包括变量名、方法名、访问表示、类型信息等。

#### OOP-KLASS二分模型

> KLASS用于保存类元信息，而OOP用于表示JVM所创建的类实例对象，KLASS信息被保存在perm永久区，而oop则被分配在heap堆区，同时JVM为了支持反射等技术，必须在OOP中保存一个指针，用于指向其所属的类型KLASS，这样Java开发者便可以基于反射技术，在Java程序运行期获取Java的类型信息。

HotSpot中的oop体系

```
typedef class oopDesc*                            oop;
typedef class   instanceOopDesc*            instanceOop;
typedef class   methodOopDesc*                    methodOop;
typedef class   constMethodOopDesc*            constMethodOop;
typedef class   methodDataOopDesc*            methodDataOop;
typedef class   arrayOopDesc*                    arrayOop;
typedef class     objArrayOopDesc*            objArrayOop;
typedef class     typeArrayOopDesc*            typeArrayOop;
typedef class   constantPoolOopDesc*            constantPoolOop;
typedef class   constantPoolCacheOopDesc*   constantPoolCacheOop;
typedef class   klassOopDesc*                    klassOop;
typedef class   markOopDesc*                    markOop;
typedef class   compiledICHolderOopDesc*    compiledICHolderOop;
```

Hotspot使用klass来描述Java类型信息，定义了如下几种

```
class Klass;
class   instanceKlass;
class     instanceMirrorKlass;
class     instanceRefKlass;
class   methodKlass;
class   constMethodKlass;
class   methodDataKlass;
class   klassKlass;
class     instanceKlassKlass;
class     arrayKlassKlass;
class       objArrayKlassKlass;
class       typeArrayKlassKlass;
class   arrayKlass;
class     objArrayKlass;
class     typeArrayKlass;
class   constantPoolKlass;
class   constantPoolCacheKlass;
class   compiledICHolderKlass;
```

#### oopDesc

> C++中的instanceOopDesc等类最终全部来自顶级父类oopDesc，Java的面向对象和运行期反射的能力便是由其予以提现和支撑。

```
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
```

> 抛开友元类VMStructs以及用于内存屏障的_bs，就只剩下了两个成员变量，_mark和_metadata。

- Java类在整个生命周期中，会涉及到线程状态、并发所、GC分代信息等内部表示，这些标识全部打在_mark变量上。
- _metadata用于表示元数据，也就是前面所说的数据结构，起到指针的作用，指向Java类的数据结构被解析后所保存的内存位置。前面所说的instanceOop内部会有一个指针指向instanceKlass便是_metadata，klass是Java类的对等题。

### 大端与小端

- Little-Endian小端：低位字节排放在内存的低位地址
- Big-Endian大端：高位字节排放在内存的低位地址。
- 网络字节序，TCP/IP协议中使用的字节序通常称为网络字节序，使用Big-Endian

大小端产生的本质原因是由于CPU的标准不同，假定一个双字节的寄存器，如果他的左边为高字节端，那么0x0102在寄存器所存的数值就是0x0102，而如果左边是低字节的话就成了0x0201，并最终在内存的存储顺序上反应出来。

#### 大小端问题的避免

- 在单机上使用同一种编程语言读写变量，文件，进行网络通信，所读取的字节与所写入字节序相同。
- 在分布式场景下，使用同一种编程语言，在同样大小端模式的不同机器上所读写的文件与网络信息，字节序相同。

> 然而Java的跨平台特性让它不一定能满足上面所说的，那么JVM是怎么处理大小端问题呢。

- Java所输出的字节信息全部是大端模式，对于小端模式的硬件平台，JVM定义了响应的转换接口。

# HotSpot源代码

最近想看看JDK8源码，但JDK中JVM（安装在本地C:\Program Files\Java\jdk1.8.0_121\jre\bin\server下jvm.dll）本身并不开源，只能找来openJDK来看（说是和JDK相似度很高）。源码包下载飞机票：http://download.java.net/openjdk/jdk8 

```
openjdk
—— corba：不流行的多语言、分布式通讯接口
—— hotspot：Java 虚拟机
—— jaxp：XML 处理
—— jaxws：一组 XML web services 的 [Java ](http://lib.csdn.net/base/java)API
—— jdk：java 开发工具包
—— —— 针对[操作系统](http://lib.csdn.net/base/operatingsystem)的部分
—— —— share：与平台无关的实现
—— langtools：Java 语言工具
—— nashorn：JVM 上的 [JavaScript](http://lib.csdn.net/base/javascript) 运行时
```

其中Hotspot就是我们要看的JVM包

Hotspot：全称 Java HotSpot Performance Engine，是 Java 虚拟机的一个实现，包含了服务器版和桌面应用程序版。利用 JIT 及自适应优化技术（自动查找性能热点并进行动态优化）来提高性能。

使用 `java -version` 可以查看 Hotspot 的版本。

## HotSpot VM源码结构

HotSpot VM目录下主要由agent、make、src和test这4个子目录构成。

其中agent目录下包含了Serviceability Agent的客户端实现；

make目录下包含了用于编译HotSpot VM的各种配置文件；

src目录是最重要的一个目录，本书讲解的所有源代码都包含在这个目录下；

test目录下包含HotSpot相关的一些单元测试用例。

src目录下包含了HotSpot项目的主体源代码，主要由cpu、os、os_cpu与share这4个子目录构成。下面简单介绍一下这4个子目录，如下：

- cpu目录：此目录下包含一些依赖具体处理器架构的代码，现在主流的处理器架构主要有sparc、x86和zero，其中x86又最为常见，所以在涉及到相关源代码实现时，只会讲解x86目录下的源代码实现。
- os目录：此目录下包含一些依赖操作系统的代码，主要的操作系统有基于linux内核的操作系统、基于unix的操作系统（posix）、windows和solaris。在涉及到相关源代码实现时，只会讲解linux目录下的源代码实现。
- os_cpu目录：此目录下包含一些依赖操作系统和处理器架构的代码，如linux_x86目录。linux_x86目录下包含的就是基于linux内核的操作系统和x86处理器架构相关的代码，也是要讲解的源代码实现。
- share目录：此目录下包含了独立于操作系统和处理器架构的代码，尤其是vm子目录，这个子目录中包含的内容比较多，HotSpot主要的一些功能模块都包含在这个子目录下。下表介绍了vm目录下一些重要子目录的功能。

### 源码目录结构

| 目录              | 描述                      |
| ----------------- | ------------------------- |
| adlc              | 平台描述文件              |
| asm               | 汇编器                    |
| c1                | C1编译器，即client编译器  |
| ci                | 动态编译器                |
| classfile         | Class文件解析和类的链接等 |
| code              | 机器码生成                |
| compiler          | 调用动态编译器的接口      |
| Opto              | C2编译器，即server编译器  |
| gc_interface      | GC接口                    |
| gc_implementation | 垃圾收集器的具体实现      |
| interpreter       | 解释器                    |
| libadt            | 抽象数据结构              |
| memory            | 内存管理                  |
| oops              | JVM内部对象表示           |
| prims             | HotSpot对外接口           |
| runtime           | 运行时                    |
| services          | JMX接口                   |
| utilizes          | 内部工具类和公共函数      |

JVM系列主要以HotSpot为主，按HotSpot各模块来一一展开，包括Class类文件成分,字节码文件加载及解析，HotSpot内存结构，JVM解析器，Kclass-Oop模型，垃圾回收器，执行引擎，字节码指令系统，JIT, AOT, 逃逸分析，栈顶缓存等等。

```
├─agent                            Serviceability Agent的客户端实现
├─make                             用来build出HotSpot的各种配置文件
├─src                              HotSpot VM的源代码
│  ├─cpu                            CPU相关代码（汇编器、模板解释器、ad文件、部分runtime函数在这里实现）
│  ├─os                             操作系相关代码
│  ├─os_cpu                         操作系统+CPU的组合相关的代码
│  └─share                          平台无关的共通代码
│      ├─tools                        工具
│      │  ├─hsdis                      反汇编插件
│      │  ├─IdealGraphVisualizer       将server编译器的中间代码可视化的工具
│      │  ├─launcher                   启动程序“java”
│      │  ├─LogCompilation             将-XX:+LogCompilation输出的日志（hotspot.log）整理成更容易阅读的格式的工具
│      │  └─ProjectCreator             生成Visual Studio的project文件的工具
│      └─vm                           HotSpot VM的核心代码
│          ├─adlc                       平台描述文件（上面的cpu或os_cpu里的*.ad文件）的编译器
│          ├─asm                        汇编器接口
│          ├─c1                         client编译器（又称“C1”）
│          ├─ci                         动态编译器的公共服务/从动态编译器到VM的接口
│          ├─classfile                  类文件的处理（包括类加载和系统符号表等）
│          ├─code                       动态生成的代码的管理
│          ├─compiler                   从VM调用动态编译器的接口
│          ├─gc_implementation          GC的实现
│          │  ├─concurrentMarkSweep      Concurrent Mark Sweep GC的实现
│          │  ├─g1                       Garbage-First GC的实现（不使用老的分代式GC框架）
│          │  ├─parallelScavenge         ParallelScavenge GC的实现（server VM默认，不使用老的分代式GC框架）
│          │  ├─parNew                   ParNew GC的实现
│          │  └─shared                   GC的共通实现
│          ├─gc_interface               GC的接口
│          ├─interpreter                解释器，包括“模板解释器”（官方版在用）和“C++解释器”（官方版不在用）
│          ├─libadt                     一些抽象数据结构
│          ├─memory                     内存管理相关（老的分代式GC框架也在这里）
│          ├─oops                       HotSpot VM的对象系统的实现
│          ├─opto                       server编译器（又称“C2”或“Opto”）
│          ├─prims                      HotSpot VM的对外接口，包括部分标准库的native部分和JVMTI实现
│          ├─runtime                    运行时支持库（包括线程管理、编译器调度、锁、反射等）
│          ├─services                   主要是用来支持JMX之类的管理功能的接口
│          ├─shark                      基于LLVM的JIT编译器（官方版里没有使用）
│          └─utilities                  一些基本的工具类
└─test                             单元测试
```

## HotSpot的组成结构

JVM(全称Java Virtual Machine), 是一台虚拟的计算机，本身不跨平台。是Java，Python等编程语言生成的字节码文件的执行平台。具体的JVM可以由JRockit，HotSpot 等多种实现。但是需要遵循JVM规范，以实现语言的跨平台兼容性。最新的graalvm是一个多语言通用虚拟机。

HotSpot的组成包括：

- 类加载器子系统：负责加载二进制字节码文件流。
- 内存模型：Java栈，本地方法栈，Java堆，方法区，PC寄存器，直接内存
- 垃圾收集器：主要负责Java堆空间内存回收。
- 类文件解析器：解析二进制流字节码文件为对象模型
- 对象模型：虚拟机内部klass-Oop模型
- 解释器：虚拟机内部字节码执行单元，包括字节码解释器，模板解释器
- 编译器：虚拟机内置编译器，字节码指令的动态优化，包括JIT, AOT
- 监控: 虚拟机对外提供的运行时统计监控
- 运行时：虚拟机运行时环境
- 服务：虚拟机内部服务，包括内存，线程，运行时，类加载服务
- os模块：负责与具体的寄生系统交互

## HotSpot 启动流程

学习启动过程可以帮助我们了解程序的入口，并对虚拟机的运行有个整体的把握，方便日后深入学习具体的一些模块。

### 整体感知启动过程

整体的感知启动过程可以在启动时添加`_JAVA_LAUNCHER_DEBUG=1`的环境变量。这样JVM会输出详细的打印。
通过这些打印，我们大致能了解到启动过程发生了什么。

```
----_JAVA_LAUNCHER_DEBUG----
Launcher state:
	debug:on
	javargs:off
	program name:java
	launcher name:openjdk
	javaw:off
	fullversion:1.8.0-internal-debug-xieshang_2020_12_18_09_49-b00
	dotversion:1.8
	ergo_policy:DEFAULT_ERGONOMICS_POLICY
Command line args:
argv[0] = /home/xieshang/learn-jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java
argv[1] = com.insanexs/HelloHotspot
JRE path is /home/xieshang/learn-jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk
jvm.cfg[0] = ->-server<-
jvm.cfg[1] = ->-client<-
1 micro seconds to parse jvm.cfg
Default VM: server
Does `/home/xieshang/learn-jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/server/libjvm.so' exist ... yes.
mustsetenv: FALSE
JVM path is /home/xieshang/learn-jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/server/libjvm.so
1 micro seconds to LoadJavaVM
JavaVM args:
    version 0x00010002, ignoreUnrecognized is JNI_FALSE, nOptions is 5
    option[ 0] = '-Dsun.java.launcher.diag=true'
    option[ 1] = '-Djava.class.path=/home/xieshang/learn-open-jdk'
    option[ 2] = '-Dsun.java.command=com.insanexs/HelloHotspot'
    option[ 3] = '-Dsun.java.launcher=SUN_STANDARD'
    option[ 4] = '-Dsun.java.launcher.pid=4485'
1 micro seconds to InitializeJVM
Main class is 'com.insanexs/HelloHotspot'
App's argc is 0
1 micro seconds to load main class
----_JAVA_LAUNCHER_DEBUG----
```

从上面的打印大致可以看出有这么几步：

1. 打印了启动器的状态，包括版本号、程序名等
2. 打印了传给程序命令行参数，第一个是java命令的相信路径，第二个虚拟机将要执行的java代码
3. 解析JRE路径，解析jvm.cfg
4. 加载libjvm库
5. 解析虚拟机参数
6. 初始化虚拟机
7. 虚拟机加载要执行的Java主类，解析参数并执行

### 启动过程说明

我们就以上面划分的阶段为整体脉络，再深入的看看各阶段的具体逻辑。

#### 1 启动入口

虚拟机程序运行的入口是在main.c/main方法中。之后会调用java.c/JLI_Launch方法。

```cpp
int
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* ergonomics class policy */
)
{   
    /************************** 前期初始化工作和状态打印 ********************/
    int mode = LM_UNKNOWN;
    char *what = NULL;
    char *cpath = 0;
    char *main_class = NULL;
    int ret;
    InvocationFunctions ifn; //和创建虚拟机相关的结构体 指向三个关键的函数
    jlong start, end;
    char jvmpath[MAXPATHLEN];
    char jrepath[MAXPATHLEN];
    char jvmcfg[MAXPATHLEN];

    _fVersion = fullversion;
    _dVersion = dotversion;
    _launcher_name = lname;
    _program_name = pname;
    _is_java_args = javaargs;
    _wc_enabled = cpwildcard;
    _ergo_policy = ergo;

    InitLauncher(javaw);
    DumpState(); //打印相关状态
    
    //打印参数
    if (JLI_IsTraceLauncher()) {
        int i;
        printf("Command line args:\n");
        for (i = 0; i < argc ; i++) {
            printf("argv[%d] = %s\n", i, argv[i]);
        }
        AddOption("-Dsun.java.launcher.diag=true", NULL);
    }
    
    /************************** 检验版本 ********************/
    /*
     * Make sure the specified version of the JRE is running.
     *
     * There are three things to note about the SelectVersion() routine:
     *  1) If the version running isn't correct, this routine doesn't
     *     return (either the correct version has been exec'd or an error
     *     was issued).
     *  2) Argc and Argv in this scope are *not* altered by this routine.
     *     It is the responsibility of subsequent code to ignore the
     *     arguments handled by this routine.
     *  3) As a side-effect, the variable "main_class" is guaranteed to
     *     be set (if it should ever be set).  This isn't exactly the
     *     poster child for structured programming, but it is a small
     *     price to pay for not processing a jar file operand twice.
     *     (Note: This side effect has been disabled.  See comment on
     *     bugid 5030265 below.)
     */
    SelectVersion(argc, argv, &main_class); //版本检测
    
    /************************** 创建执行环境 ********************/
    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));//解析相关环境 获取jre路径、jvmlib库和jvm.cfg
    
    /************************** 设置虚拟机环境 ********************/
    if (!IsJavaArgs()) {
        SetJvmEnvironment(argc,argv);
    }

    ifn.CreateJavaVM = 0;
    ifn.GetDefaultJavaVMInitArgs = 0;

    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }
    
    /************************** 加载虚拟机 ********************/
    if (!LoadJavaVM(jvmpath, &ifn)) { //加载 主要是从jvmlib库中解析函数地址 赋值给ifn
        return(6);
    }

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
    }

    JLI_TraceLauncher("%ld micro seconds to LoadJavaVM\n",
             (long)(jint)Counter2Micros(end-start));

    ++argv;
    --argc;

    if (IsJavaArgs()) {
        /* Preprocess wrapper arguments */
        TranslateApplicationArgs(jargc, jargv, &argc, &argv);
        if (!AddApplicationOptions(appclassc, appclassv)) {
            return(1);
        }
    } else {
        /* Set default CLASSPATH */
        cpath = getenv("CLASSPATH"); //添加CLASSPATH
        if (cpath == NULL) {
            cpath = ".";
        }
        SetClassPath(cpath);
    }
    
    /************************** 解析参数 ********************/
    /* Parse command line options; if the return value of
     * ParseArguments is false, the program should exit.
     */
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath))
    {
        return(ret);
    }

    /* Override class path if -jar flag was specified */
    if (mode == LM_JAR) { //如果是java -jar 则覆盖classpath
        SetClassPath(what);     /* Override class path */
    }

    /* set the -Dsun.java.command pseudo property */ //解析特殊属性
    SetJavaCommandLineProp(what, argc, argv);

    /* Set the -Dsun.java.launcher pseudo property */
    SetJavaLauncherProp();

    /* set the -Dsun.java.launcher.* platform properties */
    SetJavaLauncherPlatformProps();
    
    /************************** 初始化虚拟机 ********************/
    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

这个方法比较长，但是可以划分为几个部分去分析：

##### 1 前期初始化工作和状态打印

这里的初始化部分包括一些参数值的声明，特殊结构体`InvocationFuntions`的声明，启动器的初始化。
其中声明的参数会在后续的启动过程用来存储相关信息，例如保存JVM、JRE相关路径等。
`InvocationFuntions`是个重要的结构体，其中包含了创建JVM会被调用的三个函数指针。

```cpp
typedef struct {
    CreateJavaVM_t CreateJavaVM; //指向负责创建JavaVM和JNIEnv结构的函数指针
    GetDefaultJavaVMInitArgs_t GetDefaultJavaVMInitArgs; //指向获取默认JVM初始参数的函数指针
    GetCreatedJavaVMs_t GetCreatedJavaVMs; //指向获取JVM的函数指针
} InvocationFunctions;
```

`InitLaucher`方法主要就是根据`_JAVA_LAUNCHER_DEBUG`这个环境变量会决定后续是否输出DEBUG的打印。
在开启了launcher_debug后，`DumpState()`方法会打印出启动状态，并且之后打印出命令行参数。

##### 2 检验版本

`SelectVersion`会验证用户指定的java版本和实际执行的java版本是否兼容，如果不兼容会退出进程。用户可以通过`_JAVA_VERSION_SET`的环境变量或是jar包中manifest文件等方式指定运行的java版本。

##### 3 创建执行环境

`CreateExecutionEnvironment`会为后续的启动创建执行环境，这一步骤中主要是确定jdk所在的路径，解析jvmcfg和确认libjvm是否存在等。

1. 主要是根据处理器类型和主路径确定出JRE的路径
2. 以同样的方式确定jvm.cfg的文件位置，并解析jvm.cfg（jvm.cfg里面是一些虚拟机的默认配置，如常见的指定以客户端或服务端模式运行）
3. 检查虚拟机类型(-server/-client)，可以是jvm.cfg指定或是由启动参数指定
4. 确定libjvm库的位置，校验库是否存在，这个库核心的函数库

##### 4 设置虚拟机环境

`SetJvmEnviroment`主要解析NativeMemoryTracking参数，可以用来追踪本地内存的使用情况

##### 5 加载虚拟机

前期环境准备好之后，`LoadJavaVM()`会从之前确定的路径，加载libjvm库，并将其中的库中`JNI_CreateJavaVM`,`JNI_GetDefaultJavaVMInitArgs`和`JNI_GetCreatedJavaVMs`三个函数赋值给ifn。
这三个函数会在之后创建虚拟机时被使用。

##### 6 解析参数

这里有两个部分，一是解析命令行传入的参数，看是否有特定的JVM配置选项。这些参数会被用于后续虚拟机的创建上。这一过程主要发生在`ParseArguments()`中。另一个部分就是添加一些特定的虚拟机参数，发生在`SetJavaCommandLineProp`、`SetJavaLaucherProp`和`SetJavaLaucherPlatformProps`中。

##### 7 虚拟机初始化

在环境都准备好之后，会由`JVMInit()`执行虚拟机初始化工作，首先会通过`ShowSplashScreen()`方法加载启动动画，之后会进入`CountinueInNewThread()`方法，由新的线程负责创建虚拟机的工作。

#### 2 在新线程中继续虚拟机的创建

通过上文的介绍，我们找到了`java.c/ConutinueInNewThread()`的方法。这个方法分为两个部分，第一部分就是确定线程栈的深度，第二部分就是由`ContinueInNewThread0()`这个方法实现真正的虚拟机创建过程。

```cpp
int
ContinueInNewThread0(int (JNICALL *continuation)(void *), jlong stack_size, void * args) {
    int rslt;
#ifndef __solaris__
    pthread_t tid;
    //声明线程属性
    pthread_attr_t attr;
    //初始化线程属性并设置相关属性值
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

    //设置线程栈深度
    if (stack_size > 0) {
      pthread_attr_setstacksize(&attr, stack_size);
    }

    //创建线程并执行方法
    if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) { //创建线程 并将运行函数的起始地址和运行参数传入
      void * tmp;
      pthread_join(tid, &tmp);//阻塞当前线程 等待新线程运行结束返回
      rslt = (int)tmp;
    } else {
     /*
      * Continue execution in current thread if for some reason (e.g. out of
      * memory/LWP)  a new thread can't be created. This will likely fail
      * later in continuation as JNI_CreateJavaVM needs to create quite a
      * few new threads, anyway, just give it a try..
      */
      rslt = continuation(args);
    }

    pthread_attr_destroy(&attr);
#else /* __solaris__ */
    thread_t tid;
    long flags = 0;
    if (thr_create(NULL, stack_size, (void *(*)(void *))continuation, args, flags, &tid) == 0) {
      void * tmp;
      thr_join(tid, NULL, &tmp);
      rslt = (int)tmp;
    } else {
      /* See above. Continue in current thread if thr_create() failed */
      rslt = continuation(args);
    }
#endif /* !__solaris__ */
    return rslt;
}
```

在这个方法中，首先调用了`pthread_create()`函数创建了一个新线程，同时旧线程被jion等待新线程运行完成后返回。
`pthread_create()`是unix操作系统创建线程的函数，它的第一个参数表示线程标识，第二参数表示线程属性，第三个参数表示创建线程所要执行函数的地址，第四个参数则是将要执行的函数的参数。
等到新线程运行完成后，旧的线程也会返回。此时说明运行结束，进程将会退出。

需要注意的是此时传入的函数地址，它是指向`java.c/JavaMain()`函数。也就是说新创建的线程将会开始执行该函数。

#### 3 虚拟机创建、Java程序运行的主过程——JavaMain

新创建的线程会去执行`JavaMain()`函数，正式进入了创建虚拟机、运行Java代码的过程。

```cpp
int JNICALL
JavaMain(void * _args)
{   
    /*********************获取相关参数****************************/
    JavaMainArgs *args = (JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = NULL;
    jclass appClass = NULL; // actual application class being launched
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    RegisterThread();

    /*******************初始化JVM、打印相关信息********************************/
    start = CounterGet();
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    if (showSettings != NULL) {
        ShowSettings(env, showSettings);
        CHECK_EXCEPTION_LEAVE(1);
    }

    if (printVersion || showVersion) {
        PrintJavaVersion(env, showVersion);
        CHECK_EXCEPTION_LEAVE(0);
        if (printVersion) {
            LEAVE();
        }
    }

    /* If the user specified neither a class name nor a JAR file */
    if (printXUsage || printUsage || what == 0 || mode == LM_UNKNOWN) {
        PrintUsage(env, printXUsage);
        CHECK_EXCEPTION_LEAVE(1);
        LEAVE();
    }

    FreeKnownVMs();  /* after last possible PrintUsage() */

    if (JLI_IsTraceLauncher()) {
        end = CounterGet();
        JLI_TraceLauncher("%ld micro seconds to InitializeJVM\n",
               (long)(jint)Counter2Micros(end-start));
    }

    /* At this stage, argc/argv have the application's arguments */
    //打印Java程序的参数
    if (JLI_IsTraceLauncher()){
        int i;
        printf("%s is '%s'\n", launchModeNames[mode], what);
        printf("App's argc is %d\n", argc);
        for (i=0; i < argc; i++) {
            printf("    argv[%2d] = '%s'\n", i, argv[i]);
        }
    }
    
    /******************获取Java程序的主类***************************/
    ret = 1;

    /*
     * Get the application's main class.
     *
     * See bugid 5030265.  The Main-Class name has already been parsed
     * from the manifest, but not parsed properly for UTF-8 support.
     * Hence the code here ignores the value previously extracted and
     * uses the pre-existing code to reextract the value.  This is
     * possibly an end of release cycle expedient.  However, it has
     * also been discovered that passing some character sets through
     * the environment has "strange" behavior on some variants of
     * Windows.  Hence, maybe the manifest parsing code local to the
     * launcher should never be enhanced.
     *
     * Hence, future work should either:
     *     1)   Correct the local parsing code and verify that the
     *          Main-Class attribute gets properly passed through
     *          all environments,
     *     2)   Remove the vestages of maintaining main_class through
     *          the environment (and remove these comments).
     *
     * This method also correctly handles launching existing JavaFX
     * applications that may or may not have a Main-Class manifest entry.
     */
    mainClass = LoadMainClass(env, mode, what);//加载mainClass
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    /*
     * In some cases when launching an application that needs a helper, e.g., a
     * JavaFX application with no main method, the mainClass will not be the
     * applications own main class but rather a helper class. To keep things
     * consistent in the UI we need to track and report the application main class.
     */
    appClass = GetApplicationClass(env); //获取application class
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    /*
     * PostJVMInit uses the class name as the application name for GUI purposes,
     * for example, on OSX this sets the application name in the menu bar for
     * both SWT and JavaFX. So we'll pass the actual application class here
     * instead of mainClass as that may be a launcher or helper class instead
     * of the application class.
     */
    PostJVMInit(env, appClass, vm); // JVM 初始化后置处理
    /*
     * The LoadMainClass not only loads the main class, it will also ensure
     * that the main method's signature is correct, therefore further checking
     * is not required. The main method is invoked here so that extraneous java
     * stacks are not in the application stack trace.
     */
     
    /******************找主类的main方法************************/
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V"); //获取main class的 main(String[] args)方法
    CHECK_EXCEPTION_NULL_LEAVE(mainID);
    
    /*******************封装参数，调用main方法*****************/
    /* Build platform specific argument array */
    mainArgs = CreateApplicationArgs(env, argv, argc); //封装 main(String[] args) 方法的参数args
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs); //调用main(String args)方法

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    /*******************获取执行结果 并退出虚拟机**************/
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1; //根据是否有异常 确定退出码
    LEAVE(); //线程解绑 销毁JVM
}
```

##### 1 参数解析

之前上文解析得到的命令行参数等都被封装在`JavaMainArgs`结构体中，传给了`JavaMain`方法。因此需要从这个结构体中取回参数。
另外，还创建了一些变量用于之后的过程中存储值，譬如jclass，jmethodID等。

##### 2 初始化虚拟机，打印相关信息

上述代码中的`InitializeJVM()`方法会负责虚拟机的初始化过程。其代码如下：

```cpp
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

    memset(&args, 0, sizeof(args));
    args.version  = JNI_VERSION_1_2;
    args.nOptions = numOptions;
    args.options  = options;
    args.ignoreUnrecognized = JNI_FALSE;

    if (JLI_IsTraceLauncher()) {
        int i = 0;
        printf("JavaVM args:\n    ");
        printf("version 0x%08lx, ", (long)args.version);
        printf("ignoreUnrecognized is %s, ",
               args.ignoreUnrecognized ? "JNI_TRUE" : "JNI_FALSE");
        printf("nOptions is %ld\n", (long)args.nOptions);
        for (i = 0; i < numOptions; i++)
            printf("    option[%2d] = '%s'\n",
                   i, args.options[i].optionString);
    }

    r = ifn->CreateJavaVM(pvm, (void **)penv, &args); //通过ifn的函数指针 调用CreateJavaVM函数初始化JavaVM 和 JNIEnv
    JLI_MemFree(options);
    return r == JNI_OK;
}
```

先获取虚拟机参数，在通过ifn结构体中CreateJavaVM指针，调用正式创建Java虚拟机的函数`JNI_CreateJavaVM`。
`JNI_CreateJavaVM`代码的主要流程如下：

1. 先由`Threads::create_vm()`方法创建虚拟机
2. 给两个重要的指针赋值，分别是JavaVM * 和 JNIEnv
3. 一些后置处理，例如通过JVMTI（可以说是虚拟机的工具接口，提供了对虚拟机调试、监测等等的功能）、事件提交等

针对第一点，`Threads::create_vm()`是负责创建虚拟机，整个过程相对复杂，需要初始化很多模块，创建虚拟机的后台线程，加载必要的类等等，这里不做深入分析。之后有时间可以单独分析这一过程。
针对第二点中提到的两个数据结构，非常重要。我们可以看看它们的具体的内容。

###### JavaVM

JavaVM结构内部包的是`JNIInvokeInterface_`结构，因此我们直接看一下`JNIInvokeInterface_`的结构

```scss
struct JNIInvokeInterface_ {
    //预留字段
    void *reserved0;
    void *reserved1;
    void *reserved2;

    jint (JNICALL *DestroyJavaVM)(JavaVM *vm); //销毁虚拟机的函数指针

    jint (JNICALL *AttachCurrentThread)(JavaVM *vm, void **penv, void *args); //绑定线程的函数指针

    jint (JNICALL *DetachCurrentThread)(JavaVM *vm); //解绑线程的函数指针

    jint (JNICALL *GetEnv)(JavaVM *vm, void **penv, jint version); //获取JNIEnv结构的函数指针

    jint (JNICALL *AttachCurrentThreadAsDaemon)(JavaVM *vm, void **penv, void *args);//将线程转为后台线程
};
```

可以看到主要是一些和虚拟机操作的相关函数。

###### JNIEnv

`JNIEnv`结构内部包的是JNINativeInterface结构，这个结构同样定义了很多函数指针，代码太长，这里就不直接贴出了。有兴趣的可以在`jni.h`中自行查看。如果对结构中的方法分类的话，可以分成以下几类：

- 获取虚拟机信息
- 获取相关类和方法，方法执行
- 获取/设置对象字段
- 静态方法、静态变量的获取与设置
- 常见类型的对象的创建和释放
- 创建直接内存、访问锁等

总之，提供了通过C++代码访问Java程序的能力（这对于从事JNI开发的人来说十分重要）。

##### 3 确定Java程序的主类

了解完成虚拟机的初始化过程后，再回到JavaMain()方法中，之后是通过`LoadMainClass()`或`GetApplicationClass()`方法确定Java代码的主类。
如果我们在运行指定了Java类，那么这个类就是主类。这里还会调用`LauncherHelper.checkAndLoadMain()`检验主类是否合法。`LauncherHelper`的Java代码，这里就是上面介绍的JNIEnv的能力在C++的代码中执行Java代码。
对于一些没有主类的程序，需要通过`LaucherHelper.getApplicationClass()`确定程序类。

##### 4 从主类中获取main方法的methodID，并调用方法

再确定了mainClass之后，还需要找到该类定义的`main()`，获取main()方法，然后将程序参数封装，传递给`main()`执行，线程会以此为入口，开始执行Java程序。
这里的找方法和执行方法同样是依赖了JNIEnv中`GetStaticMethodID`和`CallStaticVoidMethod`。
所以我们的main()方法总是`static void`的。

##### 5 获取执行结果，退出虚拟机

当线程从Main()方法中返回，说明Java程序已经执行完成(或是异常退出)，这时候虚拟机会检查运行结果，并解绑线程销毁虚拟机，最终退出。

## HotSpot二分模型

## HotSpot的类模型

## HotSpot的对象模型

## 操作句柄Handle

## 类加载器 

类加载器可以加载类，这些类被HotSpot加载后，都以Klass对象表示。涉及到的主要的类加载器有启动类加载器/引导类加载器（Bootstrap ClassLoader）、扩展类加载器（Extension ClassLoader）和应用类加载器/系统类加载器（Application ClassLoader）。

### 引导类加载器/启动类加载器 

引导类加载器由ClassLoader类实现，这个ClassLoader类是用C++语言实现的，负责将 <JAVA_HOME>/lib目录、 -Xbootclasspath选项指定的目录或系统属性sun.boot.class.path指定的目录下的核心类库加载到内存中。 

用C++语言定义的类加载器及重要的函数如下： 

源代码位置：hotspot/src/share/vm/classfile/classLoader.hpp

```
class ClassLoader::AllStatic {
private:
   // First entry in linked list of ClassPathEntry instances
   // ClassPathEntry类指针，ClassPathEntry用于表示单个classpath路径，
   // 所有的ClassPathEntry实例以链表的形式关联起来，_first_entry表示链表的第一个实例
   static ClassPathEntry* _first_entry;
   // Last entry in linked list of ClassPathEntry instances
   // 表示链表的最后一个实例
   static ClassPathEntry* _last_entry;
   // Hash table used to keep track of loaded packages
   // 用于保存已经加载过的包名
   static PackageHashtable* _package_hash_table;
 
   // ...
   // 加载类
   static instanceKlassHandle load_classfile(Symbol* h_name,TRAPS);
   // 设置加载路径
   static void setup_bootstrap_search_path();
public:
   // 初始化类加载器
   static void initialize();
   // ...
}
```

通过_first_entry链表保存这个类加载器可以加载的一些类路径。在虚拟机启动时会通过调用ClassLoader::setup_bootstrap_search_path()函数来设置。

load_classfile()方法可以根据类名加载类，具体实现如下： 

源代码位置：openjdk/hotspot/src/share/vm/classfile/classLoader.cpp

```
instanceKlassHandle ClassLoader::load_classfile(Symbol* h_name, TRAPS) {
  // 获取类名
  const char* class_name = h_name->as_C_string();
  ....
  
  stringStream st;
  st.print_raw(h_name->as_utf8());
  st.print_raw(".class");
  // 获取文件名
  const char* file_name = st.as_string();
  ClassLoaderExt::Context context(class_name, file_name, THREAD);
  
  // ClassFileStream表示Class文件的字节流
  ClassFileStream* stream = NULL;
  int classpath_index = 0;
  ClassPathEntry* e = NULL;
  instanceKlassHandle h;
  {
    //从第一个ClassPathEntry开始遍历所有的ClassPathEntry
    e = _first_entry;
    while (e != NULL) {
      stream = e->open_stream(file_name, CHECK_NULL);
      // 如果检查返回false则返回null，check方法默认返回true
      if (!context.check(stream, classpath_index)) {
        return h; // NULL
      }
      // 如果找到目标文件则跳出循环
      if (stream != NULL) {
        break;
      }
      e = e->next();
      ++classpath_index;
    }
  }
  //如果找到了目标Class文件
  if (stream != NULL) {
    // 构建一个ClassFileParser实例
    ClassFileParser parser(stream);
    // 构建一个ClassLoaderData实例
    ClassLoaderData* loader_data = ClassLoaderData::the_null_class_loader_data();
    Handle protection_domain;
    TempNewSymbol parsed_name = NULL;
    // 解析并加载class文件，注意此时并未开始链接
    instanceKlassHandle  result = parser.parseClassFile(h_name,loader_data,protection_domain,parsed_name,false,CHECK_(h));
    ...
    // 调用ClassLoader的add_package方法，把当前类的包名加入到_package_hash_table中
    if (add_package(name, classpath_index, THREAD)) {
       h = result;
    }
  } 
 
  return h;
}
```

每个类加载器都对应着一个ClassLoaderData对象，通过ClassLoaderData::the_null_class_loader_data()函数获取引导类加载器对应的ClassLoaderData对象。

调用add_package()将已经解析过的类进行保存，避免重复加载解析。逻辑实现并不复杂，这里不在介绍。

parseClassFile()方法就是解析Class文件中的类、字段、常量池等信息，然后转换为C++内部的对等表示，如类元信息存储在InstanceKlass实例中，常量池信息存储在ConstantPool中，部分的C++对等实现（类模型）在之前已经介绍过，这里不再介绍。后续会详细介绍parseClassFile()方法解析Class文件的过程。

### 扩展类加载器 

扩展类加载器用Java语言编写，由sun.misc.Launcher$ExtClassLoader类实现，负责将 <JAVA_HOME >/lib/ext目录或者由系统变量-Djava.ext.dir所指定的目录中的类库加载到内存中。 

扩展类加载器ExtClassLoader的实现如下： 

源代码位置：openjdk/jdk/src/share/classes/sun/misc/Launcher.java

```
static class ExtClassLoader extends URLClassLoader {
  
    /**
     * create an ExtClassLoader. The ExtClassLoader is created
     * within a context that limits which files it can read
     */
    public static ExtClassLoader getExtClassLoader() throws IOException
    {
        final File[] dirs = getExtDirs(); // 获取要加载类的加载路径
  
        ...
        return new ExtClassLoader(dirs); // 实例化扩展类加载器
        ...
    }
  
    /*
     * Creates a new ExtClassLoader for the specified directories.
     */
    public ExtClassLoader(File[] dirs) throws IOException {
        super(getExtURLs(dirs), null, factory); // parent传递的参数为null，所以并不是引导类加载器
    }
  
    private static File[] getExtDirs() {
        String s = System.getProperty("java.ext.dirs");
        File[] dirs;
        if (s != null) {
            StringTokenizer st = new StringTokenizer(s, File.pathSeparator);
            int count = st.countTokens();
            dirs = new File[count];
            for (int i = 0; i < count; i++) {
                dirs[i] = new File(st.nextToken());
            }
        } else {
            dirs = new File[0];
        }
        return dirs;
    } 
    ...
}
```

ExtClassLoader类的构造函数中在调用父类的构造函数时，传递的第2个参数的值为null，这个值最终会赋值给parent字段，所以后面将会讲到，当这个字段的值为null时，java.lang.ClassLoader类中实现的loadClass()方法在加载一个类时将会调用findBootstrapClassOrNull()方法，而这个方法最终会调用C++实现的ClassLoader类的相关方法进行类加载。

### 系统类加载器/应用类加载器

系统类加载器用Java语言编写，由sun.misc.Launcher$AppClassLoader类实现，负责将系统环境变量-classpath、-cp或系统属性java.class.path指定的路径下的类库加载到内存中。

扩展类加载器AppClassLoader的实现如下：

源代码位置：openjdk/jdk/src/share/classes/sun/misc/Launcher.java

```
        public static ClassLoader getAppClassLoader(final ClassLoader extcl)throws IOException {
            final String s = System.getProperty("java.class.path");
            final File[] path = (s == null) ? new File[0] : getClassPath(s);
  
            // ...
            return new AppClassLoader(urls, extcl);
        }
  
        /*
         * Creates a new AppClassLoader
         */
        AppClassLoader(URL[] urls, ClassLoader parent) {
            super(urls, parent, factory); // parent是一个扩展类加载器实例
        }
  
        /**
         * Override loadClass so we can checkPackageAccess.
         */
        public Class loadClass(String name, boolean resolve)throws ClassNotFoundException{
            // ...
            return (super.loadClass(name, resolve));
        }
         
        // ...
}
```

在Launcher类的构造函数中实例化系统类加载器AppClassLoader时，会调用getAppClassLoader()方法获取系统类加载器，传入的参数是一个扩展类加载器ExtClassLoader实例，这样系统类加载器的父加载器就变成了扩展类加载器（与父加载器并非继承关系）。用户自定义的无参类加载器的父类加载器默认就是AppClassloader类加载器。

### 构造类加载器实例

HotSpot在启动过程中会在<JAVA_HOME>/lib/rt.jar包里面的sun.misc.Launcher类中完成扩展类加载器和系统类加载器的实例化，也会进行引导类加载器的初始化，也就是调用C++语言编写的ClassLoader类的initialize()方法。 

HotSpot在初始化时，会初始化一个重要的变量，定义如下：

源代码位置：hotspot/src/share/vm/classfile/systemDictionary.cpp

```
oop SystemDictionary::_java_system_loader = NULL;
```

这个属性保存系统类加载器实例，HotSpot在加载主类时会使用这个类加载器加载主类。属性在compute_java_system_loader()方法中初始化，调用链路如下：

```
JavaMain()                                      java.c  
InitializeJVM()                                 java.c
JNI_CreateJavaVM()                              jni.cpp 
Threads::create_vm()                            thread.cpp
SystemDictionary::compute_java_system_loader()  systemDictionary.cpp
```

方法的实现如下：

```

void SystemDictionary::compute_java_system_loader(TRAPS) {
  KlassHandle  system_klass(THREAD, WK_KLASS(ClassLoader_klass));
  JavaValue    result(T_OBJECT);
  // 调用java.lang.ClassLoader类的getSystemClassLoader()方法
  JavaCalls::call_static(&result, // 调用Java静态方法的返回值存储在result中
                         KlassHandle(THREAD, WK_KLASS(ClassLoader_klass)), // 调用的目标类为java.lang.ClassLoader
                         vmSymbols::getSystemClassLoader_name(), // 调用目标类中的目标方法为getSystemClassLoader
                         vmSymbols::void_classloader_signature(), // 调用目标方法的方法签名
                         CHECK);
  // 获取调用getSystemClassLoader()方法的结果并保存到_java_system_loader属性中
  _java_system_loader = (oop)result.get_jobject();  // 初始化属性为系统类加载器/应用类加载器/AppClassLoader
}
```

通过JavaClass::call_static()方法调用java.lang.ClassLoader类的getSystemClassLoader()方法。JavaClass::call_static()方法非常重要，它是HotSpot调用Java静态方法的API，后面会详细介绍。

下面看一下getSystemClassLoader()方法的实现，如下：

源代码位置：openjdk/jdk/src/share/classes/java/lang/ClassLoader.java　

```
private static ClassLoader scl;<br><br>public static ClassLoader getSystemClassLoader() {
        initSystemClassLoader();
        if (scl == null) {
            return null;
        }
        return scl;
}

private static synchronized void initSystemClassLoader() {
        if (!sclSet) {
            sun.misc.Launcher l = sun.misc.Launcher.getLauncher(); // 获取Launcher实例
            if (l != null) {
                scl = l.getClassLoader(); // 获取类加载器实例
                // ...               
            }
            sclSet = true;
        }
}
```

如上方法及变量定义在java.lang.ClassLoader类中。

在getSystemClassLoader()函数中调用Launcerh.getLauncher()方法获取Launcher实例，实例通过静态变量launcher来保存，静态变量的定义如下：

源代码位置：openjdk/jdk/src/share/classes/sum/misc/Launcher.java

```
private  static   Launcher launcher = new  Launcher();
```

调用l.getClassLoader()方法获取类加载器实例，如下：

源代码位置：openjdk/jdk/src/share/classes/sum/misc/Launcher.java

```
public ClassLoader getClassLoader() {
     return loader; // 返回的loader就是Launcher类的loader，也就是应用类加载器AppClassLoader
}
```

Launcher()类的构造函数如下：

源代码位置：openjdk/jdk/src/share/classes/sun/misc/Launcher.java

```
private ClassLoader loader;
 
public Launcher() {
        // Create the extension class loader
        ClassLoader extcl;
        try { 
            // 首先创建了扩展类加载器
            extcl = ExtClassLoader.getExtClassLoader();
        } catch (IOException e) {
            throw new InternalError("Could not create extension class loader", e);
        }
  
        // Now create the class loader to use to launch the application
        try { 
            // 以ExtClassloader作为父加载器创建了AppClassLoader
            loader = AppClassLoader.getAppClassLoader(extcl);
        } catch (IOException e) {
            throw new InternalError("Could not create application class loader", e);
        }
  
        // Also set the context class loader for the primordial thread. 
        // 默认线程上下文加载器为AppClassloader
        Thread.currentThread().setContextClassLoader(loader); 
}
```

如上方法及变量定义在sumn.misc.Lanucher类中。

可以看到有对ExtClassLoader与AppClassLoader实例创建的逻辑，这样HotSpot就可以通过_java_system_loader属性获取AppClassLoader实例，通过AppClassLoader实例中的parent属性使用ExtClassLoader。　

### 类的双亲委派机制 

双亲委派模型中的各个类加载器之间并不表示继承关系，而是表示工作过程，具体说就是，对于一个加载类的具体请求，首先要委派给自己的父类去加载，只有当父类无法完成加载请求时，子类自己才会去尝试加载。类加载器加载类时，不会涉及到类的连接、初始化等步骤，仅将Class文件中的可用信息转换为C++中对应的Klass实例存储。类加载器具体的委派逻辑在java.lang.ClassLoader类的loadClass()方法中实现。loadClass()方法的实现如下：

源代码位置：java/lang/ClassLoader.java

```
protected Class<?> loadClass(Stringname,boolean resolve) throws ClassNotFoundException  
{  
       synchronized (getClassLoadingLock(name)) {  
           // 判断此类是否已经加载过
           Class c = findLoadedClass(name); // (1)
           if (c ==null) {  
               try {  // 委托给父类加载器进行加载
                   if (parent !=null) {  
                       c = parent.loadClass(name,false);  (2)
                   } else {  // 父类加载器为null，委托给启动类加载器加载
                       c = findBootstrapClassOrNull(name);  (3)
                   }  
               } catch (ClassNotFoundException) {  
                   // 如果父类加载器抛出ClassNotFoundException异常，表明父类无法完成加载请求
               }  
 
               if (c ==null) {  
                   // 若仍然没有找到则调用findClass()方法查找
                   c = findClass(name);  (4)
                   ...  
               }  
           }  
           if (resolve) {  
               resolveClass(c);  //（5）
           }  
           return c;  
       }  
}
```

代码首先通过调用findLoadedClass()方法查找此类是否已经被加载过了，如果没有，则需要优先调用父类加载器去加载。除了用C++实现的引导类加载器需要通过调用findBootstrapClassOrNull()方法外，其它用Java实现的类加载器都通过parent字段查找父类加载器。因为这些类都继承了java.lang.ClassLoader这个基类（这个类中有对parent字段的定义），如实现了扩展类加载器的ExtClassLoader类和实现了应用类加载器/系统类加载器的AppClassLoader类的继承关系如下图所示。

当父类无法实现加载请求时，也就是c为null时，当前类加载器调用findClass()方法尝试自己完成加载请求。

编写一个自定义的类加载器，如下：

实例1

```
package com.jvm;
 
import java.net.URL;
import java.net.URLClassLoader;
 
public class UserClassLoader extends URLClassLoader {
      
    public UserClassloader(URL[] urls) {
        super(urls);
    }
  
    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        return super.loadClass(name, resolve);
    }
}
```

可以看到UserClassLoader继承了URLClassLoader类并覆写了loadClass()方法，调用super.loadClass()方法其实就是在调用ClassLoader类中实现的loadClass()方法。

下面详细介绍loadClass()方法中调用的findLoaderClass()、findBootstrapClassOrNull()与findClass()方法的实现。

### findLoadedClass()方法

方法用来判断此类是否已经加载过。调用的findLoadedClass()方法的实现如下： 

```

protected final Class<?> findLoadedClass(String name) {
        return findLoadedClass0(name);
}
```

调用本地方法findLoadedClass0()方法，这个方法的实现如下：

源代码位置：hotspot/src/share/vm/prims/jvm.cpp

```
JVM_ENTRY(jclass, JVM_FindLoadedClass(JNIEnv *env, jobject loader, jstring name))
  JVMWrapper("JVM_FindLoadedClass");
  // THREAD表示当前线程
  ResourceMark rm(THREAD);
  
  Handle h_name (THREAD, JNIHandles::resolve_non_null(name));
  // 获取类名对应的Handle
  Handle string = java_lang_String::internalize_classname(h_name, CHECK_NULL);
  
  // 检查是否为空
  const char* str   = java_lang_String::as_utf8_string(string());
  if (str == NULL) return NULL;
  
  // 判断类名是否超长
  const int str_len = (int)strlen(str);
  if (str_len > Symbol::max_length()) {
    return NULL;
  }
  // 创建一个临时的Symbol
  TempNewSymbol klass_name = SymbolTable::new_symbol(str, str_len, CHECK_NULL);
  
  // 获取类加载器对应的Handle
  Handle h_loader(THREAD, JNIHandles::resolve(loader));
  // 查找目标类是否存在
  Klass* k = SystemDictionary::find_instance_or_array_klass(klass_name,h_loader,Handle(),CHECK_NULL);
  // 将Klass转换成jclass                                                         
  return (k == NULL) ? NULL : (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END
```

JVM_ENTRY是宏定义，用于处理JNI调用的预处理，如获取当前线程的JavaThread指针。因为GC等原因，JNI函数不能直接访问Klass、oop对象，只能借助jobject、jclass等来访问，所以会调用JNIHandles::resolve_non_null()、JNIHandles::resolve()与JNIHandles::mark_local()等函数进行转换。

调用的find_instance_or_array_klass()函数的实现如下： 

源代码位置：hotspot/src/share/vm/classfile/systemDicitonary.cpp

```
// Look for a loaded instance or array klass by name.  Do not do any loading.
// return NULL in case of error.
Klass* SystemDictionary::find_instance_or_array_klass(
  Symbol*   class_name,
  Handle    class_loader,
  Handle    protection_domain,
  TRAPS
){
 
  Klass* k = NULL;
 
  assert(class_name != NULL, "class name must be non NULL");
 
  if (FieldType::is_array(class_name)) { // 数组的查找逻辑
    // The name refers to an array.  Parse the name.
    // dimension and object_key in FieldArrayInfo are assigned as a side-effect of this call
    FieldArrayInfo fd;
    BasicType t = FieldType::get_array_info(class_name, fd, CHECK_(NULL));
    if (t != T_OBJECT) {
        k = Universe::typeArrayKlassObj(t);
    } else {
        Symbol* sb = fd.object_key();
        k = SystemDictionary::find(sb, class_loader, protection_domain, THREAD);
    }
    if (k != NULL) {
        // k是个表示类的InstanceKlass对象或表示是一维数组的基本类型数组，而class_name可能表示的是多维数组，
        // 所以还可能需要根据维度创建出ObjArrayKlass对象
        k = k->array_klass_or_null(fd.dimension());
    }
  } else {  // 类的查找逻辑
      k = find(class_name, class_loader, protection_domain, THREAD);
  }
  return k;
}
```

其中有对数组和类的查询逻辑，方法并不涉及类的加载。如果是数组，首先要找到组成数组的基本元素的类型t；如果基本元素的类型是是基本类型，调用Universe::typeArrayKlassObj()函数找到表示基本类型的Klass对象，如果基本元素的类型是对象，调用SystemDictionary::find()方法查找。

调用的Universe::typeArrayKlassObj()函数是从Universe::typeArrayKlassObjs数组中获取Klass*。在之前介绍HotSpot的类模型时以boolean类型的一维数组创建TypeArrayKlass为例介绍过基本一维数组的创建过程，创建完成后会存储到Universe::typeArrayKlassObjs数组中。

SystemDictionary类中的find()方法的实现如下：

源代码位置：hotspot/src/share/vm/classfile/systemDictionary.cpp

```
Klass* SystemDictionary::find(
  Symbol*  class_name,
  Handle   class_loader,
  Handle   protection_domain,
  TRAPS
) {
 
  // ...
  class_loader = Handle(THREAD, java_lang_ClassLoader::non_reflection_class_loader(class_loader()));
  ClassLoaderData* loader_data = ClassLoaderData::class_loader_data_or_null(class_loader());
 
  // ...
  unsigned int  d_hash = dictionary()->compute_hash(class_name, loader_data);
  int           d_index = dictionary()->hash_to_index(d_hash);
 
  {
    // ... 
    return dictionary()->find(d_index, d_hash, class_name, loader_data,protection_domain, THREAD);
  }
}
```

将已经加载的类存储在Dictionary中，为了加快查找采用了hash存储。只有类加载器和类才能确定唯一的表示Java类的Klass实例，所以在计算d_hash时必须传入class_name和loader_data这两个参数。计算出具体索引d_index后，就可以调用Dictionary类的find()方法进行查找了。调用的Dictionary::find()函数的实现如下：

```
Klass* Dictionary::find(
    int                index,
    unsigned int       hash,
    Symbol*            name,
    ClassLoaderData*   loader_data,
    Handle             protection_domain,
    TRAPS
) {
  DictionaryEntry* entry = get_entry(index, hash, name, loader_data);
  if (entry != NULL && entry->is_valid_protection_domain(protection_domain)) {
    return entry->klass();
  } else {
    return NULL;
  }
}
```

调用get_entry()函数从hash表中查找实体，如果找到并且验证是合法的，则返回Klass对象，否则返回NULL。 

### findBootstrapClassOrNull()方法

调用findBootstrapClassOrNull()方法请求引导类加载器完成类加载请求，这个方法会调用本地方法findBootstrapClass()方法，源代码如下：

```

private Class<?> findBootstrapClassOrNull(String name){
    return findBootstrapClass(name);
}


// return null if not found
private native Class<?> findBootstrapClass(String name);
```

这个本地方法在HotSpot中的实现如下：

源代码位置：/src/share/native/java/lang/ClassLoader.c 

```
JNIEXPORT jclass JNICALL Java_java_lang_ClassLoader_findBootstrapClass(
   JNIEnv     *env, 
   jobject    loader,
   jstring    classname
){
    jclass cls = 0;
    // ...
    cls = JVM_FindClassFromBootLoader(env, clname);
    // ...
    return cls;
}
```

调用JVM_FindClassFromBootLoader()函数查找启动类加载器加载的类，如果没有查找到，方法会返回NULL。函数的实现如下：

源代码位置：hotspot/src/share/vm/prims/jvm.cpp

```
// Returns a class loaded by the bootstrap class loader; or null
// if not found.  ClassNotFoundException is not thrown.
 
JVM_ENTRY(jclass, JVM_FindClassFromBootLoader(JNIEnv* env,const char* name))
 
  // Java libraries should ensure that name is never null...
  if (name == NULL || (int)strlen(name) > Symbol::max_length()) {
    // It's impossible to create this class;  the name cannot fit
    // into the constant pool.
    return NULL;
  }
 
  TempNewSymbol h_name = SymbolTable::new_symbol(name, CHECK_NULL);
  Klass* k = SystemDictionary::resolve_or_null(h_name, CHECK_NULL);
  if (k == NULL) {
    return NULL;
  }
 
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END
　　
调用SystemDictionary::resolve_or_null()函数对类进行查找，函数的实现如下： 

Klass* SystemDictionary::resolve_or_null(Symbol* class_name, TRAPS) {
  return resolve_or_null(class_name, Handle(), Handle(), THREAD);
}
 
Klass* SystemDictionary::resolve_or_null(Symbol* class_name, Handle class_loader, Handle protection_domain, TRAPS) {
  // 数组，通过签名的格式来判断
  if (FieldType::is_array(class_name)) {
    return resolve_array_class_or_null(class_name, class_loader, protection_domain, CHECK_NULL);
  }
  // 普通类，通过签名的格式来判断
  else if (FieldType::is_obj(class_name)) {
    ResourceMark rm(THREAD);
    // 去掉签名中的开头字符L和结束字符;
    TempNewSymbol name = SymbolTable::new_symbol(class_name->as_C_string() + 1,
                                                 class_name->utf8_length() - 2,
                         CHECK_NULL);
    return resolve_instance_class_or_null(name, class_loader, protection_domain, CHECK_NULL);
  }
  else {
    return resolve_instance_class_or_null(class_name, class_loader, protection_domain, CHECK_NULL);
  }
}
```

调用resolve_array_class_or_null()方法查找数组时，如果组成数组元素的基本类型为引用类型，同样会调用resolve_instance_class_or_null()方法来查找类对应的Klass实例。方法的实现如下：

```
Klass* SystemDictionary::resolve_array_class_or_null(
   Symbol*   class_name,
   Handle    class_loader,
   Handle    protection_domain,
   TRAPS
){
  assert(FieldType::is_array(class_name), "must be array");
  Klass*          k = NULL;
  FieldArrayInfo  fd;
  // dimension and object_key in FieldArrayInfo are assigned as a side-effect of this call
  BasicType t = FieldType::get_array_info(class_name, fd, CHECK_NULL);
  if (t == T_OBJECT) {
    // naked oop "k" is OK here -- we assign back into it
    Symbol* sb = fd.object_key();
    k = SystemDictionary::resolve_instance_class_or_null(sb,class_loader,protection_domain,CHECK_NULL);
    if (k != NULL) {
       k = k->array_klass(fd.dimension(), CHECK_NULL);
    }
  } else {
    k = Universe::typeArrayKlassObj(t);
    int x = fd.dimension();
    TypeArrayKlass* tak = TypeArrayKlass::cast(k);
    k = tak->array_klass(x, CHECK_NULL);
  }
  return k;
}
包含对元素类型为引用类型和元素类型为一维数组的基本类型的Klass实例的查找。一维数组的基本类型的查找和find_instance_or_array_klass()方法的实现基本类似。下面看调用的resolve_instance_class_or_null()方法对引用类型的查找，实现如下： 

源代码位置：hotspot/src/share/vm/classfile/systemDictionary.cpp
 
Klass* SystemDictionary::resolve_instance_class_or_null(
   Symbol*   name,
   Handle    class_loader,
   Handle    protection_domain,
   TRAPS 
){
 
  // UseNewReflection
  // Fix for 4474172; see evaluation for more details
  class_loader = Handle(THREAD, java_lang_ClassLoader::non_reflection_class_loader(class_loader()));
  ClassLoaderData  *loader_data = register_loader(class_loader, CHECK_NULL);
 
  // Do lookup to see if class already exist and the protection domain has the right access
  // This call uses find which checks protection domain already matches
  // All subsequent calls use find_class, and set has_loaded_class so that
  // before we return a result we call out to java to check for valid protection domain
  // to allow returning the Klass* and add it to the pd_set if it is valid
  // 在变量SystemDictionary::_dictionary中查找是否类已经加载，如果加载就直接返回
  Dictionary* dic = dictionary();
  // 通过类名和类加载器计算hash值，class_loader是Handle类型，而Handle._value的类型是oop*。而loader_data是ClassLoaderData*类型
  unsigned int d_hash = dic->compute_hash(name, loader_data);
  // 计算在hash中的索引位置
  int d_index = dic->hash_to_index(d_hash);
  // 根据hash和index 查到对应的klassOop
  Klass* probe = dic->find(d_index, d_hash, name, loader_data,protection_domain, THREAD);
  if (probe != NULL){
       return probe; // 如果直接找到的话,则返回
  }
  // ...  省略其它查找的逻辑
}
```

如果类还没有加载，那么当前的方法还需要负责加载类。在实现的过程中考虑的因素比较多，比如解决并行加载、触发父类的加载、域权限的验证等，不过这些都不是我们要讨论的重点，我们仅看加载的过程，此方法中有如下调用：

```

// Do actual loading
k = load_instance_class(name, class_loader, THREAD);
```

调用的方法如下

```
// 体现出“双亲委派”机制，只要涉及到类的加载，都会调用这个函数
instanceKlassHandle SystemDictionary::load_instance_class(Symbol* class_name, Handle class_loader, TRAPS) {
  instanceKlassHandle nh = instanceKlassHandle(); // null Handle
  if (class_loader.is_null()) { // 使用引导类加载器来加载类
 
    // Search the shared system dictionary for classes preloaded into the shared spaces.
    // 在共享系统字典中搜索预加载到共享空间中的类，默认不使用共享空间，所以查找的结果为NULL
    instanceKlassHandle k;
    {
      k = load_shared_class(class_name, class_loader, THREAD);
    }
 
    if (k.is_null()) {
      // Use VM class loader，也就是系统类加载器进行类加载
      k = ClassLoader::load_classfile(class_name, CHECK_(nh));
    }
    // find_or_define_instance_class may return a different InstanceKlass
    // 调用SystemDictionary::find_or_define_instance_class->SystemDictionary::update_dictionary-> Dictionary::add_klass()将
    // 生成的Klass对象存起来。Dictionary是个hash表实现，使用的也是开链法解决hash冲突。
    if (!k.is_null()) {
      // 支持并行加载，也就是允许同一个类加载器同时加载多个类
      k = find_or_define_instance_class(class_name, class_loader, k, CHECK_(nh)); 
    }
    return k;
  } else { // 使用指定的类加载器加载，最终会调用java.lang.ClassLoader类中的loadClass()方法执行类加载
    // Use user specified class loader to load class. Call loadClass operation on class_loader.
    ResourceMark rm(THREAD);
 
    JavaThread* jt = (JavaThread*) THREAD;
 
    Handle s = java_lang_String::create_from_symbol(class_name, CHECK_(nh));
    // Translate to external class name format, i.e., convert '/' chars to '.'
    Handle string = java_lang_String::externalize_classname(s, CHECK_(nh));
 
    JavaValue result(T_OBJECT);
 
    KlassHandle spec_klass (THREAD, SystemDictionary::ClassLoader_klass());
    // 调用java.lang.ClassLoader类中的loadClass()方法进行类加载
    JavaCalls::call_virtual(&result,
                              class_loader,
                              spec_klass,
                              vmSymbols::loadClass_name(),
                              vmSymbols::string_class_signature(),
                              string,
                              CHECK_(nh));
//    assert(result.get_type() == T_OBJECT, "just checking");
    oop obj = (oop) result.get_jobject(); // 获取调用loadClass()方法返回的Class对象
 
    // Primitive classes return null since forName() can not be
    // used to obtain any of the Class objects representing primitives or void
    if ((obj != NULL) && !(java_lang_Class::is_primitive(obj))) {
      // 获取Class对象表示的Java类，也就是获取表示Java类的instanceKlass对象
      instanceKlassHandle k = instanceKlassHandle(THREAD, java_lang_Class::as_Klass(obj));
 
      // For user defined Java class loaders, check that the name returned is
      // the same as that requested.  This check is done for the bootstrap
      // loader when parsing the class file.
      if (class_name == k->name()) {
        return k;
      }
    }
    // Class is not found or has the wrong name, return NULL
    return nh;
  }
}
```

当class_loader为NULL时，表示使用启动类加载器加载类，调用ClassLoader::load_classfile()方法加载类；当class_loader不为NULL时，会调用java.lang.ClassLoader类中的loadClass()方法，相关方法在前面详细介绍过，这里不再介绍。

使用引导类加载器加载类时，调用ClassLoader::load_classfile()方法加载类，这样就得到了Klass对象，随后调用的SystemDictionary::find_or_define_instance_class()方法只是会将这个Klass对象添加到字典中。函数的实现如下：

```

instanceKlassHandle SystemDictionary::find_or_define_instance_class(
    Symbol*               class_name,
    Handle                class_loader,
    instanceKlassHandle   k,
    TRAPS
) {
 
  instanceKlassHandle  nh = instanceKlassHandle(); // null Handle
  Symbol*              name_h = k->name(); // passed in class_name may be null
  ClassLoaderData*     loader_data = class_loader_data(class_loader);
 
  unsigned int         d_hash = dictionary()->compute_hash(name_h, loader_data);
  int                  d_index = dictionary()->hash_to_index(d_hash);
 
  // ...
  {
    MutexLocker mu(SystemDictionary_lock, THREAD);
    // First check if class already defined
    if (UnsyncloadClass || (is_parallelDefine(class_loader))) {
      Klass* check = find_class(d_index, d_hash, name_h, loader_data);
      if (check != NULL) {
        return(instanceKlassHandle(THREAD, check));
      }
    }
 
    // ...
  }
   
  define_instance_class(k, THREAD);
 
  // ...
 
  return k;
}
```

方法同样会调用find_class()方法从字典中检查一下这个类是否已经加载，如果加载了就直接返回，否则调用define_instance_class()函数。调用SystemDictionary::update_dictionary()函数将已经加载的类添加到系统词典Map里面，如下：

```
// Update system dictionary - done after check_constraint and add_to_hierachy have been called.
源代码位置：hotspot/src/share/vm/classfile/systemDictionary.cpp
void SystemDictionary::update_dictionary(
    int                  d_index,
    unsigned int        d_hash,
    int                  p_index,
    unsigned int        p_hash,
    instanceKlassHandle  k,
    Handle               class_loader,
    TRAPS
) {
  // Compile_lock prevents systemDictionary updates during compilations
  assert_locked_or_safepoint(Compile_lock);
  Symbol*  name  = k->name();
  ClassLoaderData *loader_data = class_loader_data(class_loader);
 
  {
      MutexLocker mu1(SystemDictionary_lock, THREAD);
 
      // Make a new system dictionary entry.
      Klass* sd_check = find_class(d_index, d_hash, name, loader_data);
      if (sd_check == NULL) {
          dictionary()->add_klass(name, loader_data, k);
          notice_modification();
      }
          SystemDictionary_lock->notify_all();
  }
}
```

其中key使用类的包路径+类名，类加载器两者确定，value则为具体加载的类对应的instanceKlassHandle对象，其中维护这kclass对象。也就是系统词典里面使用类加载器和类的包路径类名唯一确定一个类。这也验证了在Java中同一个类使用两个类加载器进行加载后，加载的两个类是不一样的，是不能相互赋值的。 



## 核心类的预装载

## Java主类的装载  

## 触发类的装载  

## 类文件结构

### 类文件(ClassFile)

之前在介绍类的双亲委派机制时，多次涉及到对ClassFileParser类中的parseClassFile()的调用。因为类的装载不仅仅是找到类对应的Class文件二进制流，更要解析出Class文件中包含的信息，将其转换为C/C++表示方式，这样虚拟机在运行过程中才能更方便的进行操作。

在介绍parseClassFile()方法之前，先认识一下Java虚拟机规范定义的Class文件格式及ClassFileParse类中定义的一些重要属性。

类文件的格式如下：

```
ClassFile {
    u4             magic;
 
    u2             minor_version;
    u2             major_version;
 
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
 
    u2             access_flags;
    u2             this_class;
    u2             super_class;
 
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
 
    u2             fields_count;
    field_info     fields[fields_count];
 
    u2             methods_count;
    method_info    methods[methods_count];
 
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

### 解析类(ClassFileParser)

HotSpot定义了ClassFileParser类来辅助读取及保存类解析的相关信息，类及重要属性的定义如下：

源代码位置：src/share/vm/classfile/classLoader.hpp

```
class ClassFileParser VALUE_OBJ_CLASS_SPEC {
 private:
  u2        _major_version;
  u2        _minor_version;
  Symbol*   _class_name;
  ClassLoaderData*         _loader_data;
  KlassHandle              _host_klass;
  GrowableArray<Handle>*   _cp_patches; // overrides for CP entries
 
 
  // class attributes parsed before the instance klass is created:
  bool        _synthetic_flag;
  int         _sde_length;
  char*       _sde_buffer;
  u2          _sourcefile_index;
  u2          _generic_signature_index;
 
  // Metadata created before the instance klass is created.  Must be deallocated
  // if not transferred to the InstanceKlass upon successful class loading
  // in which case these pointers have been set to NULL.
  instanceKlassHandle _super_klass;
  ConstantPool*    _cp;
  Array<u2>*       _fields;
  Array<Method*>*  _methods;
  Array<u2>*       _inner_classes;
  Array<Klass*>*   _local_interfaces;
  Array<Klass*>*   _transitive_interfaces;
  // ...
  InstanceKlass*   _klass;  // InstanceKlass once created.
  …
  ClassFileStream* _stream;              // Actual input stream
  ...
}
```

类中的属性通过名称就可以知道存储的相关信息，其中最主要的就是通过cp保存常量池信息、通过fields保存域信息、通过methods保存方法、通过klass保存类相关的信息。通过_stream属性可以方便地读取流的信息。

类还定义了许多重要的函数，例如解析常量池的parse_constant_pool()与parse_constant_pool_entries()函数、解析方法的parse_methods()函数、解析字段的parse_fields()函数等。　

### 文件流(_stream)

_stream这个属性保存的是字节码文件流。如果要读取Class文件的内容，首先需要获取文件对应的字节流，ClassFileStream 内部维护了一个buffer，该buffer指向Class文件所对应的字节流。

ClassFileStream对象是在ClassLoader::load_classfile()函数中创建的，这个方法在之前介绍类的双亲委派机制时提到过，当装载一个类时，可能会调用到SystemDictionary::load_instance_class()函数，而这个函数会体现出“双亲委派”的逻辑。如果使用启动类加载器，那么可能需要调用load_classfile()方法装载类。load_classfile()方法的实现如下：

源代码位置：src/share/vm/classfile/classLoader.cpp

```
instanceKlassHandle ClassLoader::load_classfile(Symbol* h_name, TRAPS) {
 
  stringStream st;
  st.print_raw(h_name->as_utf8());
  st.print_raw(".class");
  const char* name = st.as_string(); // 通过st获取对应的文件名
 
  // Lookup stream for parsing .class file
  ClassFileStream* stream = NULL;
  {
    ClassPathEntry* e = _first_entry;
    while (e != NULL) {
      stream = e->open_stream(name, CHECK_NULL);
      if (stream != NULL) {
        break;
      }
      e = e->next();
    }
  }
  ...
}
```

遍历class_path找到要加载的类文件，获取到文件的绝对路径后就创建ClassFileStream对象。ClassPathEntry 是一个链表结构（因为class path有多个），同时在ClassPathEntry中还声明了一个虚函数open_stream()。这样就可以通过循环遍历链表上的结构，直到查找到某个路径下名称为name的文件为止，这时候open_stream()函数会返回ClassFileStream实例。

在load_classfile()方法中获取到ClassFileStream实例后会调用ClassFileParser类中的parseClassFile()方法，如下：

```

instanceKlassHandle ClassLoader::load_classfile(Symbol* h_name, TRAPS) {
  // ...
 
  instanceKlassHandle h;
  if (stream != NULL) {
    // class file found, parse it
    ClassFileParser parser(stream);
    ClassLoaderData* loader_data = ClassLoaderData::the_null_class_loader_data();
    Handle protection_domain;
    TempNewSymbol parsed_name = NULL;
    instanceKlassHandle result = parser.parseClassFile(h_name,loader_data,protection_domain,parsed_name,false,CHECK_(h));
    // add to package table
    if (add_package(name, classpath_index, THREAD)) {
      h = result;
    }
  }
 
  return h;
}
```

调用parseClassFile()方法后返回表示Java类的instanceKlass对象，最终方法返回的是操作instanceKlass对象的句柄instanceKlassHandle。下一篇开始将详细介绍parseClassFile()方法的实现。

简单介绍一下ClassFileStream类中的一些被频繁调用的方法，如下：

```

u1 ClassFileStream::get_u1(TRAPS) {
  return *_current++;
}
 
u2 ClassFileStream::get_u2(TRAPS) {
  u1* tmp = _current;
  _current += 2;
  return Bytes::get_Java_u2(tmp);
}
 
u4 ClassFileStream::get_u4(TRAPS) {
  u1* tmp = _current;
  _current += 4;
  return Bytes::get_Java_u4(tmp);
}
 
u8 ClassFileStream::get_u8(TRAPS) {
  u1* tmp = _current;
  _current += 8;
  return Bytes::get_Java_u8(tmp);
}
 
void ClassFileStream::skip_u1(int length, TRAPS) {
  _current += length;
}
 
void ClassFileStream::skip_u2(int length, TRAPS) {
  _current += length * 2;
}
 
void ClassFileStream::skip_u4(int length, TRAPS) {
  _current += length * 4;
}
```

Class文件由字节为单位的字节流组成，所有的16位、32位和64位长度的数据将被构造成 2个、4个和8个8字节单位来表示。多字节数据项总是按照Big-Endian的顺序进行存储，而x86等处理器则是使用了相反的Little-Endian顺序来存储数据。
因此，在x86平台上需要进行转换。代码如下：

源代码位置：openjdk/hotspot/src/cpu/x86/vm/bytes_x86.hpp

```
// Efficient reading and writing of unaligned unsigned data in Java
// byte ordering (i.e. big-endian ordering). Byte-order reversal is
// needed since x86 CPUs use little-endian format.
static inline u2   get_Java_u2(address p)           { return swap_u2(get_native_u2(p)); }
static inline u4   get_Java_u4(address p)           { return swap_u4(get_native_u4(p)); }
static inline u8   get_Java_u8(address p)           { return swap_u8(get_native_u8(p)); }
```

调用的相关函数如下：

源代码位置：openjdk/hotspot/src/cpu/x86/vm/bytes_x86.hpp

```
// Efficient reading and writing of unaligned unsigned data in platform-specific byte ordering
// (no special code is needed since x86 CPUs can access unaligned data)
static inline u2   get_native_u2(address p)         { return *(u2*)p; }
static inline u4   get_native_u4(address p)         { return *(u4*)p; }
static inline u8   get_native_u8(address p)         { return *(u8*)p; }
```

### 解析Class文件 

类文件解析的入口是ClassFileParser类中定义的parseClassFile()方法。上一小节得到了文件字节流stream后，接着会在ClassLoader::load_classfile()函数中调用parseClassFile()函数，调用的源代码实现如下：

源代码位置：src/share/vm/classfile/classLoader.cpp

```
instanceKlassHandle h;
if (stream != NULL) {
    // class file found, parse it
    ClassFileParser parser(stream);
    ClassLoaderData* loader_data = ClassLoaderData::the_null_class_loader_data();
    Handle protection_domain;
    TempNewSymbol parsed_name = NULL;
    instanceKlassHandle result =
                            parser.parseClassFile(h_name,loader_data,protection_domain,parsed_name,false,CHECK_(h));
    // add to package table
    if (add_package(name, classpath_index, THREAD)) {
      h = result;
    }
}
```

另外还有一些函数也会在必要的时候调用parseClassFile()函数，如装载Java主类时调用的SystemDictionary::resolve_from_stream()函数等。

调用的parseClassFile()函数的实现如下：　　

```
instanceKlassHandle parseClassFile(Symbol* name,
                                     ClassLoaderData* loader_data,
                                     Handle protection_domain,
                                     TempNewSymbol& parsed_name,
                                     bool verify,
                                     TRAPS) {
    KlassHandle no_host_klass;
    return parseClassFile(name, loader_data, protection_domain, no_host_klass, NULL, parsed_name, verify, THREAD);
}
```

调用的另外一个方法的原型如下：

```
instanceKlassHandle ClassFileParser::parseClassFile(Symbol* name,
                                                    ClassLoaderData* loader_data,
                                                    Handle protection_domain,
                                                    KlassHandle host_klass,
                                                    GrowableArray<Handle>* cp_patches,
                                                    TempNewSymbol& parsed_name,
                                                    bool verify,
                                                    TRAPS)
```

这个方法的实现太复杂，这里简单分几个步骤详细介绍。

### 1. 解析魔数、主版本号与次版本号

```
ClassFileStream* cfs = stream();
...
u4 magic = cfs->get_u4_fast();
guarantee_property(magic == JAVA_CLASSFILE_MAGIC,"Incompatible magic value %u in class file %s",magic, CHECK_(nullHandle));
// Version numbers
u2 minor_version = cfs->get_u2_fast();
u2 major_version = cfs->get_u2_fast();
…
_major_version = major_version;
_minor_version = minor_version;
```

读取魔数主要是为了验证值是否为0xCAFEBABE。读取到Class文件的主、次版本号并保存到ClassFileParser实例的major_version和minor_version中。　

### 2. 解析访问标识 

```
// Access flags
AccessFlags access_flags;
jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_CLASS_MODIFIERS;
 
if ((flags & JVM_ACC_INTERFACE) && _major_version < JAVA_6_VERSION) {
    // Set abstract bit for old class files for backward compatibility
    flags |= JVM_ACC_ABSTRACT;
}
access_flags.set_flags(flags);
```

读取并验证访问标识，这个访问标识在进行字段及方法解析过程中会使用，主要用来判断这些字段或方法是定义在接口中还是类中。JVM_RECOGNIZED_CLASS_MODIFIERS是一个宏，定义如下：

```
#define JVM_RECOGNIZED_CLASS_MODIFIERS (JVM_ACC_PUBLIC     |    \
                                        JVM_ACC_FINAL      |    \
                                        JVM_ACC_SUPER      |    \  // 辅助invokespecial指令
                                        JVM_ACC_INTERFACE  |    \
                                        JVM_ACC_ABSTRACT   |    \
                                        JVM_ACC_ANNOTATION |    \
                                        JVM_ACC_ENUM       |    \
                                        JVM_ACC_SYNTHETIC)
```

最后一个标识符是由前端编译器（如Javac等）添加上去的，表示是合成的类型。

### 3. 解析当前类索引 

类索引（this_class）是一个u2类型的数据，类索引用于确定这个类的全限定名。类索引指向常量池中类型为CONSTANT_Class_info的类描述符，再通过类描述符中的索引值找到常量池中类型为CONSTANT_Utf8_info的字符串。

```
// This class and superclass
u2 this_class_index = cfs->get_u2_fast();
 
Symbol*  class_name  = cp->unresolved_klass_at(this_class_index);
assert(class_name != NULL, "class_name can't be null");
 
// Update _class_name which could be null previously to be class_name
_class_name = class_name;
```

将读取到的当前类的名称保存到ClassFileParser实例的_class_name属性中。

调用的cp->unresolved_klass_at()方法的实现如下：

源代码位置：/hotspot/src/share/vm/oops/constantPool.hpp

```

// 未连接的返回Symbol*
// This method should only be used with a cpool lock or during parsing or gc
Symbol* unresolved_klass_at(int which) {     // Temporary until actual use
    intptr_t* oaar = obj_at_addr_raw(which);
    Symbol* tmp = (Symbol*)OrderAccess::load_ptr_acquire(oaar);
    Symbol* s = CPSlot(tmp).get_symbol();
    // check that the klass is still unresolved.
    assert(tag_at(which).is_unresolved_klass(), "Corrupted constant pool");
    return s;
}
```

举个例子如下：

```

#3 = Class         #17        // TestClass
...
#17 = Utf8          TestClass　
```

类索引为0x0003，去常量池里找索引为3的类描述符，类描述符中的索引为17，再去找索引为17的字符串，就是“TestClass”。调用obj_at_addr_raw()方法找到的是一个指针，这个指针指向表示“TestClass”这个字符串的Symbol对象，也就是在解析常量池项时会将本来存储索引值17替换为存储指向Symbol对象的指针。　

调用的obj_at_addr_raw()方法的实现如下：

```
intptr_t*   obj_at_addr_raw(int which) const {
    assert(is_within_bounds(which), "index out of bounds");
    return (intptr_t*) &base()[which];
}
intptr_t*   base() const {
  return (intptr_t*) (
     (  (char*) this  ) + sizeof(ConstantPool)
  );
}
```

base()是ConstantPool中定义的方法，所以this指针指向当前ConstantPool对象在内存中的首地址，加上ConstantPool类本身需要占用的内存大小后，指针指向了常量池相关信息，这部分信息通常就是length个指针宽度的数组，其中length为常量池数量。通过(intptr_t*)&base()[which]获取到常量池索引which对应的值，对于上面的例子来说就是一个指向Symbol对象的指针。　

### 4. 解析父类索引

父类索引（super_class）是一个u2类型的数据，父类索引用于确定这个类的父类全限定名。由于java语言不允许多重继承，所以父类索引只有一个。父类索指向常量池中类型为CONSTANT_Class_info的类描述符，再通过类描述符中的索引值找到常量池中类型为CONSTANT_Utf8_info的字符串。

```
u2 super_class_index = cfs->get_u2_fast();
instanceKlassHandle super_klass = parse_super_class(super_class_index,CHECK_NULL);
```

调用的parse_super()方法的实现如下： 

```

instanceKlassHandle ClassFileParser::parse_super_class(int super_class_index,TRAPS) {
 
  instanceKlassHandle super_klass;
  if (super_class_index == 0) { // 当为java.lang.Object类时，没有父类
    check_property(_class_name == vmSymbols::java_lang_Object(),
                   "Invalid superclass index %u in class file %s",super_class_index,CHECK_NULL);
  } else {
    check_property(valid_klass_reference_at(super_class_index),
                   "Invalid superclass index %u in class file %s",super_class_index,CHECK_NULL);
    // The class name should be legal because it is checked when parsing constant pool.
    // However, make sure it is not an array type.
    bool is_array = false;
    constantTag mytemp = _cp->tag_at(super_class_index);
    if (mytemp.is_klass()) {
       super_klass = instanceKlassHandle(THREAD, _cp->resolved_klass_at(super_class_index));
    }
  }
  return super_klass;
}
```

如果类已经连接，那么可通过super_class_index直接找到表示父类的InstanceKlass实例，否则返回的值就是NULL。　

resolved_klass_at()方法的实现如下：

源代码位置：/hotspot/src/share/vm/oops/constantPool.hpp

```
// 已连接的返回Klass*
Klass* resolved_klass_at(int which) const {  // Used by Compiler
    // Must do an acquire here in case another thread resolved the klass
    // behind our back, lest we later load stale values thru the oop.
    Klass* tmp = (Klass*)OrderAccess::load_ptr_acquire(obj_at_addr_raw(which));
    return CPSlot(tmp).get_klass(); 
}　
```

其中的CPSlot类的实现如下：

```
class CPSlot VALUE_OBJ_CLASS_SPEC {
  intptr_t  _ptr;
 public:
  CPSlot(intptr_t ptr): _ptr(ptr) {}
  CPSlot(Klass*   ptr): _ptr((intptr_t)ptr) {}
  CPSlot(Symbol*  ptr): _ptr((intptr_t)ptr | 1) {} // 或上1表示已经解析过了，Symbol*本来不需要解析
 
  intptr_t value()     { return _ptr; }
  bool is_resolved()   { return (_ptr & 1) == 0; }
  bool is_unresolved() { return (_ptr & 1) == 1; }
 
  Symbol* get_symbol() {
    assert(is_unresolved(), "bad call");
    return (Symbol*)(_ptr & ~1);
  }
  Klass* get_klass() {
    assert(is_resolved(), "bad call");
    return (Klass*)_ptr;
  }
};  
```

### 5. 解析实现接口

接口表，interfaces[]数组中的每个成员的值必须是一个对constant_pool表中项目的一个有效索引值， 它的长度为 interfaces_count。每个成员interfaces[i] 必须为CONSTANT_Class_info类型常量，其中 0 ≤ i <interfaces_count。在interfaces[]数组中，成员所表示的接口顺序和对应的源代码中给定的接口顺序（从左至右）一样，即interfaces[0]对应的是源代码中最左边的接口。

```
u2 itfs_len = cfs->get_u2_fast();
Array<Klass*>* local_interfaces =
parse_interfaces(itfs_len, protection_domain, _class_name,&has_default_methods, CHECK_(nullHandle));
```

parse_interfaces()方法的实现如下：

```
Array<Klass*>* ClassFileParser::parse_interfaces(int     length,
                                                 Handle  protection_domain,
                                                 Symbol* class_name,
                                                 bool*   has_default_methods,
                                                 TRAPS
){
  if (length == 0) {
    _local_interfaces = Universe::the_empty_klass_array();
  } else {
    ClassFileStream* cfs = stream();
    _local_interfaces = MetadataFactory::new_array<Klass*>(_loader_data, length, NULL, CHECK_NULL);
 
    int index;
    for (index = 0; index < length; index++) {
      u2 interface_index = cfs->get_u2(CHECK_NULL);
      KlassHandle interf;
 
      if (_cp->tag_at(interface_index).is_klass()) {
        interf = KlassHandle(THREAD, _cp->resolved_klass_at(interface_index));
      } else {
        Symbol*  unresolved_klass  = _cp->klass_name_at(interface_index);
 
        Handle   class_loader(THREAD, _loader_data->class_loader());
 
        // Call resolve_super so classcircularity is checked
        Klass* k = SystemDictionary::resolve_super_or_fail(class_name,
                                                           unresolved_klass,
                               class_loader,
                               protection_domain,
                                                           false, CHECK_NULL);
        // 将表示接口的InstanceKlass实例封装为KlassHandle实例
        interf = KlassHandle(THREAD, k);
      }
 
      if (InstanceKlass::cast(interf())->has_default_methods()) {
         *has_default_methods = true;
      }
      _local_interfaces->at_put(index, interf());
    }
 
    if (!_need_verify || length <= 1) {
       return _local_interfaces;
    }
  }
  return _local_interfaces;
}
```

循环对类实现的每个接口进行处理，通过interface_index找到接口在C++类中的表示InstanceKlass实例，然后封装为KlassHandle后，存储到_local_interfaces数组中。需要注意的是，如何通过interface_index找到对应的InstanceKlass实例，如果接口索引在常量池中已经是对应的InstanceKlass实例，说明已经连接过了，直接通过_cp_resolved_klass_at()方法获取即可；如果只是一个字符串表示，需要调用SystemDictionary::resolve_super_or_fail()方法进行连接，这个方法在连接时会详细介绍，这里不做过多介绍。

klass_name_at()方法的实现如下：

```

Symbol* ConstantPool::klass_name_at(int which) {
  assert(tag_at(which).is_unresolved_klass() || tag_at(which).is_klass(),
         "Corrupted constant pool");
  // A resolved constantPool entry will contain a Klass*, otherwise a Symbol*.
  // It is not safe to rely on the tag bit's here, since we don't have a lock, and the entry and
  // tag is not updated atomicly.
  CPSlot entry = slot_at(which);
  if (entry.is_resolved()) { // 已经连接时，获取到的是指向InstanceKlass实例的指针
    // Already resolved - return entry's name.
    assert(entry.get_klass()->is_klass(), "must be");
    return entry.get_klass()->name();
  } else {  // 未连接时，获取到的是指向Symbol实例的指针
    assert(entry.is_unresolved(), "must be either symbol or klass");
    return entry.get_symbol();
  }
}
```

其中的slot_at()方法的实现如下：

```
CPSlot slot_at(int which) {
    assert(is_within_bounds(which), "index out of bounds");
    // Uses volatile because the klass slot changes without a lock.
    volatile intptr_t adr = (intptr_t)OrderAccess::load_ptr_acquire(obj_at_addr_raw(which));
    assert(adr != 0 || which == 0, "cp entry for klass should not be zero");
    return CPSlot(adr);
}
```

同样调用obj_at_addr_raw()方法，获取ConstantPool中对应索引处存储的值，然后封装为CPSlot对象返回即可。

### 6. 解析类属性

```
ClassAnnotationCollector parsed_annotations;``parse_classfile_attributes(&parsed_annotations, CHECK_(nullHandle));
```

调用parse_classfile_attributes()方法解析类属性，方法的实现比较繁琐，只需要按照各属性的格式来解析即可，有兴趣的读者可自行研究。

### 7.常量池解析 

在调用ClassFileParser::parseClassFile()方法对类文件进行解释时，会调ClassFileParser::parse_constant_pool()方法对常量池进行解释，调用的语句如下：

```
constantPoolHandle cp = parse_constant_pool(CHECK_(nullHandle));
```

方法parse_constant_pool()的实现如下：

```

constantPoolHandle ClassFileParser::parse_constant_pool(TRAPS) {
  ClassFileStream* cfs = stream();
  constantPoolHandle nullHandle;
 
  u2 length = cfs->get_u2_fast();
  ConstantPool* constant_pool = ConstantPool::allocate(_loader_data, length,
                                                        CHECK_(nullHandle));
  _cp = constant_pool; // save in case of errors
  constantPoolHandle cp (THREAD, constant_pool);
  // ...
  // parsing constant pool entries
  parse_constant_pool_entries(length, CHECK_(nullHandle));
  return cp;
}
```

调用ConstantPool::allocate()创建ConstantPool对象，然后调用parse_constant_pool_entries()解析常量池中的项并将这些项保存到ConstantPool对象中。　　

首先介绍一下ConstantPool类，这个类的对象代码具体的常量池，保存着常量池元信息。

### 1、ConstantPool类

类的定义如下：

```
class ConstantPool : public Metadata {
 private:
  Array<u1>*           _tags;        // the tag array describing the constant pool's contents
  ConstantPoolCache*   _cache;       // the cache holding interpreter runtime information 解释执行时的运行时信息
  InstanceKlass*       _pool_holder; // the corresponding class
  Array<u2>*           _operands;    // for variable-sized (InvokeDynamic) nodes, usually empty
 
  // Array of resolved objects from the constant pool and map from resolved
  // object index to original constant pool index
  jobject              _resolved_references; // jobject是指针类型
  Array<u2>*           _reference_map;
 
  int                  _flags;  // old fashioned bit twiddling
  int                  _length; // number of elements in the array
 
  union {
    // set for CDS to restore resolved references
    int                _resolved_reference_length;
    // keeps version number for redefined classes (used in backtrace)
    int                _version;
  } _saved;
 
  Monitor*             _lock;
  ...
}
```

类表示常量池元信息，所以继承了类Metadata。_tags表示常量池中的内容，常量池中的总项数通过_length来保存，所以_tags数组的长度也为_length，具体存储的内容就是每一项的tag值，这都是虚拟机规范定义好的；_cache辅助解释运行来保存一些信息，在介绍解释运行时会介绍。其它的属性暂时不做过多介绍。

### 2、创建ConstantPool实例

在解析常量池的方法ClassFileParser::parse_constant_pool()中首先会调用ConstantPool::allocate()方法创建ConstantPool实例，方法的实现如下：

```
ConstantPool* ConstantPool::allocate(ClassLoaderData* loader_data, int length, TRAPS) {
  // Tags are RW but comment below applies to tags also.
  Array<u1>* tags = MetadataFactory::new_writeable_array<u1>(loader_data, length, 0, CHECK_NULL);
 
  int size = ConstantPool::size(length);
 
  // CDS considerations:
  // Allocate read-write but may be able to move to read-only at dumping time
  // if all the klasses are resolved.  The only other field that is writable is
  // the resolved_references array, which is recreated at startup time.
  // But that could be moved to InstanceKlass (although a pain to access from
  // assembly code).  Maybe it could be moved to the cpCache which is RW.
  return new (loader_data, size, false, MetaspaceObj::ConstantPoolType, THREAD) ConstantPool(tags);
}
```

参数length就表示常量池项的数量，调用ConstantPool::size()计算所需要分配内存的大小，然后创建ConstantPool对象返回。size()方法的实现如下：

```
static int size(int length){
      int s = header_size();
      return align_object_size(s + length);
}
 
// Sizing (in words)
static int header_size() {
      int num = sizeof(ConstantPool);
      return num/HeapWordSize;
}
```

由方法实现可知，就是ConstantPool实例本身占用的内存大小加上length个指针长度。ConstantPool对象最终的内存布局如下图所示。

_valid是定义在Metadata中的int类型，只有debug版本才有，如果是product版本，则没有这个属性，那么Metadata就只占用8字节。关于对象的内存布局在之前已经介绍过，这里不再介绍。

调用header_size()在debug版本下得到的值为88（在不压缩指针的情况下，也就是使用命令XX禁止指针压缩），然后还需要加上length个指针宽度，这就是ConstantPool对象需要的内存空间大小。

通过重载new运算符进行堆内存分配，new运算符的重载定义在MetaspaceObj（ConstantPool间接继承此类）类中，如下：

```
void* MetaspaceObj::operator new(size_t size, ClassLoaderData* loader_data,
                                 size_t word_size, bool read_only,
                                 MetaspaceObj::Type type, TRAPS) throw() {
  // Klass has it's own operator new
  return Metaspace::allocate(loader_data, word_size, read_only,
                             type, CHECK_NULL);
}
```

调用的Metaspace::allocate()方法在堆中分配内存，这个方法在介绍垃圾收集时将详细介绍，这里只需要知道，这个方法会在堆中分配size大小的内存并且会将内存清零。

调用ConstantPool构造函数初始化一些属性，如下：

```
ConstantPool::ConstantPool(Array<u1>* tags) {
  set_length(tags->length());
  set_tags(NULL);
  set_cache(NULL);
  set_reference_map(NULL);
  set_resolved_references(NULL);
  set_operands(NULL);
  set_pool_holder(NULL);
  set_flags(0);
 
  // only set to non-zero if constant pool is merged by RedefineClasses
  set_version(0);
  set_lock(new Monitor(Monitor::nonleaf + 2, "A constant pool lock"));
 
  // initialize tag array
  int length = tags->length();
  for (int index = 0; index < length; index++) {
    tags->at_put(index, JVM_CONSTANT_Invalid);
  }
  set_tags(tags);
}
```

可以看到对tags、_length及_lock等属性的初始化。其中tags数组中存储了JVM_CONSTANT_Invalid值，在分析具体的常量池项时会更新为如下枚举类中定义的值：

源代码位置：hotspot/src/share/vm/prims/jvm.h

```
enum {
    JVM_CONSTANT_Utf8 = 1,      // 1
    JVM_CONSTANT_Unicode,       // 2      /* unused */
    JVM_CONSTANT_Integer,       // 3
    JVM_CONSTANT_Float,         // 4
    JVM_CONSTANT_Long,          // 5
    JVM_CONSTANT_Double,        // 6
    JVM_CONSTANT_Class,         // 7
    JVM_CONSTANT_String,        // 8
    JVM_CONSTANT_Fieldref,      // 9
    JVM_CONSTANT_Methodref,     // 10
    JVM_CONSTANT_InterfaceMethodref,   // 11
    JVM_CONSTANT_NameAndType,          // 12
    JVM_CONSTANT_MethodHandle           = 15,  // JSR 292
    JVM_CONSTANT_MethodType             = 16,  // JSR 292
    //JVM_CONSTANT_(unused)             = 17,  // JSR 292 early drafts only
    JVM_CONSTANT_InvokeDynamic          = 18,  // JSR 292
    JVM_CONSTANT_ExternalMax            = 18   // Last tag found in classfiles
};
```

这就是常量池项中的tag值，不过常量池第一项仍然为JVM_CONSTANT_Invalid。

下面介绍一下虚拟机规范规定的格式：

```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
 
CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
}
 
CONSTANT_Float_info {
    u1 tag;
    u4 bytes;
}
 
CONSTANT_Long_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
 
CONSTANT_Double_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
 
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
 
 
CONSTANT_String_info {
    u1 tag;
    u2 string_index;
}
 
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
 
CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
 
CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
 
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
 
 
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
 
CONSTANT_MethodType_info {
    u1 tag;
    u2 descriptor_index;
}
 
CONSTANT_InvokeDynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```

在常量池解析过程中，通过索引确定了常量池项后会将tag放到ConstantPool类中的_tags数组中，数组的下标与常量池索引相对应；剩下的信息只能存储到ConstantPool类后开辟的length个指针宽度的空间中，也可以成是length长度的指针数组，其中的下标也与常量池索引对应。指针在64位上的长度为8，所以能够存储除CONSTANT_Utf8_info外的所有常量池项信息（除tag外）。例如对于CONSTANT_Double_info来说，高4位存储high_bytes，低4位存储low_bytes。遇到CONSTANT_Utf8_info常量池项时，直接封装为Symbol对象，这样只要存储指向Symbol对象的指针即可。

在parse_constant_pool()方法中调用parse_constant_pool_entries()方法对常量池中的各个项进行解析，方法的实现如下：

```
void ClassFileParser::parse_constant_pool_entries(int length, TRAPS) {
  // Use a local copy of ClassFileStream. It helps the C++ compiler to optimize
  // this function (_current can be allocated in a register, with scalar
  // replacement of aggregates). The _current pointer is copied back to
  // stream() when this function returns. DON'T call another method within
  // this method that uses stream().
  ClassFileStream*  cfs0 = stream();
  ClassFileStream   cfs1 = *cfs0;
  ClassFileStream*  cfs  = &cfs1;
 
  Handle class_loader(THREAD, _loader_data->class_loader());
 
  // Used for batching symbol allocations.
  const char*   names[SymbolTable::symbol_alloc_batch_size];
  int           lengths[SymbolTable::symbol_alloc_batch_size];
  int           indices[SymbolTable::symbol_alloc_batch_size];
  unsigned int  hashValues[SymbolTable::symbol_alloc_batch_size];
  int           names_count = 0;
 
  // parsing  Index 0 is unused
  for (int index = 1; index < length; index++) {
    // Each of the following case guarantees one more byte in the stream
    // for the following tag or the access_flags following constant pool,
    // so we don't need bounds-check for reading tag.
    u1 tag = cfs->get_u1_fast();
    switch (tag) {
      case JVM_CONSTANT_Class :
        {
          cfs->guarantee_more(3, CHECK);  // name_index, tag/access_flags
          u2 name_index = cfs->get_u2_fast();
          _cp->klass_index_at_put(index, name_index);
        }
        break;
      case JVM_CONSTANT_Fieldref :
        {
          cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
          u2 class_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          _cp->field_at_put(index, class_index, name_and_type_index);
        }
        break;
      case JVM_CONSTANT_Methodref :
        {
          cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
          u2 class_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          _cp->method_at_put(index, class_index, name_and_type_index);
        }
        break;
      case JVM_CONSTANT_InterfaceMethodref :
        {
          cfs->guarantee_more(5, CHECK);  // class_index, name_and_type_index, tag/access_flags
          u2 class_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          _cp->interface_method_at_put(index, class_index, name_and_type_index);
        }
        break;
      case JVM_CONSTANT_String :
        {
          cfs->guarantee_more(3, CHECK);  // string_index, tag/access_flags
          u2 string_index = cfs->get_u2_fast();
          _cp->string_index_at_put(index, string_index);
        }
        break;
      case JVM_CONSTANT_MethodHandle :
      case JVM_CONSTANT_MethodType :
        if (tag == JVM_CONSTANT_MethodHandle) {
          cfs->guarantee_more(4, CHECK);  // ref_kind, method_index, tag/access_flags
          u1 ref_kind = cfs->get_u1_fast();
          u2 method_index = cfs->get_u2_fast();
          _cp->method_handle_index_at_put(index, ref_kind, method_index);
        } else if (tag == JVM_CONSTANT_MethodType) {
          cfs->guarantee_more(3, CHECK);  // signature_index, tag/access_flags
          u2 signature_index = cfs->get_u2_fast();
          _cp->method_type_index_at_put(index, signature_index);
        } else {
          ShouldNotReachHere();
        }
        break;
      case JVM_CONSTANT_InvokeDynamic :
        {
          cfs->guarantee_more(5, CHECK);  // bsm_index, nt, tag/access_flags
          u2 bootstrap_specifier_index = cfs->get_u2_fast();
          u2 name_and_type_index = cfs->get_u2_fast();
          if (_max_bootstrap_specifier_index < (int) bootstrap_specifier_index)
            _max_bootstrap_specifier_index = (int) bootstrap_specifier_index;  // collect for later
          _cp->invoke_dynamic_at_put(index, bootstrap_specifier_index, name_and_type_index);
        }
        break;
      case JVM_CONSTANT_Integer :
        {
          cfs->guarantee_more(5, CHECK);  // bytes, tag/access_flags
          u4 bytes = cfs->get_u4_fast();
          _cp->int_at_put(index, (jint) bytes);
        }
        break;
      case JVM_CONSTANT_Float :
        {
          cfs->guarantee_more(5, CHECK);  // bytes, tag/access_flags
          u4 bytes = cfs->get_u4_fast();
          _cp->float_at_put(index, *(jfloat*)&bytes);
        }
        break;
      case JVM_CONSTANT_Long :
        {
          cfs->guarantee_more(9, CHECK);  // bytes, tag/access_flags
          u8 bytes = cfs->get_u8_fast();
          _cp->long_at_put(index, bytes);
        }
        index++;   // Skip entry following eigth-byte constant, see JVM book p. 98
        break;
      case JVM_CONSTANT_Double :
        {
          cfs->guarantee_more(9, CHECK);  // bytes, tag/access_flags
          u8 bytes = cfs->get_u8_fast();
          _cp->double_at_put(index, *(jdouble*)&bytes);
        }
        index++;   // Skip entry following eigth-byte constant, see JVM book p. 98
        break;
      case JVM_CONSTANT_NameAndType :
        {
          cfs->guarantee_more(5, CHECK);  // name_index, signature_index, tag/access_flags
          u2 name_index = cfs->get_u2_fast();
          u2 signature_index = cfs->get_u2_fast();
          _cp->name_and_type_at_put(index, name_index, signature_index);
        }
        break;
      case JVM_CONSTANT_Utf8 :
        {
          cfs->guarantee_more(2, CHECK);  // utf8_length
          u2  utf8_length = cfs->get_u2_fast();
          u1* utf8_buffer = cfs->get_u1_buffer();
          assert(utf8_buffer != NULL, "null utf8 buffer");
          // Got utf8 string, guarantee utf8_length+1 bytes, set stream position forward.
          cfs->guarantee_more(utf8_length+1, CHECK);  // utf8 string, tag/access_flags
          cfs->skip_u1_fast(utf8_length);
 
          if (EnableInvokeDynamic && has_cp_patch_at(index)) {
            Handle patch = clear_cp_patch_at(index);
 
            char* str = java_lang_String::as_utf8_string(patch());
            // (could use java_lang_String::as_symbol instead, but might as well batch them)
            utf8_buffer = (u1*) str;
            utf8_length = (int) strlen(str);
          }
 
          unsigned int hash;
          Symbol* result = SymbolTable::lookup_only((char*)utf8_buffer, utf8_length, hash);
          if (result == NULL) {
            names[names_count] = (char*)utf8_buffer;
            lengths[names_count] = utf8_length;
            indices[names_count] = index;
            hashValues[names_count++] = hash;
            if (names_count == SymbolTable::symbol_alloc_batch_size) {
              SymbolTable::new_symbols(_loader_data, _cp, names_count, names, lengths, indices, hashValues, CHECK);
              names_count = 0;
            }
          } else {
            _cp->symbol_at_put(index, result);
          }
        }
        break;
      default:
        classfile_parse_error("Unknown constant tag %u in class file %s", tag, CHECK);
        break;
    }
  }
 
  // Allocate the remaining symbols
  if (names_count > 0) {
    SymbolTable::new_symbols(_loader_data, _cp, names_count, names, lengths, indices, hashValues, CHECK);
  }
 
  cfs0->set_current(cfs1.current());
}
```

循环处理length个常量池项，不过第一个常量池项不需要处理，所以循环下标index的值初始化为1。

如果要了解各个常量池项的具体结构，代码的逻辑理解起来其实并不难。所有项的第一个字节都是用来描述常量池元素类型，调用cfs->get_u1_fast()获取元素类型后，就可以通过switch语句分情况进行处理。













## 字段解析

字段解析之伪共享 

25、字段解析（3）  

26、字段解析之OopMapBlock（4）

27、方法解析之Method与ConstMethod介绍  

28、方法解析

29、klassVtable与klassItable类的介绍 

30、计算vtable的大小 

31、计算itable的大小 

32、解析Class文件之创建InstanceKlass对象 

33、字段解析之字段注入 

34、类的连接  

35、类的连接之验证 

36、类的连接之重写（1） 

37、类的连接之重写（2）

38、方法的连接  

39、初始化vtable 

40、初始化itable  

41、类的初始化 

## 对象的创建  

## Java引用类型 

44、Java引用类型之软引用（1）

45、Java引用类型之软引用（2）

46、Java引用类型之弱引用与幻像引用 

47、Java引用类型之最终引用 

## Java对象的内存分配

　Java对象的分配，根据其过程，将其分为快速分配和慢速分配两种形式，其中快速分配使用无锁的指针碰撞技术在新生代的Eden区上进行分配，而慢速分配根据堆的实现方式、GC的实现方式、代的实现方式不同而具有不同的分配调用层次。 
下面就以bytecodeInterpreter解释器对于new指令的解释出发，分析实例对象的内存分配过程：

### 　一、快速分配

　 1.实例的创建首先需要知道该类型是否被加载和正确解析，根据字节码所指定的CONSTANT_Class_info常量池索引，获取对象的类型信息并调用is_unresovled_klass()验证该类是否被解析过，在创建类的实例之前，必须确保该类型已经被正确加载和解析。

```
 CASE(_new): {
        u2 index = Bytes::get_Java_u2(pc+1);
        constantPoolOop constants = istate->method()->constants();
        if (!constants->tag_at(index).is_unresolved_klass()) {
```

　　2.接下来获取该类型在虚拟机中的表示instanceKlass(具体可以参考前文[实例探索Java对象的组织结构](http://www.cnblogs.com/iceAeterNa/p/4877470.html))　

```
oop entry = constants->slot_at(index).get_oop();
          assert(entry->is_klass(), "Should be resolved klass");
          klassOop k_entry = (klassOop) entry;
          assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass");
          instanceKlass* ik = (instanceKlass*) k_entry->klass_part();
```

　　3.当类型已经被初始化并且可以被快速分配时，那么将根据UseTLAB来决定是否使用TLAB技术(Thread-Local Allocation Buffers，线程局部分配缓存技术)来将分配工作交由线程自行完成。TLAB是每个线程在Java堆中预先分配了一小块内存，当有对象创建请求内存分配时，就会在该块内存上进行分配，而不需要在Eden区通过同步控制进行内存分配。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
            size_t obj_size = ik->size_helper();
            oop result = NULL;
            // If the TLAB isn't pre-zeroed then we'll have to do it
            bool need_zero = !ZeroTLAB;
            if (UseTLAB) {
              result = (oop) THREAD->tlab().allocate(obj_size);
            }
            if (result == NULL) {
              need_zero = true;
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　4.如果不使用TLAB或在TLAB上分配失败，则会尝试在堆的Eden区上进行分配。Universe::heap()返回虚拟机内存体系所使用的CollectedHeap，其top_addr()返回的是Eden区空闲块的起始地址变量_top的地址，end_addr()是Eden区空闲块的结束地址变量_end的地址。故这里compare_to是Eden区空闲块的起始地址，new_top为使用该块空闲块进行分配后新的空闲块起始地址。这里使用CAS操作进行空闲块的同步操作，即观察_top的预期值，若与compare_to相同，即没有其他线程操作该变量，则将new_top赋给_top真正成为新的空闲块起始地址值，这种分配技术叫做bump-the-pointer(指针碰撞技术)。

　　

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 retry:
              HeapWord* compare_to = *Universe::heap()->top_addr();
              HeapWord* new_top = compare_to + obj_size;
              if (new_top <= *Universe::heap()->end_addr()) {
                if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
                  goto retry;
                }
                result = (oop) compare_to;
              }
            }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　5.根据是否需要填0选项，对分配空间的对象数据区进行填0

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (result != NULL) {
              // Initialize object (if nonzero size and need) and then the header
              if (need_zero ) {
                HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
                obj_size -= sizeof(oopDesc) / oopSize;
                if (obj_size > 0 ) {
                  memset(to_zero, 0, obj_size * HeapWordSize);
                }
              }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　6.根据是否使用偏向锁，设置对象头信息，然后设置对象的klassOop引用(这样对象本身就获取了获取类型数据的途径)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (UseBiasedLocking) {
                result->set_mark(ik->prototype_header());
              } else {
                result->set_mark(markOopDesc::prototype());
              }
              result->set_klass_gap(0);
              result->set_klass(k_entry);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　7.把对象地址引入栈，并继续执行下一个字节码

```
SET_STACK_OBJECT(result, 0);
              UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
```

　　8.若该类型没有被解析，就会调用InterpreterRuntime的_new函数完成慢速分配

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// Slow case allocation
        CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
                handle_exception);
        SET_STACK_OBJECT(THREAD->vm_result(), 0);
        THREAD->set_vm_result(NULL);
        UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

以上就是快速分配的过程，其流程图如下，关键在于快速分配在Eden区所使用的无锁指针碰撞技术 

　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014154751882-1593932352.png)

### 　二、慢速分配

　　接下来看看慢速分配是如何进行的： 
　　1.InterpreterRuntime的_new函数定义在/hotspot/src/share/vm/interpreter/interpreterRuntime.cpp中：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
IRT_ENTRY(void, InterpreterRuntime::_new(JavaThread* thread, constantPoolOopDesc* pool, int index))
  klassOop k_oop = pool->klass_at(index, CHECK);
  instanceKlassHandle klass (THREAD, k_oop);

  // Make sure we are not instantiating an abstract klass
  klass->check_valid_for_instantiation(true, CHECK);

  // Make sure klass is initialized
  klass->initialize(CHECK);

  oop obj = klass->allocate_instance(CHECK);
  thread->set_vm_result(obj);
IRT_END
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　该函数在进行了对象类的检查(确保不是抽象类)和对该类型进行初始化后，调用instanceKlassHandle的allocate_instance进行内存分配。 
　　其中instanceKlassHandle类由DEF_KLASS_HANDLE宏进行声明，注意该类重载了成员访问运算符”->”，这里的一系列成员方法的访问实际上是instanceKlass对象的访问。

```
 type*    operator -> () const       { return (type*)obj()->klass_part(); }
```

　　2.所以实际上是调用了instanceKlass的allocate_instance()成员函数： 
　　allocate_instance()定义在/hotspot/src/share/vm/oops/instanceKlass.cpp 
　　(1).检查是否设置了Finalizer函数，获取对象所需空间的大小

```
instanceOop instanceKlass::allocate_instance(TRAPS) {
     bool has_finalizer_flag = has_finalizer(); // Query before possible GC
     int size = size_helper();  // Query before forming handle.
```

　　(2).调用CollectedHeap的obj_allocate()创建一个instanceOop(堆上的对象实例)，并根据情况注册Finalizer函数

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
　　　 KlassHandle h_k(THREAD, as_klassOop());

      instanceOop i;

      i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
      if (has_finalizer_flag && !RegisterFinalizersAtInit) {
        i = register_finalizer(i, CHECK_NULL);
      }
      return i;
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.CollectedHeap::ojb_allocate()定义在/hotspot/src/share/vm/gc_interface/CollectedHeap.hpp中，它将转而调用内联函数obj_allocate()

　　4.obj_allocate()定义在/hotspot/src/share/vm/gc_interface/CollectedHeap.inline.h中，若当正处于gc状态时，不允许进行内存分配申请，否则将调用common_mem_allocate_init()进行内存分配并返回获得内存的起始地址，随后将调用post_allocation_setup_obj()进行一些初始化工作　

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
oop CollectedHeap::obj_allocate(KlassHandle klass, int size, TRAPS) {
 //...assert
  HeapWord* obj = common_mem_allocate_init(size, false, CHECK_NULL);
  post_allocation_setup_obj(klass, obj, size);
  NOT_PRODUCT(Universe::heap()->check_for_bad_heap_word_value(obj, size));
  return (oop)obj;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　5.common_mem_allocate_init()分为两部分，将分别调用common_mem_allocate_noinit()进行内存空间的分配和调用init_obj()进行对象空间的初始化

```
HeapWord* CollectedHeap::common_mem_allocate_init(size_t size, bool is_noref, TRAPS) {
  HeapWord* obj = common_mem_allocate_noinit(size, is_noref, CHECK_NULL);
  init_obj(obj, size);
  return obj;
}
```

　　6.common_mem_allocate_noinit()如下： 
　　(1).若使用了本地线程分配缓冲TLAB，则会调用allocate_from_tlab()尝试从TLAB中分配内存

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  HeapWord* result = NULL;
  if (UseTLAB) {
    result = CollectedHeap::allocate_from_tlab(THREAD, size);
    if (result != NULL) {
      assert(!HAS_PENDING_EXCEPTION,
             "Unexpected exception, will result in uninitialized storage");
      return result;
    }
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(2).否则会调用堆的mem_allocate()尝试分配

　

```
  bool gc_overhead_limit_was_exceeded = false;
  result = Universe::heap()->mem_allocate(size,
                                          is_noref,
                                          false,
                                          &gc_overhead_limit_was_exceeded);
```

　　(3).统计分配的字节数

```
 if (result != NULL) {
   //...
    THREAD->incr_allocated_bytes(size * HeapWordSize);
    return result;
  }
```

　　(4).否则说明申请失败，若在申请过程中gc没有超时，则抛出OOM异常

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (!gc_overhead_limit_was_exceeded) {
    // -XX:+HeapDumpOnOutOfMemoryError and -XX:OnOutOfMemoryError support
    report_java_out_of_memory("Java heap space");

    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_JAVA_HEAP,
        "Java heap space");
    }

    THROW_OOP_0(Universe::out_of_memory_error_java_heap());
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　7.对象内存分配后的初始化过程包括两部分，一个是init_obj()完成对对象内存空间的对齐和填充，一个是post_allocation_setup_obj()对堆上的oop对象进行初始化。 

　　(1).init_obj()：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void CollectedHeap::init_obj(HeapWord* obj, size_t size) {
  assert(obj != NULL, "cannot initialize NULL object");
  const size_t hs = oopDesc::header_size();
  assert(size >= hs, "unexpected object size");
  ((oop)obj)->set_klass_gap(0);
  Copy::fill_to_aligned_words(obj + hs, size - hs);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　hs就是对象头的大小，fill_to_aligned_words将对象空间除去对象头的部分做填0处理，该函数定义在/hotspot/src/share/vm/utilities/copy.h中，并转而调用pd_fill_to_aligned_words()。 
　　pd_fill_to_aligned_words根据不同平台实现，以x86平台为例，该函数定义在/hotspot/src/cpu/x86/vm/copy_x86.h中：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
static void pd_fill_to_words(HeapWord* tohw, size_t count, juint value) {
#ifdef AMD64
  julong* to = (julong*) tohw;
  julong  v  = ((julong) value << 32) | value;
  while (count-- > 0) {
    *to++ = v;
  }
#else
  juint* to = (juint*)tohw;
  count *= HeapWordSize / BytesPerInt;
  while (count-- > 0) {
    *to++ = value;
  }
#endif // AMD64
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　该函数的作用就是先将地址类型转换，然后把堆的字数转化为字节数，再对该段内存进行填值(value = 0)处理

　　(2).post_allocation_setup_obj()调用了post_allocation_setup_common()进行初始化工作，然后调用post_allocation_notify()通知JVMTI和dtrace
　　

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void CollectedHeap::post_allocation_setup_obj(KlassHandle klass,
                                              HeapWord* obj,
                                              size_t size) {
  post_allocation_setup_common(klass, obj, size);
  assert(Universe::is_bootstrapping() ||
         !((oop)obj)->blueprint()->oop_is_array(), "must not be an array");
  // notify jvmti and dtrace
  post_allocation_notify(klass, (oop)obj);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　post_allocation_setup_common()如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void CollectedHeap::post_allocation_setup_common(KlassHandle klass,
                                                 HeapWord* obj,
                                                 size_t size) {
  post_allocation_setup_no_klass_install(klass, obj, size);
  post_allocation_install_obj_klass(klass, oop(obj), (int) size);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　post_allocation_setup_no_klass_install()根据是否使用偏向锁，设置对象头信息等，即初始化oop的_mark字段。post_allocation_install_obj_klass()设置对象实例的klassOop引用，即初始化oop的_metadata(_klass/_compressed_klass)字段 。

　　以上内容就是堆实现无关的慢速分配过程，其流程图如下： 

　　　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014155642913-1652270363.png)

### 　三、堆的分配实现

　　1.mem_allocate将由堆的实现类型定义，以GenCollectedHeap为例：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.由之前分析，GenCollectedHeap根据用户配置有着不同的GC策略(默认的和配置UseSerialGC的MarkSweepPolicy、配置UseComcMarkSweepGC和UseAdaptiveSizePolicy的ASConcurrentMarkSweepPolicy、只配置UseComcMarkSweepGC的ConcurrentMarkSweepPolicy)，但这里，对象内存空间的基本结构和分配的思想是一致的，所以统一由GenCollectorPolicy实现进行分代层级的对象分配操作，但具体的工作将交由各代的实现者来完成。

　　GenCollectedPolicy的mem_allocate_work()函数如下： 
　　(1).gch指向GenCollectedHeap堆，内存分配请求将循环不断地进行尝试，直到分配成功或GC后分配失败

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　对于占用空间比较大的对象，如果经常放在新生代，那么剩余的内存空间就会非常紧张，将可能会导致新生代内存垃圾回收的频繁触发。故若对象的大小超过一定值，那么就不应该分配在新生代。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
　　 //...紧接上面部分
 　　HandleMark hm; // discard any handles allocated in each iteration

    // First allocation attempt is lock-free.
    Generation *gen0 = gch->get_gen(0);

    if (gen0->should_allocate(size, is_tlab)) {
      result = gen0->par_allocate(size, is_tlab);
      if (result != NULL) {
        assert(gch->is_in_reserved(result), "result not in heap");
        return result;
      }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　若对象应该在新生代上分配，就会调用新生代的par_allocate()进行分配，注意在新生代普遍是采用复制收集器的，而内存的分配对应采用了无锁式的指针碰撞技术。

　　(2).在新生代上尝试无锁式的分配失败，那么就获取堆的互斥锁，并尝试在各代空间内进行内存分配

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
unsigned int gc_count_before;  // read inside the Heap_lock locked region
    {
      MutexLocker ml(Heap_lock);
     //...
      bool first_only = ! should_try_older_generation_allocation(size);

      result = gch->attempt_allocation(size, is_tlab, first_only);
      if (result != NULL) {
        assert(gch->is_in_reserved(result), "result not in heap");
        return result;
      }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　其中should_try_older_generation_allocation()如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　当进行gc前，新生代的空闲空间大小不足以分配对象，或者有线程触发了gc，或前一次的FullGC是由MinorGC触发的情况，都应该不再尝试再更高的内存代上进行分配，以保证新分配的对象尽可能在新生代空间上。 

　　attempt_allocation()实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　即由低内存代向高内存代尝试分配内存 

　　(3).从各个代空间都找不到可用的空闲内存(或不应该在更高的内存代上分配时)，如果已经有线程触发了gc，那么当各代空间还有virtual space可扩展空间可用时，将会尝试扩展代空间并再次尝试进行内存分配，有点在gc前想尽一切办法获得内存的意思。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(4).否则各代已经没有可用的可扩展空间时，当当前线程没有位于jni的临界区时，将释放堆的互斥锁，以使得请求gc的线程可以进行gc操作，等待所有本地线程退出临界区和gc完成后，将继续循环尝试进行对象的内存分配

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
JavaThread* jthr = JavaThread::current();
        if (!jthr->in_critical()) {
          MutexUnlocker mul(Heap_lock);
          // Wait for JNI critical section to be exited
          GC_locker::stall_until_clear();
          continue;
        }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(5).若各代无法分配对象的内存，并且没有gc被触发，那么当前请求内存分配的线程将发起一次gc，这里将提交给VM一个GenCollectForAllocation操作以触发gc，当操作执行成功并返回时，若gc锁已被获得，那么说明已经由其他线程触发了gc，将继续循环以等待gc完成

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
VM_GenCollectForAllocation op(size,
                                  is_tlab,
                                  gc_count_before);
    VMThread::execute(&op);
    if (op.prologue_succeeded()) {
      result = op.result();
      if (op.gc_locked()) {
         assert(result == NULL, "must be NULL if gc_locked() is true");
         continue;  // retry and/or stall as necessary
      }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　否则将等待gc完成，若gc超时则会将gc_overhead_limit_was_exceeded设置为true返回给调用者，并重置超时状态，并对分配的对象进行填充处理

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
　　　 const bool limit_exceeded = size_policy()->gc_overhead_limit_exceeded();
      const bool softrefs_clear = all_soft_refs_clear();
      assert(!limit_exceeded || softrefs_clear, "Should have been cleared");
      if (limit_exceeded && softrefs_clear) {
        *gc_overhead_limit_was_exceeded = true;
        size_policy()->set_gc_overhead_limit_exceeded(false);
        if (op.result() != NULL) {
          CollectedHeap::fill_with_object(op.result(), size);
        }
        return NULL;
      }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　以上内容就是堆的实现相关、但代/GC实现无关的分配过程，其流程图归纳如下： 

　　　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014160306132-589837693.png)

 

分类: [Hotspot源码探索](https://www.cnblogs.com/iceAeterNa/category/743704.html)

## 源码分析：Java堆的创建

　　虚拟机在内存中申请一片区域，由虚拟机自动管理，用来满足应用程序对象分配的空间需求，即堆空间。

　　由于程序运行的局部特性，程序创建的大多数对象都具有非常短的生命周期，而程序也会创建一些生命周期特别长的对象。简单的复制收集器无论对象的生命周期是长是短，都会进行复制操作。而生命周期较长的对象在多次垃圾回收期间内并不会被回收，这就使得这些对象被来回复制而使得算法性能大大下降。

　　分代收集把堆分为多个子堆，分别用来存放不同寿命的对象。新生对象空间的将经历最频繁的垃圾回收，而对于经历了若干次垃圾收集后仍然存活的对象，将成长为成熟对象，并移动到成熟对象的子堆中，而对老生代子堆的垃圾回收就不会像新生对象子堆那么频繁。

　　HotSpot的堆空间分为新生代(YoungGen)和老年代(OldGen，此外还有位于非堆空间的永久代，但在Java8中将移除永久代)，新生代又分为Eden区和2个Survivor区(From/To)用以进行复制收集垃圾对象。 
对Java堆和对象的分析将从Java堆的创建开始，然后分析Java对象的分配与垃圾回收。

### 　一、堆的实现方式

　  在虚拟机的创建初始化过程中，通过调用Universe的成员函数initialize_heap()将完成Java堆的初始化。在Universe模块下的初始化将根据虚拟机选项来选择堆的具体实现方式： 
　　1.若虚拟机配置UseParallelGC，则Java堆的堆类型为ParallelScavengeHeap(并行收集堆)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
//定义在/hotspot/src/share/vm/memory/universe.cpp中
if (UseParallelGC) {
#ifndef SERIALGC
    Universe::_collectedHeap = new ParallelScavengeHeap();
#else  // SERIALGC
    fatal("UseParallelGC not supported in java kernel vm.");
#endif // SERIALGC

  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.若虚拟机配置UseG1GC，那么将选择堆类型为G1CollectedHeap，垃圾收集策略将使用专用的G1CollectorPolicy(垃圾优先收集)策略

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 else if (UseG1GC) {
#ifndef SERIALGC
    G1CollectorPolicy* g1p = new G1CollectorPolicy_BestRegionsFirst();
    G1CollectedHeap* g1h = new G1CollectedHeap(g1p);
    Universe::_collectedHeap = g1h;
#else  // SERIALGC
    fatal("UseG1GC not supported in java kernel vm.");
#endif // SERIALGC

  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.否则，虚拟机将使用GenCollectedHeap(分代收集堆)

```
Universe::_collectedHeap = new GenCollectedHeap(gc_policy);
```

　　各个堆实现类的类关系如下：

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014150203913-1537118629.png)

　　对于默认情况下的堆实现，还要根据配置选择垃圾回收策略gc_policy来构造一个GenCollectedHeap，这里根据虚拟机配置选择不同的GC策略： 
　　(1).若虚拟机配置UseSerialGC，那么将使用MarkSweepPolicy(标记-清除)策略

```
GenCollectorPolicy *gc_policy;

    if (UseSerialGC) {
      gc_policy = new MarkSweepPolicy();
    }
```

　　(2).若虚拟机配置UseConcMarkSweepGC和UseAdaptiveSizePolicy，那么将使用ASConcurrentMarkSweepPolicy(自适应并发标记-清除)策略，若没有指定UseAdaptiveSizePolicy，虚拟机将默认使用ConcurrentMarkSweepPolicy(并发标记-清除)策略

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
else if (UseConcMarkSweepGC) {
#ifndef SERIALGC
      if (UseAdaptiveSizePolicy) {
        gc_policy = new ASConcurrentMarkSweepPolicy();
      } else {
        gc_policy = new ConcurrentMarkSweepPolicy();
      }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(3).若没有进行配置，虚拟机将默认使用MarkSweepPolicy策略

```
else { // default old generation
      gc_policy = new MarkSweepPolicy();
    }
```

如下表所示：

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151015142210022-1367997635.jpg) 

其中垃圾回收策略类的关系如下图：

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014150851257-1894774377.png)

　　4.接下来是相应实现的堆的初始化

```
jint status = Universe::heap()->initialize();
  if (status != JNI_OK) {
    return status;
  }
```

　　5.堆空间初始化完成后，是LP64平台上的指针压缩以及TLAB的相关内容 。
　　通常64位JVM消耗的内存会比32位的大1.5倍，这是因为在64位环境下，对象将使用64位指针，这就增加了一倍的指针占用内存开销。从JDK 1.6 update14开始，64 bit JVM正式支持了 -XX:+UseCompressedOops 选项来压缩指针，以节省内存空间。 
指针压缩的地址计算如下：

```
addr = <narrow_oop_base> + <narrow_oop> << 3 + <field_offset>
```

　　若堆寻址空间小于4GB(2^32)时，直接使用32位的压缩对象指针< narrow_oop >就可以找到该对象 
　　若堆寻址空间大于4GB(2^32)但小于32GB时，就必须借助偏移来获得真正的地址(对象是8字节对齐的)。 
　　若堆寻址空间大于32GB时，就需要借助堆的基址来完成寻址了，< narrow_oop_base >为堆的基址，< field_offset >为一页的大小。 
　　(1).若heap的地址空间的最大地址大于OopEncodingHeapMax(32GB)，则设置基础地址为当前堆的起始地址-页大小，设置偏移为LogMinObjAlignmentInBytes(3)，即使用普通的对象指针压缩技术

```
if ((uint64_t)Universe::heap()->reserved_region().end() > OopEncodingHeapMax) {
      // Can't reserve heap below 32Gb.
      Universe::set_narrow_oop_base(Universe::heap()->base() - os::vm_page_size());
      Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
}
```

　　(2).否则设置基础地址为0

```
else {
      Universe::set_narrow_oop_base(0);
      //...
      }
```

　　若heap的地址空间的最大地址大于NarrowOopHeapMax(4GB，小于32GB)，则设置偏移为LogMinObjAlignmentInBytes(默认为3)，即使用零基压缩技术，否则设置偏移为0，即直接使用压缩对象指针进行寻址

```
if((uint64_t)Universe::heap()->reserved_region().end() > NarrowOopHeapMax) {
        // Can't reserve heap below 4Gb.
        Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
      } else {
        Universe::set_narrow_oop_shift(0);
```

### 　二、堆的初始化：分代实现方式

　接下来分析特定堆的初始化过程，这里以GenCollectedHeap和MarkSweepPolicy为例： 　

　　GenCollectedHeap的构造函数中使用传入的策略作为_gen_policy(代策略)。以MarkSweepPolicy为例，看看其构造函数：

```
//定义在/hotspot/src/share/vm/memory/collectorPolicy.cpp中
MarkSweepPolicy::MarkSweepPolicy() {
  initialize_all();
}
```

　　MarkSweepPolicy的构造函数调用了initialize_all()来完成策略的初始化，initialize_all()是父类GenCollectorPolicy()的虚函数，它调用了三个子初始化虚函数，这三个子初始化过程由GenCollectorPolicy的子类实现。**其中initialize_flags()初始化了永久代的一些大小配置参数，initialize_size_info()设置了Java堆大小的相关参数，initialize_generations()根据用户参数，配置各内存代的管理器。**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
//hotspot/src/share/vm/memory/collectorPolicy.hpp中
virtual void initialize_all() {
    initialize_flags();
    initialize_size_info();
    initialize_generations();
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　下面通过initialize_generations()来看看各代有哪些实现方式： 
　　1.若配置了UseParNewGC，并且并行GC线程数大于1，那么新生代就会使用ParNew实现

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
//永久代初始化
  _generations = new GenerationSpecPtr[number_of_generations()];
  //...

  if (UseParNewGC && ParallelGCThreads > 0) {
    _generations[0] = new GenerationSpec(Generation::ParNew, _initial_gen0_size, _max_gen0_size);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.默认新生代使用DefNew实现

```
else {
    _generations[0] = new GenerationSpec(Generation::DefNew, _initial_gen0_size, _max_gen0_size);
  }
```

　　3.老年代固定使用MarkSweepCompact实现

```
_generations[1] = new GenerationSpec(Generation::MarkSweepCompact, _initial_gen1_size, _max_gen1_size);
```

(其中DefNew、ParNew、MarkSweepCompact等均为Generation的枚举集合Name的成员，描述了可能实现的各种代实现类型) 
MarkSweepPolicy、ConcurrentMarkSweepPolicy、ASConcurrentMarkSweepPolicy对各代的实现综合如下表所示： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151015142229397-1007054738.jpg) 

### 　三、堆的初始化：堆内存空间分配

　　分析完了构造函数，回到Universe模块中堆的initialize()。 
　　以GenCollectedHeap为例： 
　　1.根据构造函数传入的gc_policy(分代策略)来初始化分代数

```
//定义在/hotspot/src/share/vm/memory/genCollectedHeap.cpp中
jint GenCollectedHeap::initialize() {
  //...
  _n_gens = gen_policy()->number_of_generations();
```

　　根据GenCollectedHeap的定义可以看到，GenCollectedHeap最多支持10个分代

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 enum SomeConstants {
    max_gens = 10
  };

//...
 private:
  int _n_gens;
  Generation* _gens[max_gens];
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　其实并不需要这么多分代，MarkSweepPolicy、ConcurrentMarkSweepPolicy、ASConcurrentMarkSweepPolicy(ConcurrentMarkSweepPolicy的子类)均有着共同的祖先类TwoGenerationCollectorPolicy，其分代只有2代，即新生代和老年代。 

　　2.每代的大小是基于GenGrain大小对齐的

```
 // The heap must be at least as aligned as generations.
  size_t alignment = Generation::GenGrain;
```

GenGrain定义在/hotspot/src/share/vm/memory/generation.h中，在非ARM平台中是2^16字节，即64KB大小 
　　3.获取各分代的管理器指针数组和永久代的管理器指针，并对齐各代的大小到64KB

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
PermanentGenerationSpec *perm_gen_spec =
                                collector_policy()->permanent_generation();

  // Make sure the sizes are all aligned.
  for (i = 0; i < _n_gens; i++) {
    _gen_specs[i]->align(alignment);
  }
  perm_gen_spec->align(alignment);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　GenerationSpec的align()定义在/hotspot/src/share/vm/memory/generationSpec.h，使初始和最大大小值向上对齐至64KB的倍数

```
// Alignment
  void align(size_t alignment) {
    set_init_size(align_size_up(init_size(), alignment));
    set_max_size(align_size_up(max_size(), alignment));
  }
```

　　4.调用allocate()为堆分配空间，其起始地址为heap_address

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
char* heap_address;
  size_t total_reserved = 0;
  int n_covered_regions = 0;
  ReservedSpace heap_rs(0);

  heap_address = allocate(alignment, perm_gen_spec, &total_reserved,
                          &n_covered_regions, &heap_rs);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　5.初始分配所得的空间将被封装在_reserved(CollectedHeap的MemRegion成员)中

```
_reserved = MemRegion((HeapWord*)heap_rs.base(),
                        (HeapWord*)(heap_rs.base() + heap_rs.size()));
```

　　调整实际的堆大小为去掉永久代的misc_data和misc_code的空间，并创建一个覆盖整个空间的数组，数组每个字节对应于堆的512字节，用于遍历新生代和老年代空间

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  _reserved.set_word_size(0);
  _reserved.set_start((HeapWord*)heap_rs.base());
  size_t actual_heap_size = heap_rs.size() - perm_gen_spec->misc_data_size()
                                           - perm_gen_spec->misc_code_size();
  _reserved.set_end((HeapWord*)(heap_rs.base() + actual_heap_size));

  _rem_set = collector_policy()->create_rem_set(_reserved, n_covered_regions);
  set_barrier_set(rem_set()->bs());
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　7.调用heap_rs的的first_part()，依次为新生代和老年代分配空间并调用各代管理器的init()将其初始化为Generation空间，最后为永久代分配空间和进行初始化

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

### 　　四、内存空间申请实现

　　那么GenCollectedHeap是如何向系统申请内存空间的呢？ 
　　答案就在allocate()函数中 
　　1.在申请之前，当然要对内存空间的大小和分块数进行计算 
　　(1).内存页的大小将根据虚拟机是否配置UseLargePages而不同，large_page_size在不同平台上表现不同，x86使用2/4M(物理地址扩展模式)的页大小，AMD64使用2M，否则，Linux默认内存页大小只有4KB，接下来会以各代所配置的最大大小进行计算，若最大值设置为负数，那么jvm将报错退出，默认的新生代和老年代的分块数为1，而永久代的分块数为2

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　加上永久代空间的大小和块数

```
total_reserved += perm_gen_spec->max_size();
if (total_reserved < perm_gen_spec->max_size()) {
    vm_exit_during_initialization(overflow_msg);
  }
  n_covered_regions += perm_gen_spec->n_covered_regions();
```

　　(2).加上永久代的misc_data和misc_code的空间大小(数据区和代码区)，但其实并不是堆的一部分

```
size_t s = perm_gen_spec->misc_data_size() + perm_gen_spec->misc_code_size();

  total_reserved += s;
```

　　(3).如果配置了UseLargePages，那么将向上将申请的内存空间大小对齐至页

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (UseLargePages) {
    assert(total_reserved != 0, "total_reserved cannot be 0");
    total_reserved = round_to(total_reserved, os::large_page_size());
    if (total_reserved < os::large_page_size()) {
      vm_exit_during_initialization(overflow_msg);
    }
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(4).对象地址压缩的内容 
根据UnscaledNarrowOop(直接使用压缩指针)选取合适的堆起始地址，并尝试在该地址上分配内存

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 if (UseCompressedOops) {
      heap_address = Universe::preferred_heap_base(total_reserved, Universe::UnscaledNarrowOop);
      *_total_reserved = total_reserved;
      *_n_covered_regions = n_covered_regions;
      *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                                   UseLargePages, heap_address);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　若不能再该地址进行分配内存，则尝试使用ZereBasedNarrowOop(零基压缩)尝试在更高的地址空间上进行分配

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (heap_address != NULL && !heap_rs->is_reserved()) {
        // Failed to reserve at specified address - the requested memory
        // region is taken already, for example, by 'java' launcher.
        // Try again to reserver heap higher.
        heap_address = Universe::preferred_heap_base(total_reserved, Universe::ZeroBasedNarrowOop);
        *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                                     UseLargePages, heap_address);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　若仍然失败，则使用普通的指针压缩技术在其他地址上进行分配

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.调用ReservedHeapSpace的构造函数进行内存空间的申请

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  *_total_reserved = total_reserved;
  *_n_covered_regions = n_covered_regions;
  *heap_rs = ReservedHeapSpace(total_reserved, alignment,
                               UseLargePages, heap_address);

  return heap_address;
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　在构造函数中并没有发现对内存空间进行申请，那么继续看父类ReservedSpace的构造函数

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
ReservedSpace::ReservedSpace(size_t size, size_t alignment,  
                             bool large,  
                             char* requested_address,  
                             const size_t noaccess_prefix) {  
  initialize(size+noaccess_prefix, alignment, large, requested_address,  noaccess_prefix, false);  
} 
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.initialize()的实现如下： 
　　(1).如果目标操作系统不支持large_page_memory，那么将进行特殊处理，此外，对指针压缩处理还需要对请求分配的内存空间大小进行调整

```
if (requested_address != 0) {  
    requested_address -= noaccess_prefix; // adjust requested address  
    assert(requested_address != NULL, "huge noaccess prefix?");  
  } 
```

　　(2).对于上述特殊情况，会调用reserve_memory_special()进行内存空间的申请，并若申请成功会进行空间大小的对齐验证

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(3).若配置了UseSharedSpace或UseCompressedOops，那么堆将在指定地址进行申请，就会调用attempt_reserve_memory_at()进行申请，否则，调用reserve_memory()进行申请

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(4).若分配成功，还需要对分配的起始地址进行对齐验证。若没有对齐，则会进行手工调整。调整的方法为尝试申请一块size+alignment大小的空间，若成功则向上对齐所得的内存空间的起始地址(失败则无法对齐，直接返回)，并以此为起始地址重新申请一块size大小的空间，这块size大小的空间必然包含于size+alignment大小的空间内，以此达到对齐地址的目的。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　最后，在地址空间均已分配完毕，GenCollectedHeap的initialize()中为各代划分了各自的内存空间范围，就会调用各代的GenerationSpec的init()函数完成各代的初始化。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　各分代实现类的类关系如下： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014152208554-995279278.png)

归纳堆初始化的流程图如下： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014152222866-78373739.png)



## HotSpot模板解释器目标代码生成过程源码分析

　　虽然说解释执行模式是逐字逐句翻译给目标平台运行的，但这样的过程未免太过缓慢，如果能把字节码说的话做成纸条，运行时只要把对应的纸条交给目标平台就可以了，这样，执行速度就会明显提升。JVM的Hotspot虚拟机的模板解释器就是用这种方法来解释执行的。在开始分析之前，先了解一下JVM的执行方式。

　　(1).边解释边运行，即每次解释一条字节码并运行其解释的本地代码，这种执行引擎速度相对很慢 
　　(2).JIT(即时编译)具有更快的运行速度但是需要更多的内存，方法被第一次调用时，字节码编译生成的本地代码将会被缓存，这样在该方法下次被调用的时候，将取出缓冲的本地代码直接运行 
　　(3).自适应优化，对于经常被调用的方法，则会缓存其编译产生成的本地代码，对于其他较少被调用的代码，仍对其采用解释执行的方法。 
　　(4).片上虚拟机，即虚拟机的执行引擎直接嵌入在片上

　　HotSpot虚拟机可以配置为以下运行模式： 
   -Xint：解释模式 
   -Xcomp：编译模式 
   -Xmixed：混合模式 
(通过java -version就可以查看虚拟机的运行模式)

　　HotSpot在启动时，会为所有字节码创建在目标平台上运行的解释运行的机器码，并存放在CodeCache中，在解释执行字节码的过程中，就会从CodeCache中取出这些本地机器码并执行。

　　Hotspot虚拟机的细节技术实现值得借鉴，如果你觉得源码甚至汇编代码比较枯燥的话，也可以大致了解相关模块的组件、工作流程，对相关实现有一定的认识。

　　下面就从模板解释器的初始化开始，分析HotSpot的解释代码的生成。
　　在创建虚拟机时，在初始化全局模块过程中，会调用interpreter_init()初始化模板解释器，模板解释器的初始化包括抽象解释器AbstractInterpreter的初始化、模板表TemplateTable的初始化、CodeCache的Stub队列StubQueue的初始化、解释器生成器InterpreterGenerator的初始化。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreter::initialize() {
 2   if (_code != NULL) return;
 3   // assertions
 4   //...
 5 
 6   AbstractInterpreter::initialize();
 7 
 8   TemplateTable::initialize();
 9 
10   // generate interpreter
11   { ResourceMark rm;
12     TraceTime timer("Interpreter generation", TraceStartupTime);
13     int code_size = InterpreterCodeSize;
14     NOT_PRODUCT(code_size *= 4;)  // debug uses extra interpreter code space
15     _code = new StubQueue(new InterpreterCodeletInterface, code_size, NULL,
16                           "Interpreter");
17     InterpreterGenerator g(_code);
18     if (PrintInterpreter) print();
19   }
20 
21   // initialize dispatch table
22   _active_table = _normal_table;
23 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1.AbstractInterpreter是基于汇编模型的解释器的共同基类，定义了解释器和解释器生成器的抽象接口。 
2.模板表TemplateTable保存了各个字节码的模板(目标代码生成函数和参数)。 
TemplateTable的初始化调用def()将所有字节码的目标代码生成函数和参数保存在_template_table或_template_table_wide(wide指令)模板数组中

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  //                              interpr. templates
  // Java spec bytecodes          ubcp|disp|clvm|iswd  in    out   generator      argument
  def(Bytecodes::_nop           , ____|____|____|____, vtos, vtos, nop           ,  _      );
  def(Bytecodes::_aconst_null   , ____|____|____|____, vtos, atos, aconst_null   ,  _      );
  def(Bytecodes::_iconst_m1     , ____|____|____|____, vtos, itos, iconst        , -1      );
  def(Bytecodes::_iconst_0      , ____|____|____|____, vtos, itos, iconst        ,  0      );
  def(Bytecodes::_iconst_1      , ____|____|____|____, vtos, itos, iconst        ,  1      );
  def(Bytecodes::_iconst_2      , ____|____|____|____, vtos, itos, iconst        ,  2      );
//...其他字节码的模板定义
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其中，def()是查看数组对应项是否为空，若为空则初始化该数组项。

```
Template* t = is_wide ? template_for_wide(code) : template_for(code);
  // setup entry
  t->initialize(flags, in, out, gen, arg);
```

_template_table或_template_table_wide的数组项就是Template对象，即字节码的模板，Template的结构如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
class Template VALUE_OBJ_CLASS_SPEC {
 private:
  enum Flags {
    uses_bcp_bit,                                // set if template needs the bcp pointing to bytecode
    does_dispatch_bit,                           // set if template dispatches on its own
    calls_vm_bit,                                // set if template calls the vm
    wide_bit                                     // set if template belongs to a wide instruction
  };

  typedef void (*generator)(int arg);

  int       _flags;                  // describes interpreter template properties (bcp unknown)
  TosState  _tos_in;                 // tos cache state before template execution
  TosState  _tos_out;                // tos cache state after  template execution
  generator _gen;                    // template code generator
  int       _arg;   
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

_flags为标志，该项的低四位分别标志：

- uses_bcp_bit，标志需要使用字节码指针(byte code pointer，数值为字节码基址+字节码偏移量)
- does_dispatch_bit，标志是否在模板范围内进行转发，如跳转类指令会设置该位
- calls_vm_bit，标志是否需要调用JVM函数
- wide_bit，标志是否是wide指令(使用附加字节扩展全局变量索引)

 _tos_in表示模板执行前的TosState(操作数栈栈顶元素的数据类型，TopOfStack，用来检查模板所声明的输出输入类型是否和该函数一致，以确保栈顶元素被正确使用) 
 _tos_out表示模板执行后的TosState 
 _gen表示模板生成器(函数指针) 
 _arg表示模板生成器参数

3.StubQueue是用来保存生成的本地代码的Stub队列，队列每一个元素对应一个InterpreterCodelet对象，InterpreterCodelet对象继承自抽象基类Stub，包含了字节码对应的本地代码以及一些调试和输出信息。 
其内存结构如下：在对齐至CodeEntryAlignment后，紧接着InterpreterCodelet的就是生成的目标代码。 

　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014105801272-2057292949.png)

4.InterpreterGenerator根据虚拟机使用的解释器模型不同分为别CppInterpreterGenerator和TemplateInterpreterGenerator 
根据不同平台的实现，以x86_64平台为例，TemplateInterpreterGenerator定义在/hotspot/src/cpu/x86/vm/templateInterpreter_x86_64.cpp

```
1 InterpreterGenerator::InterpreterGenerator(StubQueue* code)
2   : TemplateInterpreterGenerator(code) {
3    generate_all(); // down here so it can be "virtual"
4 }
```

(1).在TemplateInterpreterGenerator的generate_all()中，将生成一系列JVM运行过程中所执行的一些公共代码和所有字节码的InterpreterCodelet：

- error exits：出错退出处理入口
- 字节码追踪入口(配置了-XX:+TraceBytecodes)
- 函数返回入口
- JVMTI的EarlyReturn入口
- 逆优化调用返回入口
- native调用返回值处理handlers入口
- continuation入口
- safepoint入口
- 异常处理入口
- 抛出异常入口
- 方法入口(native方法和非native方法)
- 字节码入口

(2).其中，set_entry_points_for_all_bytes()会对所有被定义的字节码生成目标代码并设置对应的入口(这里只考虑is_defined的情况)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreterGenerator::set_entry_points_for_all_bytes() {
 2   for (int i = 0; i < DispatchTable::length; i++) {
 3     Bytecodes::Code code = (Bytecodes::Code)i;
 4     if (Bytecodes::is_defined(code)) {
 5       set_entry_points(code);
 6     } else {
 7       //未被实现的字节码(操作码)
 8       set_unimplemented(i);
 9     }
10   }
11 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

(3).set_entry_points()将取出该字节码对应的Template模板，并调用set_short_enrty_points()进行处理，并将入口地址保存在转发表(DispatchTable)_normal_table或_wentry_table(使用wide指令)中

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreterGenerator::set_entry_points(Bytecodes::Code code) {
 2   CodeletMark cm(_masm, Bytecodes::name(code), code);
 3   // initialize entry points
 4   // ... asserts
 5   address bep = _illegal_bytecode_sequence;
 6   address cep = _illegal_bytecode_sequence;
 7   address sep = _illegal_bytecode_sequence;
 8   address aep = _illegal_bytecode_sequence;
 9   address iep = _illegal_bytecode_sequence;
10   address lep = _illegal_bytecode_sequence;
11   address fep = _illegal_bytecode_sequence;
12   address dep = _illegal_bytecode_sequence;
13   address vep = _unimplemented_bytecode;
14   address wep = _unimplemented_bytecode;
15   // code for short & wide version of bytecode
16   if (Bytecodes::is_defined(code)) {
17     Template* t = TemplateTable::template_for(code);
18     assert(t->is_valid(), "just checking");
19     set_short_entry_points(t, bep, cep, sep, aep, iep, lep, fep, dep, vep);
20   }
21   if (Bytecodes::wide_is_defined(code)) {
22     Template* t = TemplateTable::template_for_wide(code);
23     assert(t->is_valid(), "just checking");
24     set_wide_entry_point(t, wep);
25   }
26   // set entry points
27   EntryPoint entry(bep, cep, sep, aep, iep, lep, fep, dep, vep);
28   Interpreter::_normal_table.set_entry(code, entry);
29   Interpreter::_wentry_point[code] = wep;
30 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这里以非wide指令为例分析set_short_entry_points()。bep(byte entry point), cep, sep, aep, iep, lep, fep, dep, vep分别为指令执行前栈顶元素状态为byte/boolean、char、short、array/reference(对象引用)、int、long、float、double、void类型时的入口地址。

(4).set_short_entry_points()根据操作数栈栈顶元素类型进行判断，首先byte类型、char类型和short类型都应被当做int类型进行处理，对于非void类型将调用generate_and_dispatch()产生目标代码，这里以iconst_0为例对TOS的处理进行介绍： 
对于iconst，其期望的_tos_in(执行前栈顶元素类型)是void类型(vtos)，期望的_tos_out(执行后栈顶元素类型)是int类型(itos)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreterGenerator::set_short_entry_points(Template* t, address& bep, address& cep, address& sep, address& aep, address& iep, address& lep, address& fep, address& dep, address& vep) {
 2   assert(t->is_valid(), "template must exist");
 3   switch (t->tos_in()) {
 4     case btos:
 5     case ctos:
 6     case stos:
 7       ShouldNotReachHere();  // btos/ctos/stos should use itos.
 8       break;
 9     case atos: vep = __ pc(); __ pop(atos); aep = __ pc(); generate_and_dispatch(t); break;
10     case itos: vep = __ pc(); __ pop(itos); iep = __ pc(); generate_and_dispatch(t); break;
11     case ltos: vep = __ pc(); __ pop(ltos); lep = __ pc(); generate_and_dispatch(t); break;
12     case ftos: vep = __ pc(); __ pop(ftos); fep = __ pc(); generate_and_dispatch(t); break;
13     case dtos: vep = __ pc(); __ pop(dtos); dep = __ pc(); generate_and_dispatch(t); break;
14     case vtos: set_vtos_entry_points(t, bep, cep, sep, aep, iep, lep, fep, dep, vep);     break;
15     default  : ShouldNotReachHere();                                                 break;
16   }
17 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其中__定义如下：

```
# define __ _masm->
```

即模板解释器的宏汇编器

(5).以期望的栈顶状态为vtos状态为例，分析set_vtos_entry_points()：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreterGenerator::set_vtos_entry_points(Template* t,
 2                                                          address& bep,
 3                                                          address& cep,
 4                                                          address& sep,
 5                                                          address& aep,
 6                                                          address& iep,
 7                                                          address& lep,
 8                                                          address& fep,
 9                                                          address& dep,
10                                                          address& vep) {
11   assert(t->is_valid() && t->tos_in() == vtos, "illegal template");
12   Label L;
13   aep = __ pc();  __ push_ptr();  __ jmp(L);
14   fep = __ pc();  __ push_f();    __ jmp(L);
15   dep = __ pc();  __ push_d();    __ jmp(L);
16   lep = __ pc();  __ push_l();    __ jmp(L);
17   bep = cep = sep =
18   iep = __ pc();  __ push_i();
19   vep = __ pc();
20   __ bind(L);
21   generate_and_dispatch(t);
22 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

以ftos入口类型为例(vtos即当前字节码的实现不关心栈顶元素的状态)，分析该入口的处理指令： 
push_f()： 
　　定义在 /hotspot/src/cpu/x86/vm/interp_masm_x86_64.cpp中

```
1 void InterpreterMacroAssembler::push_f(XMMRegister r) {
2   subptr(rsp, wordSize);
3   movflt(Address(rsp, 0), r);
4 }
```

　　其中r的默认值为xmm0，wordSize为机器字长(如64位机器为8字节)

subptr()实际上调用了subq()：

```
1 void MacroAssembler::subptr(Register dst, int32_t imm32) {
2   LP64_ONLY(subq(dst, imm32)) NOT_LP64(subl(dst, imm32));
3 }
```

subq()的实现如下：

```
1 void Assembler::subq(Register dst, int32_t imm32) {
2   (void) prefixq_and_encode(dst->encoding());
3   emit_arith(0x81, 0xE8, dst, imm32);
4 }
```

而emit_arith()将调用emit_byte()/emit_long()写入指令的二进制代码”83 EC 08”(由于8可由8位有符号数表示，第一个字节为0x81 | 0x02，即0x83，rsp的寄存器号为4，第二个字节为0xE8 | 0x04，即0xEC，第三个字节为0x08 & 0xFF，即0x08)，该指令即AT&T风格的sub $0x8,%rsp

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void Assembler::emit_arith(int op1, int op2, Register dst, int32_t imm32) {
 2   assert(isByte(op1) && isByte(op2), "wrong opcode");
 3   assert((op1 & 0x01) == 1, "should be 32bit operation");
 4   assert((op1 & 0x02) == 0, "sign-extension bit should not be set");
 5   if (is8bit(imm32)) { //iconst_0的操作数为0，即可以用8位二进制数表示
 6     emit_byte(op1 | 0x02); // set sign bit
 7     emit_byte(op2 | encode(dst));
 8     emit_byte(imm32 & 0xFF);
 9   } else {
10     emit_byte(op1);
11     emit_byte(op2 | encode(dst));
12     emit_long(imm32);
13   }
14 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

emit_byte()定义在/hotspot/src/share/vm/asm/assembler.inlilne.hpp中： 
该函数将把该字节复制到_code_pos处

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 inline void AbstractAssembler::emit_byte(int x) {
2   assert(isByte(x), "not a byte");
3   *(unsigned char*)_code_pos = (unsigned char)x;
4   _code_pos += sizeof(unsigned char);
5   sync();
6 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

故subq()向代码缓冲写入了指令sub $0x8,%rsp 
类似地，movflt()向代码缓冲写入了指令 movss %xmm0,(%rsp) 
jmp()向代码缓冲写入了指令jmpq (addr为字节码的本地代码入口) 

set_vtos_entry_points()产生的入口部分代码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 push %rax        .....(atos entry)
 2 jmpq <addr> 
 3 sub $0x8,%rsp     .....(ftos entry)
 4 movss %xmm0,(%rsp)
 5 jmpq <addr>(addr为字节码的本地代码入口)
 6 sub $0x10,%rsp    .....(dtos entry)
 7 movsd %xmm0,(%rsp)
 8 jmpq <addr>
 9 sub $0x10,%rsp     .....(ltos entry)
10 mov %rax,(%rsp)
11 jmpq <addr>
12 push %rax         ...(itos entry)
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

set_vtos_entry_points()的最后调用generate_and_dispatch()写入了当前字节码的解释代码和跳转到下一个字节码继续执行的逻辑处理部分

generate_and_dispatch()主要内容如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void TemplateInterpreterGenerator::generate_and_dispatch(Template* t, TosState tos_out) {
 2   // ...
 3   // generate template
 4   t->generate(_masm);
 5   // advance
 6   if (t->does_dispatch()) {
 7     //asserts
 8   } else {
 9     // dispatch to next bytecode
10     __ dispatch_epilog(tos_out, step);
11   }
12 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

这里我们以iconst()为目标代码生成器为例，分析generate()：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 void Template::generate(InterpreterMacroAssembler* masm) {
2   // parameter passing
3   TemplateTable::_desc = this;
4   TemplateTable::_masm = masm;
5   // code generation
6   _gen(_arg);
7   masm->flush();
8 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

generate()会调用生成器函数_gen(_arg)，该函数根据平台而不同，如x86_64平台下，定义在/hotspot/src/cpu/x86/vm/templateTable_x86_64.cpp中

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 void TemplateTable::iconst(int value) {
2   transition(vtos, itos);
3   if (value == 0) {
4     __ xorl(rax, rax);
5   } else {
6     __ movl(rax, value);
7   }
8 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

我们知道，iconst_i指令是将i压入栈，这里生成器函数iconst()在i为0时，没有直接将0写入rax，而是使用异或运算清零，即向代码缓冲区写入指令”xor %rax, %rax”；在i不为0时，写入指令”mov $0xi, %rax” 

当不需要转发时，会调用dispatch_epilog()生成取下一条指令和分派的目标代码：

```
1 void InterpreterMacroAssembler::dispatch_epilog(TosState state, int step) {
2   dispatch_next(state, step);
3 }
```

dispatch_next()实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 void InterpreterMacroAssembler::dispatch_next(TosState state, int step) {
2   // load next bytecode (load before advancing r13 to prevent AGI)
3   load_unsigned_byte(rbx, Address(r13, step));
4   // advance r13
5   increment(r13, step);
6   dispatch_base(state, Interpreter::dispatch_table(state));
7 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

dispatch_next()首先调用load_unsigned_byte()写入指令”movzbl (%r13),%rbx”，再调用increment()写入指令”inc/add (,)%r13”指令，最后调用dispatch_base()写入”jmp *(%r10,%rbx,8)”。这类似于PC自增一条指令的宽度再继续取值运行的过程。

 

分析到这里，不禁有一个疑问，_code_pos是哪里？之前说过，StubQueue是用来保存生成的本地代码的Stub队列，队列每一个元素对应一个InterpreterCodelet对象，InterpreterCodelet对象包含了字节码对应的本地代码以及一些调试和输出信息。那么_code_pos是如何和InterpreterCodelet对应的呢？ 

我们注意到无论是为JVM的各种入口函数，还是为字节码生成本地代码，都会构造一个CodeletMark对象

```
CodeletMark cm(_masm, Bytecodes::name(code), code);
```

CodeletMark的构造函数如下：在初始值列表中，调用了StubQueue的request()创建了一个InterpreterCodelet对象，并以该InterpreterCodelet目标代码地址和大小为参数构造了一块CodeBuffer用来存放生成的目标代码。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public:
  CodeletMark(
    InterpreterMacroAssembler*& masm,
    const char* description,
    Bytecodes::Code bytecode = Bytecodes::_illegal):
    _clet((InterpreterCodelet*)AbstractInterpreter::code()->request(codelet_size())),
    _cb(_clet->code_begin(), _clet->code_size())

  { // request all space (add some slack for Codelet data)
    assert (_clet != NULL, "we checked not enough space already");

    // initialize Codelet attributes
    _clet->initialize(description, bytecode);
    // create assembler for code generation
    masm  = new InterpreterMacroAssembler(&_cb);
    _masm = &masm;
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

但在此时还未生成目标代码，所以并不知道生成的目标代码有多大，所以这里会向StubQueue申请全部的空闲空间(只留有一点用来对齐空间，注意StubQueue实际上是一片连续的内存空间，所有Stub都在该空间上进行分配) 
随后初始化该InterpreterCodelet的描述部分和对应字节码，并以该CodeBuffer为参数构造了一个编译器对象InterpreterMacroAssembler

分析到这里，就应该明白编译器的_code_pos指的就是生成代码在CodeBuffer中的当前写位值 
还需一提的就是CodeletMark的析构函数，这里确认编译器的生产代码完全写入到CodeBuffer中后，就会调用StubQueue的commit()将占用的空间划分为当前Stub(InterpreterCodelet)所有

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
~CodeletMark() {
    // align so printing shows nop's instead of random code at the end (Codelets are aligned)
    (*_masm)->align(wordSize);
    // make sure all code is in code buffer
    (*_masm)->flush();


    // commit Codelet
    AbstractInterpreter::code()->commit((*_masm)->code()->pure_insts_size());
    // make sure nobody can use _masm outside a CodeletMark lifespan
    *_masm = NULL;
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

分类: [Hotspot源码探索](https://www.cnblogs.com/iceAeterNa/category/743704.html)

## GC过程

对于HotSpot虚拟机垃圾回收过程，这里将分析介绍默认配置下MarkSweepPolicy的DefNewGeneration和TenuredGeneration的垃圾回收内容以及介绍其他GC策略和代实现的GC思想。GC的过程姑且简单地分为内存代实现无关的GC过程和内存代GC过程。 
本文将先进行内存代实现无关的GC过程分析，内存代GC过程将在后面进行分析。

　　从GenCollectedHeap的do_collection()说起： 
　　1.在GC之前有许多必要的检查和统计任务，比如对回收内存代的统计、堆内存大小的统计等，注意本节内容将不再去分析一些性能统计的内容，有兴趣的可自行分析。 
　　(1).检查是否已经GC锁是否已经激活，并设置需要进行GC的标志为true，这时，通过is_active_and_needs_gc()就可以判断是否已经有线程触发了GC。

```
 if (GC_locker::check_active_before_gc()) {
    return; // GC is disabled (e.g. JNI GetXXXCritical operation)
  }
```

　　(2).检查是否需要回收所有的软引用。

```
 const bool do_clear_all_soft_refs = clear_all_soft_refs ||
                          collector_policy()->should_clear_all_soft_refs();
```

　　(3).记录永久代已经使用的内存空间大小。

```
const size_t perm_prev_used = perm_gen()->used();
```

　　(4).确定回收类型是否是FullGC以及gc触发类型(GC/Full GC(system)/Full GC，用作Log输出)。

```
bool complete = full && (max_level == (n_gens()-1));
    const char* gc_cause_str = "GC ";
    if (complete) {
      GCCause::Cause cause = gc_cause();
      if (cause == GCCause::_java_lang_system_gc) {
        gc_cause_str = "Full GC (System) ";
      } else {
        gc_cause_str = "Full GC ";
      }
    }
```

　　(5).gc计数加1操作(包括总GC计数和FullGC计数)。

```
increment_total_collections(complete);
```

　　(6).统计堆已被使用的空间大小。

```
size_t gch_prev_used = used();
```

　　(7).如果是FullGC，那么从最高的内存代到最低的内存代，若某个内存代不希望对比其更低的内存代进行单独回收，那么就以该内存代作为GC的起始内存代。这里说明下什么是单独回收。新生代比如DefNewGeneration的实现将对新生代使用复制算法进行垃圾回收，而老年代TenuredGeneration的垃圾回收则会使用其标记-压缩-清理算法对新生代也进行处理。所以可以说DefNewGeneration的垃圾回收是对新生代进行单独回收，而TenuredGeneration的垃圾回收则是对老年代和更低的内存代都进行回收。

```
　　int starting_level = 0;
　　if (full) {
      // Search for the oldest generation which will collect all younger
      // generations, and start collection loop there.
      for (int i = max_level; i >= 0; i--) {
        if (_gens[i]->full_collects_younger_generations()) {
          starting_level = i;
          break;
        }
      }
    }
```

　　2.接下来从GC的起始内存代开始，向最老的内存代进行回收 。
　　(1).其中should_collect()将根据该内存代GC条件返回是否应该对该内存代进行GC。若当前回收的内存代是最老的内存代，如果本次gc不是FullGC，将调用increment_total_full_collections()修正之前的FulllGC计数值。

```
　　 int max_level_collected = starting_level;
　　 for (int i = starting_level; i <= max_level; i++) {
      if (_gens[i]->should_collect(full, size, is_tlab)) {
        if (i == n_gens() - 1) {  // a major collection is to happen
          if (!complete) {
            // The full_collections increment was missed above.
            increment_total_full_collections();
          }
```

　　(2).统计GC前该内存代使用空间大小以及其他记录工作 。
　　(3).验证工作 。

　　先调用prepare_for_verify()使各内存代进行验证的准备工作(正常情况下什么都不需要做)，随后调用Universe的verify()进行GC前验证

```
if (VerifyBeforeGC && i >= VerifyGCLevel &&
            total_collections() >= VerifyGCStartAt) {
          HandleMark hm;  // Discard invalid handles created during verification
          if (!prepared_for_verification) {
            prepare_for_verify();
            prepared_for_verification = true;
          }
          gclog_or_tty->print(" VerifyBeforeGC:");
          Universe::verify(true);
        }
```

　　线程、堆(各内存代)、符号表、字符串表、代码缓冲、系统字典等，如对堆的验证将对堆内的每个oop对象的类型Klass进行验证，验证对象是否是oop，类型klass是否在永久代，oop的klass域是否是klass 。那么为什么在这里进行GC验证？GC前验证和GC后验证又分别有什么作用？ VerifyBeforeGC和VerifyAfterGC都需要和UnlockDiagnosticVMOptions配合使用以用来诊断JVM问题，但是验证过程非常耗时，所以在正常的编译版本中并没有将验证内容进行输出。 
　　(4).保存内存代各区域的碰撞指针到该区域的_save_mark_word变量。

```
save_marks();
```

　　(5).初始化引用处理器。

```
ReferenceProcessor* rp = _gens[i]->ref_processor();
if (rp->discovery_is_atomic()) {
            rp->verify_no_references_recorded();
            rp->enable_discovery();
            rp->setup_policy(do_clear_all_soft_refs);
          } else {
            // collect() below will enable discovery as appropriate
          }
```

　　(6).由各内存代完成gc

```
_gens[i]->collect(full, do_clear_all_soft_refs, size, is_tlab);
```

　　(7).将不可触及的引用对象加入到Reference的pending链表

```
if (!rp->enqueuing_is_done()) {
            rp->enqueue_discovered_references();
          } else {
            rp->set_enqueuing_is_done(false);
          }
          rp->verify_no_references_recorded();
        }
```

　　其中enqueue_discovered_references根据是否使用压缩指针选择不同的enqueue_discovered_ref_helper()模板函数 ，enqueue_discovered_ref_helper()实现如下：

```
template <class T>
bool enqueue_discovered_ref_helper(ReferenceProcessor* ref,
                                   AbstractRefProcTaskExecutor* task_executor) {
  T* pending_list_addr = (T*)java_lang_ref_Reference::pending_list_addr();
  T old_pending_list_value = *pending_list_addr;
  ref->enqueue_discovered_reflists((HeapWord*)pending_list_addr, task_executor);

  oop_store(pending_list_addr, oopDesc::load_decode_heap_oop(pending_list_addr));

  ref->disable_discovery();

  return old_pending_list_value != *pending_list_addr;
}
```

　　pending_list_addr是Reference的私有静态(类)成员pending链表的首元素的地址，gc阶段当引用对象的可达状态变化时，会将引用加入到pending链表中，而Reference的私有静态(类)成员ReferenceHandler将不断地从pending链表中取出引用加入ReferenceQueue。 
　　enqueue_discovered_reflists()根据是否使用多线程有着不同的处理方式，若采用多线程则会创建一个RefProcEnqueueTask交由AbstractRefProcTaskExecutor进行处理，这里我们分析单线程的串行处理情况： 
这里，DiscoveredList数组_discoveredSoftRefs保存了最多_max_num_q*subclasses_of_ref个软引用的链表。在将引用链表处理后会将引用链表的起始引用置为哨兵引用，并设置引用链长度为0，表示该列表为空。

```
void ReferenceProcessor::enqueue_discovered_reflists(HeapWord* pending_list_addr,
  AbstractRefProcTaskExecutor* task_executor) {
  if (_processing_is_mt && task_executor != NULL) {
    // Parallel code
    RefProcEnqueueTask tsk(*this, _discoveredSoftRefs,
                           pending_list_addr, sentinel_ref(), _max_num_q);
    task_executor->execute(tsk);
  } else {
    // Serial code: call the parent class's implementation
    for (int i = 0; i < _max_num_q * subclasses_of_ref; i++) {
      enqueue_discovered_reflist(_discoveredSoftRefs[i], pending_list_addr);
      _discoveredSoftRefs[i].set_head(sentinel_ref());
      _discoveredSoftRefs[i].set_length(0);
    }
  }
}
```

　　enqueue_discovered_reflist()如下：

　　取出refs_list链上的首元素，next为discovered域所成链表上的下一个元素

```
  oop obj = refs_list.head();
  while (obj != sentinel_ref()) {
    assert(obj->is_instanceRef(), "should be reference object");
    oop next = java_lang_ref_Reference::discovered(obj);
```

　　如果next是最后的哨兵引用，那么，原子交换discovered域所成链表上的表尾元素与pending_list_addr的值，即将其加入到pending链表的表头，接下来根据插入到表头的链表的处理方式，当pending链表为空时，作为表尾元素其next域指向自身，否则，将其next域指向链表的原表头元素，这样就将该元素插入到pending链表的原表头位置，即：

```
if (next == sentinel_ref()) {  // obj is last
      // Swap refs_list into pendling_list_addr and
      // set obj's next to what we read from pending_list_addr.
      oop old = oopDesc::atomic_exchange_oop(refs_list.head(), pending_list_addr);
      // Need oop_check on pending_list_addr above;
      // see special oop-check code at the end of
      // enqueue_discovered_reflists() further below.
      if (old == NULL) {
        // obj should be made to point to itself, since
        // pending list was empty.
        java_lang_ref_Reference::set_next(obj, obj);
      } else {
        java_lang_ref_Reference::set_next(obj, old);
      } 
```

　　否则若next不是最后的哨兵引用，设置引用对象的next域为next，**即将从引用链表的表头元素开始，将虚拟机所使用的discovered域所成链表转化为Java层可使用的next域所成pending列表。**

```
} else {
      java_lang_ref_Reference::set_next(obj, next);
    }
```

　　最后设置引用对象的discovered域为NULL，即切断当前引用在discovered域所成链表中的引用关系，并继续遍历引用链 　

```
java_lang_ref_Reference::set_discovered(obj, (oop) NULL);
    obj = next;
  }
```

　　综上所述，**入队的操作就是通过原来的discovered域进行遍历，将引用链表用next域重新连接后切断discovered域的关系并将新链表附在pending链表的表头。**

　　(9).回到GC完成后的处理：更新统计信息和进行GC后验证

　　3.输出一些GC的日志信息

```
    complete = complete || (max_level_collected == n_gens() - 1);
    
    if (complete) { // We did a "major" collection
      post_full_gc_dump();   // do any post full gc dumps
    }

    if (PrintGCDetails) {
      print_heap_change(gch_prev_used);

      // Print perm gen info for full GC with PrintGCDetails flag.
      if (complete) {
        print_perm_heap_change(perm_prev_used);
      }
    }
```

　　4.更新各内存代的大小

```
 for (int j = max_level_collected; j >= 0; j -= 1) {
      // Adjust generation sizes.
      _gens[j]->compute_new_size();
    }
```

　　5.FullGC后更新和调整永久代内存大小

```
if (complete) {
      // Ask the permanent generation to adjust size for full collections
      perm()->compute_new_size();
      update_full_collections_completed();
    }
```

6.若配置了ExitAfterGCNum，则当gc次数达到用户配置的最大GC计数时退出VM

```
 if (ExitAfterGCNum > 0 && total_collections() == ExitAfterGCNum) {
    tty->print_cr("Stopping after GC #%d", ExitAfterGCNum);
    vm_exit(-1);
  }
```

### DefNewGeneration的GC过程

由于虚拟机的分代实现，虚拟机不会考虑各个内存代如何实现垃圾回收，具体的工作(对象内存的分配也是一样)由各内存代根据垃圾回收策略自行实现。

　　DefNewGeneration的使用复制算法进行回收。复制算法的思想是将eden和from区活跃的对象复制到to区，并清空eden区和from区，如果to区满了，那么部分对象将会被晋升移动到老年代，随后交换from和to区，即原来的to区存放了存活的对象作为新的from区存在，而from区被清空后当做新的to区而存在，移动次数超过一定阈值的对象也会被移动到老年代。

　　此外，在分析DefNewGeneration的垃圾回收之前，可以了解一下，在垃圾回收过程中，对对象的遍历处理定义一个抽象基类OopClosure(对象表)，并使用其不同的实现类来完成对对象的不同处理。 
其中使用FastScanClosure来处理所有的根对象，FastEvacuateFollowersClosure处理所有的递归引用对象等。

　　在前文分析中，会调用各个内存代的collect()来完成垃圾回收。话不多说，直接上代码： 

### 一、DefNewGeneration的GC基本过程：　　

　　DefNewGeneration::collect()定义在/hotspot/src/share/vm/memory/defNewGeneration.cpp中 。
　　1.当Survivor区空间不足，从Eden区移动过来的对象将会晋升到老年代，然而当老年代空间不足时，那么垃圾回收就是不安全的，将直接返回。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (!collection_attempt_is_safe()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: Collection attempt not safe :: ");
    }
    gch->set_incremental_collection_failed(); // Slight lie: we did not even attempt one
    return;
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　collection_attempt_is_safe()**判断垃圾回收是否安全**有以下判定条件：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(1).To区非空，则可能有不够充足的转移空间 
　　(2).调用下一个内存代的promotion_attempt_is_safe()进行判断，是否有充足的空间容纳新生代的所有对象

　　2.一些准备工作 

　　(1).统计堆的使用空间大小(仅留作输出，可以不管)

　　(2).准备IsAliveClosure、ScanWeakRefClosure。

```
IsAliveClosure is_alive(this);
ScanWeakRefClosure scan_weak_ref(this);
```

　　(3).清空ageTable和to区。

```
age_table()->clear();
to()->clear(SpaceDecorator::Mangle);
```

　　(4).在初始化堆的过程，会创建一个覆盖整个空间的数组GenRemSet，数组每个字节对应于堆的512字节，用于遍历新生代和老年代空间，这里对GenRemSet进行初始化准备。

```
 gch->rem_set()->prepare_for_younger_refs_iterate(false);
```

　　(5).准备FastEvacuateFollowersClosure。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  FastScanClosure fsc_with_no_gc_barrier(this, false);
  FastScanClosure fsc_with_gc_barrier(this, true);

  set_promo_failure_scan_stack_closure(&fsc_with_no_gc_barrier);
  FastEvacuateFollowersClosure evacuate_followers(gch, _level, this,
                                                  &fsc_with_no_gc_barrier,
                                                  &fsc_with_gc_barrier);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.调用GenCollectedHeap的gen_process_strong_roots()将当前代上的根对象复制到转移空间中。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　4.递归处理根集对象的引用对象。

```
  // "evacuate followers".
  evacuate_followers.do_void();
```

　　5.处理发现的引用。

```
FastKeepAliveClosure keep_alive(this, &scan_weak_ref);
  ReferenceProcessor* rp = ref_processor();
  rp->setup_policy(clear_all_soft_refs);
  rp->process_discovered_references(&is_alive, &keep_alive, &evacuate_followers,
                                    NULL);
```

　　6.若没有发生晋升失败： 

　　(1).那么此刻eden区和from区的对象应该已经全部转移了，将调用clear()情况这两片内存区域　。

```
if (!promotion_failed()) {
    // Swap the survivor spaces.
    eden()->clear(SpaceDecorator::Mangle);
    from()->clear(SpaceDecorator::Mangle);
```

　　(2).交换from和to区域，为下次gc做准备。

```
swap_spaces();
```

　　swap_spaces只是交换了_from_space和_to_space的起始地址，并设置eden的下一片需要进行压缩的区域为现在的from区(与TenuredGeneration的标记-压缩-清理垃圾回收相关，用来标志各内存区的压缩顺序)，即原来的to区，而新的from区的下一片需要进行压缩的区域为为NULL。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(3).计算新的survior区域的对象进入老年代的经历的MinorGC次数阈值。

```
 // Set the desired survivor size to half the real survivor space
    _tenuring_threshold =
      age_table()->compute_tenuring_threshold(to()->capacity()/HeapWordSize);
```

　　(4).当gc成功，会重新计算gc超时的时间计数。

```
 AdaptiveSizePolicy* size_policy = gch->gen_policy()->size_policy();
    size_policy->reset_gc_overhead_limit_count();
```

　　7.若发生了晋升失败，即老年代没有足够的内存空间用以存放新生代所晋升的对象： 
　　(1).恢复晋升失败对象的markOop(被标记的活跃对象的markword内容为转发指针，指向经过复制后对象的新地址)。

```
remove_forwarding_pointers();
```

　　remove_forwarding_pointers()会调用RemoveForwardPointerClosure对eden和from区内的对象进行遍历，RemoveForwardPointerClosure将调用其do_object()初始化eden和from区所有对象的对象头部分。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　在晋升失败处理的handle_promotion_failure()中，会将晋升失败对象以<oop, markOop>作为一对分别保存在_objs_with_preserved_marks和_preserved_marks_of_objs栈中，这里就会恢复晋升失败对象的对象头，并清除这两个栈。

　　(2).仍然需要交换from和to区域，设置from的下一片需要进行压缩的区域为to

```
 swap_spaces();
 from()->set_next_compaction_space(to());
```

　　当没有晋升失败是，gc成功，会清空eden和from区、交换from和to区、survivor区对象成熟阈值调整等，以准备下次gc；而当晋升失败时，虽然会在后面交换from和to区，但是并不会清空eden和from区，而是会清空eden和from区所有对象的对象头，而只恢复晋升失败部分的对象头(加上to区的部分就是全部活跃对象了)，这样，在随后触发的FullGC中能够对From和To区进行压缩处理。 

　　(3).设置堆的MinorGC失败标记，并通知老年代(更高的内存代)晋升失败，比如在ConcurrentMarkSweepGeneration会根据配置进行dump输出以供JVM问题诊断

　　

```
gch->set_incremental_collection_failed();

    // Inform the next generation that a promotion failure occurred.
    _next_gen->promotion_failure_occurred();
```

　　8.设置from和to区域的并发遍历指针的安全值为碰撞指针所在位置，并更新堆的最后一次gc的时间

```
  from()->set_concurrent_iteration_safe_limit(from()->top());
  to()->set_concurrent_iteration_safe_limit(to()->top());
  SpecializationStats::print();
  update_time_of_last_gc(os::javaTimeMillis());
```

　　下面将分别对根集对象标记、活跃对象标记、引用处理进行分析：

### 二、DefNewGeneration的根集对象标记过程：　

　　在分析gen_process_strong_roots()之前，首先看下处理函数会做哪些工作： 
　　处理函数封装在之前构造的FastScanClosure中，而FastScanClosure的do_oop()调用了的工作函数do_oop_work()。让我们看看do_oop_work()究竟做了什么。 
　　(1).这里使用模板函数来解决压缩指针的不同类型(实际的oop和压缩指针narrowOop)问题，并当对象非空时，获取该oop/narrowOop对象(narrowOop需要进行指针解压)

```
 T heap_oop = oopDesc::load_heap_oop(p);
  // Should we copy the obj?
  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
```

　　(2).若该对象在遍历区域内(_boudary是在FastScanClosure初始化的时候，为初始化时指定代的结束地址，与当前遍历代的起始地址_gen_boundary共同作为对象的访问边界，故新生代DefNewGeneration会将其自身内存代和更低的内存代的活跃对象都标记复制到to区域中)，若该对象没有被标记过，即其标记状态不为marked_value，就会将该对象复制到to区域内，随后根据是否使用指针压缩将新的对象地址进行压缩

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if ((HeapWord*)obj < _boundary) {
      assert(!_g->to()->is_in_reserved(obj), "Scanning field twice?");
      oop new_obj = obj->is_forwarded() ? obj->forwardee()
                                        : _g->copy_to_survivor_space(obj);
      oopDesc::encode_store_heap_oop_not_null(p, new_obj);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　copy_to_survivor_space()的过程如下： 
　　当该对象占用空间小于应当直接移动到老年代的阈值时，就会将其分配到to区

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
size_t s = old->size();
  oop obj = NULL;

  // Try allocating obj in to-space (unless too old)
  if (old->age() < tenuring_threshold()) {
    obj = (oop) to()->allocate(s);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　否则会尝试将该对象晋升，若晋升失败，则调用handle_promotion_failure()处理

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 if (obj == NULL) {
    obj = _next_gen->promote(old, s);
    if (obj == NULL) {
      handle_promotion_failure(old);
      return old;
    }
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　将原对象的数据内容复制到to区域新分配的对象上，并增加该对象的复制计数和更新ageTable (Prefetch使用的是目标架构的prefetch指令，用于将指定地址和长度的内存预取到cache，用于提升存取性能)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　最后调用forward_to()设置原对象的对象头为转发指针(表示该对象已被复制，并指明该对象已经被复制到什么位置)

```
  // Done, insert forward pointer to obj in this header
  old->forward_to(obj);
```

　　接下来分析gen_process_strong_roots()： 
　　现在考虑一个问题：我们知道，被根对象所触及的所有对象都是活跃对象，那么如何确定一个内存代中的活跃对象呢？或者换个思路，内存代中哪些对象是不可触及的垃圾对象呢？如果其他内存代没有指向该对象的引用并且该对象也没有被内存代内其他对象引用，那么该对象就是一个垃圾对象。据此，把内存代内活跃对象的处理分为两步：第一步，将内存代内正常的根对象和其他内存代内直接引用的内存代内的对象移动到To区域，这些对象作为活跃对象(虽然其他内存代的对象可能在下次Full GC成为垃圾对象，但显然Minor GC显然不能将这些对象当做垃圾对象)，这样，活跃对象的引用判断范围就缩小到了当前内存代，内存代内剩下的对象只要不是被这些活跃对象所引用，那么就必然是垃圾对象了；第二步，递归遍历这些对象，将其所引用的在该内存代的对象移动到To区域。最终，剩下的对象就是垃圾对象了。 
　　1.调用SharedHeap的process_strong_roots()处理根集对象，在当前内存代(新生代的eden和from区)的根集对象将会被复制到to区

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　结合FastScanClosure可知，process_strong_roots()主要将当前内存代上的正常根对象复制到To区域。 
　　2.处理更低的内存代

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(1).内存代的oop_iterate()是调用space_iterate()对该内存代的内存空间进行遍历

```
//定义在/hotspot/src/share/vm/memory/generation.cpp中
void Generation::oop_iterate(OopClosure* cl) {
  GenerationOopIterateClosure blk(cl, _reserved);
  space_iterate(&blk);
}
```

　　(2).space_iterate()由Generation的实现类重写，以OneContigSpaceCardGeneration为例(后面会处理更高的内存代，这里DefNewGeneration并没有更低的内存代)，将遍历代上的内存空间。

```
void OneContigSpaceCardGeneration::space_iterate(SpaceClosure* blk,
                                                 bool usedOnly) {
  blk->do_space(_the_space);
}
```

　　(3).GenerationOopIterateClosure的do_space()如下：

```
virtual void do_space(Space* s) {
    s->object_iterate(_cl);
  }
```

　　(4).space的oop_iterate()根据Eden和from/to的实现如下： 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　该函数的作用是遍历该区域的起始地址到空闲分配指针之间的所有对象，并调用对象的oop_iterate()进行处理。

　　(5).oop是在堆上的对象的基类型，其oop_iterate()调用了Klass的oop_oop_iterate##nv_suffix()

```
inline int oopDesc::oop_iterate(OopClosureType* blk) {                     \
  SpecializationStats::record_call();                                      \
  return blueprint()->oop_oop_iterate##nv_suffix(this, blk);               \
}    
```

　　(6).oop_oop_iterate##nv_suffix()由具体的Klass子类(如对象在堆上的实现instanceKlass)实现，以访问和处理其所包含的引用对象

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　instanceKlass的OopMapBlock描述了在实例对象空间中一连串引用类型域的起始位置和数量，而InstanceKlass_OOP_MAP_ITERATE(是一个语句块)会遍历OopMapBlock的所有块

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　InstanceKlass_SPECIALIZED_OOP_ITERATE如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　其中T为所要处理的对象的指针或压缩指针，start_p为OopMapBlock中引用域的起始地址，count为OopMapBlock中引用的数量，do_oop为引用的处理，assert_fn为断言，该宏所定义的语句块就是将对象引用域的引用调用FastScanClosure的do_oop_nv进行处理。 
　　所以，**对更低内存代的遍历和处理就是把更低内存代的对象在DefNewGeneration内存代所引用的对象移动到To区域。**

　　3.处理更高的内存代

```
for (int i = level+1; i < _n_gens; i++) {
    older_gens->set_generation(_gens[i]);
    rem_set()->younger_refs_iterate(_gens[i], older_gens);
    older_gens->reset_generation();
  }
```

　　类似地，**把更高内存代的对象在DefNewGeneration内存代所引用的对象移动到To区域**。这样就完成了第一步，将回收范围限定在DefNewGeneration内存代内。

 

　　假设堆上有如图所示的对象引用模型：其中深色对象为根对象，箭头代表对象的引用关系，我们主要关注当前内存代(DefNewGeneration)的对象和其处理过程。

　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014170129976-892493116.png)

　　

　　那么根集对象的处理将如下：

　　遍历所有的根对象，将在DefNewGeneration的根对象复制到To区域中，其中橙色对象表示该根对象已被复制移动到To空间，其头部为转发指针：

　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014170310960-751028492.png)

　　将更高或更低内存代所引用的在DefNewGeneration中的对象复制到To区域中

　　　　　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014170819054-896286614.png)

 

### 三、DefNewGeneration的存活对象的递归标记过程：

　　在分析递归标记活跃对象的过程之前，不妨先了解一下递归标记所使用的cheney算法。 
　　在广优先遍历扫描活跃对象的过程中，对于所需的遍历队列，将复用to的从空闲指针开始的一段空间作为隐式队列。在之前，根集对象已经被拷贝到to区域的空闲空间，而scanned指针仍然停留在没有复制根集对象时空闲指针的位置，即scanned指针到当前空闲分配指针(to()->top())的这段空间保存着已经标记的根集对象，所以只需要继续遍历这段空间的根集对象，将发现的引用对象复制到to区域后，让scanned指针更新到这段空间的结束位置，而若还有未标记的对象的话，那么，空间指针必然又前进了一段距离，继续遍历这段新的未处理空间的对象，直至scanned指针追上空闲分配指针即可 
FastEvacuateFollowersClosure的do_void()将完成递归标记工作： 
当各分代的空闲分配指针不在变化时，说明所有可触及对象都已经递归标记完成，否则，将调用oop_since_save_marks_iterate()进行遍历标记。　

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void DefNewGeneration::FastEvacuateFollowersClosure::do_void() {
  do {
    _gch->oop_since_save_marks_iterate(_level, _scan_cur_or_nonheap,
                                       _scan_older);
  } while (!_gch->no_allocs_since_save_marks(_level));
  guarantee(_gen->promo_failure_scan_is_complete(), "Failed to finish scan");
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　1.循环条件oop_since_save_marks_iterate()是对当前代、更高的内存代以及永久代检查其scanned指针_saved_mark_word是否与当前空闲分配指针位置相同，即检查scanned指针是否追上空闲分配指针

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
bool GenCollectedHeap::no_allocs_since_save_marks(int level) {
  for (int i = level; i < _n_gens; i++) {
    if (!_gens[i]->no_allocs_since_save_marks()) return false;
  }
  return perm_gen()->no_allocs_since_save_marks();
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　在DefNewGeneration中，eden和from区的分配指针不应当有所变化，只需要检查to区的空闲分配指针位置是否变化即可

```
bool DefNewGeneration::no_allocs_since_save_marks() {
  assert(eden()->saved_mark_at_top(), "Violated spec - alloc in eden");
  assert(from()->saved_mark_at_top(), "Violated spec - alloc in from");
  return to()->saved_mark_at_top();
}
```

　　2.循环处理oop_since_save_marks_iterate()： 
　　(1).oop_since_save_marks_iterate()是对当前代、更高的内存代以及永久代的对象遍历处理

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　那么为什么要处理更高的内存代对象？因为在复制过程中，有对象通过晋升移动到了更高的内存代。 
　　不过为什么老年代TenuredGeneration不像ConcurrentMarkSweepGeneration一样维护一个晋升对象的链表PromotionInfo来加快晋升对象的处理呢？ 

　　oop_since_save_marks_iterate##nv_suffix()在DefNewGeneration中的定义如下，实际上是调用eden、to、from区的同名函数进行处理，并更新各区的空闲分配指针。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(2).之前说到，在空间分配指针到scanned指针之间的区域就是已分配但未扫描的对象，所以在这里将对这片区域内的对象调用遍历函数进行处理，以标记遍历的对象所引用的对象，并保存新的scanned指针。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

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

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　下面继续递归处理活跃对象的过程：

　　递归标记的开始时，Scanned指针为To区域的起点，Top指向空闲区域的起点，Scanned到Top之间的对象就是需要进行递归标记的对象

　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171006163-1933605595.png)

　　第一轮递归标记后，根集对象中C3引用了C5，C5被移动至To区域，Scanned指针指向已处理完的对象，这时C1、C2、C3均已被遍历完毕，现在C5需要被遍历。其中绿色对象代表被移动到To区域的非根集对象。

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171045210-1834593524.png)

　　第二轮递归标记后，C5引用了C7、C8，这两个对象被移动到了To区域，这时C5已被遍历完毕，现在C7、C8需要被遍历

　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171101476-616715924.png)

![img]()

　　第三轮标记没有任何引用被发现，Scanned指针追上了Top指针，所有存活对象被遍历完毕

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171122382-1596008681.png)

　　在以上分析的基础之上，我们假设还有一个C12为C8所引用，但是To区域没有足够的空间，那么C12就会晋升到更高的内存代(老年代)

 ![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171153897-1373523647.png)

　　最后，将清理Eden和From区域，并交换From和To区域

　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014171227116-82751736.png)

 

### 四、DefNewGeneration的引用处理：

　　1.处理_discoveredSoftRefs数组中的软引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  // Soft references
  {
    TraceTime tt("SoftReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.处理_discoveredWeakRefs数组中的弱引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// Weak references
  {
    TraceTime tt("WeakReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                               is_alive, keep_alive, complete_gc, task_executor);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.处理_discoveredFinalRefs数组中的Final引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// Final references
  {
    TraceTime tt("FinalReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                               is_alive, keep_alive, complete_gc, task_executor);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　4.处理_discoveredPhantomRefs列表中的影子引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// Phantom references
  {
    TraceTime tt("PhantomReference", trace_time, false, gclog_or_tty);
    process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                               is_alive, keep_alive, complete_gc, task_executor);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　5.处理JNI弱全局引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
{
    TraceTime tt("JNI Weak Reference", trace_time, false, gclog_or_tty);
    if (task_executor != NULL) {
      task_executor->set_single_threaded_mode();
    }
    process_phaseJNI(is_alive, keep_alive, complete_gc);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　process_discovered_reflist()过程比较复杂，这里就不继续进行了，有兴趣的可以自行分析，DefNewGeneration的GC流程图如下： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014170020335-1296968176.png)

分类: [Hotspot源码探索](https://www.cnblogs.com/iceAeterNa/category/743704.html)

### TenuredGeneration的GC过程

老年代TenuredGeneration所使用的垃圾回收算法是标记-压缩-清理算法。在回收阶段，将标记对象越过堆的空闲区移动到堆的另一端，所有被移动的对象的引用也会被更新指向新的位置。看起来像是把杂陈的箱子一股脑推到房间的一侧一样。 下面，从TenuredGeneration的collect()开始，分析TenuredGeneration的GC过程。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void TenuredGeneration::collect(bool   full,
                                bool   clear_all_soft_refs,
                                size_t size,
                                bool   is_tlab) {
  retire_alloc_buffers_before_full_gc();
  OneContigSpaceCardGeneration::collect(full, clear_all_soft_refs,
                                        size, is_tlab);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　转而调用了父类OneContigSpaceCardGeneration的collect()：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void OneContigSpaceCardGeneration::collect(bool   full,
                                           bool   clear_all_soft_refs,
                                           size_t size,
                                           bool   is_tlab) {
  SpecializationStats::clear();
  // Temporarily expand the span of our ref processor, so
  // refs discovery is over the entire heap, not just this generation
  ReferenceProcessorSpanMutator
    x(ref_processor(), GenCollectedHeap::heap()->reserved_region());
  GenMarkSweep::invoke_at_safepoint(_level, ref_processor(), clear_all_soft_refs);
  SpecializationStats::print();
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　接着看GenMarkSweep的invoke_at_safepoint()： 
　　1.前面的实现都是进行一些gc前的初始化工作和统计工作 
　　(1).设置引用处理器和引用处理策略为clear_all_softrefs

```
_ref_processor = rp;
  rp->setup_policy(clear_all_softrefs);
```

　　(2).增加调用计数，并统计gc前的堆的使用大小

```
 gch->perm_gen()->stat_record()->invocations++;

  // Capture heap size before collection for printing.
  size_t gch_prev_used = gch->used();
```

　　(3).保存当前内存代和更低的内存代、以及永久代的已使用区域

```
gch->save_used_regions(level, true /* perm */);
```

　　(4).创建遍历栈

```
  allocate_stacks();
```

　　2.接下来就是MarkSweepCompact算法的实现了，算法的实现分为四个阶段： 
　　mark_sweep_phase1-4，其中： 
　　mark_sweep_phase1：递归标记所有活跃对象 
　　mark_sweep_phase2：计算所有活跃对象在压缩后的偏移地址 
　　mark_sweep_phase3：更新对象的引用地址 
　　mark_sweep_phase4：移动所有活跃/存活对象到新的位置

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  mark_sweep_phase1(level, clear_all_softrefs);

  mark_sweep_phase2();

 //...

  mark_sweep_phase3(level);

 //...

  mark_sweep_phase4();
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.在将对象标记入栈的时候，会将原MarkWord保存在_preserved_marks，MarkWord被设置为转发指针，当四个处理阶段结束后，恢复这些”废弃”对象的MarkWord，以防止下次GC时干扰标记，虽然没有真正“清空”死亡对象的内存空间，但由于对象引用将指向新的位置，原来的这些对象所占用内存空间将会被看作是空闲空间。

```
  restore_marks();
```

　　保存各内存代的mark指针为当前空闲分配指针

```
  gch->save_marks();
```

　　4.一些gc后的处理工作，例如清空所有遍历栈、更新堆的一些使用信息和最近一次gc发生的时间等

　　

### mark_sweep_phase1：**递归标记所有活跃对象** 

　　1.与新生代类似，标记根集对象。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  follow_root_closure.set_orig_generation(gch->get_gen(level));

  gch->gen_process_strong_roots(level,
                                false, // Younger gens are not roots.
                                true,  // activate StrongRootsScope
                                true,  // Collecting permanent generation.
                                SharedHeap::SO_SystemClasses,
                                &follow_root_closure,
                                true,   // walk code active on stacks
                                &follow_root_closure);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　follow_root_closure的工作函数如下：

```
void MarkSweep::FollowRootClosure::do_oop(oop* p)       { follow_root(p); }
void MarkSweep::FollowRootClosure::do_oop(narrowOop* p) { follow_root(p); }
```

　　不考虑压缩指针的解压，follow_root()实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
template <class T> inline void MarkSweep::follow_root(T* p) {
  assert(!Universe::heap()->is_in_reserved(p),
         "roots shouldn't be things within the heap");
//...
  T heap_oop = oopDesc::load_heap_oop(p);
  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
    if (!obj->mark()->is_marked()) {
      mark_object(obj);
      obj->follow_contents();
    }
  }
  follow_stack();
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　对于没有被标记的活跃对象，follow_root()会调用mark_object()标记该对象(设置转发指针)，随后调用follow_contents()和follow_stack()处理该对象，根据借助栈进行递归标记的思想，递归标记的过程就是遍历根集对象，把根集对象进行标记后，将其所引用的对象压入栈，然后遍历栈中元素，递归标记活跃对象及其所引用的对象，直至栈空为止。

　　oop_follow_contents()就应该是将当前活跃对象所引用的对象标记并压入栈的过程：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void instanceKlass::oop_follow_contents(oop obj) {
  assert(obj != NULL, "can't follow the content of NULL object");
  obj->follow_header();
  InstanceKlass_OOP_MAP_ITERATE( \
    obj, \
    MarkSweep::mark_and_push(p), \
    assert_is_in_closed_subset)
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　InstanceKlass_OOP_MAP_ITERATE()语句块在”源码分析HotSpot GC过程(二)：DefNewGeneration的GC过程“一文中已经分析过了，其作用就是遍历对象的引用域，使用OopClosure进行处理。 
　　故follow_contents()处理活跃对象就是将该对象标记后，将该对象所引用的对象标记后压入_marking_stack。那么，可以预见，follow_stack()的处理必然就是遍历栈中的对象，并递归地将其引用对象标记和入栈直到栈空为止，那么下面看看follow_stack()的具体实现：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void MarkSweep::follow_stack() {
  do {
    while (!_marking_stack.is_empty()) {
      oop obj = _marking_stack.pop();
      assert (obj->is_gc_marked(), "p must be marked");
      obj->follow_contents();
    }
    // Process ObjArrays one at a time to avoid marking stack bloat.
    if (!_objarray_stack.is_empty()) {
      ObjArrayTask task = _objarray_stack.pop();
      objArrayKlass* const k = (objArrayKlass*)task.obj()->blueprint();
      k->oop_follow_contents(task.obj(), task.index());
    }
  } while (!_marking_stack.is_empty() || !_objarray_stack.is_empty());
}　　
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　那么结果如何呢？好消息是，follow_stack()的前半段确实如此，坏消息是栈空了并不一定会结束，因为，光有一个_marking_stack栈是不够的，对于数组对象的引用如果全都放在标记栈中时，当数组非常大时，就会出现爆栈的问题，这里就需要一个_objArrayKlass和一个ObjArrayTask用来处理数组对象的引用问题。具体的实现这里就不再深入下去了。

　　分析完活跃对象的处理过程，我们回到mark_sweep_phase1()中： 
　　注意gen_process_strong_roots()传入的younger_gens_as_roots参数为false，即不会对更低的内存代进行处理，因为在SharedHeap::process_strong_roots的处理过程中，就已经标记了所有的活跃对象。但是，如果存在更高内存代，那么更低内存代是无法将更高内存代的没有被引用的对象当做垃圾对象处理的，所以虽然不会再处理更低的内存代，但仍要将更高内存代的对象当做根集对象递归遍历。(Hotspot中TenuredGeneration没有更高的内存代了) 
　　2.递归标记发现的引用

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 // Process reference objects found during marking
  {
    ref_processor()->setup_policy(clear_all_softrefs);
    ref_processor()->process_discovered_references(
      &is_alive, &keep_alive, &follow_stack_closure, NULL);
  }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.卸载不再使用的类

```
  // Follow system dictionary roots and unload classes
  bool purged_class = SystemDictionary::do_unloading(&is_alive);
```

　　4.部分类卸载后，需要清理CodeCache，此外，需要清空标记栈

```
// Follow code cache roots
  CodeCache::do_unloading(&is_alive, &keep_alive, purged_class);
  follow_stack(); // Flush marking stack
```

　　5.部分类卸载后，更新存活类的子类、兄弟类、实现类的引用关系

```
  follow_weak_klass_links();
```

　　6.清理未被标记的软引用和弱引用

```
 follow_mdo_weak_refs();
```

　　7.删除拘留字符串表中未被标记的字符串对象

```
StringTable::unlink(&is_alive);
```

　　8.清理符号表中没有被引用的符号

```
SymbolTable::unlink();
```

### mark_sweep_phase2：**计算所有活跃对象在压缩后的偏移地址**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void GenMarkSweep::mark_sweep_phase2() {
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  Generation* pg = gch->perm_gen();

  //...

  VALIDATE_MARK_SWEEP_ONLY(reset_live_oop_tracking(false));

  gch->prepare_for_compaction();

  VALIDATE_MARK_SWEEP_ONLY(_live_oops_index_at_perm = _live_oops_index);
  CompactPoint perm_cp(pg, NULL, NULL);
  pg->prepare_for_compaction(&perm_cp);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　GenCollectedHeap的prepare_for_compaction()如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void GenCollectedHeap::prepare_for_compaction() {
  Generation* scanning_gen = _gens[_n_gens-1];
  // Start by compacting into same gen.
  CompactPoint cp(scanning_gen, NULL, NULL);
  while (scanning_gen != NULL) {
    scanning_gen->prepare_for_compaction(&cp);
    scanning_gen = prev_gen(scanning_gen);
  }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　看到还记得在DefNewGeneration的GC分析中下一片压缩区域的设置么？ 根据内存代的不同实现，如DefNewGeneration分为Eden区(EdenSpace，ContiguousSpace的子类)、From/To区(ContiguousSpace)，TenuredGeneration只有一个_the_space区(ContiguousSpace)，这里直接看ContiguousSpace对prepare_for_compaction的实现：

```
// Faster object search.
void ContiguousSpace::prepare_for_compaction(CompactPoint* cp) {
  SCAN_AND_FORWARD(cp, top, block_is_always_obj, obj_size);
}
```

　　SCAN_AND_FORWARD()，该函数定义在/hotspot/src/share/vm/memory/space.hpp中 
　　1.compact_top为压缩指针，指向压缩的目标内存空间的起始地址，在压缩地址计算的开始，指向当前内存区域的起始位置

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
HeapWord* compact_top; /* This is where we are currently compacting to. */ \
                                                                             \
  /* We're sure to be here before any objects are compacted into this        \
   * space, so this is a good time to initialize this:                       \
   */                                                                        \
  set_compaction_top(bottom());  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　2.初始化CompactPoint，若CompactPoint的压缩区域为空，即这是内存代的第一片区域，那么初始化CompactPoint的压缩区域为内存代的第一片区域，初始化压缩指针为区域的起始地址，初始化区域的压缩的目标区域起始地址为该区域的起始地址，初始化压缩边界为区域边界(默认实现)，若CompactPoint的压缩区域不为空，那么之前继续进行该区域的压缩工作，即初始化压缩指针为原压缩指针的值。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (cp->space == NULL) {                                                   \
   //assert
    cp->space = cp->gen->first_compaction_space();                           \
    compact_top = cp->space->bottom();                                       \
    cp->space->set_compaction_top(compact_top);                              \
    cp->threshold = cp->space->initialize_threshold();                       \
  } else {                                                                   \
    compact_top = cp->space->compaction_top();                               \
  }   
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　3.每经过MarkSweepAlwaysCompactCount次GC，就允许当前区域空间的MarkSweepDeadRatio%(TenuredSpace)/PermMarkSweepDeadRatio%(ContigPermSpace)大小被用来将死亡对象当做存活对象处理，这里姑且将这些对象称为弥留对象，把这片空间称为弥留空间好了(实际上并没有这样的明确定义)。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
nt invocations = SharedHeap::heap()->perm_gen()->stat_record()->invocations;\
  bool skip_dead = ((invocations % MarkSweepAlwaysCompactCount) != 0);       \
                                                                             \
  size_t allowed_deadspace = 0;                                              \
  if (skip_dead) {                                                           \
    const size_t ratio = allowed_dead_ratio();                               \
    allowed_deadspace = (capacity() * ratio / 100) / HeapWordSize;           \
  }  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　4.q为遍历指针，t为扫描边界，end_of_live为最后一个活跃对象的地址，LiveRange保存着死亡对象后面存活对象的地址范围，first_dead为第一个死亡对象的地址

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
HeapWord* q = bottom();                                                    \
  HeapWord* t = scan_limit();                                                \
                                                                             \
  HeapWord*  end_of_live= q;    /* One byte beyond the last byte of the last \
                                   live object. */                           \
  HeapWord*  first_dead = end();/* The first dead object. */                 \
  LiveRange* liveRange  = NULL; /* The current live range, recorded in the   \
                                   first header of preceding free area. */   \
  _first_dead = first_dead; 
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　在边界内遍历，若当前遍历的对象被标记过，即这是一个活跃对象，那么为该对象计算压缩后的地址，设置转发指针，并更新压缩指针和最后一个活跃对象的地址，并继续遍历

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 while (q < t) {   
      if (block_is_obj(q) && oop(q)->is_gc_marked()) {                         \
          /* prefetch beyond q */                                                \
          Prefetch::write(q, interval);                                          \
          /* size_t size = oop(q)->size();  changing this for cms for perm gen */\
          size_t size = block_size(q);                                           \
          compact_top = cp->space->forward(oop(q), size, cp, compact_top);       \
          q += size;                                                             \
          end_of_live = q;                                                       \
    }        
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　否则，跳过死亡对象，遍历直到遇到一个活跃对象为止

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
else {                                                                 \
      /* run over all the contiguous dead objects */                         \
      HeapWord* end = q;                                                     \
      do {                                                                   \
        /* prefetch beyond end */                                            \
        Prefetch::write(end, interval);                                      \
        end += block_size(end);                                              \
      } while (end < t && (!block_is_obj(end) || !oop(end)->is_gc_marked()));
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　若仍有弥留空间可以用，那么在这片空间上调用insert_deadspace()构造弥留对象，当做活跃对象进行压缩的计算处理

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (allowed_deadspace > 0 && q == compact_top) {                       \
        size_t sz = pointer_delta(end, q);                                   \
        if (insert_deadspace(allowed_deadspace, q, sz)) {                    \
          compact_top = cp->space->forward(oop(q), sz, cp, compact_top);     \
          q = end;                                                           \
          end_of_live = end;                                                 \
          continue;                                                          \
        }                                                                    \
      } 
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　更新上一个LiveRange的活跃对象结束地址，这个活跃范围对象设置在死亡对象的MarkWord上，由于在死亡对象后遇到了一个新的活跃对象，于是需要重新构造一个LiveRange对象来记录下一片活跃对象的地址范围。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 if (liveRange) {                                                       \
        liveRange->set_end(q);                                               \
      }
  liveRange = (LiveRange*)q;                                             \
  liveRange->set_start(end);                                             \
  liveRange->set_end(end);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　保存首个死亡对象的地址，并继续遍历

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
/* see if this is the first dead region. */                            \
      if (q < first_dead) {                                                  \
        first_dead = q;                                                      \
      }                                                                      \
                                                                             \
      /* move on to the next object */                                       \
      q = end;                                                               \
    }     
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　循环结束，更新最后一个死亡对象的活跃对象范围、最后一个活跃对象的地址、第一个死亡对象的地址

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (liveRange != NULL) {                                                   \
    liveRange->set_end(q);                                                   \
  }                                                                          \
  _end_of_live = end_of_live;                                                \
  if (end_of_live < first_dead) {                                            \
    first_dead = end_of_live;                                                \
  }                                                                          \
  _first_dead = first_dead;  
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　保存当前空间的压缩指针

```
cp->space->set_compaction_top(compact_top);   
```

### mark_sweep_phase3：**更新对象的引用地址** 

　　1.adjust_root_pointer_closure和adjust_pointer_closure都是静态创建的对象引用地址调整函数的封装对象，这里将调用gen_process_strong_roots()并使用这两个处理函数调整根集对象指针的引用地址。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 adjust_root_pointer_closure.set_orig_generation(gch->get_gen(level));
  adjust_pointer_closure.set_orig_generation(gch->get_gen(level));

  gch->gen_process_strong_roots(level,
                                false, // Younger gens are not roots.
                                true,  // activate StrongRootsScope
                                true,  // Collecting permanent generation.
                                SharedHeap::SO_AllClasses,
                                &adjust_root_pointer_closure,
                                false, // do not walk code
                                &adjust_root_pointer_closure);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　adjust_root_pointer_closure()的工作函数如下：

```
void MarkSweep::AdjustPointerClosure::do_oop(oop* p)       { adjust_pointer(p, _is_root); }
void MarkSweep::AdjustPointerClosure::do_oop(narrowOop* p) { adjust_pointer(p, _is_root); }
```

　　MarkSweep的adjust_pointer将会解析引用对象的MarkWord，若该引用对象已经被标记，就会解析转发指针，并设置引用地址为引用对象新的地址。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
template <class T> inline void MarkSweep::adjust_pointer(T* p, bool isroot) {
  T heap_oop = oopDesc::load_heap_oop(p);
  if (!oopDesc::is_null(heap_oop)) {
    oop obj     = oopDesc::decode_heap_oop_not_null(heap_oop);
    oop new_obj = oop(obj->mark()->decode_pointer());
  //...assert
    if (new_obj != NULL) {
      //...assert
      oopDesc::encode_store_heap_oop_not_null(p, new_obj);
    }
  }
  VALIDATE_MARK_SWEEP_ONLY(track_adjusted_pointer(p, isroot));
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　所以对引用地址的更新就是遍历各内存代对象/引用，若对象所引用的对象已经被标记，则更新其引用地址为转发指针所转向的新地址。 
gen_process_strong_roots()完成了对初始根对象的引用地址更新 

　　2.调整引用指针的引用地址

```
CodeBlobToOopClosure adjust_code_pointer_closure(&adjust_pointer_closure,
                                                   /*do_marking=*/ false);
  gch->gen_process_weak_roots(&adjust_root_pointer_closure,
                              &adjust_code_pointer_closure,
                              &adjust_pointer_closure);
```

　　3.使用GenAdjustPointersClosure遍历各内存代，以更新引用对象的引用地址

```
  GenAdjustPointersClosure blk;
  gch->generation_iterate(&blk, true);
  pg->adjust_pointers();
```

　　其基本思想如图所示： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014172621554-103662546.png)

### mark_sweep_phase4：**移动所有active对象到新的位置** 

　　1.永久代对象压缩，只有在永久代对象压缩后，实例才能获得正确的类数据地址

```
 pg->compact();
```

　　2.使用GenCompactClosure遍历堆上的对象

```
GenCompactClosure blk;
gch->generation_iterate(&blk, true);
```

　　GenCollectedHeap的generation_iterate()将调用GenCompactClosure的do_generation()遍历各个内存代

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void GenCollectedHeap::generation_iterate(GenClosure* cl,
                                          bool old_to_young) {
  if (old_to_young) {
    for (int i = _n_gens-1; i >= 0; i--) {
      cl->do_generation(_gens[i]);
    }
  } else {
    for (int i = 0; i < _n_gens; i++) {
      cl->do_generation(_gens[i]);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　do_generation()实际上是调用各个内存代的compact()进行处理(因为各个内存代的区域组织形式不同，比如新生代有Eden和From/To区，而老年代只有一个区域存在)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
class GenCompactClosure: public GenCollectedHeap::GenClosure {
public:
  void do_generation(Generation* gen) {
    gen->compact();
  }
};
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　compact()调用了CompactibleSpace(ContiguousSpace的父类)的SCAN_AND_COMPACT()完成对象内容的复制

```
void CompactibleSpace::compact() {
  SCAN_AND_COMPACT(obj_size);
}
```

　　SCAN_AND_COMPACT()定义在/hotspot/src/share/vm/memory/space.hpp中 
　　(1).q是遍历指针，t是最后一个活跃对象的位置，记录最后一个活跃对象的位置，就不必再遍历全部内存区域，否则当gc后剩余的活跃对象较少时，将会进行很多不必要的遍历

```
  HeapWord*       q = bottom();                                                 \
  HeapWord* const t = _end_of_live;                                             \
```

　　(2).跳过死亡对象区域

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
if (q < t && _first_dead > q &&                                               \
      !oop(q)->is_gc_marked()) {                                                \
   //...
    HeapWord* const end = _first_dead;                                          \
                                                                                \
    while (q < end) {                                                           \
      size_t size = obj_size(q);                                                \
     //...assert
      q += size;                                                                \
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　当第一个死亡对象的地址与最后一个活跃对象的地址不相同时，即有连续多个死亡对象存在，那么第一个死亡对象的MarkWord就是之前保存的LiveRange，通过LiveRange可以获取下一个活跃对象的地址

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    if (_first_dead == t) {                                                     \
      q = t;                                                                    \
    } else {                                                                    \
      /* $$$ Funky */                                                           \
      q = (HeapWord*) oop(_first_dead)->mark()->decode_pointer();               \
    }    
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(3).开始遍历，对于死亡对象，同样通过LiveRange获取下一个存活对象的地址

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  const intx scan_interval = PrefetchScanIntervalInBytes;                       \
  const intx copy_interval = PrefetchCopyIntervalInBytes;                       \
  while (q < t) {                                                               \
    if (!oop(q)->is_gc_marked()) {                                              \
      /* mark is pointer to next marked oop */                                  \
      q = (HeapWord*) oop(q)->mark()->decode_pointer();                         \
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　(4).复制原对象的数据内容到压缩后的地址，并初始化新的位置的对象的MarkWord

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
else {                                                                    \
      /* prefetch beyond q */                                                   \
      Prefetch::read(q, scan_interval);                                         \
                                                                                \
      /* size and destination */                                                \
      size_t size = obj_size(q);                                                \
      HeapWord* compaction_top = (HeapWord*)oop(q)->forwardee();                \
                                                                                \
      /* prefetch beyond compaction_top */                                      \
      Prefetch::write(compaction_top, copy_interval);                           \
                                                                                \
     //...
      Copy::aligned_conjoint_words(q, compaction_top, size);                    \
      oop(compaction_top)->init_mark();                                         \
     //...
      q += size;                                                                \
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　我们以如下分代和引用模型为基础进行TenuredGeneration的GC分析： 
　　其中蓝色对象为正常的根对象，箭头代表引用关系。 

　　![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014172850991-575986795.png)

 

　　1.MarkSweepPhase1过程分为两步，第一步是递归标记所有根对象： 
　　以根集对象C1为例，借助标记栈的标记过程如下，其中橙色对象为被标记的正常的根对象，绿色为被标记的其他对象： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014172908038-1381134420.png)

　　第二步是递归标记作为更高内存代的对象，这里即为H3，和其所引用的C7对象： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014172929007-751635801.png)

　　2.MarkSweepPhase2过程，以TenuredGeneration为代表进行分析： 
　　C3、C5、C9对象均为垃圾对象，由于前面没有垃圾对象，C1、C2的转发指针均指向自身所在地址 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014172945757-111953938.png)

　　而C3是垃圾对象，C3的MarkWord被设置为LiveRange，指向前一块活跃对象的范围，而C4的转发指针将压缩到C3的起始地址 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014173118538-1141438142.png)

　　以此类推，计算所有活跃对象的转发地址和活跃范围，注意这里压缩指针位置是根据前一次的指针位置和前一个活跃对象的大小计算的

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014173005382-1401758692.png)

　　3.MarkSweepPhase3过程，这里C4、C6、C7、C8的压缩后地址均已发生变化，但对象内容尚未复制到新地址，所以以虚线边框浅色来代表这个”名义上存在的”新对象，并更新引用这些对象的指针地址： 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014173138710-318426870.png)

　　4.MarkSweepPhase3过程，C4、C6、C7、C8的对象内容复制到新地址。 

![img](https://images2015.cnblogs.com/blog/821477/201510/821477-20151014173149819-1168686887.png)

 

分类: [Hotspot源码探索](https://www.cnblogs.com/iceAeterNa/category/743704.html)

## [ZGC gc策略及回收过程](https://www.cnblogs.com/JunFengChan/p/11707542.html)

源码文件：/src/hotspot/share/gc/z/zDirector.cpp

##  一、回收策略

main入口函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void ZDirector::run_service() {
  // Main loop
  while (_metronome.wait_for_tick()) {
    sample_allocation_rate();
    const GCCause::Cause cause = make_gc_decision();
    if (cause != GCCause::_no_gc) {
      ZCollectedHeap::heap()->collect(cause);
    }
  }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

ZMetronome::wait_for_tick 是zgc定义的一个循环时钟函数，sample_allocation_rate函数则用于rule_allocation_rate策略估算可能oom的时间。重点关注：make_gc_decision函数，在判断从make_gc_decision函数返回的结果不是no_gc后，zgc将进行一次gc。

 

make_gc_decision函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
GCCause::Cause ZDirector::make_gc_decision() const {
  // Rule 0: Timer
  if (rule_timer()) {
    return GCCause::_z_timer;
  }

  // Rule 1: Warmup
  if (rule_warmup()) {
    return GCCause::_z_warmup;
  }

  // Rule 2: Allocation rate
  if (rule_allocation_rate()) {
    return GCCause::_z_allocation_rate;
  }

  // Rule 3: Proactive
  if (rule_proactive()) {
    return GCCause::_z_proactive;
  }

  // No GC
  return GCCause::_no_gc;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

make_gc_decision一共提供了4种被动gc策略：

rule 1：固定间隔时间

通过配置ZCollectionInterval参数，可以控制zgc在一个固定的时间间隔进行gc，默认值为0，表示不采用该策略，否则则判断从上次gc到现在的时间间隔是否大于ZCollectionInterval秒，是则gc。源码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
bool ZDirector::rule_timer() const {
  if (ZCollectionInterval == 0) {
    // Rule disabled
    return false;
  }

  // Perform GC if timer has expired.
  const double time_since_last_gc = ZStatCycle::time_since_last();
  const double time_until_gc = ZCollectionInterval - time_since_last_gc;

  log_debug(gc, director)("Rule: Timer, Interval: %us, TimeUntilGC: %.3lfs",
                          ZCollectionInterval, time_until_gc);

  return time_until_gc <= 0;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

rule 2：预热规则

is_warm函数判断gc次数是否已超过3次，是则不使用该策略。

注释说的很清楚，当gc次数少于3时，判断堆使用率达到10%/20%/30%时，使用该策略

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
bool ZDirector::rule_warmup() const {
  if (is_warm()) {
    // Rule disabled
    return false;
  }

  // Perform GC if heap usage passes 10/20/30% and no other GC has been
  // performed yet. This allows us to get some early samples of the GC
  // duration, which is needed by the other rules.
  const size_t max_capacity = ZHeap::heap()->current_max_capacity();
  const size_t used = ZHeap::heap()->used();
  const double used_threshold_percent = (ZStatCycle::ncycles() + 1) * 0.1;
  const size_t used_threshold = max_capacity * used_threshold_percent;

  log_debug(gc, director)("Rule: Warmup %.0f%%, Used: " SIZE_FORMAT "MB, UsedThreshold: " SIZE_FORMAT "MB",
                          used_threshold_percent * 100, used / M, used_threshold / M);

  return used >= used_threshold;
}

bool ZDirector::is_warm() const {
  return ZStatCycle::ncycles() >= 3;
}

// 位置：ZStat.cpp
uint64_t ZStatCycle::ncycles() {
  return _ncycles; // gc次数
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

rule 3：分配速率预估

is_first函数判断如果是首次gc，则直接返回false。

ZAllocationSpikeTolerance默认值为2，分配速率策略采用正态分布模型预测内存分配速率，加上ZAllocationSpikeTolerance修正因子，可以覆盖超过99.9%的内存分配速率的可能性

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
bool ZDirector::rule_allocation_rate() const {
  if (is_first()) {
    // Rule disabled
    return false;
  }

  // Perform GC if the estimated max allocation rate indicates that we
  // will run out of memory. The estimated max allocation rate is based
  // on the moving average of the sampled allocation rate plus a safety
  // margin based on variations in the allocation rate and unforeseen
  // allocation spikes.

  // Calculate amount of free memory available to Java threads. Note that
  // the heap reserve is not available to Java threads and is therefore not
  // considered part of the free memory.
  const size_t max_capacity = ZHeap::heap()->current_max_capacity();
  const size_t max_reserve = ZHeap::heap()->max_reserve();
  const size_t used = ZHeap::heap()->used();
  const size_t free_with_reserve = max_capacity - used;
  const size_t free = free_with_reserve - MIN2(free_with_reserve, max_reserve);

  // Calculate time until OOM given the max allocation rate and the amount
  // of free memory. The allocation rate is a moving average and we multiply
  // that with an allocation spike tolerance factor to guard against unforeseen
  // phase changes in the allocate rate. We then add ~3.3 sigma to account for
  // the allocation rate variance, which means the probability is 1 in 1000
  // that a sample is outside of the confidence interval.
  const double max_alloc_rate = (ZStatAllocRate::avg() * ZAllocationSpikeTolerance) + (ZStatAllocRate::avg_sd() * one_in_1000);
  const double time_until_oom = free / (max_alloc_rate + 1.0); // Plus 1.0B/s to avoid division by zero

  // Calculate max duration of a GC cycle. The duration of GC is a moving
  // average, we add ~3.3 sigma to account for the GC duration variance.
  const AbsSeq& duration_of_gc = ZStatCycle::normalized_duration();
  const double max_duration_of_gc = duration_of_gc.davg() + (duration_of_gc.dsd() * one_in_1000);

  // Calculate time until GC given the time until OOM and max duration of GC.
  // We also deduct the sample interval, so that we don't overshoot the target
  // time and end up starting the GC too late in the next interval.
  const double sample_interval = 1.0 / ZStatAllocRate::sample_hz;
  const double time_until_gc = time_until_oom - max_duration_of_gc - sample_interval;

  log_debug(gc, director)("Rule: Allocation Rate, MaxAllocRate: %.3lfMB/s, Free: " SIZE_FORMAT "MB, MaxDurationOfGC: %.3lfs, TimeUntilGC: %.3lfs",
                          max_alloc_rate / M, free / M, max_duration_of_gc, time_until_gc);

  return time_until_gc <= 0;
}

bool ZDirector::is_first() const {
  return ZStatCycle::ncycles() == 0;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

rule 4：积极回收策略

通过ZProactive可启用积极回收策略，is_warm函数判断启用该策略必须是在预热之后(gc次数超过3次)

自上一次gc后，堆使用率达到xmx的10%或者已过了5分钟，这个参数是弥补第三个规则中没有覆盖的场景，从上述分析可以得到第三个条件更多的覆盖分配速率比较高的场景。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
bool ZDirector::rule_proactive() const {
  if (!ZProactive || !is_warm()) {
    // Rule disabled
    return false;
  }

  // Perform GC if the impact of doing so, in terms of application throughput
  // reduction, is considered acceptable. This rule allows us to keep the heap
  // size down and allow reference processing to happen even when we have a lot
  // of free space on the heap.

  // Only consider doing a proactive GC if the heap usage has grown by at least
  // 10% of the max capacity since the previous GC, or more than 5 minutes has
  // passed since the previous GC. This helps avoid superfluous GCs when running
  // applications with very low allocation rate.
  const size_t used_after_last_gc = ZStatHeap::used_at_relocate_end();
  const size_t used_increase_threshold = ZHeap::heap()->current_max_capacity() * 0.10; // 10%
  const size_t used_threshold = used_after_last_gc + used_increase_threshold;
  const size_t used = ZHeap::heap()->used();
  const double time_since_last_gc = ZStatCycle::time_since_last();
  const double time_since_last_gc_threshold = 5 * 60; // 5 minutes
  if (used < used_threshold && time_since_last_gc < time_since_last_gc_threshold) {
    // Don't even consider doing a proactive GC
    log_debug(gc, director)("Rule: Proactive, UsedUntilEnabled: " SIZE_FORMAT "MB, TimeUntilEnabled: %.3lfs",
                            (used_threshold - used) / M,
                            time_since_last_gc_threshold - time_since_last_gc);
    return false;
  }

  const double assumed_throughput_drop_during_gc = 0.50; // 50%
  const double acceptable_throughput_drop = 0.01;        // 1%
  const AbsSeq& duration_of_gc = ZStatCycle::normalized_duration();
  const double max_duration_of_gc = duration_of_gc.davg() + (duration_of_gc.dsd() * one_in_1000);
  const double acceptable_gc_interval = max_duration_of_gc * ((assumed_throughput_drop_during_gc / acceptable_throughput_drop) - 1.0);
  const double time_until_gc = acceptable_gc_interval - time_since_last_gc;

  log_debug(gc, director)("Rule: Proactive, AcceptableGCInterval: %.3lfs, TimeSinceLastGC: %.3lfs, TimeUntilGC: %.3lfs",
                          acceptable_gc_interval, time_since_last_gc, time_until_gc);

  return time_until_gc <= 0;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

最后，当所有策略都不满足时，返回_no_gc，表示不进行gc

## 二、回收过程

gc整个周期：

![img](https://img2018.cnblogs.com/blog/435918/201910/435918-20191020160005517-765742382.png)

 

彩色指针示意图：

![img](https://img2018.cnblogs.com/blog/435918/201910/435918-20191020160145264-93549046.png)

 

 

 

 

- **(STW)**Pause Mark Start，开始标记，这个阶段只会标记(**Mark0**)由root引用的object，组成Root Set
- Concurrent Mark，并发标记，从Root Set出发，并发遍历Root Set object的引用链并标记(**Mark1**)
- **(STW)**Pause Mark End，检查是否已经并发标记完成，如果不是，需要进行多一次Concurrent Mark
- Concurrent Process Non-Strong References，并发处理弱引用
- Concurrent Reset Relocation Set
- Concurrent Destroy Detached Pages
- Concurrent Select Relocation Set，并发选择Relocation Set；
- Concurrent Prepare Relocation Set，并发预处理Relocation Set
- **(STW)**Pause Relocate Start，开始转移对象，依然是遍历root引用
- Concurrent Relocate，并发转移，将需要回收的Page里的对象转移到Relocation Set，然后回收Page给系统重新利用

run_gc_cycle函数（/src/hotspot/share/gc/z/zDriver.cpp）：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
void ZDriver::run_gc_cycle(GCCause::Cause cause) {
  ZDriverCycleScope scope(cause);

  // Phase 1: Pause Mark Start
  {
    ZMarkStartClosure cl;
    vm_operation(&cl);
  }

  // Phase 2: Concurrent Mark
  {
    ZStatTimer timer(ZPhaseConcurrentMark);
    ZHeap::heap()->mark();
  }

  // Phase 3: Pause Mark End
  {
    ZMarkEndClosure cl;
    while (!vm_operation(&cl)) {
      // Phase 3.5: Concurrent Mark Continue
      ZStatTimer timer(ZPhaseConcurrentMarkContinue);
      ZHeap::heap()->mark();
    }
  }

  // Phase 4: Concurrent Process Non-Strong References
  {
    ZStatTimer timer(ZPhaseConcurrentProcessNonStrongReferences);
    ZHeap::heap()->process_non_strong_references();
  }

  // Phase 5: Concurrent Reset Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentResetRelocationSet);
    ZHeap::heap()->reset_relocation_set();
  }

  // Phase 6: Concurrent Destroy Detached Pages
  {
    ZStatTimer timer(ZPhaseConcurrentDestroyDetachedPages);
    ZHeap::heap()->destroy_detached_pages();
  }

  // Phase 7: Concurrent Select Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentSelectRelocationSet);
    ZHeap::heap()->select_relocation_set();
  }

  // Phase 8: Concurrent Prepare Relocation Set
  {
    ZStatTimer timer(ZPhaseConcurrentPrepareRelocationSet);
    ZHeap::heap()->prepare_relocation_set();
  }

  // Phase 9: Pause Relocate Start
  {
    ZRelocateStartClosure cl;
    vm_operation(&cl);
  }

  // Phase 10: Concurrent Relocate
  {
    ZStatTimer timer(ZPhaseConcurrentRelocated);
    ZHeap::heap()->relocate();
  }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

## [对象内存布局、synchronized终极原理 ](https://www.cnblogs.com/dennyzhangdd/p/6734638.html)

**目录**

- 一、启蒙知识预热
  - [1.1.cas操作](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label0_0)
  - [1.2.对象头](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label0_1)
- [二、JVM中synchronized锁实现原理（优化）](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label1)
- [2.1.偏向锁](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label2)
- [2.2.轻量级锁和重量级锁](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label3)
- 三、从C++源码看synchronized
  - [3.1 同步和互斥](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label4_0)
  - [3.2 synchronized C++源码](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label4_1)
- [四.总结](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label5)

 

**正文**

 很多人一提到锁，自然第一个想到了synchronized，但一直不懂源码实现，现特地追踪到C++层来剥开synchronized的面纱。

网上的很多描述大都不全，让人看了不够爽，看完本章，你将彻底了解synchronized的核心原理。

[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 一、启蒙知识预热

开启本文之前先介绍2个概念



### 1.1.cas操作

为了提高性能，JVM很多操作都依赖CAS实现，一种乐观锁的实现。本文锁优化中大量用到了CAS，故有必要先分析一下CAS的实现。

CAS：Compare and Swap。

JNI来完成CPU指令的操作：

unsafe.compareAndSwapInt(this, valueOffset, expect, update);

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。如果A=V，那么把B赋值给V，返回V；如果A！=V，直接返回V。

打开源码：openjdk\hotspot\src\os*cpu\windows*x86\vm\ atomic*windows*x86.inline.hpp，如下图：0

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170420182404587-2056746472.png)

os::is_MP() 这个是runtime/os.hpp，实际就是返回是否多处理器，源码如下：

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170420185547446-1592190732.png)

 

如上面源代码所示（看第一个int参数即可），LOCK_IF_MP:会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

 



### 1.2.对象头

HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

HotSpot虚拟机的对象头(Object Header)包括两部分信息:

第一部分"Mark Word":用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等.

第二部分"Klass Pointer"：对象指向它的类的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。(数组，对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。 )

32位的HotSpot虚拟机对象头存储结构：（下图摘自网络）

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170420091115212-1624858175.jpg)

​                               图1 32位的HotSpot虚拟机对象头Mark Word组成

为了证实上图的正确性，这里我们看openJDK--》hotspot源码markOop.hpp，虚拟机对象头存储结构：

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170420092816227-246023476.jpg)

​                       图2 HotSpot源码markOop.hpp中注释

单词解释：

**hash：** 保存对象的哈希码
**age：** 保存对象的分代年龄
**biased_lock：** 偏向锁标识位
**lock：** 锁状态标识位
**JavaThread\*：** 保存持有偏向锁的线程ID
**epoch：** 保存偏向时间戳

上图中有源码中对锁标志位这样枚举：

```
1 enum {   locked_value             = 0,//00 轻量级锁
2          unlocked_value           = 1,//01 无锁
3          monitor_value            = 2,//10 监视器锁，也叫膨胀锁，也叫重量级锁
4          marked_value             = 3,//11 GC标记
5          biased_lock_pattern      = 5 //101 偏向锁
6   };
```

下面是源码注释：

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170420152450040-2037151928.png)

​                    图3 HotSpot源码markOop.hpp中锁标志位注释

看图3，不管是32/64位JVM，都是1bit偏向锁+2bit锁标志位。上面红框是偏向锁（第一行是指向线程的显示偏向锁，第二行是匿名偏向锁）对应枚举biased_lock_pattern，下面红框是轻量级锁、无锁、监视器锁、GC标记，分别对应上面的前4种枚举。我们甚至能看见锁标志11时，是GC的markSweep(标记清除算法)使用的。（这里就不再拓展了）

对象头中的Mark Word，synchronized源码实现就用了Mark Word来标识对象加锁状态。

[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 二、JVM中synchronized锁实现原理（优化）

大家都知道java中锁synchronized性能较差，线程会阻塞。本节将以图文形式来描述JVM的synchronized锁优化。

在jdk1.6中对锁的实现引入了大量的优化来减少锁操作的开销：

```
锁粗化（Lock Coarsening）：将多个连续的锁扩展成一个范围更大的锁，用以减少频繁互斥同步导致的性能损耗。

锁消除（Lock Elimination）：JVM及时编译器在运行时，通过逃逸分析，如果判断一段代码中，堆上的所有数据不会逃逸出去从来被其他线程访问到，就可以去除这些锁。

轻量级锁（Lightweight Locking）：JDK1.6引入。在没有多线程竞争的情况下避免重量级互斥锁，只需要依靠一条CAS原子指令就可以完成锁的获取及释放。

偏向锁（Biased Locking）：JDK1.6引入。目的是消除数据再无竞争情况下的同步原语。使用CAS记录获取它的线程。下一次同一个线程进入则偏向该线程，无需任何同步操作。

适应性自旋（Adaptive Spinning）：为了避免线程频繁挂起、恢复的状态切换消耗。产生了忙循环（循环时间固定），即自旋。JDK1.6引入了自适应自旋。自旋时间根据之前锁自旋时间和线程状态，动态变化，用以期望能减少阻塞的时间。
```

 锁升级：**偏向锁--》轻量级锁--》重量级锁**

[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 2.1.偏向锁

```
　　按照之前的HotSpot设计，每次加锁/解锁都会涉及到一些CAS操作（比如对等待队列的CAS操作），CAS操作会延迟本地调用，因此偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象“偏向”这个线程，之后的多次调用则可以避免CAS操作。
　　简单的讲，就是在锁对象的对象头（开篇讲的对象头数据存储结构）中有个ThreaddId字段，这个字段如果是空的，第一次获取锁的时候，就将自身的ThreadId写入到锁的ThreadId字段内，将锁头内的是否偏向锁的状态位置1.这样下次获取锁的时候，直接检查ThreadId是否和自身线程Id一致，如果一致，则认为当前线程已经获取了锁，因此不需再次获取锁，略过了轻量级锁和重量级锁的加锁阶段。提高了效率。
注意：当锁有竞争关系的时候，需要解除偏向锁，进入轻量级锁。
```

每一个线程在准备获取共享资源时：

#### 第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁”.跳过轻量级锁直接执行同步体。

```
获得偏向锁如下图：
```

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170419194339446-1408410540.png)

 

[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 2.2.轻量级锁和重量级锁

![img](https://images2015.cnblogs.com/blog/584866/201704/584866-20170419191951321-2145960409.png)

如上图所示：

#### 第二步，如果MarkWord不是自己的ThreadId,锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。

#### 第三步，两个线程都把对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作，把共享对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord.

#### 第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋.

#### 第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于轻量级锁的状态，如果自旋失败 第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己.

注意点：JVM加锁流程

**偏向锁--》轻量级锁--》重量级锁**

从左往右可以升级，从右往左不能降级



[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 三、从C++源码看synchronized

前两节讲了synchronized锁实现原理，这一节我们从C++源码来剖析synchronized。



### 3.1 同步和互斥

同步：多个线程并发访问共享资源时，保证同一时刻只有一个（信号量可以多个）线程使用。

实现同步的方法有很多，常见四种如下：

1）临界区（CriticalSection，又叫关键段）：通过对多线程的串行化来访问公共资源或一段代码，速度快，适合控制数据访问。进程内可用。

2）互斥量：**互斥量用于线程的互斥。\**只能为0/1\**。**一个互斥量只能用于一个资源的互斥访问，可跨进程使用。

3）信号量：**信号线用于线程的同步。\**可以为非负整数\**，**可实现多个同类资源的多线程互斥和同步。当信号量为单值信号量是，也可以完成一个资源的互斥访问。可跨进程使用。

4）事件：用来通知线程有一些事件已发生，从而启动后继任务的开始,可跨进程使用。

*synchronized的底层实现就用到了临界区和互斥锁（重量级锁的情况下）这两个概念。*



### 3.2 synchronized C++源码

重点来了，之前在第一节中的图1，看过了对象头Mark Word。现在我们从C++源码来剖析具体的数据结构和获取释放锁的过程。

2.2.1 C++中的监视器锁数据结构

oopDesc--继承-->markOopDesc--方法monitor()-->ObjectMonitor-->enter、exit 获取、释放锁

1.oopDesc类

openjdk\hotspot\src\share\vm\oops\oop.hpp下oopDesc类是JVM对象的顶级基类,故每个object都包含markOop。如下图所示：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 class oopDesc {
 2   friend class VMStructs;
 3  private:
 4   volatile markOop  _mark;//markOop:Mark Word标记字段
 5   union _metadata {
 6     Klass*      _klass;//对象类型元数据的指针
 7     narrowKlass _compressed_klass;
 8   } _metadata;
 9 
10   // Fast access to barrier set.  Must be initialized.
11   static BarrierSet* _bs;
12 
13  public:
14   markOop  mark() const         { return _mark; }
15   markOop* mark_addr() const    { return (markOop*) &_mark; }
16 
17   void set_mark(volatile markOop m)      { _mark = m;   }
18 
19   void    release_set_mark(markOop m);
20   markOop cas_set_mark(markOop new_mark, markOop old_mark);
21 
22   // Used only to re-initialize the mark word (e.g., of promoted
23   // objects during a GC) -- requires a valid klass pointer
24   void init_mark();
25 
26   Klass* klass() const;
27   Klass* klass_or_null() const volatile;
28   Klass** klass_addr();
29   narrowKlass* compressed_klass_addr();....省略...}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

2.markOopDesc类

openjdk\hotspot\src\share\vm\oops\markOop.hpp下markOopDesc继承自oopDesc，并拓展了自己的方法monitor(),如下图

```
1 ObjectMonitor* monitor() const {
2     assert(has_monitor(), "check");
3     // Use xor instead of &~ to provide one extra tag-bit check.
4     return (ObjectMonitor*) (value() ^ monitor_value);
5   }
```

该方法返回一个ObjectMonitor*对象指针。

其中value()这样定义：

 1 uintptr_t value() const { return (uintptr_t) this; }

可知：将this转换成一个指针宽度的整数（uintptr_t），然后进行"异或"位操作。

```
monitor_value是常量
1 enum {   locked_value             = 0,//00轻量级锁
2          unlocked_value           = 1,//01无锁
3          monitor_value            = 2,//10监视器锁，又叫重量级锁
4          marked_value             = 3,//11GC标记
5          biased_lock_pattern      = 5 //101偏向锁
6   };
指针低2位00，异或10，结果还是10.（拿一个模板为00的数，异或一个二位数=数本身。因为异或：“相同为0，不同为1”.操作）
```

3.ObjectMonitor类

在HotSpot虚拟机中，最终采用ObjectMonitor类实现monitor。

openjdk\hotspot\src\share\vm\runtime\objectMonitor.hpp源码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 ObjectMonitor() {
 2     _header       = NULL;//markOop对象头
 3     _count        = 0;
 4     _waiters      = 0,//等待线程数
 5     _recursions   = 0;//重入次数
 6     _object       = NULL;//监视器锁寄生的对象。锁不是平白出现的，而是寄托存储于对象中。
 7     _owner        = NULL;//指向获得ObjectMonitor对象的线程或基础锁
 8     _WaitSet      = NULL;//处于wait状态的线程，会被加入到wait set；
 9     _WaitSetLock  = 0 ;
10     _Responsible  = NULL ;
11     _succ         = NULL ;
12     _cxq          = NULL ;
13     FreeNext      = NULL ;
14     _EntryList    = NULL ;//处于等待锁block状态的线程，会被加入到entry set；
15     _SpinFreq     = 0 ;
16     _SpinClock    = 0 ;
17     OwnerIsThread = 0 ;// _owner is (Thread *) vs SP/BasicLock
18     _previous_owner_tid = 0;// 监视器前一个拥有者线程的ID
19   }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

每个线程都有两个ObjectMonitor对象列表，分别为free和used列表，如果当前free列表为空，线程将向全局global list请求分配ObjectMonitor。

ObjectMonitor对象中有两个队列：_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表；

![img](https://images2017.cnblogs.com/blog/584866/201708/584866-20170809154230261-239335176.png)

2.获取锁流程

 synchronized关键字修饰的代码段，在JVM被编译为monitorenter、monitorexit指令来获取和释放互斥锁.。

解释器执行monitorenter时会进入到`InterpreterRuntime.cpp`的`InterpreterRuntime::monitorenter`函数，具体实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
 2 #ifdef ASSERT
 3   thread->last_frame().interpreter_frame_verify_monitor(elem);
 4 #endif
 5   if (PrintBiasedLockingStatistics) {
 6     Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
 7   }
 8   Handle h_obj(thread, elem->obj());
 9   assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
10          "must be NULL or an object");
11   if (UseBiasedLocking) {//标识虚拟机是否开启偏向锁功能,默认开启
12     // Retry fast entry if bias is revoked to avoid unnecessary inflation
13     ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
14   } else {
15     ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
16   }
17   assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
18          "must be NULL or an object");
19 #ifdef ASSERT
20   thread->last_frame().interpreter_frame_verify_monitor(elem);
21 #endif
22 IRT_END
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

先看一下入参：

1、JavaThread thread指向java中的当前线程；
2、BasicObjectLock基础对象锁：包含一个BasicLock和一个指向Object对象的指针oop。

```
openjdk\hotspot\src\share\vm\runtime\basicLock.hpp中BasicObjectLock类源码如下：
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 class BasicObjectLock VALUE_OBJ_CLASS_SPEC {
 2   friend class VMStructs;
 3  private:
 4   BasicLock _lock;                                    // the lock, must be double word aligned
 5   oop       _obj;                                     // object holds the lock;
 6 
 7  public:
 8   // Manipulation
 9   oop      obj() const                                { return _obj;  }
10   void set_obj(oop obj)                               { _obj = obj; }
11   BasicLock* lock()                                   { return &_lock; }
12 
13   // Note: Use frame::interpreter_frame_monitor_size() for the size of BasicObjectLocks
14   //       in interpreter activation frames since it includes machine-specific padding.
15   static int size()                                   { return sizeof(BasicObjectLock)/wordSize; }
16 
17   // GC support
18   void oops_do(OopClosure* f) { f->do_oop(&_obj); }
19 
20   static int obj_offset_in_bytes()                    { return offset_of(BasicObjectLock, _obj);  }
21   static int lock_offset_in_bytes()                   { return offset_of(BasicObjectLock, _lock); }
22 };
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

3、BasicLock类型_lock对象主要用来保存：指向Object对象的对象头数据；

```
basicLock.hpp中BasicLock源码如下：
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 class BasicLock VALUE_OBJ_CLASS_SPEC {
 2   friend class VMStructs;
 3  private:
 4   volatile markOop _displaced_header;//markOop是不是很熟悉？1.2节中讲解对象头时就是分析的markOop源码
 5  public:
 6   markOop      displaced_header() const               { return _displaced_header; }
 7   void         set_displaced_header(markOop header)   { _displaced_header = header; }
 8 
 9   void print_on(outputStream* st) const;
10 
11   // move a basic lock (used during deoptimization
12   void move_to(oop obj, BasicLock* dest);
13 
14   static int displaced_header_offset_in_bytes()       { return offset_of(BasicLock, _displaced_header); }
15 };
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#### `偏向锁的获取ObjectSynchronizer::fast_enter`

在HotSpot中，偏向锁的入口位于openjdk\hotspot\src\share\vm\runtime\synchronizer.cpp文件的`ObjectSynchronizer::fast_enter`函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 2  if (UseBiasedLocking) {
 3     if (!SafepointSynchronize::is_at_safepoint()) {
 4       BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
 5       if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
 6         return;
 7       }
 8     } else {
 9       assert(!attempt_rebias, "can not rebias toward VM thread");
10       BiasedLocking::revoke_at_safepoint(obj);
11     }
12     assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
13  }
14  //轻量级锁
15  slow_enter (obj, lock, THREAD) ;
16 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

偏向锁的获取由`BiasedLocking::revoke_and_rebias`方法实现，由于实现比较长，就不贴代码了，实现逻辑如下：
1、通过`markOop mark = obj->mark()`获取对象的markOop数据mark，即对象头的Mark Word；
2、判断mark是否为可偏向状态，即mark的偏向锁标志位为 **1**，锁标志位为 **01**；
3、判断mark中JavaThread的状态：如果为空，则进入步骤（4）；如果指向当前线程，则执行同步代码块；如果指向其它线程，进入步骤（5）；
4、通过CAS原子指令设置mark中JavaThread为当前线程ID，如果执行CAS成功，则执行同步代码块，否则进入步骤（5）；
5、如果执行CAS失败，表示当前存在多个线程竞争锁，当达到全局安全点（safepoint），获得偏向锁的线程被挂起，撤销偏向锁，并升级为轻量级，升级完成后被阻塞在安全点的线程继续执行同步代码块；

##### 偏向锁的撤销

只有当其它线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，偏向锁的撤销由`BiasedLocking::revoke_at_safepoint`方法实现：

 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
 2   assert(SafepointSynchronize::is_at_safepoint(), "must only be called while at safepoint");//校验全局安全点
 3   oop obj = h_obj();
 4   HeuristicsResult heuristics = update_heuristics(obj, false);
 5   if (heuristics == HR_SINGLE_REVOKE) {
 6     revoke_bias(obj, false, false, NULL);
 7   } else if ((heuristics == HR_BULK_REBIAS) ||
 8              (heuristics == HR_BULK_REVOKE)) {
 9     bulk_revoke_or_rebias_at_safepoint(obj, (heuristics == HR_BULK_REBIAS), false, NULL);
10   }
11   clean_up_cached_monitor_info();
12 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、偏向锁的撤销动作必须等待全局安全点；
2、暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态；
3、撤销偏向锁，恢复到无锁（标志位为 **01**）或轻量级锁（标志位为 **00**）的状态；

偏向锁在Java 1.6之后是默认启用的，但在应用程序启动几秒钟之后才激活，可以使用`-XX:BiasedLockingStartupDelay=0`参数关闭延迟，如果确定应用程序中所有锁通常情况下处于竞争状态，可以通过`XX:-UseBiasedLocking=false`参数关闭偏向锁。

##### `轻量级锁的获取`

当关闭偏向锁功能，或多个线程竞争偏向锁导致偏向锁升级为轻量级锁，会尝试获取轻量级锁，其入口位于`ObjectSynchronizer::slow_enter`

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
 2   markOop mark = obj->mark();
 3   assert(!mark->has_bias_pattern(), "should not see bias pattern here");
 4 
 5   if (mark->is_neutral()) {//是否为无锁状态001
 6     // Anticipate successful CAS -- the ST of the displaced mark must
 7     // be visible <= the ST performed by the CAS.
 8     lock->set_displaced_header(mark);
 9     if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {//CAS成功，释放栈锁
10       TEVENT (slow_enter: release stacklock) ;
11       return ;
12     }
13     // Fall through to inflate() ...
14   } else
15   if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
16     assert(lock != mark->locker(), "must not re-lock the same lock");
17     assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
18     lock->set_displaced_header(NULL);
19     return;
20   }
21 
22 #if 0
23   // The following optimization isn't particularly useful.
24   if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
25     lock->set_displaced_header (NULL) ;
26     return ;
27   }
28 #endif
29 
30   // The object header will never be displaced to this lock,
31   // so it does not matter what the value is, except that it
32   // must be non-zero to avoid looking like a re-entrant lock,
33   // and must not look locked either.
34   lock->set_displaced_header(markOopDesc::unused_mark());
35   ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
36 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、`markOop mark = obj->mark()`方法获取对象的markOop数据mark；
2、`mark->is_neutral()`方法判断mark是否为无锁状态：mark的偏向锁标志位为 **0**，锁标志位为 **01**；
3、如果mark处于无锁状态，则进入步骤（4），否则执行步骤（6）；
4、把mark保存到BasicLock对象的_displaced_header字段；
5、通过CAS尝试将Mark Word更新为指向BasicLock对象的指针，如果更新成功，表示竞争到锁，则执行同步代码，否则执行步骤（6）；
6、如果当前mark处于加锁状态，且mark中的ptr指针指向当前线程的栈帧，则执行同步代码，否则说明有多个线程竞争轻量级锁，轻量级锁需要膨胀升级为重量级锁；

**假设线程A和B同时执行到临界区`if (mark->is_neutral())`**：
1、线程AB都把Mark Word复制到各自的_displaced_header字段，该数据保存在线程的栈帧上，是线程私有的；
2、`Atomic::cmpxchg_ptr`原子操作保证只有一个线程可以把指向栈帧的指针复制到Mark Word，假设此时线程A执行成功，并返回继续执行同步代码块；
3、线程B执行失败，退出临界区，通过`ObjectSynchronizer::inflate`方法开始膨胀锁；

##### 轻量级锁的释放

轻量级锁的释放通过ObjectSynchronizer::slow_exit--->调用`ObjectSynchronizer::fast_exit`完成。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
 2   assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
 3   // if displaced header is null, the previous enter is recursive enter, no-op
 4   markOop dhw = lock->displaced_header();
 5   markOop mark ;
 6   if (dhw == NULL) {
 7      // Recursive stack-lock.
 8      // Diagnostics -- Could be: stack-locked, inflating, inflated.
 9      mark = object->mark() ;
10      assert (!mark->is_neutral(), "invariant") ;
11      if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
12         assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
13      }
14      if (mark->has_monitor()) {
15         ObjectMonitor * m = mark->monitor() ;
16         assert(((oop)(m->object()))->mark() == mark, "invariant") ;
17         assert(m->is_entered(THREAD), "invariant") ;
18      }
19      return ;
20   }
21 
22   mark = object->mark() ;
23 
24   // If the object is stack-locked by the current thread, try to
25   // swing the displaced header from the box back to the mark.
26   if (mark == (markOop) lock) {
27      assert (dhw->is_neutral(), "invariant") ;
28      if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {//成功的释放了锁
29         TEVENT (fast_exit: release stacklock) ;
30         return;
31      }
32   }
33 
34   ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;//锁膨胀升级
35 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、确保处于偏向锁状态时不会执行这段逻辑；
2、取出在获取轻量级锁时保存在BasicLock对象的mark数据dhw；
3、通过CAS尝试把dhw替换到当前的Mark Word，如果CAS成功，说明成功的释放了锁，否则执行步骤（4）；
4、如果CAS失败，说明有其它线程在尝试获取该锁，这时需要将该锁升级为重量级锁，并释放；

### 重量级锁

重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。

##### 锁膨胀过程

锁的膨胀过程通过`ObjectSynchronizer::inflate`函数实现

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  1 ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  2   // Inflate mutates the heap ...
  3   // Relaxing assertion for bug 6320749.
  4   assert (Universe::verify_in_progress() ||
  5           !SafepointSynchronize::is_at_safepoint(), "invariant") ;
  6 
  7   for (;;) {//自旋
  8       const markOop mark = object->mark() ;
  9       assert (!mark->has_bias_pattern(), "invariant") ;
 10 
 11       // The mark can be in one of the following states:
 12       // *  Inflated     - just return
 13       // *  Stack-locked - coerce it to inflated
 14       // *  INFLATING    - busy wait for conversion to complete
 15       // *  Neutral      - aggressively inflate the object.
 16       // *  BIASED       - Illegal.  We should never see this
 17 
 18       // CASE: inflated已膨胀，即重量级锁
 19       if (mark->has_monitor()) {//判断当前是否为重量级锁
 20           ObjectMonitor * inf = mark->monitor() ;//获取指向ObjectMonitor的指针
 21           assert (inf->header()->is_neutral(), "invariant");
 22           assert (inf->object() == object, "invariant") ;
 23           assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
 24           return inf ;
 25       }
 26 
 27       // CASE: inflation in progress - inflating over a stack-lock.膨胀等待（其他线程正在从轻量级锁转为膨胀锁）
 28       // Some other thread is converting from stack-locked to inflated.
 29       // Only that thread can complete inflation -- other threads must wait.
 30       // The INFLATING value is transient.
 31       // Currently, we spin/yield/park and poll the markword, waiting for inflation to finish.
 32       // We could always eliminate polling by parking the thread on some auxiliary list.
 33       if (mark == markOopDesc::INFLATING()) {
 34          TEVENT (Inflate: spin while INFLATING) ;
 35          ReadStableMark(object) ;
 36          continue ;
 37       }
 38 
 39       // CASE: stack-locked栈锁（轻量级锁） 
 40       // Could be stack-locked either by this thread or by some other thread.
 41       //
 42       // Note that we allocate the objectmonitor speculatively, _before_ attempting
 43       // to install INFLATING into the mark word.  We originally installed INFLATING,
 44       // allocated the objectmonitor, and then finally STed the address of the
 45       // objectmonitor into the mark.  This was correct, but artificially lengthened
 46       // the interval in which INFLATED appeared in the mark, thus increasing
 47       // the odds of inflation contention.
 48       //
 49       // We now use per-thread private objectmonitor free lists.
 50       // These list are reprovisioned from the global free list outside the
 51       // critical INFLATING...ST interval.  A thread can transfer
 52       // multiple objectmonitors en-mass from the global free list to its local free list.
 53       // This reduces coherency traffic and lock contention on the global free list.
 54       // Using such local free lists, it doesn't matter if the omAlloc() call appears
 55       // before or after the CAS(INFLATING) operation.
 56       // See the comments in omAlloc().
 57 
 58       if (mark->has_locker()) {
 59           ObjectMonitor * m = omAlloc (Self) ;//获取一个可用的ObjectMonitor 
 60           // Optimistically prepare the objectmonitor - anticipate successful CAS
 61           // We do this before the CAS in order to minimize the length of time
 62           // in which INFLATING appears in the mark.
 63           m->Recycle();
 64           m->_Responsible  = NULL ;
 65           m->OwnerIsThread = 0 ;
 66           m->_recursions   = 0 ;
 67           m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;   // Consider: maintain by type/class
 68 
 69           markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
 70           if (cmp != mark) {//CAS失败//CAS失败，说明冲突了，自旋等待//CAS失败，说明冲突了，自旋等待//CAS失败，说明冲突了，自旋等待
 71              omRelease (Self, m, true) ;//释放监视器锁
 72              continue ;       // Interference -- just retry
 73           }
 74 
 75           // We've successfully installed INFLATING (0) into the mark-word.
 76           // This is the only case where 0 will appear in a mark-work.
 77           // Only the singular thread that successfully swings the mark-word
 78           // to 0 can perform (or more precisely, complete) inflation.
 79           //
 80           // Why do we CAS a 0 into the mark-word instead of just CASing the
 81           // mark-word from the stack-locked value directly to the new inflated state?
 82           // Consider what happens when a thread unlocks a stack-locked object.
 83           // It attempts to use CAS to swing the displaced header value from the
 84           // on-stack basiclock back into the object header.  Recall also that the
 85           // header value (hashcode, etc) can reside in (a) the object header, or
 86           // (b) a displaced header associated with the stack-lock, or (c) a displaced
 87           // header in an objectMonitor.  The inflate() routine must copy the header
 88           // value from the basiclock on the owner's stack to the objectMonitor, all
 89           // the while preserving the hashCode stability invariants.  If the owner
 90           // decides to release the lock while the value is 0, the unlock will fail
 91           // and control will eventually pass from slow_exit() to inflate.  The owner
 92           // will then spin, waiting for the 0 value to disappear.   Put another way,
 93           // the 0 causes the owner to stall if the owner happens to try to
 94           // drop the lock (restoring the header from the basiclock to the object)
 95           // while inflation is in-progress.  This protocol avoids races that might
 96           // would otherwise permit hashCode values to change or "flicker" for an object.
 97           // Critically, while object->mark is 0 mark->displaced_mark_helper() is stable.
 98           // 0 serves as a "BUSY" inflate-in-progress indicator.
 99 
100 
101           // fetch the displaced mark from the owner's stack.
102           // The owner can't die or unwind past the lock while our INFLATING
103           // object is in the mark.  Furthermore the owner can't complete
104           // an unlock on the object, either.
105           markOop dmw = mark->displaced_mark_helper() ;
106           assert (dmw->is_neutral(), "invariant") ;
107           //CAS成功，设置ObjectMonitor的_header、_owner和_object等
108           // Setup monitor fields to proper values -- prepare the monitor
109           m->set_header(dmw) ;
110 
111           // Optimization: if the mark->locker stack address is associated
112           // with this thread we could simply set m->_owner = Self and
113           // m->OwnerIsThread = 1. Note that a thread can inflate an object
114           // that it has stack-locked -- as might happen in wait() -- directly
115           // with CAS.  That is, we can avoid the xchg-NULL .... ST idiom.
116           m->set_owner(mark->locker());
117           m->set_object(object);
118           // TODO-FIXME: assert BasicLock->dhw != 0.
119 
120           // Must preserve store ordering. The monitor state must
121           // be stable at the time of publishing the monitor address.
122           guarantee (object->mark() == markOopDesc::INFLATING(), "invariant") ;
123           object->release_set_mark(markOopDesc::encode(m));
124 
125           // Hopefully the performance counters are allocated on distinct cache lines
126           // to avoid false sharing on MP systems ...
127           if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
128           TEVENT(Inflate: overwrite stacklock) ;
129           if (TraceMonitorInflation) {
130             if (object->is_instance()) {
131               ResourceMark rm;
132               tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
133                 (void *) object, (intptr_t) object->mark(),
134                 object->klass()->external_name());
135             }
136           }
137           return m ;
138       }
139 
140       // CASE: neutral 无锁
141       // TODO-FIXME: for entry we currently inflate and then try to CAS _owner.
142       // If we know we're inflating for entry it's better to inflate by swinging a
143       // pre-locked objectMonitor pointer into the object header.   A successful
144       // CAS inflates the object *and* confers ownership to the inflating thread.
145       // In the current implementation we use a 2-step mechanism where we CAS()
146       // to inflate and then CAS() again to try to swing _owner from NULL to Self.
147       // An inflateTry() method that we could call from fast_enter() and slow_enter()
148       // would be useful.
149 
150       assert (mark->is_neutral(), "invariant");
151       ObjectMonitor * m = omAlloc (Self) ;
152       // prepare m for installation - set monitor to initial state
153       m->Recycle();
154       m->set_header(mark);
155       m->set_owner(NULL);
156       m->set_object(object);
157       m->OwnerIsThread = 1 ;
158       m->_recursions   = 0 ;
159       m->_Responsible  = NULL ;
160       m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;       // consider: keep metastats by type/class
161 
162       if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr(), mark) != mark) {
163           m->set_object (NULL) ;
164           m->set_owner  (NULL) ;
165           m->OwnerIsThread = 0 ;
166           m->Recycle() ;
167           omRelease (Self, m, true) ;
168           m = NULL ;
169           continue ;
170           // interference - the markword changed - just retry.
171           // The state-transitions are one-way, so there's no chance of
172           // live-lock -- "Inflated" is an absorbing state.
173       }
174 
175       // Hopefully the performance counters are allocated on distinct
176       // cache lines to avoid false sharing on MP systems ...
177       if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
178       TEVENT(Inflate: overwrite neutral) ;
179       if (TraceMonitorInflation) {
180         if (object->is_instance()) {
181           ResourceMark rm;
182           tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
183             (void *) object, (intptr_t) object->mark(),
184             object->klass()->external_name());
185         }
186       }
187       return m ;
188   }
189 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

膨胀过程的实现比较复杂，大概实现过程如下：
1、整个膨胀过程在自旋下完成；
2、`mark->has_monitor()`方法判断当前是否为重量级锁（上图18-25行），即Mark Word的锁标识位为 **10**，如果当前状态为重量级锁，执行步骤（3），否则执行步骤（4）；
3、`mark->monitor()`方法获取指向ObjectMonitor的指针，并返回，说明膨胀过程已经完成；
4、如果当前锁处于膨胀中（上图33-37行），说明该锁正在被其它线程执行膨胀操作，则当前线程就进行自旋等待锁膨胀完成，这里需要注意一点，虽然是自旋操作，但不会一直占用cpu资源，每隔一段时间会通过os::NakedYield方法放弃cpu资源，或通过park方法挂起；如果其他线程完成锁的膨胀操作，则退出自旋并返回；
5、如果当前是轻量级锁状态（上图58-138行），即锁标识位为 **00**，膨胀过程如下：

1. 通过omAlloc方法，获取一个可用的ObjectMonitor monitor，并重置monitor数据；
2. 通过CAS尝试将Mark Word设置为markOopDesc:INFLATING，标识当前锁正在膨胀中，如果CAS失败，说明同一时刻其它线程已经将Mark Word设置为markOopDesc:INFLATING，当前线程进行自旋等待膨胀完成；
3. 如果CAS成功，设置monitor的各个字段：_header、_owner和_object等，并返回；

6、如果是无锁（中立，上图150-186行），重置监视器值；

##### monitor竞争

当锁膨胀完成并返回对应的monitor时，并不表示该线程竞争到了锁，真正的锁竞争发生在`ObjectMonitor::enter`方法中。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
  1 void ATTR ObjectMonitor::enter(TRAPS) {
  2   // The following code is ordered to check the most common cases first
  3   // and to reduce RTS->RTO cache line upgrades on SPARC and IA32 processors.
  4   Thread * const Self = THREAD ;
  5   void * cur ;
  6 
  7   cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  8   if (cur == NULL) {//CAS成功
  9      // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
 10      assert (_recursions == 0   , "invariant") ;
 11      assert (_owner      == Self, "invariant") ;
 12      // CONSIDER: set or assert OwnerIsThread == 1
 13      return ;
 14   }
 15 
 16   if (cur == Self) {//重入锁
 17      // TODO-FIXME: check for integer overflow!  BUGID 6557169.
 18      _recursions ++ ;
 19      return ;
 20   }
 21 
 22   if (Self->is_lock_owned ((address)cur)) {
 23     assert (_recursions == 0, "internal state error");
 24     _recursions = 1 ;
 25     // Commute owner from a thread-specific on-stack BasicLockObject address to
 26     // a full-fledged "Thread *".
 27     _owner = Self ;
 28     OwnerIsThread = 1 ;
 29     return ;
 30   }
 31 
 32   // We've encountered genuine contention.
 33   assert (Self->_Stalled == 0, "invariant") ;
 34   Self->_Stalled = intptr_t(this) ;
 35 
 36   // Try one round of spinning *before* enqueueing Self
 37   // and before going through the awkward and expensive state
 38   // transitions.  The following spin is strictly optional ...
 39   // Note that if we acquire the monitor from an initial spin
 40   // we forgo posting JVMTI events and firing DTRACE probes.
 41   if (Knob_SpinEarly && TrySpin (Self) > 0) {
 42      assert (_owner == Self      , "invariant") ;
 43      assert (_recursions == 0    , "invariant") ;
 44      assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
 45      Self->_Stalled = 0 ;
 46      return ;
 47   }
 48 
 49   assert (_owner != Self          , "invariant") ;
 50   assert (_succ  != Self          , "invariant") ;
 51   assert (Self->is_Java_thread()  , "invariant") ;
 52   JavaThread * jt = (JavaThread *) Self ;
 53   assert (!SafepointSynchronize::is_at_safepoint(), "invariant") ;
 54   assert (jt->thread_state() != _thread_blocked   , "invariant") ;
 55   assert (this->object() != NULL  , "invariant") ;
 56   assert (_count >= 0, "invariant") ;
 57 
 58   // Prevent deflation at STW-time.  See deflate_idle_monitors() and is_busy().
 59   // Ensure the object-monitor relationship remains stable while there's contention.
 60   Atomic::inc_ptr(&_count);
 61 
 62   EventJavaMonitorEnter event;
 63 
 64   { // Change java thread status to indicate blocked on monitor enter.
 65     JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);
 66 
 67     DTRACE_MONITOR_PROBE(contended__enter, this, object(), jt);
 68     if (JvmtiExport::should_post_monitor_contended_enter()) {
 69       JvmtiExport::post_monitor_contended_enter(jt, this);
 70     }
 71 
 72     OSThreadContendState osts(Self->osthread());
 73     ThreadBlockInVM tbivm(jt);
 74 
 75     Self->set_current_pending_monitor(this);
 76 
 77     // TODO-FIXME: change the following for(;;) loop to straight-line code.
 78     for (;;) {
 79       jt->set_suspend_equivalent();
 80       // cleared by handle_special_suspend_equivalent_condition()
 81       // or java_suspend_self()
 82 
 83       EnterI (THREAD) ;
 84 
...省略...139 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、通过CAS尝试把monitor的_owner字段设置为当前线程；
2、如果设置之前的_owner指向当前线程，说明当前线程再次进入monitor，即重入锁，执行_recursions ++ ，记录重入的次数；
3、如果之前的_owner指向的地址在当前线程中，这种描述有点拗口，换一种说法：之前_owner指向的BasicLock在当前线程栈上，说明当前线程是第一次进入该monitor，设置_recursions为1，_owner为当前线程，该线程成功获得锁并返回；
4、如果获取锁失败，则等待锁的释放；

##### monitor等待

monitor竞争失败的线程，通过自旋执行`ObjectMonitor::EnterI`方法等待锁的释放，EnterI方法的部分逻辑实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 ObjectWaiter node(Self) ;
 2     Self->_ParkEvent->reset() ;
 3     node._prev   = (ObjectWaiter *) 0xBAD ;
 4     node.TState  = ObjectWaiter::TS_CXQ ;
 5 
 6     // Push "Self" onto the front of the _cxq.
 7     // Once on cxq/EntryList, Self stays on-queue until it acquires the lock.
 8     // Note that spinning tends to reduce the rate at which threads
 9     // enqueue and dequeue on EntryList|cxq.
10     ObjectWaiter * nxt ;
11     for (;;) {
12         node._next = nxt = _cxq ;
13         if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;
14 
15         // Interference - the CAS failed because _cxq changed.  Just retry.
16         // As an optional optimization we retry the lock.
17         if (TryLock (Self) > 0) {
18             assert (_succ != Self         , "invariant") ;
19             assert (_owner == Self        , "invariant") ;
20             assert (_Responsible != Self  , "invariant") ;
21             return ;
22         }
23     }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ；
2、在for循环中，通过CAS把node节点push到_cxq列表中，同一时刻可能有多个线程把自己的node节点push到_cxq列表中；
3、node节点push到_cxq列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当前线程挂起，等待被唤醒，实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 for (;;) {
 2 
 3         if (TryLock (Self) > 0) break ;
 4         assert (_owner != Self, "invariant") ;
 5 
 6         if ((SyncFlags & 2) && _Responsible == NULL) {
 7            Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
 8         }
 9 
10         // park self
11         if (_Responsible == Self || (SyncFlags & 1)) {
12             TEVENT (Inflated enter - park TIMED) ;
13             Self->_ParkEvent->park ((jlong) RecheckInterval) ;
14             // Increase the RecheckInterval, but clamp the value.
15             RecheckInterval *= 8 ;
16             if (RecheckInterval > 1000) RecheckInterval = 1000 ;
17         } else {
18             TEVENT (Inflated enter - park UNTIMED) ;
19             Self->_ParkEvent->park() ;//当前线程挂起
20         }
21 
22         if (TryLock(Self) > 0) break ;
23 
24         // The lock is still contested.
25         // Keep a tally of the # of futile wakeups.
26         // Note that the counter is not protected by a lock or updated by atomics.
27         // That is by design - we trade "lossy" counters which are exposed to
28         // races during updates for a lower probe effect.
29         TEVENT (Inflated enter - Futile wakeup) ;
30         if (ObjectMonitor::_sync_FutileWakeups != NULL) {
31            ObjectMonitor::_sync_FutileWakeups->inc() ;
32         }
33         ++ nWakeups ;
34 
35         // Assuming this is not a spurious wakeup we'll normally find _succ == Self.
36         // We can defer clearing _succ until after the spin completes
37         // TrySpin() must tolerate being called with _succ == Self.
38         // Try yet another round of adaptive spinning.
39         if ((Knob_SpinAfterFutile & 1) && TrySpin (Self) > 0) break ;
40 
41         // We can find that we were unpark()ed and redesignated _succ while
42         // we were spinning.  That's harmless.  If we iterate and call park(),
43         // park() will consume the event and return immediately and we'll
44         // just spin again.  This pattern can repeat, leaving _succ to simply
45         // spin on a CPU.  Enable Knob_ResetEvent to clear pending unparks().
46         // Alternately, we can sample fired() here, and if set, forgo spinning
47         // in the next iteration.
48 
49         if ((Knob_ResetEvent & 1) && Self->_ParkEvent->fired()) {
50            Self->_ParkEvent->reset() ;
51            OrderAccess::fence() ;
52         }
53         if (_succ == Self) _succ = NULL ;
54 
55         // Invariant: after clearing _succ a thread *must* retry _owner before parking.
56         OrderAccess::fence() ;
57     }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

4、当该线程被唤醒时，会从挂起的点继续执行，通过`ObjectMonitor::TryLock`尝试获取锁，TryLock方法实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 int ObjectMonitor::TryLock (Thread * Self) {
 2    for (;;) {
 3       void * own = _owner ;
 4       if (own != NULL) return 0 ;
 5       if (Atomic::cmpxchg_ptr (Self, &_owner, NULL) == NULL) {//CAS成功，获取锁
 6          // Either guarantee _recursions == 0 or set _recursions = 0.
 7          assert (_recursions == 0, "invariant") ;
 8          assert (_owner == Self, "invariant") ;
 9          // CONSIDER: set or assert that OwnerIsThread == 1
10          return 1 ;
11       }
12       // The lock had been free momentarily, but we lost the race to the lock.
13       // Interference -- the CAS failed.
14       // We can either return -1 or retry.
15       // Retry doesn't make as much sense because the lock was just acquired.
16       if (true) return -1 ;
17    }
18 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其本质就是通过CAS设置monitor的_owner字段为当前线程，如果CAS成功，则表示该线程获取了锁，跳出自旋操作，执行同步代码，否则继续被挂起；

##### monitor释放

当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于`ObjectMonitor::exit`方法中。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) {
 2    Thread * Self = THREAD ;
 3    if (THREAD != _owner) {
 4      if (THREAD->is_lock_owned((address) _owner)) {
 5        // Transmute _owner from a BasicLock pointer to a Thread address.
 6        // We don't need to hold _mutex for this transition.
 7        // Non-null to Non-null is safe as long as all readers can
 8        // tolerate either flavor.
 9        assert (_recursions == 0, "invariant") ;
10        _owner = THREAD ;
11        _recursions = 0 ;
12        OwnerIsThread = 1 ;
13      } else {
14        // NOTE: we need to handle unbalanced monitor enter/exit
15        // in native code by throwing an exception.
16        // TODO: Throw an IllegalMonitorStateException ?
17        TEVENT (Exit - Throw IMSX) ;
18        assert(false, "Non-balanced monitor enter/exit!");
19        if (false) {
20           THROW(vmSymbols::java_lang_IllegalMonitorStateException());
21        }
22        return;
23      }
24    }
25 
26    if (_recursions != 0) {
27      _recursions--;        // this is simple recursive enter
28      TEVENT (Inflated exit - recursive) ;
29      return ;
30    }...省略...
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

1、如果是重量级锁的释放，monitor中的_owner指向当前线程，即THREAD == _owner；
2、根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过`ObjectMonitor::ExitEpilog`方法唤醒该节点封装的线程，唤醒操作最终由unpark完成，实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 void ObjectMonitor::ExitEpilog (Thread * Self, ObjectWaiter * Wakee) {
 2    assert (_owner == Self, "invariant") ;
 3 
 4    // Exit protocol:
 5    // 1. ST _succ = wakee
 6    // 2. membar #loadstore|#storestore;
 7    // 2. ST _owner = NULL
 8    // 3. unpark(wakee)
 9 
10    _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
11    ParkEvent * Trigger = Wakee->_event ;
12 
13    // Hygiene -- once we've set _owner = NULL we can't safely dereference Wakee again.
14    // The thread associated with Wakee may have grabbed the lock and "Wakee" may be
15    // out-of-scope (non-extant).
16    Wakee  = NULL ;
17 
18    // Drop the lock
19    OrderAccess::release_store_ptr (&_owner, NULL) ;
20    OrderAccess::fence() ;                               // ST _owner vs LD in unpark()
21 
22    if (SafepointSynchronize::do_call_back()) {
23       TEVENT (unpark before SAFEPOINT) ;
24    }
25 
26    DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
27    Trigger->unpark() ;
28 
29    // Maintain stats and report events to JVMTI
30    if (ObjectMonitor::_sync_Parks != NULL) {
31       ObjectMonitor::_sync_Parks->inc() ;
32    }
33 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

3、被唤醒的线程，继续执行monitor的竞争；

[回到顶部](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_labelTop)

## 四.总结

本文重点介绍了Synchronized原理以及JVM对Synchronized的优化。简单来说解决三种场景：

1）只有一个线程进入临界区，偏向锁

2）多个线程交替进入临界区，轻量级锁

3）多线程同时进入临界区，重量级锁

如果还是模糊，看一张图，截自网络

![img](https://img2020.cnblogs.com/blog/584866/202008/584866-20200803135640778-810911645.png)

 

========================================================

 

参考：

《深入理解 Java 虚拟机》

[JVM源码分析之synchronized实现](http://www.jianshu.com/p/c5058b6fe8e5)

# 参考资料

https://www.cnblogs.com/insaneXs/p/13378994.html

Hotspot GC过程https://www.cnblogs.com/iceAeterNa/p/4877814.html

1.https://wiki.openjdk.java.net/display/zgc

https://www.cnblogs.com/dennyzhangdd/p/6734638.html



2.https://openjdk.java.net/jeps/304





3.https://openjdk.java.net/jeps/376





4.https://malloc.se/blog/zgc-jdk16





5.https://mp.weixin.qq.com/s/ag5u2EPObx7bZr7hkcrOTg





6.https://mp.weixin.qq.com/s/FIr6r2dcrm1pqZj5Bubbmw





7.https://www.jianshu.com/p/664e4da05b2c





8.https://www.cnblogs.com/jimoer/p/13170249.html





9.https://www.jianshu.com/p/12544c0ad5c1

https://www.cnblogs.com/WCFGROUP/p/6213943.html
