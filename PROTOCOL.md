# PROTOCOL.md — M.M.O. 7+ HID protocol

Derived entirely by static analysis of Mad Catz's Windows drivers.
**Nothing here has been confirmed against hardware.** See `OPEN-QUESTIONS.md`.

---

## 1. Transport — PROVEN

The native driver `hidDriver.dll` imports from `HID.DLL`:

```text
HidD_GetHidGuid          HidD_GetPreparsedData     HidD_FreePreparsedData
HidD_GetAttributes       HidD_GetFeature           HidD_SetFeature
HidP_GetCaps             HidP_GetSpecificButtonCaps
HidP_GetSpecificValueCaps  HidP_MaxUsageListLength
```

plus `KERNEL32.dll`: `CreateFileW/A`, `ReadFile`, `WriteFile`, `SetFilePointerEx`.

**Conclusion:** the protocol is standard **HID feature reports** via the Windows
HID API. There is no vendor kernel driver, no custom IOCTL and no bulk protocol.

Implications:

| Platform | Mechanism |
| --- | --- |
| Windows | `HidD_GetFeature` / `HidD_SetFeature` |
| Linux | `hidraw` ioctls, or `hidapi` (`hid_get_feature_report` / `hid_send_feature_report`) |
| Browser | WebHID `receiveFeatureReport()` / `sendFeatureReport()` |

No DKMS module. No root beyond hidraw permissions. Fully userspace.

---

## 2. USB identifiers — PROVEN

Extracted from `.NET` IL constants in
`mcz.allinone.devices.MOJO7.Service.DeviceService.WndHandleSetted`:

```text
1848 = 0x0738   VID
3097 = 0x0C19   PID  (wired)
1848 = 0x0738   VID
7170 = 0x1C02   PID  (2.4 GHz dongle receiver)
```

`Set_VIDPID` (export ordinal 1) merely stores its arguments into globals:

```asm
mov  eax, [ebp+0x8]
mov  ds:0x1004ea78, eax     ; VID
mov  eax, [ebp+0xc]
mov  ds:0x1004ea7c, eax     ; PID
```

That is why `hidDriver.dll` and `hidDriver_dongle.dll` are **byte-identical**
(md5 `80b6612d9ad7e11a27b5d8fcd1e39ab0`) — one binary, two VID/PID pairs.

---

## 3. Two HID channels — PROVEN

The driver exports two independent open/close pairs:

```text
Open_DevMonitor      / Close_DevMonitor       (wired)
Open_FeatureDevice   / Close_FeatureDevice    (wired)
...and a _Dongle variant of every single command
```

So configuration goes over a "feature device" handle, and there is a second
"monitor" channel. The `.NET` layer selects wired vs dongle by calling the
matching P/Invoke set. `Set_VIDPID` / `Set_VIDPID_Dongle` configure which pair
is targeted.

---

## 4. Command table — PROVEN

Each setting has a `Send<X>` (write) and `Query<X>` (read) pair.

| Command | Report ID (byte 0) | Length byte (byte 1) | Extracted as |
| --- | --- | --- | --- |
| Sensor | `0x04` | `0x38` = 56 | `0x3804` |
| Light | `0x05` | `0x0F` = 15 | `0x0F05` |
| PollRate | `0x06` | `0x09` = 9 | `0x0906` |
| WakeMode | `0x07` | `0x08` = 8 | `0x0807` |
| Keys | `0x08` | `0x5F` = 95 | `0x5F08` |
| Keys_Fn | `0x09` | `0x83` = 131 | `0x8309` |
| ProfileId | `0x0C` | `0x0A` = 10 | `0x0A0C` |
| Macro | `0x18` | `0x5F` = 95 | `0x5F18` |

Extracted as the immediate in each `Send*` function's
`mov WORD PTR [ebp-...], 0x????` — the word is little-endian, so `0x0906`
means byte 0 = `0x06`, byte 1 = `0x09`.

**Report ID vs command byte:** the buffer passed to `HidD_SetFeature` begins at
the command byte, and under Windows HID semantics the first byte of that buffer
*is* the report ID. So these command bytes are report IDs, and the device almost
certainly declares one numbered feature report per command — rather than a single
shared report. Supporting evidence: the stack buffers built by the two
disassembled functions differ in size (200 bytes for `SendProfileId`, 100 bytes
for `SendPRate`), which fits per-command reports with different declared sizes.

---

## 5. The read path cannot write — PROVEN

