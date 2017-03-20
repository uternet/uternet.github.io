---
layout: post
title: Y-Combinator推导
date: 2014-04-27
---

卡这里好几天了，杂务又多，今天抽空把TLS第9章的代码敲了一遍，MS有一点点理解了

定义一个盲肠函数，这是一个无限递归函数，永远也不会结束

```scheme
(define eternity
  (lambda (x)
    (eternity x)))
```

一个匿名函数，只能求出空列表的长度：0  
如果列表非空，则(eternity (cdr l))被触发

```scheme
((lambda (l)
   (cond
    ((null? l) 0)
    (else (add1 (eternity (cdr l)))))))
```

一个匿名函数，只能求出长度小于等于1的列表  
如果列表元素超过1个，(eternity (cdr l))被触发

```scheme
(lambda (l)
  (cond
   ((null? l) 0)
   (else (add1 ((lambda (l)
                  (cond
                   ((null? l) 0)
                   (else (add1 (eternity (cdr l))))))
                (cdr l))))))
```

与上面类似，这个匿名函数的参数不能超过2个（可以是0个、1个、2个）

```scheme
(lambda (l)
  (cond
   ((null? l) 0)
   (else (add1
          ((lambda (l)
             (cond
              ((null? l) 0)
              (else (add1
                     ((lambda (l)
                        (cond
                         ((null? l) 0)
                         (else (add1 (eternity (cdr l))))))
                      (cdr l))))))
           (cdr l))))))
```

将上面的匿名函数变换一下   
那个盲肠函数作为参数被传递给了内层的匿名函数，用于处理(cdr l)   
但是实际上它永远也不能运行，不然程序就挂了。

```scheme
((lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l)))))))
 eternity)
```

用同样的办法可以写出嵌套的匿名函数，用于处理长度小于、等于1的列表  
第一个匿名函数参数f的实参是第二个匿名函数  
而第二个匿名函数参数g的实参则是那个无用的函数eternity

```scheme
((lambda (f)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (f (cdr l)))))))
 ((lambda (g)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else (add1 (g (cdr l)))))))
  eternity))
```

再变换一下,第2个匿名函数作为参数传递给最外层的匿名函数，形参为 mk-length  
在最外层的函数内部，它获得了一个参数传入，形参为 length 实参数为 eternity  
就是那个无限递归函数  
这一整个函数可以处理空列表，展开后和最初的版本是一样的

```scheme
((lambda (mk-length)
   (mk-length eternity))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (length (cdr l))))))))
```

要处理更大长度的列表，多次调用就行了  
这次可以处理长度不大于1的列表

```scheme
((lambda (mk-length)
   (mk-length
    (mk-length eternity)))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (length (cdr l))))))))
```

现在可以处理长度不大于2的列表

```scheme
((lambda (mk-length)
   (mk-length
    (mk-length
     (mk-length eternity))))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (length (cdr l))))))))
```

从length0开始动手，切掉盲肠  
将一个匿名函数作为参数传递给另一个匿名函数，并且在调用函数内部再调用它自己  
`(mk-length mk-length)`在这里造成了混乱，有点难以理解。。。  
看起来这里已经实现了递归，可实际上还没有。  
必须结合上面的`(mk-length eternity)`来理解  
在上面的函数中，eternity是在最后执行的（实际上不能执行），所以，这里的   
`(mk-length mk-lenght)`并不重要，它先处理空列表，如果列表非空，才会触发下面的  
`(add1 (length (cdr l)))`  
在这里length就是该匿名函数自身，它定义了两个参数，一个过程length，还有一个列表l  
`(add1 (length (cdr l)))`只传递了一个参数l，缺了一个参数，于是结果就不对了，  
所以，此函数仍然只能处理空列表   

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1
             (length (cdr l))))))))
```

与上面的函数相比，这里只是改变了形参的名字，事实上是一样的。

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1
             (mk-length (cdr l))))))))
```

为什么盲肠又出现了？  
这是 length=<1 函数

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1
             ((mk-length eternity)
              (cdr l))))))))
```

**真正递归的匿名函数实现了**  
现在可以成功求出任意列表的长度了，下面的推导将以这个函数为出发点

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1
             ((mk-length mk-length)
              (cdr l))))))))
```

把(mk-length mk-length)抽取出来，作为参数再传递给第三层匿名函数   
可是为什么我在racket上调不通过？（搞半天原来这个函数是错的）

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   ((lambda (length)
      (lambda (l)
        (cond
         ((null? l) 0)
         (else (add1 (length (cdr l)))))))
    (mk-length mk-length))))

;;展开如下
((lambda (mk-length)
   ((lambda (length)
      (lambda (l)
        (cond
         ((null? l) 0)
         (else (add1 (length (cdr l)))))))
    (mk-length mk-length)))
 (lambda (mk-length)
   ((lambda (length)
      (lambda (l)
        (cond
         ((null? l) 0)
         (else (add1 (length (cdr l)))))))
    (mk-length mk-length))))
```

上面一大段白费时间，我不明白作者写它作什么。   
下面都是正确的函数，与最初那个实现递归的函数不同在于

    (add1
     ((lambda (x)
        ((mk-length mk-length) x))
      (cdr l)))
      
替换了

    (add1
     ((mk-length mk-length) (cdr l)))
     
两者是等价的，只不过单独抽象出一个匿名函数

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1
             ((lambda (x)
                ((mk-length mk-length) x)) (cdr l))))))))
```

将抽象出来的(lambda (x))放到外层，作为参数length传入

```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   ((lambda (length)
      (lambda (l)
        (cond
         ((null? l) 0)
         (else (add1
                (length (cdr l)))))))
    (lambda (x)
      ((mk-length mk-length) x)))))
```

令人眼花缭乱的变换

```scheme
((lambda (le)
   ((lambda (mk-length)
      (mk-length mk-length))
    (lambda (mk-length)
      (le (lambda (x)
            ((mk-length mk-length) x))))))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (length (cdr l))))))))
```

把上面的部分单独拿出来，这就是 Y-combinator  
而下半部分就是匿名的length函数，它需要借助Y-combinator来实现递归.

```scheme
(lambda (le)
  ((lambda (mk-length)
     (mk-length mk-length))
   (lambda (mk-length)
     (le (lambda (x)
           ((mk-length mk-length) x))))))
```

把参数名改一下,取个名字 Y

```scheme
(define Y
  (lambda (le)
    ((lambda (f) (f f))
     (lambda (f)
       (le (lambda (x) ((f f) x)))))))
```

OK，把length也define一个名字

```scheme
(define length
  (lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else (add1 (length (cdr l))))))))
```

调用

    ((Y length) '(a b c d e))
    => 5


至此，推是推导出来了，还是觉得理解不清晰，脑子里还是一团浆糊，帽子依然不够大。

再把玩一下这个Y

```scheme
((Y (lambda (f)
      (lambda (n)
        (cond
         ((zero? n) 1)
         (else (* n (f (sub1 n))))))))
 4)
=> 24
```