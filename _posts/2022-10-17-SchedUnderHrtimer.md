---
layout: post
title: 【内核】基于高精度计时器(hrtimer)的调度
date: 2022-11-10
tags: 内核
---

## 1.前言


## 2.初始化sched_timer

per cpu类型的tick_sched为sched_timer核心数据结构，如下：
```
/*
 * Per-CPU nohz control structure
 */
static DEFINE_PER_CPU(struct tick_sched, tick_cpu_sched);
...
struct tick_sched {
    // hrtimer to schedule the periodic tick in high
	struct hrtimer			sched_timer;
    // Notification mechanism about clocksource changes
	unsigned long			check_clocks;
    // Mode - one state of tick_nohz_mode
	enum tick_nohz_mode		nohz_mode;

	unsigned int			inidle		: 1;
	unsigned int			tick_stopped	: 1;
	unsigned int			idle_active	: 1;
	unsigned int			do_timer_last	: 1;
	unsigned int			got_idle_tick	: 1;

	ktime_t				last_tick;
	ktime_t				next_tick;
	unsigned long			idle_jiffies;
	unsigned long			idle_calls;
	unsigned long			idle_sleeps;
	ktime_t				idle_entrytime;
	ktime_t				idle_waketime;
	ktime_t				idle_exittime;
	ktime_t				idle_sleeptime;
	ktime_t				iowait_sleeptime;
	unsigned long			last_jiffies;
	u64				timer_expires;
	u64				timer_expires_base;
	u64				next_timer;
	ktime_t				idle_expires;
	atomic_t			tick_dep_mask;
};
```

其中注册timer是在函数tick_setup_sched_timer中执行

调用战如下:
```
#0  tick_setup_sched_timer () at kernel/time/tick-sched.c:1328
#1  0xffffffff811094df in hrtimer_switch_to_hres () at kernel/time/hrtimer.c:757
#2  hrtimer_run_queues () at kernel/time/hrtimer.c:1759
#3  0xffffffff81107cfe in run_local_timers () at kernel/time/timer.c:1810
#4  0xffffffff81107d51 in update_process_times (user_tick=0) at kernel/time/timer.c:1737
#5  0xffffffff811174fb in tick_periodic (cpu=<optimized out>) at ./arch/x86/include/asm/ptrace.h:131
#6  0xffffffff81117575 in tick_handle_periodic (dev=0xffff88803fd16080) at kernel/time/tick-common.c:109
#7  0xffffffff81e0254f in local_apic_timer_interrupt () at arch/x86/kernel/apic/apic.c:1123
#8  smp_apic_timer_interrupt (regs=<optimized out>) at arch/x86/kernel/apic/apic.c:1148
#9  0xffffffff81e01a6f in apic_timer_interrupt () at arch/x86/entry/entry_64.S:832
#10 0xffffc90000073dd8 in ?? ()
```

具体作用如下：
```
tick_setup_sched_timer
    |
    o -> hrtimer_init(&ts->sched_timer, CLOCK_MONOTONIC, HRTIMER_MODE_ABS_HARD);
         // 初始化hr_timer, CLOCK_MONOTONIC表示到期时间是个绝对值+回调函数会在中断处理程序中被执行
    |
    o -> ts->sched_timer.function = tick_sched_timer; // 注册回调函数，也就是每次timer到期时都会调用这个函数
    |
    o -> hrtimer_forward(&ts->sched_timer, now, tick_period); // 注册间隔为tick_period, 根据CONFIG_HZ计算间隔
    |
    o -> tick_nohz_activate(ts, NOHZ_MODE_HIGHRES); // 启动nohz, nohz相关的单独看
```


## 3.触法hrtimer
注册hrtimer的时候将sched hrtimer注册为ABS，HARD模式，说明触发间隔是个绝对值，而且在硬件中断中调用回调函数tick_sched_timer。其触发时调用栈如下:
```
#3  0xffffffff811196ec in tick_sched_timer (timer=0xffff88803fc1d000) at kernel/time/tick-sched.c:1296
#4  0xffffffff81108ad7 in __run_hrtimer (flags=<optimized out>, now=<optimized out>, timer=<optimized out>, base=<optimized out>, 
    cpu_base=<optimized out>) at kernel/time/hrtimer.c:1535
#5  __hrtimer_run_queues (cpu_base=<optimized out>, now=<optimized out>, flags=<optimized out>, active_mask=<optimized out>)
    at kernel/time/hrtimer.c:1597
#6  0xffffffff811092c1 in hrtimer_interrupt (dev=<optimized out>) at kernel/time/hrtimer.c:1659
#7  0xffffffff81e0254f in local_apic_timer_interrupt () at arch/x86/kernel/apic/apic.c:1123
#8  smp_apic_timer_interrupt (regs=<optimized out>) at arch/x86/kernel/apic/apic.c:1148
#9  0xffffffff81e01a6f in apic_timer_interrupt () at arch/x86/entry/entry_64.S:832
#10 0xffffffff82803d68 in init_thread_union ()
Backtrace stopped: Cannot access memory at address 0xffffc90000004000
```

