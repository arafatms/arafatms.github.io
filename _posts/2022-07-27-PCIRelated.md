---
layout: post
title: 【PCIe】一，PCI基础知识总结
date: 2022-6-17
tags: PCIe
---

内容均来自[转载http://blog.chinaaet.com/justlxy/p/5100057779](http://blog.chinaaet.com/justlxy/p/5100057779)以及笔者阅读PCI spec后的补充

# PCI架构综述
## 基于PCI总线的结构

<img src="/images/posts/20220727-PCIRelated/1_2.png" width="700px" />

该平台包含三级总线: FSB(Front-side-bus)、 PCI 和 ISA， FSB是处理器子系统的总线，原称为Host总线，总线 定义完全取决于系统所使用的处理器； PCI局部总线是一个完全与处理器无关的总线， 不受制于系统所使用的微处理器的种类； ISA总线亦称为IO扩展总线，在实际系统中， 也有采用EISA或MC总线的，以ISA总线用得较多。 不同的总线之间通过相应的桥芯 片来隔离和转接。

平台中两级桥是必须具有的，一是HOST到PCI的桥（常称为主桥-Host桥） ，即 图中的北桥；另一个是PCI到扩展IO总线的桥（常称为扩展总线桥), 即图中的南桥。

PCI总线是一种树型结构，并且独立于CPU总线，可以和CPU总线并行操作。PCI总线上可以挂接PCI设备和PCI桥，PCI总线上只允许有一个PCI主设备（同一时刻），其他的均为PCI 从设备，而且读写操作只能在主从设备之间进行，从设备之间的数据交换需要通过主设备中转。

> 注：这并不意味着所有的读写操作都需要通过北桥中转，因为PCI总线上的主设备和从设备属性是可以变化的。比如Ethernet和SCSI需要传输数据，可以通过一种叫做Peer-to-Peer的方式来完成，此时Ethernet或者SCSI则作为主机，其它的设备则为从机。具体会在后面的博文中详细介绍。

一个典型的33MHz的PCI总线系统如上图所示，处理器通过FSB与北桥相连接，北桥上挂载着图形加速器（显卡）、SDRAM（内存）和PCI总线。PCI总线上挂载着南桥、以太网、SCSI总线（一种老式的小型机总线）和若干个PCI插槽。CD和硬盘则通过IDE连接至南桥，音频设备以及打印机、鼠标和键盘等也连接至南桥，此外南桥还提供若干的USB接口。南桥上还挂有ISA总线,为慢速设备提供兼容接口，称之为SIO桥（System IO）。

PCI总线是一种共享总线，所以需要特定的仲裁器（Arbiter）来决定当前时刻的总线的控制权。一般该仲裁器位于北桥中，而仲裁器（主机）则通过一对引脚，REQ#（request） 和GNT# （grant）来与各个从机连接。

需要注意的是，并不是所有的设备都有能力成为仲裁器（Arbiter）或者initiator 。

最初的PCI总线的时钟频率为33MHz，但是随着版本的更新，时钟频率也逐渐的提高。但是由于PCI采用的是一种Reflected-Wave Signaling信号模型（后面会详细的介绍），导致了时钟频率越高，总线的最大负载越少，如下图所示：

<img src="/images/posts/20220727-PCIRelated/1_3.png" width="700px" />