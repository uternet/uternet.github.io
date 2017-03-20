---
layout: post
title: "发现一个好玩的scheme编译器"
date:  2014-03-14
---
URScheme，是一个可以将scheme代码编译成原生机器码的编译器。准确地说它是将scheme代码先编译成x86汇编代码，与SBCL不同的是，接下来它调用GNU汇编器和链接器再将汇编代码弄成ELF可执行文件，这让我不由得想起了传说中的chez scheme编译器。

这个项目只是一个练习写作编译器的作品，只实现了R5RS的一个小子集，很多特性仍然缺失中。

项目主页：<http://www.canonical.org/~kragen/sw/urscheme/>

编译之前，需要先有一个支持R5RS的第三方实现来帮助实现自举，几个成熟的实现都可以充当这个角色：chicken, gambit-c, guile, mzscheme等等，默认使用guile，如果用户的机器上使用别的实现，需要先编辑Makefile，然后再使用make命令编译。

事实上运行单独的`make`命令会在编译完成后执行一系列的测试，我研究了一下它的Makefile，compiler.scm就是编译器源码，可以用这样的命令编译：

    csi -s compiler.scm < compiler.scm > compiler.s
    gcc -nostdlib -m32 -Wa,-adhlns=tmp.s.lst compiler.s -o urscheme-compiler


其中，compiler.s就是汇编代码，urscheme-compiler是最终生成的可执行文件。可以看到，它并不使用命令行参数，而是通过输入重定向读入scheme源码，然后再通过输出重定向将汇编代码写入另一个文件，然后再调用gcc编译汇编代码，最终生成可执行文件。

一旦在chicken的帮助下生成了编译器的可执行文件urscheme-compiler，它就可以实现自我编译：

    ./urscheme-compiler < compiler.scm > compiler.s1
    gcc -nostdlib -m32 -Wa,-adhlns=tmp.s.lst compiler.s1 -o urscheme-compiler1

为了方便起见，我将可执行文件urscheme-compiler扔到~/bin (我事先已经将~/bin添加到$PATH变量中)，再写一个脚本实现流水线编译，不用再输入输出重定向：

    #!/bin/bash
    urscheme-compiler < $1 > /tmp/tmp.s
    gcc -nostdlib -m32 -Wa,-adhlns=/tmp/tmp.s.lst /tmp/tmp.s -o a.out
    
事实上，这个编译器从scheme源码生成标准的AT&T格式的汇编代码，我们可以不用gcc，而用标准的GNU汇编器和链接器生成ELF可执行文件。

    as -o foo.o foo.s
    ld -s -o foo foo.o
