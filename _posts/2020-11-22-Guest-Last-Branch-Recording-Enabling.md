---
layout:     post
title:      Guest Last Branch Recording Enabling
subtitle:   perf LBR KVM
date:       2020-11-22
author:     qzwerty
header-img: img/blog-bj-1.png
catalog: true
tags:
    - vmi



---

## [Guest Last Branch Recording Enabling](https://lwn.net/Articles/814824/)

最后分支记录(LBR)是Intel处理器上的一个性能监视器单元(PMU)特性，它记录LBR堆栈中处理器获取的最近的分支的运行轨迹。本补丁系列将为大量KVM guest启用此特性。

用户空间可以通过`vm_ioctl KVM_CAP_X86_GUEST_LBR`来配置是否为每个客户机启用它。作为第一步，客户机只能在其cpu模型与主机相同的情况下启用LBR特性，因为LBR特性仍然是特定于模型的特性之一。
启用后，`IA32_PERF_CAPABILITIES`的[0,5]位和cpuid PDCM位中定义的记录格式也向客户公开。

如果在客户机上启用了它，那么客户机LBR驱动程序将像主机那样访问LBR MSR(包括IA32_DEBUGCTLMSR和堆栈MSR)。对与LBR相关的msr的第一次来宾访问总是可截获的。KVM陷阱将创建一个特殊的LBR事件(称为guest LBR事件)它启用callstack模式，并且不绑定任何硬件计数器。主机perf将像往常一样启用和调度此事件。

Guest对与lbr相关的msr的第一次访问被捕获到KVM上，创建一个guest LBR perf事件。这是一个常规的LBR perf事件，，它从perf子系统获得分配的LBR设施。一旦成功,LBR堆栈msr被传递给客户端以实现高效访问。但是，如果另一个主机LBR事件进入并接管该LBR设施，LBR msrs将被拦截，客人跟随对LBR msrs的访问将被捕获并且没有意义。

因为保存/恢复几十个LBR msr(例如32个LBR堆栈项)VMX转换会给频繁的VMX转换带来过多的开销，客户LBR事件本身将帮助保存/恢复LBR堆栈msr，在本地LBR事件调用堆栈的帮助下在上下文切换时，包括LBR_SELECT msr。

如果客户端不再在调度中访问与lbr相关的msr
时间片和LBR使能位未设置，vPMU将释放它的客户端
LBR事件作为未使用的vPMC的正常事件进行透传
LBR堆叠msrs状态被取消。

您可以在每个提交消息中检查更多的细节。