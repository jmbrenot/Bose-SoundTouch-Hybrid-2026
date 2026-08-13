# Spécification Modbus — DEYE SUN-6K-SG05LP1-EU-AM2-P

**Ce n'est pas la documentation officielle du fabricant Deye.** Le site deyeinverter.com et les moteurs de recherche généralistes sont inaccessibles depuis cette session (même blocage réseau que pour votre instance HA) — je n'ai pas pu récupérer le PDF constructeur.

C'est la meilleure alternative trouvée : le dépôt **[VMrenato/homeassistant-deye-tcan485-esphome](https://github.com/VMrenato/homeassistant-deye-tcan485-esphome)**, un projet ESPHome documentant une intégration validée sur du matériel réel identique au vôtre — un DEYE SUN-6K-SG05LP1-EU-AM2-P branché en RS485 direct (LilyGO T-CAN485). Sa table de registres provient à l'origine de `StephanJoubert/home_assistant_solarman` (le fichier `deye_hybrid.yaml` qu'on a utilisé pour Solarmodbus), recroisée avec `slipx06/Sunsynk-Home-Assistant-Dash`, puis **vérifiée sur l'onduleur réel** au fil de plusieurs jours de tests.

Récupéré le 13 août 2026 depuis les fichiers `README.md`, `docs/registers.md`, `docs/troubleshooting.md` et `docs/hardware.md` du dépôt (branche `main`), licence MIT.

---

## 1. README — contexte, câblage résumé, profil de polling

# Deye Hybrid Inverter + LilyGO T-CAN485 — ESPHome Modbus RTU

Monitor a **Deye SUN-SG05LP1-EU** hybrid inverter locally from Home Assistant using a **LilyGO T-CAN485** board talking Modbus RTU over RS485 — no cloud, no stick logger, ~5 second updates. This config was hardened over several days of testing against a real **Deye SUN-6K-SG05LP1-EU-AM2-P** and documents every pitfall found along the way (especially the T-CAN485 enable pins, which the official LilyGO docs get wrong for the current board revision).

## Why

Deye's newer WiBLE plug-and-play loggers **block local access entirely** (port 8899 is closed), pushing you into the Solarman cloud. The wired RS485 path on the inverter itself is always available, logger-free, and fully local: PV, battery, grid, load, energy totals and temperatures straight into Home Assistant over your LAN, with sub-5-second freshness and no third-party servers involved.

## Features

- **~5 s polling** of PV (2 MPPT), battery, grid, load, daily/total energy, and temperatures — 33 Modbus sensors + 5 template sensors
- **Grid-connected binary sensor** (register 194) — detect grid loss / ATS transfer to backup and trigger automations
- **WS2812 activity LED** on the board: blue flash = TX (request), green flash = RX (response) — instant visual confirmation the bus is alive
- **Home Assistant auto-discovery** via the native ESPHome API — every entity appears with proper device/state classes, ready for the Energy dashboard
- **No flow control pin needed** — the T-CAN485's MAX13487E transceiver is AutoDirection

## Tested hardware

| Component | Details |
|---|---|
| Inverter | **Deye SUN-6K-SG05LP1-EU-AM2-P** (single-phase hybrid, 6 kW, 48 V battery, 2 MPPT) |
| Board | **LilyGO T-CAN485** (ESP32 WROOM-32, MAX13487E RS485 transceiver, WS2812 RGB LED, CAN bus) — 2026 revision |
| Cable | Standard Ethernet patch cable, cut — only 3 wires used |

Should also work with the whole **SUN-3.6/5/6/7/8/10K-SG05LP1-EU** family (same register map). See [docs/hardware.md](docs/hardware.md) for details.

<!-- PHOTO: inverter wiring compartment with the RS485/MODBUS port highlighted -->
<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/1caf2e5f-b1a7-4151-a5d9-4cbf86ff63a3" />


## Wiring

Use the inverter's **RS485 / MODBUS port** inside the wiring compartment — **not** the BMS port and **not** the Meter/CT port. A cut Ethernet patch cable (T568B) works perfectly:

| RJ45 pin (T568B) | Wire color | Signal | Inverter terminal |
|---|---|---|---|
| 1 | White-orange | RS485 **B** | B |
| 2 | Orange | RS485 **A** | A |
| 3 | White-green | **GND** | GND (optional, recommended) |

The inverter is a Modbus RTU slave at **address 1, 9600 8N1**. Full details in [docs/hardware.md](docs/hardware.md).

<!-- DIAGRAM: T-CAN485 A/B/GND -> inverter RS485 port terminals -->

## Quick start

1. **Hardware** — flash nothing yet. Wire A → A, B → B, GND → GND between the T-CAN485 screw terminals and the inverter's RS485/MODBUS port (table above). Power off the inverter's AC and DC before opening the wiring compartment.
2. **Wiring check** — with everything powered, measure DC bias between A and B with a multimeter; you should see a small measurable voltage once the enable pins are driven (see the callout below).
3. **ESPHome** — copy [`esphome/deye-inversor.yaml`](esphome/deye-inversor.yaml) into your ESPHome dashboard (or `esphome run`), keeping the `!secret wifi_ssid` / `!secret wifi_password` references in your `secrets.yaml`. Flash over USB the first time.
4. **Adopt in Home Assistant** — the device is discovered automatically via the ESPHome integration; all `Deye *` entities appear within a minute. Watch the board LED: blue/green flicker every 5 s means the bus is working.

> [!IMPORTANT]
> **Enable pins — the #1 gotcha.** The official LilyGO README says RS485 EN is **GPIO9**. That is **wrong for the 2026 board revision** and leaves the bus completely dead. What actually works, validated on hardware:
>
> | GPIO | Function | State |
> |---|---|---|
> | **GPIO19** | RS485 EN | HIGH (active high) |
> | **GPIO17** | RS485 SE | HIGH (active high) |
> | **GPIO16** | 5V booster enable | HIGH (active high) |
>
> The YAML implements them as `gpio` switches with `restore_mode: ALWAYS_ON`, so they are driven automatically at boot. No `flow_control_pin` is needed — the MAX13487E handles direction itself.

## Register map

The config polls 33 Modbus registers (holding registers, slave 1): PV power/voltage/production, battery SOC/power/voltage/current/temperature/charge/discharge, grid power/CT/voltage/frequency/import/export, load power/consumption, inverter power and DC/AC temperatures — plus a grid-connected binary sensor on register 194.

Deye 32-bit energy totals are **low-word-first**: the YAML reads the lo/hi words separately and combines them in template sensors, e.g. total production = `(lo + hi × 65536) × 0.1` kWh.

Full table with addresses, types and scaling: **[docs/registers.md](docs/registers.md)**.

## Troubleshooting

The short version of a multi-day debugging journey:

- **Total silence on the bus** → enable pins wrong or not driven (the GPIO9-vs-GPIO19 trap above). Verify A-B bias with a multimeter.
- **Values landing on the wrong sensors (shifted by one)** → command queue overlap: too many commands per cycle for a slow slave. Fixed by relying on block reads (the controller batches consecutive registers) and an interval that fits the cycle: 19 commands ≈ 4–7 s at 9600 baud, so 5 s polling works.
- **Truncated frames / mid-frame byte loss** → cheap generic 5 V MAX485 modules. Avoid them; use the T-CAN485 (or a 3.3 V-native MAX3485).
- **Boot loop / rollback after OTA** → don't touch or reset the device for ~90 s after an OTA flash (`safe_mode` marks the boot successful after 60 s).
- **Grid frequency 0 / grid voltage off by 10×** → grid voltage is register **150** at ×0.1 (2377 = 237.7 V); don't use register 152.

Full details: **[docs/troubleshooting.md](docs/troubleshooting.md)**.

## Contributing

Issues and PRs welcome — especially confirmations or register-map notes for other Deye/Sunsynk models (SUN-*K-SG05LP1-EU family, SG04LP3 three-phase, Sunsynk rebadges). If you validate this on a different model or board revision, please report back.

## Credits

- [StephanJoubert/home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman) — `deye_hybrid.yaml` register definitions
- [slipx06/Sunsynk-Home-Assistant-Dash](https://github.com/slipx06/Sunsynk-Home-Assistant-Dash) — ESPHome-1P config this register map was cross-checked against
- **"Saentist"** — T-CAN485 enable-pins config that cracked the GPIO19/17/16 puzzle
- [Xinyuan-LilyGO/T-CAN485](https://github.com/Xinyuan-LilyGO/T-CAN485) — board documentation
- [ESPHome](https://esphome.io/) — the `modbus_controller` platform doing all the heavy lifting

## License

MIT — see [LICENSE](LICENSE).


---

## 2. Table de registres complète (docs/registers.md)

# Register map

All registers are **holding registers** (function code 0x03) on Modbus RTU slave **address 1**, 9600 8N1. Addresses below are decimal, exactly as used in [`esphome/deye-inversor.yaml`](../esphome/deye-inversor.yaml).

Sources: the register definitions were built from StephanJoubert's [home_assistant_solarman](https://github.com/StephanJoubert/home_assistant_solarman) `deye_hybrid.yaml` profile and cross-checked against slipx06's [Sunsynk-Home-Assistant-Dash](https://github.com/slipx06/Sunsynk-Home-Assistant-Dash) ESPHome-1P config, then validated against a live SUN-6K-SG05LP1-EU-AM2-P.

`skip_updates` in the tables is the ESPHome throttle (publish only every Nth polled value) used to reduce HA database churn on slow-changing values.

## Solar

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye PV1 Power | 186 | U_WORD | ×1 | W | — |
| Deye PV2 Power | 187 | U_WORD | ×1 | W | — |
| Deye PV1 Voltage | 109 | U_WORD | ×0.1 | V | 2 |
| Deye PV2 Voltage | 111 | U_WORD | ×0.1 | V | 2 |
| Deye Daily Production | 108 | U_WORD | ×0.1 | kWh | 5 |
| *(total_production_lo, internal)* | 96 | U_WORD | low word | — | 5 |
| *(total_production_hi, internal)* | 97 | U_WORD | high word | — | 5 |
| **Deye Total Production** (template) | 96-97 | — | `(lo + hi x 65536) x 0.1` | kWh | — |
| **Deye PV Total Power** (template) | — | — | `PV1 + PV2` | W | — |

## Battery

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Battery SOC | 184 | U_WORD | ×1 | % | — |
| Deye Battery Power | 190 | S_WORD | ×1 (signed; discharge/charge by sign) | W | — |
| Deye Battery Voltage | 183 | U_WORD | ×0.01 | V | 2 |
| Deye Battery Current | 191 | S_WORD | ×0.01 (signed) | A | 2 |
| Deye Battery Temperature | 182 | U_WORD | ×0.1 - 100 | °C | 5 |
| Deye Daily Battery Charge | 70 | U_WORD | ×0.1 | kWh | 5 |
| Deye Daily Battery Discharge | 71 | U_WORD | ×0.1 | kWh | 5 |
| *(total_battery_charge_lo, internal)* | 72 | U_WORD | low word | — | 5 |
| *(total_battery_charge_hi, internal)* | 73 | U_WORD | high word | — | 5 |
| **Deye Total Battery Charge** (template) | 72-73 | — | `(lo + hi x 65536) x 0.1` | kWh | — |
| *(total_battery_discharge_lo, internal)* | 74 | U_WORD | low word | — | 5 |
| *(total_battery_discharge_hi, internal)* | 75 | U_WORD | high word | — | 5 |
| **Deye Total Battery Discharge** (template) | 74-75 | — | `(lo + hi x 65536) x 0.1` | kWh | — |

## Grid

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Grid Power | 169 | S_WORD | ×1 (signed) | W | — |
| Deye Grid CT Power | 172 | S_WORD | ×1 (signed, external CT/meter) | W | — |
| Deye Grid Voltage L1 | 150 | U_WORD | ×0.1 (2377 = 237.7 V) | V | 2 |
| Deye Grid Frequency | 79 | U_WORD | ×0.01 (5002 = 50.02 Hz) | Hz | 5 |
| Deye Daily Energy Bought | 76 | U_WORD | ×0.1 | kWh | 5 |
| Deye Daily Energy Sold | 77 | U_WORD | ×0.1 | kWh | 5 |
| Deye Total Grid Import | 78 | U_WORD | ×0.1 | kWh | 5 |
| Deye Total Grid Export | 81 | U_WORD | ×0.1 | kWh | 5 |

> **Note:** grid voltage is register **150** at ×0.1. Register 152 is *not* the right source — using it gives a 0 Hz frequency / wrongly scaled voltage.

## Load

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Load Power | 178 | U_WORD | ×1 | W | — |
| Deye Daily Load Consumption | 84 | U_WORD | ×0.1 | kWh | 5 |
| *(total_load_consumption_lo, internal)* | 85 | U_WORD | low word | — | 5 |
| *(total_load_consumption_hi, internal)* | 86 | U_WORD | high word | — | 5 |
| **Deye Total Load Consumption** (template) | 85-86 | — | `(lo + hi x 65536) x 0.1` | kWh | — |

## Inverter

| Sensor | Address | Type | Scale / filter | Unit | skip_updates |
|---|---|---|---|---|---|
| Deye Inverter Power | 175 | S_WORD | ×1 (signed) | W | — |
| Deye DC Temperature | 90 | U_WORD | ×0.1 - 100 | °C | 5 |
| Deye AC Temperature | 91 | U_WORD | ×0.1 - 100 | °C | 5 |

## Binary sensor

| Entity | Address | Notes |
|---|---|---|
| Deye Grid Connected | 194 | 1 = grid present, 0 = grid lost (ATS/backup detection for automations) |

## 32-bit energy totals: low-word-first

Deye stores cumulative energy counters as 32-bit values in two consecutive 16-bit registers, **low word first**. The YAML reads each word as a separate internal `U_WORD` sensor and combines them in a template sensor:

```yaml
lambda: "return (id(total_production_lo).state + id(total_production_hi).state * 65536.0) * 0.1;"
```

This applies to registers 96-97 (total production), 72-73 (total battery charge), 74-75 (total battery discharge) and 85-86 (total load consumption). Total grid import (78) and export (81) fit in a single word on this model and are read directly.

## Polling profile

- `update_interval: 5s` — one full read cycle every 5 s
- `command_throttle: 50ms` — gap between consecutive Modbus commands
- `offline_skip_updates: 3` — declare the inverter offline after 3 failed cycles
- Consecutive registers are automatically batched into block reads by `modbus_controller`; a full cycle is ~19 commands and takes roughly 4-7 s at 9600 baud, which is why 5 s polling is the sweet spot.


---

## 3. Câblage matériel (docs/hardware.md)

# Hardware

## Inverter

**Deye SUN-6K-SG05LP1-EU-AM2-P** — single-phase hybrid, 6 kW, 48 V battery, 2 MPPT trackers.

This config should work unchanged with the whole single-phase low-voltage family, which shares the same register map:

- SUN-3.6K-SG05LP1-EU
- SUN-5K-SG05LP1-EU
- SUN-6K-SG05LP1-EU (tested)
- SUN-7K/8K/10K-SG05LP1-EU

The inverter exposes a **Modbus RTU slave at address 1, 9600 baud, 8N1** on its RS485 port.

## Board

**LilyGO T-CAN485** (2026 revision):

- ESP32 classic (WROOM-32)
- Integrated **MAX13487E** RS485 transceiver with **AutoDirection** — no DE/RE flow control required from software
- WS2812 RGB LED (GPIO4) — used here as a bus activity indicator
- CAN bus transceiver (unused in this project)
- Screw terminals for RS485 A/B and power

### Enable pins (critical)

The RS485 section of the board is gated by three GPIOs that **must be driven HIGH** or the bus is electrically dead. The official LilyGO README documents RS485 EN as GPIO9 — **this is wrong for the 2026 revision**. The validated mapping:

| GPIO | Function | Required state |
|---|---|---|
| GPIO19 | RS485 EN | HIGH (active high) |
| GPIO17 | RS485 SE | HIGH (active high) |
| GPIO16 | 5V booster enable | HIGH (active high) |

The ESPHome config drives them via `gpio` switches with `restore_mode: ALWAYS_ON`, so they come up automatically at every boot and can also be toggled from Home Assistant for debugging.

### UART

| Signal | GPIO |
|---|---|
| TX | GPIO22 |
| RX | GPIO21 |

9600 baud, 8 data bits, no parity, 1 stop bit. **No `flow_control_pin`** — the MAX13487E handles direction automatically.

## Wiring

Use the inverter's port labeled **RS485** or **MODBUS** inside the wiring compartment.

- **NOT the BMS port** (that's CAN/RS485 for the battery BMS)
- **NOT the Meter/CT port**

A standard Ethernet patch cable, cut, is all you need — only 3 wires are used:

| RJ45 pin (T568B) | Wire color | Signal | Connects to |
|---|---|---|---|
| 1 | White-orange | RS485 B | Inverter terminal B |
| 2 | Orange | RS485 A | Inverter terminal A |
| 3 | White-green | GND | Inverter GND (optional but recommended) |

> **Safety:** the wiring compartment contains mains-voltage terminals. Disconnect AC (grid + backup) and DC (PV + battery) and wait for the inverter to fully discharge before opening it.

### Verifying the connection

With the inverter on and the board flashed:

1. The three enable switches (`RS485 EN`, `RS485 SE`, `Booster 5V EN`) must be ON.
2. Measure DC voltage between A and B with a multimeter — a small bias (typically a few hundred mV to a couple of volts, fluctuating) should be measurable. Zero volts means the transceiver is not enabled.
3. The board LED flickers **blue on TX, green on RX** every 5 s polling cycle.

## Notes on alternatives

Cheap generic **MAX485 modules (5 V)** caused truncated frames and mid-frame byte loss in testing — avoid them. If you don't use a T-CAN485, use a **MAX3485** (3.3 V native) or another AutoDirection 3.3 V transceiver.


---

## 4. Dépannage (docs/troubleshooting.md)

# Troubleshooting

Distilled from several days of bring-up against a live Deye SUN-6K-SG05LP1-EU-AM2-P. Symptoms first, then cause and fix.

## 1. Total silence on the bus (no responses at all)

**Symptom:** ESPHome logs show TX requests but zero RX; every sensor stays `unavailable`/`unknown`. The board LED flashes blue (TX) but never green (RX).

**Cause:** the T-CAN485 enable pins are wrong or not driven. The official LilyGO README says RS485 EN = **GPIO9** — that is **wrong for the 2026 board revision**. Without the enables, the MAX13487E never drives the bus.

**Fix:**

- Drive **GPIO19 (RS485 EN), GPIO17 (RS485 SE) and GPIO16 (5V booster) HIGH** — all active HIGH. The YAML does this with `gpio` switches and `restore_mode: ALWAYS_ON`.
- Verify electrically: with a multimeter on DC between A and B you should measure a small bias voltage once the enables are on. No bias = transceiver not enabled.
- Double-check A/B are not swapped and you are on the **RS485/MODBUS port** — not BMS, not Meter/CT.

## 2. Values landing on the wrong sensors (shifted by one)

**Symptom:** responses arrive, but values appear under the wrong entities — e.g. battery SOC showing what looks like PV power, everything offset by one register.

**Cause:** command queue overlap. Too many Modbus commands per cycle for a slow slave: the controller's queue and the inverter's responses drift out of sync, and a response gets matched to the next queued command.

**Fix:**

- Let the controller use **block reads** — `modbus_controller` automatically batches consecutive registers into a single request, drastically cutting the command count.
- Pick an interval that fits the cycle: the full poll is ~19 commands taking **≈ 4–7 s at 9600 baud**, so `update_interval: 5s` works; anything faster reintroduces the overlap.
- Keep `command_throttle: 50ms` so the slave gets breathing room between commands.

## 3. Truncated frames / mid-frame byte loss

**Symptom:** intermittent CRC errors, responses that stop mid-frame, or sporadic single-byte loss — bus mostly works but corrupts regularly.

**Cause:** cheap generic **MAX485 modules (5 V)**. They are not reliably compatible with the ESP32's 3.3 V logic and tend to mangle frames.

**Fix:** don't use them. Use the **T-CAN485** (integrated MAX13487E, AutoDirection) or a **MAX3485** (3.3 V native) if building from discrete parts.

## 4. Boot loop / rollback right after an OTA flash

**Symptom:** device reboots into the previous firmware after an OTA update.

**Cause:** ESPHome `safe_mode` only marks a boot as successful after **60 s** of uptime. Power-cycling or resetting within that window triggers a rollback.

**Fix:** after an OTA flash, **don't touch or reset the device for ~90 s**. Let it sit, confirm it stays up, then walk away.

## 5. Grid frequency reads 0 / grid voltage at the wrong scale

**Symptom:** grid frequency stuck at 0 Hz and/or grid voltage off by a factor of 10 (or nonsense like 23 V).

**Cause:** wrong register / wrong scale. Grid voltage lives at register **150** with a **×0.1** scale — raw 2377 means 237.7 V. Register 152 is *not* the correct source. Grid frequency is register **79** at ×0.01 (5002 = 50.02 Hz).

**Fix:** use 150 (×0.1) for grid voltage and 79 (×0.01) for frequency, as in the YAML — don't use 152.

## Quick diagnostic checklist

1. Are `RS485 EN`, `RS485 SE`, `Booster 5V EN` switches all ON?
2. Multimeter: measurable DC bias between A and B?
3. LED: blue (TX) *and* green (RX) flicker every 5 s?
4. Correct inverter port (RS485/MODBUS, not BMS/Meter)? A on pin 2 (orange), B on pin 1 (white-orange)?
5. ESPHome logs: `uart debug` in the YAML prints both directions — compare TX requests against RX frames.


---

*Archivé le 13 août 2026 par Claude, dans le cadre de l'intégration DEYE Gruissan — voir `deye-modbus-gruissan-runbook.md` dans ce dépôt pour le suivi complet.*
