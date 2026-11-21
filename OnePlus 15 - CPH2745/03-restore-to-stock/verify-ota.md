# Download and verify the Indian full OTA on the Mac

Use the serial-selected `phone` function and private `RESTORE_DIR` from the [restore guide](README.md#get-the-latest-full-indian-firmware). These checks use macOS system tools, Python 3 and the Protobuf compiler (`protoc`). Use the build offered on the phone. If it shows **Up to date**, use its installed display build when that matches the live index. A newer unoffered India release requires an independent match in OnePlus release notes and confirmation of eligibility before installation. Enter the identified build through the last digit or include the region suffix if shown. The command compares it with fresh full-package metadata and derives the local filename from the matching release. [Python for Mac](https://www.python.org/downloads/macos/) · [Protobuf compiler installation](https://protobuf.dev/installation/)

## Obtain the package

The firmware index locates the manufacturer download. The following command resolves a fresh URL, confirms the HTTPS CDN and the size of its ZIP, and downloads it on the Mac. If the file is already in this run directory, it verifies the remote identity and continues with the local file. Leave the full OTA on the Mac.

```sh
printf 'Verified Indian build from the phone or official release notes: '
read -r OFFERED_BUILD
OTA=$(python3 - "$OFFERED_BUILD" "$RESTORE_DIR" "$(phone shell getprop ro.product.name)" <<'PY'
import json, re, shutil, sys
from pathlib import Path
from urllib.parse import urlencode, urlsplit
from urllib.request import Request, urlopen

offered, run, product = sys.argv[1:]
if product != "CPH2745IN":
    sys.exit("The selected phone does not report the Indian CPH2745 product.")
index = "https://oosdownloader-gui.fly.dev/api/"
cdn = "gauss-compota-c-in.allawnofs.com"
with urlopen(index + "oneplus/oneplus_15/IN/full/info", timeout=30) as r:
    info = json.load(r)
build = info["version_number"]
identity = re.fullmatch(r"(CPH2745_[0-9]+(?:\.[0-9]+)+)\(([A-Za-z0-9]+)\)", build)
if not identity or offered.strip() not in (identity[1], build):
    sys.exit("The identified build and India full-package metadata differ. Review before downloading.")
if not re.fullmatch("[0-9a-fA-F]{32}", info["md5sum"]):
    sys.exit("The index does not supply a usable MD5 checksum.")
ota = Path(run) / f"{identity[1]}_{identity[2]}_IN-full.zip"
resolver = index + "ota/resolve-url?" + urlencode({"url": info["download_url"]})
with urlopen(resolver, timeout=30) as r:
    url = json.load(r)["url"]
def official(address):
    parts = urlsplit(address)
    return parts.scheme == "https" and parts.netloc == cdn
if not official(url):
    sys.exit("Confirm the new distribution host with OnePlus before downloading.")
with urlopen(Request(url, headers={"Range": "bytes=0-3"}), timeout=30) as r:
    extent = re.fullmatch(r"bytes 0-3/(\d+)", r.headers.get("Content-Range", ""))
    if not official(r.geturl()) or r.status != 206 or not extent or r.read(4) != b"PK\x03\x04":
        sys.exit("No confirmed ZIP response. Refresh the official link.")
    size = int(extent[1])
print(build, cdn, size, "bytes", "MD5", info["md5sum"], file=sys.stderr)
if not ota.exists():
    partial = ota.with_suffix(".zip.part")
    if partial.exists():
        sys.exit("Partial ZIP already present. Keep it for inspection and start a fresh run.")
    with urlopen(url, timeout=120) as r:
        if not official(r.geturl()) or r.status != 200:
            sys.exit("Unexpected download response.")
        with partial.open("xb") as out:
            shutil.copyfileobj(r, out, length=1024*1024)
    if partial.stat().st_size != size:
        sys.exit("Incomplete download. Review the .part file before retrying.")
    partial.rename(ota)
if ota.stat().st_size != size:
    sys.exit("Local ZIP size differs from the manufacturer response.")
(ota.parent / "expected-md5").write_text(info["md5sum"].lower())
print(ota)
PY
) && printf 'Package on Mac: %s\n' "$OTA"
```

Check the checksum against the index and require the ZIP test to pass. Inspect the identifiers and compare them with the intended Indian release. For the **November 21, 2026 C.80 check**, the identifiers were `OP611FL1`, `CPH2745IN`, `CPH2745_11.C.80_1800_202610290517`, with target fingerprint `OnePlus/CPH2745IN/OP611FL1:17/CP2A.260605.016/B.R4T3.50a3663-29f0eba-29fce1c:user/release-keys`. Its full ZIP is 9,301,191,889 bytes with SHA-256 `c7ebbcf62c64895ec9c49fe0988b6f0b40a682d95b5c8a310457afb67de5d9db`.

```sh
[ "$(md5 -q "$OTA")" = "$(cat "$RESTORE_DIR/expected-md5")" ] &&
  shasum -a 256 "$OTA" &&
  unzip -tq "$OTA" &&
  unzip -p "$OTA" META-INF/com/android/metadata |
    grep -E '^(pre-device|post-build|ota-id|product_name|version_name)='
```

## Check the whole-file signer and payload

Read the OTA trust store from the selected phone. Extract the Android package signature and the payload manifest without copying the multi-gigabyte payload. Run each block after the preceding check succeeds. [AOSP whole-file verification](https://android.googlesource.com/platform/bootable/recovery/+/refs/heads/main/otautil/verifier.cpp) · [AOSP payload format](https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto)

```sh
set -o pipefail
phone pull /system/etc/security/otacerts.zip "$RESTORE_DIR/otacerts.zip" &&
  unzip -p "$RESTORE_DIR/otacerts.zip" '*.x509.pem' > "$RESTORE_DIR/trusted.pem"
python3 - "$OTA" "$RESTORE_DIR" <<'PY'
import struct, sys, zipfile
from pathlib import Path
ota, out = map(Path, sys.argv[1:])
with ota.open('rb') as src:
    src.seek(-6, 2)
    distance, marker, comment = struct.unpack('<HHH', src.read(6))
    if marker != 0xffff or not 6 < distance <= comment:
        sys.exit('Invalid Android OTA footer.')
    src.seek(-22 - comment, 2)
    end = src.read()
    magic = b'PK\x05\x06'
    if end[:4] != magic or magic in end[4:] or int.from_bytes(end[20:22], 'little') != comment:
        sys.exit('Invalid or ambiguous ZIP footer.')
    (out / 'signature.der').write_bytes(end[-distance:-6])
    (out / 'signed-bytes').write_text(str(ota.stat().st_size - comment - 2))
with zipfile.ZipFile(ota) as z, z.open('payload.bin') as payload:
    magic, version, size, _ = struct.unpack('>4sQQI', payload.read(24))
    if magic != b'CrAU' or version != 2 or size > z.getinfo('payload.bin').file_size - 24:
        sys.exit('Invalid or unsupported payload header.')
    manifest = payload.read(size)
    if len(manifest) != size:
        sys.exit('Truncated payload manifest.')
    (out / 'manifest.pb').write_bytes(manifest)
PY
head -c "$(cat "$RESTORE_DIR/signed-bytes")" "$OTA" |
  /usr/bin/openssl cms -verify -binary -inform DER -in "$RESTORE_DIR/signature.der" \
    -content /dev/stdin -nointern -certfile "$RESTORE_DIR/trusted.pem" -noverify \
    -signer "$RESTORE_DIR/signer.pem" -out /dev/null
```

Require `Verification successful`. OpenSSL selects the signer from the phone’s trusted certificates (`-nointern`). With this explicit trust list, `-noverify` skips CA-chain building. OpenSSL still verifies the signed ZIP content. On C.80 the signer matches `local_OTA.x509.pem` from this phone.

```sh
curl -fsSL 'https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto?format=TEXT' |
  /usr/bin/base64 -D > "$RESTORE_DIR/update_metadata.proto" &&
  protoc --proto_path="$RESTORE_DIR" --decode=chromeos_update_engine.DeltaArchiveManifest \
    update_metadata.proto < "$RESTORE_DIR/manifest.pb" > "$RESTORE_DIR/manifest.txt" &&
  grep -c '^partitions {' "$RESTORE_DIR/manifest.txt" &&
  grep -E '^partial_update:|^  partition_name: "(boot|init_boot|vbmeta)"' "$RESTORE_DIR/manifest.txt" &&
  grep '^    type:' "$RESTORE_DIR/manifest.txt" | sort | uniq -c
```

For C.80, expect 65 target partitions, including `boot`, `init_boot` and `vbmeta`. Its operations are 416 `REPLACE`, 100 `REPLACE_BZ`, 5,378 `REPLACE_XZ` and 133 `ZERO`. They carry complete replacement data or define zero-filled blocks and require no previous image. The manifest declares `partial_update: true`. These checks identify what the package delivers. The manufacturer’s required slot coverage, rollback compatibility, AVB acceptance and lock sequence still need to be established for the device. [Continue the factory restore](README.md#2-restore-the-complete-firmware)
