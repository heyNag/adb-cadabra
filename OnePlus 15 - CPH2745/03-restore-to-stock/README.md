# Restore the OnePlus 15 to factory stock

On November 21, 2026, this Indian OnePlus 15 went from KernelSU on C.75 slot B to the official C.80 release on slot A, with user data erased and the bootloader locked. The owner saw **Hello ColorOS 17** after the wipe. After the owner chose to finish setup, Android reported **locked**, **green** and the AVB digest from the signed C.80 archive. The steps below describe that handset and those two builds. [Measured results](../02-updates/current/verification.json)

For the two fastboot visits, start with [the stock image write on slot B](#restore-the-c75-stock-boot-image-on-b), then follow [entry on slot A and the bootloader checks](#from-running-android-to-the-bootloader-menu) and [the lock confirmation on the phone](#show-the-lock-confirmation-and-hand-the-phone-to-the-owner).

## 1. Identify the phone and stage the two releases

Back up wanted data off the phone and verify the copy. Confirm the owner accepts loss of all local data and can access the screen-lock and previously synced account credentials that setup may require. Leave Google Photos closed. Connect the unlocked screen to the Mac with a USB data cable. With **USB debugging** enabled, accept the USB debugging authorization dialog for your Mac. In Terminal, require the intended serial to show `device` in the first command below. If it shows `unauthorized`, unlock the screen and accept the dialog before continuing. Use current Android Platform Tools and keep the same Terminal session. [Mac tool setup](../01-root-and-reinstall/README.md#0-set-up-a-fresh-mac-and-phone) [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools)

```sh
adb devices -l
printf 'Enter the serial from the authorized OnePlus row: '
read -r PHONE_SERIAL
phone() { adb -s "${PHONE_SERIAL:?Select the intended phone}" "$@"; }
fb() { fastboot -s "${PHONE_SERIAL:?Select the intended phone}" "$@"; }
phone get-state
for prop in ro.product.model ro.product.name ro.product.device ro.boot.prjname \
            ro.vendor.oplus.regionmark ro.build.display.id ro.build.fingerprint \
            ro.boot.slot_suffix ro.boot.flash.locked ro.boot.verifiedbootstate; do
  printf '%s ' "$prop"
  phone shell getprop "$prop"
done
```

Our starting values were `device`, `CPH2745`, `CPH2745IN`, `OP611FL1`, project `24863`, region `IN`, Android 17 `CPH2745_17.0.0.102(EX01)`, `_b`, `0` and `orange`. The prior root installation had patched `init_boot_b`. Different models, regions, firmware, slot layouts or a history of region conversion need their own firmware and recovery checks. OnePlus India publishes standard software update instructions and offers walk-in or pickup repair. We found no public OnePlus factory-service procedure for this exact transition. [OnePlus update guidance](https://service.oneplus.com/in/search/search-detail?id=2097031&articleIndex=1) · [OnePlus India repair](https://service.oneplus.com/in/repair)

### Get the latest full Indian firmware

Create one [private Mac working folder](../../README.md#temporary-mac-working-folder). Keep the original ZIPs, extracted stock images and the pre-flash copy in it through the last verification.

```sh
umask 077
RUN_BASE="$HOME/Downloads/adb-cadabra/OnePlus 15 - CPH2745"
mkdir -p "$RUN_BASE" &&
  RESTORE_DIR=$(mktemp -d "$RUN_BASE/factory-$(date +%Y%m%d-%H%M%S)-XXXXXXXX") &&
  printf '%s\n' "$RESTORE_DIR"
```

- **Source firmware.** Obtain the official full archive matching the running build. For C.75 follow [Obtain the matching C.75 source archive](verify-ota.md#obtain-the-matching-c75-source-archive). It covers the archived manufacturer link, authentication, extraction of stock `init_boot.img` and the return to the main run directory. This archive passed verification with the phone’s trusted OTA certificate. Extracted `init_boot.img` was 8,388,608 bytes. [C.75 archive, identity and extraction commands](../01-root-and-reinstall/README.md#2-select-a-matching-image) · [Recorded C.75 image digests](../01-root-and-reinstall/kit/IMAGE-SHA256SUMS)
- **Destination firmware.** In **Settings → About device → Software update**, read the offered build. Do the firmware download on the Mac. If Android reports **Up to date**, compare its installed build with the live India full index. For a newer unoffered release, independently match the India release notes and check rollout eligibility. Select **OnePlus 15 → India → Full** and use the verification guide to resolve the manufacturer's HTTPS URL, download, authenticate and assign `OTA` to the resulting local ZIP. A 403, error 2306 or an HTML response calls for a fresh link. [India full-package index](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info) · [Mac download, ZIP signature and payload checks](verify-ota.md) · [OnePlus release notes](https://community.oneplus.com/)
- **C.80 identification in this run.** The manufacturer’s basename was `69a6c985451e47f78054a256ac54218d.zip`. The verification command saved `CPH2745_17.0.1.201_EX01_IN-full.zip`. Both the index and OS Updater showed MD5 `51e2aa56649482ce9c6941430c333dbd`. The file measured **9,301,191,889 bytes** and SHA-256 `c7ebbcf62c64895ec9c49fe0988b6f0b40a682d95b5c8a310457afb67de5d9db`. Its signed payload contained 65 complete partition images and declared `partial_update=true`. [Package identity, signer and offer details](../02-updates/current/verification.json)

## 2. Restore the complete firmware

### Restore the C.75 stock boot image on B

**What we flashed.** KernelSU had previously patched the initial boot ramdisk in `init_boot_b`. For the restore we saved that patched image, then wrote the **original OnePlus C.75 `init_boot.img`** back to the same partition. That was the only manual partition write in the successful run. The later C.80 system update went through the built-in installer.

Before removing root, we compared all 65 active-slot partitions at their signed C.75 image lengths. Sixty-three matched. The other two were the known KernelSU patch and five bytes at the start of `splash_b`. The seven other boot-chain images matched. The update engine reported IDLE and the Boot Control HAL reported no active merge. B was current, next and bootable. A was unbootable and had an older snapshot marked Initiated. We preserved both differing images and left `splash_b` as found. To repeat the 65 comparisons while the original Shell root grant works, follow [Compare the active C.75 slot](verify-ota.md#compare-the-active-c75-slot-before-removing-root). [Partition comparisons and slot evidence](../02-updates/current/verification.json)

**Read the source slot and update state while Shell still has root.** On this C.75 build the Boot Control service exposed stable AIDL v1. First read its interface version (transaction 16777215) and hash (16777214). This phone returned version `1` and hash `2400346954240a5de495a1debc81429dd012d7b7`. Use the numeric queries in the second block only for that matching interface.

```sh
phone shell "su -c 'service call android.hardware.boot.IBootControl/default 16777215'"
phone shell "su -c 'service call android.hardware.boot.IBootControl/default 16777214'"
```

```sh
for query in '1' '2' '3' '4' '6 i32 0' '6 i32 1' '7 i32 0' '7 i32 1'; do
  printf '%s ' "$query"
  phone shell "su -c 'service call android.hardware.boot.IBootControl/default $query'"
done
phone shell "su -c 'timeout 3 /system/bin/update_engine_client --follow'" 2>&1 |
  grep -F onStatusUpdate
phone exec-out "su -c 'cat /metadata/ota/state'" | protoc --decode_raw
```

After the Binder success word, the eight results were `1, 1, 2, 0, 0, 1, 1, 1`. In order these mean next slot B, current B, two slots, no HAL merge, A unbootable, B bootable, and both slots marked successful. The three-second sample of the update engine returned `onStatusUpdate(UPDATE_STATUS_IDLE (0), 0)`. The older snapshot record still showed field 1 equal to 1 (Initiated) and field 8 identifying C.75. These are the **before-update** measurements. The later C.80 merge and slot A checks appear below.

While the existing Shell root grant still works, capture the fallback from the **confirmed C.75 active slot B**. The first block creates a new private recovery directory. On a retry, verify any existing fallback before proceeding.

```sh
C75_STOCK="$RESTORE_DIR/stock-CPH2745_17.0.0.102_EX01/init_boot.img"
RECOVERY_DIR="$RESTORE_DIR/pre-restore-CPH2745_17.0.0.102_EX01-slot-b"
FALLBACK="$RECOVERY_DIR/kernelsu-init_boot_b.img"
mkdir -m 700 "$RECOVERY_DIR" &&
  phone exec-out "su -c 'cat /dev/block/by-name/init_boot_b'" > "$FALLBACK" &&
  chmod 600 "$FALLBACK" &&
  phone exec-out "su -c 'head -c 15507456 /dev/block/by-name/splash_b'" > "$RECOVERY_DIR/splash_b.img"
```

Verify all three saved files. The successful flash used `C75_STOCK`. The other two files are readbacks from the still-rooted phone. A different patch or firmware needs its own independently identified images.

```sh
[ "$(stat -f '%z' "$C75_STOCK")" = 8388608 ] &&
  [ "$(stat -f '%z' "$FALLBACK")" = 8388608 ] &&
  [ "$(stat -f '%z' "$RECOVERY_DIR/splash_b.img")" = 15507456 ] &&
  printf '%s  %s\n' \
    8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 "$C75_STOCK" \
    e0012d4c9765dcbf2275070dd807ee4c1b691543422b9489effb0ae60aeb0a73 "$FALLBACK" \
    b30e8403705662848f7d2bc5c3595367fac6613b6b797a7104fa61e99e2628e6 "$RECOVERY_DIR/splash_b.img" |
  shasum -a 256 -c -
```

```sh
phone reboot bootloader
fastboot devices -l
for key in product is-userspace current-slot unlocked partition-size:init_boot_b; do
  fb getvar "$key"
done
```

Wait for the same USB serial to appear in `fastboot devices -l`. Require product `canoe`, `is-userspace: no`, slot `b`, `unlocked: yes` and partition size `0x800000`. With the owner’s approval, run the following **only while those C.75 slot B conditions hold**.

```sh
fastboot -s "$PHONE_SERIAL" flash init_boot_b "$C75_STOCK"
```

Require **Sending OKAY** and **Writing OKAY**. Then run `fb reboot` and wait for Android. Confirm `phone shell getprop sys.boot_completed` returns `1`, the original C.75 fingerprint and slot B still match, and `phone shell su -c id` reports that `su` is unavailable. If that first boot fails, return to the still-unlocked bootloader, repeat the same serial, mode, B and size checks and restore the verified fallback with `fb flash init_boot_b "$FALLBACK"`. This recovery applies before the upgrade to C.80. [Physical fastboot entry and target checks](../01-root-and-reinstall/README.md#3-unlock-only-if-required-then-verify-the-write-target)

We also matched this first unrooted C.75 boot against its signed AVB digest, `4f37f0cf8d94f052f2096c45c3e08c03e12b010d1c1ca7918b7a1bd45b4ef1b9`. To repeat that check from the same Terminal, save the destination ZIP path with `DESTINATION_OTA=$OTA`, set `OTA=$C75_OTA`, then follow [Calculate the AVB digest](verify-ota.md#calculate-the-avb-digest) for the booted C.75 build. Restore `OTA=$DESTINATION_OTA` before the C.80 transfer.

### Install the authenticated C.80 package and check the new boot

Check free space. Transfer the verified full ZIP from the Mac to a dedicated phone folder, then compare its byte count and SHA-256 to the Mac values. The verifier generates a filename safe for the device shell. If the file already exists, verify it before deciding whether to repeat the transfer.

```sh
phone shell df -k /sdcard/Download
PHONE_DIR=/sdcard/Download/adb-cadabra-restore
PHONE_ZIP="$PHONE_DIR/$(basename "${OTA:?Verify the destination ZIP first}")"
phone shell mkdir -p "$PHONE_DIR" &&
  phone shell test ! -e "$PHONE_ZIP" &&
  phone push -Z "$OTA" "$PHONE_ZIP"
stat -f '%z' "$OTA"
phone shell stat -c %s "$PHONE_ZIP"
shasum -a 256 "$OTA"
phone shell sha256sum "$PHONE_ZIP"
```

Open **Settings → About device → Software update → More options → Local install**. In the file picker choose the full ZIP whose bytes and checksum you verified. The Mac copy was staged under **Downloads → adb-cadabra-restore**. The owner also started the same release in OS Updater. During this run the agent told the owner to choose the manufacturer filename, `69a6c985451e47f78054a256ac54218d.zip`. The next recorded screen showed the target build. The exact file-picker path is the one unobserved step in that handoff. OS Updater’s install guide recommended this built-in option for the Indian A/B device. If **Local install** is disabled, disconnect Wi-Fi and mobile data, open **Apps → App management → Show system → Software update → Storage usage → Clear data**, then reopen **Software update → More options** offline. To open the right App info page over authorized USB, use `phone shell am start -a android.settings.APPLICATION_DETAILS_SETTINGS -d package:com.oplus.ota`. The owner reported this worked, and we verified the control was enabled beside **No network connection**. [OnePlus 15 troubleshooting](https://community.oneplus.com/thread/2241561632796835845)

Confirm the intended target on the native screen. Here it showed **CPH2745_17.0.1.201 | 8.66 GB**, while OS Updater’s download screen showed **8.87 GB** for the same filename and MD5. The owner selected **Install now**, waited for the installer and selected **Restart**. The update engine recorded successful filesystem verification and post-install. Let Android finish starting and the virtual A/B snapshot merge finish before proceeding. This run recorded `MergeCompleted` and the bootloader subsequently reported `snapshot-update-status: none`. [Installer, change log and merge evidence](../02-updates/current/verification.json)

Compute the expected AVB digest from the **authenticated destination ZIP** using the [host AVB calculation](verify-ota.md#calculate-the-avb-digest). Then read the actual Android boot state.

```sh
for prop in sys.boot_completed ro.build.display.id ro.build.fingerprint ro.boot.slot_suffix \
            ro.boot.vbmeta.digest ro.boot.flash.locked ro.boot.verifiedbootstate ro.boot.veritymode; do
  printf '%s ' "$prop"
  phone shell getprop "$prop"
done
```

The new build was Indian `CPH2745_17.0.1.201(EX01)`, slot `_a`, boot complete `1`, dm-verity `enforcing`, bootloader unlocked and Verified Boot orange. `su` was unavailable and `kernelsu` absent from `/proc/modules`. The SHA-256 AVB digests from the signed package and Android both read `b3a36735edddc817c5e7c14a05f2bda6935ab3c7d92b56632251ccb78f65b2c3`. Investigate a mismatch or failed update while the bootloader is still unlocked.

## 3. Relock, erase and verify

### From running Android to the bootloader menu

1. Keep USB connected. From the Android session above, verify the **Indian C.80 fingerprint**, slot `_a`, completed boot and matching AVB digest. Check the update engine’s retained log for the completed snapshot merge:

   ```sh
   phone logcat -d -b all -s 'update_engine:I' 'update_verifier:I' | grep -F 'Merge finished with state MergeCompleted'
   ```

   The update engine logged `Merge finished with state MergeCompleted` at 17:15:17.435 local time on this phone. An empty log search is inconclusive. If the update is still applying or merging, leave the phone in Android to finish. The bootloader snapshot check below supplied a second reading.

2. From the Mac, enter **bootloader fastboot**. The owner can leave the phone on the fastboot menu without selecting anything. `adb reboot fastboot` opens userspace fastbootd, a different mode.

   ```sh
   adb -s "${PHONE_SERIAL:?Select the authorized phone in section 1}" reboot bootloader
   ```

3. Allow USB to reconnect. Run `fastboot devices -l` until the **same serial** appears with `fastboot`. If it is missing, check the cable and screen, then repeat the device listing. Run the following **read-only** queries for that serial. Fastboot writes many results to stderr. Read the value of each response, including any `FAILED`.

   ```sh
   for key in product is-userspace current-slot unlocked secure slot-count \
              slot-successful:a slot-unbootable:a slot-retry-count:a \
              slot-successful:b slot-unbootable:b slot-retry-count:b snapshot-update-status; do
     fastboot -s "$PHONE_SERIAL" getvar "$key"
   done
   ```

| Bootloader query | Measured before the lock | What the agent checked |
|---|---|---|
| `product` / `is-userspace` | `canoe` / `no` | This is the expected bootloader product and mode. Android identified the same handset as `CPH2745IN` / `OP611FL1`. |
| `current-slot` / `slot-count` | `a` / `2` | Slot A is the C.80 installation we just booted and verified. |
| `unlocked` / `secure` | `yes` / `yes` | The bootloader was still unlocked on entry. The separate `secure` flag also read `yes`. |
| `slot-successful:a` / `slot-unbootable:a` | `yes` / `no` | The bootloader marked the active slot successful and bootable. |
| `snapshot-update-status` | `none` | No snapshot operation remained pending after the completed Android merge. |
| A retries / B successful, unbootable, retries | `6` / `yes`, `yes`, `7` | Recorded for this run. We left the active slot on A. |

For completeness, the run also queried `fb flashing get_unlock_ability` (returned `1`) and `fb getvar` for `avb-custom-key`, `has-slot:avb_custom_key` and `partition-size:avb_custom_key`. Those three returned `GetVar Variable Not found`, so they provided no custom-key evidence. The signed-image digest, successful active slot and post-lock green boot are the recorded verification.

A missing serial, fastbootd, an unexpected product or slot, a pending merge, or a failed active-slot check means to pause before the lock command. A different build needs its own signed firmware, digest and slot verification. [Android Verified Boot states](https://source.android.com/docs/security/features/verifiedboot/device-state) · [OnePlus India repair](https://www.oneplus.in/support/repair)

### Show the lock confirmation and hand the phone to the owner

1. **Agent at the Mac.** Confirm the phone passed the checks above and the owner accepts the factory wipe. With the handset still on the fastboot menu, request the confirmation **once**:

   ```sh
   fastboot -s "${PHONE_SERIAL:?Recheck the fastboot serial}" flashing lock
   ```

   On this phone the Mac returned `OKAY [  0.008s]` and the bootloader displayed its lock confirmation. The Mac response acknowledges the request. The next action belongs to the owner.

2. **Owner at the phone.** When you see **LOCK THE BOOTLOADER**, use the volume keys to highlight that exact option. **Press and release Power once** to confirm the lock and factory wipe. Leave USB connected and let the phone restart. If the screen shows an error or a different choice, leave it as it is and tell the agent what it says. If the prompt is delayed, wait and inspect the handset while keeping USB connected.
3. **Observe the first boot.** The owner saw the OnePlus logo, ColorOS loading and **Hello ColorOS 17**. That welcome screen is the handoff point for a phone intended to stay in first-time setup. This run used the wipe triggered by lock confirmation.

### Verify the completed wipe and locked boot

For a direct Android readout, the owner of this unit chose to complete setup without Google sign-in. At Home, open **Settings → About device → Version**, tap **Version number** seven times to enable Developer options, then search Settings for **Developer options** and enable **USB debugging**. The owner accepted the Mac’s USB authorization prompt and left **OEM unlocking** untouched. Check that `adb devices -l` again shows the intended serial and `phone get-state` returns `device`.

```sh
for prop in ro.boot.flash.locked ro.boot.vbmeta.device_state ro.boot.verifiedbootstate \
            ro.boot.vbmeta.digest ro.build.display.id ro.build.fingerprint ro.boot.slot_suffix sys.boot_completed; do
  printf '%s ' "$prop"
  phone shell getprop "$prop"
done
phone shell settings get global device_provisioned
phone shell getenforce
phone shell su -c id
phone shell cat /proc/modules | grep -iw kernelsu
phone shell pm path --user 0 me.weishu.kernelsu
phone shell pm path --user 0 com.kinginu.pixelmask
```

Android returned **`1`, `locked`, `green`**, the same signed C.80 AVB digest and Indian fingerprint, slot `_a`, boot complete `1` and provisioned `1`. SELinux was `Enforcing`. The `su` command returned exit 127 (command unavailable), the module search produced no match, and both package-path queries returned no APK path for user 0. Repeat the [host-to-phone AVB comparison](verify-ota.md#calculate-the-avb-digest) to verify the post-lock digest too. The pre-wipe orange boot changed to green on the first verified, locked C.80 boot. Keep recovery material until verification is complete. The previous components and settings remain in the [source-only rebuild guide](../01-root-and-reinstall/README.md#recorded-rooted-components). [Raw restore evidence](../02-updates/current/verification.json) · [Android boot-flow indicators](https://source.android.com/docs/security/features/verifiedboot/boot-flow)

### Scope of other reports

OnePlus lists distinct India, North America and EU/global packages with the same .102 version suffix. The US CPH2749 owner restored the original North American .102 slot after a PIN/eSE failure and reported a locked green boot. The Indian C.75 kernel report documents an unlocked orange boot. The Indian C.80 locked/green result above was measured on this handset. [OnePlus regional release announcement](https://community.oneplus.com/thread/2241561632796835845) · [US owner’s report](https://www.reddit.com/r/oneplus/comments/1xv0qo5/oneplus_15_on_coloros_1700102_pin_rejected/) · [Indian C.75 kernel test](https://github.com/travismills82/oneplus15-sm8850-oos-kernel/releases/tag/coloros17.0.0.102-linux-6.12.115-kernelsu-susfs-canary)

Other examples concern different variants or conversions. iFixDroid explains the stock prerequisite at 0:38 and leaves the relock command unexecuted at 4:16. [French CPH2747 discussion](https://xdaforums.com/posts/90566499/) · [French follow-up](https://xdaforums.com/posts/90567414/) · [Canadian result](https://xdaforums.com/posts/90642117/) · [System Hacks’ China conversion](https://www.youtube.com/watch?v=QhOwUawyWQw&t=640s) and [build screen](https://www.youtube.com/watch?v=QhOwUawyWQw&t=677s) · [TechX Studio’s conversion](https://www.youtube.com/watch?v=mlIp5sO3OBM&t=777s) · [iFixDroid’s explanation](https://www.youtube.com/watch?v=awbeXSDwivo)
