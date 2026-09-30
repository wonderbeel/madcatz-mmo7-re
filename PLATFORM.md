# PLATFORM.md — Linux, browser, Wine, and the fallback options

How this mouse can plausibly be driven outside Windows, and what is already known
about each route.

---

## 1. Current Linux support: none — PROVEN

| Project | Support for M.M.O. 7+ |
| --- | --- |
| libratbag / Piper | **none** — zero Mad Catz device files |
| ratbagd (Fedora/Bazzite package) | none |
| OpenRazer | N/A (Razer only) |
| Vendor Linux software | **does not exist** |
| Vendor macOS software | does not exist |

The vendor's own support package for the M.M.O. 7+ is a single Windows-only `.rar`.
ArchWiki states plainly that Saitek/Mad Catz provide no Linux configuration
software. Even the *old* R.A.T. mice only ever had community hacks
(`MayeulC/Saitek`, Xorg/udev remaps) covering DPI and battery — never macros.

**So there is no existing path. This research is the path.**

---

## 2. Browser (WebHID) viability

### API fit

The protocol is HID feature reports with numbered report IDs, and WebHID exposes
exactly that:

| Driver export | WebHID |
| --- | --- |
| `Send*` | `device.sendFeatureReport(reportId, data)` |
| `Query*` | `device.receiveFeatureReport(reportId)` |
| `Hid_Read` | `inputreport` event |
| `Hid_Write` | `device.sendReport(reportId, data)` |

The report ID is a **separate argument**; `data` excludes it. Importantly, MDN
documents that *"the `reportId` for each of the report formats that this device
supports can be retrieved from `HIDDevice.collections`"* — so a client can
**discover** report IDs and sizes from the descriptor rather than hardcoding them.

### Constraints

| Constraint | Detail |
| --- | --- |
| Engines | Chromium only — Chrome, Edge, Opera. **No Firefox, no Safari, ever.** |
| Baseline | Officially *not* Baseline; MDN labels WebHID experimental |
| Secure context | HTTPS or `localhost` required |
| User gesture | `requestDevice()` needs a click and shows a device picker |
| Report size | **No hardcoded 64-byte cap.** Chromium's Linux HID backend allocates from the descriptor (`device_info->max_feature_report_size() + 1`, `services/device/hid/hid_connection_linux.cc`) |

### The blocklist question

The WebHID spec is explicit that devices capable of generating *trusted input* get
restricted, and calls out mice specifically:

> *"A HID keyboard or mouse may contain advanced features like programmable
> macros, which store a sequence of inputs and allow the sequence to be played
> back at a later time. Device manufacturers must take care to design such
> features in a way that would prevent a malicious app from reprogramming the
> device with an unexpected input sequence."*

In practice mice **are** accessible — Keychron ships a WebHID configurator for its
M5/M6/M7 mice, and VIA/`usevia.app` does the same for keyboards. But if this
mouse's descriptor exposes a keyboard-like top-level collection (plausible for an
MMO mouse with macro keys), browser behaviour is unverified. This is the one place
a browser may be stricter than a desktop tool.

---

## 3. Permissions

WebHID needs read/write access to `/dev/hidraw*`. This varies by distribution;
it is the most common reason WebHID "sees nothing".

**Observed on Bazzite (Fedora 44 derivative), this was already satisfied:**

```text
$ ls -l /dev/hidraw0
crw-rw-rw-+ 1 root root 239, 0 ...
$ getfacl -p /dev/hidraw0
user:alberto:rw-
```

World read/write plus an explicit user ACL — no udev rule needed. Also verified:

- Chrome is installed as a Flatpak (`com.google.Chrome`, 154.x) with
  `devices=all`, so the sandbox permits `/dev/hidraw*`.
- No Firefox installed (and it would not work regardless).

On a distribution where hidraw is root-only, a udev rule granting
`TAG+="uaccess"` (or group access) is required. Nothing in this project needs
root beyond that.

---

## 4. Wine — try this first for *anything* that must run vendor tooling

The vendor binaries are unusually Wine-friendly:

| Property | Value |
| --- | --- |
| Format | native PE32 i386 (not .NET) |
| Device API | `hidapi.dll` **only** — no WinUSB, no SetupAPI, no libusb |
| hidapi backend | wraps Windows' `hid.dll` (`HidD_GetFeature` / `HidD_SetFeature`) |

Wine **implements `hid.dll` on top of Linux `hidraw`**, and implements the
enumeration calls hidapi uses. So the vendor's config tool and firmware updater
have a real chance of running unmodified on Linux.

