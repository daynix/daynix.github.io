---
layout: post
title:  "Guest Windows debugging and crashdumping under QEMU/KVM: Introduction"
date:   2023-01-20 14:22:39 +0300
author:
  name: "Viktor Prutyanov"
  url: "https://github.com/viktor-prutyanov"
---

![image](https://user-images.githubusercontent.com/8286747/213822987-6f590160-6e14-4862-a3e1-f6fdfbb8cda7.png)

Hypervisor virtualization is a complex process, so problems such as BSoDs or hangs can occur when the guest operating system is running. These problems can be caused either by bugs on the guest side or by bugs in the hypervisor itself. There are two methods to analyze such problems on the guest OS side: debugging and dump analysis.

This post starts a series of posts dedicated to debugging and capturing dumps of Windows guests running under the QEMU/KVM hypervisor. In this introductory post, we will talk about debugging and crashdumping with the builtin Windows system software, and why this is sometimes not enough.

## Kernel debugging

For kernel debugging you will need another Windows VM on which to run WinDbg. Kernel debugging can utilize several interfaces for connection between debugger and target: network (IPv4/IPv6), USB, serial, IEEE1394.

The simplest way in terms of setup is to use a serial connection.

![image](https://user-images.githubusercontent.com/8286747/213817641-da3959ae-90c1-46a7-8898-1a77e8ac102c.png)

To do this we need to add a serial port to each of the VMs.

Debugger VM:
```
-serial tcp::4445,server,nowait
```
Target VM:
```
-serial tcp:127.0.0.1:4445
```
The TCP port can be changed from 4445 to any unused port.

The following command should be executed on the target VM side to enable kernel debugging of the default boot entry:
```
bcdedit.exe /debug on
```
The following command enables serial debugging through the `COM1' port at 115200 bit/s:
```
bcdedit.exe /dbgsettings serial debugport:1 baudrate:115200
```

![image](https://user-images.githubusercontent.com/8286747/213817833-3a1a1374-f594-4d4c-b7f6-c23cb32c32e2.png)

The above changes will take effect at the next boot. Now everything is ready for debugging.

The following command runs kernel debugging in WinDbg through the `COM1' port at 115200 bit/s:
```
windbg.exe -k com:baud=115200,port=com1
```
WinDbg will wait for a connection:

![image](https://user-images.githubusercontent.com/8286747/213817882-135473a1-58ca-4aab-a066-5145d9aa8b6a.png)

Now we can start target VM. The boot manager will tell us that debugging is turned on:

![image](https://user-images.githubusercontent.com/8286747/213817923-a96eb030-f62e-47dd-ba9b-e3dba50ad11c.png)

After some time WinDbg will tell us that connection is established:

![image](https://user-images.githubusercontent.com/8286747/213817982-01a27f76-f6a0-4c50-9648-3d86cc9f9663.png)

At the same time the target machine works as usual. Now we can send a Break command:

![image](https://user-images.githubusercontent.com/8286747/213818008-0f2d3546-d0ab-45cf-b915-b71aac7fc18a.png)

The execution of the target machine will be interrupted and we will be able to enter WinDbg commands in the command line, such as `k`, `lm n t` and so on:

![image](https://user-images.githubusercontent.com/8286747/213818025-12eceea8-140e-4598-8d53-1083a668df3a.png)

## Memory dump on BSoD

In case of a kernel-mode error Windows can automatically save a dump on the disk. Specific behavior can be controlled in the "Startup and Recovery" menu:

![image](https://user-images.githubusercontent.com/8286747/213818055-d470584c-a4c9-4a1f-bfd2-377222de2591.png)

By default, when failure occurs, Automatic Memory Dump is captured and saved to `C:\Windows\MEMORY.DMP`. The existing file is overwritten. Then the system reboots. 

![image](https://user-images.githubusercontent.com/8286747/213818067-7af1358b-6d34-49c4-bf14-5f7818af8b84.png)

After that, if the crash dump were successfully retrieved from the disk, it can be opened in WinDbg for analysis:

![image](https://user-images.githubusercontent.com/8286747/213818082-47f877d7-842a-485a-a449-b2105b8c9763.png)

The screenshot from WinDbg shows that Windows saved `BugcheckCode` (`0xd1`) and `BugcheckParameters` to help us understand what the problem is.

## Memory dump from LiveKD

LiveKD from Sysinternals Suite can create _Complete Memory Dump_ from a live system:

![image](https://user-images.githubusercontent.com/8286747/214035925-43874d48-b5ed-4dda-a472-a84fd138c8ee.png)

Such a dump can be opened in WinDbg as a crash dump:

![image](https://user-images.githubusercontent.com/8286747/214035506-db3c325c-714b-4886-ac9f-17150869b96d.png)

## Limitations

The main and perhaps the only problem with kernel debugging is that it must be configured and started before an error occurs. So, it is convenient for a development environment but it cannot be used on the customer server or during automatic testing. Memory dumps are much better suited for analyzing a problem that has already occurred, but the built-in methods of collecting them have some disadvantages.

First of all, the default settings are not very good. _Automatic Memory Dump_, captured by default, is similar to _Kernel Memory Dump_. It only includes memory allocated to the Windows kernel, HAL, kernel-mode drivers and other kernel-mode programs. So, the system must be specially configured by the user to create a _Complete Memory Dump_ that most fully reflects the state of the system.

There are also situations where a dump cannot be created. For example, if there is not enough free space on the disk. _Complete Memory Dump_ consumes as much space as the total physical memory. Also, sometimes the storage driver is the source of the problem.

Besides that, the built-in dump generation mechanism can't help when a dump is needed in the moment of VM freeze or another point of interest unrelated to the crash.

Thus, it would be nice to have tools that allow us to get the most detailed dump of the guest Windows at any point in time on the hypervisor side, preferably without any preconfiguration. All these circumstances have led to development of more convenient ways of capturing dumps, which will be described in the following posts.

