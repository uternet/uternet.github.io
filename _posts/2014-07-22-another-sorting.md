---
layout: post
title: 另一种排序算法
date: 2014-07-22
---

昨天学到一种快速排序的写法，今天就用到了。

PAT 上有一道题，要求输入一系列学号及考试成绩，然后按一定的规则输出。这就涉及到了不同的排序方法，有升序有降序。而且输入的数据比较复杂，每个条目代表一个学生，一个条目中第一个数是学号，后两个数是两门课的考试成绩。我想了半天，最后决定用三个成员的列表来表示一个条目，然后用嵌套的列表来保存整个成绩单。这样就形成了类似于数据库的表结构。

一般的排序是对单个的列表进行的，而现在要从一个表中，选出一列来进行排序，真是难煞老夫也。

```scheme
;;定义几个选择器，从列表中取出特定位置的数据
(define (num-of l)  ;;学号
  (car l))

(define (de-of l)   ;;德分
  (cadr l))

(define (de+cai l)  ;;才分
  (+ (cadr l) (caddr l)))
  
;;利用上面的选择器，从一个嵌套列表中取出一列，组成一个单维列表
(define (gen-single-list l f)
  (let iter ((lst l) (select f) (result '()))
    (cond ((null? lst) result)
          (else
           (iter (cdr lst) select (cons (select (car lst)) result))))))

;;传入一个表，然后在内部生成一个对应的单维列表，在这个单维列表上递归，但是操作对象
;;却是传入的表，这样便实现了对表的排序
(define (cut item table select)
  (let ((n (select item))
        (lst (gen-single-list table select)))
    (let iter ((lst lst) (table table) (small-and-equal '()) (big '()))
      (cond ((null? lst) (cons small-and-equal big))
            ((> (car lst) n)
             (iter (cdr lst)
                   (cdr table)
                   small-and-equal
                   (cons (car table) big)))
            (else (iter (cdr lst)
                        (cdr table)
                        (cons (car table) small-and-equal)
                        big))))))

(define (sort-table table select)
  (cond ((null? table) '())
        (else
         (let ((cuted (cut (car table) (cdr table) select)))
           (append (sort-table (car cuted) select)
                   (cons (car table)
                         (sort-table (cdr cuted) select)))))))
```

另一个问题是这里的需求是既有升序，也有降序。一个表总共有三列 A B C，A 代表学号，B 表示德分，C 表示才分，总的排序是排照 B + C 降序排列，如果有两个同学的总成绩一样，则按 B 列降序排列，如果还有相同的，那就按 A 列（学号）升序排列。

昨天实现的快速排序只能按升序排列，如果要降序，只能整体 reverse 。可是涉及到多级排序的时候，整体 reverse 会打乱原先的顺序。reverse 两次后我的大脑便打结了。最后猜测，这样写是无法实现的。也许可以采用补丁的办法，在一级排序完成后，检测到重复，则对重复的部分应用二级排序，如果还有重复，再应用三级排序。我没有尝试这种写法，我觉得这样干太复杂了，转而寻找另外的解决办法。

最直接的办法是更换排序算法，还真给我找到了。

下面的代码可以对一个列表按指定的顺序排序，具体的顺序可以在调用时传入，比如 < >

```scheme
(define (insert2list item operator lst)
  (cond
   ((or (null? lst) (operator item (car lst)))
    (cons item lst))
   (else
    (cons (car lst)
          (insert2list item operator (cdr lst))))))

(define (sorting operator lst)
  (cond ((null? lst) '())
        (else (insert2list (car lst)
                           operator
                           (sorting operator (cdr lst))))))
```

这样做的好处是可以随意排序，不需要 reverse 结果。我修改了一下，使得排序仅仅影响到相关的元素，而不会影响到别的元素的相对位置。改了之后发现，其实这是不必要的。比如要按升序排序，传入一个小于等于号 <=，而不是小于号 <，这样就不会影响到相同元素的相对位置了。

这是我用这个方法实现的“表”排序：

