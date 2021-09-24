---
layout: post
title: 【内核之kvm】x, KVM脏页跟踪机制
date: 2021-08-18 
tags: 内核之kvm
---

# KVM脏页跟踪机制

>kvm内核主要完成传递脏页位图给VMM、清空脏页位图、清除脏页对应EPT页表项的脏标志。

## kvm_vm_ioctl(KVM_GET_DIRTY_LOG)
由kvm_log_sync->kvm_physical_sync_dirty_bitmap进行调用。在看下面的代码时，可以回到这里查看总体结构图，以防迷失：
<img src="/images/posts/20210818-KvmDirtyMap/1554690667848.png" width="700px" />

```
kvm_vm_ioctl(s, KVM_GET_DIRTY_LOG, &d)  |
    case KVM_GET_DIRTY_LOG:
        copy_from_user(&log, argp, sizeof(log))
        ...
        kvm_vm_ioctl_get_dirty_log(kvm, &log);  |
            mutex_lock(&kvm->slots_lock);
            ...
            kvm_get_dirty_log_protect(kvm, log, &is_dirty); |
                根据log->slot找到kvm_memory_slot，对应关系见其他文章
                dirty_bitmap = memslot->dirty_bitmap;

                //计算需要多少字节的位图，才能映射完当前的kvm_memory_slot
	            n = kvm_dirty_bitmap_bytes(memslot);

                //等下会将dirty_bitmap位图复制到dirty_bitmap_buffer开始的位置
                dirty_bitmap_buffer = dirty_bitmap + n / sizeof(long);
                
                //初始化dirty_bitmap_buffer开始的n个字节为0
                //dirty_bitmap_buffer主要是用来保存当前的dirty_bitmap，后面会设置到log->dirty_bitmap返回给Qemu
                //memslot->dirty_bitmap前n字节用于kvm记录脏页位图，后面用于返回给VMM
                memset(dirty_bitmap_buffer, 0, n);

                spin_lock(&kvm->mmu_lock);
                *is_dirty = false;
                for (i = 0; i < n / sizeof(long); i++) {//n是8字节对齐的
                    if (!dirty_bitmap[i])//非脏页直接跳过
                        continue;

                    *is_dirty = true;

                    mask = xchg(&dirty_bitmap[i], 0);//把dirty_bitmap对应的项置0，且置换出原来的值
                    dirty_bitmap_buffer[i] = mask;//设置对应dirty_bitmap_buffer

                    //接下来开始清除EPT页表项的脏标志
                    if (mask) {
                        //脏页位图上的偏移
                        offset = i * BITS_PER_LONG;
                        //由于mask里有一些可能还是0，所以需要传入mask避免不需要的 清除脏标志操作
                        //从offset开始的64bit，都需要检查
                        kvm_arch_mmu_enable_log_dirty_pt_masked(kvm, memslot, offset, mask);    ｜
                            kvm_x86_ops->enable_log_dirty_pt_masked(kvm, slot, gfn_offset,mask);    ｜
                                vmx_enable_log_dirty_pt_masked  ｜
                                    kvm_mmu_clear_dirty_pt_masked   ｜
                                        //对mask 64bit轮训调用
                                        // 利用rmap？
                                        rmap_head = __gfn_to_rmap(slot->base_gfn + gfn_offset + __ffs(mask),PT_PAGE_TABLE_LEVEL, slot);
                                        // 清除EPT页表项脏标志
                                        __rmap_clear_dirty(kvm, rmap_head);
                    }
                }

                spin_unlock(&kvm->mmu_lock);
                //位图返回给用户态VMM
                copy_to_user(log->dirty_bitmap, dirty_bitmap_buffer, n)

            ...
            mutex_unlock(&kvm->slots_lock);


```

参考文章:
版权声明：本文为CSDN博主「Toby Jiang」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_34372407/article/details/89084321