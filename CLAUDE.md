# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant dashboard cards for solar power monitoring and forecasting using the **ApexCharts Card** (`custom:apexcharts-card`). Two YAML configurations provide alternative forecast visualizations for a SENEC battery/solar system:

- **Solar_Forecast_ML.yaml** — Uses the Solar Forecast ML integration (local ML-based predictions)
- **Solcast_PV_Forecast.yaml** — Uses the Solcast API (cloud-based forecast service)

All UI labels are in **German**.

## Architecture

Both cards share the same base structure:

1. **Chart series (rendered on graph):** Battery charge % (green, secondary y-axis), house power consumption in kW (red), solar generation in kW (orange), plus a forecast line (grey/blue)
2. **Header-only series (`header_only` y-axis, `in_chart: false`):** Display forecast values (today, remaining, tomorrow) and metadata as text in the card header without plotting on the chart
3. **Y-axes:** `kW` (primary, power), `capacity` (secondary/opposite, battery %), `header_only` (hidden, display-only)

Key differences between the two cards:
- **Solcast** uses `data_generator` with JavaScript to map `entity.attributes.detailedForecast` into chart data points
- **Solar Forecast ML** references dedicated `sensor.none_*` entities and includes AI model/metrics info
- Solcast's "Letztes Update" transform computes days (divides by `60/60/24`); ML version computes minutes (divides by `1000/60`)

## Conventions

- Power sensor values are in **Watts** and converted to **kW** via `transform: return x/1000;`
- Real-time series use `extend_to: now` and `group_by` with 5-minute intervals
- Forecast/prediction series use 30-minute grouping or `data_generator`
- Color scheme: green=battery, red=consumption, orange=generation, blue=current/now, grey=forecast
- Default series config: `type: area`, `opacity: 0.9`, `stroke_width: 1`

## No Build System

These are raw YAML configuration files consumed directly by Home Assistant. There is no build, lint, or test pipeline. To deploy, paste the YAML into a Home Assistant Lovelace dashboard card configuration (manual YAML mode).

## Required Home Assistant Integrations

- **SENEC** — `sensor.senec_battery_charge_percent`, `sensor.senec_house_power`, `sensor.senec_solar_generated_power`
- **Solar Forecast ML** — `sensor.solar_forecast_ml_*`, `sensor.none_*`, `update.solar_forecast_ml_update`
- **Solcast PV Forecast** — `sensor.solcast_pv_forecast_*`
- **Custom** — `sensor.tt_solar_generated` (total daily solar generation)
- **Frontend** — [apexcharts-card](https://github.com/RomRider/apexcharts-card) custom Lovelace card
