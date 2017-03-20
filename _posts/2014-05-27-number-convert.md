---
layout: post
title: Scheme 实现的数字进制转换
date: 2014-05-27
---

我已经用 shell 脚本写过一个了，调用 linux 命令行下的计算器 bc实现：

```bash
#!/bin/bash

if [ $# -lt 1 ]; then
    echo "用法： 本命令 [选项] 数字"
    echo -e "    选项用于指定输入的数制（10进制数可省略）"
    echo -e "选项："
    echo -e "\t-d\t指定输入的是10进制数(可选)"
    echo -e "\t-o\t指定输入的是8进制数"
    echo -e "\t-x\t指定输入的是16进制数"
    echo -e "\t-b\t指定输入的是2进制数"
    exit 1
fi

if [ $# -lt 2 ];then
    ibase=10
    num=$1

elif [ "$1" = -d ]; then
    ibase=10
    num=$2
    
elif [ "$1" = -b ]; then
    ibase=2
    num=$2
    
elif [ "$1" = -o ]; then
    ibase=8
    num=$2
    
elif [ "$1" = -x ]; then
    ibase=16
    num=$2
fi

echo -e "0x`echo "obase=16;ibase=${ibase};${num}"|bc`\t\c"
echo -e "`echo "obase=10;ibase=${ibase};${num}"|bc`\t\c" 
echo -e "0`echo "obase=8;ibase=${ibase};${num}"|bc`\t\c"
echo "obase=2;ibase=${ibase};${num}"|bc
exit 0
```

今天突然想用Scheme再写一个，可是一涉及到输出就比较头疼了。r5rs 标准里定义的输出函数只有两个：display 和 write，这是远远不够用的。标准外的SRFI倒是定义了一些类似于其它语言的输出函数，比如模仿 Lisp 的 format 函数，但那是可选的，并非每一个 Scheme 实现都支持。事实上几乎每一个成熟的 Scheme 都会提供标准之外的扩展，但都是以模块的方式实现的。然而，每一个实现调用模块的方式又不一样。于是，想写出能在大多数实现上运行的代码是几乎不可能的。Scheme 大概是可移植性最差的高级语言了。

没办法的办法是指定要运行的 Scheme 实现。

```scheme
#!/usr/bin/scheme-script
(import (ikarus))

(define (getnum arg-list)
  (cond
   ((member "-b" arg-list)
    (string->number (car (cdr (member "-b" arg-list))) 2))
   ((member "-o" arg-list)
    (string->number (car (cdr (member "-o" arg-list))) 8))
   ((member "-x" arg-list)
    (string->number (car (cdr (member "-x" arg-list))) 16))
   ((member "-d" arg-list)
    (string->number (car (cdr (member "-d" arg-list))) 10))
   (else (string->number (car (cdr arg-list)) 10))))
   
   
(let ((arg-list (command-line)))
  (cond
   ((< (length arg-list) 2)
    (display "Need a argument!\n"))
   (else
    (let ((num (getnum arg-list)))
      (printf "0x~a\t" (number->string num 16))
      (printf "~a\t" (number->string num 10))
      (printf "0~a\t" (number->string num 8))
      (printf "~a\n" (number->string num 2))))))
```

我把开头写成`#!/usr/bin/scheme-script` 而不是 `#!/usr/bin/env scheme-script` 的原因是，我的机器上有两个地方能找到 scheme-script 这个程序，位于 /usr/bin 目录下的这个，是 ikarus 提供的，另外一个位于 ~/bin/larceny 目录下，是由 Larceny 提供的。好像 r6rs 标准规定，每个实现应当提供一个叫做 scheme-script 的程序来运行脚本。连可执行文件名都规定好了，hoho！


使用方法：

    convert.ss -b 10011
    => 0x13    19      023     10011
    
    convert.ss -o 253
    => 0xAB    171     0253    10101011
    
    convert.ss -x ff
    => 0xFF    255     0377    11111111
    
    convert.ss -d 123
    => 0x7B    123     0173    1111011    
    convert.ss 123
    => 0x7B    123     0173    1111011 ;;转换十进制也可以不加参数

事实上，Scheme 的 string->number 函数可以判断参数的数制，前提是你传给它一个合法的常数，比如：#x32f。`#`号在 shell 里有别的用途，所以要加转义符。

    convert.ss \#x32f
    => 0x32F   815     01457   1100101111
    
    convert.ss \#b110110
    => 0x36    54      066     110110
