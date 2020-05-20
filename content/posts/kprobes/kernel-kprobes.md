---
title: "译｜2019｜Kernel Probes (Kprobes)"
date: 2020-05-19T11:03:35+08:00
description:
# draft: false
hideToc: false
enableToc: true
enableTocContent: false
tags:
- kprobe
---

<!--
## 对于文中一些疑惑：**
* trap 怎么理解？陷阱？调试？捕捉？
* notifier_call_chain 机制？
* routine 翻译为函数或者程序
* debug 翻译为调试
* probepoint 翻译为探测点
* 查查以 TOC 作为函数调用的架构，及其原理
* CPU 如何区分二进制中的指令和数据，或者说是可执行文件如何存储指令以及数据的？
* 指令与函数怎么对应？编译原理？
-->

## 译者序
这篇文章翻译自 Linux 内核源码树中的 [kprobes.txt](https://github.com/torvalds/linux/blob/master/Documentation/kprobes.txt) 文件，此文件描述了 Kprobes 的概念、工作原理、限制等内容。因为文件的最后一次提交是在 2019 年，所以文章标题中的年份也就是指这个意思的。

下面的表格是对关键词的一点解释，概念其实就这几个。

关键词    | 解释
--------  | ----
Kprobes   | 指的是内核的探测机制、框架，依赖于硬件的特定功能实现的，比如 int3 指令
kprobe    | 指的是 Kprobes 的对象或结构体，关联探测点和探针
probepoint| **探测点**，一个可用于观察、监视目的的具体位置，比如：一个函数的入口、返回地址
probe     | **探针**，在探测点做具体事情的对象，比如：分析、追踪

**如果用一句话解释 Kprobes 原理就是**：Kprobes 在探测点上注册了一些探针，当 CPU 执行到探测点的时候 Kprobes 会调用所有相关探针的回调函数。CPU 是怎么从执行流转到 Kprobes 的呢？

**注：因为水平有限，文中难免存在遗漏或者错误的地方。如有疑问，建议直接阅读原文。**
***
<!--
Concepts: Kprobes and Return Probes
===================================

Kprobes `enables you to dynamically break into` any kernel `routine` and
collect debugging and performance information `non-disruptively`. You
can trap at almost any kernel code address [1]_, specifying a handler
routine to be `invoked` when the breakpoint is hit.

.. [1] some parts of the kernel code can not be trapped, see

       :ref:`kprobes_blacklist`)
-->
## 概念： Kprobes 和 Return Probes
Kprobes 能够让你动态的介入内核的任意函数，且无中断的收集调试和性能信息。几乎可以调试任意内核代码地址[^1]，指定一个回调函数，在断点命中的时候调用。

