# ADB Cadabra

A macOS walkthrough and verification record for the **OnePlus 15, Indian model CPH2745**. The device overview brings together the [installed versions, channels and download sources](OnePlus%2015%20-%20CPH2745/README.md#verified-setup-on-the-reference-phone) and the [folder tree](OnePlus%2015%20-%20CPH2745/README.md#folder-tree). Check your own phone’s build, region, active slot and lock state before using a procedure.

For every install, reinstall or update, open **Latest on this channel** in the device inventory and select the newest compatible release from that component’s chosen channel. For example, KernelSU uses stable and Vector uses canary (the Release ZIP). **Installed from** and **Version on this phone** document what ran on the reference phone. [Component inventory and live download sources](OnePlus%2015%20-%20CPH2745/README.md#verified-setup-on-the-reference-phone)

## Verified C.80 locked boot

On November 21, 2026, this phone **booted stock C.80 on slot A with the bootloader locked**. We started from rooted C.75 on slot B, restored stock firmware, installed the official Indian C.80 update, and relocked the bootloader. After the wipe, the phone reached **Hello ColorOS 17**. Android then reported a locked bootloader and green Verified Boot. [Verification record](OnePlus%2015%20-%20CPH2745/02-updates/current/verification.json)

We later unlocked and rooted that same C.80 installation on slot A. The only remaining question about restoring this phone to locked C.80 is whether we can **start from its current rooted C.80 setup**, restore stock C.80, relock, and confirm a second locked boot. [Restore guide for the rooted C.80 starting point](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md#return-the-rebuilt-c80-slot-a-to-stock-boot)

## Choose a workflow

| What do you want to do? | Start here |
|---|---|
| Root a stock phone, or erase and rebuild a rooted phone | [Clean install or redo](OnePlus%2015%20-%20CPH2745/01-root-and-reinstall/README.md) |
| Check for or install updates to existing apps and root components | [Read the phone and compare its installed channels](OnePlus%2015%20-%20CPH2745/02-updates/README.md) |
| Remove root or return to a locked, factory-reset phone | [Restore to stock](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md). Choose the branch that matches the running build. |

ADB communicates with Android after you authorize the Mac over USB. Fastboot communicates with the bootloader. This phone has two firmware slots, A and B. A system update can change which slot boots. Each guide tells you when to switch modes, which values to read, and when the owner needs to press a button. Agents should also read the [repository navigation guide](AGENTS.md).

To repeat the **IMD USB-only test**, leave the cable connected while IMD turns USB debugging off. On the reference phone, tapping **Revert to default** restored wired ADB. The guide covers the stable download, permissions, saved settings and how to verify recovery. [IMD setup and USB test](OnePlus%2015%20-%20CPH2745/01-root-and-reinstall/README.md#imd-usb-debugging-setup)

## Temporary Mac working folder

Each install, update or factory restore uses one private run directory under `~/Downloads/adb-cadabra/OnePlus 15 - CPH2745/`. The workflow creates it with a purpose, local date and time, and a random suffix, for example `factory-YYYYMMDD-HHMMSS-XXXXXXXX`. Keep the same Terminal session and run directory for firmware, installers, tools, extracted images, patches, settings exports and verification. Group stock images by their source firmware build. Keep recovery inputs through verification or rollback. For cleanup on this Mac, move the requested temporary items to Trash so they remain recoverable. Limit cleanup to the completed run and the files the owner selected. Git holds the instructions and text verification records.
