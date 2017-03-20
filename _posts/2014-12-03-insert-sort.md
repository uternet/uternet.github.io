---
layout: post
title: 插入排序 scheme 版
date: 2014-12-03
---

看 htdp 看到一个插入排序实现：

```scheme
(define (insert-sort alon)
  (cond
   ((null? alon) '())
   (else (insert (car alon) (insert-sort (cdr alon))))))


(define (insert n alon)
  (cond ((or (null? alon)
             (<= n (car alon)))
         (cons n alon))
        (else
         (cons (car alon) (insert n (cdr alon))))))
```

理解起来很简单，但是它是个普通递归程序，效率上有很大问题。我试图将它改成尾递归的时候遇到了难题。好象这个问题就是个递归问题，用递归来描述是最合适的。转成尾递归来考虑一下子就醉了。

后来又翻看《算法导论》，开篇第一个例子就是插入排序。它用了一个很浅显的例子来说明插入排序：

桌上有一叠扑克牌，从桌上摸起一张牌放到左手。左手只有这一张牌，不需要排序。

再一次摸起一张牌，在左手牌的序列中找到合适的位置插入进去。

重复这个过程，直到桌上的牌没有了。排序完毕！

这里有一个关键点：任何时刻，左手中的牌都是经过排序的。

很显然，这个算法需要两层嵌套循环：首先得遍历桌上的每一张牌（摸牌）；其次，每一次摸牌都要遍历左手中已有的牌（插入）。

看了扑克牌的例子，连伪代码我都没看，直接就动手写了。没有再参考 htdp 的写法，结果写成了这个样子：

```scheme
(define (insert-sort lst)
  (let iter ((lst lst) (ret '()))
    (if (null? lst)
        ret
        (iter (cdr lst) (insert (car lst) ret)))))

(define (insert n lst)
  (let iter ((prefix '()) (lst lst))
    (cond ((or (null? lst)
               (<= n (car lst)))
           (append prefix (cons n lst)))
          (else
           (iter (append prefix (list (car lst)))
                 (cdr lst))))))
```