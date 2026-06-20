# Sugar Valley / NeoPool — Write Safety Analysis

Why this package exposes **setpoint writes** but **not relay control**, and what it
would take to add relay control safely later. This is the rationale behind the
write surface in `neopool.yaml`; read it before adding any new write endpoint.

> **TL;DR.** A write to this controller is safe **iff** it is a *full-word write to a
> dedicated parameter register, needs no manual-control bracket, and carries a clamped
> value*. **Setpoints** meet that bar. **Relay control does not** — it is a bitmask
> read-modify-write of a single shared register under a manual-control bracket, which
> ESPHome's high-level `modbus_controller` cannot perform atomically or with a correct
> base. In testing this **stopped a live pool pump** (see §4). Relay control is therefore
> removed from this package; §7 documents how to add it correctly.

---

## 1. The NeoPool write/commit model

Writing a parameter register **does not take effect on its own.** NeoPool stages writes
and applies them only on an explicit commit. (Sources: official *Sugar Valley MODBUS
Register Description*, Welldana D30-190045; Tasmota `xsns_83_neopool.ino`; cross-checked on
a real Oxilife unit. See §8.)

| Register | Addr | Role |
|---|---|---|
| `MBF_EXEC` | `0x02F5` | Write `1` → apply staged writes to RAM **immediately** (live, volatile) |
| `MBF_SAVE_TO_EEPROM` | `0x02F0` | Write `1` → persist to EEPROM (survives reboot). Blocks the device <~1 s. EEPROM wear — use sparingly |
| `MBF_SET_MANUAL_CTRL` | `0x0289` | Write `1` **before** touching `MBF_RELAY_STATE`, `0` after — i.e. seize manual control of the relays |

**Setpoint write** = `write value register` → `MBF_EXEC=1` → *(optional)* `MBF_SAVE_TO_EEPROM=1`.

**Relay write** = `MBF_SET_MANUAL_CTRL=1` → `MBF_RELAY_STATE` (set/clear one bit, read-modify-write) → `MBF_SET_MANUAL_CTRL=0` → `MBF_EXEC=1`.

The relay write needs the extra manual-control bracket because a relay is an **output the
controller is actively driving** by its own automatic logic; you must explicitly take it
out of auto before you can override it. A setpoint is just a *target the controller
consults* — no seizure required.

---

## 2. The two-axis safety model

A write's safety has two **independent** axes. It is safe only when green on both.

