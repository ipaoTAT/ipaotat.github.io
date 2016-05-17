---
layout: post
title: Quagga OSPF模块源码简析
category: 项目
tags: [Gears, 源码，Quagga]
comments: true
---


##Quagga OSPF模块源码简析

###1. Quagga简介

<!-- more -->
Quagga软件原名是Zebra是由一个日本开发团队编写的一个以GNU版权方式发布的软件。当前版本是0.99.23。Quagga能够同时支持RIPv1、RIPv2、RIPng、OSPFv2、OSPFv3、BGP-4和 BGP-4+等诸多TCP/IP协议。
Quagga的特性：
**A） 模块化设计：**Quagga基于模块化方案的设计，即对每一个路由协议使用单独的守护进程。
**B） 运行速度快：**因为使用了模块化的设计，使得Quagga的运行速度比一般的路由选择程序要快。
**C） 可靠性高：**在所有软件模块都失败的情况下，路由器可以继续保持连接并且daemons也会继续运行。故障诊断不必离线的状态下被诊断和更正。
**D） 持Ipv6：**Quagga不仅支持Ipv4，还支持Ipv6。
###2. Quagga OSPF模块结构
Quagga OSPF模块的基本结构如下图：


![quagga ospf 结构图](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/quagga_ospf.png)


###3. Quagga OSPF各子模块结构及其相互调用细节
**1.ospf_read**

![ospf_read](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/ospf_read.png)

ospf_read函数是ospf模块底层一个重要的函数，它的主要工作是调用ospf_recv_packet函数读取接收到的消息，并根据消息类型交给不同的函数进行处理。
Ospf消息有五种：
第一种为***Hello***报文，用来发现和维持邻居关系，选举DR，BDR。
第二种为***DD***报文，用来描述本地LSDB的情况，装载所有LSA的Head。
第三种为***LSR***报文，用来向对端请求本地没有的LSA或更新的LSA。
第四种为***LSU***报文，用来向对方更新LSA。
第五种为***LSAck***报文，用来收到更新LSA后的确认。

**2.邻居发现及状态管理**

![ospf_find_nbr](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/ospf_find_nbr.png)

osfp_hello是一个非常重要的函数。它的主要职责是处理Hello包，也就是对邻居关系进行管理，切换邻居状态机，出发响应ospf操作。
邻居状态机有如下八种状态：
1. down
刚启动OSPF进程，还未收到邻居的任何信息。
2. attempt
只发生NBMA网络中，使用单播更新，发送HELLO分组，但从邻居没有收到任何信息。
3. init
只有一方收到的另一方的HELLO数据包，并且在邻居字段中收到对方的route-id.
4. two-way
本路由器收到对方的HELLO数据包，并且在邻居字段中看到自己的route-id.
5. exstart
在交换DBD之前阶段选出主/从路由器。
6. exchange
完成协商交换DBD.
7. loading
向对方发送LSA请求分组确定自己少哪些LSA，发送LSU告诉对方自己详细的LSA信息。并用LSA ACK确认。
8. full
数据同步，完成邻接关系。

**3.LSDB管理**

![ospf_lsdb_mgr](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/lsdb_mgr.png)

由于lsa/lsdb的管理函数众多，上图只节选其中比较有代表性的几个。Lsdb管理模块的主要职责是对lsdb中的lsa进行增删查，这些操作的调用者，可能是ospf消息处理函数、spf路由计算模块或者定时更新处理。

**4.spf路由计算**

![ospf_spf_calculate](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/spf.png)

Spf模块的主要功能是计算路由，并安装到ospf的RIB中。

**5.路由表管理**

Quagga不直接使用底层的路由表或者硬件转发表，它自己维护的路由表是RIB表，包含除主机路由以外的所有路由（因为主机路由是通过ARP学到的，而ARP是OS底层实现的）。RIB表包括直连路由，静态路由，动态路由。而OS维护的表是转发表，即FIB表，Quagga负责维护RIB与FIB间的同步，也负责各种路由协议的路由表与RIB间的同步；Quagga提供了几种方式与内核通信，类UNIX系统下的常见方式ioctl, sysctl, proc, netlink都有支持。RIB表是radix树结构，而FIB表是hash table结构。radix是一个二叉树，如下图所示：

![rib_tree](/post_imgs/2014-11-25-Quagga_OSPF模块源码简析/rib_tree.png)

 该radix二叉树用结构休route_table表示，根节点用结构体route_node表示，包括四个成员，表示前缀的prefix结构体，左孩子、右孩子和表示下一路的info指针，info可以是下一跳的接口，或者IP，也可以是黑洞，分别对应着三种路由目标的类型：IFNAME, GATEWAY, BLACKHOLE。
