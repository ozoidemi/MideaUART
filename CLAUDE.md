# MideaUART — ozoidemi fork — ionizer support work

This fork adds Mshield/ionizer support for the Midea MAW12AV1QWT-C window AC
(Costco-exclusive variant of the U-shape line). Upstream `dudanov/MideaUART`
does not expose this feature.

## Why this fork exists

The Mshield ionizer on the MAW12AV1QWT-C is a real hardware feature but is
**panel-only** on this model:
- No button on the IR remote.
- Not exposed in Midea's SmartHome app.
- Not exposed via Matter.
- Toggled by holding SWING+FLASHCOOL on the unit's panel for 3 seconds.
- Display flashes "ON" or "OF" (sic, no second F) to indicate state.

Midea's MCU does, however, publish ionizer state on the SmartKey UART. We
verified this empirically (see "Empirical findings" below). The bit just
isn't decoded by the upstream library, because no public Midea app or
remote ever exposes it.

## Current status (as of 2026-05-13)

### ac-living-room.yaml — current version: 2.13

- v2.8 — switched to `external_components` for our fork
- v2.9/2.10 — ECO+ionizer test (passed 24/24, removed in v2.11)
- v2.11 — test scaffolding removed
- v2.12 — `HEAT_COOL` added explicitly to `supported_modes` (was autoconf-gated)
- v2.13 — Remap `HEAT_COOL` → `AUTO` in `ac_adapter.cpp` and `climate.py`; HA now renders the mode as "Auto". Wire protocol unchanged. Commit `01fd494`. Rollback: `git revert 01fd494` (returns to `da68d08`).

### C++ library changes — DONE

All committed to `feat/ionizer-support`:

- **Read path** (`e1a67d3`) — `getIonizer()` decodes `m_data[9]` (raw UART
  byte 19) mask 0x20 from 0xC0 status frames. Note: original commit used
  index 8 (wrong); corrected to index 9.
- **Write path** (`52100f5`) — `Control.ionizer`, `setIonizer()`, and
  `control()` propagation all implemented. `setIonizer()` writes `m_data[9]`
  bit 5 (mask 0x20) — **same byte and bit as read side. CONFIRMED empirically
  via firmware test (logs(3): 3 complete ON/OFF cycles, all AC echoes correct).**
- **library.json** (`c5d019a`) — Added `"ESP8266WiFi": "*"` dependency so
  PlatformIO puts ESP8266WiFi headers on the library's include path when
  building from a git URL (not the registry). Without this, `ApplianceBase.cpp`
  line 6 (`#include <ESP8266WiFi.h>`) fails to compile.
- **m_setEco fix** — `m_setEco(false)` now explicitly sets bit 4 (0x10) so the
  AC receives the "disable ECO" signal rather than the no-op 0x00. Fixes the
  ECO dropout observed when toggling ionizer while ECO was active (logs(3)).
  See "ECO three-state SET protocol" in Empirical findings.

### ESPHome component changes — DONE

In `esphome/components/midea/` within this repo (used via `external_components`):

- **`air_conditioner.h`** (`146bd52`) — Added `get_ionizer()` and
  `set_ionizer(bool)` public methods to the `AirConditioner` class.
- **`climate.py`** (`3313de1`) — Modified `to_code()` to reference our fork
  and to explicitly add `ESP8266WiFi` (ESP8266) or `WiFi` (ESP32) as a
  project-level library dep.
- **`climate.py` + `ac_adapter.cpp`** — Renamed `HEAT_COOL` → `AUTO` in
  `ALLOWED_CLIMATE_MODES` and switched all three `CLIMATE_MODE_HEAT_COOL`
  references in `ac_adapter.cpp` to `CLIMATE_MODE_AUTO`. HA now displays the
  mode as "Auto" natively. Wire protocol unchanged: `MideaMode::MODE_AUTO`
  still sent over UART.

### Hardware testing status

- **Read path**: VERIFIED. Firmware flashed successfully; ionizer state
  changes tracked correctly in ESPHome when toggled from the panel.
- **Write path**: VERIFIED. Switch entity tested end-to-end; ionizer LED on
  the AC turns on/off in sync with HA switch state. Write bit position
  confirmed: `m_data[9]` bit 5 (mask 0x20), same as read. No display message
  expected — the "ON"/"OF" flash is physical-panel-only (SWING+FLASHCOOL chord);
  UART SET commands bypass that display path entirely.
