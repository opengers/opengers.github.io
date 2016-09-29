---
title: "讨论下ceph集群中pg总数的计算"
author: opengers
layout: post
permalink: /ceph/ceph-pgs-total-number/
categories: ceph
tags:
  - ceph
  - pg
format: quote
---

首先说下系统环境  

``` shell
uname -a
Linux mon3 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
ceph -v
ceph version 10.2.2 (45107e21c568dd033c2f0a3107dec8f0b0e58374)
```

ceph集群环境如下   

``` shell
[root@mon1 ~]# ceph -s
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

ceph集群在不改变crush map，不添加新osd情况下，集群中pg数量是固定不变的，上面也可以看到集群中pg总数为`1612` ，我们也可以验证下集群的pg数量     

``` shell
#ceph pg dump 输出集群所有pg详细信息  
ceph pg dump 2>/dev/null | egrep '^[0-9]+\.[0-9a-f]+\s' | wc -l
1612

#或者可以使用如下命令
ceph pg ls | grep -v '^pg_stat' | wc -l
1612
```

## 计算pg数量  

所有的pg最终是落在`osd`上的，我们来计算下所有osd上pg数量之和，`ceph osd df`可以查看每个集群osd使用信息，以及每个osd上的pg数    

``` shell
[root@mon1 ~]# ceph osd df tree
ceph osd df tree | grep 'osd\.' | awk '{a+=$(NF-1)} END{print a}'
3812
```

所有的pg都肯定属于某个`pool`，我们来计算下所有`pool`上的pg总数，`ceph pg ls-by-pool poolname`可以查看某个pool上的pg详细信息    

``` shell
[root@mon1 ~]# for i in `ceph osd pool ls`;do ceph pg ls-by-pool $i | grep -v '^pg_stat' | wc -l;done | awk '{a+=$1} END{print a}'
1612
```

## pg与pg_num  

上面根据osd计算时会出现3812，如果对这个值比较迷惑，先看看下面的计算结果  

我们查看集群有多少个pool    

``` shell
[root@mon1 ~]# ceph osd dump |grep pool | awk '{print $1,$3,$4,$5":"$6,$13":"$14}'
pool 'rbd' replicated size:2 pg_num:1024
pool 'ecpool' erasure size:3 pg_num:64
pool 'hotstorage' replicated size:3 pg_num:12
pool 'pool14' replicated size:3 pg_num:512
``` 

可以看到有4个pool，以pool `rbd`来说，其有2个副本，pg数量为`1024`  

``` shell
[root@mon1 ~]# ceph osd dump |grep pool | awk '{a+=$6 * $14} END{print a}'
3812
```

这里计算的3812与上面根据osd计算的pg总数一样，原因就在于pool的副本数这个概念，`pool`是一个namespace，我们定义`rbd`有2副本，意思是存在于`rbd`上每个`pg`都有2个副本，`rbd`中共有`pg_num * replicated size`个`pg`，我们来看`rbd`上的一个pg `1.fc`  

``` shell
[root@mon1 ~]# ceph pg dump | grep '1.fc' | awk '{print $1,$15}'
1.fc [7,2]
```

`[7,2]`表示此pg的两个副本存在于`osd.7`和`osd.2`上，对于每一个pg都是这样，当我们计算所有osd上的pg时，`1.fc`会被计算两次，同理其它pg会被计算`replicated size`次，因此最终根据osd计算出的pg是包括副本数的   



