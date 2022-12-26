---
layout: post
title: 【Intel-VT_d-Spec】一，中断重映射
date: 2022-12-21
tags: Intel-VT_d-Spec    
---
 
[参考1](https://blog.csdn.net/lindahui2008/article/details/83954861?spm=1001.2014.3001.5502)

[参考2 Intel VT-d spec chapter 5]()

VT-d硬件中除了包含DMA重映射硬件外，也会包含中断重映射硬件，该中断重映射单元让系统软件能够对I/O设备产生的中断（包括从I/O APIC发送过来的中断，I/O设备产生的以MSI、MSI-X形式传递的中断，不包含中断重映射硬件本身产生的中断）的传输进行控制，而不仅仅取决于硬件的连接。

对于VT-d硬件而言，中断请求就是从外面发送进来对物理地址范围0xFEEX_XXXXh的dma写请求DWORD。VT-d中，中断重映射功能由Extended Capability Register寄存器来决定，该寄存器的bit3表示Interrupt Remapping Support，如果为1，则表示支持中断重映射，如果为0，则表示不支持中断重映射。

<img src="/images/posts/20221221-InterruptRemapping/p1.png" width="700px" />

## 中断源
为了实现同一物理系统中，不同Domain之间的隔离，需要VT-d的中断重映射硬件对接收到的中断请求提取其中断的来源（一般是PCI设备的Bus、Device和Function号之类的信息），然后根据不同设备所属的Domain，将该中断请求转发到相应的Domain中，实现不同Domain之间，中断请求的隔离。

中断源有如下几种类型:
- 来自PCIe设备的MSI中断：源id为BDF
- 来自Root complex集成设备的MSI中断：源id为PCI请求id
- 来自PCIe -> PCI/PCI-x桥后设备的MSI中断
- ...


在Intel x86架构中，中断请求的格式有两种：兼容格式(Compatibility Format)和可重映射格式(Remappable Format)。并且每个请求中都包含了访问的地址和数据，格式的选择由地址信息的bit 4（Interrupt Format）来决定。

对于兼容格式的中断请求会直接向上传递到CPU的LAPCI，不会被重新映射。

### 可重映射格式的中断请求

<img src="/images/posts/20221221-InterruptRemapping/p2.png" width="700px" />


可重映射格式的中断请求的各个位含义如下：
- Interrupt format位为1，表示格式类型
- handle，shv，Subhandle位决定中断重映射表索引

## 中断重映射表

中断重映射硬件利用一张位于内存的单层表，即中断重映射表，来确定中断请求需要被如何重新生成并转发。该表由VMM/OS配置，并且该表的物理地址将会被写到VT-d硬件中的Interrupt Remap Table Address Register，用于告知硬件中断重映射表的位置。并且低4 bit用于表示中断重映射表中包含的entry个数，即2的（1+S）次方，最高达2的16次方，即64K。

<img src="/images/posts/20221221-InterruptRemapping/p3.png" width="700px" />

中断重映射表中的每个表项大小为128 bit，即16 Byte，称为Interrupt Remap Table Entry（IRET），其格式如下所示：

<img src="/images/posts/20221221-InterruptRemapping/p4.png" width="700px" />

主要包含了重映射目标的Destination ID、Vector和一些其他中断传输相关的信息（对于兼容格式的中断而言，其中断的属性都是在中断请求中说明，而可重映射的中断，其中断属性则在IRTE中说明）。另外bit 15必须为0，该bit表示IRTE Mode，如果为0，则表示为Remapped Interrupt，如果为1，则表示Posted Interrupt（下一篇文章讲解）。中断重映射硬件，将以前面提到的中断请求中包含的Handle和Sub-Handle计算索引值，对中断重映射表进行索引。其算法如下所示：

<img src="/images/posts/20221221-InterruptRemapping/p5.png" width="700px" />

从硬件的角度来看，整个中断重映射的过程为：硬件检测到向0xFEEX_XXXXh地址DWORD的写请求，判定其为中断请求，并将其拦截。如果中断重映射的功能没有打开（Global Status Register的IRES为0），则所有的中断请求都以兼容格式的中断来处理。如果中断重映射功能被打开（Global Status Register的IRES为1），则查看中断请求的格式，如果是兼容格式，则直接跳过中断重映射硬件，继续中断请求的传递。如果是可重映射格式，则检测中断请求中的数据正确性，计算Interrupt_index，读取相应的IRTE，检测IRTE是否存在及其正确性，如果一切正常则中断重映射硬件将会根据读取的IRTE产生一个新的中断请求，并向上传递。其基本流程如下所示：

<img src="/images/posts/20221221-InterruptRemapping/p6.png" width="700px" />

对于软件而言，为了实现中断重映射，则需要进行如下操作：
1. 如果中断重映射表没有被分配，则在内存中分配一块区域作为中断重映射表，并将该表的位置告诉给中断重映射硬件，即将该表的位置写到Interrupt Remap Table Address Register(intel_setup_irq_remapping为中断初始化入口函数)。
2. 找到一个可用的IRTE（Interrupt Remapping Table Entry），然后设置需要重新转发的中断的一些属性，如传递的目标、中断模式，中断向量等(intel_irq_remapping_alloc)。
3. 对中断源（一般是I/O设备或者是I/O APIC）进行设置，让中断源能够产生可重映射格式的中断，并且由其Handle、Sub-Handle、SHV等区域计算出来的interrupt_index正好匹配到之前设置好的IRTE。不同的I/O设备的设置方法会有区别。


对于I/O xAPIC而言，系统软件通过设置I/O xAPIC的Redirection Table Entries（RTEs）来设置I/O xAPIC产生的可重映射中断。RTE的格式如下所示：

<img src="/images/posts/20221221-InterruptRemapping/p7.png" width="700px" />

将Interrupt Format设置为1表示产生的中断为可重映射中断，并且可以设置Interrupt_Index、Vector、Trigger Mode等信息。

对于可以产生MSI（Message Signaled Interrupt）或者MSI-X的设备而言，对产生中断的设置包括Address和Data寄存器，其格式如下所示：

<img src="/images/posts/20221221-InterruptRemapping/p8.png" width="700px" />

当Address的bit 4（Interrupt Format）为1的时候，则表示产生的中断为可重映射中断，并且可以设置Interrupt_Index和SHV等值。对于支持多个（必须是2的n次方）Vector中断的MSI/MSI-X消息而言，其Data寄存器的低n位对应到具体的Vector数值，并且SHV（Sub-Handle Valid）为1，这时候IRTE的索引值就是Interrupt_index[15:0]的值加上Vector的值。

总的来说，VT-d的中断重映射就是指VT-d会拦截其下面挂载的I/O设备产生的中断，然后根据接收到的中断请求索引中断重映射表，根据找到的中断重映射表的表项产生新的中断请求，上传到CPU的LAPIC。VT-d就是通过这个重映射动作，实现了同一物理系统中不同Domain的I/O设备中断请求的隔离。同时为了让I/O设备能够产生可重映射的中断，并对中断重映射表进行正确的索引，系统软件还需要对I/O设备的中断请求生成进行配置。