---
title: "译｜2017｜Linux 追踪系统&如何组合在一起的"
date: 2020-05-28T18:40:02+08:00
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
tags:
- trace
series:
-
categories:
-
image:
---

# 译者序
在 Linux 系统上用来追踪、调试的工具有很多，有内核态的、用户态的、网络、IO 等等不同层次的工具。本文翻译自 [Linux tracing systems & how they fit together - Julia Evans](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)，这是在学习 Systemtap 原理时找到的资料，文章中就粗略讲了 Linux 的追踪系统，以及一点点来龙去脉，实际上工作中用到的大部分工具或多或少都是基于文章当中提到的某一种机制实现的。

**注：因为水平有限，文中难免存在遗漏或者错误的地方。如有疑问，建议直接阅读原文。**
***
<!--
I’ve been confused about Linux tracing systems for *years*. There’s strace, and ltrace, kprobes, and tracepoints, and uprobes, and ftrace, and perf, and eBPF, and how does it all fit together and what does it all MEAN? -->
我对于 Linux 追踪系统一直很困惑，有好几年了。strace、ltrace、kprobes、tracepoints、uprobes、ftrace、perf、eBPF，它们怎么串联在一起的，又有什么意义呢？

<!-- Last week I went to Papers We Love and later me & Kamal hung out with [Suchakra](https://twitter.com/tuxology) at [Polytechnique Montréal](http://www.dorsal.polymtl.ca/) (where LTTng comes from) and finally I think I understand how all these pieces fit together, more or less. There are still probably some mistakes in this post, please let me know what they are! (I’m b0rk on twitter). -->
上周，我去了"[Papers We Love](https://paperswelove.org/)"[^1]，后来我 & Kamal 到[蒙特利尔理工大学](http://www.dorsal.polymtl.ca/)（LTTng 项目开始的地方）与 [Suchakra](https://twitter.com/tuxology) 一起出去玩，终于我想我理解了所有的这些部件是如何组合在一起的了，或多或少吧。不过，在这篇文章中依然会有一些错误，如果发现了请让我知道！（twitter ID：b0rk）。

[^1]: 译注：Papers We Love 是一个计算机科学论文网站，经常组织线下活动

<!-- I’m going to leave strace out of this post (even though it’s my favorite thing) because the overhead is so high – in this post we’re only going to talk about tracing systems that are relatively fast / low overhead. This post also isn’t about sampling `profilers` at all (which is a whole other topic!). Just tracing. -->
这篇文章我要把 strace 抛开（它是我最喜欢的工具），因为开销太高了 - 在这篇文章中我们只会讨论相对高效/低开销的追踪系统。这篇文章也不是关于样本采集器的（这是另外一个主题！）。只是追踪。

<!-- The thing I learned last week that helped me really understand was – you can split linux tracing systems into **data sources** (where the tracing data comes from), **mechanisms for collecting data for those sources** (like "ftrace") and **tracing frontends** (the tool you actually interact with to collect/analyse data). The overall picture is still kind of `fragmented` and confusing, but it’s at least a more `approachable` fragmented/confusing system. -->
我上周学到的东西帮助我真正的了解了 —— 你可以把 linux 追踪系统拆分为数据源（追踪数据的来源），收集数据源的机制（类似 ftrace）和追踪前端（以交互式方式收集/分析数据的工具）。整体看上去还有些零散和模糊，但至少是一个更容易理解的方法。

<!-- here’s what we’ll talk about: (with links if you want to jump to a specific section). -->
以下是我们将要讨论的内容：
- [图片版本](#图片版本)
- [能追踪什么？](#能追踪什么)
- [数据源：kprobes, tracepoints, uprobes, dtrace probes 等等](#数据源kprobes-tracepoints-uprobes-dtrace-probes-等等)
  - [kprobes](#kprobes)
  - [uprobes](#uprobes)
  - [USDT/dtrace probes](#usdtdtrace-probes)
  - [kernel tracepoints](#kernel-tracepoints)
  - [lttng-ust](#lttng-ust)
- [收集"真香"数据的机制](#收集真香数据的机制)
  - [ftrace](#ftrace)
  - [perf_events](#perf_events)
  - [eBPF](#ebpf)
  - [sysdig](#sysdig)
  - [systemtap](#systemtap)
  - [LTTng](#lttng)
- [前端](#前端)
  - [perf 前端](#perf-前端)
  - [ftrace 前端](#ftrace-前端)
  - [eBPF 前端: bcc](#ebpf-前端-bcc)
  - [LTTng & SystemTap 前端](#lttng--systemtap-前端)
- [所以我应该使用什么样的追踪工具](#所以我应该使用什么样的追踪工具)
- [希望这篇文章有用！](#希望这篇文章有用)

<!-- It’s still kind of complicated but `breaking it up` this way really helps me understand (thanks to Brendan Gregg for suggesting this `breakdown` on twitter!) -->
这仍然有点复杂，但以这样的方式分解它，确实有助于我理解（感谢 Brendan Gregg 在 Twitter 上建议这样分解！）

<!-- ## a picture version
here are 6 drawings summarizing what this post is about: -->
## 图片版本
这里有 6 个草图，汇总了这篇文章的内容：
- 数据源、提取方法、前端
![](https://drawings.jvns.ca/drawings/linux-tracing-1.png)

- 数据源：kprobes、uprobes、tracepoints、USTD
![](https://drawings.jvns.ca/drawings/linux-tracing-2.png)

- 提取方法：ftrace 文件系统、perf_event 系统调用、eBPF 程序
![](https://drawings.jvns.ca/drawings/linux-tracing-3.png)

- 更多方式：Systemtap 脚本、LTTng 工具、sysdig 命令
![](https://drawings.jvns.ca/drawings/linux-tracing-4.png)

- 前端：更容易使用、更友好的显示数据
![](https://drawings.jvns.ca/drawings/linux-tracing-5.png)

- 更现代的前端
![](https://drawings.jvns.ca/drawings/linux-tracing-6.png)

<!-- ## What can you trace?
A few different kinds of things you might want to trace:
* System calls
* Linux kernel function calls (which functions in my TCP stack are being called?)
* Userspace function calls (did `malloc` get called?)
* Custom "events" that you’ve defined either in userspace or in the kernel -->
## 能追踪什么？
你也许想要追踪几种不同类型的事情：
* 系统调用
* Linux 内核函数调用（TCP 栈中的哪个函数正在被调用？）
* 用户空间函数调用（`malloc` 被调用了吗？）
* 在用户空间或是内核空间自定义的"事件"

<!-- All of these things are possible, but `it turns out` the tracing `landscape` is actually pretty complicated. -->
以上所有事情都是可能的，不过追踪环境实际上是非常复杂的。

<!-- ## Data sources: kprobes, tracepoints, uprobes, dtrace probes & more
Okay, let’s do data sources! This is kind of the most fun part – there are so many EXCITING PLACES you can get data about your programs. -->
## 数据源：kprobes, tracepoints, uprobes, dtrace probes 等等
Okay，让我们谈谈数据源！这是最有意思的部分——有太多地方可以获取程序的数据。

<!-- I’m going to split these up into "probes" (kprobes/uprobes) and "tracepoints" (USDT/kernel tracepoints / lttng-ust). I’m think I’m not using the right `terminology` exactly but there are 2 distinct ideas here that are useful to understand -->
我要把它们分成 "probes"（kprobes/uprobes）和 "tracepoints"（USDT/kernel tracepoints/ltng-ust）。实际上，我认为我没有使用正确的术语，这有 2 个不同的概念能帮助你去理解。

<!-- A **probe** is when the kernel dynamically modifies your assembly program at runtime (like, it changes the instructions) in order to enable tracing. This is super powerful (and kind of scary!) because you can enable a probe on literally any instruction in the program you’re tracing. (though dtrace probes aren’t "probes" in this sense). Kprobes and uprobes are examples of this pattern. -->
"probe" 是一种机制，内核可以在运行时修改你的汇编程序（类似，改变指令）来开启追踪。这点非常强大（有点可怕！），因为可以在你追踪的程序任意指令上启用一个 probe。（不过 dtrace probes 不是这种形式的 "probes"）。Kprobes 和 uprobes 就是这种形式的例子。

<!-- A **tracepoint** is something you compile into your program. When someone using your program wants to see when that tracepoint is hit and extract data, they can "enable" or "activate" the tracepoint to start using it. Generally a tracepoint in this sense doesn’t cause any extra overhead when it’s not activated, and is relatively low overhead when it is activated. USDT ("dtrace probes"), lttng-ust, and kernel tracepoints are all examples of this pattern. -->
"tracepoint" 是一种编译到你的程序中的东西。当使用程序的人，想要看看 tracepoint 什么时候命中及提取数据的时候，他们可以先"启用"或"激活" tracepoint 。通常，在 tracepoint 没有被激活的时候不会产生任何额外的开销，甚至在它被激活的时候开销也是相当的低。USDT（"dtrace probes"）、lttng-ust 、kernel tracepoints 它们都是这种模式的例子。

### kprobes
<!-- Next up is kprobes! What’s that? [From an LWN article](https://lwn.net/Articles/132196/): -->
接下来是 kprobes！那是什么？下面这段话来自 [LWN 的一篇文章](https://lwn.net/Articles/132196/):
<!-- > KProbes are a debugging mechanism for the Linux kernel which can also be used for monitoring events inside a production system. You can use it to weed out performance bottlenecks, log specific events, trace problems etc. -->
> KProbes 是一种 Linux 内核调试机制，也可以用来监控生产系统内部的事件。还能用它来发现性能瓶颈、记录特定的事件、追踪问题等等。

<!-- To `reiterate` – basically kprobes let you dynamically change the Linux kernel’s assembly code at runtime (like, insert extra assembly instructions) to trace when a given instruction is called. I usually think of kprobes as tracing Linux kernel function calls, but you can actually trace **any instruction inside the kernel and inspect the registers**. Weird, right? -->
复述一下，基本上 kprobes 能让你在运行时动态的改变 Linux 内核的汇编代码（比如，插入额外的汇编指令），追踪一个指令什么时候被调用。我以为 kprobes 就是追踪 linux 内核函数调用，实际上它可以追踪**内核中的任意指令以及检查寄存器**。神奇吧？

<!-- [Brendan Gregg has a kprobe script](https://github.com/brendangregg/perf-tools/blob/master/kernel/kprobe) that you can use to play around with kprobes. -->
[Brendan Gregg 有个 krpobe 脚本](https://github.com/brendangregg/perf-tools/blob/master/kernel/kprobe)，可以用来玩一玩 krpboes。

<!-- For example! Let’s use kprobes to `spy` on which files are being opened on our computer. I ran -->
例如！让我们用 kprobes 追踪，我们电脑上正在打开的文件。我执行了下面这条命令
```bash
$ sudo ./kprobe 'p:myopen do_sys_open filename=+0(%si):string'
```

<!-- from the examples and `right away` it started printing out every file that was being opened on my computer. Neat!!! -->
之后，我的电脑立即输出正在打开的文件。干净整齐！

<!-- You’ll notice that the kprobes interface by itself is a little `gnarly` though – like, you have to know that the filename argument to `do_sys_open` is in the `%si` register and dereference that pointer and tell the kprobes system that it’s a string. -->
你会注意到 kprobes 接口本身有点晦涩 - 比如，你必须知道给 `do_sys_open` 的 filename 参数是在 `%si` 寄存器中，还要取消对这个指针的引用，并告诉 kprobes 系统它是一个字符串。

<!-- I think kprobes are useful in 3 `scenarios`: 1. You’re tracing a system call. System calls all have corresponding kernel functions like `do_sys_open` 2. You’re debugging some performance issue in the network stack or to do with file I/O and you understand the kernel functions that are called well enough that it’s useful for you to trace them (not impossible!!! The linux kernel is just code after all!) 3. You’re a kernel developer,or you’re otherwise trying to debug a kernel bug, which happens sometimes!! (I am not a kernel developer) -->
我认为 kprobes 在以下 3 种情况中很有用：
1. 你正在追踪一个系统调用。所有系统调用都有对应的内核函数，比如：`do_sys_open`。
2. 你正在调试一些网络栈或者文件 IO 的性能问题，且足够了解被调用的内核函数，对你追踪它们很有帮助（不是不可能！毕竟 linux 内核也是代码）。
3. 你是一名内核开发者又或者在试着调试不常发生的内核 bug ！（我不是一名内核开发者）

### uprobes
<!-- Uprobes are kind of like kprobes, except that instead of instrumenting a *kernel* function you’re instrumenting userspace functions (like malloc). [brendan gregg has a good post from 2015](http://www.brendangregg.com/blog/2015-06-28/linux-ftrace-uprobe.html). -->
Uprobes 有点像 kprobes，只不过它不是检测内核函数的，而是检测用户空间函数的（例如 malloc）。可以阅读 brendan gregg 在 2015 年发布的一篇[文章](http://www.brendangregg.com/blog/2015-06-28/linux-ftrace-uprobe.html).

<!-- My understanding of how uprobes work is:
1. You decide you want to trace the `malloc` function in libc
2. You ask the linux kernel to trace malloc for you from libc
3. Linux goes and finds the copy of libc that’s loaded into memory (there should be just one, shared across all processes), and changes the code for `malloc` so that it’s traced
4. Linux reports the data back to you somehow (we’ll talk about how "asking linux" and "getting the data back somehow" works later) -->
我对 uprobes 工作原理的理解是：
1. 你决定想要去追踪在 libc 中的 `malloc` 函数
2. 你要求 linux 内核为你追踪 libc 中的 `malloc` 函数
3. Linux 找到 libc 被加载到内存中的副本（应该只有一份，共享给所有进程），然后改变 `malloc` 的代码方便被追踪
4. Linux 用某种办法把数据传递给你（稍后我们会谈怎么"要求 linux"以及"用某种办法获取数据"的原理）

<!-- This is pretty cool! One example of a thing you can do is `spy on` what people are typing into their bash terminals -->
你可以用它做的一件事情是监视其他人在他们的 bash 终端中输入的内容，很炫哝！
```bash
[email protected]~/c/perf-tools> sudo ./bin/uprobe 'r:bash:readline +0($retval):string'
Tracing uprobe readline (r:readline /bin/bash:0x9a520 +0($retval):string). Ctrl-C to end.
            bash-10482 [002] d...  1061.417373: readline: (0x42176e <- 0x49a520) arg1="hi"
            bash-10482 [003] d...  1061.770945: readline: (0x42176e <- 0x49a520) arg1=(fault)
            bash-10717 [002] d...  1063.737625: readline: (0x42176e <- 0x49a520) arg1="hi"
            bash-10717 [002] d...  1067.091402: readline: (0x42176e <- 0x49a520) arg1="yay"
            bash-10717 [003] d...  1067.738581: readline: (0x42176e <- 0x49a520) arg1="wow"
            bash-10717 [001] d...  1165.196673: readline: (0x42176e <- 0x49a520) arg1="cool-command"
```

### USDT/dtrace probes
<!-- USDT `stands for` "Userland Statically Defined Tracing", and "USDT probe" means the same thing as "dtrace probe" (which was surprising to me!). You might have heard of dtrace on BSD/Solaris, but you can actually also use dtrace probes on Linux, though the system is different. It’s basically a way to expose custom events. For example! [Python 3 has dtrace probes](https://docs.python.org/3/howto/instrumentation.html), if you compile it right. -->
USDT 全称为"用户级静态定义的追踪"，"USDT 探针" 和 "dtrace 探针"是一个东西（有点出乎我的意料！）。你也许已经听过 BSD/Solaris 上 dtrace，其实你也能在 Linux 上用 dtrace 探针。其实，它是一种暴露自定义事件的方法。举个例子，假如你编译正确的话， [Python 3 是有 dtrace 探针的](https://docs.python.org/3/howto/instrumentation.html)。

`python.function.entry(str filename, str funcname, int lineno, frameptr)`

<!-- This means that if you have a tool that can consume dtrace probes, (like eBPF / systemtap), and a version of Python compiled with dtrace support, you can automagically trace Python function calls. That’s really cool! (though this is a little bit of an "if" – not all Pythons are compiled with dtrace support, and the version of Python I have in Ubuntu 16.04 doesn’t seem to be) -->
如果有一个能够消费 dtrace 探针的工具（例如： eBPF/systemtap），和支持 dtrace 的 Python 版本，你就可以自动的追踪 Python 函数调用。很棒对吧！（然而这是一种"假设"————并不是所有编译的 Python 都支持 dtrace 探针的，在 Ubuntu 16.04 中的 Python 就不支持）

<!-- **How to tell if you have dtrace probes**, from [the Python docs](https://docs.python.org/3/howto/instrumentation.html). Basically you poke around in the binaries with readelf and look for the string "stap" in the notes. -->
怎么判断支不支持 dtrace 探针，根据 [Python 文档](https://docs.python.org/3/howto/instrumentation.html)，其实你可以用 readelf 工具在二进制文件中查找的 "stap" 字符串。

```bash
$ readelf -S ./python | grep .note.stapsdt
[30] .note.stapsdt        NOTE         0000000000000000 00308d78
# sometimes you need to look in the .so file instead of the binary
$ readelf -S libpython3.3dm.so.1.0 | grep .note.stapsdt
[29] .note.stapsdt        NOTE         0000000000000000 00365b68
$ readelf -n ./python
```

<!-- If you want to read more about dtrace you can read [this paper from 2004](https://www.cs.princeton.edu/courses/archive/fall05/cos518/papers/dtrace.pdf) but I’m not actually sure what the best reference is. -->
如果想阅读更多关于 dtrace 的内容，你可以阅读 [2004 年的论文](https://www.cs.princeton.edu/courses/archive/fall05/cos518/papers/dtrace.pdf)，不过我不知道是不是最好的参考。

### kernel tracepoints
<!-- Tracepoints are also in the Linux kernel. (here’s an [LWN article](https://lwn.net/Articles/379903/) ). The system was written by Mathieu Desnoyers (who’s from Montreal! :)). Basically there’s a `TRACE_EVENT` macro that lets you define tracepoints like this one (which has something to do with UDP… queue failures?): -->
Tracepoints 也在 Linux 内核中（这有篇[LWN 的文章](https://lwn.net/Articles/379903/)）。这个方法是 Mathieu Desnoyers（他来自 Montreal！）写的。简单来讲，有一个 `TRACE_EVENT` 宏，它能让你定义追踪点，像下面这样（它对 UDP 失败队列做些事情？）

```c
TRACE_EVENT(udp_fail_queue_rcv_skb,
           TP_PROTO(int rc, struct sock *sk),
        TP_ARGS(rc, sk),
        TP_STRUCT__entry(
                __field(int, rc)
                __field(__u16, lport)
        ),
```

<!-- I don’t really understand how it works (I think it’s pretty involved) but basically tracepoints:
* Are better than kprobes because they stay more constant across kernel versions (kprobes just depend on whatever code happens to be in the kernel at that time)
* Are worse than kprobes because somebody has to write them explicitly -->
我不太明白它是怎么工作的（我认为它涉及面很广），但基本上追踪点：
* 比 kprobes 好的一点，因为它们在内核版本之间更稳定（kprobes 依赖内核代码）（译注：换句话说就是不同内核版本函数会有很大变化，而 kprobes 又依赖于内核符号）
* 比 kprobes 差的一点，因为必须有人维护它们（译注：一般是内核开发人员或者是模块开发人员在关键的位置放置追踪点，以便可以读取关键数据）。

### lttng-ust
<!-- I don’t understand LTTng super well yet but – my understanding is that all of the 4 above things (dtrace probes, kprobes, uprobes, and tracepoints) all need to go through the kernel at some point. `lttng-ust` is a tracing system that lets you compile tracing probes into your programs, and all of the tracing happens in userspace. This means it’s faster because you don’t have to do context switching. I’ve still used LTTng 0 times so that’s mostly all I’m going to say about that. -->
我还不太了解 LTTng，但我的理解是，以上 4 种东西（dtrace probes，kprobes，uprobes，tracepoints）在某种程度上都需要内核的支持。`lttng-ust` 是一个追踪系统，它能让你把追踪探针编译到你的程序中，而且所有追踪都发生在用户空间。意味着它更快开销更小，因为你不必做上下文切换。我还没使用过 LTTng，所以这些就是我要说的内容。

<!-- ## Mechanisms for collecting your delicious delicious data
To understand the frontend tools you use to collect & analyze tracing data, it’s important to understand the `fundamental` mechanisms by which tracing data gets out of the kernel and into your grubby hands. Here they are. (there are just 5! ftrace, perf_events, eBPF, systemtap, and lttng).

Let’s start with the 3 that are actually part of the core Linux kernel: ftrace, perf_events, and eBPF. -->
## 收集"真香"数据的机制
要理解你用来收集&分析追踪数据的前端工具，理解把内核中的追踪数据取出来的基础机制是非常重要的。（这就只有 ftrace、perf_events、eBPF、systemtap、lttng 5 个工具）。

让我们从 ftrac，perf_events，eBPF 开始，实际上它们是 Linux 内核的一部分。

### ftrace
<!-- Those `./kprobe` and `./uprobe` scripts up there? Those both use `ftrace` to get data out of the kernel. Ftrace is a kind of `janky` interface which is a pain to use directly. Basically there’s a filesystem at `/sys/kernel/debug/tracing/` that lets you get various tracing data out of the kernel. -->
上面的 `./kprobe` 和 `./uprobe` 脚本？它们都使用 `ftrace` 从内核中获取数据。Ftrace 有点像一种名称接口，直接使用它有点痛苦。在 `/sys/kernel/debug/tracing/` 内有个文件系统，能让你从内核中获得各式各样的追踪数据。

<!-- The way you fundamentally interact with ftrace is
1. Write to files in `/sys/kernel/debug/tracing/`
2. Read output from files in `/sys/kernel/debug/tracing/` -->
与 ftrace 交互的基本方法是：
1. 把数据写入在 `/sys/kernel/debug/tracing/` 中的文件
2. 从 `/sys/kernel/debug/tracing/` 中的文件读出数据

<!-- Ftrace supports: * Kprobes * Tracepoints * Uprobes * I think that’s it. -->
Ftrace 支持：Kprobes、Tracepoints、Uprobes，我认为就是这样。

<!-- Ftrace’s output looks like this and it’s a pain to parse and `build on top of`: -->
基于下面 Ftrace 的输出很难做分析：
```bash
bash-10482 [002] d...  1061.417373: readline: (0x42176e <- 0x49a520) arg1="hi"
            bash-10482 [003] d...  1061.770945: readline: (0x42176e <- 0x49a520) arg1=(fault)
            bash-10717 [002] d...  1063.737625: readline: (0x42176e <- 0x49a520) arg1="hi"
            bash-10717 [002] d...  1067.091402: readline: (0x42176e <- 0x49a520) arg1="yay"
            bash-10717 [003] d...  1067.738581: readline: (0x42176e <- 0x49a520) arg1="wow"
            bash-10717 [001] d...  1165.196673: readline: (0x42176e <- 0x49a520)
```

### perf_events
<!-- The second way to get data out of the kernel is with the `perf_event_open` system call. The way this works is:
1. You call the `perf_event_open` system call
2. The kernel writes events to a ring buffer in user memory, which you can read from -->
从内核中获取数据的第二种方法，是使用 `perf_event_open` 系统调用。原理是：
1. 调用 `perf_event_open` 系统调用
2. 内核写把事件到一个在用户空间的环形缓冲区中，应用程序可以从中读取数据

<!-- As far as I can tell the only thing you can read this way is tracepoints. This is what running `sudo perf trace` does (there’s a tracepoint for every system call) -->
我知道的唯一的一件事情是，你可以以这种方式读取 tracepoints。这也就是执行 `sudo perf trace`  命令时做的事情（每个系统调用都有一个 tracepoints）

### eBPF
<!-- eBPF is a VERY EXCITING WAY to get data. Here’s how it works.
1. You write an "eBPF program" (often in C, or likely you use a tool that generates that program for you).
2. You ask the kernel to attach that probe to a kprobe/uprobe/tracepoint/dtrace probe
3. Your program writes out data to an eBPF map / ftrace / perf buffer
4. You have your `precious` precious data! -->
eBPF 是一种非常先进的获取数据的方式。它的工作原理。
1. 你写一段 "eBPF 程序"（通常是 C 语言，或者用一种生成这种程序的工具）
2. 要求内核把探针附着到 kprobe/uprobe/tracepoint/dtrace 探针
3. "eBPF 程序"把数据输出到一个 eBPF map/ftrace/perf 缓冲区
4. 你拥有了自己的数据！

<!-- eBPF is cool because it’s part of Linux (you don’t have to install any kernel modules) and you can define your own programs to do any fancy `aggregation` you want so it’s really powerful. You usually use it with the [bcc](https://github.com/iovisor/bcc) frontend which we’ll talk about a bit later. It’s only available on newer kernels though (the kernel version you need depends on what data sources you want to attach your eBPF programs to) -->
eBPF 非常好，因为它是 Linux 的一部分（不用安装内核模块），而且你可以定义自己的程序，去做任何你想做的奇怪的事情，因此它非常强大。平时是通过 [bcc](https://github.com/iovisor/bcc) 前端间接使用它，稍后再讨论 bcc。不过  eBPF 只在较新的内核上可用（选择内核的哪个版本，取决于想把 eBPF 程序附着到什么样的数据源）

<!-- Different eBPF features are available at different kernel versions, here’s a slide with an awesome summary: -->
不同的内核版本提供不同的 eBPF 特性，以下是一份很好的总结：
<script async class="speakerdeck-embed" data-id="130bc7df16db4556a55105af45cdf3ba" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

### sysdig
<!-- Sysdig is a kernel module + tracing system. It lets you trace system calls and maybe some other things? I find their site kind of confusing to navigate, but I think [this file](https://github.com/draios/sysdig/blob/dev/driver/event_table.c) contains the list of all the events sysdig supports. So it will tell you what files are being opened but not the weird internal details of what your TCP stack is doing. -->
Sysdig 是一个内核模块 + 追踪系统。它能让你追踪系统调用和一些其他的事情？我发现他们的站点有点难用，不过我认为[这个文件](https://github.com/draios/sysdig/blob/dev/driver/event_table.c)包含所有 sysydig 支持的事件列表。Sysdig 会告诉你正在打开文件描述符，但不告诉你 TCP 栈的内部细节。

### systemtap
<!-- I’m a little fuzzy how SystemTap works so we’re going to go from this [architecture document](https://sourceware.org/systemtap/archpaper.pdf)
	1. You decide you want to trace a kprobe
	2. You write a "systemtap program" & compile it into a kernel module
	3. That kernel module, when inserted, creates kprobes that call code from your kernel module when triggered (it calls [register_kprobe](https://github.com/torvalds/linux/blob/v4.10/Documentation/kprobes.txt) )
	4. You kernel modules prints output to userspace (using [relayfs or something](https://lwn.net/Articles/174669/) ) -->
我对 SystemTap 的工作原理有点不清楚，所以我们从[架构文档](https://sourceware.org/systemtap/archpaper.pdf)开始
1. 你决定要追踪一个 kprobe
2. 编写一个 "systemtap 程序"，把它编译到一个内核模块中
3. 内核模块在插入后，在激活时会从你的内核模块中调用代码创建 kprobes（调用 [register_kprobe](https://github.com/torvalds/linux/blob/v4.10/Documentation/kprobes.txt)）
4. 内核模块（使用 [relayfs or something](https://lwn.net/Articles/174669/)）打印输出到用户空间

<!-- SystemTap supports: * tracepoints * kprobes * uprobes * USDT -->
SystemTap 支持：tracepoints、kprobes、uprobes、USDT

<!-- Basically lots of things! There are some more useful words about systemtap in [choosing a linux tracer](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html) -->
支持许多东西！在这篇 [选择一个 linux 追踪工具](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html) 的文章中对 systemtap 有更多的描述

### LTTng
<!-- [LTTng](https://lttng.org/) (linux tracing: the next generation) is from Montreal (a lab at ecole polytechnique)!! which makes me super happy (montreal!!). I saw an AMAZING demo of tool called [trace compass](http://tracecompass.org/) the other day that reads data that comes from LTTng. Basically it was able to show all the `sched_switch` `transitions` between programs and system calls when running `tar -xzf somefile.tar.gz`, and you could really see exactly what was happening in a super clear way. -->
[LTTng](https://lttng.org/) (linux tracing: the next generation) 是出自 Montreal （巴黎高等理工学院的实验室）！！，让我超级高兴。前几天，我见过一个被称作[追踪罗盘](http://tracecompass.org/)的了不起的 demo 工具，它从 LTTng 读取数据。在执行 `tar -zxf somefile.tar.gz` 命令时，它可以显示程序与系统调用之间的所有 `sched_switch` 变化过程，而且真的可以以一种非常清晰的方式看见正在发生的事情。

<!-- The downside of LTTng (like SystemTap) is that you have to install a kernel module for the kernel parts to work. With `lttng-ust` everything happens in userspace and there’s no kernel module needed. -->
LTTng 的缺点（类似 SystemTap）是，你必须安装一个内核模块才能使内核的部分正常工作。使用 `lttng-ust` 时发生的一切都在用户空间，而且没有内核模块（译注：systemtap 会编译出一个内核模块）。

<!-- ## Frontends
Okay! Time for frontends! I’m going to categorize them by mechanism (how the data gets out of the kernel) to make it easier -->
## 前端
Okay！该谈前端了！我要把他们按照机制（怎么将数据从内核中取出来）分类，理解起来更容易一点

<!-- ### perf frontends
The only frontend here is `perf`, it’s simple.

`perf trace` will trace system calls for you, fast. That’s great and I love it. `perf trace` is the only one of these I actually use day to day right now. (the ftrace stuff is more powerful and also more confusing / difficult to use) -->
### perf 前端
这儿唯一简单的前端是 `perf` 。

`perf trace` 迅速为你追踪系统调用。简单，喜欢。实际上，现在 `perf trace` 是这些前端当中的唯一一个我每天都在使用的工具。（ftrace 这个东西非常强大，但也更难以使用）

<!-- ### ftrace frontends
Ftrace is a pain to use on its own and so there are various frontend tools to help you. I haven’t found the best thing to use yet but here are some starting points: -->
### ftrace 前端
Ftrace 用起来有点痛苦，因此也有各式各样的前端工具可以帮到你。我还没发现最好的工具，不过可以从下面这些开始：

<!-- * **trace-cmd** is a frontend for ftrace, you can use it to collect and display ftrace data. I wrote about it a bit in [this blog post](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/) and there’s an [article on LWN](https://lwn.net/Articles/410200/) about it
* [Catapult](https://github.com/catapult-project/catapult) lets you analyze ftrace output. It’s for Android / chrome performance originally but you can also just analyze ftrace. So far the only thing I’ve gotten it to do is graph `sched_switch` events so you know which processes were running at what time exactly, and which CPU they were on. Which is pretty cool but I don’t really have a use for yet?
* [kernelshark](http://rostedt.homelinux.com/kernelshark/) consumes ftrace output but I haven’t tried it yet
* The **perf** command line tool is a perf frontend and (confusingly) also a frontend for some ftrace functionality (see [perf ftrace](http://man7.org/linux/man-pages/man1/perf-ftrace.1.html) ) -->
* **trace-cmd** 作为 ftrace 的一种前端，你可以用它收集和显示 ftrace 数据。在这篇[文章](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/)中我写了一点，另外在 [LWN](https://lwn.net/Articles/410200/) 上有一篇关于它的文章
* [Catapult](https://github.com/catapult-project/catapult) 能让你分析 ftrace 的输出。最初它是为了分析 Android/chrome 性能，但你也可以只分析 ftrace 。目前为止，唯一要做的事情就是绘制 `sched_switch` 事件图，以便知道实际是哪些进程在运行，以及它们在哪个 CPU 上。我还没有真的使用过，哪个更好？
* [kernelshark](http://rostedt.homelinux.com/kernelshark/) 消费 ftrace 的输出，只是我还没有尝试过
* **perf** 命令行工具是一个 perf 前端，并且还是一些 ftrace 功能的前端（有点乱）（见 [perf ftrace](http://man7.org/linux/man-pages/man1/perf-ftrace.1.html)）

<!-- ### eBPF frontends: bcc
The only I know of is the **bcc** framework: [https://github.com/iovisor/bcc](https://github.com/iovisor/bcc). It lets you write eBPF programs, it’ll insert them into the kernel for you, and it’ll help you get the data out of the kernel so you can process it with a Python script. It’s pretty easy to get started with. -->
### eBPF 前端: bcc
我唯一了解的是 **bcc** 框架：[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)。它能够让你编写 eBPF 程序，帮你把它们插入到内核中，然后它会帮你把数据从内核中读出来，让你可以用 Python 脚本来处理。这框架非常容易上手。

<!-- If you’re `curious` about the relationship between eBPF and the BPF you use in tcpdump I wrote a [post about eBPF & its relationship with BPF for packet filtering the other day](https://jvns.ca/blog/2017/06/28/notes-on-bpf---ebpf/). I think it might be easiest though to think of them as `unrelated` because eBPF is so much more powerful. -->
如果你好奇 tcpdump 中的 BPF 和 eBPF 的关系，可以看看前几天我写过一篇它们之间关系的[文章](https://jvns.ca/blog/2017/06/28/notes-on-bpf---ebpf/)。不过我认为最简单的是把它们看作是两个东西就好，因为 eBPF 强大的多。

<!-- bcc is a bit weird because you write a C program inside a Python program but there are a lot of examples. Kamal and I wrote a program with bcc the other day for the first time and it was pretty easy. -->
bcc 有点怪异，因为你要在 Python 程序内部写一个 C 程序，不过倒是有很多例子可以参考。前几天 Kamal 和我第一次用 bcc 编写了一个非常简单的程序。

<!-- ### LTTng & SystemTap frontends
LTTng & SystemTap both have their own sets of tools that I don’t really understand. THAT SAID – there’s this cool graphical tool called [Trace Compass](http://tracecompass.org/) that seems really powerful. It consumes a trace format called CTF ("common trace format") that LTTng emits. -->
### LTTng & SystemTap 前端
LTTng & SystemTap 都有一套它们自己的工具，我不太了解。说真的————这个被称作[追踪罗盘](http://tracecompass.org/)的图形工具，它好像真的很强大。它消费一种 LTTng 产生的数据，称作 CTF（"通用追踪格式"）追踪格式。

<!-- ### what tracing tool should I use though
Here’s kind of how I think about it right now (though you should note that I only just figured out how all this stuff fits together very recently so I’m not an expert):
* if you’re mostly interested in computers running kernels > linux 4.9, probably just learn about eBPF
* `perf trace` is good, it will trace system calls with low overhead and it’s super simple, there’s not much to learn. A+.
* For everything else, they’re, well, an investment, they take time to get used to.
* I think playing with kprobes is a good idea (via eBPF/ftrace/systemtap/lttng/whatever, for me right now ftrace is easiest). Being able to know what’s going on in the kernel is a good superpower.
* eBPF is only available in kernel versions above 4.4, and some features only above 4.7. I think it makes sense to invest in learning it but on older systems it won’t help you out yet
* ftrace is kind of a huge pain, I think it would be worth it for me if I could find a good frontend tool but I haven’t managed it yet. -->
## 所以我应该使用什么样的追踪工具
我现在的想法是（你应该知道，我只是刚刚弄清楚它们如何组装在一起的，我不是专家）：
* 如果你计算机中主要运行 > linux 4.9 版本的内核，可能只用学习 eBPF
* `perf trace` 它可以追踪系统调用，低开销，且非常简单，么有太多要学的。A+。
* 至于其他的，都是一种投资，要花些时间来学习。
* 我认为 kprobes 用来玩玩还是可以的（eBPF/ftrace/systemtap/lttng/ 等方式，对于我来说 ftrace 是最简单的）。能够知道在内核中发生的一切是一种很好的超能力。
* eBPF 只在 4.4 以上的内核版本中可用，且有些特性只在 4.7 以上。我想是有意义的，投资学习它，但在比较旧的系统上，它还不能帮助你
* ftrace 有点太痛苦了，不过对于我来说如果可以找到一个好的前端工具它是有价值的，但是还没找到

<!-- ### I hope this helped!
I’m really excited that now I (mostly) understand how all these pieces fit together, I am literally typing some of this post at a free show at the jazz `festival` and listening to blues and having a fun time. -->
## 希望这篇文章有用！
我真的很高兴，现在我了解了（大部分）所有的组件是如何组装在一起的，这篇文章是我在爵士音乐节上的一场免费的表演中写的，听着布鲁斯过的很愉快。

<!-- Now that I know how everything fits together, I think I’ll have a much easier time navigating the landscape of tracing frontends! -->
现在，我知道它们是如何组合在一起的，我认为我可以更轻松的跟进追踪前端的进展！

<!-- [Brendan Gregg](http://www.brendangregg.com/blog/index.html) ’s awesome blog discusses a ton of these topics in a lot of detail – if you’re interested in hearing about improvements in the Linux tracing ecosystem as they happen (it’s always changing!), that’s the best place to subscribe. -->
[Brendan Gregg](http://www.brendangregg.com/blog/index.html) 的博客中有大量的这些主题中的细节——如果你有兴趣了解 linux 追踪生态中的改进（一直在变化！），那是最好的博客。

<!-- thanks to Annie Cherkaev, Harold Treen, Iain McCoy, and David Turner for reading a draft of this. -->
感谢 Annie Cherkaev, Harold Treen, Iain McCoy, David Turner 阅读这篇文章的草稿。