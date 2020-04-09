---
title: 浅入理解JVM（一）
tags: Java JVM
show_author_profile: true
---

<div>{%- include extensions/netease-cloud-music.html id='551722261' -%}</div>

# Part1.	Java内存区域


## 1.运行时数据区域



在程序运行过程中，Java 虚拟机会把它所管理的内存区域分为五个部分：程序计数器、虚拟机栈、本地方法栈、堆以及方法区。其中，在JDK 1.7及以前版本，方法区保留在堆中，但在 JDK 1.8 中方法区被转移到了直接内存中并更名为元空间，并且“永久代”这一概念也随之放弃了。
{:.success}

（01）JDK 7 内存区域：

![1.7内存区域](/assets/images/jvm/image/Java1_7.png)

（02）JDK 8 内存区域：

![1.8内存区域](/assets/images/jvm/image/Java1_8.png)

### 1.1 程序计数器

程序计数器又叫行号指示器，它用来指示下一条待执行的字节码指令。而由于在多线程环境下，线程会轮流切换，为了在线程切换后能够正确执行当前线程的下一条指令，需要给每一个线程都配一个独立的程序计数器，因此程序计数器所在内存区域是“**线程私有**”的。

注意：

（01）如果线程正在执行一个“**本地方法**”，则程序计数器不再指示行号，而是将值设为空。
{:.error}

（02）程序计数器是唯一不会出现 `OutOfMemoryError` 情况的内存区域。
{:.error}

### 1.2 虚拟机栈

Java 虚拟机栈很像我们通常泛泛而谈的“<b>栈</b>”，每当进入一个方法，虚拟机栈就会创建并压入一个“<b>栈帧</b>”，当方法执行完毕后，虚拟机栈又会把这个“栈帧”弹出来。

这个栈帧包含了“<b>局部变量表</b>”和其他一些东西，而这个局部变量表可以存放编译期可知的 **8 种基本数据类型**、**对象引用**（不是对象本身而是一个指向对象起始地址的指针）和 ***returnAddress* 类型**（指向一条字节码指令的地址）。

这些数据类型通过局部变量槽 *Slot*（我觉得英文好记些）来存储，除了 long 和 double 这俩类型要占两个 *Slot*，其余数据类型都是一个。每个方法要分配 *Slot* 数目在进入方法时是完全确定的且运行期间不会改变，但每个 *Slot* 具体大小却是由实际执行的虚拟机决定。

异常情况分析：

（01）线程请求栈深度过大，将抛出 `StackOverflowError` 异常；
{:.error}

（02）虚拟机栈动态扩展时内存不足，则抛出 `OutOfMemoryError` 异常。
{:.error}


### 1.3 本地方法栈

简单来说，就是为执行“本地方法”服务的虚拟机栈，比如调用 **Open CV** 的方法就会涉及到它，其他的基本一模一样。



### 1.4 堆

Java 堆是线程共享的内存区域，我们创建的对象实例就存放在这里，同时它也是<b>垃圾收集器</b>管理的内存区域（后面的重点）。不过虽然说是线程共享，但现在其实已经可以从堆里面划分出多个线程私有的缓冲区了。

Java 堆可以是物理不连续的（想象无数大大小小的方块），但在逻辑上我们把它视为连续的（想象一个大方块）。

异常情况分析：

当Java堆剩余内存无法完成当前实例分配，且堆也无法继续扩展时，则会抛出`OutOfMemoryError`异常。
{:.error}



### 1.5 方法区

在 JDK 1.6 以前，它是完全保存在 Java 堆中的，用来保存被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等。

由于当时为了方便管理这一部分内存，是用“**永久代**”去实现方法区的，所以容易遇到内存溢出问题（永久代有 `-XX:MaxPermSize` 的上限），于是从 JDK 6 开始逐步放弃“**永久代**”这一概念，改成在本地内存中实现方法区。在 JDK 7 中，已经把字符串常量池、静态变量等移出，而到了JDK 8，则全部转移并改成了JRockit架构里的“<b>元空间</b>”形式。

异常情况分析：

如果方法区剩余内存无法完成内存分配时，将抛出 `OutOfMemoryError` 异常。
{:.error}


### 1.6 运行时常量池

运行时常量池本身是方法区的一部分，在类加载后，它会存放编译期生成的各种字面量和符号引用。字面量就是平常说的常量，而符号引用则是指全限定类名信息、字段名和描述符、方法名和描述符。

异常情况分析：

如果常量池剩余内存无法完成内存分配时，将抛出 `OutOfMemoryError` 异常。
{:.error}

### 1.7 直接内存

不属于虚拟机运行时数据区，如果Java内存区域超过了直接内存大小，将抛出 `OutOfMemoryError` 异常。



## 2.对象的创建、布局和访问

### 2.1 创建一个实例对象

