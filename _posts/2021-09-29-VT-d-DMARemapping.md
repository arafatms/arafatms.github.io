---
layout: post
title: 【Intel-VT_d-Spec】一，DMA重映射
date: 2021-9-24
tags: Intel-VT_d-Spec    
---
 
This chapter describes the hardware architecture for DMA remapping. The architecture envisions remapping hardware to be implemented in the Root-Complex integrated into the Processor complex or in core logic chipset components.

## DMA请求类型
Remapping hardware treats inbound memory requests from root-complex integrated devices and PCI Express* attached discrete devices into two categories:
- Requests without address-space-identifier: These are the normal memory requests from endpoint devices (Requests-without-PASID)
- Requests with address-space-identifier: with additional informations (Requests-with-PASID)
    - targeted process address space identifier (PASID)
    - Execute-Requested (ER) flag
    - Privileged-mode-Requested (PR) flag

可以看出DMA重映射的主要作用对象是Requests-with-PASID，使用唯一的 PASID 标记 DMA 流的请求可实现 I/O 设备的可扩展和细粒度共享，以及具有主机应用程序虚拟内存的设备操作。

## Domains and Address Translation
A domain is abstractly defined as an isolated environment in the platform, to which a subset of the host physical memory is allocated. I/O devices that are allowed to access physical memory directly are allocated to a domain and are referred to as the domain’s assigned devices. For virtualization usages, software may treat each virtual machine as a domain.

The isolation property of a domain is achieved by blocking access to its physical memory from resources not assigned to it. Multiple isolated domains are supported in a system by ensuring that all I/O devices are assigned to some domain (possibly a null domain), and that they can only access the physical resources allocated to their domain. The DMA remapping architecture facilitates flexible assignment of I/O devices to an arbitrary number of domains. Each domain has a view of physical address space that may be different than the host physical address space. Remapping hardware treats the address in inbound requests as DMA Address. Depending on the software usage model, the DMA address space of a device (be it a Physical Function, SR-IOV Virtual Function, or Intel® Scalable IOV Assignable Device Interface (ADI)) may be the Guest-Physical Address (GPA) space of a virtual machine to which it is assigned, Virtual Address (VA) space of host application on whose behalf it is performing DMA requests, Guest Virtual Address (GVA) space of a client application executing within a virtual machine, I/O virtual address (IOVA) space managed by host software, or Guest I/O virtual address (GIOVA) space managed by guest software. <u> In all cases, DMA remapping transforms the address in a DMA request issued by an I/O device to its corresponding Host-Physical Address (HPA). </u>

<img src="/images/posts/20210929-DMARemapping/DMA-Address-Translation.png" width="700px" />

The host platform may support one or more remapping hardware units. Each hardware unit supports remapping DMA requests originating within its hardware scope. The architecture supports configurations in which these hardware units may either share the same translation data structures (in system memory) or use independent structures, depending on software programming.

The remapping hardware translates the address in a request to host physical address (HPA) before further hardware processing.

## Remapping Hardware - Software View
For example, an implementation may expose a remapping hardware unit that supports one or more integrated devices on the root bus, and additional remapping hardware units for devices behind one or a set of PCI Express root ports.

## Mapping Device to Domains
### Source Identifier
简称BDF：
- Bus [15:8]
- Device [7:3]
- Function [2:0]

### Legacy Mode Address Translation

<img src="/images/posts/20210929-DMARemapping/LegacyMode.png" alt="ssss" width="700px" />

The root-table functions as the top level structure to map devices to their respective domains. The location of the root-table in system memory is programmed through the <u> Root Table Address Register RTADDR_REG </u>

- Root Entry:
<img src="/images/posts/20210929-DMARemapping/RootEntry.png" width="700px" />

### Scalable Mode Address Translation
For implementations supporting Scalable Mode Translation (SMTS=1 in Extended Capability Register), the Root Table Address Register (RTADDR_REG) points to a scalable-mode root-table when the Translation Table Mode field in the RTADDR_REG register is programmed to scalable mode (RTADDR_REG.TTM is 01b).

<img src="/images/posts/20210929-DMARemapping/ScalableModeMapStructure.png" width="700px" />

## Hierarchical Translation Structures
DMA remapping uses hierarchical translation structures for both first-level translation and secondlevel
translation. 这相当于MMU的PML4或者PML5。