Caveats:

- Verify enumeration first — Wine must see the device (a udev rule or
  `WINEDLLOVERRIDES` may be needed).
- **Validate Wine against the config tool (harmless) before trusting it with
  firmware.** If its HID passthrough is faithful for feature reports it will be
  faithful for flashing, but do not discover a fidelity bug mid-erase.
- It is a stopgap for one user, not something distributable — a browser cannot
  use Wine.

---

## 5. Capture setup (the RE accelerator)

Two independent things this buys you: the HID report descriptor, and the firmware
flash sequence (see `FIRMWARE.md` §5).

### Preferred: capture on the Linux host

If the vendor tool runs under Wine, or inside a Windows VM with USB passthrough,
capture on the **Linux host**:

```bash
sudo modprobe usbmon
# then open Wireshark on the usbmonN interface
```

Host-side `usbmon` sees everything including the enumeration-time
`GET_DESCRIPTOR(REPORT_DESCRIPTOR)` transfers. That is strictly more than USBPcap
gives you inside the guest, and needs no extra tooling.

Useful filters:

```text
usb.device_address == N
usb.transfer_type == 0x02 && usb.setup.bRequest == 0x09   # SET_REPORT
usb.transfer_type == 0x02 && usb.setup.bRequest == 0x01   # GET_REPORT
```

### Alternative: USBPcap on native Windows

Works fine, slightly less coverage. On bare metal Windows, install USBPcap and
capture in Wireshark.

### What capture resolves

- Real report IDs and sizes (from the descriptor) — `OPEN-QUESTIONS.md` Q1
- Whether the response buffer includes the report ID leading byte — Q2
- Whether byte 1 is really a length — Q3
- Field offsets and enum values, by differential capture — Q5, Q6
- Undiscovered commands, by clicking every HUD control — Q8
- The firmware flash state machine

---

## 6. If this project stalls: hardware that *does* work on Linux

Recorded here so the fallback is documented rather than re-researched.

### Native libratbag support (no kernel module, full config including button remap)

| Mouse | Notes |
| --- | --- |
| **ASUS ROG Chakram X** | `Wireless=1`, 14 button entries, 5 profiles, 4 DPI slots (100–36000). Tri-mode (2.4 GHz + BT + wired), 8000 Hz wired, up to 150 h, Qi charging, 36K AimPoint, 11 buttons + analog thumb joystick, **hot-swappable switches** (Push-Fit Socket II, ships with 2 spare Omron D2F-01F + tweezers). 128 g. |
| **ASUS ROG Spatha X** | `Wireless=1`, 14 buttons with full mapping list, 5 profiles. Magnetic charging dock, 6 side + 2 top buttons, hot-swap switches, 67 h. **168 g, and no tilt or free-spin wheel.** |
| **ASUS ROG Chakram** (original) | `Wireless=1` but only **5** buttons mapped — the X is much better covered. |

The hot-swap switch sockets matter: the classic failure mode for a long-lived
mouse is double-clicking switches, and these are a parts swap rather than an RMA.

### OpenRazer (DKMS kernel module — works, but more fragile on atomic distros)

| Mouse | Notes |
| --- | --- |
| **Razer Basilisk V3 Pro** | Closest thing to a G502: **HyperScroll tilt wheel with free-spin**, 10+1 buttons, Focus Pro 30K, 112 g, tri-mode. Tooling: OpenRazer daemon (DPI/polling/RGB) + `razer-control-center` (remap/macros/layers). |
| **Razer Naga V2 Pro** | Swappable 12/6/2-button side plates. |

Both are in OpenRazer's supported list (232 devices).

### Ruled out

| Mouse | Why |
| --- | --- |
| Keychron M6 | browser config works, but recurring reliability complaints (middle-click failures, reported sensor glitch) |
| SteelSeries Aerox 9, Roccat Kone XP Air, Glorious Model I 2, Corsair Scimitar Elite | none in libratbag; vendors ship no Linux config. Roccat's brand is defunct — `roccat.com` now 301-redirects to `turtlebeach.com` |
| Mad Catz M.M.O. 7+ / R.A.T. series | the subject of this document |

### Libre alternative worth knowing

Keychron's **Launcher** is a vendor-supported **web app** that configures mice
over WebHID, and the product page states it supports *"Windows, macOS, Linux and
more"* via a Chromium browser — including firmware updates. That is the concrete
proof that a browser-based mouse configurator is viable on Linux, and it is the
model this project would follow.
