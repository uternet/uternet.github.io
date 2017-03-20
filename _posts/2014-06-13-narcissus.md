---
layout: post
title: 打印指定位数的水仙花数
date: 2014-06-13
---

先来C语言版：

```c
#include <stdio.h>

int power(int m, int n)
{
    int i, r;
    r = 1;
    for (i = 1; i <= n; i++)
        r *= m;
    return r;
}

int main(void)
{
    int i, j, n, min, max, sum;
    scanf("%d", &n);

    min = power(10, (n - 1));
    max = min * 10 -1;
  
    for (i = min; i <= max; i++)
    {
        sum = 0;
        j = i;
        
        do
        {
            sum += power(j % 10, n);
            j /= 10;
        } while (j > 0);
        
        if (sum == i)
            printf("%d\n", i);
    }
    return 0;
}
```


C语言的写出来了，就想用 Scheme 写一遍，转成递归颇费了些脑力，出于性能考虑，最后又全都改成了尾递归。

```scheme
(define (bits n)
  (define (iter n i)
    (cond
     ((= (quotient n 10) 0) i)
     (else (iter (quotient n 10) (add1 i)))))
  (iter n 1))

(define (pow-sum n p)
  (define (iter n p sum)
    (cond
     ((= n 0) sum)
     (else (iter (quotient n 10) p
                 (+ sum (expt (modulo n 10) p))))))
  (iter n p 0))
                         
(define (narcissus? n)
  (= n (pow-sum n (bits n))))

(define (list-narcissus min max)
  (define (iter min max l)
    (cond
     ((= min max) l)
     ((narcissus? min)
      (iter (add1 min) max (cons min l)))
    (else (iter (add1 min) max l))))
  (iter min max '()))

(define (start n)
  (list-narcissus (expt 10 (- n 1)) (- (expt 10 n) 1)))

(define result (start (read)))

(define (print-list l)
  (cond
   ((null? l) #t)
   (else
    (begin
      (display (car l)) (newline)
      (print-list (cdr l))))))

(print-list result)
```

既然用 Scheme 写出来了，又想用递归的办法再用C写一次：

```c
#include <stdio.h>
#include <math.h>

int bits(int n)
{
    if (n == 0)
        return 0;
    else
        return bits(n / 10) + 1;
}

double pow_sum(int n, int p)
{
    if (n == 0)
        return 0;
    else
        return pow(n % 10, p) + pow_sum(n / 10, p);
}

int is_nar(int n)
{
    if (n == (int)pow_sum(n, bits(n)))
        return 1;
    else
        return 0;
}

int list_nar(int min, int max)
{
    if (min > max)
        return;
    
    else if (is_nar(min))
    {
        printf("%d\n", min);
        return list_nar(min+1, max);
    }
    else
        return list_nar(min+1, max);
}

int main(void)
{
    int n, min, max;
    scanf("%d", &n);
    min = (int)pow(10, (n - 1));
    max = min * 10 -1;
    
    list_nar(min, max);

    return 0;
}
```

这个是普通递归，不是尾递归，运行到6位就报错了，抛出“段错误”。我重新用gcc -O2编译一次，没问题了。看来再牛B的语言也还得有个牛B的编译器才行啊。