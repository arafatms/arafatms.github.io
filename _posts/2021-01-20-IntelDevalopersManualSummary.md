---
layout: post
title: 【AMD64开发手册】一，系统编程摘要
date: 2021-9-2 
tags: AMD64开发手册    
---


# 系统编程摘要
此文章介绍AMD64架构的基础特性和能力。一般AMD64处理器可以运行在以下6种模式下：
- 传统模式 legacy modelong mode
    - 实模式 real mode：处理器reset或者供电时跑在实模式下，类似80286处理器，只能寻址1MB
    - 保护模式 protected mode：此模式下系统提供4GB的物理和虚拟内存，OS运行在PL0或者PL1，应用程序运行在PL3
    - virtual-8086 mode：
    - system managment mode：电源管理（ACPI）等系统管理程序
- 长模式 long mode：64位linux最后会跑在长模式下，进入长模式之前系统必须使能保护模式，转换过程见后续图
    - Compatibility mode
    - 64-bit mode

各个模式之间的差异见下图：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/OPModes.png?raw=true)

模式之间的转换见下图：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/OPModeChange.png?raw=true)


## 内存模型 Memory Model
AMD64架构内存模型给OS提供管理应用程序以及相关数据的安全方式。在长模式（long mode）下使用内存平坦模型，而在传统模式下实现了所有传统内存模型。

### 内存地址 Memory Adressing
AMD64架构提供地址重定位。为了实现这个需要了解以下四种地址类型：
- 逻辑地址 Logical address：逻辑地址用于进入段地址，一般来说逻辑地址 = 段选择器 + 偏移
- 有效地址或者段偏移 Effective address：用于从某个段描述符获取逻辑地址
- 线性地址（虚拟地址） Linear address：线性地址=段的基地址 + 有效地址
- 物理地址 Physical address：

#### 虚拟地址
OS需要负责利用页面映射来映射虚拟地址到物理地址的转换
- 保护模式下：提供32位的虚拟地址一共可以寻址4GB地址空间
- 长模式下：提供64位的虚拟地址，有些处理器可能提供小于64位的虚拟地址空间，可以用CPUID指令（EFLAGS[21]=1时可以使用）查看

#### 物理地址
用于直接访问主存，AMD64架构提供以下几种地址转换模型：
- 实地址模式：处理器跑在实模式下会用到。提供20位1MB的空间；
- 传统保护模式：可以使用段转换机制和页面转换机制，提供32位的4GB地址空间；
- 长模式：提供52位的地址空间，需要开启页面转换和PAE；

## 内存管理
内存管理包含软件产生的地址转换成物理地址的方案（段机制或者页面转换）。

### 段机制 Segmentation
源本用于隔离系统软件，一些信息等。目前很多主流的操作系统都不适用这个机制，反而使用页面保护（page-level protection）机制。因为后者更简单，高效。
如linux系统传统模式和兼容模式只是用来过渡，所以对分段机制不需要了解太多。
以上两种模式下可以提供16383个单独的段，这就会导致无法创建过多的线程。

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/SegmentedMemMode.png?raw=true)

### 内存映射
PML4 就是通过四级映射找到虚拟地址对应的物理地址。

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/PML.png?raw=true)

### 64位模式下的内存管理
64为模式下，所有段基地址都会设为0，因此有效地址=虚拟地址，成功找到虚拟地址以后利用PML4机制找到对应的物理地址。

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/64BitMemoryMode.png?raw=true)

## 系统寄存器

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210902-AMDChapter1/SystemRegisters.png?raw=true)

系统寄存器包含：
- 控制寄存器（control register）：用于控制系统操作或者特性；
- 系统标志寄存器（system-flag register）：RFLAG寄存器包含系统状态标志和掩码；
- 描述符表寄存器（descriptor table register）：用于保存描述符表的基地址，如GDT；
- task register：包含task state segment的基地址和长度；
- debug register：用于调试；
除此之外还包含一系列MSR（model-specific register）寄存器：
- EFER（Extended feature enable register）：用于使能和反馈状态一些CS寄存器没有记录的特性，比如用EFER.LME=1启动长模式；
- 其他


## 系统数据结构（system data structures）
由OS创建和维护并给处理器使用，当运行在保护模式下时这些数据结构用与管理内存和保护，以及当发生中断或者任务切换是保存信息。

<img src="/images/posts/20210902-AMDChapter1/SystemDataStructure.png" width="700px" />

系统数据结构包含：
- descriptor描述符：提供段的信息（位置，大小，PL）等给处理器。一类描述符的集合称为gate门。门用于找到对应的描述符。
- descriptor table描述符列表：用于保存一系列描述符。全局描述符列表GDT用于保存所有程序可见的描述符，本地描述符列表LDT用于保存本程序用的列表，中断描述符列表IDT用于保存被中断处理器使用的门描述符（gate descriptors）
- task state segment：用于保存某个程序的处理器信息和状态，栈的地址。任务硬件切换就是基于TSS做的。尽管不使用应切换还是得初始化起码一个TSS
- page translation tables：在长模式下必需要定义四级页面映射来支持64位虚拟地址切换至52为物理地址空间。传统保护模式下可以选择使用2，3级映射，也可以不用页面映射。

## 中断interrupts
AMD64架构提供当中断或者异常发生时，自动暂停应用执行并且切换到中断处理器的机制。中断可以被以下几种方式触发：
- 系统硬件触发
- 软件利用中断指令触发，如INITx
- 异常发生在处理器发现异常

OS不仅要设置中断处理器，而且需要创建和初始化中断相关的数据结构，如：
- 中断处理代码段描述符
- 中断处理数据段描述符
- 栈
以上代码和数据段描述符均保存在GDT or LDT。

当发生中断时，处理器利用中断向量在IDT中找到对应的interrupt gate，这个门提供中断处理器代码段和入口地址，然后处理器进行切换。切换之前处理器会保存被中断的程序上下文。
特别的：
缺页异常的向量值为14

## 附加的系统机制

### 硬件切换
为每一个任务提供了一个TSS，当需要切换时利用硬件的方式切换到对应的任务上下文中。在长模式下不使用硬件切换，因为比较不灵活，反而使用软件切换。

### Machine check

### Software debugging

### performance monitoring