---
layout:     post
title:      Record-Replay Architecture as a General Security Framework 
subtitle:   (HPCA'18)RnR-safe
date:       2020-02-17
author:     qzwerty
header-img: img/blog-bj-1.png
catalog: true
tags:
    - RR security
---

## (HPCA'18)32-Record-Replay Architecture as a General Security Framework
> 	RnR-Safe使用两个replayer。
> 
* 	一个总是打开的、快速的*Checkpoint* replayer，定期创建监控执行的检查点。
* 	一个用于详细分析的*Alarm* replayer，当出现威胁情报时触发。从检查点开始分析执行，来辨别报警原因是真正的攻击还是误报。可以多次重放进行不同层次的分析，直到完全理解。
	
### 1 INTRODUCTION

	代码注入攻击 -> 
	W⊕X(强制规定可写或可执行) -> 
	ROP（面向返回编程，基于代码重用，通过链接受害者程序的多个代码片段来构建攻击代码）

使用RnR，工作负载的初始执行将创建一个日志，可以在另一台机器上确定性的重播该日志。可以便于了解攻击发生的方式和时间。

作者探讨了一种硬件安全设计的方法，其中RnR用来弥补平衡「卸载入侵检查」与「消除检查的不确定性」。称这个框架为RnR-Safe，允许硬件在检测攻击时不那么精确（从而存在误报的可能），降低了安全硬件的成本。对于产生误报的问题，作者使用一个*on-the-fly* replayer，能够透明的验证警报是真正的攻击还是误报。

RnR-Safe依赖于运行在不同机器上的两种类型的on-the-fly replayer。具体如开头介绍。

#### 这篇文章的贡献：
1. 提出了RnR-Safe，一种全新的基于RR的硬件-软件方法来增强安全性。
2. 使用RnR-Safe来探测ROP攻击（重点关注内核），并讨论了实现过程中的挑战。
3. 评估。

#### 假设的系统和威胁模型
ROP攻击可以发生在内核或者用户上下文中，RnR-Safe可以保护这两者。最难保护的是内核，因为他具有特权和多上下文。受保护的系统（内核和应用程序）在vm中运行，VM的执行被连续记录。记录的同时在不同机器上重播，可能会发现ROP攻击。

作者假设攻击者可以通过「地址泄露」和「内存损坏漏洞」的任何组合对内核发起ROP攻击，从而劫持和损坏内核堆栈。假设用于RR的机器的主机OS和hypervisor是良性的，并且他们可以通过传统的内存页权限保护来保护guest vm不受损害。

### 2 背景
#### 2.1 RR
 本文假设：使用vm级别的RR，考虑单处理器硬件。因此，不确定性事件的来源是引发的中断和虚拟设备复制到guest machine中的数据。还假设使用hypervisor-mediated I/O的模型，如Xen或Qemu使用的模型。
 
> ReVirt[8]展示了一个在Linux内核中使用vm级别的RnR对检查时间和使用时间竞争条件进行事后离线分析的示例。
> 
> IntroVirt[11]使用vm级别的RnR进行探索，以确定一旦发现零日攻击，以前是否利用过系统。
> 
> Speck[13]探索了使用操作系统级猜测和程序级RnR的组合来删除程序关键路径的安全检查。
> 
> ParanoidAndroid[19]和Secloud[26]探讨了在云中维护移动设备副本的可能性，并在云中执行程序级的RnR。
> 
> Aftersight[14]建议使用vm级别的RnR对系统执行进行在线动态分析。

上述文章没有解决一些重要的软硬件设计问题（section 9）。

#### 2.2 ROP
本文中主要考虑针对内核的ROP攻击。
在较高的层次上，ROP攻击可以通过**影子堆栈（shadow stack）**探测。当调用指令被执行时，调用的指令地址被push到影子堆栈。在返回指令处，影子堆栈的栈顶被pop。当处理器使用的返回地址与从影子堆栈中弹出的返回地址不匹配时，就可以检测到ROP攻击。

影子堆栈在实现上存在的挑战：

1. 影子堆栈的有效性取决于他的完整性。因此影子堆栈必须被保护，使它不被自己所保护的软件攻击。这包括保护它不被内核（内核本身可能受到攻击）本身攻击。

2. 代码可能是高度嵌套的（如，递归）、多上下文的（如，内核），或不完全嵌套的（如，错误和异常处理）。在这些情况下，要正确处理误报（false positive），大多数解决方案在操作影子堆栈时添加了一些指令。这些指令会带来安全漏洞。

#### 2.3 ROP依然没有解决
从实用的角度看，现有的ROP保护技术存在局限性的原因有三：

**硬件介入**：有些方法需要介入硬件修改，难度较大。

**软件影响**：性能、适用性问题。

**完整性**：如ASLR，能被绕过。

还有Intel的控制流强制技术（Control-Flow Enforcement Technology，CET），以及基于分支轨迹分析的CFI验证技术。

本文使用RnR-Safe支持ROP保护。不使用硬件影子堆栈。通过软件在安全机器中的replayer实现影子堆栈。

#### 2.4 返回地址堆栈（Return Address Stack，RAS）
现代处理器使用硬件RAS来预测返回指令的目标。当执行函数调用时，硬件将函数返回地址push到RAS的top。当一条返回指令被解码时，硬件把RAS的top的条目pop出，作为预测的返回目标。软件无法访问RAS。

ROP攻击会导致RAS错误预测，是因为攻击者强制返回到一个意外的指令。然而RAS 的mispridictions也可能出现在正常程序执行中，不能说明发生了ROP攻击。

### 3 RnR安全框架
![](https://tva1.sinaimg.cn/large/0082zybpgy1gc20bjh6m4j30kg0ag762.jpg)

RnR-Safe的组织结构。

工作负载运行在Recorded VM上。它的hypervisor在软件日志中记录了所有的不确定性事件的执行。（记录的开销平均不到15%，by Pokam）作者在vm级别进行记录，以便也能保护操作系统。设计者在Recorded VM的硬件（如处理器和存储系统）中增加了一些支持，使得能够对于探测某些等级的攻击（存在误报可能）。当硬件或者负责记录的hypervisor怀疑被攻击，hypervisor就在日志中插入一个警报标记。根据工作负载的风险承受力，recorded vm可能继续运行，或者停止，直到警报被分析。

*Checkpointing Replayer VM*在本地重新执行工作负载。使用日志来插入所有的非确定性事件，使得执行确定性地遵循原始执行。如果在日志中发现警报，*Alarm Replayer VM*会对警报进行特征描述，检测它是攻击还是误报。

#### 3.1 RnR-Safe 执行模式
在RnR-safe中，监控记录的是正常执行。所有不确定性的输入都被hypervisor记录到日志中，硬件和hypervisor监控安全违规行为。这对于recorded VM是透明的。如果发现了违规行为，就在日志中插入警报条目。探测器必须捕获所有的潜在威胁，不允许漏报。

重放执行由两种replayer实现。

* Checkpointing Replayer：一直运行，大致相当于记录速度。通过日志进行工作负载的确定性重放，定期创建状态检查点。当在日志中发现警报标记时，checkpointing replayer将从最近的（通常是最新的）检查点启动Alarm Replayer。旧的检查点和日志条目会定期丢弃，以节约存储空间。

* Alarm Replayer：从给定的检查点重放日志条目，直到警报标记，同时对重放执行进行广泛的、特定于攻击的分析。目的是区分警报，是误报/描述攻击特征。比checkpointing replayer慢得多。

#### 3.2 RnR-Safe提供的服务
RnR-Safe具有三个安全优势：

**在相对简单的硬件上的健壮性**：将报警检测与攻击验证分开。

**灵活性**：当攻击者设计出新的攻击时，防御者可以从硬件和软件方面改进recorded vm和hypervisor，使他们能产生新的警报；从软件角度改进replaying vm，来产生新的分析技术。分析过程是在软件中进行的，所以增加新的replayer可以同时跟踪多种类型的攻击。

**执行审计**：RnR-Safe还允许对执行进行离线的详细分析。可以重放执行上下文来审计代码和数据。通过审计系统中的敏感流来识别安全违规。

#### 3.3 如何使用RnR-Safe
RnR-Safe可以被定制来检测不同的攻击。对于每种攻击，需要先针对机器中的威胁进行首次探测。
![](https://tva1.sinaimg.cn/large/0082zybpgy1gc46jsfxzkj30p00cyq69.jpg)
对于**ROP**：警报触发器是RAS的错误预测。首次检测是基于管理一个多线程的RAS，并使用了可接受RAS错误预测的白名单。重放的作用是模拟一个内核兼容的影子堆栈。

对于**JOP**：通过重定向间接分支或者调用指令来执行受害者的代码。防止这种攻击需要保留调用和分支指令的CFI。首次检测技术可以是最常见函数的开始和结束的地址表。将间接分支或调用目标与表进行比较，如果目标是函数的第一条指令，那么它就是合法的，当前功能内的间接分支目标也是合法的。否则将触发警报，replayer会对不常用的函数也进行验证。

对于**DOS**：os上的DOS攻击可以通过计数器检测到，这个计数器在每次内核进行上下文切换时递增。如果计数器在一段时间内没有增加很多，就会产生警报，然后replayer就会分析控制系统执行时间的代码。

### 4 针对内核ROP的案例
#### 4.1 主要思想
用来检测ROP的结构原语（primitive）是RAS。RAS存储返回指令的预测目标地址，不能被软件和内核访问。ROP通过导致返回到非预期的地址，从而导致RAS的错误预测（misprediction）。

ROP payload的执行一定会引起RAS的错误预测，不存在漏报。

RnR-Safe要求减少误报。在基本的RAS操作中，有几个主要的误报来源。作者用linux内核示例解释这些来源。

1. 多线程的影响。在多线程的环境中，在从Thread i 到thread j 的上下文切换中，RAS可能仍然保留一些属于 i 的条目，在执行 j 的代码时，这些条目可能被错误地pop，然后被用来预测。这样的话 j 会发生错误预测，当 i 再次执行时这些条目也不能再用了，所以 i 也会错误预测。

2. 内核中的非过程性返回。在某些情况下，比如上下文切换期间，内核向软件堆栈中插入一个地址，作为返回指令的目标。由于之前没有来自这个地址的调用，所以RAS不会包括这个条目，产生错误预测。

3. RAS下溢。如果代码执行了很多嵌套程序调用，RAS可能会删除一些早期的返回地址。之后，当执行从内部调用返回，尝试弹出外部调用对应的条目时，RAS是空的，产生错误预测。

4. 程序调用的不完全嵌套。程序被调用但是没有返回的情况。在内核中，这些事件通常只作为内核中bug恢复过程的一部分发生。当内核执行遇到可恢复的错误时，会启动一个恢复进程，终止当前线程执行，使当前线程所有RAS条目处于孤立状态。对于用户模式代码，发生更频繁，尤其是setjmp/longjmp调用中，用于异常处理。

可以看到，误报发生的情况很多。所以需要增加支持来降低误报率。RnR-Safe增加了对前两个误报源的支持。但是要完全消除误报，需要对软硬件进行重大修改。RnR-Safe会对后两个误报源发出警报，并依赖replayer将事件标记为误报，然后丢弃。

#### 4.2 基本设计
如图1，workload（应用程序+内核）在Recorded VM中运行。hypervisor创建一个输入日志，该日志被发送到Checkpointing Replayer vm并由其使用。如果workload执行了一条返回指令，并且硬件发现实际返回目标与RAS中预测的目标不匹配，就会触发vm exit。然后，hypervisor在输入日志中插入一条ROP警报条目。根据判断决定在replaying vm处理警报时hypervisor是否需要停止recorded vm。

当checkpointing replayer 使用日志时，如果发现警报条目，就会触发Alarm Replayer从最近的检查点执行。Alarm Replayer确定警报是误报还是真实的ROP。

这个基本的RnR-Safe设计不会漏报，但是会有很多误报。下面将如何扩展这个基本设计来减少误报（刚才提到的4种误报情况）。

#### 4.3 支持多线程环境
> 在多线程的环境中，当一个线程被取消调度，它可能会在RAS中留下返回地址。这样的地址可能会被后续线程pop，产生错误预测。另外，当重新调度原始线程时，它可能会弹出属于其他线程的RAS条目，导致错误预测。由此产生误报。

为了解决这个问题，RnR-Safe改进了microcode的虚拟化硬件，在上下文切换中额外执行以下操作。

1. 将当前的RAS保存到内核无法触及的安全内存区域。

2. 根据即将运行的线程的需要恢复RAS状态。

为了知道要移动数据的正确内存区域，硬件使用hypervisor设置的新硬件指针。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc6jzoksjpj30mw0bwwgr.jpg)

结构如图2所示。memory structure（software）是备份的RAS 数组（BackRAS array）。数组中的每个条目代表一个线程，并且有一个RAS和一个计数器，计数器表示RAS的条目数。需要使用计数器来知道后续需要重新加载的条目数量。处理器硬件包括一个指针（BackASptr），指向当前正在运行的线程的BackRAS。指针由hypervisor设置。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc6ko8w1n8j30pi09a40h.jpg)

使用的逻辑如图3。在上下文切换中，转换到hypervisor的过程中**（VMExit）**，硬件将RAS保存到BackRASptr指向的BackRAS条目。此外还保存了已保存条目的计数。我们的测量结果显示，转换到hypervisor需要大约1000个周期。我们估计备份RAS会增加大概200个周期。然后，当hypervisor运行时，它将更改BackRASptr以指向新线程的BackRAS条目。最后，在转换回客户端时**（VMEnter）**，微编程硬件将正确的BackRAS条目加载到RAS中，又花费200周期。

要操作BackRASptr，guest内核中的上下文切换需要通知hypervisor，hypervisor要确定要调度的新线程。（5.2.1解释如何在不修改guest 内核情况下完成任务）

#### 4.4 支持非程序性返回
> 有时候，内核会在软件堆栈中插入一个地址，然后执行一个将该地址作为目标的返回。这样的话没有相关的过程调用，RAS没有push一个条目，会发生错误预测。在这种情况下，不应该pop出RAS，因为这样做会破坏RAS状态。

> 比如说，在上下文切换完成时，在启动下一个线程之前，内核会执行这样一个返回，代表新线程开始执行代码。这段代码是用汇编语言编写的，并将控制流定向到内核代码中三个定义良好的位置。这些位置依照是否涉及fork线程、执行内核线程、重新调度任务，完成任务切换。

为了解决这个问题，RnR-Safe扩展了处理器硬件，提供了一个白名单地址表。包括一个单条目返回白名单*RetWhitelist*，记录非过程返回指令的PC，和一个目标白名单*TarWhitelist*，记录能够被作为返回目标的三种指令的PC。在返回地址预测时，如果一个返回的PC和目标PC能匹配表中的条目，那么RAS不会弹出，也不会发生警报。这些表只对hypervisor可写。

使用逻辑和时间轴如下。当一条指令被解码并识别为返回时，硬件检查它的PC是否在RetWhitelist中，如果是，就不会pop RAS，并设置一个Whitelisted flag。之后，目标地址生成之后，如果设置了Whitelisted flag，硬件会检查这个PC是否在TarWhitelist中。如果不是，就触发vm Exit。

通过分析guest kernel的二进制映像可以找到白名单地址。5.1节会解释hypervisor如何在进入vm时使用识别出的地址填充RetWhitelist和TarWhitelist。
#### 4.5 RAS下溢和不完全嵌套
这两种误报，很少发生。RnR-Safe的处理方法是不在Recorded
 vm中使用复杂的硬件，而是发出警报并依赖replayer vm来处理。
 
* 下溢发生在内核或应用程序执行了足够多的嵌套程序调用，RAS从很早以前的返回地址退出时（就是说为了节约空间这条RAS已经被清除了）。当硬件试图从返回指令的RAS中取出该条目时，发现RAS是空的。称为RAS下溢，产生错误预测。
 
 处理方法。当一个RAS条目将要在recorded vm中被清除时，硬件触发一个vm exit，然后将这个即将被清除的RAS条目转移存储到内核无法访问的特殊内存中。然后，hypervisor在日志中将这些数据记录为已清除。当下溢产生导致错误预测时，硬件会发出警报，hypervisor在日志中设置警报记录。

* 程序嵌套不完善，如setjump和longjump造成的嵌套。很少发生，但需要非常复杂的硬件来透明地处理。当不完全的程序嵌套导致RAS错误预测时，RnR-Safe只在日志中设置警报记录。

根据警报记录，replayer会对两种情况进行处理。replayer能够把被清除的记录和由于RAS下溢产生的警报记录进行匹配。也能够轻松识别setjump和longjump，并修复软件RAS。

这些误报对总体性能影响很小。

#### 4.6 重放平台
##### 4.6.1 Checkpointing Replayer（CR）
首先描述检查点的内容。
![](https://tva1.sinaimg.cn/large/0082zybpgy1gc7pi0ewuuj30pa0gktav.jpg)
图4显示了三个检查点，每个检查点有三个组件。

1. 所有的vm状态。包括vm的所有内存页，在检查点时的处理器状态（PC，堆栈指针，其他的寄存器）页面和虚拟磁盘映像内容。虚拟磁盘映像的内容是指vm写入虚拟磁盘的状态。需要它的原因是，如果以后的执行读取了这些数据，他们不会出现在输入日志里。但是，状态检查点是递增的，由于采用了周期性的检查点，检查点只会保留从上一个检查点以来修改过的页面和块的副本；对于没有修改过的页面和块，在最晚修改它的检查点中保留一个指向它的指针。

2. 指向输入日志缓冲区的指针（InputLogPtr）。指针指向检查点之后要处理的下一个输入日志记录。

3. 检查点此时的BackRAS。见4.6.2.。

在这个基础上，介绍CR的操作。CR确定性的重放workload。当距离最后一个检查点的时间超过某个阈值，并且vm exit发生时，设置一个检查点。在这个检查点建立过程中，首先，硬件自动把RAS保存到BackRAS中。然后，CR通过保存以下内容来创建检查点：（1）具有处理器状态的页面，自前一个检查点以来修改的所有内存页面和磁盘块（以及指向未修改部分的指针），（2）当前的BackRAS，（3）当前的InputLogPtr。然后，CR把所有页面标记为copy-on-write，继续执行。在执行期间，当自上一个检查点以来首次写入一个页面时，将生成一个副本并持续使用。

CR会定期回收检查点。然而，只能回收没有被后续检查点指向的内存页面或磁盘块。

重放发生在一个安全的平台上，硬件触发ROP警报的能力是被禁用的。因为重放不会产生警报。但是，硬件仍然会在RAS上下文切换点转储RAS。这可以确保在检查点处，CR可以重建完整的BackRAS的最新状态，并将其隐藏在检查点中。

##### 4.6.2 Alarm Replayer（AR）
当CR遇到警报时，会从紧接警报之前的检查点启动一个AR vm。AR确定警报是由ROP引起的还是误报。AR首先使用检查点初始化vm状态。将检查点中的所有页面和块标记为copy-on-write，避免修改初始状态。然后，它将检查点的BackRAS读入软件数据结构中，用于模拟RAS。然后，他把保存在内存中的处理器状态加载到处理器寄存器中。最后，他开始执行，从检查点InputLogPtr开始读取日志。

AR以确定性的方式依照输入日志在本地执行原始的workload，直到到达警报标记。AR阻断每一个调用和返回指令，产生vm exit并将控制权转移到hypervisor。然后，在软件中构建一个无界的RAS，添加了我们对于多线程和非程序性返回的扩展。运行AR的硬件既不会转储RAS的状态，也不会触发ROP警报。这两个功能被禁用。

一旦AR遇到输入日志中的警报，它会检查RAS错误匹配是否只能被解释为ROP攻击。如果是，AR提供ROP攻击时系统的状态。一个专业的程序员可以研究状态来收集攻击信息。此外，AR可以多次重放，with increasing levels of instrumentation，从不同的检查点开始，来完全描述攻击。

输入日志中的删除记录导致输入日志中的下溢警报记录以特殊方式处理。CR自己来处理更简单。在CR处理输入日志并查找退出记录时，把他们保存到软件结构中。当CR在输入日志中发现一条下溢警报记录时，它会将其与来自同一线程的最新evict记录进行比较。如果匹配，警报为假，CR从结构中删除对应的evict记录。否则，启动AR来处理警报。

Section 6展示了一次攻击。允许同时运行多个AR，分析相同或不同的ROP警报。

### 5 实现
本节总结所述体系结构所需的硬件和hypervisor支持。根据Intel VT的术语，我们使用VMCS（VM Control Structure）来表示内存（in-memory）中的控制结构，hypervisor通过这种结构与虚拟化硬件通信，并配置虚拟化硬件。使用VMEnter表示执行从hypervisor转移到vm，VMExit表示执行从vm转移到hypervisor。

#### 5.1 硬件支持
RnR-Safe所需的硬件支持包括重放平台，小型的RAS硬件扩展，BackRASptr寄存器和两个白名单表。RR基础结构可用于各种debug和安全分析。RAS扩展包括:当RAS要退出一个条目时触发一个异常，以及将microcode转储到内存中并恢复BackRAS数组（？），以及将被逐出的RAS条目转储到内存中。还有用于维护BackRASptr和白名单表的microcode。

对于BackRASptr的和两个白名单，我们给VMCS扩展了三个新的字段。microcode读取这些字段来编写这三种处理器硬件结构。然后，microcode使用BackRASptr的值将RAS的内容转储到某些VMExit的活跃BackRAS条目中，并将活跃BackRAS条目的内容读取到某些VMEnter的RAS中。

#### 5.2 hypervisor支持
##### 5.2.1 在上下文交换中编程BackRASPtr
hypervisor需要在RR期间对guest kernel中的所有上下文切换进行干预。在Linux中有一条指令将堆栈指针从指向当前线程的堆栈改为指向下一个线程的堆栈。通过在这个指令上设置一个陷阱，hypervisor可以在guest执行这个指令时强制执行VMExit。作为VMExit微代码的一部分，硬件将RAS转储到BackRASPtr指向的内存位置。

VMExit完成并将控制转移到hypervisor之后，hypervisor可以内省guest kernel的状态，来确定下一个要调度的线程。在Linux中，如果线程的堆栈指针已知，则很容易找到线程的描述符（task_struct）。由于已经在改变处理器堆栈的指针的指令上设置了陷阱，就可以通过检查vm寄存器内容来找到下一个线程的堆栈指针。使用这个堆栈指针，可以在vm的内存中找到相应的task_struct描述符，并从该描述符中读取下一个线程的ID。

hypervisor将BackRAS存储在guest machine无法访问的内存区域中。它将其存储为一个hash表，把线程的ID（key）映射到BackRAS条目（value）。这样的话，一旦找到下一个线程的ID，hypervisor将检查映射来确定BackRAS条目。然后，hypervisor会把VMCS的BackRAS字段设置为指向BackRAS条目。

##### 5.2.2 回收BackRAS条目
在linux中，线程可以被kill，ID会被重用。为了保持BackRAS一致，当线程被杀死时需要移除该线程在BackRAS中的条目。与上下文切换的情况类似，hypervisor对在guest kernel中实现此功能的函数设置了一个陷阱，在它执行时强制执行VMExit。这样就可以通过内省找到线程ID，然后删除相关的BackRAS条目。当创建一个线程并需要分配BackRAS条目时，也采用类似的办法。

### 6 内核ROP攻击
![](https://tva1.sinaimg.cn/large/0082zybpgy1gc9yn2cexoj30pc0pon1j.jpg)
构建并发起了如图所示的ROP攻击。在recorded vm中，当workload调用了c中的脆弱性代码，硬件将调用点的返回地址（称为Return Address）推入RAS。这和存储在e中软件堆栈缓冲区顶层的是同一个地址。当恶意的strcpy完成后，软件栈变成f。当程序执行到恶意过程return时，硬件使用RAS来预测执行将转移到的Return Address。如f所示，实际上，return的目标被解析为gadget G1，这个错误匹配会导致vm发出警报。

然后recorded vm hypervisor在日志中插入一个标记，并可能决定停止vm。
当CR看到日志中的警报标记，会从最近的检查点启动AR。当AR开始执行，会在软件中生成（model？）RAS。在警报点，它观测到的返回预测目标（RAS中的）和实际目标（软件堆栈中）不匹配，说明是ROP攻击。（就是AR检查是否是误报。。？）

之后，hypervisor可以分析系统。可以使用虚拟机内省来分析还没有被gadget执行造成污染的vm状态，还可以在更早的时间启动额外的AR，来对系统进行更深入的分析。

1. 攻击是怎么开始的？hypervisor通过使用触发警报的返回指令来判断攻击发生的脆弱代码是哪。使用模拟RAS的顶部的地址来确定调用点。对脆弱过程（vulnerable procedure）的分析可以得出缓冲区溢出的存在。

2. 谁攻击的？hypervisor还可以确定当前线程的ID，提取已登陆的用户，确定建立了那些网络连接。

3. 攻击者做了些什么？对软件堆栈的分析可以揭示攻击者使用的gadget。在这种情况下它们没有执行。如果执行了，hypervisor可以使用vm内省来分析哪些文件被修改，使用了哪些socket，fork了哪些线程。这些信息很容易获取，因为workload没有运行。

### 7 实验
#### 7.1 评价目标
本文评估目标是评估记录的开销，以及使用检查点和警报重放器的开销。还有日志生成的速率，保存/恢复RAS所需的带宽和警报的频率。其他信息包括攻击和检测之间的时间窗口，在此窗口期间产生的日志以及系统需要保留的检查点数量。

本文只使用了section 6中描述的ROP攻击。收集和分析多个真实的内核ROP是将来的工作。

#### 7.2 评估环境
使用了两个评估环境。

第一个评估记录和重放模式的性能。使用了Insight，这是一个基于修改Linux KVM hypervisor和QEMU设备的工具。由于KVM hypervisor可以利用Intel VTx扩展在硬件中虚拟化处理器，因此该设置的性能数字代表了真实世界的机器。

第二个评估技术的正确性和硬件功能特性。为此，在仿真模式中使用qemu。在这种模式下，qemu还通过系统软件的动态翻译模拟处理器。这个模式可以方便地对硬件进行仿真和功能评估。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gca0n6bmunj30oq0jqq7b.jpg)

表2显示用于性能评估的系统配置。

表3显示benchmark。benchmark：ﬁleio and mysql from SysBench [63], apache, make (Linux内核的编译), and radiosity from SPLASH-2。

#### 7.3 处理非确定性事件
**同步非确定性事件** 诸如rdtsc（读取时间戳计数器）或rdrand（读取随机数生成器）之类的指令返回不确定的结果。对内存映射IO（memory mapping IO，MMIO）等内存区域的访问也是不确定的。VMCS控制处理器何时执行VMExit。在记录期间，将这些控件配置为同步捕获，允许hypervisor记录他们的结果。重放时相同，这些事件被确定性的重新生成。

网络输入是一种特殊的情况，在本系统中也是同步。网络数据包到达物理网卡（NIC）本质上是异步的，但数据是在同步VMExit边界上交付给vm的。由此简化了网络事件的记录和重放。

**异步非确定性事件** 更具有挑战性。来自外部中断。这些中断来自于其他处理器或磁盘等物理设备。VMCS可以在这些事件上产生VMExit，然而这些VMExit是异步的，不会在重放时重复相同的指令。因此必须手动重新创建。

在同一处理器上下文捕获vm并不简单。Insight使用性能计数器来产生尽可能接近回放中需要的点的VMExit。这样，处理器是单步执行的，直到执行到达所需的注入点。每一步都要产生一个VMExit的开销（1000个周期）。

#### 7.4 评估重放开销
为评估checkpointing replay的开销，我们重用「在fork系统调用期间使用过的」Linux copy-on-write实现。vm的虚拟内存是在宿主机上运行的用户空间QEMU进程中分配的。只需稍作修改，就可以通过fork qemu进程来创建检查点。

AR在每次调用和返回指令处model RAS。然而当前Intel VTx扩展不支持捕获调用和返回指令。因此为了度量alarm replay的性能影响，我们将gcc修改为通过在内核上下文切换之前，调用和返回指令之前插入debug exception来检测二进制文件。debug exception是一个单字节的op-code（0xCC），通过引发调试异常来捕获指令。VMCS被配置为在debug exception中引发VMExit。这使我们能够模拟AR的行为，模块化由于Linux二进制文件的大小增加了0.11%而对性能造成的轻微影响。

#### 7.5 评估硬件方案
在二进制转换模式下，qemu只使用软件虚拟化处理器。这种模式慢得多，但允许对硬件进行模拟。使用这种模式来评估我们在RnR-Safe中提出的硬件修改。在缺省情况下模拟48个条目的RAS。

### 8 评估
#### 8.1 记录
在记录阶段生成日志，在上下文切换时保存和恢复RAS。我们要求hypervisor作为中介的I/O，防止使用半虚拟化（PV）网络驱动程序。图5（a）比较了本方案（Rec）与其他三种设置的执行时间：不记录PV驱动（NoRecPV），不记录也没有PV驱动（NoRec），以及记录时不保存和恢复上下文切换中的RAS（RecNoRAS）。，每个基准都规范化为NoRec。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gcb3r03r0ij31680e8n2v.jpg)

可以看到，禁用PV会使这些基准的执行时间增加25-150%。Apache和fileio受影响最大，mysql受影响不大，因为它通过在内存中缓存最近访问的表来避免访问磁盘。注意，RnR已经成功应用于PV驱动了[25]，把这些技术应用到我们的解决方案中将从系统中消除这些开销。

Rec平均比NoRec多花27%的时间，RecNoRAS比NoRec长24%。这些开销是适度的，并且会在对记录的合理优化改进中减少。如Pokam测量他们对记录的改进只增加13%的开销。

为了理解开销的来源，图5（b）将造成Rec减速的原因根据来源分解为rdtsc，pio/mmio，中断，网络包内容，以及在上下文切换时保存和恢复RAS。

可以看到所有基准测试的主要开销都来源于记录rdtsc。这个事件发生很频繁，尤其是在fileio和mysql中，应用程序本身会产生多次计时器读取来度量事务速度。此外，fileio使用pio发出磁盘命令和控制信号。它也有DMA活动，在信号文件访问完成时导致中断事件。Apache接收网络数据包，使用mmio访问来检索数据包。计算密集型的基准测试（make和radiosity）开销很小。最后，保存和恢复RAS平均只带来了4%的开销。

图6（a）和（b）分别显示所有benchmark的输入日志生成速率和RAS保存和恢复的带宽。没有进行数据压缩。可以看到日志的生成速率很低。Apache的输入日志生成速率很高（4MB/s），因为记录了网络包的内容。而且，在上下文切换时保存和恢复RAS的带宽很小。总的来说，架构对内存系统的影响是有限的。
![](https://tva1.sinaimg.cn/large/0082zybpgy1gcb3nne4o6j30pc0dw40o.jpg)

#### 8.2 最小化内核中的误报
RnR-Safe安全硬件消除了内核中的大多数错误警报，只允许向replayer报告一部分错误警报。图8显示了内核错误警报的数量，分为使用白名单和BackRAS抑制的内核错误警报（就是匹配到的），和向replayer报告的错误警报（FalseAlarm）。图中显示了每百万条指令中的数量。由于传递给replayer的数量很少，FalseAlarm都看不见，把数字放在bar上面了。
![](https://tva1.sinaimg.cn/large/0082zybpgy1gcb4tq88edj30p80dktb5.jpg)
除了Apache之外所有的benchmark都没有传递任何内核错误警报。Apache传递的错误警报是RAS下溢。他们是由网络驱动程序代码在极端负载下的深度嵌套造成的。白名单和BackRAS都非常有效的抑制了错误警报。

#### 8.3 重放
##### 8.3.1 Checkpointing Replay

![](https://tva1.sinaimg.cn/large/0082zybpgy1gcb50zmo7lj31bo0fin3v.jpg)
图7（a）比较了各种检查点回放设置与记录设置（Rec）的执行时间。重放设置包括不使用检查点（RepNoChk），每隔5s（RepChk5），1s（RepChk1），0.2s（RepChk02）秒设置一个检查点。以REC为基准。

可以看到，RepChk1会比Rec执行时间平均增加59%。结果表明检查点重放的运行速度大概相当于录制的速度（rouphly comparable）。因此检查点重放可以一直运行。虽然CR稍慢，但是它可以很容易赶上记录，因为recoder经常由于多种原因处于等待状态。如果回放严重滞后，可以使用back pressure来暂时降低记录执行速度。

图中还可以看出，增加或减少检查点周期会改变回放速度。然而即使没有检查点，重放时间也比Rec长48%。

为了理解这些影响，图7（b）把RepChk1相对Rec延迟的原因分解成几个来源。包括我们在图5（b）中看见的，以及创建检查点（Chk）。

图中显示，Chk明显增加了总开销。这就是为什么检查点的频率很重要。实际的开销取决于workload的内存写特性，糟糕的内存位置会导致更多的页面复制，增加检查点开销。

有趣的是，中断开销占主导地位。因为中断是异步事件，而rdtsc，pio/mmio和网络是同步的。在重放期间确定应该注入异步中断的指令非常耗时。如section 7.3所示，需要几个指令的单步VMExit。它还解释了RepNoChk仍然比Rec有很大开销。

##### 8.3.2 Alarm Replay


![](https://tva1.sinaimg.cn/large/0082zybpgy1gcb7bgzqxfj30pa0be40h.jpg)现在考虑AR检查内核中ROP攻击的速度。图9比较了AR执行时间和前面提到的环境，RepChk1和Rec。以Rec为基准。AR需要捕获每一个内核调用和返回指令。因此这种模型的减速直接与「执行了多少内核调用和返回指令」相关。alarm replay make和mysql导致比记录多花30-40倍时间。apache需要50x。另一方面，对于radiosity，内核活动最少，需要2.8x。
#### 8.4 响应攻击的时间窗口
检测ROP所需的时间是AR确认ROP的时间与记录警报写入日志时间的差值。这个时间窗口和在两个时间点生成的结果日志的大小取决于多种因素。最重要的两个是workload特性和重放用的机器数量。对于我们的系统，我们测量了时间窗口平均为几秒，日志大小平均几MB（图6（a））。

系统要保留的检查点数量取决于我们希望回滚到多远才能完全理解攻击。严格地说，要重现攻击点的状态，AR只需要从最近的检查点开始执行。在RAepChk1中，最坏的情况也只有1秒钟。如果这就是需要的，那么RnR-Safe只需要保持与上面提到的时间窗口持续时间（秒）+2——加2是为了保证不会过早覆盖正确的检查点。

如果用户希望分析触发攻击前的最后N秒执行情况，以了解攻击的上下文。那么需要保留额外的N个检查点。如果用户希望记录整个历史记录，也可无限期存储检查点。记录的历史可以用于取证或审计先前的执行来检测入侵。


### 9 相关工作



### 10 结论

### 附录A：什么是ROP攻击？














