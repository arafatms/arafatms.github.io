---
layout: post
title: 【内核】进程退出之退出映射
date: 2022-8-5
tags: 内核
---

浅谈一下最近遇到的一个mmap_exit问题

## 进程退出内核栈
```
do_exit+0x7e5/0xbb0
[75453.860195]  ? __switch_to_asm+0x34/0x70
[75453.860196]  ? __switch_to_asm+0x34/0x70
[75453.860197]  ? __switch_to_asm+0x34/0x70
[75453.860197]  ? __switch_to_asm+0x40/0x70
[75453.860199]  do_group_exit+0x43/0xb0
[75453.860201]  get_signal+0x138/0x880
[75453.860204]  ? put_timespec64+0x3f/0x60
[75453.860207]  do_signal+0x37/0x680
[75453.860209]  ? hrtimer_nanosleep+0xbc/0x1b0
[75453.860211]  ? hrtimer_init_sleeper+0xa0/0xa0
[75453.860213]  exit_to_usermode_loop+0x82/0xf0
[75453.860214]  do_syscall_64+0x11e/0x150
[75453.860215]  entry_SYSCALL_64_after_hwframe+0x44/0xa9
```

## do_exit

```
do_exit
    |
    o-> blk_needs_flush_plug
    |
    o-> force_uaccess_begin DS段调整至USER_DS，用于后续的mm_release->clear_child_tid
    |
    ...
    |
    o-> exit_signals(tsk);  /* sets PF_EXITING */
    |
    o-> sync_mm_rss 同步进程的mm的rss_stat
    |
    ...
    |
    o-> exit_mm();
            |
            o-> exit_mm_release
                    |
                    o-> mmput
```

## mm->mm_users && mm->mm_count

mm_users的控制通过mmget,mmput来进行，mmput中当refcount==0时会释放所有mm相关的子系统，如mmap,aio...
```
static inline void __mmput(struct mm_struct *mm)
{
	VM_BUG_ON(atomic_read(&mm->mm_users));

	uprobe_clear_state(mm);
	exit_aio(mm);
	ksm_exit(mm);
	khugepaged_exit(mm); /* must run before exit_mmap */
	exit_mmap(mm);  /* 会一直走到驱动的vm_operations_struct来释放mmap内存
	mm_put_huge_zero_page(mm);
	set_mm_exe_file(mm, NULL);
	if (!list_empty(&mm->mmlist)) {
		spin_lock(&mmlist_lock);
		list_del(&mm->mmlist);
		spin_unlock(&mmlist_lock);
	}
	if (mm->binfmt)
		module_put(mm->binfmt->module);
	mmdrop(mm);
}
```

mm_count通过mmgrab，mmdrop来进行，mmdrop中当refcount==0时会释放mm结构体
```
void __mmdrop(struct mm_struct *mm)
{
	BUG_ON(mm == &init_mm);
	WARN_ON_ONCE(mm == current->mm);
	WARN_ON_ONCE(mm == current->active_mm);
	mm_free_pgd(mm);
	destroy_context(mm);
	mmu_notifier_subscriptions_destroy(mm);
	check_mm(mm);
	put_user_ns(mm->user_ns);
	free_mm(mm);
}
```

## 记录一个kthread无锁队列来处理用户请求
### 初始化/释放流程
```
get_task_struct(current);
mmgrab(current->mm); mm->count++ 防止mm被释放
.....
put_task_struct(ctx->sqo_task);
mmdrop(ctx->mm_account);
```

### kthread switch mm

```
static inline void kthread_use_mm(struct mm_struct *mm)
{
	struct task_struct *tsk = current;

	WARN_ON_ONCE(!(tsk->flags & PF_KTHREAD));
	WARN_ON_ONCE(tsk->mm);

	use_mm(mm); // mm->count++
	set_fs(USER_DS);
}

static inline void kthread_unuse_mm(struct mm_struct *mm, mm_segment_t oldfs)
{
	struct task_struct *tsk = current;

	WARN_ON_ONCE(!(tsk->flags & PF_KTHREAD));
	WARN_ON_ONCE(!tsk->mm);

	set_fs(oldfs);
	unuse_mm(mm);
}

static inline int thread_get_mm(user_task)
{
	struct mm_struct *mm;

	task_lock(user_task);
	mm = user_task->mm;
	if (unlikely(!mm || !mmget_not_zero(mm)))   // mm->users++
		mm = NULL;
	task_unlock(user_task);

	if (mm) {
		kthread_use_mm(mm);
		return 0;
	}

	return -EFAULT;
}

// 上层需要记录old kernel fs
static void thread_drop_mm(mm_segment_t oldfs)
{
	struct mm_struct *mm = current->mm;

	if (mm) {
		kthread_unuse_mm(mm, oldfs);
		mmput(mm);
		current->mm = NULL;
	}
}
```