---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码源码线程池ExecutorService
ExecutorService 是 Java 中的一个接口，它扩展了 Executor 接口，并提供了更多的方法来处理多线程任务。它是 Java 中用于执行多线程任务的框架之一，可以创建一个线程池，将多个任务提交到线程池中执行。

## ExecutorService接口
ExecutorService 接口提供了许多方法，如 shutdown()、shutdownNow()、submit()、execute()、invokeAll() 等，可以更方便地提交任务、执行任务、关闭线程池等操作。同时，ExecutorService 还提供了线程池的管理和监控功能，可以更好地控制和管理线程池中的线程。

在实际应用中，ExecutorService 通常与 Callable 和 Future 接口一起使用，可以实现更加灵活和高效的多线程编程。

ExecutorService，其中定义了线程池的具体行为：

- execute(Runnable runnable)：执行Runnable类型的任务
- submit(task)：用来提交Callable或者Runnable任务，并返回代表此任务的Future对象
- shutdown()：在完成已经提交的任务后封闭办事，不在接管新的任务
- shutdownNow()：停止所有正在履行的任务并封闭办事
- isTerminated()：是一个钩子函数，测试是否所有任务都履行完毕了
- isShutdown()：是一个钩子函数，测试是否该ExecutorService是否被关闭

## ExecutorService作用
ExecutorService 是 Java 中用于执行多线程任务的框架，它允许我们创建一个线程池，将多个任务提交到线程池中执行。它的主要作用是优化线程的创建和销毁过程，提高程序的效率和性能。通过 ExecutorService，我们可以更好地控制线程的数量、避免线程阻塞和死锁，从而更好地管理和调度线程，提高应用程序的并发处理能力。

在实际开发中，ExecutorService 可以用于处理大量的并发请求、分批处理大数据等多线程场景。

## ExecutorService原理
ExecutorService 的实现原理主要是基于线程池的概念。当我们创建一个 ExecutorService 对象时，实际上就是创建了一个线程池。线程池中包含了若干个线程，这些线程可以执行我们提交的任务。当线程池中的线程空闲时，它们会等待任务的到来，一旦有任务提交，就会从线程池中选择一个空闲的线程执行该任务。

如果线程池中的线程都在执行任务，那么新的任务就会被暂时放在任务队列中，等待线程空闲时再来执行。

在 ExecutorService 的实现中，任务的提交和执行是异步的，也就是说，我们提交任务时不会阻塞当前线程，而是将任务交给线程池中的线程去执行。当任务执行完成后，线程会将执行结果返回给我们。同时，我们可以通过调用 ExecutorService 的方法来管理和控制线程池，如增加或减少线程数量、关闭线程池等。总之，ExecutorService 的实现原理是基于线程池的概念，通过管理和调度线程，提高程序的效率和性能，同时避免线程阻塞和死锁等问题，从而更好地管理和调度线程，提高应用程序的并发处理能力。

## 方法
线程池中有五种状态，分别是 RUNNING、STOP、SHUTDOWN、TIDYING 和 TERMINATED。它们的含义和区别如下：

RUNNING：表示线程池处于运行状态，接受新的任务并且处理任务队列中的任务，直到线程池被显式地关闭。

SHUTDOWN：表示线程池处于关闭状态，不再接受新的任务，但是会尝试执行任务队列中的任务，直到任务队列为空。在任务队列为空后，线程池会进入 TIDYING 状态。

STOP：表示线程池处于停止状态，不再接受新的任务，也不会继续执行任务队列中的任务。此时，线程池会尝试中断正在执行的任务，并立即返回任务队列中的所有任务。在任务队列为空后，线程池会进入 TIDYING 状态。

TIDYING：表示线程池正在进行线程回收的操作，此时线程池中的所有任务都已经执行完成，而线程池中的线程也已经被销毁。在线程回收完成后，线程池会进入 TERMINATED 状态。

