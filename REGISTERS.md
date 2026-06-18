# Sugar Valley NeoPool — Modbus register reference & write-endpoint guide

Everything needed to extend this package — especially to add **write** endpoints
(setpoints, modes, relays) later. Sourced from Tasmota `xsns_83_neopool.ino` and
the official Sugar Valley *MODBUS Register Description* (Welldana D30-190045), and
cross-checked against a verified real-unit (Oxilife) configuration.

> Branding note: the same controller ships as **Oxilife, Hidrolife, Aquascenic,
> Bionet, Hidroniser, UVScenic, Station, Brilix, Bayrol, Hay**. Machine type is in
> `MBF_PAR_UICFG_MACHINE` (0x0600); Oxilife = `3`.

---

## 1. Protocol basics

| Property | Value |
|---|---|
| Physical | RS-485 half-duplex (or direct TTL on some service ports) |
| Serial | **19200 baud, 8N1** |
| Slave address | `0x01` (default) |
| Register width | 16-bit "holding" words — addresses are **register numbers, not bytes** |
| Reads | **FC4** (`register_type: read`) — the driver reads everything with FC4, including `MBF_PAR_*` |
| Writes | **FC16 only** (`register_type: holding` + `use_write_multiple: true`). The controller's write handler is function code 0x10; single-register FC6 is not used by the reference driver — **always set `use_write_multiple: true`**. |
| 32-bit values | Stored as two consecutive registers, **low word first** → `value_type: U_DWORD` |
| Error handling | Reference driver **retries a failed write up to 2×**; mirror with sane retry/availability settings |

---

## 2. The write / commit procedure ⚠️ (read before adding any write endpoint)

Writing a parameter register **does not take effect on its own**. NeoPool stages
writes and applies them only on an explicit commit:

| Register | Addr | Meaning |
|---|---|---|
| `MBF_EXEC` | `0x02F5` | Write `1` → **apply staged writes to RAM immediately** (live, volatile) |
| `MBF_SAVE_TO_EEPROM` | `0x02F0` | Write `1` → **persist to EEPROM** (survives reboot). Device may be unresponsive <1 s. **Do NOT call on every change — EEPROM wear.** |
| `MBF_SET_MANUAL_CTRL` | `0x0289` | Write `1` to enable manual relay control before touching `MBF_RELAY_STATE`, `0` after |
| `MBF_ESCAPE` | `0x0297` | Write (any) → same as the front-panel ESC button; clears an active alarm |
| `MBF_ACTION_COPY_TO_RTC` | `0x04F0` | Write (any) → commit `MBF_PAR_TIME_*` into the RTC |

**Setpoint write pattern** (pH/redox/chlorine/hydrolysis):
1. Write the value register (e.g. `0x0504 = 740` for pH 7.40)
2. Write `MBF_EXEC` (`0x02F5`) = 1 → applies live
3. *(optional, sparingly)* Write `MBF_SAVE_TO_EEPROM` (`0x02F0`) = 1 → persists

**Relay write pattern** (`MBF_RELAY_STATE` 0x010E):
1. `MBF_SET_MANUAL_CTRL` (0x0289) = 1
2. set/clear the target bit in `MBF_RELAY_STATE` (0x010E) — read-modify-write
3. `MBF_SET_MANUAL_CTRL` (0x0289) = 0
4. `MBF_EXEC` (0x02F5) = 1

> This package implements the relay pattern for one relay (see `neopool.yaml`
> → the single queued `np_write` script). It is the reference for all other writes.

### Concurrency, timing & traps (audited against the reference driver)

- **FC16 is mandatory** for every write (see §1). `use_write_multiple: true`.
- **The reference driver suspends its own polling for the whole value→EXEC(→SAVE)
  critical section.** ESPHome cannot do that — its `modbus_controller` shares one
  command queue for reads and writes. ESPHome *does* serialize the bus (one
  request/response at a time), so writes never physically collide with reads, but
  a periodic **read can land between your value write and the EXEC**.
  - *Setpoints:* benign — the controller holds the staged value until EXEC, so an
    interleaved read does not lose it.
  - *Relay:* the `MANUAL_CTRL=1 → RELAY_STATE → MANUAL_CTRL=0 → EXEC` bracket is
    order-sensitive. An interleaved **write** (e.g. a setpoint EXEC fired at the
    same instant) could land inside the bracket. Don't drive the relay and a
    setpoint in the same tick; treat the relay write as **experimental** until
    bench-verified.
- **Setpoints should persist:** the reference driver writes value→EXEC→**SAVE**.
  EXEC-only applies live but is lost on the controller's next power cycle. Use the
  `np_write` op 3 (EXEC + SAVE) for setpoints, op 2 (EXEC only) for transient mode changes.
- **`SAVE_TO_EEPROM` blocks the controller for up to ~1 s** (it busy-waits on
  `MBF_NOTIFICATION` to come back). Expect a one-off read timeout / availability
  blip after each save. **Never** put SAVE on a frequently-changing value.
- **Settle timing:** the driver allows ~2 ms after a write and `NEOPOOL_READ_TIMEOUT`
  = 25 ms per transaction. The package's commit scripts use 30–60 ms spacing.

