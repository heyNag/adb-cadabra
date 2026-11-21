# Download and verify the Indian full OTA on the Mac

For a restore or standalone firmware check, run the [USB identity commands](README.md#1-identify-the-phone-and-stage-the-firmware) and [create the private `RESTORE_DIR`](README.md#get-the-latest-full-indian-firmware). For a clean rebuild, use the [Mac and phone setup](../01-root-and-reinstall/README.md#0-set-up-a-fresh-mac-and-phone) and [USB identity checks](../01-root-and-reinstall/README.md#1-check-the-phone-and-manager), then set `RESTORE_DIR=$DOWNLOAD_DIR`. Keep that Terminal session for the serial-selected `phone` function and all package checks. Have Python 3 and the Protobuf compiler (`protoc`) available. [Python for Mac](https://www.python.org/downloads/macos/) · [Protobuf compiler installation](https://protobuf.dev/installation/)

| What do you need the ZIP for? | Start here |
|---|---|
| Extract a stock image for a phone running India C.80 or C.75 | Select the exact installed [C.80 source](#obtain-the-matching-c80-source-archive) or [C.75 source](#obtain-the-matching-c75-source-archive). Both work after the live index advances. |
| Install an offered India update | Select the [destination from the live index](#obtain-the-package). For the historical C.75 → C.80 restore, authenticate the C.75 source first, then obtain the offered destination separately. |

## Obtain the package

Read the build offered in **Settings → About device → Software update**. If Android reports **Up to date**, the installed build can be used to obtain its own image when it matches the index. For a newer unoffered release, independently match OnePlus’s India release notes and confirm eligibility before installation. Enter the display build through its last digit, including the region suffix if shown.

```sh
OTA_LOOKUP=index
OTA_DIR=${RESTORE_DIR:?Create the private run directory first}
OTA_ENTRY=
printf 'Enter the identified India display build: '
read -r OTA_BUILD
```

Continue at [Download and check the selected ZIP](#download-and-check-the-selected-zip).

## Obtain the matching C.80 source archive

Use this branch for a phone running India `CPH2745_17.0.1.201(EX01)`, including when a later version appears in Software update or in the index. Open the C.80 archive entry. Under **下载地址** (downloads), copy the address of **官方链接** (official link). The directory identifies the release and the link points to the manufacturer's download-check service. [India C.80 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.1.201%28EX01%29%20C.80/) · [Recorded C.80 identity and SHA-256](../02-updates/current/verification.json)

```sh
OTA_LOOKUP=archive
OTA_BUILD='CPH2745_17.0.1.201(EX01)'
OTA_DIR=${RESTORE_DIR:?Create the private run directory first}
printf 'Paste the C.80 官方链接 address: '
read -r OTA_ENTRY
```

Continue at [Download and check the selected ZIP](#download-and-check-the-selected-zip), then authenticate it with the phone’s OTA signing keys.

## Obtain the matching C.75 source archive

Use this branch for a phone running India `CPH2745_17.0.0.102(EX01)`. Open the C.75 archive entry and copy **官方链接** under **下载地址**. Keep any previously authenticated destination path for the restore. [India C.75 archive entry](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2015/ColorOS%20CPH2745_17.0.0.102%28EX01%29%20C.75/) · [Recorded C.75 identity and SHA-256](../01-root-and-reinstall/kit/firmware-identification.json)

```sh
DESTINATION_OTA=${OTA-}
DESTINATION_VERIFY_DIR=${VERIFY_DIR-}
C75_DIR="${RESTORE_DIR:?Create the private run directory first}/stock-CPH2745_17.0.0.102_EX01"
C75_OTA="$C75_DIR/CPH2745_17.0.0.102_EX01_IN-full.zip"
unset C75_MANIFEST
OTA_LOOKUP=archive
OTA_BUILD='CPH2745_17.0.0.102(EX01)'
OTA_DIR=$C75_DIR
printf 'Paste the C.75 官方链接 address: '
read -r OTA_ENTRY
```

Continue at [Download and check the selected ZIP](#download-and-check-the-selected-zip). Each package gets a fresh verification directory when you run the signer check.

## Download and check the selected ZIP

Run this after **one** of the three selections above. It resolves a fresh manufacturer URL, checks a four-byte ZIP range and reuses a complete file already at the expected path. The two recorded releases also have known filenames, lengths, SHA-256 digests and OTA IDs. A failed download leaves a partial file for inspection. Use a fresh run or move the failed file aside before retrying.

```sh
unset VERIFY_DIR
OTA=$(OTA_ENTRY="$OTA_ENTRY" python3 - "$OTA_LOOKUP" "$OTA_BUILD" "$OTA_DIR" \
  "$(phone shell getprop ro.product.name)" "$(phone shell getprop ro.build.display.id)" \
  "$(phone shell getprop ro.build.fingerprint)" <<'PY'
import hashlib, json, os, re, shutil, sys, zipfile
from pathlib import Path
from urllib.parse import urlencode, urlsplit
from urllib.request import Request, urlopen

lookup, selected, directory, product, installed, fingerprint = sys.argv[1:]
if product != "CPH2745IN":
    sys.exit("Select the Indian CPH2745 handset.")
api = "https://oosdownloader-gui.fly.dev/api/"
recorded = {
    "CPH2745_17.0.0.102(EX01)": (
        "/component-ota/26/10/18/847936259a7446bbb7d62be1a8c8bff8.zip", 9240351005,
        "645afc94fa6f31ac2bd62a172d95f883edbc5d0819001a97d33406683ed62a75",
        "CPH2745_11.C.75_1750_202610181120"),
    "CPH2745_17.0.1.201(EX01)": (
        "/component-ota/26/10/29/69a6c985451e47f78054a256ac54218d.zip", 9301191889,
        "c7ebbcf62c64895ec9c49fe0988b6f0b40a682d95b5c8a310457afb67de5d9db",
        "CPH2745_11.C.80_1800_202610290517"),
}
md5_expected = None
if lookup == "index":
    with urlopen(api + "oneplus/oneplus_15/IN/full/info", timeout=30) as r:
        info = json.load(r)
    build, entry, md5_expected = info["version_number"], info["download_url"], info["md5sum"]
    if selected.strip() not in (build, build.partition("(")[0]):
        sys.exit("The selected build differs from the live India full-package index.")
    if not re.fullmatch(r"[a-fA-F0-9]{32}", md5_expected):
        sys.exit("Missing or invalid index MD5.")
elif lookup == "archive" and selected in recorded and selected == installed:
    build, entry = selected, os.environ["OTA_ENTRY"].strip()
else:
    sys.exit("Select the source archive matching the installed India build.")
identity = re.fullmatch(r"(CPH2745_[0-9]+(?:\.[0-9]+)+)\(([A-Za-z0-9]+)\)", build)
u = urlsplit(entry)
if not identity or (u.scheme, u.netloc, u.path) != ("https", "component-ota-in.allawnos.com", "/downloadCheck"):
    sys.exit("Expected the India display build and a manufacturer download-check link.")
with urlopen(api + "ota/resolve-url?" + urlencode({"url": entry}), timeout=30) as r:
    url = json.load(r)["url"]
expected = recorded.get(build)
def official(address):
    u = urlsplit(address)
    return (u.scheme == "https" and u.netloc == "gauss-compota-c-in.allawnofs.com"
            and (not expected or u.path.endswith(expected[0])))
if not official(url):
    sys.exit("Unexpected manufacturer host or filename. Check the release source.")
with urlopen(Request(url, headers={"Range": "bytes=0-3"}), timeout=30) as r:
    extent = re.fullmatch(r"bytes 0-3/(\d+)", r.headers.get("Content-Range", ""))
    if not official(r.geturl()) or r.status != 206 or not extent or r.read(4) != b"PK\x03\x04":
        sys.exit("No confirmed ZIP response. Resolve a fresh official link.")
size = int(extent[1])
if expected and size != expected[1]:
    sys.exit("The manufacturer response differs from the recorded package length.")
ota = Path(directory) / f"{identity[1]}_{identity[2]}_IN-full.zip"
ota.parent.mkdir(parents=True, exist_ok=True)
candidate = ota
if not ota.exists():
    candidate = ota.with_suffix(".zip.part")
    with candidate.open("xb") as out, urlopen(url, timeout=120) as r:
        if r.status != 200 or not official(r.geturl()):
            sys.exit("Unexpected download response.")
        shutil.copyfileobj(r, out, length=4*1024*1024)
if candidate.stat().st_size != size:
    sys.exit("Incomplete ZIP. Inspect the file before retrying.")
sha, md5 = hashlib.sha256(), hashlib.md5()
with candidate.open("rb") as src:
    for block in iter(lambda: src.read(4*1024*1024), b""):
        sha.update(block)
        md5.update(block)
if expected and sha.hexdigest() != expected[2]:
    sys.exit("ZIP differs from the recorded whole-file SHA-256.")
if md5_expected and md5.hexdigest() != md5_expected.lower():
    sys.exit("ZIP differs from the index MD5.")
with zipfile.ZipFile(candidate) as archive:
    if archive.testzip() is not None:
        sys.exit("ZIP integrity test failed.")
    metadata = archive.read("META-INF/com/android/metadata").decode().splitlines()
fields = dict(line.split("=", 1) for line in metadata if "=" in line)
required = {"pre-device": "OP611FL1", "product_name": product, "version_name": build}
if expected:
    required["ota-id"] = expected[3]
if build == installed:
    if not fingerprint:
        sys.exit("Cannot read the running build fingerprint.")
    required["post-build"] = fingerprint
if any(fields.get(key) != value for key, value in required.items()):
    sys.exit("ZIP metadata differs from the intended India release or running source.")
if candidate != ota:
    candidate.rename(ota)
print(build, size, "bytes", "SHA-256", sha.hexdigest(), file=sys.stderr)
print("Target fingerprint:", fields["post-build"], file=sys.stderr)
print(ota)
PY
) && printf 'Package on Mac: %s\n' "$OTA"
```

Require the package path and expected target fingerprint. In the same Terminal, run [Check the whole-file signer and payload](#check-the-whole-file-signer-and-payload) and require it to publish `VERIFY_DIR` for this ZIP.

- **C.80 source.** Keep `OTA` set to this authenticated ZIP. Return to [Get the Mac OTA extractor](../01-root-and-reinstall/README.md#get-the-mac-ota-extractor) and continue with the stock-image extraction.
- **C.75 source.** Continue with the C.75 extraction below.
- **Update destination.** Keep `OTA` set to the authenticated destination and return to the [restore walkthrough](README.md#2-restore-the-complete-firmware).

### Extract the verified C.75 stock image

Once the C.75 signature and manifest checks pass, save the manifest path for the active-slot comparison. Get the extractor in the same Terminal before running this block. [Get the Mac OTA extractor](../01-root-and-reinstall/README.md#get-the-mac-ota-extractor)

```sh
[ "$OTA" = "${C75_OTA:?Select C.75 first}" ] &&
  [ -s "${VERIFY_DIR:?Authenticate the C.75 ZIP first}/manifest.txt" ] &&
  C75_MANIFEST="$VERIFY_DIR/manifest.txt" &&
  payload-dumper-go -p init_boot -o "$C75_DIR" "$C75_OTA" &&
  printf '%s  %s\n' 8910c3ce57893273120434866ef29a436771ae931290e83156807153c8936002 "$C75_DIR/init_boot.img" |
  shasum -a 256 -c -
```

Require `init_boot.img: OK`. For a C.75 rebuild, keep `OTA=$C75_OTA` and return to [the clean-install guide](../01-root-and-reinstall/README.md#extract-and-verify-the-stock-image). For the C.75 → C.80 restore, start at [Obtain the package](#obtain-the-package) for the offered destination. If you already authenticated the destination in this Terminal, restore both `OTA=$DESTINATION_OTA` and `VERIFY_DIR=$DESTINATION_VERIFY_DIR` and return to the [restore walkthrough](README.md#2-restore-the-complete-firmware). Keep `C75_OTA`, `C75_DIR` and `C75_MANIFEST`. The walkthrough uses those source files to [compare the active C.75 slot](#compare-the-active-c75-slot-before-removing-root) before removing root.

### Compare the active C.75 slot before removing root

Run this after verifying the C.75 whole-file signature and decoding its manifest, while C.75 is still running on B and the existing Shell root grant works. It asks the phone to hash the signed image length for each target. The table printed on the Mac records the expected and actual digests. This replay checks the two differences observed on this handset by their exact hashes. Investigate a different result while the original slot remains available.

```sh
python3 - "$PHONE_SERIAL" "${C75_MANIFEST:?Authenticate and decode the C.75 archive first}" <<'PY'
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

With `OTA` set to the selected full ZIP and `RESTORE_DIR` set to its working directory, run the block below in the same Terminal. It uses the selected phone’s OTA trust store and a fresh directory on each attempt. For the two documented builds, also require their recorded whole-archive SHA-256 and matching India build metadata. If the phone’s trust store was modified or its origin is unknown, establish a trusted manufacturer key independently before relying on it. [AOSP whole-file verification](https://android.googlesource.com/platform/bootable/recovery/+/refs/heads/main/otautil/verifier.cpp) · [AOSP payload format](https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto)

```sh
verify_ota() {
  unset VERIFY_DIR
  set -o pipefail
  local attempt
  attempt=$(mktemp -d "${RESTORE_DIR:?Set the private working directory}/ota-check-XXXXXXXX") || return 1
  phone pull /system/etc/security/otacerts.zip "$attempt/otacerts.zip" || return 1
  unzip -p "$attempt/otacerts.zip" '*.x509.pem' > "$attempt/trusted.pem" || return 1
  python3 - "${OTA:?Select the matching archive}" "$attempt" <<'PY' || return 1
import struct, sys
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
PY
  head -c "$(cat "$attempt/signed-bytes")" "$OTA" |
    /usr/bin/openssl cms -verify -binary -inform DER -in "$attempt/signature.der" \
      -content /dev/stdin -nointern -certfile "$attempt/trusted.pem" -noverify \
      -signer "$attempt/signer.pem" -out /dev/null || return 1
  python3 - "$OTA" "$attempt" <<'PY' || return 1
import struct, sys, zipfile
from pathlib import Path
ota, out = map(Path, sys.argv[1:])
with zipfile.ZipFile(ota) as archive, archive.open('payload.bin') as payload:
    magic, version, size, sigsize = struct.unpack('>4sQQI', payload.read(24))
    if magic != b'CrAU' or version != 2 or not 0 < size <= archive.getinfo('payload.bin').file_size - 24 - sigsize:
        sys.exit('Invalid or unsupported payload header.')
    manifest = payload.read(size)
    if len(manifest) != size:
        sys.exit('Truncated payload manifest.')
    (out / 'manifest.pb').write_bytes(manifest)
PY
  curl -fsSL 'https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/update_metadata.proto?format=TEXT' |
    /usr/bin/base64 -D > "$attempt/update_metadata.proto" || return 1
  protoc --proto_path="$attempt" --decode=chromeos_update_engine.DeltaArchiveManifest \
    update_metadata.proto < "$attempt/manifest.pb" > "$attempt/manifest.txt" || return 1
  grep -c '^partitions {' "$attempt/manifest.txt" || return 1
  grep -E '^partial_update:|^  partition_name: "(boot|init_boot|vbmeta)"' "$attempt/manifest.txt" || return 1
  grep '^    type:' "$attempt/manifest.txt" | sort | uniq -c || return 1
  VERIFY_DIR=$attempt
  printf 'Authenticated archive and decoded manifest: %s\n' "$VERIFY_DIR/manifest.txt"
}
verify_ota
```

Require both the OpenSSL success message and the final path printed above. The command selects the signer from the explicitly supplied phone certificates (`-nointern`). Here `-noverify` skips CA-chain building; the CMS check still verifies the signed ZIP contents. C.75 and C.80 matched the handset’s trusted `local_OTA.x509.pem`. On failure, resolve the failing step and run the entire block again with the same selected archive. Only a successful run sets `VERIFY_DIR`.

For C.80, expect 65 target partitions, including `boot`, `init_boot` and `vbmeta`. Its operations are 416 `REPLACE`, 100 `REPLACE_BZ`, 5,378 `REPLACE_XZ` and 133 `ZERO`. They carry complete replacement data or define zero-filled blocks and require no previous image. The manifest declares `partial_update: true`. For another release inspect its own partition names, sizes and operation types. A source-dependent operation (for example `SOURCE_COPY` or `SOURCE_BSDIFF`) requires the corresponding base image. Check that the required partitions are present and that the operations supply their data. Next, choose the handoff for this archive:

- **C.75 source image** — [extract the verified C.75 stock image](#extract-the-verified-c75-stock-image).
- **C.80 source image** — [get the extractor and verify the stock image](../01-root-and-reinstall/README.md#get-the-mac-ota-extractor). For a restore, return to [the C.80 stock-boot procedure](README.md#return-the-rebuilt-c80-slot-a-to-stock-boot) after its image reports `OK`.
- **Verified destination** — return to the [restore branch for the running build](README.md#1-identify-the-phone-and-stage-the-firmware) and use the target’s own fingerprint, boot, merge and active-slot checks.

## Calculate the AVB digest

After authenticating the ZIP for the build you are checking, calculate its expected Android Verified Boot (AVB) digest on the Mac. This is the digest of the chained verification metadata. The ZIP checksum and the checksum of `vbmeta.img` alone are different measurements. In the same Terminal, set `OTA` to the archive for the boot being checked. Use the C.75 source ZIP for the first unrooted C.75 boot, then the C.80 destination ZIP for the later C.80 boot. [AOSP VBMeta digest](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/README.md#the-vbmeta-digest)

1. Put the upstream macOS build of `payload-dumper-go` on the temporary run's PATH using the extractor instructions. The tested version was 2.1.0. [Get the extractor](../01-root-and-reinstall/README.md#get-the-mac-ota-extractor) · [Upstream releases](https://github.com/ssut/payload-dumper-go/releases)
2. Extract the six images used by the C.80 chain to one fresh directory. Download AOSP's `avbtool.py` there and calculate the aggregate digest. If the extractor reports a missing partition, or avbtool reports a missing chained image, resolve the chain for that firmware before proceeding. [AOSP avbtool source](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/avbtool.py)

```sh
set -o pipefail
AVB_DIR=$(mktemp -d "${RESTORE_DIR:?Create the private run directory first}/avb-XXXXXXXX") &&
  payload-dumper-go -p vbmeta,boot,dtbo,recovery,vbmeta_system,vbmeta_vendor \
    -o "$AVB_DIR" "${OTA:?Select the authenticated ZIP for this boot}" &&
  curl -fsSL 'https://android.googlesource.com/platform/external/avb/+/refs/heads/main/avbtool.py?format=TEXT' |
    /usr/bin/base64 -D > "$AVB_DIR/avbtool.py" &&
  python3 "$AVB_DIR/avbtool.py" version &&
  EXPECTED_AVB=$(python3 "$AVB_DIR/avbtool.py" calculate_vbmeta_digest \
    --hash_algorithm sha256 --image "$AVB_DIR/vbmeta.img") &&
  printf 'Expected stock AVB digest: %s\n' "$EXPECTED_AVB"
```

3. Once the phone has booted that build, compare its live digest with the Mac result. Require a 64-character lowercase hexadecimal value from the tool and an exact match. Repeat after the wipe and USB reauthorization if the owner continues past the welcome screen.

```sh
LIVE_AVB=$(phone shell getprop ro.boot.vbmeta.digest) &&
  printf '%s\n' "${EXPECTED_AVB:?Calculate the digest for this build first}" | grep -Eq '^[0-9a-f]{64}$' &&
  [ "$LIVE_AVB" = "$EXPECTED_AVB" ] &&
  printf 'Phone and signed firmware match: %s\n' "$LIVE_AVB"
```

On November 21, 2026, avbtool 1.3.0 calculated `b3a36735edddc817c5e7c14a05f2bda6935ab3c7d92b56632251ccb78f65b2c3` for the Indian C.80 package. Android returned that value on slot A both before and after locking. The earlier C.75 stock boot test on slot B produced `4f37f0cf8d94f052f2096c45c3e08c03e12b010d1c1ca7918b7a1bd45b4ef1b9` using that build's separate signed archive. Recompute the expected value for each release. Use the restore guide to check the update merge, bootloader and slot flags before asking for the lock screen. [Tested avbtool revision](https://android.googlesource.com/platform/external/avb/+/761178607206f4cb2af79ed9eec52d8cbd814adb/avbtool.py) · [Return to the restore procedure](README.md#3-relock-erase-and-verify)