TERMINATED：表示线程池已经完全终止，不再接受任何任务，也不会执行任何任务。此时，线程池中的所有线程都已经被销毁，线程池对象也可以被垃圾回收。

总之，这五种状态代表了 ThreadPoolExecutor 在不同时间点的不同状态，分别表示线程池的运行状态、关闭状态、停止状态、回收状态和终止状态。它们的区别在于线程池在不同状态下的行为和状态转换。

## shutdown()方法
ExecutorService 中的 shutdown() 方法是用于关闭线程池的，它会停止接受新的任务，并尝试将已经提交但尚未执行的任务执行完成。在调用 shutdown() 方法后，线程池不再接受新的任务，而是等待之前提交的任务执行完成后关闭线程池。具体来说，shutdown() 方法会将线程池的状态设置为 SHUTDOWN，然后中断所有没有开始执行的任务，并尝试将已提交但未开始执行的任务从任务队列中移除，最后等待所有正在执行的任务执行完成后关闭线程池。如果线程池中的任务是通过 Future 返回结果的，那么在调用 shutdown() 方法后，我们可以通过 Future 的 isDone() 方法来判断任务是否执行完成。需要注意的是，shutdown() 方法并不会强制终止任务的执行，它只会尝试将任务执行完成。如果任务本身存在死循环或者其他无法正常结束的情况，那么线程池可能无法正常关闭。如果想要强制终止所有任务的执行，可以使用 shutdownNow() 方法。

总之，shutdown() 方法是 ExecutorService 中用于关闭线程池的方法，它会尝试将已提交但未执行的任务执行完成后关闭线程池，是一个比较温和的关闭方式。

## shutdownNow()方法
ExecutorService 中的 shutdownNow() 方法是用于强制关闭线程池的，它会尝试立即停止所有正在执行的任务，并返回那些未执行的任务列表。在调用 shutdownNow() 方法后，线程池会尝试中断所有正在执行的任务，并将任务队列中未执行的任务返回给调用者。如果线程池中的任务是通过 Future 返回结果的，那么可以通过 Future 的 isDone() 方法来判断任务是否执行完成。需要注意的是，shutdownNow() 方法并不是强制终止任务的执行，它只是尝试中断正在执行的任务。如果任务本身存在死循环或者其他无法正常结束的情况，那么线程池可能无法正常关闭。因此，在使用 shutdownNow() 方法时，需要谨慎考虑，避免对任务造成不可逆的影响。

总之，shutdownNow() 方法是 ExecutorService 中用于强制关闭线程池的方法，它会尝试立即停止所有正在执行的任务，并返回未执行的任务列表，是一种比较强硬的关闭方式。

## isShutdown()方法
ExecutorService 中的 isShutdown() 方法是用于判断线程池是否已经关闭的，它返回一个 boolean 类型的值，如果线程池已经关闭，则返回 true，否则返回 false。当线程池被关闭后，它就不能再接受新的任务，但是它可能还有一些已经提交但还未执行完成的任务。调用 isShutdown() 方法可以判断当前线程池是否已经关闭，从而可以根据需要进行进一步的处理，如等待未执行的任务执行完成后再关闭线程池等。需要注意的是，isShutdown() 方法只能判断线程池是否已经关闭，而不能判断线程池是否已经完全终止。如果要判断线程池是否已经完全终止，可以使用 isTerminated() 方法。isTerminated() 方法返回一个 boolean 类型的值，如果线程池已经完全终止，则返回 true，否则返回 false。

总之，isShutdown() 方法是 ExecutorService 中用于判断线程池是否已经关闭的方法，可以根据返回值来判断线程池是否已经关闭，从而进行进一步的处理。