[^1]: 有部分内核代码是不能被捕获的，详情见[黑名单](#黑名单)

<!--
There are currently two types of probes: kprobes, and kretprobes
(also called return probes).  A kprobe can be inserted on `virtually`
any `instruction` in the kernel.  A return probe `fires` when a specified
function returns.
-->
目前，探针有两种类型：kprobes ，kretprobes（也被称为 `return` 探针）。基本上， kprobe 可以安插到任意指令上。当一个指定的函数返回时触发 `return` 探针。

<!--
In the typical case, Kprobes-based `instrumentation` is packaged as
a kernel module.  The module’s init function installs (“registers”)
one or more probes, and the exit function unregisters them.  A
`registration` function such as register_kprobe() specifies where
the probe is to be inserted and what handler is to be called when
the probe is hit.
-->
通常，基于 Kprobes 的探测工具被打包成了一个内核模块。模块的初始化函数会安装（注册）一个或多个探针，而模块的卸载函数会注销它们。 `register_kprobe()` 注册函数指明探针要插入到什么位置，探针命中的时候要调用什么样的函数。

<!--
There are also `register_/unregister_*probes()` functions for `batch`
registration/unregistration of a group of `*probes`. These functions
can `speed up` unregistration process when you have to unregister
a lot of probes at once.
-->
也有一些用来批量注销或注册一组探针的 `register_/unregister_*probes()` 函数。在必须一次性注销大量探针的时候，这些函数可以加快注销过程。

<!--
The next four subsections explain how the different types of
probes work and how jump optimization works.  They explain certain
things that you’ll need to know in order to make the best use of
Kprobes — e.g., the difference between a pre_handler and
a post_handler, and how to use the maxactive and nmissed fields of
a kretprobe.  But if you’re in a hurry to start using Kprobes, you
can skip ahead to :ref:`kprobes_archs_supported`.
-->
接下来的四个小节会解释不同类型的探针及跳转优化是如何工作的。这些内容讲了一些必须要知道的事项，以便充分利用 Kprobes ，例如： `pre_handler` 与 `post_handler` 之间的区别，以及如何使用 kretprobes 的 `maxactive` 、 `nmissed` 字段。不过，假如你想马上试试 Kprobes 的话，可以直接跳至[##支持的架构]章节。

<!--
## How Does a Kprobe Work?
When a kprobe is registered, Kprobes `makes a copy of` the probed
`instruction` and replaces the first byte(s) of the probed instruction
with a breakpoint instruction (e.g., int3 on i386 and x86_64).
-->
### Kprobe 如何工作？
Kprobes 在注册了一个 kprobe 后，复制被探测的指令，并且把被探测指令的第一个字节替换为断点指令（例如：在 i386、x86_64 平台上的 `int3`）。

<!--
When a CPU hits the breakpoint instruction, a trap `occurs`, the CPU’s
registers are saved, and control passes to Kprobes via the
notifier_call_chain `mechanism`.  Kprobes executes the “pre_handler”
`associated with` the kprobe, passing the handler the addresses of the
kprobe struct and the saved registers.
-->
在 CPU 命中断点指令时，会发生一个 `trap`，CPU 的寄存器会被保存，而控制通过 notifier_call_chain 机制转移到 Kprobes 。Kprobes 执行与 kprobe 关联的 `pre_handler`，并且向它传递 kprobe 结构体和保存的寄存器地址。

<!--
Next, Kprobes single-steps its copy of the probed instruction.
(It would be simpler to single-step the actual instruction `in place`,
but then Kprobes would have to temporarily remove the breakpoint
instruction.  This would open a small time window when another CPU
could `sail` right past the probepoint.)
-->
接着，Kprobes 单步执行被探测指令的副本。（虽然单步执行原始指令会更简单，但过后 Kprobes 还必须移除断点指令。当另一个 CPU 执行过探测点的时候，将会打开一个小的时间窗口。）

<!--
After the instruction is single-stepped, Kprobes executes the
“post_handler”, if any, that is associated with the kprobe.
Execution then continues with the instruction following the probepoint.
-->
在指令单步执行完之后，Kprobes 会执行与 kprobe 关联的 `post_handler`，如果有的话。然后，继续执行探测点之后的指令。

<!--
## Changing Execution Path
Since kprobes can probe into a running kernel code, it can change the
register set, including instruction pointer. This operation requires
maximum care, such as keeping the stack frame, recovering the execution
path etc. Since it operates on a running kernel and needs deep knowledge
of computer architecture and `concurrent` computing, you can easily shoot
your foot.
-->
### 改变执行路径
kprobes 能够探测一段正在运行的内核代码，因此它能改变寄存器，包括指令指针。类似保存栈帧、恢复执行路径，这类操作需要非常地小心，因为 kprobes 作用在正在运行的内核上面，需要深入的了解计算结构体系和并行计算才行。

<!--
If you change the instruction pointer (and set up other `related`
registers) in pre_handler, you must return !0 `so that` kprobes stops
single stepping and just returns to the given address.
This also means post_handler should not be called anymore.
-->
假如你在 `pre_handler` 回调函数内改变指令指针（以及设置其它相关的寄存器），那你必须返回非零值，好让 kprobes 停止单步执行并立即返回到指定地址。这也表示 `post_handler` 不应该再被调用。

<!--
Note that this operation may be harder on some architectures which use
TOC (Table of Contents) for function call, `since` you have to setup a new
TOC for your function in your module, and recover the old one after
returning from it.
-->
注意，在那些使用 TOC （目录）进行函数调用的架构上，这种操作可能会更困难，因为你必须在你的模块中为你的函数设置新的 TOC，并且在函数返回之后还要恢复先前的 TOC。

<!--
## Return Probes
How Does a Return Probe Work?

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When you call register_kretprobe(), Kprobes establishes a kprobe at
the entry to the function.  When the probed function is called and this
probe is hit, Kprobes saves a copy of the return address, and replaces
the return address with the address of a “`trampoline`.”  The trampoline
is an `arbitrary` piece of code — typically just a nop instruction.
At boot time, Kprobes registers a kprobe at the trampoline.
-->
### Return 探针

#### Return 探针如何工作的？
在你调用 `register_kretprobe()` 函数的时候， Kprobes 会在函数的入口处建立一个 kprobe。在调用被探测函数的时候命中这个探针， Kprobes 会保存 return 地址的一个副本，且用一个 “trampoline”（蹦床）的地址替换 return 地址。trampoline 是一段任意的代码 — 通常只是 nop 指令。在启动的时候， Kprobes 在蹦床注册了一个 kprobe。

<!--
When the probed function executes its return instruction, control
passes to the trampoline and that probe is hit.  Kprobes’ trampoline
handler calls the `user-specified` return handler associated with the
kretprobe, then sets the saved instruction pointer to the saved return
address, and that’s where execution `resumes` upon return from the trap.
生词：user-specified, set ... to, resume, upon（此后的意思，跟在单词后面的句子表达的事情完成之后的意思）
-->
在被探测的函数执行 return 指令时，控制转移到蹦床命中探针。 Kprobes 的蹦床处理函数调用与 kretprobe 关联的用户指定的回调函数，然后把保存的指令指针设置为保存的 return 地址，一旦从 trap 返回，执行会在这里恢复。

<!--
While the probed function is executing, its return address is
stored in an object of type kretprobe_instance.  Before calling
register_kretprobe(), the user sets the maxactive field of the
kretprobe struct to specify how many instances of the specified
function can be probed `simultaneously`.  register_kretprobe()
pre-allocates the `indicated` number of kretprobe_instance objects.
生词：simultaneous, indicated（形容词）
-->
在被探测的函数执行期间，它的返回地址被保存在一个 `kretprobe_instance` 类型的对象中。在调用 `register_kretprobe()` 函数前，用户设置 kretprobe 结构体的 `maxactive` 字段表示可同时探测指定函数的实例数量。 `register_kretprobe()` 函数会预先分配规定数量的 `kretprobe_instance` 对象。

<!--
For example, if the function is non-recursive and is called with a
spinlock held, maxactive = 1 should be enough.  If the function is
non-recursive and can never `relinquish` the CPU (e.g., via a semaphore
or `preemption`), NR_CPUS should be enough.  If maxactive <= 0, it is
set to a default value.  If CONFIG_PREEMPT is enabled, the default
is max(10, 2*NR_CPUS).  Otherwise, the default is NR_CPUS.
生词： with a spinlock held, relinquish, preemption
-->
例如，假设该函数是非递归的，且在自旋锁锁住的情况下被调用，那 `maxactive` 的值为 1 应该足够。如果函数是非递归的，且永远不放弃 CPU （例如：通过信号量或抢占），那 NR_CPUS 应该足够。要是 `maxactive` 的值小于等于零的话，会设置成默认值。如果开启 CONFIG_PREEMPT 选项，默认值为 `max(10, 2*NR_CPUS)`（在两倍的 CPU 数量与 10 之间取较大的值）。其他情况，默认值为 NR_CPUS（CPU 数量）。

<!--
It’s not a `disaster` if you set maxactive too low; you’ll just miss
some probes.  In the kretprobe struct, the nmissed field is set to
zero when the return probe is registered, and is `incremented` every
time the probed function is entered but there is no kretprobe_instance
object available for establishing the return probe.
生词：disaster, incremented
-->
如果你把 `maxactive` 设置的很小的话，也不会有什么问题，只是会漏掉一些探针而已。在 kretprobe 结构体中，return 探针注册后 `nmissed` 字段会设置为 0，之后在每次进入被探测函数且没有可用的 `kretprobe_instance` 对象关联 return 探测的时候累加。

<!--
### Kretprobe entry-handler
Kretprobes also provides an optional user-specified handler which runs
on function entry. This handler is specified by setting the entry_handler
field of the kretprobe struct. Whenever the kprobe `placed by` kretprobe at the
function entry is hit, the user-defined entry_handler, if any, is invoked.
If the entry_handler returns 0 (success) then a `corresponding` return handler
is `guaranteed` to be called upon function return. If the entry_handler
returns a non-zero error then Kprobes leaves the return address as is, and
the kretprobe has no further effect for that particular function instance.
生词：placed by, corresponding（形容词）, guarantee, further（进一步）, effect（效果，影响）
-->
#### Kretprobe 入口回调函数
Kretprobes 还提供一个可选的用户回调函数，它运行于函数入口。这个回调函数通过 kretprobe 结构体 `entry_handler` 字段指定。每当命中放置在函数入口处的 kprobe 时，就会调用用户自定义的 `entry_handler` 函数。如果 `entry_handler` 函数返回零（成功），那么对应的 return 回调函数保证会在函数返回的时候被调用。如果 `entry_handler` 返回一个非零错误， Kprobes 会保留返回地址，而 kretprobe 对特定的函数实例没有影响。

<!--
Multiple entry and return handler `invocations` are matched using the unique
kretprobe_instance object associated with them. Additionally, a user
may also specify per return-instance private data to be part of each
kretprobe_instance object. This is especially useful when sharing private
data between corresponding user entry and return handlers. The size of each
private data object can be specified at kretprobe registration time by
setting the data_size field of the kretprobe struct. This data can be
accessed through the data field of each kretprobe_instance object.
生词：invocation（名词，调用）, corresponding
-->
使用与它们关联的唯一对象 kretprobe_instance，可以匹配许多的 entry 和 return 回调函数调用。另外，用户也可以把每个 return-instace 的私有数据指定为每个 kretprobe_instance 对象的一部分。这在 entry 和 return 回调函数之间共享私有数据时尤其有用。每个私有数据对象的大小可以在 kretprobe 注册时通过 kretprobe 结构体中的 data_size 字段指定。私有数据可以通过每个 kretprobe_instance 对象的 data 字段访问。

<!--
`In case` probed function is entered but there is no kretprobe_instance
object available, then `in addition to` incrementing the nmissed count,
the user entry_handler invocation is also skipped.
-->
如果已经进入被探测函数但没有可用的 kretprobe_instance 对象，那么除了增加 `nmissed` 的计数之外，还会跳过 `entry_handler` 调用。

<!--
.. _kprobes_jump_optimization:
How Does Jump Optimization Work?
---------------------------------
If your kernel is built with CONFIG_OPTPROBES=y (currently this flag
is automatically set ‘y’ on x86/x86-64, non-preemptive kernel) and
the “debug.kprobes_optimization” kernel parameter is set to 1 (see
sysctl(8)), Kprobes tries to reduce probe-hit `overhead` by using a jump
instruction instead of a breakpoint instruction at each probepoint.
-->
### 跳转优化如何工作的？
如果内核使用 CONFIG_OPTPROBES=y （x86/x86-64、非抢占式内核上该标记自动地设置为 y）编译，且内核参数 “debug.kprobes_optimization” 设置为 1 （见 sysctl(8) ），那么 Kprobes 尝试在每个探测点用 jump 指令代替 breakpoint 指令来减少命中探针的开销。

<!--
Init a Kprobe
^^^^^^^^^^^^^

When a probe is registered, before `attempting` this optimization,
Kprobes inserts an `ordinary`, breakpoint-based kprobe at the specified
address. So, even if it’s not possible to optimize this particular
probepoint, there’ll be a probe there.
-->
#### 初始化 Kprobe
在注册了一个探针试图优化之前，Kprobes 会在指定的地址插入一个普通的，基于断点的 kprobe。所以，即便不能优化这个特定的探测点，也会有一个探针在那儿。

<!--
Safety Check
^^^^^^^^^^^^

Before optimizing a probe, Kprobes `performs` the following safety checks:

	* Kprobes verifies that the `region` that will be replaced by the jump
	instruction (the “optimized region”) lies entirely within one function.
	(A jump instruction is multiple bytes, and so may `overlay` multiple
	instructions.)

	* Kprobes analyzes the entire function and verifies that there is no
	jump into the optimized region.  Specifically:
		* the function contains no `indirect` jump;
		* the function contains no instruction that causes an exception (since
		the `fixup` code triggered by the exception could jump back into the
		optimized region — Kprobes checks the exception tables to verify this);
		* there is no near jump to the optimized region (`other than` to the first
		byte).

	* For each instruction in the optimized region, Kprobes verifies that
	the instruction can be executed `out of line`.
-->
#### 安全检查
在进行优化探针之前，Kprobes 会做以下安全检查：

* Kprobes 校验会被 jump 指令替换的区域（”已优化的区域“）完全处于一个函数内部。（jump 指令是多字节指令，因此可能会覆盖多个指令）

* Kprobes 分析整个函数，并且确认不会跳入已优化的区域。特别是：
  * 该函数不包含间接跳转
  * 该函数不包含引起异常的指令（因为被异常触发的固定代码可能会跳回到已优化的区域 — Kprobes 会检查异常表来验证这一点）
  * 该函数附近没有跳转到已优化的区域（除了第一个字节）

* 对于已优化区域中的每一个指令，Kprobes 会验证它们能否离线执行。

<!--
Preparing Detour Buffer
^^^^^^^^^^^^^^^^^^^^^^^

Next, Kprobes prepares a "detour" buffer, which contains the following
instruction sequence:

	- code to push the CPU's registers (emulating a breakpoint trap)
	- a call to the trampoline code which calls user's probe handlers.
	- code to restore registers
	- the instructions from the optimized region
	- a jump back to the original execution path.
-->
#### 准备 detour 缓冲区
> detour 意思是像交通节点（环岛）那样

接着，Kprobes 准备一个“环形”缓冲区，包含以下指令序列：

* 推进 CPU 寄存器的代码（模拟断点 trap）
* 调用蹦床代码，再间接调用用户的探针回调函数
* 恢复寄存器的代码
* 优化区域的指令
* 跳回原始执行路径的指令

<!--
Pre-optimization
^^^^^^^^^^^^^^^^^

After preparing the detour buffer, Kprobes verifies that none of the
following situations exist:

	* The probe has a post_handler.
	* Other instructions in the optimized region are probed.
	* The probe is disabled.
-->
#### 优化前
在准备 detour 缓冲区后， Kprobes 会检查确保不出现以下情况：

* 探针有一个 post_handler 回调函数
* 在优化区域中的其他指令被探测了
* 已禁用的探针

<!--
In any of the above cases, Kprobes won’t start optimizing the probe.
Since these are temporary situations, Kprobes tries to start
optimizing it again if the situation is changed.
-->
在上述任何一种情况下，Kprobes 都不会优化探针。因为这都是临时情况，如果情况有变化，Kprobes 会再次进行优化。

<!--
If the kprobe can be optimized, Kprobes enqueues the kprobe to an
optimizing list, and kicks the kprobe-optimizer workqueue to optimize
it.  If the to-be-optimized probepoint is hit before being optimized,
Kprobes returns control to the original instruction path by setting
the CPU’s instruction pointer to the copied code in the detour buffer
— thus at least avoiding the single-step.
-->
如果 kprobe 可以被优化，Kprobes 会把 kprobe 列入优化队列中，然后启动工作队列 kprobe-optimizer 去优化它。如果被优化的 probepoint 在优化之前命中， Kprobes 通过把 CPU 的指令指针设置为 detour 缓冲区中被复制的代码，将控制权返回到原始指令路径 — 这样做至少避免了单步执行。

<!--
Optimization
^^^^^^^^^^^^^

The Kprobe-optimizer doesn’t insert the jump instruction immediately;
`rather`, it calls synchronize_rcu() for safety first, because it’s
possible for a CPU to be interrupted in the middle of executing the
optimized region [^3].  As you know, synchronize_rcu() can ensure
that all interruptions that were active when synchronize_rcu()
was called are done, but only if CONFIG_PREEMPT=n.  So, this version
of kprobe optimization supports only kernels with CONFIG_PREEMPT=n [^4].

After that, the Kprobe-optimizer calls stop_machine() to replace
the optimized region with a jump instruction to the detour buffer,
using text_poke_smp().
-->
#### 优化
Kprobe-optimizer 并不会立即插入 jump 指令，相反为了安全它会先调用 `synchronize_rcu()` 函数，因为在处理优化区域的过程中 CPU 可能会被中断 [^3]。如你所知， `synchronize_rcu()` 函数可以确保所有活跃的中断在调用 `synchronize_rcu()` 的时候已经完成，但前提是 `CONFIG_PREEMPT=n` 的时候。所以，kprobe 的优化版本只支持 `CONFIG_PREEMPT=n` 的内核 [^4]。

之后， Kprobe-optimizer 调用 `stop_machine()` 函数替换优化区域，用一个跳转到 detour 缓冲区指令，使用 `text_poke_smp()` 函数。

<!--
Unoptimization
^^^^^^^^^^^^^^^^^
When an optimized kprobe is unregistered, disabled, or blocked by
another kprobe, it will be unoptimized.  If this happens before
the optimization is complete, the kprobe is just `dequeued` from the
optimized list.  If the optimization has been done, the jump is
replaced with the original code (except for an int3 breakpoint in
the first byte) by using text_poke_smp().
-->
#### 取消优化
当优化的 kprobe 被其他 kprobe 注销、禁用或阻塞的时候，它将不会被优化。如果这种情况在优化完成之前发生，则只是将 kprobe 从优化队列中移除。如果优化已经完成，会通过调用 `text_poke_smp()` 函数把 jump 指令替换为原始代码（除了第一个字节中的 int3 断点）。

<!--
[^3]: Please imagine that the 2nd instruction is interrupted and then
the optimizer replaces the 2nd instruction with the jump address
while the interrupt handler is running. When the interrupt returns to
original address, there is no valid instruction, and it causes an unexpected result.
-->
[^3]: 请想象第二个指令被中断，然后优化器在中断回调函数正在运行的时候用跳转地址替换它。当中断返回到原始地址时，没有有效指令，这会导致一个意外的结果。

<!--
[^4]: This optimization-safety checking may be replaced with the
stop-machine method that ksplice uses for supporting a CONFIG_PREEMPT=y
kernel.
-->
[^4]: 优化安全检查在 ksplice 用于支持 `CONFIG_PREEMPT=y` 内核上可以用 stop-machine 方法替换。

<!--
NOTE for geeks:
The jump optimization changes the kprobe’s pre_handler behavior.
Without optimization, the pre_handler can change the kernel’s execution
path by changing regs->ip and returning 1.  However, when the probe
is optimized, that `modification` is ignored.  `Thus`, if you want to
`tweak` the kernel’s execution path, you need to `suppress` optimization,
using one of the following techniques:

	* Specify an empty function for the kprobe’s post_handler.
	or
	* Execute ‘sysctl -w debug.kprobes_optimization=n’
-->
geeks 注意：
跳转优化改变了 kprobe 的 `pre_handler` 的行为。优化前，`pre_handler` 通过改变 `regs->ip` 的同时返回 1 可以改变内核的执行路径。然而，在 probe 被优化的时候，修改会被忽略。因此，如果你想微调内核的执行路径，需要使用一个方法去抑制优化：

* 给 kprobe 的 post_handler 指定一个空函数
* 执行 `sysctl -w debug.kprobes_optimization=n` 命令

<!--
.. _kprobes_blacklist:
Blacklist
---------
Kprobes can probe most of the kernel except itself. This means
that there are some functions where kprobes cannot probe. Probing
(trapping) such functions can cause a recursive trap (e.g. double
fault) or the `nested` probe handler may never be called.
Kprobes manages such functions as a blacklist.
If you want to add a function into the blacklist, you just need
to (1) include linux/kprobes.h and (2) use NOKPROBE_SYMBOL() macro
to specify a blacklisted function.
Kprobes checks the given probe address `against` the blacklist and
rejects registering it, if the given address is in the blacklist.
-->
### 黑名单
Kprobes 可以探测除自身以外的大部分内核空间。这表示有一些函数 kprobes 无法探测。探测（trapping）这样的函数会导致递归 trap （比如：双重故障）或者嵌套的 probe 回调函数可能永远不会被调用。如果你想添加一个函数到黑名单中，只需要两个步骤：首先，引入` linux/kprobes.h` 文件；其次，使用 `NOKPROBE_SYMBOL()` 宏指定一个要被列入黑名单的函数。 Kprobes 对照黑名单检查传入的 probe 地址，如果传入的地址在黑名单中会拒绝注册。

<!--
.. kprobes_archs_supported:
## Architectures Supported
Kprobes and return probes are implemented on the following
architectures:
-->
## 支持的架构
Kprobes 和 Kretprobes 已在下面的这些结构体系上实现：
* i386 (Supports jump optimization)（支持跳转优化）
* x86_64 (AMD-64, EM64T) (Supports jump optimization)（支持跳转优化）
* ppc64
* ia64 (Does not support probes on instruction slot1.)（在 slot1 指令上不支持 probes）
* sparc64 (Return probes not yet implemented.)（返回 probes 还没实现）
* arm
* ppc
* mips
* s390
* parisc

<!--
Configuring Kprobes
===================
When configuring the kernel using make menuconfig/xconfig/oldconfig,
ensure that CONFIG_KPROBES is set to “y”. Under “General setup”, look
for “Kprobes”.

So that you can load and unload Kprobes-based instrumentation modules,
make sure “Loadable module support” (CONFIG_MODULES) and “Module
unloading” (CONFIG_MODULE_UNLOAD) are set to “y”.

Also make sure that CONFIG_KALLSYMS and perhaps even CONFIG_KALLSYMS_ALL
are set to “y”, since kallsyms_lookup_name() is used by the in-kernel
kprobe `address` resolution code.
-->
## 配置 Kprobes
在使用 `make menuconfig/xconfig/oldconfig` 配置内核时，确保 `CONFIG_KPROBES ` 设置为 “y”。在 “General setup” 字符后搜索 “Kprobes”。

为了可以加载和卸载基于 Kprobes 的探测模块，请确保将“支持模块加载”（CONFIG_MODULES）和“模块卸载”（CONFIG_MODULE_UNLOAD）设置为 “y”。

还要确保将  `CONFIG_KALLSYMS ` 甚至是 `CONFIG_KALLSYMS_ALL` 都设置为 “y”，因为 `kallsyms_lookup_name()` 函数被内核里的 kprobe 地址解析代码使用。

<!--
If you need to insert a probe in the middle of a function, you may find
it useful to “Compile the kernel with debug info” (CONFIG_DEBUG_INFO),
so you can use “objdump -d -l vmlinux” to see the source-to-object
code mapping.
疑惑：you may find it useful to ...（it 代词，指后面的从句）
-->
如果你需要在函数中间插入 probe，也许发现“使用 debug info 编译内核” (`CONFIG_DEBUG_INFO`) 非常有用，因此可以使用 `objdump -d -l vmlinux` 命令去查看源码到目标代码的映射关系。

<!--
API Reference
=============
The Kprobes API includes a “register” function and an “unregister”
function for each type of probe. The API also includes “register_*probes”
and “unregister_*probes” functions for (un)registering arrays of probes.
Here are `terse`, mini-man-page specifications for these functions and
the associated probe handlers that you’ll write. See the files in the
samples/kprobes/ sub-directory for examples.
-->
## API 参考
Kprobes API 为每种探针类型提供了一个”注册“和“注销”函数。还包括批量注册、注销探针的 `register_*probes` 和 `unregister_*probes` 函数。这有些迷你手册以及将会用到的相关的探针回调函数的简短说明。相关例子，可查看在 `samples/kprobes/` 子目录内的文件。

### register_kprobe
```c
#include <linux/kprobes.h>
int register_kprobe(struct kprobe *kp);
```
<!--
Sets a breakpoint at the address kp->addr.  When the breakpoint is
hit, Kprobes calls kp->pre_handler.  After the probed instruction
is single-stepped, Kprobe calls kp->post_handler.  If a fault
occurs during execution of kp->pre_handler or kp->post_handler,
or during single-stepping of the probed instruction, Kprobes calls
kp->fault_handler.  Any or all handlers can be NULL. If kp->flags
is set KPROBE_FLAG_DISABLED, that kp will be registered but disabled,
so, its handlers aren’t hit until calling enable_kprobe(kp).
-->
在 `kp->addr` 地址设置一个断点。命中断点时，Kprobes 调用 `kp->pre_handler`。在探测的指令单步执行后，Kprobe 调用 `kp->post_handler`。如果一个错误发生，在执行 `kp->pre_handler` 或 `kp->post_handler` 期间，又或者是在单步执行被探测指令期间，Kprobes 会调用 `kp->fault_handler`。所有回调函数都可以是 `NULL`。如果 `kp->flags` 设置为  `KPROBE_FLAG_DISABLED` ，`kp` 将会被注册且处于禁用状态。所以 `kp` 的回调函数在调用 `enable_kprobe(kp)` 之前不会被调用。

<!--
.. note
	1. With the introduction of the “symbol_name” field to struct kprobe, the probepoint address resolution will now be taken care of by the kernel. The following will now work:

	kp.symbol_name = "symbol_name";

	(64-bit powerpc `intricacies` such as function descriptors are handled `transparently`)
	2. Use the “offset” field of struct kprobe if the offset into the symbol
	to install a probepoint is known. This field is used to calculate the
	probepoint.
	3. Specify either the kprobe "symbol_name" OR the "addr". If both are
	specified, kprobe registration will fail with -EINVAL.
	4. With CISC architectures (such as i386 and x86_64), the kprobes code
	does not validate if the kprobe.addr is at an instruction `boundary`.
	Use “offset” with `caution`.

register_kprobe() returns 0 on success, or a negative errno otherwise.
-->
注意：
1. 通过引入 `symbol_name` 字段来构造 kprobe，探测点地址解析将会由内核来处理。可以使用以下内容：

	`kp.symbol_name = "symbol_name";`

	（64 位 powepc 错综复杂，例如透明地处理函数描述符）
2. 如果在符号中用于安装探测点的偏移量是已知的，请使用 kprobes 结构体的 `offset` 字段。这个字段用于计算探测点。
3. kprobe 的 `symbol_name` 或者 `addr` 字段都被指定，kprobe 注册会失败且返回 `EINVAL`。
4. 使用 CISC 架构（如：i386，x86_64），kprobes 代码不会验证，如果 `kprobe.addr` 在指令边界。谨慎使用 `offset`。

`register_kprobe()` 函数成功返回 0，其他情况返回一个负的 `errno`。

用户的 pre-handler（`kp->pre_handler`）函数原型:
```c
#include <linux/kprobes.h>
#include <linux/ptrace.h>
int pre_handler(struct kprobe *p, struct pt_regs *regs);
```
<!--
Called with p pointing to the kprobe associated with the breakpoint,
and regs pointing to the struct containing the registers saved when
the breakpoint was hit.  Return 0 here unless you’re a Kprobes geek.
-->
用指向与断点关联的 kprobe 指针 `p` 以及命中断点时保存的寄存器指针 `regs` 调用。

<!-- User’s post-handler (kp->post_handler): -->
用户的 post-handler （`kp->post_handler`）函数原型：
```c
#include <linux/kprobes.h>
#include <linux/ptrace.h>
void post_handler(struct kprobe *p, struct pt_regs *regs,
		  unsigned long flags);
```
<!--
p and regs are as described for the pre_handler.  flags always seems to be zero.
-->
`p` 和 `regs` 同  `pre_handler` 所述。`flags` 看起来一直是 0。

<!--
User’s fault-handler (kp->fault_handler):
-->
用户的 fault-handler （`kp->fault_handler`）函数原型：
```c
#include <linux/kprobes.h>
#include <linux/ptrace.h>
int fault_handler(struct kprobe *p, struct pt_regs *regs, int trapnr);
```
<!--
p and regs are as described for the pre_handler.  trapnr is the
architecture-specific trap number associated with the fault (e.g.,
on i386, 13 for a general protection fault or 14 for a page fault).
Returns 1 if it successfully handled the exception.
-->
`p` 和 `regs` 同 `pre_handler` 所述 。 `trapnr` 是故障相关的特定架构下的 trap 号（例如：在 i386 上， 13 为普通防护故障，14 为页面故障）。如果成功的处理了异常返回 1。

### register_kretprobe
```c
#include <linux/kprobes.h>
int register_kretprobe(struct kretprobe *rp);
```
<!--
Establishes a return probe for the function whose address is
rp->kp.addr.  When that function returns, Kprobes calls rp->handler.
You must set rp->maxactive `appropriately` before you call
register_kretprobe(); see “How Does a Return Probe Work?” for details.

register_kretprobe() returns 0 on success, or a negative errno otherwise.
-->
为 `rp->kp.addr` 地址的函数建立一个 return 探针。在函数返回时，kprobes 调用 `rp->handler` 。在调用 `register_kretprobe()` 之前必须设置合适的 `rp->maxactive`，细节参考 “Return Probe 如何工作？” 。

`register_kretprobe()` 成功返回 0，其他情况返回一个负的 `errno`。

<!--
User’s return-probe handler (rp->handler):
-->
用户的 return 探针回调函数（`rp->handler`）原型：
```c
#include <linux/kprobes.h>
#include <linux/ptrace.h>
int kretprobe_handler(struct kretprobe_instance *ri,
		      struct pt_regs *regs);
```
<!--
regs is as described for kprobe.pre_handler.  ri points to the
kretprobe_instance object, of which the following fields may be of interest:

	* ret_addr: the return address
	* rp: points to the corresponding kretprobe object
	* task: points to the corresponding task struct
	* data: points to per return-instance private data; see “Kretprobe
	entry-handler” for details.
-->
`regs` 同 kprobe.pre_handler 描述那样。`ri` 指向 `kretprobe_instance` 对象，其中可能涉及以下字段：
 * ret_addr：返回地址
 * rp：指向相关的 kretprobe 对象
 * task：指向相关的 task 结构体
 * data：指向每个 return-instace 私有数据，细节参考 “kretprobe entry-handler”。

<!--
The regs_return_value(regs) macro provides a simple abstraction to
`extract` the return value from the `appropriate` register as defined by
the architecture’s ABI.

The handler’s return value is currently ignored.
-->
`regs_return_value(regs)` 宏提供一个简单的抽象方法，从架构的 ABI 定义的合适的寄存器中提取返回值。

目前，回调函数的返回值是被忽略的。

### unregister_*probe
```c
	#include <linux/kprobes.h>
	void unregister_kprobe(struct kprobe *kp);
	void unregister_kretprobe(struct kretprobe *rp);
```
<!--
Removes the specified probe.  The unregister function can be called
at any time after the probe has been registered.

.. note

	If the functions find an incorrect probe (ex. an unregistered probe),
	they clear the addr field of the probe.
-->
移除探针。注销函数可以在探针被注册后调用。

注意：
	如果这些发现一个不正确的探针（不包括未注册的探针），它们会清除探针的 `addr` 字段。

### register_*probes
```c
#include <linux/kprobes.h>
int register_kprobes(struct kprobe **kps, int num);
int register_kretprobes(struct kretprobe **rps, int num);
```
<!--
Registers each of the num probes in the specified array.  If any
error occurs during registration, all probes in the array, up to
the bad probe, are safely unregistered before the register_*probes
function returns.

	* kps/rps: an array of pointers to `*probe` data structures
	* num: the number of the array entries.

.. note

	You have to allocate(or define) an array of pointers and set all
	of the array entries before using these functions.
-->
注册数组中 `num` 个探针。如果在注册期间发生错误，在 `register_*probes` 函数返回之前会安全地注销数组中已注册的探针，直到发生错误的探针为止。

* `kps/rps`：指向 `*probe` 数据结构的指针数组
* `num`：数组的大小

注意：
	必须分配（或定义）指针数组，且在使用这些函数之前设置数组的所有元素。

### unregister_*probes
```c
#include <linux/kprobes.h>
void unregister_kprobes(struct kprobe **kps, int num);
void unregister_kretprobes(struct kretprobe **rps, int num);
```
<!--
Removes each of the num probes in the specified array at once.

.. note

	If the functions find some incorrect probes (ex. unregistered
	probes) in the specified array, they clear the addr field of those
	incorrect probes. However, other probes in the array are
	unregistered correctly.
-->
一次性移除指定数组中 `num` 个探针。

注意：
	如果这些函数在数组中发现一些不正确的探针（比如：未注册的探针），会清除那些不正确探针的 `addr` 字段。数组中其他的探针会被注销掉。

### disable_*probe
```c
#include <linux/kprobes.h>
int disable_kprobe(struct kprobe *kp);
int disable_kretprobe(struct kretprobe *rp);
```
<!--
Temporarily disables the specified `*probe`. You can enable it again by using enable_*probe(). You must specify the probe which has been registered.
-->
临时地禁用某个探针。调用 `enable_*probe()` 函数可再次启用。必须是已经注册的探针。

### enable_*probe
```c
#include <linux/kprobes.h>
int enable_kprobe(struct kprobe *kp);
int enable_kretprobe(struct kretprobe *rp);
```
Enables `*probe` which has been disabled by disable_*probe(). You must specify the probe which has been registered.
通过 `disable_*probe()` 启用已经被禁用的 `*probe`。必须指定已经注册的 probe。

<!--
Kprobes Features and Limitations
================================
Kprobes allows multiple probes at the same address. Also,
a probepoint for which there is a post_handler cannot be optimized.
So if you install a kprobe with a post_handler, at an optimized
probepoint, the probepoint will be unoptimized automatically.
-->
## Kprobes 特性与限制
kprobes 允许在同一个地址插入多个探针。此外，带有 `post_handler` 的探测点无法被优化。所以，如果在已优化的探测点插入带有 `post_handler` 回调函数的 kprobe 探针，探测点会自动地变成未优化的。

<!--
In general, you can install a probe anywhere in the kernel.
In particular, you can probe interrupt handlers.  Known exceptions
are discussed in this section.
-->
通常，可以在内核的任意位置插入探针。特别的是，它可以探测中断处理函数。本节讨论了已知的异常。

<!--
The register_*probe functions will return -EINVAL if you attempt
to install a probe in the code that implements Kprobes (mostly
kernel/kprobes.c and `arch/*/kernel/kprobes.c`, but also functions such
as do_page_fault and notifier_call_chain).
-->
如果试图在实现 Kprobes 的代码中插入一个探针，`register_*probe` 函数将返回 `-EINVAL`。（在 `kernel/kprobes.c` 和 `arch/*/kernel/kprobes.c` 文件中，还有像 `do_page_fault` 和 `notifier_call_chain` 这类的函数）。

<!--
If you install a probe in an inline-able function, Kprobes makes
no attempt to `chase down` all inline instances of the function and
install probes there.  gcc may inline a function without being asked,
so keep this in mind if you’re not seeing the probe hits you expect.
-->
如果在可内联的函数中插入探针，Kprobes 并不会给所有内联实例插入探针。如果没有命中期望的探针，记住一点， gcc 可能会自动内联一个函数。

<!--
A probe handler can modify the environment of the probed function
-- e.g., by modifying kernel data structures, or by modifying the
contents of the pt_regs struct (which are restored to the registers
upon return from the breakpoint).  So Kprobes can be used, for example,
to install a bug fix or to inject faults for testing.  Kprobes, of
course, has no way to `distinguish` the `deliberately` injected faults
from the accidental ones.  Don't drink and probe.
-->
探针回调函数可以修改被检测函数的环境 -- 例如，改变内核数据结构或者 `pt_regs` 数据结构的内容（从断点返回时恢复到寄存器中）。因此，Kprobes 可用于安装 bug 修复或测试时注入错误。当然， Kprobes 是没有办法把故意地注入的错误与意外的错误区分开。不要喝大了搞事情。

<!--
Kprobes makes no attempt to prevent probe handlers from stepping on
each other — e.g., probing printk() and then calling printk() from a
probe handler.  If a probe handler hits a probe, that second probe’s
handlers won’t be run in that instance, and the kprobe.nmissed member
of the second probe will be incremented.
-->
Kprobes 不会阻止探针回调函数之间的相互作用 -- 比如，先给 `printk()` 函数插入探针，接着又从另一个探针回调函数中调用 `printk()` 函数。如果探针回调函数命中一个探针，那么这第二个探针的回调函数不会执行，将只会累加探针的 `kprobe.nmissed` 值。

<!--
As of Linux v2.6.15-rc1, multiple handlers (or multiple instances of
the same handler) may run concurrently on different CPUs.
-->
自 Linux v2.6.15-rc1 开始，多个回调函数（或者相同回调函数的多个实例）可以同时在不同的 CPU 上运行。

<!--
Kprobes does not use `mutexes` or allocate memory except during
registration and unregistration.
-->
除了注册和注销探针之外，Kprobes 不会用互斥锁或分配内存。

<!--
Probe handlers are run with preemption disabled or interrupt disabled,
which depends on the architecture and optimization state.  (e.g.,
kretprobe handlers and optimized kprobe handlers run without interrupt
disabled on x86/x86-64).  In any case, your handler should not yield
the CPU (e.g., by attempting to `acquire` a `semaphore`, or waiting I/O).
-->
探针回调函数在禁用抢占或者禁用中断的情况下运行，这取决于架构以及优化状态（例如，kretprobe 和优化的 kprobe 回调函数在 x86/x86-64 上运行时没有禁用中断）。不管如何，你的回调函数都不应该让出 CPU （比如，试图获取信号量或等待 I/O）。

<!--
Since a return probe is implemented by replacing the return
address with the trampoline's address, stack `backtraces` and calls
to __builtin_return_address() will typically yield the trampoline’s
address instead of the real return address for kretprobed functions.
(As far as we can tell, __builtin_return_address() is used only for instrumentation
and error reporting.)
-->
因为 return 探针是通过把蹦床的地址替换为返回地址来实现的，所以堆栈回溯以及调用  `__builtin_return_address()` 函数得到的是蹦床的地址，而不是 kretprobed 函数实际 return 地址（就目前我们知道的而言，`__builtin_return_address()` 函数只用于测试工具和报告错误）。

<!--
If the number of times a function is called does not match the number
of times it returns, registering a return probe on that function may
produce `undesirable` results. In such a case, a line:
kretprobe BUG!: Processing kretprobe d000000000041aa8 @ c00000000004f48c
gets printed. With this information, one will be able to `correlate` the
exact instance of the kretprobe that caused the problem. We have the
do_exit() case covered. do_execve() and do_fork() are not an issue.
We’re `unaware of` other specific cases where this could be a problem.
-->
如果一个函数的调用次数不能匹配返回的次数，在那个函数上注册的探针可能产生不想要的结果。这种情况，会输出一行 `kretprobe BUG!: Processing kretprobe d000000000041aa8 @ c00000000004f48c`。有了这行信息，就可以关联导致问题的 kretprobe 实例。涵盖了 `do_exit()` 函数的情况。 `do_execve()` 和 `do_fork()` 函数都不是问题。我们不知道的其他特定的情况，可能会出现问题。

<!--
If, upon entry to or exit from a function, the CPU is running on
a stack other than that of the current task, registering a return
probe on that function may produce undesirable results.  For this
reason, Kprobes doesn’t support return probes (or kprobes)
on the x86_64 version of __switch_to(); the registration functions
return -EINVAL.
-->
如果在进入或者退出某个函数时，CPU 在除当前 task 以外的堆栈上运行，那在这个函数上注册 return 探针可能会产生不想要的结果。因为这个原因，Kprobes 不支持 `__switch_to()` 函数 x86_64 版本的 return 探针（或 kprobes），注册函数会返回 `-EINVAL` 。

<!--
On x86/x86-64, since the Jump Optimization of Kprobes modifies
instructions widely, there are some limitations to optimization. To
explain it, we introduce some `terminology`. Imagine a 3-instruction
sequence consisting of a two 2-byte instructions and one 3-byte
instruction.
-->
在 x86/x86-64 架构上，由于 Kprobes 跳转优化修改指令普遍存在，会对优化有一些限制。为解释这一点，我们引入些术语。想象一下，一个由 2 字节指令和 3 字节指令组成的 3 个指令序列。

```c
	    IA
	    |
[-2][-1][0][1][2][3][4][5][6][7]
	    [ins1][ins2][  ins3 ]
	    [<-     DCR       ->]
	    [<- JTPR ->]
```

```c
ins1: 1st Instruction
ins2: 2nd Instruction
ins3: 3rd Instruction
IA:  Insertion Address
JTPR: Jump Target Prohibition Region
DCR: Detoured Code Region
```
<!--
The instructions in DCR are copied to the out-of-line buffer
of the kprobe, because the bytes in DCR are replaced by
a 5-byte jump instruction. So there are several limitations.

	a) The instructions in DCR must be relocatable.
	b) The instructions in DCR must not include a call instruction.
	c) JTPR must not be targeted by any jump or call instruction.
	d) DCR must not `straddle` the border between functions.

