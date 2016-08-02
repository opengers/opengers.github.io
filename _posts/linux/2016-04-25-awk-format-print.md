---
title: 使用awk格式化输出文本
author: opengers
layout: post
permalink: /linux/awk-format-print/
categories: linux
tags:
  - awk
  - format
  - print
format: quote
---

><small>注意本文并不是一篇awk入门文章，而是偏重实例讲解  
awk借鉴了c语法，因此awk在许多地方还保留有c语言的痕迹，比如printf语句；for，if的语法结构等</small>

### 介绍  
最简单地说，AWK是一种用于处理文本的编程语言工具，处理模式是只要在输入数据中有模式匹配，就执行一系列指令。awk命令格式为：

``` shell 
awk {pattern + action} {filenames}
```

awk可以读取后接的文件，也可以读取来自前一命令的标准输入，它分别扫描输入数据的每一行，查找命令行中pattern是否匹配。如果匹配，则进行后续动作action。如果pattern不匹配或action部分处理完毕，则继续处理下一行，直到结束  
相比于sed常常作用于一整行的处理，awk则比较倾向于将一行分成数个字段来处理。awk将输入数据视为一个文本数据库，像数据库一样，它也有记录和字段的概念。默认情况下，记录的分隔符是回车，字段的分隔符是空白符(空格,\t)，所以输入数据的每一行表示一个记录，而每一行中的内容被空白分隔成多个字段。利用字段和记录，awk可以非常灵活地处理文件


### awk语法  
**语法**  
一个典型的awk语法如下：

    
    awk '{ 
    
         BEGIN{stat1} 
         BEGIN{stat2} 
         pattern1{action1} 
         pattern2{action2} 
         ... 
         patternn{actionn} 
         {默认动作,无条件,始终执行} 
    
         END{stat1} 
         END{stat2} 
    }'




其中BEGIN为处理文本前的操作,一般用于改变FS,OFS,RS,ORS等，BEGIN部分完成之后，awk读取第一行输入，并将第一行的数据填入$0,$1,$2,NR,NF等变量，然后进入正式处理阶段，待所有行处理完毕之后，进入END部分，END一般用于总结,打印报表等。正式处理是一个内建的循环,每一次循环读取一行数据，每一行的处理分为多模式,多动作,文本行符合条件pattern1就执行动作action1符合pattern2就执行动作action2…，还可以有默认的动作, 即没有pattern判断，始终执行此{}内的action。







BEGIN，END部分不是必须出现，可以没有，也可以有任意多个




pattern部分的写法有:








	
  * /reg/: 在整行范围内匹配reg，匹配到就执行后续动作











	
  * ! /reg/: 整行没匹配到reg，才执行后续动作











	
  * $1 ~ /reg/：只在第一字段匹配reg











	
  * $1 !~ /reg/: 不匹配








	
  * NR>=2： 从第二行开始处理


pattern，部分和随后的if，for部分，能用到的符号有：


#### **2 内建变量**



    
    $0
    当前记录（这个变量中存放着整个行的内容） 
    $1~$n
    当前记录的第n个字段，字段间由FS分隔 
    FS
    输入字段分隔符 默认是空格或\t
    NF
    当前记录中的字段个数，就是有多少列 
    NR
    已经读出的记录数，就是行号，从1开始，如果有多个文件话，这个值也是不断累加中。 
    FNR
    当前记录数，与NR不同的是，这个值会是各个文件自己的行号 
    RS
    输入的记录分隔符， 默认为换行符 
    OFS
    输出字段分隔符， 默认也是空格 
    ORS
    输出的记录分隔符，默认为换行符 
    FILENAME
    当前输入文件的名字




#### **3 if,for语句**






#在任何时候{}内都可以跟多个并列动作(使用“;”分隔)，下面的{action1} 和 {action1；action2；...} 都表示{}体内有多个动作，两种表示没有任何区别，写第二种仅仅是为了直观的表示可以有多个动作

