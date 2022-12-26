---
layout: post
title: 【Intel-VT_d-Spec】二，Interrupt Posting
date: 2022-12-26
tags: Intel-VT_d-Spec    
---
 
[参考1](https://blog.csdn.net/lindahui2008/article/details/84162715?spm=1001.2014.3001.5502)

[参考2 Intel VT-d spec chapter 5]()

Interrupt-posting是VT-d中中断重映射功能的一个扩展功能，该功能也是针对可重映射的中断请求。Interrupt-posting功能让一个可重映射的中断请求能够被暂时以一定的数据形式post（记录）到物理内存中，并且可选择性地主动告知CPU物理内存中暂时记录着pending的中断请求。

在x86处理器的虚拟化中，Interrupt-posting再加上APIC Virtualization让VMM能够更加高效地处理分配各虚拟机的设备产生的中断请求。VT-d重映射硬件是否支持Interrupt-posting功能可以通过查询Capability Register的bit 59 Posted Interrupt Support（PI）知道硬件是否支持该功能。

## IRTE for posted interrupt
对于VT-d而言，所有可重定向的中断都需要经过IRTE（Interrupt Remapping Table Entry）的处理，在进行处理之前会先通过IRTE的bit 15（IRTE Mode）判断该IRTE的模式，如果为0，则VT-d硬件将会以Remapped Interrupt的形式来解析该IRTE（前一篇文章讲的中断重映射），如果为1，则VT-d硬件将会以Posted Interrupt的形式来解析该IRTE，如下图所示：

<img src="/images/posts/20221226-InterruptPosting/p1.png" width="700px" />

Posted Interrupt格式的IRTE的定义会有一些不同，主要是：
- 多了一个Posted Descriptor Address Low/High，该区域保存一个指向内存的指针，该指针指向的位置就是保存中断（Posted Interrupt）的结构体。
- Urgent位，该位用于表示该中断是否是紧急的，是否需要目标CPU的立即响应。
- Vector用于设置Posted Interrupt Descriptor数据结构中相应的位，而不是用于设置直接产生的中断的vector值。

### Posted Interrupt Descriptor

每个Posted Interrupt Descriptor的大小为64 Byte，用于记录VT-d硬件那边post过来的中断，即在内存中暂时记录一些中断请求，等待CPU的处理，而不是主动打断CPU的运行，让其跳转过来处理，其格式如下所示, 各个描述符必须又OS分配内存（WB类型内存）。

<img src="/images/posts/20221226-InterruptPosting/p2.png" width="700px" />

```
/* Posted-Interrupt Descriptor */
struct pi_desc {
	u32 pir[8];     /* Posted interrupt requested */
	union {
		struct {
				/* bit 256 - Outstanding Notification */
			u16	on	: 1,
				/* bit 257 - Suppress Notification */
				sn	: 1,
				/* bit 271:258 - Reserved */
				rsvd_1	: 14;
				/* bit 279:272 - Notification Vector */
			u8	nv;
				/* bit 287:280 - Reserved */
			u8	rsvd_2;
				/* bit 319:288 - Notification Destination */
			u32	ndst;
		};
		u64 control;
	};
	u32 rsvd[6];
} __aligned(64);
```

- Posted Interrupt Requests （PIR），一共256 bit，每个bit对应一个中断向量（Vector），当VT-d硬件将中断请求post过来的时候，中断请求相应的vector对应的bit将会被置起来。
- Outstanding Notification（ON），表示对于该Posted Interrupt Descriptor当前是否已经发出了一个Notification Event等待CPU的处理。当VT-d硬件将中断请求记录到PIR的时候，如果ON为0，并且允许立即发出一个Notification Event时，则将会将ON置起来，并且产生一个Notification Event；如果ON已经被置起来，则VT-d硬件不做其他动作。
- Suppress Notification（SN），表示当PIR寄存器记录到non-urgent的中断时，是否不发出Notification Event，如果该位为1，则当PIR记录到中断的时候，则不发出Notification Event，并且不更改Outstanding Notification位的值。
- Notification Vector（NV）表示如果发出Notification Event时，具体的Vector值。
- Notification Destination（NDST），表示若产生Notification Event时，传递的目标逻辑CPU的LAPIC ID（系统中以逻辑CPU的LAPIC ID来表示具体的逻辑CPU，BIOS/UEFI其初始化系统的时候，会为每个逻辑CPU分配一个唯一的LAPIC ID）。

在硬件上整个Posted Interrupt的处理过程如下所示：

<img src="/images/posts/20221226-InterruptPosting/p3.png" width="700px" />

当VT-d硬件接收到其旗下I/O设备传递过来的中断请求时，会先查看自己的中断重定向功能是否打开，如果没有打开则，直接上传给LAPIC。如果中断重定向功能打开，则会查看中断请求的格式，如果是不可重定向格式，则直接将中断请求提交给LAPIC。如果是可重定向的格式，则会根据算法计算Interrupt_Index值，对中断重定向表进行索引找到相应的IRTE。然后，查看IRTE中的Interrupt Mode，如果为0，则该IRTE的格式为Remapped Format，即立即根据IRTE的信息产生一个新的中断请求，提交到LAPCI。如果Interrupt Mode为1，则表示该IRTE的格式为Posted Format，根据IRTE中提供的Posted Interrupt Descriptor的地址，在内存中找到相应Posted Interrupt Descriptor，并根据其ON、URG和SN的设置判断是否需要立即产生一个Notification Event（Outstanding Notification为0，并且该中断为Urgent（URG=1）或者不抑制该Notification（SN==0）），如果不需要，则只是将该中断信息记录到Posted Interrupt Descriptor的PIRR（Posted Interrupt Request Register）字段，等待CPU的主动处理。如果需要立即产生一个Notification Event，则根据Posted Interrupt Descriptor（会提供目标APIC ID、vector、传输模式和触发模式等信息）产生一个Notification Event，并且将该中断请求记录到PIRR字段。

硬件在对Posted Interrupt Descriptor进行修改的时候，要保证该修改是原子操作，即对Posted Interrupt Descriptor的读取、修改和写入必须是原子操作，并且在写入之后，要保证相应内存在各个cache agent之间的一致性，即所有的CPU应该立马能够看到该内存修改。

从软件的角度来看，VMM可能会对Interrupt Posting做如下设置和操作：
1. 对于每个vCPU而言，VMM都会分配一个对应的Posted Interrupt Descriptor用于记录和传递经过重定向，并且目的地为对应vCPU的所有中断请求。
2. VMM软件为所有的Notification Event分配两个物理中断vector：
   1. 第一个称作Active Notification Vector（ANV），(sysvec_kvm_posted_intr_ipi) 该Vector对应到当中断目标vCPU当前正在被逻辑CPU执行（即vCPU的状态为active）时，Notification Event所使用的中断vector。
   2. 第二个称作Wake-up Notification Vector（WNV），(sysvec_kvm_posted_intr_wakeup_ipi) 该Vector对应到中断目标vCPU当前不在逻辑CPU上被执行时，由于Urgent被置起来产生的Notification Event所使用的中断Vector。
3. 对于从直接分配给VM的I/O设备产生的中断而言，VMM会为每个这样的中断分配一个IRTE。并且VMM可能会为vCPU使能硬件上的APIC Virtualization，APIC Virtualization主要包括两方面功能：virtual-interrupt delivery和process posted-interrupts，其主要工作形式表现在
   1. 当一个vCPU被VMM调度即将执行的时候，该vCPU的状态为active，该状态的一个表现形式是VMM会将Posted Interrupt Descriptor的Notification Vector字段设置为ANV（Active Notification Vector）。这样就允许当这个vCPU在逻辑CPU上执行的时候，所有指向该vCPU的中断都能够直接被该vCPU处理，不需要经过VMM。vCPU通过将记录在Posted Interrupt Descriptor中的中断直接转移到Virtual-APIC Page中，并直接将中断信号传递给vCPU，让vCPU直接获取到该中断信号的方式来处理Notification Event。
   2. 当一个vCPU被抢占后，即vCPU的状态为ready-to-run，该状态的一个表现形式是VMM会将Posted Interrupt Descriptor的Notification Vector字段设置为WNV（Wake-up Notification Vector），并且SN（Suppress Notification）设置为1。只有当接收到的中断请求为Urgent的时候，才会发出Notification Event，触发VMM的执行，让VMM调度目标vCPU处理该紧急中断。
   3. 当一个vCPU处于Halt状态的时候，逻辑CPU执行VMM软件，VMM将该vCPU标记为halted状态。该状态的一个表现形式就是将Posted Interrupt Descriptor的Notification Vector字段设置为WNV（Wake-up Notification Vector），并且SN（Suppress Notification）设置为0，即任何到达该Posted Interrupt Descriptor的中断请求都会触发Notification Event，让VMM唤醒vCPU，让vCPU处理中断请求。
4. 当VMM调度并执行一个vCPU的时候，VMM会对被记录到Posted Interrupt Descriptor的中断请求进行处理:
   1. 首先，VMM会通过将Posted Interrupt Descriptor的Notification Vector字段的值改为ANV将vCPU的状态变为active。
   2. VMM检测在Posted Interrupt Descriptor中是否有待处理的中断请求
   3. 如果有待处理的中断请求，则VMM会给当前CPU发送一个sefl-IPI中断（即CPU自己给自己发送一个中断），并且中断向量值为ANV。当vCPU一使能中断的时候，就能够立马识别到该中断。该中断的处理类似于vCPU处于active状态时，接收到了Active Notification Vector的中断请求，vCPU可以直接对其进行处理，不需要VMM的参与。
5. 同样VMM可以可以利用Posted Interrupt的处理机制，通过设置Posted Interrupt Descriptor向vCPU注入中断请求。

总体上来说，Interrupt Posting的功能就是让系统可以选择性地将中断请求暂存在内存中，让CPU主动去获取，而不是中断请求一过来就让CPU进行处理，这样在虚拟化的环境中，就能够防止外部中断频繁打断vCPU的运行，从而提高系统的性能。