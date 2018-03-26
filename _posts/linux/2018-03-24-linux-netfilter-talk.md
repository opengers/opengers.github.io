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

ok，图中长方形小方框已经解释清楚了，还有一种椭圆形的方框`conntrack`，即Connection Tracking，这是netfilter框架中的连接跟踪机制，连接跟踪允许内核持续跟踪通过此处的所有网络连接，从而将所有构成该连接的数据包关联起来，通俗点说，conntrack机制能够"审查"通过此处的所有网络数据包，并把数据包按网络连接相关联(比如数据包a属于`IP1:8888->IP2:80`这个tcp连接，数据包b属于`ip3:9999->IP4:53`这个udp连接),图中可以清楚看到连接跟踪机制所处的网络栈位置，因此如果不想让某个数据包被跟踪(`NOTRACK`),那就要找位于椭圆形方框`conntrack`之前的表和链来设置规则。conntrack机制是iptables实现状态匹配以及NAT的基础，它由单独的内核模块`nf_conntrack`实现，iptables中可以用`-m state`来使用改模块进行网络连接状态匹配。下面还会详细介绍          
 
接着看图中左下方`bridge check`方框，数据包从主机上的某个网络接口进入(`ingress`), 在`bridge check`处会检查此网络接口是否属于某个Bridge的port，如果是就会进入Bridge代码处理逻辑(`broute`->...), 否则就会送入网络层处理(`raw`-->...)       

图中下方中间位置的`bridging decision`环节就是根据数据包目的MAC地址判断此数据包是转发还是交给上层处理，这个另一篇文章也有解释[linux-bridge](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/#linux-bridge)       

图中中心位置的`routing decision`就是路由选择，根据系统路由表(`ip route`查看), 决定数据包是forward，还是交给本地处理       

总的来看，图中整个packet flow表示的是数据包总是从主机的某个接口进入(左下方`ingress`), 然后经过一系列表和链处理，最后，或是交由主机上某应用进程处理(上中位置`local process`)，或是从主机另一个接口发出(右下方`egress`),这里的接口即可以是物理网卡em1，也可以是虚拟网卡tun0/vnetx，还可以是Bridge上的一个port。   

上面就是关于这张图的一些解释，如果还有其它疑问，欢迎留言讨论，下面说说netfilter的应用方面            

未完待续...

# Connection Tracking     

pass

# Bridge与netfilter   

pass 

# netfilter与LVS

pass

# openstack安全组实现   

pass




       

