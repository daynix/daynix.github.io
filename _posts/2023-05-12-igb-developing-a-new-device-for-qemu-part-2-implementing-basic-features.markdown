---
layout: post
title:  "igb: Developing a new device for QEMU (part 2: Implementing basic features)"
date:   2023-05-12 21:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

This is the second post of a walkthrough of new device emulator development for
QEMU.
[The first post](/2023/05/06/igb-developing-a-new-device-for-qemu-part-1-introduction.html)
briefly described the development process and device which we will discuss, igb,
an Intel NIC. This post details the first steps of the development: filling a
boilerplate and adding basic features necessary to get the device _work_ and to
establish the foundation of further development.

## Writing a boilerplate and adding basic features

QEMU provides infrastructure common for different devices. The first step of 
device emulator development is to write a boilerplate to utilize it. Then, you
can implement some basic features like MMIO register accesses.

There are three possible options when implementing a new device.
1. Extending the existing code for a similar device.
2. Copying and rewriting the existing code for a similar device.
3. Writing from scratch.

The first option is the easiest if the new device has small differences from an
existing device or it is a strict superset of one, but in the case of igb, there
are so many differences from the predecessor, e1000e, that we had to choose
option 2.

Even if there is no device that is particularly similar to the device you are
going to implement and you are going with option 3, it is still a good idea to
look for a device recently added that uses the same type of bus (e.g., PCIe) or
implements the same category of feature (e.g., NIC). This ensures that the new
device implementation follows the convention already established in the QEMU
code base.

The initial goal of the development is to get the device to _work_. Concretely,
this implies:
- The code can be built.
- The operating system can see and initialize the device.

If you choose option 2, the first thing to do after copying the code is to
rename C identifiers to make the copied code buildable. Once renaming the
identifiers is done, rename the device type name.

In the copied code, the type name is defined with the following line:

~~~ C
#define TYPE_IGB "e1000e"
~~~

We can simply replace it with `igb`.

~~~ C
#define TYPE_IGB "igb"
~~~

Before doing any more changes, we ensure that we can build the code and it
functions as a drop-in replacement for e1000e. First, write a simple command
line to run QEMU with e1000e.

~~~ shell
build/qemu-system-aarch64 -M virt -device e1000e,netdev=netdev -netdev user,id=netdev...
~~~

Make sure this command line works, and then replace `e1000e` with `igb`.

~~~ shell
build/qemu-system-aarch64 -M virt -device igb,netdev=netdev -netdev user,id=netdev...
~~~

Now the new device should be working 🎉

...but the operating system sees e1000e instead of igb because the actual code
is just simply copied from e1000e. Before making it function as igb, let's
check in this first version of the new device into Git so that you can always go
back to the buildable state. Implementing a device is complicated and it is
likely to have regressions during development; frequently committing to Git may
save hours in such a situation.

We can then change the identifiers exposed to the guest to make it function as
a new device. We have the following lines in the class initialization function
of the copied code.

~~~ C
    c->realize = igb_pci_realize;
    c->exit = igb_pci_uninit;
    c->vendor_id = PCI_VENDOR_ID_INTEL;
    c->device_id = E1000_DEV_ID_82574L;
    c->revision = 0;
    c->romfile = "efi-e1000e.rom";
    c->class_id = PCI_CLASS_NETWORK_ETHERNET;

    rc->phases.hold = igb_qdev_reset_hold;

    dc->desc = "Intel 82574L GbE Controller";
    dc->vmsd = &igb_vmstate;
~~~

Replace the PCI identifiers and other fields.

~~~ C
    c->realize = igb_pci_realize;
    c->exit = igb_pci_uninit;
    c->vendor_id = PCI_VENDOR_ID_INTEL;
    c->device_id = E1000_DEV_ID_82576;
    c->revision = 1;
    c->class_id = PCI_CLASS_NETWORK_ETHERNET;

    rc->phases.hold = igb_qdev_reset_hold;

    dc->desc = "Intel 82576 Gigabit Ethernet Controller";
    dc->vmsd = &igb_vmstate;
~~~

Now the guest thinks the device is igb, but the actual implementation is e1000e
so the guest cannot properly initialize it. In theory, you can implement all
features igb has according to
[the datasheet](https://www.intel.com/content/dam/www/public/us/en/documents/datasheets/82576eg-gbe-datasheet.pdf)
and get the working code, but it's unrealistic considering that the datasheet
has even _960 pages_. A practical approach is to read the device driver code and
implement features accessed during the initialization.

<figure style="text-align: center">
{% include 2023-05-12-igb-developing-a-new-device-for-qemu-part-2-implementing-basic-features/82576eg-gbe-datasheet.svg %}
<figcaption>Yes, this innocent-looking datasheet has 960 pages.</figcaption>
</figure>

Linux is a good target for the initial bring-up. Its drivers are open-source,
and, combined with QEMU, it can provide
[interactive debugging experience with gdb](https://www.kernel.org/doc/html/v6.3/dev-tools/gdb-kernel-debugging.html)
which is comparable with usual user-space development.

Once you get confidence that you implemented everything necessary for the
initialization, run the guest and debug it until it works. We'll cover the
details of debugging technique in a future post, but basically, normal
debugging tools like GDB work well with QEMU. QEMU also has
[a tracing feature](https://www.qemu.org/docs/master/devel/tracing.html) which
greatly helps debugging.

~~~
-trace e1000* -trace e1000e* -trace igb*
~~~

Once it works, check in the code again.

Fortunately, this step was already done by Gal Hammer and Marcel Apfelbaum so I
had working code when I started working on igb. Nevertheless, I performed this
step by myself again to understand the difference between igb and e1000e, and
to backport the recent changes made for e1000e to igb, which was already
diverged from e1000e.

## Adding QTest

[QTest](https://www.qemu.org/docs/master/devel/qtest.html) is a device emulation
testing framework and provides low-level access to the emulated device such as
MMIO access. While QTest does not run a real guest and is not suitable for
integration testing, it can cheaply test specific aspects of the device.

A recommended usage of QTest is to write tests for the basic features. As QTest
is ran by [the CI](https://www.qemu.org/docs/master/devel/ci.html), it can
effectively prevent fatal regressions after the code is merged into the
upstream. Of course, you can run it during development too: QTest is so fast
that you can run the test cases for a device always after compiling.

It is also possible to use the framework in a manner more like unit testing and
write tests for most behaviors. Some argue that low-level software components
are not suited for unit testing and while I believe otherwise, it is also true
that unit testing incurs significant cost. For igb, I deemed the gain from the
unit testing to be limited compared to the cost.

The process of writing QTest is not so different from writing the device
emulation itself: copy the existing code, rename identifiers, and tweak it for
the new device.

## Conclusion

With these steps finished, now we have:
- Buildable code
- Running Linux guest
- Basic QTest

These are the foundations of the further development. New features will be added
and more testing will be done in the future, but that cannot be done without
these essential components.

[The next post](/2023/05/21/igb-developing-a-new-device-for-qemu-part-3-implementing-basic-features.html)
will discuss implementing advanced features and libvirt support to make the
device practically _usable_.
