# RUNBOOK.md — laptop setup and the hardware session

Field procedure for getting real data out of an M.M.O. 7+ when you only have it
for a short window (e.g. a borrowed unit at an event).

**Read `OPEN-QUESTIONS.md` §"Safety first" before you touch anything.** The short
version: reading is provably safe, writing is untested, flashing is off the table.

---

## 0. Hard rules

| Rule | Why |
| --- | --- |
| **Never flash firmware.** | No recovery path, and the transfer protocol is unknown (`FIRMWARE.md`). Not negotiable on a loaner. |
| **Run the probe before any write test.** | If you only get one thing done, get Q1–Q4. |
| **Writes only on a unit the owner has agreed can be lost.** | Realistic failure mode is corrupted profiles. |
| **Export and *save* after every stage.** | A capture you didn't download is a capture you don't have. |

---

## 1. What you need on the laptop

| | Item | Notes |
| --- | --- | --- |
| 1 | Chromium browser | Chrome, Edge or Opera. **Firefox cannot do WebHID.** |
| 2 | This repo (or just `tools/`) | Copy it locally — assume **no usable wifi**. |
| 3 | `python3` | Only to serve the page on `localhost`. |
| 4 | USB-C cable + a USB-A/C hub | The demo unit may be wired *or* on a dongle. |
| 5 | `usbutils` (`lsusb`, `usbhid-dump`) | For the raw report descriptor and enumeration. |
| 6 | *(optional)* Wireshark + `usbmon` | Only if you attempt the differential capture (§5). |
| 7 | *(optional)* a Windows VM or Bottles/Wine with the vendor HUD | §5. High effort, high payoff. |

Nothing here needs root except the optional descriptor dump and the udev rule.

---

## 2. Laptop setup (do this the day before, not on site)

### 2.1 Serve the probe locally

```bash
cd /path/to/madcatz-mmo7-re/tools
python3 -m http.server 8000
# then open: http://localhost:8000/mmo7-webhid-diagnostic.html
```

`localhost` is a secure context, which WebHID requires. Opening the file over
`file://` sometimes works and sometimes doesn't — **don't risk it at the event.
Use the local server.** It needs no network.

### 2.2 Give the browser access to `/dev/hidraw*`

On most desktops this is already fine. Check:

```bash
ls -l /dev/hidraw*            # world rw, or an ACL for your user
getfacl -p /dev/hidraw0 2>/dev/null
```

If it's root-only, add a udev rule:

```bash
sudo tee /etc/udev/rules.d/99-madcatz-mmo7.rules >/dev/null <<'EOF'
# Mad Catz M.M.O. 7+ — wired 0x0C19 and 2.4 GHz dongle 0x1C02
KERNEL=="hidraw*", ATTRS{idVendor}=="0738", ATTRS{idProduct}=="0c19", MODE="0660", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="0738", ATTRS{idProduct}=="1c02", MODE="0660", TAG+="uaccess"
EOF
sudo udevadm control --reload && sudo udevadm trigger
```

If Chrome is a Flatpak, confirm the sandbox allows USB devices:

```bash
flatpak info --show-permissions com.google.Chrome | grep devices   # want: devices=all
# if not:
flatpak override --user --device=all com.google.Chrome
```

### 2.3 Rehearse on *any* HID device

This is the single most valuable preparation. Plug in any mouse/keyboard/whatever,
open the probe, click through **Connect → Read all reports → Download JSON file**.
You are testing the *tooling*, not the device:

- Does the picker appear at all?
- Does **Download JSON file** actually write a file?
- Does the page render when there is no network?

Fix anything broken here, on hardware you don't care about, **before** the event.

---

## 3. At the event — the read-only script (5 minutes)

Do this first, whatever else happens. It answers Q1–Q4, which gate the whole project.

1. **Ask** to move the demo unit to your laptop. Say it is **read-only**: the page
   calls HID GET_FEATURE only and cannot change any setting (see `PROTOCOL.md` §5).
2. Open the probe at `http://localhost:8000/mmo7-webhid-diagnostic.html`.
3. **Connect to mouse** → pick the device.
   - *Picker empty?* Tick **show all HID devices** and try again. This catches a
     VID/PID we didn't predict.
   - If it still fails, see §6.
4. **Read all reports.**
5. **Download JSON file.** ← do not skip. Nothing is saved until you do.
6. **Repeat for the dongle** if they have one. The probe keeps every capture in
   one file, so you only export once at the end — but export after each stage
   anyway in case the window closes.
