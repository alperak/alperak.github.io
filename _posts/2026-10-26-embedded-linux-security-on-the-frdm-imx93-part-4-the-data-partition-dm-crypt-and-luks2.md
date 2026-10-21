---
title: "Part 4: The Data Partition (dm-crypt and LUKS2) - Embedded Linux Security on the NXP FRDM i.MX93"
date: 2026-10-26 10:00:00 +0300
categories: [Embedded Linux, Secure Boot]
tags: [secure-boot, security, dm-crypt, luks2, op-tee, yocto, nxp, imx93, embedded-linux]
---

[Part 3: The Root Filesystem (dm-verity)](/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/) extended the chain of trust from the hardware root of trust to the root filesystem. The EdgeLock Enclave (ELE) authenticates the bootloader chain under AHAB, U-Boot verifies the signed FIT configuration that binds the kernel, device tree and initramfs, and dm-verity verifies root filesystem data as it is read.

This part addresses a different requirement. The chain established so far protects the integrity and authenticity of system software but it does not provide confidentiality for persistent data or solve the need for writable storage.

## What this part builds

In this part, we add encrypted persistent storage that the initramfs unlocks automatically on every boot, with no operator, stored key file or manually entered passphrase:

- A new `/data` partition holds a LUKS2 volume backed by dm-crypt.
- A Trusted Application (TA) in OP-TEE, the board's Trusted Execution Environment (TEE), derives the secret behind the unlock credential from the OP-TEE HUK (Hardware Unique Key), which on the i.MX93 is rooted in the EdgeLock Enclave (ELE). Because the HUK differs on every board, the same software image produces a different credential on each device and no key file or sealed blob needs to be stored on disk.
- An initramfs module, added to the initramfs that already sets up dm-verity, provisions the empty partition on first boot and opens the existing volume on later boots. Once the mapping is open, it instructs the TA to reject further derivation requests before control passes to the real root filesystem. Nothing running there can obtain the credential through this TA for the rest of that boot.
- The root filesystem mounts `/data` from that mapping.
- On the device, we verify the LUKS2 metadata, encrypted data on the eMMC, the credential across a reboot and the derivation lock.

> This series uses the Yocto Project **Wrynose 6.0 LTS** release with NXP's `imx-6.18.20-2.0.0` BSP manifest. The [`meta-frdm-imx93-security`](https://github.com/alperak/meta-frdm-imx93-security) layer holds the finished form of every recipe, patch, configuration file and build change these parts describe.

## Why a read-only root filesystem is not enough

dm-verity protects the integrity of the immutable system data on the read-only root filesystem but the system still needs writable persistent storage for configuration, logs and application data. dm-verity is designed for read-only data, so it cannot protect a partition that must remain writable. The root filesystem itself is left unencrypted on purpose. It is the same image on every board and contains no device specific secrets, so this design relies on integrity protection rather than encryption for the root filesystem.

| Storage need                                | Mechanism                                            |
| ------------------------------------------- | ---------------------------------------------------- |
| Integrity of immutable system data          | dm-verity on the read-only root filesystem partition |
| Confidentiality of writable persistent data | LUKS2 backed by dm-crypt on the `/data` partition    |

Without encryption, raw access to the writable storage bypasses filesystem permissions and exposes its contents.

dm-crypt is the device-mapper target that transparently encrypts a block device. LUKS2 is version 2 of the Linux Unified Key Setup. The LUKS2 volume on the `/data` partition is created and unlocked with cryptsetup, a tool from the same project as the `veritysetup` used in [Part 3: The Root Filesystem (dm-verity)](/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/).

## Where the unlock credential comes from

Encrypting the partition is only part of the problem. The harder part is unlocking it without an operator. A key file stored beside the encrypted data would be available to anyone who can read the same storage, so the unlock credential has to be rooted in the device itself.