- **ECO+ionizer coexistence**: VERIFIED 24/24. Automated 3-cycle / 8-check
  script (ac-living-room.yaml v2.10) confirmed all ECO↔ionizer state crossings
  correct: preset changes don't affect ionizer, ionizer changes don't affect
  ECO, and both can be active simultaneously without interference.

### Remaining work

1. **DO NOT open a PR to upstream dudanov/MideaUART.** This fork is
   model-specific (MAW12AV1QWT-C) and will not be submitted upstream.
2. **HA dashboard card** — redesign using native HA cards (no HACS). The
   "Auto" label now comes from the fork directly; no template climate needed.

---

## Auto mode label (resolved in the fork)

HA's frontend displays `CLIMATE_MODE_HEAT_COOL` as "Heat/Cool" and
`CLIMATE_MODE_AUTO` as "Auto" — these are two distinct ESPHome/HA enum
values. The MAW12AV1QWT-C uses its "Auto" mode to maintain a set temperature
(cooling-only); `CLIMATE_MODE_AUTO` is semantically correct for this.

**Fix (committed to `feat/ionizer-support`):** Renamed `HEAT_COOL` → `AUTO`
in `ALLOWED_CLIMATE_MODES` (`climate.py`) and switched all three
`CLIMATE_MODE_HEAT_COOL` references in `ac_adapter.cpp` to
`CLIMATE_MODE_AUTO`. `ac-living-room.yaml` updated to `- AUTO` in
`supported_modes`. Wire protocol unchanged — `MideaMode::MODE_AUTO` is still
what goes over UART.

**Note:** HA's `climate` template platform does NOT exist in stock HA
(`No module named 'homeassistant.components.template.climate'`). The fork
approach is the correct solution — no HA-side configuration needed.

---

## Empirical findings (verified, do not re-derive)

### Read side (status response, opcode 0xC0)

- Frame layout: `AA <len=0x2C> AC ... <body starts at byte 11> <counter at len-3> <CRC16 at len-2..len-1>`
- Total length: 44 bytes
- **Ionizer state: raw UART byte 19 (0-indexed), bit 5 (mask 0x20).**
- Verified by capturing UART traffic across multiple panel toggles with a
  clean negative control (3 markers fired, 0 spurious transitions, every
  toggle correlated to a panel action within 1-3 seconds).

**Library index mapping:** `Frame::OFFSET_DATA = 10`, so FrameData `m_data[0]` =
raw byte 10 (the opcode, 0xC0). Therefore raw byte 19 = `m_data[9]`.
`getIonizer()` uses `m_getValue(9, 32)` which correctly maps to raw byte 19
bit 5. (Earlier commits used index 8 = raw byte 18 — wrong byte; fixed.)

ECO mode also lives in `m_data[9]`: bit 4 (0x10) is the ECO read flag. The
ionizer bit 5 (0x20) and ECO bit 4 (0x10) coexist in the same byte without
conflict.

### Write side (SET_STATUS, opcode 0x40)

- Frame layout: `AA <len=0x23> AC ... <body starts at byte 11> ...`
- Total length: 35 bytes (body is shorter than the response body!)
- **Ionizer write bit CONFIRMED: `m_data[9]` bit 5 (mask 0x20) — same byte
  and bit as read side.** Verified via firmware test (logs(3)): 3 complete
  ON/OFF cycles; AC LED and echoed status byte both tracked the SET command.
- Unlike EcoMode (read bit 4 / write bit 7), ionizer read and write use the
  same bit 5. The asymmetry is model-specific; symmetry held here.

### ECO three-state SET protocol (verified empirically, logs(3))

In SET frames (opcode 0x40), `m_data[9]` encodes ECO intent as three states:
- `0x80` (bit 7 set): enable ECO
- `0x10` (bit 4 set): disable ECO
- `0x00` (neither): no change — AC ignores, keeps current ECO state