当我们使用 new 关键字创建一个对象时，Java 虚拟机会经历五个步骤，按顺序依次是类加载检查、内存分配、内存初始化、设置对象头和构造函数执行。
{:.success}

（01）**类加载检查**：首先去运行时常量池中寻找是否有一个类的符号引用与之对应，然后检查该符号引用所代表的类是否已被加载、解析和初始化过。如果还没有，则要先执行相应的类加载过程；

（02）**内存分配**：为新对象分配内存有两种方式，一个是“<b>指针碰撞</b>”，由于已分配内存和未分配内存由一个指针隔开，因此分配新内存只需要移动该指针即可，对应带空间压缩整理能力的垃圾回收器（比如：*Serial 收集器*、*ParNew 收集器*）；另一个是“<b>空闲列表</b>”，从列表中找一块足够大的空间划分出去，并更新列表，对应 *CMS收集器* 这种基于清除算法的垃圾回收器。

在多线程环境下，内存分配通过两种方式来维护线程安全，一个是对“<b>内存分配</b>”这个动作进行同步处理，通过CAS以及失败重试的方法保证更新操作的原子性；另一个是为每个线程预分配一块内存，即<b>本地线程分配缓冲</b>（ TLAB ），可以通过设置参数 `-XX:+/-UseTLAB` 来设定是否使用 TLAB。

（03）**内存初始化**：将分配的内存空间，除内存头外，都初始化为零值。

（04）**设置对象头**：对象所属类、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息存放在对象头中。另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

（05）**构造函数执行**：就是字面意思，执行对象的构造方法。这一步的特殊点在于，虚拟机其实已经创建好了一个新对象了，但是从程序员角度来看，该对象才“刚刚开始构造”。



### 2.2 对象的内存布局

对象的内存布局可以分为三个部分，分别是**对象头**、**实例数据**和**对齐填充**。
{:.success}

（01）**对象头**又分为“**Mark Word**”和**类型指针**。前者用于存储自身运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。后者指向指向对象的类型元数据，用来判断该对象属于哪个类。

（02）**实例数据**部分是对象真正存储的有效信息。

（03）因为虚拟机要求对象的起始地址必须是8字节的整数倍，也即任何对象的大小都必须是8字节的整数倍，而对象头部分已经被设计为 8 字节的倍数（1 倍或者 2 倍），因此**对齐填充**部分就是用来补全对象实例数据部分的，相当于占位符。



### 2.3 如何访问对象

Java 程序可以通过栈上的 reference 数据来访问操作具体对象，主流访问方式有两种：

（01）**使用句柄访问**

Java堆中会划分一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据和类型数据各自具体的地址信息。

![句柄访问](/assets/images/jvm/image/visit-by-handler.png)

（02）**通过直接指针访问**

![句柄访问](/assets/images/jvm/image/visit-by-index.png)

这两种方式各有千秋，使用句柄的最大好处是 reference 存储的是稳定句柄地址，在对象移动时只会改变到对象实例数据的指针。



## 3.OutOfMemoryError异常

一个最简单的堆溢出异常示例如下：

```java
public class HeapOOM {
    public static void main(String[] args) {
        List<TestObject> list = new ArrayList<>();

        while(true){
            list.add(new TestObject());
        }
    }
}

class TestObject {

}
```

为了尽快得到结果，我们在虚拟机参数中将堆的最小值和最大值都设为20MB：

<img src="/assets/images/jvm/image/jvm-option.png" alt="VM Args" style="zoom: 80%;" />







# Part2.	类加载机制

Java 虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这个过程被称为虚拟机的类加载机制。
{:.info}

## 1.类加载过程

类加载分为五个阶段，分别是加载、验证、准备、解析和初始化。

![类加载流程](/assets/images/jvm/image/load-class-process.png)

其中，解析阶段由于Java的运行时绑定特性，可能会在初始化阶段之后再开始。

### 1.1 加载

类加载过程的第一步，主要完成下面 3 件事情：

（01）通过全类名获取定义此类的二进制字节流

（02）将字节流所代表的静态存储结构转换为方法区的运行时数据结构

（03）在内存中生成一个代表该类的 Class 对象，作为方法区这些数据的访问入口

虚拟机规范多上面这 3 点并不具体，因此是非常灵活的。比如："*通过全类名获取定义此类的二进制字节流*" 并没有指明具体从哪里获取、怎样获取。比如：比较常见的就是从 ZIP 包中读取（日后出现的 JAR、EAR、WAR 格式的基础）、其他文件生成（典型应用就是 JSP）等等。

一个非数组类的加载阶段（加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，这一步我们还可以自定义类加载器去控制字节流的获取方式（重写一个类加载器的 `loadClass()` 方法）。至于数组类型，它不通过类加载器创建，而是由 Java 虚拟机直接创建。加载阶段和连接阶段的部分内容是交叉进行的，加载阶段尚未结束，连接阶段可能就已经开始了。

