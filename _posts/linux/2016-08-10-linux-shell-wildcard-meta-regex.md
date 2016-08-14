---
title: shell中的扩展(Expansions)
author: opengers
layout: post
permalink: /linux/linux-shell-Expansions/
categories: linux
tags:
  - shell
  - Expansions
format: quote
---

><small>对于一条shell命令，shell内部会对此命令字符串进行一系列处理，然后才真正开始执行此命令，这些处理称作shell扩展(Expansions)   
本文基于centos7.2，bash版本4.2</small>

###  shell扩展介绍  
每当我们在shell下键入一条命令，并按下`Enter`，shell扩展(Expansions)会对我们键入的命令字符串进行一系列处理，然后才执行此命令。shell扩展是shell内建的处理程序，它发生在命令执行之前，因此与我们键入的命令无关   

shell把整条命令按功能分割为多个独立单元，每个独立单元作为整体对待，叫做一个word，也称为token。比如`cat /etc/passwd | grep "root"`中有五个token，分别是`cat`、`/etc/passwd`、`|`、`grep`、`"root"`  

我们在命令行键入一条命令，经过shell扩展处理，然后才交给具体的命令去执行，下面我们用一条简单例子演示一下  
 
``` shell
ls *.sh
1.sh  test.sh  b.sh
```

键入的命令是`ls *.sh`，作用是列出当前目录下所有以`.sh`结尾的文件。shell扩展会把`*.sh`变为当前目录下所有以`.sh`结尾的文件名组成的字符串`1.sh  test.sh  b.sh`，然后才交给具体的命令`ls`去执行。shell扩展是shell内部处理程序，因此`ls`看不到`*.sh`，它看到的是shell扩展后的字符串`1.sh  test.sh  b.sh`

shell扩展有以下几种，并按以下顺序处理，当然如果没找到匹配的扩展格式，那就不处理  

* brace expansion  花括号({})扩展  
* tilde expansion  `~`字符扩展  
* parameter and variable expansion  参数和变量扩展  
* arithmetic expansion  算术扩展  
* command substitution  命令替换  
* process substitution  过程替换  
* word splitting    
* Filename Expansion  通配符扩展  

以上扩展中，只有brace expansion，word splitting，filename expansion 三种扩展可以改变token个数，其它扩展只是把一个token改为另一个token，唯一例外的token是`$@`，`${name[@]}`  

### 花括号扩展(brace expansion)  
花括号扩展是首先被执行的扩展，格式有两种，字符串输出，或序列输出  

**字符串输出**  
 
	prestring{str1,str2,...}poststring
	
这种形式会从左至右输出花括号中的所有字符串`str1，str2，...strn`，花括号中的字符串以逗号`,`分隔。prestring，poststring是可以添加到花括号中每个str上面的字符串，prestring被添加在每个str的前面，poststring被添加在每个str的后面，如下

	echo Front-{A,B,C}-Back
	Front-A-Back Front-B-Back Front-C-Back
	#在这个例子中，prestring为"Front-"，poststring为"-Back"

**序列输出**

	prestring{A..B}poststring
	
这种形式会一个序列，序列范围是从A到B，因此不难猜出，A，B可以是整数，或英文字母。prestring，poststring作用和第一种类似，下面是几个序列输出的例子

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

**高级用法**   

花括号扩展的语法很简单，使用格式也很固定，但还是有一些需要注意的地方  

shell仅仅是把花括号中的字符串以逗号`,`分隔，然后从左至右输出各个字符串，并不会对字符串做任何语法解释处理  
  
为了避免与后续的参数扩展(parameter)冲突，花括号扩展中字符串`${`会被认为是非法使用   
	
``` shell
a=test
#输出为空
echo $a{1..10}
#错误提示
echo ${1..10}
-bash: ${1..10}: bad substitution
#下面这个才正确
echo ${a}{1..10}
test1 test2 test3 test4 test5 test6 test7 test8 test9 test10
``` 

花括号扩展中的一对花括号不能被引号引住，否则字符串会被原样输出，并且花括号中需要至少有一个不带引号的逗号或者一个正确的序列表达式

``` shell
#输出a和z
echo {a,z}
a z
#有引号将不会被处理，原样输出
echo "{a,z}"
{a,z}
#逗号也不能带引号
echo {a","z}
{a,z}
echo {a".."z}
{a..z}
echo {a..z}
a b c d e f g h i j k l m n o p q r s t u v w x y z
```

也可以在花括号扩展中使用反斜杠`\`转义特殊字符，如下例子

``` shell
echo {a,b,\,,\{}
a b , {
echo \${a..z}
$a $b $c $d $e $f $g $h $i $j $k $l $m $n $o $p $q $r $s $t $u $v $w $x $y $z
```

花括号扩展虽然语法简单，但结合其字符串输出和序列输出还是能玩出一些小花样来，如下

``` shell
#快速创建多个目录
mkdir /usr/local/src/bash/{old,new,dist,bugs}

#目录嵌套

#生成2个宽度的数字序列
echo {01..99}
01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99

#生成倒序的英文字母
echo {z..a}
z y x w v u t s r q p o n m l k j i h g f e d c b a

#可以嵌套
echo {A{1,2},B{3,4}}
A1 A2 B3 B4

[root@lijtools-161 tmp]# echo {2007..2009}-0{a,b}
2007-0a 2007-0b 2008-0a 2008-0b 2009-0a 2009-0b
```

### 波浪号扩展(Tilde Expansion)











