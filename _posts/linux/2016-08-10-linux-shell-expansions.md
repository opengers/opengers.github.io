---
title: shell中的扩展(Expansions)
author: opengers
layout: post
permalink: /linux/linux-shell-brace-parameter-command-pathname-expansion/
categories: linux
tags:
  - shell
  - expansions
---

><small>对于一条shell命令，shell内部会对此命令字符串进行一系列处理，然后才真正开始执行此命令，这些处理称作shell扩展(Expansions)   
本文基于centos7.2，bash版本4.2</small>

## shell扩展介绍  
每当我们在shell下键入一条命令，并按下`Enter`，shell扩展(Expansions)会对我们键入的命令字符串进行一系列处理，然后才执行此命令。shell扩展是shell内建的处理程序，它发生在命令执行之前，因此与我们键入的命令无关   

shell把整条命令按功能分割为多个独立单元，每个独立单元作为整体对待，叫做一个word，也称为token。比如`cat /etc/passwd | grep "root"`中有五个token，分别是`cat`、`/etc/passwd`、`|`、`grep`、`"root"` ，本文使用token这一名称

我们在命令行键入一条命令，经过shell扩展处理，然后才交给具体的命令去执行，下面我们用一条简单例子演示一下  
 
``` shell
ls *.sh
1.sh  test.sh  b.sh
```

键入的命令是`ls *.sh`，作用是列出当前目录下所有以`.sh`结尾的文件。shell扩展会把`*.sh`展开为当前目录下所有以`.sh`结尾的文件名组成的字符串`1.sh  test.sh  b.sh`，然后才交给具体的命令`ls`去执行。shell扩展是shell内部处理程序，因此`ls`看不到`*.sh`，它看到的是shell展开后的字符串`1.sh  test.sh  b.sh`

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

## 花括号扩展(brace expansion)  
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
  
花括号扩展中字符串`${`是不允许的，这是为了避免与后续的参数扩展(parameter)冲突，但是你可以使用转义符解决`\$`，稍后有说明  
	
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

#生成2个宽度的数字序列
echo {01..20}
01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17 18 19 20

#生成倒序的英文字母
echo {z..a}
z y x w v u t s r q p o n m l k j i h g f e d c b a

#可以嵌套
echo {A{1,2},B{3,4}}
A1 A2 B3 B4

echo {2007..2009}-0{a,b}
2007-0a 2007-0b 2008-0a 2008-0b 2009-0a 2009-0b
```

## 波浪号扩展(Tilde Expansion)  
波浪号扩展我们最熟悉的用法就是`cd ~`，切换回当前用户家目录，下面来看看它是怎么发生的  

首先我们需要知道什么样的字符串会被波浪号扩展(Tilde Expansion)处理，若一个token(word)以波浪号`~`开始，并且这个`~`没有带引号，那么从这个`~`开始直到一个不带引号的斜杠`/`之间的字符串才会被波浪号扩展(Tilde Expansion)处理，这个字符串被称作`tilde-prefix`，我们标记为`~string`，处理依据如下

* string字符串中带有引号，不会进行扩展
* string不为空，并且string这个用户不名存在，不会进行扩展
* string不为空，并且string这个用户名存在，`tilde-prefix`替换为此用户名家目录
* string为空，`tilde-prefix`被替换为`#HOME`这个变量指定的用户的家目录
* string为空，`$HOME`这个变量也为空，`tilde-prefix`被替换为当前运行此shell的用户的家目录
* string为`+otherstrs`，`tilde-prefix`被环境变量`PWD`替换
* string为`-otherstrs`，`tilde-prefix`被环境变量`OLDPWD`替换

`~`用法很简单，上面文字表述无感的话，敲几遍命令就明白了

``` shell
#测试环境：系统存在admin这个用户，当前以root身份运行
#当string为空
echo ~
/root
#当string为admin
echo ~admin
/home/admin
#当string为adminaa，用户不存在，不扩展
echo ~adminaa
~adminaa
#string为admin'abc'，有引号，不处理
echo ~'admin'
~admin
echo ~admin'abc'
~adminabc

#~-，~+的用法
cd /etc/ssh
echo ~+
/etc/ssh
echo ~+/softs
/etc/ssh/softs
#~-显示的是上个目录
cd /root/
echo ~-
/etc/ssh
```

波浪号扩展很少用到，特别是`~+, ~-`的形式，最常用的也就是个`~`，这里只是记录下这种扩展

## 参数扩展(parameter expansion)   
参数扩展的基本格式是`${parameter}`，扩展的结果是`${parameter}`被替换为相应的值。`$`是前导符，`parameter`是一个可以存储值的参数，其有多种形式，下面会细说。字符对`{、}`不是必须的，它可以明确表示这是一个参数扩展，并且明确扩展范围。  

像`$var`这种变量形式只是参数扩展的一种形式，这里的参数`parameter`是一个变量`var`，参数扩展相对比较复杂，下面分几部分谈   

