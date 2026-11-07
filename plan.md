# Plan: Add Linux FFB Support to ArduinoJoystickWithFFBLibrary

## Goal

Bring Linux Force Feedback compatibility to the latest upstream v1.0.0 (YukMingLaw) by porting the proven fixes from sgtnoodle's fork, while preserving upstream's v1.0.0 improvements and API.

---

## Context & Source Repos

| Folder | Remote | Role |
|---|---|---|
| `ArduinoJoystickWithFFBLibrary` | [YukMingLaw](https://github.com/YukMingLaw/ArduinoJoystickWithFFBLibrary) @ `ed86e27` (v1.0.0) | **Upstream** — latest release, Windows-only FFB |
| `ArduinoJoystickWithFFBLibraryNoodleLinux` | [sgtnoodle](https://github.com/sgtnoodle/ArduinoJoystickWithFFBLibrary) @ `5224477` | **Linux reference** — older base + 4 Linux-fix commits |
| `ArduinoJoystickWithFFBLibraryLocalDev` | [MiraculousMinotaur](https://github.com/MiraculousMinotaur/ArduinoJoystickWithFFBLibrary) branch `add/linux` | **Working copy** — starts at upstream v1.0.0 |

### Why the upstream doesn't work on Linux

The Linux kernel's `hid-pidff` driver is strict about USB HID PID spec compliance. The upstream library:

1. **Missing `StartDelay` in HID descriptor** — Linux's `pidff_set_effect[]` array expects usage `0xA7` (Start Delay). Without it, the driver fails to map effect fields and rejects the device.
2. **Missing `startDelay` in data struct** — The `USB_FFBReport_SetEffect_Output_Data_t` struct comments out `startDelay`, so the struct size doesn't match the descriptor.
3. **Bug in DownloadForceSample Report Count** — v1.0.0 changed `Report Count` from 2 to 1 (the comment still says "Report Count (2)") — this is a regression bug.

### What sgtnoodle fixed (commits `8c54794` → `1c5afbe`)

| Commit | Description |
|---|---|
| `8c54794` | Debug macros replacing commented-out Serial.print |
| `61ac57e` | **HID descriptor: added StartDelay field + fixed period type** |
| `7a7df99` | **Auto-centering, default effect system, StopEffect/FreeEffect bug fixes** |
| `1c5afbe` | Refined debug print toggle |

---

## Detailed Change Plan

### Phase 1: Critical Linux HID Fixes (Required for Linux)

These are the minimum changes to make the device recognized by Linux's `hid-pidff` driver.

#### 1.1 `src/FFBDescriptor.h` — Add StartDelay to HID descriptor

Add `ENABLE_START_DELAY` define and the StartDelay HID usage block (Usage `0xA7`) after the Direction collection and before the Type Specific Block Offset section.

```
+#define ENABLE_START_DELAY 1
 ...
 (after End Collection Datalink for Direction, line ~144)
+#if ENABLE_START_DELAY
+   0x05,0x0F,        // USAGE_PAGE (Physical Interface)
+   0x09,0xA7,        // USAGE (Start Delay)
+   0x66,0x03,0x10,   // UNIT (Eng Lin:Time)
+   0x55,0xFD,        // UNIT_EXPONENT (-3)
+   0x15,0x00,        // LOGICAL_MINIMUM (00)
+   0x26,0xFF,0x7F,   // LOGICAL_MAXIMUM (7FFF)
+   0x35,0x00,        // PHYSICAL_MINIMUM (00)
+   0x46,0xFF,0x7F,   // PHYSICAL_MAXIMUM (7FFF)
+   0x75,0x10,        // REPORT_SIZE (16)
+   0x95,0x01,        // REPORT_COUNT (01)
+   0x91,0x02,        // OUTPUT (Data,Var,Abs)
+   0x66,0x00,0x00,   // UNIT (None)
+   0x55,0x00,        // UNIT_EXPONENT (00)
+#endif
```

#### 1.2 `src/FFBDescriptor.h` — Fix DownloadForceSample Report Count (bug fix)

The v1.0.0 release changed `0x95, 0x02` (Report Count 2) to `0x95, 0x01` (Report Count 1) but the report has X **and** Y fields — this is a regression. Fix it back:

```diff
-   0x95, 0x01,           //     Report Count (2)   ← says 2 but value is 1!
+   0x95, 0x02,           //     Report Count (2)
```

#### 1.3 `src/DynamicHID/PIDReportType.h` — Uncomment `startDelay` field

The struct must match the descriptor. Uncomment `startDelay` so the host-to-device packet parsing stays aligned:

```diff
    uint8_t  directionY;   // angle (0=0 .. 255=360deg)
-   //  uint16_t  startDelay;  // 0..32767 ms
+   uint16_t  startDelay;  // 0..32767 ms
  } USB_FFBReport_SetEffect_Output_Data_t;
```

> **Impact**: This changes the struct layout. Any code reading `SetEffect` data from USB will now correctly parse the extra 2 bytes that Linux sends.

---

### Phase 2: Effect System Bug Fixes (Recommended)

These fix real bugs in the effect lifecycle, improving correctness on both Windows and Linux.

#### 2.1 `src/DynamicHID/PIDReportHandler.cpp` — Fix `StopEffect()`

Upstream clears the `PLAYING` bit AND frees RAM — but stopping ≠ freeing. The RAM should only be freed in `FreeEffect()`:

```diff
  void PIDReportHandler::StopEffect(uint8_t id)
  {
      if (id > MAX_EFFECTS) return;
-     g_EffectStates[id].state &= ~MEFFECTSTATE_PLAYING;
-     pidBlockLoad.ramPoolAvailable += SIZE_EFFECT;
+     g_EffectStates[id].state = MEFFECTSTATE_ALLOCATED;
  }
```

#### 2.2 `src/DynamicHID/PIDReportHandler.cpp` — Fix `FreeEffect()`

Move the RAM accounting here where it belongs:

```diff
  void PIDReportHandler::FreeEffect(uint8_t id)
  {
      ...
      g_EffectStates[id].state = 0;
      if (id < nextEID) nextEID = id;
+     pidBlockLoad.ramPoolAvailable += SIZE_EFFECT;
  }
```

#### 2.3 `src/DynamicHID/PIDReportHandler.cpp` — Zero-init effect states in constructor

```diff
  PIDReportHandler::PIDReportHandler()
  {
      nextEID = 1;
      devicePaused = 0;
+     memset(&g_EffectStates, 0, sizeof(g_EffectStates));
  }
```

---

### Phase 3: Default Effect / Auto-Centering (Optional, Enhances Linux UX)

Linux applications often expect a default spring/centering effect. sgtnoodle's fork implements this via a "slot 0" default effect system.

#### 3.1 `src/DynamicHID/PIDReportHandler.h` — Add new method declarations

```cpp
// Enables a default effect with the given effect parameters.
// This function must be called during initialization.
void EnableDefaultEffect(const TEffectState &effect);

// Debug prints the effect at the specified id.
void PrintEffect(uint8_t id);
```

Also update the `nextEID` comment:

```diff
- volatile uint8_t nextEID;  // FFP effect indexes starts from 1
+ // FFP effect indexes start from 1.
+ // The 0th effect is used to remember default effect parameters, and is copied
+ // to index 1 after freeing all effects.
+ volatile uint8_t nextEID;
```

#### 3.2 `src/DynamicHID/PIDReportHandler.cpp` — Add `EnableDefaultEffect()` and `PrintEffect()`

Port the two new functions from sgtnoodle. `EnableDefaultEffect()` stores a template in slot 0 and copies it to slot 1 for immediate use. `PrintEffect()` is a debug helper.

#### 3.3 `src/DynamicHID/PIDReportHandler.cpp` — Update `FreeAllEffects()`

Instead of wiping all slots, preserve slot 0 (the default effect template) and restore it to slot 1:

```diff
  void PIDReportHandler::FreeAllEffects(void)
  {
      nextEID = 1;
-     memset((void*)& g_EffectStates, 0, sizeof(g_EffectStates));
+     for (uint8_t i = 1; i < MAX_EFFECTS + 1; ++i)
+         memset(&g_EffectStates[i], 0, sizeof(TEffectState));
+     if (g_EffectStates[0].state != MEFFECTSTATE_FREE)
+     {
+         // Default effect is enabled — restore it.
+         memcpy(&g_EffectStates[1], &g_EffectStates[0], sizeof(TEffectState));
+         nextEID += 1;
+         StartEffect(1);
+     }
      pidBlockLoad.ramPoolAvailable = MEMORY_SIZE;
  }
```

#### 3.4 `src/Joystick.cpp` + `src/Joystick.h` — Add `EnableAutoCenter()`

Public API for sketches to enable a default spring centering effect:

```cpp
void Joystick_::EnableAutoCenter(int16_t coefficient, int16_t saturation)
{
    TEffectState effect;
    effect.effectType = USB_EFFECT_SPRING;
    effect.gain = 255;
    effect.enableAxis = X_AXIS_ENABLE;
    effect.conditions[0].cpOffset = 0;
    effect.conditions[0].positiveCoefficient = coefficient;
    effect.conditions[0].negativeCoefficient = coefficient;
    effect.conditions[0].positiveSaturation = saturation;
    effect.conditions[0].negativeSaturation = saturation;
    effect.conditions[0].deadBand = 0;
    effect.duration = 0;
    DynamicHID().pidReportHandler.EnableDefaultEffect(effect);
}
```

#### 3.5 `src/Joystick.cpp` — Fix `forceCalculator()` loop start

Start iterating from id `1` (not `0`) since slot 0 is now the default-effect template, not a live effect:

```diff
- for (int id = 0; id < MAX_EFFECTS; id++) {
+ for (int id = 1; id < MAX_EFFECTS; id++) {
```

Also fix the duration check to handle `duration == 0` (used for infinite-duration default effects):

```diff
- ((effect.elapsedTime <= effect.duration) ||
-  (effect.duration == USB_DURATION_INFINITE))
+ (effect.duration == 0 || effect.duration == USB_DURATION_INFINITE ||
+  effect.elapsedTime <= effect.duration)
```

---

### Phase 4: v1.0.0 Upstream Cleanup (Decisions)

These are changes v1.0.0 introduced that conflict with or differ from sgtnoodle's approach. We need to decide what to keep.

#### 4.1 `getUSBPID()` vs merged `getForce()` — **KEEP BOTH**

- **Keep** `getUSBPID()` for backward compatibility with v1.0.0 examples that use timer ISR polling.
- **Also** call `RecvfromUsb()` at the start of `getForce()` so simple sketches that only call `getForce()` still work (the sgtnoodle approach).
- This way both usage patterns work:
  - **v1.0.0 pattern**: ISR calls `getUSBPID()`, loop calls `getForce()`
  - **Simple pattern**: loop just calls `getForce()` (which internally polls USB first)

#### 4.2 `out_ffbdata` placement — **KEEP LOCAL** (sgtnoodle)

Move `out_ffbdata[64]` from `DynamicHID_` class member to a local variable in `RecvfromUsb()`. This is safer (stack-allocated, no stale data between calls) and matches the sgtnoodle approach.

```diff
  // DynamicHID.h: remove member
- uint8_t out_ffbdata[64];

  // DynamicHID.cpp: make local
  void DynamicHID_::RecvfromUsb() {
      if (usb_Available() > 0) {
+         uint8_t out_ffbdata[64];
          uint16_t len = USB_Recv(PID_ENDPOINT_OUT, &out_ffbdata, 64);
```

#### 4.3 Unused variables — **REMOVE**

- `unsigned int timecnt = 0;` in Joystick.cpp — declared but never used.
- `bool is_calculating_force = true;` in Joystick.h — declared but never referenced.

#### 4.4 Force range mapping — **KEEP ±255** (sgtnoodle/original)

v1.0.0 changed `map(..., -255, 255)` to `map(..., -250, 250)` without explanation. Revert to ±255 which matches the documented API range.

#### 4.5 `inline` keyword — **REMOVE** (sgtnoodle)

v1.0.0 added `inline` to `NormalizeRange()`, `ApplyGain()`, `ApplyEnvelope()`. These are class member functions defined in .cpp — `inline` on definitions in .cpp doesn't do what's intended (the compiler inlines based on its own heuristics anyway). Remove for clarity.

---

### Phase 5: Debug Infrastructure (Optional)

#### 5.1 Add configurable debug macros to `PIDReportHandler.cpp`

```cpp
#if 0  // Set to 1 to enable FFB debug output over Serial
#define DEBUG_PRINT(x)   Serial.print(x)
#define DEBUG_PRINTLN(x) Serial.println(x)
#else
#define DEBUG_PRINT(x)
#define DEBUG_PRINTLN(x)
#endif
```

> Default to **OFF** (`#if 0`) for production use. sgtnoodle had it ON for development.

#### 5.2 Add debug prints in key handler functions

Replace the commented-out `//Serial.println(...)` calls in `UppackUsbData()`, `SetEffect()`, and `SetPeriodic()` with the `DEBUG_PRINT`/`DEBUG_PRINTLN` macros.

---

## Commit Strategy

Implement changes as logical, reviewable commits on the `add/linux` branch:

| # | Commit | Phases |
|---|---|---|
| 1 | `fix: add StartDelay to HID descriptor for Linux hid-pidff compliance` | 1.1, 1.2, 1.3 |
| 2 | `fix: correct StopEffect/FreeEffect RAM accounting` | 2.1, 2.2, 2.3 |
| 3 | `feat: add default effect system and EnableAutoCenter API` | 3.1–3.5 |
| 4 | `refactor: cleanup unused vars, fix force range, localize out_ffbdata` | 4.1–4.5 |
| 5 | `feat: add toggleable debug macros for FFB handlers` | 5.1, 5.2 |

---

## Files Changed Summary

| File | Phase(s) | Type |
|---|---|---|
| `src/FFBDescriptor.h` | 1.1, 1.2 | **Linux critical** |
| `src/DynamicHID/PIDReportType.h` | 1.3 | **Linux critical** |
| `src/DynamicHID/PIDReportHandler.cpp` | 2.1–2.3, 3.2–3.3, 5.1–5.2 | Bug fix + feature |
| `src/DynamicHID/PIDReportHandler.h` | 3.1 | Feature |
| `src/DynamicHID/DynamicHID.cpp` | 4.2 | Refactor |
| `src/DynamicHID/DynamicHID.h` | 4.2 | Refactor |
| `src/Joystick.cpp` | 3.4–3.5, 4.1, 4.3–4.5 | Feature + cleanup |
| `src/Joystick.h` | 3.4, 4.1, 4.3 | Feature + cleanup |

---

## Testing Plan

1. **Compile check** — Verify the library compiles for Arduino Leonardo / Pro Micro (ATmega32U4)
2. **Windows smoke test** — Verify existing FFB functionality on Windows isn't broken (using `SingleJoystickFFB` or `YourFirstFFBController` examples)
3. **Linux smoke test** — Connect to a Linux machine and verify:
   - `dmesg` shows successful PID FF device initialization (no `hid-pidff` errors)
   - `evtest` lists FF effects as supported
   - `fftest /dev/input/eventX` can upload and play a constant force effect
4. **Auto-center test** — If using `EnableAutoCenter()`, verify the spring effect engages on device reset

---

## Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| StartDelay changes struct alignment → Windows breaks | High | Test on Windows immediately after Phase 1 |
| Existing sketches using `getUSBPID()` break | Medium | Keep `getUSBPID()` alongside merged `getForce()` |
| Default effect system causes unexpected forces | Low | Default effect only activates if `EnableAutoCenter()` is called |
| Debug output left ON tanks performance | Low | Default debug macros to OFF (`#if 0`) |

