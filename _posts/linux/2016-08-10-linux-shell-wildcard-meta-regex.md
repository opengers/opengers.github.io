---
title: shell中的特殊字符概述
author: opengers
layout: post
permalink: /linux1/linux-shell-speical-character/
categories: linux1
tags:
  - shell
  - wildcard
  - metacharacter
format: quote
---

><small>本文介绍shell中能够用到的各种特殊字符，包括通配符、元字符、转义字符、正则表达式字符  
这几种特殊字符所使用到的字符有重合，但它们表示的意思并不一样  
本文基于centos7.2，bash版本4.2.46</small>

##  shell扩展  
我们在命令行下键入一条命令，shell内部对此命令经过一系列处理，然后才真正开始执行此命令，然后才打印结果给我们，了解shell内部的处理过程，有助于写出更流畅的shell脚本，下面，先从最简单的`echo`命令开始

``` shell
	echo this is a echo
	echo *
	1.log 2.log kvm-install phy-init push-server rh sshexec vminfo.txt vm_init zabbix-init
```


