---
layout: post
title: scheme 中的位运算与二进制支持
date: 2014-05-26
---

按位与 bitwise-and  相当于C语言的 &

    (bitwise-and 240 43)
    => 32

按位或 bitwise-ior  相当于C语言的 |

按位非 bitwise-not  相当于C语言的 ～

按位异或 bitwise-xor  相当于C语言的 ^

    (bitwise-xor 1 1)
    => 0
    (bitwise-xor 1 0)
    => 1

左移位 (arithmetic-shift 数字 移动位数)

右移位 (arithmetic-shift 数字 负的移动位数)

    (arithmetic-shift 12 -2)
    => 3

类似地，Lisp 中也有 logand, logor等函数。

scheme 可以直接使用各个数制的常数，比如：

\#b11011010 二进制  
\#o231563   八进制  
\#x23a8f    十六进制  
\#d3115     默认十进制，前缀可省略

输出,可以用 number->string 直接指定数制 2 8 10 16

    (number->string 1024 2)
    => "10000000000"
    
    (number->string 1024 16)
    => "400"

当然，转换也可以逆过来：

    (string->number "10" 2)
    => 2
    (string->number "FFFF" 16)
    => 65535