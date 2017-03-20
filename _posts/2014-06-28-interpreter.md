---
layout: post
title: 一个相对完整的解释器
date: 2014-06-28
---

把 The Little Schemer 第10章的解释器改巴改巴，实现了几个重要的功能：  

1.define 定义顶级环境变量绑定，局部定义 let 和嵌套的 define 暂时实现不了。  

2.`+ - * /` 调用了底层系统的，用 sub1 和 add1 来递归实现的话效率太慢，而且我不知道要怎么实现浮点数运算。  

3.实现了递归。  

可以用 chicken 或者 racket 编译成可执行文件，脱离宿主系统运行（如果用racket编译的话，其实还是带有racket自己的运行环境的）。如果把代码写在一个初始化脚本里，比如 init.scm ，再在这个小系统里读入并执行，理论上可以让它自己扩展自己，从而变成一个完整的实现。不过意义不大，而且这个小系统结构也有很大的问题，还得再改巴改巴。

几乎每个函数后都有个 table 参数，实参在这些函数间传来传去很容易就跟丢了，调试实在是困难，没办法在代码里写了N多的 display，输出中间结果。我仿佛又看到那个小老头在一脸坏笑的说：你的头大成这样，帽子还合适吗？

```scheme
#!/usr/bin/env racket
#lang racket/base

(define envt '())

;;init

(define (repl evaluator)
  (display "repl> ")
  (let ((expr (read)))
    (cond ((eq? expr 'exit)
           (display "Exiting read-eval-print loop")
           (newline))
          (else
           (write (evaluator expr))
           (newline)
           (repl evaluator)))))
  
;;辅助函数
(define build
  (lambda (a b)
    (cons a (cons b '()))))

(define atom?
  (lambda (s)
    (and (not (pair? s))
         (not (null? s)))))

(define add1
  (lambda (n)
    (+ n 1)))

(define sub1
  (lambda (n)
    (- n 1)))

;;用嵌套列表来保存“环境”，这种嵌套的列表在TLS中称为表，每一个表由“条目(entry)“
;;构成，条目也是嵌套列表，它由两个列表构成，而且两个列表的成员数量一致，前一个列表不
;;包含重复的元素，这种列表在TLS中被称为 set 。这样就构成了 名称-值 的关联。
;;下面的函数实现了通过给定第一个列表中的 名字 ，从后一个列表中找出对应的值。

(define lookup-in-entry
  (lambda (name entry entry-f)
    (lookup-in-entry-help name
                          (car entry)
                          (cadr entry)
                          entry-f)))

(define lookup-in-entry-help
  (lambda (name names values entry-f) ;键列表为空，或者没有匹配的情况下
    (cond                             ;交给 entry-f 去处理
     ((null? names) (entry-f name))
     ((eq? (car names) name)
      (car values))
     (else (lookup-in-entry-help name
                                 (cdr names)
                                 (cdr values)
                                 entry-f)))))

;;向下面的函数传递一个 名字 和一个 表，历遍表中的条目返回对应的应的值
;;很显然，需要用上面的 lookup-in-entry 作为辅助函数
(define lookup-in-table
  (lambda (name table table-f)
    (cond
     ((null? table) (table-f name))
     (else
      (lookup-in-entry name
                       (car table)
                       (lambda (name)  ;这里的匿名函数对应的是上面的entry-f
                         (lookup-in-table 
                          name (cdr table) table-f)))))))

(define is-member?
  (lambda (a l)
    (cond ((null? l) #f)
          ((eq? a (car l)) #t)
          (else (is-member? a (cdr l))))))

(define env-exist?
  (lambda (a table)
    (cond ((null? table) #f)
          ((is-member? a (caar table)) #t)
          (else (env-exist? a (cdr table))))))

(define *get-body
  (lambda (e table)
    (meaning 
     (cons (lookup-in-table (car e) envt initial-table) 
           (cdr e)) 
     table)))

;;这里有六种基本的类型：
;;1、常量
;;2、引用
;;3、标识符
;;4、lambda
;;5、cond
;;6、程序（代入了参数的 lambda）

 ;;传入一个S-expr，根据类型选择不同的动作，首先判断是原子还是列表
(define expression-to-action
  (lambda (e)
    (cond
     ((atom? e) (atom-to-action e))
     (else (list-to-action e)))))


;;根据不同类型的原子，选择执行动作
(define atom-to-action
  (lambda (e)
    (cond
     ((number? e) *const)
     ((eq? e #t) *const)
     ((eq? e #f) *const)
     ((eq? e (quote cons)) *const)
     ((eq? e (quote car)) *const)
     ((eq? e (quote cdr)) *const)
     ((eq? e (quote null?)) *const)
     ((eq? e (quote eq?)) *const)
     ((eq? e (quote atom?)) *const)
     ((eq? e (quote zero?)) *const)
     ((eq? e (quote add1)) *const)
     ((eq? e (quote sub1)) *const)
     ((eq? e (quote number?)) *const)
     ((eq? e '+) *const)
     ((eq? e '-) *const)
     ((eq? e '*) *const)
     ((eq? e '/) *const)
     ((eq? e '>) *const)
     ((eq? e '<) *const)
     ((eq? e '=) *const)
     (else *identifier))))

;;解析列表了
(define list-to-action
  (lambda (e)
    (cond
     ((atom? (car e))
      (cond
       ((eq? (car e) (quote quote)) *quote)
       ((eq? (car e) (quote lambda)) *lambda)
       ((eq? (car e) (quote cond)) *cond) 
       ((eq? (car e) (quote define)) *define)
       ((env-exist? (car e) envt) *get-body)
       (else *application)))  ;前置操作符中，除了上面的 quote lambda cond
     (else *application))))   ;统统都是过程
      ;缺少前置操作符的嵌套列表也是过程，这里表示的是接收到实参的 lambda 表达式

(define *define
  (lambda (e table)
    (set! envt (cons (build (cons (cadr e) '())
                           (cons (caddr e) '()))
                    envt))))

;;求值函数,把表达式 e 和一个空表传给下面的 meaning 函数
(define value
  (lambda (e)
    (meaning e '())))


(define meaning
  (lambda (e table)
    ;(display "expr-> ") (write e) (newline)
    ;(display "table-> ") (write table) (newline) (newline)
    ((expression-to-action e) e table)))


;;expression-to-action 先对表达式 e 进行判断，返回执行具体动作的另一个函数
;;再把表达式 e 传入动作函数，同时传入的还家一个空列表
;;这里返回的函数就是上面提到的:
;;  *const
;;  *identifier
;;  *quote
;;  *lambda
;;  *cond
;;  *application
;; 
;;最简单的 常量 动作函数
(define *const
  (lambda (e table)
    (cond
     ((number? e) e)
     ((eq? e #t) #t)
     ((eq? e #f) #f) ;;数字和布尔值，返回它们自身
     (else
      (build (quote primitive) e)))))
;;cons car cdr null? eq? atom? zero? add1 sub1 number?
;;等等返回一个类似的列表： (primitive cons)
;;表示这是一个原始的内置过程

;;处理 quote 引用，简单地返回它后面的值
(define *quote
  (lambda (e table)
    (cadr e)))

;;envt是 toplevel 环境，table 是局部环境
(define *identifier
  (lambda (e table)
    (or (lookup-in-table e table initial-table)
        (lookup-in-table e envt initial-table))))

;;这里的 initial-table 就是上面缺少的 table-f 啊
(define initial-table
  (lambda (name)
    #f))


;;处理 lambda 了
;; lambda表达式 的形参和函数体就是整个表达式的 cdr 部分
(define *lambda
  (lambda (e table)
    (build (quote non-primitive)
           (cons table (cdr e)))))

;;   (non-primitive (   (传入的参数)   (形参) (函数体) ) )
;;                         table

;这就是传递给 evcon 的 lines 参数
(define evcon
  (lambda (lines table)
    (cond
     ((else? (car (car lines))) ;car 取出一行，用问题部分与 'else比较
      (meaning (cadr (car lines)) table)) ;对回答部分求值
     ((meaning (car (car lines)) table)
      (meaning (cadr (car lines)) table))
     (else (evcon (cdr lines) table))))) ;递归剩下的行

;;判断是否是 else 行
(define else?
  (lambda (x)
    (cond
     ((not (pair? x)) (eq? x 'else))
     (else #f))))

;;完成 *cond 函数
(define *cond
  (lambda (e table)
    (evcon (cdr e) table)))


;;处理传给函数的参数，从参数列表中取出一个参数（car），连同 table
;;一起传给 meaning 函数进行处理。
;; meaning 再交给 expression-to-action 函数判定要执行的动作
;; 简单起见，这里假定列表中的参数都是自求值的原子，于是递归 cons 的结果
;; 是 args 被原样返回了
;; 当然，args 里面可以包含更复杂的东西。引入 meaning 可以递归地处理它们
(define evlis
  (lambda (args table)
    (cond
     ((null? args) '())
     (else
      (cons (meaning (car args) table)
            (evlis (cdr args) table))))))

;;处理过程了。这里的过程包括两种形式：
;;(+ 12 44) 这种以及 ((lambda ...) 23 43) 这种
;;不管哪一种，它们在形式上都是统一的：car 是过程，cdr 是参数
;;取出 过程 及 参数部分，再把它们交由 apply 函数进行处理
(define *application
  (lambda (e table)
    (Myapply
     (meaning (car e) table)
     (evlis (cdr e) table))))

;;首先判断是否是原始（内置）函数
(define primitive?
  (lambda (l)
    (eq? (car l) (quote primitive))))

(define non-primitive?
  (lambda (l)
    (eq? (car l) (quote non-primitive))))

;;这里传入的 fun 在前面带有 primitive 以及 non-privitive 标记
;;所以要用 cadr 取出函数。
;; primitive传入参数示例：
;;   (primitive cons)
;; non-primitive (lambda) 传入参数示例：
;;   (non-primitive ((table) (形参) (函数体)))
;; 经过apply变换后，lambda 表达式变成了 ((table) (形参) (函数体))
(define Myapply
  (lambda (fun vals)
    (cond
     ((primitive? fun)
      (apply-primitive (cadr fun) vals))
     ((non-primitive? fun)
      (apply-closure (cadr fun) vals)))))

;;调用同名的内置函数进行处理
(define apply-primitive
  (lambda (name vals)
    (cond
     ((eq? name 'cons)
      (cons (car vals) (cadr vals)))
     ((eq? name 'car)
      (car (car vals)))
     ((eq? name 'cdr)
      (cdr (car vals)))
     ((eq? name 'null?)
      (null? (car vals)))
     ((eq? name 'eq?)
      (eq? (car vals) (cadr vals)))
     ((eq? name 'atom?)
      (:atom? (car vals)))
     ((eq? name 'zero?)
      (zero? (car vals)))
     ((eq? name 'add1)
      (add1 (car vals)))
     ((eq? name 'sub1)
      (sub1 (car vals)))
     ((eq? name 'number?)
      (number? (car vals)))
     ((eq? name '+)
      (apply + vals))
     ((eq? name '-)
      (apply - vals))
     ((eq? name '*)
      (apply * vals))
     ((eq? name '/)
      (apply / vals))
     ((eq? name '>)
      (apply > vals))
     ((eq? name '<)
      (apply < vals))
     ((eq? name '=)
      (apply = vals)))))


(define :atom?
  (lambda (x)
    (cond
     ((atom? x) #t)
     ((null? x) #t)
     ((eq? (car x) 'primitive) #t)
     (else #f))))

;;应用非内置函数
(define apply-closure
  (lambda (closure vals)
    (meaning (caddr closure) ;caddr 从闭包中取出函数体
             (cons
              (build (cadr closure) vals) ;cadr取出形参，build把形参和实参组成一个新列表
              (car closure))))) ;car 取出环境表

(repl value)
```