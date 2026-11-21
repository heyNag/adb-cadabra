# OnePlus 15

## Which folder to open

| Workflow | When to use it | Result |
|---|---|---|
| [01 · Clean install or redo](01-root-and-reinstall/README.md) | Start on official unrooted firmware, or [wipe and rebuild this already-rooted test phone](01-root-and-reinstall/README.md#redo-on-an-already-rooted-phone). | Start with fresh user data and install the selected modules from their official channels. On a redo, check whether the existing KernelSU patch survived the reset before patching anything. |
| [02 · Update what’s installed](02-updates/README.md) | The phone already has components you want to update. | Update each installed component to the latest compatible release on its existing channel, preserve its settings and verify the running version. |
| [03 · Restore to official factory state](03-restore-to-stock/README.md) | Return a rooted phone to factory condition. This erases user data. | Restore official OnePlus firmware for the device and region, verify it before relocking, then confirm a locked bootloader, green Android Verified Boot and a fresh setup screen. |

## Installed on your OnePlus 15

The phone boots Android 17, `CPH2745_17.0.1.201(EX01)`, on slot A. The AVB digest matches the signed India C.80 archive and the update engine confirmed the snapshot merge completed. Shell has no `su` command and KernelSU is absent from the loaded modules. The bootloader is unlocked and Verified Boot is orange. User data and the prior app installation have not been erased. [Restore measurements and current state](02-updates/current/verification.json)

### Recorded rooted components

The table records the last working rooted setup on November 21, 2026 at 13:56 Bangkok time. All seven then matched their selected release channels. Use these versions and tagged sources to reproduce that setup after the factory handoff.

**Vector is the only component on canary** because PixelMask 1.2.0 requires its Android 17 compatibility fix. The other six components use stable releases. [Compatibility notes](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0)

For a fresh download, use **Latest on this channel**. **Where we installed it from** identifies the exact build verified on the phone.

| Component or module | Installed channel | Version in rooted setup | Where we installed it from | Latest on this channel |
|---|---|---|---|---|
| KernelSU root and manager | Official stable, LKM mode | **3.3.1**, build **32680** | [tiann/KernelSU v3.3.1](https://github.com/tiann/KernelSU/releases/tag/v3.3.1). The official manager patched the phone’s matching stock `init_boot` image. | [Stable manager APK](https://github.com/tiann/KernelSU/releases/latest) · choose the release APK |
| bindhosts | Stable | **2.1.5**, code **215** | `bindhosts.zip` from [bindhosts/bindhosts v2.1.5](https://github.com/bindhosts/bindhosts/releases/tag/v2.1.5) | [Download stable ZIP](https://github.com/bindhosts/bindhosts/releases/latest/download/bindhosts.zip) |
| KernelSU-Tailscaled module | Stable | **2.0.5**, code **20000017** | ARM64 module ZIP from [WayneShao/KernelSU-Tailscaled v2.0.5](https://github.com/WayneShao/KernelSU-Tailscaled/releases/tag/v2.0.5) | [Stable integration](https://github.com/WayneShao/KernelSU-Tailscaled/releases/latest) · choose the ARM64 ZIP |
| Tailscale engine | Official Tailscale stable | **1.104.7** | The module’s engine updater installed [Tailscale’s official ARM64 package](https://pkgs.tailscale.com/stable/tailscale_1.104.7_arm64.tgz) | [Stable Linux binaries](https://pkgs.tailscale.com/stable/#static) · choose ARM64, or use the module’s engine updater |
| Zygisk Next | Stable | **1.5.3**, build **880** | Installed from [LSPosed/ZygiskNext v1.5.3](https://github.com/LSPosed/ZygiskNext/releases/tag/v1.5.3). | [Stable loader](https://github.com/Dr-TSNG/ZygiskNext/releases/latest) · choose the release ZIP |
| Vector | **Canary**, Release ZIP | **2.2**, build **3119**, API **102** | `Vector-v2.2-3119-Release.zip` from [JingMatrix/Vector canary-3119](https://github.com/JingMatrix/Vector/releases/tag/canary-3119) | [Canary releases](https://github.com/JingMatrix/Vector/releases?q=prerelease%3Atrue%20canary) · choose the highest canary build’s `*-Release.zip` |
| PixelMask | Stable | **1.2.0**, code **18** | `PixelMask-1.2.0.apk` from [kinginu/PixelMask 18-1.2.0](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0) | [Stable add-on](https://github.com/kinginu/PixelMask/releases/latest) · choose the PixelMask APK |

After a check, replace the timestamp and [verification record](02-updates/current/verification.json). After a change, refresh the affected inventory rows and workflow instructions once the running state has been verified. Update these same files in place. [Update procedure](02-updates/README.md)

## Recovery sources and regeneration

**To get the latest full firmware**, open **Settings → About device → Software update**, read the build offered for this phone, then select **OnePlus 15 → India → Full** in the firmware index. Use the phone only to read the offered build. Download the ZIP on the Mac from the OnePlus/OPlus distribution link located by the index. Resolve a fresh link each time. [Latest India full-package metadata](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info) · [Download and verification instructions](03-restore-to-stock/README.md#get-the-latest-full-indian-firmware)

**Last firmware check, November 21, 2026.** The built-in local installer updated this phone from C.75 on slot B to C.80 on slot A. The full India ZIP is 9,301,191,889 bytes. Its signature matched a trusted OTA key on the phone. It carries complete images for 65 target partitions and declares a partial partition set. A full factory service image and relock procedure remain unverified. Both releases were available from the manufacturer's CDN with freshly resolved links. The C.75 stock and patched boot images belong to the previous build. For a later C.80 reroot, extract the C.80 stock `init_boot` from its authenticated India OTA and generate a fresh patch for the verified active slot. [Recorded C.75 identity](01-root-and-reinstall/kit/firmware-identification.json) · [Exact-build extraction](01-root-and-reinstall/README.md#2-select-a-matching-image)

**To recreate the rooted setup**, use each project’s **Latest on this channel** link for its APK, module ZIP or engine. The installed-source column identifies the build to select if you need to reproduce a particular version. Use one [temporary Mac working folder](../README.md#temporary-mac-working-folder) for the entire operation.

| Item | Source and how to obtain it |
|---|---|
| `init_boot.img` | Check the full OTA’s device and target build, then extract the `init_boot` partition from its `payload.bin` into a temporary directory. Verify its SHA-256 before use. This stock image reverses the root patch on the matching build and slot. [Extraction and verification steps](01-root-and-reinstall/README.md#2-select-a-matching-image) |
| `boot.img`, `vbmeta.img` | If needed, select the `boot` and `vbmeta` partitions from that same payload. Their expected hashes are in the [image checksum list](01-root-and-reinstall/kit/IMAGE-SHA256SUMS). |
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
