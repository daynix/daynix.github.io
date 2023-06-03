---
layout: post
title:  "Developing QEMU on Asahi Linux (or Arm in general)"
date:   2023-06-03 15:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

{% include 2023-06-03-developing-qemu-on-asahi-linux-linux-port-for-apple-silicon/logo.svg %}

Recently I described
[how igb, a new Intel NIC support for QEMU, was added](https://daynix.github.io/2023/05/06/igb-developing-a-new-device-for-qemu-part-1-introduction.html)
here. The development was done on an x86 machine until I buy a M2 Macbook Air,
which carries Arm-based Apple Silicon. This computer is way faster than the x86
machine so I decided to use [Asahi Linux](https://asahilinux.org/), a community
port of Linux for Apple Silicon and to move the development to this new machine.

You can find details like currently supported features on
[its wiki](https://github.com/AsahiLinux/docs/wiki) and fantastic write-ups
about the development on [its blog](https://asahilinux.org/blog/). Here, I
describe what worked well during my QEMU development on the platform, or what
didn't (and how I fixed them).

## Asahi Linux (usually) works well

Asahi Linux works so well, which may be surprising if you don't closely follow
the development, finding skilled and motivated developers and tooling like
[m1n1 hypervisor](https://github.com/AsahiLinux/m1n1).

[The introduction of GPU driver](https://asahilinux.org/2022/12/gpu-drivers-now-in-asahi-linux/)
made it truly pleasant to work with. The GPU driver is one of the challenging
features and while it still lacks functionalities of modern OpenGL versions like
4.0+, it implements everything that is needed to provide a decent desktop
experience. Things I regularly use like GNOME, Firefox, Thunderbird, and
Visual Studio Code all work just fine[^1].

And the GPU driver was _the last missing piece_ for the simple workload. There
are many things not implemented like speakers, cameras, and external display
support. However, Asahi Linux is perfect if you just need the keyboard, touchpad
(with multi-finger detection), display, audio jack, (minimal) power management,
and USB. It's perfect for QEMU development.

## When Asahi Linux does not work well

### Legacy programs

If your desktop environment on Asahi Linux does not work well, it's probably
because your configuration includes legacy components, notably Pulse Audio and
X.

Well, Pulse Audio is not _that_ old compared to X, but
[it's known to be faulty on Asahi Linux](https://asahilinux.org/2022/11/november-2022-report/#audio-advances-track--1)
so it's better to use PipeWire.

X is known for various protocol limitations that are nearly impossible to fix.
Fortunately, Wayland compositors that substitute X are so mature today that you
find fewer quirks on Wayland compositors than X. I see nothing quirky with
GNOME's Wayland implementation these days.

### (Superficial) lack of Arm support

The more serious problem in practice is (superficial) lack of Arm support on
Linux system. Arm is supported well by Linux kernel and userspace when compared
with other non-x86 architectures like RISC-V, but you likely find something
missing.

The reason I call it _superficial_ is that in many cases things do not work just
because Arm is not listed in the list of supported architectures. For example,
many packages on Flathub are not available on Arm just because they are marked
only for x86_64. Such things just work fine by adding Arm as a supported
architecture.

Some programs actually lack Arm support. Adding Arm support is usually not hard
in such a case either as _the hard part_ like porting runtimes with JIT
compilers is already done for Arm.

An emulator like [FEX](https://fex-emu.com/) help in unfortunate cases of binary
distribution.

### Lack of big.LITTLE support

It seems big.LITTLE confuses some programs that assume only one processor type
is supported and they need to be handled separately.

## What did not work for me on Asahi Linux and how I fixed them

As I described earlier, usually Asahi Linux just works. If something does not
work, well, it's a chance for an open-source contribution. This section
describes what didn't work and how I fixed them. The fixes are all applicable
for any Arm or big.LITTLE systems, meaning fixes for Asahi Linux actually
benefits the entire Arm community.

### KVM and big.LITTLE

KVM works fine if you pin the vCPU threads only to big or LITTLE cores; it
magically fails if you don't. Unfortunately, QEMU does not support pinning vCPU
threads to distinct cores[^2] so you can't use both big and LITTLE cores on a
VM at the same time.

It is somewhat unsafe to allow a vCPU thread to migrate between big and LITTLE
cores. An operating system often carries workarounds for cores and can be
confused by such migration. To limit the confusion, the register that identifies
the processor must be overridden. This requires extra caution and
[QEMU has lengthy documentation to describe safe configurations for x86](https://qemu.readthedocs.io/en/v8.0.0/system/qemu-cpu-models.html).
KVM on Arm64 does not support overriding the register, `MIDR` anyway[^3].
Nevertheless, I decided to use this _unsafe_ configuration because it should
work for me™.

When QEMU fails with big.LITTLE, the serial console outputs the following before
it shows GRUB menu:
~~~
Synchronous Exception at 0xXXXXXXXXXXXXXXXX
~~~

This corresponds to a line in
[EDK2](https://github.com/tianocore/edk2/blob/edk2-stable202305/ArmPkg/Library/DefaultExceptionHandlerLib/AArch64/DefaultExceptionHandler.c#L208)
so something in EDK2 should have gone wrong. To investigate further, I built
EDK2 with debug information and ran it to find that it results in an assertion
failure in
[the following function of `ArmPlatformPkg/PrePi/MainUniCore.c`](https://github.com/tianocore/edk2/blob/edk2-stable202305/ArmPlatformPkg/PrePi/MainUniCore.c):
~~~ C
VOID
EFIAPI
SecondaryMain (
  IN UINTN  MpId
  )
{
  ASSERT (FALSE);
}
~~~

This implies the following possibilities:
1. EDK2 wrongly used `MainUniCore.c` for QEMU.
2. QEMU wrongly started secondary processors.

I decided to investigate possibility 2. KVM should have some interface to start
or not to start vCPU so I looked at
[the kernel documentation](https://www.kernel.org/doc/html/v6.3/virt/kvm/api.html#kvm-arm-vcpu-init)
and found `KVM_ARM_VCPU_POWER_OFF` is passed to `KVM_ARM_VCPU_INIT` ioctl to
start vCPU is started powered off, and `KVM_SET_MP_STATE` ioctl is used to start
or stop vCPU during execution. I instrumented both ioctl calls and pinned QEMU
to big cores to see how they should work in an ideal state.

Surprisingly, QEMU didn't start secondary processors but used `KVM_SET_MP_STATE`
ioctl to make vCPU stopped instead of passing the `KVM_ARM_VCPU_POWER_OFF` flag
to `KVM_ARM_VCPU_INIT`. Anyway, I looked at the QEMU code to find what caused
the `KVM_SET_MP_STATE` ioctl, which led to the following lines:

~~~ C
for (cs = first_cpu; cs; cs = CPU_NEXT(cs)) {
    Object *cpuobj = OBJECT(cs);

    object_property_set_int(cpuobj, "psci-conduit", info->psci_conduit,
                            &error_abort);
    /*
     * Secondary CPUs start in PSCI powered-down state. Like the
     * code in do_cpu_reset(), we assume first_cpu is the primary
     * CPU.
     */
    if (cs != first_cpu) {
        object_property_set_bool(cpuobj, "start-powered-off", true,
                                 &error_abort);
    }
}
~~~

This proved possibility 2 is true. I then added `printf` before
`KVM_SET_MP_STATE` ioctl and ran QEMU without pinning vCPUs. Surprisingly the
ioctl was not called in this scenario.

Looking back at the execution path before QEMU reaches `KVM_SET_MP_STATE` ioctl,
I found the following lines at the end of the `kvm_arch_put_registers` function:

~~~ C
    if (!write_list_to_kvmstate(cpu, level)) {
        return -EINVAL;
    }

   /*
    * Setting VCPU events should be triggered after syncing the registers
    * to avoid overwriting potential changes made by KVM upon calling
    * KVM_SET_VCPU_EVENTS ioctl
    */
    ret = kvm_put_vcpu_events(cpu);
    if (ret) {
        return ret;
    }

    kvm_arm_sync_mpstate_to_kvm(cpu);

    return ret;
~~~

In a big.LITTLE environment, the `write_list_to_kvmstate` function fails when
writing `CCSIDR_EL1` register. This failure didn't result in a user-friendly
message because the value `kvm_arch_put_registers` was never checked.
[I submitted a fix to add error checks](https://patchew.org/QEMU/20221201102728.69751-1-akihiko.odaki@daynix.com/),
but this of course doesn't make it compatible with big.LITTLE.

`CCSIDR_EL1` is not a register, but is an array of registers whose element is
selected with the `CSSELR_EL1` register. Each element describes a level of
cache. It looks a bit awkward in KVM; KVM calls such a register as a demux
register and exposes each element as a separate register. KVM usually does not
list read-only registers with `KVM_GET_REG_LIST` ioctl, but somehow it lists
demux registers even though they are read-only and writing different values
fails. This makes QEMU try to restore the `CCSIDR_EL1` values as VM states.

As big and LITTLE cores have different cache configurations, they have different
`CCSIDR_EL1` values, and migrating a vCPU thread between big and LITTLE cores
make the values change. If QEMU tries to restore `CCSIDR_EL1` values before
migration, it conflicts with the current values of `CCSIDR_EL1` and results in
an error.

After discussing with upstream maintainers, I decided to write patches to
generate a common cache configuration that looks valid for any physical CPU.
[This patch is now merged and included since 6.3](https://patchwork.kernel.org/project/linux-arm-kernel/list/?series=711156&state=%2A&archive=both).
Now you can run guests on QEMU/KVM without pinning to big or LITTLE cores.

Note that this configuration is strictly not safe as described above. There are
room for improvement in both KVM and QEMU when it comes to big.LITTLE-like
configuration.
[Even Alder Lake, Intel's own big.LITTLE design is not supported well on QEMU/KVM](https://gitlab.com/qemu-project/qemu/-/issues/777).

### KVM and watchpoint

Now QEMU/KVM works without doing anything special on Apple Silicon. However,
when debugging igb I realized the watchpoint exception passes through the host
and reaches the guest when I put watchpoint. QEMU receives the notification of
the watchpoint hit, but the notification shows some random address as the cause
of the watchpoint exception. This prevents QEMU from looking up the watchpoint
and makes it think the exception is not from the watchpoint QEMU set and passes
it to the guest.

It turned out that KVM just forgot to retrieve the address triggering the
watchpoint. This bug was present for a year and 8 months; you cannot expect such
a bug will be found soon like x86 bugs. It is still always possible to fix a bug
by yourself.

[The fix is submitted to the upstream](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20230530024651.10014-1-akihiko.odaki@daynix.com/)
and will be backported to stable kernels soon.

### libvirt

The next thing I tried after QEMU is libvirt. Anything specific to architecture
should be handled by QEMU so I didn't expect it to fail, but it did fail. It
executes QEMU to probe its features, but this probing QEMU process failed to
start.

It was easy to debug as QEMU showed a nice error message this time. It said
`KVM_CREATE_VM` ioctl fails with `EINVAL`. This ioctl has a `type` parameter
that defines the configuration of the VM. When the `none` machine type, which is
a machine that _does nothing_, is used for feature probing, QEMU passes `0` for
the parameter, meaning the VM should be created with the default configuration.

The default configuration should work for anyone, but Apple Silicon was an
exception. The `type` parameter in KVM/arm64 has a field called `IPA_Bits`,
which denotes the physical address size of the guest.
[The kernel documentation describes it as follows:](https://www.kernel.org/doc/html/v6.3/virt/kvm/api.html#kvm-create-vm)

> ~~~
> The requested size (IPA_Bits) must be:
>
>  ==   =========================================================
>   0   Implies default size, 40bits (for backward compatibility)
>   N   Implies N bits, where N is a positive integer such that,
>       32 <= N <= Host_IPA_Limit
>  ==   =========================================================
>
> Host_IPA_Limit is the maximum possible value for IPA_Bits on the host and is
> dependent on the CPU capability and the kernel configuration. The limit can be
> retrieved using KVM_CAP_ARM_VM_IPA_SIZE of the KVM_CHECK_EXTENSION ioctl() at
> run-time.
> ~~~

`Host_IPA_Limit` is 36 bits on Apple Silicon, meaning the default size, 40 bits,
is not valid for the system. 32 bits should work for anyone so
[I submitted a patch to always specify 32 bits as `IPA_Bits` on Arm64](https://patchew.org/QEMU/20230109062259.79074-1-akihiko.odaki@daynix.com/).

The kernel should hide most details of the underlying architecture, and even
leaked details should be covered by QEMU, but sometimes there are still leaked
details and a hardware-independent component like libvirt can fail due to that.

### QEMU memory leak

Another bug I caught when using QEMU/KVM is a simple memory leak that always
happens in SMP configuration. It seemed nobody has tried LeakSanitizer on Arm64.
This was an easy one, and
[the fix is already upstream](https://gitlab.com/qemu-project/qemu/-/commit/ad5c6ddea327758daa9f0e6edd916be39dce7dca).

### lscpu

DPDK Test Suite uses lscpu to enumerate all cores and to identify them, but it
was not working properly on big.LITTLE system. lscpu shows something like the
following on big.LITTLE system:

~~~
Architecture:           aarch64
  CPU op-mode(s):       64-bit
  Byte Order:           Little Endian
CPU(s):                 8
  On-line CPU(s) list:  0-7
Vendor ID:              Apple
  Model name:           -
    Model:              0
    Thread(s) per core: 1
    Core(s) per socket: 4
    Socket(s):          1
    Stepping:           0x1
    CPU(s) scaling MHz: 25%
    CPU max MHz:        2424.0000
    CPU min MHz:        600.0000
    BogoMIPS:           48.00
    Flags:              fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint i8mm bf16 bti ecv
  Model name:           -
    Model:              0
    Thread(s) per core: 1
    Core(s) per socket: 4
    Socket(s):          1
    Stepping:           0x1
    CPU(s) scaling MHz: 21%
    CPU max MHz:        3204.0000
    CPU min MHz:        660.0000
    BogoMIPS:           48.00
    Flags:              fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint i8mm bf16 bti ecv
Caches (sum of all):    
  L1d:                  768 KiB (8 instances)
  L1i:                  1.3 MiB (8 instances)
  L2:                   20 MiB (2 instances)
Vulnerabilities:        
  Itlb multihit:        Not affected
  L1tf:                 Not affected
  Mds:                  Not affected
  Meltdown:             Not affected
  Mmio stale data:      Not affected
  Retbleed:             Not affected
  Spec store bypass:    Mitigation; Speculative Store Bypass disabled via prctl
  Spectre v1:           Mitigation; __user pointer sanitization
  Spectre v2:           Not affected
  Srbds:                Not affected
  Tsx async abort:      Not affected
~~~

Note that each CPU "model" has a field named "Socket(s)" and
"Core(s) per socket". lscpu assumes different CPU models are installed into
different sockets (or another larger unit if present) and tells how many sockets
each model occupies, but in reality, they are on the same socket, and the CPU
package on the socket has 8 cores in total.

What is more awkward is the output of `lscpu -p=CPU,Core,Socket`. This command
is expected to give unique identifiers for CPUs (threads), cores, and sockets.

~~~
# The following is the parsable format, which can be fed to other
# programs. Each different item in every column has an unique ID
# starting usually from zero.
# CPU,Core,Socket
0,0,0
1,1,0
2,2,0
3,3,0
4,0,0
5,1,0
6,2,0
7,3,0
~~~

Note that the same core number appears twice. It looks like this processor
implements SMT and a core has two threads, but in reality, the two threads are
in distinct cores. It is because the Core ID is assigned independently in each
CPU model type.
[I proposed a patch to decouple the topology information from CPU model type](https://lore.kernel.org/util-linux/5092a67c-0b86-134a-df77-433a6db10900@daynix.com/T/).

### Pixman on Arch Linux ARM

[I received a report that QEMU does not display correctly when emulating MorphOS on macOS on Apple Silicon](https://lore.kernel.org/qemu-devel/483662d9-2565-db44-0e19-fb9128f28bde@eik.bme.hu/).
I tried the emulation on Asahi Linux and its display output also had glitches so
I assumed it should be a bug common for Arm or Apple Silicon. Eventually, I
realized Pixman library is not working so I built it myself to find it
surprisingly works.

The fact is that Arch Linux ARM copied the script to build Pixman for x86 to
reuse it for Arm, and it was forgot to enable Arm-specific code. It didn't work
on macOS because
[the Arm-specific code was disabled due to incompatibility with the LLVM assembler](https://gitlab.freedesktop.org/pixman/pixman/-/issues/46).

I submitted a change to enable
[Arm-specific code for Arch Linux ARM](https://github.com/archlinuxarm/PKGBUILDs/pull/1985).
There is also
[a change proposed to make it compatible with LLVM assembler](https://gitlab.freedesktop.org/pixman/pixman/-/merge_requests/71).

## Conclusion

Asahi Linux works very well. Sometimes it doesn't when doing something
somewhat peculiar like using KVM, but it is often trivial to fix things even in
such a case. Perhaps it may not be recommended for everyone, but it's a nice
platform for a software developer.

[^1]: Except for various WebGL pages, which may require functionalities not
      implemented yet. Providing environment variable `LIBGL_ALWAYS_SOFTWARE=1`
      only for such applications will solve the problem while not damaging the
      performance of the entire system.

[^2]: And I don't know its performance implication. Please tell me if you know
      Linux scheduler internals and the answer. I know
      [crosvm](https://chromium.googlesource.com/chromiumos/platform/crosvm/)
      has an option for this.

[^3]: Hypervisor.framework on macOS does support this. In general,
      Hypervisor.framework just exposes everything to the VMM and the kernel
      behaves like a shim. This gives greater control to VMM but also hurts the
      performance so this architecture is not suited for systems like servers
      that KVM targets.
