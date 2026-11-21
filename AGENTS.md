# Start here

Open the [OnePlus 15 device overview](OnePlus%2015%20-%20CPH2745/README.md) first. It identifies the Indian CPH2745, lists the last verified firmware, components, release channels and sources, and maps the folders. The repository contains instructions and text verification records.

| Owner’s task | Where to start |
|---|---|
| Root a stock phone or wipe and rebuild the rooted setup | [01 · Clean install](OnePlus%2015%20-%20CPH2745/01-root-and-reinstall/README.md). Use its **Redo on an already-rooted phone** branch when applicable. |
| Check for updates or update installed modules in place | [02 · Update installed components](OnePlus%2015%20-%20CPH2745/02-updates/README.md). Inventory the running components first. A request to check versions ends with the comparison. |
| Remove root or return to a locked factory setup | [03 · Restore to stock](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md). Identify the running build and slot, then take the matching branch. The C.75-to-C.80 case documents the observed wipe, physical lock confirmation and locked boot. |
| Identify or verify a firmware download | [Mac OTA verification](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/verify-ota.md). Match an existing installation to its exact India archive. For an upgrade, identify the offered destination separately. |

## Using the guides

- For documentation work, start from the [verification record](OnePlus%2015%20-%20CPH2745/02-updates/current/verification.json). The top-level `device` and `current_state` describe the last verified handset. `clean_rebuild` identifies the inputs and results of that rebuild. `rooted_rebuild_baseline` and `factory_restore_check` describe earlier states.
- For device work, read the current model, region, build fingerprint, slot and lock state using the workflow’s USB checks. Match the same serial after each change of mode. Slot letters and hashes in a historical run apply to its recorded build and images. Follow the owner’s requested scope and use the owner handoff in the guide for a wipe, unlock or relock.
- Use each component’s **Latest on this channel** link for ordinary rebuilds and updates. The version-specific sources and hashes identify the exact releases used in the recorded run.
- Keep firmware, images, installers, exported settings and recovery material in the workflow’s [temporary Mac run directory](README.md#temporary-mac-working-folder). Commit only the instructions and relevant text evidence. Keep USB serials, credentials and signed download query strings out of public records.
- After device work, update the device overview and current verification record from measured results. Record a documentary or host-only check as such. When a new reader encounters a missing step, fix it in the workflow that owns the procedure and link back to it.