On the i.MX93, that root comes from the EdgeLock Enclave (ELE). OP-TEE does not obtain the underlying Die Root Key (DRK). Its [`tee_otp_get_hw_unique_key()`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/core/drivers/crypto/ele/ele.c#L588) implementation calls [`imx_ele_derive_key()`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/core/drivers/crypto/ele/ele.c#L506) with the context `TEE_for_HUK_ELE`. ELE returns a device specific derived key, which OP-TEE uses as its HUK (Hardware Unique Key). If that derivation fails, OP-TEE panics.

OP-TEE does not expose the HUK itself. It derives a separate subkey for each usage through [`huk_subkey_derive()`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/core/kernel/huk_subkey.c#L61), so one subkey reveals neither the HUK nor any other subkey. This BSP uses the default implementation, which is based on HMAC-SHA256. For the interface used here, the System PTA, a pseudo Trusted Application built into OP-TEE core, prepends the calling TA's UUID to the supplied context and derives a subkey with the [`HUK_SUBKEY_UNIQUE_TA`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/core/include/kernel/huk_subkey.h#L33) usage. The result, called the derived secret in this part, is therefore bound to the device, the TA identity and the supplied context.

### Deriving rather than storing

There are two straightforward ways to bind an automatic unlock credential to the HUK:

|                            | Seal                                              | Derive                                             |
| -------------------------- | ------------------------------------------------- | -------------------------------------------------- |
| Credential                 | Randomly generated                                | Deterministically derived                          |
| Persistent unlock artefact | Sealed blob                                       | None                                               |
| Boot path                  | Load and unseal the blob                          | Re-derive from HUK, TA identity and context        |
| Main failure mode          | Loss or corruption of the blob can prevent unlock | Changing a derivation input changes the credential |

A sealed design generates a random credential, protects it with a HUK derived key inside the TEE (Trusted Execution Environment) and stores the resulting blob.

OP-TEE runs in the Arm TrustZone secure world, while Linux runs in the normal world, which OP-TEE calls the REE (Rich Execution Environment). In this BSP configuration, OP-TEE secure storage is backed by the normal-world filesystem and accessed through `tee-supplicant`, which keeps it under `/var/lib/tee`. Because the root filesystem is read-only, this image redirects writes under `/var/lib` to tmpfs, so nothing stored there survives a reboot. The supplicant is also not running in the initramfs. A sealed design would therefore need another place for the blob that can be read before the encrypted volume is opened, such as a small separate partition or the LUKS2 header. It would also need a first boot path that creates the blob.

We use derivation because it removes a persistent artefact that would otherwise have to remain paired with the volume across updates. The trade-off is that the derivation inputs become part of the volume's unlock protocol. OP-TEE returns the same derived secret for the same HUK, TA identity and context. Changing the TA UUID or context therefore changes the credential, so the new credential must be added to an existing volume before either value changes.

The ELE key derivation also depends on the device lifecycle, as NXP's [UG10163 Section 10.5.5](https://www.nxp.com/docs/en/user-guide/UG10163.pdf) notes. OP-TEE obtains its HUK through this derivation, so a volume provisioned in one lifecycle must not be assumed to remain unlockable after a lifecycle transition.

> ⚠️ **Provision only after the device has reached the lifecycle in which the encrypted volume will operate.** This series moves the device to [`OEM Closed`](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#10-move-the-device-to-the-oem-closed-lifecycle) before `/data` is provisioned. If the product will later transition to another lifecycle, provide an [independent recovery path](#why-we-chose-design-a) before the transition. After the transition, that path can be used to add the newly derived credential to the volume.

## Why LUKS2 rather than plain dm-crypt

Plain dm-crypt needs an encryption key, cipher parameters and a backing device. The derived secret could be used directly as the encryption key if the cipher and key size were chosen accordingly, but the device bound credential and the encryption key would then be the same key material.

LUKS2 adds key management around dm-crypt. The data is encrypted with a volume key, which in this design cryptsetup generates randomly when the volume is formatted. One or more keyslots allow different credentials, called passphrases in cryptsetup's terminology, to unlock the same volume key. Each credential is processed by its keyslot's PBKDF (Password Based Key Derivation Function) to derive a keyslot encryption key. This key is used to recover the protected volume key material stored in that keyslot. Once recovered, the volume key is used to activate the dm-crypt mapping.

The complete key path is therefore:

![The key path from the ELE rooted OP-TEE HUK to the 32 byte derived secret. The secret leaves the secure world, is hex encoded into a 64 byte LUKS keyslot credential and unlocks the 512-bit LUKS volume key used by dm-crypt with AES-256-XTS.](/assets/img/posts/embedded-linux-security-on-the-frdm-imx93-part-4-the-data-partition-dm-crypt-and-luks2/1-key-path.png){: width="921" height="1110" }

Hex encoding changes only the representation of the derived secret. The credential that cryptsetup receives is the 64 byte hex string, not the raw 32 byte secret.

| Capability | Why it matters |
|---|---|
| Credential rotation without data re-encryption | Keyslot credentials can be added, changed or removed without changing the volume key or re-encrypting the data, provided at least one usable keyslot remains. |
| Multiple unlock paths | A separate credential, such as a recovery credential, can protect another keyslot that unlocks the same volume key. Plain dm-crypt has no equivalent keyslot mechanism. |
| Stored volume parameters | The LUKS2 metadata stores the cipher, volume key size, sector size and keyslot PBKDF parameters, so they do not need to be reproduced in external configuration. |
| Credential verification | A wrong credential is rejected because cryptsetup verifies the recovered volume key against its digest in the LUKS2 metadata. Plain dm-crypt performs no such check, so a wrong key still creates a mapping whose data reads as random. |

LUKS2 is not required to encrypt the partition. We use it because it separates the device bound credential from the volume key and provides the key management and recovery model this design needs.

## Three design options and why we chose one

There are three useful ways to build this hardware rooted storage design. All three ultimately rely on the i.MX93's ELE rooted OP-TEE HUK. They differ in how the key material used for activation is obtained, what persistent state has to be stored and whether plaintext key material is exposed to normal-world userspace. Design C uses a kernel trusted key, a random symmetric key that a trusted key backend generates and seals, so userspace only ever sees it as an encrypted blob.

| | A: derive + LUKS2 | B: seal + LUKS2 | C: kernel trusted key + plain dm-crypt |
|---|---|---|---|
| Key material used for activation | HUK derived LUKS keyslot credential | Random LUKS keyslot credential recovered from a sealed blob | Kernel trusted key passed directly to dm-crypt |
| Additional persistent unlock artefact | None | Sealed blob | Sealed trusted key blob |
| Plaintext key material exposed to userspace | Yes, during LUKS activation | Yes, during LUKS activation | No |
| LUKS keyslots and recovery | Yes | Yes | No |
| Application specific TA | Yes | Yes | No |

LUKS activation requires cryptsetup to read the volume key and verify it against the volume key digest in the LUKS metadata, so it cannot use a kernel trusted key. Plain dm-crypt has no such metadata check and can use the trusted key directly from the kernel keyring, keeping the encryption key unavailable to userspace. Sealing itself does not require plain dm-crypt. Design B can still use LUKS2 because the unsealed credential is made available to cryptsetup.

### The NXP trusted key path

Designs A and C share the same ELE rooted hardware trust but use it differently.

![Designs A and C side by side. Design A derives a deterministic unlock credential from the ELE rooted OP-TEE HUK through its own TA, frdm_luks_ta. Design C creates a random kernel trusted key through OP-TEE, seals it using a separate HUK derived key and lets plain dm-crypt reference it from the kernel keyring without exposing the plaintext key to normal-world userspace.](/assets/img/posts/embedded-linux-security-on-the-frdm-imx93-part-4-the-data-partition-dm-crypt-and-luks2/2-design-a-vs-c.png){: width="1649" height="615" }

Design A deterministically derives the unlock credential from the ELE rooted OP-TEE HUK through its own TA, which this part adds as [`frdm_luks_ta`](#2-write-the-trusted-application). Design C instead creates a random kernel trusted key. OP-TEE's [`trusted_keys`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/ta/trusted_keys/entry.c#L87) TA uses a TA specific key derived through the System PTA to seal and unseal the trusted key for persistent storage.

There is no separate ELE trusted key backend in the Linux kernel. On the i.MX93, the TEE trusted key backend connects the kernel trusted key framework to OP-TEE. NXP documents this architecture in [UG10163 Section 10.5.5, "DM-Crypt usage on i.MX Platforms without CAAM hardware IP"](https://www.nxp.com/docs/en/user-guide/UG10163.pdf) for the i.MX 91, 93, 95 and 943 families.

The two designs also differ in how they use randomness. By default, design C's trusted key is generated by the OP-TEE RNG, which on this BSP is a software PRNG that is not fed by the ELE TRNG. Design A's device bound credential is deterministic, so it does not depend on the OP-TEE RNG. First boot provisioning still depends on the Linux kernel RNG because `cryptsetup luksFormat` generates the independent LUKS volume key. On this BSP, the ELE TRNG feeds that RNG through the hardware RNG framework, which reports `ele-trng` as its current source in `/sys/class/misc/hw_random/rng_current`.

### Where the NXP trusted key path is stronger

The NXP trusted key path provides stronger isolation of the encryption key. In the documented design, the key is kept in Secure OCRAM, a region reserved for secure-world access. The AES operations run inside OP-TEE through the Trusted Arm CE PTA. The plaintext key is never handled by userspace.

This does not protect plaintext data from a compromised running kernel. Once `/data` is mounted, the kernel can still access data through the open dm-crypt mapping. If key isolation is a priority in the threat model, design C isolates the encryption key better than the LUKS2 design used here.

### Why we chose design A

The deciding requirement is the ability to recover the volume if the device bound credential can no longer be reproduced. That credential depends on the original device's ELE and OP-TEE derivation path. If the SoC can no longer operate, the storage may still be physically readable but the credential cannot be reproduced from the storage alone.

LUKS2 provides a second path to the same volume key. An independent recovery credential can be enrolled in another keyslot and stored outside the device. This built-in recovery path is the decisive advantage of design A over design C.

## Three implementation components

The implementation spans the normal world and the secure world. Each component has one responsibility:

![The implementation is split between the secure world and the initramfs. frdm_luks_ta derives the secret through the System PTA, while frdm-luks-client hex encodes it and 81-dmcrypt controls provisioning, activation and the derivation lock.](/assets/img/posts/embedded-linux-security-on-the-frdm-imx93-part-4-the-data-partition-dm-crypt-and-luks2/3-implementation-pieces.png){: width="1248" height="563" }

- The **Trusted Application**, [`frdm_luks_ta`](#2-write-the-trusted-application), is the secure-world bridge to the System PTA. The System PTA derives the secret through its [`PTA_SYSTEM_DERIVE_TA_UNIQUE_KEY`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/lib/libutee/include/pta_system.h#L51) command, which is available only to user TAs (OP-TEE's term for TAs like ours) and not directly to normal-world applications. Our TA therefore invokes the System PTA and exposes only the operations needed by this design. It does not handle the LUKS volume or decide how the derived secret is used, so all cryptsetup policy remains in the normal world.

- The **client**, [`frdm-luks-client`](#3-write-the-client), runs in the normal world. It uses libteec, the OP-TEE client library, which communicates with OP-TEE through the kernel's `/dev/tee0` device. It opens a session to our TA, requests the derived secret and hex encodes it into the credential passed to cryptsetup. It also exposes the command used to lock further derivations.

- The **initramfs module**, [`81-dmcrypt`](#6-open-the-volume-in-the-initramfs), implements the boot policy for first boot provisioning and volume activation. It uses the client to supply the credential to cryptsetup, locks further derivations once the mapping is active and creates the filesystem on a newly provisioned volume.

## 1. Enable dm-crypt in the kernel

Add a new fragment beside [the dm-verity fragment from the previous part](/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/#2-enable-dm-verity-and-erofs-in-the-kernel):

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-kernel/linux/linux-imx/dm-crypt.cfg <<'EOF'
# Device-mapper is configured under the MD subsystem.
CONFIG_MD=y
CONFIG_BLK_DEV_DM=y

# The initramfs opens the encrypted volume, so build dm-crypt into the kernel.
CONFIG_DM_CRYPT=y

# The LUKS2 volume uses aes-xts-plain64. Keep the generic XTS and AES
# implementations as a fallback.
CONFIG_CRYPTO_XTS=y
CONFIG_CRYPTO_AES=y

# Accelerate AES block modes, including XTS, with the Armv8 Crypto Extensions.
CONFIG_CRYPTO_AES_ARM64_CE_BLK=y

# Enable the kernel keyring. The default LUKS2 activation path in cryptsetup
# uses it to keep the volume key out of the dm-crypt target table.
CONFIG_KEYS=y

# The initramfs derives the credential through /dev/tee0, so build the OP-TEE
# driver into the kernel.
CONFIG_TEE=y
CONFIG_OPTEE=y

# The encrypted volume holds the ext4 filesystem mounted at /data.
CONFIG_EXT4_FS=y

# This design does not use encrypted or trusted keys, and the NXP defconfig
# builds trusted keys as a module. If either option is built as a module,
# Kconfig limits CONFIG_DM_CRYPT to a module as well, so disable both.
# CONFIG_ENCRYPTED_KEYS is not set
# CONFIG_TRUSTED_KEYS is not set
EOF
```

Then add it to the existing kernel bbappend:

```bash
cat >> ../sources/meta-frdm-imx93-security/recipes-kernel/linux/linux-imx_%.bbappend <<'EOF'

# Enable kernel configurations for dm-crypt.
SRC_URI:append = " file://dm-crypt.cfg"
EOF
```

## 2. Write the Trusted Application

The TA and its normal-world client need to agree on the TA identity, command IDs and data sizes, so define those in a shared header:

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files

cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/frdm_luks_ta.h <<'EOF'
/* SPDX-License-Identifier: BSD-2-Clause */
/*
 * Interface shared between the FRDM i.MX93 LUKS TA and its client.
 */

#ifndef FRDM_LUKS_TA_H
#define FRDM_LUKS_TA_H

/*
 * The System PTA includes the calling TA's UUID in the derivation input.
 * Changing this UUID therefore changes the derived secret and requires a
 * migration plan for any volume provisioned with the old value.
 */
#define TA_FRDM_LUKS_UUID \
    { 0x62d8c105, 0xbc79, 0x4657, \
        { 0xbc, 0xe0, 0x2d, 0xeb, 0xee, 0xd9, 0x3f, 0xbb } }

/*
 * Derive the secret for one volume.
 *
 * [in]  memref[0]  Context identifying the volume.
 * [out] memref[1]  Exactly FRDM_LUKS_DERIVED_SECRET_SIZE bytes.
 *
 * Returns TEE_ERROR_ACCESS_DENIED after the lock command has been issued.
 */
#define TA_FRDM_LUKS_CMD_DERIVE_SECRET 0

/*
 * Refuse further derivations for the lifetime of this TA instance.
 * There is no command that clears the lock, by design.
 */
#define TA_FRDM_LUKS_CMD_LOCK          1

/* The System PTA derives at most 32 bytes. */
#define FRDM_LUKS_DERIVED_SECRET_SIZE  32

#endif /* FRDM_LUKS_TA_H */
EOF
```

The UUID above can be shared across devices because the derived secret also depends on the device specific HUK, so each board still derives its own credential. If you choose a different UUID for your product, generate one with `uuidgen` and replace every occurrence before any volume is provisioned.

The TA itself is small because the System PTA already performs the derivation:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/frdm_luks_ta.c <<'EOF'
// SPDX-License-Identifier: BSD-2-Clause
/*
 * FRDM i.MX93 LUKS TA.
 *
 * Derives the secret through OP-TEE's System PTA.
 */

#include <inttypes.h>
#include <pta_system.h>
#include <stdbool.h>
#include <string.h>
#include <string_ext.h>
#include <tee_internal_api.h>

#include "frdm_luks_ta.h"

static const TEE_UUID system_uuid = PTA_SYSTEM_UUID;

/* The TA_FLAGS in user_ta_header_defines.h keep this flag across sessions. */
static bool derivation_locked;

static TEE_Result derive_secret(uint32_t param_types, TEE_Param params[TEE_NUM_PARAMS])
{
    const uint32_t exp_param_types =
        TEE_PARAM_TYPES(TEE_PARAM_TYPE_MEMREF_INPUT,
                        TEE_PARAM_TYPE_MEMREF_OUTPUT,
                        TEE_PARAM_TYPE_NONE,
                        TEE_PARAM_TYPE_NONE);
    TEE_TASessionHandle session = TEE_HANDLE_NULL;
    TEE_Param pta_params[TEE_NUM_PARAMS] = { };
    uint8_t secret[FRDM_LUKS_DERIVED_SECRET_SIZE] = { };
    TEE_Result res = TEE_ERROR_GENERIC;
    uint32_t ret_origin = 0;

    /* The System PTA enforces the context size limit, not this TA. */
    if (param_types != exp_param_types ||
        !params[1].memref.buffer ||
        params[1].memref.size != sizeof(secret))
        return TEE_ERROR_BAD_PARAMETERS;

    if (derivation_locked) {
        EMSG("Derivation is locked");
        return TEE_ERROR_ACCESS_DENIED;
    }

    res = TEE_OpenTASession(&system_uuid, TEE_TIMEOUT_INFINITE, 0, NULL, &session, &ret_origin);
    if (res != TEE_SUCCESS) {
        EMSG("Cannot open a session to the System PTA: %#" PRIx32, res);
        goto out;
    }

    /* The System PTA requires its output buffer in secure memory. */
    pta_params[0].memref.buffer = params[0].memref.buffer;
    pta_params[0].memref.size = params[0].memref.size;
    pta_params[1].memref.buffer = secret;
    pta_params[1].memref.size = sizeof(secret);

    res = TEE_InvokeTACommand(session, TEE_TIMEOUT_INFINITE,
                              PTA_SYSTEM_DERIVE_TA_UNIQUE_KEY,
                              exp_param_types, pta_params, &ret_origin);

    TEE_CloseTASession(session);

    if (res != TEE_SUCCESS) {
        EMSG("PTA_SYSTEM_DERIVE_TA_UNIQUE_KEY failed: %#" PRIx32, res);
        goto out;
    }

    memcpy(params[1].memref.buffer, secret, sizeof(secret));

out:
    memzero_explicit(secret, sizeof(secret));
    return res;
}

static TEE_Result lock_derivation(uint32_t param_types)
{
    if (param_types != TEE_PARAM_TYPES(TEE_PARAM_TYPE_NONE,
                                       TEE_PARAM_TYPE_NONE,
                                       TEE_PARAM_TYPE_NONE,
                                       TEE_PARAM_TYPE_NONE))
        return TEE_ERROR_BAD_PARAMETERS;

    derivation_locked = true;
    IMSG("Derivation locked");

    return TEE_SUCCESS;
}

/* OP-TEE requires these entry points. This TA keeps no per session state. */
TEE_Result TA_CreateEntryPoint(void)
{
    return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void)
{
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types __unused,
                                    TEE_Param params[TEE_NUM_PARAMS] __unused,
                                    void **session __unused)
{
    return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *session __unused)
{
}

TEE_Result TA_InvokeCommandEntryPoint(void *session __unused, uint32_t cmd,
                                      uint32_t param_types,
                                      TEE_Param params[TEE_NUM_PARAMS])
{
    switch (cmd) {
    case TA_FRDM_LUKS_CMD_DERIVE_SECRET:
        return derive_secret(param_types, params);

    case TA_FRDM_LUKS_CMD_LOCK:
        return lock_derivation(param_types);

    default:
        EMSG("Command ID %#" PRIx32 " is not supported", cmd);
        return TEE_ERROR_NOT_SUPPORTED;
    }
}
EOF
```

The System PTA [enforces that secure memory requirement](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/core/pta/system.c#L84-L98), but the derived secret still enters normal-world memory briefly, because the client needs it to build the credential for cryptsetup.

`TA_FRDM_LUKS_CMD_LOCK` sets `derivation_locked` and there is no command that clears it. Derivation and locking happen in [separate client invocations](#6-open-the-volume-in-the-initramfs), so the TA instance must remain loaded after sessions close. Otherwise, a later invocation could create a new instance with `derivation_locked` cleared. If the TA crashes, it stays unavailable until OP-TEE restarts, so it fails closed instead of re-enabling derivation. A reboot restarts OP-TEE, so the TA starts with `derivation_locked` cleared on the next boot.

That lifetime policy is defined in the TA header:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/user_ta_header_defines.h <<'EOF'
/* SPDX-License-Identifier: BSD-2-Clause */

#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

#include "frdm_luks_ta.h"

#define TA_UUID TA_FRDM_LUKS_UUID

/*
 * Share one instance across sessions and keep it loaded after they close.
 * If the TA crashes, prevent OP-TEE from restarting it with the derivation
 * lock reset.
 */
#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | \
                         TA_FLAG_INSTANCE_KEEP_ALIVE | \
                         TA_FLAG_INSTANCE_KEEP_CRASHED)

#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (8 * 1024)

#define TA_DESCRIPTION  "FRDM i.MX93 LUKS TA"

#endif /* USER_TA_HEADER_DEFINES_H */
EOF
```

Finally, add the two files read by the OP-TEE TA dev kit, the build environment for user TAs:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/sub.mk <<'EOF'
# global-incdirs-y puts this directory on the include path so the dev kit's
# own user_ta_header.c finds user_ta_header_defines.h.
global-incdirs-y += .
srcs-y += frdm_luks_ta.c
EOF

cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/Makefile <<'EOF'
# ta_dev_kit.mk names its outputs after BINARY, including the
# <uuid>.stripped.elf that optee-os will later embed.
BINARY = 62d8c105-bc79-4657-bce0-2debeed93fbb

include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk
EOF
```

## 3. Write the client

The TA is invoked on demand, so a normal-world client has to open a session and invoke its commands.

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/files/frdm_luks_client.c <<'EOF'
// SPDX-License-Identifier: BSD-2-Clause
/*
 * Client for the FRDM i.MX93 LUKS TA.
 *
 * A successful derivation is printed as 64 lowercase hex characters with no
 * trailing newline.
 */

#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <tee_client_api.h>

#include "frdm_luks_ta.h"

static const TEEC_UUID ta_uuid = TA_FRDM_LUKS_UUID;

static int invoke(uint32_t cmd, const char *context)
{
    TEEC_Operation op = { };
    TEEC_Context ctx = { };
    TEEC_Session session = { };
    TEEC_Result res = TEEC_ERROR_GENERIC;
    uint8_t secret[FRDM_LUKS_DERIVED_SECRET_SIZE] = { };
    uint32_t err_origin = 0;
    int rc = EXIT_FAILURE;

    res = TEEC_InitializeContext(NULL, &ctx);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_InitializeContext failed: 0x%x\n", res);
        return EXIT_FAILURE;
    }

    res = TEEC_OpenSession(&ctx, &session, &ta_uuid, TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_OpenSession failed: 0x%x (origin 0x%x)\n", res, err_origin);
        goto out_context;
    }

    if (cmd == TA_FRDM_LUKS_CMD_DERIVE_SECRET) {
        op.paramTypes = TEEC_PARAM_TYPES(TEEC_MEMREF_TEMP_INPUT,
                                         TEEC_MEMREF_TEMP_OUTPUT,
                                         TEEC_NONE, TEEC_NONE);

        /*
         * The context length is part of the derivation input. Do not include the
         * terminating NUL, because doing so would derive a different credential.
         */
        op.params[0].tmpref.buffer = (void *)context;
        op.params[0].tmpref.size = strlen(context);
        op.params[1].tmpref.buffer = secret;
        op.params[1].tmpref.size = sizeof(secret);
    }

    res = TEEC_InvokeCommand(&session, cmd, &op, &err_origin);
    if (res != TEEC_SUCCESS) {
        fprintf(stderr, "TEEC_InvokeCommand failed: 0x%x (origin 0x%x)\n", res, err_origin);
        goto out_session;
    }

    if (cmd == TA_FRDM_LUKS_CMD_DERIVE_SECRET) {
        if (op.params[1].tmpref.size != sizeof(secret)) {
            fprintf(stderr, "unexpected derived secret size %zu\n", op.params[1].tmpref.size);
            goto out_session;
        }

        for (size_t i = 0; i < sizeof(secret); i++)
            printf("%02x", (unsigned int)secret[i]);

        if (fflush(stdout) == EOF) {
            fprintf(stderr, "failed to write credential\n");
            goto out_session;
        }
    }

    rc = EXIT_SUCCESS;

out_session:
    TEEC_CloseSession(&session);

out_context:
    TEEC_FinalizeContext(&ctx);

    explicit_bzero(secret, sizeof(secret));

    return rc;
}

int main(int argc, char *argv[])
{
    if (argc == 2 && !strcmp(argv[1], "--lock"))
        return invoke(TA_FRDM_LUKS_CMD_LOCK, NULL);

    if (argc == 3 && !strcmp(argv[1], "--context")) {
        if (argv[2][0] == '\0') {
            fprintf(stderr, "context must not be empty\n");
            return EXIT_FAILURE;
        }

        return invoke(TA_FRDM_LUKS_CMD_DERIVE_SECRET, argv[2]);
    }

    fprintf(stderr, "usage: %s --context <string>\n       %s --lock\n", argv[0], argv[0]);

    return EXIT_FAILURE;
}
EOF
```

`TEEC_LOGIN_PUBLIC` provides no login data and the TA does not authenticate the caller. The design therefore relies on boot sequencing. Only trusted initramfs code should run while derivation remains available and the initramfs locks further derivations immediately after the dm-crypt mapping opens.

## 4. Add a recipe for the TA and the client

The initramfs must open `/data` before the real root filesystem and its `tee-supplicant` service are available. User TAs are normally loaded from the REE filesystem through `tee-supplicant`, so this part embeds the TA in `tee.bin`, the OP-TEE core image, as an **early TA** instead.

The TA and client implement the two sides of the same interface, so one recipe builds them together from the shared header:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-security/frdm-luks/frdm-luks_1.0.bb <<'EOF'
SUMMARY = "Trusted Application and client for the data partition unlock credential"

LICENSE = "BSD-2-Clause"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/BSD-2-Clause;md5=cb641bc04cda31daea161b1bc15da69f"

DEPENDS = "optee-os-tadevkit python3-cryptography-native optee-client"

inherit python3native

SRC_URI = " \
    file://Makefile \
    file://sub.mk \
    file://frdm_luks_ta.c \
    file://frdm_luks_ta.h \
    file://user_ta_header_defines.h \
    file://frdm_luks_client.c \
"

S = "${UNPACKDIR}"
B = "${WORKDIR}/build"

# Must match BINARY in the Makefile and TA_FRDM_LUKS_UUID in the shared header.
TA_UUID = "62d8c105-bc79-4657-bce0-2debeed93fbb"

TA_DEV_KIT_DIR = "${STAGING_INCDIR}/optee/export-user_ta"

EXTRA_OEMAKE = " \
    TA_DEV_KIT_DIR=${TA_DEV_KIT_DIR} \
    CROSS_COMPILE=${HOST_PREFIX} \
    LIBGCC_LOCATE_CFLAGS=--sysroot=${STAGING_DIR_HOST} \
    O=${B} \
"

do_compile() {
    oe_runmake -C ${S}

    ${CC} ${CFLAGS} ${CPPFLAGS} -I${S} \
        -o ${B}/frdm-luks-client ${S}/frdm_luks_client.c ${LDFLAGS} -lteec
}

do_compile[cleandirs] = "${B}"

do_install() {
    # optee-os embeds the stripped ELF as an early TA. The REE-FS .ta image
    # normally loaded through tee-supplicant is not needed for this boot path.
    install -D -m 0644 ${B}/${TA_UUID}.stripped.elf \
        ${D}${datadir}/early-ta/${TA_UUID}.stripped.elf

    install -D -m 0755 ${B}/frdm-luks-client \
        ${D}${bindir}/frdm-luks-client
}

PACKAGES =+ "${PN}-early-ta"
FILES:${PN}-early-ta = "${datadir}/early-ta/${TA_UUID}.stripped.elf"

PACKAGE_ARCH = "${MACHINE_ARCH}"
EOF
```

The `${PN}-early-ta` package keeps the stripped TA ELF out of the runtime `frdm-luks` package, which the initramfs needs only for the client. Packaging does not limit sysroot staging and `${datadir}` is available to build time dependencies, so in the next step `optee-os` receives the ELF in its recipe sysroot and passes it through [`EARLY_TA_PATHS`](https://github.com/nxp-imx/imx-optee-os/blob/37c7fbf84c40eb9e5828532972ffb133b4917390/mk/config.mk#L406) to embed it as an early TA.

### Embed the TA in OP-TEE

Add an `optee-os` bbappend:

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-security/optee

cat > ../sources/meta-frdm-imx93-security/recipes-security/optee/'optee-os_%.bbappend' <<'EOF'
DEPENDS += "frdm-luks"

# Must match TA_UUID in frdm-luks_1.0.bb and the UUID in the shared header.
FRDM_LUKS_TA_UUID = "62d8c105-bc79-4657-bce0-2debeed93fbb"

# This TA requires user TA support and the System PTA for HUK subkey derivation.
# CFG_SYSTEM_PTA depends on CFG_WITH_USER_TA, so enable both explicitly.
# EARLY_TA_PATHS below supplies the TA ELF and enables early TA embedding.
#
# Explicitly select the ELE backed HUK implementation. The generic insecure
# HUK stub returns all zeros, which would make the same TA and context derive
# the same secret on every device.
EXTRA_OEMAKE:append = " CFG_WITH_USER_TA=y CFG_SYSTEM_PTA=y CFG_EARLY_TA=y CFG_IMX_ELE=y"
EXTRA_OEMAKE:append = " EARLY_TA_PATHS=${STAGING_DATADIR}/early-ta/${FRDM_LUKS_TA_UUID}.stripped.elf"
EOF
```

The main build artefact flow is:

![The main build artefact flow from the TA dev kit and libteec through frdm-luks, optee-os and imx-boot, showing how the stripped early TA ELF becomes part of tee.bin.](/assets/img/posts/embedded-linux-security-on-the-frdm-imx93-part-4-the-data-partition-dm-crypt-and-luks2/4-build-dependency.png){: width="1521" height="283" }

`optee-os-tadevkit` provides the TA build environment, while `optee-client` provides `libteec` for the normal-world client. `python3-cryptography-native` is needed because the dev kit always builds the signed `.ta` image as well, even though this boot path uses only the stripped ELF. The separate TA dev kit recipe allows `frdm-luks` to be built first and provide its stripped TA ELF to `optee-os` without introducing a BitBake dependency cycle.

Embedding also changes the TA authentication path. OP-TEE verifies the signature of a `.ta` image when it loads the TA, but an early TA is embedded from its stripped ELF, so there is no separate TA signature to verify at runtime. In this series, the ELE firmware authenticates `tee.bin` as part of the OEM container that also carries ATF and U-Boot. The embedded early TA is therefore authenticated as part of `tee.bin`.

OP-TEE searches early TAs before any other TA store. A TA installed elsewhere with the same UUID, such as a `.ta` file in the REE filesystem, therefore cannot override this one and obtain its derived secret.

## 5. Add the data partition to the layout

The layout from [Part 2](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#3-create-the-partition-layout) gains a third partition for persistent data. The dm-verity rootfs partition from [Part 3](/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/#4-write-the-verity-image-to-the-rootfs-partition) also gets a fixed size. Update the layout diagram at the top of `frdm-imx93-secure.wks.in` first:

```text
# mmcblk0                                p1                   p2              p3
#                                        |                    |               |
#                                        v                    v               v
# - -------------------- ------- ------------------- --------------- ---------------
# | |      imx-boot      |  env  |      fitImage     |    rootfs     |     data      |
# - -------------------- ------- ------------------- --------------- ---------------
# ^ ^                    ^       ^                   ^               ^
# | |                    |       |                   |               |
# 0 32KiB                7MiB    8MiB                72MiB           1096MiB
#   ${IMX_BOOT_SEEK}, the offset the Boot ROM reads the bootable image from
#
```

Then give the rootfs a fixed size partition:

```diff
 # The root filesystem, mmcblk0p2. Instead of creating the filesystem from
 # the rootfs contents, copy the complete dm-verity image into the partition.
-part /          --source rawcopy --sourceparams="file=${IMGDEPLOYDIR}/${DM_VERITY_IMAGE}-${MACHINE}${IMAGE_NAME_SUFFIX}.${DM_VERITY_IMAGE_TYPE}.verity" --ondisk mmcblk0 --align 8192
+#
+# The complete dm-verity image must fit within 1024 MiB, otherwise Wic fails
+# the image build. Increasing this size also requires moving the data
+# partition's --offset by the same amount.
+part /          --source rawcopy --sourceparams="file=${IMGDEPLOYDIR}/${DM_VERITY_IMAGE}-${MACHINE}${IMAGE_NAME_SUFFIX}.${DM_VERITY_IMAGE_TYPE}.verity" --ondisk mmcblk0 --offset 72M --fixed-size 1024M
```

Previously, the rootfs partition grew with the generated image. That was harmless while it was the final partition, but persistent data now follows it and needs a fixed start offset.

Finally, add the data partition after the rootfs partition:

```diff
+# The encrypted data partition, mmcblk0p3.
+#
+# Wic defaults --fstype to vfat, so --fstype=none is needed to leave the
+# partition unformatted. The device creates the LUKS2 volume and ext4
+# filesystem during first boot.
+#
+# --part-name sets the GPT partition name "data", allowing the initramfs to
+# find it through /dev/disk/by-partlabel/data before LUKS or ext4 exists.
+part            --ondisk mmcblk0 --part-name data --fstype=none --offset 1096M --fixed-size 2048M
```

The data partition is intentionally left unformatted in the generated image. The build host cannot derive the device bound credential from each board's HUK. Preformatting a single shared image would embed the same volume key and keyslot state in every copy, requiring a shared initial credential rather than the device bound credential.

## 6. Open the volume in the initramfs

The unlock path belongs in the same `initramfs-framework` that already activates dm-verity. Add it as `81-dmcrypt`, immediately after meta-security's `80-dmverity` and before `99-finish` runs `switch_root` to enter the real root filesystem.

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-core/initrdscripts/initramfs-framework-dmcrypt

cat > ../sources/meta-frdm-imx93-security/recipes-core/initrdscripts/initramfs-framework-dmcrypt/dmcrypt <<'EOF'
#!/bin/sh
#
# Opens the encrypted data partition, provisioning it on first boot.
#
# Only the mapping is opened here. /dev/mapper/<name> survives switch_root, and
# the real root filesystem mounts /data later.

DMCRYPT_NAME=data

# Public input to the credential derivation, not a secret. Changing it changes
# the credential, so migrate the LUKS keyslot first.
DMCRYPT_CONTEXT=frdm-imx93-data-v1

DMCRYPT_DEV="/dev/disk/by-partlabel/${DMCRYPT_NAME}"
DMCRYPT_MAPPER="/dev/mapper/${DMCRYPT_NAME}"

# AES-XTS uses two AES-256 keys, giving a 512-bit LUKS volume key.
DMCRYPT_CIPHER="--cipher aes-xts-plain64 --key-size 512"

# The credential comes from a 32 byte derived secret rather than a human
# passphrase, so use PBKDF2 at a fixed low cost.
DMCRYPT_PBKDF="--pbkdf pbkdf2 --pbkdf-force-iterations 1000"

# The pipeline hides the client's exit status, so make cryptsetup read
# exactly 64 bytes rather than accept a truncated credential.
DMCRYPT_KEYFILE="--key-file=- --keyfile-size=64"

dmcrypt_enabled() {
    return 0
}

# 01-udev normally discovers both devices first, but udevadm settle only waits
# for the current event queue, so keep a bounded wait.
dmcrypt_wait_for() {
    _path="$1"
    _what="$2"
    _c=0
    _delay=${bootparam_rootdelay:-1}
    _timeout=${bootparam_roottimeout:-5}

    while [ ! -e "$_path" ]; do
        if [ $((_c * _delay)) -gt "$_timeout" ]; then
            return 1
        fi

        debug "dmcrypt: waiting for ${_what} (${_path})"
        sleep "$_delay"
        _c=$((_c + 1))
    done

    return 0
}

# Derive on demand rather than retaining the credential in a shell variable.
dmcrypt_credential() {
    frdm-luks-client --context "${DMCRYPT_CONTEXT}"
}

dmcrypt_format() {
    msg "dmcrypt: provisioning ${DMCRYPT_DEV}"

    dmcrypt_credential | cryptsetup luksFormat \
        --type luks2 \
        --batch-mode \
        ${DMCRYPT_CIPHER} \
        ${DMCRYPT_PBKDF} \
        ${DMCRYPT_KEYFILE} \
        "${DMCRYPT_DEV}"
}

dmcrypt_open() {
    dmcrypt_credential | cryptsetup open \
        --type luks2 \
        ${DMCRYPT_KEYFILE} \
        "${DMCRYPT_DEV}" "${DMCRYPT_NAME}"
}

dmcrypt_run() {
    _new_volume=0

    if ! dmcrypt_wait_for "${DMCRYPT_DEV}" "data partition"; then
        fatal "dmcrypt: ${DMCRYPT_DEV} never appeared"
    fi

    if ! dmcrypt_wait_for /dev/tee0 "TEE device"; then
        fatal "dmcrypt: /dev/tee0 never appeared, cannot derive the credential"
    fi

    # isLuks fails on an unrecoverable LUKS header as well as on an empty
    # partition, so such a header is reformatted too. Production provisioning
    # needs separate state to tell the two apart.
    if ! cryptsetup isLuks "${DMCRYPT_DEV}"; then
        dmcrypt_format || fatal "dmcrypt: luksFormat failed"
        _new_volume=1
    fi

    if ! dmcrypt_open; then
        fatal "dmcrypt: cannot open ${DMCRYPT_DEV} with the derived credential"
    fi

    if ! frdm-luks-client --lock; then
        cryptsetup close "${DMCRYPT_NAME}" >/dev/null 2>&1
        fatal "dmcrypt: could not set the derivation lock"
    fi

    if [ "${_new_volume}" -eq 1 ]; then
        msg "dmcrypt: creating a filesystem on ${DMCRYPT_MAPPER}"

        if ! mkfs.ext4 -q -F -L "${DMCRYPT_NAME}" "${DMCRYPT_MAPPER}"; then
            cryptsetup close "${DMCRYPT_NAME}" >/dev/null 2>&1
            fatal "dmcrypt: mkfs.ext4 failed"
        fi
    else
        # An existing volume must already contain ext4. Treat anything else as
        # possible corruption rather than permission to create a new filesystem.
        _fstype="$(blkid -s TYPE -o value "${DMCRYPT_MAPPER}" 2>/dev/null)"

        if [ "${_fstype}" != "ext4" ]; then
            cryptsetup close "${DMCRYPT_NAME}" >/dev/null 2>&1
            fatal "dmcrypt: expected ext4 on ${DMCRYPT_MAPPER}, found '${_fstype:-none}'"
        fi
    fi

    info "dmcrypt: ${DMCRYPT_MAPPER} is open"
}
EOF
```

First boot provisioning also has an intermediate state. If power is lost after `luksFormat` but before `mkfs.ext4`, the next boot recognizes the existing LUKS volume, opens it and then stops because no ext4 filesystem is present. A production design that needs to recover from interrupted provisioning therefore requires independent provisioning state.

Package the module through an `initramfs-framework` bbappend:

```bash
cat > ../sources/meta-frdm-imx93-security/recipes-core/initrdscripts/'initramfs-framework_%.bbappend' <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/initramfs-framework-dmcrypt:"

SRC_URI:append = " file://dmcrypt"

# Keep the filename to one hyphen. initramfs-framework derives the function
# prefix with `basename $m | cut -d'-' -f 2`, so 81-dm-crypt would incorrectly
# look for dm_enabled() and dm_run().
do_install:append() {
    install -m 0755 ${S}/dmcrypt ${D}/init.d/81-dmcrypt
}

PACKAGES:append = " initramfs-module-dmcrypt"

SUMMARY:initramfs-module-dmcrypt = "initramfs dm-crypt encrypted data partition support"

FILES:initramfs-module-dmcrypt = "/init.d/81-dmcrypt"

RDEPENDS:initramfs-module-dmcrypt = " \
    ${PN}-base \
    initramfs-module-udev \
    cryptsetup \
    frdm-luks \
    e2fsprogs-mke2fs \
    util-linux-blkid \
"
EOF
```

Rather than defining a second initramfs, extend meta-security's [`dm-verity-image-initramfs`](https://git.yoctoproject.org/meta-security/tree/recipes-core/images/dm-verity-image-initramfs.bb?h=wrynose), which [Part 3](/posts/embedded-linux-security-on-the-frdm-imx93-part-3-the-root-filesystem-dm-verity/#3-add-the-dm-verity-settings-to-the-machine-configuration) already selected as `INITRAMFS_IMAGE` to set up dm-verity. The signed FIT still contains a single ramdisk, with both `80-dmverity` and `81-dmcrypt` running inside it:

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-core/images

cat > ../sources/meta-frdm-imx93-security/recipes-core/images/dm-verity-image-initramfs.bbappend <<'EOF'
# This image recipe assigns PACKAGE_INSTALL directly, so extend that variable
# rather than IMAGE_INSTALL. The module's RDEPENDS pull in the rest of the
# unlock path.
PACKAGE_INSTALL:append = " initramfs-module-dmcrypt"
EOF
```

## 7. Mount `/data` from the root filesystem

`/dev/mapper/data` remains active across `switch_root`, so the filesystem can be mounted later from the real root filesystem. This differs from `/dev/mapper/rootfs`, which must be mounted before `switch_root` can enter the verified root filesystem. Two pieces therefore belong in the root filesystem: the `/data` mount point and an `/etc/fstab` entry for the existing mapping.

Add both through a `base-files` bbappend:

```bash
mkdir -p ../sources/meta-frdm-imx93-security/recipes-core/base-files

cat > ../sources/meta-frdm-imx93-security/recipes-core/base-files/'base-files_%.bbappend' <<'EOF'
# systemd would create a missing mount point at boot, but the root
# filesystem is read-only, so /data must already exist in the image.
dirs755 += "/data"

# The initramfs has already opened the LUKS2 volume as /dev/mapper/data.
do_install:append() {
    echo "/dev/mapper/data /data ext4 defaults,noatime,nodev,nosuid 0 2" >> ${D}${sysconfdir}/fstab
}

# The fstab entry requests an ext4 filesystem check before /data is mounted.
# systemd skips the check if fsck.ext4 is unavailable, so make it explicit.
RDEPENDS:${PN}:append = " e2fsprogs-e2fsck"
EOF
```

To keep this part focused on `/data`, the entry is appended to oe-core's stock fstab. That file is a generic template whose first line suggests overriding it with a machine specific one, so a product should ship its own complete fstab instead.

## 8. Build and flash

`frdm-luks-client` and `cryptsetup` are normally needed only in the initramfs. To run the checks in [section 10](#10-verify-the-volume-and-unlock-path) from the real root filesystem, add them temporarily in `local.conf` before building:

```bitbake
IMAGE_INSTALL:append:pn-imx-image-core = " frdm-luks cryptsetup"
```

Remove that addition after testing. Leaving these tools out of the production root filesystem is not the security boundary, because any normal-world program with access to the TEE interface could implement the same client protocol. The lock inside the TA is what denies later derivation requests.

Build the image:

```bash
bitbake imx-image-core
```

BitBake resolves the dependency chain automatically. Building `imx-image-core` also builds `frdm-luks`, embeds the TA in `optee-os` and rebuilds `imx-boot` before assembling the final image.

Embedding the TA changes `tee.bin`, which is part of the bootable image. The previously signed bootable image therefore no longer matches this build. Repeat the [AHAB signing step](/posts/embedded-linux-security-on-the-frdm-imx93-part-2-the-kernel-signed-fit/#7-sign-the-bootable-image) for the rebuilt bootable image before flashing the board.

The storage layout has changed as well. `mmcblk0p3` is new and the rootfs now occupies a fixed size partition. For the first deployment of this layout, write the complete `.wic` image rather than modifying the previous partition table in place.

Set the board to Serial Download Mode, then power cycle or reset it. From the host, confirm that the board is detected:

```bash
nxpuuu list-devices
```

Then flash the newly signed bootable image together with the new system image:

```bash
nxpuuu write -b emmc_all signed-flash.bin imx-image-core-frdm-imx93-secure.rootfs-<timestamp>.wic.zst
```

After flashing completes, return the boot mode switches to eMMC boot and power cycle or reset the board.

> ⚠️ **A full `.wic` flash overwrites `/data` as well.** That is safe for this first deployment because the data partition has not been provisioned yet. Once first boot has created the LUKS2 volume and persistent filesystem, updates that must preserve `/data` need a partition aware update procedure that writes only the intended system partitions or slots.

## 9. First boot provisions `/data`

On the first boot after flashing the new layout, the data partition is unformatted and has no LUKS header yet. The boot log shows the initramfs provisioning it:

```console
dmcrypt: provisioning /dev/disk/by-partlabel/data
dmcrypt: creating a filesystem on /dev/mapper/data
```

This is the provisioning path from [section 6](#6-open-the-volume-in-the-initramfs). Every failure on that path stops the boot, so `dmcrypt: creating a filesystem` appears only after the credential derivation, `luksFormat`, `cryptsetup open` and the derivation lock have all succeeded. All of it runs from the authenticated initramfs before `tee-supplicant` is started, which confirms the early TA integration.

After `switch_root`, systemd checks and mounts the filesystem through the fstab entry:

```console
[  OK  ] Found device /dev/mapper/data.
         Starting File System Check on /dev/mapper/data...
[  OK  ] Finished File System Check on /dev/mapper/data.
         Mounting /data...
[   12.161955] EXT4-fs (dm-1): mounted filesystem 7301c9d2-1812-4d9d-b8a2-b615c82c6ffa r/w with ordered data mode. Quota mode: none.
[  OK  ] Mounted /data.
```

## 10. Verify the volume and unlock path

### Verify the LUKS2 metadata

Run `cryptsetup luksDump` on the data partition and inspect the fields that define this volume:

```console
root@frdm-imx93-secure:~# cryptsetup luksDump /dev/disk/by-partlabel/data

LUKS header information

Version:        2

Data segments:
  0: crypt
    offset: 16777216 [bytes]
    length: (whole device)
    cipher: aes-xts-plain64
    sector: 512 [bytes]

Keyslots:
  0: luks2
    Key:        512 bits
    Priority:   normal
    Cipher:     aes-xts-plain64
    Cipher key: 512 bits
    PBKDF:      pbkdf2
    Hash:       sha256
    Iterations: 1000
    Digest ID:  0
```

`Version: 2` confirms that the volume was created as LUKS2. The data segment uses `aes-xts-plain64`, and `Key: 512 bits` confirms the volume key size, matching the `--key-size 512` setting in the initramfs module. PBKDF2 with 1000 iterations confirms the keyslot PBKDF settings specified by the module. The `512` byte encryption sector matches the eMMC, which reports a 512 byte physical block size in `/sys/block/mmcblk0/queue/physical_block_size`. Only keyslot 0 is populated, so the device bound credential is currently the only credential that unlocks this volume.

Also, verify that a wrong credential is rejected with `--test-passphrase`, which checks it against the keyslots without creating a mapping:

```console
root@frdm-imx93-secure:~# echo wrong-credential | cryptsetup open --test-passphrase /dev/disk/by-partlabel/data
No key available with this passphrase.
```

### Verify that `/data` is encrypted on the eMMC

Write a known string to `/data`, flush it to the eMMC and search for it both through the mapping and on the underlying partition:

```console
root@frdm-imx93-secure:~# echo frdm-imx93-plaintext-marker > /data/marker.txt
root@frdm-imx93-secure:~# sync
root@frdm-imx93-secure:~# strings /dev/mapper/data | grep -c frdm-imx93-plaintext-marker
1
root@frdm-imx93-secure:~# strings /dev/disk/by-partlabel/data | grep -c frdm-imx93-plaintext-marker
0
```

Both searches read the same partition. `/dev/mapper/data` returns its sectors after dm-crypt decrypts them, so the marker is found. `/dev/disk/by-partlabel/data` returns the same sectors as they are stored on the eMMC, and the marker is not there. dm-crypt encrypts every sector written through the mapping before it reaches the eMMC, so the partition holds the contents of `/data` only as ciphertext.

### Verify the credential across a reboot

Boot the same device again. The first boot provisioning messages are gone and the existing filesystem then mounts normally:

```console
[   11.883496] EXT4-fs (dm-1): mounted filesystem 7301c9d2-1812-4d9d-b8a2-b615c82c6ffa r/w with ordered data mode. Quota mode: none.
```

The filesystem UUID matches the first boot, so the existing filesystem was opened rather than created again. A mismatched credential would have stopped the boot, because the initramfs treats a failed `cryptsetup open` as fatal. The second boot therefore confirms that the device bound credential is reproducible across reboots and that the derivation lock does not survive a reboot.

### Verify that derivation stays locked

The final check verifies that the derivation interface remains locked after `switch_root`. From the running system, request the same credential again:

```console
root@frdm-imx93-secure:~# frdm-luks-client --context frdm-imx93-data-v1
TEEC_InvokeCommand failed: 0xffff0001 (origin 0x4)
```

`0xffff0001` is [`TEEC_ERROR_ACCESS_DENIED`](https://github.com/nxp-imx/imx-optee-client/blob/5ac1e19a41d8f7e43a7ddfefc83708030103e7ff/libteec/include/tee_client_api.h#L183), while origin `0x4` is [`TEEC_ORIGIN_TRUSTED_APP`](https://github.com/nxp-imx/imx-optee-client/blob/5ac1e19a41d8f7e43a7ddfefc83708030103e7ff/libteec/include/tee_client_api.h#L227). The request therefore reached the TA and the TA itself refused the derivation. The client had already closed its sessions in the initramfs, so this also confirms the TA lifetime policy from [section 2](#2-write-the-trusted-application): `derivation_locked` remains set for later sessions.

## 11. Where we are and what comes next

The authenticated boot chain now extends through the read-only dm-verity root filesystem, while the writable `/data` partition adds confidentiality through dm-crypt and LUKS2. AES-XTS does not authenticate the encrypted data, so this design gives `/data` no integrity protection similar to dm-verity.

Each device derives its own credential from its hardware and provisions its own random volume key on first boot. The same system image can therefore be deployed across boards without a shared credential.

The security of this device bound unlock path depends on the integrity of the secure world and control of the boot flow before the lock is set. The derivation lock only rejects requests made after the initramfs sets it. Code that runs earlier can still request the same credential through the genuine TA.

In the configuration built for this series, U-Boot's autoboot can be interrupted from the serial console and the writable environment supplies the kernel command line. This allows the boot flow to be changed without modifying the signed FIT. For example, `rdinit=/bin/sh` can start a shell inside the authenticated initramfs before `81-dmcrypt` runs. As built here, `/data` is protected against an attacker who obtains the storage alone but not against one who can change the boot flow on the board.

In addition, this implementation does not enroll a recovery credential. If recovery is added to first boot provisioning, enrollment should happen after `luksFormat` has created the volume and the device bound credential has successfully opened it, but before `frdm-luks-client --lock`. A normal `luksAddKey` operation needs access to the existing volume key, typically by unlocking an existing keyslot.

The recovery credential should be random, like the recovery key that [`systemd-cryptenroll --recovery-key`](https://www.freedesktop.org/software/systemd/man/259/systemd-cryptenroll.html#--recovery-key) generates. It unlocks the same volume key as the device bound credential, so a credential chosen by a person would lower the protection against an attacker who obtains the storage alone. The off device credential store becomes part of the security architecture as well. Reusing one recovery credential across the fleet means that compromise of that credential exposes every device on which it is enrolled.

A recovery keyslot does not protect against loss of the LUKS metadata and keyslot area itself. A product that must tolerate this failure as well as SoC failure therefore needs a LUKS header backup policy. That backup must also be treated as security sensitive state.

Part 5 will add an A/B update architecture with RAUC: two system slots, bootloader state for selecting them, update failure and fallback handling, rollback protection and a persistent encrypted `/data` partition that remains independent of either system slot.