---

## 3. Scaling cheat-sheet

| Quantity | Register | Raw → real | ESPHome filter |
|---|---|---|---|
| pH (measure & setpoints) | 0x0102 / 0x0504 / 0x0505 | ÷100 | `multiply: 0.01` |
| Redox / ORP | 0x0103 / 0x0508 | ×1 (mV) | — |
| Chlorine | 0x0104 / 0x050A | ÷100 (ppm) | `multiply: 0.01` |
| Conductivity / salt | 0x0105 | ×1 (%) | — |
| Temperature | 0x0106 / setpoints | ÷10 (°C) | `multiply: 0.1` |
| Hydrolysis production* | 0x0101 / 0x0502 | ÷10 (g/h) **on g/h-mode units** | `multiply: 0.1` |
| Cell voltage | 0x0111 | ÷10 (V) *(verify)* | `multiply: 0.1` |

\* Hydrolysis is **% or g/h** depending on `MBF_PAR_HIDRO_NOM` (0x0306). If your
unit is %-mode, drop the ÷10 and use `%`. Never write above `MBF_PAR_HIDRO_NOM`.

---

## 4. Write-endpoint recipes (ESPHome)

The package already provides the single serialized writer **`np_write`** (the only
thing that issues Modbus writes; `mode: queued` so sequences never interleave) and
the internal `np_exec` / `np_save` helpers. Every write control just calls it:

```yaml
on_value:
  - script.execute:
      id: np_write
      op: 3        # 0/1 = relay off/on · 2 = EXEC only · 3 = EXEC + SAVE (persist)
```

Use **op 3** for setpoints (they should persist) and **op 2** for transient mode
changes. Recipes below show the value entities; pair each with the call above.

### 4.1 Setpoint `number`s (pH high, redox, chlorine, hydrolysis)

```yaml
number:
  - platform: modbus_controller
    modbus_controller_id: oxilife
    name: "Pool pH Setpoint High"
    address: 0x0504          # MBF_PAR_PH1  (must stay > MBF_PAR_PH2)
    register_type: holding
    value_type: U_WORD
    use_write_multiple: true
    multiply: 100            # 7.40 -> 740
    min_value: 6.5
    max_value: 8.0
    step: 0.05
    mode: box
    on_value:
      - script.execute:
          id: np_write
          op: 3

  - platform: modbus_controller
    modbus_controller_id: oxilife
    name: "Pool Redox Setpoint"
    address: 0x0508          # MBF_PAR_RX1  (range 0..1000 mV)
    register_type: holding
    value_type: U_WORD
    use_write_multiple: true
    min_value: 0
    max_value: 1000
    step: 5
    on_value:
      - script.execute:
          id: np_write
          op: 3

  - platform: modbus_controller
    modbus_controller_id: oxilife
    name: "Pool Chlorine Setpoint"
    address: 0x050A          # MBF_PAR_CL1  (range 0..1000 -> 0..10 ppm)
    register_type: holding
    value_type: U_WORD
    use_write_multiple: true
    multiply: 100            # 1.50 ppm -> 150
    min_value: 0
    max_value: 10
    step: 0.1
    on_value:
      - script.execute:
          id: np_write
          op: 3

  - platform: modbus_controller
    modbus_controller_id: oxilife
    name: "Pool Electrolysis Setpoint"
    address: 0x0502          # MBF_PAR_HIDRO (g/h mode here, /10)
    register_type: holding
    value_type: U_WORD
    use_write_multiple: true
    multiply: 10             # 12.0 g/h -> 120
    min_value: 0
    max_value: 100           # clamp to your MBF_PAR_HIDRO_NOM!
    step: 1
    on_value:
      - script.execute:
          id: np_write
          op: 3
```

### 4.2 Filtration mode `select`

```yaml
select:
  - platform: modbus_controller
    modbus_controller_id: oxilife
    name: "Pool Filtration Mode"
    address: 0x0411          # MBF_PAR_FILT_MODE (select has no register_type)
    use_write_multiple: true
    optionsmap:
      "Manual": 0            # MBV_PAR_FILT_*
      "Auto": 1
      "Heating": 2
      "Smart": 3
      "Intelligent": 4
      "Backwash": 13
    on_value:
      - script.execute:
          id: np_write
          op: 3
```

For **manual filtration on/off**: set mode to Manual (0), write
`MBF_PAR_FILT_MANUAL_STATE` (0x0413) = 0/1, then commit. (`MBF_PAR_FILTRATION_STATE`
0x0421 is the *resulting* state.)

### 4.3 Boost `switch` (`MBF_CELL_BOOST` 0x020C)

Boost is a masked register, not a simple bit — write whole-word values:
`0x0000` off · `0x05A0` boost on (`STATE`|`START`) · `0x85A0` boost on with redox
control disabled (`|NO_REDOX_CTL`). Use a `number`/`output` with `write_lambda`
to emit the exact word, then call `np_write` (op 2/3).

### 4.4 Relay `switch` — see `neopool.yaml`

