---
layout: post
title: 【AMD64开发手册】七，内存系统
date: 2021-9-17
tags: AMD64开发手册    
---

# Memory System
本章主要讲述以下内容
- Cache coherency
- Cache control
- Memory typing
- Memory mapped IO
- Memory ordering rules
- Serializing instructions

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/processorAndMemorySystem.png?raw=true)

主存位于处理器芯片之外，而且离处理器执行单元较远。所以引入缓存Cache，缓存可以位于处理器内部或者外部。
- L1 Data Cache: 缓存最常访问的数据
- L1 Instruction Cache: 缓存最常执行的指令
- L2 Cache:比L1缓存大，但是更慢，可以缓存指令或者数据

内存访问首先会检查缓存，如果该数据或者指令在缓存里则直接使用（read/write hit),否则需要从主存访问(read/write miss). 缓存由多个对其的块组成称为cache line。比如 如果cache line的大小为32B，则0007h和001Eh的数据都在一个cache line里。cache line大小一般为32/64B，可以利用CPUID Fn8000_0005查看

处理器加载数据到cacheline称为填充cacheline，如果只需要加载1B的数据，整个cacheline都得从主存加载。外部设备也可以处理器的外部探头（external probe）查看缓存中的最新的数据。

写缓存（write buffer）当主存或者缓存忙绿的时候暂存一些写请求。

write-combining buffers：缓存一些对应用程序不仅要的不可缓存写请求，然后对多个请求进行合并并下发。

## Single-Processor Memory Access Ordering
内存访问顺序的灵活性跟处理器执行和退出指令的调节能力密切相关。指令执行（execute）创建结果和状态并确定指令是否导致异常。指令的推出（retire）顺序提交指令执行结果到软件可见资源，如内存，缓存，WCB，寄存器，或者引起一个异常。
指令的推出必须是顺序的，而执行在数据依赖性前提下可以是乱序的。当然处理器也可以猜测性的执行指令。

一般的读请求不会影响程序的顺序：
- 乱序读是被允许的
- 猜测性读是被允许的
- 目标地址不同时读请求可以被放在写请求之前 reorder
- 目标地址一样时读请求不能reorder写请求之前。读请求必须等待写请求的完成

写入会影响程序顺序，因为它们会影响软件可见资源的状态。 以下规则控制写入顺序：
- 一般写入是不能乱序的
- 写入组合类型（write combining）可以是乱序的
- 猜测性写是不允许的
- 写缓存是允许的

当程序执行顺序被严格限制时程序可以使用以下指令来限制处理器的行为：
- LFENCE, SFENCE, MFENCE

## Multiprocessor Memory Access Ordering
为了提高应用程序的性能，AMD64处理器可以推测性地执行无序的指令并暂时保留无序的结果。 但是，对于在自然对齐的 WB 内存边界上的正常可缓存访问，遵循某些规则。

- 来自单个处理器的所有加载、存储和 I/O 操作似乎都是按照程序顺序发生的，在该处理器上运行的代码，所有指令似乎都是按照程序顺序执行的。
- 来自单个处理器的连续写入按程序顺序的提交给系统内存并且按程序顺序对其他处理器可见。写入不能在读取同样位置之前提交。
- 。。。。。。。

## Memory Coherency and Protocol
支持缓存的实现支持缓存一致性协议，用于维护主内存和缓存之间的一致性。 缓存一致性协议还用于维护多处理器系统中所有处理器之间的一致性。 AMD64 架构支持的缓存一致性协议是 MOESI（修改的、拥有的、独占的、共享的、无效的）协议：
- Invalid: 当前处理器中失效，有效的副本可能在主存或者在其他处理器缓存中；
- Exclusive：保存着最新，最正确的数据副本（和主存一致），其他处理器没有此数据的副本；
- Shared：所有处理器都共享着最新的数据副本，如果没有其他处理器中的标志是owned的话，主存中的数据也是最新的；
- Modified：被修改的副本，主存中的数据是错误的。没有其他处理器拷贝此副本；
- Owned：跟Shared一样，区别在于Owned时，主存数据是错误的。只有一个处理器can own。

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MOESI.png?raw=true)

为了保持内存一致性，外部总线主控器（通常是具有自己内部缓存的其他处理器）需要在内部缓存数据之前获取最新的数据副本。 该副本可以在主内存中或在其他总线主控设备的内部缓存中。 当外部总线主控器有缓存读取未命中或写入未命中时，它会探测其他主控设备以确定最新的数据副本是否保存在它们的任何缓存中。 如果其他主控设备之一拥有最新的副本，则它将其提供给请求设备。 否则，最新的副本由主存储器提供。

## Memory Types
Memory type is an attribute that can be associated with a specific region of virtual or physical memory.Memory type designates certain caching and ordering behaviors for loads and stores to addresses in that region. Most memory types are explicitly assigned, although some are inferred by the hardware from current processor state and instruction context.

The AMD64 architecture defines the following memory types:
- Uncacheable (UC)
- Cache Disable (CD)
- Write-Combining (WC)
- Write-Combining Plus (WC+)
- Write-Protect (WP)
- Writethrough (WT)
- Writeback (WB)

