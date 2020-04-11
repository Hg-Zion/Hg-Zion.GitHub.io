---
title: 浅入理解JVM（四）—— 线程安全与锁优化
tags: Java JVM
show_author_profile: true
---



# 线程安全与锁优化

## 1.Java 语言中的线程安全

我们可以将 Java 语言中的各种操作共享的数据按照“安全程度”由强至弱分为五类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。
{:.success}

### 1.1 不可变

在 Java 中，不可变的对象一定是线程安全的。
{:.conclude}

这种不可变是由 `final` 关键字实现的，而之前我们有讲过关于 `final` 关键字带来的可见性。只要一个不可变的对象被正确地构建出来（没有发生 `this` 引用逃逸的情况），那么其外部的可见状态永远都不会改变。

在 Java 中，如果共享的数据是一个基本数据类型，那么只要在定义时使用 `final` 关键字去修饰它就可以保证它是不可变的。至于对象类型，就需要其自行保证其行为不会对其状态产生任何影响。比如 `String` 类的对象实例，无论调用它的哪种方法都不会影响其原来的值，任何修改都只会返回一个新构造的字符串对象。

当然，保证对象行为不影响自己状态的方法不止一种，最简单的就是将对象里面带状态的变量都声明为 `final`，比如 `java.lang.Integer` 构造函数：

```java
/**
 * The value of the {@code Integer}.
 *
 * @serial
 */
private final int value;

/**
 * Constructs a newly allocated {@code Integer} object that
 * represents the specified {@code int} value.
 *
 * @param   value   the value to be represented by the
 *                  {@code Integer} object.
 */
public Integer(int value) {
    this.value = value;
}
```

在 Java 库中符合不可变要求的类型除了上面提到过的 `String` 外，常用的还有枚举类型和 `java.lang.Number` 的部分子类，如 `Long` 和 `Double` 等数值包装类型、`BigInteger` 和 `BigDecimal` 等大数据类型。但同为 `Number` 子类型的原子类 `AtomicInteger` 和 `AtomicLong` 则是可变的，它没有使用 `final` 而是使用 `volatile` 去修饰值，从而使其能够被多个线程所共享。

```java
private volatile int value;

public AtomicInteger(int initialValue) {
    value = initialValue;
}

public AtomicInteger() {
}
```

### 1.2 绝对线程安全

一个类不管运行时环境如何，调用者都不需要任何额外的同步措施，这个类就是绝对安全的。
{:.conclude}

但是这里的“绝对”其实是很难达到的，通常我们认为 `Vector` 就算一个线程安全的类了，它的每个方法都用 `synchronized` 修饰过，但即使这样，如果在调用它时也有可能出现同步问题。在下面的测试代码中，由于线程 A 调用的 `remove` 方法可能会将线程 B 要访问的元素删除，从而导致报错。

```java
private static Vector<Integer> vector = new Vector<>();

public static void main(String[] args) {
    while (true) {
        for (int i = 0; i < 10; i++) {
            vector.add(i);
        }

        Thread removeThread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < vector.size(); i++)
                    vector.remove(i);
            }
        });

        Thread printThread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < vector.size(); i++) {
                    System.out.print(vector.get(i));
                }
            }
        });

        removeThread.start();
        printThread.start();

        /**
         *  产生异常:
         *  at java.util.Vector.get(Vector.java:751)
         * 	at VectorTests$2.run(VectorTests.java:28)
         * 	at java.lang.Thread.run(Thread.java:748)
         */
    }
}
```

类似这种由于外部的方法调用缺少同步措施导致的线程安全问题，都无法简单通过对类本身进行修改来解决。假如真的要做到绝对线程安全，那么就必须还要在 `Vector` 类内部维护一组一致性的快照访问才行，当然这样做会导致需要付出的时间和空间成本急剧增加。

### 1.3 相对线程安全

我们通常所说的线程安全的类就是属于相对线程安全的，它只需要保证这个对象的单次操作是线程安全的即可。

