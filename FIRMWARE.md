# FIRMWARE.md — firmware findings and why flashing is the risky one

> ## ⚠️ Do not flash firmware using anything derived from this document.
>
> The firmware **transfer protocol has not been reverse engineered**. What follows
> is analysis of the vendor's tooling and firmware *images*, not a flashing
> implementation. A failed flash has no confirmed recovery path.

---

## 1. The firmware is a public download — PROVEN

`MOJO7.Constant` in `mcz.allinone.devices.dll` contains a `FW_UPDATE_URL` string:

```text
https://mcz_global.gitlab.io/allinone/MMO7+/fwupdate.json
```

It redirects (HTTP 308) to a GitLab Pages host and returns live JSON:

```json
{
  "version": "1.22",
  "exec": "MMO7+_Fw_Updater\\MMO7+_FW_Updater.exe",
  "bin": "MMO7+_Fw_Updater\\MMO7+_FW_v1.22.bin",
  "sha256": "7f69c620a4d3c423f62f6f1fce526ce329d21cbd36a1f2fe94c8cfa9d9539a1d",
  "url": "https://mcz_global.gitlab.io/allinone/MMO7+/MMO7+_Fw_Updater.zip"
}
```

That zip contains five firmware versions, the dedicated updater, and `hidapi.dll`:

| File | Size | sha256 (first 16) |
| --- | --- | --- |
| `MMO7+_FW_v1.16.bin` | 123,644 | `08ccbea3b1963645` |
| `MMO7+_FW_v1.19.bin` | 123,900 | `bc5fc70b4fa3b7fe` |
| `MMO7+_FW_v1.20.bin` | 123,900 | `21bd6dc744795c29` |
| `MMO7+_FW_v1.21.bin` | 124,668 | `cd431e2a4c0c00c2` |
| `MMO7+_FW_v1.22.bin` | 124,668 | `66da4d6da03a2226` |
| `MMO7+_FW_Updater.exe` | 11,214,848 | `c50c1f9f7ed092b3` |
| `hidapi.dll` | 86,016 | `9dd6e0a1e36607d0` |

**Consequence:** you do **not** need to extract firmware from the vendor binary.
Official images are a normal download.

### ⚠️ The published SHA-256 is wrong

The manifest's `sha256` value `7f69c620…` **matches none of the files in the zip** —
not the `.bin`, not the `.exe`, not the `hidapi.dll`. The vendor's own integrity
hash is stale or refers to something else. **Do not enforce it** in any client:
it would reject every genuine update.

Also note the version progression is three releases in ~8 months
(v1.16 → v1.22), so firmware updates are infrequent.

---

## 2. Firmware image format — PROVEN

The image is **raw ARM firmware, not encrypted**:

- Opens with the classic ARM exception vector table:
  `18 f0 9f e5` = `0xE59FF018` = `LDR PC, [PC, #0x18]`
- Contains readable debug strings, e.g. `Rf mode = %x`, `Mouse reset_reason`,
  `flash7E1KL00 SumAddr:`, `Key_CurrBBentStatus`, `flash7E0^`
- Byte entropy ≈ 7.01, all 256 byte values present (dense code, not scrambled —
  the plaintext strings above rule out encryption)

First 16 bytes of v1.22:

```text
45 7b cd c9 23 00 bf 79 42 42 42 42 ff ff cc 6e
```

Target SoC is a **Beken BK3633**, per the leaked PDB path in `hidDriver.dll`
(see `PROTOCOL.md` §7).

### Cross-check that ties the package together

The support package's `Update.exe` embeds a **124,668-byte** resource whose first
16 bytes are byte-identical to `MMO7+_FW_v1.22.bin`. So the standalone installer
and the download carry the same image — the two distribution paths agree.

---

## 3. The updater uses HID only — PROVEN

`MMO7+_FW_Updater.exe`: PE32 i386, native (not .NET), 11 sections. Imports:

