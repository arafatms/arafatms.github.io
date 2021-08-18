---
layout: post
title: 【RustVMM】2.Cloud-Hypervisor 脏页跟踪机制
date: 2021-08-18 
tags: RustVMM
---

# Cloud-Hypervisor 脏页跟踪机制

## 控制kvm记录脏页
### 开启脏页记录
创建用户空间映射时，提供启用脏页日志记录的控件, 主要是开启kvm模块的脏页跟踪机制
```
# hypervisor/src/kvm/mod.rs
//构造memoryRegion结构时如果需要将KVM_MEM_LOG_DIRTY_PAGES添加到flags成员中
fn make_user_memory_region {
    flags = * | if log_dirty_pages {KVM_MEM_LOG_DIRTY_PAGES} 
}

fn create_user_memory_region {
    // Keep track of the regions that need dirty pages log
    对于每一个用户映射空间都创建一个KvmDirtyLogSlot结构，并在KvmVm结构中的哈希表dirty_log_slots中保存
    struct KvmDirtyLogSlot {
        slot: u32,
        guest_phys_addr: u64,
        memory_size: u64,
        userspace_addr: u64,
    }
}

fn start_dirty_log {
    根据每一个KvmDirtyLogSlot构造MemoryRegion，将flags设为KVM_MEM_LOG_DIRTY_PAGES
    VmFd.set_user_memory_region 通知KVM开启脏页跟踪
}

fn stop_dirty_log //与start相反
```

此时我们有两种方式开启脏页跟踪
- 创建MemoryRegion时开启
```
make_user_memory_region
create_user_memory_region   |
    set_user_memory_region
```
- 后续调用start_dirty_log开启
```
make_user_memory_region(log=false)
create_user_memory_region
start_dirty_log |
    set_user_memory_region
```

### 获取脏页
通过调用vmfd.opts同步内核中的脏页到cloudhypervisoer进程(kvm模块处理流程见<https://arafatms.github.io/2021/08/KvmDirtyPageTracking/>)
```
# hypervisor/src/kvm/mod.rs
fn get_dirty_log {
    VmFd.get_dirty_log(slot, memory_size) -> vm::Result<Vec<u64>>
}

```

## MemoryRangeTable
### MemoryRangeTable相关数据结构
用于记录内存脏页范围，提供一种创建用户空间映射时指定跟踪脏页的功能。记录以块(64Page)为单位.
```
pub struct MemoryRange {
    pub gpa: u64,
    pub length: u64,
}

pub struct MemoryRangeTable {
    data: Vec<MemoryRange>,
}

struct GuestRamMapping {
    slot: u32,
    gpa: u64,
    size: u64,
}

pub struct MemoryManager {
    ...
    user_provided_zones: bool,
    snapshot_memory_regions: Vec<MemoryRegion>,
    memory_zones: MemoryZones,

    // Keep track of calls to create_userspace_mapping() for guest RAM.
    // This is useful for getting the dirty pages as we need to know the
    // slots that the mapping is created in.
    guest_ram_mappings: Vec<GuestRamMapping>,
}

```

### MemoryRangeTable 相关函数
在这一层咱们需要做的是对于每一个MemoryRange根据kvm模块的接口生产脏块位图
```
# vm-migration/src/protocol.rs
impl MemoryRangeTable {
    fn from_bitmap(bitmap: Vec<u64>, start_addr: u64) {
        根据block为单位构造出MemoryRange，并push到table //此处算法感觉可以优化
    }
}
```

## 总结
CloudHypervisor实现了以最简单的方式获取脏页位图，并完成结构化。后续讲解怎么根据构造的MemoryRangeTable实现snapshot/restore或者在线迁移等功能。