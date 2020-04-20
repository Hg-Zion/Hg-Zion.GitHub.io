---
title: 并发编程伪实战——任务执行
tags: Java 并发
show_author_profile: false
---





> “喜欢蓝色，等以后学会配图了，就用黄、蓝、白、灰或者红、蓝、黑配色吧！”

## 1.在线程中执行任务

### 1.1 为任务创建线程

这里是一个串行处理服务器请求的 *Web* 服务器程序：

```java
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            Socket connection =socket.accept();
            handleRequest(connection);  // 处理接收到的请求
        }
    }
}
```

`SingleThreadWebServer` 本身很简单而且理论上也没有问题，但是在实际生活中会表现地非常糟糕，其原因在于它每次只能服务一个请求，而且处理请求和接收请求操作不能并发执行，这意味着新到来的请求必须等待前一个请求处理完毕后才能被接收。

```java
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            final Socket connection =socket.accept();
            Runnable task = new Runnable() {
                @Override
                public void run() {
                    // 处理接收到的请求
                    handleRequest(connection);
                }
            };
            new Thread(task).start(); // 不会停顿
        }
    }
}
```

我们对其进行了改进，让它能够迅速接收新的请求而无需等待之前请求的处理操作完成。请求的处理过程被我们用 `Runnable` 包装起来放进一个新创建的线程里，然后用 `start()` 方法启动新建的线程，而主线程则继续去接收新的请求不会原地停顿。

总结：（01）任务处理过程要从主线程中**分离**出来，方便主线程去接收下一个连接；

​			  （02）任务和任务之间可以**并行**处理，提高程序吞吐量；

​			  （03）任务处理的代码必须是**线程安全**的。



1.2 显式地为任务创建线程

### 1.2 无限制创建线程的不足

改进后的代码仍然有不足，最主要的一点是没有对线程创建数量进行限制，而创建线程的代价是很高的。事实上，为每一个任务专门创建线程去执行会存在一些通用的毛病：

（01）**线程生命周期的开销非常高**：线程的创建和销毁都需要消耗一定时间，也会有延迟处理的请求；

（02）**资源消耗**：尤其是内存，当可用线程数高于可用处理器数时，大量闲置的线程会占据很多内存，可用内存的减少会给垃圾回收器带来回收压力，另一方面大量闲置线程对 *CPU* 的争用也会产生很多开销；

（03）**稳定性**：在可创建线程的数量上存在一个限制。这个限制包括 *JVM* 的启动参数、`Thread` 构造函数中请求栈的大小，以及底层系统对线程的限制等。如果破坏了这些限制，会引起 `OutOfMemoryError` 异常。

## 2.Executor 框架

之前讲到过的串行执行任务的问题在于其糟糕的响应性和吞吐量，而“为每一个任务分配一个线程”又有资源管理复杂性的问题，而 `Executor` 则为异步任务执行框架提供了基础。
{:.conclude}

```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

`Executor` 是一个基于生产者 - 消费者模式的简单接口，提交任务的操作相当于生产者，执行任务的线程相当于消费者。



### 2.1 基于 Executor 的 Web 服务器

```java
public class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec
            = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true){
            final Socket connection =socket.accept();
            Runnable task = new Runnable() {
                @Override
                public void run() {
                    // 处理接收到的请求
                    handleRequest(connection);
                }
            };
            exec.execute(task);
        }
    }
}
```

通过使用 Executor，我们将任务提交和任务执行解耦。并且，如果我们想要改变服务器的行为，只需要采用另外一种 Executor 去执行即可。

```java
// 通过 Executor 重新实现“为每个请求启动一个新线程”
public class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
}

