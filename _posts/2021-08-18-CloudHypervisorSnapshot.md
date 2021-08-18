---
layout: post
title: 【RustVMM】3.Cloud-Hypervisor 系统快照机制
date: 2021-08-18 
tags: RustVMM
---

# CloudHypervisor 快照机制

## snapshot
snapshot结构是个树，记录所有子节点的snapshot。

```
/// A Snapshottable component's snapshot is a tree of snapshots, where leafs
/// contain the snapshot data. Nodes of this tree track all their children
/// through the snapshots field, which is basically their sub-components.
/// Leaves will typically have an empty snapshots map, while nodes usually
/// carry an empty snapshot_data.
///
/// For example, a device manager snapshot is the composition of all its
/// devices snapshots. The device manager Snapshot would have no snapshot_data
/// but one Snapshot child per tracked device. Then each device's Snapshot
/// would carry an empty snapshots map but a map of SnapshotDataSection, i.e.
/// the actual device snapshot data.
#[derive(Clone, Default, Deserialize, Serialize)]
pub struct Snapshot {
    /// The Snapshottable component id.
    pub id: String,

    /// The Snapshottable component snapshots.
    pub snapshots: std::collections::BTreeMap<String, Box<Snapshot>>,

    /// The Snapshottable component's snapshot data.
    /// A map of snapshot sections, indexed by the section ids.
    pub snapshot_data: std::collections::HashMap<String, SnapshotDataSection>,
}
```

### snapshot数据结构行为
提供三种行为trait，用来通用记录各个子节点的snapshot结构：
- The snapshottable component id
- Take a component snapshot.
- Restore a component from its snapshot.

对此树状结构的操作有如下集中，上述trait均通过这些impl实现对snapshot树的操作：

例如，对于MemoryManager构造snapshot树的子节点，对于其他设备如CpuManager，DeviceManager请读者自行阅读
```
impl Snapshottable for MemoryManager {
    fn snapshot {
        let mut memory_manager_snapshot = Snapshot::new(MEMORY_MANAGER_SNAPSHOT_ID); //初始化snapshot节点
        构造Vec<MemoryRegion>
        memory_manager_snapshot.add_data_section    //填充MemoryManager的snapshot节点
            将Vec<MemoryRegion>插入到snapshot节点的snapshot_data成员中

    }
}
```

### 收集系统所有组件的snapshot结构
```
vm_snapshot |
    vm.snapshot()   |
        检查vm.get_state()==VmState::Paused
        Get the Vm state. Return VM specific data
        vm_snapshot_data = serde_json::to_vec(VmSnapshot {     //保存众多vm相关信息
            config: self.get_config(),
            #[cfg(all(feature = "kvm", target_arch = "x86_64"))]
            clock: self.saved_clock,
            state: Some(vm_state),
            #[cfg(all(feature = "kvm", target_arch = "x86_64"))]
            common_cpuid
        }

        vm_snapshot.add_snapshot(self.cpu_manager.lock().unwrap().snapshot()?);
        vm_snapshot.add_snapshot(self.memory_manager.lock().unwrap().snapshot()?);
        vm_snapshot.add_snapshot(self.device_manager.lock().unwrap().snapshot()?);
        vm_snapshot.add_data_section(vm_snapshot_data)  //最后将之前保存的vm相关信息添加到根节点的snapshot_data成员中
        Ok(vm_snapshot)
```

## 发送/保存快照
CloudHypervisor提供一种行为trait来发送或者保存快照到指定路径,每个组件都可以自行注册处理逻辑, 最后构成树状调用栈，根节点依旧为Vm对象。
```
/// A transportable component can be sent or receive to a specific URL.
pub trait Transportable: Pausable + Snapshottable
```
提供两种行为，如：
- Send a component snapshot.
- Receive a component snapshot.

还是拿MemoryManager为例：
```
impl Transportable for MemoryManager {
    fn send(
        &self,
        _snapshot: &Snapshot,
        destination_url: &str,
    ) -> result::Result<(), MigratableError> {
        for region in self.snapshot_memory_regions.iter()   |
            memory_region_path = destination_url + MemoryRegion->content;   //对每一个内存区域打开一个文件用来保存guest内存中的数据
            guest_memory.write_all_to //将内存中的内容写入到文件memory_region_path中
    }
}
```

Vm对象作为保存快照行为的根节点，具体流程如下：
```
impl Transportable for Vm {
    fn send(
        &self,
        snapshot: &Snapshot,
        destination_url: &str,
    ) -> std::result::Result<(), MigratableError> {
        vm_snapshot_path = destination_url + 'vm.json'
        // Create the snapshot file
        // Serialize and write the snapshot
        // Tell the memory manager to also send/write its own snapshot.
        Ok(())
    }
```

## 总结
整个快照过程可以分为两种：
- 收集系统中所有组件的信息，构造json
- 将所有信息json保存到vm.json文件中
- 将内存中的数据保存到本地文件中
其中CloudHypervisor灵活使用Rust的trait特性，给各个组件提供了快照和发送/写入行为。

还原操作大致与快照相反，请读者自行阅读