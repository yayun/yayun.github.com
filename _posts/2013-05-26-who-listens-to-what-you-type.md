---
layout: post
title: "Who listens to what yo type"
description: ""
category: 
tags: [linux]
---
{% include JB/setup %}
---

   可能对于想成为unix高级用户的人们来说，它们最需要了解的一点就是你不是直接与unix交流的。你是和一个叫做shell的程序在打交道,我们把shell理解成一个command line interface。shell可以说是保护unix kernel。

   Unix os通常叫做内核。通常仅仅程序能和内核直接交流（通过系统调用）。用户和shell交流。shell能够解释你输入的命令，或者直接执行，又或者把这些命令参数传递给其它的程序（这些程序会向内核请求更低级的系统服务）。

   例如，当你输入一个命令，这个命令显示一个文件名是四个字符的文件，并且该文件名是以m开头：`cat m???`。具体过程是这样的：shell解释器先找到这个文件名，补全文件名，然后调用cat命令打印完整的文件名。cat命令通过系统调用（linux下的系统调用是通过0x80中断来完成的）去查找磁盘上的文件，然后以字符流的形式打印文件中的内容。

   其次，shell解释你输入的命令行然后为你即将调用的命令打包。因为shell首先读取命令行，所以明白shell是怎样改变它读取的信息很重要。

    shell是按照如下结构进行工作的：
    1：打印一个提示符
    2：读入用户输入
    3：把用户的输入解析成为程序的名字和参数
    4：调用fork（）系统调用产生一个子线程
        子线程调用exec（）系统调用去启动指定的程序
        父线程（shell program）调用wait（）等待子线程结束
    5：当子线程结束的时候（启动的程序）结束后。shell掉转到第一步。


 如果你键入ls，shell程序就会开始一个新的进程执行ls程序，然后当代知道这个进程完成，完成后，控制权又交回给shell。尽管用户键入在命令提示符下键入的大多数命令都是unix程序的名字（像 ls ...），shell 也可以识别一些不是程序名字的特殊的命令（shell的内部命令你可以使用`man bash-builtins`来查看它们）。例如，`exec`结束shell程序，`cd` 改变当前的工作目录。当你在键入`exec ls`的时候，会先退出shell程序然后使用ls程序来代替当前进程（而不是复制一个子线程）。shell直接使用系统调用来执行这些命令，而不是复制一个子线程去接管它们。

 Here shows what happens when you type `ls`:
 
 
    +--------+
    | pid=7  |
    | ppid=4 |
    | bash   |
    +--------+
        |
        | calls fork
        V
    +--------+             +--------+
    | pid=7  |    forks    | pid=22 |
    | ppid=4 | ----------> | ppid=7 |
    | bash   |             | bash   |
    +--------+             +--------+
    |                      |
    | waits for pid 22     | calls exec to run ls
    |                      V
    |                  +--------+
    |                  | pid=22 |
    |                  | ppid=7 |
    |                  | ls     |
    V                  +--------+
    +--------+                 |
    | pid=7  |                 | exits
    | ppid=4 | <---------------+
    | bash   |
    +--------+
    |
    | continues
    V
 

reference：[who listens to what you type](http://docstore.mik.ua/orelly/unix/upt/ch01_02.htm)；

[the shell and system calls](https://www.cs.duke.edu/courses/spring05/cps210/Lab1.html)

