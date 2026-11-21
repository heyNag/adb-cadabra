# Update installed OnePlus 15 components

Update each component already on the phone to the latest compatible release on its installed channel. The live inventory determines the scope. Skip absent components. If the phone is stock, report that the rooted setup is absent and finish the update check. To install for the first time on a clean slate, use the [clean-slate installation guide](../01-root-and-reinstall/README.md).

Use the **Latest on this channel** links in the [installed inventory](../README.md#installed-on-your-oneplus-15). The [current verification](current/verification.json) provides a baseline. Keep Google Photos closed, leave automatic backup off, and preserve Tailscale’s sign-in and preferences.

## 1. Read the phone

On the Mac, install Android SDK Platform-Tools for `adb` and have Python 3 available for the JSON checks. For APK updates, Android SDK Build-Tools supplies `apksigner`. The other Mac commands below use system tools. [Android Platform-Tools](https://developer.android.com/tools/releases/platform-tools) · [Python for macOS](https://www.python.org/downloads/macos/) · [Android Build-Tools](https://developer.android.com/tools/releases/build-tools)

Run `adb devices -l` and set `PHONE_SERIAL` to the intended phone’s serial. These helpers target that phone. Pass only the fixed commands shown to `rootcmd`.

```sh
phone() { adb -s "${PHONE_SERIAL:?Set PHONE_SERIAL first}" "$@"; }
rootcmd() { phone shell "su -c '$1'"; }
vector() { rootcmd "/data/adb/modules/zygisk_vector/cli --json $*"; }
tsctl() { rootcmd "sh /data/adb/modules/kernelsu-tailscaled/control.sh $1"; }

phone shell getprop ro.product.model
phone shell getprop ro.build.fingerprint
phone shell getprop ro.boot.slot_suffix
phone shell uname -r
phone shell getprop ro.product.cpu.abi
phone shell cat /proc/sys/kernel/random/boot_id
phone shell getenforce
phone shell pm path me.weishu.kernelsu
phone shell pm path com.kinginu.pixelmask
```

For an established KernelSU setup, confirm **Working / LKM** and check the already-authorized Shell root grant. A denied Shell grant alone leaves root status uncertain. Once root is confirmed, inventory KernelSU modules and their enabled or pending-removal states.

```sh
rootcmd id
rootcmd '/data/adb/ksud module list'
```

Run the following checks only for components present in that inventory or installed Android packages. Check the manager APK and LKM versions separately. Run the PixelMask scope query only when both Vector and PixelMask are present.

```sh
phone shell dumpsys package me.weishu.kernelsu | grep -E 'version(Name|Code)='
phone shell dumpsys package com.kinginu.pixelmask | grep -E 'version(Name|Code)='
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

Keep the firmware, active slot, boot ID, enabled modules and applicable configuration readbacks for comparison after the update.

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

Check Android, kernel, ABI and dependency requirements. A version check needs only the release metadata and phone inventory. Record each result and check time in the root inventory and current verification, including any feed or live check that was unavailable. Finish a check-only request after reporting the live counts of installed components checked, already current, and eligible for update. Record any incompatible newer release or unresolved check separately. For an update request, proceed only with installed components that have a newer compatible release on their existing channel.

## 3. Prepare an authorized update

Resolve the selected channel to its exact tag, asset name and version-specific official URL. Download the candidate into a private temporary directory on the Mac (`umask 077`, then `mktemp -d`). For the standalone Tailscale engine, use the integration’s download and verification in step 4. Record the source URL, release tag or package version, size and SHA-256 in the current verification record. Use the exact installed source in the inventory to fetch a matching previous installer into temporary staging only if the rollback plan needs it.

Compare the downloaded size and local SHA-256 (`shasum -a 256 FILE`) with the published metadata. Verify any upstream signed checksum. For a ZIP, use `unzip -tq FILE`, inspect the installer, check its module ID and version code in `module.prop`, and validate its bundled checksum manifest when present. For an APK, run Android SDK `apksigner verify --print-certs FILE`, check the package and version code, and compare its signing identity with the installed APK, accounting for a verified signing-key rotation.

### Temporary Vector settings export

When Vector is present and an update could affect it, its loader or a configured module, export the live settings while the current framework works. Use the Mac shell and the `phone`, `rootcmd` and `vector` helpers from step 1. This creates a private temporary directory on each device. Keep the same shell open through verification or rollback so the cleanup variables remain available.

```sh
export_vector_settings() {
  umask 077
  VECTOR_MAC_DIR=$(mktemp -d /private/tmp/oneplus-vector.XXXXXXXX) || return 1
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

Proceed after the export succeeds, both hashes match and SQLite reports `ok`. Record the framework build and schema together for rollback. The snapshot contains the settings at export time. Preserve any custom bindhosts lists separately when updating bindhosts, and leave Tailscale’s private state on the phone.

## 4. Apply and verify

- **Module ZIPs.** Confirm that the candidate’s module ID already exists in the live inventory, then update it in place. Create an empty staging directory with `phone shell mktemp -d /data/local/tmp/ksu-update.XXXXXX`. Push the ZIP there and compare `phone shell sha256sum REMOTE_ZIP` with the verified local digest. Then run `rootcmd '/data/adb/ksud module install REMOTE_ZIP'`. Replace `REMOTE_ZIP` with the actual absolute path in both commands. Keep the configuration snapshot and any rollback installer in temporary staging until the new build runs.
- **APKs and kernel root.** Confirm the package is present with `phone shell pm path PACKAGE`, replace it in place with the verified APK using `phone install -r FILE`, and re-read its version. A manager APK update alone leaves the running LKM unchanged. For an existing LKM update, verify a stock recovery image for the current firmware and slot before using the official KernelSU release’s supported upgrade procedure. Keep the existing bootloader state and user data. [Extract and verify the matching stock image](../01-root-and-reinstall/README.md#2-select-a-matching-image) · [Official KernelSU instructions](https://kernelsu.org/guide/installation.html)
- **Standalone engine.** With an integration that supports independent updates, run `tsctl engine-check`, then `tsctl engine-prepare`. Confirm the staged version and verified package, run `tsctl engine-apply`, then `tsctl engine-status`. The service restarts during application.
- **Activation.** Reboot only when the selected update needs boot-time activation. For a required reboot, use `phone reboot`. Check `adb devices -l` and `phone shell getprop sys.boot_completed` every five seconds for up to three minutes. Require a new boot ID, boot completion `1`, root and Enforcing SELinux. For a service-only update, check the restarted service directly.
- **Read back.** Repeat the checks from step 1 for the components that were present. For the installed loader and its dependents, check loader status with `rootcmd '/data/adb/modules/zygisksu/bin/zygiskd status'`, Vector’s running build, and PixelMask’s enablement and scope. For bindhosts, compare the addresses printed by `phone shell ping -c 1 -W 2 doubleclick.net` and the same command for `example.com`. The blocked domain should resolve to the configured sink and the control to a public address. Confirm Tailscale’s running engine, backend and preferences against the baseline. If installation times out, inspect staged and running versions before retrying. If rollback is needed, follow the component’s supported recovery procedure using the exact previous release and compatible configuration. Restore a Vector database only after the matching framework build is running.

After verifying the running state, refresh the affected rows in the [five-column inventory](../README.md#installed-on-your-oneplus-15). Record the exact installed release and asset under **Where we installed it from** and keep the channel discovery link under **Latest on this channel**. Refresh [current verification](current/verification.json) with the observed results. Once verification or rollback is complete, remove the task’s temporary installers and downloads from the Mac and phone.

If you ran the Vector export, remove only that session’s temporary database and empty directories after verification or rollback. No database belongs in the workspace.

```sh
rootcmd "rm -f ${VECTOR_PHONE_DB:?Missing phone export path}"
phone shell rmdir "${VECTOR_PHONE_DIR:?Missing phone staging directory}"
rm -f "${VECTOR_MAC_DB:?Missing local export path}"
rmdir "${VECTOR_MAC_DIR:?Missing local staging directory}"
```