比如：`Vector`、`HashTable`、`ConcurrentHashMap` 等。

### 1.4 线程兼容

线程兼容是指对象本身并非线程安全的，但是可以通过在外部调用端正确的使用同步手段来保证对象在并发环境中可以安全地使用。

比如：`ArrayList`、`HashMap` 等。

### 1.5 线程对立

线程对立是指不管调用端是否采取了同步措施都无法在多线程环境下并发使用代码，比如 `Thread` 类的 `suspend()` 和 `resume()` 方法。如果有两个线程同时持有一个线程对象，一个尝试去中断，一个尝试去恢复。假如要中断的对象正好是尝试去恢复的线程，那必然会导致死锁。也正因为如此，这两个方法后来都被废弃了。

## 2.实现线程安全

### 2.1 互斥同步

互斥是实现同步的一种手段，而同步则是互斥的目的。
{:.conclude}

**1 synchronized**

在 Java 中，最基本的互斥同步手段就是 `synchronized` 关键字，这是一种块结构的同步语法。经过编译后，会在同步块前后形成 `monitorenter` 和 `monitorexit` 两个字节码指令。这两个字节码指令都需要一个 `reference` 类型的参数来指明要锁定和解锁的对象。一般来说，如果 `synchronized` 指明了对象参数，那就用这个对象的引用作为 `reference`；如果没有明确指明，那就会根据 `synchronized` 修饰的方法类型来判断，如果是类方法就取类型对应的 `Class` 对象，如果是实例方法就取对象实例来作为线程要持有的锁。

此外，在执行 `monitorenter` 指令时，首先要尝试获取对象的锁，如果该对象没有被锁定或者当前线程已经持有该对象的锁，就把锁的计数器的值加一，而放过来执行 `monitorexit` 时则减一。当计数器值重回零时，锁随即释放。

**2 Lock 接口**

在 `Java.util.concurrent` （简称 `J.U.C` ） 包中，其 `locks.Lock` 接口提供了一种全新的互斥同步手段，能够让用户以非块结构来实现互斥同步。重入锁 `ReentrantLock` 是 `Lock` 接口最常见的一种实现，顾名思义，它和 `synchronized` 一样是可重入的。它同 `synchronized` 相比，多了三个高级功能：**等待可中断**、**公平锁**和**锁绑定多个条件**。

（01）等待可中断：当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待。

（02）公平锁：是指多个等待线程按照申请锁的顺序来依次获得锁，`ReentrantLock` 默认是非公平的，但可以通过带布尔值的构造函数要求使用公平锁；

（03）锁绑定多个条件：一个 `ReentrantLock` 对象可以同时绑定多个 `Condition` 对象，但在 `synchronized` 中，只能通过 `wait()`、`notify()` 或 `notifyAll()` 方法配合实现一个隐含的条件。



### 2.2 非阻塞同步

非阻塞同步本质上是一种基于冲突检测的乐观并发策略。
{:.conclude}

通俗地说就是，先不管线程安全问题，直接进行操作，如果没有其他线程争用共享数据，那操作就成功了；如果确实发生了冲突，那就进行相应的“补偿”比如最常见的不断重试。

这种乐观并发策略需要硬件指令集的发展支持，更确切地说是通过硬件来保证“操作”和“冲突检测”这两个步骤具有原子性（通过互斥同步显然就没有意义了啊）。硬件指令常用的有：

（01）测试并设置 Test-and-Set；

（02）获取并增加 Fetch-and-Increment；

（03）交换 Swap；

（04）**比较并交换 Compare-and-Swap，简称 CAS；**

（05）加载链接/条件存储 Load-Linked/Store-Condition，简称 LL/SC；

后两条是现代处理器新增的，而且功能和目的也类似，但由于 Java 只暴露了 `CAS`，因此我们主要关注的就是第四条。`CAS` 指令需要有三个操作数，分别是内存位置、旧的预期值和准备设置的新值，`CAS` 指令执行时当且仅当前两者相等时，才会将新值更新内存位置上的值，这个过程是一个原子操作。

