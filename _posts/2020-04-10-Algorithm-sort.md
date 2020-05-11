---
title: 算法办没——排序算法
tags: sort 算法 java
show_author_profile: false
---

## 1 初级排序算法

### 1.1 选择排序

首先，找到数组中最小的元素，然后将它同数组的第一个元素交换位置，再然后找到剩余元素中最小的元素同第二个元素交换位置，循环往复直至整个数组有序。
{:.conclude}

```java
public class Selection {
    public static void sort(Comparable[] a) {
        int N = a.length;
        for (int i = 0; i < N; i++) {
            int min = i;
            for (int j = i + 1; j < N; j++){
                if (less(a[j], a[min])) min = j;
            }
            exch(a, i, min);
        }
    }
    
    private static boolean less(Comparable v, Comparable w){
        return v.compareTo(w) < 0;
    }

    private static void exch(Comparable[] a, int i, int j) {
        Comparable t = a[i];
        a[i]         = a[j];
        a[j]         = t;
    }
}
```

代码中的 `less` 和 `exch` 方法都是书上的模板方法，目的是提高代码可读性以及支持 `Comparable` 变量的泛用。
{:.success}

对于长度为 **N** 的数组，选择排序需要大约 ***N^2 / 2*** 次比较和 **N** 次交换。比较次数是由 **1** 到 **N-1** 的累加得来的，本质是每次都将剩余的无序数组进行比较；至于交换次数则可以从代码中可以明显看出。

![selection-sort](\assets\images\sort\selection.png){:.shadow .rounded}

### 1.2 插入排序

插入排序就像我们斗地主理牌一样，将每一张牌插入到其他已经有序的牌中的适当位置。
{:.conclude}

对于随机排序数组，虽然插入排序和选择排序的运行时间同样是平方级别，但二者常系数不同，实际表现为插入排序更快。究其原因，插入排序的插入元素无须同全部有序数比较而选择排序的选择元素需要同全部无序数比较。

```java
public static void sort(Comparable[] a) {
    int N = a.length;
    for (int i = 1; i < N; i++) {
        for (int j = i; j > 0 && less(a[j], a[j - 1]); j--)
            exch(a, j, j - 1);
    }
}
```

对于随机排列的长度为 N 且主键不重复的数组，平均情况下插入排序需要 ***~ N^2/4*** 次比较以及 ***~ N^2/4*** 次交换。在最坏情况下（降序排列）需要 ***~ N^2 / 2*** 次比较以及 ***~ N^2 / 2*** 次交换，而在最好情况（升序排列）则需要 ***N - 1*** 次比较以及 ***0*** 次交换。通过下图，可以很明显论证。

![selection-sort](\assets\images\sort\insertion.png){:.shadow .rounded}

### 1.3 希尔排序

希尔排序基于插入排序，核心是让任意间隔为 h 的元素局部有序，当 h 为 ***1*** 时整个数组有序。
{:.conclude}

希尔排序是针对插入排序只能比较相邻元素这一缺点进行改进的，并基于“插入排序对于基本有序的数组运算很快”这一特点，让 h 由大到小递减。

```java
public static void sort(Comparable[] a) {
    int N = a.length;
    int h = 1;
    while (h < N / 3) h = h * 3 + 1;
    while (h >= 1) {
        for (int i = h; i < N; i++) {
            for (int j = i; j >= h && less(a[j], a[j - h]); j -= h)
                exch(a, j, j - h);
        }
        h = h / 3;
    }

}
```

使用递增序列 1，4，13，40··· 的希尔排序所需的比较次数为 `O(N^(3/2))` ，因此我们在排序伊始会对 h 进行选择赋值，而具体到每个 h - 数组 则将原本插入排序的差值从 **1** 改为 **h**。

![selection-sort](\assets\images\sort\h-sorted.png){:.shadow .rounded}

### 1.4 洗牌算法

让当前“扑克牌”同包括它自己在内的前面所有牌中的随机一张进行交换，这个算法是线性的，由 Fisher Yetes 在 1938 年提出。

```java
public static void shuffle(Comparable[] a) {
    Random random = new Random();
    int N = a.length;

    for (int i = 1; i < N; i++) {
        int j = random.nextInt(i + 1);
        exch(a, i, j);
    }
}
```

最基础的算法是为每张“扑克牌”生成一个随机数，然后对随机数进行排序，相当于按权值排列，不过由于要重新排序开销较大。

### 1.5 凸包问题

目前我只能说是按照葛立恒扫描算法（Graham Scan）对这类问题进行解决。



## 2 归并排序

### 2.1 自顶向下的归并排序

归并排序最大的亮点是能够对任意长度的数组进行所需时间为 ***NlogN*** 的排序，其思想基于分治法，由冯诺依曼为他的 *EDVAC* 提出的排序算法。
{:.conclude}

