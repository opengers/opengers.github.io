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

## netfilter框架      

netfilter是linux内核中的一个数据包处理框架，用于替代原有的ipfwadm和ipchains等数据包处理程序。netfilter的功能包括数据包过滤，修改，NAT等。netfilter在内核协议栈的不同位置实现了5个钩子函数(hooks),不同类型的网络数据包会通过不同的hook点(比如数据包是要forward，或是交给local process处理)，用户层工具iptables/ebtables通过操作不同的hook点以实现对通过此hook点的数据包的处理，关于netfilter框架，可以参考[Netfilter Architecture](https://netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html)       

下面这张图来自维基百科[netfilter](https://en.wikipedia.org/wiki/Netfilter), 它展示了netfilter注册的hook点在内核协议栈的分布，以及packet在通过内核协议栈时，所经过的hook点，你们可能发现这张图在我的博客中出现多次了，那是因为这张图太经典了。图中有清晰的网络模型划分，因此很容易看到数据包在经过某条链时所处的层次       

![netfilter](/images/openstack/openstack-virtual-devices/netfilter.png)      

我们知道iptables只处理IP数据包(IP/PORT/SNAT/DNAT/...)，而ebtables只工作在链路层`Link Layer`处理以太网帧(比如修改源/目mac地址)。图中用有颜色的长方形方框表示iptables或ebtables的表和链，绿色小方框表示`network level`，即iptables的表和链。蓝色小方框表示`bridge level`，即ebtables的表和链，由于处理以太网帧相对简单，因此链路层的蓝色小方框相对较少。  

我们还注意到一些代表iptables表和链的绿色小方框位于链路层，这是因为`bridge_nf`代码的作用(从2.6 kernel开始)，`bridge_nf`的引入是为了解决在链路层Bridge中处理IP数据包的问题(需要通过内核参数开启)，那为什么要在链路层Bridge中处理IP数据包，而不等数据包通过网络层时候再处理呢，这是因为不是所有的数据包都一定会通过网络层，有可能数据包从Bridge的一个port进入，经过`bridging decision`后从另一个port发出，`bridge_nf`也是openstack中实现安全组功能的基础，关于Bridge下的数据包过滤，我的另一篇文章有总结[Bridge与netfilter](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/#bridge%E4%B8%8Enetfilter)   

`bridge_nf`代码有时候会引起困惑，就像我们在图中看到的那样，iptables的表和链(绿色小方框)跑到了链路层，netfilter文档对此也有说明[ebtables/iptables interaction on a Linux-based bridge](http://ebtables.netfilter.org/br_fw_ia/br_fw_ia.html)              

><small>It should be noted that the br-nf code sometimes violates the TCP/IP Network Model. As will be seen later, it is possible, f.e., to do IP DNAT inside the Link Layer</small>      

ok，图中长方形小方框已经解释清楚了，还有一种椭圆形的方框`conntrack`，即connection tracking，这是netfilter提供的连接跟踪机制，此机制允许内核"审查"通过此处的所有网络数据包，并能识别出此数据包属于哪个网络连接(比如数据包a属于`IP1:8888->IP2:80`这个tcp连接，数据包b属于`ip3:9999->IP4:53`这个udp连接)。因此，连接跟踪机制使内核能够跟踪并记录通过此处的所有网络连接及其状态。图中可以清楚看到连接跟踪代码所处的网络栈位置，如果不想让某些数据包被跟踪(`NOTRACK`),那就要找位于椭圆形方框`conntrack`之前的表和链来设置规则。conntrack机制是iptables实现状态匹配(`-m state`)以及NAT的基础，它由单独的内核模块`nf_conntrack`实现。下面还会详细介绍             
 
接着看图中左下方`bridge check`方框，数据包从主机上的某个网络接口进入(`ingress`), 在`bridge check`处会检查此网络接口是否属于某个Bridge的port，如果是就会进入Bridge代码处理逻辑(`broute`->...), 否则就会送入网络层处理(`raw`-->...)       

图中下方中间位置的`bridging decision`环节就是根据数据包目的MAC地址判断此数据包是转发还是交给上层处理，这个另一篇文章也有解释[linux-bridge](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/#linux-bridge)       

图中中心位置的`routing decision`就是路由选择，根据系统路由表(`ip route`查看), 决定数据包是forward，还是交给本地处理       

总的来看，图中整个packet flow表示的是数据包总是从主机的某个接口进入(左下方`ingress`), 然后经过一系列表和链处理，最后，或是交由主机上某应用进程处理(上中位置`local process`)，或是从主机另一个接口发出(右下方`egress`),这里的接口即可以是物理网卡em1，也可以是虚拟网卡tun0/vnetx，还可以是Bridge上的一个port。   

上面就是关于这张图的一些解释，如果还有其它疑问，欢迎留言讨论，下面说说netfilter的应用方面            

## connection tracking       

当加载内核模块`nf_conntrack`后，conntrack机制就开始工作，如上图，椭圆形方框`conntrack`在内核中有两处位置(PREROUTING和OUTPUT之前)能够跟踪数据包。对于每个通过`conntrack`的数据包，内核都为其生成一个conntrack条目用以跟踪此连接，对于后续通过的数据包，内核会判断若此数据包属于一个已有的连接，则更新所对应的conntrack条目的状态(比如更新为ESTABLISHED状态)，否则内核会为它新建一个conntrack条目。所有的conntrack条目都存放在一张表里，称为连接跟踪表                   

那么内核如何判断一个数据包是否属于一个已有的连接呢，我们先来了解下连接跟踪表       
      
### 连接跟踪表            

连接跟踪表存放于系统内存中，可以用`cat /proc/net/nf_conntrack`查看当前跟踪的所有conntrack条目。如下是代表一个tcp连接的conntrack条目，根据连接协议不同，下面显示的字段信息也不一样，比如icmp协议                                  
       
<small>ipv4     2 tcp      6 431955 ESTABLISHED <font color="blue">src=172.16.207.231 dst=172.16.207.232 sport=51071 dport=5672</font> <font color="red">src=172.16.207.232 dst=172.16.207.231 sport=5672 dport=51071</font> [ASSURED] mark=0 zone=0 use=2</small>        

每个conntrack条目表示一个连接，连接协议可以是tcp，udp，icmp等，它包含了数据包的原始方向信息(蓝色)和期望的响应包信息(红色)，这样内核能够在后续到来的数据包中识别出属于此连接的双向数据包，并更新此连接的状态。连接跟踪表中能够存放的conntrack条目的最大值，即系统允许的最大连接跟踪数记作`CONNTRACK_MAX`    

![netfilter_hash_table](/images/openstack/openstack-netfilter/hash_table.jpeg)        

在内核中，连接跟踪表是一个二维数组结构的哈希表(hash table)，哈希表的大小称作`HASHSIZE`，哈希表的每一项(hash table entry)称作bucket，因此哈希表中有`HASHSIZE`个bucket存在，每个bucket包含一个链表(linked list)，每个链表能够存放若干个conntrack条目(`bucket size`)。对于一个新收到的数据包，内核使用如下步骤判断其是否属于一个已有连接：                   
- 内核将此数据包信息(源目IP，port，协议号)进行hash计算得到一个hash值，在哈希表中以此hash值做索引，得到数据包所属的bucket(链表)。这一步hash计算时间是固定的并且很短                 
- 遍历bucket，查找是否有匹配的conntrack条目。这一步是比较耗时的操作，`bucket size`越大，遍历时间越长                        

### 如何设置最大连接跟踪数             

根据上面对哈希表的解释，系统最大允许连接跟踪数`CONNTRACK_MAX` = `连接跟踪表大小(HASHSIZE) * Bucket大小(bucket size)`。从连接跟踪表获取bucket是hash操作时间很短，而遍历bucket相对费时，因此为了conntrack性能考虑，`bucket size`越小越好，默认为8     

``` shell
#查看系统当前最大连接跟踪数CONNTRACK_MAX
sysctl -a | grep net.netfilter.nf_conntrack_max
#net.netfilter.nf_conntrack_max = 3200000

#查看当前连接跟踪表大小HASHSIZE
sysctl -a | grep net.netfilter.nf_conntrack_buckets
#400384
#或者
cat /sys/module/nf_conntrack/parameters/hashsize
#400384

#可以得出此时bucket size = 3200000 / 400384     
```

如下，现在需求是设置系统最大连接跟踪数为320w，由于`bucket size`值不能直接设置，为了使`bucket size`值为8，我们需要同时设置`CONNTRACK_MAX`和`HASHSIZE`，因为他们的比值就是`bucket size`                           

``` shell
#HASHSIZE = 3200000 / 8
echo 400000 > /sys/module/nf_conntrack/parameters/hashsize

#系统最大连接跟踪数
sysctl -w net.netfilter.nf_conntrack_max=3200000     

#注意nf_conntrack内核模块需要加载     
```  

为了使重启后生效(nf_conntrack模块重新加载)                 

``` shell   
#nf_conntrack模块提供了设置HASHSIZE的参数          
echo "options nf_conntrack hashsize=400000" > /etc/modprobe.d/nf_conntrack.conf                  
```       

只需要固化HASHSIZE值，nf_conntrack模块在重新加载时会自动设置CONNTRACK_MAX =  `hashsize * 8`，但如果你`bucket size`不使用默认值8，则CONNTRACK_MAX也需要固化，以保持其比值为你想要的`bucket size`      

上面我们没有改变`bucket size`的默认值8，但是若内存足够并且性能很重要，你可以考虑每个bucket一个conntrack条目(`bucket size` = 1)，即`HASHSIZE = CONNTRACK_MAX`       

``` shell
#HASHSIZE
echo 3200000 > /sys/module/nf_conntrack/parameters/hashsize
#CONNTRACK_MAX
sysctl -w net.netfilter.nf_conntrack_max=3200000
```      

### 如何计算连接跟踪所占内存             

连接跟踪表存储在系统内存中，设置的最大连接跟踪数越多，消耗的最大系统内存就越多，可以用下面公式计算设置不同的最大连接跟踪数所占最大系统内存                                   

``` shell
size_of_mem_used_by_conntrack (in bytes) = CONNTRACK_MAX * sizeof(struct ip_conntrack) + HASHSIZE * sizeof(struct list_head)
```

假如我们需要设置最大连接跟踪数为320w，在centos7系统上，`sizeof(struct ip_conntrack)` = 376，`sizeof(struct list_head)` = 16，并且`bucket size`默认为8，因此`HASHSIZE = CONNTRACK_MAX / 8`，因此          

``` shell
size_of_mem_used_by_conntrack (in bytes) = 3200000 * 376 + (3200000 / 8) * 16
# = 1153MB ~= 1GB
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

### conntrack条目                         

``` shell
ipv4     2 tcp      6 33 SYN_SENT src=172.16.200.119 dst=172.16.202.12 sport=54786 dport=10051 [UNREPLIED] src=172.16.202.12 dst=172.16.200.119 sport=10051 dport=54786 mark=0 zone=0 use=2
```   

如上是一条conntrack条目，它代表当前某个被跟踪的连接，我们可以得到如下信息：     

- 此连接使用ipv4协议，是一条tcp连接(tcp的协议类型代码是6)                       
- `33`是这条conntrack条目在当前时间点的生存时间(从设置值开始倒计时，倒计时完后此条目将被删除)，可以使用`sysctl -a |grep conntrack | grep timeout`查看不同协议不同状态下生存时间设置值，当然这些设置值都可以调整，注意若后续有收到属于此连接的数据包，则此生存时间将被重置(重新从设置值开始倒计时)，并且状态改变，生存时间设置值也会改变       
- `SYN_SENT`是这个连接在当前时间点的状态(内核角度)，`SYN_SENT`表示这个连接只在一个方向发送了一TCP SYN包，还未看到响应的SYN+ACK包。           
- `src=172.16.200.119 dst=172.16.202.12 sport=54786 dport=10051`是此连接的源目地址、源目端口。即`172.16.200.119:54786 -SYN-> 172.16.202.12:10051`    
- `[UNREPLIED]`说明此时这个连接还没有收到任何回应，当一个连接在两个方向上都有收到数据包时，[UNREPLIED]标志就会被移除              
- 接下来的`src=172.16.202.12 dst=172.16.200.119 sport=10051 dport=54786`地址和端口和前面是相反的，是内核希望收到的响应包的信息。意思是若后续conntrack跟踪到某个数据包信息与此匹配，则此数据包属于此连接，并且是一个响应数据包                                                

conntrack条目信息依据IP所包含的协议不同而不同，相比tcp连接，udp和icmp没有连接建立和关闭过程，因此其conntrack条目也有不同        

``` shell
#udp
ipv4     2 udp      17 22 src=172.16.200.119 dst=172.16.200.1 sport=33636 dport=53 [UNREPLIED] src=172.16.200.1 dst=172.16.200.119 sport=53 dport=33636 mark=0 zone=0 use=2    
#icmp
ipv4     2 icmp     1 29 src=114.14.240.77 dst=111.31.136.9 type=8 code=0 id=57960 src=111.31.136.9 dst=114.14.240.77 type=0 code=0 id=57960 mark=0 zone=0 use=2
```

要注意的是，conntrack机制作用只是跟踪并记录通过它的网络连接及其状态，并把信息更新在连接跟踪表，以提供给iptables做状态匹配使用                           
        
## iptables状态匹配                            

首先我们先明确下conntrack机制跟踪数据包的位置，除了本地产生的包由OUTPUT链处理外，所有连接跟踪都是在PREROUTING链里进行处理的，意思就是， iptables会在PREROUTING链里从新计算所有的状态。如果我们发送一个流的初始化包，状态就会在OUTPUT链 里被设置为NEW，当我们收到回应的包时，状态就会在PREROUTING链里被设置为ESTABLISHED。如果第一个包不是本地产生的，那就会在PREROUTING链里被设置为NEW状 态。综上，所有状态的改变和计算都是在nat表中的PREROUTING链和OUTPUT链里完成的。            

iptables是带有状态匹配的防火墙，它使用`-m state`模块从连接跟踪表查找数据包所属连接及其状态。上面我们分析的那条conntrack条目处于`SYN_SENT`状态，这是内核记录的状态，数据包在内核中可能会有几种不同的状态，但是映射到用户空间iptables，只有4种状态可用：NEW，ESTABLISHED，RELATED 和INVALID，注意我们这里说的状态不是tcp/ip协议中tcp连接的各种状态，这里的4种状态只是iptables用于完成状态匹配而定义的，如下是4种状态详细解释    

| State | 解释 |   
| --------- | -------- |     
| NEW | NEW说明这个包是我们看到的第一个 包。意思就是，这是conntrack模块看到的某个连接第一个包，它即将被匹配了。比如，我们看到一个SYN 包，是我们所留意的连接的第一个包，就要匹配它。第一个包也可能不是SYN包，但它仍会被认为是NEW状态。这样做有时会导致一些问题，但对某些情况是有非常大的帮助的。例如，在 我们想恢复某条从其他的防火墙丢失的连接时，或者某个连接已经超时，但实际上并未关闭时。 |    
| ESTABLISHED | ESTABLISHED已经注意到两个方向上 的数据传输，而且会继续匹配这个连接的包。处于ESTABLISHED状态的连接是非常容 易理解的。只要发送并接到应答，连接就是ESTABLISHED的了。一个连接要从NEW变 为ESTABLISHED，只需要接到应答包即可，不管这个包是发往防火墙的，还是要由防 火墙转发的。ICMP的错误和重定向等信息包也被看作是ESTABLISHED，只要它们是我 们所发出的信息的应答。 |    
| RELATED | RELATED是个比较麻烦的状态。当一 个连接和某个已处于ESTABLISHED状态的连接有关系时，就被认为是RELATED的了。换句话说，一个连接要想 是RELATED的，首先要有一个ESTABLISHED的连接。这个ESTABLISHED连接再产生一个主连接之外的连接，这 个新的连接就是RELATED的了，当然前提是conntrack模块要能理解RELATED。ftp是个很好的例子，FTP-data 连接就是和FTP-control有RELATED的。还有其他的例子，比如，通过IRC的DCC连接。有了这个状态，ICMP应 答、FTP传输、DCC等才能穿过防火墙正常工作。注意，大部分还有一些UDP协议都依赖这个机制。这些协议 是很复杂的，它们把连接信息放在数据包里，并且要求这些信息能被正确理解。 |     
| INVALID | INVALID说明数据包不能被识别属于 哪个连接或没有任何状态。有几个原因可以产生这种情况，比如，内存溢出，收到不知属于哪个连接的ICMP 错误信息。一般地，我们DROP这个状态的任何东西。 |    
| UNTRACKED | This is the UNTRACKED state. In brief, if a packet is marked within the raw table with the NOTRACK target, then that packet will show up as UNTRACKED in the state machine. This also means that all RELATED connections will not be seen, so some caution must be taken when dealing with the UNTRACKED connections since the state machine will not be able to see related ICMP messages et cetera. |        
{:.mbtablestyle}                         

              
## Bridge与netfilter   

pass 

## netfilter与LVS

pass

## openstack安全组实现   

pass




       


参考文章    

><small>https://wiki.khnet.info/index.php/Conntrack_tuning    
https://voipmagazine.wordpress.com/2015/02/27/tuning-the-linux-connection-tracking-system/    
https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE
</small>