The `${neopool_name} Aux Relay` switch + the `np_write` script (op 0/1) implement the
full manual-control + EXEC relay sequence. Change `neopool_aux_relay_bit` to pick
the relay. **Bit map (typical factory):** see §6.

---

## 5. Value enums

**Filtration mode** (`MBF_PAR_FILT_MODE` 0x0411): 0 Manual · 1 Auto · 2 Heating ·
3 Smart · 4 Intelligent · 13 Backwash.

**pH dosing config** (`MBF_PAR_RELAY_PH` 0x0430): 0 acid+base · 1 acid only · 2 base only.

**Boost mask** (`MBF_CELL_BOOST` 0x020C): `STATE`=0x0500 · `START`=0x00A0 ·
`NO_REDOX_CTL`=0x8000.

**Machine** (`MBF_PAR_UICFG_MACHINE` 0x0600): 1 Hidrolife · 2 Aquascenic ·
**3 Oxilife** · 4 Bionet · 5 Hidroniser · 6 UVScenic · 7 Station · 8 Brilix ·
9 Generic · 10 Bayrol · 11 Hay.

**Notification page bits** (`MBF_NOTIFICATION` 0x0110): 0x01 modbus · 0x02 global ·
0x04 factory · 0x08 installer · 0x10 user · 0x20 misc.

---

## 6. Relay bit map — `MBF_RELAY_STATE` (0x010E)

Bit → relay. **Assignment is configurable** via the `*_GPIO` registers below;
these are the typical factory defaults. Confirm yours by reading the GPIO regs.

| Bit | Mask | Typical function |
|---|---|---|
| 0 | 0x0001 | **Relay 1 — pH acid pump** ⚠️ |
| 1 | 0x0002 | Relay 2 — filtration |
| 2 | 0x0004 | Relay 3 — lighting |
| 3 | 0x0008 | Relay 4 — aux |
| 4 | 0x0010 | Relay 5 — aux |
| 5 | 0x0020 | Relay 6 — aux |
| 6 | 0x0040 | Relay 7 — aux/heating |
| 8–10 | 0x0700 | Filtration speed (low/mid/high) |

**Relay assignment registers:** `MBF_PAR_PH_ACID_RELAY_GPIO` 0x040A ·
`MBF_PAR_PH_BASE_RELAY_GPIO` 0x040B · `MBF_PAR_RX_RELAY_GPIO` 0x040C ·
`MBF_PAR_CL_RELAY_GPIO` 0x040D · `MBF_PAR_LIGHTING_GPIO` 0x0410 ·
`MBF_PAR_FILT_GPIO` 0x0412 (dflt relay 2) · `MBF_PAR_HEATING_GPIO` 0x0415 (dflt relay 7).
Relay GPIO *n* → bit *n-1*.

---

## 7. Status bitmasks (decode into `binary_sensor`s)

`MBF_HIDRO_STATUS` (0x010D): 0x0001 on-target · 0x0002 low · 0x0008 FL1 flow ·
0x0010 cover · 0x0020 module active · 0x0040 ctrl active · 0x0080 redox-enabled ·
0x0100 shock · 0x0200 FL2 · 0x0400 cl-enabled · 0x1000/0x2000/0x4000 polarity.

`MBF_PH_STATUS` (0x0107): 0x000F alarm · 0x0400 ctrl-by-flow · 0x0800 acid pump ·
0x1000 base pump · 0x2000 ctrl active · 0x4000 measure active · 0x8000 module present.

`MBF_RX_STATUS` (0x0108): 0x0007 alarm · 0x0080 redox too low · 0x1000 pump ·
0x2000 ctrl active · 0x4000 measure active · 0x8000 module present.

`MBF_CL_STATUS` (0x0109): 0x0008 probe flow · 0x1000 pump · 0x2000 ctrl ·
0x4000 measure · 0x8000 module present.

`MBF_CD_STATUS` (0x010A) & `MBF_ION_STATUS` (0x010C): same 0x1000–0x8000 pattern;
ION adds 0x0001 on-target / 0x0002 low / 0x0008 prog-time-exceeded.

---

## 8. Full register table (154 registers)

`!` = internal/diagnostic in Tasmota. `(DO NOT WRITE!)` = factory/calibration.