### 什么是参数扩展   
在shell中，符号`$`用作参数扩展、命令替换、算数扩展的前导符，就是说`$`符号告诉shell接下来要匹配的可能是这三种扩展中的一种。当然前提是这里的符号`$`没被转义，也没被引号引用。转义符这个问题贯穿所有的扩展中，我们后面会专门讨论。   

字符对`{、}`定义了扩展范围，虽然它不是必须的，但坚持一直使用是个好习惯，它可以保护变量名。在下面两种情况下必须要有`{、}`

1. `parameter`为值大于9的数字，比如$`{10}`表示命令行传入的第10个参数
1. `parameter`是一个变量名，其后面紧跟一个可能会被解释成变量名一部分的字符   

``` shell
#下面这种情况无需{..}，$USER后面就是空格,因此shell可以分离出变量名就是USER
echo $USER is here
root is here
#shell无法分离USERlog字符串
echo $USERlog is here
is here
echo ${USER}log is here
rootlog is here
```

解释第一种情况，我们经常需要向shell脚本中传递参数，然后使用`$1 $2 $n`形式引用他们，假如向脚本中传递10个参数，第十个参数会是`$10`，问题是shell会认为它是`$1`后面跟一个数字0，此时就需要用`{}`明确告诉shell是`${10}`    

第二种情况很好理解，假如我需要输出变量`USER`的值，并且后面紧跟着输出一个字符串`log`，这时不加`{}`是无法做到的，因为shell无法分离`USERlog`   

### 参数parameter的形式    
这部分我们讨论`parameter`有哪几种形式。其形式可以是变量名(例如`LANG、PWD、USER`)，数字(例如`1、2、...、n`)，特殊字符(例如`?、*、@`)，感叹号(格式:${!strings})，数组引用(例如`{num[1]}`)。具体可以查看[bash手册](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameters)

**变量名**     
变量我们很熟悉，此时`parameter`是一个变量，扩展的结果是`${parameter}`被替换为`parameter`的值。   

可以对`parameter`进行赋值，或修改其值，`parameter`根据shell中变量名命名规则命名，可能需要使用花括号`{}`明确变量名范围。同时，如果你真的是想在屏幕输出`${LANG}`这几个字符，可以使用转义符，像这样`echo \${LANG}`     

可能你还见过`${LANG:-strings}`或`${parameter/pattern/string}`这种用法，它可以让你在显示变量值之前对变量进行替换或修改，我们稍后会说  

**数字**   
此时`parameter`为数字，称作位置参数，扩展的结果是`$n`被替换为命令行传入的第n个参数  

在脚本中，可以使用`$n`引用从命令行传入的第n个参数，不能使用赋值语句对其赋值，内建命令`set、shift`用于对位置参数进行控制，具体可以通过内建命令的man手册`man set`，定位到set命令处查看。当脚本中有函数(functions)时，在函数内部，`$n`表示传递给此函数的第n个参数    

``` shell
cat main.sh
#!/usr/bin/env bash

#定义函数function_a
function_a() {
#函数内部，$1,$2表示传递给此函数的第n个参数
echo "function_a:$1"
echo "function_a:$2"	
}

echo "main:$1"
echo "main:$2"
#传递给函数两个参数fa fb
function_a fa fb
echo "main:$3"

#命令行传递给脚本的是ma mb mc三个参数
sh main.sh ma mb mc
main:ma
main:mb
function_a:fa
function_a:fb
main:mc
```

**特殊字符**  
此时，`parameter`为特殊字符，此时可以不需要`{}`。特殊字符只能被引用，像这样`echo $@`，不能赋值。
`$*` 展开所有的位置参数，默认用空格分割，使用引号
特殊字符有`@、*、#、?、-、$、!、0、_`，其意义看下面    

``` shell
cat 3.sh
#!/bin/bash
echo '$@:'$@
echo '$*:'$*
echo '$#:'$#
echo '$?:'$?
echo '$-:'$-
echo '$$:'$$
echo '$!:'$!
echo '$0:'$0
echo '$_:'$_
#执行3.sh，传入3个参数
sh 3.sh arg1 arg2 arg3
$@:arg1 arg2 arg3
$*:arg1 arg2 arg3
$#:3
$?:0
$-:hB
$$:12024
$!:
$0:3.sh
$_:$0:3.sh
#可以再试试加入单/双引号的输出，例如echo "$@"
```

**感叹号**   
当`parameter`字符串第一个字符是`!`时，shell把随后的字符串解释为一个变量名，就是`${!strings}`这种形式(当然，strings要是一个正确的变量名，不能带有特殊字符)，它称作间接扩展，其特点是shell把`strings`变量的值依然看做一个变量，展开了两次。例外的是${!prefix*}和${!name[@]}。需要与上面提到的特殊字符形式`$!`区别的是这里需要有`{}`   

``` shell
#定义一个变量name
name=LANG
#shell把变量name的值"LANG"依然看做变量，因此得到的是变量LANG的值
echo ${!name}
en_US.UTF-8
#没有{}存在，就是上面提到的特殊字符变量，$!是最近放入后台的job进程ID，这里为空
echo $!name
name
echo $name
LANG
```

