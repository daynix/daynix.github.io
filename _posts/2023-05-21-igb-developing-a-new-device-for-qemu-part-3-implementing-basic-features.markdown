---
layout: post
title:  "igb: Developing a new device for QEMU (part 3: Implementing advanced features)"
date:   2023-05-21 18:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

This is the third post of a walkthrough of new device emulator development for
QEMU which uses [igb](https://qemu.readthedocs.io/en/v8.0.0/system/devices/igb.html),
a recently introduced Intel NIC emulation as an example.
[The first post](/2023/05/06/igb-developing-a-new-device-for-qemu-part-1-introduction.html)
roughly described there are several development steps:

1. Writing a boilerplate and adding basic features
2. Adding QTest
3. Adding advanced features and libvirt support
4. Running more tests and increasing feature coverage
5. Debugging
6. Contributing to the upstream

This post discusses 3 and 4. The goal is to make the device implementation,
which was introduced in
[the previous post](/2023/05/12/igb-developing-a-new-device-for-qemu-part-2-implementing-basic-features.html),
_usable_ for practical purposes.

## Adding advanced features and libvirt support

### Determining test workloads

The previous post discussed bringing up the first device implementation and
letting the guest initialize the device. In the case of igb, it is not very
difficult to bring features implemented for e1000e, the predecessor in Intel's
NIC product line, to igb as these devices are very similar, and igb development
started by forking e1000e's code. However, this initial state of the device
emulation is not really usable; you would expect the new device will bring
features not present in the existing devices.

Therefore, the next step of the device development will be to implement such
advanced features. As described in the first post, the most notable features of
igb are VMDq and SR-IOV. VMDq implements L2 switching among (nested) VMs, and
SR-IOV allows to attach some tx/rx queues as a _virtual function_ to a VM.
The first thing to do when implementing a new feature is to fabricate a workload
that will exercise it. To exercise VMDq and SR-IOV features, we have to perform
the following operations on a Linux guest:
- Enabling SR-IOV virtual functions
- Sending and receiving packets with different combinations of peers:
  - The physical function and an external host
  - The physical function and a virtual function
  - A virtual function and an external host
  - A virtual function and the physical function
  - A virtual function and another virtual function

The physical function is the PCI function always available. Testing the
various combinations of peers is important to ensure the L2 switching capability
VMDq provides correctly functions.

Once the workloads were determined, you can test it and implement missing
features and fix bugs for them, just like the basic features are tested and
fixed. This process is _incremental_ so it is important to prevent regressions
and to ensure forward progress. It is better to continuously run qtest and
simple workloads like ping and curl. Commit changes to Git frequently so that
you can isolate regressions by e.g., running
[git bisect](https://git-scm.com/docs/git-bisect).

Note that
__it is _not_ necessary to complete this step before sending the new device implementation to the upstream__.
It is actually often _better_ to send the patches upstream as soon as you get
confidence in the implementation of the basic functionalities. A small patch
that implements the basics is easier to review than a long patch that includes
many advanced features. The upstream reviewers can also discuss the design of
the implementation early and you can correct it before writing more changes if
you send patches early. It is even possible to send patches before you
thoroughly test them as
[RFC](https://qemu.readthedocs.io/en/v8.0.0/devel/submitting-a-patch.html#use-the-rfc-tag-if-needed).

For igb, I sent the first version of patches when VMDq and SR-IOV features
become usable on Linux. The other target platforms, Windows and DPDK followed
later.

### libvirt support

libvirt is a very popular frontend for QEMU so we need it to have the support
for the new device to some extent at least. The details of the code change
vary by the type of the device, but the principle same with QEMU applies:
find a similar device, copy the code for it, and extend it if necessary.

libvirt often requires few changes for a new device. The libvirt change for igb
is just a copy of e1000e support, and you can see it at:
[https://gitlab.com/akihiko.odaki/libvirt/-/commit/f0e85eed7398c35b77a775d8178edc23b757ae6d](https://gitlab.com/akihiko.odaki/libvirt/-/commit/f0e85eed7398c35b77a775d8178edc23b757ae6d)

## Running more tests and increasing feature coverage

As you add more and more features, it gets harder to continuously test all of
the implemented features. Automated testing is essential to keep various
features working.

It is not common that there are tests for the emulated device as hardware
vendors do not release tests they presumably created during the hardware
development. It is theoretically possible to develop a complete unit testing
suite that covers all the features, but in practice, it consumes too much time.

A more practical way to introduce automated testing is to borrow existing test
suites for drivers and platforms. As they target more high-level constructs,
sometimes they do not suit well for device testing. For example, such a test
suite may include tests for a pure software construct, which costs time and
contributes nothing to device development. Such a test suite neither tests the
device functionalities the platform does not use. Nevertheless, high-level tests
are pragmatically very useful as they can ensure the success of the ultimate
objective: to get the device to work for the target platform.

Usually, there are test suites specialized for a device type and an operating
system so you have to choose appropriate ones. Such test suites for NICs and
popular platforms will be presented below:

### ethtool

[ethtool](https://mirrors.edge.kernel.org/pub/software/network/ethtool/) is a
Linux utility for controlling network devices. The kernel exposes an ioctl
that triggers device tests when requested by ethtool. The coverage of the
tests is fairly limited but it is so easy to use that it is automated for igb
using Avocado testing framework. The automating script is available at:
[https://gitlab.com/qemu-project/qemu/-/blob/v8.0.0/tests/avocado/netdev-ethtool.py](https://gitlab.com/qemu-project/qemu/-/blob/v8.0.0/tests/avocado/netdev-ethtool.py)

### Linux Test Project (LTP)

[Its README](https://github.com/linux-test-project/ltp/blob/20230516/README.md)
describes itself as follows:
> The LTP testsuite contains a collection of tools for testing the Linux kernel
> and related features.

As the README states, LTP is not specific to networking and contains various
tests for Linux.
[The network tests have their own README](https://github.com/linux-test-project/ltp/blob/20230516/testcases/network/README.md).

The main advantage of LTP is that it contains various kinds of tests from ping
to applications like HTTP, FTP, telnet, etc. The downside is that its network
tests are somewhat unmaintained, and require manual setup of network
applications.

I have sent patches to
[fix the tests for a modern system](https://github.com/linux-test-project/ltp/commits?author=akihikodaki),
and wrote [some scripts to manage VMs](https://github.com/akihikodaki/q). These
scripts are particularly helpful as they allow us to easily reproduce the model
system for manual testing/debugging and to automatically run the LTP. The
automated scripts test all of the combinations of peers described above.

As Linux was the first target platform, the LTP was the first test suite igb
passes. However, later changes for the other platforms sometimes regressed the
Linux support and automated LTP runs greatly helped to spot such regressions.

### Windows Hardware Lab Kit (HLK)

> The Windows Hardware Lab Kit (Windows HLK) is a test framework used to test
> hardware devices and drivers for Windows 11, Windows 10 and all versions of
> Windows Server starting with Windows Server 2016.

[Windows Hardware Lab Kit \| Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/test/hlk/)

Windows HLK includes comprehensive tests for network devices and exercises more
advanced features like VMDq and SR-IOV. We maintain
[AutoHCK](https://github.com/HCK-CI/AutoHCK), a tool for automating HLK runs
on QEMU so it is quite easy to set up VMs using it. Windows HLK spotted several
bugs which were missed by the LTP in the case of igb.

The downside of Windows HLK, or rather Windows platform is that it is
closed-source. When I first tried igb with Windows, its driver just hung and
repeated resets without any helpful information. You can only see traces from
QEMU and the disassembly of the driver. The next post will discuss debugging
with this closed-source platform, but you will want to get the device to work
with an open-source platform before testing with Windows.

### DPDK Test Suite (DTS)

DPDK is not an operating system like Linux and Windows, but an application
framework for low-level packet processing. DPDK implements its own driver to
implement the interaction with the hardware in the userspace and to eliminate
context switches with the kernel. The DPDK driver needs to be tested as it
employs low-level hardware access as operating system drivers do.

The nice thing with DPDK is that
_it can be debugged as a normal user-space application_. It can also use more
minor device features, and DPDK Test Suite covers details of such features.
However, DPDK is not for a user-facing application like curl so DTS cannot
ensure the usability of the device emulation in common situations.

## Conclusion

This post discussed how to add advanced features and libvirt support, and to
test them. This part of device development takes a significantly long time so
you need some strategy. You have to choose features and tests appropriate for
the current development stage and sometimes send (RFC) patches to get design
reviews.

The development process is not always smooth, and you may have difficulty
getting a new feature to function, or the device implementation may suffer from
regressions. Therefore
[the next post](/2023/06/01/igb-developing-a-new-device-for-qemu-part-4-debugging-and-submitting-patches.html)
will mostly be spent on one thing: __debugging__.