#for循环写法

    
    for(i=1;i<=NF;i++){action1; action2; ..} #{}中用分号分隔多个动作
    for(i=1;i<=NF;i++)if; else if;else #for后接一个if结构
    for(i=1;i<=NF;i++)printf “for add” #简单的循环打印


#if 判断写法

    
    if($1 ~ /reg/){action1}; else if($1 ~ /reg2/){action2}; else{action3} #else if部分可以没有
    if($1 ~ /reg/ && $2 ~ /reg2/){action} #多个条件用”&&”,”||”表示
    if($1 ~ /reg/ || NR >= 5){action}


# if,for 混合写法

    
    { for(i=1;i<=NF;i++)if(…) printf “test”; else if(…) printf “test2”; else printf “test3”; print "not_for" } 
    #print “not_for”部分是并列与for循环结构的另一个action，在for循环之外，只会打印一次
    { for(i=1;i<=NF;i++){if(…) printf “test”; else if(…) printf “test2”; else printf “test3”；print “in_for“}; print "not_in_for" }
    { for(i=1;i<=NF;i++){if {s1;s2;} else if {s3;s4;} else {s5;s6;}; print "test"} }  #else if前不加分号 
    { for(i=1;i<=NF;i++)printf "for_add"; if(…);else if(…); else } #if并不在for循环体内







for循环的作用范围为：





	
  * 其后紧跟的if; else if; else语句

	
  * 其后紧跟的{}中的多个动作

	
  * 其后紧跟的一个第一个普通动作


if语句的作用范围：

	
  * if后紧跟的第一个动作

	
  * if后紧跟的{}中的多个动作




#### 4 awk**技巧**


1: AWK使用的RE为ERE

2: 如果在BEGIN中设置了OFS, 只有$0有改动OFS才能生效

3: printf 与 print 的区别: printf 不自动打印换行符, print 则自动打印

4: gsub的返回值并不是替换后的字符串,而是返回替换的次数

5: 字符串常量一定在用" "包围起来,否则当作变量使用, 如 $1=="ipaddress"

6: AWK 的 for 循环为 C-Style,即为 for(), 区别于shell中的for i in ...

7: AWK中可以使用多个分隔符,要封装在方括号里,用' '包围,以防 shell 对它们进行解释,如 awk -F '[ :/t]' ,使用空格,冒号,tab作为分隔符

8: next语句:从输入文件中取得下一个输入行,在AWK命令表顶部重新执行命令,一般用于跳过一些特殊的行

9: awk 匹配多个条件: awk '/kobe/ && /james/' #匹配同时有kobe和james的行

10: FS的默认值是[ /t/n]+, OFS的默认值为空格,RS,ORS的默认值都是换行

11: 定位行有两种方法: 1: NR==行号 2: 用RE /Love$/

12: exit语句:终止AWK程序,但不跳过END语句

13：$1..$n表示第几列(字段),$0表示整个行.

14：awk可用比较运算符：!=, >, <, >=, <=

15: 字符串匹配：~: 匹配 !~: 不匹配

16： &&：多个条件且, || 多个条件或

17: {s1;s2;s3;...}中多个语句用分号隔开;if; else if; else

18: print 后不带任何参数时，相当于print $0 ，将会打印整行记录


### awk字符函数


<table style="width: 464px;" border="0" bgcolor="#666666" cellpadding="4" cellspacing="1" class="mceItemTable" >
<tbody >
<tr >

<td bgcolor="#cccccc" width="163" >**函数**
</td>

<td bgcolor="#cccccc" width="296" >**说明**
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >gsub( Ere, Repl, [ In ] )
</td>

<td bgcolor="#ffffff" width="296" >除了正则表达式所有具体值被替代这点，它和 sub 函数完全一样地执行，。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >sub( Ere, Repl, [ In ] )
</td>

<td bgcolor="#ffffff" width="296" >用 Repl 参数指定的字符串替换 In 参数指定的字符串中的由 Ere 参数指定的扩展正则表达式的第一个具体值。sub 函数返回替换的数量。出现在 Repl 参数指定的字符串中的 &（和符号）由 In 参数指定的与 Ere 参数的指定的扩展正则表达式匹配的字符串替换。如果未指定 In 参数，缺省值是整个记录（$0 记录变量）。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >index( String1, String2 )
</td>

