# Restore the OnePlus 15 to factory stock

The intended result is an Indian OnePlus 15 CPH2745 running official OnePlus firmware, with user data erased, KernelSU and its modules removed, the bootloader locked and Android Verified Boot enforcing the manufacturer’s key. The latest pre-restore check found slot B still running the known KernelSU patch, with an unlocked bootloader and orange Verified Boot. [Current phone state](../02-updates/current/verification.json)

## 1. Identify the phone and obtain a supported restore route

Back up wanted data off the phone and verify you can access it there. A factory wipe removes local accounts, apps, module settings, Tailscale’s private device identity and USB debugging authorization. Verify access to the Google account and screen-lock credentials that setup may require.

If Android boots and USB debugging is authorized, use current Android Platform Tools on the Mac. From the device list, set `PHONE_SERIAL` to the intended phone’s serial. These commands only read its present state. [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools)

```sh
adb devices -l
printf 'Enter the serial from the authorized OnePlus row: '
read -r PHONE_SERIAL
phone() { adb -s "${PHONE_SERIAL:?Set PHONE_SERIAL first}" "$@"; }
phone get-state
for prop in ro.product.model ro.product.name ro.product.device ro.boot.prjname \
            ro.vendor.oplus.regionmark ro.build.display.id ro.build.fingerprint ro.boot.slot_suffix \
            ro.boot.flash.locked ro.boot.vbmeta.device_state ro.boot.verifiedbootstate; do
  printf '%s ' "$prop"
  phone shell getprop "$prop"
done
```

Require `device` from `phone get-state`. On this unit, the product is **CPH2745IN**, device **OP611FL1**, project **24863**, vendor region **IN**. Active slot B contains a known KernelSU-patched `init_boot_b`. With the existing Shell root grant, compare `phone shell "su -c 'sha256sum /dev/block/by-name/init_boot_b'"` with the recorded patch digest. Select the actual active partition when checking a different phone or slot. [Recorded C.75 image hashes](../01-root-and-reinstall/kit/IMAGE-SHA256SUMS)

