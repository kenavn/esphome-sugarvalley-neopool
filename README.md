# esphome-sugarvalley-neopool

An ESPHome **YAML package** for Sugar Valley **NeoPool** pool/spa controllers over
RS-485 Modbus — sold as **Oxilife, Hidrolife, Aquascenic, Bionet, Hidroniser,
UVScenic, Station, Brilix, Bayrol, Hay** (and Albixon/VistaPool rebrands).

Unlike a compiled external component, this is pure configuration: the controller
speaks standard Modbus, which ESPHome's built-in `modbus_controller` already
handles. You drop the package in, point it at your Modbus hub, and get all the
sensors. See **[REGISTERS.md](REGISTERS.md)** for the complete register map, and
**[WRITE_SAFETY.md](WRITE_SAFETY.md)** for which writes are safe and why (read it
before adding any write endpoint).

## Status

| Area | State |
|---|---|
| Measurements (pH, redox, chlorine, salt, temp, electrolysis, cell V, ionization) | ✅ reads |
| Setpoint read-backs (pH hi/lo, redox, chlorine, hydrolysis) | ✅ reads |
| Status / alarms (flow, **pump/filtration**, on-target, cover, shock, acid pump, pH alarm, …) | ✅ binary sensors |
| Filtration mode | ✅ decoded text |
| Relay control | ❌ removed — unsafe via ESPHome's high-level API ([WRITE_SAFETY.md](WRITE_SAFETY.md)) |
| Setpoint **writes** (pH hi/lo, redox, chlorine, electrolysis) + filtration-mode select | 🔌 wired but **commented out** — uncomment to enable (validated; the only write class) |
| Boost write | 📋 documented (REGISTERS.md §4.3) |

## Requirements

- ESPHome with the **`esp-idf`** framework (recommended; native USB logging keeps
  the UART free).
- An RS-485 transceiver wired to the controller's Modbus port, **19200 8N1**,
  slave address `0x01`. (Some service ports are direct 3.3 V TTL — then omit
  `flow_control_pin`.)

## Usage

You provide the UART + Modbus hub (board-specific pins); the package provides the
`modbus_controller` and all entities.

```yaml
uart:
  - id: uart_rs485
    tx_pin: GPIO4          # your transceiver DI
    rx_pin: GPIO5          # your transceiver RO
    baud_rate: 19200

modbus:
  id: modbus_bus
  uart_id: uart_rs485
  flow_control_pin: GPIO6  # SP3485 DE/~RE; omit for direct TTL

packages:
  neopool: github://kenavn/esphome-sugarvalley-neopool/neopool.yaml@main

substitutions:
  neopool_modbus_id: modbus_bus   # must match your modbus: id
  neopool_name: "Pool"
```

During development you can include it locally instead:

```yaml
packages:
  neopool: !include ../esphome-sugarvalley-neopool/neopool.yaml
```

## Configuration (substitutions)

| Substitution | Default | Purpose |
|---|---|---|
| `neopool_modbus_id` | `modbus_bus` | id of **your** `modbus:` hub |
| `neopool_id` | `oxilife` | id of the `modbus_controller` this package creates |
| `neopool_address` | `0x01` | NeoPool Modbus slave address |
| `neopool_name` | `Pool` | entity name prefix |
| `neopool_update_interval` | `10s` | base poll cadence |
| `neopool_filt_relay_bitmask` | `0x0002` | which `MBF_RELAY_STATE` bit drives the **read-only** "Pump (Filtration) Active" sensor (Relay 2 = 0x0002 by default; confirm vs `MBF_PAR_FILT_GPIO` 0x0412) |

## Entities provided

Sensors: pH, Redox, Chlorine, Salt/Conductivity, Water Temperature, Ionization
Level, Electrolysis Production, Cell Voltage, plus diagnostic setpoint read-backs.
Binary sensors: Cell Flow (FL1), **Pump (Filtration) Active**, On Target, Low,
Cover, Module Active, Redox Control, Shock Mode, Acid Pump, pH Alarm. Text:
Filtration Mode. (No relay switch — relay control removed; see WRITE_SAFETY.md.)

## Adding write endpoints

The setpoint writes (pH hi/lo, redox, chlorine, electrolysis) and the
filtration-mode select are **already wired at the bottom of `neopool.yaml`, but
commented out** so nothing can write to a live pool until you opt in. To enable:
uncomment that block and comment out the matching read-back sensors (the
instructions are in the file). Each write applies live via `MBF_EXEC` and does
not touch EEPROM. For boost and anything else, **[REGISTERS.md §2 & §4](REGISTERS.md)**
has the full procedure and copy-paste recipes.

## Safety

These writes drive real chemical dosing and pumps. Keep `min`/`max` clamps, prefer
live `MBF_EXEC` over EEPROM saves, and never write factory `*_NOM` registers.
**Relay control is deliberately not exposed** — a bitmask read-modify-write of the
shared `MBF_RELAY_STATE` word under ESPHome's high-level API stopped a live pump in
testing. **[WRITE_SAFETY.md](WRITE_SAFETY.md)** has the full analysis, the holistic
safe/unsafe classification, and the low-level plan to add relay control correctly.

## Credits & sources

Register map from the Tasmota `xsns_83_neopool.ino` driver and the official Sugar
Valley *MODBUS Register Description* (Welldana D30-190045). Sibling projects:
[`svasek/homeassistant-vistapool-modbus`](https://github.com/svasek/homeassistant-vistapool-modbus),
[`alexdelprete/ha-sugar-valley-neopool`](https://github.com/alexdelprete/ha-sugar-valley-neopool).

## License

MIT
