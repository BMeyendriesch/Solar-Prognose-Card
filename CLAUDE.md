# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant dashboard cards for solar power monitoring and forecasting using the **ApexCharts Card** (`custom:apexcharts-card`). Two YAML configurations provide alternative forecast visualizations for a SENEC battery/solar system:

- **Solar_Forecast_ML.yaml** — Uses the Solar Forecast ML integration (local ML-based predictions)
- **Solcast_PV_Forecast.yaml** — Uses the Solcast API (cloud-based forecast service)
- **SFML_Stats_Gesamt.yaml** — Prognose vs. Messung: Gesamtanlage (full width)
- **SFML_Stats_String_Ost.yaml** — Prognose vs. Messung: String Ost (Gruppe 1)
- **SFML_Stats_String_West.yaml** — Prognose vs. Messung: String West (Gruppe 2)
- **SFML_Stats_String_Sued.yaml** — Prognose vs. Messung: String Süd (Gruppe 3)

All UI labels are in **German**.

## Architecture

Both cards share the same base structure:

1. **Chart series (rendered on graph):** Battery charge % (green, secondary y-axis), house power consumption in kW (red), solar generation in kW (orange), plus a forecast line (grey/blue)
2. **Header-only series (`header_only` y-axis, `in_chart: false`):** Display forecast values (today, remaining, tomorrow) and metadata as text in the card header without plotting on the chart
3. **Y-axes:** `kW` (primary, power), `capacity` (secondary/opposite, battery %), `header_only` (hidden, display-only)

