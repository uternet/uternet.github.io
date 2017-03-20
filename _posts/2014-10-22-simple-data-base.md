---
layout: post
title: 用 Scheme 写的简单数据库
date: 2014-10-22
---

《实用 Common Lisp 编程》开篇用CL实现了一个简单的数据库，我自认为知道点 Scheme, 同样作为上古 Lisp 的方言，两者应该相差不会太大，结果一开篇就看不太懂了。

plist 是啥？ Scheme 里有同样的东西吗？不知道

可以在嵌套列表上递归打印的 format 函数，Scheme 里好象没有。

两者的差异还是挺大的。

其实我更喜欢 Scheme 的语法，想传递一个函数作为参数就直接传，哪里需要加什么引用和#号，也没有 funcall这玩意。多干净！

涉及到数据转换的内置函数，函数名里全部有 ->；谓词全部以问号?结尾；有副作用的函数统一加感叹号结尾。多统一！不象CL，有些谓词结尾有p而有的没有, 函数有没有副作用，从函数名上根本看不出来。

既然不适应CL的语法，那就重操旧业，用 Scheme 把这个数据库实现出来，就当练手了。

因为数据结构不一致，以及一些高度抽象的 CL 函数在 Scheme 里没有对应的替代品（或者是我不知道），所以代码变长了点。初步测试，可以正确运行。

```scheme
(define title-of car)
(define artist-of cadr)
(define rating-of caddr)
(define ripped-of cadddr)
(define make-cd list)

;;用来保存数据库的全局变量
(define *db* '())

(define (add-record item) 
  (set! *db* (cons item *db*)))

(define (dump-item item)
  (printf "TITLE:\t~a\n" (title-of item))
  (printf "ARTIST:\t~a\n" (artist-of item))
  (printf "RATING:\t~a\n" (rating-of item))
  (printf "RIPPED:\t")
  (if (ripped-of item)
      (printf "Yes\n")
      (printf "No\n"))
  (newline))

(define (dump-db)
  (let iter ((db *db*))
    (if (null? db) (display "")
        (begin
          (dump-item (car db))
          (iter (cdr db))))))

(define (prompt-read prompt)
  (printf "~a: " prompt)
  (my-read-line))

;; racket的read-line有一个bug，在repl里会读到空字符串，在DrRacket里和脚本
;; 却不会，这里加了一个自己的 my-read-line 函数，如果读到空字符串就重新读取
(define (my-read-line)
  (let ((contents (read-line)))
    (if (string=? contents "")
        (read-line)
        contents)))

(define (y-or-n str)
  (let ((answer (prompt-read str)))
    (cond ((or (string=? answer "y")
               (string=? answer "Y"))
           #t)
          ((or (string=? answer "n")
               (string=? answer "N"))
           #f)
          (else
           (begin
             (printf "Must answer 'Y' or 'N'\n")
             (y-or-n str))))))

(define (prompt-for-cd)
  (make-cd
   (prompt-read "Title")
   (prompt-read "Artist")
   (string->number
    (prompt-read "Rating"))
   (y-or-n "Ripped [y/n]")))

;;批量添加 CD 记录
(define (add-cds)
  (let iter ()
    (add-record (prompt-for-cd))
    (if (y-or-n "Another? [y/n]")
        (iter)
        'Done)))

;;将数据库保存到文件
(define (save-db filename)
  (if (not (file-exists? filename))
      (let ((op (open-output-file filename)))
        (write *db* op)
        (close-output-port op))
      (begin
        (delete-file filename)
        (save-db filename))))

;;从文件加载数据
(define (load-db filename)
  (let ((ip (open-input-file filename)))
    (let ((data (read ip)))
      (if (eof-object? data)
          (printf "File is empty!\n")
          (set! *db* data))
      (close-input-port ip))))

(define (remove-if-not test lst)
  (cond ((null? lst) '())
        ((test (car lst))
         (cons (car lst) (remove-if-not test (cdr lst))))
        (else (remove-if-not test (cdr lst)))))

(define (select selector-fn)
  (remove-if-not selector-fn *db*))

(define (where . args)
  (lambda (cd)
    (let iter ((args args))
      (cond ((null? args) #f)
            (else (let ((field (car args)) (value (cadr args)))
                    (or (parsing-and-test field value cd)
                        (iter (cddr args)))))))))

(define (parsing-and-test field value cd)
  (cond ((symbol=? field 'title)
         (equal? (title-of cd) value))
        ((symbol=? field 'artist)
         (equal? (artist-of cd) value))
        ((symbol=? field 'rating)
         (equal? (rating-of cd) value))
        ((symbol=? field 'ripped)
         (equal? (ripped-of cd) value))
        (else
         #f)))

(define (update selector-fn . args)
  (set! *db* (update-if-match selector-fn *db* args)))

(define (update-if-match test db replace)
  (cond ((null? db) '())
        ((not (test (car db)))
         (cons (car db) (update-if-match test (cdr db) replace)))
        (else
         (cons (update-item (car db) replace)
               (update-if-match test (cdr db) replace)))))

(define (update-item item replace)
  (let ((title (or (get-field-value 'title replace)
                   (title-of item)))
        (artist (or (get-field-value 'artist replace)
                    (artist-of item)))
        (rating (or (get-field-value 'rating replace)
                    (rating-of item)))
        (ripped (if (field-exists? 'ripped replace) ;;必需先检查是否传入了'ripped,否则无法把#f传递进去
                    (get-field-value 'ripped replace)
                    (ripped-of item))))
    (list title artist rating ripped)))

(define (get-field-value field replace)
  (cond ((null? replace) #f)
        ((symbol=? field (car replace)) (cadr replace))
        (else
         (get-field-value field (cddr replace)))))

(define (field-exists? field replace)
  (cond ((null? replace) #f)
        ((symbol=? field (car replace)) #t)
        (else (field-exists? field (cddr replace)))))
```

实现的功能：

1、添加记录

    (add-cds)

2、将数据库保存到文件

    (save-db "filename")

3、从硬盘文件读入数据库

    (load-db "filename")

3、格式化输出数全部的数据库条目

    (dump-db)

4、查找记录

    (select (where '字段 "匹配值")) 

字段 是引用的symbol类型，包括 'title 'artist 'rating 'ripped 四个   
字段和匹配值可以指定多对

5、更新记录

    (update (where '字段 "匹配值") '字段 "替换值") 

字段与匹配值以及字段与替换值可以指定多对