Upstream `m_setEco(false)` only cleared bit 7, never set bit 4. Any SET
after an ECO-active status response carried `0x10` (the read flag from
`copyStatus`) silently — and the AC interpreted that as "disable ECO".
Fixed: `m_setEco` now calls `m_setMask(9, !state, 16)` in addition to the
existing `m_setMask(9, state, 128)`.

`copyStatus` contamination: `copyStatus` memcpys `m_data[1..10]` from
response frames. When ECO is active, `m_data[9] = 0x10` (bit 4 = ECO read
flag). This persisted in `m_status` and rode into every subsequent SET clone
— the mechanism behind the ECO dropout in logs(3).

### Critical asymmetry: EcoMode (resolved)

`StatusData.h` read vs write bits are not symmetric:
```cpp
bool m_getEco() const { return this->m_getValue(9, 16); }   // bit 4 (0x10) — read
void m_setEco(bool state) { ... m_setMask(9, state, 128); } // bit 7 (0x80) — write enable
                                                             // bit 4 (0x10) — write disable (fixed)
```

This proves the response and SET frame layouts are not field-for-field
symmetric. Ionizer (bit 5 = 0x20) was verified to use the same bit on
both read and write — asymmetry does not apply here.

### No overlap with Turbo

`StatusData.h`:
```cpp
bool m_getTurbo() const { return this->m_getValue(8, 32) || this->m_getValue(10, 2); }
bool getIonizer() const { return this->m_getValue(9, 32); }  // different byte!
```

`m_getTurbo()` reads `m_data[8]` (raw byte 18) and `m_data[10]` (raw byte 20).
Ionizer is at `m_data[9]` (raw byte 19). **These are entirely different bytes —
no overlap.** The earlier belief that they shared a bit was an off-by-one error
(using index 8 instead of 9 for ionizer).

---

## Build infrastructure (hard-won lessons)

### PlatformIO cache layers — know which one is stale

| Cache | Path | Cleared by | Contents |
|-------|------|-----------|---------|
| `.pioenvs` | `/data/build/ac-living-room/.pioenvs/` | ESPHome "Clean Build Files" | Compiled objects + external_components C++ files |
| `.piolibdeps` | `/data/build/ac-living-room/.piolibdeps/` | Manual only (`rm -rf`) | Downloaded library source (MideaUART C++) |

**"Clean Build Files" does NOT clear `.piolibdeps`.** If the MideaUART C++
source is stale, you must delete `.piolibdeps` manually (or delete the entire
`/data/build/ac-living-room/` directory to nuke both at once).

### Source content hash

PlatformIO hashes the C++ source files to identify a cached library version
(`src-<hash>`). Changing only `library.json` does NOT change this hash —
PlatformIO will keep using the cached source. A C++ source file must change to
force re-download, OR `.piolibdeps` must be deleted manually.

### Registry libraries vs git URL libraries

- **Registry** (`dudanov/MideaUART@1.1.9`): PlatformIO knows the platform and
  framework, automatically puts ESP8266WiFi headers on the include path.
- **Git URL** (our fork): compiled more in isolation. `ESP8266WiFi.h` is NOT on
  the include path unless the library declares it in `library.json` dependencies.

That is why `"ESP8266WiFi": "*"` is required in our `library.json`. Without it,
`ApplianceBase.cpp` fails to compile from a git URL source.

### external_components and ESPHome YAML

The ESPHome YAML must use `external_components`, not `platformio_options: lib_deps`:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/ozoidemi/MideaUART
      ref: feat/ionizer-support
    components: [midea]
