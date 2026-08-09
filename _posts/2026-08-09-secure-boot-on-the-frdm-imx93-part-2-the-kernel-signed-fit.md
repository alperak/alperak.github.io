---
title: "Secure Boot on the NXP FRDM i.MX93 - Part 2: The Kernel (Signed FIT)"
date: 2026-08-09 11:00:00 +0300
categories: [Embedded Linux, U-Boot, Secure Boot, FIT Image]
tags: [embedded-linux, u-boot, secure-boot, fit-image, linux-kernel, nxp, imx93]
---

In [Part 1: The Bootloader (AHAB)](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/), we built the first part of the chain of trust. The Boot ROM loaded the ELE firmware, the ELE ROM authenticated the ELE firmware against NXP's fused SRK hash and the ELE firmware authenticated the OEM containers against the OEM SRK hash we programmed into the fuses.

![Where Part 1 left us: the root of trust is the Boot ROM, the ELE ROM and the fuses, and from there the ELE firmware authenticates the SPL with the DDR firmware and then ATF, OP-TEE and U-Boot, all signed with our SRKs. The chain then breaks, because nothing checks the Linux kernel and device tree, which are whatever happens to be on the eMMC](/assets/img/posts/secure-boot-on-the-frdm-imx93-part-2-the-kernel-signed-fit/part2-the-gap.png){: width="1657" height="361" }

At that point, the chain of trust ended at U-Boot. Although the bootable image containing U-Boot had been authenticated by the ELE firmware, the Linux kernel was still loaded directly from storage without any independent verification by U-Boot.

## What this part builds

In this part, we extend the chain of trust to the Linux kernel by protecting it with a signed FIT (Flattened Image Tree). We will build and sign the FIT image, embed the FIT public key into U-Boot's control device tree, modify the boot flow so U-Boot verifies the FIT image before booting Linux, and finally show that tampered and unsigned FIT images are rejected.

## Why we choose FIT over the OS container

When AHAB is enabled, NXP's default boot flow expects the kernel and device tree to be provided in a signed OS container (`os_cntr_signed.bin`). U-Boot loads this container with `loadcntr`, then `auth_os` requests the ELE firmware to authenticate it and only continues if the authentication succeeds. We already saw this path in [section 8 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#8-enable-ahab-in-u-boot-and-read-ahab_status), where `loadcntr` searched for `os_cntr_signed.bin`.

In this guide, we take a different approach. Instead of packaging the kernel and device tree in an OS container, we keep them in a signed U-Boot FIT image and let U-Boot verify that image before booting Linux. The underlying hardware root of trust does not change. The difference is only where the kernel is checked. The OS container is authenticated by the ELE firmware, while a FIT image is verified by U-Boot using a public key embedded in U-Boot's control device tree. That control device tree is itself authenticated as part of the OEM container that carries U-Boot, so the chain of trust naturally extends from the ELE firmware to U-Boot and finally to the Linux kernel.

The reasons for choosing FIT are:

- **A standard, portable mechanism.** FIT verification is a mainline U-Boot feature rather than an NXP specific one, so the same concepts, tooling and boot flow can be carried to other platforms.

- **One signed image, many targets.** A FIT can contain multiple kernels, device trees and other boot artifacts, with each configuration selecting a specific combination of them. Each configuration can be signed and verified as a unit. This gives us a natural way to support multiple board variants or boot configurations without creating a separate FIT format for each combination.

- **A natural fit for future extensions.** A FIT can carry additional boot artifacts alongside the kernel, such as an initramfs or other firmware images, and a configuration can reference the artifacts that belong together. The configuration signature covers that configuration together with the hashes of the images it references, while U-Boot verifies those hashes against the image data. This gives us a flexible foundation for extending the verified boot chain without introducing a separate verification mechanism for every additional artifact.

## Inside a signed FIT image

The Linux kernel is protected by a signed FIT (Flattened Image Tree) image. Like an AHAB container, it bundles several binaries together with the metadata needed to check them before they run. What changes is the component performing that check. AHAB authentication is performed by the ELE (ROM and firmware), while FIT verification is performed entirely by U-Boot.

