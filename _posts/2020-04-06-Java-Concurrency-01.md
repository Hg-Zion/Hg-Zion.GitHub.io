---
title: 并发编程伪实战——并发基础构建模块
tags: Java 并发
show_author_profile: false
---



## 1. 同步容器类

同步容器类包括 `Vector` 和 `Hashtable`，二者是早期 JDK 的一部分，此外还包括一些由 `Collections.synchronizedXxx` 等工厂方法创建的封装类。它们实现线程安全的方式是：对每个公有方法都进行同步，使得每次只能有一个线程能访问容器。
{:.success}

### 1.1 同步容器类的问题

同步容器类在某些情况下需要在客户端加锁以保护复合操作。
{:.conclude}

这些复合操作通常包括迭代、跳转以及条件运算等，尽管在多线程环境下它们仍是“线程安全”的，但当有其他线程并发修改容器时，就有可能会出现一些意料之外的问题。

```java
public static Object getLast(Vector list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
}
```

尽管上面的 `get()` 方法和 `remove()` 方法都是线程安全的，无论多少个线程去调用它们都不会破坏 `Vector`。但是从调用者的角度来看，如果当线程 A 调用 `get()` 方法时，线程 B 恰好执行了 `remove()` 方法，那么 `getLast()` 将会抛出一个 `ArrayIndexOutOfBoundsException` 异常。

![vector-not-syn](/assets/images/concurrency/vector-not-syn.png)

解决的方法通常是在客户端加锁，同样的例子，我们对传入参数 `list` 进行加锁，就可以保证 `Vector` 的大小在调用 `size()` 和 `get()` 之间不会发生变化。

```java
public static Object getLast(Vector list) {
    synchronized(list) {
        int lastIndex = list.size() - 1;
    return list.get(lastIndex);
    }
}

public static void deleteLast(Vector list) {
    synchronized(list) {
        int lastIndex = list.size() - 1;
    	list.remove(lastIndex);
    }
}
```

另外，虽然不在客户端加锁的程序会抛出异常，但这把并不意味着 Vector 就是线程不安全的。Vector 的状态仍然有效，而抛出的异常也与规范保持一致，只是这种结果不是我们所期望的，不过这种东西了解即可。

### 1.2 迭代器和 fail-fast

由于在设计同步容器类迭代器时未考虑到并发修改问题，因此当它们发现容器在迭代过程中被修改时，就会抛出 `ConcurrentModificationException` 异常。
{:.conclude}

无论在直接迭代还是在 `for-each` 循环语法中，对容器类进行迭代的标准方式都是使用 **Iterator**。在迭代期间，如果有其他线程并发地修改容器，那么将无可避免地对容器加锁。当它们发现容器在迭代过程中被修改时，表现出的行为是及时失败 `fail-fast`，这意味着会抛出 `ConcurrentModificationException` 异常。这种“及时失败”的迭代器并不能捕获并发错误，而只能充当一个并发问题预警器。

`fail-fast` 的实现方式通常是将计数器的变化同容器关联，如果在迭代期间计数器被修改，那么 `hasNext` 或 `next` 将会抛出 `ConcurrentModificationException` 异常。
{:.info}

如果我们不想使用 `fail-fast` 这种软绵绵的保护方式，同时又不希望面对加锁容器后出现的吞吐量降低和CPU利用率降低问题，那么我们可以选择**克隆容器**，并在副本上进行迭代（当然克隆过程中仍需要加锁）。

## 2. 并发容器类

并发容器类是基于旧有的线程安全类在并发性问题上作出的改进。
{:.conclude}

在 *Java 5.0* 中增加了 `ConcurrentHashMap` 用于取代 `HashTable`，增加了 `CopyOnWriteArrayList` 用于取代 `Vector`。此外，还增加了两种新容器类型：`Queue` 和 `BlockingQueue` 用来临时保存一组待处理元素。它们提供的实现包括一个传统的 *FIFO* 队列 `ConcurrentLinkedQueue` 以及一个（非并发）优先队列 `PriorityQueue`。

虽然是通过 `LinkedList` 来模仿并实现队列容器的，但是还是需要一个 `Queue` 接口用来去掉 List 的随机访问需求，从而实现更好的高并发。另外，`BlockingQueue` 有可阻塞的插入和获取操作，无论是获取空队列元素的操作还是插入满队列元素的操作都会让线程阻塞，直到非空或者非满状态的出现。在“生产者-消费者”模式中，阻塞队列是很有用的。

在 *Java 6* 中，还引入了 `ConcurrentSkipListMap` 和 `ConcurrentSkipListSet`。

