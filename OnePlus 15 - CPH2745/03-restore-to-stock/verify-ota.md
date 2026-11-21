# Download and verify the Indian full OTA on the Mac

Use the serial-selected `phone` function and private `RESTORE_DIR` from the [restore guide](README.md#get-the-latest-full-indian-firmware). These checks use macOS system tools, Python 3 and the Protobuf compiler (`protoc`). Use the build offered on the phone. If it shows **Up to date**, use its installed display build when that matches the live index. A newer unoffered India release requires an independent match in OnePlus release notes and confirmation of eligibility before installation. Enter the identified build through the last digit or include the region suffix if shown. The command compares it with fresh full-package metadata and derives the local filename from the matching release. [Python for Mac](https://www.python.org/downloads/macos/) · [Protobuf compiler installation](https://protobuf.dev/installation/)

For the C.75 → C.80 restore, use the [matching C.75 archive](#obtain-the-matching-c75-source-archive) to recover the original stock image. Then use [Obtain the package](#obtain-the-package) below for the offered C.80 destination and jump to [Check the whole-file signer and payload](#check-the-whole-file-signer-and-payload). Each archive has its own verification directory. Return to the [restore walkthrough](README.md#2-restore-the-complete-firmware) for actions on the handset.

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

## Obtain the matching C.75 source archive

Use this branch when the running build is the recorded India C.75 on slot B and the current full-package index offers a newer build. The archive page’s **官方链接** (“official link”) is the manufacturer’s download-check address. The resolver exchanges it for a fresh CDN URL. The basename, byte count and SHA-256 below identify the original C.75 file. Other installed builds require their own matching source package. [C.75 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.0.102%28EX01%29%20C.75/) · [Recorded firmware identity](../01-root-and-reinstall/kit/firmware-identification.json)

Copy the **官方链接** link address. In the same Terminal, keep the destination package and its verification files separate from C.75.

```sh
RUN_ROOT=${RESTORE_DIR:?Create the private restore directory first}
DESTINATION_OTA=${OTA-}
C75_DIR="$RUN_ROOT/stock-CPH2745_17.0.0.102_EX01"
C75_OTA="$C75_DIR/CPH2745_17.0.0.102_EX01_IN-full.zip"
mkdir -p "$C75_DIR"
printf 'Paste the C.75 official link address: '
read -r C75_ENTRY
python3 - "$C75_ENTRY" "$C75_OTA" <<'PY'
import json, shutil, sys
from pathlib import Path
from urllib.parse import urlencode, urlsplit
from urllib.request import urlopen

entry, path = sys.argv[1:]
def matches(url, host, path_suffix):
    u = urlsplit(url)
    return u.scheme == 'https' and u.netloc == host and u.path.endswith(path_suffix)
if not matches(entry, 'component-ota-in.allawnos.com', '/downloadCheck'):
    sys.exit('Select the manufacturer download-check link on the C.75 page.')
resolver = 'https://oosdownloader-gui.fly.dev/api/ota/resolve-url?'
with urlopen(resolver + urlencode({'url': entry}), timeout=30) as r:
    url = json.load(r)['url']
identity = ('gauss-compota-c-in.allawnofs.com',
            '/component-ota/26/10/18/847936259a7446bbb7d62be1a8c8bff8.zip')
if not matches(url, *identity):
    sys.exit('Resolved URL does not identify the recorded India C.75 archive.')
ota = Path(path)
if not ota.exists():
    partial = ota.with_suffix('.zip.part')
    with urlopen(url, timeout=120) as r:
        if r.status != 200 or not matches(r.geturl(), *identity):
            sys.exit('Unexpected manufacturer response. Obtain a fresh link.')
        with partial.open('xb') as out:
            shutil.copyfileobj(r, out, length=4*1024*1024)
    if partial.stat().st_size != 9240351005:
        sys.exit('Incomplete C.75 ZIP. Keep the partial file for inspection.')
    partial.rename(ota)
if ota.stat().st_size != 9240351005:
    sys.exit('Existing C.75 archive has the wrong length.')
PY
```

Select C.75 for verification and check the recorded digest. Run **both** code blocks in [Check the whole-file signer and payload](#check-the-whole-file-signer-and-payload) with these C.75 values. Compare the ZIP metadata to the running C.75 phone. The full target fingerprint must match, together with product `CPH2745IN`, device `OP611FL1` and OTA ID `CPH2745_11.C.75_1750_202610181120`. Require a verified signature before extracting an image.

```sh
RESTORE_DIR=$C75_DIR
OTA=$C75_OTA
printf '%s  %s\n' 645afc94fa6f31ac2bd62a172d95f883edbc5d0819001a97d33406683ed62a75 "$OTA" |
  shasum -a 256 -c - &&
  unzip -p "$OTA" META-INF/com/android/metadata |
    grep -E '^(pre-device|post-build|ota-id|product_name|version_name)='
phone shell getprop ro.build.fingerprint
```

With the upstream extractor available, extract stock C.75 `init_boot` and verify it. During this run we also compared the active slot before flashing. For each of the 65 signed manifest partitions we read its `new_partition_info.size` bytes from offset zero on the corresponding B block device (`/dev/block/by-name/<name>_b` or the dynamic `/dev/block/mapper/<name>_b`) using the existing Shell root grant. We compared the SHA-256 of that extent with `new_partition_info.hash`. For example, `vbmeta_b` has a capacity of 65,536 bytes while its official image measures 12,288 bytes. Sixty-three digests matched. The restore record identifies both differences and the seven matching boot-chain images. [Extractor source and setup](../01-root-and-reinstall/README.md#2-select-a-matching-image) · [Partition measurements](../02-updates/current/verification.json)

```sh
payload-dumper-go -p init_boot -o "$C75_DIR" "$C75_OTA" &&
  printf '%s  %s\n' 8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 "$C75_DIR/init_boot.img" |
  shasum -a 256 -c -
RESTORE_DIR=$RUN_ROOT
OTA=$DESTINATION_OTA
```

The device restore guide uses `"$C75_DIR/init_boot.img"` for the first flash. Obtain and authenticate the destination ZIP separately using [Obtain the package](#obtain-the-package) and its own signature and manifest files in `"$RUN_ROOT"`.

### Compare the active C.75 slot before removing root

Run this after verifying the C.75 whole-file signature and decoding its manifest, while C.75 is still running on B and the existing Shell root grant works. It asks the phone to hash the signed image length for each target. The table printed on the Mac records the expected and actual digests. This replay checks the two differences observed on this handset by their exact hashes. Investigate a different result while the original slot remains available.

```sh
python3 - "$PHONE_SERIAL" "$C75_DIR/manifest.txt" <<'PY'
import ast, re, shlex, subprocess, sys
from pathlib import Path

serial, manifest = sys.argv[1:]
def shell(command):
    return subprocess.check_output(['adb', '-s', serial, 'shell', command], text=True).strip()
if (shell('getprop ro.product.name'), shell('getprop ro.build.display.id'),
        shell('getprop ro.boot.slot_suffix')) != ('CPH2745IN', 'CPH2745_17.0.0.102(EX01)', '_b'):
    sys.exit('This comparison requires the original Indian C.75 boot on slot B.')
parts = re.split(r'^partitions \{', Path(manifest).read_text(), flags=re.M)[1:]
if len(parts) != 65:
    sys.exit('Expected the 65 signed C.75 partition records.')
differences = {}
print('partition\tbytes\texpected_sha256\tdevice_sha256')
for part in parts:
    name = re.search(r'^  partition_name: "([a-z0-9_]+)"$', part, re.M)[1]
    info = re.search(r'^  new_partition_info \{(.*?)^  \}', part, re.M | re.S)[1]
    size = int(re.search(r'^    size: (\d+)$', info, re.M)[1])
    expected = ast.literal_eval('b' + re.search(r'^    hash: (.+)$', info, re.M)[1]).hex()
    if size < 1 or len(expected) != 64:
        sys.exit(f'Invalid manifest length or digest for {name}.')
    command = f'''set -o pipefail
p=/dev/block/by-name/{name}_b
[ -b "$p" ] || p=/dev/block/mapper/{name}_b
[ -b "$p" ] && [ "$(blockdev --getsize64 "$p")" -ge {size} ] || exit 1
head -c {size} "$p" | sha256sum'''
    actual = shell('su -c ' + shlex.quote(command)).split()[0]
    print(name, size, expected, actual, sep='\t', flush=True)
    if actual != expected:
        differences[name] = actual
if differences != {
    'init_boot': 'e0012d4c9765dcbf2275070dd807ee4c1b691543422b9489effb0ae60aeb0a73',
    'splash': 'b30e8403705662848f7d2bc5c3595367fac6613b6b797a7104fa61e99e2628e6',
}:
    sys.exit('The differences from stock differ from the recorded C.75 handset.')
print('63 stock matches and the two recorded C.75 differences.')
PY
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

For C.80, expect 65 target partitions, including `boot`, `init_boot` and `vbmeta`. Its operations are 416 `REPLACE`, 100 `REPLACE_BZ`, 5,378 `REPLACE_XZ` and 133 `ZERO`. They carry complete replacement data or define zero-filled blocks and require no previous image. The manifest declares `partial_update: true`. These checks establish the contents of the archive. Continue with the AVB digest, boot, merge and active-slot checks to establish what actually runs on the handset. [Continue the factory restore](README.md#2-restore-the-complete-firmware)

## Calculate the AVB digest

After the destination ZIP passes the signer, identity and payload checks above, calculate the expected Android Verified Boot (AVB) digest on the Mac. This is the digest of the chained verification metadata. The ZIP checksum and the checksum of `vbmeta.img` alone are different measurements. Use the same Terminal and keep `OTA` set to the authenticated **destination** ZIP. [AOSP VBMeta digest](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/README.md#the-vbmeta-digest)

1. Put the upstream macOS build of `payload-dumper-go` on the temporary run's PATH using the extractor instructions. The tested version was 2.1.0. [Get the extractor](../01-root-and-reinstall/README.md#2-select-a-matching-image) · [Upstream releases](https://github.com/ssut/payload-dumper-go/releases)
2. Extract the six images used by the C.80 chain to one fresh directory. Download AOSP's `avbtool.py` there and calculate the aggregate digest. If the extractor reports a missing partition, or avbtool reports a missing chained image, resolve the chain for that firmware before proceeding. [AOSP avbtool source](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/avbtool.py)

```sh
set -o pipefail
AVB_DIR=$(mktemp -d "${RESTORE_DIR:?Create the private run directory first}/avb-XXXXXXXX") &&
  payload-dumper-go -p vbmeta,boot,dtbo,recovery,vbmeta_system,vbmeta_vendor \
    -o "$AVB_DIR" "${OTA:?Select the authenticated destination ZIP}" &&
  curl -fsSL 'https://android.googlesource.com/platform/external/avb/+/refs/heads/main/avbtool.py?format=TEXT' |
    /usr/bin/base64 -D > "$AVB_DIR/avbtool.py" &&
  python3 "$AVB_DIR/avbtool.py" version &&
  EXPECTED_AVB=$(python3 "$AVB_DIR/avbtool.py" calculate_vbmeta_digest \
    --hash_algorithm sha256 --image "$AVB_DIR/vbmeta.img") &&
  printf 'Expected stock AVB digest: %s\n' "$EXPECTED_AVB"
```

3. Once the phone has booted the destination build, compare its live digest with the Mac result. Require a 64-character lowercase hexadecimal value from the tool and an exact match. Repeat after the wipe and USB reauthorization if the owner continues past the welcome screen.

```sh
LIVE_AVB=$(phone shell getprop ro.boot.vbmeta.digest) &&
  printf '%s\n' "${EXPECTED_AVB:?Calculate the destination digest first}" | grep -Eq '^[0-9a-f]{64}$' &&
  [ "$LIVE_AVB" = "$EXPECTED_AVB" ] &&
  printf 'Phone and signed firmware match: %s\n' "$LIVE_AVB"
```

On November 21, 2026, avbtool 1.3.0 calculated `b3a36735edddc817c5e7c14a05f2bda6935ab3c7d92b56632251ccb78f65b2c3` for the Indian C.80 package. Android returned that value on slot A both before and after locking. The earlier C.75 stock boot test on slot B produced `4f37f0cf8d94f052f2096c45c3e08c03e12b010d1c1ca7918b7a1bd45b4ef1b9` using that build's separate signed archive. Recompute the expected value for each release. Use the restore guide to check the update merge, bootloader and slot flags before asking for the lock screen. [Tested avbtool revision](https://android.googlesource.com/platform/external/avb/+/761178607206f4cb2af79ed9eec52d8cbd814adb/avbtool.py) · [Return to the restore procedure](README.md#3-relock-erase-and-verify)
