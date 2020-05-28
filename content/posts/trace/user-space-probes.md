---
title: "译｜2008｜User-Space Probes (Uprobes)"
date: 2020-05-21T14:07:20+08:00
Lastmod: 2020-05-22
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
tags:
- uprobe
- trace
marp: false
---


## 译者序
这篇文章翻译自 SystemTap 项目中 [uprobes.txt](https://sourceware.org/git/?p=systemtap.git;a=blob;f=runtime/uprobes/uprobes.txt;h=edd524be022691498c7f38133b0e32192dcbb35f;hb=a2b182f549cf64427a474803f3c02286b8c1b5e1) 文件，此文件描述了 Uprobes 的概念、工作原理、限制等内容。用途跟 Kprobes 一样，用来追踪运行在用户态的应用程序的。看提交[历史](https://lore.kernel.org/linux-mm/20120209092642.GE16600@linux.vnet.ibm.com/)，这个功能在 2012 年才提交到 Linux 内核中。

文章最后提供的例子，可以修改来玩一玩。

**注：因为水平有限，文中难免存在遗漏或者错误的地方。如有疑问，建议直接阅读原文。**
***
<!-- ## Concepts: Uprobes, Return Probes
Uprobes enables you to dynamically break into any routine in a
user application and collect debugging and performance information
non-disruptively. You can trap at any code address, specifying a
kernel handler routine to be invoked when the breakpoint is hit. -->
## 概念：Uprobes、Return 探针
Uprobes 能够动态的介入应用程序的任意函数，采集调试和性能信息，且不引起混乱。你可以在任意地址上，指定断点命中时调用的内核函数。

<!-- There are currently two types of user-space probes: uprobes and
uretprobes (also called return probes).  A uprobe can be inserted on
any instruction in the application’s virtual address space.  A return
probe fires when a specified user function returns.  These two probe
types are discussed in more detail later. -->
目前，用户态探针有两种类型： uprobes 和 uretprobes（也叫 return 探针）。可以在应用程序的虚拟地址空间的任意指令上插入 uprobe 。 当用户函数返回的时候触发 return 探针。后续内容会详细的讨论这两类探针的细节。

<!-- A registration function such as register_uprobe() specifies which
process is to be probed, where the probe is to be inserted, and what
handler is to be called when the probe is hit. -->
`register_uprobe()` 注册函数设定要探测的进程，探针插入的位置，以及命中探针时调用什么回调函数。

<!-- Typically, Uprobes-based instrumentation is packaged as a kernel
module.  In the simplest case, the module’s init function installs
("registers") one or more probes, and the exit function unregisters
them.  However, probes can be registered or unregistered in response
to other events as well.  For example:
- A probe handler itself can register and/or unregister probes.
- You can establish Utrace callbacks to register and/or unregister
probes when a particular process forks, clones a thread,
execs, enters a system call, receives a signal, exits, etc.
See Documentation/utrace.txt. -->
通常，基于 Uprobes 的探测工具是被打包成内核模块。最简单的内核模块，初始化函数安装（"注册"）一个或多个探针，而后在 exit 函数中注销。其实还可以在响应其他事件中注册或注销探针。例如：
* 探针回调函数自身可以注册或注销探针
* 可以创建 Utrace 回调函数来注册或注销探针，探测特定的进程什么时候派生子进程、克隆线程、执行、进入系统调用、接收信号、退出等等。参考 `Documentation/utrace.txt`。

<!-- ### How Does a Uprobe Work?
When a uprobe is registered, Uprobes makes a copy of the probed
instruction, stops the probed application, replaces the first byte(s)
of the probed instruction with a breakpoint instruction (e.g., int3
on i386 and x86_64), and allows the probed application to continue.
(When inserting the breakpoint, Uprobes uses the same copy-on-write
mechanism that ptrace uses, so that the breakpoint affects only that
process, and not any other process running that program.  This is
true even if the probed instruction is in a shared library.) -->
### Uprobe 是怎么工作的？
当一个 uprobe 被注册后，Uprobes 会创建一个被探测指令的副本，停止被探测的应用程序，用断点指令替换被探测指令的首字节（在 i386 和 x86_64 上是 `int3`），之后让应用程序继续运行。（在插入断点的时候，Uprobes 使用与 ptrace 使用的相同的 `copy-on-write` 机制，这样断点也只影响那个进程，不会影响其他运行相同程序的进程。甚至是被探测的指令在共享库中也一样。）

<!-- When a CPU hits the breakpoint instruction, a trap occurs, the CPU’s
user-mode registers are saved, and a SIGTRAP signal is generated.
Uprobes `intercepts` the SIGTRAP and finds the associated uprobe.
It then executes the handler associated with the uprobe, passing the
handler the addresses of the uprobe struct and the saved registers.
The handler may block, but `keep in mind` that the probed thread remains
stopped while your handler runs. -->
当 CPU 命中断点指令的时候，发生了一个 trap，CPU 用户模式的寄存器都被保存起来，产生了一个 `SIGTRAP` 信号。Uprobes 拦截 `SIGTRAP` 信号，找到关联的 uprobe。然后，用 uprobe 结构体和先前保存的寄存器地址调用与 uprobe 关联的回调函数。这个回调函数可能会阻塞，但要记住回调函数执行期间，被探测的线程一直是停止的。

<!-- Next, Uprobes single-steps its copy of the probed instruction and
resumes execution of the probed process at the instruction following
the probepoint.  (It would be simpler to single-step the actual
instruction in place, but then Uprobes would have to temporarily
remove the breakpoint instruction.  This would create problems in a
multithreaded application.  For example, it would open a time window
when another thread could sail right past the probepoint.) -->
接下来，Uprobes 会单步执行被探测指令的副本，之后会恢复被探测的程序，让它在探测点之后的指令处继续执行。（实际上单步执行原始指令会更简单，但之后，Uprobes 必须移除断点指令。这在多线程应用程序中会引起问题。比如，当另一个线程执行过探测点的时会打开一个时间窗口。）

<!-- Instruction copies to be single-stepped are stored in a per-process
"single-step out of line (SSOL) area," which is a little VM area
created by Uprobes in each probed process’s address space. -->
被单步执行的指令副本存储在每个进程的"单步跳出（SSOL）区域"中，它是由 Uprobes 在每个被探测进程的地址空间中创建的很小的 VM 区域。

<!-- ### The Role of Utrace
When a probe is registered on a previously unprobed process,
Uprobes establishes a tracing "engine" with Utrace (see
Documentation/utrace.txt) for each thread (task) in the process.
Uprobes uses the Utrace "`quiesce`" mechanism to stop all the threads
`prior` to insertion or removal of a breakpoint.  Utrace also notifies
Uprobes of breakpoint and single-step traps and of other interesting
events in the lifetime of the probed process, such as fork, clone,
exec, and exit. -->
### Utrace 的作用
当先前被取消探测的进程上又注册一个探针的时候，Uprobes 用 Utrace 为进程中每个线程建立了一个追踪"引擎"。Uprobes 使用 Utrace "静默"机制，在插入或移除断点之前停止所有线程。Utrace 在被探测进程的生命周期中（fork, clone, exec, exit），通知 Uprobes 断点和单步执行陷阱以及其他感兴趣的事件。

<!-- ### How Does a Return Probe Work?
When you call register_uretprobe(), Uprobes establishes a uprobe
at the entry to the function.  When the probed function is called
and this probe is hit, Uprobes saves a copy of the return address,
and replaces the return address with the address of a "trampoline"
— a piece of code that contains a breakpoint instruction. -->
### Return 探针怎么工作的？
当你调用 `register_uretprobe()` 函数的时候，Uprobes 在函数的入口处创建一个 uprobe 。当调用被探测函数的时候命中这个探针，Uprobes 会保存 return 地址的一个副本，然后用"蹦床"的地址替换 return 地址 —— 一段包含一个断点指令代码。

<!-- When the probed function executes its return instruction, control
passes to the trampoline and that breakpoint is hit.  Uprobes’s
trampoline handler calls the user-specified handler associated with the
uretprobe, then sets the saved instruction pointer to the saved return
address, and that’s where execution resumes upon return from the trap. -->
当被探测的函数执行它的 return 指令时，控制转移到蹦床，命中断点。Uprobes 的蹦床回调函数调用与 uretprobe 关联的回调函数，然后把已保存的指令指针设置为已保存的 return 地址，再然后就从 trap 返回后的地方恢复执行。

<!-- The trampoline is stored in the SSOL area. -->
蹦床存储在 SSOL 区域中。

<!-- ### Multithreaded Applications
Uprobes supports the probing of multithreaded applications.  Uprobes
`imposes` no limit on the number of threads in a probed application.
All threads in a process use the same text pages, so every probe
in a process affects all threads; of course, each thread hits the
probepoint (and runs the handler) independently.  Multiple threads
may run the same handler simultaneously.  If you want a particular
thread or set of threads to run a particular handler, your handler
should check current or current->pid to determine which thread has
hit the probepoint. -->
###  多线程应用
Uprobes 支持多线程应用的探测。Uprobes 在被探测的应用中没有线程数量的限制。
在单个进程中的所有线程，使用相同的文本页，所以进程中的每个探针，会影响所有线程；当然，每个线程命中探测点（以及运行回调函数）是相对独立的。多个线程可能同时运行相同的回调函数。如果你想要一个特定的线程或是一组线程运行一个特定的回调函数，那回调函数应该检查 `current` 或 `current->pid` 来确认哪个线程命中了探测点。

<!-- When a process clones a new thread, that thread automatically shares
all current and future probes established for that process. -->
当进程克隆一个新的线程时，该线程自动的共享所有为进程创建的探针。

<!-- Keep in mind that when you register or unregister a probe, the
breakpoint is not inserted or removed until Utrace has stopped all
threads in the process.  The register/unregister function returns
after the breakpoint has been inserted/removed (but see the next
section). -->
要记住，注册或注销探针的时候，要等到 Utrace 停止了进程中的所有线程后，才会插入或删除断点。注册/注销函数在断点已经被插入或移除之后才返回（看下一章节）。

<!-- ### Registering Probes within Probe Handlers
A uprobe or uretprobe handler can call any of the functions
in the Uprobes API ([un]register_uprobe(), [un]register_uretprobe()).
A handler can even unregister its own probe.  However, when invoked
from a handler, the actual [un]register operations do not take
place immediately.  Rather, they are queued up and executed after
all handlers for that probepoint have been run.  In the handler,
the [un]register call returns -EINPROGRESS.  If you set the
registration_callback field in the uprobe object, that callback will
be called when the [un]register operation completes. -->
### 探针回调函数內注册探针
`uprobe` 或 `uretprobe` 回调函数可以调用 Uprobes API 中的任何函数（`[un]register_uprobe()`, `[un]register_uretprobe()`）。探针回调函数甚至可以注销它自己。不过，在回调函数中调用的时候，实际的注册/注销操作不会立刻执行。反而，它们会被放入队列，在该探测点已经运行所有回调函数之后执行。在回调函数中，注册/注销函数会返回 `-EINPROGRESS`。如果在 uprobe 对象中设置了 `registration_callback` 字段，会在注册/注销操作完成的时候调用。

<!-- ## Architectures Supported
Uprobes and uretprobes are implemented on the following
architectures: -->
## 已支持的 CPU 架构
`uprobes` 和 `uretprobes` 被实现，在下面的架构上：
- i386
- x86_64 (AMD-64, EM64T)
- ppc64
- s390x
<!--
## Configuring Uprobes
// TODO: The patch actually puts Uprobes configuration under "Instrumentation
// Support" with Kprobes.  Need to decide which is the better place. -->
## 配置 Uprobes
// TODO: 补丁实际上把 Uprobes 配置放在 "Instrumentation Support" 下面与 Kprobes 一起。需要决定哪个更好。

<!-- When configuring the kernel using make menuconfig/xconfig/oldconfig,
ensure that CONFIG_UPROBES is set to "y".  Under "Process debugging
support," select "Infrastructure for tracing and debugging user
processes" to enable Utrace, then select "Uprobes". -->
在使用 `make menuconfig/xconfig/oldconfig` 配置内核的时候，确保 `CONFIG_UPROBES` 设置为 "y"。在 "Process debugging support" 下面，选择 "Infrastructure for tracing and debugging user processes" 开启 Utrace，然后选择 "Uprobes"。

<!-- So that you can load and unload Uprobes-based instrumentation modules,
make sure "Loadable module support" (CONFIG_MODULES) and "Module
unloading" (CONFIG_MODULE_UNLOAD) are set to "y". -->
确保 "Loadable module support"（CONFIG_MODULES）和 "Module unloading" (CONFIG_MODULE_UNLOAD) 都被设置为 "y"，这样就可以加载或卸载基于 Uprobes 的测试工具模块。

<!-- ## API Reference
The Uprobes API includes a "register" function and an "unregister"
function for each type of probe.  Here are `terse`, mini-man-page
specifications for these functions and the associated probe handlers
that you'll write.  See the latter half of this document for examples. -->
## API 参考
Uprobes API 为每种类型的探针分别提供了"注册"和"注销"函数。这有一份这些函数以及相关的探针处理函数的简短说明。例子见文档后半部分。

### register_uprobe
```c
#include <linux/uprobes.h>
int register_uprobe(struct uprobe *u);
```
<!-- Sets a breakpoint at virtual address u->vaddr in the process whose
pid is u->pid.  When the breakpoint is hit, Uprobes calls u->handler. -->
在 pid 是 `u->pid` 的进程中，虚拟地址 `u->vaddr`处设置断点。在命中断点的时候，Uprobes 调用 `u->handler`。

<!-- register_uprobe() returns 0 on success, -EINPROGRESS if
register_uprobe() was called from a uprobe or uretprobe handler
(and therefore delayed), or a negative errno otherwise. -->
`register_uprobe()` 调用成功返回 0，如果是在 uprobe 或 uretprobe 回调函数中（因此延迟了）调用返回 `-EINPROGRESS`，否则返回负的 errno 。

<!-- Section 4.4, "User's Callback for Delayed Registrations",
explains how to be notified upon completion of a delayed
registration. -->
["延迟注册回调"](#延迟注册回调)，解释了在完成延迟注册后如何通知。

<!-- User's handler (u->handler): -->
用户的回调函数（u->handler）:
```c
#include <linux/uprobes.h>
#include <linux/ptrace.h>
void handler(struct uprobe *u, struct pt_regs *regs);
```

<!-- Called with u pointing to the uprobe associated with the breakpoint,
and regs pointing to the struct containing the registers saved when
the breakpoint was hit. -->
在断点命中的时候调用，传入指向断点关联的 uprobe 指针 `u` 和含有保存的寄存器的结构体指针 `regs`。

### register_uretprobe
```c
#include <linux/uprobes.h>
int register_uretprobe(struct uretprobe *rp);
```

<!-- Establishes a return probe in the process whose pid is rp->u.pid for
the function whose address is rp->u.vaddr.  When that function returns,
Uprobes calls rp->handler. -->
在 pid 是 `rp->u.pid` 的进程中，函数地址 `rp->u.vaddr` 处创建 return 探针。当该函数返回时，Uprobes 调用 `rp->handler`。

<!-- register_uretprobe() returns 0 on success, -EINPROGRESS if
register_uretprobe() was called from a uprobe or uretprobe handler
(and therefore delayed), or a negative errno otherwise. -->
`register_uretprobe()` 成功返回 0 ，如果是在 uprobe 或 uretprobe 回调函数中（因此被延迟）调用返回 `-EINPROGRESS`，否则返回负的 errno。

<!-- Section 4.4, "User's Callback for Delayed Registrations",
explains how to be notified upon completion of a delayed
registration. -->
["延迟注册回调"](#延迟注册回调)，解释了在完成延迟注册后如何通知。

<!-- User's return-probe handler (rp->handler): -->
用户的 return 探针回调函数（rp->handler）：
```c
#include <linux/uprobes.h>
#include <linux/ptrace.h>
void uretprobe_handler(struct uretprobe_instance *ri, struct pt_regs *regs);
```

<!-- regs is as described for the user's uprobe handler.  ri points to
the uretprobe_instance object associated with the particular function
instance that is currently returning.  The following fields in that
object may be of interest:
	- ret_addr: the return address
	- rp: points to the corresponding uretprobe object -->
`regs`表示用户的 uprobe 处理函数。`ri` 指向 `uretprobe_instance` 对象，其关联了当前正在返回的函数实例。可以关注对象中的两个字段：
- `ret_addr`：return 地址
- `rp`：指向对应的 uretprobe 对象

<!-- In ptrace.h, the regs_return_value(regs) macro provides a simple
`abstraction` to extract the return value from the appropriate register
as defined by the architecture's ABI. -->
在 `ptrace.h` 文件中，`regs_return_value(regs)` 宏提供了一种简单的抽象，从架构的 ABI 定义的相关寄存器中获得返回值。

### unregister_*probe
```c
#include <linux/uprobes.h>
void unregister_uprobe(struct uprobe *u);
void unregister_uretprobe(struct uretprobe *rp);
```

<!-- Removes the specified probe.  The unregister function can be called
at any time after the probe has been registered, and can be called
from a uprobe or uretprobe handler. -->
移除探针。注销函数可以在探针注册之后的任何时间调用，还能在 uprobe 或 uretprobe 回调函数中调用。

<!-- ### User's Callback for Delayed Registrations -->
### 延迟注册回调
```c
#include <linux/uprobes.h>
void registration_callback(struct uprobe *u, int reg,
	enum uprobe_type type, int result);
```

<!-- As previously mentioned, the functions described in Section 4 can be
called from within a uprobe or uretprobe handler.  When that happens,
the [un]registration operation is delayed until all handlers associated
with that handler's probepoint have been run.  Upon completion of the
[un]registration operation, Uprobes checks the registration_callback
member of the associated uprobe: u->registration_callback
for [un]register_uprobe or rp->u.registration_callback for
[un]register_uretprobe.  Uprobes calls that callback function, if any,
passing it the following values:
- u = the address of the uprobe object.  (For a uretprobe, you can use
container_of(u, struct uretprobe, u) to `obtain` the address of the
uretprobe object.)

- reg = 1 for register_u[ret]probe() or 0 for unregister_u[ret]probe()

- type = UPTY_UPROBE or UPTY_URETPROBE

- result = the return value that register_u[ret]probe() would have
returned if this weren't a delayed operation.  This is always 0
for unregister_u[ret]probe(). -->
像前面提到的函数，可以在 uprobe 或 uretprobe 回调函数内部调用。当发生这种情况的时候，注销/注册操作会被延迟，直到与探测点关联的所有回调函数都已运行之后执行。在完成注销/注册操作之后，Uprobes 会检查 uprobe 关联的 `registration_callback` 成员变量：uprobe 对应 `u->registration_callback` 或者 uretprobe 对应 `rp->u.registration_callback`。如果存在 Uprobes 会调用 `registration_callback` 回调函数，并传入下面的值：

* u = uprobe 对象的地址。（uretprobe 对象，可以使用 `container_of(u, struct uretprobe, u)` 获得 uretprobe 对象的地址。）
- reg = 1 for register_u[ret]probe() or 0 for unregister_u[ret]probe()
- type = UPTY_UPROBE or UPTY_URETPROBE
* result = 如果不是延迟操作，作为 register_u[ret]probe() 的返回值。对于 unregister_u[ret]probe() 总是返回 0 。

<!-- NOTE: Uprobes calls the registration_callback ONLY in the case of a
delayed [un]registration. -->
注意：Uprobes **只在**延迟注销/注册的情况下调用 `registration_callback`。

<!-- ## Uprobes Features and Limitations
The user is expected to assign values to the following members
of struct uprobe: pid, vaddr, handler, and (as needed)
registration_callback.  Other members are `reserved` for Uprobes's use.
Uprobes may produce unexpected results if you:
- assign non-zero values to reserved members of struct uprobe;
- change the contents of a uprobe or uretprobe object while it is registered; or
- attempt to register a uprobe or uretprobe that is already registered. -->
## Uprobes 功能与限制
希望用户给 uprobe 结构体的成员赋值：pid, vaddr, handler, （如果需要）registration_callback。其他保留的成员给 Uprobes 使用。如果做了下面这些事情，Uprobes 可能会产生不期望的结果：
- 把保留的 uprobe 结构体成员设置为非 0 值
- 在注册期间改变 uprobe 或 uretprobe 对象的内容
- 注册已注册的 uprobe 或 uretprobe

<!-- Uprobes allows any number of probes (uprobes and/or uretprobes)
at a particular address.  For a particular probepoint, handlers are
run in the order in which they were registered. -->
Uprobes 允许在特定的地址上注册任意数量的探针（uprobes、uretprobe 都可以）。探针回调函数是按照它们注册的顺序调用的。

<!-- Any number of kernel modules may probe a particular process
simultaneously, and a particular module may probe any number of
processes simultaneously. -->
任意数量的内核模块可以同时探测一个特定的进程，而特定的模块也可以同时探测任意数量的进程。

<!-- Probes are shared by all threads in a process (including newly created -->
threads).
在进程中的所有线程之间的探针是共享的（包括新创建的线程）。

<!-- If a probed process exits or execs, Uprobes automatically unregisters
all uprobes and uretprobes associated with that process.  `Subsequent`
attempts to unregister these probes will be treated as no-ops. -->
如果被探测进程退出或执行，Uprobes 会自动注销所有与之关联的 uprobes 和 uretprobes 。之后再注销这些探针将会视为无效。

<!-- On the other hand, if a probed memory area is removed from the
process's virtual memory map (e.g., via dlclose(3) or munmap(2)),
it's currently up to you to unregister the probes first. -->
另外，如果从进程的虚拟内存映射删除探测的内存区域的话（例如：通过 `dlclose(3)` 或 `munmap(2)`），目前需要先主动注销探针。

<!-- There is no way to specify that probes should be inherited across fork;
Uprobes removes all probepoints in the newly created child process.
See Section 7, "Interoperation with Utrace", for more information on
this topic. -->
没有方法在 fork 进程时继承探针，Uprobes 会在新创建的子进程中移除所有探测点。关于这点更多的信息见"与 Utrace 交互"。

<!-- On at least some architectures, Uprobes makes no attempt to verify
that the probe address you specify actually marks the start of an
instruction.  If you get this wrong, chaos may ensue. -->
至少在某些架构上，Uprobes 不会尝试校验，指定的探针地址是否是一条指令的开始。如果你弄错了，可能造成会混乱。

<!-- To avoid `interfering` with interactive debuggers, Uprobes will refuse
to insert a probepoint where a breakpoint instruction already exists,
unless it was Uprobes that put it there.  Some architectures may
refuse to insert probes on other types of instructions. -->
为避免干扰交互式调试工具，Uprobes 会拒绝在已存在断点指令的地方插入探测点，除非是 Uprobes 放那里的。有一些架构可能会拒绝在其他类型的指令上插入探针。

<!-- If you install a probe in an inline-able function, Uprobes makes
no attempt to chase down all inline instances of the function and
install probes there.  gcc may inline a function without being asked,
so keep this in mind if you're not seeing the probe hits you expect. -->
如果在可内联的函数中插入一个探针，Uprobes 并不会尝试给该函数的所有内联实例插入探针。如果没有命中期望的探针，要记住，gcc 可能会自动内联一个函数。

<!-- A probe handler can modify the environment of the probed function
-- e.g., by modifying data structures, or by modifying the
contents of the pt_regs struct (which are restored to the registers
upon return from the breakpoint).  So Uprobes can be used, for example,
to install a bug fix or to inject faults for testing.  Uprobes, of
course, has no way to distinguish the deliberately injected faults
from the accidental ones.  Don't drink and probe. -->
探针回调函数可以修改目标函数的环境 ——例如：修改数据结构，或修改 `pt_regs` 结构体的内容（从断点返回之后保存的寄存器）。因此，Uprobes 可以用来，安装补丁或测试时注入错误。当然 Uprobes 没有办法区分错误是故意注入的还是意外发生的。所以不要搞事情。

<!-- Since a return probe is implemented by replacing the return
address with the trampoline's address, stack backtraces and calls
to __builtin_return_address() will typically yield the trampoline's
address instead of the real return address for uretprobed functions. -->
因为 return 探针是通过使用蹦床的地址替换 return 地址实现的，那么栈回溯和调用 `__builtin_return_address()` 产生的是蹦床的地址，而不是uretprobed 函数的实际 return 地址。

<!-- If the number of times a function is called does not match the
number of times it returns (e.g., if a function exits via longjmp()),
registering a return probe on that function may produce undesirable
results. -->
如果函数调用的次数与 return 次数不匹配（例如：如果函数调用 `longjmp()` 退出），在这种函数上注册 return 探针，可能会产生不期望的结果。

<!-- When you register the first probe at probepoint or unregister the
last probe probe at a probepoint, Uprobes asks Utrace to "`quiesce`"
the probed process so that Uprobes can insert or remove the breakpoint
instruction.  If the process is not already stopped, Utrace stops it.
If the process is running an interruptible system call, this may cause
the system call to finish early or fail with EINTR.  (The PTRACE_ATTACH
request of the ptrace system call has this same limitation.) -->
当在探测点注册第一个探针或者注销最后一个探针的时候，Uprobes 要求 Utrace 去"暂停"目标进程，这样 Uprobes 就可以插入或者移除断点指令。如果进程还没有停止，Utrace 会停止它。如果进程正在运行一个可中断的系统调用，可能会让系统调用提早完成或失败而产生 `EINTR` 信号。（ptrace 系统调用的 `PTRACE_ATTACH` 请求有同样的问题。）

<!-- When Uprobes establishes a probepoint on a previous unprobed page
of text, Linux creates a new copy of the page via its copy-on-write
mechanism.  When probepoints are removed, Uprobes makes no attempt
to `consolidate` identical copies of the same page.  This could affect
memory availability if you probe many, many pages in many, many
long-running processes. -->
当 Uprobes 在先前的未探测的页面上建立探测点的时候，Linux 会通过 `copy-on-write` 机制创建了这个页面的新副本。在移除探测点的时候，Uprobes 并不会尝试合并同一个页面的副本。如果探测在大量长时间运行的进程中探测大量的页面，会影响内存可用性。

<!-- ## Interoperation with Kprobes
Uprobes is `intended` to interoperate usefully with Kprobes (see
Documentation/kprobes.txt).  For example, an instrumentation module
can make calls to both the Kprobes API and the Uprobes API. -->
## 与 Kprobes 交互
Uprobes 打算与 Kprobes 进行有效的相互操作（见 `Documentation/kprobes.txt` 文件）。例如，检测模块可以同时调用 Kprobes API 和 Uprobes API。

<!-- A uprobe or uretprobe handler can register or unregister kprobes,
jprobes, and kretprobes, as well as uprobes and uretprobes.  On the
other hand, a kprobe, jprobe, or kretprobe handler must not sleep, and
therefore cannot register or unregister any of these types of probes.
(Ideas for removing this restriction are welcome.) -->
uprobe 或 uretprobe 回调函数可以注册或注销 kprobes、jprobes、kretprobes，以及 uprobes 和 uretprobes。另外，kprobe、jprobe、kretprobe 回调函数一定不能休眠，不然会无法注册或注销这些探针。（欢迎移除这种限制的想法）

<!-- Note that the overhead of a u[ret]probe hit is several times that of
a k[ret]probe hit. -->
注意，命中 `u[ret]probe` 的开销是命中 `k[ret]probe` 的几倍。

<!-- ## Interoperation with Utrace
As mentioned in Section 1.2, Uprobes is a client of Utrace.  For each
probed thread, Uprobes establishes a Utrace engine, and registers
callbacks for the following types of events: clone/fork, exec, exit,
and "core-dump" signals (which include breakpoint traps).  Uprobes
establishes this engine when the process is first probed, or when
Uprobes is notified of the thread's creation, whichever comes first. -->
## 与 Utrace 交互
在"Utrace 的作用"章节中提到，Uprobes 是 Utrace 的客户端。Uprobes 为每个被探测的线程建立了一个 Utrace 引擎，及为 clone/fork, exec, exit, "core-dump" 信号（其包括断点陷阱）这类事件创建寄存器回调函数。Uprobes 是在进程首次被探测的时候创建引擎，或者在创建线程时通知，反正先到先处理。

<!-- An instrumentation module can use both the Utrace and Uprobes APIs (as
well as Kprobes).  When you do this, keep the following facts in mind:
- For a particular event, Utrace callbacks are called in the order in
which the engines are established.  Utrace does not currently provide
a mechanism for altering this order.
- When Uprobes learns that a probed process has forked, it removes
the breakpoints in the child process. -->
检测模块可以同时使用 Utrace 和 Uprobes APIs（以及 Kprobes）。这么做的时候，请记住下面的事情：
* 对于特定的事件，Utrace 回调函数是按引擎的创建顺序调用的。目前 Utrace 没有机制来改变顺序。
* 在 Uprobes 得知目标进程创建了子进程后，会在子进程中移除断点。

<!-- - When Uprobes learns that a probed process has exec-ed or exited,
it `disposes of` its data structures for that process (first allowing
any outstanding [un]registration operations to terminate). -->
- 在 Uprobes 得知目标进程已经执行或退出后，将会清理这个进程中的数据结构（先允许终止未完成的注销、注册操作）。

<!-- - When a probed thread hits a breakpoint or completes single-stepping
of a probed instruction, engines with the UTRACE_EVENT(SIGNAL_CORE)
flag set are notified.  The Uprobes signal callback prevents (via
UTRACE_ACTION_HIDE) this event from being reported to engines later
in the list.  But if your engine was established before Uprobes's,
you will see this this event. -->
- 当目标线程命中断点或被探测指令单步执行完成的时候，通知已设置 `UTRACE_EVENT(SIGNAL_CORE)` 标记的引擎。Uprobes 信号回调函数防止（通过 `UTRACE_ACTION_HIDE`）这个事件，报告给在列表后面的引擎。但，如果你的引擎是在 Uprobes 的引擎之前创建的，还是会收到这个事件。

<!-- If you want to establish probes in a newly forked child, you can use
the following `procedure`: -->
如果你想在新的子进程中创建探针，可以用以下办法：

<!-- - Register a report_clone callback with Utrace.  In this callback,
the CLONE_THREAD flag distinguishes between the creation of a new
thread vs. a new process. -->
- 用 Utrace 注册一个 `report_clone` 回调函数。在这个回调函数中，以 `CLONE_THREAD` 标记区分创建新线程还是进程。

<!-- - In your report_clone callback, call utrace_attach() to attach to
the child process, and set the engine's UTRACE_ACTION_QUIESCE flag.
The child process will `quiesce` at a point where it is ready to
be probed. -->
- 在你的 `report_clone` 回调函数中，调用 `utrace_attach()` 附着到子进程，以及设置引擎的 `UTRACE_ACTION_QUIESCE` 标记。子进程将会停顿在准备要探测的位置。

<!-- - In your report_quiesce callback, register the desired probes.
(Note that you cannot use the same probe object for both parent
and child.  If you want to duplicate the probepoints, you must
create a new set of u[ret]probe objects.) -->
- 在 `report_quiesce` 回调函数中，注册所需要的探针。（注意，不能对父子进程使用同一个探测对象。如果想要复制探测点，必须创建一个新的 `u[ret]probe` 对象集合。）

<!-- ## Probe Overhead
// TODO: This is out of date.
// TODO: Adjust as other architectures are tested.
On a typical CPU in use in 2007, a uprobe hit takes about 3
microseconds to process.  Specifically, a benchmark that hits the
same probepoint repeatedly, firing a simple handler each time, reports
300,000 to 350,000 hits per second, depending on the architecture.
A return-probe hit typically takes 50% longer than a uprobe hit.
When you have a return probe set on a function, adding a uprobe at
the entry to that function adds essentially no overhead. -->

Here are sample overhead figures (in usec) for different architectures.
## 探针开销
// TODO: 已经过时。
// TODO: 根据其他架构的测试整理。
在 2007 年常见的 CPU 上，处理 uprobe 命中大约需要3微秒的时间。基准测试反复命中相同的探测点，每次触发一个简单的处理程序，每秒报告 30w-35w 次命中，具体取决于架构。通常，return 探针命中比 uprobe 命中多花 50％ 的时间。当在某个函数上设置了 return 探针，会在该函数的入口处添加 uprobe ，本质上不会增加开销。

下面是些不同架构的样本（纳秒）。
```
u = uprobe; r = return probe; ur = uprobe + return probe

i386: Intel Pentium M, 1495 MHz, 2957.31 bogomips
u = 2.9 usec; r = 4.7 usec; ur = 4.7 usec

x86_64: AMD Opteron 246, 1994 MHz, 3971.48 bogomips
// TODO

ppc64: POWER5 (gr), 1656 MHz (SMT disabled, 1 virtual CPU per physical CPU)
// TODO
```

## TODO
<!-- * SystemTap (http://sourceware.org/systemtap): Provides a simplified
programming interface for probe-based instrumentation.  SystemTap
already supports kernel probes.  It could exploit Uprobes as well.
* Support for other architectures. -->
* [Systemtap](http://sourceware.org/systemtap)：基于探针的检测工具，提供简化的编程接口。SystemTap 已经支持内核探针。还可以利用 Uprobes 。
* 支持其他 CPU 架构

<!-- ## Uprobes Team
The following people have made major contributions to Uprobes: -->
## Uprobes 团队
下面的成员对 Uprobes 作出了主要的贡献：
* Jim Keniston - jkenisto@us.ibm.com
* Ananth Mavinakayanahalli - ananth@in.ibm.com
* Prasanna Panchamukhi - prasanna@in.ibm.com
* Dave Wilder - dwilder@us.ibm.com

<!-- ## Uprobes Example
Here's a sample kernel module showing the use of Uprobes to count the
number of times an instruction at a particular address is executed,
and optionally (unless verbose=0) report each time it's executed. -->
## Uprobes 例子
这儿有份内核模块样本，展示 Uprobes 的用法，统计在特定地址的指令执行了多少次，以及可选的（除非 verbose=0）输出每次执行。
```c
/* uprobe_example.c */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/uprobes.h>

/*
 * Usage: insmod uprobe_example.ko pid=<pid> vaddr=<address> [verbose=0]
 * where <pid> identifies the probed process and <address> is the virtual
 * address of the probed instruction.
 */

static int pid = 0;
module_param(pid, int, 0);
MODULE_PARM_DESC(pid, "pid");

static int verbose = 1;
module_param(verbose, int, 0);
MODULE_PARM_DESC(verbose, "verbose");

static long vaddr = 0;
module_param(vaddr, long, 0);
MODULE_PARM_DESC(vaddr, "vaddr");

static int nhits;
static struct uprobe usp;

static void uprobe_handler(struct uprobe *u, struct pt_regs *regs)
{
	nhits++;
	if (verbose)
		printk(KERN_INFO "Hit #%d on probepoint at %#lx\n",
			nhits, u->vaddr);
}

int __init init_module(void)
{
	int ret;
	usp.pid = pid;
	usp.vaddr = vaddr;
	usp.handler = uprobe_handler;
	printk(KERN_INFO "Registering uprobe on pid %d, vaddr %#lx\n",
		usp.pid, usp.vaddr);
	ret = register_uprobe(&usp);
	if (ret != 0) {
		printk(KERN_ERR "register_uprobe() failed, returned %d\n", ret);
		return -1;
	}
	return 0;
}

void __exit cleanup_module(void)
{
	printk(KERN_INFO "Unregistering uprobe on pid %d, vaddr %#lx\n",
		usp.pid, usp.vaddr);
	printk(KERN_INFO "Probepoint was hit %d times\n", nhits);
	unregister_uprobe(&usp);
}
MODULE_LICENSE("GPL");
```

<!-- You can build the kernel module, uprobe_example.ko, using the following Makefile: -->
你可以用下面的 Makefile 编译内核模块 `uprobe_example.ko`：
```makefile
obj-m := uprobe_example.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
clean:
	rm -f *.mod.c *.ko *.o .*.cmd
	rm -rf .tmp_versions
```

<!-- For example, if you want to run myprog and monitor its calls to myfunc(),
you can do the following: -->
例如，如果你想要运行 `myprog` ，然后监控 `myfunc()` 的调用情况，你可以这样做：
```shell
$ make			// Build the uprobe_example module.
...
$ nm -p myprog | awk '$3=="myfunc"'
080484a8 T myfunc
$ ./myprog &
$ ps
  PID TTY          TIME CMD
 4367 pts/3    00:00:00 bash
 8156 pts/3    00:00:00 myprog
 8157 pts/3    00:00:00 ps
$ su -
...
# insmod uprobe_example.ko pid=8156 vaddr=0x080484a8
```

<!-- In /var/log/messages and on the console, you will see a message of the
form "kernel: Hit #1 on probepoint at 0x80484a8" each time myfunc()
is called.  To turn off probing, remove the module: -->
每次调用 `myfunc()` 函数，将会在 `/var/log/messages` 文件中和终端上，看到这种信息："kernel: Hit #1 on probepoint at 0x80484a8"。要关闭探测，就移除模块：
```shell
# rmmod uprobe_example
```

<!-- In /var/log/messages and on the console, you will see a message of the
form "Probepoint was hit 5 times". -->
将会在 `/var/log/messages` 文件中和终端上看见这种信息："Probepoint was hit 5 times"。

<!-- ## Uretprobes Example
Here's a sample kernel module showing the use of a return probe to
report a function's return values. -->
## Uretprobes 例子
这是展示 return 探针用法的内核模块样本，输出函数的返回值。
```c
/* uretprobe_example.c */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/uprobes.h>
#include <linux/ptrace.h>

/*
 * Usage:
 * insmod uretprobe_example.ko pid=<pid> func=<addr> [verbose=0]
 * where <pid> identifies the probed process, and <addr> is the virtual
 * address of the probed function.
 */

static int pid = 0;
module_param(pid, int, 0);
MODULE_PARM_DESC(pid, "pid");

static int verbose = 1;
module_param(verbose, int, 0);
MODULE_PARM_DESC(verbose, "verbose");

static long func = 0;
module_param(func, long, 0);
MODULE_PARM_DESC(func, "func");

static int ncall, nret;
static struct uprobe usp;
static struct uretprobe rp;

static void uprobe_handler(struct uprobe *u, struct pt_regs *regs)
{
	ncall++;
	if (verbose)
		printk(KERN_INFO "Function at %#lx called\n", u->vaddr);
}

static void uretprobe_handler(struct uretprobe_instance *ri,
	struct pt_regs *regs)
{
	nret++;
	if (verbose)
		printk(KERN_INFO "Function at %#lx returns %#lx\n",
			ri->rp->u.vaddr, regs_return_value(regs));
}

int __init init_module(void)
{
	int ret;

	/* Register the entry probe. */
	usp.pid = pid;
	usp.vaddr = func;
	usp.handler = uprobe_handler;
	printk(KERN_INFO "Registering uprobe on pid %d, vaddr %#lx\n",
		usp.pid, usp.vaddr);
	ret = register_uprobe(&usp);
	if (ret != 0) {
		printk(KERN_ERR "register_uprobe() failed, returned %d\n", ret);
		return -1;
	}

	/* Register the return probe. */
	rp.u.pid = pid;
	rp.u.vaddr = func;
	rp.handler = uretprobe_handler;
	printk(KERN_INFO "Registering return probe on pid %d, vaddr %#lx\n",
		rp.u.pid, rp.u.vaddr);
	ret = register_uretprobe(&rp);
	if (ret != 0) {
		printk(KERN_ERR "register_uretprobe() failed, returned %d\n",
			ret);
		unregister_uprobe(&usp);
		return -1;
	}
	return 0;
}

void __exit cleanup_module(void)
{
	printk(KERN_INFO "Unregistering probes on pid %d, vaddr %#lx\n",
		usp.pid, usp.vaddr);
	printk(KERN_INFO "%d calls, %d returns\n", ncall, nret);
	unregister_uprobe(&usp);
	unregister_uretprobe(&rp);
}
MODULE_LICENSE("GPL");
```

<!-- Build the kernel module as shown in the above uprobe example. -->
像在上面的 uprobe 例子那样编译内核模块。

```shell
$ nm -p myprog | awk ‘$3=="myfunc"’
080484a8 T myfunc
$ ./myprog &
$ ps
  PID TTY          TIME CMD
 4367 pts/3    00:00:00 bash
 9156 pts/3    00:00:00 myprog
 9157 pts/3    00:00:00 ps
$ su -
…
# insmod uretprobe_example.ko pid=9156 func=0x080484a8
```

<!-- In /var/log/messages and on the console, you will see messages such
as the following: -->
在 `/var/log/messages` 文件中和终端上，会看到如下信息：
```shell
kernel: Function at 0x80484a8 called
kernel: Function at 0x80484a8 returns 0x3
```

<!-- To turn off probing, remove the module: -->
移除模块关闭探测：
```shell
# rmmod uretprobe_example
```

<!-- In /var/log/messages and on the console, you will see a message of the
form "73 calls, 73 returns". -->
在 `/var/log/messages` 文件中和终端上，会看到信息："73 calls, 73 returns"。