### 2.1 ConcurrentHashMap

 `ConcurrentHashMap` 使用**分段锁机制**，允许任意数量的读线程并发访问，允许读线程和部分写线程（不在同一段上）并发访问以及允许部分写线程（不在同一段上）并发修改。
{:.conclude}

![CHM](/assets/images/concurrency/lock_compare.png)

### 2.2 CopyOnWriteArrayList

`CopyOnWriteArrayList` 用于取代 `Vector`，在迭代期间不需要对容器进行加锁或复制。类似地，`CopyOnWriteArraySet` 用于取代同步 `Set`。
{:.conclude}

`Copy-On-Write` 意为写入时复制，其线程安全性体现在每次修改时都会创建并重新发布一个新的容器副本，从而实现可变性。这种“写入时复制”的容器返回的迭代器不会抛出 `ConcurrentModificationException` 异常，而且返回的元素和迭代器创建时的元素完全一致，我们只需要通过 volatile 使其底层数组保持可见性即可。

## 3. 阻塞队列和生产者-消费者模式

### 3.1 阻塞队列

阻塞队列提供了可阻塞的 `put` 和 `take` 方法，前者在队列已满时阻塞，后者在队列为空时阻塞。阻塞队列还支持返回布尔值的 `offer` 和 `poll` 方法。

类库中包含了 `BlockingQueue` 的多种实现，其中 `LinkedBlockingQueue` 和 `ArrayBlockingQueue` 是 *FIFO* 队列，与 `LinkedList` 和 `ArrayList` 相似，但是却拥有比同步 `List` 更好的并发性能。`PriorityBlockingQueue` 是一个按优先级顺序排序的队列，当你不希望按照 FIFO 的属性处理元素时，这个 `PriorityBolckingQueue` 是非常有用的。正如其他排序的容器一样，`PriorityBlockingQueue` 可以比较元素本身的自然顺序（如果它们实现了 `Comparable`），也可以使用一个 `Comparator` 进行排序。

最后一个 `BlockingQueue` 的实现是 `SynchronousQueue`，它根本上不是一个真正的队列，因为它不会为队列元素维护任何存储空间。不过，它维护一个排队的线程清单，这些线程等待把元素加入（`enqueue`）队列或者移出（`dequeue`）队列。因为 `SynchronousQueue` 没有存储能力，所以除非另一个线程已经准备好参与移交工作，否则 `put` 和 `take` 会一直阻止。`SynchronousQueue` 这类队列只有在消费者充足的时候比较合适，它们总能为下一个任务作好准备。

### 3.2 双端队列和工作密取

Java 6 新增加了两种容器类型，`Deque` 和 `BlockingDeque`，它们分别是 `Queue` 和 `BlockingQueue` 的双端队列版。具体实现包括 `ArrayDeque` 和 `LinkedBlockingDeque` 。

正如阻塞队列适用于 **生产者-消费者** 模式，双端队列则适用于 **工作密取** 模式。在 生产者-消费者 模式中，所有消费者只有一个共享的工作队列，每次获取工作都将面临竞争，而 工作密取 模式里每个消费者都有一条属于自己的双端工作队列。每次消费者都会从自己的工作队列的头部获取工作，当属于自己的工作队列为空时，它会从别的消费者队列的队尾“偷偷”取走工作，这就是工作密取。

## 4. 阻塞方法和中断方法

线程有时候会阻塞并暂停执行，比如：等待 I/O 操作结束，等待获取一个锁，等待从 `Thread.sleep` 方法中醒来，或是等待另一个线程的计算结果。当线程阻塞时，它将处于 ***BLOCKED***、***WAITING*** 或 ***TIMED_WAITING*** 三种状态中的一种，直到当某个外部事件发生后，它才能重回 ***RUNNABLE*** 状态，并可以被再次调度执行。

阻塞方法会抛出受检查异常 `InterruptedException`，比如 `BlockingQueue` 的 `put` 和 `take` 方法或者 `Thread.sleep` 等。

线程中断可以在线程内部设置一个中断标识，同时让处于(可中断)阻塞的线程抛出 `InterruptedException` 中断异常，使线程跳出阻塞状态。
{:.conclude}

Java 中断机制是一种协作机制，也就是说通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理中断。这好比是家里的父母叮嘱在外的子女要注意身体，但子女是否注意身体，怎么注意身体则完全取决于自己。Java 中断模型也是这么简单，每个线程对象里都有一个 `boolean` 类型的标识（不一定就要是 `Thread` 类的字段，实际上也的确不是，这几个方法最终都是通过 `native` 方法来完成的），代表着是否有中断请求（该请求可以来自所有线程，包括被中断的线程本身）。例如，当线程 **t1** 想中断线程 **t2**，只需要在线程 **t1** 中将线程 **t2** 对象的中断标识置为 `true`，然后线程 **t2** 可以选择在合适的时候处理该中断请求，甚至可以不理会该请求，就像这个线程没有被中断一样。

