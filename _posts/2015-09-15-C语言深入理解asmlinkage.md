---
layout: post
categories: [C]
description: none
keywords: C
---
# C语言深入理解asmlinkage

## asmlinkage的简介与定义
asmlinkage是一个Linux内核宏函数，它指示编译器不使用常规的函数调用机制，而是使用汇编实现的调用机制。它的作用是将函数的参数从普通的C语言调用规约转换为系统调用规约。

在定义有asmlinkage的函数中，所有参数都必须在栈中传递，而不是在寄存器中传递。在32位模式下，前6个参数可以通过寄存器进行传递，后面的参数则必须通过栈进行传递。而在64位模式下，前8个参数通过寄存器传递，后面的参数也必须通过栈进行传递。
```
asmlinkage long sys_mkdir(const char __user *pathname, int mode);
```

## asmlinkage的原理
在C语言中，参数是在寄存器和栈之间传递的，这些寄存器和栈的使用往往由编译器决定。而在系统调用中，参数必须以固定的方式传递，这就需要使用汇编语言来精确控制参数。

宏定义asmlinkage没有让编译器为我们生成常规函数调用代码，而是使用了汇编进行了手工编写。当编译器遇到使用asmlinkage宏定义的函数时，它会插入一些特殊的汇编代码。这些代码的作用是将函数的参数从C语言规约转换为系统调用规约。

## asmlinkage的使用场景
asmlinkage宏定义的函数在Linux内核中非常常见，它们以系统调用的方式暴露给用户空间的程序。例如sys_read和sys_write这些系统调用函数，在内核源代码中都有使用asmlinkage定义。

另一个常见的使用场景是中断处理函数。中断处理函数需要快速地处理中断，而不能涉及到线程的切换和系统调用等等。因此，只能在寄存器和栈之间传递参数，这恰恰是asmlinkage的优势所在。

## asmlinkage使用示例
下面是一个简单的例子，它演示了如何写一个系统调用函数，并在其中使用asmlinkage宏定义：
```
#include <linux/kernel.h>
#include <linux/linkage.h>

asmlinkage long sys_hello(void)
{
    printk(KERN_ALERT "Hello, World!\n");
    return 0;
}
```
在上面的代码中，可以看到asmlinkage宏定义出现在函数的前面。这让编译器知道它需要使用汇编调用规约来调用这个函数。同时，函数sys_hello也是一个特殊的函数，它的返回值是long，这是系统调用的标准返回类型。

## asmlinkage的注意事项
在使用asmlinkage时，需要注意以下几点：

1. 所有的参数都必须通过栈进行传递。

2. 在32位模式下，前6个参数可以通过寄存器进行传递，后面的参数则必须通过栈进行传递。在64位模式下，前8个参数通过寄存器传递，后面的参数也必须通过栈进行传递。

3. 在定义函数时，必须使用asmlinkage宏定义。

4. 在函数体中，不能访问用户空间的内存。

在大型C语言项目工程或者linux内核中我们都会经常见到两个FASTCALL和armlinkage

两个标识符(修饰符)，那么它们各有什么不同呢？今天就给大家共同分享一下自己的心得.

大家都知道在标准C系中函数的形参在实际传入参数的时候会涉及到参数存放的问题，那么这些参数存放在哪里呢？ 有一定理论基础的朋友一定会肯定地回答：这些函数参数和函数内部局部变量一起被分配到了函数的局部堆栈中，真的是这样吗？其实还有例外的情况：

首先作为linux操作系统,它不一定就只运行在X86平台下面,还有其他平台例如ARM,PPC，达芬奇等等，所以在不同的处理器结构上不能保证都是通过 局部栈传递参数的，可能此时就有朋友就会问：不放在栈中能放在哪里呢？熟悉ARM的朋友一定知道ARM对函数调用过程中的传参定义了一套规则，叫 ATPCS（内地叫AAPCS），规则中明确指出ARM中R0-R4都是作为通用寄存器使用，在函数调用时处理器从R0-R4中获取参数，在函数返回时再 将需要返回的参数一次存到R0-R4中，也就是说可以将函数参数直接存放在寄存器中，所以为了严格区别函数参数的存放位置，引入了两个标记，即 asmlinkage和FASTCALL，前者表示将函数参数存放在局部栈中,后者则是通知编译器将函数参数用寄存器保存起来