<img src="/images/posts/20210929-DMARemapping/IOMMUPML4.png" width="700px" />

## First-Level Translation
Scalable-mode context-entries can be configured to translate requests (with or without PASID) using first-level translation. Requests-without-PASID use the PASID value configured in the RID_PASID field in the scalable-mode context-entry to process the request.

> First-level translation supports the same paging structures as Intel® 64 processors when operating in 64-bit mode.

<img src="/images/posts/20210929-DMARemapping/FirstLevelPagingStructure.png" width="700px" />

### Access Rights
Requests can result in first-level translation faults for either of two reasons: (1) there is no valid translation for the input address; or (2) there is a valid translation for the input address, but its access rights do not permit the access.
The accesses permitted for a request whose input address is successfully translated through first-level translation is determined by the attributes of the request and the access rights specified by the paging-structure entries controlling the translation.

## Second-Level Translation
Context entries and scalable-mode PASID-Table entries can be configured to support second-level translation. With context entries, second-level translation applies only to requests-without-PASID. With scalable-mode PASID-Table entries, second-level translation can be applied to all requests (with or without PASID), and can be applied nested with first-level translation for all requests (with or without PASID). This section describes the use of second-level translation without nesting.
Each context entry contains a pointer to the base of a second-level translation structure.
<img src="/images/posts/20210929-DMARemapping/SecondLevelPageingStructure.png" width="700px" />

关于权限和A，D Bit见intel手册，但基本上与MMU是一样的。

## Nested Translation
When PASID Granular Translation Type (PGTT) field is set to 011b in scalable-mode PASID-tableentry, requests translated through first-level translation are also subjected to nested second-level translation.

<img src="/images/posts/20210929-DMARemapping/Nested.png" width="700px" />

Just like root-table and context-table in legacy mode address translation are in host physical address, scalable-mode root/context/PASID-directory/PASID-tables are in host physical address and not subjected to first or second level translation.
> With nested translation, a guest operating system running within a virtual machine may utilize firstlevel translation, while the virtual machine monitor may virtualize memory by enabling nested second-level translations.

## Memory Types
The memory type of a memory access (to a translation structure entry or access to the mapped page) refers to the type of caching used for that access. Refer to Intel® 64 processor specifications for definition and properties of each supported memory-type (UC, UC-, WC, WT, WB, WP). Support for memory typing in remapping hardware is reported through the Memory-Type-Support (MTS) field in the Extended Capability Register.
- Memory-type 对来自在处理器一致性域之外运行的设备的内存访问没有意义（因此被忽略）。

也一样有PAT和MTRR，不做解释。

## Device-TLB Operation
Device-TLB support in endpoint devices requires standardized mechanisms to:
- Request and receive translations from the Root-Complex
- Indicate if a memory request (with or without PASID) has a translated or un-translated address
- Invalidate translations cached at Device-TLBs.

<img src="/images/posts/20210929-DMARemapping/DeviceTLBOperations.png" width="700px" />

Figure 4-13 illustrates the basic interaction between the Device-TLB in an endpoint and remapping hardware in the Root-Complex, as defined by Address Translation Services (ATS) in PCI Express Base Specification.

ATS defines the ‘Address Type’ (AT) field in the PCI Express transaction header for memory requests. The AT field indicates if transaction is a memory request with ‘Untranslated’ address (AT=00b), ‘Translation Request’ (AT=01b), or memory request with ‘Translated’ address (AT=10b). ATS also define Device-TLB invalidation messages. Following sections describe details of these transactions.

### Translation Request
Translation-requests-without-PASID specify the following attributes that are used by remapping hardware to process the request:
- Address Type (AT)
- Address: Input Address
- Length: Length field indicates how many sequential translations may be returned in response to this request.
- No Write (NW) flag:

Translation-requests-with-PASID specify the same attributes as above, and also specify the additional
attributes (PASID value, Execute-Requested (ER) flag, and Privileged-mode-Requested (PR) flag) in
the PASID TLP Prefix.

### Translation Completion
如果转换不成功就返回一个带如下错误的Translation Completion
- Unsupported Request
- Completer Abort

如果成功转换requests-without-PASID请求时返回：
- Size：
- Non-Snooped access flag (N):
- Untranslated access only flag (U):
- Read permission：
- Write permission：
- Translated Address

