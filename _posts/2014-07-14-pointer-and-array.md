---
layout: post
title: 数组与指针初步
date: 2014-07-14
---

1.指针作为参数

调用函数时传递指向变量的指针，可以使得函数能够改变变量的值。

```c
#include <stdio.h>

void ChangeVar(int *n)
{
    *n += 1;
}

int main()
{
    int num = 10;
    ChangeVar(&num);
    printf("%d\n", num);
    return 0;
}
```

下面的程序传递了一个数组及两个指针给被调函数，被调函数将最大值和最小值写入指针指向的内存地址。

```c
#include <stdio.h>
#define N 10

void max_min(int a[], int n, int *max, int *min);

main()
{
    int b[N], i, big, small;

    printf("Enter %d numbers: ", N);
    for (i = 0; i < N; i++)
    {
        scanf("%d", &b[i]);
    }

    max_min(b, N, &big, &small);

    printf("Largest: %d\n", big);
    printf("Smallest: %d\n", small);

    return 0;
}

void max_min(int a[], int n, int *max, int *min)
{
    int i;

    *max = *min = a[0];
    for (i = 1; i < n; i++)
    {
        if (a[i] > *max)
            *max = a[i];
        else if (a[i] < *min)
            *min = a[i];
    }
}
```

2.指针作为返回值

```c
#include <stdio.h>

int *max(int *a, int *b)
{
    if (*a > *b)
        return a;
    else
        return b;
}

int main()
{
    int *p, x, y;
    x = 50;
    y = 30;
    p = max(&x, &y);
    printf("%d\n", *p);
    return 0;
}
```

3.指针用于数组处理

```c
#include <stdio.h>
#define N 10

int main()
{
    int a[N], sum, *p;
    sum = 0;
    for (p = &a[0]; p < &a[N]; p++)
        sum += *p;
        
}
```

```c
for (p = &a[0]; p < &a[N]; p++)
    sum += *p;

//可以写成

p = &a[0];
while (p < &a[N])
    sum += *p++;
```

用数组名作为指针：可以用数组名作为指向数组第一个元素的指针。

```c
int a[10]

*a = 7;        //给第一个元素赋值
*(a + 1) = 12; //给第二个元素赋值
```

```c
for (p = &a[0]; p < &a[N]; p++)
    sum += *p;

//可以写成：

for (p = a; p < a + N; p++)
    sum += *p;
```

虽然可以把数组名用作指针，但是不能给数组名赋新的值。

```c
while (*a != 0)
    a++;   //这是错的

//可以将 a 复制到一个指针变量，然后修改它
p = a;
while (*p != 0)
    P++;
```

反向显示数列

```c
#include <stdio.h>

#define N 10

int main()
{
    int a[N], *p;

    printf("Enter %d numbers: ", N);
    for (p = a; p < a + N; p++)
        scanf("%d", p);

    printf("In reverse order:");
    for (p = a + N - 1; p >= a; p--)
        printf(" %d", *p);
    printf("\n");

    return 0;
}
```

用数组作为函数调用的参数。

有这样一个函数，接受一个数组和作为参数，因为在函数内部无法获取数组的长度，所以要一并把数组的长度传递进来（int n）。

```c
int find_largest(int a[], int n)
{
        int i, max;

        max = a[0];
        for (i = 1; i < n; i++)
                if (a[i] > max)
                        max = a[i];
        return max;
}
```

调用方法：

```c
largest = find_largest(b, N);
```

其中 b 就是数组，事实上它是一个指针，指向数组 b 的第一个元素。传递一个普通变量与传递一个数组给函数的区别是，普通变量会被复制给函数内部的本地变量，对本地变量的改变不会影响到原来的变量。而传递数组不能防止改变，因为事实上传递的是指针。

如果想要指针，可以把 find_largest 函数声明成这样

```c
int find_largest(int *a, int n)
{
        ...
}
```

声明 a 是指针就相当于声明它是数组。编译器处理这两类声明就好像它们完全一样。这里隐含了下面的规则：指针名可以作为数组名使用。所以，上例的两种声明方法完全是等价的。


用指针名作为数组名

```c
int find_largest(int a[], int n)
{
        int i, max;

        max = a[0];
        for (i = 1; i < n; i++)
                if (a[i] > max)
                        max = a[i];
        return max;
}
```

C 处理数组时本质上是处理指针，a[1] 等价于 *(a + 1)，同样等价于 *(1 + a)。所以，a[1] 其实可以写成 1[a]，不过这种写法太令人迷惑了。