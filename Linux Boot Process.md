# Linux Boot Process

## 1. High-Level Overview

When you power on a Linux machine (or virtual machine), it doesn’t immediately start up your user programs. It goes through a multi-stage process:

- Hardware initialization (firmware, POST)
- Bootloader stage (loading the kernel)
- Kernel startup (initializing hardware, memory, mounting root FS)
- Init / user space startup (services, daemons, login)

This is often remembered by the acronym B → G → K → I (BIOS / GRUB / Kernel / Init)

## 2. Detailed Breakdown

Below is a detailed breakdown of each stage.

### 2.1 Firmware Initialization (BIOS / UEFI)

Purpose: Start the machine and prepare minimal hardware to load something from storage.

- BIOS (Basic Input/Output System) or UEFI (Unified Extensible Firmware Interface) is firmware stored on the motherboard.
Performs POST (Power-On Self Test) to check RAM, CPU, storage controllers, etc.

- After POST, firmware reads boot configuration (which devices are bootable, boot order).

- BIOS reads from the Master Boot Record (MBR) in classic systems; UEFI can read from a special EFI partition (FAT formatted) and directly execute .efi files.

- It then transfers control to the bootloader (in BIOS → MBR / in UEFI → EFI application).

### 2.2 Bootloader Stage

Once firmware has given control:

- In BIOS systems: the first 512 bytes (MBR) contain bootstrap code and partition table. This is very small — it cannot do much, so it loads a second-stage bootloader (e.g. GRUB) from elsewhere.

- In UEFI systems: the firmware looks at the EFI system partition and runs the configured EFI application (e.g. grubx64.efi).

- The bootloader (GRUB, sometimes LILO, systemd-boot, etc.) lets you choose which kernel (and options) to boot.

- Bootloader loads the kernel image and an initramfs / initrd (initial RAM disk) into memory.

- It passes kernel parameters and then transfers execution to the kernel.

### 2.3 Kernel Initialization

Once the kernel begins executing:

- It decompresses itself (since kernel images are compressed).

- Initializes core subsystems: memory management, scheduling, interrupt handling, device drivers.

- It mounts the initramfs (initial RAM filesystem) as a temporary root filesystem. This is needed because the real root filesystem might need drivers/modules to access (e.g. disk, RAID, LVM).

- The kernel then loads necessary modules from initramfs, sets up device nodes, udev (device manager).

- Then it switches from the initramfs to the real root filesystem (like /) and mounts it.

After that, it starts the first user-space process, usually init (or systemd).

### 2.4 Init / User-Space Startup

This is when your Linux distribution’s init system takes over and starts services, daemons, and user sessions.

- Traditional init: SysV init (uses /etc/inittab, runlevels, scripts in /etc/rcX.d/)

- Modern systems: systemd, Upstart, OpenRC etc.

- The init system reads configuration (targets or runlevels) and starts services (networking, SSH server, GUI, etc.).

- Eventually, it presents you with a login prompt (console or GUI).

### 2.5 Runlevels / Targets

- In SysV init, runlevels are numbered states (0–6) such as single-user mode, multi-user, graphical, halt, etc.

- Scripts are in /etc/rcN.d/ directories, prefixed with S or K (start/kill).

- In systemd, runlevels are replaced by targets. For example, multi-user.target, graphical.target, emergency.target, etc.

- systemd resolves dependencies of units and starts them in parallel where possible.

## 3. Variations & Elaborations

### 3.1 UEFI vs BIOS

- UEFI replaces older BIOS and is more flexible: supports larger disks (GPT), secure boot, etc.

- In UEFI, bootloader is a file in the EFI partition; firmware reads it directly, no need for MBR trickery.

- GRUB or systemd-boot can be launched by UEFI.

### 3.2 initramfs / initrd

- The initramfs is a temporary root filesystem loaded into memory; it helps load essential modules before the real root FS is ready.

- It is used when the root disk or filesystems require modules (e.g. encrypted root, LVM, RAID).

- After doing its job, it “switches” execution to the real root.

### 3.3 systemd

- In modern Linux distributions, systemd is the de facto init system.

- It starts services defined by .service units, handles dependencies, timers, targets.

- It has features like socket activation, snapshots, parallel startup.

## 4. Common Pitfalls / Interview-Level Details

**kexec:** A way to jump to a new kernel from the running system without rebooting via firmware. 
Wikipedia

**Secure Boot:** Firmware may require that kernel or bootloader is signed.

**Bootloader chaining:** in multi-boot setups, one bootloader hands off to another.

**Recovery / emergency modes:** booting with init=/bin/bash or systemd’s rescue target.

**Race conditions in init scripts** if dependencies are misdeclared.

**Parallel startup:** systemd speeds boot by starting independent services in parallel.

**Switch_root vs pivot_root:** switching from initramfs to real root is non-trivial.