如果成功转换带PASID的请求时就外加一下几种返回值：
- Execute permission (EXE):
- Privilege Mode Access (PRIV):
- Global Mapping （就像Global page一样？）

### Translated Request
Translated-requests are regular memory read/write/atomics requests with Address Type (AT) field value of 10b. When generating a request to a given input (untranslated) address, the endpoint may look in the local Device-TLB for a cached translation (result of a previous translation-request) for the input address. If a cached translation is found with appropriate permissions and privilege, the endpoint may generate a translated-request (AT=10b) specifying the Translated address obtained from the Device-TLB lookup. Translated-requests are issued as requests-without-PASID.

### Invalidation Request & Completion
Invalidation requests are issued by software through remapping hardware to invalidate translations cached at endpoint Device-TLBs.
Invalidation-requests-without-PASID specify the following attributes:
- Device ID
- Size (S):
- Untranslated Address

如果是Invalidation-requests-with-PASID就会外加一个global-invalidate flag. If the global-invalidate flag is 1, the invalidation affects all PASID values.

## Device TLB in System-on-Chip (SoC) integrated devices
The Figure 4-14 below shows a typical SoC integrated device that is connected to the Root Complex. All such devices that support Address Translation Service (ATS) must create an isolation boundary outside of which Host Physical Address (HPA) are not exposed. The part of the device outside the isolation boundary is referred to as “Device Core”.
<img src="/images/posts/20210929-DMARemapping/DevTLBWithSOC.png" width="700px" />
Depending on the software usage model the Device Core may receive various kinds of address from software. These addresses are translated by Device TLB (i.e. DevTLB) into HPAs using ATS.
All SoC integrated devices must meet the following requirements:
- HPA 隔离边界不得将转换后的地址（来自 DevTLB）暴露给设备核心 The HPA isolation boundary must not expose translated addresses (from DevTLB) to the Device Core
- 将 SoC 集成设备连接到 Root Complex 的接口必须完全包含在 HPA 隔离边界内。
- The Device Core must not be able to put a transaction on the interface(s) that connects device to the Root Complex. The Device Core must always request that the HPA isolation partition send transaction to the Root Complex.
- The HPA isolation partition must host the DevTLB and must faithfully follow the ATS specification.
- The Device Core can specify that a transaction bypass the DevTLB; in that case, the HPA isolation partition will put the transaction as an “Untranslated Request” on the I/O fabric.
- The Device Core (including software/firmware running on the device) must not be able to manipulate (read or modify) the contents of DevTLB or the data cache.
- If a device needs a data cache tagged with translated addresses (from DevTLB), such a cache must be hosted in the HPA isolation boundary.

SoC-integrated devices connecting to the Root Complex using the System Fabric must meet the following additional requirements:
- Before sending an ATS Invalidate Completion message to the IOMMU, the HPA isolation partition must ensure that all transactions using stale information on the System Fabric have reached the global observation point.
- When DMA remapping is enabled in the IOMMU, any access on the System Fabric from the HPA isolation partition must use HPA.
- The Root Complex may implement a product specific mechanism to inform the HPA isolation partition in each SoC integrated device that DMA remapping is enabled. 以通知每个 SoC 集成设备中的 HPA 隔离分区启用了 DMA 重映射

## Guidance to software on enabling and disabling ATS
Enabling ATS for a device requires software to program two independent bits and hence it can not be an atomic operation and there will be a window of time where the system is an inconsistent state. It is strongly recommended that software quiesce DMA operations from the device before programming the required bits.

### Recommended software sequence to enable ATS
1. Software should quiesce all DMA operations from the device.
2. Software should program the scalable-mode context entry (or context entry) associated with the DevTLB to allow ATS requests.
3. Software should program Enable (E) field, in the ATS Control register of the device with a value of 1.

### Recommended software sequence to disable ATS
1. Software should quiesce all DMA operations from the device.
2. Software should program Enable (E) field, in the ATS Control register of the device with a value of 0.
3. Software should issue global Device-TLB Invalidate descriptor followed by Invalidation Wait descriptor to invalidate all translations from the DevTLB and also to drain all prior ATS traffic to the global observation point.
4. Software should program the scalable-mode context entry (or context entry) associated with the DevTLB to block ATS requests.