## isTerminated()方法
ExecutorService 接口中的 isTerminated() 方法用于判断线程池是否已经终止。如果线程池已经被显式地关闭，并且所有任务都已经执行完成并且所有线程都已经被销毁，那么 isTerminated() 方法会返回 true；否则，返回 false。isTerminated() 方法的实现原理是通过检查线程池的状态来判断线程池是否已经终止，具体来说，它会判断线程池是否处于 TERMINATED 状态，如果是，就返回 true；否则，返回 false。在使用 ExecutorService 管理线程池时，可以使用 isTerminated() 方法来判断线程池的状态，从而决定是否需要等待线程池执行完所有任务，并且所有线程都被销毁之后再进行下一步操作。常见的做法是在关闭线程池后，使用 isTerminated() 方法轮询线程池的状态，直到返回 true 为止。

## awaitTermination(long timeout, TimeUnit unit) 方法
ExecutorService 接口中的 awaitTermination(long timeout, TimeUnit unit) 方法用于等待线程池中所有任务执行完成，并且所有线程都被销毁。它会阻塞当前线程，直到线程池中的所有任务都已经执行完成，并且所有线程都已经被销毁，或者等待超时（由 timeout 和 unit 指定）。如果在指定的等待时间内所有任务都已经执行完成并且所有线程都已经被销毁，方法会返回 true；否则，返回 false。也就是说，如果线程池在等待超时前已经终止，方法会返回 true；否则，返回 false。

在使用 ExecutorService 管理线程池时，可以使用 awaitTermination(long timeout, TimeUnit unit) 方法来等待线程池中的所有任务执行完成，并且所有线程都被销毁。常见的做法是在关闭线程池后，先调用 shutdown() 方法关闭线程池，然后调用 awaitTermination(long timeout, TimeUnit unit) 方法等待线程池中的所有任务执行完成，并且所有线程都被销毁。如果需要在等待超时后继续执行下一步操作，可以根据返回值来决定是否调用 shutdownNow() 方法强制中断线程池中的任务并销毁所有线程。

## <T> Future<T> submit(Callable<T> task)方法
在 ExecutorService接口中，<T> Future<T> submit(Callable<T> task)方法用于向线程池提交一个需要返回结果的任务，该方法接受一个 Callable<T> 类型的参数，该参数表示要执行的任务，Callable<T> 接口中的 T表示任务执行后的返回值类型。 该方法将任务提交到线程池中，并返回一个 Future<T> 对象，用于获取任务执行的结果。在任务执行完毕后，可以通过调用 Future<T>对象的 get()方法来获取任务执行的结果，如果任务还没有执行完，则 get()方法会阻塞等待任务执行完毕并获取结果。 如果任务执行过程中发生异常，get()方法会抛出相应的异常，可以通过捕获异常来进行相应的处理。

## List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)方法
在 ExecutorService 接口中，List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) 方法用于向线程池提交一组任务，并等待所有任务执行完毕并返回结果。该方法接受一个 Collection<? extends Callable<T>> 类型的参数，该参数表示要执行的任务集合，Callable<T> 接口中的 T表示任务执行后的返回值类型。 该方法将所有任务提交到线程池中，并返回一个 List<Future<T>>对象，用于获取所有任务执行的结果。在所有任务执行完毕后，可以通过循环遍历 List<Future<T>> 对象来获取每个任务执行的结果，如果任务还没有执行完，则 get()方法会阻塞等待任务执行完毕并获取结果。 如果任务执行过程中发生异常，get() 方法会抛出相应的异常，可以通过捕获异常来进行相应的处理。

## <T> T invokeAny(Collection<? extends Callable<T>> tasks) 方法
ExecutorService.invokeAny()方法会阻塞直到至少有一个任务完成，并返回其执行结果。如果其中一个任务抛出异常，该异常将被传递给调用者。如果所有任务都抛出了异常，则抛出 ExecutionException异常。

此方法适用于需要执行多个任务，只需要获取其中任意一个任务执行成功的结果的场景。此方法例如，在多个引擎上搜索某个关键字，只要有一个搜索引擎返回了结果，就可以将其作为最终结果返回给用户。







