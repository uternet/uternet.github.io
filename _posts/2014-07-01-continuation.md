---
layout: post
title: call/cc 探秘
date: 2014-07-01
---

对我来讲，Scheme 中的 Continuation 是个过于神秘的东西，一直都搞不懂（好吧，我承认我被吓坏了，我从来没尝试过搞懂它，因为我一遇见它就决定绕开）。

近来比较纠结，我把《TLS》看完了，不再害怕递归，解释器也写过了，《TLS》和《SICP》里的解释器都被我修改过，添加过东西，虽然比较丑陋，但还好它们都能跑起来。

接下来的道路选择我更倾向于CL，并不是因为 Scheme 没有好的实现，事实上 racket 就很好，最近几个版本的优化很给力，运行效率至少达到JAVA级别，远远超过一大堆脚本语言，库和文档也很齐全。但是，与 Scheme 相比，CL的标准要权威得多。完全不象 Scheme 社区，标准摆在那里，但是大家各行其是。事实上 racket 已经不能算是一个 Scheme 实现，现在应该称它为一个 Scheme 方言了。CL就不同了，虽然兼容性的问题仍然存在，但是标准执行得还是相对严格的。一个CL编译器可以借助别的CL编译器来实现自举，这在 Scheme 世界中是很难想象的。

废话完了，但是向 CL 进发前还有两个心结未了：continuation 和 宏。我知道CL的宏比 Scheme 强大得多，这部分可以去CL中学习，但是 continuation 是 Lisp 中学不到的。所以我花了好久的时间来下决心，决定把 continuation 这个神秘而又重要的东西搞懂，至少也要搞个半懂（信心不足ing）

接下来边看边写。

先从一个例子开始：

```scheme
(define retry #f)

(define factorial
  (lambda (x)
    (if (= x 0)
        (call/cc
         (lambda (k)
           (set! retry k)
           1))
        (* x (factorial (- x 1))))))
```

这是个阶乘函数，从[这里](http://blog.chinaunix.net/uid-7471615-id-3203533.html)看来的，但是作者的语言描述把我绕晕了，我决定自己理解。

代入参数 4 ,计算 (factorial 4)，然后分解递归过程：

```scheme
(* 4 (factorial 3))
(* 4 (* 3 (factorial 2)))
(* 4 (* 3 (* 2 (factorial 1))))
(* 4 (* 3 (* 2 (* 1 (factorial 0)))))
(* 4 (* 3 (* 2 (* 1 (call/cc (lambda (k) (set! retry k) 1))))))
```

(factorial 4) 得到结果 24，函数中的 call/cc 没有导致 factorial 发生异常，结果是对的。那么背后发生了什么呢？在上面递归分解的最后一步，call/cc 捕获了 continuation，并把它保存到一个变量 retry 里面。

接下来有趣的事发生了。把 retry 当成函数去调用不同的数字试试：

```scheme
(retry 1)
=> 24

(retry 2)
=> 48

(retry 3)
=> 72
```

这到底发生了什么？这些值是怎么得来的？

当执行 (retry n)的时候，世界停止了，retry 乘着时光机，不对，是月光宝盒，回到了当初捕获continuation 的地方，就是上面递归分解的最后一步，那里当初发生了什么呢？递归遇到了出口，最后一个表达式准备向上一层返回值（1）。上层还有一堆的表达式 `(* 4 (* 3 (* 2 (* 1`等待着这个返回值来进行递归回溯，以得到最终的结果。而 (retry n) 的出现，把这个值用 n 替换了，于是， (retry 2) 使得结果变成了 48，(retry 3) 得到了72.

这个retry并非是固定的，它取决于上一次运行的 factorial 函数，如果先计算 (factorial 5)，再来计算 (retry n) 又会得到不同的结果，相当于 factorial 在运行过程中做了一个快照，然后保存在变量 retry 中。

那么 continuation 是什么呢？它是数据吗？显然不是，如果是数据的话，捕获的应该是数值1,你不能把 1 当成函数来调用。如果 continuation 是函数，那么它的结构是怎么样的呢？暂时不懂中。

边看边写，有了理解来再更。