A FIT image is described by an `.its` (Image Tree Source) file, which U-Boot refers to as an **image source file**. It uses device tree syntax and is processed by `dtc`, but instead of describing hardware, it describes the images that make up the FIT. The Yocto build generates this file automatically when we build the image in [section 5](#5-build-the-signed-fit-image) and `fit-image.its` will appear in the deploy directory. The example below has been trimmed to a single device tree so that it fits on the page.

```dts
/dts-v1/;

/ {
        description = "Kernel fitImage for NXP i.MX Minimal Distro with Wayland/1.0/frdm-imx93-secure";
        #address-cells = <1>;
        images {
                kernel-1 {
                        description = "Linux kernel";
                        type = "kernel";
                        compression = "gzip";
                        data = /incbin/("linux.bin");
                        arch = "arm64";
                        os = "linux";
                        load = <0x80400000>;
                        entry = <0x80400000>;
                        hash-1 {
                                algo = "sha256";
                        };
                };
                fdt-imx93-11x11-frdm.dtb {
                        description = "Flattened Device Tree blob";
                        type = "flat_dt";
                        compression = "none";
                        data = /incbin/("imx93-11x11-frdm.dtb");
                        arch = "arm64";
                        hash-1 {
                                algo = "sha256";
                        };
                };
        };
        configurations {
                default = "conf-imx93-11x11-frdm.dtb";
                conf-imx93-11x11-frdm.dtb {
                        description = "1 Linux kernel, FDT blob";
                        kernel = "kernel-1";
                        fdt = "fdt-imx93-11x11-frdm.dtb";
                        hash-1 {
                                algo = "sha256";
                        };
                        signature-1 {
                                algo = "sha256,rsa4096";
                                key-name-hint = "fit_signing_key";
                                padding = "pkcs-1.5";
                                sign-images = "kernel", "fdt";
                        };
                };
        };
};
```

The table below summarizes the most relevant fields:

| Field | What it is |
|---|---|
| `type`, `arch` | Identifies the image type (for example `kernel` or `flat_dt`) and the target architecture. |
| `compression` | Specifies how the image is stored inside the FIT. |
| `data = /incbin/(...)` | Includes the referenced binary in the FIT image. |
| `load`, `entry` | Defines the load and entry addresses used when booting the image. |
| `hash-1` | Stores the SHA-256 hash of an individual image. |
| `default` | Selects the default boot configuration. |
| `kernel`, `fdt` | Associates a kernel and device tree with a configuration. |
| `signature-1` | Stores the signature protecting the configuration. |
| `sign-images` | Lists which image hashes are covered by the configuration signature. |
| `padding` | Specifies the signature padding scheme. |

The FIT has only two top level sections and understanding the difference between them is the key to understanding how FIT verification works:

- The `images` section contains the individual images together with the metadata and hashes used to verify each of them.
- The `configurations` section defines valid combinations of those images. Each configuration references the image nodes it uses. In the default and recommended configuration used throughout this guide, the configuration node carries the signature, while the individual image nodes carry only hashes.

U-Boot first verifies the configuration signature and then verifies every referenced image against the hashes protected by that signature. So the kernel and device tree carry hashes rather than signatures of their own.

The configuration signature is verified with a public key stored in U-Boot's control device tree, not the Linux device tree. This is the `u-boot.dtb` that we will inspect in [section 6](#6-verify-the-fit-signature-before-you-flash). When that key is marked `required`, U-Boot refuses to boot a FIT image whose configuration signature cannot be verified.

### Why the configuration is signed instead of the images

U-Boot's own documentation is direct about the reason. Signing the individual images sounds more thorough but leaves two holes open:

- The first is the mix-and-match attack: *"It is possible to create a FIT with the same signed images, but with the configuration changed such that a different one is selected."* Valid images from different releases can be recombined into a configuration that was never intended to exist.

- The second is the rollback attack: *"It is also possible to substitute a signed image from an older FIT version into a newer FIT."* A correctly signed but outdated kernel can therefore be brought back after a vulnerability has already been fixed.

As the U-Boot documentation puts it: *"It is the configurations that are signed, not the image. Each image has its own hash and we include the hash in the configuration signature."*

Signing the configuration prevents the mix-and-match attack because the configuration and the hashes of its referenced images are signed together. However, a configuration signature by itself does not prevent rollback. An older FIT with a valid signature is still cryptographically valid, so preventing rollback requires an additional version or rollback protection mechanism that rejects older images.

## Why the FIT lives in a raw partition

The default BSP uses [`imx-imx-boot-bootpart.wks.in`](https://git.yoctoproject.org/meta-freescale/tree/files/wic/imx-imx-boot-bootpart.wks.in?h=wrynose), which creates a 256 MiB VFAT `/boot` partition. In that layout, the FIT image is stored as a file and loaded with `load mmc`. In this guide, the FIT instead resides in its own raw partition and is loaded directly with `mmc read`.

One reason is to minimize the amount of code that processes untrusted data before signature verification. With a filesystem based boot partition, U-Boot must first parse the FAT filesystem and its metadata before it can locate the FIT image. A raw partition removes filesystem parsing from this pre-verification path, reducing the amount of code that handles attacker controlled data before the FIT signature is verified. The cryptographic guarantees remain the same but considerably less code is involved before the FIT signature is checked.

A raw partition also improves robustness. Because it contains only the FIT image, there is no filesystem metadata that can become inconsistent if power is lost during an update. The FIT itself can still be left partially written by an interrupted update but there is no additional filesystem state that U-Boot must parse or recover from.

The trade-off is that the partition layout must stay synchronized between the `.wks` file and the U-Boot environment. With a filesystem based `/boot` partition, U-Boot only needs a partition number and file name. With a raw partition, it must also know the partition's start block and size. In practice, this is a small maintenance cost for a simpler, more deterministic and easier to audit boot path.

## 1. Create the FIT signing key

The first thing we need is the key that will sign the FIT image. This is a new key, separate from the SRKs (Super Root Keys). Reusing the SRKs may seem like the obvious choice but they serve different purposes. The SRKs anchor the hardware root of trust. The FIT signing key has a narrower role. Its private half signs the FIT image during the build and only the corresponding public key is embedded in U-Boot's control device tree. Keeping the FIT signing key separate from the SRKs limits the impact of a compromise. If the FIT signing key is compromised, it can simply be replaced by deploying a new U-Boot with an updated control device tree. If the SRK private keys are compromised, there is no equivalent field update because the fused SRK hash is immutable.

> **Production note.** For simplicity, this guide stores the FIT signing key on the development machine. In production, private signing keys are typically generated and protected inside a Hardware Security Module (HSM) and accessed through interfaces such as PKCS#11, so the private keys do not leave the secure hardware. Depending on the release process, image signing may be performed as a dedicated release step or directly from the Yocto build if PKCS#11 integration is available.

Create the FIT signing key inside the same `artifacts-and-tools` directory we created in [section 9 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#9-create-the-srks-super-root-keys). Keeping the FIT signing key together with the other generated artifacts keeps the generated keys and signing artifacts in one place.

```bash
mkdir -p fit-keys
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out fit-keys/fit_signing_key.key
openssl req -new -x509 -sha256 -days 36500 -subj "/CN=FIT Signing Key" -key fit-keys/fit_signing_key.key -out fit-keys/fit_signing_key.crt
```

| Part | Meaning |
|---|---|
| `genpkey -algorithm RSA` | Generates the RSA private key (`fit_signing_key.key`). This key must remain secret. |
| `rsa_keygen_bits:4096` | Uses a 4096-bit RSA key. |
| `req -new -x509` | Creates a self-signed X.509 certificate (`fit_signing_key.crt`) containing the corresponding public key. |
| `-sha256` | Uses SHA-256 when signing the self-signed certificate. |
| `-days 36500` | Gives the certificate a long validity period. U-Boot does not check certificate expiration, so this simply avoids regenerating the certificate unnecessarily. |
| `-subj "/CN=FIT Signing Key"` | Supplies the certificate subject on the command line, avoiding interactive prompts. |

```console
ls -l fit-keys/
-rw-rw-r-- 1 alper alper 1939 fit_signing_key.crt
-rw------- 1 alper alper 3268 fit_signing_key.key

openssl rsa -in fit-keys/fit_signing_key.key -noout -text | head -1
Private-Key: (4096 bit, 2 primes)
```

## 2. Enable FIT verification and replace the boot command

In [section 8 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#8-enable-ahab-in-u-boot-and-read-ahab_status), we created `secure-boot.cfg` to enable AHAB support:

```bash
CONFIG_AHAB_BOOT=y
```

We now extend the same configuration fragment to enable FIT signature verification:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-bsp/u-boot/u-boot-imx/secure-boot.cfg <<'EOF'
# Enable AHAB  boot support.
CONFIG_AHAB_BOOT=y

# Enable FIT image support and RSA signature verification.
CONFIG_FIT=y
CONFIG_FIT_SIGNATURE=y
CONFIG_RSA=y
CONFIG_RSA_VERIFY=y

# Replace the BSP's default boot command instead of modifying bsp_bootcmd.
# The default path executes
# "run sr_ir_v2_cmd;bootflow scan -lb; run bsp_bootcmd"
#
# "secure_bootcmd" lives in the board environment (imx93_frdm.env)
CONFIG_BOOTCOMMAND="run secure_bootcmd"
EOF
```

`CONFIG_BOOTCOMMAND` replaces the BSP's default boot command with a single `run secure_bootcmd`. We define `secure_bootcmd` in the board environment file, which the next two sections cover.

### Why we replace the default boot command

It is worth looking at the boot command the BSP provides:

```bash
CONFIG_BOOTCOMMAND="run sr_ir_v2_cmd;bootflow scan -lb; run bsp_bootcmd"
```

Each stage conflicts with the verified FIT boot flow we are building:

| Command | What it does | Why we replace it |
|---|---|---|
| `run sr_ir_v2_cmd` | Prepares an EBBR/SystemReady device tree for `bootflow scan`, for operating systems that do not provide one of their own. | Our kernel always boots with the device tree embedded in the FIT, so this step is unnecessary. |
| `bootflow scan -lb` | Scans boot devices for EFI, extlinux or boot scripts and boots the first valid one it finds. | It can bypass the verified boot path entirely by booting whatever it discovers before our logic is reached. |
| `run bsp_bootcmd` | Executes the BSP's standard boot logic. | It expects either an unsigned `boot.scr` or a raw `Image`. If AHAB is enabled, it expects a signed OS container (`os_cntr_signed.bin`). Our design uses none of these, we want to boot a signed FIT image directly from a dedicated raw partition. |

One important point is worth mentioning. Replacing `CONFIG_BOOTCOMMAND` changes only U-Boot's compiled-in default environment. If a persistent environment has been saved, it takes precedence over the compiled-in defaults. The saved copy is protected by a CRC32, which catches corruption but not tampering, because anyone able to write that area can compute a matching checksum. On this board, the environment is stored in writable eMMC, so a signed FIT image alone does not protect the boot command from being modified. Hardening the U-Boot environment is a separate topic and outside the scope of this guide.

### Defining what the boot command runs

The default U-Boot environment is defined in the board environment file, [`board/nxp/imx93_frdm/imx93_frdm.env`](https://github.com/nxp-imx/uboot-imx/blob/6eeef838dac4ddbc06ff14450531a95e8c5cb346/board/nxp/imx93_frdm/imx93_frdm.env#L87).

This is the one place in this guide that needs a patch rather than a `.cfg` fragment. The boot command needs control flow and an error path, which do not fit naturally into a Kconfig string. The patch itself is kept small on purpose and explains each of its choices in its own comments.

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-bsp/u-boot/u-boot-imx/0001-imx93_frdm-add-secure_bootcmd-for-FIT-boot.patch <<'EOF'
From bba858b679b1a25025819e4b333124a8c74a0dd9 Mon Sep 17 00:00:00 2001
From: Alper Ak <alperyasinak1@gmail.com>
Date: Thu, 6 Aug 2026 21:04:09 +0300
Subject: [PATCH] imx93_frdm: add secure_bootcmd for FIT boot

Add secure_bootcmd, which reads the FIT image from a dedicated raw partition
and lets bootm verify it before transferring control to Linux.
CONFIG_BOOTCOMMAND is updated to invoke secure_bootcmd directly.

Upstream-Status: Inappropriate [product specific boot configuration]

Signed-off-by: Alper Ak <alperyasinak1@gmail.com>
---
 board/nxp/imx93_frdm/imx93_frdm.env | 41 +++++++++++++++++++++++++++++
 1 file changed, 41 insertions(+)

diff --git a/board/nxp/imx93_frdm/imx93_frdm.env b/board/nxp/imx93_frdm/imx93_frdm.env
index 8dee8b63ba4..624ff7b3931 100644
--- a/board/nxp/imx93_frdm/imx93_frdm.env
+++ b/board/nxp/imx93_frdm/imx93_frdm.env
@@ -107,3 +107,44 @@ bsp_bootcmd=
 		fi;
 	fi;
 scriptaddr=0x83500000
+
+/*
+ * Verified FIT boot.
+ *
+ * Reads the signed FIT image from a dedicated raw partition and lets bootm
+ * verify its configuration signature before transferring control to Linux.
+ * CONFIG_BOOTCOMMAND points to "run secure_bootcmd", bypassing the BSP's
+ * default boot flow.
+ *
+ * fit_blk and fit_cnt must match the FIT partition defined in the .wks file.
+ *
+ *      fit_blk = 0x4000  -> 8 MiB offset (512 byte MMC blocks)
+ *      fit_cnt = 0x20000 -> 64 MiB partition size
+ *
+ * fit_addr is only a temporary staging address for the FIT image.
+ * bootm relocates the kernel to the load address stored inside the FIT
+ * (0x80400000), so fit_addr does not need to match the kernel load address.
+ */
+
+
+/* Temporary RAM location for the FIT image */
+fit_addr=0x83000000
+
+/* FIT partition offset (8 MiB) */
+fit_blk=0x4000
+
+/* FIT partition size (64 MiB) */
+fit_cnt=0x20000
+
+/* Kernel command line */
+secure_bootargs=setenv bootargs console=ttyLP0,115200 earlycon root=/dev/mmcblk0p2 rootwait rw
+
+secure_bootcmd=
+    echo "Booting signed FIT image...";
+    run secure_bootargs;
+    if mmc dev 0; then
+        if mmc read ${fit_addr} ${fit_blk} ${fit_cnt}; then
+            bootm ${fit_addr};
+        fi;
+    fi;
+    echo "ERROR: FIT boot failed (verification or image loading)"
-- 
2.43.0
EOF
```

Update the existing `.bbappend` to include the patch alongside `secure-boot.cfg`:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-bsp/u-boot/u-boot-imx_%.bbappend <<'EOF'
# secure-boot.cfg enables FIT and AHAB.
# The patch adds secure_bootcmd to the board environment.

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI:append = " \
    file://secure-boot.cfg \
    file://0001-imx93_frdm-add-secure_bootcmd-for-FIT-boot.patch \
"
EOF
```

The RAM addresses deserve attention. The kernel is loaded at `0x80400000`, which `meta-freescale` sets for this SoC through `UBOOT_ENTRYPOINT`, while the FIT image is staged at `0x83000000`, leaving about 44 MiB between them. The current uncompressed kernel is roughly 34 MiB, leaving around 10 MiB of headroom. If the kernel eventually grows beyond that, it can overwrite the staged FIT image before `bootm` has finished processing it. U-Boot detects this condition and prints:

```console
ERROR: new format image overwritten - must RESET the board to recover
```

Whenever the kernel configuration changes, check the size of the uncompressed kernel `Image` in the deploy directory and make sure enough space remains between the kernel load address and the FIT staging address.

## 3. Create the partition layout

The default BSP `.wks` file ([imx-imx-boot-bootpart.wks.in](https://git.yoctoproject.org/meta-freescale/tree/files/wic/imx-imx-boot-bootpart.wks.in?h=wrynose)) creates a 256 MiB VFAT `/boot` partition. Our design replaces it with a dedicated raw partition for the FIT image and explicitly reserves the U-Boot environment area instead of leaving it as an unnamed gap, so we need a custom partition layout. Rather than modifying the BSP file directly, we create our own `.wks` file.

```bash
mkdir -p ../sources/meta-frdm-imx93-security/files/wic
```

Create `frdm-imx93-secure.wks.in` with the following contents:

```bash
cat > ../sources/meta-frdm-imx93-security/files/wic/frdm-imx93-secure.wks.in <<'EOF'
# short-description: eMMC layout for secure boot with a raw signed FIT
# long-description:
# Stores the signed FIT image in its own raw partition instead of a VFAT /boot
# filesystem.
#
# mmcblk0                                p1                   p2
#                                        |                    |
#                                        v                    v
# - -------------------- ------- ------------------- ---------------------
# | |      imx-boot      |  env  |      fitImage     |        rootfs       |
# - -------------------- ------- ------------------- ---------------------
# ^ ^                    ^       ^                   ^
# | |                    |       |                   |
# 0 32KiB                7MiB    8MiB                72MiB
#   ${IMX_BOOT_SEEK}, the offset the Boot ROM reads the bootable image from
#
# The bottom row shows start offsets, not partition sizes. imx-boot starts
# at 32 KiB and is currently about 2.2 MiB, so it fits well before the
# reserved environment area at 7 MiB.
#
# Two addresses defined outside this file must match this layout. U-Boot reads
# its environment at CONFIG_ENV_OFFSET = 0x700000 (7 MiB), while secure_bootcmd
# reads the FIT from fit_blk = 0x4000 MMC blocks (8 MiB). Neither address is
# discovered from the partition layout at run time.
#
# The offsets are therefore explicit. Alignment only rounds the current
# position up to the requested boundary, so a larger image before an aligned
# partition can silently move that partition. An explicit --offset instead
# makes wic fail if the requested location is already occupied.
#
# U-Boot writes only CONFIG_ENV_SIZE = 0x4000 bytes (16 KiB) into the 1 MiB
# reserved for the environment. If the environment layout or size changes,
# the reserved area must be reviewed together with the FIT offset.
#
# These values must stay synchronized with the board environment
# (the u-boot-imx patch in this layer):
#   fit_blk = 0x4000  MMC blocks =  8 MiB  (offset)
#   fit_cnt = 0x20000 MMC blocks = 64 MiB  (size)

# The bootable image, at the offset expected by the Boot ROM. --no-table keeps
# it out of the partition table.
part imx-boot   --source rawcopy --sourceparams="file=imx-boot.tagged" --ondisk mmcblk0 --no-table --align ${IMX_BOOT_SEEK}

# Reserves the area saveenv writes to. It holds no data and gets no partition
# table entry. If imx-boot ever grew beyond 7 MiB, wic would fail here instead
# of producing an image whose bootloader overlaps the environment.
part u-boot-env --ondisk mmcblk0 --no-table --offset 7M --fixed-size 1M

# The raw FIT partition, which Linux sees as mmcblk0p1. Its offset and size are
# kept in sync with fit_blk and fit_cnt. Wic fails the build if the FIT
# outgrows its 64 MiB partition or if the requested offset is already occupied.
part fit        --source rawcopy --sourceparams="file=fitImage" --ondisk mmcblk0 --offset 8M --fixed-size 64M

# The root filesystem, mmcblk0p2. Nothing outside the image depends on where it
# starts, so it follows the FIT partition rather than being pinned.
part /          --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 8192

# An MBR partition table rather than GPT.
bootloader --ptable msdos
EOF
```

The `.wks` file references `imx-boot.tagged` rather than `flash_singleboot`. The Yocto BSP generates this file automatically by appending a **40-byte trailer** containing the bootable image size to `flash_singleboot`.

The trailer lets `uuu` locate the bootable image inside a standalone disk image such as a `.wic` file. If you give `uuu` only a `.wic`, without a separate bootloader file, it has to extract the bootable image from inside the `.wic`, load it into RAM and use it to bring up Fastboot. The trailer tells it where that image ends. You still flash `flash_singleboot` itself, `imx-boot.tagged` is only used when embedding the bootable image into the `.wic`.

A practical note is worth keeping in mind. This partition is sized for the FIT image, not the kernel. The kernel is gzip compressed inside the FIT, so our 34 MiB uncompressed kernel results in a 15.5 MiB FIT image. A 64 MiB partition therefore leaves comfortable room for future additions such as an initramfs. The uncompressed kernel size has a separate limit but that limit is about RAM, not storage. A larger FIT image does not reduce that space. The kernel expands upward from `0x80400000` toward the FIT staging address, while the staged FIT occupies `fit_cnt` bytes upward from `0x83000000` into free memory. Only a growing kernel can close this gap and eventually overwrite the staged FIT.

## 4. Add the FIT settings to the machine configuration

In [section 4 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#4-create-a-custom-machine-configuration), we created a custom machine configuration that contained only:

```bash
require conf/machine/imx93-11x11-lpddr4x-frdm.conf
```

We now extend that same machine configuration to enable FIT image generation and signing with the following:

```bash
cat > ../sources/meta-frdm-imx93-security/conf/machine/frdm-imx93-secure.conf <<'EOF'
require conf/machine/imx93-11x11-lpddr4x-frdm.conf

# Makes the kernel artifacts available to the `linux-yocto-fitimage` recipe that builds the FIT image.
KERNEL_CLASSES += "kernel-fit-extra-artifacts"

# Ensures `linux-yocto-fitimage` is built before `wic`, so `fitImage` is available when the disk image is assembled.
WKS_FILE_DEPENDS:append = " linux-yocto-fitimage"

# Enables FIT signing and embeds the public key into U-Boot's control device tree, marked as `required`.
UBOOT_SIGN_ENABLE  = "1"

# Tell the build where the signing key is and what it is called.
FIT_KEYS_DIR      ??= ""
UBOOT_SIGN_KEYDIR  ?= "${FIT_KEYS_DIR}"
UBOOT_SIGN_KEYNAME = "fit_signing_key"

# Uses RSA-4096 signatures, matching the key generated in the previous section.
FIT_SIGN_ALG = "rsa4096"

# Explicitly selects SHA-256 for image hashes.
FIT_HASH_ALG = "sha256"

# Signs FIT configurations rather than individual images, following U-Boot's recommended model.
FIT_SIGN_INDIVIDUAL = "0"

# Selects the custom partition layout we created for raw FIT image.
WKS_FILE = "frdm-imx93-secure.wks.in"
EOF
```

Point the build to your FIT signing key directory, either in `conf/local.conf` or through the environment:

```bash
export FIT_KEYS_DIR=/path/to/fit-keys
export BB_ENV_PASSTHROUGH_ADDITIONS="$BB_ENV_PASSTHROUGH_ADDITIONS FIT_KEYS_DIR"
```

`FIT_KEYS_DIR` must be added to `BB_ENV_PASSTHROUGH_ADDITIONS`, otherwise BitBake does not import it from the shell environment and the build fails when it tries to access the signing key.

## 5. Build the signed FIT image

```bash
bitbake -c cleansstate imx-image-core && bitbake imx-image-core
```

The build now generates the signed FIT image and the new bootable image. It assembles a `fitImage` from the kernel and its device trees, signs it with `fit_signing_key.key`, embeds the matching public key from `fit_signing_key.crt` into U-Boot's control device tree, rebuilds the bootable image with the updated U-Boot and packages everything into the new `.wic` image using our custom partition layout.

## 6. Verify the FIT signature before you flash

Before writing anything to the board, verify that the generated FIT image is correctly signed and that its configuration signature can be verified with the FIT public key embedded in U-Boot's control device tree. 
U-Boot provides a host utility called `fit_check_sign` for this purpose. It is built alongside `mkimage` under `tools/fit_check_sign` in the U-Boot build directory.

The verification requires two files:

- The generated FIT image.
- The U-Boot control device tree containing the embedded FIT public key.

Only the `fit_check_sign` utility itself has to come from the U-Boot build directory. The `fitImage` and `u-boot.dtb` artifacts are available in the deploy directory. Adjust the path to match your build.

```bash
export UBOOT_BUILD="$(pwd)/tmp/work/frdm_imx93_secure-poky-linux/u-boot-imx/2026.04/build/imx93_11x11_frdm_defconfig-sd"

$UBOOT_BUILD/tools/fit_check_sign -k u-boot.dtb -f fitImage
```

The important line in the output is:

```console
sha256,rsa4096:fit_signing_key+
```

The `sha256,rsa4096:fit_signing_key+` line confirms that the FIT configuration signature was successfully verified using the embedded FIT public key. The trailing `+` indicates successful signature verification.

Do not rely on the final `Signature check OK` message alone. The tool can print that message and exit with status 0 even when the supplied device tree contains no FIT public key. In that case, it only checks the image hashes and does not perform signature verification.

If `sha256,rsa4096:fit_signing_key+` line is missing, the build may still have completed successfully but the FIT configuration signature is not being verified against the public key embedded in U-Boot. In that case, U-Boot will reject the FIT image during boot.

## 7. Sign the bootable image

Embedding the FIT public key modifies U-Boot's control device tree, which changes the U-Boot binary itself. The build then produces a new bootable image and it must be signed.

Copy the generated `imx-boot-frdm-imx93-secure-sd.bin-flash_singleboot` from the deploy directory into the `artifacts-and-tools` directory we created in [section 9 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#9-create-the-srks-super-root-keys), alongside `sign_config.yaml`, then run:

```bash
nxpimage -v ahab sign \
    -c sign_config.yaml \
    -b imx-boot-frdm-imx93-secure-sd.bin-flash_singleboot \
    -o signed-flash.bin \
    --force

nxpimage -v ahab verify -f mimx9352 -b signed-flash.bin
```

The signing process itself is unchanged. We still use the same SRKs and the same `sign_config.yaml`. The only difference is that the bootable image now contains an updated U-Boot with the embedded FIT public key.

## 8. Flash, boot and verify both stages

At this point, two build artifacts are required for flashing:

- `signed-flash.bin` which is the signed bootable image containing U-Boot with the embedded FIT public key.
- `imx-image-core-frdm-imx93-secure.rootfs-<date>.wic.zst` which is the complete eMMC image containing the raw signed FIT partition and the root filesystem.

Copy the generated `.wic.zst` image from the deploy directory into the same `artifacts-and-tools` directory if you want all flashing artifacts in one place. Set the board's boot mode switches to Serial Download Mode, power-cycle or reset the board and run `nxpuuu list-devices` from the host to confirm that the board is detected. Then flash the newly signed bootable image together with the new full system image:

```bash
nxpuuu write -b emmc_all signed-flash.bin imx-image-core-frdm-imx93-secure.rootfs-<date>.wic.zst
```

Once flashing completes, return the boot mode switches to eMMC boot and power-cycle or reset the board and let the system boot normally.

The first thing to verify is that U-Boot successfully verifies the FIT image before handing control to Linux. The boot log must contain lines like:

```console
Hit any key to stop autoboot: 0
Booting signed FIT image ...
switch to partitions #0, OK
mmc0(part 0) is current device
MMC read: dev # 0, block # 16384, count 131072 ... 131072 blocks read: OK
## Loading kernel (any) from FIT Image at 83000000 ...
   Using 'conf-imx93-11x11-frdm.dtb' configuration
   Verifying Hash Integrity ... sha256,rsa4096:fit_signing_key+ OK
   Trying 'kernel-1' kernel subimage
     Description:  Linux kernel
     Created:      2011-04-05  23:00:00 UTC
     Type:         Kernel Image
     Compression:  gzip compressed
     Data Start:   0x83000128
     Data Size:    15664288 Bytes = 14.9 MiB
     Architecture: AArch64
     OS:           Linux
     Load Address: 0x80400000
     Entry Point:  0x80400000
     Hash algo:    sha256
     Hash value:   7068673e7c8494073dca2eddb697d11ae0d74c817736a5f312fc82293e506542
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt (any) from FIT Image at 83000000 ...
   Using 'conf-imx93-11x11-frdm.dtb' configuration
   Verifying Hash Integrity ... sha256,rsa4096:fit_signing_key+ OK
   Trying 'fdt-imx93-11x11-frdm.dtb' fdt subimage
     Description:  Flattened Device Tree blob
     Created:      2011-04-05  23:00:00 UTC
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x83ef06e0
     Data Size:    61248 Bytes = 59.8 KiB
     Architecture: AArch64
     Hash algo:    sha256
     Hash value:   a2d785983901d9b8c572f4c007c58a23eb15e10f62acf2697f3200784a711b5f
   Verifying Hash Integrity ... sha256+ OK
   Booting using the fdt blob at 0x83ef06e0
Working FDT set to 83ef06e0
   Uncompressing Kernel Image to 80400000
   Loading Device Tree to 000000008ffee000, end 000000008fffff3f ... OK
Working FDT set to 8ffee000

Starting kernel ...
```

The key line is `sha256,rsa4096:fit_signing_key+ OK`. It confirms that U-Boot successfully verified the FIT image using the embedded public key before booting Linux.

After Linux has booted successfully, reboot the board, stop at the U-Boot prompt and verify that `ahab_status` reports no authentication events:

```console
u-boot=> ahab_status
Lifecycle: 0x00000008, OEM Open


	No Events Found!
```

At this point, the complete secure boot chain is working:

- The ELE firmware authenticated the OEM containers.
- U-Boot verified the signed FIT image before booting Linux.

> ⚠️ If the FIT verification messages are missing, verification fails or `ahab_status` reports authentication events, stop here and investigate before continuing. The remaining sections assume that both stages are working correctly.

## 9. Prove that FIT verification is actually enforced

A successful boot proves only that the valid path works. It does not prove that U-Boot actually enforces FIT verification or rejects modified or untrusted FIT images. To show that FIT verification is really being enforced, rather than just enabled, we perform three negative tests.

Each test exercises a different security property:

- **Integrity.** Modifying the contents of a signed FIT image must be detected by hash verification.
- **Signature requirement.** An unsigned FIT image must be rejected.
- **Key authenticity.** A FIT image signed with an untrusted key must also be rejected.

None of these tests permanently modifies the eMMC. The first test corrupts a copy of the FIT image after it has been loaded into RAM with `mmc read`. The other two stage temporary FIT images into RAM using `fastboot`. In every case, a reset restores the board to its original state.

### Test 1: A FIT with modified kernel data

We first load the same FIT image that `secure_bootcmd` normally boots from eMMC, corrupt a few bytes inside the kernel payload on purpose and then attempt to boot it. Run the following commands at the U-Boot prompt:

```console
u-boot=> mmc dev 0
u-boot=> mmc read 0x83000000 0x4000 0x20000
u-boot=> mw.b 0x83010000 0xff 16
u-boot=> bootm 0x83000000
```

`0x83010000` is 64 KiB into the staged FIT image. From the successful boot log in the previous section, we know that the kernel payload starts at `0x83000128`. So the overwrite lands inside the kernel payload rather than the FIT header or metadata. The FIT structure itself remains valid but the kernel data no longer matches the signed hash stored in the FIT, so its hash check must fail.

```console
## Loading kernel (any) from FIT Image at 83000000 ...
   Using 'conf-imx93-11x11-frdm.dtb' configuration
   Verifying Hash Integrity ... sha256,rsa4096:fit_signing_key+ OK
   Trying 'kernel-1' kernel subimage
     Description:  Linux kernel
     Created:      2011-04-05  23:00:00 UTC
     Type:         Kernel Image
     Compression:  gzip compressed
     Data Start:   0x83000128
     Data Size:    15664288 Bytes = 14.9 MiB
     Architecture: AArch64
     OS:           Linux
     Load Address: 0x80400000
     Entry Point:  0x80400000
     Hash algo:    sha256
     Hash value:   7068673e7c8494073dca2eddb697d11ae0d74c817736a5f312fc82293e506542
   Verifying Hash Integrity ... sha256 error!
Bad hash value for 'hash-1' hash node in 'kernel-1' image node
Bad Data Hash
ERROR -2: can't get kernel image!
```

The important detail is that the FIT configuration signature still verifies successfully. This is expected because neither the configuration node, its signature nor the stored image hashes were modified. The failure occurs one step later, when U-Boot calculates the hash of the kernel payload and compares it with the signed hash stored in the FIT. Since the kernel data was modified in RAM, the calculated hash no longer matches the signed hash and U-Boot aborts the boot process.

### Test 2: An unsigned FIT

The first test proved that the kernel payload inside a signed FIT cannot be modified. This test verifies a different property: U-Boot refuses to boot a FIT image that carries no signature at all. This is the purpose of the `required = "conf"` property added to U-Boot's control device tree.

Build a FIT image without a signature node. `-f auto` tells `mkimage` to assemble the FIT automatically, so no `.its` file is needed. Instead, the kernel and device tree are provided directly with `-d` and `-b`. Both are referenced by relative path, so run the commands from the deploy directory that contains them. Adjust the `mkimage` path to match your build.

```bash
export MKIMAGE="$(pwd)/tmp/work/frdm_imx93_secure-poky-linux/u-boot-imx/2026.04/build/imx93_11x11_frdm_defconfig-sd/tools/mkimage"

$MKIMAGE -f auto -A arm64 -O linux -T kernel -C gzip \
    -a 0x80400000 -e 0x80400000 -n "unsigned test" \
    -d linux.bin -b imx93-11x11-frdm.dtb \
    /tmp/fitImage-unsigned
```

Start Fastboot from the U-Boot prompt, stage the unsigned FIT image into RAM, leave Fastboot with **Ctrl+C** and then boot the staged image:

```console
u-boot=> fastboot 0
```

```bash
sudo fastboot stage /tmp/fitImage-unsigned
```

```console
u-boot=> bootm 0x82800000
```

`0x82800000` is `CONFIG_FASTBOOT_BUF_ADDR`, where `fastboot stage` places the downloaded image. The transfer happens entirely in RAM, so the eMMC is never modified and a reset restores the original system.

```console
## Loading kernel (any) from FIT Image at 82800000 ...
   Using 'conf-1' configuration
   Verifying Hash Integrity ...  error!
No 'signature' subnode found for '<NULL>' hash node in 'conf-1' config node
Failed to verify required signature 'key-fit_signing_key'
Bad Data Hash
ERROR -2: can't get kernel image!
```

The failure occurs before U-Boot verifies either the kernel or the device tree because the selected configuration carries no signature. Since the embedded FIT public key is marked as `required`, a valid configuration signature is mandatory. Without one, U-Boot rejects the FIT image immediately, so verification never even reaches the kernel or device tree.

### Test 3: A FIT signed with an untrusted key

Generate a second RSA-4096 key and sign the same `.its` with that key instead of the trusted one. The key files must use the same basename as the original (`fit_signing_key`) because the `key-name-hint` property inside the `.its` determines the filenames to use, while `-k` only specifies the directory containing them.

```bash
mkdir -p /tmp/untrusted-keys

openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 \
    -out /tmp/untrusted-keys/fit_signing_key.key

openssl req -batch -new -x509 -key /tmp/untrusted-keys/fit_signing_key.key \
    -out /tmp/untrusted-keys/fit_signing_key.crt

$MKIMAGE -f fit-image.its -k /tmp/untrusted-keys /tmp/fitImage-untrusted
```

Start Fastboot from the U-Boot prompt, stage the FIT image signed with the untrusted key into RAM, leave Fastboot with **Ctrl+C** and then boot the staged image:

```console
u-boot=> fastboot 0
```

```bash
sudo fastboot stage /tmp/fitImage-untrusted
```

```console
u-boot=> bootm 0x82800000
```

```console
## Loading kernel (any) from FIT Image at 82800000 ...
   Using 'conf-imx93-11x11-frdm.dtb' configuration
   Verifying Hash Integrity ... sha256,rsa4096:fit_signing_key-  error!
Verification failed for '<NULL>' hash node in 'conf-imx93-11x11-frdm.dtb' config node
Failed to verify required signature 'key-fit_signing_key'
Bad Data Hash
ERROR -2: can't get kernel image!
```

Unlike Test 2, this FIT image does contain a configuration signature. The failure is that the signature was generated with a private key other than the one U-Boot trusts, so it cannot be verified with the public key embedded in U-Boot's control device tree.

The `sha256,rsa4096:fit_signing_key-` line is the counterpart of the successful `+` case shown earlier. The trailing `-` indicates that signature verification failed. U-Boot therefore cannot satisfy the required signature and refuses to boot the FIT image.

Together, the three tests demonstrate the complete FIT trust model:

- **Integrity:** modifying the kernel payload inside a signed FIT is detected by hash verification.
- **Signature requirement:** an unsigned FIT is rejected because a valid configuration signature is required.
- **Key authenticity:** a FIT signed with any private key other than the trusted one is rejected during signature verification.

All three tests operate entirely in RAM. The eMMC contents remain unchanged, so a reset or power-cycle restores the original verified boot path.

## 10. Move the device to the OEM Closed lifecycle

In [section 15 of Part 1](/posts/secure-boot-on-the-frdm-imx93-part-1-the-bootloader-ahab/#15-the-final-step-closing-the-device-not-in-this-part), we left the device in the `OEM Open` lifecycle state on purpose. At that point, only the authentication of the OEM containers by the ELE firmware had been confirmed.

Now, this part completed the second stage of the secure boot chain by adding FIT signature verification in U-Boot and confirming that both stages correctly accept valid images. With the complete chain now verified end to end, the device can safely be transitioned to the `OEM Closed` lifecycle.

> ⚠️ **Before closing the device:** Do not proceed if `ahab_status` reports any authentication events related to the current boot. In particular, verify that the device reports `No Events Found!` and that the fused OEM SRK hash matches the SRK table used to sign the bootable image. If the wrong SRK hash was programmed into the fuses or the corresponding signing keys are no longer available, closing the device make the board permanently unbootable. In `OEM Open`, authentication failures may still be reported without stopping the boot process. After transitioning to `OEM Closed`, those failures are enforced and the device will reject the image. Because the lifecycle transition is irreversible, there is no safe way to recover from closing a device that does not have a known good authenticated boot image and the corresponding signing keys.

> ⚠️ **Before continuing:** Boot the board, interrupt autoboot, stop at the `u-boot=>` prompt and close your serial terminal. In this guide, `nxpele` communicates with U-Boot through the serial backend (`-d uboot_serial`) over `/dev/ttyACM0`, so the serial port must not be in use when the command is executed.

Run:

```bash
nxpele -p /dev/ttyACM0 \
    -d uboot_serial \
    -f mimx9352 \
    forward-lifecycle-update -l OEM_CLOSED
```

A successful execution prints:

```console
Forward Lifecycle update ends successfully.
```

After the command completes, reconnect to the serial console, interrupt autoboot if necessary and verify the new lifecycle state:

```console
u-boot=> ahab_status
Lifecycle: 0x00000020, OEM closed

        No Events Found!
```

`OEM Closed` confirms that the lifecycle transition completed successfully, while `No Events Found!` confirms that the ELE firmware has no authentication events to report for the current boot. The board should also continue booting normally, showing that the ELE firmware still authenticates the OEM containers successfully and that U-Boot can still verify the signed FIT image.

### Enforcement also covers the USB recovery path

Closing the device does not only protect the eMMC boot path. Serial Download Mode is simply another boot path, not a way around secure boot. Before Fastboot can write anything, `uuu` must first boot the bootable image it transfers to the board and the ELE firmware authenticates that image exactly the same way as when booting from eMMC.

To demonstrate this, generate a second set of SRKs (Super Root Keys) with `nxpcrypto`. Be careful not to overwrite the SRKs you previously generated and used for programming the fuses. If those keys are lost or overwritten, you no longer be able to sign bootable images that existing devices will accept, which leave the device permanently **bricked**. Then update `sign_config.yaml` to use this second, untrusted set and sign the bootable image again:

```bash
for i in 0 1 2 3; do
    nxpcrypto key generate -k rsa4096 -o untrusted-keys/untrusted_srk_${i}.pem
done
```

```bash
nxpimage ahab sign \
    -c sign_config.yaml \
    -b imx-boot-frdm-imx93-secure-sd.bin-flash_singleboot \
    -o signed-flash-untrusted.bin \
    --force
```

The result is a well-formed image and `nxpimage -v ahab verify -f mimx9352 -b signed-flash-untrusted.bin` reports no errors. The container carries its own SRK table together with a signature that matches it. The only difference is that the hash of this table no longer matches the fused `OEM_SRKH` value and no host side check can detect this mismatch.

Set the board's boot mode switches to Serial Download Mode, power-cycle or reset the board and run `nxpuuu list-devices` from the host to confirm that the board is detected. Then try to flash the bootable image signed with the untrusted SRKs:

```bash
nxpuuu write -b emmc signed-flash-untrusted.bin
```

After running the command, you see an error like this:

```console
SPSDKError: SPSDK: wait_uuu_finish: Failed while executing UUU command SDPS: boot -f signed-flash-untrusted.bin(exit code: 0, status: -1)
Error details: HID(W): LIBUSB_ERROR_NO_DEVICE (-4)
```

The failure happens before Fastboot even starts. The first command executed by `uuu` is:

```console
SDPS: boot -f signed-flash-untrusted.bin
```

This hands the image to the Boot ROM so that U-Boot can start and expose the Fastboot interface. The ELE firmware authenticates the OEM containers inside that image against the fused `OEM_SRKH`, exactly as it does on every boot. What the `OEM Closed` lifecycle changes is the consequence: the authentication failure is now fatal instead of simply being reported. The OEM containers never execute, the board immediately disconnects from USB and Fastboot never becomes available. As a result, no data is written to the eMMC.

From this point onward, only bootable images whose OEM containers are signed with private keys corresponding to the SRK table whose hash matches the fused `OEM_SRKH` can boot or be used through the USB recovery path.

## 11. Beyond this guide: what we did not cover

Secure boot is only one layer of a production security architecture. It establishes a chain of trust during boot but it does not by itself secure the system once Linux is running.

A production device should also consider:

* **U-Boot hardening**, such as protecting the environment, disabling unnecessary commands, restricting console access and implementing rollback protection.
* **Kernel hardening**, including a secure kernel configuration, module restrictions, memory protection features and attack surface reduction.
* **Userspace hardening**, for example least-privilege services, read-only filesystems, SELinux or AppArmor and secure update mechanisms.
* **Hardware and manufacturing security**, such as lifecycle management, debug interface lockdown, secure key storage and protection of signing keys in an HSM or other controlled key management system.

These topics are outside the scope of this guide, which focuses on building a complete secure boot chain with AHAB and FIT. They are also important for a production deployment and should be considered as part of a broader defense in depth strategy.