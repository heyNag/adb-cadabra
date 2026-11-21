# Clean install or redo a rooted OnePlus 15

On a fresh phone running official, unrooted firmware, start at [Set up a fresh Mac and phone](#0-set-up-a-fresh-mac-and-phone). To rebuild this already-rooted test phone with empty user data, start below. Both paths finish with working KernelSU and freshly installed modules.

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

**Choose which releases to install.** By default, use **Latest on this channel** for the newest compatible stable releases and Vector’s canary **Release** ZIP. When reproducing the recorded versions, use **Where we installed it from** for the exact tags, build numbers and assets. For the first root check after erasing an already-rooted phone, install the exact Manager APK matching its previously running KernelSU build. In either mode, check Android, kernel, ABI and dependency compatibility with the current phone. [Component inventory](../README.md#installed-on-your-oneplus-15) · [KernelSU releases](https://github.com/tiann/KernelSU/releases)

For each installer or extractor release archive, set `ASSET_URL`, `ASSET_NAME` and `EXPECTED_SHA256` from the selected official release. Use the exact asset filename and the 64-character hex digest supplied by the release or its GitHub asset metadata. Run the download and checksum together and require an `OK` result before installation.

```sh
asset="$DOWNLOAD_DIR/$ASSET_NAME"
curl --fail --location --proto '=https' --proto-redir '=https' --output "$asset" "$ASSET_URL" &&
  printf '%s  %s\n' "$EXPECTED_SHA256" "$asset" | shasum -a 256 -c -
```

Select the official release APK for KernelSU Manager (package `me.weishu.kernelsu`). After checksum verification set `KSU_APK="$asset"` and run `phone install "$KSU_APK"`. Open KernelSU and confirm the phone supports the LKM installation route. The boot component will be installed after the manager patches the matching image.

## 2. Select a matching image

The three recorded stock images (`init_boot`, `boot`, `vbmeta`) come from the same OnePlus India OTA. **Only `init_boot.img` is needed for this phone’s ramdisk-based LKM root or its reversal.** The recorded target is CPH2745 / CPH2745IN, Android 17 build `CPH2745_17.0.0.102(EX01)`, kernel family `android16-6.12`, slot B. For that firmware, compare section 1’s live values with the table below. A fresh phone can ship on a different build or slot. Match its own OTA, kernel compatibility and boot partition layout, and determine the write target from the slot actually reported by the phone. [Firmware identity and OTA source](kit/firmware-identification.json) · [Known image SHA-256 digests](kit/IMAGE-SHA256SUMS) · [KernelSU installation and compatibility](https://kernelsu.org/guide/installation.html)

**Obtain the matching package.** For the recorded phone, select **OnePlus 15 India CPH2745**, full OTA **CPH2745_17.0.0.102(EX01)**, OTA ID **CPH2745_11.C.75_1750_202610181120**. Use the archive entry’s **官方链接** (official link) as the manufacturer link to resolve with the [firmware download instructions](../03-restore-to-stock/README.md#get-the-latest-full-indian-firmware). On November 21, 2026 a fresh link served this archive from `gauss-compota-c-in.allawnofs.com`. The old unsigned URL returns HTTP 403. For any other running build, select its own matching archive. Verify the package and extracted stock image before writing a boot partition. [C.75 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.0.102%28EX01%29%20C.75/) · [OnePlus India software support](https://www.oneplus.in/support/softwareupgrade) · [Recorded firmware identity](kit/firmware-identification.json)

**Verify before extracting.** Set `OTA` to the absolute path of the acquired ZIP. Run `unzip -p "$OTA" META-INF/com/android/metadata`. Match `pre-device` and the full `post-build` fingerprint. Cross-check the OnePlus product, display version and OTA ID in the accompanying vendor metadata or official package listing. The recorded target values are:

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

For a newer build, require the OTA’s embedded product/device and full target fingerprint to match the actual phone from section 1. The table above describes only the recorded build. Allow space for the full ZIP and extracted payload (about 9.24 GB each for that build), plus the selected images. Choose a build-labelled subdirectory for the images. The following block derives its name from the package’s verified display build and checks it against the selected phone. Keep that directory through recovery and verification.

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
unzip -p "$OTA" payload.bin > "$DOWNLOAD_DIR/payload.bin" &&
  payload-dumper-go -p init_boot -o "$STOCK_DIR" "$DOWNLOAD_DIR/payload.bin"
```

Keep the extractor’s source and output hash verification enabled. For the recorded **C.75 build only**, check that the complete stock image is 8,388,608 bytes and that its full SHA-256 matches the OnePlus reference. Require an `OK` result.

```sh
stock_image="$STOCK_DIR/init_boot.img"
[ "$(stat -f '%z' "$stock_image")" = 8388608 ] &&
  printf '%s  %s\n' 8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 "$stock_image" |
  shasum -a 256 -c -
```

The C.75 full-image digest covers the boot header too. To inspect the layout directly, the following commands should print the eight bytes `41 4e 44 52 4f 49 44 21` (`ANDROID!`), four zero bytes for kernel size, and `04 00 00 00` for header version 4, respectively.

```sh
od -An -tx1 -N 8 "$stock_image"
od -An -tx1 -j 8 -N 4 "$stock_image"
od -An -tx1 -j 40 -N 4 "$stock_image"
```

For another firmware, use its own payload manifest’s expected partition size and hash. Check `stat -f '%z' "$STOCK_DIR/init_boot.img"` and `shasum -a 256 "$STOCK_DIR/init_boot.img"` against those values. A source-dependent operation requires the matching base image or a suitable full package. To obtain all three stock images, select `-p init_boot,boot,vbmeta`. [C.75 image checksums](kit/IMAGE-SHA256SUMS) · [Extractor flags](https://github.com/ssut/payload-dumper-go/blob/2.1.0/cmd/root.go)

**Prepare the image to flash.** Generate a fresh patch from the verified stock image with the selected official KernelSU manager:

1. Run `phone push "$STOCK_DIR/init_boot.img" /sdcard/Download/init_boot.img`. Confirm `phone shell sha256sum /sdcard/Download/init_boot.img` matches the verified stock digest.
2. In KernelSU choose **Select a file and patch**, select the verified stock file, and confirm the detected kernel family. Read the completed patch log and set `PATCH_ON_PHONE` to its exact output path. This step creates a file.
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

Require the same serial, the expected OnePlus bootloader product, `is-userspace: no`, a fastboot slot matching the Android slot, and sufficient verified partition size. Compare the reported hex size with the byte lengths of the matching stock and patched images (`wc -c`). For the recorded layout the product was `canoe`, slot `b`, size `0x800000` (8 MiB). If the bootloader is **locked** and the owner specifically requested rooting with the required wipe, require OEM unlocking enabled and `fb flashing get_unlock_ability` reporting `1`. Then run `fb flashing unlock` and have the owner confirm the reset on the phone. After setup, reauthorize USB debugging, recheck the Android identity, recalculate the target partition and repeat the bootloader queries. A phone reporting **`unlocked: yes`** already meets the unlock requirement.

With all those checks passing and **`unlocked: yes`**, write the verified patched file from section 2 to the one confirmed active partition.

```sh
fb flash "${ROOT_PARTITION:?Verify the active init_boot partition first}" "${PATCH_TO_FLASH:?Verify the patched image first}"
```

Require successful **Sending** and **Writing** before running `fb reboot`. If the first boot fails, follow [Recover a failed first boot](#recover-a-failed-first-boot). If unlocking erased the manager, reinstall the same verified APK with `phone install "$KSU_APK"`. Open KernelSU and verify **Working / LKM**. Under **Superuser → Show system apps → Shell** (UID 2000), enable the Shell grant. Verify `phone shell su -c id` (UID 0), `phone shell getprop sys.boot_completed` (1), `phone shell getenforce` (Enforcing) and the expected fingerprint and slot.

### Recover a failed first boot

If the patch was the only partition write, the verified stock image from section 2 can undo that write. Return to the bootloader and repeat the serial, product, mode, slot, unlock-state and partition-size checks above. Confirm the active slot is still the one just flashed and the stock file still matches the digest verified before flashing. Then run `fb flash "${ROOT_PARTITION:?Recheck the original target}" "${STOCK_DIR:?Verify the stock build}/init_boot.img"`. Require successful sending and writing before `fb reboot`, then confirm Android boots on the original fingerprint. If the slot changed, the write target is uncertain or the bootloader is unreachable, stop further writes and contact OnePlus. [OnePlus India service](https://www.oneplus.in/support/repair)

## 4. Install and verify the selected modules

For each module, download the release selected in section 1 into `DOWNLOAD_DIR`. For a KernelSU ZIP, run `unzip -tq "$asset"`, inspect its module ID and version code, then `phone push "$asset" "/sdcard/Download/$ASSET_NAME"` and `phone shell sha256sum "/sdcard/Download/$ASSET_NAME"`. Compare the phone digest with the release digest. Select that file in **KernelSU → Modules → Install from storage**, confirm installer success, and reboot when required for activation.

1. **bindhosts and KernelSU-Tailscaled** are independent. For fresh bindhosts lists, verify the phone has TLS-capable `curl`, then run `phone shell "su -c '/data/adb/modules/bindhosts/bindhosts.sh --force-update'"`. Verify a blocked domain resolves to its sink address while `example.com` resolves normally. For Chrome, set the KernelSU profile to **Unmount modules off** with **Superuser off**. Select the stable ARM64 Tailscaled integration and use its verified engine updater for the official stable Tailscale engine. For an exact-version rebuild, confirm the engine version it offers matches the recorded ARM64 package before applying it. If the updater only offers a newer version, report that difference and leave the engine step pending. Verify the local service and leave sign-in to the owner. [Engine and health checks](../02-updates/README.md)
2. Install **Zygisk Next**, then **Vector** using the selected canary Release ZIP. After reboot, check `phone shell "su -c '/data/adb/modules/zygisksu/bin/zygiskd status'"` for active injection and Vector loaded, then `phone shell "su -c '/data/adb/modules/zygisk_vector/cli --json status'"` for the expected build.
3. Select and verify the chosen stable **PixelMask** APK, then run `phone install "$asset"`. Enable it in Vector for **Google Photos** and **PixelMask itself**, Android user 0 only. In PixelMask select the **original Pixel** profile. Verify enablement and scope in Vector. Keep Google Photos closed and automatic backup off. The owner manually chooses individual uploads.

After verification, refresh the [shared inventory](../README.md#installed-on-your-oneplus-15) and [current verification](../02-updates/current/verification.json) with the running versions, selected release links and digests. Record the verified firmware, patched filename and actual SHA-256. After the checks and any recovery, remove only this install run directory.
