# METHOD.md — provenance and how to reproduce

So that anyone can verify (or refute) these findings independently.

---

## 1. Inputs

All inputs are **publicly distributed by the vendor**. No binaries or firmware
images are redistributed in this repository — only observations, offsets and byte
layouts, for interoperability purposes.

### Support package

| Property | Value |
| --- | --- |
| Name | `M.M.O. 7+ Software and Support Package (2026-9-14)-B.rar` |
| Size | 80,570,988 bytes |
| URL | `https://www.madcatz.com/FileUploads/SupportFile/M.M.O.%207+%20Software%20and%20Support%20Package.rar` |
| Linked from | `https://www.madcatz.com/En/Support/Downloads` (product id `1048`) |

The download page is JavaScript-driven; the underlying API is:

```text
https://www.madcatz.com/En/Support/GetDownloadsProducts?language=En&productCategoryId=1
https://www.madcatz.com/En/Support/GetProductDownloadsResult?language=En&productId=1048
```

Contents:

```text
M.M.O. 7+ Software and Support Package (2026-9-14)-B/
  MadCatz H.U.D._v1.0.38/
    mcz.allinone.install.msi          66.6 MB
    setup.exe                         PE32, native
  M.M.O. 7+ Firmware Factory Reset Tool/
    MMO 7+ Ver1.22 Upgrade/
      Update.exe                      3,642,368 bytes, PE32 6.00 i386
      hidapi.dll                      86,016 bytes
    *.pdf                             firmware update instructions (EN + CN)
  Quick Start Guide (M.M.O.7+).pdf
  M.M.O. 7+ Troubleshooting guide v3.1.pdf
```

### Firmware

| Property | Value |
| --- | --- |
| Manifest | `https://mcz_global.gitlab.io/allinone/MMO7+/fwupdate.json` (HTTP 308 → GitLab Pages host) |
| Archive | `https://mcz_global.gitlab.io/allinone/MMO7+/MMO7+_Fw_Updater.zip` |
| Contains | `MMO7+_FW_v1.{16,19,20,21,22}.bin`, `MMO7+_FW_Updater.exe` (11,214,848 B), `hidapi.dll` |

### Extracted artifact identities

Because the MSI stores files under hashed stream names, these are the stable
identifiers:

| Artifact | Cab stream name | Size | Identity |
| --- | --- | --- | --- |
| `mcz.allinone.devices.dll` | `_9DF704C76ECCA8A889701889000CE286` | 984,576 | .NET 8, 123 types, 977 methods |
| `hidDriver.dll` | `_7A28ECDFC7882457D2630CF5F278BD15` | 327,168 | md5 `80b6612d9ad7e11a27b5d8fcd1e39ab0` |
| `hidDriver_dongle.dll` | `_B5C7F102FC1D7F1B0E2F7597BC62A594` | 327,168 | **byte-identical to the above** |
| MSI CAB payload | `_7EAB04A64CE45DDA13F074EA801D683A` | 66,620,011 | 497 files, 465 .NET assemblies |

---

## 2. Tooling

Everything used is standard and free:

| Tool | Used for |
| --- | --- |
| `unrar` | extract the support package |
| `7z` (p7zip) | extract the MSI's embedded CAB; list RAR contents; extract firmware zip |
| `objdump` (binutils, with `pei-i386` support) | PE headers, import/export tables, disassembly of the native drivers |
| `file`, `strings`, `xxd`, `sha256sum` | triage and hashing |
| `python3` + `dnfile`, `dncil`, `pefile` | .NET metadata/IL parsing; PE resource enumeration |

`msitools` is **not** required — `7z` extracts the MSI's payload CAB directly.

```bash
python3 -m venv venv && ./venv/bin/pip install dnfile dncil pefile
```

---

## 3. Reproduction

### 3.1 Unpack

```bash
curl -L -o pkg.rar 'https://www.madcatz.com/FileUploads/SupportFile/M.M.O.%207+%20Software%20and%20Support%20Package.rar'
unrar x -o+ pkg.rar

MSI="M.M.O. 7+ Software and Support Package (2026-9-14)-B/MadCatz H.U.D._v1.0.38/mcz.allinone.install.msi"
7z x -o./msi_raw "$MSI"                      # -> hashed MSI streams incl. the CAB payload
7z x -o./cab ./msi_raw/_7EAB04A64CE45DDA13F074EA801D683A   # -> 497 files, hashed names
```

### 3.2 Identify the assemblies

Parse each candidate's `.NET` Assembly table — this is how
`mcz.allinone.devices.dll` was located without needing the MSI File table:

```python
import glob, dnfile
for p in sorted(glob.glob("cab/_*")):
    try:
        pe = dnfile.dnPE(p)
        if pe.net and pe.net.mdtables.Assembly.rows:
            print(pe.net.mdtables.Assembly.rows[0].Name, p)
        pe.close()
    except Exception:
        pass     # not a .NET PE
```

