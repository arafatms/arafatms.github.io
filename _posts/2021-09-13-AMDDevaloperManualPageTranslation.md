---
layout: post
title: 【AMD64开发手册】五，页面映射以及保护机制
date: 2021-9-13
tags: AMD64开发手册    
---

# Page Translation and Protection
X86 页面映射机制提供OS为应用或者处理器创建彼此隔离的地址空间。OS利用页表进行虚拟地址到物理地址的映射。页面映射机制给各个进程提供私有的物理页面区域。一个进程无法访问没有映射到自己的虚拟地址空间的物理页面。
共享映射一般用于将共享库映射到各个进程中，此过程中会使用只读共享。
用户进程必须无法访问内核使用的物理内存（supervisor page）。同样的如果设置SMAP/SMEP内核也将无法访问应用程序所使用的物理内存。如果CR4.SMAP和SMEP使能并且RFLAGS.AC=1内核(CPL=0,1,2)访问应用程序空间(CPL=3)会触发#PF（page fault）。
AMD64架构中64位模式下一般使用PML4页面映射：PML4E -> PDPE -> PDE -> PTE -> Page,PML4E基地址保存在CR3寄存器中。

## 页面映射摘要
AMD64架构定义从48位虚拟地址映射到52位物理地址的页面映射机制(保留64位映射功能)。为了支持页面映射选择性开启以下四种特性：
- Page-Translation Enable CR0.PG
- Physical-Address Extention CR4.PAE:让虚拟地址能够映射进52位物理地址
- Page-Size Extention CR4.PSE
- Long-Mode active EFER.LMA

根据模式不同所使用的特性也不通，见下图：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/SupportedPagingAlternatives.png?raw=true)

其中Page-Directory Page Size (PS) 位用来控制此页表是否处于页面转换的最底层，比如如果PDE.PS=1，PDE作为映射的最底层可以映射2MB大小的大内存页面，此方法同样用于映射1GB大页内存。

## 长模式下的页面映射
进入长模式条件有：
- CR4.PAE = 1
- CR0.PG = 1

PAE映射下新增了一项映射表称之为PML4，长模式下R4.PSE是被忽略的。

### CR3寄存器
PML4E基地址保存在CR3寄存器中。CR3寄存器结构如下：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/CR3.png?raw=true)

- 表基地址域：40位的PML4基地址，因为要4K对其所以补充12位的0即可得到52位的物理地址。
- Page-Level Writethrough(PWT):上级页表cache是否要writethrough还是writeback；
- Page-Level Cache Disable：是否缓存上级页表
- Process Context Identifier(PCID):当前处理器PCID

### 4K页面映射
话不多说见图：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/4KPML4.png?raw=true)

其中：
![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/4KPML41.png?raw=true)
![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/4KPML42.png?raw=true)

### 2M页面映射
![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/2MPML4.png?raw=true)

其中：
![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210913-AMDPageTranslation/2MPML41.png?raw=true)

可以看出2M页面映射中，PDE.PS位职位1，表示最底层页表。

## Page-Translation-Table Entry Fields

- Translation-Table Base Address Field：指向下一级页表的基地址(PA),52bit,11:0bit为0.

- Physical-Page Base Address Field：指向最终映射的页面基地址，根据页面大小的不同将地位设为0（4K映射下11:0位为0，2M映射下20:0位为0，1G映射下29:0位为0）
- Present（P）Bit：表示下级页表或者物理页面是否存在，不存在就会触发#PF，发生缺页异常时内核需要分配物理页面并将此为设为1. 当P bit = 0时TLB不会缓存此映射，并且处理器不会设置Accessed，Dirty位
- Read/Write（R/W）Bit：
- User/Supervisor（U/S）Bit：
- Accessed（A）Bit：表示页表或者物理页面是否被访问过，访问时处理器会设置此位，但由内核清理；
- Dirty（D）Bit：仅在最底层页表中存在，表示PTE对应的页面被改写过，由处理器设置，内核清理；A和D位可能在处理器运行乱序执行和分支预测时设为1；
- Page Size（PS）Bit：表示为最底层页表
- Global Page（G）Bit：表示物理页面为全局页面，当更新CR3寄存器时这类页面的TLB不会失效，需要CR4.PGE=1
- Available to Software(AVL)Bit:系统软件使用位
- Page-Attribute Table(PAT)BIT:PAT寄存器索引
- Memory Protection Key(MPK)Bit:当CR4.PKE=1时，这四位索引选择一个内存保护Key。PKRU寄存器中一般保存多种权限组合。
- No Execute（NX）Bit：当EFER.NXE=1时有效

## Translation-Lookaside Buffer(TLB)
TLB用来消除查询四级页表带来的性能耗损，任何内存引用都会检查TLB，如果存在映射关系，就立刻向处理器返回物理地址，而不是去查询PML4。我们最希望任何一条映射关系都能在TLB中查找（On-chip）。
内核需要负责更新TLB上的任何一条linear-to-physical映射关系。映射关系中任何数据结构的更改不会自动的更新TLB，所以内核需要在TLB中失效（invalid）掉有修改的页表。
> 只有CPL=0的程序能操作TLB

### Process Context Identifier（PCID）
早期的处理器中，更新CR3寄存器（switch context）都会刷新整个TLB，这使得频繁任务切换中TLB命中率很低。当使能PCID（CR4.PCIDE=1）后处理器允许TLB缓存多种虚拟地址空间映射关系。处理器会把每一项TLB映射与当前PCID（CR3.PCID)做关联，这样在地址转换过程中只选择使用与当前PCID所匹配的TLB映射。
内核负责往CR3寄存器写入12位的PCID，如果内核切换任务（更新CR3中页表基地址），处理器可能会使用之前的PCID所对用的映射关系。
PCID是个很美好的新技术，但也存在一定的问题。加入PCID概念后没有人主动刷新TLB，同时一个任务可能会在多个处理器中调度，这时这个进程的映射关系可能会存在在多个核的TLB中。当需要修改或者删除页表时光是干掉本地的TLB项是不行的，还需要往别的核发送IPI来失效掉对应的TLB项。另外一个问题是只能最多有4096个PCID，所以内核需要管理PCID。

### Global Pages
更新CR3会显示或者隐适的刷新TLB，接下来的一些内存引用会消耗多个时钟来查表，直到新的项加载到TLB中。可以将物理页面设为Global来避免这类问题。linux中内核所使用的物理页面都是全局页面。
> CR4.GPE = 1生效，INVLPG刷新TLB中的项

### TLB管理
