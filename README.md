# 🔥 Level 1 — TBH Heater: PV Surplus Utilization

## Description

This automation turns on the **TBH electric boiler heater (2.9 kW)** when three conditions are met simultaneously:
- the energy storage battery is nearly full
- the PV installation is producing above the threshold
- energy is being exported to the grid anyway

Instead of exporting surplus PV energy to the grid, it is consumed locally for domestic hot water (DHW) heating.

---

## Hardware

| Device | Model | Protocol | Link |
|---|---|---|---|
| Hybrid inverter | Deye SUN 12k | ha-solarman (Modbus) | [ha-solarman](https://github.com/davidrapan/ha-solarman) |
| Energy storage | 10 kWh, min SoC 15% | — | — |
| Boiler heater | TBH 2.9 kW | Zigbee (TNCE RMDZB-1PNL63) | — |
| Energy meter | TNCE RMDZB-1PNL63 | Zigbee2MQTT | — |
| Heat pump | Kaisai Arctic KHC-10RY3-B | Modbus RS485 / ESPHome | [Mosibi ESPHome](https://github.com/Mosibi/Midea-heat-pump-ESPHome) |

> **Kaisai Arctic KHC-10RY3-B** is a Midea clone — controlled via Modbus RS485
> using the [Mosibi/Midea-heat-pump-ESPHome](https://github.com/Mosibi/Midea-heat-pump-ESPHome) project.
> ESPHome configuration file: [`heatpump.yaml`](https://github.com/Mosibi/Midea-heat-pump-ESPHome/blob/master/heatpump.yaml)

---

## Home Assistant Entities

| Role | Entity ID | Description |
|---|---|---|
| Battery SoC | `sensor.deye_sun_12k_battery` | State of charge [%] |
| PV production | `sensor.deye_sun_12k_microinverter_power` | Instantaneous PV power [W] |
| Grid power | `sensor.deye_sun_12k_grid_power` | Negative = export [W] |
| House load | `sensor.deye_sun_12k_load_power` | Current consumption [W] |
| Heater switch | `switch.tbh_grzalka_odwrocony` | Inverted logic (OFF = turn on) |
| Heater energy | `sensor.tbh_grzalka_energy` | Energy counter kWh [total_increasing] |
| DHW temperature | `sensor.heatpump_water_tank_temperature_t5` | Boiler water temperature [°C] |

> **Note:** `switch.tbh_grzalka_odwrocony` is a helper entity with inverted logic —
> the physical switch `switch.tbh_grzalka` works in reverse (OFF = heater ON).
> Deye entities are provided by the [ha-solarman](https://github.com/davidrapan/ha-solarman) integration.

---

## Decision Logic

```
Every minute check:

  Heater is OFF?
    + SoC > 98%
    + PV > 3000 W
    + Grid export < -2800 W
    + Water temperature < 70°C
    → TURN ON heater

  Heater is ON?
    + SoC > 95%               ← allows battery discharge down to 95%
    + Grid import < 300 W     ← tolerance for momentary fluctuations
    + Water temperature < 70°C
    → KEEP ON

  Heater is ON and conditions not met?
    → Wait 5 minutes          ← buffer for temporary cloud cover
    → Check again:
      OK  → stays ON
      NOK → TURN OFF
```

### Decision Thresholds

| Parameter | ON threshold | KEEP ON threshold | Reason |
|---|---|---|---|
| Battery SoC | > 98% | > 95% | Battery full; 95% = buffer for heater operation |
| PV production | > 3000 W | — | Stable production, not a momentary peak |
| Grid export | < -2800 W | < 300 W | Energy going to grid → heater takes it |
| Water temperature | < 70°C | < 70°C | Protection against boiler overheating |
| Turn-off delay | — | 5 min | Resilience against momentary cloud cover |

---

## Consumption Model

| State | Consumption |
|---|---|
| House without heat pump | ~1.0 kWh/h |
| House with heat pump active (Kaisai Arctic) | ~2.5 kWh/h |
| TBH heater | 2.9 kW (real-time Zigbee measurement) |

---

## Automation File

→ [`level1_tbh_grzalka.yaml`](level1_tbh_grzalka.yaml)

### Installation

1. Open Home Assistant
2. **Settings → Automations → Create Automation**
3. Click three dots `⋮` → **Edit in YAML**
4. Paste the contents of `level1_tbh_grzalka.yaml`
5. Click **Save**

### Post-installation Verification

Check in **Developer Tools → States** that entities return numeric values:

```
sensor.deye_sun_12k_battery                   → e.g. 99
sensor.deye_sun_12k_microinverter_power        → e.g. 4500
sensor.deye_sun_12k_grid_power                 → e.g. -3200
sensor.heatpump_water_tank_temperature_t5      → e.g. 52.3
```

---

## Energy Monitoring

Heater added to **Energy Dashboard** (Settings → Energy → Individual devices):

- `sensor.tbh_grzalka_energy` — kWh counter (Zigbee, real measurement)
- Daily/monthly charts: **Electricity → Individual devices** tab

Optional resettable counters (`configuration.yaml`):

```yaml
utility_meter:
  heater_energy_daily:
    source: sensor.tbh_grzalka_energy
    cycle: daily
  heater_energy_monthly:
    source: sensor.tbh_grzalka_energy
    cycle: monthly
```

---

## Known Limitations

- Checks every **1 minute** — no immediate reaction
- 5-minute buffer minimizes cycling effect with dynamic PV output
- `sensor.deye_sun_12k_battery` returns an integer (e.g. `99`, not `99%`)

---

## Related Integrations

| Integration | Description | Link |
|---|---|---|
| ha-solarman | Deye SUN 12k inverter control | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) |
| Mosibi ESPHome | Kaisai Arctic heat pump control via Modbus RS485 | [github.com/Mosibi/Midea-heat-pump-ESPHome](https://github.com/Mosibi/Midea-heat-pump-ESPHome) |
| Solcast | PV production forecast | [github.com/BJReplay/ha-solcast-solar](https://github.com/BJReplay/ha-solcast-solar) |

---

## Related Automations

| Level | Name | Status |
|---|---|---|
| Level 1 | TBH Heater — PV surplus utilization | ✅ Active |
| Level 2 | Heat pump — control by outdoor and indoor temperature | 🔧 In progress |
| Level 3 | Night BESS charging based on Solcast forecast | 🔧 In progress |
| Level 4 | Hourly BESS orchestrator | 🔧 In progress |