This matters because it means a read-only tool is provably safe.

Every `Query*` export calls **only** `HidD_GetFeature`. Call counts, counted over
each function's full body:

| Function | `HidD_GetFeature` calls | `HidD_SetFeature` calls |
| --- | --- | --- |
| QueryFWVersion | 3 | **0** |
| QueryKeys | 3 | **0** |
| QueryKeys_Fn | 3 | **0** |
| QueryLight | 3 | **0** |
| QueryMacro | 3 | **0** |
| QueryOneProfileData | 3 | **0** |
| QueryPRate | 3 | **0** |
| QueryProfileId | 3 | **0** |
| QuerySensorSetting | 2 | **0** |
| QueryWakeMode | 1 | **0** |

The thunks make this unambiguous:

```asm
1000442c:  jmp DWORD PTR ds:0x10044020    ; -> HidD_GetFeature
10004432:  jmp DWORD PTR ds:0x10044024    ; -> HidD_SetFeature
```

The `SetFeature` wrapper passes the buffer and length straight through:

```asm
10004376:  push DWORD PTR [ebp+0xc]      ; length
10004379:  push DWORD PTR [ebp-0x6c]     ; buffer  (= arg1, includes byte 0)
1000437c:  push eax                      ; device handle
1000437d:  call 0x10004432               ; HidD_SetFeature
```

Consequence: a client that only ever issues `receiveFeatureReport()` /
`hid_get_feature_report()` performs HID GET_FEATURE control transfers, which
cannot modify stored settings. **A read-only client cannot brick or reconfigure
the mouse.**

---

## 6. Framing — PARTIALLY PROVEN

Two of eight commands were fully disassembled. The general shape:

```text
offset 0 : command / report ID
offset 1 : length
offset 2+: payload data, with complement bytes interleaved
remainder: zero-filled (buffer is memset before building)
```

There is **no CRC**. Integrity appears to be a bitwise-complement redundancy byte
(`not al`) after some data bytes.

### 6.1 `SendProfileId` (RVA `0x3f60`) — fully analysed

```asm
10003f73:  cmp  DWORD PTR [ebp+0xc], 0x2        ; payload must be exactly 2 bytes
10003f8f:  push 0xc2                            ; memset 194 bytes
10003f96:  lea  eax, [ebp-0x12a]                ; start 6 bytes after header
10003f9e:  call memset
10003faf:  mov  WORD PTR [ebp-0x130], 0x0a0c    ; header
10003fb8:  mov  al, BYTE PTR [ecx]              ; data[0]
10003fba:  mov  BYTE PTR [ebp-0x12e], al
10003fc0:  not  al
10003fc2:  mov  BYTE PTR [ebp-0x12d], al        ; ~data[0]
10003fc8:  mov  al, BYTE PTR [ecx+0x1]          ; data[1]
10003fcb:  mov  BYTE PTR [ebp-0x12c], al
10003fd1:  not  al
10003fd3:  mov  BYTE PTR [ebp-0x12b], al        ; ~data[1]
```

Resulting buffer (200 bytes total):

| Offset | Value |
| --- | --- |
| 0 | `0x0C` (command) |
| 1 | `0x0A` (length) |
| 2 | `data[0]` |
| 3 | `~data[0]` |
| 4 | `data[1]` |
| 5 | `~data[1]` |
| 6..199 | zero |

### 6.2 `SendPRate` (RVA `0x3e40`) — fully analysed

```asm
10003e53:  cmp  DWORD PTR [ebp+0xc], 0x2        ; payload must be exactly 2 bytes
10003e72:  mov  al, BYTE PTR [edi+0x1]          ; data[1] = rate enum
10003e79:  mov  bl, 0x8                         ; 0 -> 0x08
10003e81:  mov  bl, 0x4                         ; 1 -> 0x04
10003e87:  sete bl / inc bl                     ; 2 -> 0x02, else -> 0x01
10003e8c:  push 0x5f                            ; memset 95 bytes
10003e90:  lea  eax, [ebp-0x63]
10003e9f:  mov  BYTE PTR [ebp-0x65], bl         ; rate
10003ea2:  not  bl
10003ea4:  mov  BYTE PTR [ebp-0x66], al         ; profileId
10003ea9:  mov  BYTE PTR [ebp-0x64], bl         ; ~rate
10003eb4:  mov  WORD PTR [ebp-0x68], 0x0906     ; header
```

Resulting buffer (100 bytes total):