| Register | Addr | Description |
|---|---|---|
| `MBF_POWER_MODULE_VERSION` | 0x0002 | ! Power module version (MSB=Major, LSB=Minor) |
| `MBF_POWER_MODULE_NODEID` | 0x0004 | ! Power module Node ID (6 register 0x0004 - 0x0009) |
| `MBF_POWER_MODULE_REGISTER` | 0x000C | ! Writing an address in this register causes the power module register address to be read out into MBF_POWER_MODULE_DATA, see MBF_POWER_MODULE_REG_* |
| `MBF_POWER_MODULE_DATA` | 0x000D | ! power module data as requested in MBF_POWER_MODULE_REGISTER |
| `MBF_VOLT_24_36` | 0x0022 | ! Current 24-36V line in mV |
| `MBF_VOLT_12` | 0x0023 | ! Current 12V line in mV |
| `MBF_VOLT_5` | 0x006A | ! 5V line in mV / 0,62069 |
| `MBF_AMP_4_20_MICRO` | 0x0072 | ! 2-40mA line in µA * 10 (1=0,01mA) |
| `MBF_ION_CURRENT` | 0x0100 | Ionization level measured |
| `MBF_HIDRO_CURRENT` | 0x0101 | Hydrolysis intensity level |
| `MBF_MEASURE_PH` | 0x0102 | ph     pH level measured in 1/100 (700 = 7.00) |
| `MBF_MEASURE_RX` | 0x0103 | mV     Redox level measured in mV |
| `MBF_MEASURE_CL` | 0x0104 | ppm    Chlorine level measured in 1/100 ppm (100 = 1.00 ppm) |
| `MBF_MEASURE_CONDUCTIVITY` | 0x0105 | %      Conductivity level measured in % |
| `MBF_MEASURE_TEMPERATURE` | 0x0106 | °C     Temperature sensor measured in 1/10° C (100 = 10.0°C) |
| `MBF_PH_STATUS` | 0x0107 | mask   Status of the pH-module |
| `MBF_RX_STATUS` | 0x0108 | mask   Status of the Rx-module |
| `MBF_CL_STATUS` | 0x0109 | mask   Status of the Chlorine-module |
| `MBF_CD_STATUS` | 0x010A | mask   Status of the Conductivity-module |
| `MBF_ION_STATUS` | 0x010C | mask   Status of the Ionization-module |
| `MBF_HIDRO_STATUS` | 0x010D | mask   Status of the Hydrolysis-module |
| `MBF_RELAY_STATE` | 0x010E | mask   Status of each configurable relay |
| `MBF_HIDRO_SWITCH_VALUE` | 0x010F | INTERNAL - contains the opening of the hydrolysis PWM. |
| `MBF_NOTIFICATION` | 0x0110 | mask   Bit field that informs whether a property page has changed since the last time it was queried. (see MBMSK_NOTIF_*). This register makes it possible to... |
| `MBF_HIDRO_VOLTAGE` | 0x0111 | The voltage applied to the hydrolysis cell. This register, together with that of MBF_HIDRO_CURRENT allows extrapolation of water salinity. |
| `MBF_CELL_RUNTIME_LOW` | 0x0206 | ! Cell runtime (32 bit value - low word) |
| `MBF_CELL_RUNTIME_HIGH` | 0x0207 | ! Cell runtime (32 bit value - high word) |
| `MBF_CELL_RUNTIME_PART_LOW` | 0x0208 | ! Cell part runtime (32 bit value - low word) |
| `MBF_CELL_RUNTIME_PART_HIGH` | 0x0209 | ! Cell part runtime (32 bit value - high word) |
| `MBF_CELL_BOOST` | 0x020C | mask   ! Boost control (see MBMSK_CELL_BOOST_*) |
| `MBF_CELL_RUNTIME_POLA_LOW` | 0x0214 | ! Cell runtime polarity 1 (32 bit value - low word) |
| `MBF_CELL_RUNTIME_POLA_HIGH` | 0x0215 | ! Cell runtime polarity 1 (32 bit value - high word) |
| `MBF_CELL_RUNTIME_POLB_LOW` | 0x0216 | ! Cell runtime polarity 2 (32 bit value - low word) |
| `MBF_CELL_RUNTIME_POLB_HIGH` | 0x0217 | ! Cell runtime polarity 2 (32 bit value - high word) |
| `MBF_CELL_RUNTIME_POL_CHANGES_LOW` | 0x0218 | ! Cell runtime polarity change count (32 bit value - low word) |
| `MBF_CELL_RUNTIME_POL_CHANGES_HIGH` | 0x0219 | ! Cell runtime polarity change count (32 bit value - high word) |
| `MBF_HIDRO_MODULE_VERSION` | 0x0280 | ! Hydrolysis module version |
| `MBF_HIDRO_MODULE_CONNECTIVITY` | 0x0281 | ! Hydrolysis module connection quality (in myriad: 0..10000) |
| `MBF_SET_COUNTDOWN_KEY_AUX_OFF` | 0x0287 | mask   ! switch AUX1-4 OFF - only for AUX which is assigned as AUX and set to MBV_PAR_CTIMER_COUNTDOWN_KEY_* mode  - see MBV_PAR_CTIMER_COUNTDOWN_KEY_AUX* |
| `MBF_SET_COUNTDOWN_KEY_AUX_ON` | 0x0288 | mask   ! switch AUX1-4 ON  - only for AUX which is assigned as AUX and set to MBV_PAR_CTIMER_COUNTDOWN_KEY_* mode - see MBV_PAR_CTIMER_COUNTDOWN_KEY_AUX* |
| `MBF_SET_MANUAL_CTRL` | 0x0289 | ! write a 1 before manual control MBF_RELAY_STATE, after done write 0 and do MBF_EXEC |
| `MBF_ESCAPE` | 0x0297 | ! A write operation to this register is the same as using the ESC button on main screen - clears error |
| `MBF_SAVE_TO_EEPROM` | 0x02F0 | A write operation to this register immediately starts a EEPROM storage operation. During the EEPROM storage procedure, the system may be unresponsive to MODB... |
| `MBF_EXEC` | 0x02F5 | ! A write operation to this register immediately take over settings of the previous written data |
| `MBF_PAR_VERSION` | 0x0300 | Software version of the PowerBox |
| `MBF_PAR_MODEL` | 0x0301 | mask   System model options |
| `MBF_PAR_SERNUM` | 0x0302 | Serial number of the PowerBox |
| `MBF_PAR_ION_NOM` | 0x0303 | Ionization maximum production level (DO NOT WRITE!) |
| `MBF_PAR_HIDRO_NOM` | 0x0306 | Hydrolysis maximum production level. (DO NOT WRITE!) If the hydrolysis is set to work in percent mode, this value will be 100. If the hydrolysis module is se... |
| `MBF_PAR_SAL_AMPS` | 0x030A | Current command in regulation for which we are going to measure voltage |
| `MBF_PAR_SAL_CELLK` | 0x030B | Specifies the relationship between the resistance obtained in the measurement process and its equivalence in g / l (grams per liter) |
| `MBF_PAR_SAL_TCOMP` | 0x030C | Specifies the deviation in temperature from the conductivity. |
| `MBF_PAR_HIDRO_MAX_VOLTAGE` | 0x0322 | Allows setting the maximum voltage value that can be reached with the hydrolysis current regulation. The value is specified in tenths of a volt. The default ... |
| `MBF_PAR_HIDRO_FLOW_SIGNAL` | 0x0323 | Allows to select the operation of the flow detection signal associated with the operation of the hydrolysis (see MBV_PAR_HIDRO_FLOW_SIGNAL*). The default val... |
| `MBF_PAR_HIDRO_MAX_PWM_STEP_UP` | 0x0324 | This register sets the PWM ramp up of the hydrolysis in pulses per duty cycle. This register makes it possible to adjust the rate at which the power delivere... |
| `MBF_PAR_HIDRO_MAX_PWM_STEP_DOWN` | 0x0325 | This register sets the PWM down ramp of the hydrolysis in pulses per duty cycle. This register allows adjusting the rate at which the power delivered to the ... |
| `MBF_PAR_ION_POL0` | 0x0400 | Time in minutes that the equipment must remain working in positive polarization in the copper-silver ionization. |
| `MBF_PAR_ION_POL1` | 0x0401 | Time in minutes that the equipment must remain working in negative polarization in the copper-silver ionization. |
| `MBF_PAR_ION_POL2` | 0x0402 | Time in minutes that the equipment must remain working in dead time (without delivering power) in the copper-silver ionization. |
| `MBF_PAR_HIDRO_ION_CAUDAL` | 0x0403 | mask   Bitmask register regulates the external control mode of ionization, hydrolysis and pumps (see MBMSK_HIDRO_ION_CAUDAL_*) |
| `MBF_PAR_HIDRO_MODE` | 0x0404 | Regulates the external control mode of hydrolysis from the modules of measure. 0 = no control, 1 = standard control (on / off), 2 = with timed pump |
| `MBF_PAR_HIDRO_POL0` | 0x0405 | Time in minutes that the equipment must remain in positive polarization during hydrolysis/electrolysis. |
| `MBF_PAR_HIDRO_POL1` | 0x0406 | Time in minutes that the equipment must remain in negative polarization during hydrolysis/electrolysis. |
| `MBF_PAR_HIDRO_POL2` | 0x0407 | Time in minutes that the equipment must remain working in dead time (without delivering power) in the hydrolysis/electrolysis. |
| `MBF_PAR_TIME_LOW` | 0x0408 | System timestamp as unix timestamp (32 bit value - low word). |
| `MBF_PAR_TIME_HIGH` | 0x0409 | System timestamp as unix timestsmp (32 bit value - high word). |
| `MBF_PAR_PH_ACID_RELAY_GPIO` | 0x040A | Relay number assigned to the acid pump function (only with pH module). |
| `MBF_PAR_PH_BASE_RELAY_GPIO` | 0x040B | Relay number assigned to the base pump function (only with pH module). |
| `MBF_PAR_RX_RELAY_GPIO` | 0x040C | Relay number assigned to the Redox level regulation function. If the value is 0, there is no relay assigned, and therefore there is no pump function (ON / OF... |
| `MBF_PAR_CL_RELAY_GPIO` | 0x040D | Relay number assigned to the chlorine pump function (only with free chlorine measuring modules). |
| `MBF_PAR_CD_RELAY_GPIO` | 0x040E | Relay number assigned to the conductivity (brine) pump function (only with conductivity measurement modules). |
| `MBF_PAR_TEMPERATURE_ACTIVE` | 0x040F | Indicates whether the equipment has a temperature measurement or not. |
| `MBF_PAR_LIGHTING_GPIO` | 0x0410 | Relay number assigned to the lighting function. 0: inactive. |
| `MBF_PAR_FILT_MODE` | 0x0411 | Filtration mode (see MBV_PAR_FILT_*) |
| `MBF_PAR_FILT_GPIO` | 0x0412 | Relay selected to perform the filtering function (by default it is relay 2). When this value is at zero, there is no relay assigned and therefore it is under... |
| `MBF_PAR_FILT_MANUAL_STATE` | 0x0413 | Filtration status in manual mode (on = 1; off = 0) |
| `MBF_PAR_HEATING_MODE` | 0x0414 | Heating mode: 0 = the equipment is not heated. 1 = the equipment is heating. |
| `MBF_PAR_HEATING_GPIO` | 0x0415 | Relay number assigned to perform the heating function (by default it is relay 7). When this value is at zero, there is no relay assigned and therefore it is ... |
| `MBF_PAR_HEATING_TEMP` | 0x0416 | Heating mode: Heating setpoint temperature |
| `MBF_PAR_CLIMA_ONOFF` | 0x0417 | Activation of the climate mode (0 = inactive, 1 = active). |
| `MBF_PAR_SMART_TEMP_HIGH` | 0x0418 | Smart mode: Upper temperature |
| `MBF_PAR_SMART_TEMP_LOW` | 0x0419 | Smart mode: Lower temperature |
| `MBF_PAR_SMART_ANTI_FREEZE` | 0x041A | Smart mode: Antifreeze mode activated (1) or not (0). |
| `MBF_PAR_SMART_INTERVAL_REDUCTION` | 0x041B | Smart mode: This register is read-only and reports to the outside what percentage (0 to 100%) is being applied to the nominal filtration time. 100% means tha... |
| `MBF_PAR_INTELLIGENT_TEMP` | 0x041C | Intelligent mode: Setpoint temperature |
| `MBF_PAR_INTELLIGENT_FILT_MIN_TIME` | 0x041D | Intelligent mode: Minimum filtration time in minutes |
| `MBF_PAR_INTELLIGENT_BONUS_TIME` | 0x041E | Intelligent mode: Bonus time for the current set of intervals |
| `MBF_PAR_INTELLIGENT_TT_NEXT_INTERVAL` | 0x041F | Intelligent mode: Time to next filtration interval. When it reaches 0 an interval is started and the number of seconds is reloaded for the next interval (2x3... |
| `MBF_PAR_INTELLIGENT_INTERVALS` | 0x0420 | Intelligent mode: Number of started intervals. When it reaches 12 it is reset to 0 and the bonus time is reloaded with the value of MBF_PAR_INTELLIGENT_FILT_... |
| `MBF_PAR_FILTRATION_STATE` | 0x0421 | Filtration state: 0 is off and 1 is on. The filtration state is regulated according to the MBF_PAR_FILT_MANUAL_STATE register if the filtration mode held in ... |
| `MBF_PAR_HEATING_DELAY_TIME` | 0x0422 | Timer in seconds that counts up when the heating is to be enabled. Once this counter reaches 60 seconds, the heating is then enabled. This counter is for int... |
| `MBF_PAR_FILTERING_TIME_LOW` | 0x0423 | Internal timer for the intelligent filtering mode (32-bit value - low word). It counts the filtering time done during a given day. This register is only for ... |
| `MBF_PAR_FILTERING_TIME_HIGH` | 0x0424 | Internal timer for the intelligent filtering mode (32-bit value - high word) |
| `MBF_PAR_INTELLIGENT_INTERVAL_TIME_LOW` | 0x0425 | Internal timer that counts the filtration interval assigned to the the intelligent mode (32-bit value - low word). This register is only for internal use and... |
| `MBF_PAR_INTELLIGENT_INTERVAL_TIME_HIGH` | 0x0426 | Internal timer that counts the filtration interval assigned to the the intelligent mode (32-bit value - high word) |
| `MBF_PAR_UV_MODE` | 0x0427 | UV mode active or not - see MBV_PAR_UV_MODE*. To enable UV support for a given device, add the mask MBMSK_MODEL_UV to the MBF_PAR_MODEL register. |
| `MBF_PAR_UV_HIDE_WARN` | 0x0428 | mask   Suppression for warning messages in the UV mode (see MBMSK_UV_HIDE_WARN_*) |
| `MBF_PAR_UV_RELAY_GPIO` | 0x0429 | Relay number assigned to the UV function. |
| `MBF_PAR_PH_PUMP_REP_TIME_ON` | 0x042A | mask   Time that the pH pump will be turn on in the repetitive mode (see MBMSK_PH_PUMP_*). Contains a special time format, see desc for MBMSK_PH_PUMP_TIME. |
| `MBF_PAR_PH_PUMP_REP_TIME_OFF` | 0x042B | mask   Time that the pH pump will be turn off in the repetitive mode. Contains a special time format, see desc for MBMSK_PH_PUMP_TIME, has no upper configura... |
| `MBF_PAR_HIDRO_COVER_ENABLE` | 0x042C | mask   Options for the hydrolysis/electrolysis module (see MBMSK_HIDRO_*) |
| `MBF_PAR_HIDRO_COVER_REDUCTION` | 0x042D | Configured levels for the cover reduction and the hydrolysis shutdown temperature options: LSB = Percentage for the cover reduction, MSB = Temperature level ... |
| `MBF_PAR_PUMP_RELAY_TIME_OFF` | 0x042E | Time level in minutes or seconds that the dosing pump must remain off when the temporized pump mode is selected. This time level register applies to all pump... |
| `MBF_PAR_PUMP_RELAY_TIME_ON` | 0x042F | Time level in minutes or seconds that the dosing pump must remain on when the temporized pump mode is selected. This time level register applies to all pumps... |
| `MBF_PAR_RELAY_PH` | 0x0430 | Determine what pH regulation configuration the equipment has (see MBV_PAR_RELAY_PH_*) |
| `MBF_PAR_RELAY_MAX_TIME` | 0x0431 | Maximum amount of time in seconds, that a dosing pump can operate before rising an alarm signal. The behavior of the system when the dosing time is exceeded ... |
| `MBF_PAR_RELAY_MODE` | 0x0432 | Behavior of the system when the dosing time is exceeded (see MBMSK_PAR_RELAY_MODE_* and MBV_PAR_RELAY_MODE_*) |
| `MBF_PAR_RELAY_ACTIVATION_DELAY` | 0x0433 | Delay time in seconds for the pH pump when the measured pH value is outside the allowable pH setpoints. The system internally adds an extra time of 10 second... |
| `MBF_PAR_TIMER_BLOCK_BASE` | 0x0434 | Block of 180 registers containing the configuration of the 12 timers with 15 registers each (see MBV_TIMER_OFFMB_*). The system has a set of 12 fully configu... |
| `MBF_PAR_TIMER_BLOCK_FILT_INT1` | 0x0434 | Filtration timer 1 |
| `MBF_PAR_TIMER_BLOCK_FILT_INT2` | 0x0443 | Filtration tiemr 2 |
| `MBF_PAR_TIMER_BLOCK_FILT_INT3` | 0x0452 | Filtration timer 3 |
| `MBF_PAR_TIMER_BLOCK_AUX1_INT2` | 0x0461 | Aux1 timer 2 |
| `MBF_PAR_TIMER_BLOCK_LIGHT_INT` | 0x0470 | Lighting timer |
| `MBF_PAR_TIMER_BLOCK_AUX2_INT2` | 0x047F | Aux2 timer 2 |
| `MBF_PAR_TIMER_BLOCK_AUX3_INT2` | 0x048E | Aux3 timer 2 |
| `MBF_PAR_TIMER_BLOCK_AUX4_INT2` | 0x049D | Aux4 timer 2 |
| `MBF_PAR_TIMER_BLOCK_AUX1_INT1` | 0x04AC | Aux1 timer 1 |
| `MBF_PAR_TIMER_BLOCK_AUX2_INT1` | 0x04BB | Aux2 timer 1 |
| `MBF_PAR_TIMER_BLOCK_AUX3_INT1` | 0x04CA | Aux3 timer 1 |
| `MBF_PAR_TIMER_BLOCK_AUX4_INT1` | 0x04D9 | Aux4 timer 1 |
| `MBF_PAR_FILTVALVE_ENABLE` | 0x04E8 | Filter cleaning functionality mode (0 = off, 1 = Besgo) |
| `MBF_PAR_FILTVALVE_MODE` | 0x04E9 | Filter cleaning valve timing mode, possible modes: MBV_PAR_CTIMER_ENABLED, MBV_PAR_CTIMER_ALWAYS_ON, MBV_PAR_CTIMER_ALWAYS_OFF |
| `MBF_PAR_FILTVALVE_GPIO` | 0x04EA | Relay associated with the filter cleaning function. default AUX2 (value 5) |
| `MBF_PAR_FILTVALVE_START_LOW` | 0x04EB | Start timestamp of filter cleaning (32-bit value - low word) |
| `MBF_PAR_FILTVALVE_START_HIGH` | 0x04EC | Start timestamp of filter cleaning (32-bit value - high word) |
| `MBF_PAR_FILTVALVE_PERIOD_MINUTES` | 0x04ED | Period in minutes between cleaning actions. For example, if a value of 60 is stored in this registry, a cleanup action will occur every hour. |
| `MBF_PAR_FILTVALVE_INTERVAL` | 0x04EE | Cleaning action duration in seconds. |
| `MBF_PAR_FILTVALVE_REMAINING` | 0x04EF | Time remaining for the current cleaning action in seconds. If this register is 0, it means that there is no cleaning function running. When a cleanup functio... |
| `MBF_ACTION_COPY_TO_RTC` | 0x04F0 | A write (any value) forces the writing of the RTC time registers MBF_PAR_TIME_LOW (0x0408) and MBF_PAR_TIME_HIGH (0x0409) into the RTC internal microcontroll... |
| `MBF_PAR_ION` | 0x0500 | Ionization target production level. The value adjusted in this register must not exceed the value set in the MBF_PAR_ION_NOM factory register. |
| `MBF_PAR_ION_PR` | 0x0501 | Amount of time in minutes that the ionization must be activated each time that the filtration starts. |
| `MBF_PAR_HIDRO` | 0x0502 | Hydrolisis target production level. When the hydrolysis production is to be set in percent values, this value will contain the percent of production. If the ... |
| `MBF_PAR_PH1` | 0x0504 | Higher limit of the pH regulation system. The value set in this register is multiplied by 100. This means that if we want to set a value of 7.5, the numerica... |
| `MBF_PAR_PH2` | 0x0505 | Lower limit of the pH regulation system. The value set in this register is multiplied by 100. This means that if we want to set a value of 7.0, the numerical... |
| `MBF_PAR_RX1` | 0x0508 | Set point for the redox regulation system. This value must be in the range of 0 to 1000. |
| `MBF_PAR_CL1` | 0x050A | Set point for the chlorine regulation system. The value stored in this register is multiplied by 100. This mean that if we want to set a value of 1.5 ppm, we... |
| `MBF_PAR_FILTRATION_CONF` | 0x050F | mask   ! filtration type and speed, see MBMSK_PAR_FILTRATION_CONF_* |
| `MBF_PAR_FILTRATION_SPEED_FUNC` | 0x0513 | ! filtration speed function control |
| `MBF_PAR_FUNCTION_DEPENDENCY` | 0x051B | mask   Specification for the dependency of different functions, such as heating, from external events like FL1 (see MBMSK_FCTDEP_HEATING/MBMSK_DEPENDENCY_*) |
| `MBF_PAR_UICFG_MACHINE` | 0x0600 | Machine type (see MBV_PAR_MACH_* and  kNeoPoolMachineNames[]) |
| `MBF_PAR_UICFG_LANGUAGE` | 0x0601 | Selected language (see MBV_PAR_LANG_*) |
| `MBF_PAR_UICFG_BACKLIGHT` | 0x0602 | Display backlight brightness (in %, upper part (8-bit MSB)=0-100) and function (lower part 8-bit LSB, see MBV_PAR_BACKLIGHT_*) |
| `MBF_PAR_UICFG_SOUND` | 0x0603 | mask   Audible alerts (see MBMSK_PAR_SOUND_*) |
| `MBF_PAR_UICFG_PASSWORD` | 0x0604 | System password encoded in BCD |
| `MBF_PAR_UICFG_VISUAL_OPTIONS` | 0x0605 | mask   Stores the different display options for the user interface menus (bitmask). Some bits allow you to hide options that are normally visible (bits 0 to ... |
| `MBF_PAR_UICFG_VISUAL_OPTIONS_EXT` | 0x0606 | mask   This register stores additional display options for the user interface menus (see MBMSK_VOE_*) |
| `MBF_PAR_UICFG_MACH_VISUAL_STYLE` | 0x0607 | mask   This register is an expansion of register MBF_PAR_UICFG_MACHINE and MBF_PAR_UICFG_VISUAL_OPTIONS. If MBF_PAR_UICFG_MACHINE is MBV_PAR_MACH_GENERIC the... |
| `MBF_PAR_UICFG_MACH_NAME_BOLD` | 0x0608 | Machine name bold part title displayed during startup if MBF_PAR_UICFG_MACHINE is MBV_PAR_MACH_GENERIC. Note: Only lowercase letters (a-z) can be used. 4 reg... |
| `MBF_PAR_UICFG_MACH_NAME_LIGHT` | 0x060C | Machine name normal intensity part title displayed during startup if MBF_PAR_UICFG_MACHINE is MBV_PAR_MACH_GENERIC. Note: Only lowercase letters (a-z) can be... |
| `MBF_PAR_UICFG_MACH_NAME_AUX1` | 0x0610 | Aux1 relay name: 5 register ASCIIZ string with up to 10 characters |
| `MBF_PAR_UICFG_MACH_NAME_AUX2` | 0x0615 | Aux2 relay name: 5 register ASCIIZ string with up to 10 characters |
| `MBF_PAR_UICFG_MACH_NAME_AUX3` | 0x061A | Aux3 relay name: 5 register ASCIIZ string with up to 10 characters |
| `MBF_PAR_UICFG_MACH_NAME_AUX4` | 0x061F | Aux4 relay name: 5 register ASCIIZ string with up to 10 characters |

---

## 9. Safety

- Writes drive **real chemical dosing and pumps**. Validate each new write endpoint
  against your unit with `min_value`/`max_value` clamps before trusting automations.
- Never write `MBF_PAR_*_NOM` (factory maxima) or registers marked `(DO NOT WRITE!)`.
- Prefer `MBF_EXEC` (live) over `MBF_SAVE_TO_EEPROM` (persistent) for routine changes.
- Bit 0 of `MBF_RELAY_STATE` is the **acid pump** on factory units — never the demo relay.

## 10. Sources
- Tasmota NeoPool driver `xsns_83_neopool.ino` — definitive register map
- Tasmota docs: https://tasmota.github.io/docs/NeoPool/
- Official Sugar Valley *MODBUS Register Description* (Welldana D30-190045)
- HA references: `svasek/homeassistant-vistapool-modbus`, `alexdelprete/ha-sugar-valley-neopool`
