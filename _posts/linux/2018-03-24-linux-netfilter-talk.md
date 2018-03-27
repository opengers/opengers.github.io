---
title: openstack底层技术-netfilter框架        
author: opengers
layout: post
permalink: /openstack/openstack-base-netfilter/
categories: openstack
tags:
  - openstack
  - netfilter
  - iptables
---   

* TOC
{:toc}    

# netfilter框架      

netfilter是linux内核中的一个数据包处理框架，用于替代原有的ipfwadm和ipchains等数据包处理程序。netfilter的功能包括数据包过滤，修改，NAT等。netfilter在内核协议栈的不同位置实现了5个钩子函数(hooks),不同类型的网络数据包会通过不同的hook点(比如数据包是要forward，或是交给local process处理)，用户层工具iptables/ebtables通过操作不同的hook点以实现对通过此hook点的数据包的处理，关于netfilter框架，可以参考[Netfilter Architecture](https://netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html)       

下面这张图来自维基百科[netfilter](https://en.wikipedia.org/wiki/Netfilter), 它展示了netfilter注册的hook点在内核协议栈的分布，以及packet在通过内核协议栈时，所经过的hook点，你们可能发现这张图在我的博客中出现多次了，那是因为这张图太经典了。图中有清晰的网络模型划分，因此很容易看到数据包在经过某条链时所处的层次       

![netfilter](/images/openstack/openstack-virtual-devices/netfilter.png)      

我们知道iptables只处理IP数据包(IP/PORT/SNAT/DNAT/...)，而ebtables只工作在链路层`Link Layer`处理以太网帧(比如修改源/目mac地址)。图中用有颜色的长方形方框表示iptables或ebtables的表和链，绿色小方框表示`network level`，即iptables的表和链。蓝色小方框表示`bridge level`，即ebtables的表和链，由于处理以太网帧相对简单，因此链路层的蓝色小方框相对较少。  

我们还注意到一些代表iptables表和链的绿色小方框位于链路层，这是因为`bridge_nf`代码的作用(从2.6 kernel开始)，`bridge_nf`的引入是为了解决在链路层Bridge中处理IP数据包的问题(需要通过内核参数开启)，那为什么要在链路层Bridge中处理IP数据包，而不等数据包通过网络层时候再处理呢，这是因为不是所有的数据包都一定会通过网络层，有可能数据包从Bridge的一个port进入，经过`bridging decision`后从另一个port发出，`bridge_nf`也是openstack中实现安全组功能的基础，关于Bridge下的数据包过滤，我的另一篇文章有总结[Bridge与netfilter](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/#bridge%E4%B8%8Enetfilter)   

`bridge_nf`代码有时候会引起困惑，就像我们在图中看到的那样，iptables的表和链(绿色小方框)跑到了链路层，netfilter文档对此也有说明[ebtables/iptables interaction on a Linux-based bridge](http://ebtables.netfilter.org/br_fw_ia/br_fw_ia.html)              

><small>The br-nf code makes bridged IP frames/packets go through the iptables chains. Ebtables filters on the Ethernet layer, while iptables only filters IP packets       
It should be noted that the br-nf code sometimes violates the TCP/IP Network Model. As will be seen later, it is possible, f.e., to do IP DNAT inside the Link Layer</small>      

ok，图中长方形小方框已经解释清楚了，还有一种椭圆形的方框`conntrack`，即Connection Tracking，这是netfilter框架提供的连接跟踪机制，其能够"审查"通过此处的所有网络数据包，并能识别出此数据包属于哪个网络连接(比如数据包a属于`IP1:8888->IP2:80`这个tcp连接，数据包b属于`ip3:9999->IP4:53`这个udp连接)，因此，连接跟踪能够跟踪并记录通过此处的所有网络连接及其状态。图中可以清楚看到连接跟踪代码所处的网络栈位置，因此如果不想让某些数据包被跟踪(`NOTRACK`),那就要找位于椭圆形方框`conntrack`之前的表和链来设置规则。conntrack机制是iptables实现状态匹配(`-m state`)以及NAT的基础，它由单独的内核模块`nf_conntrack`实现。下面还会详细介绍             
 
接着看图中左下方`bridge check`方框，数据包从主机上的某个网络接口进入(`ingress`), 在`bridge check`处会检查此网络接口是否属于某个Bridge的port，如果是就会进入Bridge代码处理逻辑(`broute`->...), 否则就会送入网络层处理(`raw`-->...)       

图中下方中间位置的`bridging decision`环节就是根据数据包目的MAC地址判断此数据包是转发还是交给上层处理，这个另一篇文章也有解释[linux-bridge](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/#linux-bridge)       

图中中心位置的`routing decision`就是路由选择，根据系统路由表(`ip route`查看), 决定数据包是forward，还是交给本地处理       

总的来看，图中整个packet flow表示的是数据包总是从主机的某个接口进入(左下方`ingress`), 然后经过一系列表和链处理，最后，或是交由主机上某应用进程处理(上中位置`local process`)，或是从主机另一个接口发出(右下方`egress`),这里的接口即可以是物理网卡em1，也可以是虚拟网卡tun0/vnetx，还可以是Bridge上的一个port。   

上面就是关于这张图的一些解释，如果还有其它疑问，欢迎留言讨论，下面说说netfilter的应用方面            

# Connection Tracking       

当加载内核模块`nf_conntrack`后，conntrack机制就开始工作，如上图，内核中有三个位置(椭圆形方框`conntrack`)能够跟踪数据包。对于每个通过`conntrack`的数据包，内核都为其生成一个conntrack条目用以跟踪此连接，对于后续通过的数据包，内核会判断若此数据包属于一个已有的连接，则更新所对应的连接跟踪条目的状态(比如更新为ESTABLISHED状态)，否则内核会为它新建一个conntrack条目。所有的conntrack条目存放在一张表里，称为连接跟踪表                 

那么内核如何判断一个数据包是否属于一个已有的连接呢，我们先来了解下连接跟踪表       
      
**连接跟踪表**     

连接跟踪表存放于系统内存中，可以用`cat /proc/net/nf_conntrack`查看当前连接跟踪表中所有的conntrack条目，每个conntrack条目表示一个连接，可以是tcp/udp/icmp/...，conntrack条目像下面这样，它包含了数据包的原始方向信息(蓝色)和期望的回复包信息(红色)，这样内核能够在后续到来的数据包中识别出属于此连接的双向数据包，并更新此连接的状态。连接跟踪表中能够存放的conntrack条目的最大值称作CONNTRACK_MAX                       
       
<small>ipv4     2 tcp      6 431955 ESTABLISHED <font color="blue">src=172.16.207.231 dst=172.16.207.232 sport=51071 dport=5672</font> <font color="red">src=172.16.207.232 dst=172.16.207.231 sport=5672 dport=51071</font> [ASSURED] mark=0 zone=0 use=2</small>       

当然，根据conntrack跟踪的协议不同，上面条目信息也不一样，比如icmp协议           

在内核中，连接跟踪表是一个二维数组结构的哈希表(hash table)，哈希表的大小称作HASHSIZE，哈希表的每一项(hash table entry)称作bucket，因此哈希表中有HASHSIZE个bucket存在，每个bucket包含一个链表(linked list)，每个链表能够存放若干个conntrack条目(Bucket Size)。对于一个新收到的数据包，内核使用如下步骤判断其是否属于一个已有连接：                   
- 内核将此数据包信息(源目IP，port，协议号)进行hash计算得到一个hash值，在哈希表中以此hash值做索引，得到数据包所属的bucket(链表)。这一步hash计算时间是固定的并且很短                 
- 遍历bucket，查找是否有匹配的conntrack条目。这一步是比较耗时的操作，bucket size越大，遍历时间越长                        

**如何设置最大连接跟踪数**       

根据上面对哈希表的解释，系统最大允许连接跟踪数(CONNTRACK_MAX) = `连接跟踪表大小(HASHSIZE) * Bucket大小(Bucket Size)`。从连接跟踪表获取bucket是hash操作时间很短，因此HASHSIZE值可以很大。而遍历bucket相对费时，因此为了conntrack性能考虑，Bucket Size越小越好，默认为8     

若`nf_conntrack`模块未加载，此时只需要更改`HASHSIZE`值，模块加载后`CONNTRACK_MAX`会自动设置(HASISIZE x 8)      

``` shell   
#设置hashsize为40w，也即设置CONNTRACK_MAX为320w    
echo "options nf_conntrack hashsize=400000" > /etc/modprobe.d/nf_conntrack.conf
#加载模块
modprobe nf_conntrack
#查看当前CONNTRACK_MAX
sysctl -a | grep 'net.netfilter.nf_conntrack_max'
#可以根据这里显示的CONNTRACK_MAX值验证Bucket Size是否为8，Bucket Size = CONNTRACK_MAX / HASISIZE
```      

若`nf_conntrack`模块已加载，则直接更改CONNTRACK_MAX值                

``` shell
sysctl -w net.netfilter.nf_conntrack_max=3200000
#为了下次重启生效
echo "options nf_conntrack hashsize=400000" > /etc/modprobe.d/nf_conntrack.conf
```   

**如何计算连接跟踪所占内存**     

连接跟踪表存储在系统内存中，设置的最大连接跟踪数越多，消耗的最大系统内存就越多，可以用下面公式计算设置不同的最大连接跟踪数所占最大系统内存                               

``` shell
size_of_mem_used_by_conntrack (in bytes) = CONNTRACK_MAX * sizeof(struct ip_conntrack) + HASHSIZE * sizeof(struct list_head)
```

假如我们需要设置最大连接跟踪数为320w，在centos7系统上，`sizeof(struct ip_conntrack)` = 376，`sizeof(struct list_head)` = 16，并且Bucket Size默认为8，因此`HASHSIZE = CONNTRACK_MAX / 8`，因此          

``` shell
size_of_mem_used_by_conntrack (in bytes) = 3200000 * 376 + (3200000 / 8) * 16
#= 1153MB
```

因此可以得到，在centos7系统上，设置320w的最大连接跟踪数，所消耗的内存大约为1GB        

关于上面两个`sizeof(struct *)`值在你系统上的大小，可以使用如下python代码计算       
          
``` shell
import ctypes

#不同系统可能此库名不一样  
LIBNETFILTER_CONNTRACK = 'libnetfilter_conntrack.so.3.6.0'

nfct = ctypes.CDLL(LIBNETFILTER_CONNTRACK)
print 'sizeof(struct nf_conntrack):', nfct.nfct_maxsize()
print 'sizeof(struct list_head):', ctypes.sizeof(ctypes.c_void_p) * 2
```

要注意的是，conntrack机制作用只是跟踪并记录通过它的所有网络连接及其状态，并把记录信息存放在连接跟踪表里。conntrack机制本身并不能够修改或过滤数据包，能够修改过滤数据包的是iptables，正是有了conntrack提供的连接跟踪表，iptables才能够实现对数据包的状态匹配      

# iptables状态匹配                

pass       

# Bridge与netfilter   

pass 

# netfilter与LVS

pass

# openstack安全组实现   

pass




       


参考文章    

><small>https://wiki.khnet.info/index.php/Conntrack_tuning    
https://voipmagazine.wordpress.com/2015/02/27/tuning-the-linux-connection-tracking-system/    
https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE
</small>