// 通过 Executor 重新实现单线程行为
public class WithinThreadExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();
    }
}
```



### 2.2 线程池

通过重用现有线程而不是创建新线程，可以在处理多个请求时分摊在线程创建或销毁过程中产生的巨大开销。另外，当请求到达时，工作线程通常已经存在，因此不会因为等待创建线程而延迟任务的执行。
{:.conclude}

创建线程池可用通过调用 `Executors` 中的静态方法来实现：

（01）**newFixedThreadPool**：创建一个固定长度的线程池，每当有新任务提交时创建一个新线程直到达到预定规模上限。在此之后，如果有线程因为意外的 `Exception` 而结束，线程池会补充一个新的线程。

（02）**newCachedThreadPool**：多了就回收一些，不够就新创建一些，基本没什么限制，好像是个很鸡肋的家伙呢。

（03）**newSingleThreadPool**：顾名思义，在这个线程池中只有一个线程，如果该线程因为意外的 `Exception` 而结束，线程池会补充一个新的线程。`newSingleThreadPool` 最有用的地方在于它能够保证任务被顺序执行（例如 *FIFO*、*LIFO*、优先级）。

（04）**newScheduledThreadPool**：是 `newFixedThreadPool` 的增强版，它能以延时或定时的方式去执行任务，类似于 `Timer`。

### 2.3 Executor 的生命周期

我们知道，*JVM* 只有在所有非守护线程全部终止后才会退出，而 `Executor` 内部的线程却是常驻的，因此，如果无法正确关闭 `Executor`，那么 *JVM* 将无法结束。为了解决这个问题，`Executor` 扩展了一个 `ExecutorService` 接口用来管理生命周期以及一些方便提交任务的方法。

```java
public interface ExecutorService extends Executor {
	void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    
    // ============= 一些用于提交任务的便利方法 =======================
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    // ..后面就懒得贴了
}
```

`ExecutorService` 的生命周期有三种状态：运行、关闭和终止，通常如果我们想要关闭线程池会先调用 `awaitTermination` 后立即调用 `shutdown()`。
{:.conclude}

`shutdown()` 是一种比较温和的关闭方式，它会拒绝新的任务提交，然后将已提交任务包括还在等待执行的任务执行完毕，当然它的“粗暴模式”就是 `shutdownNow()` 方法了，后者甚至会尝试取消正在执行的任务。

## 3.找出可利用的并行性

### 3.1 携带结果的任务 Callable 和 Future

很多时候我们需要执行的任务都是存在延时的计算——执行数据库查询、从网络上获取资源、计算某个复杂的功能等，对于这些任务，`Callable` 是一种比 `Runnable` 更好的任务抽象：它会通过 `call()` 方法返回一个值，并可能会抛出一个异常。（我们也可以通过 `Executor` 将 `Runnable` 封装成一个 `Callable<Void>` 对象）

`Future` 表示一个任务的生命周期，它提供了相应的方法来判断任务是否已经完成或取消，以及获取任务的结果和取消任务等。

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

public interface Future<V> {
	boolean cancel(boolean mayInterruptIfRunning);
	boolean isCancelled();
	boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

`get` 方法行为取决于任务的状态，如果已经完成了那么它会立即返回结果或者抛出一个异常；如果没有完成，那么 `get` 方法将阻塞直到任务完成。

创建 `Future` 的方式也很多，`ExecutorService` 中的所有 `submit` 方法都将返回一个 `Future`，从而将一个 `Runnable` 或 `Callable` 提交给 `Executor`，并得到一个 `Future` 用来获取任务执行的结果或取消任务。还可以显式地将某个 `Runnable` 或 `Callable` 实例化为 `FutureTask`。

【示例】使用 Future 实现页面渲染器

```java
public class FutureRenderer {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    void renderPage(CharSequence source) {
        final List<ImageInfo> imageInfos = scanForImageInfo(source);
        Callable<List<ImageInfo>> task = new Callable<List<ImageInfo>>() {
            @Override
            public List<ImageInfo> call() throws Exception {
                List<ImageInfo> result = new ArrayList<>();
                for(ImageInfo imageInfo:imageInfos)
                    result.add(imageInfo.downloadImage());
                return result;
            }
        };
        // 创建 Future
        Future<List<ImageInfo>> future = executor.submit(task);
        rederText(source);
        
        try{
            // 获取结果
            List<ImageInfo> imageData = future.get();
            for(ImageInfo imageInfo:imageData)
                renderImage(Data);
            return result;
        } catch (InterruptedException e){
            // 重新设置中断
            Thread.currentThread().interrupt();
            // 由于不需要结果，取消
            future.cancel(true);
        } catch (ExecutionException e) {
            // 获取被 ExecutionException 封装的异常
            throw launderThrowable(e.getCause());
        }
    }
}

```



