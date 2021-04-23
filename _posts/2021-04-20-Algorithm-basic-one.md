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

**1.3.1 AcWing 789. 数的范围**

[题目链接](https://www.acwing.com/problem/content/791/)
使用二分算法时，尤其要注意边界问题，具体为考虑左右端点选择 `mid` 变量的取值，从而防止程序陷入死循环。取左端则命令 `mid = l + r >> 1`，取右端则命令 `mid = l + r >> 1`。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 100010;

int a[N];
int n, q;

int main()
{
    cin >> n >> q;
    for (int i = 0; i < n; i++) cin >> a[i];
    
    while (q--)
    {
        int x;
        cin >> x;
        
        int l = 0, r = n - 1;
        while (l < r)
        {
            int mid = l + r >> 1;           // 寻找左端点
            if (a[mid] >= x) r = mid;       // x [l, mid]
            else l = mid + 1;               // x [mid + 1, r]
        }
        if (a[l] != x) cout << "-1 -1" << endl; // l r 都一样
        else
        {
            cout << l << ' ';
            
            l = 0, r = n - 1;
            while (l < r)
            {
                int mid = l + r + 1 >> 1;   // 寻找右端点
                if (a[mid] <= x) l = mid;   // x [mid, r]
                else r = mid - 1;           // x [l, mid - 1]
            }
            
            cout << l << endl;  // l r 都一样
        } 
    }   
}
```

**1.3.2 AcWing 790. 数的三次方根**

[题目链接](https://www.acwing.com/problem/content/792/)
二分法的简单应用。
{:.conclude}

```c++
#include <iostream>

using namespace std;

int main()
{
    double x;
    cin >> x;
    
    double l = -1000, r = 1000;
    while(r - l > 1e-7)
    {
        double mid = (l + r) / 2;
        if(mid * mid * mid >= x) r = mid;   // x [l^3, mid^3]
        else l = mid;                       // x (mid^3, r]
    }
    
    printf("%.6f", l);
    
    return 0;
}
```



### 1.4 高精度

**1.4.1 AcWing 791. 高精度加法**

[题目链接](https://www.acwing.com/problem/content/793/)
使用高精度算法时，直接按低位存储。这样做虽然反人类直觉，但便于机器运算。其中唯一需要注意的是临界束前要判断借位 `t` 是否为零。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 100010;

vector<int> add(vector<int> a, vector<int> b)
{
    vector<int> c;
    
    int i = 0, j = 0, t = 0;
    while (i < a.size() && j < b.size())
    {
        t = a[i++] + b[j++] + t;
        c.push_back(t % 10);
        t = t / 10;
    }
    while (i < a.size())
    {
        t = a[i++] + t;
        c.push_back(t % 10);
        t = t / 10;
    }
    while (j < b.size())
    {
        t = b[j++] + t;
        c.push_back(t % 10);
        t = t / 10;
    }
    if (t) c.push_back(t);
    
    return c;
}

int main()
{
    string p, q;
    cin >> p >> q;
    
    vector<int> a, b, c;
    for (int i = p.size() - 1; i >= 0; i--) a.push_back(p[i] - '0');
    for (int j = q.size() - 1; j >= 0; j--) b.push_back(q[j] - '0');
    
    c = add(a, b);
    
    for (int k = c.size() - 1; k >= 0; k--) cout << c[k];
    cout << endl;
    
    return 0;
}
```

**1.4.2 AcWing 792. 高精度减法**

[题目链接](https://www.acwing.com/problem/content/794/)
思路同加法类似，注意删除高位的无效 `0` 。
{:.conclude}

```c++
#include <iostream>
#include <algorithm>
#include <cstring>

using namespace std;

const int N = 100010;

bool cmp(vector<int> a, vector<int> b)
{
    if (a.size() != b.size()) return a.size() > b.size();
    
    for (int i = a.size() - 1; i >= 0; i--)
        if (a[i] != b[i]) return a[i] > b[i];
    
    return true;
}

vector<int> sub(vector<int> a, vector<int> b)
{
    vector<int> c;
    
    for (int i = 0, t = 0; i < a.size(); i++)
    {
        t = a[i] - t;       // t = 1 or 0
        if (i < b.size()) t -= b[i];
        c.push_back((t + 10) % 10);
        t = t < 0 ? 1 : 0;
    }
    while (c.size() - 1 && !c.back()) c.pop_back();     // 删除高位 0
    
    return c;
}

int main()
{
    string p, q;
    cin >> p >> q;
    
    vector<int> a, b, c;
    for (int i = p.size() - 1; i >= 0; i--) a.push_back(p[i] - '0');
    for (int j = q.size() - 1; j >= 0; j--) b.push_back(q[j] - '0');
    
    if (cmp(a, b)) c = sub(a, b);       //  a >= b
    else c = sub(b, a), cout << '-';    //  b > a
    
    for (int k = c.size() - 1; k >= 0; k--) cout << c[k];
    cout << endl;
    
    return 0;
}
```

**1.4.3 AcWing 793. 高精度乘法**

[题目链接](https://www.acwing.com/problem/content/795/)
该乘法允许一个大数乘以一个较小的数，通过错位相加实现，同样需要注意删除高位的无效 `0` 。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

vector<int> muti(vector<int> a, int u)
{
    vector<int> b;
    
    for (int i = 0, t = 0; t || i < a.size(); i++)
    {
        if (i < a.size()) t = a[i] * u + t;
        b.push_back(t % 10);
        t = t / 10;
    }
    while (b.size() - 1 && !b.back()) b.pop_back();
    
    return b;
}

int main()
{
    string p;
    int u;
    cin >> p >> u;
    
    vector<int> a, b;
    for (int i = p.size() - 1; i >= 0; i--) a.push_back(p[i] - '0');
    
    b = muti(a, u);
    for (int i = b.size() - 1; i >= 0; i--) cout << b[i];
    cout << endl;
    
    return 0;
}
```

**1.4.4 AcWing 794. 高精度除法**

[题目链接](https://www.acwing.com/problem/content/796/)
从最高位开始除。
{:.conclude}

```c++
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

const int N = 1e9 + 10;

vector<int> div(vector<int> A, int b, int& r)
{
    vector<int> C;
    
    for (int i = A.size() - 1; i >= 0; i--) 
    {
        r = r * 10 + A[i];
        C.push_back(r / b);
        r = r % b;
    }
    reverse(C.begin(), C.end());
    while (C.size() > 1 && C.back() == 0) C.pop_back();
    
    return C;
}

int main()
{
    string a;
    int b;
    vector<int> A, C;
    
    cin >> a >> b;
    for (int i = a.size() - 1; i >= 0; i--) A.push_back(a[i] - '0');
    
    int r = 0;
    C = div(A, b, r);
    for (int i = C.size() - 1; i >= 0; i--) cout << C[i];
    cout << endl << r << endl;
    
    return 0;
}
```



### 1.5 前缀和

前缀和常用于辅助很多算法的化简步骤。
{:.success}

**1.5.1 AcWing 795. 前缀和**

[题目链接](https://www.acwing.com/problem/content/797/)
前缀和基础题。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 1e5 + 10;
int d[N], n, m;

int main()
{
    cin >> n >> m;
    // d[0] = 0;
    int ans = 0;
    for (int i = 1; i <= n; i++) 
    {
        int num;
        cin >> num;
        d[i] = d[i - 1] + num;
    }
    
    for (int i = 1; i <= m; i++)
    {
        int a, b;
        cin >> a >> b;
        cout << d[b] - d[a - 1] << endl;
    }
    
    return 0;
}
```

**1.5.2 AcWing 796. 子矩阵的和**

[题目链接](https://www.acwing.com/problem/content/798/)
前缀和基础题。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 1010;

int g[N][N], area[N][N];
int n, m, q;

int main()
{
    cin >> n >> m >> q;
    for (int i = 1; i <= n; i++)
    {
        int d = 0;
        for (int j = 1; j <= m; j++)
        {
            int a;
            scanf("%d", &a);
            d += a;
            g[i][j] = g[i - 1][j] + d;
        }
    }
    
    for (int i = 1; i <= q; i++)
    {
        int x1, y1, x2, y2;
        scanf("%d%d%d%d", &x1, &y1, &x2, &y2);
        int s1 = g[x1 - 1][y1 - 1], s2 = g[x2][y1 - 1], s3 = g[x1 - 1][y2], s4 = g[x2][y2]; // 注意将点转换成格子
        int s = s4 - s2 - s3 + s1;
        cout << s << endl;
    }
    
    return 0;
    
}
```



### 1.6 差分

**1.6.1 AcWing 797. 差分**

[题目链接](https://www.acwing.com/problem/content/799/)
核心思路是将序列区间分解为两个后缀之差。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 100010;

int n, m;
int a[N], d[N];

int main()
{
    cin >> n >> m;
    for (int i = 1; i <= n; i++) cin >> a[i];
    for (int i = 1; i <= m; i++)
    {
        int l, r, c;
        cin >> l >> r >> c;
        
        d[l] += c;
        d[r + 1] -= c;
    }
    
    int ex = 0;
    for (int i = 1; i <= n; i++)
    {
        ex += d[i];
        cout << a[i] + ex << ' ';
    }
    cout << endl;
    
    return 0;
}
```

**1.6.2 AcWing 798. 差分矩阵**

[题目链接](https://www.acwing.com/problem/content/800/)
基础题。
{:.conclude}

```c++
#include <iostream>
#include <cstring>
#include <algorithm>

using namespace std;

const int N = 1010;
int g[N][N], d[N][N], n, m, q;

int main()
{
    cin >> n >> m >> q;
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            scanf("%d", &g[i][j]);
    
    for (int i = 1; i <= q; i++)
    {
        int x1, y1, x2, y2, c;
        scanf("%d%d%d%d%d", &x1, &y1, &x2, &y2, &c);
        // 按右下角对齐分布 s = s1 + s4 - s2 - s3;
        d[x1][y1] += c;
        d[x2 + 1][y2 + 1] += c;
        d[x1][y2 + 1] -= c;
        d[x2 + 1][y1] -= c;
    }
    for (int i = 1; i <= n; i++)
    {
        int dist = 0;
        for (int j = 1; j <= m; j++)
        {
            dist += d[i][j];
            d[i][j] = dist + d[i - 1][j];
            g[i][j] += d[i][j];
            printf("%d ", g[i][j]);
        }
        printf("\n");
    }
    
    return 0;
}
```



### 1.7 双指针

**1.7.1 AcWing 799. 最长连续不重复子序列**

[题目链接](https://www.acwing.com/problem/content/801/)
基础题。
{:.conclude}

```c++
#include <iostream>
#include <algorithm>
#include <cstring>

using namespace std;

const int N = 100010;

int a[N], st[N];
int n;

int main()
{
    cin >> n;
    for (int i = 1; i <= n; i++) cin >> a[i];
    
    int ans = 0;
    for (int i = 1, j = 1; i <= n; i++)
    {
        st[a[i]]++;
        while (st[a[i]] > 1) st[a[j++]]--;
        ans = max(ans, i - j + 1);
    }
    
    cout << ans << endl;
    
    return 0;
}
```

**1.7.1 AcWing 800. 数组元素的目标和**

[题目链接](https://www.acwing.com/problem/content/802/)
基础题。
{:.conclude}

```c++
#include <iostream>

using namespace std;

const int N = 1e5 + 10;

int a[N], b[N];
int n, m, x;

int main()
{
    cin >> n >> m >> x;
    for (int i = 0; i < n; i++) scanf("%d", &a[i]);
    for (int i = 0; i < m; i++) scanf("%d", &b[i]);
    
    int l = 0, r = m - 1;
    while (l < n)
    {
        while (r > 0 && a[l] + b[r] > x) r--;
        if (a[l] + b[r] == x) break;
        l++;
    }
    
    cout << l << ' ' << r << endl;
    
    return 0;
}
```



### 1.8 位运算



### 1.9 离散化



### 1.10 区间合并



## 2 数据结构

### 2.1 单链表



### 2.2 归并排序的改进