| Offset | Value |
| --- | --- |
| 0 | `0x06` (command) |
| 1 | `0x09` (length) |
| 2 | `data[0]` (profile ID) |
| 3 | `data[1]` (rate) |
| 4 | `~data[1]` |
| 5..99 | zero |

> **Note the inconsistency:** in `SendProfileId` *both* data bytes get a
> complement; in `SendPRate` only the rate byte does. **Complement placement is
> not uniform and must be determined per command.** Confirmed only for these two.

### 6.3 Poll-rate encoding — PROVEN

The rate byte is `1000 / Hz`:

| Enum | Byte |
| --- | --- |
| `HZ_125` | 8 |
| `HZ_250` | 4 |
| `HZ_500` | 2 |
| `HZ_1000` | 1 |
| *(any other)* | 1 |

So **`HZ_2000` in the enum is not actually encodable** — it falls through to the
same branch as 1000 Hz and silently becomes 1000 Hz. Leftover enum value.

---

## 7. Export map — PROVEN

`hidDriver.dll` / `hidDriver_dongle.dll`: PE32 i386, 327,168 bytes, image base
`0x10000000`, 6 sections, **same-named exports**. PDB path leaked by the binary:

```text
E:\Work\Hiddriver\Wirless(5 Profile-BK3633_stdcall_VS2019-API-MS661)\
    Driver_20260402\bin\HidDriver.pdb
```

That identifies the target SoC as a **Beken BK3633**, built with VS2019, stdcall,
5 profiles.

| Ordinal | Export | RVA |
| --- | --- | --- |
| 1 | `Set_VIDPID` | `0x43e0` |
| 2 | `Open_DevMonitor` | `0x2fe0` |
| 3 | `Close_DevMonitor` | `0x2c80` |
| 4 | `Open_FeatureDevice` | `0x31b0` |
| 5 | `Close_FeatureDevice` | `0x2d70` |
| 6 | `SetFeature` | `0x4330` |
| 7 | `GetFeature` | `0x2da0` |
| 8 | `Set_Device_Version` | `0x43c0` |
| 9 | `SendProfileId` | `0x3f60` |
| 10 | `SendSensorSetting` | `0x4070` |
| 11 | `SendLight` | `0x3a50` |
| 12 | `SendPRate` | `0x3e40` |
| 13 | `SendKeys` | `0x3510` |
| 14 | `SendKeys_Fn` | `0x3ba0` |
| 15 | `SendMacro` | `0x37b0` |
| 16 | `QueryProfileId` | `0x3420` |
| 17 | `QuerySensorSetting` | `0x3470` |
| 18 | `QueryPRate` | `0x32c0` |
| 19 | `QueryLight` | `0x33d0` |
| 20 | `QueryKeys` | `0x3220` |
| 21 | `QueryKeys_Fn` | `0x3270` |
| 22 | `QueryMacro` | `0x3310` |
| 23 | `QueryFWVersion` | `0x31d0` |
| 24 | `QueryOneProfileData` | `0x3370` |
| 25 | `SendWakeMode` | `0x4240` |
| 26 | `QueryWakeMode` | `0x34c0` |
| 27 | `Hid_Read` | `0x2dd0` |
| 28 | `Hid_Write` | `0x2ee0` |

Ordinal → RVA mapping is recovered from the export address table; note that
naively pairing the address table index with the name table index is off by one
because the name table's ordinal base is 1.

---

## 8. API mapping for a Linux / browser client

| Driver export | hidapi | WebHID |
| --- | --- | --- |
| `Send*` | `hid_send_feature_report()` | `device.sendFeatureReport(id, data)` |
| `Query*` | `hid_get_feature_report()` | `device.receiveFeatureReport(id)` |
| `Hid_Read` | `hid_read()` | `inputreport` event |
| `Hid_Write` | `hid_write()` | `device.sendReport(id, data)` |

WebHID note: the report ID is a **separate argument**; the `data` array excludes
it. Report IDs and sizes are discoverable at runtime from
`HIDDevice.collections[].featureReports[]`, so a client can read the descriptor
instead of hardcoding the table above.

---

## 9. Still unknown

See `OPEN-QUESTIONS.md` for the full list and how to test each. In short:

- the real report IDs and sizes as declared in the HID descriptor
- whether byte 1 of a response is a length, a data byte, or something else
- whether the complement bytes are validated on read, and *enforced* on write
- per-command complement placement for the five larger commands
- field offsets inside each payload
- whether any command exists beyond these eight