<td bgcolor="#ffffff" width="296" >在由 String1 参数指定的字符串（其中有出现 String2 指定的参数）中，返回位置，从 1 开始编号。如果 String2 参数不在 String1 参数中出现，则返回 0（零）。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >length [(String)]
</td>

<td bgcolor="#ffffff" width="296" >返回 String 参数指定的字符串的长度（字符形式）。如果未给出 String 参数，则返回整个记录的长度（$0 记录变量）。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >blength [(String)]
</td>

<td bgcolor="#ffffff" width="296" >返回 String 参数指定的字符串的长度（以字节为单位）。如果未给出 String 参数，则返回整个记录的长度（$0 记录变量）。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >substr( String, M, [ N ] )
</td>

<td bgcolor="#ffffff" width="296" >返回具有 N 参数指定的字符数量子串。子串从 String 参数指定的字符串取得，其字符以 M 参数指定的位置开始。M 参数指定为将 String 参数中的第一个字符作为编号 1。如果未指定 N 参数，则子串的长度将是 M 参数指定的位置到 String 参数的末尾 的长度。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >match( String, Ere )
</td>

<td bgcolor="#ffffff" width="296" >在 String 参数指定的字符串（Ere 参数指定的扩展正则表达式出现在其中）中返回位置（字符形式），从 1 开始编号，或如果 Ere 参数不出现，则返回 0（零）。RSTART 特殊变量设置为返回值。RLENGTH 特殊变量设置为匹配的字符串的长度，或如果未找到任何匹配，则设置为 -1（负一）。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >split( String, A, [Ere] )
</td>

<td bgcolor="#ffffff" width="296" >将 String 参数指定的参数分割为数组元素 A[1], A[2], . . ., A[n]，并返回 n 变量的值。此分隔可以通过 Ere 参数指定的扩展正则表达式进行，或用当前字段分隔符（FS 特殊变量）来进行（如果没有给出 Ere 参数）。除非上下文指明特定的元素还应具有一个数字值，否则 A 数组中的元素用字符串值来创建。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >tolower( String )
</td>

<td bgcolor="#ffffff" width="296" >返回 String 参数指定的字符串，字符串中每个大写字符将更改为小写。大写和小写的映射由当前语言环境的 LC_CTYPE 范畴定义。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >toupper( String )
</td>

<td bgcolor="#ffffff" width="296" >返回 String 参数指定的字符串，字符串中每个小写字符将更改为大写。大写和小写的映射由当前语言环境的 LC_CTYPE 范畴定义。
</td>
</tr>
<tr >

<td bgcolor="#ffffff" width="163" >sprintf(Format, Expr, Expr, . . . )
</td>

<td bgcolor="#ffffff" width="296" >根据 Format 参数指定的 [printf](http://www.cnblogs.com/chengmo/admin/zh_CN/libs/basetrf1/printf.htm#a8zed0gaco) 子例程格式字符串来格式化 Expr 参数指定的表达式并返回最后生成的字符串。
</td>
</tr>
</tbody>
</table>


##### 以上函数转自：[linux awk 内置函数详细介绍（实例）](http://www.cnblogs.com/chengmo/archive/2010/10/08/1845913.html)




### awk 实例




##### 1.awk作用域



    
    awk '/AL/ {printf $1; print $2}' emp.txt
    awk '/AL/{print $1} {print $2}' emp.txt


1.第一种只处理匹配到AL的行; 然后打印这些行的第一字段和第二字段

2.第二种只有在匹配到AL的行才打印字段一，但是字段二是无条件的，始终打印


#####  2.awk内置函数用法


如下文本test.log，需要把文本中单位是"M"的字段转换为"G"

    
    16G    16G    1.9G    40G none
    4G     4G     952M    60G
    16G    16G    1.6G    40G none
    5G     780M   5G      80G


若我们单纯地想替换字符串，则可以使用一下命令,注意：print没带默认参数时，默认打印整行记录

    
    cat 1.txt | awk '{
    　　sub(/M/,"G",$i)
    　　print
    }'


