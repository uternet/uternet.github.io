---
layout: post
title: Racket 里的迭代
date: 2014-12-20
---

只知道 Scheme 里有个不太好用的 do 循环，一直不知道 Racket 里竟然还有 for 循环，而且还有好几种。

最简单的 for 循环可以这样写：

```scheme
> (for ((i 10))
       (display (* i i)))
       
=> 0149162536496481
```

##生成序列的函数

###in-range

Racket 里有一个生成序列的函数 in-range，in-range 的返回值是一种叫做“流”的数据结构：

```scheme
> (in-range 10)
=> #<stream>
```

把 in-range 用于 for 循环中是这样子的：

```scheme
> (for ((i (in-range 10)))
       (if (= i (- 10 1))
           (printf "~a\n" i)
           (printf "~a " i)))
=> 0 1 2 3 4 5 6 7 8 9
```

可以发现，in-range 生成了一个以0开始，以传参数为止的序列（并不包括参数本身）。和 Python 的 `for i in range(10)` 基本一样。

in-range 的第二个和第三个参数是可选的，如果指定两个参数，那么第一个参数指定开始的数字，第二个参数指定结尾的数字（这个数字并不包括在数列里面）

```scheme
> (for ((i (in-range 3 10)))
       (if (= i (- 10 1))
           (printf "~a\n" i)
           (printf "~a " i)))
=> 3 4 5 6 7 8 9
```

如果给 in-range 传递3个参数，那第3个参数表示的是步进：

```scheme
> (for ((i (in-range 3 11 2)))
       (if (>= i (- 11 2))
           (printf "~a\n" i)
           (printf "~a " i)))
=> 3 5 7 9
```

序列甚至可以倒着走：

```scheme
> (for ([i (in-range 4 1 -1)])
    (display i))
=> 432
```

步进不一定是整数，可以是小数或分数：

```scheme
> (for ((i (in-range 1 4 0.5)))
       (printf " ~a " i))
=> 1  1.5  2.0  2.5  3.0  3.5
```

###in-naturals

Racket 中还有一个类似于 in-range 的函数 in-naturals，起点总是从1开始，步进始终是1,而且没有上限，需要在循环体中检测停止点，不然就死循环了。不知这个函数用在什么地方。

###stop-before && stop-after

提供一个序列和一个过程作为参数，返回一个新序列。

```scheme
(for ((i (stop-before "abc def"
                        char-whitespace?)))
       (display i))
=> abc
```

传入的 char-whitespace? 过程检测空格，如果检测到空格则截断。stop-before 和 stop-after 这两者的区别就是名字的区别：前者在空格之前截断，而后者在空格之后截断（包括空格）。

###in-list, in-vector, in-string

分别在列表、矢量、字符串上进行迭代。数列、列表、矢量、字符串都是天然的序列，有趣的是，Racket 可以不需要 in-range、in-string、in-list 等函数显式地生成一个流，以上数据类型可直接用于 for 结构中：

```scheme
> (for ((i '((a 1) (b 2) (c 3))))
       (let ((id (car i))
             (val (cadr i)))
         (printf "~a: ~a\n" id val)))
=> a: 1
=> b: 2
=> c: 3
```

##for 和 for*

for 可以同时在多个序列上并行迭代：

```scheme
> (for ([i (in-range 1 4)]
        [chapter '("Intro" "Details" "Conclusion")])
    (printf "Chapter ~a. ~a\n" i chapter))
=> Chapter 1. Intro
=> Chapter 2. Details
=> Chapter 3. Conclusion
```

当有多个序列并行迭代时，如果序列长度不一致，只要到达其中一个序列末尾迭代就结束了。可以利用这个特性，用 in-naturals 生成一个无限的序列，反正别的序列结束时循环就结束了，不会陷入死循环。

for* 循环与 for 循环具有相同的语法，不同的是，当有多个序列时，它不是并行迭代，而是嵌套循环：

```scheme
> (for* ([book '("Guide" "Reference")]
         [chapter '("Intro" "Details" "Conclusion")])
    (printf "~a ~a\n" book chapter))
=> Guide Intro
=> Guide Details
=> Guide Conclusion
=> Reference Intro
=> Reference Details
=> Reference Conclusion
```

for* 循环是嵌套的 for 结构的一种简单写法，就象 let* 之于嵌套的 let.

##for/list 和 for*/list

类似于 for 循环，不同的是循环结束后生成并返回一个新列表。

```scheme
> (for/list ([i (in-naturals 1)]
             [chapter '("Intro" "Details" "Conclusion")])
    (string-append (number->string i) ". " chapter))
=> '("1. Intro" "2. Details" "3. Conclusion")
```

##for/vector 和 for*/vector

与 for/list 类似，但是返回一个矢量。

##for/and 和 for/or

对迭代过程中的中间结果分别执行 and 或者 or 求值，并返回一个布尔值。

##for/first 和 for/last

分别返回第一次迭代和最后一次迭代的结果，如果迭代没有执行（body没有求值）, 则返回 #f。

##for/fold 和 for*/fold

暂时没弄懂。