---
layout: post
title: 【内核之系统调用】一，系统调用详解
date: 2021-9-14
tags: 内核之系统调用
---

# 系统调用详解
> 由操作系统实现提供的所有系统调用所构成的集合即程序接口或应用编程接口(Application Programming Interface，API)。是应用程序同系统之间的接口。
## SYSCALL
用户态程序调用SYSCALL指令进入内核态。
> SYSCALL and SYSRET are low-latency system call and return instructions.

### 初始化
涉及到很多MSR寄存器相关内容，可以先看看【AMD64开发手册】六，系统指令 
```
cpu_init->syscall_init()
void syscall_init(void)
{
    // 设置STAR寄存器
	wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);

	wrmsrl(MSR_CSTAR, (unsigned long)ignore_sysret);
	wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
	wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
	wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);

	/* Flags to clear on syscall */
	wrmsrl(MSR_SYSCALL_MASK,
	       X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
	       X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
}
```

#### 设置STAR寄存器
因64位模式下平台内存模型中也有DPL限制，所以对应内核态和用户态（所有应用程序）在GDT中各分配一个CS。
```
#define GDT_ENTRY_KERNEL32_CS		1
#define GDT_ENTRY_KERNEL_CS		2
#define GDT_ENTRY_KERNEL_DS		3

/*
 * We cannot use the same code segment descriptor for user and kernel mode,
 * not even in long flat mode, because of different DPL.
 *
 * GDT layout to get 64-bit SYSCALL/SYSRET support right. SYSRET hardcodes
 * selectors:
 *
 *   if returning to 32-bit userspace: cs = STAR.SYSRET_CS,
 *   if returning to 64-bit userspace: cs = STAR.SYSRET_CS+16,
 *
 * ss = STAR.SYSRET_CS+8 (in either case)
 *
 * thus USER_DS should be between 32-bit and 64-bit code selectors:
 */
#define GDT_ENTRY_DEFAULT_USER32_CS	4
#define GDT_ENTRY_DEFAULT_USER_DS	5
#define GDT_ENTRY_DEFAULT_USER_CS	6
```

STAR寄存器包含以下区域：
    - SYSRET CS and SS Selectors：系统调用返回时CS，SS的段选择索引，设置为__USER32_CS
    - SYSCALL CS and SS Selectors：系统调用时CS，SS的段选择索引，设置为__KERNEL_CS
    - 32-bit SYSCALL Target EIP：设置为0

#### 设置LSTAR寄存器
LSTAR寄存器为64位下SYSCALL指令的目标RIP；在此保存entry_SYSCALL_64函数对应的地址，说明当调用SYSCALL时会跳转到entry_SYSCALL_64函数。

#### 设置SFMASK寄存器
SFMASK表明RFLAGS中的哪些位会在调用SYSCALL过程中清理。在此代码中会清理以下特性：
- X86_EFLAGS_TF：单步调试位（见过哪个gdb能进入系统调用的）
- X86_EFLAGS_DF：Direction Flag
- X86_EFLAGS_IF：需要屏蔽中断
- X86_EFLAGS_IOPL：IO privilege level
- X86_EFLAGS_AC：Alignment Check
- X86_EFLAGS_NT：Nested Task

### 进入entry_SYSCALL_64之前
当用户态调用syscall指令时处理器从STAR寄存器中获取内核CS和SS，这样就可以通过段检查机制访问内核空间；并且从LSTAR中获取entry_SYSCALL_64函数地址；最后将上述讲过的RFLAGS清零。
此时中断已经被屏蔽，处理器EIP指向entry_SYSCALL_64，然后可以愉快的执行entry_SYSCALL_64了～

### 进入entry_SYSCALL_64之后-1
```
ENTRY(entry_SYSCALL_64)
	swapgs  //首先会调用swapgs指令切换用户态数据结构和内核数据结构
    /* tss.sp2 is scratch space. */
    /*
	 * Since Linux does not use ring 2, the 'sp2' slot is unused by
	 * hardware.  entry_SYSCALL_64 uses it as scratch space to stash
	 * the user RSP value. TSS_sp2用于暂缓用户态rsp值
	 */
	movq	%rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
	SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp   //切换至内核cr3
    /*
	 * We store cpu_current_top_of_stack in sp1 so it's always accessible.
	 * Linux does not use ring 1, so sp1 is not otherwise needed.
	 */
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp //切换当前task内核rsp，值保存在tss.sp1中，特别的对每个task rsp栈低保存着thread_struct
```

