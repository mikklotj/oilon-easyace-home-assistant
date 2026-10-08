# Energy dashboard and COP

The heat pump reports three cumulative energy counters over Modbus, all in kWh:

| Entity | What it counts |
|---|---|
| `sensor.oilon_easyace_electrical_energy` | Electricity used by the heat pump |
| `sensor.oilon_easyace_heating_energy` | Heat delivered to the heating and hot water side |
| `sensor.oilon_easyace_ground_energy` | Heat taken from the ground loop |

On the reference unit, heating energy ≈ electrical energy + ground energy, which
is what you would expect for a ground-source heat pump. The REST sensor
`sensor.oilon_easyace_electric_heater_energy` counts the energy of the electric
backup heater separately. **Whether the Modbus electrical energy counter includes
the electric heater has not been verified.**

All three counters use `state_class: total_increasing`, so Home Assistant records
long-term statistics for them automatically.

## Adding the heat pump to the Energy dashboard

Settings → Dashboards → Energy:

- **Electricity grid / Individual devices**: add
  `sensor.oilon_easyace_electrical_energy` as an individual device. The heat pump
  then shows up in the per-device breakdown of your electricity use.
- Home Assistant's Energy dashboard has no "heat produced" category. Do not add
  `heating_energy` or `ground_energy` as grid or solar sources, because that would
  distort your electricity figures. Use the statistics cards below instead.

If you prefer MWh, change the display unit in the entity settings (gear icon →
Unit of measurement). Home Assistant then converts the long-term statistics
automatically.

## COP (coefficient of performance)

- `sensor.oilon_easyace_heating_cop` (Modbus) is the controller's instantaneous
  COP. It jumps to very high values for a moment when the compressor starts or
  stops, so it is only meaningful while the compressor runs steadily.
- `sensor.oilon_easyace_average_cop` (REST) is the controller's own long-term
  average.
- A **daily or monthly COP** from the energy counters is usually the most useful
  number. Example:

```yaml
# Package or configuration.yaml
utility_meter:
  oilon_easyace_heating_energy_daily:
    source: sensor.oilon_easyace_heating_energy
    cycle: daily
  oilon_easyace_electrical_energy_daily:
    source: sensor.oilon_easyace_electrical_energy
    cycle: daily
  oilon_easyace_heating_energy_monthly:
    source: sensor.oilon_easyace_heating_energy
    cycle: monthly
  oilon_easyace_electrical_energy_monthly:
    source: sensor.oilon_easyace_electrical_energy
    cycle: monthly

template:
  - sensor:
      - name: "Oilon EasyAce COP today"
        unique_id: oilon_easyace_cop_today
        state_class: measurement
        availability: >-
          {{ states('sensor.oilon_easyace_electrical_energy_daily') | float(0) > 0.5 }}
        state: >-
          {{ (states('sensor.oilon_easyace_heating_energy_daily') | float(0)
              / states('sensor.oilon_easyace_electrical_energy_daily') | float(1))
             | round(2) }}

      - name: "Oilon EasyAce COP this month"
        unique_id: oilon_easyace_cop_month
        state_class: measurement
        availability: >-
          {{ states('sensor.oilon_easyace_electrical_energy_monthly') | float(0) > 1 }}
        state: >-
          {{ (states('sensor.oilon_easyace_heating_energy_monthly') | float(0)
              / states('sensor.oilon_easyace_electrical_energy_monthly') | float(1))
             | round(2) }}
```

The counters only change in whole kWh, so short periods (one hour, or a day in
summer when only hot water is made) give rough COP values. The `availability`
limits hide the COP until enough energy has been counted.

A statistics graph card with `period: month` and `stat_types: [change]` over the
heating and electrical energy sensors shows monthly heat and electricity side by
side:

```yaml
type: statistics-graph
title: Heat pump energy per month
period: month
chart_type: bar
stat_types:
  - change
entities:
  - sensor.oilon_easyace_heating_energy
  - sensor.oilon_easyace_electrical_energy
  - sensor.oilon_easyace_ground_energy
```
