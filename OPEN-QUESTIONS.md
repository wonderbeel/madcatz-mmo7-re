# OPEN-QUESTIONS.md — what still needs hardware

Everything in `PROTOCOL.md` and `STRUCTURES.md` came from static analysis.
**None of it has been checked against a real M.M.O. 7+.** This file lists exactly
what is unknown, why it matters, and how to test it.

---

## Safety first

- **Reading is safe.** HID GET_FEATURE transfers cannot modify stored settings —
  proof in `PROTOCOL.md` §5.
- **Do not flash firmware** based on this document. The flash sequence is *not*
  described anywhere here, and there is no confirmed recovery path if it fails.
  See `FIRMWARE.md`.
- **Writing is untested.** Nothing here has been validated in the write direction
  except two `Send*` functions read statically. If you experiment with writes,
  know that the realistic failure mode is corrupted profile data — recoverable,
  but the vendor's recovery tool is Windows-only.

---

## High priority — these block everything else

### Q1. What report IDs and sizes does the device actually declare?

**Why it matters:** the entire command table is inferred from the Windows side.
If the descriptor disagrees, everything downstream changes.

**How to test:** the read-only tool dumps `HIDDevice.collections[].featureReports[]`.
Look for report IDs `4, 5, 6, 7, 8, 9, 12, 24` (decimal `4 5 6 7 8 9 12 24`).

**Expected if the hypothesis holds:** feature report IDs matching the table, with
sizes consistent with the declared lengths (8, 9, 10, 15, 56, 95, 95, 131).

**Falsifies it if:** different IDs, extra IDs we did not predict, or a single
unnumbered report (report ID 0) instead of eight numbered ones.

### Q2. Does a response echo the report ID, or is the buffer shifted?

**Why it matters:** it decides whether byte 0 of a response is the command echo
or the first data byte. Every parser offset depends on this.

**How to test:** the tool flags this explicitly. It checks whether
`response[0] == requestedReportId`.

**Note:** Chrome's WebHID and `hidapi` have historically differed on whether the
returned buffer includes the report ID as a leading byte, and it can vary by
platform. This is worth pinning down precisely.

### Q3. Is byte 1 really a length?

**Why it matters:** the framing model assumes `byte0=cmd, byte1=len`.

**How to test:** the tool compares `response[1]` (or `response[0]` if Q2 goes the
other way) against the expected length for that command.

**Watch for:** byte 1 being a *data* byte that merely happens to equal the
payload size in the Windows code path.

### Q4. Do the complement bytes validate on read?

**Why it matters:** it tells us whether the device *enforces* the complement
scheme, or whether the Windows driver just emits redundant bytes nobody checks.

**How to test:** the tool scans response bytes in pairs and checks whether each
odd-indexed byte equals `~previous`.

**If they all validate:** the device likely enforces integrity, which makes
malformed writes much safer (they would be rejected).

**If none validate:** the complement scheme may be Windows-driver-only padding,
which means **writes have no integrity protection at all** — a materially
different risk profile. This is the single most important answer in this list.

---

## Medium priority — needed to build a config tool

### Q5. Per-command complement placement and field offsets

Only `SendProfileId` and `SendPRate` were fully disassembled, and they disagree
about which bytes get a complement byte:

| Command | Placement |
| --- | --- |
| `SendProfileId` | both data bytes complemented |
| `SendPRate` | only the rate byte complemented |

The five larger commands (`Sensor` 56 B, `Light` 15 B, `Keys` 95 B, `Keys_Fn`
131 B, `Macro` 95 B) are **unanalysed**. Their offsets must be recovered either by
disassembling each `Send*` body (RVAs in `PROTOCOL.md` §7) or by differential
capture — set a value, capture, change it, diff.

### Q6. Numeric enum values

Member *names* are known (`STRUCTURES.md` §3) but the numeric values were not
recovered — they appear to be declared implicitly, so likely sequential from 0,
but **that is a guess**. A packet cannot be built without real numbers.

**How to test:** differential capture is the easiest route — change a button's
assignment in the Windows HUD, capture, and see which byte changed and to what.

### Q7. Is the complement enforced on write?

Related to Q4 but distinct: even if reads are unchecked, writes might be
validated. Determining this affects how defensive a write implementation must be.

### Q8. Are there commands beyond these eight?

Eight report IDs came from the export table. Undiscovered commands may exist
(e.g. for the 5D button, shift-mode layers, or sleep timers) that are implemented
inside the firmware without a dedicated export.

**How to test:** capture while clicking **every** control in the Windows HUD and
look for report IDs outside the table.

### Q9. Does the `Keys_Fn` layer (131 bytes) need special handling?

It is by far the largest packet (131 B vs 95 B for `Keys`), consistent with the
FN/shift layer doubling the input count. Its structure is unverified, and this is
the layer that matters most for a 21-button mouse.

---

## Lower priority — nice to confirm

| # | Question |
| --- | --- |
| Q10 | `WakeMode` (8 bytes) — what do the fields mean? Undocumented in the structs. |
| Q11 | `SENSOR.AngleSnap` / `Lift` / `CenterOffset` semantics and ranges |
| Q12 | `DpiX` / `DpiY` array encoding — how many levels, what width per level? |
| Q13 | Does the declared `Light` length (15) exceed the `LIGHT` struct (9)? Why? |
| Q14 | Does the embedded `*_DEFAULT` blob content match a live read? |
| Q15 | How do the extra buttons enumerate to the OS — as HID buttons or keycodes? |
| Q16 | Does `Query*` require the device opened in a particular mode first? |
| Q17 | Does the vendor tool run under **Wine** on Linux? (`PLATFORM.md` §4) |
| Q18 | Does a Windows-side capture reproduce the framing described here? |

---

## Tester procedure

### Step 1 — run the read-only probe (safe, no writes)

Requires Chrome, Edge or Opera — **WebHID does not exist in Firefox**.

```bash
cd tools
python3 -m http.server 8000
```

Open `http://localhost:8000/mmo7-webhid-diagnostic.html`, then:

1. **Connect to mouse** → select the M.M.O. 7+ (try both USB-C and the 2.4 GHz
   dongle if you have both)
2. **Read all reports**
3. **Export JSON** (copies to clipboard)

The tool records the descriptor, every response as hex, and its own verdict on
whether byte 0 matches, whether the length matches, and whether the complement
pairs validate.

### Step 2 — report back

The exported JSON is the single most useful artifact. If JSON export fails, the
on-screen hexdumps are enough. Please also include:

- which connection you used (wired vs dongle) and the reported PID
- OS and browser version
- whether the device appeared in the picker at all

### Step 3 — optional, and genuinely valuable

```bash
libinput debug-events      # how do the extra buttons enumerate?
ratbagctl list             # confirm libratbag sees nothing (expected)
```

### Step 4 — for the ambitious (needs a Windows machine + Wireshark)

Capture a session with the vendor HUD while changing one setting at a time, then
diff. That resolves Q5 and Q6 quickly. See `PLATFORM.md` §5 for the capture setup.

---

## If you own the mouse and want to help further

The highest-leverage single contribution is **Step 1**. It resolves Q1–Q4 in one
run, and those four answers determine whether the rest of this is worth building.
