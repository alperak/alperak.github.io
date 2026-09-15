---
title: "Part 3: The Root Filesystem (dm-verity) - Embedded Linux Security on the NXP FRDM i.MX93"
date: 2026-09-15 10:00:00 +0300
categories: [Embedded Linux, Secure Boot]
tags: [secure-boot, security, dm-verity, rootfs, erofs, yocto, nxp, imx93, embedded-linux]
---

In [Part 2: The Kernel (Signed FIT)](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/), we extended the chain of trust from U-Boot to the Linux kernel. U-Boot verifies the FIT configuration signature, then verifies the kernel and device tree against the hashes that signature covers. Both are therefore trusted before control passes to the kernel.

![Where Part 2 left us: the root of trust is the Boot ROM, the ELE ROM and the fuses and from there the ELE firmware authenticates the SPL with the DDR firmware and then ATF, OP-TEE and U-Boot, all signed with our SRKs. U-Boot then verifies the signed FIT carrying the kernel and the device tree, so the chain now reaches the Linux kernel. The chain then breaks because nothing checks the root filesystem, which is whatever happens to be on the eMMC](/assets/img/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/part3-the-gap.png){: width="1972" height="362" }

At that point the chain of trust reaches the kernel but stops there. The kernel then mounts a root filesystem that nothing in the boot chain has checked. System binaries, shared libraries and the init process are read from the eMMC without integrity verification, so an attacker who can modify that partition can change what the running system executes, even though every stage before it carried a valid signature.

## What this part builds

In this part, we extend the chain of trust to the root filesystem with dm-verity:

- The root filesystem becomes a read-only EROFS image with a dm-verity hash tree appended to it.
- An initramfs sets up the verity device and switches root onto it.
- The root hash travels in that initramfs, inside the signed FIT.
- Corrupting a block on the device then demonstrates that reading it back returns an I/O error instead of the corrupted contents.

> This series uses the Yocto Project **Wrynose 6.0 LTS** release with NXP's `imx-6.18.20-2.0.0` BSP manifest. The [`meta-frdm-imx93-security`](https://github.com/alperak/meta-frdm-imx93-security) layer holds the finished form of every recipe, patch, configuration file and build change these parts describe.

## What dm-verity is and how it works

dm-verity is a device-mapper target in the Linux kernel. Device-mapper creates virtual block devices on top of underlying storage and each target defines how I/O is handled through that mapping. A dm-verity target creates a virtual device that is read-only and verifies the integrity of data as it is read. The filesystem is mounted from the dm-verity mapping rather than directly from the partition that contains the data. If verification succeeds, the requested data is returned normally. If verification fails, the default behaviour is to return an I/O error instead of returning the corrupted data. Later in this part, we will look at the other responses dm-verity can use when verification fails.

dm-verity verifies data using a tree of hashes. The lowest level contains one digest for each data block. The levels above it contain digests of the hash blocks below them. With the 4096 byte hash blocks and SHA-256 used in this part, each hash block can hold 128 digests. Each level becomes much smaller than the level below it until the tree reaches its root. This structure is commonly called a Merkle tree, although the kernel documentation usually calls it a hash tree.

The root hash is the trusted value that anchors the whole tree. When a data block is read, dm-verity verifies its digest and the related hash blocks up the tree against that root hash. It can therefore verify one block without hashing the entire filesystem.

A dm-verity mapping uses a data device and a hash device. They can be separate devices or the same device with the hash tree stored after the data it protects. This part uses the second approach: the build appends the hash tree to the filesystem image.

In this part, building the hash tree and activating the mapping are both done with `veritysetup`, a tool from the cryptsetup project. The kernel's dm-verity target then performs the actual verification. `veritysetup format` runs over a finished filesystem image and creates the hash tree and root hash. `veritysetup open` activates the dm-verity mapping using that root hash and the parameters that describe the layout. `create` is an older syntax for the same activation operation that `veritysetup` still accepts and that `meta-security` uses later in this part.

## Why dm-verity rather than hashing the image once

The obvious alternative is to hash the entire root filesystem image, protect that hash with a signature and verify the whole image once during boot. That works, but it has two limitations:

- A complete image verification requires reading and hashing the entire root filesystem before the system can continue booting. This work has to be repeated on every boot, even if the system only accesses a small part of the filesystem.
- The verification also says nothing about changes made afterwards. If the storage is modified after the verification has finished, that earlier verification cannot detect the change.

dm-verity avoids both problems by verifying data when it is read. Blocks that are never read do not need to be verified. If a protected block changes, the mismatch is detected when that block is later read through the dm-verity mapping. Integrity verification therefore remains active while the system is running instead of ending after a single verification during boot.

dm-verity also supports the [`check_at_most_once`](https://docs.kernel.org/admin-guide/device-mapper/verity.html) option. With this option, a data block is verified only the first time it is read. Later reads of the same block skip verification. This reduces verification overhead, but it also weakens the protection because a block changed after its first successful verification may not be detected during the same boot.

## Why EROFS and not SquashFS

dm-verity does not require a specific filesystem. Filesystems such as ext4 and btrfs can be used on top of it when mounted read-only. SquashFS and EROFS are different because they are designed as read-only image filesystems. Between those two, we chose EROFS because its design fits the access pattern we expect from the root filesystem.

The root filesystem is often read in small and scattered pieces rather than in one sequential pass, because executables and shared libraries are paged in as they run rather than read from end to end. EROFS is designed for efficient random access and aims to reduce unnecessary I/O and memory use.

Compression layout is one of the main differences between EROFS and SquashFS. EROFS mainly uses fixed-size output compression, where variable amounts of input data are compressed into physical clusters with a fixed output size. EROFS is designed to work well with small physical clusters, which helps random access. It also supports in place decompression, which can reduce the need for temporary buffers.

SquashFS compresses file data in fixed-size input blocks. A small random read can therefore require a compression block much larger than the requested data to be read and decompressed.

We prefer the smaller compression units and random read behaviour of EROFS, which matters more here because every read also passes through dm-verity. Within EROFS, we build the image using LZ4-HC compression. The HC part only affects the build process, as the compressor spends more time looking for better matches, while the resulting data is still decompressed as plain LZ4 at runtime.

Image size depends on the compression algorithm, block size and filesystem contents. SquashFS can produce a smaller image and EROFS can match or beat it in other configurations, so neither filesystem has an automatic advantage in image size.

A device where storage space is the main limit or where the workload is mostly sequential reads may make a different choice.

> For a more detailed comparison of EROFS and SquashFS, see [EROFS vs. SquashFS: A Gentle Benchmark](https://sigma-star.at/blog/2022/07/squashfs-erofs/)

## Where the root hash can live and where we put it

With the mechanism and the filesystem settled, the remaining question is where the trusted root hash should come from. The root hash is not secret but it must be authentic. dm-verity does not provide that authenticity itself. It expects the root hash used to create the mapping to already be trusted.

There are several ways to provide or authenticate the root hash. Most of them still need an initramfs to create the mapping, so what separates them is where the trusted value comes from:

- **The kernel command line.** A boot environment can place the root hash on the kernel command line. For example, [systemd-veritysetup-generator](https://www.freedesktop.org/software/systemd/man/latest/systemd-veritysetup-generator.html) reads `roothash=` when setting up a verity protected root filesystem. This is simple, but the root hash is only as trustworthy as the command line that carries it.

- **A signed root hash.** With `CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG`, the kernel can verify a PKCS#7 signature over the root hash against its trusted keyring. This allows the root hash to be authenticated even when it comes from an untrusted location. The cost is an additional signature verification path and the key management needed to support it.

- **A TPM.** On hardware that has one, the root hash can be sealed to a set of PCRs so that it only unseals when the measured boot state matches. This binds the hash to that state rather than simply storing it elsewhere.

- **Partition discovery from the root hash.** The [Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/) defines a convention where partition UUIDs are derived from the root hash. A system that already has the trusted root hash can then find the matching data and hash partitions without fixed device names. This is a discovery mechanism, not an authentication mechanism. The root hash still has to come from a trusted source and the convention assumes a separate hash partition. The [optional layout](#optional-the-separate-hash-layout) later in this part follows it.

- **The verity table from the kernel command line.** With `CONFIG_DM_INIT`, the kernel can create the dm-verity mapping during early boot from the `dm-mod.create` command line parameter. The complete verity table, including the root hash, can be supplied this way without an initramfs. It has the same trust requirement as the first option because the root hash still arrives through the kernel command line. The signed root hash option above does not help here, because the kernel looks that signature up in a user keyring entry and `dm-mod.create` runs before any userspace process exists to add one.

- **The initramfs.** The root hash can be stored in a file inside the initramfs. If that initramfs is already authenticated as part of the boot chain, the root hash is protected by the same chain. This does not require another signature mechanism. The cost is that the initramfs has to be rebuilt whenever the root hash changes.

We use the initramfs approach because it fits the chain of trust we already have. The initramfs is carried inside the signed FIT that U-Boot already verifies. The root hash inside it is therefore authenticated by the existing FIT verification, without introducing another key or signature mechanism.

## 1. Add meta-security to the build

[`meta-security`](https://git.yoctoproject.org/meta-security) provides the Yocto integration used for dm-verity in this part.

Add the layer:

```bash
bitbake-layers add-layer ../sources/meta-security
```

## 2. Enable dm-verity and EROFS in the kernel

The BSP defconfig does not enable dm-verity and EROFS by default. Instead of modifying the BSP defconfig directly, we add the required options through a kernel configuration fragment.

Create the bbappend that adds the fragment:

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-kernel/linux/linux-imx

cat > ../sources/meta-frdm-imx93-security/recipes-kernel/linux/linux-imx_%.bbappend <<'EOF'
# Enable kernel configurations for dm-verity and EROFS.

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI:append = " file://dm-verity.cfg"
EOF
```

Then create the fragment itself:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-kernel/linux/linux-imx/dm-verity.cfg <<'EOF'
# Device mapper support. CONFIG_BLK_DEV_DM is a module in the BSP defconfig.
# Building it into the kernel avoids having to load it from the initramfs
# before the root filesystem can be mounted.
CONFIG_MD=y
CONFIG_BLK_DEV_DM=y

# dm-verity support.
CONFIG_DM_VERITY=y

# EROFS filesystem support.
CONFIG_EROFS_FS=y

# Compressed EROFS support. The root filesystem is built with LZ4-HC.
# LZ4-HC affects compression when the image is built. The kernel still uses
# the normal LZ4 decompressor when reading the filesystem.
CONFIG_EROFS_FS_ZIP=y

# Allow EROFS decompression work to use per CPU workers.
CONFIG_EROFS_FS_PCPU_KTHREAD=y

# Extended attribute support. This is needed when the root filesystem uses
# features such as file capabilities or security labels.
CONFIG_EROFS_FS_XATTR=y

# Initramfs support. The FIT carries the initramfs as a cpio.gz archive.
# The kernel therefore needs built-in gzip decompression support.
CONFIG_BLK_DEV_INITRD=y
CONFIG_RD_GZIP=y
EOF
```

Some of these options may already be enabled by the BSP or selected through Kconfig dependencies. They are still listed here so that the kernel features required by this part can be seen in one place.

## 3. Add the dm-verity settings to the machine configuration

In [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#4-add-the-fit-settings-to-the-machine-configuration) we created `frdm-imx93-secure.conf` for the FIT and signing configurations. This part adds the dm-verity configuration without changing the existing settings.

Append it:

```bash
cat >> ../sources/meta-frdm-imx93-security/conf/machine/frdm-imx93-secure.conf <<'EOF'

# Enable dm-verity image generation. The class builds the verity image and
# records the verity parameters, including the root hash, in an `.env` file.
IMAGE_CLASSES += "dm-verity-img"

# Select the image protected by dm-verity.
DM_VERITY_IMAGE = "imx-image-core"

# Select the filesystem image used as the dm-verity data area.
DM_VERITY_IMAGE_TYPE = "erofs-lz4hc"

# Append the hash tree to the filesystem image instead of using a separate
# hash partition. This is the default in `dm-verity-img.bbclass`, but it
# is set explicitly because the partition layout in this part uses the appended
# form.
DM_VERITY_SEPARATE_HASH = "0"

# Select the initramfs that creates the dm-verity mapping before switching
# to the real root filesystem.
INITRAMFS_IMAGE = "dm-verity-image-initramfs"

# Keep the initramfs as a separate FIT ramdisk instead of bundling it into
# the kernel image. This is the default in `kernel.bbclass`, pinned here
# because the FIT layout in this part assumes the separate ramdisk.
INITRAMFS_IMAGE_BUNDLE = ""

# Build the initramfs as a gzip compressed cpio archive. This is the default
# in `bitbake.conf`, but it is set explicitly here to make the FIT ramdisk
# format clear.
# This is a compression choice, not a security setting.
INITRAMFS_FSTYPES = "cpio.gz"

# BitBake expands a `.wks.in` template before Wic sees it, so this is not
# needed here. A plain `.wks` is not expanded that way and would need every
# variable it references listed here.
#WICVARS:append = " IMX_BOOT_SEEK DM_VERITY_IMAGE DM_VERITY_IMAGE_TYPE IMAGE_NAME_SUFFIX"
EOF
```

`DM_VERITY_IMAGE_DATA_BLOCK_SIZE` and `DM_VERITY_IMAGE_HASH_BLOCK_SIZE` are left at their [dm-verity-img.bbclass](https://git.yoctoproject.org/meta-security/tree/classes/dm-verity-img.bbclass?h=wrynose#n40) defaults of 1024 and 4096 bytes. The block counts and offsets shown later follow from those two values.

Keeping the initramfs separate means the FIT contains the kernel and initramfs as separate image nodes, so the initramfs can be updated without rebuilding the kernel. Either way it is covered by the signed FIT configuration, so this is a packaging choice rather than a security one.

## 4. Write the verity image to the rootfs partition

Nothing else in the partition layout from [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#3-create-the-partition-layout) changes, so replace only that block in `frdm-imx93-secure.wks.in`:

```diff
-# The root filesystem, mmcblk0p2. Nothing outside the image depends on where it
-# starts, so it follows the FIT partition rather than being pinned.
-part /          --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 8192
+# The root filesystem, mmcblk0p2. Instead of creating the filesystem from
+# the rootfs contents, copy the complete dm-verity image into the partition.
+part /          --source rawcopy --sourceparams="file=${IMGDEPLOYDIR}/${DM_VERITY_IMAGE}-${MACHINE}${IMAGE_NAME_SUFFIX}.${DM_VERITY_IMAGE_TYPE}.verity" --ondisk mmcblk0 --align 8192
```

The `.verity` image comes from `dm-verity-img.bbclass`, which also adds the required build dependency when Wic is used, so the image is available before the Wic image is created.

## 5. Adjust the kernel command line for the verity root

The boot arguments change in one small way. The U-Boot patch we added in [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#2-enable-fit-verification-and-replace-the-boot-command), `recipes-bsp/u-boot/u-boot-imx/0001-imx93_frdm-add-secure_bootcmd-for-FIT-boot.patch`, modifies `board/nxp/imx93_frdm/imx93_frdm.env`, which U-Boot uses to build its default environment.

In that patch, change `secure_bootargs` from `rw` to `ro` and keep `root=/dev/mmcblk0p2` unchanged:

```diff
-+secure_bootargs=setenv bootargs console=ttyLP0,115200 earlycon root=/dev/mmcblk0p2 rootwait rw
++secure_bootargs=setenv bootargs console=ttyLP0,115200 earlycon root=/dev/mmcblk0p2 rootwait ro
```

Keeping `root=/dev/mmcblk0p2` is intentional. In this initramfs flow, the [dmverity](https://git.yoctoproject.org/meta-security/tree/recipes-core/initrdscripts/initramfs-framework-dm/dmverity?h=wrynose) module, installed as [80-dmverity](https://git.yoctoproject.org/meta-security/tree/recipes-core/initrdscripts/initramfs-framework.inc?h=wrynose), uses the value from `root=` to find the block device that contains the dm-verity data. The root hash and the other verity parameters come from the `dm-verity.env` file carried inside the initramfs. `veritysetup` then creates `/dev/mapper/rootfs` on top of that device and the initramfs mounts that mapping as the new root filesystem. `root=` therefore identifies the backing device, not the root filesystem the system ends up running from.

Without `root=`, the `dmverity` module cannot resolve the backing device and fails with:

```console
Root device resolution failed
```

The `ro` argument describes the root filesystem as read-only, but it is not what provides dm-verity integrity protection. The dm-verity target itself is read-only and the initramfs mounts `/dev/mapper/rootfs` with `-o ro`. Keeping `ro` in the kernel command line makes the boot arguments match the filesystem design.

## Optional: the separate hash layout

We do not use this approach in this part. So far the verity metadata and hash tree have been appended to the filesystem image and the remaining steps assume that layout. This section describes the alternative for anyone who wants to try it.

`dm-verity-img.bbclass` also supports a separate hash layout with `DM_VERITY_SEPARATE_HASH = "1"`. The filesystem and the verity data are then written as two images that need two partitions: `.verity` for the filesystem and `.vhash` for the dm-verity superblock and hash tree. The verity `.env` file records the choice as `SEPARATE_HASH=1` and adds `ROOT_UUID` and `RHASH_UUID`, one partition UUID for each image.

This layout uses two kinds of GPT identifier. The partition UUIDs are derived from the root hash, with the first 128 bits used for the data partition and the last 128 bits for the hash partition, following the [Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/). The partition type GUIDs describe what each partition contains. [dm-verity-img.bbclass](https://git.yoctoproject.org/meta-security/tree/classes/dm-verity-img.bbclass?h=wrynose#n52) takes them from `DM_VERITY_ROOT_GUID` and `DM_VERITY_RHASH_GUID`, whose defaults are the x86-64 values that the class itself notes have no effect on functionality today. An AArch64 board should still set them in `frdm-imx93-secure.conf` to the matching values from the same specification, so that tools following the specification can identify the partition types correctly:

```diff
-DM_VERITY_SEPARATE_HASH = "0"
+DM_VERITY_SEPARATE_HASH = "1"
+
+DM_VERITY_ROOT_GUID  = "b921b045-1df0-41c3-af44-4c6f280d3fae"
+DM_VERITY_RHASH_GUID = "df3300ce-d69f-4c92-978c-9bfb0f38d820"
```

Because the partition UUIDs change with the root hash, the class also generates a Wic fragment containing both partition definitions, with the derived UUIDs and configured type GUIDs already filled in. It writes that fragment to [STAGING_VERITY_DIR](https://git.yoctoproject.org/meta-security/tree/classes/dm-verity-img.bbclass?h=wrynose#n34), which is `tmp/work-shared/<machine>/dm-verity/`. The single root filesystem partition in `frdm-imx93-secure.wks.in` can then be replaced by an include of that fragment:

```diff
-part /          --source rawcopy --sourceparams="file=${IMGDEPLOYDIR}/${DM_VERITY_IMAGE}-${MACHINE}${IMAGE_NAME_SUFFIX}.${DM_VERITY_IMAGE_TYPE}.verity" --ondisk mmcblk0 --align 8192
+include ${STAGING_VERITY_DIR}/${DM_VERITY_IMAGE}.${DM_VERITY_IMAGE_TYPE}.wks.in
```

One detail in the generated fragment is worth knowing about. `dm-verity-img.bbclass` writes `--ondisk sda` into both partition definitions, while the rest of the partition layout uses `mmcblk0`. `sda` is also Wic's own default for the option and the class provides no variable to change it. Wic places all partitions into a single image regardless of the value, so the image built from this layout is still correct.

The initramfs activation path also changes. With `SEPARATE_HASH=1`, the [dmverity](https://git.yoctoproject.org/meta-security/tree/recipes-core/initrdscripts/initramfs-framework-dm/dmverity?h=wrynose) module reads `ROOT_UUID` and `RHASH_UUID` from the verity `.env` file and uses `/dev/disk/by-partuuid/` to locate the data and hash partitions. It therefore does not use the kernel command line `root=` to select the dm-verity data device in this path.

We use the appended layout instead, so the generated separate hash `.wks.in` fragment is not needed. We keep the partition layout in our own `frdm-imx93-secure.wks.in`, while `dm-verity-img.bbclass` provides the finished `.verity` image that Wic copies into the root filesystem partition.

## 6. Build and flash

Build the image again:

```bash
bitbake -c cleansstate imx-image-core && bitbake imx-image-core
```

The generated `.wic` image now contains the completed dm-verity image in the root filesystem partition.

[Section 5](#5-adjust-the-kernel-command-line-for-the-verity-root) also changed `board/nxp/imx93_frdm/imx93_frdm.env`, which changes the U-Boot binary. The bootable image we signed in [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#7-sign-the-bootable-image) therefore no longer matches this build. Repeat the AHAB signing step for the newly built bootable image before flashing the board.

Set the board to Serial Download Mode, then power cycle or reset it. From the host, confirm that the board is detected:

```bash
nxpuuu list-devices
```

Then flash the newly signed bootable image together with the new system image:

```bash
nxpuuu write -b emmc_all signed-flash.bin imx-image-core-frdm-imx93-secure.rootfs-<timestamp>.wic.zst
```

After flashing completes, return the boot mode switches to eMMC boot and power cycle or reset the board.

## 7. Verify the chain on the device

Three things should be true after boot.

**The FIT now carries the initramfs as another protected image.** [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#8-flash-boot-and-verify-both-stages) showed the kernel and device tree being verified. The boot log now also shows the `ramdisk-1` image being loaded from the signed configuration and its SHA-256 hash being verified:

```console
## Loading ramdisk (any) from FIT Image at 83000000 ...
   Using 'conf-imx93-11x11-frdm.dtb' configuration
   Verifying Hash Integrity ... sha256,rsa4096:fit_signing_key+ OK
   Trying 'ramdisk-1' ramdisk subimage
     Description:  dm-verity-image-initramfs
     Created:      2011-04-05  23:00:00 UTC
     Type:         RAMDisk Image
     Compression:  uncompressed
     Data Start:   0x83f95db0
     Data Size:    35161591 Bytes = 33.5 MiB
     Architecture: AArch64
     OS:           Linux
     Load Address: unavailable
     Entry Point:  unavailable
     Hash algo:    sha256
     Hash value:   a76a484af6d7f1ab50e17dc67238fd5d9343582176e0cb76316fc155b72b051c
   Verifying Hash Integrity ... sha256+ OK
```

**The root filesystem is mounted through the dm-verity device.** The kernel log shows EROFS being mounted on `dm-0` instead of directly on `mmcblk0p2`:

```console
[    6.224503] device-mapper: verity: sha256 using "sha256-lib"
[    6.313865] erofs (device dm-0): initialized per-cpu workers successfully.
[    6.320828] erofs (device dm-0): mounted with root inode @ nid 36.
```

The physical partition is still `/dev/mmcblk0p2`, but Linux runs from the device-mapper mapping created on top of it. The second line also confirms that the per CPU EROFS workers enabled by the kernel configuration fragment are active.

`lsblk` shows the same relationship:

```console
# lsblk
NAME         MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
mmcblk0      179:0    0  29.1G  0 disk
|-mmcblk0p1  179:1    0    64M  0 part
`-mmcblk0p2  179:2    0 645.8M  0 part
  `-rootfs   253:0    0 626.1M  1 crypt /
mmcblk0boot0 179:32   0     4M  1 disk
mmcblk0boot1 179:64   0     4M  1 disk
```

The partition is larger than the mapping because the appended verity metadata and hash tree are not exposed through `/dev/mapper/rootfs`. `RO` is `1` because the dm-verity mapping is read-only. `TYPE` reads `crypt` because `lsblk` takes it from the device-mapper UUID, which cryptsetup prefixes with `CRYPT-` for every target it creates, including dm-verity.

**The active mapping matches the image that was built.** `veritysetup` is not installed in this root filesystem, so `dmsetup` can be used to inspect the mapping after boot:

```console
# dmsetup table rootfs
0 1282152 verity 1 179:2 179:2 1024 4096 641076 160270 sha256 4971d0e6b886cca3cc34a9ef6a80c9f218d1732fecd4edad3dbc42d550c5ff94 a923d8a789c9e7cb2131bc762f1bf5ff594174db8282d132f832c84a81d4c4e3
```

These values can be compared with `imx-image-core.erofs-lz4hc.verity.env`, which `dm-verity-img.bbclass` writes under `tmp/work-shared/frdm-imx93-secure/dm-verity/`. [dm-verity-image-initramfs.bb](https://git.yoctoproject.org/meta-security/tree/recipes-core/images/dm-verity-image-initramfs.bb?h=wrynose) installs it into the initramfs as `/usr/share/misc/dm-verity.env`. Both names are part of the meta-security integration rather than anything dm-verity defines:

```bash
UUID=2136d2f6-fa65-47f1-a9d0-a99e7a1f9207
HASH_TYPE=1
DATA_BLOCKS=641076
DATA_BLOCK_SIZE=1024
HASH_BLOCKS=5050
HASH_BLOCK_SIZE=4096
HASH_ALGORITHM=sha256
SALT=a923d8a789c9e7cb2131bc762f1bf5ff594174db8282d132f832c84a81d4c4e3
ROOT_HASH=4971d0e6b886cca3cc34a9ef6a80c9f218d1732fecd4edad3dbc42d550c5ff94
DATA_SIZE=656461824
SEPARATE_HASH=0
```

The important fields line up directly:

| Field | Value | What it matches |
|---|---|---|
| Mapped size | `1282152` sectors | `1282152 × 512 = 656461824`, which matches `DATA_SIZE` |
| Format version | `1` | `HASH_TYPE` |
| Data and hash devices | `179:2` and `179:2` | `/dev/mmcblk0p2` is used for both because `SEPARATE_HASH=0` |
| Data block size | `1024` | `DATA_BLOCK_SIZE` |
| Hash block size | `4096` | `HASH_BLOCK_SIZE` |
| Data blocks | `641076` | `DATA_BLOCKS` |
| Hash tree start | `160270` | `DATA_SIZE / 4096 = 160269`, with the hash tree starting in the next hash block after the verity superblock |
| Hash algorithm | `sha256` | `HASH_ALGORITHM` |
| Root hash | `4971d0e6...` | `ROOT_HASH` |
| Salt | `a923d8a7...` | `SALT` |

The `UUID` is the identifier stored in the verity metadata, not a GPT partition UUID. It can also be seen in the device-mapper UUID:

```console
# dmsetup info rootfs
...
UUID: CRYPT-VERITY-2136d2f6fa6547f1a9d0a99e7a1f9207-rootfs
...
```

The device-mapper UUID is the `UUID` from `dm-verity.env` with its hyphens removed, behind the `CRYPT-VERITY-` prefix and followed by the mapping name passed to `veritysetup create`. The UUID, salt and root hash are generated during the build, so another build will produce different values.

At this point the complete chain can be followed on the device. U-Boot verifies the signed FIT configuration and the hashes of the images it references. The trusted root hash arrives inside the protected initramfs and is used to create the dm-verity mapping. The initramfs then mounts EROFS through that mapping, so protected filesystem data is verified by dm-verity before it is returned.

> ⚠️ If the `ramdisk-1` image is not verified in the boot log, if EROFS is mounted directly on `mmcblk0p2`, or if `dmsetup table rootfs` does not match the values produced by the build, stop here and investigate before continuing. The next section corrupts the root filesystem on purpose and assumes that dm-verity is already working correctly.

## 8. Prove that verification is actually enforced

A successful boot proves that the valid filesystem works. It does not prove that dm-verity will reject modified data.

> ⚠️ **This test is destructive.** It writes directly to the root filesystem partition, so the board must be reflashed afterwards.

On the board, overwrite part of the root filesystem data, then read through the dm-verity mapping:

```bash
dd if=/dev/urandom of=/dev/mmcblk0p2 bs=1M seek=300 count=1 conv=fsync

dd if=/dev/mapper/rootfs of=/dev/null bs=1M
```

The first command writes 1 MiB of random data starting 300 MiB into `/dev/mmcblk0p2`. It writes directly to the underlying partition because the dm-verity mapping itself is read-only. `conv=fsync` makes `dd` synchronize the output before it finishes.

The second command reads through the dm-verity mapping from the beginning. When the read reaches the modified region, dm-verity has to verify those blocks against the hash tree.

The result, with repeated corruption messages removed, is:

```console
device-mapper: verity: 179:2: data block 307200 is corrupted
Buffer I/O error on dev dm-0, logical block 76800, async page read
device-mapper: verity: 179:2: reached maximum errors
dd: error reading '/dev/mapper/rootfs': Input/output error
300+0 records out
314572800 bytes (315 MB, 300 MiB) copied
```

Both reported block numbers point to the same 300 MiB offset:

```text
307200 × 1024 = 314572800 bytes
76800 × 4096 = 314572800 bytes
```

dm-verity reports data block `307200` because this mapping uses 1024 byte data blocks. The `Buffer I/O error` line reports logical block `76800`, which corresponds to the same offset using 4096 byte blocks. The `300 MiB` reported by `dd` is the amount of data read successfully before the modified region is reached.

The important result is what `dd` does not receive. With the default corruption behaviour used here, dm-verity does not return data that fails verification. The read returns an I/O error instead.

### Verification is lazy and that is the point

Reboot with the corruption still in place and the board still comes up, reaches a login prompt and appears to work. Verification happens when data is read. This is the behaviour described earlier, now visible on the device.

Different services can therefore fail at different times as they reach the corrupted region:

```console
[   11.148543] device-mapper: verity: 179:2: data block 308216 is corrupted
[   11.158803] erofs (device dm-0): read error -5 @ 0 of nid 12772437
[   15.950573] device-mapper: verity: 179:2: data block 307276 is corrupted
[   15.965093] erofs (device dm-0): read error -5 @ 0 of nid 12772400
```

The two failures are almost five seconds apart and affect different blocks. Nothing scans the complete filesystem for corruption, so each failure appears only when something reads the affected data.

dm-verity also records this state in the mapping itself. `dmsetup status rootfs` reports `V` while every verification performed so far has succeeded and changes to `C` after a verification failure.

For a production device, returning an I/O error and continuing to run may not be the desired response. The next section looks at the available choices.

### What the device does about corruption is a separate choice

Returning an I/O error is the default dm-verity behaviour but other responses are available:

| Option | What the kernel does |
|---|---|
| no option | Returns an I/O error when verification fails. |
| `ignore_corruption` | Logs the corruption and allows the read to continue. |
| `restart_on_corruption` | Restarts the system when corrupted data is detected. |
| `panic_on_corruption` | Panics the kernel when corrupted data is detected. |

[veritysetup](https://man7.org/linux/man-pages/man8/veritysetup.8.html) exposes these modes as `--ignore-corruption`, `--restart-on-corruption` and `--panic-on-corruption`.

Restarting can be useful because a system that can no longer read part of its root filesystem may not be able to continue operating correctly. Instead of allowing different services to fail over time, the failure becomes immediately visible.

Restarting is not a recovery mechanism by itself. If the corrupted data is still present on persistent storage, the same verification failure will happen again when that data is read. If the data is needed during boot, the device can enter a restart loop. A restart policy becomes more useful when another known good system image is available.

`meta-security` does not provide a dedicated variable for selecting this runtime corruption policy. The `veritysetup` activation command is part of its [`dmverity`](https://git.yoctoproject.org/meta-security/tree/recipes-core/initrdscripts/initramfs-framework-dm/dmverity?h=wrynose) initramfs module.

For example, using `restart_on_corruption` requires adding the option to the `veritysetup` call inside that module:

```diff
     veritysetup \
         --data-block-size=${DATA_BLOCK_SIZE} \
         --hash-offset=${DATA_SIZE} \
+        --restart-on-corruption \
         create rootfs \
```

The module handles both appended and separate hash layouts, so the option should be added to both activation paths if both layouts need to be supported.

Whether you patch the module through a bbappend or ship your own copy, check that layer priority and the file search path actually select your version. Otherwise the build still succeeds with `meta-security`'s module and the intended corruption policy is silently lost.

dm-verity provides the integrity decision. How the system responds to a verification failure is a separate product decision. This part keeps the default I/O error behaviour and leaves the recovery policy for the later system design.

## 9. Where we are and what comes next

The chain of trust now reaches the running root filesystem:

- The ELE ROM authenticates the ELE firmware against NXP's fused SRK hash.
- The ELE firmware authenticates the OEM containers and therefore U-Boot, against the OEM SRK hash we programmed into the fuses.
- U-Boot verifies the FIT configuration signature, which covers the kernel, the device tree and the initramfs.
- The initramfs sets up dm-verity with a root hash that arrived inside that signed FIT.
- The kernel verifies every root filesystem block as it is read.

The FIT key is trusted because the U-Boot control device tree containing that key is itself part of the authenticated bootloader chain.

This part establishes the integrity boundary for the immutable system software. Making the root filesystem read-only means everything mutable has to live somewhere else. That storage holds writable state and may hold secrets, so what matters there is confidentiality rather than the integrity dm-verity provides.

Part 4 will start there, with dm-crypt and a LUKS2 keyslot unlocked by a credential derived from hardware-backed key material through OP-TEE.
