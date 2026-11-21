# Restore the OnePlus 15 to factory stock

The intended result is an Indian OnePlus 15 CPH2745 running official OnePlus firmware, with user data erased, KernelSU and its modules removed, the bootloader locked and Android Verified Boot enforcing the manufacturer’s key. The latest pre-restore check found slot B still running the known KernelSU patch, with an unlocked bootloader and orange Verified Boot. [Current phone state](../02-updates/current/verification.json)

## 1. Identify the phone and obtain a supported restore route

Back up wanted data off the phone and verify you can access it there. A factory wipe removes local accounts, apps, module settings, Tailscale’s private device identity and USB debugging authorization. Verify access to the Google account and screen-lock credentials that setup may require.

If Android boots and USB debugging is authorized, use current Android Platform Tools on the Mac. From the device list, set `PHONE_SERIAL` to the intended phone’s serial. These commands only read its present state. [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools)

```sh
adb devices -l
phone() { adb -s "${PHONE_SERIAL:?Set PHONE_SERIAL first}" "$@"; }
for prop in ro.product.model ro.product.device ro.build.fingerprint ro.boot.slot_suffix \
            ro.boot.flash.locked ro.boot.vbmeta.device_state ro.boot.verifiedbootstate; do
  printf '%s ' "$prop"
  phone shell getprop "$prop"
done
```

Give OnePlus India the original hardware model and sales region, current fingerprint and slot, and the known modifications. The recorded device is **CPH2745 / CPH2745IN (OP611FL1)**. It was rooted by patching `init_boot_b`. Request the **full manufacturer-signed factory restoration and relock procedure** approved for this hardware and its current anti-rollback level. OnePlus may specify a newer supported build or require service. [OnePlus India software support](https://www.oneplus.in/support/softwareupgrade) · [OnePlus India repair service](https://www.oneplus.in/support/repair)

### Get the latest full Indian firmware

1. On the phone, open **Settings → About device**, tap the software-version card or **New version available**, and read the offered build. Compare it with the OnePlus release announcement for **OnePlus 15, IN, CPH2745**. The phone may offer a smaller update from its current build. [OnePlus Community releases](https://community.oneplus.com/) · [OnePlus India software support](https://www.oneplus.in/support/softwareupgrade)
2. Open the firmware index and select **OnePlus 15 → India → Full**. Its live metadata gives the current `version_number`, `md5sum` and manufacturer `download_url`. Compare the version with the release you intend to install. For recovery of an existing boot partition, use the exact installed build’s archive entry instead. [Firmware index](https://oosdownloader-gui.fly.dev/) · [Latest full India metadata](https://oosdownloader-gui.fly.dev/api/oneplus/oneplus_15/IN/full/info)
3. Use the index’s download-link resolver to generate a fresh OnePlus/OPlus CDN URL. For programmatic use, send the official `download_url` URL-encoded as the `url` query parameter to `https://oosdownloader-gui.fly.dev/api/ota/resolve-url`. Read `url` from the JSON response. The successful India downloads came from the manufacturer’s HTTPS host `gauss-compota-c-in.allawnofs.com`. Treat the index as a locator. Keep signed URLs out of documents and logs. A JSON response with error 2306, an HTML page or HTTP 403 means you have yet to obtain the ZIP.
4. Download the **full ZIP** into a private temporary folder on the Mac. Verify its size, transport checksum, ZIP integrity and embedded `META-INF/com/android/metadata`. Match `pre-device` to `OP611FL1` and match the full target fingerprint to the India `CPH2745IN` release. Check the payload’s partition coverage and use OnePlus’s signature verification before installation. A full OTA can supply stock partition images. Relocking also requires the checks below.

**Verified for the current C.80 release.** The full ZIP passed its transport checksum, ZIP integrity checks and Android whole-file signature verification against the connected phone’s trusted OTA signing key. Its payload contains complete images for 65 partitions with no dependency on old partition data, including `boot`, `init_boot` and `vbmeta`. The manifest also sets `partial_update=true`. This release does not claim to cover every partition in a factory service flash. The owner’s requested end state still requires a supported restoration and verified locked boot. [Current verification](../02-updates/current/verification.json) · [Android OTA manifest definition](https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto#435)

## 2. Restore the complete firmware

If OnePlus supplies an owner-executable procedure, obtain the package and tools directly from OnePlus using its official links. When a download is required, stage it temporarily on the Mac.

```sh
umask 077
RESTORE_DIR=$(mktemp -d /private/tmp/oneplus-stock.XXXXXXXX)
```

Check the package identity and any manufacturer-provided digest, and use OnePlus’s prescribed signature verification. **Before the first write or wipe**, confirm the exact device, accepted firmware and loss of data with the owner. Follow OnePlus’s partition and slot sequence. It must cover the affected partitions and all bootable slots the vendor requires, including the stock boot chain and verified-boot metadata. Follow its instructions for any pending OTA or slot transition. An ordinary OTA or a stock `init_boot` on one slot alone cannot establish this coverage.

Complete the vendor-prescribed stock boot or pre-lock validation. A normal boot while unlocked provides a functional check, but the unlocked bootloader can boot modified software. If firmware coverage, rollback compatibility or manufacturer-key validation is unresolved, **leave the bootloader unlocked and use OnePlus service**. [Android Verified Boot device states](https://source.android.com/docs/security/features/verifiedboot/device-state) · [OnePlus repair service](https://www.oneplus.in/support/repair)

## 3. Relock, erase and verify

Proceed only when OnePlus confirms that the restored manufacturer-signed firmware is suitable for a locked boot on this unit. **Confirm relocking and its data wipe with the owner** before accepting the device prompt. Use OnePlus’s approved relock and reset sequence. The required factory-data wipe clears KernelSU Manager, module data under `/data/adb`, PixelMask and remaining local settings. If OnePlus’s relock already performs that wipe, verify the result before considering a separate reset.

| Completion check | Evidence |
| --- | --- |
| Official firmware boots normally | The installed build matches the OnePlus-approved target and reaches Android setup without a Verified Boot warning. |
| Bootloader is locked under the manufacturer’s key | Verify the lock and Verified Boot result using the OnePlus procedure or service diagnostic. When Android properties are already accessible, the expected values are `ro.boot.flash.locked=1`, `ro.boot.vbmeta.device_state=locked` and `ro.boot.verifiedbootstate=green`. [Android boot-flow indicators](https://source.android.com/docs/security/features/verifiedboot/boot-flow) |
| Factory data has been erased | The authorized wipe completed, the device reaches initial setup and the previous user profile is gone. For handoff, leave it at setup. |
| Current state is documented | Record the verified model, firmware, wipe, bootloader and Verified Boot results. Update the [installed inventory](../README.md#installed-on-your-oneplus-15) and [current verification](../02-updates/current/verification.json). Record any check that only OnePlus service could perform. |

Before replacing the rooted inventory with the verified factory state, carry its component names, channels, sources and desired settings into the clean-install guide as the target for a later rebuild. Leave the phone at the first welcome screen until the owner is ready to root again. After verification, remove the temporary firmware and tools acquired for this restore. These checks establish the phone’s present software and boot state. Prior unlock history, tamper records and third-party attestation results are separate questions.
