---
title: Java集合之ArrayList
tags: Java 集合
show_author_profile: true
---

ArrayList和HashMap是集合学习的两座大山，由于其庞杂的体量，很容易让我们初学者在源码中迷失方向。个人认为，学习集合，应该从最基本的数据结构（包含集合的创建与扩容）、常用的增删查改方法、多线程环境的线程安全性三个方面来学习。虽然堪堪入门，但管中窥豹，可见一斑，以后慢慢深入也不迟。
{:.success}

## 1.数据结构

### 1.1 底层结构

可以看到，`ArrayList`底层就是一个名为`elementData`的动态数组，暂时还没初始化。

```java
transient Object[] elementData; // non-private to simplify nested class access
```

其中，使用关键字`transient`修饰的目的是让集合在序列化时只存储实际大小。在这其中的玄机在writeObject和readObject两个方法中，`ArrayList`在序列化的时候会调用writeObject方法，直接将`size`和`element`写入`ObjectOutputStream`；反序列化时调用readObject，从`ObjectInputStream`获取`size`和`element`，再恢复到`elementData`。



### 1.2 ArrayList的创建

`ArrayList`有三种创建方式，对应三个构造器

（01）默认的空参构造器创建方式：

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

`elementData`被初始化为一个空数组。

（02）带容量（**int**型）参数构造器创建方式：

```java
private static final Object[] EMPTY_ELEMENTDATA = {};

public ArrayList(int initialCapacity) {
	if (initialCapacity > 0) {
	this.elementData = new Object[initialCapacity];
	} else if (initialCapacity == 0) {
		this.elementData = EMPTY_ELEMENTDATA;
	} else {
		throw new IllegalArgumentException("Illegal Capacity: "+
			initialCapacity);
	}
}
```

当参数为0，`elementData`被初始化为一个空数组，当参数大于0，则用`new`方式创建指定大小的数组。

（03）带**Collection**接口实现类参数构造器创建方式：

```java
private static final Object[] EMPTY_ELEMENTDATA = {};

public ArrayList(Collection<? extends E> c) {
	elementData = c.toArray();
	if ((size = elementData.length) != 0) {
		// c.toArray 没有正确转换为 Object[] (see 6260652)
		if (elementData.getClass() != Object[].class)
			elementData = Arrays.copyOf(elementData, size, Object[].class);
	} else {
		// 用空数组替换.
		this.elementData = EMPTY_ELEMENTDATA;
	}
}
```

本意是将传入集合转化为一个`ArrayList`对象，具体操作依然是数组之间的值传递，先将集合通过`toArray`方法转换为数组，然后赋给`elementData`对象。在元素个数不等于零的情况，还会检测`c.toArray`是否有正确转换成`Object`数组。



### 1.3 ArrayList的扩容

当容量不足不足时，涉及到添加元素的操作，ArrayList就会触发扩容机制，具体来讲就是调用`add(E)`和`add(int, E)`两个方法时。观察两个方法，发现如果要向指定位置插入元素，需要检测下标是否越界，以及对下标之后的元素进行后移，而方法的核心还是`ensureCapacityInternal`方法（意味确保内部容量）。

```java
public void add(int index, E element) {
	rangeCheckForAdd(index);

	ensureCapacityInternal(size + 1);  // Increments modCount!!
	System.arraycopy(elementData, index, elementData, index + 1,
						size - index);
    elementData[index] = element;
	size++;
}

public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

进入`ensureCapacityInternal`，如果集合为空（`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`），会对传入的参数（**该参数=原容量+1**）同`DEFAULT_CAPACITY`（**默认大小参数=10**）进行比较，返回较大的参数。

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 判断集合是否为空
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

之后继续进入`ensureExplicitCapacity`方法，modCount是用来实现fail-fast机制的，先不管它。

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

继续进入grow方法，里面看着很乱，但本质就是**新容量等于原来容量的1.5倍**，其他的判断语句是为了限制新容量过大导致越界错误。

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```



## 2.ArrayList的常用方法

### 2.1 ArrayList的变动操作