Key differences between the two cards:
- **Solcast** uses `data_generator` with JavaScript to map `entity.attributes.detailedForecast` into chart data points
- **Solar Forecast ML** uses `sensor.sfml_tagesprognose` (SQL-Sensor) with `data_generator` to map `entity.attributes.hourly_forecast` into full-day chart data points
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
- **SQL Integration** — `sensor.sfml_tagesprognose`, `sensor.sfml_stats_gesamt`, `sensor.sfml_stats_string_ost`, `sensor.sfml_stats_string_west`, `sensor.sfml_stats_string_sued` (manuell angelegt, siehe unten)
- **Custom** — `sensor.tt_solar_generated` (total daily solar generation)
- **Frontend** — [apexcharts-card](https://github.com/RomRider/apexcharts-card) custom Lovelace card

## SQL-Sensor "SFML Tagesprognose" manuell anlegen

Die Solar Forecast ML Card benötigt einen SQL-Sensor, der die stündlichen Prognosen aus der lokalen SQLite-Datenbank liest. Dieser Sensor muss **manuell über die HA-Oberfläche** angelegt werden (nicht per YAML).

### Anleitung

1. **Einstellungen → Geräte & Dienste → Integration hinzufügen → "SQL"** suchen und auswählen
2. Folgende Felder ausfüllen:

| Feld | Wert |
|---|---|
| **Name** | `SFML Tagesprognose` |
| **Database URL** | `sqlite:////config/solar_forecast_ml/solar_forecast.db` |
| **Column** | `state` |
| **Unit of measurement** | `kWh` |

3. **Query** (einzeilig einfügen):

```sql
SELECT ROUND(SUM(prediction_kwh), 1) as state, json_group_array(json_object('hour', target_hour, 'kwh', ROUND(prediction_kwh, 3))) as hourly_forecast FROM hourly_predictions WHERE target_date = date('now', 'localtime') ORDER BY target_hour;
```

4. Absenden — HA legt `sensor.sfml_tagesprognose` an

### Verifikation

Unter **Entwicklerwerkzeuge → Zustände** nach `sfml_tagesprognose` suchen:
- **State** = Tagessumme in kWh (z.B. `4.2`)
- **Attribut `hourly_forecast`** = JSON-Array mit 24 Einträgen: `[{"hour": 0, "kwh": 0.0}, {"hour": 1, "kwh": 0.0}, ...]`

### Hintergrund

Die bisherige Prognosekurve nutzte `sensor.solar_forecast_ml_next_hour_forecast` mit `hours_list` — diese Liste enthält nur noch verbleibende Stunden des Tages, weshalb die Kurve im Tagesverlauf schrumpfte. Der SQL-Sensor liest stattdessen alle 24 Stunden aus `hourly_predictions`, sodass die Prognosekurve den gesamten Tag abdeckt.

## SQL-Sensoren "SFML Stats" für Prognose vs. Messung

Die 4 Stats-Cards benötigen jeweils einen SQL-Sensor, der Prognose- und IST-Werte aus `solar_forecast.db` liest. Alle **manuell über die HA-Oberfläche** anlegen.

### Gemeinsame Felder

| Feld | Wert |
|---|---|
| **Database URL** | `sqlite:////config/solar_forecast_ml/solar_forecast.db` |
| **Column** | `state` |
| **Unit of measurement** | `kWh` |

### Sensor 1: SFML Stats Gesamt

**Name:** `SFML Stats Gesamt` → Entity: `sensor.sfml_stats_gesamt`

**Query:**
```sql
SELECT ROUND(COALESCE(SUM(actual_kwh), 0), 2) as state, ROUND(SUM(prediction_kwh), 2) as prediction_total, CASE WHEN COALESCE(SUM(actual_kwh), 0) = 0 AND SUM(prediction_kwh) = 0 THEN 100 WHEN COALESCE(SUM(actual_kwh), 0) = 0 OR SUM(prediction_kwh) = 0 THEN 0 ELSE ROUND(MIN(COALESCE(SUM(actual_kwh), 0), SUM(prediction_kwh)) * 100.0 / MAX(COALESCE(SUM(actual_kwh), 0), SUM(prediction_kwh)), 0) END as accuracy, json_group_array(json_object('hour', target_hour, 'pred', ROUND(prediction_kwh, 4), 'actual', actual_kwh)) as hourly_data FROM (SELECT target_hour, prediction_kwh, actual_kwh FROM hourly_predictions WHERE target_date = date('now', 'localtime') ORDER BY target_hour);
```

### Sensor 2: SFML Stats String Ost

**Name:** `SFML Stats String Ost` → Entity: `sensor.sfml_stats_string_ost`

**Query:**
```sql
SELECT ROUND(COALESCE(SUM(actual_kwh), 0), 3) as state, ROUND(SUM(prediction_kwh), 3) as prediction_total, CASE WHEN COALESCE(SUM(actual_kwh), 0) = 0 AND SUM(prediction_kwh) = 0 THEN 100 WHEN COALESCE(SUM(actual_kwh), 0) = 0 OR SUM(prediction_kwh) = 0 THEN 0 ELSE ROUND(MIN(COALESCE(SUM(actual_kwh), 0), SUM(prediction_kwh)) * 100.0 / MAX(COALESCE(SUM(actual_kwh), 0), SUM(prediction_kwh)), 0) END as accuracy, json_group_array(json_object('hour', target_hour, 'pred', ROUND(prediction_kwh, 4), 'actual', actual_kwh)) as hourly_data FROM (SELECT hp.target_hour as target_hour, ppg.prediction_kwh as prediction_kwh, ppg.actual_kwh as actual_kwh FROM hourly_predictions hp JOIN prediction_panel_groups ppg ON hp.prediction_id = ppg.prediction_id WHERE hp.target_date = date('now', 'localtime') AND ppg.group_name = 'Gruppe 1' ORDER BY hp.target_hour);
```

### Sensor 3: SFML Stats String West

**Name:** `SFML Stats String West` → Entity: `sensor.sfml_stats_string_west`

**Query:** Gleich wie String Ost, aber `ppg.group_name = 'Gruppe 2'`

### Sensor 4: SFML Stats String Sued

**Name:** `SFML Stats String Sued` → Entity: `sensor.sfml_stats_string_sued`

**Query:** Gleich wie String Ost, aber `ppg.group_name = 'Gruppe 3'`

### Verifikation

Unter **Entwicklerwerkzeuge → Zustände** nach `sfml_stats` suchen. Jeder Sensor hat:
- **State** = IST-Summe in kWh (0.0 wenn noch keine Messdaten)
- **Attribut `prediction_total`** = Prognose-Summe in kWh
- **Attribut `accuracy`** = Genauigkeit in % (MIN/MAX-Verhältnis)
- **Attribut `hourly_data`** = JSON-Array mit 24 Einträgen: `[{"hour": 0, "pred": 0.0, "actual": null}, ...]`

### Datenquellen

- **Gesamt:** Tabelle `hourly_predictions` → `prediction_kwh` (Prognose) und `actual_kwh` (IST)
- **Panel-Gruppen:** Tabelle `prediction_panel_groups` (JOIN über `prediction_id`) → `prediction_kwh` und `actual_kwh` pro Gruppe
- Mapping: **Gruppe 1 = String Ost**, **Gruppe 2 = String West**, **Gruppe 3 = String Süd**
