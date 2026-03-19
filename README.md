# 🔥 Level 1 — TBH Heater: PV Surplus to DHW

## Description

This automation turns on the **TBH electric boiler heater (2.9 kW)** when three conditions are met simultaneously:
- the energy storage battery is nearly full
- real PV surplus (PV minus house load) exceeds heater demand
- water temperature is below the safety threshold

Instead of exporting surplus PV energy to the grid, it is consumed locally for domestic hot water (DHW) heating.

> **Note on Zero Export:** This system uses Zero Export with CT clamp, meaning
> `grid_power` stays near 0 at all times and cannot be used as a surplus indicator.
> Surplus is calculated directly as `PV power − house load`.

---

## Hardware

| Device | Model | Protocol | Link |
|---|---|---|---|
| Hybrid inverter | Deye SUN 12k | ha-solarman (Modbus) | [ha-solarman](https://github.com/davidrapan/ha-solarman) |
| Energy storage | 10 kWh, min SoC 15% | — | — |
| Boiler heater | TBH Heater 2.9 kW | Zigbee (TNCE RMDZB-1PNL63) | — |
| Energy meter | TNCE RMDZB-1PNL63 | Zigbee2MQTT | — |

---

## Home Assistant Entities

| Role | Entity ID | Description |
|---|---|---|
| Battery SoC | `sensor.deye_sun_12k_battery` | State of charge [%] |
| PV production | `sensor.deye_sun_12k_microinverter_power` | Instantaneous PV power [W] |
| House load | `sensor.deye_sun_12k_load_power` | Current consumption [W] |
| Grid power | `sensor.deye_sun_12k_grid_power` | Negative = export [W] |
| Heater switch | `switch.tbh_grzalka_odwrocony` | Inverted logic (OFF = turn on) |
| Heater energy | `sensor.tbh_grzalka_energy` | Energy counter kWh [total_increasing] |
| DHW temperature | `sensor.heatpump_water_tank_temperature_t5` | Boiler water temperature [°C] |

> **Note:** `switch.tbh_grzalka_odwrocony` is a helper entity with inverted logic —
> the physical switch `switch.tbh_grzalka` works in reverse (OFF = heater ON).
> Deye entities are provided by the [ha-solarman](https://github.com/davidrapan/ha-solarman) integration.

---

## Decision Logic

```
Every minute (+ immediately on temperature trigger):

  PRIORITY 1 — Overheat protection:
    Water temperature > 70°C
    → TURN OFF immediately (no delay)

  PRIORITY 2 — Turn ON:
    Heater is OFF
    + Water temperature < 70°C
    + Battery SoC > 97%
    + PV − Load > 2500 W    ← ~400W buffer above heater demand (2900W)
    → TURN ON heater

  PRIORITY 3 — Keep ON:
    Heater is ON
    + Water temperature < 70°C
    + Battery SoC > 95%     ← allows discharge down to 95%
    + PV − Load > 0 W       ← relaxed: accept momentary battery draw
    → KEEP ON

  PRIORITY 4 — Graceful turn-off:
    Heater is ON, conditions not met
    → Wait 5 minutes         ← buffer for temporary cloud cover
    → Re-check:
      Conditions OK  → stays ON
      Conditions NOK → TURN OFF
```

### Decision Thresholds

| Parameter | Turn ON | Keep ON | Reason |
|---|---|---|---|
| Battery SoC | > 97% | > 95% | Battery nearly full; 95% = buffer for heater operation |
| PV surplus (PV − Load) | > 2500 W | > 0 W | Strict ON, relaxed KEEP — avoids cycling |
| Water temperature | < 70°C | < 70°C | Boiler overheat protection |
| Turn-off delay | — | 5 min | Resilience against momentary cloud cover |

### Changelog vs previous version

| Change | Reason |
|---|---|
| Removed `grid_power < -2800` from turn-on condition | Zero Export CT keeps grid near 0 — condition was never true |
| Replaced with `pv - load > 2500` | Real surplus calculation with ~400W buffer above heater power |
| Keep-ON uses `pv - load > 0` instead of `grid_power < 300` | Consistent logic; accepts momentary battery draw when heater running |
| Added `temperature < 70` explicitly to Priority 3 | Previously missing from keep-on check — potential safety gap |
| Added immediate temperature trigger | Turns off instantly at 70°C without waiting for next 1-min cycle |

---

## Automation File

→ [`tbh_heater_pv_surplus.yaml`](tbh_heater_pv_surplus.yaml)

### Installation

1. Open Home Assistant
2. **Settings → Automations → Create Automation**
3. Click three dots `⋮` → **Edit in YAML**
4. Paste the contents of `tbh_heater_pv_surplus.yaml`
5. Click **Save**

### Post-installation Verification

Check in **Developer Tools → States** that entities return numeric values:

```
sensor.deye_sun_12k_battery                   → e.g. 99
sensor.deye_sun_12k_microinverter_power        → e.g. 4800
sensor.deye_sun_12k_load_power                 → e.g. 1100
sensor.heatpump_water_tank_temperature_t5      → e.g. 52.3
```

Verify surplus calculation in **Developer Tools → Template**:

```jinja
{% set pv = states('sensor.deye_sun_12k_microinverter_power') | float %}
{% set load = states('sensor.deye_sun_12k_load_power') | float %}
PV: {{ pv }} W
Load: {{ load }} W
Surplus: {{ pv - load }} W
Heater can turn on: {{ (pv - load) > 2500 }}
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

- Checks every **1 minute** — except temperature which triggers immediately
- `mode: single` prevents parallel execution — if delay is active, new triggers are ignored
- `sensor.deye_sun_12k_battery` returns an integer (e.g. `99`, not `99%`)
- PV and load values may spike briefly — 5-minute turn-off delay handles this

---

## Related Integrations

| Integration | Description | Link |
|---|---|---|
| ha-solarman | Deye SUN 12k inverter control | [github.com/davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman) |

---

## Automations

| Level | Name | Status |
|---|---|---|
| Level 1 | TBH Heater — PV Surplus to DHW | ✅ Active |
