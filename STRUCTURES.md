# STRUCTURES.md — payload layouts

The `.NET` layer declares typed structs for every command. These come from
`mcz.allinone.devices.dll` metadata and are **unobfuscated** — real names, no
symbol stripping.

> **Caveat, with evidence:** metadata *field order is not necessarily wire order*.
> See §5. Field offsets must be confirmed against hardware or by disassembling
> each `Send*` function.

---

## 1. Assembly layout

The configuration app (`MadCatz H.U.D.`) is a **.NET 8 WPF application**,
`net8.0/win-x86`, using `CommunityToolkit.Mvvm`. It ships as 497 files, of which
**465 are .NET assemblies**. The relevant ones:

| Assembly | Size | Role |
| --- | --- | --- |
| `mcz.allinone.devices.dll` | 984,576 B | **the protocol layer** (`MOJO7.*`) |
| `mcz.allinone.resources.dll` | 353,280 B | images/strings |
| `mcz.allinone.controls.dll` | 291,840 B | UI controls |
| `MadCatz H.U.D..dll` | 205,312 B | app shell |
| `mcz.allinone.common.dll` | 48,640 B | shared |
| `mcz.allinone.viewmodel.dll` | 36,352 B | MVVM |
| `mcz.allinone.view.dll` | 32,768 B | MVVM |
| `mcz.allinone.model.dll` | 20,992 B | MVVM |
| `mcz.allinone.interfaces.dll` | 14,336 B | interfaces |

Device implementation lives under the `MOJO7` namespace — the internal codename
for this mouse. `mcz.allinone.devices.dll` contains 123 types and 977 methods.

**No obfuscation.** Zero matches for ConfuserEx, Eazfuscator, SmartAssembly,
.NET Reactor, DeepSea or Dotfuscator. A PDB ships that still references real
source file names (`MainWindow.xaml.cs`, `MainContainerViewModel.xaml.cs`, …).

### Notable classes

| Class | Methods | Notes |
| --- | --- | --- |
| `MOJO7.Api.DriverApi` | 47 | P/Invoke into `hidDriver.dll` / `hidDriver_dongle.dll` |
| `MOJO7.Device` | 54 | device lifecycle |
| `MOJO7.Service.DeviceService` | 10 | holds the VID/PID constants |
| `MOJO7.Service.AppLinkService` | 4 | per-application profile switching |
| `MOJO7.Model.KeysModel` | 62 | key mapping |
| `MOJO7.Api.DriverApi` | 47 | transport |

`DeviceBase` exposes the domain actions: `SetDpiLevel`, `SetProfileId`,
`SendProfile`, `SaveProfiles`, `LoadProfiles`, `SetWakeMode`, `ResetToFactory`,
`SleepDetection`, `CheckFwNewUpdate`, `Wakeup`.

`AppLinkServiceBase` implements foreground-app detection via `user32`
(`GetForegroundWindow`, `GetWindowThreadProcessId`, `OpenProcess`,
`QueryFullProcessImageName`) — i.e. **profiles can auto-switch per application**.

---

## 2. The wire structs

Every struct ends in `ProfileId` — one packet per onboard profile.

| Struct | Fields |
| --- | --- |
| `PROFILE` | `ProfileCount` (byte), `ProfileId` (byte) |
| `RATE` | `HZ` (`RATE_ENUM`), `ProfileId` (byte) |
| `SENSOR` | `Lift`, `AngleSnap`, `DpiFlag`, `Reserve`, `CenterOffset` (bytes), `DpiX` (byte[]), `DpiY` (byte[]), `CurrentDpiLevel` (byte), `DpiLevelIndication` (byte[]), `DpiIndicationType` (byte), `ProfileId` (byte) |
| `LIGHT` | `LightMode` (`LIGHT_MODE`), `BreathingSpeed` (byte), `LightBrightness` (byte), `FixedColor` (byte[]), `SleepTime` (byte), `Reserve` (byte), `ProfileId` (byte) |
| `KEYS` | `ButtonSetting` (`BUTTON_INFO[]`), `ButtonFunction` (`BUTTON_FUNCTION_TABLE`) |
| `BUTTON_INFO` | `ModifyKeyCodeType` (byte), `KeyIndex` (byte), `AttributeCode` (byte) |
| `MACRO` | `MacroCodeType` (`MACRO_CODE_TYPE`), `Reserve` (byte[]), `MacroExecBehavior` (byte), `MacroName` (byte[]), `KeyMacroNumber` (byte), `KeyInfo` (`MACRO_KEY_INFO[]`) |
| `MACRO_KEY_INFO` | `KeyCode` (byte), `Index` (byte) |

`DpiX` / `DpiY` being arrays implies **per-axis DPI** with multiple levels.
`FixedColor` is presumably 3 (or 4) bytes of RGB.

### Sanity check against declared packet lengths

`LIGHT` has 7 fields including a 3-byte RGB `FixedColor` → 1+1+1+3+1+1+1 = **9 bytes**.
The command table declares Light length `0x0F` = 15. Not equal — so the declared
length is the full payload including reserving room for more color data, or the
struct is padded. **Open question, see `OPEN-QUESTIONS.md`.**

---

## 3. Enums — members known, **numeric values NOT extracted**

This is important: the `.NET` Constant table did not yield usable enum values for
these types, which suggests they are declared implicitly (0, 1, 2, …). The member
names are exact; **the numbers are still unknown** and must be recovered from IL
or from hardware captures before packets can be built.