主要是`ArrayList`的增添、删除、修改等涉及底层元素变化的方法，了解如何使用即可，下面随便列出一些最常用到的操作。

```java
public E set(int index, E element);
public boolean add(E e);
public void add(int index, E element);
public E remove(int index);
public boolean remove(Object o);
public void clear();
```



### 2.2 ArrayList的查询操作

`ArrayList`的一大优势在于它实现了`RandomAccess`接口，表面它是可以随机访问的，也即通过下标位置寻找到指定元素只需要常量时间`O(1)`。



### 2.3 ArrayList的遍历

ArrayList遍历方式有三种：

（01）**迭代器遍历**，就是通过iterator去遍历，通过ArrayList对象的iterator方法获取Iterator对象，然后在循环中用迭代器对象的hasNext方法判断，以及用next方法获取；

```java
List list = new ArrayList();
Iterator i = list.iterator();
while(i.hasNext()) {
    String s = (String)i.next();
}
```

（02）**for-each遍历**，本质上仍然是迭代器遍历；

（03）**通过索引遍历**，由于ArrayList实现了`RandomAccess`接口，可以通过索引值随机访问。



## 3.线程安全性

`ArrayList`**是线程不安全的类**，如果想要得到线程安全版的`ArrayList`，一般来说可以使用`Vector`，它相当于`ArrayList`的线程安全版；也可以用`Collections.synchronizedList`方法把一个普通`ArrayList`包装成一个线程安全版本的数组容器，两种方法本质都是给所有方法用`Synchronized`加上同步锁。

不过我更推荐使用`CopyOnWriteArrayList`，看名字这么复杂就很厉害了。当然，其实主要原因在于`Vector`哪怕已经用`Synchronized`对自己全副武装了，也无法保证绝对的线程安全。考虑以下场景：线程1对集合进行最末元素的删除动作，而线程2对集合进行最末元素访问动作，如果线程1刚好将要访问的元素删除了，线程2将会抛出数组越界错误。而在`CopyOnWriteArrayList`中，由于`volatile`的声明，我们就不必担心此类问题了。



## 4.其他

### 4.1 System.arraycopy() 和 Arrays.copyOf()方法

在ArrayList中我们会看到这两个方法被频繁地使用，它们都是用实现数组复制的。

首先看`Arrays.copyOf()`，我们很容易发现其内部调用了`System.arraycopy()`方法。

```java
public static <T,U> T[] copyOf(U[] original, int newLength, 
                               Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
```

而`System.arraycopy()` 是一个**本地方法**，表示从源数组`src`的`srcPos`处开始复制一段`length`长度的数组，然后赋值到`dest`数组的`destPos`处。

```java
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);	

//使用示范
int[] fun ={0,1,2,3,4,5,6}; 
System.arraycopy(fun,0,fun,3,3);//结果为：{0,1,2,0,1,2,6};
```



### 4.2 fail-fast事件

**fail-fast**是一种错误检测机制，如果发生**fail-fast**事件（它代表多个线程对同一个集合内容进行了操作），则抛出`ConcurrentModificationException`异常。

**fail-fast**是通过**checkForComodification**方法检测`expectedModCount`和`modCount`是否相等从而判断在`Iterator`操作期间是否有其他线程调用了`add`、`remove`、`clear`等方法的。

如果想要避免发生**fail-fast**事件，一般用`java.util.concurrent`包中的类去替换即可。这些类之所以不会发生**fail-fast**的直接原因是它们没有使用**checkForComodification**方法，不检测自然不会抛出；而之所以这么做，是因为它们通过`volatile`关键字有效保证了线程安全，就不再需要这套检测机制了；

### 4.3 toArray方法

使用**toArray**方法时，常常会碰到类型转化异常，这是因为`ArrayList`提供了两个**toArray**方法，一个是不带参数的，一个是带泛型参数的方法。

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

由于`ArrayList`中的`elementData`是一个`Object`数组，因此要想返回特定类型的数组，则只有调用带参方法，且该类型数组参数的长度应该小于`elementData`的长度。

```java
Integer[] newText = (Integer[])list.toArray(new Integer[0]);
String[] newString = (String[])list.toArray(new String[0]);
```