```text
DLL Name: hidapi.dll
  hid_init          hid_enumerate      hid_open_path
  hid_read          hid_write          hid_send_feature_report
  hid_free_enumeration                 hid_version

WinUSB / SetupAPI / libusb matches: 0
```

Three conclusions:

1. **Flashing rides the HID interface** — the same device, same VID/PID. There is
   no DFU re-enumeration to a vendor-specific interface.
2. Therefore the browser API for this is **WebHID, not WebUSB**. WebUSB cannot
   claim an interface already owned by a kernel driver, and on Linux `usbhid`
   owns the HID interface.
3. Note the absence of `hid_get_feature_report` — responses come back as **input
   reports** (`hid_read`). So the flash protocol is an event-driven handshake, not
   simple request/response.

Consistent with "no re-enumeration": the binary contains **zero** VID/PID
immediates and uses `hid_open_path` rather than `hid_open(vid, pid)`.

### Flash protocol hints from strings

```text
flash_erase:            FLASHADDR_B            flash_mid=%x
FLASH_RD_Global         FLASH_RD_Profile        FLASH_WR_Profile
flash_write_some_data   flash_wp_256k:          flash_wp_ALL:
EODFU                   DfUD
app part upgrade        "app and stack upgrade"
```

`EODFU` / `DfUD` are **in-band protocol tokens**, not a USB DFU class device.
"app and stack upgrade" indicates a **two-partition** write (app + Beken stack),
which widens the partial-failure window.

`FLASH_RD_Profile` / `FLASH_WR_Profile` also reveal that the flasher can read and
write the profile region directly — which is how the bundled *Factory Reset Tool*
likely works.

---

## 4. Why flashing is categorically riskier than config

| Aspect | Config writes | Firmware flashing |
| --- | --- | --- |
| Region touched | profile / macro data | **code + stack partitions** |
| Failure mode | wrong buttons or DPI — device still enumerates | no valid code — **may not enumerate at all** |
| Recovery | rewrite the profile, or the vendor reset tool | **vendor tool may be unable to reach it** — it talks to running firmware over HID |
| Tooling | safe to experiment with reads first | one shot; erasing is not reversible |

The `EODFU` token implies a ROM DFU recovery mode exists. Whether it is
reachable when no valid firmware is running — and whether it needs a hardware
condition — is **unverified**. That is the crux: a flash attempt bets the mouse
on that.

---

## 5. If you want Windows-free flashing anyway

Ordered by cost-benefit:

| Approach | Windows needed | Effort | Risk |
| --- | --- | --- | --- |
| **Try Wine first** | none | ~none | low–moderate |
| Wine to *capture*, then port | none | moderate | low |
| Capture once on Windows, then port | once | moderate | low |
| Static RE of the 11 MB updater | never | high | moderate |
| Keep a Windows VM per update | per release | trivial | lowest |

**Try Wine before anything else.** The updater is a native i386 PE whose only
device API is `hidapi.dll`, and Wine implements `hid.dll` on top of Linux
`hidraw` — so the vendor's own updater may simply run. See `PLATFORM.md` §4.

Practical notes:

- Validate Wine against the **config** tool first (harmless) before trusting it
  with firmware.
- You don't need a Windows *seat* to get a capture: run the updater under Wine on
  Linux with `usbmon` capturing the host bus.
- A byte-for-byte replay will **not** work — flashing is a handshake (write →
  await input-report ack → next chunk). You must implement the state machine.
- **An "update available" checker needs none of this.** Because the manifest is
  public and the read-only tool already returns `QueryFWVersion`, a browser page
  can tell a Linux user that firmware 1.22 exists without writing anything. That
  is the safe, useful, Windows-free 90%.

---

## 6. Recommendation

1. Ship the **read-only** diagnostic first (`OPEN-QUESTIONS.md`).
2. Add an **update checker** — public manifest + `QueryFWVersion`. Zero risk.
3. Config writes only after the read path is confirmed on real hardware.
4. Firmware flashing last, if ever, and as a desktop tool with an explicit backup
   step rather than something a stranger clicks in a browser tab.
