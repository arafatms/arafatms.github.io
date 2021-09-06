---
layout: post
title: 【AMD64开发手册】妙招，CPUID指令的妙用
date: 2021-9-6 
tags: AMD64开发手册    
---


# CPUID指令的妙用
CPUID指令可以查看cpu特性和能力。该指令可以在任何PL下执行。AMD架构为CPUID指令提供了很多微函数，只需要调用CPUID指令之前往EAX寄存器输入相应的值即可，有些函数接受往ECX寄存器输入额外的参数。
- 通用函数：EAX值 0000 0000h～0000 FFFFh
- 扩展函数：EAX值 8000 0000h～8000 FFFFh
特性结果返回至EAX，EBX，ECX，EDX寄存器中。

通常AMD开发手册中用以下公式表示CPUID相关函数

> FnXXXX_XXXX_RRR[FieldName]_xYY

- XXXX_XXXX: 传入到EAX的值
- RRR：返回到哪个寄存器，如EAX，EBX。。。
- xYY：ECX的额外传参

比如，Fn8000_001F_EAX[Bit 1] = 1 表示系统开启SEV。