在 Java 的 `sun.misc.Unsafe` 类里面的 `compareAndSwapInt()` 和 `compareAndSwapIong()` 等几个方法包装提供 `CAS`，不过由于它被限制了只有启动类加载器加载的 `Class` 才能访问它，因此普通用户程序是无法调用的，不过可以通过反射区突破这个限制。

此外，`CAS` 仍存在一个逻辑上的漏洞，那就是“ABA”问题，如果一个变量在初次读取和最后确认期间变化了很多次最后又变回原来的值了，那么 `CAS` 会误认为它没有变过。不过大部分情况这种问题都不会影响程序并发的正确性，了解即可。

### 2.3 无同步方案

同步只是保障共享数据在被多个线程争用时能够正确读写的手段，如果本来旧不涉及共享数据，那它自然就不需要任何同步措施去保证其正确性。

在 Java 中，可以通过 `java.lang.ThreadLocal` 类来实现线程本地存储的功能。具体地讲，每一个线程的 `Thread` 对象中都有一个 `ThreadLocalMap` 对象，这个对象存储了一组以 `ThreadLocal.threadLocalHashCode` 为键，以本地线程变量为值的 `K-V` 键值对，`ThreadLocal` 对象就是当前线程的 `ThreadLocalMap` 的访问入口，每一个 `ThreadLocal` 对象都包含了一个独一无二的 `threadLocalHashCode` 值，使用这个值就可以在线程 `K-V` 键值对中找回对应的本地线程变量了。

## 3.锁优化

### 3.1 自旋锁和自适应自旋

由于在 Java 虚拟机中采取的线程实现是一对一模式，因此线程的挂起、恢复都需要转入内核态中完成，而互斥同步会带来很多阻塞操作。而另一方面，共享数据的锁定状态通常只会持续一小会儿，为了这段时间去挂起和恢复线程引起的系统调用开销并不太值得。

因此，如果物理机器有两个或以上的核心可以允许两个以上的线程并行，我们就可以让后面请求锁的线程“等一会儿”，但这个等待本质是执行一个忙循环（自旋），相当于抱着一个内核在“挂机”，等前面占有锁的线程执行完毕后就退出循环继续执行代码。

在 JDK 6 中又引入了自适应的自旋，这个自适应是指自旋等待时间将不再固定，而是根据前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定的。如果锁的对象在上一次通过自旋等待成功获得过锁，那么再次遇到同一个锁对象，会延长其等待时间，相应的，如果上一次失败了再次遇到就会缩短它的自旋等待时间。

### 3.2 锁消除

锁消除是其实是一种编译优化，对被检测到不可能存在共享数据竞争的锁进行消除。

```java
public String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```

上面这段代码其实隐含了同步，由于 Javac 编译器会对 `String` 连接做优化，字符串加法将会转化为 `StringBuffer` 对象的连续 `append()` 操作（在 JDK 5 之后会转换为 `StringBuilder` 对象），等价于：

```java
public String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);

    return sb.toString();
}
```

每个 `StringBuffer` 对象的 `append()` 方法都有一个同步块，锁就是 `sb` 对象。虚拟机观察变量 `sb`，经过逃逸分析后会发现它的动态作用域被限制在 `concatString()` 方法内部，其他线程永远也无法访问到它。所以虽然这里有锁，但是可以被安全地清除掉。

### 3.3 锁粗化

同锁消除相反，锁粗化指的是合并同步范围。如果有一系列的连续操作都对同一个对象进行反复加锁和解锁，甚至加锁操作是出现在循环体中的，那么即使没有县城竞争也会导致不必要的性能损耗，刚刚的代码展示的连续三个 append() 方法正是如此。如果虚拟机探测到有这样一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展到整个操作序列的外部。

### 3.4 轻量级锁

todo

### 3.5 偏向锁

todo