## 5. 同步工具类

同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。
{:.conclude}

### 5.1 闭锁

闭锁是一种同步工具类，可以用来确保某些活动能够在指定活动完成后再开始执行。CountDownLatch 就是闭锁的一种灵活实现，它可以让一个或一组线程等待一组事件发生。它的闭锁状态包括一个计数器，该计数器被初始化为一个指定的正数，意为等待事件的数量。当其中一个事件完成，调用 countDown 方法递减计数器，而 await 方法会一直阻塞到计数器为零为止，或者等待中的线程中断，或者等待超时。

常见应用场景：

（01）确保某个计算在其所需要的所有资源都被初始化后才继续执行，二元闭锁可以用来表示“资源 R 已经被初始化”，而在此之前所有需要 R 的操作都必须在这个闭锁上等待；

（02）确保某个服务在其依赖的所有服务都启动后再启动，原理同上；

（03）等待直到某个操作的所有参与者就绪再开始执行，当所有参与者就绪时，闭锁到达结束状态。

这里是闭锁的使用示例：

```java
class TestHarness {
    long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
        // 用于等待所有线程就绪
        final CountDownLatch startGate = new CountDownLatch(1);
        // 用于等待所有线程完成任务
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread(() -> {
                try {
                    startGate.await();// await 是阻塞方法
                    try {
                        task.run();
                    } finally {
                        endGate.countDown();
                    }
                } catch (InterruptedException e) {
                }
            });
            t.start();
        }
        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
}
```



### 5.2 FutureTask

`FutureTask` 本质是 `Future` 和 `Runnable` 接口的结合体，其中 `Runnable` 自不必多言，而 `Future` 表示一个可能还没有完成的异步任务的结果。我们可以用 `Future` 去包装一个 `Callable` 接口，然后调用它的 `get()` 方法来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回。

`FutureTask` 的计算是通过 Callable 来实现的，并且可以处于以下三种状态：等待运行、正在运行和完成运行。同 `Future` 类似地，当任务完成后，它的 `get()` 方法会立即返回获取的执行结果，否则将阻塞直到任务完成后返回结果或者抛出异常。因此，我们常常用它来执行一些费时较长的运算，并且计算结果会在稍后使用。这样，通过提前启动计算，可以减少等待时间。

### 5.3 信号量

信号量通常用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。计数信号量可以用来实现某种资源池，或者对容器施加边界。

`Semaphore` 实现资源池的原理是：将 `Semaphore` 初始化为池的大小，构造一个固定长度的资源池。每当从池中获取资源时，需要通过 `acquire` 方法先向资源池请求一个许可，等资源返回池后，再调用 `release` 方法释放许可。

```java
public class BoundHashSet<T> {
    private final Set<T> set;
    private final Semaphore sem;
    
    public BoundHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem =new Semaphore(bound);
    }
    
    public boolean add(T o) throws InterruptedException {
        sem.acquire(); // 阻塞方法
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        }
        finally {
            if(!wasAdded)
                sem.release(); // 添加失败，当场释放
        }
    }
    
    public boolean remove(Object o){
        boolean wasRemoved = set.remove(o);
        if(wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
```



### 5.4 栅栏

栅栏类似于闭锁，它也能阻塞一组线程直到某个事件发生，但是闭锁是一次性对象，而栅栏则可以多次使用。
{:.conclude}

`CyclicBarrier` 可以使一定数量的参与方反复在栅栏位置集合，它在并行算法中非常有用：这种算法通常将一个问题拆分成一系列独立的子问题。在线程到达栅栏位置时会调用 `await()` 方法阻塞，直到所有线程都到达栅栏位置。如果所有线程都到达了栅栏 ，那么栅栏将打开，此时所有线程都被释放，而栅栏将被重置以便下次使用。如果对 `await()` 的调用超时，后者 `await()` 阻塞的线程被中断，那么栅栏就被认为是打破了，所有阻塞的 `await()` 调用都将终止并抛出 `BrokenBarrirerException`。如果成功地通过栅栏，那么 `await()` 将为每个线程返回一个唯一的到达索引号，我们可以利用这些索引来“选举”产生一个领导线程，并在下一次迭代中由该领导线程执行一些特殊的工作。`CyclicBarrier` 还可以使你将一个栅栏操作传递给构造函数，这是一个 `Runnable`，当成功通过栅栏会（在一个子任务线程中）执行它，但在阻塞线程被释放之前是不能执行的。

另一种栅栏是 `Exchanger`，很多消息中间件在用，这是一种两方（*Two-Party*）栅栏。