我们在搜索一些额外的线索，ARM中R0-R4用于存放传入参数，隐约告诉我们，作为高水平嵌入式系统开发者，或者高水平C语言程序员，函数的参数不应 该大于5个，那么有人就会反过来问：超过5个的那些参数又何去何从？我的回答是：传入参数如果超过5个，多余的参数还是被存放到局部栈中，此时有人可能又 会问：将函数参数传入局部栈有什么不好？我的回答是：表面上没什么不好，但是如果你是一名具有linux内核修养的程序员,你就会隐约记得linux中， 不管是系统调用，还是系统陷阱都会引起用户空间陷入内核空间，我们知道，系统空间的权限级是0，用户空间的权限级为3，系统调用从权限级为3的用户空间陷 到权限级为0的内核空间，必然引起堆栈切换，linux系统将从全局任务状态栈TSS中找到一个合适的内核栈信息保存覆盖当前SP,SS两个寄存器的内 容，以完成堆栈切换，此时处于内核空间所看到的栈已不是用户空间那个栈，所以在调用的时候压入用户栈的数据就在陷入内核的那个瞬间，被滞留在用户空间栈， 内核根本不知道它的存在了，所以作为安全考虑或者作为高水平程序员的切身修养出发，都不应该向系统调用级函数传入过多的参数。

它是GCC对C程序的一种扩展, #define asmlinkage __attribute__((regparm(0)))

表示用0个寄存器传递函数参数，这样，所有的函数参数强迫从栈中提取。

这个asmlinkage大都用在系统调用中，系统调用需要在entry.s文件中用汇编语言调用，所以必须要保证它符合C语言的参数传递规则，才能用汇编语言正确调用它。
这也是为何使用asmlinkage的原因吧！这是我的理解。

仔细看一下有asmlinkage的地方通常是系统调用的函数，因为在系统调用中，寄存器从用户空间传过来后SAVE_ALL压入堆栈，接着调用相应的系统调用函数，这样系统调用函数一定要保证是通过堆栈传递参数的

## 什么是 "asmlinkage"？

相信大家在看linux的source code的时候,都会注意到asmlinkage这个宏,它是用来做什么的呢?

The asmlinkage tag is one other thing that we should observe about this simple function. This is a #define for some gcc magic that tells the compiler that the function should

not expect to find any of its arguments in registers (a common optimization), but only on the CPU's stack. Recall our earlier assertion that system_call consumes its first

argument, the system call number, and allows up to four more arguments that are passed along to the real system call. system_call achieves this feat simply by leaving its

other arguments (which were passed to it in registers) on the stack. All system calls are marked with the asmlinkage tag, so they all look to the stack for arguments. Of

course, in sys_ni_syscall's case, this doesn't make any difference, because sys_ni_syscall doesn't take any arguments, but it's an issue for most other system calls.

看一下/usr/include/asm/linkage.h里面的定义：
#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
__attribute__是关键字，是gcc的C语言扩展，regparm(0)表示不从寄存器传递参数

如果是__attribute__((regparm(3)))，那么调用函数的时候参数不是通过栈传递，而是直接放到寄存器里，被调用函数直接从寄存器取参数

还有一种是：

#define fastcall __attribute__((regparm(3)))
#define asmlinkage __attribute__((regparm(0)))
函数定义前加宏asmlinkage ,表示这些函数通过堆栈而不是通过寄存器传递参数。
gcc编译器在汇编过程中调用c语言函数时传递参数有两种方法：一种是通过堆栈，另一种是通过寄存器。缺省时采用寄存器，假如你要在你的汇编过程中调用c语言函数，并且想通过堆栈传递参数，你定义的c函数时要在函数前加上宏asmlinkage



asmlinkage long sys_nice(int increment)
"asmlinkage" 是在 i386 system call 实作中相当重要的一个 gcc 标签（tag）。
当 system call handler 要调用相对应的 system call routine 时，便将一般用途缓存器的值 push 到 stack 里，因此 system call routine 就要由 stack 来读取 system call handler 传递的

参数。这就是 asmlinkage 标签的用意。
system call handler 是 assembly code，system call routine（例如：sys_nice）是 C code，当 assembly code 调用 C function，并且是以 stack 方式传参数（parameter）时，在 C

function 的 prototype 前面就要加上 "asmlinkage"。
加上 "asmlinkage" 后，C function 就会由 stack 取参数，而不是从 register 取参数（可能发生在程序代码最佳化后）。
更进一步的说明...
80x86 的 assembly 有 2 种传递参数的方法：
1. register method
2. stack method
   Register method 大多使用一般用途（general-purpose）缓存器来传递参数，这种方法的好处是简单且快速。另外一种传递参数的做法是使用 stack（堆栈），assembly code 的模式如下：
   push number1
   push number2
   push number3
   call sum
   在 'sum' procedure 里取值的方法，最简单的做法是：
   pop ax
   pop ax
   pop bx
   pop cx
   Stack Top 是放 IP，我们传给 sum procedure 的参数由 stack 的后一个 entry 开始读取。
   其它有关 asmlinkage
1. asmlinkage 是一个定义
2. "asmlinkage" 被定义在 /usr/include/linux/linkage.h
3. 如果您看了 linkage.h，会发现 "__attribute__" 这个语法，这是 gcc 用来定义 function attribute 的语法