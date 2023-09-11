---
layout: post
categories: [JUC]
description: none
keywords: JUC
---
# 并发源码CompletionStage


## CompletionStage接口方法介绍

依赖关系
```
// 当这个CompletionStage执行完毕后，返回一个新的CompletionStage，在这个新的CompletionStage中回调fn
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);

// 和上面不同的是，拿到上个任务的执行结果后，回调的任务会放到默认线程池ForkJoinPool执行
public <U> CompletionStage<U> thenApplyAsync
        (Function<? super T,? extends U> fn);
        
// 和上面不同的是，这个能指定线程池
public <U> CompletionStage<U> thenApplyAsync
        (Function<? super T,? extends U> fn,
         Executor executor);
         
// 当这个CompletionStage执行完毕后，返回一个新的CompletionStage，并在新的stage中调用action消费结果
public CompletionStage<Void> thenAccept(Consumer<? super T> action);

// 当这个CompletionStage执行完毕后，返回一个新的CompletionStage，并在新的stage中调用action，没有入参，也没有返回值。
public CompletionStage<Void> thenRun(Runnable action);

```

and集合关系
```
// 当目前的CompletionStage和thenCombine传入的thenCombine正常完成后，返回一个新的CompletionStage，并在新的stage中调用fn函数
public <U,V> CompletionStage<V> thenCombine
        (CompletionStage<? extends U> other,
         BiFunction<? super T,? super U,? extends V> fn);

// 当目前的CompletionStage和thenCombine传入的other正常完成后，返回一个新的CompletionStage，并在新的stage中调用action消费结果，注意这里的没有返回值
public <U> CompletionStage<Void> thenAcceptBoth
        (CompletionStage<? extends U> other,
         BiConsumer<? super T, ? super U> action);       

// 当目前的CompletionStage和other传入的thenCombine正常完成后，返回一个新的CompletionStage，并执行action
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,
                                              Runnable action);

```

or聚合关系
```
// 当目前的CompletionStage和传入的CompletionStage其中一个正常完成后，返回一个新的CompletionStage，并执行fn 有返回值
public <U> CompletionStage<U> applyToEither
        (CompletionStage<? extends T> other,
         Function<? super T, U> fn);

// 当目前的CompletionStage和传入的CompletionStage其中一个正常完成后，返回一个新的CompletionStage，并执行action 这个无返回值，参数是Consumer
public CompletionStage<Void> acceptEither
        (CompletionStage<? extends T> other,
         Consumer<? super T> action);

// 任意一个执行完成，进行下一步操作，这个无传参
public CompletionStage<Void> acceptEitherAsync
        (CompletionStage<? extends T> other,
         Consumer<? super T> action);

```


总结
观察一下规律，所有的操作都封装成了一个CompletionStage，一个CompletionStage可以和另外一个CompletionStage产生关系，并且生成新的CompletionStage，并且可以设置发生异常或者正常完成时的操作，注意，这里操作分为三类：Function（有入参和出参）、Consumer（只有入参）、Runnable（没有入参和出参，仅仅执行动作）。并且我在其中省略了某些Async的异步操作，代表不同的CompletionStage可以在不同的线程池中执行。






















