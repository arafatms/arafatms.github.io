---
layout: post
title: 【RustVMM】x,VirtIO实现原理_Vring数据结构
date: 2021-07-05 
tags: RustVMM
---

# VirtIO实现原理_Vring数据结构

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210709-Vring/20191231092024327.png?raw=true)


在virtio设备上传输批量数据的机制被称为virtqueue。 每个设备可以有零个或多个virtqueues。每个队列都有一个16位的队列大小参数，它设置条目的数量并表示队列的总大小。

每个virtqueue包含三个部分：
- Descriptor Table：存放Guest Driver提供的buffer的指针，每个条目指向一个Guest Driver分配的收发数据buffer。注意，VRing中buffer空间的分配永远由Guest Driver负责，Guest Driver发数据时，还需要向buffer填写数据，Guest Driver收数据时，分配buffer空间后通知Host向buffer中填写数据
- Available Ring：存放Decriptor Table索引，指向Descriptor Table中的一个entry。当Guest Driver向Vring中添加buffer时，可以一次添加一个或多个buffer，所有buffer组成一个Descriptor chain，Guest Driver添加buffer成功后，需要将Descriptor chain头部的地址记录到Avail Ring中，让Host端能够知道新的可用的buffer是从VRing的哪个地方开始的。Host查找Descriptor chain头部地址，需要经过两次索引Buffer Adress = Descriptor Table[Avail Ring[last_avail_idx]]，last_avail_idx是Host端记录的Guest上一次增加的buffer在Avail Ring中的位置。Guest Driver每添加一次buffer，就将Avail Ring的idx加1，以表示自己工作在Avail Ring中的哪个位置。Avail Rring是Guest维护，提供给Host用
- Used Ring：同Available Ring一样，存放Decriptor Table索引。当Host根据Avail Ring中提供的信息从VRing中取出buffer，处理完之后，更新Used Ring，把这一次处理的Descriptor chain头部的地址放到Used Ring中。Host每取一次buffer，就将Used Ring的idx加1，以表示自己工作在Used Ring中的哪个位置。Used Ring是Host维护，提供给Guest用

## Descriptor Table
一般的，每个Descriptor table都指向一个Guest buffer， 可以理解为一段数据。

```
struct virtq_desc {
    /* Address (guest-physical). buffer的GPA */
    le64 addr;
    /* Length. buffer大小 */
    le32 len;

/* This marks a buffer as continuing via the next field. 表明这个buffer之后还有一段buffer，这些有关联的buffer成为一个描述符链表（Descriptor chain)*/
#define VIRTQ_DESC_F_NEXT   1
/* This marks a buffer as device write-only (otherwise device read-only). */
#define VIRTQ_DESC_F_WRITE     2
/* This means the buffer contains a list of buffer descriptors. 这段可以理解为二级表的概念，说明这个描述符的addr指向guest内存中的任意位置的一个描述符（打破了VRING的大小限制）*/
#define VIRTQ_DESC_F_INDIRECT   4
    /* The flags as indicated above. */
    le16 flags;
    /* Next field if flags & NEXT， 如果flags & VIRTQ_DESC_F_NEXT，说明这个描述符不是Descriptor chain中最后一个描述符。注意：next存放的是Descriptor table 中的索引*/
    le16 next;
};
```

## Available Ring
驱动程序（driver）利用Available Ring给设备提供buffer。每个ring入口指向一个描述符链表的头。Available Ring被驱动程序写，被设备读。
```
struct virtq_avail {
#define VIRTQ_AVAIL_F_NO_INTERRUPT      1
    le16 flags;
    /* idx 字段指示驱动程序将在Descriptor Table中放置下一个描述符条目的位置 */
    le16 idx;
    le16 ring[ /* Queue Size */ ];
};
```
例如，系统刚开始时idx=0，表示如果有请求到达时就使用ring[idx(0)]位置的Descriptor指向完成这段请求所需要的buffer。

## Used Ring
Used Ring只被设备写，被驱动读出来。
```
struct virtq_used {
#define VIRTQ_USED_F_NO_NOTIFY  1
    le16 flags;
    /* host 下次处理的buffer在Used Ring的位置 */
    le16 idx;
    /* 存放Descriptor Table索引的环。Used ring不同的是，添加了len字段
    struct virtq_used_elem ring[ /* Queue Size */];
    le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */
};

/* le32 is used here for ids for padding reasons. */
struct virtq_used_elem {
    /* Index of start of used descriptor chain. */
    le32 id;
    /* Total length of the descriptor chain which was used (written to) */
    le32 len;
};
```