```scheme
(define (num-of l)
  (car l))

(define (de-of l)
  (cadr l))

(define (de+cai l)
  (+ (cadr l) (caddr l)))

(define (gen-single-list lst fun)
  (let iter ((lst lst) (select fun) (result '()))
    (cond ((null? lst) result)
          (else
           (iter (cdr lst)
                 select
                 (cons (select (car lst)) result))))))

(define (insertToList item operator table select)
  (let ((n (select item))
        (lst (reverse (gen-single-list table select))))
    (cond ((or (null? lst)
               (operator n (car lst))
               (= n (car lst)))
           (cons item table))
          (else
           (cons (car table)
                 (insertToList item operator (cdr table) select))))))

(define (sort-table operator table select)
  (cond ((null? table) '())
        (else (insertToList (car table)
                            operator
                            (sort-table operator (cdr table) select)
                            select))))
```

这样就可以完美地进行多级排序了。不过带来的问题是效率低下，从算法上看，存在多余的计算，而且很显然上面的版本并非尾递归，我试图将它转换成尾递归形式，结果难住了。今天无力再改了，大脑已经抽筯了。

这是最后完成的作业，提交到 PAT 网站，结果是正确的，但是有两个测试点超时，效率的问题。。。

```scheme
(define (num-of l)
  (car l))

(define (de-of l)
  (cadr l))

(define (de+cai l)
  (+ (cadr l) (caddr l)))

(define (gen-single-list lst fun)
  (let iter ((lst lst) (select fun) (result '()))
    (cond ((null? lst) result)
          (else
           (iter (cdr lst)
                 select
                 (cons (select (car lst)) result))))))

(define (insertToList item operator table select)
  (let ((n (select item))
        (lst (reverse (gen-single-list table select))))
    (cond ((or (null? lst)
               (operator n (car lst))
               (= n (car lst))) ;;这里的修改其实是不必要的
           (cons item table))
          (else
           (cons (car table)
                 (insertToList item operator (cdr table) select))))))

(define (sort-table operator table select)
  (cond ((null? table) '())
        (else (insertToList (car table)
                            operator
                            (sort-table operator (cdr table) select)
                            select))))

(define (read-item n l h)
  (let iter ((cnt 1)
             (de-cai '()) (de>cai '()) (de>cai2 '()) (other '()))
    (cond ((> cnt n)
           (cons de-cai
                 (cons de>cai
                       (cons de>cai2
                             (cons other '())))))
          (else (let ((num (read)) (de (read)) (cai (read)))
                  (cond
                   ((and (>= de h) (>= cai h))
                    (iter (+ cnt 1)
                          (cons (cons num (cons de (cons cai '())))
                                de-cai)
                          de>cai de>cai2 other))
                   ((and (>= de h) (>= cai l))
                    (iter (+ cnt 1)
                          de-cai
                          (cons (cons num (cons de (cons cai '())))
                                de>cai)
                          de>cai2 other))
                   ((and (>= de l) (>= cai l) (>= de cai))
                    (iter (+ cnt 1)
                          de-cai de>cai
                          (cons (cons num (cons de (cons cai '())))
                                de>cai2)
                          other))
                   ((and (>= de l) (>= cai l))
                    (iter (+ cnt 1)
                          de-cai de>cai de>cai2
                          (cons (cons num (cons de (cons cai '())))
                                other)))
                   (else (iter (+ cnt 1)
                               de-cai de>cai de>cai2 other))))))))

(define (print-item l)
  (cond ((null? l) (display ""))
        (else (begin
                (display (car l))
                (if (null? (cdr l))
                    (newline)
                    (display " "))
                (print-item (cdr l))))))

(define (print-table l)
  (cond ((null? l) (display ""))
        (else (begin
                (print-item (car l))
                (print-table (cdr l))))))

(define (out-item table)
  (let* ((t1 (sort-table < table num-of))
         (t2 (sort-table > t1 de-of))
         (t3 (sort-table > t2 de+cai)))
    (print-table t3)))
    
(let ((n (read)) (l (read)) (h (read)))
  (let ((table (read-item n l h)))
    (let ((a (car table))
          (b (cadr table))
          (c (caddr table))
          (d (cadddr table)))
      (display (+ (length a) (length b) (length c) (length d)))
      (newline)
      (out-item a)
      (out-item b)
      (out-item c)
      (out-item d))))
```