Anyway, these limitations are checked by the in-kernel instruction
decoder, so you don’t need to worry about that.
-->
DCR 内的指令被复制到 kprobe 的离线缓冲区中，因为 DCR 内的字节被 5 字节 jump 指令替代了。所以，这儿会有几个限制。

* DCR 内的指令一定是可重定位的
* DCR 内的指令一定不能包含 `call` 指令
* JTPR 不能作为 `jump` 或 `call` 指令的目标
* DCR 不能跨越函数之间的边界

不过，这些限制由内核的指令解码器检查，所以不需要关心这些限制。

<!--
Probe Overhead
==============
On a typical CPU in use in 2005, a kprobe hit takes 0.5 to 1.0
microseconds to process.  Specifically, a benchmark that hits the same
probepoint repeatedly, firing a simple handler each time, reports 1-2
million hits per second, depending on the architecture.  A return-probe
hit typically takes 50-75% longer than a kprobe hit.
When you have a return probe set on a function, adding a kprobe at
the entry to that function adds `essentially` no overhead.

Here are sample overhead `figures` (in usec) for different architectures:
-->
## 探针的开销
在 2005 年常见的 CPU 上，处理命中 kprobe 要花费 0.5 - 1.0 微秒。具体一点，基准测试反复命中同一个探测点，每一次触发简单的回调函数，每秒 1-2 百万次命中，具体数值取决于 CPU 架构。通常，命中 return 探针比命中 kprobe 多花费 50-75% 的时间。当你把一个 kretprobe 插入到一个函数的时候，实际是在函数入口处添加一个 kprobe，基本上上函数不会增加开销。

