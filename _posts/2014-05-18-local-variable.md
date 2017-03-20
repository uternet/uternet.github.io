---
layout: post
title: Scheme 中的局部变量
date: 2014-05-18
---

Scheme 有三种方法创建局部变量：let let* letrec

最常用的是 let：

    (let ((var1 10)
          (var2 20))
      (+ var1 var2))

每个 let 表达式包含三个部分, 最前面是关键字 let，第二部分是变量定义，每一个子句定义一个变量，相当于一个内嵌的 define。就算只有一个子句，也应当将它包含在括号内部。

let* 的具体区别搞不清楚，这是wiki的示例：

    (let* ((var1 10)
           (var2 (+ var1 5)))
       var2)

在定义部分，前面定义的值可以被后面的子句引用，把这里的let* 换成 let就不行了，它的定义只能在 body 中使用。

letrec 所定义的变量可以交叉引用，通常被用于双重递归。

```
(letrec ((female (lambda(n)
                   (if (= n 0) 1
                       (- n (male (female (- n 1)))))))
         (male (lambda(n)
                 (if (= n 0) 0
                     (- n (female (male (- n 1))))))))
  (display "i male(i) female(i)")(newline)
  (do ((i 0 (+ i 1)))
      ((> i 8) #f)
    (display i) (display "   ")
    (display (male i))
    (display "         ")
    (display (female i))
    (newline)))
```

上面的程序输出的是侯世达阴阳数列：

```
i male(i) female(i)
0   0          1
1   0          1
2   1          2
3   2          2
4   2          3
5   3          3
6   4          4
7   4          5
8   5          5
#f
```