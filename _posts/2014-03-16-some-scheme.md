---
layout: post
title:  几种Scheme实现的基本用法
date:   2014-03-16
---
###一、Chicken
####1、Read-Eval-Print Loop:

    csi

####2、脚本:

    #!/path/bin/csi -s

（不加 -s 选项的话，执行完脚本会进入REPL环境，不会自动退出）

####3、编译成本地代码:

    csc hello.scm

（先转译成C，再用gcc编译）


###二、Gambit-C
####1、Read-Eval-Print Loop：

    gsi

####2、脚本：

    #!/path/bin/gsi-script

####3、编译：

    gsc [-o outfile] -exe hello.scm

（这个也是先翻译成C，再用gcc编译，不过和chicken比起来可执行文件的体积太大了，一个hello world有4M那么大，strip后还有3.5M）

###三、Larceny
####1、Read-Eval-Print Loop:

    larceny

####2、脚本：

1)R5RS脚本：

    larceny -nobanner -- hello.scm

2)R6RS脚本：

    scheme-script hello.scm

####3、编译的方法暂时没找到   
这个实现有三个版本：

>+  Larceny compiles directly to native machine code for the Intel IA32 or SPARC architectures.
>+  Petit Larceny is a portable implementation that compiles to C instead of machine code.
>+  Common Larceny runs in the Common Language Runtime (CLR) of Microsoft .NET, generating IL, which is JIT-compiled to native machine code by the CLR. 

以我蹩脚的方言水平，第一个，就是默认的Larceny，把REPL环境中输入的每一条指令都编译成机器码；而Petit Larceny则把它翻译成C代码；第三个，Common Larceny则是在.net环境上运行的。

嗯！虽然不能编译可执行文件的方法，不过这个实现在一些基准测试中成绩很靠前，值得关注。
对了，在REPL环境中可以用下面的指令编译：

    (compile-file "filename.scm")

得到一个编译后的 .fasl 文件,然后载入

    (load "filename.fasl")

###四、Guile
####1、Read-Eval-Print Loop：

    guile

####2、脚本：

    #!/path/bin/guile -s

###五、DrRacket
####1、Read-Eval-Print Loop：

1）Racket:

    racket
    mzscheme

2）R5RS:

    plt-r5rs

####2、以脚本方式运行：

    racket -r file.scm
    mzscheme -r file.scm

####3、启动GUI环境的IDE:

    drracket

