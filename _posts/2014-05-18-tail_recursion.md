---
layout: post
title: 递归与尾递归
date: 2014-05-18
---

网络上的资料真是不严谨，对于非科班出身的人，试图搞懂一个专业问题的时候，往往会被网络误导。比如关于尾递归的严格、权威的概念，我到现在都搞不清楚，网上的示例，有一些把简单的问题搞复杂，而有一些干脆就是错的。

比如这里：[漫谈递归：从斐波那契开始了解尾递归](http://www.nowamagic.net/librarys/veda/detail/2325)

短短的一小段代码就发现3个问题：  

一、先是把函数名都写错了，factorial 是阶乘，可是他的函数体明明算的是斐波那契数列；

二、main 函数里声明了一个从未用到的变量 i ；

三、最搞的是，他在main函数里使用了 scanf 函数来给 factorial 函数传递参数，然后测试程序的运行时间，最后得出尾递归版“快一倍”的结论。当程序运行后，停在那里等待用户输入，这种测试有意义吗？

事实上，程序转为尾递归后提供的加速是非线性的，随着计算量的增多而呈几何级的增加，我仅仅计算到40,尾递归版快了何止上千倍！

这是我修改的 C 语言 fibonacci 函数普通尾递归版：

```c
#include <stdio.h>

long long fibonacci(int n);

int main(void)
{
    printf("%lld\n", fibonacci(40));

    return 0;
}

long long fibonacci(int n)
{
    if (n <= 2) {
        return 1;
    }
    else {
        return fibonacci(n-1) + fibonacci(n-2);
    }
}
```

这是尾递归版：

```c
#include <stdio.h>


unsigned long long fib(int n, unsigned long long a1, unsigned long long a2)
{
    if (n < 2)
        return a1;
    else
        return fib(n - 1, a2, a1 + a2);
}

int main(int argc, char *argv[])
{
    int n;
    n = atoi(argv[1]);
    printf("%llu\n", fib(n,1,1));
    
    return 0;
}
```

这是 Scheme 写的 fibonacci 普通递归版：

```scheme
(define (fibonacci n)
  (if (<= n 2)
      1
      (+ (fibonacci (- n 1))
         (fibonacci (- n 2)))))

```

这是尾递归版：

```scheme
(define (fibonacci n a1 a2)
  (cond
   ((= n 0) 0)
   ((< n 2) a1)
   (else
    (fibonacci (- n 1) a2 (+ a1 a2)))))
    
;;调用 (fibonacci n 1 1)
```

在内层定义一个子函数 iter 包装一下，调用的时候可以用单参数，这样美观一点

```scheme
(define (fibonacci n)
  (define (iter n t1 t2)
    (cond
     ((= n 0) 0)
     ((< n 2) t1)
     (else (iter (- n 1)
                 t2
                 (+ t1 t2)))))
  (iter n 1 1))

;;调用 (fibonacci n)
```

这才是阶乘嘛

```scheme
(define (factorial n)
  (if (= n 0)
      1
      (* n (factorial (- n 1)))))
```

下面两个尾递归版都是来源于 wiki：

```scheme
(define (factorial n)
  (define (iter product counter)
    (if (> counter n)
        product
        (iter (* counter product)
              (+ counter 1))))
  (iter 1 1))
```

```scheme
(define (factorial n)
    (let fact ([i n] [acc 1])
      (if (zero? i)
          acc
          (fact (- i 1) (* acc i)))))
```

最后是来源于[知乎](http://www.zhihu.com/question/20761771/answer/19996299)的回答，计算的是累计求和，用的是Python

```python
#递归版
def recsum(x):
  if x == 1:
    return x
  else:
    return x + recsum(x - 1)
    
#迭代版
for i in range(6):
  sum += i

#尾递归版
def tailrecsum(x, running_total=0):
  if x == 0:
    return running_total
  else:
    return tailrecsum(x - 1, running_total + x)
```

下面，我用 Scheme 来实现一个相同的函数

```
(define (tailrecsum x total)
  (if (zero? x)
      total
      (tailrecsum (sub1 x) (+ total x))))
```

好了，程序怎么跑我理解了，可还是没得到一个足够权威和严谨的定义。我从“现象”中归纳出的结论就是：尾递归就是把变化的参数传递给递归函数，而所需要的结果在传递参数的过程中就计算完成了。从表现形式上看，尾递归函数在最末尾总是只有一条单独的自身调用，没有其它的计算发生。这样的函数，当递归到最内层的时候，直接就返回值了，不需要递归回溯（不知道这样表述是不是准确），而常规的递归函数，最外层的求值依依赖内层的返回值参与计算，递归展开后，还得一层一层地往外层返回值，最后，在最外层完成计算，得到结果。