### `BUTTON_FUNCTION_TABLE` — 46 members

```text
ButtonFunctionBlocked   ButtonOff          Click                Menu
UniversalScrolling      IEBackward         IEForward            DoubleClick
Firebutton              ScrollUp           ScrollDown           TiltLeft
TiltRight               DpiCycle           DpiUp                DpiDown
EasyAimDpi              AssignShortcut     AssignMacro          WindowsKey
OpenPlayer              PreTrack           NextTrack            PlayPause
Stop                    Mute               VolumeUp             VolumeDown
WindowsCalculator       WindowsEMail       WindowsWwwFavorites  WindowsWwwForward
WindowsWwwBack          WindowsWwwStop     WindowsMyComputer    WindowsWwwRefresh
WindowsIEBrowseHome     WindowsWwwSearch   FN                   ReportRateLoop
ProfileCycle            ProfileUp          ProfileDown          RFModeKeying
LeftActive              RightActive
```

Note `EasyAimDpi` — this is the "sniper"/shift-DPI clutch feature.

### `KEY_SETTING_INDEX` — 30 members (the physical button map)

```text
KEY_LEFT          KEY_RIGHT         KEY_MIDDLE        KEY_INDEX_3
KEY_INDEX_4       KEY_PROFILE_CYCLE KEY_LEFT_ACTIVE   KEY_RIGHT_ACTIVE
KEY_DPI_LOOP      KEY_FN            KEY_SHORTUCT_1    KEY_IE_BACKWARD
KEY_IE_FORWARD    KEY_SHORTUCT_2    KEY_SHORTUCT_3    KEY_SHORTUCT_4
KEY_SHORTUCT_5    KEY_SHORTUCT_6    KEY_SHORTUCT_7    KEY_DPI_EASY_AIM
KEY_INDEX_20      KEY_SHORTUCT_8    KEY_SHORTUCT_9    KEY_SHORTUCT_10
KEY_INDEX_24      KEY_INDEX_25      KEY_TILT_LEFT     KEY_TILT_RIGHT
KEY_SCROLL_UP     KEY_SCROLL_DOWN
```

`KEY_SHORTUCT_*` is a typo in the vendor's source (should be `SHORTCUT`) —
incidental proof this is original internal naming, not renamed or obfuscated.

The `KEY_INDEX_3 / 4 / 20 / 24 / 25` gaps suggest unnamed buttons that were
numbered positionally. Ten shortcut slots + FN + DPI loop + easy-aim + tilts +
scrolls + profile cycle matches the advertised 21 programmable inputs.

### Other enums

| Enum | Members |
| --- | --- |
| `LIGHT_MODE` | `Close`, `Light`, `Breathe`, `Neon`, `ColorLoop` |
| `RATE_ENUM` | `HZ_125`, `HZ_250`, `HZ_500`, `HZ_1000`, `HZ_2000` |
| `MACRO_CODE_TYPE` | `LOOP_COUNT`, `LOOP_TO_PRESS_AGAIN`, `LOOP_TO_RELEASE` |
| `MACRO_ACTION_TYPE` | `MAKE`, `BREAK` |

Recall `HZ_2000` is a dead value — see `PROTOCOL.md` §6.3.

---

## 4. Embedded default payloads

`MOJO7.Constant` declares four `byte[]` defaults plus metadata strings:

```text
MAIN_IMAGE   DEVICE_IMAGE   MODEL_IMAGE   FW_UPDATE_URL
RATE_DEFAULT  SENSOR_DEFAULT  KEYS_DEFAULT  LIGHT_DEFAULT
```

The four arrays exist as static initialised data in the DLL's PE image, with
declared sizes:

| Declared size | Likely binding |
| --- | --- |
| **9 bytes** | `LIGHT_DEFAULT` (matches `LIGHT` = 9 bytes exactly) |
| **24 bytes** | `SENSOR_DEFAULT` (inferred) |
| **48 bytes** | `RATE_DEFAULT` or `MACRO` (inferred) |
| **91 bytes** | `KEYS_DEFAULT` (inferred — 30 buttons × 3-byte `BUTTON_INFO` = 90, +1) |

The **sizes are observed**; the **bindings are inference**. Extracting the actual
byte values is straightforward (field RVA table + `pe.get_data`) and would give
known-good default packets to diff against — a useful next step.

Also present as IL constants: `1000` and `4096`.

---

## 5. Field order ≠ wire order — evidence

`RATE` is declared in metadata as:

```text
HZ        (RATE_ENUM)
ProfileId (byte)
```

But `SendPRate` reads its two input bytes as:

```asm
mov  al, BYTE PTR [edi+0x1]     ; byte 1 -> mapped through the 8/4/2/1 table
mov  BYTE PTR [ebp-0x66], al    ; ...stored as the profile id slot
mov  al, BYTE PTR [edi]         ; byte 0 -> stored as the rate slot
```

i.e. the marshal order the driver sees is reversed relative to the metadata
declaration order. **Do not assume metadata order equals packet order.** Each
`Send*`/`Query*` body must be read to fix offsets, or confirmed by capture.

---

## 6. What this enables

Because the structs and enums are readable, a client does not have to *guess* the
payload format — it has to *transcribe* it. The remaining work is:

1. Confirm the HID descriptor's report IDs and sizes (see `OPEN-QUESTIONS.md`).
2. Recover the enum numeric values from IL.
3. Determine per-command complement placement and field offsets.
4. Serialise the structs accordingly.
