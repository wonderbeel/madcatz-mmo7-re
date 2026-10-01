# tools/ — read-only WebHID diagnostic

`mmo7-webhid-diagnostic.html` is a single-file, dependency-free WebHID probe for
the Mad Catz M.M.O. 7+.

## It only reads

It calls `receiveFeatureReport()` and **never** `sendFeatureReport()`. Those are
HID GET_FEATURE control transfers, which cannot modify stored settings, so **this
tool cannot brick, misconfigure, or alter the mouse.** It does not touch firmware.

The vendor's own `Query*` functions were verified to call `HidD_GetFeature` only
(see `../PROTOCOL.md` §5) — so reading is safe by construction, not by luck.

## Running it

Chrome, Edge or Opera required. **WebHID does not exist in Firefox.**

```bash
cd tools
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000/mmo7-webhid-diagnostic.html
```

`localhost` counts as a secure context, which WebHID requires. Opening the file
via `file://` may work but is less reliable.

On Linux the browser also needs access to `/dev/hidraw*`. Most desktop
distributions already allow this; if the device picker is empty, see
`../PLATFORM.md` §3.

## What to do

1. **Connect to mouse** → pick the M.M.O. 7+ (try both USB-C and the 2.4 GHz
   dongle if you have both — they have different PIDs)
2. **Read all reports**
3. **Download JSON file** → saves one file containing every capture so far
   (there is also a clipboard button; the file download is the reliable one)

## What it captures

- Device identity (VID, PID, product name) and whether the PID is recognised
- The **full HID report descriptor** — every collection with its feature, input
  and output report IDs, plus item sizes
- The raw response for every declared feature report **and** each of the eight
  known report IDs, hexdumped
- Its own analysis per report: whether byte 0 matches the requested report ID,
  whether byte 1 matches the expected length, whether the complement pairs
  validate under both framings, and the decoded poll rate
- A warning if the descriptor declares report IDs outside the known table
- **Multiple captures per file** — a wired run and a dongle run both survive, so
  you only export once
- A per-report timeout, so one unimplemented report ID cannot hang the sweep

## What it answers

`../OPEN-QUESTIONS.md` items **Q1–Q4** — the four questions that block everything
else. If those four come back consistent with the model in `../PROTOCOL.md`, the
rest of the project is worth building.

## Please report back

The exported JSON, plus: which connection you used, your OS and browser version,
and whether the device appeared in the picker at all. If the picker is empty,
that is itself a useful finding.
