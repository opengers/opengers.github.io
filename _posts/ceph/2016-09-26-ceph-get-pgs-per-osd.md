---
title: "计算ceph集群中每个osd上的pg数"
author: opengers
layout: post
permalink: /ceph/ceph-get-pgs-per-osd/
categories: ceph
tags:
  - ceph
  - pg
  - crush
format: quote
---

可以有多种方法获取ceph集群中每个osd上的pg数量，测试环境如下  

``` shell
uname -a
Linux mon3 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
ceph -v
ceph version 10.2.2 (45107e21c568dd033c2f0a3107dec8f0b0e58374)
```

## 使用osd df命令   

``` shell
ceph osd df tree | awk '/osd\./{print $NF":"$(NF-1)}'
osd.0:207
osd.3:202
osd.4:195
osd.5:219
osd.15:176
osd.1:215
osd.7:207
osd.8:195
osd.9:177
osd.10:208
osd.2:151
osd.13:26
osd.14:24
osd.12:144
osd.6:146
```

## 使用ls-by-osd命令     

``` shell
#ceph pg ls-by-osd osd.id 根据osd输出pg信息
for i in `ceph osd ls`;do echo -n "osd.$i:";ceph pg ls-by-osd $i | grep -v '^pg_stat' | wc -l;done
osd.0:207
osd.1:215
osd.2:151
osd.3:202
osd.4:195
osd.5:219
osd.6:146
osd.7:207
osd.8:195
osd.9:177
osd.10:208
osd.12:144
osd.13:26
osd.14:24
osd.15:176
```

## 从pg dump输出中过滤    

上面两个都是ceph提供的命令，我们知道`ceph pg dump`可以输出集群中所有pg详细信息，从这些信息中可以很容易过滤出各个osd上的pg数，也是对上面命令的验证    

``` shell
for i in `ceph osd ls`;do echo -n "osd.$i:";ceph pg dump 2>/dev/null | egrep '^[0-9]+\.[0-9a-f]+\s' | awk '{print $15}' | egrep "^\[$i,|,$i,|,$i\]$" | wc -l;done
osd.0:207
osd.1:215
osd.2:151
osd.3:202
osd.4:195
osd.5:219
osd.6:146
osd.7:207
osd.8:195
osd.9:177
osd.10:208
osd.12:144
osd.13:26
osd.14:24
osd.15:176
```

上面三种方法输出结果一致，只要一种即可，但多种方法使我们可以更好的理解ceph中pg在osd上的分布，如果你的集群使用默认的`crush`，每个`osd crush weight`值一样，那么可以从输出看到各个osd上的pg数量近似相等  