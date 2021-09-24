<style>
img{
    width: 60%;
    padding-left: 20%;
}
</style>

---
layout: post
title: 【AMD64开发手册】十二，任务管理
date: 2021-9-24
tags: AMD64开发手册    
---

> 64位模式下使用软件切换，AMD手册的这个章节大部分内容介绍legacy模式下的硬件切换，挑几个重要内容讲解

虽然64位模式下没有使用硬件切换，但是内核需要提供起码一个TSS（Task State Segment），而TR寄存器指向这个TSS。
TR寄存器构造如下,其中最下面三个域是OS不感知的：
```
____________________________________________________________
|                                   | selector             |
|                                   | Doscriptor attributes|
|                   |   32 bit Descriptor table limit      |
|           64 bit Descriptor table base address           |
____________________________________________________________
```
TR寄存器中的Selector用于从GDT中获取后三个域。

TSS的构造如下：

![avatar](https://raw.githubusercontent.com/arafatms/arafatms.github.io/main/images/posts/20210924-TaskManagement/TSS.png?raw=true)