```

`external_components` fetches the Python code (climate.py) AND the C++ component
files (air_conditioner.h, etc.) from our fork. Those C++ files are cached in
`.pioenvs`. When air_conditioner.h changes in a new commit, click "Clean Build
Files" to evict the cached copy and force re-fetch.

---

## Naming convention (from source reading)

Two tiers in StatusData:
- **Protected (preset-type flags)**: `m_getEco/m_setEco`, `m_getTurbo/m_setTurbo`,
  `m_getSleep/m_setSleep`, `m_getFreezeProtection/m_setFreezeProtection`.
  Used internally via `setPreset`.
- **Public (standalone booleans)**: `setBeeper(bool)`, `isFahrenheits()/setFahrenheits(bool)`.
  Exposed directly.

Ionizer follows the public standalone pattern:
- `getIonizer() const` / `setIonizer(bool state)` on `StatusData`
- `bool m_ionizer{}` member and `getIonizer() const` public getter on `AirConditioner`
- `Optional<bool> ionizer` field on `Control` (using the library's custom
  `dudanov::Optional<T>`, not `std::optional<T>`)

## Control → SET frame data flow (from `AirConditioner.cpp`)

1. `control()` clones the last known state: `StatusData status = this->m_status`
2. For each `Optional<T>` field in `Control`: calls `.hasUpdate(currentMemberVar)` —
   if true, sets `hasUpdate = true` AND calls the corresponding `status.setXxx(value)`
   (ionizer follows this exact same pattern as targetTemp, fanMode, swingMode)
3. `status.setMode(mode)`, `status.setPreset(preset)`, `status.setBeeper(this->m_beeper)`,
   then `status.appendCRC()`
4. Passes the StatusData object directly to `m_setStatus()` →
   `m_queueRequestPriority(FrameType::DEVICE_CONTROL, std::move(status), ...)`

The `StatusData` object's `m_data` vector IS the wire frame body. No
intermediate serialization step. If ionizer is not being changed, the clone
from `m_status` already carries the correct bit — no explicit re-application needed.

## Project file structure

```
MideaUART/
├── include/
│   ├── Appliance/
│   │   ├── ApplianceBase.h           # Base class: UART loop, request queue, m_beeper
│   │   └── AirConditioner/
│   │       ├── AirConditioner.h      # AirConditioner class + Control struct
│   │       ├── StatusData.h          # StatusData, QueryStateData, DisplayToggleData
│   │       └── Capabilities.h        # Capability flags from 0xB5 report
│   ├── Frame/
│   │   ├── Frame.h                   # Full UART frame (header + FrameData)
│   │   └── FrameData.h               # Payload wrapper (m_getValue, m_setMask, etc.)
│   └── Helpers/
│       ├── Helpers.h                 # Optional<T>
│       ├── Log.h / Logger.h
│       └── Timer.h
├── src/                              # Implementations mirror include/
│   ├── Appliance/AirConditioner/
│   │   ├── AirConditioner.cpp
│   │   ├── StatusData.cpp
│   │   └── Capabilities.cpp
│   ├── Frame/{Frame.cpp, FrameData.cpp}
│   └── Helpers/{Log.cpp, Timer.cpp}
├── esphome/components/midea/         # ESPHome component override (our fork only)
│   ├── air_conditioner.h             # Added get_ionizer() / set_ionizer()
│   ├── air_conditioner.cpp
│   └── climate.py                    # References our fork, adds ESP8266WiFi dep
├── test/                             # All entry points are EMPTY STUBS
├── examples/simple/simple.ino
├── library.json                      # Has "ESP8266WiFi": "*" dependency
├── platformio.ini
└── .github/workflows/build.yml
```

## Test infrastructure

There is none worth mentioning. All three test entry points are empty
stubs (`int main() {}`). For ionizer changes, the realistic test is:
1. Library compiles standalone (PlatformIO build).
2. ESPHome compiles with the modified library as a dep.
3. End-to-end test on the actual unit.

No unit test harness to add tests to.

## Commit hygiene rules

- Each commit standalone, each compiles, each tests something coherent.
- Subject lines under 72 chars.
- Commit body explains the **why**, references the MAW12AV1QWT-C model
  and the panel-only nature of the feature.
- Match existing code style. No drive-by reformatting of code we're not
  changing.
- Spotted unrelated bugs/typos: flag, don't fix in this branch.
- DO NOT push to `origin` without explicit user approval.

## Hardware context

- Device: SLWF-01Pro v2 dongle plugged into the AC's SmartKey port.
- ESPHome flashes the dongle (ESP8266, board: esp12e).
- Library runs on the dongle, talks to the AC's mainboard over UART
  (9600 8N1) on GPIO12/14.
- ESPHome runs as a Home Assistant add-on on a Raspberry Pi.
- ESPHome build directory: `/data/build/ac-living-room/` inside the add-on container.

## Working style

- Read source before proposing diffs. Cite line numbers.
- Show diffs in chat before applying.
- Never push to GitHub without explicit approval.
- When uncertain, ask. Better to ask than assume.
