---
layout: post
title: HLA 高级汇编语言基本语法备忘
date: 2014-10-30
---

并不是我突然对汇编语言产生了兴趣。其实我对汇编一无所知，唯一知道的是汇编指令其实就是 CPU 指令的助记符。用机器码写程序，想想都觉得可怕。我开始接触汇编的原因是，我正在看 Abdulaziz Ghuloum 写的一篇未完工的 tutorial：《Compilers: Backend to Frontend and Back to Front Again》，Abdulaziz Ghuloum 正是 ikarus 的作者，ikarus 是现存的已知的最快的 Scheme 实现（除了神龙见首不见尾的 Chez Scheme，那玩意就没人见过，所以不参与比较）。而 Chez 的作者 Kent Dybvig 正是 Abdulaziz Ghuloum 在印地安那大学的老师。Abdulaziz Ghuloum 这位老兄做事有始无终，06年写了篇tutorial，结果没写完就不写了,ikarus 也停止维护了。听人说他跑到中东某所大学教书去了。好在他还有一篇论文，可以与 tut 互相参考。

我听信了别人说的，学这篇 tutorial 不需要很深的基础，懂点 Scheme ,懂点 C，再懂点汇编就可以看了。我对汇编一窍不通，看到第二小节看不下去了。所以回过头来学汇编。

关于汇编的入门书，很多人推荐王爽的书，我找来PDF看了一段，好家伙，以16位的 8086 CPU 为背景，开发环境是 DOS。我想办法在 dosbox 下面安装了一个完整的 MS-DOS 6.22, 然后找到了 MASM 5.0安装上去，顺便把 Turbo Pascal 和 Turbo C 也给装上去了，我觉得我像是在考古。我甚至还想把 UCDOS 和 CCED、WPS 等一并装上去，重温一下十几年前学电脑的美好时光。。。开玩笑了，对着黑黑的显示器，插入各种格式标记进行排版，时不时地预览一下，再重新修改排版。这种做法对上个世纪的人们来讲是先进的生产力，对现在这个时代而言，那是痛苦和折磨，还有什么好怀念的。

看书吧，开篇讲了一大堆 段+偏移 的寻址方法。那玩意是在16位CPU + 20位地址总线的年代才有的特殊方法，现在哪里还看得到这样的机器？看不下去了，转而看《深入理解计算机系统》（CSAPP）。这年头，能够拥有简称的书都是经典书，就像SICP一样。我一下子就觉得高大上了。CSAPP讲汇编的出发点不是教会读者徒手用机器指令写程序，而是教读者读懂机器指令，进而分析程序的效率，从而有能力进行优化。这个出发点甚合我意，可是看了一段，也看不下去了。此书是大学的经典教材，换句话是，它需要有个老师的。因为它讲得比较简略，再加上我的智商拖了后腿，在看的过程中我产生了疑虑：这样理解对不对呢？没法求证。如果在课堂上，可以很方便地举手向老师提问，可是现在我找谁提问呢？越是看疑虑越多，最后得出一个结论：书是好书，可是不适合现阶段的我看。

最后找到的现在看的《编程语言编程艺术》，也是经典书。书名里的“艺术”两个字平添了许多高大上的感觉。书里用一种叫”HLA“的高级汇编语言来讲授，HLA具备许多高级语言的特征，分支和循环都有，而且有一个相对比较”大“的标准库。从语法上看，就是 C 和 Pascal 杂交的后代，与 Pascal 的关系更近一些。我又产生了疑虑：我只想学一丁点汇编语言，以便看懂开篇讲的那篇《Compilers: Backend to Frontend and Back to Front Again》.难道为了这个小小的目的，我不得不再学一门”高级“语言吗？看了一小段，这个疑虑差不多打消了。现在最不爽的是，这个 HLA 语言没有合适的 IDE，WIN平台下倒是有一个，可是 Linux 底下不能用，也没有 emacs 的 mode支持。我不得不用没有语法支持的emacs来写代码，没有高亮和缩进，和用记事本写程序差不多。

这里把它的基本语法记录一下，免得我不争气的大脑过目即忘。

#一、类型：

##整数类型：

HLA 预定义了多种不同的有符号整数类型，包括 int8, int16, int32 等。MS 还有 int64 和 int128。

##字符类型：

HLA 允许使用 char 来声明单字节的 ASCII 字符对象，也可以使用单引号括起来的单个字符（与C相同）

##布尔常量：

布尔类型标识符是 boolean。

HLA 有两个内置的布尔常量，true 和 false, 分别取值 1 和 0. 布尔常量只占一个字节的空间。可以使用能直接操作8位数的任意指令来处理它们。

#二、控制结构

##HLA 布尔表达式：

```
flag_specification		标志位  
!flag_specification		标志位取反
register			寄存器（寄存器值非零即为真）
!register			寄存器取反
Boolean_variable		布尔变量
！Boolean_variable		布尔变量取反
mem_reg relop mem_reg_const
register in LowConst .. HiConst		寄值器值是否位于某个区间
register not in LowConst .. HiConst 	寄存器值是否不位于某个区间
```

