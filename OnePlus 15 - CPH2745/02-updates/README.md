# Update installed OnePlus 15 components

Update each component already on the phone to the latest compatible release on its installed channel. The live inventory determines the scope. Skip absent components. On a stock phone, skip the KernelSU checks. WiFiAnalyzer can be inventoried and updated as a regular Android app. To install for the first time on a clean slate, use the [clean-slate installation guide](../01-root-and-reinstall/README.md).

Use the top-level `device` and `current_state` fields for the present phone. `rooted_rebuild_baseline` and `factory_restore_check` record earlier states. Choose **Latest on this channel** for each component present in the live inventory. For a version check, keep Google Photos closed. To verify PixelMask inside Photos after an update, use the [runtime check](../01-root-and-reinstall/README.md#restore-and-verify-the-pixelmask-scope) with the existing scope. Preserve the current backup preference, Tailscale sign-in and settings. [Current verification](current/verification.json) · [Component channels and official downloads](../README.md#verified-setup-on-the-reference-phone)

## 1. Read the phone

On the Mac, install Android SDK Platform-Tools for `adb` and have Python 3 available for the JSON checks. For APK updates, add the Android SDK Build-Tools directory containing `aapt2` and `apksigner` to `PATH`. The other Mac commands below use system tools. [Android Platform-Tools](https://developer.android.com/tools/releases/platform-tools) · [Python for macOS](https://www.python.org/downloads/macos/) · [Android Build-Tools](https://developer.android.com/tools/releases/build-tools)

Connect the phone over USB, unlock the screen and accept its debugging prompt for this Mac. Use the row marked `device` for the intended OnePlus when prompted. Keep these helpers in the same Terminal session. The `rootcmd` helper passes the commands shown here to the phone’s authorized Shell root grant.

```sh
adb devices -l
printf 'Serial of the authorized OnePlus: '
read -r PHONE_SERIAL
phone() { adb -s "${PHONE_SERIAL:?Set PHONE_SERIAL first}" "$@"; }
rootcmd() { phone shell "su -c '$1'"; }
vector() { rootcmd "/data/adb/modules/zygisk_vector/cli --json $*"; }
tsctl() { rootcmd "sh /data/adb/modules/kernelsu-tailscaled/control.sh $1"; }

phone get-state
phone shell getprop ro.product.model
phone shell getprop ro.product.name
phone shell getprop ro.boot.flash.locked
phone shell getprop ro.build.fingerprint
phone shell getprop ro.boot.slot_suffix
phone shell uname -r
phone shell getprop ro.product.cpu.abi
phone shell cat /proc/sys/kernel/random/boot_id
phone shell getenforce
phone shell pm path me.weishu.kernelsu
phone shell pm path com.kinginu.pixelmask
phone shell pm path --user 0 com.vrem.wifianalyzer
```

For an established KernelSU setup, confirm **Working / LKM** and check the already-authorized Shell root grant. A denied Shell grant alone leaves root status uncertain. Once root is confirmed, inventory KernelSU modules and their enabled or pending-removal states.

```sh
rootcmd id
rootcmd '/data/adb/ksud debug version'
rootcmd '/data/adb/ksud module list'
```

On the checked v3.3.1 installation, `ksud debug version` reads the **running kernel-side** KernelSU build, expected `32680`. Compare it independently with the Manager APK version and with the selected upstream release. [KernelSU v3.3.1 command implementation](https://github.com/tiann/KernelSU/blob/v3.3.1/userspace/ksud/src/cli.rs#L707-L710)

Run the following checks only for components present in that inventory or installed Android packages. Check the manager APK and LKM versions separately. Run the PixelMask scope query only when both Vector and PixelMask are present.

```sh
phone shell dumpsys package me.weishu.kernelsu | grep -E 'version(Name|Code)='
phone shell dumpsys package com.kinginu.pixelmask | grep -E 'version(Name|Code)='
phone shell dumpsys package com.vrem.wifianalyzer | grep -E 'version(Name|Code)='
vector status
vector modules ls
vector scope ls com.kinginu.pixelmask
tsctl engine-status
tsctl status-json | python3 -c '
import json, sys
d = json.load(sys.stdin)
result = {k: d.get(k) for k in ("enabled", "mode", "lifecycle", "moduleVersion", "tailscaleVersion")}
result["backend"] = (d.get("tailscale") or {}).get("BackendState")
result["prefs"] = {k: (d.get("prefs") or {}).get(k) for k in ("CorpDNS", "RouteAll", "RunSSH")}
print(json.dumps(result))'
```

Keep the firmware, active slot, boot ID, enabled modules and applicable configuration readbacks for comparison after the update. For an installed Wi-Fi tool, capture its [pre-update version and settings](#update-the-wi-fi-tools) before applying the new release.

## 2. Compare the installed channels

| Component | How to choose a fresh release |
| --- | --- |
| KernelSU manager and root | Open the [latest stable release](https://github.com/tiann/KernelSU/releases/latest). Compare the release APK’s build with the installed manager and compare the running LKM separately. |
| bindhosts | Use the [latest stable release](https://github.com/bindhosts/bindhosts/releases/latest) or its [current module ZIP](https://github.com/bindhosts/bindhosts/releases/latest/download/bindhosts.zip). Record the resolved tag and compare the module version code. Blocklist refreshes have an independent schedule. |
| KernelSU-Tailscaled | Open the [latest stable integration](https://github.com/WayneShao/KernelSU-Tailscaled/releases/latest). Select the ARM64 module ZIP and check which engines it supports. |
| Tailscale engine | Read `TarballsVersion` and `Tarballs.arm64` from [Tailscale’s stable feed](https://pkgs.tailscale.com/stable/?mode=json). Compare with the running engine version. The integration’s updater fetches this Linux ARM64 tarball and verifies Tailscale’s published SHA-256. |
| Zygisk Next | Open [Dr-TSNG’s latest stable release](https://github.com/Dr-TSNG/ZygiskNext/releases/latest) and compare the ZIP’s numeric build. |
| Vector | Open the [canary prerelease list](https://github.com/JingMatrix/Vector/releases?q=prerelease%3Atrue%20canary). Choose the published `canary-N` with the highest numeric `N`, then select its **Release ZIP**. Compare that build with the running Vector CLI status. |
| PixelMask | Open the [latest stable release](https://github.com/kinginu/PixelMask/releases/latest). Compare the APK version code and its required Vector build for Android 17. |
| WiFiAnalyzer | Compare its version on its original channel. [VREM stable APK](https://github.com/VREMSoftwareDevelopment/WiFiAnalyzer/releases/latest) · [Google Play](https://play.google.com/store/apps/details?id=com.vrem.wifianalyzer) · [F-Droid](https://f-droid.org/en/packages/com.vrem.wifianalyzer/) |
| ColorOS Wi-Fi Sense | Compare the installed `coloros_wifi_sense` version code with the [author’s latest stable ZIP](https://github.com/mazemingw/coloros-wifi-sense/releases/latest). |

Check Android, kernel, ABI and dependency requirements. A version check needs only the release metadata and phone inventory. Record each result and check time in the root inventory and current verification, including any feed or live check that was unavailable. Finish a check-only request after reporting the live counts of installed components checked, already current, and eligible for update. Record any incompatible newer release or unresolved check separately. For an update request, proceed only with installed components that have a newer compatible release on their existing channel.

## 3. Prepare an authorized update

Resolve the selected channel to its exact tag, asset name and version-specific official URL. Create one [temporary Mac working folder](../../README.md#temporary-mac-working-folder) and download the candidate there. For the standalone Tailscale engine, use the integration’s download and verification in step 4. Record the source URL, release tag or package version, size and SHA-256 in the current verification record. Use the exact installed source in the inventory to fetch a matching previous installer into temporary staging only if the rollback plan needs it.

```sh
umask 077
RUN_BASE="$HOME/Downloads/adb-cadabra/OnePlus 15 - CPH2745"
mkdir -p "$RUN_BASE" &&
  UPDATE_DIR=$(mktemp -d "$RUN_BASE/update-$(date +%Y%m%d-%H%M%S)-XXXXXXXX") &&
  printf '%s\n' "$UPDATE_DIR"
```

Keep `UPDATE_DIR` for this update’s installers, settings exports and rollback files. Compare the downloaded size and local SHA-256 (`shasum -a 256 FILE`) with the published metadata. Verify any upstream signed checksum. For a ZIP, use `unzip -tq FILE`, inspect the installer, read its ID and version code with `unzip -p FILE module.prop`, and validate its bundled checksum manifest when present. For APKs, use the following identity check for each update. Set `PACKAGE` to the intended installed package (`me.weishu.kernelsu`, `com.kinginu.pixelmask` or `com.vrem.wifianalyzer`) and `APK` to the absolute path of the checksum-verified candidate. The installed base APK supplies the signing certificate for comparison.

```sh
set -o pipefail
base_apk=$(phone shell pm path --user 0 "${PACKAGE:?Select the installed package}" |
  tr -d '\r' | sed -n 's/^package:\(.*\/base\.apk\)$/\1/p')
[ -n "$base_apk" ] &&
  phone pull "$base_apk" "$UPDATE_DIR/$PACKAGE-installed-base.apk" &&
  apksigner verify --print-certs "$UPDATE_DIR/$PACKAGE-installed-base.apk" &&
  apksigner verify --print-certs "${APK:?Select the verified candidate APK}" &&
  aapt2 dump badging "$APK" | sed -n '/^package:/p'
```

Require the intended package and eligible version code in `package:`, both APKs to verify, and matching signer certificate SHA-256 digests. If a new signing key appears, verify the upstream rotation and accepted signing lineage before replacing the APK. Android’s in-place update also enforces package signing compatibility. [Android APK verification](https://developer.android.com/tools/apksigner) · [Android app signing](https://developer.android.com/studio/publish/app-signing)

### Temporary Vector settings export

When Vector is present and an update could affect it, its loader or a configured module, export the live settings while the current framework works. Use the Mac shell and the `phone`, `rootcmd` and `vector` helpers from step 1. This creates a private temporary directory on each device. Keep the same shell open through verification or rollback so the cleanup variables remain available.

```sh
export_vector_settings() {
  umask 077
  VECTOR_MAC_DIR=$(mktemp -d "${UPDATE_DIR:?Create the update run directory first}/vector-XXXXXXXX") || return 1
  VECTOR_MAC_DB="$VECTOR_MAC_DIR/settings.db"
  VECTOR_PHONE_DIR=$(phone shell mktemp -d /data/local/tmp/oneplus-vector.XXXXXXXX) || return 1
  VECTOR_PHONE_DB="$VECTOR_PHONE_DIR/settings.db"

  vector db backup "$VECTOR_PHONE_DB" | python3 -c 'import json, sys; sys.exit(0 if json.load(sys.stdin).get("success") is True else 1)' || return 1
  phone exec-out "su -c 'cat $VECTOR_PHONE_DB'" > "$VECTOR_MAC_DB" || return 1

  local phone_digest mac_digest integrity schema
  phone_digest=$(rootcmd "sha256sum $VECTOR_PHONE_DB") || return 1
  mac_digest=$(shasum -a 256 "$VECTOR_MAC_DB") || return 1
  phone_digest=${phone_digest%% *}
  mac_digest=${mac_digest%% *}
  printf 'Phone SHA-256 %s\nMac SHA-256   %s\n' "$phone_digest" "$mac_digest"
  [ -n "$phone_digest" ] && [ "$phone_digest" = "$mac_digest" ] || return 1

  integrity=$(sqlite3 -readonly "$VECTOR_MAC_DB" 'PRAGMA integrity_check;') || return 1
  [ "$integrity" = ok ] || return 1
  schema=$(sqlite3 -readonly "$VECTOR_MAC_DB" 'PRAGMA user_version;') || return 1
  printf 'SQLite integrity %s, schema %s\n' "$integrity" "$schema"
}
export_vector_settings
```

Proceed after the export succeeds, both hashes match and SQLite reports `ok`. Record the framework build and schema together for rollback. Also save the output of `vector modules ls` and `vector scope ls com.kinginu.pixelmask` when that add-on is installed. The snapshot contains the settings at export time. Preserve any custom bindhosts lists separately when updating bindhosts, and leave Tailscale’s private state on the phone.

## 4. Apply and verify

**Module ZIPs.** Confirm the candidate’s `module.prop` ID matches one in the current inventory. Set `MODULE_ZIP` to the verified archive’s absolute path on the Mac. For each update, stage the archive and compare the two hashes before running the native installer.

```sh
MODULE_STAGE=$(phone shell mktemp -d /data/local/tmp/ksu-update.XXXXXX) &&
  PHONE_ZIP="$MODULE_STAGE/candidate.zip" &&
  phone push "${MODULE_ZIP:?Select the verified module ZIP}" "$PHONE_ZIP" &&
  shasum -a 256 "$MODULE_ZIP" &&
  phone shell sha256sum "$PHONE_ZIP"
```

When the hashes match, run `rootcmd "/data/adb/ksud module install $PHONE_ZIP"` and require installer success. Keep this staging directory’s path through activation. If several ZIPs have updates, record each directory; apply dependency changes in the compatible order documented by the releases (Zygisk Next before its Vector loader module).

**Manager or add-on APK.** After the package and signer checks above, update in place with `phone install -r "$APK"`. Require `Success` and read the installed package version again. If PixelMask requires a newer Vector, verify the new framework has loaded before updating the add-on.

**KernelSU LKM.** An updated Manager and the previous LKM can have different versions until the phone reboots. When the live LKM also needs an update, confirm the phone is already rooted and unlocked, check its current slot, fingerprint and KMI, and obtain the authenticated stock `init_boot.img` for that exact build as the recovery image. In the compatible Manager open the install icon → **Direct install** for the active slot and review the patch/write log before rebooting. The inactive-slot operation is a separate OTA workflow. After boot, require **Working / LKM**, the expected build from `rootcmd '/data/adb/ksud debug version'`, the original slot/fingerprint, and the authorized UID 0. [Matching stock image](../01-root-and-reinstall/README.md#2-select-a-matching-image) · [KernelSU’s Direct install procedure](https://kernelsu.org/guide/installation.html#use-the-manager)

**Standalone Tailscale engine.** Run `tsctl engine-check`, then `tsctl engine-prepare`. Verify the reported staged version is the intended release. Run `tsctl engine-apply`, then `tsctl engine-status`. Verify the running version changed and the staging field is empty. The service restarts during application. Compare `status-json` with the original mode, preferences and sign-in state. The tested baseline is native and enabled with `NeedsLogin`. A logged-in phone should return to its original backend state. [Stable integration engine transaction](https://github.com/WayneShao/KernelSU-Tailscaled/blob/v2.0.5/tailscale/scripts/engine-update.sh)

- **Activation.** Reboot only when the selected update needs boot-time activation. For a required reboot, use `phone reboot`. Check `adb devices -l` and `phone shell getprop sys.boot_completed` every five seconds for up to three minutes. Require a new boot ID, boot completion `1`, root and Enforcing SELinux. For a service-only update, check the restarted service directly.
- **Read back.** Repeat the checks from step 1 for the components that were present. For the installed loader and its dependents, check loader status with `rootcmd '/data/adb/modules/zygisksu/bin/zygiskd status'`, Vector’s running build, and PixelMask’s enablement and scope. For bindhosts, compare the addresses printed by `phone shell ping -c 1 -W 2 doubleclick.net` and the same command for `example.com`. The blocked domain should resolve to the configured sink and the control to a public address. Confirm Tailscale’s running engine, backend and preferences against the baseline. If installation times out, inspect staged and running versions before retrying.

### Update the Wi-Fi tools

- **WiFiAnalyzer.** Retain the app’s install channel. For a VREM GitHub installation set `PACKAGE=com.vrem.wifianalyzer` and `APK` to the verified new file. In section 3 require the new APK’s signer to match **both** the installed base APK and VREM’s published **GitHub** signer before `phone install -r "$APK"`. For a Play or F-Droid installation, update through the same store. Preserve the existing Android permissions and filters, read back the new version and confirm a fresh scan in **Time Graph**. A WiFiAnalyzer-only update needs no reboot or root grant. [VREM signing policy](https://github.com/VREMSoftwareDevelopment/WiFiAnalyzer/blob/main/SECURITY.md) · [App setup and live scan](../01-root-and-reinstall/README.md#install-wifianalyzer)
- **ColorOS Wi-Fi Sense.** On the Indian CPH2745 running official KernelSU, note the installed version and run the daemon check linked below before updating. Record the installed Fluid Cloud variant, its current notification setting, threshold, countdown, recovery margin and optional switches privately. Verify the new ZIP’s digest, module ID and version code. Stage it as in section 4 and select the same variant at the installer’s volume-key prompt. After installer success, reboot, verify the new module version, and repeat the daemon and WebUI checks. Restore the previous WebUI choices if the installer reset them. Keep SSIDs and BSSIDs out of shared records. [Activation and fresh daemon response](../01-root-and-reinstall/README.md#activate-coloros-wi-fi-sense)

### Recover a failed update

- **Module regression or boot loop.** If authorized ADB root is available, inspect `rootcmd '/data/adb/ksud module list'` and disable the affected ID with KernelSU Manager or `/data/adb/ksud module disable ID` in the root shell. Reboot and verify the cause. If Android cannot finish booting after a module update, KernelSU documents repeatedly pressing and releasing Volume Down shortly after the first boot screen to enter its safe mode, which disables modules. Restore the compatible previous release and settings before re-enabling affected modules. If the LKM image write caused the first boot to fail, use the saved matching stock image and recheck the exact original flash target through the clean-install recovery instructions. [KernelSU module rescue](https://kernelsu.org/guide/rescue-from-bootloop.html) · [Recover the original boot image](../01-root-and-reinstall/README.md#recover-a-failed-first-boot)
- **Vector settings.** If the intended scope/settings changed and a rollback is required, bring the matching framework build and its CLI back first. For the snapshot created in this update, recheck both hashes and `PRAGMA integrity_check` as above, compare the saved schema to the running framework, and confirm this is the intended pre-update configuration. Then run `vector db restore "$VECTOR_PHONE_DB"` and require JSON `success: true`. If the temporary phone copy is gone, recreate it in a new private staging directory from the verified Mac copy first. Read back `vector status`, the add-on list and the **entire scope** against the pre-update readback. [Upstream Vector backup and restore implementation](https://github.com/JingMatrix/Vector/blob/canary-3119/daemon/src/main/kotlin/org/matrix/vector/daemon/ipc/CliHandler.kt#L211-L244)
- **Tailscale engine failure.** Re-read `engine-status` and `status-json` before taking further action. The v2.0.5 integration maintains its own recovery journal for a failed apply and rejects engine downgrades. Preserve its state and diagnostics when the previous service does not recover. [Engine recovery implementation](https://github.com/WayneShao/KernelSU-Tailscaled/blob/v2.0.5/tailscale/scripts/engine-update.sh#L85-L145)

After verifying the running state, refresh the [current device inventory](../README.md#verified-setup-on-the-reference-phone) with the installed versions and sources. Refresh [current verification](current/verification.json) with the observed results. After verifying the update or completing rollback, clean up the task-owned phone staging. For each recorded module upload run `phone shell rm -f "$PHONE_ZIP"` and `phone shell rmdir "$MODULE_STAGE"` using the corresponding paths from this session. Complete any Vector export cleanup below before removing this Mac run directory.

If you ran the Vector export, remove only that session’s temporary database and empty directories after verification or rollback. No database belongs in the workspace.

```sh
rootcmd "rm -f ${VECTOR_PHONE_DB:?Missing phone export path}"
phone shell rmdir "${VECTOR_PHONE_DIR:?Missing phone staging directory}"
rm -f "${VECTOR_MAC_DB:?Missing local export path}"
rmdir "${VECTOR_MAC_DIR:?Missing local staging directory}"
```
