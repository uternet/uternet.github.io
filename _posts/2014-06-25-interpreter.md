---
layout: post
title: 把王垠的 scheme 解释器改了一下
date: 2014-06-25
---

王垠的[这个](http://www.yinwang.org/blog-cn/2012/08/01/interpreter/)解释器比起 The Little Schemer 第10章那个，简单得多，很容易就搞懂了。当然也差了一些东西，比如 quote cond 等。如果把这些东西加上去，两者也就差不多一样了。它们唯一的区别是，王垠的版本用 association list 来保存环境，而 TLS 里的那个用三层嵌套的列表来干这个事，结果很容易就被绕晕了。

王垠原先的代码用到了 racket 的一些专有特性，比如结构定义和模式匹配，不能在其它环境下运行。我准备把它改写成标准的 Scheme 语法，主要还是加深理解，确保我真的理解了这个程序。

下面的代码在判断表达式类型时用列表长度来做判断，这是比较愚蠢的，但是不想改了。我现在比较感兴趣的是，怎么实现 define，这样能干的事就更多了。还有递归怎么实现，要用到 Y组合子吗？如果实现了这两者，那这个小玩意就可以算个“完整”的解释器了。

```scheme
;;#lang racket/base
(define env0 '())

(define ext-env
  (lambda (x v env)
    (cons `(,x . ,v) env)))

(define lookup
  (lambda (x env)
    (let ((p (assq x env)))
      (cond
       ((not p) #f)
       (else (cdr p))))))

(define interp1
  (lambda (exp env)
    (cond
     ((symbol? exp) (lookup exp env))
     ((number? exp) exp)
     ;;列表只有两个成员，是函数调用
     ((= (length exp) 2)    ;;如果是类似 (f n)的表达式，很显然 f 是个函数
      (let ((v1 (interp1 (car exp) env))      ;;先对 f 求值,赋给 v1
            (v2 (interp1 (cadr exp) env)))    ;;再对参数求值 赋给 v2
        (if (eq? (car v1) 'Closure)           ;;如果找到打上Clousure标记的 lambda
            (let ((e (caddr (cadr v1)))       ;;取出函数体
                  (x (car (cadr (cadr v1))))) ;;取出形参
              (interp1 e (ext-env x v2 env))) ;;把形参和实参组成一个点对，压进 env
            (display "error code 1\n"))))     ;;现在 e 是待求值的表达式，env中存有变量
     ;;三成员列表，可能是算术表达式，或者是 lambda
     ((= (length exp) 3)
      (cond
       ((eq? (car exp) 'lambda)
        (list 'Closure exp env))       
       (else (let ((op (car exp))
                   (v1 (interp1 (cadr exp) env))
                   (v2 (interp1 (caddr exp) env)))
               (cond
                ((eq? op '+)
                 (+ v1 v2))
                ((eq? op '-)
                 (- v1 v2))
                ((eq? op '*)
                 (* v1 v2))
                ((eq? op '/)
                 (/ v1 v2))))))))))

(define value
  (lambda (exp)
    (interp1 exp env0)))
```