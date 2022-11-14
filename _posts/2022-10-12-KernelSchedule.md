---
layout: post
title: 【内核】调度
date: 2022-10-12
tags: 内核
---


# Linux 内核调度器

## 1.前言

前文主要学习到什么情况下线程会倍设置need_resched标志，这里学习如何真实的进行线程切换。
调度的核心函数是
```
static void __sched notrace __schedule(bool preempt)
```

## 2.调度核心之__schedule函数

调度核心函数 __schedule 这以下情况下被调用：
- 显式阻塞：mutex，semaphore，waitqueue，etc。
- 中断或者用户态返回流程中的TIF_NEED_RESCHED检查通过。

```
static void __sched notrace __schedule(bool preempt) {
	...
	rq_lock(rq, &rf);
	...
	next = pick_next_task(rq, prev, &rf); // 选择优先级最高的线程
	...
	rq = context_switch(rq, prev, next, &rf); // 进行上下文切换, __schedule是context_switch的唯一入口
	...
}
```

## 3.try_to_wake_up

try_to_wake_up 用于唤醒一个已经睡眠的进程，该函数中主要：
1. 查看llc亲和性，如果与当前cpu吻合，就把睡眠的进程插入就绪队列，等到被__schedule
2. 如果llc亲和性不吻合，就给目标cpu发送ipi中断，让目标cpu去调度


```
static int
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
	...
	cpu = select_task_rq(p, p->wake_cpu, SD_BALANCE_WAKE, wake_flags);
	...
	ttwu_queue(p, cpu, wake_flags);
		|
		o -> ttwu_queue_wakelist -> __ttwu_queue_wakelist -> __smp_call_single_queue
				|
				o -> 如果目标cpu在非idle线程运行-> send_call_function_single_ipi // 目标cpu发送ipi, 目标cpu ipi callback是generic_smp_call_function_single_interrupt
				|
				o -> 如果目标cpu在运行idle线程就无需发送ipi, 会自动根据call_single_queue 来调度 --> flush_smp_call_function_from_idle
		|
		o -> ttwu_do_activate -> activate_task -> enqueue_task(rq, p, flags); /* 入队 */
}
```