下面有些不同架构开销的样本（微秒）：

	k = kprobe; r = return probe; kr = kprobe + return probe
	on same function

	i386: Intel Pentium M, 1495 MHz, 2957.31 bogomips
	k = 0.57 usec; r = 0.92; kr = 0.99

	x86_64: AMD Opteron 246, 1994 MHz, 3971.48 bogomips
	k = 0.49 usec; r = 0.80; kr = 0.82

	ppc64: POWER5 (gr), 1656 MHz (SMT disabled, 1 virtual CPU per physical CPU)
	k = 0.77 usec; r = 1.26; kr = 1.45

<!--
Optimized Probe Overhead
========================
Typically, an optimized kprobe hit takes 0.07 to 0.1 microseconds to
process. Here are sample overhead figures (in usec) for x86 architectures:
-->
## 已优化探针开销
通常，命中已优化的 kprobe 要花费 0.07 - 0.1 微妙来处理。这是 x86 架构开销的样本（微妙）：

	k = unoptimized kprobe, b = boosted (single-step skipped), o = optimized kprobe,
	r = unoptimized kretprobe, rb = boosted kretprobe, ro = optimized kretprobe.

	i386: Intel(R) Xeon(R) E5410, 2.33GHz, 4656.90 bogomips
	k = 0.80 usec; b = 0.33; o = 0.05; r = 1.10; rb = 0.61; ro = 0.33

	x86-64: Intel(R) Xeon(R) E5410, 2.33GHz, 4656.90 bogomips
	k = 0.99 usec; b = 0.43; o = 0.06; r = 1.24; rb = 0.68; ro = 0.30

