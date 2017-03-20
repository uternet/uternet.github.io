---
layout: post
title: 二叉搜索树的 scheme 实现
data: 2014-12-24
---

用递归很好写，但是效率不太好。尾递归不太好写，坦白讲我不知道怎么写。

分别写了两个版本，一个有副作用，一个没有。另外从网上看到另一种写法，用的是结构体而不是列表。

没有副作用的版本：

```scheme
(define left cadr)
(define right caddr)

(define (leaf? node)
  (and (null? (left node))
       (null? (right node))))
       
;; 插入新值
(define (insert abst val)
  (cond ((null? abst)
         (list val '() '()))
        ((= val (car abst))
         abst)
        ((< val (car abst))
         (list (car abst)
               (insert (left abst) val)
               (right abst)))
        (else
         (list (car abst)
               (left abst)
               (insert (right abst) val)))))

;; 找出最大值
(define (find-max abst)
  (cond ((null? (right abst))
         (car abst))
        (else
         (find-max (right abst)))))

;; 找出最小值
(define (find-min abst)
  (cond ((null? (left abst))
         (car abst))
        (else
         (find-min (left abst)))))

;; 搜索
(define (search abst val)
  (cond ((null? abst) '())
        ((= val (car abst)) abst)
        ((< val (car abst))
         (search (left abst) val))
        (else
         (search (right abst) val))))

;; 删除
(define (delete abst val)
  (cond ((null? abst) #f)
        ((< val (car abst))
         (list (car abst) (delete (left abst) val) (right abst)))
        ((> val (car abst))
         (list (car abst) (left abst) (delete (right abst) val)))
        (else
         (cond ((leaf? abst) '())
               ((null? (left abst)) (right abst))
               ((null? (right abst)) (left abst))
               (else
                (let ((min (find-min (right abst))))
                  (list min
                        (left abst)
                        (delete (right abst) min))))))))
```


有副作用的写法，写了个插入函数，删除函数不知道要怎么写了,另外还有一个问题，当传入参数是个空表时，也没有副作用，必须在调用时显式地执行 set!, 容易搞混。

```scheme
(define (insert BST val)
  (cond
   ((null? BST)
    (list val '() '()))
   ((= val (car BST))
    BST)
   ((< val (car BST))        ;;每个分支有两个语句
    (set-car! (cdr BST)      ;;1 修改值，纯纯的副作用
          (insert (left BST) val))
    BST)                     ;;2 还得返回点什么东西，递归要用到这个返回值
   (else
    (set-car! (cddr BST)
          (insert (right BST) val))
    BST)))
```

最后是别处看到的结构体实现，代码不完整，仅仅实现了删除函数，不过大体上可以知道思路。在结构体定义中有四个成员，key 是键（数字） , value 是值（任意类型），left 和 right 要么是空表，要么是同类型的结构体。

```scheme
(define (remove-root abst)
  (cond
    [(empty? abst) empty]
    [(and (empty? (node-left abst))
          (empty? (node-right abst))) empty]
    [(empty? (node-left abst)) (node-right abst)]
    [(empty? (node-right abst)) (node-left abst)]
    [else (make-node (node-key (node-right abst))
                     (node-value (node-right abst))
                     (node-left abst)
                     (remove-root (node-right abst)))]))

(define (remove-key n abst)
  (cond
    [(empty? abst) empty]
    [(= n (node-key abst)) (remove-root abst)]
    [(< n (node-key abst)) (make-node (node-key abst)
                                      (node-value abst)
                                      (remove-key n (node-left abst))
                                      (node-right abst))]
    [else (make-node (node-key abst)
                     (node-value abst)
                     (node-left abst)
                     (remove-key n (node-right abst)))]))
```

接下来，AVL树比较考验脑细胞。。。