7. **Screenshot** the Summary card and the descriptor card. Cheap insurance.

### While you are standing there, also grab

```bash
lsusb | grep -i 0738                     # confirm VID/PID and which path is active
```

and, if you can get root for ten seconds, the **raw report descriptor** — this is
strictly better evidence for Q1 than the WebHID parsed view:

```bash
sudo usbhid-dump -e descriptor -d 0738:0c19     # wired
sudo usbhid-dump -e descriptor -d 0738:1c02     # dongle
# or, per-device:
for d in /sys/bus/hid/devices/*; do
  echo "== $d"; sudo xxd "$d/report_descriptor"
done
```

And, to answer Q15 (how the extra buttons enumerate to the OS):

```bash
libinput debug-events      # press each button; note what the kernel sees
```

---

## 4. If the HUD is available on their machine

Their demo machine is configured with the software, so it can answer
product-level questions you can't get from the mouse alone. Ask them to show you:

- the **firmware version** screen → the expected `QueryFWVersion` output, free;
- whether the unit is wired, 2.4 GHz, or both;
- which HUD build they ship.

Photograph the screen. Do not install anything on their machine.

---

## 5. Optional: the differential capture (high payoff, high effort)

This is what resolves **Q5** (per-command field offsets and complement placement),
**Q6** (numeric enum values) and **Q8** (undiscovered commands). It needs the
vendor HUD running against the mouse **on a machine you control**, with `usbmon`
capturing. Realistically this needs a quiet block at the event, not five minutes.

### Capture setup (host side, always works)

```bash
sudo modprobe usbmon
# then open Wireshark on a usbmonN interface
```

Filters (`PLATFORM.md` §5):

```text
usb.device_address == N
usb.transfer_type == 0x02 && usb.setup.bRequest == 0x09   # SET_REPORT (writes)
usb.transfer_type == 0x02 && usb.setup.bRequest == 0x01   # GET_REPORT (reads)
```

### Getting the HUD to run

| Route | Reality check |
| --- | --- |
| **Windows VM + USB passthrough**, capture on the Linux host | Most likely to work. Build it in advance. |
| **Bottles / Wine** on the same laptop | Cheap to try. The HUD is **.NET 8 WPF** — WPF under Wine is historically flaky, so treat success as a bonus, not a plan. |

Note the HUD ships as `mcz.allinone.install.msi` (66.6 MB). If it is a
framework-dependent build you will also need the .NET 8 **Desktop** runtime inside
the bottle/VM.

### Method

Change **exactly one control at a time**, capture, then diff:

1. polling rate 1000 → 500 Hz
2. one DPI level's X value
3. one button's assignment
4. light mode fixed → breathe

Then click **every remaining control** in the HUD while capturing, to catch command
IDs outside our eight (`PROTOCOL.md` §4) — that is Q8, and it is free once you are
already capturing.

A diff helper is worth having ready: two capture files in, "these bytes changed" out.

---

## 6. Failure modes and what to do

| Symptom | Likely cause | Action |
| --- | --- | --- |
| Device picker is empty | hidraw permissions, or Flatpak sandbox | §2.2; or tick **show all HID devices** |
| Picker shows the device but `open()` fails | device already claimed | unplug/replug; close any other tool holding it |
| A report read fails or times out | report ID not implemented | expected — the probe records it and continues |
| `byte0` doesn't match the report ID | response omits the report ID | **this is data, not a bug** — it's Q2 |
| Complement pairs don't validate | scheme is Windows-driver-only padding | **this is the most important answer in the list** (Q4) — it means writes have no integrity protection |
| Export does nothing | clipboard permission/focus | use **Download JSON file**, then **Show JSON** as a last resort |
| No network at all | — | irrelevant: the probe and the server are local |

---

## 7. What to bring back

- [ ] `mmo7-<timestamp>.json` — one file, all captures (wired + dongle)
- [ ] `usbhid-dump` output for both PIDs (raw report descriptor)
- [ ] `libinput debug-events` notes (Q15)
- [ ] Screenshots of the Summary + descriptor cards
- [ ] Photos: retail box, labels, serial, firmware-version screen
- [ ] Whether the unit was wired, dongle, or both — and which PID each showed
- [ ] *(if §5 happened)* the `usbmon` capture files

Then update `OPEN-QUESTIONS.md` (tick off Q1–Q4) and correct `PROTOCOL.md` where
hardware disagreed with the static analysis. **Expect it to disagree somewhere —
that is the point of the exercise.**
