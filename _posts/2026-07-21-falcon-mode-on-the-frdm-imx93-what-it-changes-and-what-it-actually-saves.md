---
title: "Falcon Mode on the FRDM-IMX93: what it changes and what it actually saves"
date: 2026-07-21 12:40:23 +0300
categories: [Embedded Linux, U-Boot, Boot Time Optimization]
tags: [embedded-linux, u-boot, boot-time-optimization]
---

Falcon Mode is often described as a way to cut boot time. But how much does it actually save? I enabled it on an FRDM-IMX93 (Yocto BSP lf-6.18.2, image imx-image-core, eMMC) and compared it against a normal boot, with everything else kept the same. The short answer is about 3.8 s off the bootloader. The rest of this post shows where that time goes and where it doesn’t.

> This post stays at the level of what Falcon changes and what it saves. The step by step (deep dive) walk through the boot flow in code will be a separate post, so this one doesn’t run too long.

#### What Falcon Mode is and how it works

Falcon Mode is a U-Boot feature for faster boot. The SPL is U-Boot’s small first stage and normally its job is to start U-Boot proper, which then starts the kernel. Falcon Mode lets the SPL load and start the kernel directly and skip U-Boot proper. So, the board spends less time in the bootloader before the kernel.

First, here is a normal i.MX boot:

- **BootROM** is immutable code built into the SoC. It does the minimal hardware setup needed to boot, picks the boot device from the boot configuration, verifies the boot image if secure boot is on and loads the SPL into on-chip SRAM.
- **SPL (Secondary Program Loader)** is the small first stage of U-Boot that fits in the on-chip SRAM. Its job is to bring up the DRAM and load the rest of the firmware into it, such as BL31, U-Boot proper and optionally OP-TEE.
- **ATF (Arm Trusted Firmware)** is made of several boot stages (BL1, BL2, BL31 and optionally BL32 for OP-TEE). On i.MX, BootROM and SPL do the work of BL1 and most of BL2, so the ATF part that actually runs is BL31, the EL3 Runtime Firmware. It stays loaded after boot because Linux calls into it through Secure Monitor Calls (SMCs) when it needs PSCI (CPU power and reset).
- **U-Boot proper** is the full second stage, running from DRAM. It sets up more hardware, loads its environment, runs the boot commands, fixes up the device tree, loads the kernel (and initramfs if present) and finally starts it.
- **Kernel** takes over and the bootloader is done.

**In Falcon Mode**, the SPL loads the Linux kernel (and device tree) itself instead of U-Boot proper. It still hands control to BL31 as in a normal boot but this time BL31 makes the EL3 to kernel jump directly instead of starting U-Boot proper. So the whole U-Boot proper stage is skipped and the board reaches kernel sooner.

![Boot chain comparison: a normal boot runs BootROM, SPL, ATF, U-Boot proper and then the kernel, while Falcon Mode skips U-Boot proper and jumps from ATF straight to the kernel](/assets/img/posts/falcon-mode-on-the-frdm-imx93-what-it-changes-and-what-it-actually-saves/1.png)
_In Falcon Mode, BL31 hands off straight to the kernel and U-Boot proper never runs on the fast path._

#### The changes that make it work

The SPL can’t just jump to a kernel image. A few things have to be in place first and here is each one:

- **The kernel command line moves into the device tree.** With no U-Boot proper, nothing sets `bootargs` at boot, so the build writes the command line into the `chosen` node of the kernel device tree. Changing it means a rebuild, not a `setenv`.
- **The SPL reads the kernel from the eMMC itself.** With no U-Boot proper to do it, the SPL loads a single AHAB container (BL31 + kernel + device tree) straight from the FAT boot partition. That AHAB packaging is i.MX specific, on another SoC it would be a FIT image or a raw kernel.
- **BL31 (the EL3 runtime) enters the kernel directly.** BL31 hands off to the first non-secure image. In a normal boot that image is BL33, which on i.MX is U-Boot proper. In Falcon Mode the kernel is loaded at that same entry point instead, where U-Boot proper would normally sit, so BL31 hands off straight into the kernel and skips U-Boot proper. The handoff follows the arm64 Linux boot protocol. The only thing the bootloader hands the kernel is a pointer to the device tree. Everything else the kernel needs, like the command line and the memory layout, comes through the device tree itself.
- **U-Boot proper is still there for recovery.** On this board, pressing a key on the console during the SPL loads the U-Boot container from the boot partition instead of the kernel. The fast path never uses U-Boot proper, it is on the boot partition only for that recovery case.

#### The measurement

I captured both boots (normal and falcon) with `grabserial` from power-on, same image and Falcon the only thing changed. The key lines from the two logs:

```console
[ 1.288812] U-Boot SPL 2025.04 …
[ 2.281204] U-Boot 2025.04 … <- U-Boot proper
[ 5.609608] [ 0.000000] Booting Linux on physical CPU
[21.713059] … login:
```
{: file="normal-boot.log" }

```console
[ 1.299749] U-Boot SPL 2025.04 … -dirty
[ 1.466607] Trying to boot from MMC1
[ 1.827276] [ 0.000000] Booting Linux on physical CPU
[17.485594] … login:
```
{: file="falcon-boot.log" }

The 3.8 s comes in two parts. About 3.33 s is U-Boot proper, which runs from 2.28 s to 5.61 s in the normal log and is gone in Falcon. The other 0.45 s is the Falcon SPL reaching the kernel sooner. The kernel starts at 1.83 s, before the normal boot even reaches U-Boot proper at 2.28 s.

![Bar chart of power-on to login time: normal boot reaches login at 21.71 s, Falcon Mode at 17.49 s](/assets/img/posts/falcon-mode-on-the-frdm-imx93-what-it-changes-and-what-it-actually-saves/2.png)
_Power-on to login on the FRDM-i.MX93. The 3.8 s saved is 3.33 s of dropped U-Boot proper plus 0.45 s from the SPL reaching the kernel sooner._

#### The trade we are making

Falcon trades flexibility for boot time. These trade-offs are general to Falcon Mode, even if the exact mechanisms differ by SoC. Here is what you give up:

- The boot configuration is fixed at build time. In most Falcon setups the kernel command line comes from the device tree rather than the U-Boot environment, so changing it means rebuilding or replacing the boot image, not a `setenv`.
- Recovery has to be designed in and that is the real work, not the speed. With U-Boot proper skipped on the normal path, the SPL itself has to notice a failed or invalid kernel and fall back to a recovery image or another slot. U-Boot proper can still be there for maintenance if you wire up a trigger for it, such as a key, a GPIO or a boot switch but changes made there don’t affect the fast path.
- The rest of U-Boot proper’s logic moves too. A/B selection, boot counting and OTA hooks now live in the SPL or have to be handled another way.
- The bootloader and the kernel are more tightly coupled. Since the SPL loads the kernel directly, the two are usually versioned and updated together.
- We lose U-Boot’s services and its console. The SPL still brings up the essentials like DDR, eMMC, clocks and pinmux but what U-Boot proper adds is gone on the fast path: the interactive console, environment editing and scripting, network, USB or PXE boot. There is no interactive console to drop into if something goes wrong in the field.

Falcon Mode is one of those optimizations that looks deceptively simple. Skip U-Boot proper and the board boots faster. In practice the speedup is real but so are the trade-offs. Whether it is worth adopting depends less on the seconds saved than on how much flexibility your product can afford to lose.
