---
layout: post
title: 被The Little Schemer 第八章的 multirember&co 噎住了
date: 2014-04-15
---
经过前面的递归思维训练，我觉得前面7章都没什么难度嘛，兀地看到了这个函数，一下子就凌乱了，人肉解释器有些吃不消。

```scheme
(define multirember&co
  (lambda (a lat col)
    (cond
     ((null? lat)
      (col (quote ()) (quote ())))
     ((eq? (car lat) a)
      (multirember&co a (cdr lat)
                      (lambda (newlat seen)
                        (col newlat
                             (cons (car lat) seen)))))
     (else
      (multirember&co a (cdr lat)
                      (lambda (newlat seen)
                        (col (cons (car lat) newlat)
                             seen)))))))

(define a-friend
  (lambda (x y)
    (null? y)))
```        

我知道这里边用到的概念是continuation，我很早就知道这是scheme中一个很重要的概念，但是真的遇见了，才知道很难懂。

我觉得这个函数的精密程度就象个闹钟，这让我想起了小时候因为好奇而拆开闹钟，由于无法装回去导致的惶恐与不安。

不论如何，还是要搞懂的！

OK! 来给它一些输入，看看发生了什么？

假设lat = ()，那么cond分支第一个就是出口，不论参数 a 的值是什么，空列表都被传递给a-friend函数，得到#t，程序运行完毕，退出。

假设传递的参数是这样：

a = 'tuna  
lat = '(and tuna)

情况变得复杂了。。。

下面是代入具体参数后的推导：

```scheme
(multirember&co 'tuna '(and tuna) a-friend)


(multirember&co 'tuna '(tuna)
                (lambda (newlat seen)
                  (a-friend (cons 'and newlat) seen)))


(multirember&co 'tuna '()
                (lambda (newlat seen)
                  ((lambda (newlat seen)
                     (a-friend (cons 'and newlat) seen))
                   newlat (cons 'tuna seen))))


((lambda (newlat seen)
   ((lambda (newlat seen)
      (a-friend (cons 'and newlat) seen))
    newlat (cons 'tuna seen)))
 '() '())


((lambda (newlat seen)
   (a-friend (cons 'and newlat) seen))
 '() '(tuna))


(a-friend '(and) '(tuna))
```

我花了大概一整天来钻牛角尖，搞到最后大脑已经有些要罢工的感觉。晚上的时候，有那么几分钟的时间思维稍稍清晰，我便一边在emacs中调试一边把式子推导出来了，从最开始的函数调用  
`(multirember&co 'tuna '(and tuna) a-friend)`  
得出  
`(a-friend '(and) '(tuna))`

最后得出结果： #f

相当于在大脑中按单步调试把程序跑了一遍，等我再回过头去看推导的步骤，我发现大脑又罢工了，完全集中不起注意力来。

好吧，去睡吧，希望明天思维会清晰起来。