各种标志位:

```
@c      进位     进位标志为 1 时为真，标志位为 0 时为假
@nc     无进位   与上一条相反
@z      零       零标志位为 1 时为真，标志位为 0 时为假
@nz     非零     与上一条相反
@o      溢出     溢出标志位为 1 时为真，标志位为 0 时为假
@no     未溢出   与上一条相反
@s      符号     符号标志位为 1 时为真，标志位为 0 时为假
@ns     无符号   与上一条相反
```

关系运算符与C基本相同：

```
= 或者 ==
<> 或者 !=
<
<=
>
>=
```

比较运算符两边的操作数可以是内存变量或者寄存器，但是不能同时是内存变量。两个操作数长度必须相同。

如果左操作数是寄存器，右操作数是一个正常数或者另一个寄存器，HLA 将采用无符号比较。所以，暂时不要将一个存在寄存器中的负数与一个常数或者另一个寄存器相比较。


HLA 里有两个特殊的测试运算：in 和 not in。测试左操作数是否包含（或者不包含）在右操作数的范围内。

```
eax in 1..100
ch not in 'a'..'z'
```

##分支与循环：

1.if 分支

```
if ( expression ) then
	...
	...
elseif ( expression ) then
	...
	...
else
	...
	...
endif;
```

elseif 和 else 子句是可选的。

2.逻辑运算

与 &&
或 ||
非 !

逻辑非取反，要给布尔表达式加括号，比如 !(eax<0)

&& 的优先级比 || 高，如果表达式比较复杂，最好加括号。

与 C 基本一样。

3.while 循环

```
while ( expression ) do
	...
	...
endwhile;
```

4.for 循环

```
for (初始化; 循环条件; 步进) do
	...
	...
endfor;
```

与 C 的 for 循环基本一样。例：

```
for ( mov(0, i); i < 10; add(1, i)) do
	stdout.put("i = ", i, nl);
endfor;
```

5.直到型循环（do循环）

```
repeat
	...
	...
	...
until(expression);
```

循环条件为真则退出循环，否则就继续（这一点与 C 相反）

6.退出循环

```
break;  		退出当前一级循环
breakif (expression); 	带条件的退出
```

7.不带条件的无限循环

```
forever
	...
	...
endfor;
```

这是没有测试条件的无限循环，所以，应当在循环体内设置退出条件，然后用上面的 break 语句退出，不然就成死循环了。

7.带异常处理的 try

```
try
	...... //测试语句
	......
exception( 异常ID )
	......
	......
exception( 异常ID )
	......
	......
endtry;
```

尝试运行测试语句，如果没有错误发生，则跳转到 endtry；如果发生异常，则寻找对应的异常ID号，找到匹配的ID号后，执行对应的 exception 子句，执行完成后跳转到 endtry。

如果有异常发生，没有激活 try...endtry 语句，或者激活的 try...endtry 语句不处理指定的异常，程序就会终止并发出一条错误信息。



常见的异常ID由标准库头文件 excepts.hhf 定义，也可以根据自己的需要创建新的ID。

例：

```
repeat
	mov(false, GoodInteger);
	
	try
		stdout.put("Enter an integer: ");
		stdin.get(i);
		mov(true, GoodInteger);
	exception(ex.ConversionError);
		stdout.put("Illegal numeric value, please re-enter", nl);
	exception(ex.ValueOutOfRange);
		stdout.put("Value is out of range, please re-enter", nl);
	endtry;
until(GoodInteger);
```

常用的异常ID：

```
ex.ConversionError	字符串到数值的转换含有非法（非数值）字符
ex.ValueOutOfRange	当前操作的值过大
ex.StringOverflow	试图将过大的字符串存入一个字符串变量
```

#三、标准库

##stdio

预定义常量：

    nl              换行
    stdio.beel      ASCII响铃
    stdio.bs        ASCII退格
    stdio.tab       ASCII tab
    stdio.if        ASCII换行符
    stdio.cr        ASCII回车符

例程（函数）：

    stdout.newln();  //等效于：
    stdout.put(nl);
    
    stdout.puti8( 123 );        //输出8位有符号整数
    stdout.puti16( dx );        //输出16位有符号整数
    stdout.puti32( edx );       //输出32位有符号整数
    
    //下面的例程按指定的宽度输出数值
    stdout.puti8Size(Value8, width, padchar);
    stdout.puti16Size(Value16, width, padchar);
    stdout.puti32Size(Value32, width, padchar);
    
    //Value 可以是常量、寄存器、或者是内存单元
    //width 可以是-256 ～ +256之间的任意常量、或者是32位的寄存器以及内存单元
    //width 指定的是最小宽度，如果数值超过这个宽度，则正常显示；如果数值达不到这个宽度
    //则用 padchar 指定的字符填充，padchar 可以是一个ASCII码、单个字符，或者8位寄存器
    //width 为负值的情况暂时没搞懂，我认为翻译这本书的家伙也搞不懂自己在讲什么。
    //刚发现 p31 的一个错误，第8行代码是 add(1, ColCnt), 被搞成了 dd(1, ColCnt)
    
    
    