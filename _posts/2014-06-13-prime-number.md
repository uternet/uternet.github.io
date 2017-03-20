---
layout: post
title: 找出指定范围内的素数
date: 2014-06-13
---

这是浙大 PAT 习题中的一题，它不要求输出素数，而是输出指定范围内，素数的个数以及求和。

用C已经写过了，再用 Scheme 写一遍：

```scheme
(define (prime? n)
  (define (iter n c)
    (cond
     ((= n 1) #t)
     ((= 0 (modulo c n)) #f)
     (else
      (iter (- n 1) c))))
  (iter (- n 1) n))

(define (find-prime start end)
  (define (iter start end l)
    (cond
     ((= start end) l)
     ((prime? start)
      (iter (add1 start) end (cons start l)))
     (else
      (iter (add1 start) end l))))
  (iter start end '()))

(define (sum_list l)
  (define (iter l r)
    (cond
     ((null? l) r)
     (else (iter (cdr l) (+ (car l) r)))))
  (iter l 0))

(define result '())

(let ((start (read)) (end (read)))
  (if (= start 1)
      (set! result
            (find-prime (add1 start) (add1 end)))
      (set! result
            (find-prime start (add1 end))))
  (display (length result))
  (display " ")
  (display (sum_list result))
  (newline))
```

其中用到了“副作用”，好象违背了函数式编程的原则了。。。

如果是在 REPL 中执行程序当然不用这么复杂，可是要做成脚本，或者编译成可执行文件，输出是个麻烦事，不用赋值的话我都不知道怎么把结果输出给用户看。

用 Scheme && Lisp 的时候，老觉得在REPL中运行程序不太象编程，老想找个办法把程序输出成单一可执行文件。而当我用 C 的时候，又觉得，要是 C 也有个 REPL 那该多好，调试代码太方便了。

------

发现一个算法，如果一个整数 N 能够由另外两个整数相乘得到，那么这两个整数肯定分别分布在N的平方根上下，（即，一个大于 N 的平方根，而另一个小于 N 的平方根）。我们编程判断一个数是否是素数的时候，往往用循环，将 N 除以（N-1 -> 2）之间的所有整数。有了上面的理论基础，算法可以简化为用 N 除以（N的平方根 -> 2）之间的所有整数。结果是一样的，但是计算量大大减少了。

上面的程序，prime?函数可以改为：

```scheme
(define (prime? n)
  (define (iter n c)
    (cond
     ((= n 1) #t)
     ((= 0 (modulo c n)) #f)
     (else
      (iter (- n 1) c))))
  (iter (ceiling (sqrt n)) n))
```

上面的函数是个包装过的递归函数，内层的函数相当于一个迭代器，所以我在外层求出平方根，并直接传递给内层。

另外，sqrt 函数返回的是一个实数，所以还得取整。Scheme 里取整函数有好几个，这里使用的 ceiling 返回的是不小于参数的最小整数，比如：10.1,返回 11。

而 floor 返回不大于参数的最大整数，相当于直接丢弃小数部分。

还有 round ，返回四舍五入的整数。口恩，弄错了，不是4舍5入，而是5舍6入。