---
layout: post
title: 被bash的初始化文件搞到头大
date: 2014-04-25
---

刚装上Debian的时候，我发现自己home目录下的.profile .bashrc等文件被清空了。我误以为Debian没有为用户提供预定制的脚本，对我而言，仅仅是用它来定义几个别名，把几个目录加进 $PATH 变量。

前两天又在另一台机器上全新安装了Debian，这一次连home分区都是新的，后来我发现Debian里也自带了编写好的脚本，默认生成了.profile .bashrc等文件，在 /etc/skel 目录下有正确的模板，当一个新用户登陆系统后， /etc/skel 里的shell定制脚本会被复制到用户目录下。但是我奇怪地发现，.profile文件里的内容没有执行。

按照常理，bash分为非交互登陆shell和交互shell。

前者是在用户登陆系统时运行的SHELL，它依次执行 /etc/profile 以及 /etc/profile.d里面的脚本，然后再用户目录下寻找 .bash_profile ，找不到再找 .bash_login ，还是找不到就再找.profile，如果找到便执行其中的脚本。

而.bashrc是交互式shell脚本，每次打开终端窗口它都要运行一次。

我遇到的问题是，在我的目录下并不存在.bash_profile 和 .bash_login，但是当我登陆系统的时候.profile脚本并不运行。搞了半天不知道原因，最后，不得不把里面的命令添加到.bashrc的尾部。