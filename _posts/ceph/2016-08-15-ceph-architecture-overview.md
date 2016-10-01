---
title: "ceph存储集群架构概览"
author: opengers
layout: post
permalink: /ceph/ceph-storage-cluster-architecture-overview/
categories: ceph
tags:
  - ceph
  - architecture
  storage cluster
format: quote
---

><small>系统centos7.2  
ceph版本 ceph version 10.2.2</small>

本文是根据自己理解，对ceph架构的一篇总结

## 硬件



Ceph 元数据服务器对 CPU 敏感，它会动态地重分布它们的负载，所以你的元数据服务器应该有足够的处理能力
Ceph 的 OSD 运行着 RADOS 服务、用 CRUSH 计算数据存放位置、复制数据、维护它自己的集群运行图副本，因此 OSD 需要一定的处理能力（如双核 CPU ）
监视器只简单地维护着集群运行图的副本，因此对 CPU 不敏感；但必须考虑机器以后是否还会运行 Ceph 监视器以外的 CPU 密集型任务

元数据服务器和监视器必须可以尽快地提供它们的数据，所以他们应该有足够的内存，至少每进程 1GB
OSD 的日常运行不需要那么多内存（如每进程 500MB ）差不多了；然而在恢复期间它们占用内存比较大（如每进程每 TB 数据需要约 1GB 内存）。通常内存越多越好
btrfs 尚未稳定到可以用于生产环境的程度，但它可以同时记日志并写入数据，而 xfs 和 ext4 却不能。

单个驱动器容量越大，其对应的 OSD 所需内存就越大，特别是在重均衡、回填、恢复期间。根据经验， 1TB 的存储空间大约需要 1GB 内存。

Ceph 最佳实践指示，你应该分别在单独的硬盘运行操作系统、 OSD 数据和 OSD 日志。

你可以在同一主机上运行多个 OSD ，但要确保 OSD 硬盘总吞吐量不超过为客户端提供读写服务所需的网络带宽；还要考虑集群在每台主机上所存储的数据占总体的百分比，如果一台主机所占百分比太大而它挂了，就可能导致诸如超过 full ratio 的问题，此问题会使 Ceph 中止运作以防数据丢失。

OSD 数量较多（如 20 个以上）的主机会派生出大量线程，尤其是在恢复和重均衡期间。很多 Linux 内核默认的最大线程数较小（如 32k 个），如果您遇到了这类问题，可以把 kernel.pid_max 值调高些。理论最大值是 4194303 。例如把下列这行加入 /etc/sysctl.conf 文件
kernel.pid_max = 4194303

## 存储系统逻辑架构

![ceph-1](/images/ceph/architecture-overview/ceph0.jpg) 

如上，是ceph存储系统层次图，最底层是RADOS(Reliable, Autonomic, Distributed Object Store)