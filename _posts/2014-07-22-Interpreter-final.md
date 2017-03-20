---
layout: post
title: Scheme 解释器“最终版”
date: 2014-07-22
---

其实这是很早就写好的了，已经好多天没有鼓捣解释器了。

实现的特性：define定义 let局部变量定义 递归 环境初始化脚本

本解释器参考了SICP上的实现，但是改用哈希表来保存环境。因为不知道怎么把哈希表弄成链式的数据结构，所以在实现局部变量的时候，我笨拙地复制了一份顶层环境到局部环境，这样干效率和内存占用就很成问题了。不过这不是重点，我没想过也没有能力把这个玩意完善成有实用价值的东西。我仅仅想探究下解释器内部是怎么运行的。

另外一点是实现了从硬盘读入初始化文件，在初始化文件里可以定义一些内置函数。不过解释器本身结构就不合理，也没有完善的必要了。

主程序：

```scheme
#!/usr/local/bin/racket
#lang racket/base

;;;;;;;;;;;;;;;;;;;;;;;;;;;辅助函数;;;;;;;;;;;;;;;;;;;;;;;;;;;;
(define (tagged-list? exp tag)
  (if (pair? exp)
      (eq? (car exp) tag)
      #f))

(define (true? x)
  (not (eq? x #f)))

;;;;;;;;;;;;;;;;;;;;;;;;;;主函数;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (eval0 exp env)
;(display "expression: ") (display exp) (newline) (newline)
;(display "environment: ") (display env) (newline) (newline)
  (cond ((self-evaluating? exp) exp)                  ;自求值
        ((variable? exp) (lookup-variable-value exp env)) ;变量
        ((quoted? exp) (cadr exp))                    ;quote
        ((assignment? exp) (eval-assignment exp env)) ;set!
        ((let? exp) (eval-let exp env))
        ((definition? exp) (eval-definition exp env)) ;define
        ((if? exp) (eval-if exp env))                 ;if
        ((lambda? exp)                                ;lambda
        ;(display "make-procedure is running\n")
         (make-procedure (cadr exp)  ;形参
                         (cddr exp)  ;(函数体)
                         env))
        ((begin? exp)                                  ;begin
         (eval-sequence (cdr exp) env))
        ((cond? exp) (eval0 (cond->if exp) env))        ;cond
        ((eq? (car exp) 'or) (eval-or exp env))
        ((eq? (car exp) 'and) (eval-and exp env))
        ((eq? (car exp) 'not) (eval-not exp env))
        ((application? exp)                   ;函数调用
         (let ((env1 (hash-copy env)))
           (let ((operator (eval0 (car exp) env1)))        ;先对操作符求值
             (if (eq? (car operator) 'lambda)
                 (apply1 (eval0 operator env1)
                         (list-of-values (cdr exp) env1))
                 (apply1 operator    
                         (list-of-values (cdr exp) env1))))))  ;再对参数求值
        (else
         (error "Unknown expression type -- EVAL" exp))))

(define (apply1 procedure arguments)
;(display "Expression: ") (write procedure) (newline) (newline)
                                        ;(display "Arguments: ") (write arguments) (newline) (newline)
  (cond ((primitive-procedure? procedure)
         (apply-primitive-procedure procedure arguments))
        ;;The Closure like this:
        ;;  '(procedure (m n) (+ m n) env)
        ((compound-procedure? procedure)
         (eval-sequence
          ;(procedure-body procedure)
          (caddr procedure)  ;(+ m n)
          (extend-environment
           (cadr procedure) ;(m n)
           arguments        ;实参，对应 m 和 n
           (cadddr procedure)))) ;闭包里的env
        (else
         (error
          "Unknown procedure type -- APPLY" procedure))))

;;;;;;;;;;;;;;;;;;;;;;;;;;;自求值;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (self-evaluating? exp)
  (cond ((number? exp) #t)
        ((string? exp) #t)
        ((boolean? exp) #t)
        (else #f)))

;;;;;;;;;;;;;;;;;;;;;;;;;;;变量;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (variable? exp) (symbol? exp))

(define (lookup-variable-value exp env)
  (hash-ref env exp #f))

;;;;;;;;;;;;;;;;;;;;;;;;; quote ;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (quoted? exp)
  (tagged-list? exp 'quote))

;;;;;;;;;;;;;;;;;;;;;;;;;; set! ;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (assignment? exp)
  (tagged-list? exp 'set!))

(define (eval-assignment exp env)
  (hash-set! env
             (cadr exp)
             (eval0 (caddr exp)))
  'ok)

;;;;;;;;;;;;;;;;;;;;;;;;;; let ;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (let? exp)
  (tagged-list? exp 'let))

(define (eval-let exp env)
  (let ((env1 (hash-copy env)))
    (if (pair? (cadr exp))
        (eval0 (cons (list 'lambda
                           (map car (cadr exp))
                           (caddr exp))
                     (map cadr (cadr exp)))
        env1)

        (begin
          (hash-set! env1
                     (cadr exp)
                     (make-lambda (map car (caddr exp))
                                  (cdddr exp)))
          (eval0 (cons (cadr exp)
                       (map cadr (caddr exp))) 
                 env1)))))

;;;;;;;;;;;;;;;;;;;;;;;;;; define ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (definition? exp)
  (tagged-list? exp 'define))

(define (eval-definition exp env)
  (hash-set! env
             (definition-variable exp)
             (eval0 (definition-value exp) env))
    'ok)

(define (definition-variable exp)
  (if (symbol? (cadr exp))  ;;(define var ...)
      (cadr exp)
      (caadr exp)))         ;;(define (fun arg) ...)

(define (definition-value exp)
  (if (symbol? (cadr exp))  ;;cadr是符号，对应上一种情况,它可以是变量定义
      (caddr exp)           ;;也可以是 (define fun (lambda (arg)...)
      (make-lambda (cdadr exp)   ;;对应 (define (fun arg) ...)
                   (cddr exp)))) ;;生成一个 lambda 来代替
      ;把 (define (fun arg) ...) 转换成一个 lambda 表达式

(define (make-lambda parameters body)
  (cons 'lambda (cons parameters body)))

;;;;;;;;;;;;;;;;;;;;;;;;;; if ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (if? exp) (tagged-list? exp 'if))

(define (eval-if exp env)
  (if (true? (eval0 (cadr exp) env))
      (eval0 (caddr exp) env)
      (eval0 (if-else exp) env)))

(define (if-else exp)
  (if (not (null? (cdddr exp)))
      (cadddr exp)
      '#f))

;;;;;;;;;;;;;;;;;;;;;;;;; lambda ;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (lambda? exp) (tagged-list? exp 'lambda))

;; '(procedure (m n) (+ m n) env)
(define (make-procedure parameters body env)
  (list 'procedure parameters body env))

;;;;;;;;;;;;;;;;;;;;;;;;;; begin ;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (begin? exp) (tagged-list? exp 'begin))

(define (eval-sequence exps env)
  (cond ((last-exp? exps) (eval0 (car exps) env))
        (else (eval0 (car exps) env)
              (eval-sequence (cdr exps) env))))

(define (last-exp? seq) (null? (cdr seq)))

;;;;;;;;;;;;;;;;;;;;;;;;;; cond ;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (cond? exp) (tagged-list? exp 'cond))

(define (cond-else-clause? clause)
  (eq? (car clause) 'else))

(define (cond->if exp)
  (expand-clauses (cdr exp)))

(define (expand-clauses clauses)
  (if (null? clauses)
      '#f                          ; no else clause
      (let ((first (car clauses))
            (rest (cdr clauses)))
        (if (cond-else-clause? first)
            (if (null? rest)
                (sequence->exp (cdr first))
                (error "ELSE clause isn't last -- COND->IF"
                       clauses))
            (make-if (car first)
                     (sequence->exp (cdr first))
                     (expand-clauses rest))))))

(define (make-if predicate consequent alternative)
  (list 'if predicate consequent alternative))

(define (sequence->exp seq)
  (cond ((null? seq) seq)
        ((last-exp? seq) (car seq))
        (else (make-begin seq))))

(define (make-begin seq) (cons 'begin seq))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;逻辑;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (eval-or exp env)
  (let iter ((tests (cdr exp)))
    (cond
     ((null? tests) #f)
     ((eq? (eval0 (car tests) env) #t) #t)
     (else (iter (cdr tests))))))

(define (eval-and exp env)
  (let iter ((tests (cdr exp)))
    (cond
     ((null? tests) #t)
     ((eq? (eval0 (car tests) env) #f) #f)
     (else (iter (cdr tests))))))

(define (eval-not exp env)
  (if (eval0 (cadr exp) env)
      #f
      #t))

;;;;;;;;;;;;;;;;;;;;;;;;;; application ;;;;;;;;;;;;;;;;;;;;;;;;

(define (application? exp) (pair? exp))

(define (list-of-values exps env)
  (if (null? exps)
      '()
      (cons (eval0 (car exps) env)
            (list-of-values (cdr exps) env))))

(define (primitive-procedure? proc)
  (tagged-list? proc 'primitive))

(define (apply-primitive-procedure proc args)
  (apply
   (cadr proc) args))

(define (compound-procedure? p)
  (tagged-list? p 'procedure))

(define (extend-environment vars vals base-env)  ;;扩展环境
  (if (= (length vars) (length vals))
      (loop-add-var vars vals base-env)
      (if (< (length vars) (length vals))
          (error "Too many arguments supplied" vars vals)
          (error "Too few arguments supplied" vars vals))))

(define (make-frame variables values)
  (cons variables values))


;;;;;;;;;;;;;;;;;;;;;;;;;; init env ;;;;;;;;;;;;;;;;;;;;;;;;

(define env0 (make-hash))



(define (loop-add-var vars vals table)
  (cond ((null? vars) table)
        (else
         (and (hash-set! table (car vars) (car vals))
              (loop-add-var (cdr vars) (cdr vals) table)))))

(define (setup-environment)
  (let ((initial-env
         (extend-environment (primitive-procedure-names)
                             (primitive-procedure-objects)
                             env0)))
    (hash-set! initial-env '#t #t)
    (hash-set! initial-env '#f #f)
    initial-env))

(define primitive-procedures
  (list (list 'car car)
        (list 'cdr cdr)
        (list 'cons cons)
        (list 'null? null?)
        (list '+ +)
        (list '- -)
        (list '* *)
        (list '/ /)
        (list '> >)
        (list '< <)
        (list '= =)
        (list 'eq? eq?)
;;      more primitives
        ))

(define (primitive-procedure-names)
  (map car
       primitive-procedures))

(define (primitive-procedure-objects)
  (map (lambda (proc) (list 'primitive (cadr proc)))
       primitive-procedures))

(define the-global-environment (setup-environment))

(let ((ip (open-input-file "init.scm")))
  (let iter ((exp (read ip)))
    (if (eof-object? exp)
        (close-input-port ip)
        (begin
          (eval0 exp the-global-environment)
          (iter (read ip))))))

;;;;;;;;;;;;;;;;;;;;;;;;;;; repl ;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (driver-loop)
  (display "sicp> ")
  (let ((input (read)))
    (if (eq? input 'exit)
        (display "Bye!\n")
        (begin
          (let ((output (eval0 input the-global-environment)))
            (user-print output)
            (newline))
          (driver-loop)))))

(define (user-print object)
  (if (compound-procedure? object)
      (display (list 'compound-procedure
                     (cadr object)
                     (caddr object)
                     '<procedure-env>))
      (display object)))

(driver-loop)
```

