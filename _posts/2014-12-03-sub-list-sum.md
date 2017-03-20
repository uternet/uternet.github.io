---
layout: post
title: 求子列表最大值
date: 2014-12-03
---

这是浙大 PAT 上的题：

>给定K个整数组成的序列{ N1, N2, ..., NK }，“连续子列”被定义为{ Ni, Ni+1, ..., Nj }，其中 1 <= i <= j <= K。“最大子列和”则被定义为所有连续子列元素的和中最大者。例如给定序列{ -2, 11, -4, 13, -5, -2 }，其连续子列{ 11, -4, 13 }有最大的和20。现要求你编写程序，计算给定整数序列的最大子列和。
>
>输入格式：
>
>输入第1行给出正整数 K (<= 100000)；第2行给出K个整数，其间以空格分隔。
>
>输出格式：
>
>在一行中输出最大子列和。如果序列中所有整数皆为负数，则输出0。
>
>输入样例：   
>6   
>-2 11 -4 13 -5 -2   
>输出样例：   
>20   

幸好陈姥姥在视频里把四种算法都讲了，我就抄了个最快的算法，用 Scheme 实现一下，PASS！

```scheme
(define (sub-list-sum lst)
  (let iter ((lst lst) (max 0) (current 0))
    (cond ((null? lst) max)
          (else
           (let ((tmp (+ current (car lst))))
             (cond ((> tmp max)
                    (iter (cdr lst) tmp tmp))
                   ((< tmp 0)
                    (iter (cdr lst) max 0))
                   (else
                    (iter (cdr lst) max tmp))))))))
```

再看第二题，在原来的要求上加了点东西，不但要输入出最大子列和，还要输出该子列的头尾成员。

这题目是考研题，还是全英文的，好家伙！

>Given a sequence of K integers { N1, N2, ..., NK }. A continuous subsequence is defined to be { Ni, Ni+1, ..., Nj } where 1 <= i <= j <= K. The Maximum Subsequence is the continuous subsequence which has the largest sum of its elements. For example, given sequence { -2, 11, -4, 13, -5, -2 }, its maximum subsequence is { 11, -4, 13 } with the largest sum being 20.
>
>Now you are supposed to find the largest sum, together with the first and the last numbers of the maximum subsequence.
>
>Input Specification:
>
>Each input file contains one test case. Each case occupies two lines. The first line contains a positive integer K (<= 10000). The second line contains K numbers, separated by a space.
>
>Output Specification:
>
>For each test case, output in one line the largest sum, together with the first and the last numbers of the maximum subsequence. The numbers must be separated by one space, but there must be no extra space at the end of a line. In case that the maximum subsequence is not unique, output the one with the smallest indices i and j (as shown by the sample case). If all the K numbers are negative, then its maximum sum is defined to be 0, and you are supposed to output the first and the last numbers of the whole sequence.
>
>Sample Input:   
>10   
>-10 1 2 3 4 -5 -23 3 7 -21   
>Sample Output:   
>10 1 4

这题难住了老夫大约两天的时间，到底要怎么记录子列的头尾呢？

最后我用了折衷的做法，当前子列和大于已知的最大子列和的时候，同时更新最大子列和，以及最大子列合的“尾”，至于头，先不管它了，等数据遍历完了，再通过尾部数据去回溯找出来头部。

看起来，在记录子列尾的时候，要记录的不是实际的数据，而是索引位。因为一个数列中可能存在相同的数据，而它们的位置肯定不同。只有通过唯一的索引位去定位尾部，才有可能反推出头部。

可是我用的是 Scheme 呃，看来只能把列表当成数组去用了。幸好 Scheme 里有个 list-ref 过程，可以当成数组下标来用。

```scheme
(define (find-start sum end lst)
  (let iter ((idx end) (sum1 (list-ref lst end)))
    (if (= sum1 sum)
        idx
        (iter (- idx 1) (+ sum1 (list-ref lst (- idx 1)))))))

(define (sub-list-sum lst len)
  (let iter ((idx 0) (max 0) (current 0) (end -1)) ;这里故意把 end 设为负数
    (if (>= idx len)
        (cons max end)
        (let ((tmp (+ current (list-ref lst idx))))
          (cond ((> tmp max)
                 (iter (+ idx 1) tmp tmp idx))
                ((and (= tmp max 0) ;如果找到一个零
                      (= end -1))   ;并且 max 之前从未更新过，零也算是找到的最大数
                 (iter (+ idx 1) tmp tmp idx))
                ((< tmp 0)
                 (iter (+ idx 1) max 0 end))
                (else
                 (iter (+ idx 1) max tmp end)))))))

(define (read-list n)
  (let iter ((step 1) (ret '()))
    (if (> step n)
        (reverse ret) ;cons 进去的列表，所以要反转一下顺序才对
        (let ((n (read)))
          (iter (+ step 1) (cons n ret))))))

(let ((k (read)))
  (let ((lst (read-list k)))
    (let ((max-and-end (sub-list-sum lst k)))
      (let* ((max (car max-and-end))
             (end
              (if (< (cdr max-and-end) 0)
                  (- k 1)
                  (cdr max-and-end)))
             (start
              (if (< (cdr max-and-end) 0)
                  0
                  (find-start max end lst))))
        (printf "~a ~a ~a\n"
                max
                (list-ref lst start)
                (list-ref lst end))))))

```

这题有几个坑，我首先掉下去的坑是，我忘记了题目要求输出最大子列的头尾“值”，而是直接输出了最大子列和的值以及它的头尾“下标”。除了最后一个测试点外，全错！

为什么最后一个测试点会对呢？因为它的测试数据就是题目里面作为示例的：

    [-10 1 2 3 4 -5 -23 3 7 -21]

它的最大子列是 `[1 2 3 4]`，后面的 `[3 7]` 也是最大字列，但是题目要求相同的子列只输出第一个，所以不管它了。而 `[1 2 3 4]` 这个子列的头尾是 1 和 4, 它们的字面值正好等于它在整个数组里面的下标。

我掉进这个坑里好久没爬出来。

第二个坑是测试数据里的零，按照约定，当前子列和已经小于 0 的时候，重新归 0, 再开始累加。

刚开始的时候我是这样写的：

    (let iter ((idx 0) (max 0) (current 0) (end 0))
    
把 max current end 都初始化成 0.其中，max current 都是累加值，而 end 是下标。这么干就没法分辨这种情况了：函数返回的时候，如果它们还是 0, 就没办法知道到底数组里都是负数？还是遇到了0？特别地，如果测试数组开头第一个就是个0,余下的全是负数。

最后我改成了这样：

    (let iter ((idx 0) (max 0) (current 0) (end -1))

把 end 初始化成并不存在的数组下标 -1. 如果函数返回的时候，这个值变了，说明找到了最大子列，如果前两者还是 0,说明最大子列就是个单独的 0；反之，如果这个值没变，说明测试数组里全是负数。这时候，按照题目要求，应当输出整个测试数组的头和尾。（我苦逼的英语水平居然看懂了这个题目要求，真是幸运！）