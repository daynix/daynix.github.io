---
layout: post
title:  "igb: Developing a new device for QEMU (part 4: Debugging and submitting patches)"
date:   2023-06-01 17:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

This is the last post of a walkthrough of new device emulator development for
QEMU which uses
[igb](https://qemu.readthedocs.io/en/v8.0.0/system/devices/igb.html),
a recently introduced Intel NIC emulation as an example.

[The first post](/2023/05/06/igb-developing-a-new-device-for-qemu-part-1-introduction.html)
roughly described there are several development steps:

1. Writing a boilerplate and adding basic features
2. Adding QTest
3. Adding advanced features and libvirt support
4. Running more tests and increasing feature coverage
5. Debugging
6. Contributing to the upstream

This post discusses 5 and 6. Though this post is written with new device
emulator development in mind, the debugging techniques shown here apply to other
kinds of QEMU development as well.

## Debugging

Debugging is the most interesting (and tiresome) part of the device emulator
development.

The first step of debugging is to make a bug reproduction case. Usually, you will
start debugging by creating a minimal reproduction case, but it is often
impractical for low-level software like QEMU as the higher-level software
is just too big; you cannot trim Linux down. The higher-level software may even
lack source code (c.f., Windows). In such a situation, you can't do anything but
use the real workload as a reproduction case. That also means
_there can be bugs anywhere_: guest user-space program, guest kernel,
QEMU, or even compiler.

[![BUGS - BUGS EVERYWHERE](/bugs-bugs-everywhere.jpg)](https://makeameme.org/meme/bugs-bugs-everywhere-4o3kqj)

But don't worry. It is always possible to figure out a bug no matter how the
reproduction case is complex. You just need right tools, knowledge,
methodology, and patience.

### Tools

First, let me introduce useful tools. You can't (or don't want to) debug
everything just with `printf()`, but you don't need dozens of tools either. The
most important thing is to choose tools according to the situation.

#### Hand-written scripts

The first thing to do, even before you try to figure out the bug, is to make
the iterations fast. Write some scripts to automate repetitive tasks, especially
the program launch. Scripts I used during development are available at:
[https://github.com/akihikodaki/q](https://github.com/akihikodaki/q)

You can choose whatever script language. Below is possible options:

##### Shell script

Shell is available everywhere so it may be handy. Unfortunately, it has various
quirks and compatibility issues.

##### C

C is actually not that bad. Basically, you just need to call `system()`
everywhere and omit error handling to write it "script-like".

The downside is that it requires compiling and is bad at error handling.

##### Ruby

An interesting characteristic of Ruby is that it exposes various Unix functions.
[It can also call C functions without any external library](https://ruby-doc.org/3.2.2/exts/fiddle/Fiddle.html)
so you can have unlimited access to the underlying system.

While Ruby is a good option for _system_ scripting as described above, it is not
as popular as Python or JavaScript. For example, Visual Studio Code comes with
mature support for Python and JavaScript, but Microsoft does not directly
provide support for Ruby.

##### Node.js

Node.js employs JavaScript language. Modern JavaScript is comfortable to write,
and its rich documentation on [MDN](https://developer.mozilla.org/) and
[formal specification](https://www.ecma-international.org/publications-and-standards/standards/ecma-262/)
are helpful if you are not familiar with the language.

Node.js' asynchronous nature is also attractive for scripting. For example,
if you need to `buildProject()` and `copyFiles()` (assuming both of them return
[Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises))
in parallel, you can just write as follows:

~~~ JavaScript
await Promise.all([buildProject(), copyFiles()])
~~~

[Google's zx](https://github.com/google/zx) makes it even easier to write
scripts.

However, Node.js is somewhat more restrictive than Ruby so you cannot easily
interact with file descriptors, for example.

#### git-bisect

[The man page](https://git-scm.com/docs/git-bisect) says:
> git-bisect - Use binary search to find the commit that introduced a bug

In principle, it's so simple and very effective to debug regressions, but
sometimes the bug you are debugging may involve several regressions introduced
in different commits. Naively performing binary search may not be sufficient in
such a case so you need to keep considering what is happening during git-bisect.

#### Sanitizers

[Sanitizers](https://github.com/google/sanitizers) detect common bugs in C code
at runtime. AddressSanitizer and UndefinedBehaviorSanitizer can be enabled by
passing the `--enable-sanitizers` flag to the `configure` script. I recommend
keeping them enabled.

Note that the entire QEMU code base is not free of addressability issues and
undefined behaviors. It's better to keep your workload minimal to avoid noise
from existing issues. There were several noisy bugs affecting my minimal
workload so I fixed them to keep the console clean[^1][^2][^3][^4].

#### printf()

It may sound stupid but, in fact, _`printf()` is a great tool for debugging_. It
works everywhere, and it has all of the expressiveness of the programming
language. Want the state of the program when the loop count is 42? Just write
`if (i == 42)`. Need to parse the bytes in an array? Write a parsing function
and call it. It's so simple and effective.

A downside is that such code can be often left after debugging. You may annotate
the logging code with a debug flag to pollute the console, which _mitigates_ the
problem but does not _solve_ it; it'll still end up with a lot of logs you
cannot really digest when the debug flag is set.

The rule of thumb for `printf()` debugging is to check in the code beforehand.
You can then just `git checkout .` (or safer: `git stash`) to discard all of
debug code at once. Another useful trick is to add your name to debug logs. A
person's name is often very distinguishable from the other code; you can easily
find it with `grep`. And if you forgot to delete it, others will ask you: "Hey,
there is your name in the code, did you forget to remove it?"

Another problem with `printf()` debugging is that it needs recompilation. There
may be tools for live patching, but live patching has inherently confusing as
the state of the old code persists. Don't stick to `printf()`, and switch to
debugging using GDB/LLDB when appropriate.

#### tracepoints

QEMU has
[tracing infrastructure](https://qemu.readthedocs.io/en/v8.0.0/devel/tracing.html).
It is basically `printf()` with external tooling support, but it is still useful
even if you just use the default "log" trace backend, which outputs traces to
standard error as it allows filtering traces.

For example, you can get all
traces from igb (except e1000x-common code) by adding the following to the
command line:

~~~ shell
-trace 'igb_*'
~~~

The name of tracepoints for igb is listed in `hw/net/trace-events`. Tracepoints
for other devices are listed in corresponding `trace-events` files.

The output of this pattern will be huge as it enables _all_ tracepoints igb has
so you can first skim it and narrow down the code. If you find something related
to IRQ suspicious, you can write:

~~~ shell
-trace 'igb_irq_*'
~~~

...and keep narrowing it down. It is also possible to enable/disable tracepoints
at runtime using the QEMU monitor, which will be described later.

To add a new trace point, you can add a new line to the `trace-events` file and
call the corresponding trace function. For example,

~~~
igb_set_vfmailbox(uint32_t vf_num, uint32_t val) "VFMailbox[%d]: 0x%x"
~~~

It is easy to filter tracepoints, but it is still important not to add too many
tracepoints. The documentation gives
[some hints for adding new trace events](https://qemu.readthedocs.io/en/v8.0.0/devel/tracing.html#hints-for-adding-new-trace-events):

> 1. Trace state changes in the code. Interesting points in the code usually
>    involve a state change like starting, stopping, allocating, freeing. State
>    changes are good trace events because they can be used to understand the
>    execution of the system.
> 2. Trace guest operations. Guest I/O accesses like reading device registers
>    are good trace events because they can be used to understand guest
>    interactions.
> 3. Use correlator fields so the context of an individual line of trace output
>    can be understood. For example, trace the pointer returned by malloc and
>    used as an argument to free. This way mallocs and frees can be matched up.
>    Trace events with no context are not very useful.
> 4. Name trace events after their function. If there are multiple trace events
>    in one function, append a unique distinguisher at the end of the name.

In other words, you should think carefully if you add a trace event that does
not fit well with this description. For example, it usually does not make sense
to add tracepoints for individual `if` statements or functions. If there are
well-designed trace events to trace state changes and guest operations with
meaningful correlator fields, you should be able to infer which `if` statements
or functions will be evaluated.

#### GDB and LLDB

GDB and LLDB are debuggers for C programs. They both provide features you will
expect; it allows you to pause the execution at some point of code (breakpoint)
or to pause the execution when it accesses some region of memory (watchpoint)
without recompilation. GDB is widely used so you can expect good support from
debugged programs. LLDB is a more modern alternative and (in my experience) has
superior reliability. You can pick either of them or use both of them, depending
on the situation.

As QEMU debugging involves software at various levels, you need to attach GDB to
some or even all of them, depending on the nature of the bug. In particular,
there are three levels you may be interested in:

- QEMU
- Guest kernel
- Guest userspace

##### QEMU

Attaching GDB or LLDB to QEMU is no different from attaching it to other
userspace programs; simply put, you just type:

~~~ shell
gdb build/qemu-system-aarch64
~~~

QEMU also has some GDB scripts that tweak it for debugging QEMU so it's better
to use them. To use them, load the `.gdbinit` file in the QEMU source tree. GDB
has a feature to load `.gdbinit` automatically, but usually, it's disabled for
the security reason so type the following to load it manually:

~~~
source .gdbinit
~~~

##### Guest kernel

Kernels often implement remote debugging protocols;
[one of our blog posts covers such a protocol of Windows](/2023/01/20/Guest-Windows-debugging-and-crashdumping-under-QEMU-KVM-Introduction.html).
Linux also comes with
[a similar feature called kgdb](https://www.kernel.org/doc/html/v6.3/dev-tools/kgdb.html).
However, they are sometimes unreliable when e.g., interrupts happen.

The remote debugging feature of QEMU is more reliable as it operates in a more
lower-level.
[This feature is documented well](https://qemu.readthedocs.io/en/v8.0.0/system/gdb.html),
but, in short, start QEMU with option `-s` added to the QEMU command line, start
gdb (with no argument), and type:

~~~
target remote :1234
~~~

Now GDB is attached to the guest and you can freely place breakpoints and
watchpoints.

However, this comes with no debug information so it's not practically usable in
this state. To use debug information from a guest kernel, you need to take some
procedure specific to it. Linux comes with
[a good documentation describing the procedure to debug it with QEMU](https://www.kernel.org/doc/html/v6.3/dev-tools/gdb-kernel-debugging.html)
so I recommend reading it. A common pitfall is that KASLR confuses the
symbolizer, which is described in the documentation. I also had to disable
in-kernel pointer authentication (`CONFIG_ARM64_PTR_AUTH_KERNEL`) for my Arm64
environment.

It may be still necessary to interrupt the guest execution without knowing the
address to place a breakpoint. For example,
[Windows does support debugging the kernel with QEMU](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-qemu-kernel-mode-debugging-using-exdi),
but it requires another Windows machine and so troublesome to set up. It doesn't
come with source code so you cannot know the exact address to place a breakpoint
anyway. A useful technique in such a case is to pause the execution from the
QEMU device code.

First, add the following includes to the head of the source file:

~~~ C
#include "exec/gdbstub.h"
#include "hw/core/cpu.h"
#include "sysemu/runstate.h"
~~~

Now you can add the following to the point where you want to pause the guest:

~~~ C
gdb_set_stop_cpu(current_cpu);
vm_stop(RUN_STATE_PAUSED);
~~~

The first line switches the CPU GDB interacts with. The second line actually
stops the VM. You can add conditions with `if` or do whatever since it's C.

#### filter-dump

Sometimes there may be debug tools specialized for a particular feature.
filter-dump is such a tool included in QEMU for networking and allows to dump
network traffic in QEMU. It's usually superior to `dumpcap` running on a tap
device attached to QEMU `netdev` as it runs at the very end of QEMU; it can see
corrupted packets sent from QEMU that will be filtered out by the tap device,
for example.

This tool was not working with packets with virtio-net headers attached so I
wrote
[a fix](https://gitlab.com/qemu-project/qemu/-/commit/481c52320a26e2e3a3c8a1cdac3d1460b9b15d13).
Such investment for a specialized tool is not essential but pays off well.

#### QEMU monitor

[QEMU monitor](https://www.qemu.org/docs/master/system/monitor.html) allows you
to interact with a live QEMU session. You can change the configuration of QEMU
by changing the command line, but it requires frequent restart so it's
recommended to learn how to change the configuration at runtime with the QEMU
monitor. Sometimes live configuration change is inevitable; you may need to add
and remove filter-dump to capture only a particular part of network traffic, for
example.

Snapshotting and hotplugging are only available with interactive access with
QEMU. You should occasionally test them as they are error-prone features.

### Case study

This section describes actual bug hunting during the igb development. The cause
of the bugs are not limited to QEMU, but include guest kernel and compiler.

#### DMA reentry bug

After sending igb patches to the upstream, several crash reports generated with
fuzzer came up:

[Heap-use-after-free in e1000e_receive_internal (#1543) · Issues · QEMU / QEMU · GitLab](https://gitlab.com/qemu-project/qemu/-/issues/1543)

This crash happened with e1000e, but I still got the report as I did lots of
cleanup before copying the e1000e code for igb. It is also likely that a bug
affecting e1000e affects igb as well.

Locally reproducing the crash resulted in the following logs:

~~~
==17147==ERROR: AddressSanitizer: heap-use-after-free on address 0xfffec4259798 at pc 0xaaabab108f88 bp 0xffffe3a4a780 sp 0xffffe3a4a790
READ of size 8 at 0xfffec4259798 thread T0
    #0 0xaaabab108f84 in e1000e_write_packet_to_guest ../hw/net/e1000e_core.c:1651
    #1 0xaaabab108f84 in e1000e_receive_internal ../hw/net/e1000e_core.c:1776
    #2 0xaaabab0cb978 in net_tx_pkt_send_custom ../hw/net/net_tx_pkt.c:823
    #3 0xaaabab0fe608 in e1000e_tx_pkt_send ../hw/net/e1000e_core.c:682
    #4 0xaaabab0fe608 in e1000e_process_tx_desc ../hw/net/e1000e_core.c:762
    #5 0xaaabab0fe608 in e1000e_start_xmit ../hw/net/e1000e_core.c:953
    #6 0xaaabab102694 in e1000e_set_tctl ../hw/net/e1000e_core.c:2496
    #7 0xaaabab10f918 in e1000e_core_write ../hw/net/e1000e_core.c:3349
    #8 0xaaababc913bc in memory_region_write_accessor ../softmmu/memory.c:493
(snip)

0xfffec4259798 is located 24 bytes inside of 208-byte region [0xfffec4259780,0xfffec4259850)
freed by thread T0 here:
    #0 0xfffecfc9d7cc in __interceptor_free /usr/src/debug/gcc/libsanitizer/asan/asan_malloc_linux.cpp:52
    #1 0xaaabab0cc3d0 in net_rx_pkt_iovec_realloc ../hw/net/net_rx_pkt.c:76
    #2 0xaaabab0cc3d0 in net_rx_pkt_pull_data ../hw/net/net_rx_pkt.c:99
    #3 0xaaabab0cf33c in net_rx_pkt_attach_iovec_ex ../hw/net/net_rx_pkt.c:153
    #4 0xaaabab102e70 in e1000e_receive_internal ../hw/net/e1000e_core.c:1764
    #5 0xaaabab0cb978 in net_tx_pkt_send_custom ../hw/net/net_tx_pkt.c:823
    #6 0xaaabab0fe608 in e1000e_tx_pkt_send ../hw/net/e1000e_core.c:682
    #7 0xaaabab0fe608 in e1000e_process_tx_desc ../hw/net/e1000e_core.c:762
    #8 0xaaabab0fe608 in e1000e_start_xmit ../hw/net/e1000e_core.c:953
    #9 0xaaabab102694 in e1000e_set_tctl ../hw/net/e1000e_core.c:2496
    #10 0xaaabab10f918 in e1000e_core_write ../hw/net/e1000e_core.c:3349
    #11 0xaaababc913bc in memory_region_write_accessor ../softmmu/memory.c:493
(snip)
    #21 0xaaabab0e5e5c in pci_dma_write /home/alarm/q/var/qemu/include/hw/pci/pci_device.h:271
    #22 0xaaabab0e5e5c in e1000e_write_to_rx_buffers ../hw/net/e1000e_core.c:1474
    #23 0xaaabab105218 in e1000e_write_packet_to_guest ../hw/net/e1000e_core.c:1646
    #24 0xaaabab105218 in e1000e_receive_internal ../hw/net/e1000e_core.c:1776
    #25 0xaaabab0cb978 in net_tx_pkt_send_custom ../hw/net/net_tx_pkt.c:823
    #26 0xaaabab0fe608 in e1000e_tx_pkt_send ../hw/net/e1000e_core.c:682
    #27 0xaaabab0fe608 in e1000e_process_tx_desc ../hw/net/e1000e_core.c:762
    #28 0xaaabab0fe608 in e1000e_start_xmit ../hw/net/e1000e_core.c:953
    #29 0xaaabab102694 in e1000e_set_tctl ../hw/net/e1000e_core.c:2496
    #30 0xaaabab10f918 in e1000e_core_write ../hw/net/e1000e_core.c:3349
    #31 0xaaababc913bc in memory_region_write_accessor ../softmmu/memory.c:493
(snip)

previously allocated by thread T0 here:
    #0 0xfffecfc9ed24 in __interceptor_malloc /usr/src/debug/gcc/libsanitizer/asan/asan_malloc_linux.cpp:69
    #1 0xfffece700ea4 in g_malloc (/usr/lib/libglib-2.0.so.0+0x70ea4)
    #2 0xaaabab0cc3d8 in net_rx_pkt_iovec_realloc ../hw/net/net_rx_pkt.c:77
    #3 0xaaabab0cc3d8 in net_rx_pkt_pull_data ../hw/net/net_rx_pkt.c:99
    #4 0xaaabab0cf33c in net_rx_pkt_attach_iovec_ex ../hw/net/net_rx_pkt.c:153
    #5 0xaaabab102e70 in e1000e_receive_internal ../hw/net/e1000e_core.c:1764
    #6 0xaaabab0cb978 in net_tx_pkt_send_custom ../hw/net/net_tx_pkt.c:823
    #7 0xaaabab0fe608 in e1000e_tx_pkt_send ../hw/net/e1000e_core.c:682
    #8 0xaaabab0fe608 in e1000e_process_tx_desc ../hw/net/e1000e_core.c:762
    #9 0xaaabab0fe608 in e1000e_start_xmit ../hw/net/e1000e_core.c:953
    #10 0xaaabab102694 in e1000e_set_tctl ../hw/net/e1000e_core.c:2496
    #11 0xaaabab10f918 in e1000e_core_write ../hw/net/e1000e_core.c:3349
    #12 0xaaababc913bc in memory_region_write_accessor ../softmmu/memory.c:493
(snip)
~~~

I omitted most stack trace entries of common QEMU code for brevity.

Looking at the logs, it seems use-after-free happened at line 1651 of
`hw/net/e1000e_core.c`. The content of the line is:

~~~ C
if (iov_ofs == iov->iov_len) {
~~~

This means `iov` was a dangling pointer when AddressSanitizer got triggered.
`iov` is assigned very early in the `e1000e_write_packet_to_guest` function and
the function appears in the stack trace of _free_ so it's likely that the `iov`
was valid at the beginning of the function but it got corrupted while the
function progresses. This leads to two possible explanations for the bug:
1. It was a mistake to retrieve `iov` early and to keep it in the function.
2. It was a mistake to free `iov` in the function.

To prove explanation 2, I looked at the stack trace of _free_. The stack trace
shows it tried to write the packet with DMA. In common sense, such DMA should
write RAM, but the stack trace says it triggered _another packet transmission_.
This sounds very wrong. In a nutshell, the following is what happened here:

1. The address of the MMIO register to trigger transmission was specified as the
   address of the received packet buffer.
2. Packet transmission was triggered.
3. The packet was looped back, and written to the address of the received packet
   buffer, which in reality points to the MMIO register specified in 1.
4. Another packet transmission and loopback happened, corrupting the first
   loopback.
5. The second loopback finishes, and the first loopback resumes.
6. The first loopback sees the corrupted state and AddressSanitizer triggers.

The first condition is very unlikely to happen in the real world, but a fuzzer
and malicious actors it tries to mimic can create such an abnormal situation.

I could devise a fix with this explanation; such a fix should add a flag which
indicates an MMIO access already happened and reject another MMIO access if the
flag is set. The easiest way is to add such a flag to e1000e, but considering
the entire QEMU program, it is likely that other devices need the same flag so
I decided to add it to the common memory subsystem.

Eventually, [I finished my fix and sent the upstream](https://patchew.org/QEMU/20230316162044.31607-1-akihiko.odaki@daynix.com/)
to find out
[there is already a fix for this kind of situation](https://patchew.org/QEMU/20230427211013.2994127-1-alxndr@bu.edu/).

##### Lessons learned

- Sanitizers and fuzzers do help.
- It is sometimes better to fix the common code instead of fixing the
  device-specific code.
- Don't forget to look for patches and issue reports for a common code bug ;)

#### Linux bug

I first tried Linux Test Project with e1000e instead of igb. That led to a
kernel warning and device reset:

~~~
[  194.808645] NETDEV WATCHDOG: enp0s4 (e1000e): transmit queue 0 timed out
[  194.809321] WARNING: CPU: 3 PID: 0 at net/sched/sch_generic.c:525 dev_watchdog+0x6f4/0x7f0
[  194.810092] Modules linked in:
[  194.810388] CPU: 3 PID: 0 Comm: swapper/3 Not tainted 6.3.0-rc6+ #24
[  194.810987] Hardware name: QEMU KVM Virtual Machine, BIOS edk2-stable202302-for-qemu 03/01/2023
[  194.811805] pstate: 61400005 (nZCv daif +PAN -UAO -TCO +DIT -SSBS BTYPE=--)
[  194.812461] pc : dev_watchdog+0x6f4/0x7f0
[  194.812845] lr : dev_watchdog+0x6f4/0x7f0
[  194.813233] sp : ffff800013227ad0
[  194.813552] x29: ffff800013227ad0 x28: ffff0000ef528510 x27: ffff80000e7cc000
[  194.814237] x26: 0000000000000003 x25: 0000000000000000 x24: 0000000000000000
[  194.814916] x23: ffff80000e7cc520 x22: ffff80000e7afac0 x21: ffff0000ef528420
[  194.815602] x20: ffff0000ef528000 x19: 1ffff00002644f6a x18: 0000000000000000
[  194.816284] x17: 1ffff00001f49ab0 x16: 0000000000000000 x15: 00000000a0f72ce0
[  194.818185] x14: 0000000000000000 x13: 0000000000000001 x12: ffff700002644ef3
[  194.820795] x11: 1ffff00002644ef2 x10: ffff700002644ef2 x9 : ffff80000831e908
[  194.823443] x8 : ffff800013227797 x7 : 00008ffffd9bb10e x6 : 0000000000000001
[  194.826174] x5 : ffff800013227790 x4 : 1fffe000184e560a x3 : dfff800000000000
[  194.828648] x2 : 0000000000000000 x1 : 0000000000000000 x0 : ffff0000c272b040
[  194.831117] Call trace:
[  194.831960]  dev_watchdog+0x6f4/0x7f0
[  194.833231]  call_timer_fn+0x1f4/0x860
[  194.834378]  __run_timers+0x73c/0xa50
[  194.835201]  run_timer_softirq+0x54/0xb8
[  194.836098]  __do_softirq+0x304/0xe4c
[  194.836986]  ____do_softirq+0x14/0x28
[  194.837879]  call_on_irq_stack+0x24/0x30
[  194.838782]  do_softirq_own_stack+0x20/0x30
[  194.839736]  __irq_exit_rcu+0x23c/0x5e8
[  194.840615]  irq_exit_rcu+0x18/0x88
[  194.841421]  el1_interrupt+0x34/0x50
[  194.842314]  el1h_64_irq_handler+0x14/0x20
[  194.843321]  el1h_64_irq+0x68/0x70
[  194.844117]  cpuidle_idle_call+0x254/0x358
[  194.844856]  do_idle+0x124/0x1e0
[  194.845460]  cpu_startup_entry+0x2c/0x38
[  194.846174]  secondary_start_kernel+0x12c/0x168
[  194.846976]  __secondary_switched+0xa4/0xa8
[  194.847741] irq event stamp: 447143
[  194.848385] hardirqs last  enabled at (447142): [<ffff80000b28db18>] el1_interrupt+0x40/0x50
[  194.849731] hardirqs last disabled at (447143): [<ffff80000b28d4e0>] el1_dbg+0x20/0x90
[  194.850987] softirqs last  enabled at (446880): [<ffff800008011128>] __do_softirq+0x8f0/0xe4c
[  194.852350] softirqs last disabled at (447119): [<ffff80000801b184>] ____do_softirq+0x14/0x28
~~~

The call trace originates from a watchdog timer and does not tell much. The
error message implies something with the transmission is wrong, but it is not
enough to identify the bug.

I first minimized the reproduction case because repetitively running LTP takes
too long. The test that triggered the warning was a test to make parallel HTTP
requests. In the end, I figured out that this bug can be reproduced by running
Apache on e1000e and requesting a 2-GiB file twice in parallel.

As I wrote earlier, it originates from a watchdog timer, which triggers a bit
after a problematic behavior occurs. This means it is unlikely that you can
understand the issue by looking at the last few QEMU traces.

It is necessary to inspect the kernel to identify the true origin of the
warning. I attached GDB to the guest and put the breakpoint at the line that
emits the warning. Below is the snippet from the kernel:

~~~ C
for (i = 0; i < dev->num_tx_queues; i++) {
	struct netdev_queue *txq;

	txq = netdev_get_tx_queue(dev, i);
	trans_start = READ_ONCE(txq->trans_start);
	if (netif_xmit_stopped(txq) &&
	    time_after(jiffies, (trans_start +
				 dev->watchdog_timeo))) {
		some_queue_timedout = 1;
		atomic_long_inc(&txq->trans_timeout);
		break;
	}
}

if (unlikely(some_queue_timedout)) {
	trace_net_dev_xmit_timeout(dev, i);
	WARN_ONCE(1, KERN_INFO "NETDEV WATCHDOG: %s (%s): transmit queue %u timed out\n",
	       dev->name, netdev_drivername(dev), i);
	netif_freeze_queues(dev);
	dev->netdev_ops->ndo_tx_timeout(dev, i);
	netif_unfreeze_queues(dev);
}
~~~

When the breakpoint hit, I typed: `p *dev`. This spams the console with all of
the information related to the network device. Skimming the output, I found the
value of the `name` member is `enp0s4`, which is the name of the e1000e device
and confirms it is a problem with e1000e.

Investigating further requires a more educated methodology than just skimming
`p *dev`. Below is the definition of `netdev_get_tx_queue`.

~~~ C
static inline
struct netdev_queue *netdev_get_tx_queue(const struct net_device *dev,
					 unsigned int index)
{
	return &dev->_tx[index];
}
~~~

`p dev->num_tx_queues` tells it's 1, so `dev->_tx[0]` should be the queue that
timed out. Then I looked at `netif_xmit_stopped`:

~~~ C
static inline bool netif_xmit_stopped(const struct netdev_queue *dev_queue)
{
	return dev_queue->state & QUEUE_STATE_ANY_XOFF;
}
~~~

where `QUEUE_STATE_ANY_XOFF` is defined as
`QUEUE_STATE_DRV_XOFF | QUEUE_STATE_STACK_XOFF`. `p dev->_tx[0].state` showed
`QUEUE_STATE_DRV_XOFF` was set so certainly this transmit queue is stopped.

Searching the kernel code base for `QUEUE_STATE_DRV_XOFF`, it seemed that the
bit is cleared with `netif_tx_wake_queue` and `netif_tx_start_queue`. Looking
for these symbols in `drivers/net/ethernet/intel/e1000e` led to the
`e1000_clean_tx_irq` function, which means the guest is not receiving tx IRQ
after the queue is stopped. I looked for kernel code that requests transmission
since tx IRQ will be generated only after that, finding such code in the
`e1000_xmit_frame` function.

~~~ C
netdev_sent_queue(netdev, skb->len);
e1000_tx_queue(tx_ring, tx_flags, count);
/* Make sure there is space in the ring for the next send. */
e1000_maybe_stop_tx(tx_ring,
		    (MAX_SKB_FRAGS *
		     DIV_ROUND_UP(PAGE_SIZE,
				  adapter->tx_fifo_limit) + 2));

if (!netdev_xmit_more() ||
    netif_xmit_stopped(netdev_get_tx_queue(netdev, 0))) {
	if (adapter->flags2 & FLAG2_PCIM2PCI_ARBITER_WA)
		e1000e_update_tdt_wa(tx_ring,
				     tx_ring->next_to_use);
	else
		writel(tx_ring->next_to_use, tx_ring->tail);
}
~~~

It seemed the kernel requests transmission by writing the first TDT register,
which is pointed to by `tx_ring->tail`. If QEMU works properly, the last call
for the `e1000e_set_tdt` function in the device emulation code before the reset
happens should be matched with a call for the `e1000e_set_interrupt_cause`
function with `E1000_ICR_TXQ0` as its argument. To confirm this, I added
`printf` calls to print when `e1000e_set_tdt` is called or
`e1000e_set_interrupt_cause` is called with `E1000_ICR_TXQ0`, and enabled
`e1000e_core_ctrl_sw_reset` trace.

Surprisingly, `e1000e_set_interrupt_cause` was always happening after the last
`e1000e_set_tdt` call, meaning QEMU emulates the device properly. This made me
question the kernel driver implementation. There was an earlier call of the
`e1000_maybe_stop_tx` kernel function, which stops the Tx queue, in
`e1000_xmit_frame`, but it was not followed with TDT register write:

~~~ C
int count = 0;

/* snip */

/* reserve a descriptor for the offload context */
if ((mss) || (skb->ip_summed == CHECKSUM_PARTIAL))
	count++;
count++;

count += DIV_ROUND_UP(len, adapter->tx_fifo_limit);

nr_frags = skb_shinfo(skb)->nr_frags;
for (f = 0; f < nr_frags; f++)
	count += DIV_ROUND_UP(skb_frag_size(&skb_shinfo(skb)->frags[f]),
			      adapter->tx_fifo_limit);

if (adapter->hw.mac.tx_pkt_filtering)
	e1000_transfer_dhcp_info(adapter, skb);

/* need: count + 2 desc gap to keep tail from touching
 * head, otherwise try next time
 */
if (e1000_maybe_stop_tx(tx_ring, count + 2))
	return NETDEV_TX_BUSY;
~~~

I added `writel(tx_ring->next_to_use, tx_ring->tail);` before the `return`
statement and it worked.

So this was a kernel bug. Before submitting it as a fix, I tried to understand
the intention of `e1000_xmit_frame`. Below is the summary of this function:

1. Check if there is sufficient space to put the current packet in the tx ring.
   Otherwise, return `NETDEV_TX_BUSY`.
2. Queue the current packet to the tx ring.
3. Stop the tx queue if there is no sufficient space for the next packet in
   tx ring.
4. Trigger transmission if there are no more packets or the tx is stopped.

And what I observed is that the check in step 1 was failing. In such a case,
the previous `e1000_xmit_frame` should have stopped the tx queue in step 3
triggered transmission in step 4. In fact,
`Documentation/networking/netdevices.rst` states the check in step 1 should
never fail:

> * NETDEV_TX_BUSY Cannot transmit packet, try later
>   Usually a bug, means queue start/stop flow control is broken in
>   the driver. Note: the driver must NOT put the skb in its DMA ring.

This implies it is a bug that the tx queue was not stopped in step 3 so I
inspected the condition to stop the tx queue in step 3.
`e1000_maybe_stop_tx` stops the queue if the tx queue has space specified with
the `size` argument. In this case, the value is:
`(MAX_SKB_FRAGS * DIV_ROUND_UP(PAGE_SIZE, adapter->tx_fifo_limit) + 2)`

This looks so complex and suspicious. I followed the computation of the `size`
argument of the earlier `e1000_maybe_stop_tx` call triggering `NETDEV_TX_BUSY`
in step 1 to find its maximum value. There are two increments of `count` at the
beginning so that counts two. `DIV_ROUND_UP(len, adapter->tx_fifo_limit)` will
be `MAX_SKB_FRAGS` at maximum. The later iterations on fragments will be
summed into `MAX_SKB_FRAGS * DIV_ROUND_UP(PAGE_SIZE, adapter->tx_fifo_limit)`
at maximum. In the end, `2` is added to `count`. Summarizing this, the maximum
value will be:

~~~ C
2 + MAX_SKB_FRAGS + MAX_SKB_FRAGS * DIV_ROUND_UP(PAGE_SIZE, adapter->tx_fifo_limit) + 2
= (MAX_SKB_FRAGS + 1) * DIV_ROUND_UP(PAGE_SIZE, adapter->tx_fifo_limit) + 4
~~~

This is greater than the value used in step 4. A patch to update the value fixed
the problem so I submitted the upstream and it's now merged[^5]. This bug was
present since the introduction of the e1000e driver in 2007 and 15-year-old when
it was fixed. It is surprising considering the affected workload is a plain
common HTTP server.

##### Lessons learned

- Don't naively trust the kernel even if the code is mature.
  (I also hit several other kernel bugs[^6][^7][^8][^9].)
- The kernel source visibility matters.
- Sometimes it's necessary to track the bug both from the host and guest side.

#### GCC bug

We covered a kernel bug so the next thing to cover is obviously (?) a compiler
bug. This compiler bug was found when running
[the Internet Protocol (IP) Pipeline application of DPDK](https://doc.dpdk.org/guides/sample_app_ug/ip_pipeline.html)
under DPDK Test Suite. A DPDK program has a somewhat unconventional architecture
that _polls the hardware with an infinite loop_, which confused GCC.

It was quite easy to locate the bug except it was something that
_looked impossible_. It was clear that the DPDK application hung since the QEMU
trace and the output of the application stopped. That usually implies QEMU did
something wrong and the application got confused by that and froze. However,
I could not find any bug looking at the source code of the application or adding
lots of `printf()` and attaching GDB. It seemed the request for the worker
thread is silently ignored so I looked at the implementation of the request
queue to find no bug.

After observing this impossible behavior for an hour, eventually, I decided to
trim down the infinite loop of the worker thread. There were several conditions
to process requests and function calls so I removed each of them and replaced
the request processing function with simple `puts()`. Below is what I got in the
end:

~~~ C
#include <stdio.h>

int g;
int h;

int main(void)
{
	for (int i = 0; ; i++) {
		for (int j = 0; j < g; j++);

		if (i & 1) {
			if (h)
				continue;

			if (g)
				puts("a");

			g = 1;
		}
	}
}
~~~

This code should indefinitely emit `a`, but it did nothing. It was indeed a
compiler bug.

To identify the buggy optimization pass I replaced the `-O3` compiler flag with
[individual optimization flags listed in the documentation](https://gcc.gnu.org/onlinedocs/gcc-13.1.0/gcc/Optimize-Options.html)
and performed binary search and derived the minimal reproducing combination of
flags: `-O1 -ftree-pre -ftree-partial-pre`

The bug was reported to the upstream and promptly fixed[^10]. Meanwhile, as it
takes a long for a fix to be available on a distribution, I added
`-fno-tree-partial-pre` to the compiler flag to disable the faulty optimization.

##### Lessons learned

- Something that looks impossible happens sometimes.
  (I hit yet another seemingly-impossible bug during igb development[^11].
   Although it was not the fault of the compiler, it also teaches the same
   lesson.)
- The usual methodologies like minimizing the reproduction case or binary search
  do apply to unusual cases.

## Contributing to the upstream

With all known bugs fixed, it's time to submit the changes to the upstream.
[QEMU has extensive documentation for patch submission](https://qemu.readthedocs.io/en/v8.0.0/devel/submitting-a-patch.html).

This process is a repetition of getting reviews and submitting a new version.
New device emulation code is often large as in the case of igb, and such code
takes many rounds of reviews; it took 9 versions before igb gets merged[^12]. As
igb and e1000e have
[generic virtual-device fuzzing](https://qemu.readthedocs.io/en/v8.0.0/devel/fuzzing.html#the-generic-fuzzer)
enabled, I also received reports of bugs found with the fuzzer and had to fix
each of them.

> Patience you must have, my young Padawan. - Master Yoda

The merge of the new device code marks the end of the initial development, but
it is not the end of the entire development process. Introducing a new device
makes you its maintainer so you will get patches and bug reports for it.
Community involvement keeps the code being improved.

## Conclusion

This series covered what new device emulation code for QEMU looks like by
depicting the introduction of igb, which was a 6-month journey[^13]. It is
complex and time-consuming, but tractable if you break it down into several
steps and divide-and-conquer. The new device code will benefit the entire QEMU
community and downstream once you finish the process and get it merged.

[^1]: [target/arm: Initialize debug capabilities only once (ad5c6dde) · Commits · QEMU / QEMU · GitLab](https://gitlab.com/qemu-project/qemu/-/commit/ad5c6ddea327758daa9f0e6edd916be39dce7dca)
[^2]: [vhost-user-gpio: Configure vhost_dev when connecting (daae36c1) · Commits · QEMU / QEMU · GitLab](https://gitlab.com/qemu-project/qemu/-/commit/daae36c13abc73cf1055abc2d33cb71cc5d34310)
[^3]: [vhost-user-fs: Back up vqs before cleaning up vhost_dev (331acddc) · Commits · QEMU / QEMU · GitLab](https://gitlab.com/qemu-project/qemu/-/commit/331acddc87b739c64b936ba4e58518f8491f1c6b)
[^4]: [vhost-user-i2c: Back up vqs before cleaning up vhost_dev (0126793b) · Commits · QEMU / QEMU · GitLab](https://gitlab.com/qemu-project/qemu/-/commit/0126793bee853e7c134627f51d2de5428a612e99)
[^5]: [e1000e: Fix TX dispatch condition](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eed913f6919e253f35d454b2f115f2a4db2b741a)
[^6]: [igb: Allocate MSI-X vector when testing](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=28e96556baca7056d11d9fb3cdd0aba4483e00d8)
[^7]: [igb: Enable SR-IOV after reinit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=50f303496d92e25b79bdfb73e3707ad0684ad67f)
[^8]: [igbvf: Regard vf reset nack as success](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=02c83791ef969c6a8a150b4927193d0d0e50fb23)
[^9]: [iommu/virtio: Detach domain on endpoint release](https://git.kernel.org/pub/scm/linux/kernel/git/joro/iommu.git/commit/?h=virtio&id=809d0810e3520da669d231303608cdf5fe5c1a70)
[^10]: [109002 – [13 Regression] -O1 -ftree-pre -ftree-partial-pre results in stall value](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109002)
[^11]: [ip: Enforce strict aliasing (!131) · Merge requests · slirp / libslirp · GitLab](https://gitlab.freedesktop.org/slirp/libslirp/-/merge_requests/131)
[^12]: [[v9] Introduce igb \| Patchew](https://patchew.org/QEMU/20230223105057.144309-1-akihiko.odaki@daynix.com/)
[^13]:  This does not include the time spent on the initial bring-up that was
        already done when I started working on igb. I also didn't spend
        full-time for igb development.
