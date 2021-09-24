---
layout: post
title: 【内核】二，内核中断分析
date: 2021-9-23
tags: 内核
---

# 中断概述

<img src="https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210923-KernelInterrupts/Interrupt.png"
 width="400px" />

<img src="/images/posts/20210923-KernelInterrupts/Layer.png"
 width="400px" />

- 硬件层：最下层为硬件连阶层，对应的是具体的外设与SoC的物理连接，中断信号是从外设到中断控制器，由中断控制器统一管理，再路由到处理器上；
- 硬件相关层：这个层包括两个部分代码
    - 架构相关的
    - 中断控制器驱动代码
- 通用层：这个部分可以认为是框架层，是硬件无关层，这部分代码在所有硬件平台上是通用的；
- 用户层：这部分也就是中断的使用者了，主要是各类设备驱动，通关中断相关接口来进行申请和注册，最终在外设触发中断时，进行相应的回调处理；

# 举例
可以先看一下 [中断和异常](https://arafatms.github.io/2021/09/AMDExceptionAndInterrupts/)

首先发生中断时，CPU根据中断向量*16 + IDT Base Address 找到对应的中断描述福，中断描述符中记录着如下信息：
- Code Segment offset -> 中断处理函数CS
- CS Selector -> Global or Local Descriptor table -> 这些段会被忽略，不用关注

对于中断栈，是从中断描述符中的IST字段上获取的，以下以NMI为例
## NMI中断

```
//把IDT中的每一项叫做 `门(gate)` 
//CPU使用一个特殊的 `GDTR` 寄存器来存放全局描述符表的地址，中断描述符表也有一个类似的寄存器 `IDTR`
//64位模式下 IDT 的每一项的结构如下：
//
//
//127                                                                             96
// --------------------------------------------------------------------------------
//|                                                                               |
//|                                Reserved                                       |
//|                                                                               |
// --------------------------------------------------------------------------------
//95                                                                              64
// --------------------------------------------------------------------------------
//|                                                                               |
//|                               Offset 63..32                                   |
//|                                                                               |
// --------------------------------------------------------------------------------
//63                               48 47      46  44   42    39             34    32
// --------------------------------------------------------------------------------
//|                                  |       |  D  |   |     |      |   |   |     |
//|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
//|                                  |       |  L  |   |     |      |   |   |     |
// --------------------------------------------------------------------------------
//31                                   15 16                                      0
// --------------------------------------------------------------------------------
//|                                      |                                        |
//|          Segment Selector            |                 Offset 15..0           |
//|                                      |                                        |
// --------------------------------------------------------------------------------
//
```

中断描述符如下：
```
struct idt_bits {/* 中断 索引 */
	u16		ist	: 3,    //* `IST` - 中断堆栈表；`Interrupt Stack Table` 是 `x86_64` 中的新机制
	                    //代替传统的栈切换机制
			zero	: 5,
			type	: 5,    // `Type` 域描述了这一项的类型
                            //* 任务描述符
                            //* 中断描述符
                            //* 陷阱描述符
			dpl	: 2,    //* `DPL` - 描述符权限级别；还有个CPL(Current Privilege Level,`3`=用户空间级别)
			            //          
			p	: 1;    //* `P` - 当前段标志；段存在标志；
} __attribute__((packed));

struct gate_struct {/* 门 */
	u16		offset_low; //* `Offset` - 处理程序入口点的偏移量, 长模式下对应interrupt handler程序的VM；
	u16		segment;    //* `Selector` - 目标代码段的段选择子；
	struct idt_bits	bits;
	u16		offset_middle;
#ifdef CONFIG_X86_64
	u32		offset_high;
	u32		reserved;
#endif
} __attribute__((packed));
```

用以下宏定义去创建一个带IST的中断描述符
```
ISTG(X86_TRAP_NMI,	asm_exc_nmi,			IST_INDEX_NMI),
// asm_exc_nmi保存到各个offset中,也就是说当发生中断时RIP会从中获取
// IST_INDEX_NMI保存到.idt_bits.ist中
```

可以看出CS段已经有了，SS段记录在.ist中（只是一个索引），真实地址实际上存放在tss中
```
static inline void tss_setup_ist(struct tss_struct *tss)
{
	/* Set up the per-CPU TSS IST stacks */
	tss->x86_tss.ist[IST_INDEX_NMI] = __this_cpu_ist_top_va(NMI);
    ....
}
```

也就是说当发生NMI中断时，RSP会自动从tss->x86_tss.ist[IST_INDEX_NMI]中加载；
最终处理器会跳到asm_exc_nmi，并且RSP指向NMI中断栈，并且处理器会自动做以下处理：
1. 从IST加载中断栈RSP
2. 

## NMI中断处理函数
```
SYM_CODE_START(asm_exc_nmi)
    // rdx用为临时寄存器

    // 检测是否是来自内核的NMI，这一步怎么检测的呢？
	testb	$3, CS-RIP+8(%rsp)
	jz	.Lnmi_from_kernel
    // 交换 GS 段选择符及 `MSR_KERNEL_GS_BASE ` 特殊模式寄存器中的值
    swapgs
    // 切换至内核cr3，并把当前cr3返回到rdx中
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rdx
    // 加载内核per_cpu的栈，并把NMI栈复制给rdx
    movq	%rsp, %rdx
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

    //当一个中断或者异常发生时，新的 `ss` 选择器被强制置为 `NULL`，
    //并且 `ss` 选择器的 `rpl` 域被设置为新的 `cpl`。
    //旧的 `ss`、`rsp`、寄存器标志、`cs`、`rip` 被压入新栈。
    //在 64 位模型下，中断栈帧大小固定为 8 字节，所以我们可以得到下面的栈:
    //
    //+---------------+
    //|               |
    //|      SS       | 40
    //|      RSP      | 32
    //|     RFLAGS    | 24
    //|      CS       | 16
    //|      RIP      | 8
    //|   Error code  | 0
    //|               |
    //+---------------+
    // 以下都是利用中断栈的内容加载pt_regs
	pushq	5*8(%rdx)	/* pt_regs->ss */
	pushq	4*8(%rdx)	/* pt_regs->rsp */
	pushq	3*8(%rdx)	/* pt_regs->flags */
	pushq	2*8(%rdx)	/* pt_regs->cs */
	pushq	1*8(%rdx)	/* pt_regs->rip */
	UNWIND_HINT_IRET_REGS
	pushq   $-1		/* pt_regs->orig_ax */
	PUSH_AND_CLEAR_REGS rdx=(%rdx)
	ENCODE_FRAME_POINTER

    movq	%rsp, %rdi
	movq	$-1, %rsi
	call	exc_nmi

    // 最终会跳转到DEFINE_IDTENTRY_RAW(exc_nmi)中
```

## NMI中断返回
```
	/*
	 * Return back to user mode.  We must *not* do the normal exit
	 * work, because we don't want to enable interrupts.
	 */
	jmp	swapgs_restore_regs_and_return_to_usermode  // 这个函数在讲述缺页异常时有过说明
```

## 缺页异常
```
// 会比NMI等中断早一些时间初始化，原因待查看
/**
 * idt_setup_early_pf - Initialize the idt table with early pagefault handler
 *
 * On X8664 this does not use interrupt stacks as they can't work before
 * cpu_init() is invoked and sets up TSS. The IST variant is installed
 * after that.
 *
 * Note, that X86_64 cannot install the real #PF handler in
 * idt_setup_early_traps() because the memory intialization needs the #PF
 * handler from the early_idt_handler_array to initialize the early page
 * tables.
 *
 * 用于建立 `#PF` 处理函数，最终asm_exc_page_fault->exc_page_fault 保存到Code Segment offset中，发生缺页异常时跳转到这个函数中
 */
start_kernel()->setup_arch()->idt_setup_early_pf()
    idt_setup_from_table(idt_table, early_pf_idts, ARRAY_SIZE(early_pf_idts), true);
```

而进入exc_page_fault之前会调用idtentry AMS宏来进行一些初始化
```
idtentry->idtentry_body
    movq	%rsp, %rdi			/* pt_regs pointer into 1st argument*/
    movq	ORIG_RAX(%rsp), %rsi	/* get error code into 2nd argument*/
    call	\cfunc   //此处就会跳转到 exc_page_fault 中
```

## 缺页异常返回
```
SYM_CODE_START_LOCAL(error_return)
	UNWIND_HINT_REGS
	DEBUG_ENTRY_ASSERT_IRQS_OFF
	testb	$3, CS(%rsp)
	jz	restore_regs_and_return_to_kernel
	jmp	swapgs_restore_regs_and_return_to_usermode
SYM_CODE_END(error_return)
```

# 总结

当发生中断时处理器会根据中断号找到对应的处理函数，中断栈等，然后做一些压栈操作，结果可以从中断栈中查看，然后切换至Per CPU栈搜集pt_regs，然后可以允许相应的处理函数