初始化文件（init.scm）：

```scheme
(define (caaaar x) (car (car (car (car x)))))
(define (caaadr x) (car (car (car (cdr x)))))
(define (caadar x) (car (car (cdr (car x)))))
(define (caaddr x) (car (car (cdr (cdr x)))))
(define (cadaar x) (car (cdr (car (car x)))))
(define (cadadr x) (car (cdr (car (cdr x)))))
(define (caddar x) (car (cdr (cdr (car x)))))
(define (cadddr x) (car (cdr (cdr (cdr x)))))
(define (cdaaar x) (cdr (car (car (car x)))))
(define (cdaadr x) (cdr (car (car (cdr x)))))
(define (cdadar x) (cdr (car (cdr (car x)))))
(define (cdaddr x) (cdr (car (cdr (cdr x)))))
(define (cddaar x) (cdr (cdr (car (car x)))))
(define (cddadr x) (cdr (cdr (car (cdr x)))))
(define (cdddar x) (cdr (cdr (cdr (car x)))))
(define (cddddr x) (cdr (cdr (cdr (cdr x)))))

(define (caaar x) (car (car (car x))))
(define (caadr x) (car (car (cdr x))))
(define (cadar x) (car (cdr (car x))))
(define (caddr x) (car (cdr (cdr x))))
(define (cdaar x) (cdr (car (car x))))
(define (cdadr x) (cdr (car (cdr x))))
(define (cddar x) (cdr (cdr (car x))))
(define (cdddr x) (cdr (cdr (cdr x))))

(define (caar x) (car (car x)))
(define (cadr x) (car (cdr x)))
(define (cdar x) (cdr (car x)))
(define (cddr x) (cdr (cdr x)))

(define zero?
  (lambda (n)
    (= n 0)))

(define null?
  (lambda (lst)
    (eq? lst '())))

(define length
  (lambda (lst)
    (cond ((null? lst) 0)
          (else (+ 1 (length (cdr lst)))))))

(define member
  (lambda (a lst)
    (cond ((null? lst) '())
          ((eq? a (car lst)) lst)
          (else (member a (cdr lst))))))
```