**Check OnePlus’s public guidance first.** The India software page describes ordinary Android updates. Its repair page offers walk-in or pickup service. On November 21, 2026, neither published an owner-executable CPH2745 factory restore and relock sequence. A regular OTA and a factory service package serve different purposes. [OnePlus software update instructions](https://service.oneplus.com/in/search/search-detail?id=2097031&articleIndex=1) · [India repair options](https://service.oneplus.com/in/repair)

**If an applicable public procedure is unavailable**, the owner can request one or arrange service. Give OnePlus the model and sales region, observed product and project, current build fingerprint, slot, lock and Verified Boot states, and known `init_boot_b` modification. Request the approved target for this device’s rollback level, the official package and tool (or service-only route), required partition and slot coverage, the pre-lock checks and lock/wipe sequence, and recovery instructions if boot validation fails. The two slots can legitimately be on different builds. Before any partition write, establish how the complete procedure reaches the locked, erased welcome screen.

### Get the latest full Indian firmware

1. On the phone, open **Settings → About device**, tap the software-version card or **New version available**, and read the offered build. Use this screen for identification only. Leave download and installation untouched. For this phone it displayed `CPH2745_17.0.1.201 | 2.17 GB`. If the updater says **Up to date**, compare the installed display build with the live India full-package metadata. For a newer, unoffered build, independently identify the exact CPH2745 India release in OnePlus release notes and check its rollout eligibility before installation. In OnePlus Community, search the model, India and the build number. [OnePlus Community](https://community.oneplus.com/)
2. On the Mac, select **OnePlus 15 → India → Full** in the firmware index. Compare its `version_number` with the intended target. The index supplies a transport `md5sum` and the manufacturer's `download_url`. For the recovery image of an already-patched partition, select the exact installed build’s archive instead. [India full-package metadata](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info) · [C.75 archive and extraction](../01-root-and-reinstall/README.md#2-select-a-matching-image)
3. Create one [temporary Mac working folder](../../README.md#temporary-mac-working-folder) in Terminal. Use it through download, extraction, checks, and any approved restore or recovery. Keep stock images in a subdirectory that names their source build, for example `stock-CPH2745_17.0.0.102_EX01/init_boot.img` for C.75.

```sh
umask 077
RUN_BASE="$HOME/Downloads/adb-cadabra/OnePlus 15 - CPH2745"
mkdir -p "$RUN_BASE" &&
  RESTORE_DIR=$(mktemp -d "$RUN_BASE/factory-$(date +%Y%m%d-%H%M%S)-XXXXXXXX") &&
  printf '%s\n' "$RESTORE_DIR"
```

4. Resolve the `download_url` through the index's download-link resolver on the Mac. It takes the URL-encoded `url` parameter at `https://oosdownloader-gui.fly.dev/api/ota/resolve-url` and returns the fresh manufacturer URL in its `url` field. The India package was served over HTTPS by `gauss-compota-c-in.allawnofs.com`. Keep time-limited URLs out of notes and command logs. An HTTP 403, error 2306 or an HTML response means to obtain a fresh link and verify the response again. [Firmware index](https://oosdownloader-gui.fly.dev/)
5. [Download and verify the full OTA on the Mac](verify-ota.md). Check the manufacturer response, size, transport checksum, ZIP integrity, product, firmware identity, whole-file signer and payload coverage. This stage supplies stock images. A complete factory restore requires the OnePlus procedure in section 2.

**Verified for the current C.80 release.** The full ZIP passed its transport checksum, ZIP integrity checks and Android whole-file signature verification against the connected phone’s trusted OTA signing key. Its payload contains complete images for 65 partitions with no dependency on old partition data, including `boot`, `init_boot` and `vbmeta`. The manifest also sets `partial_update=true`. This release does not claim to cover every partition in a factory service flash. The owner’s requested end state still requires a supported restoration and verified locked boot. [Current verification](../02-updates/current/verification.json) · [Android OTA manifest definition](https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto#435)

## 2. Restore the complete firmware

If OnePlus supplies an owner-executable procedure, obtain its prescribed package and tools from the manufacturer onto the Mac in the same run directory. Establish the recovery path while the original phone still boots.

Check the package identity and any manufacturer-provided digest, and use OnePlus’s prescribed signature verification. **Before the first write or wipe**, confirm the exact device, accepted firmware and loss of data with the owner. Follow OnePlus’s partition and slot sequence. It must cover the affected partitions and all bootable slots the vendor requires, including the stock boot chain and verified-boot metadata. Follow its instructions for any pending OTA or slot transition. An ordinary OTA or a stock `init_boot` on one slot alone cannot establish this coverage.

An XDA reply recommends installing the full OTA twice for a French CPH2747 running OxygenOS 16. The owner’s last post asks which package to use. A Canadian owner separately reports a successful relock after converting to global firmware. System Hacks shows a China-to-global conversion, the relock prompt and setup, then identifies the installed build as CPH2747_16.0.1.303. A successful stock relock of an Indian CPH2745 from Android 17 C.75 to C.80 remains unverified. [XDA advice](https://xdaforums.com/posts/90566499/) · [French owner’s follow-up](https://xdaforums.com/posts/90567414/) · [Canadian owner’s result](https://xdaforums.com/posts/90642117/) · [Video relock and setup](https://www.youtube.com/watch?v=QhOwUawyWQw&t=640s) · [Video build screen](https://www.youtube.com/watch?v=QhOwUawyWQw&t=677s)

Complete the vendor-prescribed stock boot or pre-lock validation. A normal boot while unlocked provides a functional check, but the unlocked bootloader can boot modified software. If firmware coverage, rollback compatibility or manufacturer-key validation is unresolved, **leave the bootloader unlocked and use OnePlus service**. [Android Verified Boot device states](https://source.android.com/docs/security/features/verifiedboot/device-state) · [OnePlus repair service](https://www.oneplus.in/support/repair)

## 3. Relock, erase and verify

iFixDroid explains the stock-firmware prerequisite at 0:38 and leaves the lock command unexecuted on his rooted phone at 4:16. [Watch the relock explanation](https://www.youtube.com/watch?v=awbeXSDwivo)

Proceed only when OnePlus confirms that the restored manufacturer-signed firmware is suitable for a locked boot on this unit. **Confirm relocking and its data wipe with the owner** before accepting the device prompt. Use OnePlus’s approved relock and reset sequence. The required factory-data wipe clears KernelSU Manager, module data under `/data/adb`, PixelMask and remaining local settings. If OnePlus’s relock already performs that wipe, verify the result before considering a separate reset.

| Completion check | Evidence |
| --- | --- |
| Official firmware boots normally | The installed build matches the OnePlus-approved target and reaches Android setup without a Verified Boot warning. |
| Bootloader is locked under the manufacturer’s key | Verify the lock and Verified Boot result using the OnePlus procedure or service diagnostic. When Android properties are already accessible, the expected values are `ro.boot.flash.locked=1`, `ro.boot.vbmeta.device_state=locked` and `ro.boot.verifiedbootstate=green`. [Android boot-flow indicators](https://source.android.com/docs/security/features/verifiedboot/boot-flow) |
| Factory data has been erased | The authorized wipe completed, the device reaches initial setup and the previous user profile is gone. For handoff, leave it at setup. |
| Current state is documented | Record the verified model, firmware, wipe, bootloader and Verified Boot results. Update the [installed inventory](../README.md#installed-on-your-oneplus-15) and [current verification](../02-updates/current/verification.json). Record any check that only OnePlus service could perform. |

Before replacing the rooted inventory with the verified factory state, carry its component names, channels, sources and desired settings into the clean-install guide as the target for a later rebuild. Leave the phone at the first welcome screen until the owner is ready to root again. After verification and any recovery, remove only this factory run directory. These checks establish the phone’s present software and boot state. Prior unlock history, tamper records and third-party attestation results are separate questions.
