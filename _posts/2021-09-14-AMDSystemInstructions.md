---
layout: post
title: 【AMD64开发手册】六，系统指令
date: 2021-9-14
tags: AMD64开发手册    
---

# System Instructions
系统指令提供对处理器执行环境的控制。处理器执行环境包括内存管理，内存保护，任务管理，中断以及异常处理，SMM，软件调试以及性能分析，一些特性。大多是指令只能在CPL=0下执行，但有一些指令可以在任何特权级下执行。

## Fast System Call and Return
OS可以使用页面映射或者分段机制来实现被保护的内存模型。此文章中只讲述64位模式下的系统调用相关内容。

### SYSCALL and SYSRET
> SYSCALL and SYSRET are low-latency system call and return instructions.
这俩指令假定OS实现了平坦内存模型（flat-memory model），因此可以大程度的简化系统调用和系统返回流程。要求代码段基地址，大小和特性在任何应用和系统程序中都是一样的（DPL除外）。处理器假定SYSCALL目标代码段描述符入口拥有DPL=0，返回时则为DPL=3.

STAR，LSTAR，CSTAR等MRS寄存器用于保存SYSCALL的目标地址和返回地址。SFMASK寄存器用于说明rFLAGS怎么被这些指令修改。
![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210914-SystemInstructions/StarLstarCstar.png?raw=true)

- STAR：STAR寄存器包含以下区域：
    - SYSRET CS and SS Selectors：系统调用返回时CS，SS的段选择索引；
    - SYSCALL CS and SS Selectors：系统调用时CS，SS的段选择索引；
    - 32-bit SYSCALL Target EIP：
- LSTAR和CSTAR：分别为64位和兼容模式下SYSCALL指令的目标RIP；
- SFMASK：表明RFLAGS中的哪些位会在调用SYSCALL过程中清理；

### SWAPGS指令
提供快速加载系统数据结构的方法。当调用SYSCALL时SWAPGS可以快速的加载内核所使用的数据结构，当SYSRET之前可以快速加载被内核数据结构覆盖掉的应用程序数据结构。SWAPGS指令会交换KernelGSbase寄存器和GS.base寄存器。
特别的GS寄存器中保存内核数据结构的指针，处理器对此没有做特定的描述。

## System Status and Control

### 访问控制寄存器
- MOV CRn:特权级下用于在控制寄存器和通用寄存器之间交换数据；
- LMSW and SMSW：用于修改CR0[15:0]中的机器状态位；
- CLTS：clear task-switched bit用于清理CR0.TS,每当有任务切换时CR0.TS就会被处理器设为1;

### RFLAG访问指令
- POPF and PUSHF：特权级下用于在RFLAG和栈之间交换数据；
- CLI and STI：clear/set interrupt，设置RFLAG.IF（0时处理器忽略可屏蔽中断）
- CLAC and STAC：clear/set alighnment check flag，设置RFLAG.AC

### MSR访问指令
- RDMSR and WRMSR：read/write model-specific register
- RDPMC: read performance monitoring counter
- RDTSC:read time-stamp counter
- RDTSCP: read time-stamp counter and processor ID

## 访问段寄存器和描述符寄存器
### 访问段寄存器
- MOV,POP,and PUSH:
### 访问隐藏的段寄存器状态
- WRMSR and RDMSR: 可以用于设置GS，FS
- RDFSBASE,RDGSBASE,WRFSBASE,WRGSBASE:
### 访问描述符列表
- LGDT and LIDT:
- LLDT and LTR:
- SGDT and SIDT:
- SLDT and STR:

## Processor Halt
使用HLT指令进入暂停状态。当有以下几种情况发生时处理器会退出hlt状态：
- NMI
- 可屏蔽中断发生(INTR)
- RESET
- INIT
- SMI

## Cache and TLB Managment
### Cache Managment
- WBINVD and WBNOINVD:writeback and invalidate(WBINVD) 回写所有cache line之后让所有cache line失效；writeback no invalidate则相反
- INVD：失效所有的cache line
### TLB Invalidation
- INVLPG：失效指定的TLB项
- INVLPGA：失效与ASID相关联的所有项
- INVPGB：范围性失效
- INVLPCID：失效与PCID相关量的所有项