```java
public class Merge {
    private static Comparable[] aux;

    public static void sort(Comparable[] a) {
        aux = new Comparable[a.length];
        sort(a, 0, a.length - 1);
    }

    private static void sort(Comparable[] a, int lo, int hi) {
        if (lo >= hi) return;
        int mid = (lo + hi) / 2;
        sort(a, lo, mid);
        sort(a, mid + 1, hi);
        merge(a, lo, mid, hi);
    }

    private static void merge(Comparable[] a, int lo, int mid, int hi) {
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++) aux[k] = a[k];
        
        for (int k = lo; k <= hi; k++)
            if (i > mid)                   a[k] = aux[j++];
            else if (j > hi)               a[k] = aux[i++];
            else if (less(aux[j], aux[i])) a[k] = aux[j++];
            else                           a[k] = aux[i++];
    }
}
```

其中，`merge` 方法先将所有元素复制到 `aux[]` 中，然后再归并回 `a[]` 中。方法在归并时会作四个判断：左半边用尽（取右半边元素）、右半边用尽（取左半边元素）、右半边的当前元素小于左半边的当前元素（取右半边元素）以及左半边的当前元素小于右半边的当前元素（取左半边元素），这个过程辅助图像会更好理解。

![selection-sort](\assets\images\sort\merge.png){:.shadow .rounded}



### 2.2 归并排序的改进

**2.2.1 对小规模子数组使用插入排序**

尽管归并排序所需要的时间是 ***NlgN***，但想要表现得更快需要 ***N*** 很大才行，如果 ***N*** 是一个较小的数（比如 **15**）那么我们可以使用插入排序去处理它。

**2.2.2 测试数组是否有序**

当两个子数组归并时，如果发现 `a[mid]<=a[mid+1]` 那么我们认为这个子数组已经有序并跳过归并方法。



## 3 快速排序

### 3.1 基本算法

正如归并算法的核心在于归并，即将两个有序数组合并成一个有序数组，那么快速排序的核心则是切分 `partition`，即将选好的切分点两边的元素进行排序。
{:.conclude}

快速排序是时间复杂度为 ***NlgN*** 的排序算法，更重要的是它是原地排序（只需要一个很小的辅助栈）。它的缺点是比较脆弱，有很多错误会导致它在实际中的性能有平方级别。

```java
public class Quick {
    public static void sort(Comparable[] a) {
        sort(a, 0, a.length - 1);
    }

    private static void sort(Comparable[] a, int lo, int hi) {
        if (lo >= hi) return;
        int j = partition(a, lo, hi);
        sort(a, lo, j - 1);
        sort(a, j + 1, hi);

    }
    // ...
}
```

快速排序先用 `partition()` 方法将 `a[j]` 元素放到一个合适的位置，然后再递归调用其他位置的元素排序。

![selection-sort](\assets\images\sort\quicksort-overview.png){:.shadow .rounded}

到了 `partition()` 方法内部，实际上是对逆序对不断交换位置的过程，等循环结束后再将“中间”位置元素同首位元素进行交换。

```java
private static int partition(Comparable[] a, int lo, int hi) {
    int i = lo, j = hi + 1;
    Comparable v = a[lo]; // 选中第一个元素
    while (true) {
        while (less(a[++i], v)) if (i == hi) break;
        while (less(v, a[--j])) if (j == lo) break;
        if (i >= j) break;
        exch(a, i, j);
    }
    exch(a, lo, j); // 选 i|j 都一样
    return j;
}
```

![selection-sort](\assets\images\sort\partitioning-overview.png){:.shadow .rounded}

![selection-sort](\assets\images\sort\partitioning.png){:.shadow .rounded}

### 3.2 细节

**3.2.1 别越界**

如果切分元素是数组中最大或最小的元素，扫描指针将一路向前，这时候我们就要小心别让指针越界了。当然，总结成一句话就是**背板**即可。

**3.2.2 终止循环**

终止循环条件要考虑到数组中可能有同切分元素值相同的其他元素，依然是背板即可。比如左侧扫描最好是遇到大于等于切分元素时停下，右侧扫描则是遇到小于等于切分元素时停下，这样尽管会导致算法不稳定，但会避免平方级别时间的情况发生。

### 3.3 算法改进

**3.3.1 小数组用插入排序**

同归并排序类似地，如果对小数组的排序用插入排序替换，会快不少。当然修改方式也很简单，将递归方法 sort 的第一条语句从 `if (lo >= hi) return;` 改成 `if (lo + M >= hi) { Insertiong.sort(a, lo, hi); return; }` 即可。

**3.3.2 三向切分**

三向切分是针对有大部分重复元素的情况的，此时快速排序的递归性会使元素全部重复的子数组经常出现，性能较低。
{:.conclude}

```java
private static void sort(Comparable[] a, int lo, int hi) {
    if (lo >= hi) return;
    int lt = lo, i = lo + 1, gt = hi;
    Comparable v = a[lo];
    while (i <= gt) {
        int cmp = a[i].compareTo(v);
        if (cmp < 0) exch(a, lt++, i++);
        else if (cmp > 0) exch(a, i, gt--);
        else i++;
    }
    sort(a, lo, lt - 1);
    sort(a, gt + 1, hi);
}
```

三向切分主要是将数组分成了三部分，分别对应小于、等于和大于切分元素的数组元素，通过让同切分元素相等的元素归位从而避免其被包含在递归调用的数组中。对于包含大量重复元素的数组，它将排序时间从线性对数级别降低到了线性级别。