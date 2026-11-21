# Clean install or redo a rooted OnePlus 15

On a fresh phone running official, unrooted firmware, start at [Set up a fresh Mac and phone](#0-set-up-a-fresh-mac-and-phone). For a phone that still has active KernelSU, the redo procedure below starts with a wipe. For a factory-to-root rebuild, use the latest compatible release on each channel in the [component table](../README.md#verified-setup-on-the-reference-phone) and restore the settings in section 4. The tagged versions identify what was tested if an exact-version replay is requested.

## Redo on an already-rooted phone

For this development phone, a redo can discard all user data, apps, accounts, root grants and module settings. The intended result is a fresh **rooted** setup with the bootloader still unlocked. An **already-unlocked, unrooted** phone with existing user data also needs the owner-approved reset in steps 1–2 for a clean install. After that reset, continue at section 0 and use the already-unlocked path in section 3.

1. **Identify the phone before erasing.** Record the connected phone’s model, firmware fingerprint, active slot and running KernelSU build. Use the [device checks](../02-updates/README.md#1-read-the-phone) and the [installed inventory](../README.md#verified-setup-on-the-reference-phone). Have the screen PIN and previously synced Google account credentials available if Android requests them during setup. [Android factory-reset guidance](https://support.google.com/android/answer/6088915)
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
phone shell getprop ro.boot.prjname
phone shell getprop ro.vendor.oplus.regionmark
phone shell getprop ro.build.display.id
phone shell getprop ro.build.fingerprint
phone shell getprop ro.boot.slot_suffix
phone shell getprop ro.boot.flash.locked
phone shell getprop ro.boot.vbmeta.device_state
phone shell getprop sys.boot_completed
phone shell uname -r
```

Require `phone get-state` to report `device` and boot completion to report `1`. This walkthrough targets model `CPH2745`, product `CPH2745IN`, device `OP611FL1`, ABI `arm64-v8a`, project `24863` and region `IN`. Check those identities, record the full installed fingerprint, active slot and lock state, and select the firmware for that handset. Other variants or converted phones require their own verified procedure.

**Choose which releases to install.** By default, use **Latest on this channel** for the newest compatible stable releases and Vector’s canary **Release** ZIP. When reproducing the recorded versions, use **Installed from** in the device overview for the exact release tags, build numbers and assets. For the first root check after erasing an already-rooted phone, install the exact Manager APK matching its previously running KernelSU build. In either mode, check Android, kernel, ABI and dependency compatibility with the current phone. [Recorded rooted components](#recorded-rooted-components) · [KernelSU releases](https://github.com/tiann/KernelSU/releases)

For each installer or extractor release archive, set `ASSET_URL`, `ASSET_NAME` and `EXPECTED_SHA256` from the selected official release. Use the exact asset filename and the 64-character hex digest supplied by the release or its GitHub asset metadata. On GitHub, expand **Assets** to copy the download link. For a release with a GitHub asset digest, open `https://api.github.com/repos/OWNER/REPO/releases/tags/TAG` using the owner, repository and tag from that release. In the matching `assets` entry, use `browser_download_url`, `name` and the hex characters after `sha256:` in `digest`. Run the download and checksum together and require an `OK` result before installation.

```sh
asset="$DOWNLOAD_DIR/$ASSET_NAME"
curl --fail --location --proto '=https' --proto-redir '=https' --output "$asset" "$ASSET_URL" &&
  printf '%s  %s\n' "$EXPECTED_SHA256" "$asset" | shasum -a 256 -c -
```

Select the official release APK for KernelSU Manager (package `me.weishu.kernelsu`). After checksum verification set `KSU_APK="$asset"` and run `phone install "$KSU_APK"`. On the locked C.80 phone, OnePlus showed its installer scan while `adb install` was still waiting. Verify the app name and version, read the scan result, then choose **Continue installation** when it reports **No risks found**. Wait for the Mac to report `Success`. Open KernelSU. On an unrooted supported handset the upstream Manager shows **Not installed**. A Manager status of **Unsupported** calls for a different supported installation method before proceeding. The boot component will be installed after the manager patches the matching image.

### Recorded rooted components

Use the device overview’s [installed versions, channels and source links](../README.md#verified-setup-on-the-reference-phone) for all seven components. **Vector** was the only component installed from canary. Choose its `*-Release.zip` asset. PixelMask 1.2.0 needed Vector’s Android 17 compatibility fix. KernelSU used the official stable Manager. For Tailscaled, select the stable ARM64 integration ZIP and use its engine updater for the official ARM64 Tailscale engine. [PixelMask compatibility notes](https://github.com/kinginu/PixelMask/releases/tag/18-1.2.0)

For an exact replay of the November 21 rebuild, `clean_rebuild` in the verification record identifies its inputs. The earlier C.75 baseline is recorded separately under `rooted_rebuild_baseline`. [Recorded builds and artifacts](../02-updates/current/verification.json)

## 2. Select a matching image

**Start with the build on the phone.** On November 21, 2026, the Indian handset started the clean rebuild on stock `CPH2745_17.0.1.201(EX01)` (C.80), Android 17, slot A, locked with green Verified Boot. Its kernel was `6.12.69-android16-6-g5fd39602326e-ab16372235-4k`. The device uses a separate `init_boot` ramdisk. For LKM on this layout, use the stock `init_boot.img` from the running firmware and generate a fresh patch. Keep the matching stock image on the Mac for recovery. [KernelSU installation](https://kernelsu.org/guide/installation.html) · [Handset and signed firmware measurements](../02-updates/current/verification.json)

For the C.80 case, use the [India full-package download and signer checks](../03-restore-to-stock/verify-ota.md). A private Mac run from a previous restore may already contain the official ZIP. Reuse it once you have matched its identity to the running phone, checked its digest against the authenticated record, and extracted a new stock image into the current install run. Set `OTA` to that verified ZIP. When downloading from scratch into this run, set `RESTORE_DIR=$DOWNLOAD_DIR` for the verifier. If the full-package index has advanced beyond the installed C.80, follow [Obtain the matching C.80 source archive](../03-restore-to-stock/verify-ota.md#obtain-the-matching-c80-source-archive), then use the authenticated C.80 ZIP as `OTA` here. For a different installed build, obtain and authenticate its own full India archive. The flash target is always determined again from the live slot.

| Checked field | Indian C.80 value |
|---|---|
| OTA identity | `CPH2745_11.C.80_1800_202610290517` |
| Product / device | `CPH2745IN` / `OP611FL1` |
| Full OTA size / SHA-256 | 9,301,191,889 bytes / `c7ebbcf62c64895ec9c49fe0988b6f0b40a682d95b5c8a310457afb67de5d9db` |
| Stock `init_boot.img` size / SHA-256 from signed manifest | 8,388,608 bytes / `cf870b50cf44845c5e9070afe7a13188aa5a13efc1bc7cddb42f49f217074420` |
| Full target fingerprint | `OnePlus/CPH2745IN/OP611FL1:17/CP2A.260605.016/B.R4T3.50a3663-29f0eba-29fce1c:user/release-keys` |

**Earlier C.75 build only.** If the running phone reports C.75, select **OnePlus 15 India CPH2745**, full OTA **CPH2745_17.0.0.102(EX01)**, OTA ID **CPH2745_11.C.75_1750_202610181120**. Set `RESTORE_DIR=$DOWNLOAD_DIR`. Follow the [matching C.75 archive procedure](../03-restore-to-stock/verify-ota.md#obtain-the-matching-c75-source-archive) to resolve **官方链接**, verify the complete archive and extract the stock image. That procedure explains how to restore the destination variables if you are also preparing a factory restore. For a C.75 root install, set `OTA=$C75_OTA` before continuing here. On November 21, 2026 a fresh link served this archive from `gauss-compota-c-in.allawnofs.com`. The old unsigned URL returns HTTP 403. For any other running build, select its own matching archive. Verify the package and extracted stock image before writing a boot partition. [C.75 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.0.102%28EX01%29%20C.75/) · [OnePlus India software support](https://www.oneplus.in/support/softwareupgrade) · [Recorded firmware identity](kit/firmware-identification.json)

**Verify before extracting.** Set `OTA` to the absolute path of the acquired ZIP. Run `unzip -p "$OTA" META-INF/com/android/metadata`. Match `pre-device` and the full `post-build` fingerprint. Cross-check the OnePlus product, display version and OTA ID in the accompanying vendor metadata or official package listing. The following table describes the earlier C.75 example:

| OTA field | Required value |
|---|---|
| `pre-device` | `OP611FL1` |
| `product_name` / `version_name` | `CPH2745IN` / `CPH2745_17.0.0.102(EX01)` |
| `ota-id` | `CPH2745_11.C.75_1750_202610181120` |
| `post-build` | `OnePlus/CPH2745IN/OP611FL1:17/CP2A.260605.016/B.R4T3.55357f5-2eb9868-2eda512:user/release-keys` |

### Get the Mac OTA extractor

Use this setup before the first `payload-dumper-go` command, including when following the factory restore guide. In a clean-install shell the private run is `DOWNLOAD_DIR`. In a factory-restore shell use its existing `RESTORE_DIR`. Keep the same Terminal. Ensure Python 3 and the upstream XZ dependency are available (`xz --version`). The following block selects the regular ARM64 or Intel macOS archive from the latest official stable release, checks its published SHA-256 and extracts the command into the temporary run. [Official releases](https://github.com/ssut/payload-dumper-go/releases) · [Extractor prerequisites](https://github.com/ssut/payload-dumper-go#cautions) · [XZ for macOS](https://formulae.brew.sh/formula/xz)

```sh
EXTRACTOR_DIR="${DOWNLOAD_DIR:-${RESTORE_DIR:?Create the private run directory first}}/extractor"
python3 - "$EXTRACTOR_DIR" <<'PY'
import hashlib, json, platform, re, sys, tarfile
from pathlib import Path
from urllib.request import Request, urlopen

out = Path(sys.argv[1])
arch = {"arm64": "arm64", "x86_64": "amd64"}.get(platform.machine())
if sys.platform != "darwin" or not arch:
    sys.exit("Choose the upstream extractor release for this host.")
api = "https://api.github.com/repos/ssut/payload-dumper-go/releases/latest"
with urlopen(Request(api, headers={"User-Agent": "adb-cadabra"}), timeout=30) as response:
    release = json.load(response)
pattern = rf"payload-dumper-go_[A-Za-z0-9.-]+_darwin_{arch}\.tar\.gz"
matches = [a for a in release["assets"] if re.fullmatch(pattern, a["name"])]
if len(matches) != 1:
    sys.exit("Inspect the upstream stable release for the matching Mac archive.")
asset = matches[0]
digest = re.fullmatch(r"sha256:([0-9a-f]{64})", asset.get("digest") or "")
if not digest:
    sys.exit("Verify the asset against the upstream release checksums before use.")
out.mkdir(parents=True, exist_ok=True)
archive = out / asset["name"]
if archive.is_file():
    data = archive.read_bytes()
else:
    with urlopen(asset["browser_download_url"], timeout=60) as response:
        data = response.read()
if len(data) != asset["size"] or hashlib.sha256(data).hexdigest() != digest[1]:
    sys.exit("Extractor release size or SHA-256 mismatch.")
archive.write_bytes(data)
with tarfile.open(archive, "r:gz") as bundle:
    member = bundle.getmember("payload-dumper-go")
    if not member.isfile():
        sys.exit("Expected a regular payload-dumper-go executable in the release.")
    with bundle.extractfile(member) as src:
        binary = out / "payload-dumper-go"
        binary.write_bytes(src.read())
binary.chmod(0o700)
print(release["tag_name"], asset["name"], digest[1])
PY
```

When the verified release, filename and checksum print, add that directory to this Terminal and check the executable:

```sh
export PATH="$EXTRACTOR_DIR:$PATH"
"$EXTRACTOR_DIR/payload-dumper-go" -h
```

### Extract and verify the stock image

For every build, require the OTA’s embedded product/device and full target fingerprint to match the actual phone from section 1. Allow space for the full ZIP and the selected images. The upstream extractor reads the payload directly from the archive. Choose a build-labelled subdirectory for the images. The following block derives its name from the package’s verified display build and checks it against the selected phone. Keep that directory through recovery and verification.

```sh
STOCK_DIR=$(python3 - "$OTA" "$DOWNLOAD_DIR" "$(phone shell getprop ro.build.display.id)" \
  "$(phone shell getprop ro.build.fingerprint)" "$(phone shell getprop ro.product.device)" \
  "$(phone shell getprop ro.product.name)" <<'PY'
import re, sys, zipfile
from pathlib import Path
ota, run, phone_build, fingerprint, device, product = sys.argv[1:]
with zipfile.ZipFile(ota) as archive:
    metadata = archive.read("META-INF/com/android/metadata").decode().splitlines()
fields = dict(line.split("=", 1) for line in metadata if "=" in line)
build = fields.get("version_name", "")
match = re.fullmatch(r"(CPH2745_[0-9]+(?:\.[0-9]+)+)\(([A-Za-z0-9]+)\)", build)
if not match or build != phone_build:
    sys.exit("The OTA version_name must match the selected phone's installed build.")
if (device, product) != ("OP611FL1", "CPH2745IN") or not fingerprint:
    sys.exit("Select the intended Indian OnePlus handset and verify its fingerprint.")
if any(fields.get(key) != value for key, value in (
        ("pre-device", device), ("product_name", product), ("post-build", fingerprint))):
    sys.exit("The archive device, product or full target fingerprint differs from the phone.")
print(Path(run) / f"stock-{match[1]}_{match[2]}")
PY
) && printf '%s\n' "$STOCK_DIR"
```

After the printed directory matches the installed build, extract the stock image.

```sh
stock_image="${STOCK_DIR:?Match the archive to the running phone first}/init_boot.img"
payload-dumper-go -p init_boot -o "$STOCK_DIR" "${OTA:?Select the authenticated archive}"
```

Keep the extractor’s output hash verification enabled. **For the recorded C.75 or C.80 build**, run the next block and require `OK`. **For any other build**, skip directly to [Check an image against its own signed manifest](#check-an-image-against-its-own-signed-manifest).

```sh
case "$(phone shell getprop ro.build.display.id)" in
  'CPH2745_17.0.1.201(EX01)') image_sha=cf870b50cf44845c5e9070afe7a13188aa5a13efc1bc7cddb42f49f217074420 ;;
  'CPH2745_17.0.0.102(EX01)') image_sha=8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 ;;
  *) unset image_sha ;;
esac
[ "$(stat -f '%z' "$stock_image")" = 8388608 ] &&
  printf '%s  %s\n' "${image_sha:?Use the matching signed manifest for this firmware}" "$stock_image" |
  shasum -a 256 -c -
```

For C.75, the full-image digest covers the boot header too. To inspect its layout directly, the following commands should print the eight bytes `41 4e 44 52 4f 49 44 21` (`ANDROID!`), four zero bytes for kernel size, and `04 00 00 00` for header version 4, respectively.

```sh
od -An -tx1 -N 8 "$stock_image"
od -An -tx1 -j 8 -N 4 "$stock_image"
od -An -tx1 -j 40 -N 4 "$stock_image"
```

### Check an image against its own signed manifest

For another firmware, authenticate its matching full OTA and decode the manifest with the [whole-file signer and payload check](../03-restore-to-stock/verify-ota.md#check-the-whole-file-signer-and-payload), keeping `RESTORE_DIR=$DOWNLOAD_DIR` for that package. Run the following build-independent check on the extracted image and its manifest. SHA-256 in the decoded Protobuf text is an escaped byte string. Convert it to hex before comparing. A source-dependent payload needs its matching base image or another full package.

```sh
python3 - "${VERIFY_DIR:?Authenticate this exact ZIP first}/manifest.txt" "$STOCK_DIR/init_boot.img" <<'PY'
import ast, hashlib, re, sys
from pathlib import Path

manifest, image = map(Path, sys.argv[1:])
blocks = re.split(r"^partitions \{", manifest.read_text(), flags=re.M)[1:]
matches = [b for b in blocks if re.search(r'^  partition_name: "init_boot"$', b, re.M)]
if len(matches) != 1:
    sys.exit("Expected exactly one init_boot record in the authenticated manifest.")
info = re.search(r"^  new_partition_info \{(.*?)^  \}", matches[0], re.M | re.S)
if not info:
    sys.exit("Missing target image information.")
size = re.search(r"^    size: (\d+)$", info[1], re.M)
digest = re.search(r'^    hash: (".*")$', info[1], re.M)
if not size or not digest:
    sys.exit("The signed manifest must provide a target size and SHA-256.")
expected = ast.literal_eval("b" + digest[1]).hex()
if image.stat().st_size != int(size[1]) or hashlib.sha256(image.read_bytes()).hexdigest() != expected:
    sys.exit("The extracted init_boot differs from its signed firmware manifest.")
print(image.name, size[1], "bytes, SHA-256", expected, "OK")
PY
```

To obtain all three stock images, select `-p init_boot,boot,vbmeta`. [C.75 image checksums](kit/IMAGE-SHA256SUMS) · [Extractor flags](https://github.com/ssut/payload-dumper-go/blob/2.1.0/cmd/root.go)

**Before unlocking a stock phone.** Steps 1–3 below create a file while Android still has its original USB authorization. Copy the successful patch to this run on the Mac before the bootloader unlock wipes the phone. Keep the Manager APK and both images there. After the wipe, reauthorize USB and compare the new live fingerprint, kernel and active slot again before flashing. A change of firmware requires new matching inputs.

**Prepare the image to flash.** Generate a fresh patch from the verified stock image with the selected official KernelSU manager:

1. Create a dedicated folder under the phone’s Downloads, for example `PHONE_DIR=/sdcard/Download/adb-cadabra-install-CPH2745-C80` for this run. Run `phone shell mkdir -p "$PHONE_DIR"`, `phone push "$STOCK_DIR/init_boot.img" "$PHONE_DIR/init_boot.img"`, and `phone shell sha256sum "$PHONE_DIR/init_boot.img"`. Require the checksum from the signed firmware.
2. In KernelSU choose the install icon → **Select a file**. In the system picker, open **OnePlus 15 → Download → adb-cadabra-install-CPH2745-C80 → init_boot.img**. The Downloads shortcut showed an empty listing in this run. In the kernel selector, compare the detected **KMI** (kernel module interface) with `phone shell uname -r`. On this **Android 17** phone the running kernel was `6.12.69-android16-6-…`, and the correct selection in KernelSU was **android16-6.12**. The Android version in this label identifies the kernel branch used to build the module. Leave the matching selection in place. Confirm the patch and wait for the log to report unpacking, adding the KernelSU LKM, repacking, an output path, and **Done**. Set `PATCH_ON_PHONE` to that exact path.
3. Set `PATCH_TO_FLASH="$DOWNLOAD_DIR/${PATCH_ON_PHONE##*/}"`, then run `phone pull "$PATCH_ON_PHONE" "$PATCH_TO_FLASH"`. Compare `phone shell sha256sum "$PATCH_ON_PHONE"` with `shasum -a 256 "$PATCH_TO_FLASH"`. Require identical hashes, a file length matching the intended partition, and a successful patch log. A newly generated patch has its own digest. Use its actual filename and hash even when the same KernelSU release was used before. Keep the verified stock image accessible on the Mac through the first boot.

## 3. Unlock only if required, then verify the write target

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
fb() { fastboot -s "${PHONE_SERIAL:?Select the intended handset}" "$@"; }
fb getvar product
fb getvar is-userspace
fb getvar current-slot
fb getvar unlocked
fb getvar secure
fb getvar "partition-size:$ROOT_PARTITION"
fb getvar "slot-successful:${ANDROID_SLOT#_}"
fb getvar "slot-unbootable:${ANDROID_SLOT#_}"
fb getvar snapshot-update-status
```

Require the same USB serial, `product: canoe`, `is-userspace: no`, the same slot Android reported and a partition size matching both image lengths. Check each response before the unlock and again before the partition write. For this verified handset, require `secure: yes`, active slot successful `yes`, active slot unbootable `no`, and snapshot status `none`. If an update or merge is pending, finish that update in Android and re-identify the installed firmware and target. On the recorded C.80 rebuild the target was `init_boot_a`, `partition-size:init_boot_a` was `0x800000` (8 MiB), `unlocked` was `no` and `secure` was `yes`. Slot A was successful and bootable, and `snapshot-update-status` was `none`. On the older C.75 run the target was on B. Leave slot selection to the verified boot state.

**Unlock a locked phone with an owner-approved wipe.** Check `fb flashing get_unlock_ability`. This C.80 phone returned `1` after the owner-authorized OEM unlocking setting was confirmed on in Developer options. A zero or unsupported response needs investigation before continuing.

1. With the phone in the confirmed bootloader, run `fastboot -s "$PHONE_SERIAL" flashing unlock` **once**. The C.80 handset acknowledged the request with `OKAY [  0.010s]`. Check the confirmation on the phone.
2. On the handset, the **owner** uses the volume keys to highlight **UNLOCK THE BOOTLOADER**, then presses and releases **Power once**. This confirms the data wipe. Keep USB connected through the restart. If a different prompt appears, have the owner report its wording before making a selection.
3. After first-time setup, re-enable **Developer options → USB debugging** and approve the Mac. Require `adb devices -l` to show the same serial as `device`, then `phone get-state` to return `device`. Re-read section 1’s Android identity, boot completion, fingerprint, kernel, lock state and slot. Compare them with the saved patch inputs. Recalculate `ANDROID_SLOT` and `ROOT_PARTITION` using the first block of this section, run `phone reboot bootloader`, and repeat the complete query block above. Require `unlocked: yes`, the expected active slot and size, successful/bootable flags and no snapshot operation. A handset already reporting `unlocked: yes` skips the unlock and its wipe.

Immediately before writing, recheck both Mac file sizes against the reported partition capacity with `wc -c "$PATCH_TO_FLASH" "$STOCK_DIR/init_boot.img"`. Recheck both hashes with `shasum -a 256 "$PATCH_TO_FLASH" "$STOCK_DIR/init_boot.img"` against the successful phone patch pull and the signed stock reference. With those checks passing and **`unlocked: yes`**, write only the freshly patched file to the confirmed active partition.

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
3. Install the verified stable **PixelMask** APK with `phone install "$asset"`. Confirm the version with `phone shell dumpsys package com.kinginu.pixelmask`, then follow the scope and runtime checks below.

### Restore and verify the PixelMask scope

The reference setup has two Vector targets for Android user 0: **PixelMask** and **Google Photos**. The initial C.80 rebuild enabled only PixelMask itself. We later restored the second target and verified the hook inside a fresh Google Photos process. [Current and pre-reset scope records](../02-updates/current/verification.json)

1. In the same Terminal, use the [serial-selected Vector helper](../02-updates/README.md#1-read-the-phone). Run `phone shell am get-current-user` (expected `0`), `phone shell pm path --user 0 com.google.android.apps.photos`, `vector modules ls` and `vector scope ls com.kinginu.pixelmask`. An installed Photos package returns an APK path. On a clean install with a **confirmed empty scope**, the original rebuild used `vector scope set com.kinginu.pixelmask com.kinginu.pixelmask/0` to enable the module for itself. On an existing installation, use `scope add` for any missing entry to preserve the other targets. If PixelMask is already enabled and lists itself, continue to step 2.
2. If the Google Photos target is missing, append it. This preserves the existing scope. Skip the first command when Photos is already listed.

   ```sh
   vector scope add com.kinginu.pixelmask com.google.android.apps.photos/0
   vector scope ls com.kinginu.pixelmask
   vector modules ls
   ```

   For the reference setup, the complete scope lists `com.google.android.apps.photos` and `com.kinginu.pixelmask`, both for user 0. The module list shows PixelMask as `enabled`. An existing phone may have additional intentional targets. [Vector scope implementation](https://github.com/JingMatrix/Vector/blob/canary-3119/daemon/src/main/kotlin/org/matrix/vector/daemon/data/ModuleDatabase.kt#L285-L334)
3. Restart **Google Photos** so the new process loads the hook, then open **PixelMask**. On the tested phone the following commands launched the apps. If a later release changes a launcher activity, force-stop Photos from its App info, open it from the launcher, then open PixelMask. This check uses PixelMask’s Home screen and leaves backup settings alone.

   ```sh
   phone shell am start -W -S --user 0 -n com.google.android.apps.photos/.home.HomeActivity
   phone shell am start -W --user 0 -n com.kinginu.pixelmask/.MainActivity
   ```

4. Look for **Active in Google Photos** and a fresh report. The verified screen read **“Spoofing as Pixel · Photos 7.86.0.956040398 · just now.”** On a future run, expect the installed Photos version and a report from that run. **Not applied to Google Photos yet** means PixelMask has no Photos report. If it persists, recheck the Android user, both scope entries and module enablement, then restart Photos and reopen PixelMask. The result verifies the local Photos process. [PixelMask 1.2.0 status logic](https://github.com/kinginu/PixelMask/blob/18-1.2.0/app/src/main/java/com/kinginu/pixelmask/ui/screens/Home.kt#L145-L204)

After verification, refresh the [shared inventory](../README.md#verified-setup-on-the-reference-phone) and [current verification](../02-updates/current/verification.json) with the running versions, selected release links and digests. Record the verified firmware, patched filename and actual SHA-256. Keep this run’s stock fallback through boot, module activation and verification. When cleaning up the temporary downloads, remove only the selected run directory.

## Background on other unlock methods

The tested Indian CPH2745 procedure used the Platform Tools commands in section 3. The following references cover other tools and device variants.

Doug Lynch’s independent WebUSB tool provides a Chromium browser interface for unlocking a supported device after OEM unlocking is enabled. Completing an unlock erases user data. [Explaining Android bootloader unlock tool](https://explainingandroid.com/bootloader-unlock/)

For a OnePlus 15 sold in mainland China, OnePlus’s official ColorOS Upgrade Assistant describes how to apply for Deep Testing on supported ColorOS 16 and later releases. Check the updated notice for current device and account eligibility. For the Indian CPH2745, follow the serial and bootloader checks below when rooting, or use the factory-restore guide when returning to locked stock. [Original OnePlus announcement](https://bbs.oneplus.com/thread/1926504022886318086) · [Updated OnePlus eligibility rules](https://bbs.oneplus.com/thread/2175833961601695750) · [Factory restore and relock](../03-restore-to-stock/README.md) · [Third-party promotional clip claiming a remote unlock on Chinese PLK110](https://www.youtube.com/watch?v=s1F3sucMW4I)