具体实现如下:
```
tick_sched_timer
    |
    o -> tick_sched_do_timer
                |
                o -> 检查nohz情况下是否需要关闭
                |
                o -> tick_do_update_jiffies64 // 如果需要就updte jiffies
                |
                o -> ts->got_idle_tick = 1; // 如果在idle
    |
    o -> tick_sched_handle
                |
                o -> 处理nohz相关
                |
                o -> update_process_times
                            |
                            o -> account_process_tick // 进程时间片相关计算
                            |
                            o -> run_local_timers // 运行本地计时器
                            |
                            o -> scheduler_tick
```

### 3.1 scheduler_tick

在这个函数中主要有两个方面的工作：
- 更新相关统计量，管理内核中的与整个系统和各个进程的调度相关的统计量. 其间执行的主要操作 是对各种计数器+1。
- 激活负责当前进程调度类的周期性调度方法。

```
scheduler_tick
    |
    o -> arch_scale_freq_tick
    |
    o -> sched_clock_tick
    |
    o -> update_rq_clock // 处理就绪队列时钟的更新，本质上就是增加runqueue当前实例的时钟时间轴
    |
    o -> curr->sched_class->task_tick(rq, curr, 0); // 第二个工作，也就是激活进程调度类的周期性调度方法, 主要看CFS的方法
    |
    o -> calc_global_load_tick // 更新cpu的活动计数，主要是更新全局CPU就绪队列的calc_load_update
```

### 3.2 task_tick_fair 调度滴答命中CFS调度类

其主要任务如下：
- 计算当前任务的统计信息，比如vruntime，...
- 如果必要就唤醒新任务并抢占
- 也会根据se group来计算统计信息，先不考虑这情况

```
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

    // 这里对应每一个group se，如果是task se就没有parent
	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se); // 指向cfs runqueue
		entity_tick(cfs_rq, se, queued);
            |
            o -> update_curr(cfs_rq); // 计算当前任务的统计信息（虚拟时间等)
            |
            o -> update_load_avg // 更新任务及其 cfs_rq 平均负载
            |
            o -> update_cfs_group // 根据组运行队列的当前状态重新计算组实体(se group)
            |
            o -> check_preempt_tick(cfs_rq, curr); // 就绪队列中任务数量大于0时调用，决定继续运行当前任务或者抢占, 见3.3
	}

	if (static_branch_unlikely(&sched_numa_balancing))
		task_tick_numa(rq, curr);

	update_misfit_status(curr, rq);
	update_overutilized_status(task_rq(curr));
}
```

### 3.3 check_preempt_tick

> Preempt the current task with a newly woken task if needed: 如果需要，用新唤醒的任务抢占当前任务：

具体实现如下：
```
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	unsigned long ideal_runtime, delta_exec;
	struct sched_entity *se;
	s64 delta;

    // 根据cfs算法计算理想的时间片
	ideal_runtime = sched_slice(cfs_rq, curr);
	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    // delta_exec 为当前调度周期中当前任务的运行时间，如果运行的时间比较长就调度
	if (delta_exec > ideal_runtime) {
		resched_curr(rq_of(cfs_rq)); // 设置重新调度标志, 见3.4
		/*
		 * The current task ran long enough, ensure it doesn't get
		 * re-elected due to buddy favours.
		 */
		clear_buddies(cfs_rq, curr);
		return;
	}

	/*
	 * Ensure that a task that missed wakeup preemption by a
	 * narrow margin doesn't have to wait for a full slice.
	 * This also mitigates buddy induced latencies under load.
     * 为了时间片太小，任务频繁调度，如果实际运行时间小于这个值就不能被抢占
	 */
	if (delta_exec < sysctl_sched_min_granularity)
		return;

    // 选择红黑树中最左测的se，也就是vruntime最小
	se = __pick_first_entity(cfs_rq);
	delta = curr->vruntime - se->vruntime;

    // current的vruntime还要小,就退出
	if (delta < 0)
		return;

    // 这里没太看懂, 为什么差值大于ideal_runtime就得重新调度
	if (delta > ideal_runtime)
		resched_curr(rq_of(cfs_rq));
}
```

### resched_curr

主要设置resched标志，让其进程在返回用户态时exit_to_user_mode_loop根据标志进行调度, 或者在其他内核抢占的调度时机进行抢占,比如：
- 中断返回内核态
- 打开抢占的时候
- 开启软中断的时候

主要调用resched_curr有以下几个检查点：
- tick检查，也就是本文讲述内容
- 唤醒抢占：在fork和正常的唤醒路径上

```
/*
 * resched_curr - mark rq's current task 'to be rescheduled now'.
 *
 * On UP this means the setting of the need_resched flag, on SMP it
 * might also involve a cross-CPU call to trigger the scheduler on
 * the target CPU.
 */
void resched_curr(struct rq *rq)    /*  */
{
	struct task_struct *curr = rq->curr;
	int cpu;

	lockdep_assert_held(&rq->lock);

	if (test_tsk_need_resched(curr))    /* 是否需要重新调度 */
		return;

	cpu = cpu_of(rq);

	if (cpu == smp_processor_id()) {
		set_tsk_need_resched(curr); // thread_flag设置TIF_NEED_RESCHED
		set_preempt_need_resched(); // 内核抢占，抢占计数设为0
		return;
	}

	if (set_nr_and_not_polling(curr))
		smp_send_reschedule(cpu);
	else
		trace_sched_wake_idle_without_ipi(cpu);
}
```