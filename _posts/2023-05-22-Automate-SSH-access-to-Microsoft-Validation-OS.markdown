---
layout: post
title:  "Automate SSH access to Microsoft Validation OS"
date:   2023-05-22 11:48:00 +0300
author:
  name: "Konstantin Kostiuk"
  url: "https://github.com/kostyanf14"
---

Microsoft Validation OS, built on the foundation of Windows 11, offers a controlled
environment for software testing and validation, ensuring compatibility and reliability.
Microsoft Validation OS includes various core features and come with installed but not
configured OpenSSH server. However, by following a series of straightforward steps,
system administrators and developers can configure the OpenSSH server, enabling secure
remote access to Microsoft Validation OS. This article will provide a comprehensive guide,
outlining the necessary configuration steps and best practices to successfully set up
the OpenSSH server on Microsoft Validation OS, empowering users with secure remote
administration capabilities and streamlined file transfer functionalities.

Microsoft Validation OS is a lightweight, fast, and customizable Windows 11-based
operating system that you can use on the factory floor to diagnose, mitigate and
repair hardware defects during Windows device manufacturing. Validation OS boots
into a Command Line environment to increase reliability on the factory floor and
supports running Win32 apps, smoothing the transition from early hardware bring-up
to retail OS and apps development.

## Image customization in general

Microsoft provides enough documentation to customize image and add necessary software:
 - [Mount and customize a Validation OS image](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/validation-os-mount-and-customize?view=windows-11)
 - [Launching an app or command when Validation OS starts](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/validation-os-run-an-app-on-boot?view=windows-11)

The important point for all steps is select proper WIM image. From Microsoft note:
 - Use index:1 if you'll be applying your image to a device's hard drive.
 - Use index:2 if you'll be using the image to boot from a USB drive.

In case when we prepare image for VM boot from ISO we should use `/index:2` in the DISM command.

What should be done for SSH access:

1. Generate SSH key pair
1. Copy SSH public key into Validation OS as a authorized key
1. Add OpenSSH server to autostart
1. Export new image

## Image preparation

Microsoft provide non-bootable [Validation OS ISO](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/validation-os-overview?view=windows-11). This image should be used only as a base.

1. Download Validation OS from the link above and extract is to some folder (C:\validation_os)
1. Start the Deployment and Imaging Tools Environment as an administrator.
1. Run `copype` to create a working copy of the Windows PE files.
   ```batch
   copype amd64 C:\WinPE_amd64
   ```
1. Copy Validation OS wim and replace the WinPE boot.wim
   ```batch
   xcopy C:\validation_os\ValidationOS.wim C:\WinPE_amd64\media\sources\boot.wim
   ```
1. Create a folder where you'll mount your image
   ```batch
   md c:\mount
   ```
1. Use DISM to mount the image
   ```batch
   DISM /Mount-Image /imagefile:"C:\WinPE_amd64\media\sources\boot.wim" /index:2 /MountDir:"C:\mount"
   ```
1. Generate SSH key pair
   ```batch
   md c:\validation_ssh
   ssh-keygen -t ed25519 -C "your_email@example.com" -f c:\validation_ssh\id_ed25519
   ```
1. Configure SSH server and add SSH public key as a authorized key
   ```batch
   copy C:\mount\Windows\System32\OpenSSH\sshd_config_default C:\mount\ProgramData\ssh\sshd_config
   type c:\validation_ssh\id_ed25519.pub > C:\mount\ProgramData\ssh\administrators_authorized_keys
   ```
1. Create and configure the SSH startup script `C:\mount\sshd.bat` with the following content
   ```batch
   rem "Generate SSH host keys"
   cmd /c "ssh-keygen -A"
   rem "Start SSH server"
   cmd /k "c:\Windows\System32\OpenSSH\sshd.exe"
   ```
1. Add script to startup via registry
   ```batch
   reg load HKLM\Image_SOFTWARE C:\mount\windows\system32\config\software
   reg add "HKLM\Image_SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Shell /t REG_SZ /F /D "cmd /k c:\sshd.bat"
   reg unload HKLM\Image_SOFTWARE
   ```
1. Unmount image and apply changes
   ```batch
   DISM /Unmount-Image /MountDir:"C:\mount" /Commit
   ```
1. Generate bootable iso
   ```batch
   MakeWinPEMedia /ISO C:\WinPE_amd64 C:\ValidationOS_SSH.iso
   ```

## Usage of created ISO

1. Boot any VM with ethernet adapter from `ValidationOS_SSH.iso`
1. Use the following SSH command to connect
   ```batch
   ssh -i c:\validation_ssh\id_ed25519 -o IdentitiesOnly=yes Administrator@<VM_IP>
   ```

## Conclusion

In conclusion, configuring the OpenSSH server on Microsoft Validation OS opens
up a world of possibilities for secure remote access within this specialized
testing and validation environment. By following the steps outlined in this
article, you can establish a robust infrastructure that ensures the confidentiality,
integrity, and authenticity of your remote connections. With the OpenSSH server in
place, you can confidently administer and manage Microsoft Validation OS, transfer
files securely, and streamline your workflows. Embracing the power of OpenSSH
within the Microsoft Validation OS environment not only enhances your productivity
but also reinforces your commitment to maintaining a strong security posture.
