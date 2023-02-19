---
layout: post
title:  "Guest Windows debugging and crashdumping under QEMU/KVM: dump-guest-memory, vmcoreinfo and virtio-win"
date:   2023-02-19 23:15:00 +0300
author:
  name: "Viktor Prutyanov"
  url: "https://github.com/viktor-prutyanov"
---

In the previous [introductory article](/2023/01/20/Guest-Windows-debugging-and-crashdumping-under-QEMU-KVM-Introduction.html) we've discussed crash dump capturing and debugging of guest Windows with built-in tools. This post address a case when Windows can't produce the Complete Memory Dump during BSoD by itself or a live dump is needed. We will discuss QEMU's command `dump-guest-memory -w` which captures guest Windows dump in WinDbg-readable DMP format and saves it on the host side.

## Preparation

This method is applicable for all Windows versions from Windows Server 2012 to, at least, Windows Server 2022 (in other words, any of the versions supported by Microsoft as of 2023) for both 32-bit and 64-bit platforms. It requires two things:
* vmcoreinfo device on the QEMU side
* FwCfg driver on the guest side

### vmcoreinfo

If QEMU is run from the command line, add `-device vmcoreinfo` to argument list. In case of [libvirt](https://libvirt.org/formatdomain.html), add `<vmcoreinfo state='on'/>` to the XML config under `<features></features>`.

### virtio-win

Install FwCfg driver from [virtio-win](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/) to the guest Windows. **Note:** There is also [an old driver](https://github.com/virtio-win/kvm-guest-drivers-windows/tree/02.12.2021-last-buildable-point-Win7/fwcfg64) which doesn't support 32-bit platform, but supports deprecated Windows kernel versions starting from 6.1 (Windows 7 and Windows Server 2008 R2).

## Capturing crash or live dump

When either the system is still alive, or a BSoD occurs and the system does not reboot (this can be achieved using PVPanic device or unchecking "Automatically restart" option), the following command can be executed in QEMU monitor:
```
dump-guest-memory -w memory.dmp
```
The Complete Memory Dump will be written to `memory.dmp` file. Then it can be opened in WinDbg like any other dump.

## Typical issues

These are messages typically reported by `dump-guest-memory -w`:
* `win-dump: invalid vmcoreinfo note size`: most likely, guest driver is not running yet (too early stage of Windows kernel initiatization) or not present at all, so the dump can't be captured
* `win-dump: number of QEMU CPUs is bigger than NumberProcessors (%u) in guest Windows`: mostly likely, a desktop version of Windows is running and limits number of CPUs, but the dump can still be captured

## Internals

This section is dedicated to internals of the method discussed above.

### Anatomy of the Complete Memory Dump

The structure of the Complete Memory Dump consists of several large parts:

* A one-page (4 KiB) header on a 32-bit system and 2 pages (8 KiB) on a 64-bit system.
* Snapshots (so-called "runs") of contiguous regions of physical memory (Run #0 - Run #N).

![image](https://user-images.githubusercontent.com/8286747/218570184-a37f8799-1feb-4e95-9266-932ffb81c519.png)
<p align="center">Simplified scheme of the Complete Memory Dump</p>

The number of runs, start addresses, and lengths are stored in the header. The `dumpchk.exe` utility, which comes with the WinDbg debugger, is able to show information about the physical memory stored in the dump file. Usually there is more than one region due to PCI holes, because some of the physical address space is used for communication with peripherals and is not suitable for data storage. Therefore, not all of the physical address space is stored in the dump file. On 64-bit platform, each physical memory region is described in the dump header with the `_PHYSICAL_MEMORY_RUN64` structure, which stores the start and the length of the region:
```c
struct _PHYSICAL_MEMORY_RUN64 {
    ULONG64 BasePage;
    ULONG64 PageCount;
}
```

![image](https://user-images.githubusercontent.com/8286747/218570814-18b739fb-0875-4061-a758-c5a591107edf.png)
<p align="center">The list of physical memory regions is displayed by the <code>dumpchk.exe</code></p>

In addition to information about physical memory regions, the dump header contains other fields required for debugger operation, including:

* `BugcheckData` - error code and 4 parameters that describe the reason of the crash
* `RequiredDumpSpace` - total dump size in bytes
* `DirectoryTableBase` - the physical address of the root of the virtual-to-physical address translation for the debugger
* `PsLoadedModuleList` - the virtual address of the list of loaded executable modules
* `PfnDatabase` - virtual address of page frame number database
* `MinorVersion, MajorVersion` - two fields that together determine the version of Windows kernel
* `KdDebuggerDataBlock` - virtual address of Windows kernel structure, which stores information required for debugger (see below)

### KdDebuggerDataBlock

KdDebuggerDataBlock contains the addresses of another important kernel data structures and offsets within them, for example:
* `KernBase` - virtual address of the Windows kernel image loaded into memory (ntoskrnl.exe) which is required for WinDbg to download PDB symbols
* `KiProcessorBlock` is a pointer to an array of pointers to PRCBs (processor control block) where Windows stores the data on each processor in use
processor used
* `OffsetPrcbContext` - the offset of the structure inside PRCB, where Windows saves register context on crash
* `OwnerTag` - KdDebuggerDataBlock signature - ASCII characters `"KDBG"`.

The following fields contained in KdDebuggerDataBlock have their analogs in the dump header:
* `KiBugcheckData`
* `MmPfnDatabase`
* `PsLoadedModuleList`

The structure of the KdDDebuggerDataBlock contents is described in `_KDDEBUGGER_DATA64` structure in `wdbgexts.h`, which comes with WinDbg. With a new version of Windows, new fields are only added to end of this structure, and already existing fields remain at the same offsets, this can be relied on regardless of the version of Windows.

It is easy to see that header fields such as `RequiredDumpSpace` can be calculated and filled based on data from QEMU, but fields like `KdDebuggerDataBlock` and `PsLoadedModuleList` are known only to the guest kernel.

Unfortunately, practice shows that `KdDebuggerDataBlock` in modern versions of Windows may be encrypted by the system at boot time and be inaccessible during operation, and decrypted only during a crash. So we need a way to decrypt it, but let's come back to this problem a bit later.

### KeInitializeCrashDumpHeader

`KeInitializeCrashDumpHeader` is a [partially documented function](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-keinitializecrashdumphead) that driver calls to get the dump header. According to the documentation, the function returns a header that will be correct for the lifetime of the system, although it has the following limitations:
* If the amount of physical memory is changed, the header has to be retrieved again.
* The header received in this way does not contain data about the occurred exception (`BugcheckData`).

Also, according to the documentation, starting with Windows 8 the `DirectoryTableBase` address in the resulting header always corresponds to the system context, but for earlier versions the context will be the same as the context of the current process. It can also be user context, which may then prevent the debugger from accessing the system structures using the virtual addresses.

Furthermore, after analyzing of some number of such dumps, some more inconsistencies were found:
* `RequiredDumpSpace` is not filled
* `Context` structure is empty
* `PfnDatabase` field has wrong value

The `RequiredDumpSpace` value must be filled properly othewise WinDbg will not work properly. The filling of the `Context` structure does not affect the result, but the similar structures in the memory do. They are required by WinDbg to display register and call stack values. Some header values are duplicated in `KdDebuggerDataBlock`, so it can be used to restore them. The ways of restoring the header fields are gathered in the corresponding table:

| Header field      | How to fix                             |
|-------------------|----------------------------------------|
| `BugcheckData`      | Take structure pointed to by `KiBugcheckData` from `KdDebuggerDataBlock` in case of BSoD or set `BugcheckCode` to `0x161` in case of live dump |
| `PfnDatabase`       | Take `MmPfnDatabase` from `KdDebuggerDataBlock` |
| `RequiredDumpSpace` | Calculate from physical memory info |
| `Context`           | No fix needed |

### fw_cfg and vmcoreinfo

After the header is received, it must be passed to the host. The `vmcoreinfo` device is used to do this. The `vmcoreinfo` device is an add-on to the `fw_cfg` device.

`fw_cfg` is a virtual device [provided by QEMU](https://github.com/qemu/qemu/blob/master/docs/specs/fw_cfg.rst) that helps guest software to communicate with the host. From the guest point of view, it is a device with I/O ports `0x510-0x51B`. It provides access to an array of entries, which are simply blocks of arbitrary data and a string key associated with them.

`vmcoreinfo` is a virtual device [from QEMU]() accessed via a `fw_cfg` entry named `"etc/vmcoreinfo"`. The device was originally designed to [transfer some Linux kernel data](https://docs.kernel.org/admin-guide/kdump/vmcoreinfo.html) when creating guest Linux dumps.

To read from any `fw_cfg` entry, the driver does the following:
1. Sends the number of the desired entry to port `0x510`
2. Reads the data from port `0x511` byte-by-byte

To write data to `vmcoreinfo`, we first need to find the entry by its name (`"etc/vmcoreinfo"`) and check that data transfer from the guest to the host is possible:
1. Read 4-byte entry with number `0x19` to get total number of entries
2. Read entries until the desired name is found
3. Read 8 bytes from `0x514-0x51B` and check against `"QEMU CFG"` - this means that `fw_cfg` supports DMA-like writing interface

In case of VMCoreInfo a packed structure of the following type is passed:
```c
  struct FWCfgVMCoreInfo {
      uint16_t host_format;  /* formats host supports */
      uint16_t guest_format; /* format guest supplies */
      uint32_t size;         /* size of vmcoreinfo region */
      uint64_t paddr;        /* physical address of vmcoreinfo region */
  } QEMU_PACKED;
```
Data transfer to host is done through a DMA-like interface:
1. Write physical address of `FWCfgDmaAccess` structure containing the physical address of the data to be written, size, entry number and control bits to `0x514-0x51B`
2. Check control bits, if they are all equal to 0, then the data
has been successfully transferred

![image](https://user-images.githubusercontent.com/8286747/218884764-710e70b2-e79d-4d38-b72b-e2238b89e5e2.png)

After this, QEMU will have access to the data at `paddr` address. QEMU interprets them as an ELF Note section, which consists of a name (in this case `"VMCOREINFO"`) and content - the dump header. If the structure of the section is correct, it becomes available to the [QEMU dumping subsystem](https://github.com/qemu/qemu/blob/master/dump/dump.c).

All the driver logic described above is implemented [here](https://github.com/virtio-win/kvm-guest-drivers-windows/blob/master/fwcfg64/fwcfg.c).

### Register context

In the debug dump analysis process, the register values are important in and of themselves, moreover, their values are necessary for the correct reconstruction of the call stack.

In the `wdm.h` header file, which is delivered as part of the WDK, there is a definition of a structure called `CONTEXT`. After comparing this definition, the register values from QEMU and `Context` field from the saved Windows dump, it is clear that `Context` field contains an implementation of this structure. But as on modern systems there is more than one processor, register contexts of all processors can't fit into dump header (there is only space for one `CONTEXT` instance).

An address space of Windows kernel contains per-CPU `PRCB` structures with `ContextFrame` fields. Each of these fields stores an address of a context frame from a corresponding CPU. If these structures are filled with zeroes, WinDbg cannot recover the context. In addition, the context structure contains a field of flags, one of which indicates that the context is 64-bit, so when the structure is zeroed, the debugger displays a message that only the 32-bit context is available. Once the context structures are filled in, these messages do not occur. So WinDbg definetely takes the contents of registers from these structures.

Thus, in order for WinDbg to retrieve the actual register values, they must be stored in the corresponding structures inside the memory dump. This procedure is implemeted in `patch_and_save_context` function from [`dump/win_dump.c`](https://github.com/qemu/qemu/blob/master/dump/win_dump.c).

### KiBugcheckData

`BugcheckData` is automatically saved at BSoD and this data can be simply copied to the header. But it turns out that when creating a live system dump, it is not enough to write BugcheckData to the header. In order for the debugger to use this data, it must also be saved in `KdDebuggerDataBlock->KiBugcheckData`. This logic is implemented in `patch_bugcheck_data` routine from [`dump/win_dump.c`](https://github.com/qemu/qemu/blob/master/dump/win_dump.c).

### Small Memory Dump

Along with a full memory dump, there is a Small Memory Dump. For example, Windows creates a dump in this format after a BSoD at default settings. Such dump contains the following data:
* `BugcheckData` structure
* `EPROCESS` structure with an information about the faulty process
* `ETHREAD` structure with an information about the faulty thread
* `PRCB` structure for the CPU on which the error occured
* List of loaded modules
* `KdDebuggerDataBlock` structure

Unlike the Complete Memory Dump, `KdDebuggerDataBlock` is stored in the Small Memory Dump at the offset written in its header, so the address translation is not required. Thus, if we have a small dump, we can use it to create a full memory dump.

### KeCapturePersistentThreadState

`KeCapturePersistentThreadState` is an undocumented Windows kernel function. The driver can use it to retrieve a Small Memory Dump.
`KdDebuggerDataBlock` inside such dump will be decrypted. The guest driver can pass to the hypervisor not only the address of the original `KdDDebuggerDataBlock` (which can be encrypted), obtained with `KeCapturePersistentThreadState`, but also the address of its decrypted version, which will be stored in the driver's memory.

In order for the hypervisor to take advantage of this feature, it must check the signature of the `KdDebuggerDataBlock` whose address lies in its usual place in the header and, if the signature does not match, use the `KdDebuggerDataBlock` whose address is passed by the driver through one of the unused header fields, such as `BugcheckParameter1`, since this field has a null value anyway and must be filled in by the hypervisor. This is what QEMU does in `check_kdbg` from [`dump/win_dump.c`](https://github.com/qemu/qemu/blob/master/dump/win_dump.c).

![image](https://user-images.githubusercontent.com/8286747/219972209-dceb26e6-914f-4db2-be60-cc7e0d2e7e8a.png)
<p align="center">Connection between dump elements at the stage of their loading from the guest memory</p>

### Overall host-side algorithm

1. Pause the VM
2. Synchronize state with KVM
3. Take the header from the `vmcoreinfo` device
4. Calculate `RequiredDumpSpace` field value as the sum of the sizes of continuous physical memory regions described in the header
3. Use `DirectoryTableBase` field value from the header as `CR3` register value when further accessing the guest OS virtual address space from QEMU
4. Take `KdDebuggerDataBlock` structure address from the header
5. Substitute `PfnDatabase` field value with the `KdDebuggerDataBlock->MmPfnDatabase` value
6. Fill `BugcheckData` structure (error code and parameters)
* In the case of the BSoD, take the content from `KdDebuggerDataBlock->KiBugcheckData`
* In the case of the live system dump, write `0x161` (`LIVE_SYSTEM_DUMP`) code and zero error parameters to `KdDebuggerDataBlock->KiBugcheckData`
7. Write the register context to `KdDebuggerDataBlock->KiProcessorBlock[i]->ContextFrame` for each `i` in a range of CPU numbers, based on the registers from QEMU
8. Write header to the file
9. Write regions of the guest physical memory to the file
10. Unpause the VM

## Conclusion

We have discussed the usage and internals of `dump-guest-memory -w` command which is a useful tool for anyone looking to debug their Windows guests in the QEMU/KVM environment. The next post will be devoted to creating a dump with literally no action on the guest side.
