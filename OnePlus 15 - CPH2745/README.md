# OnePlus 15

## Which folder to open

| Workflow | When to use it | Result |
|---|---|---|
| [01 · Clean install or redo](01-root-and-reinstall/README.md) | Start on official unrooted firmware, or [wipe and rebuild a phone that still has root](01-root-and-reinstall/README.md#redo-on-an-already-rooted-phone). | Start with fresh user data and install the selected modules from their official channels. On a redo, check whether the existing KernelSU patch survived the reset before patching anything. |
| [02 · Update what’s installed](02-updates/README.md) | The phone already has components you want to update. | Update each installed component to the latest compatible release on its existing channel, preserve its settings and verify the running version. |
| [03 · Restore to official factory state](03-restore-to-stock/README.md) | Start from the phone’s actual build and slot. The guide covers the [current C.80 slot A starting point](03-restore-to-stock/README.md#return-the-rebuilt-c80-slot-a-to-stock-boot) and the tested C.75-to-C.80 factory reset. | The C.75-to-C.80 run reached the welcome screen and a verified locked, green boot. The C.80 entry explains the evidence needed before relocking from that starting state. |

<a id="installed-on-your-oneplus-15"></a>

## Verified setup on the reference phone

**Full inventory and channels checked November 21, 2026 at 17:56 Bangkok time.** The Indian OnePlus 15 runs Android 17 `CPH2745_17.0.1.201(EX01)` (C.80) on slot A, with the bootloader unlocked, KernelSU working in LKM mode, and SELinux enforcing. The authorized ADB shell returned UID 0. All seven versions below matched their upstream release channels at this check. Future updates should follow the **Latest on this channel** links. [Current measurements and verified artifact hashes](02-updates/current/verification.json) · [Update installed components](02-updates/README.md)

| Component | Installed channel | Version on this phone | Verified state | Installed from | Latest on this channel |
|---|---|---|---|---|---|
| KernelSU and Manager | Official stable | 3.3.1 (32680) | Working / LKM on the C.80 kernel | [Official Manager APK release](https://github.com/tiann/KernelSU/releases/tag/v3.3.1) | [Stable releases](https://github.com/tiann/KernelSU/releases/latest) |
| bindhosts | Stable | 2.1.5 (215) | Active in mode 10. At refresh, 196,894 domains. Chrome’s Superuser and Unmount modules switches are off. | [bindhosts ZIP](https://github.com/bindhosts/bindhosts/releases/tag/v2.1.5) | [Stable ZIP](https://github.com/bindhosts/bindhosts/releases/latest/download/bindhosts.zip) |
| KernelSU-Tailscaled | Stable, ARM64 | 2.0.5 (20000017) | Enabled in native mode. Supervisor, daemon and web service running. | [ARM64 integration ZIP](https://github.com/WayneShao/KernelSU-Tailscaled/releases/tag/v2.0.5) | [Stable integration](https://github.com/WayneShao/KernelSU-Tailscaled/releases/latest) |
| Tailscale engine | Official stable, ARM64 | 1.104.7 | Updater reports current and latest match. Signed out (`NeedsLogin`), as before the reset. Peer connectivity untested. | [Official ARM64 tarball](https://pkgs.tailscale.com/stable/tailscale_1.104.7_arm64.tgz) | [Stable binaries](https://pkgs.tailscale.com/stable/#static) |
| Zygisk Next | Stable | 1.5.3 (880) | Injection active, Vector loaded, zero module issues. | [Zygisk Next release ZIP](https://github.com/Dr-TSNG/ZygiskNext/releases/tag/v1.5.3) | [Stable releases](https://github.com/Dr-TSNG/ZygiskNext/releases/latest) |
| Vector | Canary, Release ZIP | 2.2 (3119), API 102 | Active, one enabled add-on. | [Canary 3119 Release ZIP](https://github.com/JingMatrix/Vector/releases/tag/canary-3119) | [Canary releases](https://github.com/JingMatrix/Vector/releases?q=prerelease%3Atrue%20canary) |
| PixelMask | Stable | 1.2.0 (18) | Enabled for PixelMask and Google Photos (user 0), matching the pre-reset scope. PixelMask reported **Active in Google Photos** after Photos restarted. | [PixelMask APK](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0) | [Stable releases](https://github.com/kinginu/PixelMask/releases/latest) |

**PixelMask runtime check, November 21, 2026.** After adding Google Photos to the existing scope and restarting it, PixelMask showed “Spoofing as Pixel · Photos 7.86.0.956040398 · just now.” This was a separate check after the full inventory. [Restore the scope and verify the result](01-root-and-reinstall/README.md#restore-and-verify-the-pixelmask-scope)

[Clean rebuild and KMI selection](01-root-and-reinstall/README.md#2-select-a-matching-image) · [Factory restore walkthrough](03-restore-to-stock/README.md) · [Fastboot checks and physical lock prompt](03-restore-to-stock/README.md#from-running-android-to-the-bootloader-menu)

## Recovery sources and regeneration

**To get the latest full firmware**, open **Settings → About device → Software update**, read the build offered for this phone, then select **OnePlus 15 → India → Full** in the firmware index. Use the phone only to read the offered build. Download the ZIP on the Mac from the OnePlus/OPlus distribution link located by the index. Resolve a fresh link each time. [Latest India full-package metadata](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info) · [Download and verification instructions](03-restore-to-stock/README.md#get-the-latest-full-indian-firmware)

**OS Updater (optional).** Before the reset, we used OS Updater (also called Oxygen Updater) to identify the full Indian C.80 firmware. OnePlus’s built-in **Local install** performed the installation. On November 21, 2026, ADB confirmed that OS Updater (`com.arjanvlek.oxygenupdater`) was absent for Android user 0 on the rebuilt phone. [Device verification record](02-updates/current/verification.json)

To use it again, install OS Updater from Google Play and select **OnePlus 15 (IN) → Full**. This third-party app works without root. Its build, ZIP filename and MD5 help identify the offered package. The Mac walkthrough verifies the firmware signature independently. [OS Updater on Google Play](https://play.google.com/store/apps/details?id=com.arjanvlek.oxygenupdater) · [Project website](https://oxygenupdater.com/) · [If Local install is missing or greyed out](03-restore-to-stock/README.md#if-local-install-is-missing-or-greyed-out)

**Last firmware check, November 21, 2026.** The built-in local installer updated this phone from C.75 on slot B to C.80 on slot A. The full India ZIP is 9,301,191,889 bytes. Its signature matched a trusted OTA key on the phone. It carries complete images for 65 target partitions and declares a partial partition set. The procedure recorded here was performed on this specific handset. OnePlus’s public India materials document routine updates and offer a service route. Both releases were available from the manufacturer's CDN with freshly resolved links. The C.75 stock and patched boot images belong to the previous build. The C.80 rebuild extracted stock `init_boot` from its authenticated India OTA and produced a fresh KernelSU patch for slot A. [Recorded C.75 identity](01-root-and-reinstall/kit/firmware-identification.json) · [Exact-build extraction](01-root-and-reinstall/README.md#2-select-a-matching-image)

**To recreate the rooted setup**, use each project’s **Latest on this channel** link for its APK, module ZIP or engine. The **Installed from** column in the [inventory](#verified-setup-on-the-reference-phone) identifies the releases used on this phone. Use one [temporary Mac working folder](../README.md#temporary-mac-working-folder) for the entire operation.

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