## TODO
<!--
a. SystemTap (http://sourceware.org/systemtap): Provides a simplified
programming interface for probe-based instrumentation.  Try it out.
b. Kernel return probes for sparc64.
c. Support for other architectures.
d. User-space probes.
e. Watchpoint probes (which fire on data references).
-->

	1. [SystemTap](http://sourceware.org/systemtap)：给基于探针的探测工具提供了一个简单的编程接口。可以试一下
	2. sparc64 架构的 kretprobe
	3. 支持其他架构
	4. 用户空间的探针
	5. 观察点探针（在数据引用时触发）

## Kprobes 例子
见 [samples/kprobes/kprobe_example.c](https://github.com/torvalds/linux/blob/master/samples/kprobes/kprobe_example.c) 文件

## Kretprobes 例子
见 [samples/kprobes/kretprobe_example.c](https://github.com/torvalds/linux/blob/master/samples/kprobes/kretprobe_example.c) 文件

<!--
For additional information on Kprobes, `refer to` the following URLs:
-->
有关 Kprobes 的其他信息，请参考以下 URL 链接：

* http://www-106.ibm.com/developerworks/library/l-kprobes.html?ca=dgr-lnxw42Kprobe
* http://www.redhat.com/magazine/005mar05/features/kprobes/
* http://www-users.cs.umn.edu/~boutcher/kprobes/
* http://www.linuxsymposium.org/2006/linuxsymposium_procv2.pdf (pages 101-115)

<!--
Deprecated Features
===================
Jprobes is now a `deprecated` feature. People who are depending on it should
migrate to other tracing features or use older kernels. Please consider to
migrate your tool to one of the following options:

* Use trace-event to trace target function with arguments.

  trace-event is a low-overhead (and almost no visible overhead if it
  is off) statically defined event interface. You can define new events
  and trace it via ftrace or any other tracing tools.

  See the following urls:

    - https://lwn.net/Articles/379903/
    - https://lwn.net/Articles/381064/
    - https://lwn.net/Articles/383362/

* Use ftrace dynamic events (kprobe event) with perf-probe.

  If you build your kernel with debug info (CONFIG_DEBUG_INFO=y), you can
  find which register/stack is assigned to which local variable or arguments
  by using perf-probe and set up new event to trace it.

  See following documents:

	* Documentation/trace/kprobetrace.rst
	* Documentation/trace/events.rst
	* tools/perf/Documentation/perf-probe.txt
-->
## 已弃用的机制
现在 Jprobes 是一个不被推荐的机制。依赖它的应该迁移到其他追踪机制或使用旧的内核。请考虑把你的工具迁移到以下工具中：

* 使用 trace-event 追踪带参数的函数
> trace-event 是个低开销的静态定义的事件接口（如果关闭，没有明显的开销）。你可以定义新事件，通过 ftrace 或者其他追踪工具追踪它。
	参考以下 URL 链接：
    - https://lwn.net/Articles/379903/
    - https://lwn.net/Articles/381064/
    - https://lwn.net/Articles/383362/

* 使用 ftrace 动态事件（kprobe 事件）和 perf-probe
> 如果你使用调试信息（`CONFIG_DEBUG_INFO=y`）编译你的内核，可以用 perf-probe 设置新事件去追踪它，能发现寄存器/栈被分配给了哪个本地变量或者参数。
	参考以下文档：
	- [Documentation/trace/kprobetrace.rst](https://github.com/torvalds/linux/blob/master/Documentation/trace/kprobetrace.rst)
	- [Documentation/trace/events.rst](https://github.com/torvalds/linux/blob/master/Documentation/trace/events.rst)
	- [tools/perf/Documentation/perf-probe.txt](https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/perf-probe.txt)

<!--
The kprobes debugfs interface
=============================
With recent kernels (> 2.6.20) the list of registered kprobes is visible
under the /sys/kernel/debug/kprobes/ directory (assuming debugfs is mounted at //sys/kernel/debug).

/sys/kernel/debug/kprobes/list: Lists all registered probes on the system:
-->
## kprobes debugfs 接口
最新的内核（> 2.6.20），已经注册的 kprobes 列表位于 `/sys/kernel/debug/kprobes/` 目录之下（假设 debugfs 被挂载到 `/sys/kernel/debug` 目录）。

`/sys/kernel/debug/kprobes/list`：列出在系统上所有已注册的 probes：
```c
c015d71a  k  vfs_read+0x0
c03dedc5  r  tcp_v4_rcv+0x0
```
<!--
The first column provides the kernel address where the probe is inserted.
The second column identifies the type of probe (k - kprobe and r - kretprobe)
while the third column specifies the symbol+offset of the probe.
If the probed function belongs to a module, the module name is also
specified. Following columns show probe status. If the probe is on
a virtual address that is no longer valid (module init sections, module
virtual addresses that correspond to modules that’ve been unloaded),
such probes are marked with [GONE]. If the probe is temporarily disabled,
such probes are marked with [DISABLED]. If the probe is optimized, it is
marked with [OPTIMIZED]. If the probe is ftrace-based, it is marked with
[FTRACE].
-->
第一列，是已插入探针的内核地址。第二列，是表示探针的类型（k - kprobe，r - kretprobe）。第三列，是指定探针的符号+偏移量（symbol+offset）。如果被探测的函数属于一个模块，那么这个模块的名字也会被列出来。随后的几列显示探针的状态。如果探针在虚拟地址上，并且地址无效（模块初始化部分，模块虚拟地址，对应的模块已经卸载），这类探针会被标记为 [GONE]。如果探针临时被禁用，会被标记为 [DISABLED]。如果探针被优化了，会被标记为 [OPTIMIZED]。如果探针是基于 ftrace 的，会被标记为 [FTRACE]。

<!--
/sys/kernel/debug/kprobes/enabled: Turn kprobes ON/OFF forcibly.
-->
`/sys/kernel/debug/kprobes/enabled`：强制开启/关闭 kprobes。

<!--
Provides a knob to globally and forcibly turn registered kprobes ON or OFF.
By default, all kprobes are enabled. By echoing “0” to this file, all
registered probes will be `disarmed`, till such time a “1” is echoed to this
file. Note that this knob just disarms and arms all kprobes and doesn’t
change each probe’s disabling state. This means that disabled kprobes (marked
[DISABLED]) will be not enabled if you turn ON all kprobes by this knob.
-->
提供一个全局按钮，强制的开启或关闭已注册的 kprobes。默认情况下，所有 kprobes 是开启的。输出 “0” 到这个文件，所有已注册的探针会被卸载，输出 “1” 到这个文件，又重新加载。请注意，这个按钮只是卸载和加载所有 kprobes，并不会改变每个探针的禁用状态。意思是，已经禁用的探针（标记为 [DISABLED]）是不会被激活的。

<!--
The kprobes sysctl interface
============================
/proc/sys/debug/kprobes-optimization: Turn kprobes optimization ON/OFF.

When CONFIG_OPTPROBES=y, this sysctl interface `appears` and it provides
a knob to globally and forcibly turn jump optimization (see section
:ref:`kprobes_jump_optimization`) ON or OFF. By default, jump optimization
is allowed (ON). If you echo “0” to this file or set
“debug.kprobes_optimization” to 0 via sysctl, all optimized probes will be
unoptimized, and any new probes registered after that will not be optimized.

Note that this knob changes the optimized state. This means that optimized
probes (marked [OPTIMIZED]) will be unoptimized ([OPTIMIZED] tag will be
removed). If the knob is turned on, they will be optimized again.
-->
## kprobes sysctl 接口
`/proc/sys/debug/kprobes-optimization`： kprobes 优化开关。

在 `CONFIG_OPTPROBES=y` 的时候， `sysctl` 接口提供一个全局按钮，强制的开启或关闭跳转优化（查看跳转章节）。默认情况下，跳转优化是开启的。如果输出 “0” 到这个文件或者通过 `sysctl` 设置 `debug.kprobes_optimization` 为 0 ，所有优化的探针将会变成未优化的，而且在这之后任何新的被注册的探针都不会被优化。

注意，这个按钮会改变优化状态。表示已优化的探针（标记为 [OPTIMIZED]）将变成未优化的（标记 [OPTIMIZED] 会被移除）。如果按钮被打开，探针将再次被优化。