### 3.3 Dump types, methods and fields

`dnfile` exposes tables directly. Watch out for two gotchas:

- `TypeDef.MethodList` / `.FieldList` are lists of `MDTableIndex` — use
  `.row_index` to index into the row tables.
- `Constant.Parent` is a coded index; resolve via `.table` and `.row_index`.

### 3.4 Disassemble the native driver

```bash
D=cab/_B5C7F102FC1D7F1B0E2F7597BC62A594        # hidDriver.dll
objdump -p "$D" | sed -n '/Export Address Table/,/^$/p'
objdump -p "$D" | sed -n '/Ordinal\/Name Pointer/,/^$/p'
objdump -p "$D" | grep -i "DLL Name"           # HID.DLL + KERNEL32 imports
objdump -h "$D"                                # sections

# export RVAs from the table, then (image base 0x10000000):
objdump -d --start-address=0x10003e40 --stop-address=0x10003f60 -M intel "$D"   # SendPRate
objdump -d --start-address=0x10003f60 --stop-address=0x10004070 -M intel "$D"   # SendProfileId
```

> **Ordinal/name pairing gotcha:** the name-pointer table's ordinal base is 1, so
> the address-table index is `ordinal - 1`. Pairing indices naively shifts every
> name by one.

### 3.5 Firmware

```bash
curl -sL 'https://mcz_global.gitlab.io/allinone/MMO7+/fwupdate.json'
curl -sL -o fw.zip 'https://mcz_global.gitlab.io/allinone/MMO7+/MMO7+_Fw_Updater.zip'
unzip fw.zip && sha256sum MMO7+_Fw_Updater/MMO7+_FW_v1.22.bin
```

---

## 4. Verifying individual claims

| Claim | How to check |
| --- | --- |
| Transport is `HidD_GetFeature`/`SetFeature` | `objdump -p hidDriver.dll` → `HID.DLL` imports |
| `Query*` never writes | count `call 0x1000442c` vs `0x10004432` in each `Query*` body |
| VID/PID `0x0738` / `0x0C19` / `0x1C02` | IL constants in `MOJO7.Service.DeviceService.WndHandleSetted`; values 1848, 3097, 7170 |
| Command IDs and length bytes | `mov WORD PTR [ebp-…], 0x????` in each `Send*` function |
| Poll rate is `1000/Hz` | the `0x8 / 0x4 / 0x2 / else→0x1` branch chain at `0x10003e72` |
| Complement-byte framing | `not al` sequences in `SendProfileId` / `SendPRate` |
| Struct and enum names | `dnfile` metadata on `mcz.allinone.devices.dll` |
| No obfuscation | `grep -a` for ConfuserEx / Eazfuscator / SmartAssembly / `.NET Reactor` / Dotfuscator |
| Firmware is raw ARM | first bytes `18 f0 9f e5` = `LDR PC,[PC,#0x18]`; readable strings present |
| Manifest hash mismatch | `sha256sum` every file in the zip vs the manifest's `sha256` |
| WebHID report-size handling | `services/device/hid/hid_connection_linux.cc` — `max_feature_report_size() + 1` |

---

## 5. Limitations of this method

1. **Static only. No hardware.** Every protocol claim is an inference from code
   that is *intended* to produce those bytes. Whether the device accepts them is
   untested. `OPEN-QUESTIONS.md` lists what needs a real mouse.
2. **Two of eight commands were fully disassembled.** Framing is confirmed for
   `SendProfileId` and `SendPRate` only. The larger payloads are unanalysed —
   and those two disagree on complement placement, so nothing should be assumed
   uniform.
3. **Enum values were not recovered.** Names are exact; the numbers are not.
   This is a hard blocker for building packets, and needs either IL analysis or
   differential capture.
4. **Field order is not wire order.** Demonstrated for `RATE` in
   `STRUCTURES.md` §5. Metadata order cannot be trusted for offsets.
5. **Firmware transfer protocol is entirely unknown.** Only the image format,
   the transport (HID), and command *names* from strings were established.
6. Chromium's HID **blocklist** behaviour for a mouse with macro-capable
   collections was not determined; the implementation file was not locatable in
   the Chromium mirror.

---

## 6. Suggested next steps

1. Run the read-only WebHID probe on real hardware → resolves Q1–Q4.
2. Extract the embedded `*_DEFAULT` blob values (field RVA table + `pe.get_data`)
   → known-good default packets to diff against a live read.
3. Disassemble the remaining `Send*` bodies at the RVAs in `PROTOCOL.md` §7
   → per-command offsets and complement placement.
4. Recover enum numeric values from IL, or by differential capture on Windows.
5. Capture one Windows HUD session to cross-check the whole model.