完整替换如下，替换的同时进行数值计算

    
    cat 1.log | \
    awk '{
        for(i=1;i<=NF;i++){
            if($i ~ /M/) 
                printf int(substr($i,1,match($i,/M/)-1)/1024*100)/100"G\t"
            else 
                printf $i"\t"　　
        }
        print "" 
    }'


1.使用for遍历每一行的每一个字段，使用if处理匹配到"M"的字段，然后调用awk内置函数int, substr, match处理, 关于awk内置函数，可以参考[linux awk 内置函数详细介绍（实例）](http://www.cnblogs.com/chengmo/archive/2010/10/08/1845913.html)

2.先乘100，再除以100，是为了转换为"G"单位后保留两位小数

3.注意 printf,print 两个函数的区别


##### 3.awk条件语句


如第二个例子中的文本，现在需要分别统计每行中带有"G", "M", "none" 字段的个数，并输出

    
    cat test.log | \
    awk '
         BEGIN{OFS="\t"}
         {    
            a=b=c=0
            for(i=1;i<=NF;i++){
                if($i ~ /G/)
                    a+=1
                else if($i ~ /M/)
                    b+=1
                else
                    c+=1
            }
            print $0,"G:"a,"M:"b,"none:"c
         }
         END{print "end!"}
    '


结果如下：

    
    16G    16G    1.9G    40G    G:4    M:0    none:0
    4G     4G     952M    60G    G:3    M:1    none:0
    16G    16G    1.6G    40G    G:4    M:0    none:0
    5G     780M   5G      80G    G:3    M:1    none:0
    end!




##### 4.awk中break用法


给出一个如下文本test2.log，在每一行中，只输出字母及其之前的字符

    
    1  2  a  5 6
    11 b  55 66
    21 22 23 c 25  26


写法如下：

    
    cat test2.log | \
    awk '{
        for(i=1;i<=NF;i++){
            if($i ~ /[a-z]/) {
                printf $i"\t"
                break }
            else 
                printf $i"\t"
        }
        print ""
    }'


1.break 用法跟c语言用法一样，跳出for循环


##### 5.awk中数组


如下文本test3.log，给定一个id值，输出其在所有id中是第几个

    
    test1
    id=615187629
    test2
    test3
    id=615183434
    test4
    id=615123789
    test5
    id=615975882


给定id值615123789，其在所有id中是第三个，计算如下：

    
    cat test3.log | \
    awk -v var1=615123789 -F [=] '
        /id/ {
            b+=1
            a[b]=$2
        }
        END {
            for(i in a) if(a[i] == var1) print "number:",i
        }
    '


6.awk中的整数计算

如下文本，这是一个kvm虚拟机进程(省略了部分文本)，我们要获取其映射到宿主机上的vnc端口号，即由"-vnc 0.0.0.0:1"字符串计算出其vnc端口号为5901(5900 + 1)，若是"-vnc 0.0.0.0:2",则端口号为5902

    
    cat kvm.txt
    qemu 144148 4.7 4.2 ... /usr/local/qemu/bin/qemu-kvm -name lnmptest-107 ... -device isa-serial,chardev=charserial0,id=serial0 -vnc 0.0.0.0:1 -vga cirrus  timestamp=on


实现有多种，sed,shell,awk都可以

    
    #awk
    cat kvm.txt | awk '{ for(i=1;i<=NF;i++){if($i == "-vnc"){sub(/0.0.0.0:/,"",$(i+1));print $(i+1)+5900}} }'
    #shell
    cat kvm.txt | egrep -o "\-vnc [^ ]*" | awk -F: '{print $2+5900}'
    #sed
    cat kvm.txt | sed "s/^.* -vnc [0-9.]*:\([0-9]*\).*/\1+5900/g" | bc