**Axis 1 — Mechanical safety** (does the write operation corrupt state it shouldn't?):

| Property | Safe | Unsafe |
|---|---|---|
| Write style | Full-word (`value_type`, no `bitmask`) — writes exactly your value | Bitmask read-modify-write — depends on a cached base that can be stale/zero |
| Register | Dedicated (one function) | Shared (many functions packed as bits, e.g. `MBF_RELAY_STATE`) |
| Actuator seizure | None | Requires `MBF_SET_MANUAL_CTRL` |
| Atomicity | Single value + EXEC; interleaved *read* is benign | Multi-step bracket that must not be interrupted |

**Axis 2 — Functional / value safety** (is the *effect* harmful even if the write is clean?):
out-of-range value (e.g. hidro above nominal over-drives the cell; pH target too low
over-doses acid), pump-affecting actuation (filtration mode/state), or subsystem corruption
(calibration, relay-GPIO reassignment).

---

## 3. Why setpoints are safe

A setpoint write (pH `0x0504`/`0x0505`, Redox `0x0508`, Chlorine `0x050A`, Hydrolysis
`0x0502`):

- targets a **dedicated** register — it physically cannot touch the relay word (`0x010E`);
- is a **full-word** write — there is no shared-bit base to corrupt;
- needs **no** `MBF_SET_MANUAL_CTRL`;
- tolerates an interleaved read — the controller holds the staged value until EXEC.

So the only remaining risk is the **value** (Axis 2), which is bounded by clamping the
entity's `min_value`/`max_value`. **A setpoint write has no mechanism by which it can stop
the pump.** This is verified against the Tasmota driver (setpoint command paths write only
the value register + EXEC; they never touch `MBF_SET_MANUAL_CTRL` or `MBF_RELAY_STATE`).

---

## 4. Why relay control is unsafe in ESPHome (case study: a stopped pump)

This package originally shipped a single "Aux Relay" demo switch that wrote
`MBF_RELAY_STATE` via the bracket above, using an ESPHome `modbus_controller` switch with a
`bitmask`. **It stopped a live pool pump.** Root cause, verified by reading the controller's
own configuration registers on the affected Oxilife unit:

- Relay map (read back live): Relay 1 = acid (`MBF_PAR_PH_ACID_RELAY_GPIO 0x040A` = 1),
  **Relay 2 = filtration/pump** (`MBF_PAR_FILT_GPIO 0x0412` = 2), Relay 3 = lighting
  (`MBF_PAR_LIGHTING_GPIO 0x0410` = 3). The relay→function map is **per-installation
  configurable** — you cannot assume it from the register map.
- The Aux Relay switch targeted bit `0x0004` = **Relay 3 (lighting)** — yet toggling it
  stopped the **pump** (Relay 2). `MBF_RELAY_STATE` read back `0x42` (bits 1+6 → filtration
  ON), confirming Relay 2 is the pump.

**Mechanism.** ESPHome's `modbus_controller` `switch`/`number`/`output` with a `bitmask`
performs a **read-modify-write**: it takes a cached register value, applies the masked bit,
and writes the whole word back. Reads here use **FC4** and writes use **FC16** — they are
*different register classes*, kept in *separate caches*. The write-side base for `0x010E`
was never populated by a read, so the RMW wrote from a **stale/zero base**, setting bit 2
while clearing the **filtration bit (Relay 2)** → EXEC under manual-control → **pump off.**

This is **structural**, not a wrong-bit config error: *any* relay write through the
high-level bitmask path can knock out filtration regardless of which bit it targets.
Compounding it, ESPHome cannot suspend its polling around the bracket (§6), so a poll-read
can land mid-sequence.

> Aside: the same switch had no `restore_mode`, so it replayed this write at **every boot**,
> stopping the pump on every reset/OTA. That symptom was first mis-attributed to an RS-485
> hardware float; reading the unit's registers disproved it. See the consuming project's
> `ERRATA.md` (ERR-003).

---

## 5. Holistic classification of the NeoPool write surface

**🟢 SAFE — mechanically clean + value-bounded (cannot touch the pump):**
Setpoints — pH (`0x0504`/`0x0505`), Redox (`0x0508`), Chlorine (`0x050A`), Hydrolysis
(`0x0502`). Clamp the range (hydrolysis ≤ `MBF_PAR_HIDRO_NOM 0x0306`). **This is the only
write class this package exposes.**

**🟡 MECHANICALLY SAFE but functionally consequential — deliberate use only:**
Filtration mode (`0x0411`) and filtration manual-state (`0x0413`) — the *correct* way to
control the pump (clean full-word writes), but they *do* start/stop it. `MBF_SAVE_TO_EEPROM`
(`0x02F0`) — EEPROM wear + ~1 s stall; commit with EXEC-only (op 2) unless persistence is
needed. `MBF_ESCAPE` (`0x0297`), `COPY_TO_RTC` (`0x04F0`) — affect alarms/clock.

**🔴 UNSAFE in ESPHome's high-level API — do not expose:**
Anything through `MBF_SET_MANUAL_CTRL 0x0289` + `MBF_RELAY_STATE 0x010E` (all relay
control). See §4. Build it only via the low-level path in §7.

**🔴 UNSAFE — semantically destructive even though the write is "clean":**
`MBF_PAR_HIDRO_NOM 0x0306` (spec: *"DO NOT WRITE"*), relay-GPIO assignments
(`0x040A`–`0x0429`, commissioning-only), calibration registers (silently corrupt readings).

---

## 6. What ESPHome `modbus_controller` can and cannot do

Verified against `esphome/components/modbus_controller/modbus_controller.h` (`dev`) and the
component docs (§8):

**Can:**
- `ModbusController` is a `PollingComponent` with a public `void update()`. Setting
  `update_interval: never` stops automatic polling; reads can then be driven on demand
  (`component.update` action / `id(ctrl)->update()`), giving a polling-suspend window.
- Construct full-word / arbitrary writes from a lambda and enqueue them:
  ```cpp
  static ModbusCommandItem create_write_single_command(ModbusController*, uint16_t address, uint16_t value);
  static ModbusCommandItem create_write_multiple_command(ModbusController*, uint16_t start, uint16_t count, const std::vector<uint16_t>& values);
  static ModbusCommandItem create_custom_command(ModbusController*, const std::vector<uint8_t>& values, std::function<...> handler = nullptr);
  void queue_command(const ModbusCommandItem &command);
  ```

**Cannot (vs the Tasmota reference driver):**
- It does **not** auto-suspend polling around a write critical-section the way Tasmota does;
  reads and writes share **one** queue (`std::list<std::unique_ptr<ModbusCommandItem>>`),
  and there is no public pause/clear. You must choreograph quiescence yourself.
- The high-level `bitmask` write does an RMW from a possibly-stale cache (§4) — never use it
  for a shared register like `MBF_RELAY_STATE`.

---

## 7. Future plan — adding relay control *correctly*

Relay control is **achievable** in ESPHome, just not via the turnkey `switch`+`bitmask`
YAML. A safe implementation must:

1. **Quiesce polling** — set the controller `update_interval: never` (or guard the window);
   drive reads explicitly so none land inside the bracket.
2. **Read a fresh full word** of `MBF_RELAY_STATE` (FC4) immediately before writing.
3. **Compute the full word** in a lambda — preserve every other relay bit, set/clear only
   the target bit (never a bitmask RMW from cache).
4. **Write the full word** via `create_write_single_command(...)` + `queue_command(...)`.
5. **Bracket** with `MBF_SET_MANUAL_CTRL=1` → write → `MBF_SET_MANUAL_CTRL=0` → `MBF_EXEC=1`,
   spacing the steps (~30–60 ms) and keeping them strictly ordered through the single queue.
6. **Crash-safety** — if the device resets between `MANUAL_CTRL=1` and the closing
   `MANUAL_CTRL=0`+EXEC, the controller is left in manual-relay mode; design a
   reconcile-on-boot and bench-test the failure path.
7. **Resolve the relay number from config** — read `MBF_PAR_FILT_GPIO`/`*_GPIO` so you never
   assume which relay is the pump; refuse to touch the filtration relay.

Validate the write **infrastructure** on the 🟢 setpoint class first (it needs none of the
above), then add relay control as a deliberate, bench-tested feature.

---

## 8. Sources

- **Official protocol:** *Sugar Valley / Oxilife MODBUS Register Description*, Welldana
  document **D30-190045** (register names, addresses, `MBF_PAR_HIDRO_NOM` "DO NOT WRITE",
  manual-control/EXEC semantics).
- **Reference driver:** Tasmota `xsns_83_neopool.ino` —
  <https://github.com/arendst/Tasmota/blob/development/tasmota/tasmota_xsns_sensor/xsns_83_neopool.ino>
  (write/commit sequences; setpoint vs relay paths; polling suspension around writes).
- **ESPHome modbus_controller:** component docs
  <https://esphome.io/components/modbus_controller.html> and source
  `esphome/components/modbus_controller/modbus_controller.h` (`PollingComponent::update`,
  `ModbusCommandItem::create_write_*`, `queue_command`, single shared command queue).
- **This repo:** `REGISTERS.md` (distilled register guide + scaling); `neopool.yaml` (the
  implementation this document governs).
- **Empirical:** live register read-back from an Oxilife unit (relay map, `HIDRO_NOM`=160 →
  16.0 g/h) and the consuming project's `ERRATA.md` (ERR-003, the stopped-pump bisection).
