---
layout: post
title:  "igb: Developing a new device for QEMU (part 1: Introduction)"
date:   2023-05-06 18:00:00 +0900
author:
  name: Akihiko Odaki
  url: https://github.com/akihikodaki
---

_[QEMU](https://www.qemu.org/)_ is a machine emulator and virtualizer; it
emulates or virtualizes various devices which consist in a machine. Such
emulated/virtualized devices include but are not limited to CPU, memory,
storage, bus, serial console, and network interface. QEMU is under active
development, and there is always demand for new device implementations. However,
the journey of new device implementation is complex and not always smooth.

This is a series of four posts and presents challenges and their resolutions
by describing the process of the development of _igb_, a device implementation
we recently contributed to the project. While igb emulates a
_PCI Express (PCIe)_ network interface card (NIC) and this series will
illustrate some aspects of PCIe and NIC, many parts of it should be generally
applicable for new device emulator development.

\* By the way, for igb development, I used a
_[M2 MacBook Air](https://www.apple.com/macbook-air-m2/)_, which contains
_Apple Silicon_ and runs _[Asahi Linux](https://asahilinux.org/)_, a community
port of Linux for this system. While there are some pitfalls, it has decent KVM
support and provides a nice environment for QEMU development.

## QEMU device emulator development in a nutshell

Briefly speaking, developing a new QEMU device emulator involves the following
steps:

1. Writing a boilerplate and adding basic features
2. Adding QTest
3. Adding advanced features and libvirt support
4. Running more tests and increasing feature coverage
5. Debugging
6. Contributing to the upstream

Writing a boilerplate is necessary to integrate the new device into QEMU. After
completing the boilerplate, basic features need to be added so that the guest
can access more advanced features.

_[qtest](https://qemu.readthedocs.io/en/v7.2.0/devel/qtest.html)_ is QEMU's
testing framework and useful to ensure there is no regression in essential
features.

With basic features covered, more advanced but non-essential features can be
added. _[libvirt](https://libvirt.org/)_ is a popular frontend of QEMU, and it
is important for usability to make the device and its features accessible from
libvirt.

During the development, tests will reveal regressions or features which are
missing but necessary for a workload. It is often unclear what is making tests
fail and a significant debugging effort will be required.

Finally, the code will be sent upstream. By upstreaming, you can ensure that the
code will be updated as the other parts of the code base change, and others can
appreciate the contributed code.

This series details each step and provides insight into how a device emulator
can be developed.

This next post will describe steps 1 and 2. The third post will describe steps 3
and 4. The last post will be about steps 5 and 6.

## What is igb?

Before describing the device development process in depth, first, let me
introduce the hardware which will be emulated by igb.

igb is actually the name of a family of Intel's NIC. In particular, QEMU's igb
implementation emulates
[82576](https://ark.intel.com/content/www/us/en/ark/products/37166/intel-82576eb-gigabit-ethernet-controller.html),
launched in 2008.

QEMU already has emulation code for several Intel NICs. The following device
implementations are present in the current QEMU (from oldest hardware to newest one):
- i82596
- eepro100 for 8255x
- e1000 for 8254x
- e1000e for 82574L

igb succeeds _e1000e_ in Intel's product line. As the name suggests, e1000 and
newer devices have gigabit bandwidth.

While paravirtualization is common these days, igb emulates actual hardware.
In most cases, a para-virtualized device is superior to emulated hardware, but
there are a few advantages emulated hardware has:
- Widely available driver. In particular, Windows bundles the driver for igb and
  works with it out-of-box.
- Feature-richness. Particularly features which make little sense for software
  implementation may be absent in a para-virtualized device, but having such
  features emulated may be required for testing purposes.

The latter is the major motivation for introducing igb. igb has so many features
added since e1000e so the later part of this section lists particularly notable
features of igb.

### VMDq

_[VMDq](https://www.intel.com/content/www/us/en/products/docs/network-io/ethernet/network-adapters/io-acceleration-technology-vmdq.html)_
is a feature to assign _queues_ to different virtual machines. (_nested_ virtual
machine in this case as the underlying system with igb is emulated/virtualized).
A network interface typically has a "queue" for each of tx and rx, and the
software puts a packet to transmit to the Tx queue and retrieves the received
packet from the rx queue.

e1000e has two queues for each of tx and rx to avoid contention of queues on a
multi-processor system. For tx, two processors can submit packets to transmit
at the same time by using different queues. For rx, e1000e can assign packets
from different peers to different queues; this feature is called RSS (receive
side scaling). Each queue can be processed in parallel by two
processors, improving the throughput.

igb has 16 queues for each of tx and rx. These queues can be used for effective
use of multiple processors as in e1000e, but igb also allows to use the queues
for efficient isolation of virtual machines. This isolation is achieved by
hardware-level L2 switching.

In a conventional system, the hypervisor emulates a NIC for each VMs and
performs L2 switching at the software level to route packets to the appropriate
VM. For example, the diagram below illustrates the case when VMs are receiving
packets from an external host:

{% comment %}
~~~ mermaid
flowchart BT
    EmulatedNIC0 --> VM0[VM 0]
    EmulatedNIC1 --> VM1[VM 1]

    subgraph Hypervisor
        Bridge --> EmulatedNIC0[Emulated NIC 0]
        Bridge --> EmulatedNIC1[Emulated NIC 1]
    end

    subgraph igb
        Queue0 --> Bridge["L2 Switch (Bridge)"]
        Queue1 --> Bridge
        Queue2 --> Bridge
        Queue3 --> Bridge
        RSS --> Queue0[Queue 0]
        RSS --> Queue1[Queue 1]
        RSS --> Queue2[Queue 2]
        RSS --> Queue3[Queue 3]
    end

    ExternalHost[External Host] --> RSS{RSS}
~~~
{% endcomment %}
{% include 2023-05-06-igb-developing-a-new-device-for-qemu-part-1-introduction/conventional.svg %}

In this diagram, packets are first stored in queues determined with RSS. The
hypervisor acquires these stored packets and performs L2 switching to route them
to appropriate VMs. This switching imposes a substantial load on the host.

With VMDq enabled, igb automatically performs this switching and assigns packets
to different queues accordingly, reducing the load of the host:

{% comment %}
~~~ mermaid
flowchart BT
    EmulatedNIC0 --> VM0[VM 0]
    EmulatedNIC1 --> VM1[VM 1]

    subgraph Hypervisor
        EmulatedNIC0
        EmulatedNIC1
    end

    subgraph igb
        Queue0 --> EmulatedNIC0[Emulated NIC 0]
        Queue1 --> EmulatedNIC0[Emulated NIC 0]
        Queue2 --> EmulatedNIC1[Emulated NIC 1]
        Queue3 --> EmulatedNIC1[Emulated NIC 1]
        RSS0 --> Queue0[Queue 0]
        RSS0 --> Queue1[Queue 1]
        RSS1 --> Queue2[Queue 2]
        RSS1 --> Queue3[Queue 3]
        L2Switch --> RSS0{RSS}
        L2Switch --> RSS1{RSS}
    end

    ExternalHost[External Host] --> L2Switch{L2 Switch}
~~~
{% endcomment %}
{% include 2023-05-06-igb-developing-a-new-device-for-qemu-part-1-introduction/vmdq.svg %}

This L2 switching is achieved with the packet filtering capability igb has.
e1000e can filter packets by the destination MAC address, and igb extends this
feature further by allowing the driver to configure different filters for
queues.

### SR-IOV

VMDq is more effective when combined with igb's _SR-IOV_ feature. SR-IOV is
an extension of PCIe to allow a device to export several _virtual functions_
which looks like different devices for the operating system. Virtual functions
can be directly assigned to VMs (by using
[VFIO](https://docs.kernel.org/driver-api/vfio.html) in the case of Linux),
which removes the hypervisor from the traffic path between the VM and hardware:

{% comment %}
~~~ mermaid
flowchart BT
    VF0 --> VM0[VM 0]
    VF1 --> VM1[VM 1]

    subgraph igb
        Queue0 --> VF0[VF 0]
        Queue1 --> VF0[VF 0]
        Queue2 --> VF1[VF 1]
        Queue3 --> VF1[VF 1]
        RSS0 --> Queue0[Queue 0]
        RSS0 --> Queue1[Queue 1]
        RSS1 --> Queue2[Queue 2]
        RSS1 --> Queue3[Queue 3]
        L2Switch --> RSS0{RSS}
        L2Switch --> RSS1{RSS}
    end

    ExternalHost[External Host] --> L2Switch{L2 Switch}
~~~
{% endcomment %}
{% include 2023-05-06-igb-developing-a-new-device-for-qemu-part-1-introduction/sriov.svg %}

The context switch between the hypervisor and VM incurs significant overhead so
bypassing the hypervisor can result in a great performance gain.

## Summary

This article introduced QEMU device emulator development in general and igb,
QEMU's new NIC emulation.
[The next post](/2023/05/12/igb-developing-a-new-device-for-qemu-part-2-implementing-basic-features.html)
will discuss the first step of a new device emulator development: writing a
boilerplate and adding basic features. The goal of this step is to get the
emulated device to work (even though the functionality may be limited) and to
create the foundation for further development.
