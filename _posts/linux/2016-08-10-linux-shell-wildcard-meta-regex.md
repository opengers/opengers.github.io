---
title: shell中的扩展(Expansions)
author: opengers
layout: post
permalink: /linux1/linux-shell-Expansions/
categories: linux1
tags:
  - shell
  - Expansions
format: quote
---

><small>我们在命令行下键入一条命令，shell内部会对此命令字符串进行一系列处理，然后才真正开始执行此命令，这些处理称作shell扩展(Expansions)   
本文基于centos7.2，bash版本4.2</small>

##  shell扩展    
每当我们在shell下键入一条命令，并按下`Enter`，shell扩展(Expansions)会对我们键入的命令字符串进行一系列处理，然后才执行此命令。shell扩展是shell内建的处理程序，它发生在命令执行之前，因此与我们键入的命令无关   

shell把整条命令按功能分割为多个独立单元，每个独立单元作为整体对待，叫做一个word，也成为token。比如`cat /etc/passwd | grep "root"`中有五个token，分别是`cat`、`/etc/passwd`、`|`、`grep`、`"root"`  

我们在命令行键入一条命令，经过shell扩展处理，然后才交给具体的命令去执行，下面我们用一条简单例子演示一下    

``` shell
ls *.sh
1.sh  test.sh  b.sh
```

键入的命令是`ls *.sh`，作用是列出当前目录下所有以`.sh`结尾的文件。shell扩展会把`*.sh`变为当前目录下所有以`.sh`结尾的文件名组成的字符串`1.sh  test.sh  b.sh`，然后才交给具体的命令`ls`去执行。shell扩展是shell内部处理程序，因此`ls`看不到`*.sh`，它看到的是shell扩展后的字符串`1.sh  test.sh  b.sh`

shell扩展有以下几种，并按以下顺序处理  
* brace expansion  花括号({})扩展  
* tilde expansion  `~`字符扩展  
* parameter and variable expansion  参数和变量扩展  
* arithmetic expansion  算术扩展  
* command substitution  命令替换  
* process substitution  过程替换  
* word splitting    
* Filename Expansion  通配符扩展  

以上扩展中，只有brace expansion，word splitting，filename expansion 三种扩展可以改变token个数，其它扩展只是把一个token改为另一个token，唯一例外的token是`$@`，`${name[@]}`  

## brace expansion 花括号扩展  
花括号扩展格式有两种，第一种格式    
	prestring{str1,str2,...}poststring
这种形式会从左至右输出花括号中的所有字符串`str1，str2，...strn`。prestring，poststring是可以添加到花括号中每个str上面的字符串，prestring被添加在每个str的前面，poststring被添加在每个str的后面，如下
	echo Front-{A,B,C}-Back
	Front-A-Back Front-B-Back Front-C-Back
	#在这个例子中，prestring为"Front-"，poststring为"-Back"

第二种格式  
	prestring{A..B}poststring
这种形式意思是输出一个序列，序列范围是从A到B，因此不难猜出，A，B可以是整数，或英文字母。prestring，poststring和第一种类似  

``` shell
#整数
echo {1..10}
1 2 3 4 5 6 7 8 9 10
#随心所欲
echo {1..10}.1
1.1 2.1 3.1 4.1 5.1 6.1 7.1 8.1 9.1 10.1
#添加前缀
echo file-{1..10}
file-1 file-2 file-3 file-4 file-5 file-6 file-7 file-8 file-9 file-10
#字母
echo {a..z}
a b c d e f g h i j k l m n o p q r s t u v w x y z
#可以倒序输出
echo {x..p}
x w v u t s r q p
```




