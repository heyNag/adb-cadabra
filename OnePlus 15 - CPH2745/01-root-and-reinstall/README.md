# Clean install or redo a rooted OnePlus 15

On a fresh phone running official, unrooted firmware, start at [Set up a fresh Mac and phone](#0-set-up-a-fresh-mac-and-phone). For a phone that still has active KernelSU, the redo procedure below starts with a wipe. For a factory-to-root rebuild, use the latest compatible release on each channel in the [component table](#recorded-rooted-components) and restore the settings in section 4. The tagged versions identify what was tested if an exact-version replay is requested.

## Redo on an already-rooted phone

For this development phone, a redo can discard all user data, apps, accounts, root grants and module settings. The intended result is a fresh **rooted** setup with the bootloader still unlocked.

1. **Identify the phone before erasing.** Record the connected phone’s model, firmware fingerprint, active slot and running KernelSU build. Use the [device checks](../02-updates/README.md#1-read-the-phone) and the [installed inventory](../README.md#installed-on-your-oneplus-15). Have the screen PIN and previously synced Google account credentials available if Android requests them during setup. [Android factory-reset guidance](https://support.google.com/android/answer/6088915)
2. **When performing the redo, erase user data.** On the intended phone, search Settings for **Erase all data** or **Factory reset**, review the device’s erase prompt and confirm. Complete Android setup as a new device and skip copying apps or restoring a backup. A data reset can leave the patched `init_boot` and unlocked bootloader in place.
3. **Check what survived.** Follow [Mac and USB setup](#0-set-up-a-fresh-mac-and-phone) to re-enable debugging and authorize the Mac again. Use [the phone and manager checks](#1-check-the-phone-and-manager) to re-read the firmware and slot and install the official KernelSU Manager matching the previously running build. Open Manager and check its status. If it shows **Working / LKM**, grant Shell (UID 2000) root in Manager and confirm `phone shell su -c id` returns UID 0. A denied Shell grant by itself is inconclusive.
4. **Rebuild the modules.** Once root is verified, go directly to [Install and verify the selected modules](#4-install-and-verify-the-selected-modules). Use the release selection in section 1 and recreate the settings. When choosing current releases, check the running KernelSU build against stable using the [update guide](../02-updates/README.md). This route needs no boot image download, flash or bootloader change when the existing root works.

If Manager reports **Not installed**, confirm its compatibility with the actual post-reset firmware and kernel. If the root patch needs replacement, obtain and verify the [exact-build stock image](#2-select-a-matching-image), then follow the [active-partition checks](#3-unlock-only-if-required-then-verify-the-write-target) using the already-unlocked path. If the required official image is unavailable or the root state is uncertain, stop before flashing. [KernelSU installation guide](https://kernelsu.org/guide/installation.html)

## 0. Set up a fresh Mac and phone

Open Terminal in this project’s root folder. Keep the same Terminal session for the steps below. Create one [temporary Mac working folder](../../README.md#temporary-mac-working-folder) for all downloads, extracted files, patches and checks in this run.

```sh
umask 077
RUN_BASE="$HOME/Downloads/adb-cadabra/OnePlus 15 - CPH2745"
mkdir -p "$RUN_BASE" &&
  DOWNLOAD_DIR=$(mktemp -d "$RUN_BASE/install-$(date +%Y%m%d-%H%M%S)-XXXXXXXX") &&
  printf '%s\n' "$DOWNLOAD_DIR"
```

**Install Android Platform Tools.** If `adb version` and `fastboot --version` already work, use your existing tools. Otherwise download Google’s Mac ZIP, unpack it and add the extracted directory to this shell’s `PATH`:

```sh
curl --fail --location --proto '=https' --proto-redir '=https' \
  --output "$DOWNLOAD_DIR/platform-tools.zip" \
  https://dl.google.com/android/repository/platform-tools-latest-darwin.zip &&
  unzip -q "$DOWNLOAD_DIR/platform-tools.zip" -d "$DOWNLOAD_DIR" &&
  export PATH="$DOWNLOAD_DIR/platform-tools:$PATH"
adb version
fastboot --version
```

Have Python 3 available for checking the package’s embedded identity in section 2. The image checks also use the Mac’s built-in `stat`, `od` and `shasum`. [Python for macOS](https://www.python.org/downloads/macos/) Get the OTA extractor in section 2 when preparing a stock image. [Google’s Platform Tools for Mac](https://developer.android.com/tools/releases/platform-tools)

**Prepare the phone.** Finish basic Android setup. Under **Settings → About device → Version**, tap **Build number** or **Version number** seven times. Enter the phone’s PIN if prompted, then search Settings for **Developer options** and enable **USB debugging**. Enable **OEM unlocking** there only when rooting a locked phone with an approved wipe. Connect the screen-unlocked phone with a USB data cable. Run `adb devices -l` and accept **Allow USB debugging?** on the phone for this Mac. Rerun the command until the intended phone shows `device`. A bootloader-unlock wipe clears this authorization. [Android developer options and USB debugging](https://developer.android.com/studio/debug/dev-options)

## 1. Check the phone and manager

Read the serial from the first column of `adb devices -l`. Enter that actual value when prompted below. Record the output, especially the firmware fingerprint, kernel, active slot suffix and bootloader lock state.

```sh
printf 'Serial of the authorized OnePlus: '
read -r PHONE_SERIAL
phone() { adb -s "${PHONE_SERIAL:?Enter the OnePlus serial first}" "$@"; }
phone get-state
phone shell getprop ro.product.model
phone shell getprop ro.product.name
phone shell getprop ro.product.device
phone shell getprop ro.product.cpu.abi
phone shell getprop ro.build.display.id
phone shell getprop ro.build.fingerprint
phone shell getprop ro.boot.slot_suffix
phone shell getprop ro.boot.flash.locked
phone shell getprop ro.boot.vbmeta.device_state
phone shell getprop sys.boot_completed
phone shell uname -r
```

Require `phone get-state` to report `device` and boot completion to report `1`. Compare the model, product, device, ABI, display build and full fingerprint with the firmware you plan to use. Record the actual slot and lock state before continuing.

**Choose which releases to install.** By default, use **Latest on this channel** for the newest compatible stable releases and Vector’s canary **Release** ZIP. When reproducing the recorded versions, use **Where we installed it from** for the exact tags, build numbers and assets. For the first root check after erasing an already-rooted phone, install the exact Manager APK matching its previously running KernelSU build. In either mode, check Android, kernel, ABI and dependency compatibility with the current phone. [Recorded rooted components](#recorded-rooted-components) · [KernelSU releases](https://github.com/tiann/KernelSU/releases)

For each installer or extractor release archive, set `ASSET_URL`, `ASSET_NAME` and `EXPECTED_SHA256` from the selected official release. Use the exact asset filename and the 64-character hex digest supplied by the release or its GitHub asset metadata. Run the download and checksum together and require an `OK` result before installation.

```sh
asset="$DOWNLOAD_DIR/$ASSET_NAME"
curl --fail --location --proto '=https' --proto-redir '=https' --output "$asset" "$ASSET_URL" &&
  printf '%s  %s\n' "$EXPECTED_SHA256" "$asset" | shasum -a 256 -c -
```

Select the official release APK for KernelSU Manager (package `me.weishu.kernelsu`). After checksum verification set `KSU_APK="$asset"` and run `phone install "$KSU_APK"`. On the locked C.80 phone, OnePlus showed its installer scan while `adb install` was still waiting. Verify the app name and version, read the scan result, then choose **Continue installation** when it reports **No risks found**. Wait for the Mac to report `Success`. Open KernelSU and confirm the phone supports the LKM installation route. The boot component will be installed after the manager patches the matching image.

### Recorded rooted components

The table records the last working rooted setup on November 21, 2026 at 13:56 Bangkok time. All seven then matched their selected release channels. Use its channels for a new rebuild. The version-specific links identify the artifacts used in that setup.

**Vector is the only component on canary** because PixelMask 1.2.0 requires its Android 17 compatibility fix. The other six components use stable releases. [Compatibility notes](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0)

For a fresh download, use **Latest on this channel**. **Where we installed it from** identifies the exact build verified on the phone.

| Component or module | Rebuild channel | Version in rooted setup | Where we installed it from | Latest on this channel |
|---|---|---|---|---|
| KernelSU root and manager | Official stable, LKM mode | **3.3.1**, build **32680** | [tiann/KernelSU v3.3.1](https://github.com/tiann/KernelSU/releases/tag/v3.3.1). The official manager patched the phone’s matching stock `init_boot` image. | [Stable manager APK](https://github.com/tiann/KernelSU/releases/latest) · choose the release APK |
| bindhosts | Stable | **2.1.5**, code **215** | `bindhosts.zip` from [bindhosts/bindhosts v2.1.5](https://github.com/bindhosts/bindhosts/releases/tag/v2.1.5) | [Download stable ZIP](https://github.com/bindhosts/bindhosts/releases/latest/download/bindhosts.zip) |
| KernelSU-Tailscaled module | Stable | **2.0.5**, code **20000017** | ARM64 module ZIP from [WayneShao/KernelSU-Tailscaled v2.0.5](https://github.com/WayneShao/KernelSU-Tailscaled/releases/tag/v2.0.5) | [Stable integration](https://github.com/WayneShao/KernelSU-Tailscaled/releases/latest) · choose the ARM64 ZIP |
| Tailscale engine | Official Tailscale stable | **1.104.7** | The module’s engine updater installed [Tailscale’s official ARM64 package](https://pkgs.tailscale.com/stable/tailscale_1.104.7_arm64.tgz) | [Stable Linux binaries](https://pkgs.tailscale.com/stable/#static) · choose ARM64, or use the module’s engine updater |
| Zygisk Next | Stable | **1.5.3**, build **880** | Installed from [LSPosed/ZygiskNext v1.5.3](https://github.com/LSPosed/ZygiskNext/releases/tag/v1.5.3). | [Stable loader](https://github.com/Dr-TSNG/ZygiskNext/releases/latest) · choose the release ZIP |
| Vector | **Canary**, Release ZIP | **2.2**, build **3119**, API **102** | `Vector-v2.2-3119-Release.zip` from [JingMatrix/Vector canary-3119](https://github.com/JingMatrix/Vector/releases/tag/canary-3119) | [Canary releases](https://github.com/JingMatrix/Vector/releases?q=prerelease%3Atrue%20canary) · choose the highest canary build’s `*-Release.zip` |
| PixelMask | Stable | **1.2.0**, code **18** | `PixelMask-1.2.0.apk` from [kinginu/PixelMask 18-1.2.0](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0) | [Stable add-on](https://github.com/kinginu/PixelMask/releases/latest) · choose the PixelMask APK |

## 2. Select a matching image

**Start with the build on the phone.** On November 21, 2026, the Indian handset started the clean rebuild on stock `CPH2745_17.0.1.201(EX01)` (C.80), Android 17, slot A, locked with green Verified Boot. Its kernel was `6.12.69-android16-6-g5fd39602326e-ab16372235-4k`. The device uses a separate `init_boot` ramdisk. For LKM on this layout, use the stock `init_boot.img` from the running firmware and generate a fresh patch. Keep the matching stock image on the Mac for recovery. [KernelSU installation](https://kernelsu.org/guide/installation.html) · [Handset and signed firmware measurements](../02-updates/current/verification.json)

For the C.80 case, use the [India full-package download and signer checks](../03-restore-to-stock/verify-ota.md). A private Mac run from a previous restore may already contain the official ZIP. Reuse it once you have matched its identity to the running phone, checked its digest against the authenticated record, and extracted a new stock image into the current install run. Set `OTA` to that verified ZIP. When downloading from scratch into this run, set `RESTORE_DIR=$DOWNLOAD_DIR` for the verifier. For a different installed build, obtain and authenticate its own full India archive. The flash target is always determined again from the live slot.

| Checked field | Indian C.80 value |
|---|---|
| OTA identity | `CPH2745_11.C.80_1800_202610290517` |
| Product / device | `CPH2745IN` / `OP611FL1` |
| Full OTA size / SHA-256 | 9,301,191,889 bytes / `c7ebbcf62c64895ec9c49fe0988b6f0b40a682d95b5c8a310457afb67de5d9db` |
| Stock `init_boot.img` size / SHA-256 from signed manifest | 8,388,608 bytes / `cf870b50cf44845c5e9070afe7a13188aa5a13efc1bc7cddb42f49f217074420` |
| Full target fingerprint | `OnePlus/CPH2745IN/OP611FL1:17/CP2A.260605.016/B.R4T3.50a3663-29f0eba-29fce1c:user/release-keys` |

**Earlier C.75 build only.** If the running phone reports C.75, select **OnePlus 15 India CPH2745**, full OTA **CPH2745_17.0.0.102(EX01)**, OTA ID **CPH2745_11.C.75_1750_202610181120**. Use the archive entry’s **官方链接** (official link) as the manufacturer link to resolve with the [firmware download instructions](../03-restore-to-stock/README.md#get-the-latest-full-indian-firmware). On November 21, 2026 a fresh link served this archive from `gauss-compota-c-in.allawnofs.com`. The old unsigned URL returns HTTP 403. For any other running build, select its own matching archive. Verify the package and extracted stock image before writing a boot partition. [C.75 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.0.102%28EX01%29%20C.75/) · [OnePlus India software support](https://www.oneplus.in/support/softwareupgrade) · [Recorded firmware identity](kit/firmware-identification.json)

**Verify before extracting.** Set `OTA` to the absolute path of the acquired ZIP. Run `unzip -p "$OTA" META-INF/com/android/metadata`. Match `pre-device` and the full `post-build` fingerprint. Cross-check the OnePlus product, display version and OTA ID in the accompanying vendor metadata or official package listing. The following table describes the earlier C.75 example:

| OTA field | Required value |
|---|---|
| `pre-device` | `OP611FL1` |
| `product_name` / `version_name` | `CPH2745IN` / `CPH2745_17.0.0.102(EX01)` |
| `ota-id` | `CPH2745_11.C.75_1750_202610181120` |
| `post-build` | `OnePlus/CPH2745IN/OP611FL1:17/CP2A.260605.016/B.R4T3.55357f5-2eb9868-2eda512:user/release-keys` |

**Get the extractor when needed.** Run `uname -m` on the Mac. Select the upstream release archive for Darwin ARM64 on Apple silicon (`arm64`) or Darwin AMD64 on Intel (`x86_64`). Download and verify the archive using section 1. Extract it into a private directory and verify the command is available. [Official payload-dumper-go releases](https://github.com/ssut/payload-dumper-go/releases)

```sh
mkdir "$DOWNLOAD_DIR/extractor" &&
  tar -xzf "$asset" -C "$DOWNLOAD_DIR/extractor" &&
  export PATH="$DOWNLOAD_DIR/extractor:$PATH"
command -v payload-dumper-go
payload-dumper-go -h
```

For every build, require the OTA’s embedded product/device and full target fingerprint to match the actual phone from section 1. Allow space for the full ZIP and the selected images. The upstream extractor reads the payload directly from the archive. Choose a build-labelled subdirectory for the images. The following block derives its name from the package’s verified display build and checks it against the selected phone. Keep that directory through recovery and verification.

```sh
STOCK_DIR=$(python3 - "$OTA" "$DOWNLOAD_DIR" "$(phone shell getprop ro.build.display.id)" <<'PY'
import re, sys, zipfile
from pathlib import Path
ota, run, phone_build = sys.argv[1:]
with zipfile.ZipFile(ota) as archive:
    metadata = archive.read("META-INF/com/android/metadata").decode().splitlines()
fields = dict(line.split("=", 1) for line in metadata if "=" in line)
build = fields.get("version_name", "")
match = re.fullmatch(r"(CPH2745_[0-9]+(?:\.[0-9]+)+)\(([A-Za-z0-9]+)\)", build)
if not match or build != phone_build:
    sys.exit("The OTA version_name must match the selected phone's installed build.")
print(Path(run) / f"stock-{match[1]}_{match[2]}")
PY
) && printf '%s\n' "$STOCK_DIR"
```

After the printed directory matches the installed build, extract the stock image.

```sh
payload-dumper-go -p init_boot -o "$STOCK_DIR" "$OTA"
```

Keep the extractor’s output hash verification enabled. For these two recorded builds, use the fingerprint-matched image and its own manifest digest. Require the printed checksum to report `OK`.

```sh
stock_image="$STOCK_DIR/init_boot.img"
case "$(phone shell getprop ro.build.display.id)" in
  'CPH2745_17.0.1.201(EX01)') image_sha=cf870b50cf44845c5e9070afe7a13188aa5a13efc1bc7cddb42f49f217074420 ;;
  'CPH2745_17.0.0.102(EX01)') image_sha=8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 ;;
  *) unset image_sha ;;
esac
[ "$(stat -f '%z' "$stock_image")" = 8388608 ] &&
  printf '%s  %s\n' "${image_sha:?Use the matching signed manifest for this firmware}" "$stock_image" |
  shasum -a 256 -c -
```

The C.75 full-image digest covers the boot header too. To inspect the layout directly, the following commands should print the eight bytes `41 4e 44 52 4f 49 44 21` (`ANDROID!`), four zero bytes for kernel size, and `04 00 00 00` for header version 4, respectively.

```sh
od -An -tx1 -N 8 "$stock_image"
od -An -tx1 -j 8 -N 4 "$stock_image"
od -An -tx1 -j 40 -N 4 "$stock_image"
```

For another firmware, use its own payload manifest’s expected partition size and hash. Check `stat -f '%z' "$STOCK_DIR/init_boot.img"` and `shasum -a 256 "$STOCK_DIR/init_boot.img"` against those values. A source-dependent operation requires the matching base image or a suitable full package. To obtain all three stock images, select `-p init_boot,boot,vbmeta`. [C.75 image checksums](kit/IMAGE-SHA256SUMS) · [Extractor flags](https://github.com/ssut/payload-dumper-go/blob/2.1.0/cmd/root.go)

**Before unlocking a stock phone.** Steps 1–3 below create a file while Android still has its original USB authorization. Copy the successful patch to this run on the Mac before the bootloader unlock wipes the phone. Keep the Manager APK and both images there. After the wipe, reauthorize USB and compare the new live fingerprint, kernel and active slot again before flashing. A change of firmware requires new matching inputs.

**Prepare the image to flash.** Generate a fresh patch from the verified stock image with the selected official KernelSU manager:

1. Create a dedicated folder under the phone’s Downloads, for example `PHONE_DIR=/sdcard/Download/adb-cadabra-install-CPH2745-C80` for this run. Run `phone shell mkdir -p "$PHONE_DIR"`, `phone push "$STOCK_DIR/init_boot.img" "$PHONE_DIR/init_boot.img"`, and `phone shell sha256sum "$PHONE_DIR/init_boot.img"`. Require the checksum from the signed firmware.
2. In KernelSU choose the install icon → **Select a file**. In the system picker, open **OnePlus 15 → Download → adb-cadabra-install-CPH2745-C80 → init_boot.img**. The Downloads shortcut showed an empty listing in this run. In the kernel selector, compare the detected **KMI** (kernel module interface) with `phone shell uname -r`. On this **Android 17** phone the running kernel was `6.12.69-android16-6-…`, and the correct selection in KernelSU was **android16-6.12**. The Android version in this label identifies the kernel branch used to build the module. Leave the matching selection in place. Confirm the patch and wait for the log to report unpacking, adding the KernelSU LKM, repacking, an output path, and **Done**. Set `PATCH_ON_PHONE` to that exact path.
3. Set `PATCH_TO_FLASH="$DOWNLOAD_DIR/${PATCH_ON_PHONE##*/}"`, then run `phone pull "$PATCH_ON_PHONE" "$PATCH_TO_FLASH"`. Compare `phone shell sha256sum "$PATCH_ON_PHONE"` with `shasum -a 256 "$PATCH_TO_FLASH"`. Require identical hashes, a file length matching the intended partition, and a successful patch log. A newly generated patch has its own digest. Use its actual filename and hash even when the same KernelSU release was used before. Keep the verified stock image accessible on the Mac through the first boot.

## 3. Unlock only if required, then verify the write target

Doug Lynch’s independent WebUSB tool provides a Chromium browser interface for unlocking a supported device after OEM unlocking is enabled. Completing an unlock erases user data. [Explaining Android bootloader unlock tool](https://explainingandroid.com/bootloader-unlock/)

For a OnePlus 15 sold in mainland China, OnePlus’s official ColorOS Upgrade Assistant describes how to apply for Deep Testing on supported ColorOS 16 and later releases. Check the updated notice for current device and account eligibility. For the Indian CPH2745, follow the serial and bootloader checks below when rooting, or use the factory-restore guide when returning to locked stock. [Original OnePlus announcement](https://bbs.oneplus.com/thread/1926504022886318086) · [Updated OnePlus eligibility rules](https://bbs.oneplus.com/thread/2175833961601695750) · [Factory restore and relock](../03-restore-to-stock/README.md) · [Third-party promotional clip claiming a remote unlock on Chinese PLK110](https://www.youtube.com/watch?v=s1F3sucMW4I)

For a verified OnePlus 15 with an `init_boot` ramdisk, resolve the target partition from the booted Android slot, then enter the real bootloader. In the Android shell values below, `_a` and `_b` map to `init_boot_a` and `init_boot_b`. If the slot is missing, the layout differs or the device is unsupported by the selected manager, establish its supported procedure before any write.

```sh
ANDROID_SLOT=$(phone shell getprop ro.boot.slot_suffix)
case "$ANDROID_SLOT" in
  _a|_b) ROOT_PARTITION="init_boot$ANDROID_SLOT" ;;
  *) unset ROOT_PARTITION ;;
esac
```

For physical Fastboot entry, Doug Lynch shows holding Volume Down during restart at 0:35. At 0:57 he uses the volume keys to move and Power to select. **Start** returns to Android (1:18). [OnePlus 15 Fastboot video](https://www.youtube.com/watch?v=lb9WqZ4IvtM)

When `ROOT_PARTITION` identifies the verified target, run `phone reboot bootloader`. Read the actual output of every query:

```sh
fastboot devices -l
fb() { fastboot -s "$PHONE_SERIAL" "$@"; }
fb getvar product
fb getvar is-userspace
fb getvar current-slot
fb getvar unlocked
fb getvar "partition-size:$ROOT_PARTITION"
```

Require the same USB serial, `product: canoe`, `is-userspace: no`, the same slot Android reported and a partition size matching both image lengths. On the recorded C.80 rebuild the target was `init_boot_a`, `partition-size:init_boot_a` was `0x800000` (8 MiB), `unlocked` was `no` and `secure` was `yes`. Slot A was successful and bootable, and `snapshot-update-status` was `none`. On the older C.75 run the target was on B. Leave slot selection to the verified boot state.

**Unlock a locked phone with an owner-approved wipe.** Check `fb flashing get_unlock_ability`. This C.80 phone returned `1` after the owner-authorized OEM unlocking setting was confirmed on in Developer options. A zero or unsupported response needs investigation before continuing.

1. With the phone in the confirmed bootloader, run `fastboot -s "$PHONE_SERIAL" flashing unlock` **once**. The C.80 handset acknowledged the request with `OKAY [  0.010s]`. Check the confirmation on the phone.
2. On the handset, the **owner** uses the volume keys to highlight **UNLOCK THE BOOTLOADER**, then presses and releases **Power once**. This confirms the data wipe. Keep USB connected through the restart. If a different prompt appears, have the owner report its wording before making a selection.
3. After first-time setup, re-enable **Developer options → USB debugging** and approve the Mac. Repeat the checks in section 1 on the same USB serial. Verify the fingerprint, kernel and slot against the saved patch inputs. Recalculate `ROOT_PARTITION` from the new reading and re-enter the bootloader. Require `unlocked: yes` and recheck product, mode, slot and size. A handset already reporting `unlocked: yes` skips the unlock and its wipe.

With all those checks passing and **`unlocked: yes`**, write the verified patched file from section 2 to the one confirmed active partition.

```sh
fb flash "${ROOT_PARTITION:?Verify the active init_boot partition first}" "${PATCH_TO_FLASH:?Verify the patched image first}"
```

Require successful **Sending** and **Writing** before running `fb reboot`. If the first boot fails, follow [Recover a failed first boot](#recover-a-failed-first-boot). If unlocking erased the manager, reinstall the same verified APK with `phone install "$KSU_APK"`. Open KernelSU and verify **Working / LKM**. Under **Superuser → Show system apps → Shell** (UID 2000), enable the Shell grant. Verify `phone shell su -c id` (UID 0), `phone shell getprop sys.boot_completed` (1), `phone shell getenforce` (Enforcing) and the expected fingerprint and slot.

### Recover a failed first boot

If the patch was the only partition write, the verified stock image from section 2 can undo that write. Return to the bootloader and repeat the serial, product, mode, slot, unlock-state and partition-size checks above. Confirm the active slot is still the one just flashed and the stock file still matches the digest verified before flashing. Then run `fb flash "${ROOT_PARTITION:?Recheck the original target}" "${STOCK_DIR:?Verify the stock build}/init_boot.img"`. Require successful sending and writing before `fb reboot`, then confirm Android boots on the original fingerprint. If the slot changed, the write target is uncertain or the bootloader is unreachable, stop further writes and contact OnePlus. [OnePlus India service](https://www.oneplus.in/support/repair)

## 4. Install and verify the selected modules

For each module, download the release selected in section 1 into `DOWNLOAD_DIR`. For a KernelSU ZIP, run `unzip -tq "$asset"`, inspect its module ID and version code, then `phone push "$asset" "/sdcard/Download/$ASSET_NAME"` and `phone shell sha256sum "/sdcard/Download/$ASSET_NAME"`. Compare the phone digest with the release digest. Select that file in **KernelSU → Modules → Install from storage**, or use the official KernelSU CLI after verifying the Shell root grant. For the CLI, run `phone shell "su -c '/data/adb/ksud module install /sdcard/Download/$ASSET_NAME'"` for each verified release filename (these names contain no shell metacharacters or spaces). Require success for each installer. Reboot to activate the staged modules, wait for boot completion, then read `phone shell "su -c '/data/adb/ksud module list'"`. Check each ID and version code and require enabled with no pending update or removal.

1. **bindhosts and KernelSU-Tailscaled** are independent. On C.80, `/system/bin/curl --version` reported curl 8.16.0-DEV with BoringSSL. bindhosts auto-selected mode 10 (per-module bind mount), using the default lists with scheduled updates off. The optional companion app was skipped. For fresh bindhosts lists, verify the phone has TLS-capable `curl`, then run `phone shell "su -c '/data/adb/modules/bindhosts/bindhosts.sh --force-update'"`. Verify a blocked domain resolves to its sink address while `example.com` resolves normally. For Chrome, set the KernelSU profile to **Unmount modules off** with **Superuser off**. Select the stable ARM64 Tailscaled integration and use its verified engine updater for the official stable Tailscale engine. The ARM64 v2.0.5 integration bundled engine 1.104.2. On this run the official stable feed offered 1.104.7. The module verified and staged it, then applied it. Read `engine-status` after application and compare `current`, `latest` and `staged`. For a same-channel rebuild use the current compatible stable release, recording its exact version once verified. If the owner specifically requests an exact older version, compare the offered engine before applying it. Verify the local service and leave sign-in to the owner. [Engine and health checks](../02-updates/README.md)
2. Install **Zygisk Next**, then **Vector** using the selected canary Release ZIP. After reboot, check `phone shell "su -c '/data/adb/modules/zygisksu/bin/zygiskd status'"` for active injection and Vector loaded, then `phone shell "su -c '/data/adb/modules/zygisk_vector/cli --json status'"` for the expected build.
3. Install the verified stable **PixelMask** APK with `phone install "$asset"`. For this C.80 rebuild the requested local configuration is enablement in Vector scoped solely to the PixelMask app for user 0. Verify the scope and enablement separately in Vector, and keep Google Photos closed. Confirm the package version with `phone shell dumpsys package com.kinginu.pixelmask`. Vector canary 3119’s CLI enables an add-on when it sets its scope. First inspect `modules ls` and the complete `scope ls com.kinginu.pixelmask` list. This clean C.80 run started with the add-on disabled, an empty scope, no automatic-inclusion rule and only Android user 0. The owner requested self-scope. With those conditions verified, run `vector scope set com.kinginu.pixelmask com.kinginu.pixelmask/0` using the [serial-selected Vector helper](../02-updates/README.md). Then run `vector modules ls`, `vector scope ls com.kinginu.pixelmask` and `vector status`. The observed results were `enabled`, exactly one target (`com.kinginu.pixelmask`, user 0), and one enabled add-on. For an existing configuration, inspect its current scope before changing it. [Vector’s scope implementation](https://github.com/JingMatrix/Vector/blob/canary-3119/daemon/src/main/kotlin/org/matrix/vector/daemon/data/ModuleDatabase.kt#L285-L334)

After verification, refresh the [shared inventory](../README.md#installed-on-your-oneplus-15) and [current verification](../02-updates/current/verification.json) with the running versions, selected release links and digests. Record the verified firmware, patched filename and actual SHA-256. Keep this run’s stock fallback through boot, module activation and verification. When cleaning up the temporary downloads, remove only the selected run directory.