#### swapgs指令
提供快速加载系统数据结构的方法。当调用SYSCALL时SWAPGS可以快速的加载内核所使用的数据结构，当SYSRET之前可以快速加载被内核数据结构覆盖掉的应用程序数据结构。SWAPGS指令会交换KernelGSbase寄存器和GS.base寄存器。
特别的GS寄存器中保存内核数据结构的指针，处理器对此没有做特定的描述。
linux内核初始化过程中中KernelGSbase寄存器中存放有0,而MSR_GS_BASE中存放percpu相关内容？
```
cpu_init
wrmsrl(MSR_KERNEL_GS_BASE, 0);

	/* Set up %gs.
	 *
	 * The base of %gs always points to the bottom of the irqstack
	 * union.  If the stack protector canary is enabled, it is
	 * located at %gs:40.  Note that, on SMP, the boot cpu uses
	 * init data section till per cpu areas are set up.
	 */
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

```
### 进入entry_SYSCALL_64之后-2
主要构造struct pt_regs，也可以从构造中看出用户上下文数据保存在哪里了
```
/* Construct struct pt_regs on stack */
	pushq	$__USER_DS				/* pt_regs->ss */   //__USER_DS是对每一个用户态应用来说是盯死的
	pushq	PER_CPU_VAR(cpu_tss_rw + TSS_sp2)	/* pt_regs->sp */   //上面保存过用户态rsp值到tss.sp2
	pushq	%r11					/* pt_regs->flags */    //用户态rflags保存在r11
	pushq	$__USER_CS				/* pt_regs->cs */       //跟__USER_DS一样
	pushq	%rcx					/* pt_regs->ip */       //下一条指令的地址
GLOBAL(entry_SYSCALL_64_after_hwframe)
	pushq	%rax					/* pt_regs->orig_ax */  //系统调用号

    PUSH_AND_CLEAR_REGS rax=$-ENOSYS    //也是构造pt_regs结构 填充各种通用寄存器啊 啥的

    movq	%rax, %rdi
	movq	%rsp, %rsi
	call	do_syscall_64		/* returns with IRQs disabled */ //最终返回值写到pt_regs->rax中
```

### 返回流程
```
    TRACE_IRQS_IRETQ		/* we're about to change IF */

	/*
	 * Try to use SYSRET instead of IRET if we're returning to
	 * a completely clean 64-bit userspace context.  If we're not,
	 * go to the slow exit path.
	 */
	movq	RCX(%rsp), %rcx
	movq	RIP(%rsp), %r11

	cmpq	%rcx, %r11	/* SYSRET requires RCX == RIP */ //如果相等就利用syscall快速返回,同样的会检查一系列寄存器，如果满足条件就进入快速返回
	jne	swapgs_restore_regs_and_return_to_usermode
```

### 快速返回sysret
```
syscall_return_via_sysret:
	/* rcx and r11 are already restored (see code above) */
	UNWIND_HINT_EMPTY
	IBRS_EXIT
	POP_REGS pop_rdi=0 skip_r11rcx=1    //还原通用寄存器等寄存器的值

	/*
	 * Now all regs are restored except RSP and RDI.
	 * Save old stack pointer and switch to trampoline stack.
	 */
	movq	%rsp, %rdi
	movq	PER_CPU_VAR(cpu_tss_rw + TSS_sp0), %rsp //切换到trampoline stack（原来tss.sp0中保存着这哥们）

	pushq	RSP-RDI(%rdi)	/* RSP */   //用户RSP和RDI入栈
	pushq	(%rdi)		/* RDI */

	/*
	 * We are on the trampoline stack.  All regs except RDI are live.
	 * We can do future final exit work right here.
	 */
	SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi   //还原用户cr3

	popq	%rdi    //还原用户RSP和RDI
	popq	%rsp
	USERGS_SYSRET64
        swapgs;
	    sysretq;
```