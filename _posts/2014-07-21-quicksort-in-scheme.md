---
layout: post
title: 数字黑洞 & Scheme 实现快速排序算法
date: 2014-07-21
---

其实这是浙大 PAT 网站上的一个题，要求输入一个小于 10000 的正整数，输出 6174 数字黑洞的计算步骤。既然要把数字位打散重新排序，那就得应用某种算法。之前从没接触过排序算法，Google 之后参考别人的写法写了一个快速排序函数。因为，PAT 侧重其它语言的测试，在输入其输出格式上颇为严格，而 Scheme 在输入及输出上比较弱，虽然 PAT 上指明了用的是实现是 guile ，但是不知道为什么，我在代码中导入 guile 的模块会发现无法通过测试。没办法，只好用 R5RS 里的标准函数来写。所以，搞得比较累，往往结果出来了，发现格式不对，于是再改代码，最后弄得比较乱。

```scheme
;;排序过滤器
(define (filter n lst)
  (let iter ((l lst) (small-and-equal '()) (big '()))
    (cond ((null? l) (cons small-and-equal big))
          ((> (car l) n)
           (iter (cdr l) small-and-equal (cons (car l) big)))
          (else (iter (cdr l) (cons (car l) small-and-equal) big)))))

;;快速排序
(define (qsort l)
  (cond ((null? l) '())
        (else
         (let ((cuted (filter (car l) (cdr l))))
           (append (qsort (car cuted))
                   (cons (car l)
                         (qsort (cdr cuted))))))))

;;分解数字
(define (make-mask n)
  (let iter ((num n) (mask 1))
    (cond ((< num 10) mask)
          (else (iter (quotient num 10) (* mask 10))))))

;;注意，这个函数分解出的数字是倒序的，不过这里没关系，因为还要经过排序
(define (cut-number n)
  (let iter ((num n) (mask (make-mask n)) (result '()))
    (cond ((< mask 1) result)
          (else (iter (modulo num mask)
                      (quotient mask 10)
                      (cons (quotient num mask) result))))))
;;位数不足四位的在前面补0
(define (cut n)
  (let iter ((ret (cut-number n)))
    (cond ((= 4 (length ret)) ret)
          (else (iter (cons 0 ret))))))

;;把列表组合成数字
(define (make-number l)
  (let iter ((lst l)
             (mask (expt 10 (- (length l) 1)))
             (result 0))
    (cond ((< mask 1) result)
          (else (iter (cdr lst)
                      (/ mask 10)
                      (+ result (* (car lst) mask)))))))
;;显示4位数字
(define (display-4bit n)
  (cond ((< n 10) (begin (display "000") (display n)))
        ((< n 100) (begin (display "00") (display n)))
        ((< n 1000) (begin (display "0") (display n)))
        (else (display n))))

;;每计算一次，输出一行
(define (out-line b s)
  (display-4bit b) (display " - ")
  (display-4bit s) (display " = ")
  (display-4bit (- b s))
  (newline))

;;主函数
(define (kaprekar n)
  (let* ((big (reverse (qsort (cut n))))
         (small (reverse big)))
    (let ((big-num (make-number big))
          (small-num (make-number small)))
      (cond ((= small-num big-num)
             (out-line big-num small-num))
            ((= (- big-num small-num) 6174)
             (out-line big-num small-num))
            (else
             (begin
               (out-line big-num small-num)
               (kaprekar (- big-num small-num))))))))

(kaprekar (read))
```


