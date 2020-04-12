---
title: "译｜2005｜ An Introduction to Kprobes"
date: 2020-04-11T15:52:54+08:00
description:
# draft: false
hideToc: false
enableToc: false
enableTocContent: false
tags:
- kprobes
series:
- kprobes
categories:
- Dynamic Trace
---

## 译者序
本文翻译自 2005 年在 [LWN](https://lwn.net/) 发布的，一篇 KProbes 入门级的文章：[An introduction to KProbes](https://lwn.net/Articles/132196/)，当时的内核版本为 2.6.11。文中的配图是用 `Omnigraffle.app` 重新做了一份，顺着作者的思路走一遍。

**注：水平有限，文中难免存在遗漏或者错误的地方。如有疑问，建议直接阅读原文。**
****
<!--
## introduction
KProbes is a debugging mechanism for the Linux kernel which can also be used for monitoring events inside a production system. You can use it to `weed out` performance `bottlenecks`, log specific events, trace problems etc. KProbes was developed by IBM as an underlying mechanism for another higher level tracing tool called DProbes. DProbes adds a number of features, including its own scripting language for the writing of probe handlers. However, only KProbes has been merged into the standard kernel.
-->
## 前言
KProbes 作为 Linux 内核的一种调试机制，也可以用来监控生产系统内部的事件。你可以用它来扫除性能瓶颈、记录特定事件、追踪问题等等。 KProbes 是由 IBM 开发出来的，作为另外一种更高级的追踪工具 Dprobes 的一种底层机制。 Dprobes 添加了很多功能，包括它自己的用来编写探针处理函数的脚本语言。不过最终，只有 KProbes 被合并到标准的内核中。

<!--
In this article I will describe the implementation of KProbes as present in the 2.6.11.7 kernel. KProbes heavily depends on processor architecture specific features and uses `slightly` different mechanisms depending on the architecture on which it's being executed. The following discussion `pertains only to` the x86 architecture. This article `assumes` a certain `familiarity with` the x86 architecture `regarding` interrupts and exceptions handling. KProbes is available on the following architectures however: ppc64, x86_64, sparc64 and i386.
-->
这篇文章将会描述 2.6.11.7 内核內 KProbes 的实现。 KProbes 非常依赖处理器架构的特殊功能，并且根据执行它的架构会使用略微不同的机制。后续的讨论只与 x86 架构相关。本文假设你对 x86 架构中的中断和异常处理有一定的了解。 目前， KProbes 在 ppc64、x86_64、sparc64、i386 架构上是可用的。

<!--
A kernel probe is a set of handlers placed on a certain instruction address. There are two types of probes in the kernel `as of now`, called "KProbes" and "JProbes." A KProbe is defined by a pre-handler and a post-handler. When a KProbe is installed at a particular instruction and that instruction is executed, the pre-handler is executed just before the execution of the probed instruction. Similarly, the post-handler is executed just after the execution of the probed instruction. JProbes are used to get access to a kernel function's arguments at runtime. A JProbe is defined by a JProbe handler with the same prototype as that of the function whose arguments are to be accessed. When the probed function is executed the control is first transferred to the user-defined JProbe handler, followed by the transfer of execution to the original function. The KProbes package `has been designed` in such a way that tools for debugging, tracing and logging could be built by extending it.
-->
kernel probe（内核探针）是一组位于某个指令地址上的处理函数。到目前为止，内核中有两种类型的探针，称作 “KProbes” 和 “JProbes”。 KProbe 由 `pre-handler` 和 `post-handler` 定义。 当 KProbe 被安装到一个特定的指令上，且指令被执行的时候， `pre-handler` 会在这之前执行。同样， `post-handler` 会在这个指令之后执行。 JProbes 用用于在运行时访问内核函数的参数。 JProbe 由 JProbe 处理函数定义，函数原型与要读取的参数的函数相同。当被探测的函数要被执行的时候，控制权会先转移到用户定义的 JProbe 处理函数，之后再将执行权转移到原始函数。 KProbes 软件包是以扩展它自身来构建用于试、追踪、记录的工具而设计的。

{{<img src="/images/KProbes/2A58CF5D-4FB6-4B42-8D43-7CB0EC634691.png" width="350px" height="350px" >}}

<!--
The figure to the right describes the architecture of KProbes. On the x86, KProbes `makes use of` the exception handling mechanisms and modifies the standard breakpoint, debug and a few other exception handlers for its own purpose. Most of the handling of the probes is done in the context of the breakpoint and the debug exception handlers which `make up` the KProbes architecture dependent layer. The KProbes architecture independent layer is the KProbes manager which is used to register and unregister probes. Users provide probe handlers in kernel modules which register probes through the KProbes manager.
-->
此图描述了 KProbes 的结构。在 x86 上， KProbes 利用异常处理机制修改了普通的断点、调试和一些其他的异常处理函数，以便达到自己的目的。探针的逻辑大多都是在断点和调试异常函数的上下文中完成的，它们构成了 KProbes 架构依赖层（Architecture Dependent Layer）。 KProbes Manager 是架构无关层（Architecture Independent Layer），它是用来注册和注销探针的。用户在内核模块中准备的探针处理函数通过 KProbes Manager 来注册。

<!--
## KProbes Interface
The data structures and functions implementing the KProbes interface have been defined in the file <linux/kprobes.h>. The following data structure describes a KProbe.
-->
## KProbes 接口
`<linux/kprobes.h>` 文件中定义了实现 KProbes 接口的数据结构和函数。以下数据结构描述了一个 KProbe 。
```c
struct kprobe {
    struct hlist_node hlist;                    /* Internal */
    kprobe_opcode_t addr;                       /* Address of probe */
    kprobe_pre_handler_t pre_handler;           /* Address of pre-handler */
    kprobe_post_handler_t post_handler;         /* Address of post-handler */
    kprobe_fault_handler_t fault_handler;       /* Address of fault handler */
    kprobe_break_handler_t break_handler;       /* Internal */
    kprobe_opcode_t opcode;                     /* Internal */
    kprobe_opcode_t insn[MAX_INSN_SIZE];        /* Internal */
};
```

<!--
Let's first talk about registering a KProbe. Users can insert their own probe inside a running kernel by writing a kernel module which implements the pre-handler and the post-handler for the probe. `In case` a fault occurs while executing a probe handler function, the user can handle the fault by defining a fault-handler and passing its address in struct kprobe. The prototypes for these are defined as below.
-->
先谈谈注册 KProbe 。用户可通过写一个内核模块把探针插入正在运行的内核内部，内核模块实现了探针的 `pre-handler` 和 `post-handler` 函数。如果在执行探针处理函数期间发生故障，用户可通过定义 `fault-handler` 函数以及传递在 `struct kprobe` 结构中的地址来处理故障。这些处理函数的原型定义如下。
```c
typedef int (*kprobe_pre_handler_t)(struct kprobe*, struct pt_regs*);
typedef void (*kprobe_post_handler_t)(struct kprobe*, struct pt_regs*,
              unsigned long flags);
typedef int (*kprobe_fault_handler_t)(struct kprobe*, struct pt_regs*,
             int trapnr);
```

<!--
`As can be seen` the pre-handler and the post-handler both receive a reference to the probe as well as the registers saved for the context `in which` the probe was hit. These values can be used in the pre-handler or post-handler or if required, they can be modified before returning control to the `subsequent` instruction. This also means that the same handlers can be used for multiple probe locations. The flags parameter is currently unused. The trapnr parameter (for the fault handler function) contains the exception number which occurred while handling the KProbe. A user defined fault handler can return 0 to let KProbe handle the fault further. It returns 1 if it has handled the fault and wants to let the execution of the probe handler continue.
-->
可以看到， `pre-handler` 和 `post-handler` 都能接受探针的引用以及在探针命中时保存的寄存器。这些值是可以在 `pre-handler` 或 `post-handler` 中或需要时使用，还可以在把控制权返回到后续的指令之前修改。也意味着同一个处理函数可用在多个探测位置上。 `flags` 参数目前还未被使用。 `trapnr` 参数（用于故障处理函数）包括在处理 KProbe 期间发生的异常编号。要让 KProbe 进一步处理故障，用户定义的故障回调函数可以返回 `0`。假如故障已经被处理，还想要探针处理函数继续执行可以返回 `1`。

<!--
Note that currently the pre-handler cannot be NULL for a probe, although the use of post-handler is optional. This is `considered` a bug since there may be cases where the pre-handler may not be required but a post-handler is needed. In such situations the user will still have to define a pre-handler. Another bug (which can oops the kernel) is `related to` probes which are activated on the ret/lret instructions. Yet another bug is related to probes activated on int3 instructions. All of these problems should be fixed in the 2.6.12 release of the kernel. However, these bugs can be easily avoided so they do not `present` any serious issues for someone who wants to use KProbes immediately without applying patches.
-->
请注意，虽然 `post-handler` 是可选的，但目前探针的 `pre-handler` 不能为 `NULL` 。因为在有些情况下可能需要 `post-handler`，不需要 `pre-handler` ，所以这点被认为是一个 bug。这种情况，用户还必须定义一个 `pre-handler`。另外一个 bug （能让内核崩溃）跟在 `ret/lret` 指令上激活的探针有关。还有一个 bug 与 `int3` 指令上激活的探针相关。这些的问题都应该在内核的 2.6.12 发行版中被修复了。不管怎样，这些 bugs 可以轻易地避开，而对于那些想要立即使用 KProbes 又没有采用补丁的人而言，不会造成任何严重的问题。

<!--
The KProbe registration functions are defined as shown below.
-->
KProbe 注册函数定义如下：
```c
int register_kprobe(struct kprobe *p);
int unregister_kprobe(struct kprobe *p);
```

<!--
The registration function takes a reference to the KProbe structure describing the probe. Note that the user's module which registers the probe should keep a reference to the structure until the probe is unregistered. Since access to KProbes is `serialized`, a probe can be registered or unregistered anytime except from inside the probe handlers themselves, which will deadlock the system. This is because probe handlers execute after the spinlock used for locking KProbes has been acquired. The same spinlock is locked just before unregistering the probe. So if `an attempt` is made to unregister a probe inside a probe handler the same path will try to lock the spinlock twice.
-->
注册函数接受一个 `KProbe` 结构体的指针。注意，注册探针的内核模块应该一直保持对这个结构体的引用直到探针被注销。由于对 KProbes 的访问已序列化，探针可以随时注册或者注销探针，探针处理函数内部除外，否则会死锁操作系统。因为，探针处理函数是在得到用来锁定 KProbes 的自旋锁之后执行。注销探针完成之前自旋锁是被锁定的，如果试图在探针处理函数内部注销探针，那将会再一次锁定自旋锁。

<!--
Multiple probes cannot be placed on the same address as of now. However, a [patch](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=111321506232570&amp;w=2) has been submitted to the kernel mailing list which allows multiple probes to be registered at the same address through another interface. It might be included in the next release of the kernel. Until then, if such an attempt is made register_kprobe() returns -EEXIST.
-->
目前，不能在相同的地址上放置多个探针。不过，已经有一个[补丁](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=111321506232570&amp;w=2)提交到了内核邮件列表，它通过另外一个接口允许在相同的地址上注册多个探针。内核的下一个发布版本中也许会包含它。在此之前，如果已经尝试过的话， `register_kprobe()` 函数会返回 `-EEXIST`。

<!--
JProbes are used to give access to a function's arguments at runtime. This is `achieved` by providing a JProbe handler with the same prototype as that of the function being probed. At runtime, when the original function is executed, control is transferred to the JProbe handler after copying the process's context. On return from the JProbe handler, the context - consisting of the process's registers and the stack - is restored, so any modifications to the context of the process in the JProbe handler are lost. The execution continues from the point at which the probe was placed with the original saved state. A JProbe is represented by the structure given below.
-->
JProbes 用来在运行时访问一个函数的参数。这是用一个与被探测函数原型相同的 JProbe 处理函数办到的。在运行时，执行原始函数的时候，先复制进程的上下文，再将控制权转移到 JProbe 处理函数。在 JProbe 处理函数返回期间，进程上下文（由寄存器和栈组成）会被恢复，因此在 JProbe 处理函数中对进程上下文所做的任何修改都无效。以先前保存的状态，在放置探针的地方恢复执行。JProbe 由以下结构体表示。
```c
struct jprobe {
    struct kprobe kp;
    kprobe_opcode_t *entry; 	/* user-defined JProbe handler address */
};
```

<!--
The user `places` the address of the function which will handle this probe in the entry field. The addr field in struct kprobe should be `populated` with the address of the function whose arguments are to be accessed. The functions used to register and unregister a JProbe are given below.
-->
用户在 `entry` 字段设置用来处理探针的函数地址。在 kprobe 结构体中的 `addr` 字段应该用被访问的函数地址来填充。下面的函数用来注册或注销一个 JProbe：
```c
int register_jprobe(struct jprobe *p);
void unregister_jprobe(struct jprobe *p);
```

<!--
The JProbe handler which is written by the user should call jprobe_return() when it wants to return instead of the return statement.
-->
用户编写的 JProbe 处理函数，应该在要返回的时候调用 `jprobe_return()` 函数而不是 `return` 语句。

## KProbes Manager
<!--
The KProbes Manager is responsible for registering and unregistering KProbes and JProbes. The file kernel/kprobes.c implements the KProbes manager. Each probe is described by the struct kprobe structure and stored in a hash table hashed by the address at which the probe is placed. Access to this hash table is serialized by the spinlock kprobe_lock. This spinlock is locked before a new probe is registered, an existing probe is unregistered or when a probe is hit. This prevents these operations from executing `simultaneously` on a SMP machine. Whenever a probe is hit, the probe handler is called with interrupts disabled. Interrupts are disabled because handling a probe is a multiple step process which involves breakpoint handling and single-step execution of the probed instruction. There is no easy way to save the state between these operations `hence` interrupts are kept disabled during probe handling.
-->
KProbes Manager 负责注册和注销 KProbes 、 JProbes。 `kernel/kprobes.c` 文件实现 KProbes manager。每个探针是由一个 `struct kprobe` 结构体来表示的，且保存在一个用探针的目标地址来计算的 hash 表中。用 `kprobe_lock` 自旋锁来串行化对哈希表的访问。在注册新的探针、注销已存在的探针之前或者命中探针的时候，自旋锁都是被锁定的。这样会阻止在 SMP 机器上并行的执行这些操作。无论什么时候命中探针，探针处理函数都是在禁用中断的情况下调用的。禁用中断，是因为处理探针是个多步骤过程，涉及断点处理以及被探测指令的单步执行。没有简单的方法来保存这些操作之间的状态，因此在处理探针期间中断一直是禁用的。

<!--
The manager is composed of these functions which are followed by a simplified description of what they do. These functions are architecture independent. A side-by-side reading of the code in kernel/kprobes.c and these steps will `clarify` the whole implementation.
-->
Manager 是由以下这些函数构成，且附带一点对它们的简短描述。这些函数是架构无关的。同步阅读 [`kernel/kprobes.c`](https://elixir.bootlin.com/linux/v2.6.11/source/kernel/kprobes.c) 文件中的代码以及这些内容将会阐明整个实现。

<!--
* void lock_kprobes(void)
	* Locks KProbes and records the CPU on which it was locked
* void unlock_kprobes(void)
	* Resets the recorded CPU and unlocks KProbes
* struct kprobe *get_kprobe(void *addr)
	* Using the address of the probed instruction, returns the probe from hash table
* int register_kprobe(struct kprobe *p)
	* This function registers a probe at a given address. Registration involves copying the instruction at the probe address in a probe specific buffer. On x86 the maximum instruction size is 16 bytes hence 16 bytes are copied at the given address. Then it replaces the instruction at the probed address with the breakpoint instruction.
* void unregister_kprobe(struct kprobe *p)
	* This function unregisters a probe. It restores the original instruction at the address and removes the probe structure from the hash table.
* void unregister_kprobe(struct kprobe *p)
	* This function unregisters a probe. It restores the original instruction at the address and removes the probe structure from the hash table.
* int register_jprobe(struct jprobe *jp)
	* This function registers a JProbe at a function address. JProbes use the KProbes mechanism. In the KProbe pre_handler it stores its own handler setjmp_pre_handler and in the break_handler stores the address of longjmp_break_handler. Then it registers struct kprobe jp->kp by calling register_kprobe()
* void unregister_jprobe(struct jprobe *jp)
	* Unregisters the struct kprobe used by this JProbe
-->
* `void lock_kprobes(void)` ：锁定 KProbes 且记录锁定它的 CPU
* `void unlock_kprobes(void)` ：解锁 KProbes 且重置已记录的 CPU
* `struct kprobe *get_kprobe(void *addr)` ：传入被探测指令的地址，从 hash 表中取回探针
* `int register_kprobe(struct kprobe *p)` ：函数在特定的地址上注册一个探针。注册涉及在探针专用缓冲区中的探针地址处复制指令。在 x86 上，最大的指令大小是 16 个字节，因此这 16 个字节会被复制到特定的地址。然后，用 `breakpoint` 指令替换位于被探测地址处的指令
* `void unregister_kprobe(struct kprobe *p)` ：注销探针。在指定地址恢复原始指令，且从哈希表中移除探针结构体
* `int register_jprobe(struct jprobe *jp)` ：在一个函数的地址上注册一个 JProbe。 JProbes 使用 KProbes 的机制，在 KProbe 的 `pre_handler` 处理函数中， JProbes 保存了它自己的函数 `setjmp_pre_handler`，而且还在 `break_handler` 函数中保存了 `longjmp_break_handler` 函数的地址。然后，调用 `register_kprobe()` 函数注册 kprobe 结构体 `jp->kp`
* `void unregister_jprobe(struct jprobe *jp)` ：注销 JProbe 使用的 kprobe 结构体


<!--
## What happens when a KProbe is hit?
The steps involved in handling a probe are architecture dependent; they are handled by the functions defined in the file arch/i386/kernel/kprobes.c. After the probes are registered, the addresses `at which` they are active contain the breakpoint instruction (int3 on x86). As soon as execution reaches a probed address the int3 instruction is executed, causing the control to reach the breakpoint handler do_int3() in arch/i386/kernel/traps.c. do_int3() is called through an interrupt gate therefore interrupts are disabled when control reaches there. This handler notifies KProbes that a breakpoint `occurred`; KProbes checks if the breakpoint was set by the registration function of KProbes. If no probe is present at the address at which the probe was hit it simply returns 0. Otherwise the registered probe function is called.
-->
## 命中 KProbe 的时候发生了什么？

{{<img src="/images/KProbes/791B014B-E5A8-4D43-B2EB-66C08B701B17.png" width="550px" height="350px" >}}

以上涉及处理探针的步骤都是架构相关的，由 `arch/i386/kernel/kprobes.c` 文件中定义的函数来处理。注册探针后，那些处于激活状态的地址包含了 breakpoint 指令（在 x86 上是 int3）。一旦执行到被探测的地址就会执行 int3 指令，也因此控制权会转到 `arch/i386/kernel/traps.c` 文件中的 `do_int3()` 函数。 `do_int3()` 是通过中断门调用的，所以在控制权转到这里的时候中断是被禁用的。这个函数会通知 KProbes 产生了一个中断， KProbes 会检查中断是不是由 KProbes 的注册函数设置的。如果命中的探测地址上没有探针，只会返回 `0`。相反，它会调用已注册的探针函数。

<!--
## What happens when a JProbe is hit?
A JProbe has to transfer control to another function which has the same prototype as the function on which the probe was placed and then give back control to the original function with the same state as there was before the JProbe was executed. A JProbe `leverages` the mechanism used by a KProbe. Instead of calling a user-defined pre-handler a JProbe specifies its own pre-handler called setjmp_pre_handler() and uses another handler called a break_handler. This is a three-step process.
-->
## 命中 JProbe 的时候发生了什么？
{{<img src="/images/KProbes/490E7DE1-867E-4CBE-B75E-D9476BDA6C2E.png" width="550px" height="350px" >}}

JProbe 必须将控制权转移到另外一个函数，这函数的原型与放置探针的函数相同，然后再将控制权交给原始函数，状态与执行 JProbe 之前相同。JProbe 利用了 KProbe 使用的机制。 JProbe 不是调用用户定义的 `pre-handler` ，而是指定自己的 `pre-handler` ，名为 `setjmp_pre_handler()` ，而且使用了另外一个称为 `break_handler` 的函数。这过程有三个步骤。

<!--
In the first step, when the breakpoint is hit control reaches kprobe_handler() which calls the JProbe pre-handler (setjmp_pre_handler()). This saves the stack contents and the registers before changing the eip to the address of the user-defined function. Then it returns 1 which tells kprobe_handler() to simply return instead of setting up single-stepping as for a KProbe. `On return` control reaches the user-defined function to access the arguments of the original function. When the user defined function is done it calls jprobe_return() instead of doing a normal return.
-->
第一步，在命中断点的时候控制权转到 `kprobe_handler()` 函数，它会调用 JProbe 的 `pre-handler` 函数(`setjmp_pre_handler()`)。在把 `eip` 改成用户定义函数的地址之前，这个函数会把栈和寄存器保存下来。然后，它会返回 `1` 让 `kprobe_handler()` 函数直接返回，而不像 KProbe 那样设置单步执行。在返回时，控制权转到用户定义的函数，这样就可以访问原始函数的参数。在用户定义的函数完事后，该调用 `jprobe_return()` 函数，而不是做普通的 `return`。

<!--
In the second step jprobe_return() truncates the current stack frame and generates a breakpoint which transfers control to kprobe_handler() through do_int3(). kprobe_handler() finds that the generated breakpoint address (address of int3 instruction in jprobe_handler()) does not have a registered probe however KProbes is active on the current CPU. It assumes that the breakpoint must have been generated by JProbes and hence calls the break_handler of the current_kprobe which it saved earlier. The break_handler restores the stack contents and the registers that were saved before transferring control to the user-defined function and returns.
-->
第二步，`jprobe_return()` 函数截断当前栈帧并生成一个断点，通过 `do_int3()` 函数把控制权转移到 `kprobe_handler()` 函数。 `kprobe_handler()` 函数发现生成的断点地址（`jprobe_handler()` 函数中 int3 指令的地址）没有注册探针，但 KProbes 在当前 CPU 上处于活跃状态。它假设断点一定是 JProbes 生成的，因此调用了它先前保存的 current_kprobe `break_hanlder` 函数。 `break_handler` 函数会恢复栈以及**在控制权转移到用户定义的函数和返回之前保存的**寄存器。

<!--
In the third step kprobe_handler() then sets up single-stepping of the instruction at which the JProbe was set and the rest of the sequence is the same as that of a KProbe.
-->
第三步， `kprobe_handler()` 函数在已设置 JProbe 的指令处设置单步执行，剩下的一系列步骤与 KProbe 相同。

## 可能出现的问题
<!--
There could be several possible problems which could occur when a probe is handled by KProbes. The first possibility is that several probes are handled in parallel on a SMP system. However, there's a common hash table shared by all probes which needs to be protected against `corruption` in such a case. In this case kprobe_lock serializes the probe handling `across` processors.
-->
在 KProbes 处理探针的时候，有可能会出现几个问题。第一种，在 SMP 系统上并行处理几个探针。但，所有的探针都共用一个普通的哈希表，那就需要保护它们避免遭到损坏。因此， `kprobe_lock` 会串行化对探针的处理。

<!--
Another problem occurs if a probe is placed inside KProbes code, causing KProbes to enter probe handling code recursively. This problem is taken care of in kprobe_handler() by checking if KProbes is already running on the current CPU. In this case the `recursing` probe is disabled `silently` and control returns back to the previous probe handling code.
-->
如果探针被放置在 KProbes 代码内部会发生另外一种问题，导致 KProbes 递归调用探针处理函数。这个问题已经在 `kprobe_handler()` 函数中处理，它通过检查 KProbes 是否已经在当前 CPU 上运行。这种递归探针会被悄悄的禁掉，并且控制权会返回到先前的探针处理函数。

<!--
If preemption occurs when KProbes is executing it can context switch to another process while a probe is being handled. The other process could cause another probe to fire which will cause control to reach kprobe_handler() again while the previous probe was not handled completely. This may result in `disarming` the new probe when KProbes discovers it's recursing. To avoid this problem, preemption is disabled when probes are handled.
-->
如果正在执行 KProbes 的时候发生抢占，在处理探针期间，上下文可以切换到另外一个进程。在先前的探针完全没有处理完的时候，其它的进程可能会触发另一个探针，控制权将再一次转到 `kprobe_handler()` 函数。当 KProbes 发现新探针正在递归的时候，可能会撤销它。为了避免这个问题，在处理探针的期间抢占是被禁用的。

<!--
Similarly, interrupts are disabled by causing the breakpoint handler and the debug handler to be invoked through interrupt gates `rather than` trap gates. This disables interrupts as soon as control is transferred to the breakpoint or debug handler. These changes are made in the file arch/i386/kernel/traps.c.
-->
同样地，中断被禁用，是因为断点和调试函数是通过中断门调用的，而不是陷阱门。一旦控制权转移到断点或者调试函数就会禁用中断。这些操作是在 `arch/i386/kernel/traps.c` 文件中做的。

<!--
A fault might occur during the handling of a probe. In this case, if the user has defined a fault handler for the probe, control is transferred to the fault handler. If the user-defined fault handler returns 0 the fault is handled by the kernel. Otherwise, it's assumed that the fault was handled by the fault handler and control reaches back to the probe handlers.
-->
在处理探针期间可能会发生故障。如果在用户已经定义了一个故障处理函数的情况下，控制权会被转移到故障处理函数。如果用户定义的故障处理函数返回 `0` ，那么这个错误由内核来处理。此外， KProbes 会假设故障已经被处理，控制权会回到探针处理函数。

<!--
## Conclusion
KProbes is an excellent tool for debugging and tracing; it can also be used for performance measuring. Developers can use it to trace the path of their programs inside the kernel for debugging purposes. System administrators can use it to trace events inside the kernel on production systems. KProbes can also be used for `non-critical` performance measurements. The current KProbes implementation, however, introduces some latency of its own in handling probes. The cause behind this latency is the single kprobe_lock which serializes the execution of probes across all CPUs on a SMP machine. Another reason is the mechanism used by KProbes which uses multiple exceptions to handle a single probe. Exception handling is an expensive operation which causes its own delays. Work needs to be done in this area to improve SMP `scalability` and improving the probe handling time to make KProbes a `viable` performance measuring tool.
-->
## 结论
KProbes 一个极好的调试、追踪工具，也可用来测量性能。开发者可以用它来追踪他们的程序在内核中的路径，以便调试。系统管理员可以用它在生产系统中追踪内核的事件。KProbes 也可以用于非关键性性能测量。不过，目前 KProbes 的实现，在处理探针的过程中引入一些延迟。延迟背后的原因是只有一个 `kprobe_lock`，它在 SMP 机器上串行化了探针的执行。另外一个因素是 KProbes 使用的机制，它使用多个异常去处理一个探针。异常处理是非常昂贵的操作，会导致延迟。需要在这方面展开工作，提升 SMP 的可扩展性，缩短处探针的处理时间，使得 KProbes 成为可行的性能测量工具。

<!--
KProbes however cannot be used directly for these purposes. In the raw form a user can write a kernel module implementing the probe handlers. However higher level tools are necessary for making it more convenient to use. Such tools could contain standard probe handlers implementing the desired features or they could contain a `means` to produce probe-handlers given simple descriptions of them in a scripting language like DProbes.
-->
但是，KProbes 不能直接用来做这些事情。原始的状态下，用户可编写一个实现探针函数的内核模块。不过，为了更方便的使用它，必须使用更高级的工具。这种高级工具可以包含标准的探针函数，用它们来实现所要的功能，或者它们可以包含一种类似 DProbes 的脚本语言，用来生成 probe-handlers。

<!--
## Related Links
* [KProbes](http://www-106.ibm.com/developerworks/library/l-kprobes.html?ca=dgr-lnxw07Kprobe) An introductory article on KProbes with some examples on how to use it.
* [DProbes](http://dprobes.sourceforge.net/) The scriptable tracing tool for Linux which works on top of KProbes.
* [Network Packet Tracing Patch](http://prdownloads.sourceforge.net/dprobes/plog.tar.gz?download) This patch is used to trace the path of network packets traveling through the kernel stack using DProbes.
* [KProbes debugfs patch](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=110624318108570&amp;w=2) This patch lists all probes applied at any addresses through debugfs
* [SysRq key for KProbes Patch](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=110551169610598&amp;w=2) This patch enables the use of SysRq key to be used for listing all applied probes.
* [SystemTap](http://sources.redhat.com/systemtap/) The Linux Kernel Tracing Tool - in the works.
-->
## 相关链接
* [KProbes](http://www-106.ibm.com/developerworks/library/l-kprobes.html?ca=dgr-lnxw07Kprobe) 一篇关于 KProbes 的介绍性文章，以及一些如何使用它的例子（译注：此链接已经失效）
* [DProbes](http://dprobes.sourceforge.net/) Linux 基于 KProbes 的脚本化追踪工具
* [Network Packet Tracing Patch](http://prdownloads.sourceforge.net/dprobes/plog.tar.gz?download) 这个补丁能让 Dprobes 追踪网络数据包经过内核栈的路径
* [KProbes debugfs patch](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=110624318108570&amp;w=2)  这个补丁列出所有探针，它们都可以通过 debugfs 应用在任意地址上（译注：此链接已失效）
* [SysRq key for KProbes Patch](http://marc.theaimsgroup.com/?l=linux-kernel&amp;m=110551169610598&amp;w=2) 这个补丁能够让 SysRq 键列出所有已应用的探针（译注：此链接已失效）
* [SystemTap](http://sources.redhat.com/systemtap/) Linux 内核追踪工具 - 正在开发中

<!--
## Acknowledgements
The author will like to thank his editor Jonathan Corbet, Kalyan T.B. (HP), Siddharth Seth (IIITB) and Bharata B. Rao (HP) for going through this article and giving their feedback, comments, suggestions etc. and helping to improve this article.
-->
## 致谢
作者要感谢他的编辑们 Jonathan Corbet, Kalyan T.B. (HP), Siddharth Seth (IIITB) 和 Bharata B. Rao (HP) 审阅这篇文章以及给出了他们的反馈、意见、建议等等，并帮助改进这片文章。