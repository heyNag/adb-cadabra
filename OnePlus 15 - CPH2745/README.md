# OnePlus 15

## Which folder to open

| Workflow | When to use it | Result |
|---|---|---|
| [01 · Clean install or redo](01-root-and-reinstall/README.md) | Start on official unrooted firmware, or [wipe and rebuild a phone that still has root](01-root-and-reinstall/README.md#redo-on-an-already-rooted-phone). | Start with fresh user data and install the selected modules from their official channels. On a redo, check whether the existing KernelSU patch survived the reset before patching anything. |
| [02 · Update what’s installed](02-updates/README.md) | The phone already has components you want to update. | Update each installed component to the latest compatible release on its existing channel, preserve its settings and verify the running version. |
| [03 · Restore to official factory state](03-restore-to-stock/README.md) | Return a rooted phone to factory condition. This erases user data. | Restore official OnePlus firmware for the device and region, verify it before relocking, then confirm a locked bootloader, green Android Verified Boot and a fresh setup screen. |

## Installed on your OnePlus 15

**Factory restore verified on November 21, 2026.** This Indian OnePlus 15 runs official Android 17 build `CPH2745_17.0.1.201(EX01)` on slot A. Android reported a locked bootloader and green Verified Boot. Its AVB digest matches the signed OnePlus India C.80 archive. The owner accepted the lock and erase, saw **Hello ColorOS 17**, then completed setup without Google sign-in and reached Home. The post-wipe check found no KernelSU runtime, KernelSU Manager or PixelMask in the owner profile. [Restore measurements and current state](02-updates/current/verification.json)

For the previous rooted setup, the [clean rebuild guide](01-root-and-reinstall/README.md#recorded-rooted-components) records the component versions, channels, original sources and settings.

## Recovery sources and regeneration

**To get the latest full firmware**, open **Settings → About device → Software update**, read the build offered for this phone, then select **OnePlus 15 → India → Full** in the firmware index. Use the phone only to read the offered build. Download the ZIP on the Mac from the OnePlus/OPlus distribution link located by the index. Resolve a fresh link each time. [Latest India full-package metadata](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info) · [Download and verification instructions](03-restore-to-stock/README.md#get-the-latest-full-indian-firmware)

**Last firmware check, November 21, 2026.** The built-in local installer updated this phone from C.75 on slot B to C.80 on slot A. The full India ZIP is 9,301,191,889 bytes. Its signature matched a trusted OTA key on the phone. It carries complete images for 65 target partitions and declares a partial partition set. The procedure recorded here was performed on this specific handset. OnePlus’s public India materials document routine updates and offer a service route. Both releases were available from the manufacturer's CDN with freshly resolved links. The C.75 stock and patched boot images belong to the previous build. For a later C.80 reroot, extract the C.80 stock `init_boot` from its authenticated India OTA and generate a fresh patch for the verified active slot. [Recorded C.75 identity](01-root-and-reinstall/kit/firmware-identification.json) · [Exact-build extraction](01-root-and-reinstall/README.md#2-select-a-matching-image)

**To recreate the rooted setup**, use each project’s **Latest on this channel** link for its APK, module ZIP or engine. The [recorded rooted components](01-root-and-reinstall/README.md#recorded-rooted-components) list the exact builds and their sources. Use one [temporary Mac working folder](../README.md#temporary-mac-working-folder) for the entire operation.

| Item | Source and how to obtain it |
|---|---|
| `init_boot.img` | Check the full OTA’s device and target build, then extract the `init_boot` partition from its `payload.bin` into a temporary directory. Verify its SHA-256 before use. This stock image reverses the root patch on the matching build and slot. [Extraction and verification steps](01-root-and-reinstall/README.md#2-select-a-matching-image) |
| `boot.img`, `vbmeta.img` | If needed, select the `boot` and `vbmeta` partitions from that same payload. The [image checksum list](01-root-and-reinstall/kit/IMAGE-SHA256SUMS) covers C.75. For another build use its authenticated payload manifest. |
| KernelSU-patched `init_boot.img` | On the phone, use a compatible official KernelSU Manager to patch the verified stock `init_boot.img`. Copy the exact output named by the manager to temporary storage on the Mac and compare the phone and Mac hashes. The reference digest for `kernelsu_patched_20261108_085524.img` identifies that particular output from KernelSU 3.3.1. A fresh patch can have a different digest. [Patching steps](01-root-and-reinstall/README.md#2-select-a-matching-image) |
| `vector-config.db` | This was an export of the phone’s Vector settings. For a clean install, recreate the recorded profile and scopes using the setup steps. For an in-place update, export the live settings to a temporary database before changing Vector. [Recreate the configuration](01-root-and-reinstall/README.md#4-install-and-verify-the-selected-modules) · [Export and verify live settings](02-updates/README.md#temporary-vector-settings-export) |

## Folder tree

```text
OnePlus 15 - CPH2745/
├── README.md                       ← current device, versions, channels and sources
├── 01-root-and-reinstall/
│   ├── README.md                   ← clean install from stock or redo after wiping a rooted phone
│   └── kit/
│       ├── firmware-identification.json
│       ├── IMAGE-SHA256SUMS        ← reference hashes for temporary images
│       └── SHA256SUMS
├── 02-updates/
│   ├── README.md                   ← update installed components on their channels
│   └── current/
│       └── verification.json       ← latest phone and upstream checks
└── 03-restore-to-stock/
    ├── README.md                   ← restore official firmware, wipe and verify locked stock
    └── verify-ota.md               ← download on the Mac and verify OTA identity, signature and coverage
```
