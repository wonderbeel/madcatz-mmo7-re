# Mad Catz M.M.O. 7+ — reverse engineering notes

Research toward making the Mad Catz **M.M.O. 7+** wireless gaming mouse
(internal codename **`MOJO7`**) configurable — and eventually flashable — on Linux
and in the browser, without the vendor's Windows-only software.

> ## ⚠️ Status: static analysis only. Not verified against hardware.
>
> Every claim below was derived by reading the vendor's own Windows software.
> **Nobody has run any of this against a real M.M.O. 7+ yet.**
> Treat the protocol sections as *hypotheses with strong evidence*, not as facts.
>
> **Do not attempt to flash firmware based on this document.** See `FIRMWARE.md`.

---

## Why this exists

The M.M.O. 7+ ships with:

- **No Linux software** (the support package is a Windows-only `.rar`)
- **No macOS software**
- **No libratbag/OpenRazer support** — no open-source Linux path at all

The mouse has 21 programmable inputs, 5 onboard profiles, macros, RGB and a DPI
sensor. Without the vendor tool, a Linux user gets none of that. This research
exists to determine whether that gap can be closed, and how much work it is.

**Short answer:** it looks very tractable — the vendor's stack turned out to be
unobfuscated .NET over plain HID feature reports.

---

## What was found (summary)

| Area | Finding | Confidence |
| --- | --- | --- |
| Transport | Plain HID feature reports (`HidD_SetFeature` / `HidD_GetFeature`) | **Proven** (imports) |
| Transport impl | Native `hidDriver.dll` / `hidDriver_dongle.dll` (byte-identical) | **Proven** |
| Linux plumbing | Pure userspace hidapi — no kernel driver, no DKMS | **Proven** |
| USB IDs | VID `0x0738`; PID `0x0C19` (wired), `0x1C02` (dongle) | **Proven** (IL constants) |
| Command set | 8 settings, each with a Send/Query pair | **Proven** (exports) |
| Report IDs | `0x04 0x05 0x06 0x07 0x08 0x09 0x0C 0x18` | **Proven** (disassembly) |
| Framing | `byte0=cmd, byte1=len`, then payload w/ complement bytes | **Partially proven** — 2 of 8 commands disassembled |
| Read path safety | `Query*` calls `HidD_GetFeature` only — **zero writes** | **Proven** (call counts) |
| Onboard data model | `KEYS`, `SENSOR`, `LIGHT`, `RATE`, `MACRO`, `PROFILE` structs | **Proven** (metadata) |
| Firmware image | Public download, raw ARM, not encrypted | **Proven** |
| Firmware protocol | In-band DFU over HID — **not reverse engineered** | ✗ unknown |

Legend: **Proven** = directly observed in binaries/metadata.
**Partially proven** = confirmed for some commands, assumed for others.
✗ = open question.

---

## Read these in order

| File | Contents |
| --- | --- |
| `PROTOCOL.md` | **Start here.** Transport, VID/PID, framing, the 8-command table, all export RVAs |
| `STRUCTURES.md` | The `.NET` structs and enums that define payload layouts |
| `OPEN-QUESTIONS.md` | Everything unverified + the tester checklist |
| `PLATFORM.md` | Linux reality: libratbag, WebHID, permissions, Wine, hardware alternatives |
| `FIRMWARE.md` | Firmware manifest, image analysis, and why flashing is the risky one |
| `METHOD.md` | How this was derived, and how to reproduce/verify it |
| `RUNBOOK.md` | Laptop setup + the step-by-step hardware session (short, borrowed-unit friendly) |
| `tools/mmo7-webhid-diagnostic.html` | **Read-only** WebHID probe — safe to run, cannot write |

---

## How to help (if you own an M.M.O. 7+)

The single most useful thing is running the read-only diagnostic and reporting back.
It **cannot modify your mouse** — it only issues HID GET_FEATURE requests
(see `PROTOCOL.md` §5 for the proof). It does not touch firmware.

```bash
# Chrome / Edge / Opera required (WebHID does not exist in Firefox)
cd tools && python3 -m http.server 8000
# open http://localhost:8000/mmo7-webhid-diagnostic.html
# click "Connect to mouse" -> pick the M.M.O. 7+ -> "Read all reports" -> "Download JSON file"
```

That answers the four biggest unknowns in one go (`OPEN-QUESTIONS.md` #1–#4).
Step-by-step setup, including permissions and the wired/dongle runs, is in
`RUNBOOK.md`. If you'd rather do it from a terminal, `python3` + `hidapi` works too.

**Please don't flash firmware** on the strength of this document. Nothing here
describes the flash sequence, and a failed flash has no confirmed recovery path.

---

## Scope and provenance

- All findings come from **static analysis** of Mad Catz's own publicly distributed
  Windows software (support package + firmware zip) and from reading Chromium and
  libratbag source.
- **No vendor binaries or firmware images are redistributed here** — only
  observations, offsets and byte layouts, for interoperability purposes.
- Exact file identities and hashes are in `METHOD.md` so anyone can reproduce
  the analysis against the same inputs.
- Firmware images are the vendor's own publicly published downloads, referenced
  by URL only.

## If a test run confirms this, the likely payoff

1. **A read-only WebHID page** — works today, no install, no root, any OS.
2. **A config tool** (WebHID or a small hidapi CLI) for DPI, polling rate, lighting,
   button remapping, the FN layer, 5 profiles and macros.
3. **An update checker** — tell Linux users when new firmware exists, without flashing.
4. *(Optional, much later, higher risk)* Linux-side firmware flashing.
