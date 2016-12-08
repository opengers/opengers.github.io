---
title: "关于ceph集群中pg总数的计算方法"
author: opengers
layout: post
permalink: /ceph/ceph-pgs-total-number/
categories: ceph
tags:
  - ceph
  - pg
format: quote
---

首先说下ceph环境    

``` shell
uname -a
Linux mon3 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
ceph -v
ceph version 10.2.2 (45107e21c568dd033c2f0a3107dec8f0b0e58374)
```

`crush map`状态      

``` shell
ceph -s
    cluster 99cc26bd-f8b0-438a-8085-a60da971afae
     health HEALTH_OK
     monmap e1: 3 mons at {mon1=192.168.7.21:6789/0,mon2=192.168.7.22:6789/0,mon3=192.168.7.23:6789/0}
            election epoch 550, quorum 0,1,2 mon1,mon2,mon3
     osdmap e20778: 15 osds: 15 up, 15 in
            flags sortbitwise
      pgmap v668984: 1612 pgs, 4 pools, 627 GB data, 157 kobjects
            1257 GB used, 15407 GB / 16665 GB avail
                1612 active+clean
```

在不改变`crush map`，不添加新osd情况下，ceph集群中pg数量是固定不变的，上面可以看到集群中pg总数为`1612` ，我们来看看这个pg数怎么来的       

## 计算pg数量  

`ceph pg dump`可以输出集群所有pg详细信息   

``` shell
ceph pg dump 2>/dev/null | egrep '^[0-9]+\.[0-9a-f]+\s' | wc -l
1612

#或者可以使用如下命令
ceph pg ls | grep -v '^pg_stat' | wc -l
1612
```

上面输出也可以验证集群中pg数量确实有`1612`个，我们这样考虑，所有的pg都肯定存在于某个`pool`上，因此计算所有`pool`上的pg数之和也可以得到总pg数  
  
``` shell
#ceph pg ls-by-pool poolname 可以查看存在于poolname上的所有pg详细信息
for i in `ceph osd pool ls`;do ceph pg ls-by-pool $i | grep -v '^pg_stat' | wc -l;done | awk '{a+=$1} END{print a}'
1612
```

根据pool计算结果也是`1612`，还可以这样考虑，所有的pg最终都是要放在某个osd上的，因此我们也可以根据osd计算集群中pg总数       

``` shell
#ceph osd df 可以查看集群中每个osd的使用信息，以及各个osd上的pg数 
ceph osd df tree | grep 'osd\.'
 2  0.50000  1.00000  1116G  65551M  1052G  5.73 0.74 281             osd.2  
13  0.09999  1.00000  1116G  12679M  1104G  1.11 0.14  50             osd.13 
14  0.09999  1.00000  1116G  13505M  1103G  1.18 0.15  46             osd.14
...

#用awk计算倒数第二个字段之和，系统总pg数
ceph osd df tree | grep 'osd\.' | awk '{a+=$(NF-1)} END{print a}'
3812
```

看来osd上存在的实际pg数`3812`跟上面的计算并不一致，要知道这些输出不一致的原因，我们先看一个具体的`pg`映射关系    

## pg与replicated size   

随便从`ceph pg dump`输出中选择一个pg，比如我们选择这个`1.fc`，看看它的映射关系  
  
``` shell
ceph pg dump | grep '1.fc' | awk '{print $1,$15}'
1.fc [7,2]
```

截取的是第一字段和第15字段，`[7,2]`意思是此pg有两个副本(副本数取决于pool的`replicated size`设置的值)，也即`osd.7`和`osd.2`上都有这个pg`1.fc`的一份拷贝。可以知道，每个pg都像这样有多个副本，存在于多个osd上，当我们计算所有osd上的pg时，`1.fc`的两个副本都会被计算一次，同理其它pg会也被计算`replicated size`次，因此最终根据osd计算出的pg是包括副本数的，下面我们从理论上计算下所有osd上的pg数量    

我们查看集群有多少个pool    

``` shell
ceph osd dump |grep pool | awk '{print $1,$3,$4,$5":"$6,$13":"$14}'
pool 'rbd' replicated size:2 pg_num:1024
pool 'ecpool' erasure size:3 pg_num:64
pool 'hotstorage' replicated size:3 pg_num:12
pool 'pool14' replicated size:3 pg_num:512
``` 

可以看到有4个pool，以pool `rbd`来说，其设置的pg数`pg_num`为`1024`，副本数`replicated size`为2，即pool `rbd`上的每个pg都有两个副本，因此`rbd`中所有pg总数为`2 * 1024`个，2048个pg，这样我们可以计算所有pool上包括副本数的`pg`总数    

``` shell
#$6 replicated
#$14 pg_num
ceph osd dump |grep pool | awk '{a+=$6 * $14} END{print a}'
3812
```

这里理论上计算的pg数`3812`与上面根据osd计算的pg总数一样，因此可以验证`ceph osd df`得到的pg数为包含副本的pg数量  

根本原因在于ceph集群中副本数(replicated size)的存在，从monitor角度来看系统中确确实实有`1612`个名称互不相同的pg，比如上面提到的`1.fc`。像`ceph pg ls`，`ceph pg dump`这些输出都可以列出所有这些pg信息，并显示其副本数。但是到了osd这一层，每个pg都复制了`replicated size`份，然后存入`osd`，从`osd`角度来计算自然也包括复制的pg。像`1.fc`这个pg，osd看到的是2个一模一样的`1.fc`，只是存在于不同的osd上，但是在monitor看，只能算一个pg，只是有多个拷贝而已   

集群中每个pg经过`crush`算法被映射到多个`osd`上去，`ceph pg ls`列出了所有pg到osd的映射信息     

   