## Memory Caches
The AMD64 architecture supports the use of internal and external caches. The size, organization, coherency mechanism, and replacement algorithm for each cache is implementation dependent. Generally, the existence of the caches is transparent to both application and system software. In some cases, however, software can use cache-structure information to optimize memory accesses or manage memory coherency. Such software can use the extended-feature functions of the CPUID instruction to gather information on the caching subsystem supported by the processor.

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MemoryCaches.png?raw=true)

Each cache line consists of three parts: a cache-data line (a fixed-size copy of a memory block), a tag, and other information. Rows of cache lines in the cache array are sets, and columns of cache lines are ways.
- Index—The index field selects the cache set (row) to be examined for a hit. All cache lines within the set (one from each way) are selected by the index field.
- Tag—The tag field is used to select a specific cache line from the cache set. The physical-address tag field is compared with each cache-line tag in the set. If a match is found, a cache hit is signalled, and the appropriate cache line is selected from the set. If a match is not found, a cache miss is signalled.
- Offset—The offset field points to the first byte in the cache line corresponding to the memory reference. The referenced data or instruction value is read from (or written to, in the case of memory writes) the selected cache line starting at the location selected by the offset field.

MOESI state and the state bits associated with the cache-replacement algorithm are typical pieces of information kept with the cache line

## Memory-Type Range Registers
This section describes a control mechanism that uses a set of programmable model-specific registers (MSRs) called the memory-type-range registers (MTRRs). The MTRR mechanism provides system software with the ability to manage hardware-device memory mapping. System software can characterize physical-memory regions by type (e.g., ROM, flash, memory-mapped I/O) and assign hardware devices to the appropriate physical-memory type.

所有MTRR寄存器都为MSR寄存器,MTRR有以下几种类型Type：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MTRRType.png?raw=true)

### Fixed-Range MTRRs
固定区域分类，一般控制最低1M内存。

### Variable-Range MTRRs
一共由8组MSR寄存器构成

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MTRRBase.png?raw=true)

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MTRRMask.png?raw=true)

和一个全局寄存器控制默认值

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/MTRRDefault.png?raw=true)

## Page-Attribute Table Mechanism
The page-attribute table (PAT) mechanism extends the page-table entry format and enhances the capabilities provided by the PCD and PWT page-level cache controls. PAT (and PCD, PWT) allow memory-type characterization based on the virtual (linear) address. The PAT mechanism provides the same memory-typing capabilities as the MTRRs but with the added flexibility of the paging mechanism. Software can use both the PAT and MTRR mechanisms to maximize flexibility in memory-type control.
控制虚拟内存的属性，一般在PML4中最低层指定类型，类型有如下几种：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/PAT.png?raw=true)


## Memory-Mapped IO
Processor implementations can independently direct reads and writes to either system memory or memory-mapped I/O. The method used for directing those memory accesses is implementation dependent. In some implementations, separate system-memory and memory-mapped I/O buses can be provided at the processor interface. In other implementations, system memory and memory-mapped I/O share common data and address buses, and system logic uses sideband signals from the processor to route accesses appropriately. Refer to AMD data sheets and application notes for more information about particular hardware implementations of the AMD64 architecture.

The I/O range registers (IORRs), and the top-of-memory registers allow system software to specify where memory accesses are directed for a given address range

### Extended Fixed-Range MTRR Type-Field Encodings
The fixed-range MTRRs support extensions to the type-field encodings that allow system software to direct memory accesses to system memory or memory-mapped I/O. The extended MTRR type-field encodings use previously reserved bits 4:3 to specify whether reads and writes to a physical-address range are to system memory or to memory-mapped I/O.

- WrMem:if 1 -> write to system memory; else -> writes are directed to memory-mapped IO;
- RdMem:if 1 -> read to system memory; else -> reads are directed to memory-mapped IO;

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/ExtendedMTTR.png?raw=true)

此功能也需要打开一些MSR特性：
- MtrrFixDramEn
- MtrrFixDramModEn

### IORRs
The IORRs operate similarly to the variable-range MTRRs. The IORRs specify whether reads and writes in any physical-address range map to system memory or memory-mapped I/O. Up to two address ranges of varying sizes can be controlled using the IORRs. A pair of IORRs are used to control each address range: IORRBasen and IORRMaskn (n is the address-range number from 0 to 1).

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/IORRBase.png?raw=true)

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/IORRMask.png?raw=true)

### Top of Memory
The top-of-memory registers, TOP_MEM and TOP_MEM2, allow system software to specify physical addresses ranges as memory-mapped I/O locations. Processor implementations can direct accesses to memory-mapped I/O differently than system I/O, and the precise method depends on the implementation. System software specifies memory-mapped I/O regions by writing an address into each of the top-of-memory registers.
The memory regions specified by the TOP_MEM registers are aligned on 8-Mbyte boundaries as follows:
- Memory accesses from physical address 0 to one less than the value in TOP_MEM are directed to system memory.
- Memory accesses from the physical address specified in TOP_MEM to FFFF_FFFFh are directed to memory-mapped I/O.
- Memory accesses from physical address 1_0000_0000h to one less than the value in TOP_MEM2 are directed to system memory.
- Memory accesses from the physical address specified in TOP_MEM2 to the maximum physical address supported by the system are directed to memory-mapped I/O.

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210917-AMDMemorySystem/TopOfMem1.png?raw=true)

> 后面的关于Secure Memory Encryption的内容想不看了