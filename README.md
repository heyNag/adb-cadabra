# ADB Cadabra

A macOS walkthrough and verification record for the **OnePlus 15, Indian model CPH2745**. The device overview brings together the [installed versions, channels and download sources](OnePlus%2015%20-%20CPH2745/README.md#verified-setup-on-the-reference-phone) and the [folder tree](OnePlus%2015%20-%20CPH2745/README.md#folder-tree). Check your own phone’s build, region, active slot and lock state before using a procedure.

| What do you want to do? | Start here |
|---|---|
| Root a stock phone, or erase and rebuild a rooted phone | [Clean install or redo](OnePlus%2015%20-%20CPH2745/01-root-and-reinstall/README.md) |
| Check for or install updates to existing root components | [Read the phone and compare its installed channels](OnePlus%2015%20-%20CPH2745/02-updates/README.md) |
| Remove root or return to a locked, factory-reset phone | [Restore to stock](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md). Choose the branch that matches the running build. |

ADB communicates with Android after you authorize the Mac over USB. Fastboot communicates with the bootloader. This phone has two firmware slots, A and B. A system update can change which slot boots. Each guide tells you when to switch modes, which values to read, and when the owner needs to press a button. Agents should also read the [repository navigation guide](AGENTS.md).

## Temporary Mac working folder

Each install, update or factory restore uses one private run directory under `~/Downloads/adb-cadabra/OnePlus 15 - CPH2745/`. The workflow creates it with a purpose, local date and time, and a random suffix, for example `factory-YYYYMMDD-HHMMSS-XXXXXXXX`. Keep the same Terminal session and run directory for firmware, installers, tools, extracted images, patches, settings exports and verification. Group stock images by their source firmware build. Keep recovery inputs through verification or rollback. For cleanup on this Mac, move the requested temporary items to Trash so they remain recoverable. Limit cleanup to the completed run and the files the owner selected. Git holds the instructions and text verification records.
