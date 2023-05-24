---
layout: post
title:  "Guest Windows debugging and crashdumping under QEMU/KVM: elf2dmp"
date:   2023-05-23 11:30:00 +0300
author:
  name: "Viktor Prutyanov"
  url: "https://github.com/viktor-prutyanov"
---

In the [previous post](/2023/02/19/Guest-Windows-debugging-and-crashdumping-under-QEMU-KVM-dump-guest-memory-vmcoreinfo-and-virtio-win.html) we discussed capturing a guest Windows crash dump using the vmcoreinfo device, the virtio-win driver called FwCfg, and the QEMU command called `dump-guest-memory -w`. In contrast to that method, this post considers a method to create a Complete Memory Dump of the 64-bit Windows guest, running inside a QEMU/KVM virtual machine, which doesn't require any actions inside the guest. We will discuss the `elf2dmp` tool which can convert QEMU ELF dump obtained with `dump-guest-memory` to WinDbg-readable DMP format.

## Preparation

### Obtaining elf2dmp

The `elf2dmp` tool is a standalone executable, but its source code is [a part of the QEMU project](https://gitlab.com/qemu-project/qemu/-/tree/master/contrib/elf2dmp) and resides in `contrib/elf2dmp`. For building along with QEMU, elf2dmp requires `tools` to be enabled in `configure` and depends on `libcurl`.

Also, `elf2dmp` can be obtained through package manager. For example, `qemu-tools` package contains `elf2dmp` on Fedora.

### Capturing ELF dump

QEMU has a `dump-guest-memory` command to dump guest memory into an ELF file. In contrast to `dump-guest-memory -w`, no complex work will be done other than writing a snapshot of physical memory and register contexts to the ELF file.

## Converting ELF to DMP

Run the following command to convert ELF dump file to DMP dump file:

```
elf2dmp memory.elf memory.dmp
```

The example output for VM with Windows Server 2022 and 4 CPUs:
```
4 CPU states has been found
CPU #0 CR3 is 0x000000012b109002
CPU #0 IDT is at 0xfffff8007d545000
CPU #0 IDT[0] -> 0xfffff80081e88100
Searching kernel downwards from 0xfffff80081e88000...
Data directory entry #0: RVA = 0x0000c000
Data directory entry #0: RVA = 0x00008000
Data directory entry #0: RVA = 0x00108000
Data directory entry #0: RVA = 0x00024000
Data directory entry #0: RVA = 0x0005b000
Data directory entry #0: RVA = 0x00065000
Data directory entry #0: RVA = 0x00007000
Data directory entry #0: RVA = 0x000190b0
Data directory entry #0: RVA = 0x0000e000
Data directory entry #0: RVA = 0x00022000
Data directory entry #0: RVA = 0x00063000
Data directory entry #0: RVA = 0x00012000
Data directory entry #0: RVA = 0x000356e0
Data directory entry #0: RVA = 0x0013a000
KernBase = 0xfffff80081400000, signature is 'MZ'
Data directory entry #6: RVA = 0x000154b8
CodeView signature is 'RSDS'
PDB name is 'ntkrnlmp.pdb', 'ntkrnlmp.pdb' expected
PDB URL is https://msdl.microsoft.com/download/symbols/ntkrnlmp.pdb/adc00fa5fc34456ba16e2687457240991/ntkrnlmp.pdb
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 11.5M  100 11.5M    0     0  1046k      0  0:00:11  0:00:11 --:--:-- 1674k
KdDebuggerDataBlock: 0x0000000000c00000(24:'.data') + 0x00000a30 = 0x000c00a30
KdDebuggerDataBlock = 0xfffff80082000a30
KdVersionBlock: 0x0000000000c00000(24:'.data') + 0x00015508 = 0x000c15508
KdVersionBlock = 0xfffff80082015508
KiWaitNever: 0x0000000000d0f000(25:'ALMOSTRO') + 0x00001bd0 = 0x000d10bd0
KiWaitNever = 0xfffff80082110bd0
KiWaitAlways: 0x0000000000d0f000(25:'ALMOSTRO') + 0x00001e38 = 0x000d10e38
KiWaitAlways = 0xfffff80082110e38
KdpDataBlockEncoded: 0x0000000000c00000(24:'.data') + 0x0010b148 = 0x000d0b148
KdpDataBlockEncoded = 0xfffff8008210b148
[KiWaitNever] = 0x1a5b030053af49fd
[KiWaitAlways] = 0x0029d7b15ae2d898
Decoding KDBG header...
Owner tag is 'KDBG'
Decoding KdDebuggerDataBlock...
Filling context for CPU #0...
Filling context for CPU #1...
Filling context for CPU #2...
Filling context for CPU #3...
Writing header to file...
Writing block #0/13 to file...
Writing block #1/13 to file...
Writing block #2/13 to file...
Writing block #3/13 to file...
Writing block #4/13 to file...
Writing block #5/13 to file...
Writing block #6/13 to file...
Writing block #7/13 to file...
Writing block #8/13 to file...
Writing block #9/13 to file...
Writing block #10/13 to file...
Writing block #11/13 to file...
Writing block #12/13 to file...
```

Since `elf2dmp` downloads the PDB file from Microsoft public symbol server, **the network access is required**. Also, both read and write accesses to the ELF dump file is required, although **the ELF file is guaranteed to remain unchanged**.

After the conversion, the DMP file can be opened in WinDbg.

## Typical issues

QEMU generates ELF dump with `0400` rights, but `elf2dmp` requires read and write accesses, so the file rights should be modified to give these permissions to the user. Otherwise `elf2dmp` will say something like that:
```
Failed to map ELF dump file 'memory.elf'
Failed to initialize QEMU ELF dump
```

## Internals

As described in the previous posts in this series, Complete Memory Dump consists of a header and a snapshot of the physical memory of the system (which contains, among other things, the register contexts for all CPUs). The physical memory and register contexts are taken from the ELF dump file and saved to the DMP file, just like `dump-guest-memory -w` does, so the main challenge for elf2dmp is to fill the dump header correctly and decrypt the `KdDebuggerDataBlock` structure.

### ELF

ELF dump file produced by QEMU with `dump-guest-memory` consists of a `PT_NOTE` with registers for each virtual CPU and several `PT_LOAD` with physical memory snapshots: 

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              CORE (Core file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         14
  Size of section headers:           0 (bytes)
  Number of section headers:         0
  Section header string table index: 0

There are no sections in this file.

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  NOTE           0x0000000000000350 0x0000000000000000 0x0000000000000000
                 0x0000000000000cc0 0x0000000000000cc0         0x0
  LOAD           0x0000000000001010 0x0000000000000000 0x0000000000000000
                 0x0000000000018000 0x0000000000018000         0x0
  LOAD           0x0000000000019010 0x0000000000018000 0x0000000000018000
                 0x0000000000001000 0x0000000000001000         0x0
  LOAD           0x000000000001a010 0x0000000000019000 0x0000000000019000
                 0x0000000000001000 0x0000000000001000         0x0
  LOAD           0x000000000001b010 0x000000000001a000 0x000000000001a000
                 0x0000000000001000 0x0000000000001000         0x0
  LOAD           0x000000000001c010 0x000000000001b000 0x000000000001b000
                 0x0000000000001000 0x0000000000001000         0x0
  LOAD           0x000000000001d010 0x000000000001c000 0x000000000001c000
                 0x0000000000084000 0x0000000000084000         0x0
  LOAD           0x00000000000a1010 0x00000000000a0000 0x00000000000a0000
                 0x0000000000010000 0x0000000000010000         0x0
  LOAD           0x00000000000b1010 0x00000000000c0000 0x00000000000c0000
                 0x0000000000004000 0x0000000000004000         0x0
  LOAD           0x00000000000b5010 0x00000000000c4000 0x00000000000c4000
                 0x000000000001c000 0x000000000001c000         0x0
  LOAD           0x00000000000d1010 0x00000000000e0000 0x00000000000e0000
                 0x0000000000020000 0x0000000000020000         0x0
  LOAD           0x00000000000f1010 0x0000000000100000 0x0000000000100000
                 0x000000007ff00000 0x000000007ff00000         0x0
  LOAD           0x000000007fff1010 0x00000000c0000000 0x00000000c0000000
                 0x0000000001000000 0x0000000001000000         0x0
  LOAD           0x0000000080ff1010 0x0000000100000000 0x0000000100000000
                 0x0000000080000000 0x0000000080000000         0x0
```

The following `QEMUCPUState` structure is available for each virtual x86 CPU:

```c
typedef struct QEMUCPUSegment {
    uint32_t selector;
    uint32_t limit;
    uint32_t flags;
    uint32_t pad;
    uint64_t base;
} QEMUCPUSegment;
        
typedef struct QEMUCPUState {
    uint32_t version;
    uint32_t size;
    uint64_t rax, rbx, rcx, rdx, rsi, rdi, rsp, rbp;
    uint64_t r8, r9, r10, r11, r12, r13, r14, r15;
    uint64_t rip, rflags;
    QEMUCPUSegment cs, ds, es, fs, gs, ss;
    QEMUCPUSegment ldt, tr, gdt, idt;
    uint64_t cr[5];
    uint64_t kernel_gs_base;
} QEMUCPUState;
```

Processing of QEMU ELF dump file is implemented in [qemu_elf.c](https://gitlab.com/qemu-project/qemu/-/blob/master/contrib/elf2dmp/qemu_elf.c).

### DirectoryTableBase

One of the header fields is `DirectoryTableBase` (DTB). DTB contains the physical address of the root of the virtual-to-physical page address translation. In the method based on vmcoreinfo device and guest driver the field was filled by the guest OS, but elf2dmp had to calculate this value based on data from the ELF file. DTB is also required to access guest kernel virtual memory when searching for `KdDebuggerDataBlock` and other header fields. In elf2dmp [addrspace.c](https://gitlab.com/qemu-project/qemu/-/blob/master/contrib/elf2dmp/addrspace.c) is responsible for handling the physical and virtual address spaces.

The elf2dmp finds DTB value for several tries. The correctness of the value is checked by accessing `0xfffff78000000000` in kernel virtual address space. This address belongs to [`SharedUserData`](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/ntexapi_x/kuser_shared_data/index.htm). There are 3 attempts in total:
1. Set DTB to CPU#0 CR3 taken from ELF.
2. Set DTB to CR3 for each CPU context from ELF with a negative GS base. Negative GS base means the CPU is in Windows kernel context.
3. Set temporary DTB to CPU#0 CR3 from ELF. Set DTB to a value from `[IA32_KERNEL_GS_BASE+0x7000]` by using temporary DTB to access virtual memory, where [`IA32_KERNEL_GS_BASE`](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/kpcr.htm) is an MSR value from ELF.

After finding the DTB, elf2dmp can access both the user and kernel virtual address space. Now elf2dmp needs to find where in virtual memory the Windows kernel PE image is located.

### Kernel base

Windows kernel image is represented by `ntoskrnl.exe`. This PE image exports all the debugging symbols required by elf2dmp, such as `nt!KdDebuggerDataBlock` and `nt!KdVersionBlock`. To be able to resolve the debug symbols, elf2dmp needs to find the address where the kernel image is loaded.

To find the kernel base address, elf2dmp relies on the fact that there are interrupt handlers somewhere inside the Windows kernel image. Because the IDT base register value is stored in the ELF dump, elf2dmp takes the first address from the interrupt descriptor table. Now elf2dmp can scan the virtual memory downwards starting at that address using the following kernel image properties:
* Any PE image starts with the signature `MZ`.
* The kernel image is aligned to the page boundary
* The kernel export table lists the image name as `ntoskrnl.exe`

Once elf2dmp has access to the loaded kernel image, it needs to find debugging symbols for it.

### ntkrnlmp.pdb

Debugging symbols are stored in PDB format. The PDB file for the Windows kernel (`ntkrnlmp.pdb`) and some other system modules can be downloaded from Microsoft public symbol store. WinDbg also downloads PDBs from this location. To do this, elf2dmp must form a download UR which looks like `https://msdl.microsoft.com/download/symbols/ntkrnlmp.pdb/adc00fa5fc34456ba16e2687457240991/ntkrnlmp.pdb`. The link contains a PDB name and a PDB hash. The PDB hash is calculated from a GUID from the debug directory of the kernel PE image. A routine named [`pe_get_pdb_symstore_hash`](https://gitlab.com/qemu-project/qemu/-/blob/master/contrib/elf2dmp/main.c#L428) does this in elf2dmp.

When the URL is generated, elf2dmp downloads the PDB file. This is described in [download.c](https://gitlab.com/qemu-project/qemu/-/blob/master/contrib/elf2dmp/download.c).

Now elf2dmp can parse the PDB file and resolve the required debugging symbols with the `pdb_find_public_v3_symbol` function from [pdb.c](https://gitlab.com/qemu-project/qemu/-/blob/master/contrib/elf2dmp/pdb.c).

### KdDebuggerDataBlock

The importance of the KdDebuggerData block structure was described in the previous post of the series. To find this structure in the virual address space, elf2dmp resolves the symbol `nt!KdDebuggerDataBlock`. Recall that when Debug Mode is disabled, this structure is encrypted. The encryption function is parameterized by the symbols `nt!KdpDataBlockEncoded`, `nt!KiWaitAlways` and `nt!KiWaitNever` and looks like this
![image](https://github.com/virtio-win/kvm-guest-drivers-windows/assets/8286747/cca132e5-68ae-4401-82c8-70ce824365d2)
Here `ror` is right circular bit shift and `bswap` is 64-bit byte swap. Obviously, the function is easily reversible. Since elf2dmp now has access to all the debug symbols, knowing these parameters it can decrypt the structure.

### KdVersionBlock

The dump contains fields such as `MinorVersion`, `MajorVersion`, `MachineType` and `SecondaryDataState`. Their copies are available in a structure called `KdVersionBlock`, the address of which can also be resolved.

### Dump file writing

When `DirectoryTableBase`, `KdDebuggerDataBlock` and `KdVersionBlock` are available, elf2dmp can completely fill the dump header. The tool always sets the `BugcheckCode` to `0x161` (`LIVE_SYSTEM_DUMP`). After the physical memory blocks are copied from the ELF file to the DMP file, the Complete Memory Dump is ready.

## Conclusion

We have discussed the usage and internals of `elf2dmp` tool which can help to capture the Complete Memory Dump of the 64-bit Windows guest running under QEMU, without any actions inside the guest. This is a useful tool when the debug dump is needed, but the vmcoreinfo device is not connected or virtio-win drivers are not installed in the guest.
