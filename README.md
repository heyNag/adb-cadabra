# ADB Cadabra

Personal Android field notes for rooting a phone, updating its installed modules and returning it to factory stock.

| Device | What’s inside |
|---|---|
| [OnePlus 15 · CPH2745](OnePlus%2015%20-%20CPH2745/README.md) | Current firmware and root components, installed channels, official download sources, and the clean-install, update and factory-restore guides. |

## Temporary Mac working folder

Each install, update or factory restore uses one private run directory under `~/Downloads/adb-cadabra/OnePlus 15 - CPH2745/`. The workflow creates it with a purpose, local date and time, and a random suffix, for example `factory-YYYYMMDD-HHMMSS-XXXXXXXX`. Keep the same Terminal session and run directory for firmware, installers, tools, extracted images, patches, settings exports and verification. Group stock images by their source firmware build. Keep recovery inputs through verification or rollback, then remove only that run directory. Git holds the instructions and text verification records.

[Clean install](OnePlus%2015%20-%20CPH2745/01-root-and-reinstall/README.md#0-set-up-a-fresh-mac-and-phone) · [Update](OnePlus%2015%20-%20CPH2745/02-updates/README.md#3-prepare-an-authorized-update) · [Factory restore walkthrough](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md) · [Bootloader checks and lock screen](OnePlus%2015%20-%20CPH2745/03-restore-to-stock/README.md#from-running-android-to-the-bootloader-menu)
