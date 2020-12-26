---
title: jdk1.8 之 CompletableFuture与CompletionService
date: 2020-12-27 15:54:38
tags: 
- java
- juc
categories: 
- java
img: https://cdn.nlark.com/yuque/0/2020/png/638783/1608742074027-20028756-59d6-47d7-825e-b19570defcf3.png?x-oss-process=image%2Fresize%2Cm_fill%2Cw_400%2Ch_250
newimg: true
zhailu: 通常来说我们使用线程池做异步回调的时候，最明显的方法事通过`future.get()`,这并没有什么问题，但是如果我如果遇到这样一个场景需要我们并行执行任务，其中某一个任务执行先结束了，那么中断其他任务或者是直接放弃其他任务这样的需求呢？
---



## 1. CompletionService
通常来说我们使用线程池做异步回调的时候，最明显的方法事通过`future.get()`,这并没有什么问题，但是如果我如果遇到这样一个场景需要我们并行执行任务，其中某一个任务执行先结束了，那么中断其他任务或者是直接放弃其他任务这样的需求呢？


### 1.1 案例
**普通使用线程池执行异步任务：**
```java


static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(100,100,0L
                                                                , TimeUnit.MILLISECONDS,new SynchronousQueue<>(),new ThreadPoolExecutor.CallerRunsPolicy());


// main方法运行50个不同休眠时间的任务
public static void main(String[] args){
    List<Callable<Integer>> callables = new ArrayList<>();
    for (int i = 0; i < 50; i++) {
        callables.add(() -> {
            int i1 = Math.abs((new Random().nextInt() + 1)) % 6;
            TimeUnit.SECONDS.sleep(i1);
            return i1;
        });
    }
    long beg = System.currentTimeMillis();

    //调用下面方法
    futures(callables);
    System.out.println("执行完成:" + (System.currentTimeMillis() - beg) +"毫秒");
}

// 此方法使用线程池的invokeAll方法提交所有任务，并使用for循环get来获取结果
public static void futures(List<Callable<Integer>> callables){
    List<Future<Integer>> futures = null;
    try {
        futures = poolExecutor.invokeAll(callables);
        for (Future<Integer> future : futures) {
            System.out.print(future.get()+ " ,");
        }
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
    }
    System.out.println("");
}
```
![image.png](https://cdn.nlark.com/yuque/0/2020/png/638783/1606293043772-6dc41c7e-50f5-4176-89e0-5149161ef54a.png#align=left&display=inline&height=238&margin=%5Bobject%20Object%5D&name=image.png&originHeight=238&originWidth=1548&size=17696&status=done&style=none&width=1548)
常规的写法正常来说并没有什么问题，问题在于第29行`future.get()`会阻塞住第一个任务而没办法检查其他任务是否执行完了，在我们开头说的问题上，这种方式是没办法实现的




**接下来我们在看看`CompletionService`的方式：**
```java


static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(100,100,0L
                                                                , TimeUnit.MILLISECONDS,new SynchronousQueue<>(),new ThreadPoolExecutor.CallerRunsPolicy());


// main方法运行50个不同休眠时间的任务
public static void main(String[] args){
    List<Callable<Integer>> callables = new ArrayList<>();
    for (int i = 0; i < 50; i++) {
        callables.add(() -> {
            int i1 = Math.abs((new Random().nextInt() + 1)) % 6;
            TimeUnit.SECONDS.sleep(i1);
            return i1;
        });
    }
    long beg = System.currentTimeMillis();
	
    //调用下面方法
    futures(callables);
    System.out.println("执行完成:" + (System.currentTimeMillis() - beg) +"毫秒");
}

// 使用CompletionService的submit提交任务，并for循环get获取结果。
public static void futures(List<Callable<Integer>> callables){
    CompletionService<Integer> completionService = new ExecutorCompletionService<>(poolExecutor);
    for (Callable<Integer> callable : callables) {
        completionService.submit(callable);
    }
    for (Callable<Integer> callable : callables) {
        try {
            Future<Integer> future = completionService.take();
            System.out.print(future.get()+ " ,");
        }catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
    System.out.println("");
}
```
![image.png](https://cdn.nlark.com/yuque/0/2020/png/638783/1606293287571-bcefcce9-e1bc-4f69-aacd-088c5deee1f1.png#align=left&display=inline&height=234&margin=%5Bobject%20Object%5D&name=image.png&originHeight=234&originWidth=1613&size=19201&status=done&style=none&width=1613)
从整体运行时间上来看并没有差距（但是在需要任务返回之后执行其他操作的情况会缩短很多时间），但是细心的朋友们会发现返回值也就是休眠时间是先获取的时间最短的，也就是说它做到了执行快的任务先返回




### 1.2 执行原理
`CompletionService`功能不多仅有四个方法，`submit`两个重载方法是提交任务，`take()`方法是阻塞等待最先执行完成的任务，`poll()`的第一个方法同样也是获取执行完的任务，但与take()方法不同的是它不会阻塞，另外一个带参数的显然就是等待超时获取了。
```java
public interface CompletionService<V> {

    // 提交任务
    Future<V> submit(Callable<V> task);

	// 提交任务，给Runnable一个默认返回值
    Future<V> submit(Runnable task, V result);
    
    // 获取第一个完成的任务(阻塞)
    Future<V> take() throws InterruptedException;

    // 获取任务，立即返回
    Future<V> poll();

	// 获取任务，超时立即返回
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}

```
`CompletionService`接口有一个默认实现`ExecutorCompletionService<V>`类，在这个类的构造器中创建了一个`LinkedBlockingQueue<Future<V>>()`的阻塞队列，这个队列是就是执行结束后的任务队列，如何在任务执行结束后加入队列，就是此类的核心关键，知道了这个问题，其他的方法就迎刃而解了。
       在我们调用`submit`方法时，会将任务再包装一层`QueueingFuture`，此任务是个内部类，并且它重写了`done()`这个方法，如果查看线程池执行流程的话，他任务执行结束会执行到`done()`这个方法中，那么通过这种流程中调用`done()`+闭包的方式，就可以将执行完成的任务在合适的时机放入我们想要的队列中，剩下就是队列的事情了。
```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor;
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue;

 //...省略代码
    
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        // 重写方法操作当地对象队列
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }
     
    public ExecutorCompletionService(Executor executor) {
        if (executor == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = new LinkedBlockingQueue<Future<V>>();
    }


    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task);
        // 执行前封装一层
        executor.execute(new QueueingFuture(f));
        return f;
    }

	//阻塞队列获取
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

//...省略代码

}
```

**总结**：通过这一点源码，也引出了不少设计模式的思想，比如将`QueueingFuture(f)`就类似装饰者模式，再看`QueueingFuture` 重写 `done()` 方法也能看作是模板方法的作用。






## 2. CompletableFuture
CompleteFuture。它实现了Future接口和CompletionStage接口,它采用了lambda和流式调用加异步回调的方式，更大程度上解决了多线程间的依赖关系，并且拥有和Stream一样的链式操作，能让整个业务变得简洁清晰、任务的可控粒度更小，并且提高完善的异常处理


### 2.1 api介绍
参考java8api文档 [https://www.matools.com/api/java8](https://www.matools.com/api/java8)
由于此类的api重复功能的方法过多，这里就不一一展开来说，大概分为一下几个功能：


#### 2.1.1 静态方法

- [allOf](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-)

合并任务，全部完成后再执行下一步操作


- [anyOf](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#anyOf-java.util.concurrent.CompletableFuture...-)

合并任务，其中一个完成后再执行就可以执行下一步操作


- **[runAsync](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#runAsync-java.lang.Runnable-)**([Runnable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Runnable.html) runnable) 

运行一个Runnable任务

- **[supplyAsync](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-)**([Supplier](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Supplier.html)<U> supplier) 

运行一个Supplier任务

- **[completedFuture](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#completedFuture-U-)**(U value)

返回已经使用给定值完成的新的CompletableFuture。




#### 2.1.2 依赖上一步完成后执行

- **[thenRun](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenRun-java.lang.Runnable-)**([Runnable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Runnable.html) action) 

下一步运行一个Runnnable任务


- **[thenAccept](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenAccept-java.util.function.Consumer-)**([Consumer](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Consumer.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)> action) 

下一步运行一个Consumer任务


- **[thenApply](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenApply-java.util.function.Function-)**([Function](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Function.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),? extends U> fn)

下一步运行一个Function任务


- **[thenCompose](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-)**([Function](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Function.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),? extends [CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<U>> fn)

下一步运行一个Fcunction任务，**它与**`**thenApply**`**不同的是它用于合并两个任务返回一个新任务**


- **[thenAcceptBoth](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenAcceptBoth-java.util.concurrent.CompletionStage-java.util.function.BiConsumer-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<? extends U> other, [BiConsumer](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/BiConsumer.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),? super U> action)

下一步运行一个可选择一个任务，等待这两个任务全部完成，会执行**BiConsumer**任务，上两个任务的返回值作为参数


- **[thenCombine](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<? extends U> other, [BiFunction](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/BiFunction.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),? super U,? extends V> fn)

下一步运行一个可选择一个任务，等待这两个任务全部完成，会执行**BiFunction**任务，上两个任务的返回值作为参数


- **[whenComplete](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#whenComplete-java.util.function.BiConsumer-)**([BiConsumer](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/BiConsumer.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),? super [Throwable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Throwable.html)> action)

下一步运行一个Consumer任务，他与then类型任务的最大不同是它可以处理异常结果，不会跳过。



#### 2.1.3 任务合并方法
- **[allOf](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-)**([CompletableFuture](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)<?>... cfs)

聚合指定任务，返回一个新任务。需要全部任务都执行完毕才继续下一步操作


- **[anyOf](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#anyOf-java.util.concurrent.CompletableFuture...-)**([CompletableFuture](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)<?>... cfs)

聚合指定任务，返回一个新任务。需某一个任务执行完毕就继续下一步操作

- **[acceptEither](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#acceptEither-java.util.concurrent.CompletionStage-java.util.function.Consumer-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<? extends [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)> other, [Consumer](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Consumer.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)> action)

其中某一个一个任务完成后就会执行**Consumer**任务

- **[applyToEither](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#applyToEither-java.util.concurrent.CompletionStage-java.util.function.Function-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<? extends [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html)> other, [Function](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/Function.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),U> fn)

其中某一个一个任务完成后就会执行**Function**任务

- **[runAfterBoth](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#runAfterBoth-java.util.concurrent.CompletionStage-java.lang.Runnable-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<?> other, [Runnable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Runnable.html) action)

当这个和另一个给定的阶段都正常完成时，执行给定的动作。

- **[runAfterEither](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#runAfterEither-java.util.concurrent.CompletionStage-java.lang.Runnable-)**([CompletionStage](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletionStage.html)<?> other, [Runnable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Runnable.html) action)

当这个或另一个给定阶段正常完成时，执行给定的操作。




#### 2.1.4 异常处理

- **[handle](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-)**([BiFunction](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/function/BiFunction.html)<? super [T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html),[Throwable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Throwable.html),? extends U> fn)

无论是否异常都会走此方法


- **[completeExceptionally](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#completeExceptionally-java.lang.Throwable-)**([Throwable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Throwable.html) ex)

异常结果处理方法




#### 2.1.5 任务中断处理

- **[completeExceptionally](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#completeExceptionally-java.lang.Throwable-)**([Throwable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Throwable.html) ex)

如果尚未完成，则调用 [get()](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get--)和相关方法来抛出给定的异常。


- **[complete](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#complete-T-)**([T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html) value)

如果尚未完成，将返回的值 [get()](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get--)和相关方法为给定值。


- **[obtrudeException](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#obtrudeException-java.lang.Throwable-)**([Throwable](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/Throwable.html) ex)

强制导致后续调用方法 [`get()`](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get--)和相关方法抛出给定的异常，无论是否已经完成。

- **[obtrudeValue](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#obtrudeValue-T-)**([T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html) value)

强制设置或重置随后方法返回的值 [get()](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get--)和相关方法，无论是否已经完成。


- **[cancel](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#cancel-boolean-)**(boolean mayInterruptIfRunning)

中断任务，如果尚未完成，请使用`CancellationException`完成此[CompletableFuture](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CancellationException.html) 。

- **[isCancelled](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#isCancelled--)**()

  验证是否终端取消

- **[isCompletedExceptionally](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#isCompletedExceptionally--)**()

验证是否异常返回取消


- **[isDone](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#isDone--)**()

验证时否取消(任何方式)



#### 2.1.7 其他方法


- **[get](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get--)**()

等待这个未来完成的必要，然后返回结果。

- **[get](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#get-long-java.util.concurrent.TimeUnit-)**(long timeout, [TimeUnit](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/TimeUnit.html) unit)

如果有必要等待这个未来完成的给定时间，然后返回其结果（如果有的话）。

- **[getNow](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#getNow-T-)**([T](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html) valueIfAbsent)

如果已完成，则返回结果值（或抛出任何遇到的异常），否则返回给定的值IfAbsent。


- **[getNumberOfDependents](https://www.matools.com/file/manual/jdk_api_1.8_google/java/util/concurrent/CompletableFuture.html#getNumberOfDependents--)**()

返回完成等待完成此CompletableFuture的CompletableFutures的估计数。

