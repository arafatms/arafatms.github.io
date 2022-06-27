---
layout: post
title: 【内核之Tool】一，waitqueue等待队列相关
date: 2022-6-17
tags: 内核之Tool
---

本章介绍linux内核中的实用工具等待队列waitqueue。

## 初始化等待队列头
```
struct wait_queue_head {    /*  */
	spinlock_t		lock;
	struct list_head	head;   /* wait_queue_entry 为 节点的链表 */
};
typedef struct wait_queue_head wait_queue_head_t;

init_waitqueue_head(wait_queue_head_t *q)
OR
init_waitqueue_head(struct wait_queue_head *q)
```
详细流程待补充，项目有点赶，哈哈哈哈哈

## 定义并初始化等待队列entry

```
/*
 * A single wait-queue entry structure:
 */
struct wait_queue_entry {   /* 等待队列 */
	unsigned int		flags;  /* 标志位 */
	void			*private;   /* 私有数据 */
	wait_queue_func_t	func;   /* 回调函数 */
	struct list_head	entry;  /* wait_queue_head->head 中的 链表节点 */
};
```

一般会用到两种方式：
- 局部定义：当前函数内部定义一个wait，便捷
- 全局定义：可以用于为一个设备定义某一种wait entry等等，可以复用

### 局部定义 (struct wait_queue_entry OR wait_queue_t)
主要讲述5.10内核的！
```
#define DEFINE_WAIT_FUNC(name, function)					\
	struct wait_queue_entry name = {					\
		.private	= current,					\
		.func		= function,					\
		.entry		= LIST_HEAD_INIT((name).entry),			\
	}

// autoremove_wake_function 主要从wait_queue_head中的链表中删掉当前entry
#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)

DEFINE_WAIT(struct wait_queue_entry *wait)
```

### 全局定义
```
wait.name = name;
init_wait(&wait.wait);
wait.wait.func = your_wake_function;
```

## 触发等待

```
// 主要是往wait_queue_head中的链表中添加当前entry，并设置current结构的一些状态,让进程下次调度时一直呆在就绪队列中
void
prepare_to_wait(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry, int state);
```

state指的是current当前进程的状态，一般有：
- TASK_INTERRUPTIBLE，（等待可以被中断）
- ....

随后用主动调度相关函数，让进程等待，比如：
```
schedule();
cond_resched();
```
当触发唤醒以后需要调用finish_wait来结束等待
```
// 如果有特殊条件想撤回prepare_to_wait时也可以调用这个函数
void finish_wait(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry);
```
