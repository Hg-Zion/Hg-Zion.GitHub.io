---
title: 算法基础（一）
tags: 算法 初阶 c++
show_author_profile: false
---

## 1 基础算法

### 1.1 快速排序

**1.1.1 AcWing 785. 快速排序**

[题目链接](https://www.acwing.com/problem/content/787/)
核心是**分治**，每次以 `j` 为分界点。难点在于初始化的“后撤步”，即 `i = l - 1` 和 `j = r + 1`，以及在交换元素时的前置判定容易遗漏。
{:.conclude}

```c++
#include <iostream>

using namespace std;

const int N = 1e6 + 10;

int q[N];
int n;

void quick_sort(int q[], int l, int r)
{
    if (l >= r) return;
    
    int i = l - 1, j = r + 1, x = q[l + r >> 1];    // 核心
    while (i < j)
    {
        do i++; while (q[i] < x);
        do j--; while (q[j] > x);
        if (i < j) swap(q[i], q[j]);    // 易遗漏判定
    }
    
    quick_sort(q, l, j), quick_sort(q, j + 1, r);
}

int main() {
	cin >> n;
	for (int i = 0; i < n; i++) cin >> q[i];

	quick_sort(q, 0, n - 1);

	for (int i = 0; i < n; i++) cout << q[i] << ' ';
    
    return 0;
}
```

**1.1.2 AcWing 785. 快速排序**

[题目链接](https://www.acwing.com/problem/content/788/)
快速排序的简单使用方法，由于数据范围是 `1e5` ，显然我们应该使用时间复杂度上界为 `O(NlogN)` 。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 100010;

int n, m;
int q[N];

void quick_sort(int q[], int l, int r)
{
    if (l >= r) return;
    
    int i = l - 1, j = r + 1, x = q[l + r >> 1];
    while (i < j)
    {
        do i++; while(q[i] < x);
        do j--; while(q[j] > x);
        if (i < j) swap(q[i], q[j]);
    }
    
    quick_sort(q, l, j);
    quick_sort(q, j + 1, r);
}

int main()
{
    cin >> n >> m;
    for (int i = 0; i < n; i++) cin >> q[i];
    
    quick_sort(q, 0, n - 1);
    
    cout << q[m - 1] << endl;
    
    return 0;
}
```



### 1.2 归并排序

**1.2.1 AcWing 787. 归并排序**

[题目链接](https://www.acwing.com/problem/content/789/)
归并排序依然使用分治思想，先将数组分解，待子数组排序完毕后对其合并。核心是用**辅助数组**更新以及按 `mid` 分割。由数据范围 `1e5` ，知时间复杂度为 `O(NlogN)` 。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;
 
const int N = 100010;

int n;
int a[N], p[N];   // 辅助数组

void merge_sort(int q[], int l, int r)
{
    if (l >= r) return;
    
    int mid = l + r >> 1;
    merge_sort(q, l, mid);
    merge_sort(q, mid + 1, r);
    
    int i = l, j = mid + 1, k = 0;      // k 新数组下标
    while (i <= mid && j <= r)
        if (q[i] <= q[j]) p[k++] = q[i++];
        else p[k++] = q[j++];
    while (i <= mid) p[k++] = q[i++];
    while (j <= r) p[k++] = q[j++];
    
    for (i = l, k = 0; i <= r; i++, k++) q[i] = p[k];
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; i++) cin >> a[i];
    
    merge_sort(a, 0, n - 1);
    
    for (int i = 0; i < n; i++) cout << a[i] << ' ';
    
    return 0;
}
```

**1.2.2 AcWing 788. 逆序对的数量**

[题目链接](https://www.acwing.com/problem/content/790/)
只需要在归并子数组，记录右子数组中先归并元素即可。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

typedef long long LL;
const int N = 100010;

int n;
int a[N], p[N];

LL merge_sort(int q[], int l, int r)
{
    if (l >= r) return 0;
    
    int mid = l + r >> 1;
    LL ans = merge_sort(q, l, mid) + merge_sort(q, mid + 1, r);
    
    int i = l, j = mid + 1, k = 0;
    while (i <= mid && j <= r)
        if (q[i] <= q[j]) p[k++] = q[i++];
        else
        {
            p[k++] = q[j++];
            ans += mid - i + 1;     // 跨越长度
        }
    while (i <= mid) p[k++] = q[i++];
    while (j <= r) p[k++] = q[j++];
    
    for (i = l, k = 0; i <= r; i++, k++) q[i] = p[k];
    
    return ans;
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; i++) cin >> a[i];
    
    cout << merge_sort(a, 0, n - 1);
    
    return 0;
}
```



### 1.3 二分



### 1.4 高精度



### 1.5 前缀和



### 1.6 差分



### 1.7 双指针



### 1.8 位运算



### 1.9 离散化



### 1.10 区间合并



## 2 数据结构

### 2.1 单链表



### 2.2 归并排序的改进

## 3 搜索和图论