### 1.2 验证

该阶段的目的主要是为了确保 Class 文件的字节流符合《Java 虚拟机规范》的全部要求，它包括文件格式验证、元数据验证、字节码验证和符号引用验证。



### 1.3 准备

对类变量（即以 static 修饰的静态变量）分配内存并设置其初始值，不过需要注意两点，一个是类变量是随 Class 对象一起存放在 Java 堆中的，另一个是这里的赋值其实是设置一个零值，真正的“赋值”需要等到类初始化阶段才执行。

```java
public static int value = 123;
```

比如针对上面这段代码，在“准备”阶段我们会将 value 设置为 0，而不是 123。

### 1.4 解析

解析阶段会将常量池中的符号引用替换为直接引用。

### 1.5 初始化

初始化阶段就是执行类构造器 `<clinit>()` 方法的过程。

我们需要注意：

（01）静态语句块只能访问到定义在静态语句块之前的变量

（02）`<clinit>()` 方法不需要调用父类构造器，因为 Java 虚拟机会保证子类的 `<clinit>()` 方法已经执行完毕

（03）只有当接口中定义的变量被使用时，接口才会执行 `<clinit>()` 方法

## 2.类加载器

### 2.1 类和类加载器

#### 2.1.1 判断类相等

想要判断两个类是否“相等”，除了比较类本身还要求它们有同一个类加载器。这里的相等，包括 `equals()` 方法、`isInstance()` 方法的返回结果，也包含使用 `instanceof` 关键字做对象所属关系判定等各种情况。

```java
public static void main(String[] args) throws Exception {
    ClassLoader my_loader = new ClassLoader() {
        @Override
        public Class<?> loadClass(String name) throws ClassNotFoundException {
            try {
                String file_name = name.substring(name.lastIndexOf(".") + 1) + ".class";
                InputStream is = getClass().getResourceAsStream(file_name);
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

    Object object = my_loader.loadClass("ClassLoaderTest").newInstance();
    System.out.println(object.getClass()); 			// class ClassLoaderTest
    System.out.println(object instanceof ClassLoaderTest); 	// false
}
```

#### 2.1.2 类加载器分类

![parents delegation model](/assets/images/jvm/image/parents-delegation-model.png)

（01）启动类加载器：负责加载存放在 `<JAVA_HOME>\lib` 目录，或者被 -Xbootclasspath 指定的路径中存放的类库，而且库名会被识别判断（如 `rt.jar`、`tools.jar`，名字不符合的不会被加载）；

（02）扩展类加载器：负责加载 `<JAVA_HOME>\lib\ext` 目录中，或者被 java.ext.dirs 系统变量所指定的路径中所有的类库；

（03）应用程序类加载器：程序中默认的类加载器；

（04）自定义类加载器：就是自己定义的类加载器：）

### 2.2 双亲委派模型

对于类加载器的区分，站在虚拟机角度来看只有两种：启动类加载器和其他类加载器。但是从开发人员角度来看，肯定不能划分地这么粗糙，通常我们将其划分后的架构称为为三层类加载器、双亲委派的类加载架构。

![parents delegation model](/assets/images/jvm/image/parents-delegation-model.png)

同样是这个图，里面展示的各种类加载器之间的层次关系被称为类加载器的“双亲委派模型”，该模型要求除顶层的启动类加载器外，其余类加载器都要自己的父亲加载器。虽然看着和 Object 的类继承关系很像，但是它们这种关系其实是基于组合关系实现的。

#### 2.2.1 工作流程

如果一个类加载器收到了类加载请求，它首先把这个请求委派给父亲加载器完成，以此类推。只有当父亲加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己完成。

这样做的好处是让 Java 中的类同它的加载器一起具备了一种带有优先级的层次关系，从而保证了同名类之间不会发生冲突。如果没有双亲委派模型，用户自己定义了一个 `Object` 类，系统将会出现混乱。

#### 2.2.2 破坏双亲委派模型

有重写 `loadClass()` 方法，针对 SPI 服务提供者接口的线程上下文类加载器是一种父类加载器请求子类加载器完成类加载的行为，以及 OSGI 对类加载器的使用三次，我们主要了解第一个。

由于双亲委派模型晚于类加载器概念出现，导致无法避免 `loadClass()` 方法被子类覆盖的可能性，因此 Java 只能在 `java.lang.ClassLoader` 中添加一个新的 `protected` 方法 `findClass()`，并引导用户尽量重写这个方法去编写类的加载逻辑。

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
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
                // 如果非空父类抛出 ClassNotFoundException 
                // 说明父类加载器搞不定，还得靠自己
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
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

代码的英文也很好理解，如果父类加载失败，就会调用自己的 `findClass()` 方法。而正是由于双亲委派的具体逻辑是写在 `loadClass()` 方法里的，我们用子类去覆盖这个方法就可以“轻松”破坏掉模型了。