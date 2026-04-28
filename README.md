# 🌱 Garden Watering Automation (Home Assistant)

## Overview

This repository contains the Home Assistant configuration for an automated garden watering system.

The system controls a Zigbee water timer (`switch.water_timer`) based on:

* Soil moisture readings
* Weather forecast (Met.no)
* Time-of-day constraints (dawn/dusk only)
* Safety and failover logic

---

## Repository Structure

```
.
├── configuration.yaml          # Main HA config — includes automations & input_booleans
├── input_boolean.yaml          # Run-tracking helpers (morning / evening flags)
└── automations/
    ├── a1_morning_watering.yaml             # Soil-based watering at 06:00
    ├── a2_evening_watering.yaml             # Soil-based watering at 20:00
    ├── a3_morning_fallback.yaml             # Fallback watering at 06:00 (sensor failure)
    ├── a4_evening_fallback.yaml             # Fallback watering at 20:00 (sensor failure)
    ├── a5_safety_shutoff.yaml               # Hard safety shutoff watchdog
    ├── a6_sensor_failure_notification.yaml  # Mobile alert when sensor is unavailable
    └── a7_reset_watering_flags.yaml         # Midnight reset of daily run-tracking flags
```

---

## Entities

### Water Control

| Entity                                | Description            |
| ------------------------------------- | ---------------------- |
| `switch.water_timer`                  | Main irrigation valve  |
| `sensor.water_timer_volume_flow_rate` | Flow rate (diagnostic) |
| `sensor.water_timer_battery`          | Battery level          |

### Soil Sensor

| Entity                             | Description       |
| ---------------------------------- | ----------------- |
| `sensor.soil_sensor_soil_moisture` | Soil moisture (%) |
| `sensor.soil_sensor_temperature`   | Soil temperature  |
| `sensor.soil_sensor_battery`       | Battery level     |

### Weather

| Entity         | Description              |
| -------------- | ------------------------ |
| `weather.home` | Met.no forecast provider |

---

## Thresholds & Configuration

| Parameter                    | Value      |
| ---------------------------- | ---------- |
| `dry_soil_threshold`         | 35 %       |
| `wet_soil_threshold`         | 55 %       |
| `watering_duration_normal`   | 10 minutes |
| `watering_duration_fallback` | 5 minutes  |
| `sensor_stale_timeout`       | 6 hours    |
| `morning_time`               | 06:00      |
| `evening_time`               | 20:00      |
| `battery_minimum`            | 20 %       |

---

## Automations

| ID  | Name                              | Trigger | Description                                               |
| --- | --------------------------------- | ------- | --------------------------------------------------------- |
| A1  | Morning Watering (Soil-Based)     | 06:00   | Primary watering — soil moisture, weather & battery check |
| A2  | Evening Watering (Soil-Based)     | 20:00   | Same as A1 but in the evening window                      |
| A3  | Morning Fallback Watering         | 06:00   | Fallback when soil sensor is unavailable / stale          |
| A4  | Evening Fallback Watering         | 20:00   | Same as A3 but in the evening window                      |
| A5  | Safety Shutoff                    | State   | Forces valve OFF if it has been ON for ≥ 10 minutes       |
| A6  | Sensor Failure Notification       | Template| Mobile alert when sensor is unavailable or stale          |
| A7  | Reset Daily Watering Flags        | 00:00   | Resets morning/evening run-tracking flags each midnight   |

---

## Operating Modes

### Primary Mode (Soil-Based — A1 / A2)

Watering is allowed only when **all** of the following are true:

1. Soil moisture < 35 %
2. Soil sensor is live (not `unknown`/`unavailable` and updated within 6 h)
3. No rain forecast tomorrow
4. Water timer battery > 20 %
5. Soil sensor battery > 20 %
6. The window (morning / evening) has not already been used today

### Fallback Mode (Sensor Failure — A3 / A4)

Triggered when the soil sensor is `unknown`, `unavailable`, `none`, or stale (> 6 h old).

Additional restrictions:
* No rain forecast **today or tomorrow** (both checked)
* Shorter run time: 5 minutes instead of 10

### Safety Watchdog (A5)

Hard limit: if `switch.water_timer` is ON for ≥ 10 minutes, it is forced OFF.
This catches stuck valves, HA restarts mid-run, and manual activation.

---

## Failure Modes

| Failure                  | Behaviour                               |
| ------------------------ | --------------------------------------- |
| Soil sensor offline      | Switch to fallback mode (A3/A4) + alert |
| Weather unavailable      | Watering blocked (fail safe)            |
| HA restart mid-run       | Safety watchdog (A5) stops watering     |
| Valve stuck ON           | Safety shutoff triggers after 10 min    |
| Battery low              | Watering blocked by battery conditions  |

---

## Installation

1. Copy `configuration.yaml` and `input_boolean.yaml` into your HA config directory.
2. Copy the entire `automations/` directory into your HA config directory.
3. Reload automations in HA (Developer Tools → YAML → Automations) or restart HA.

> If you already use an `automation:` key in `configuration.yaml`, merge the
> `!include_dir_merge_list automations/` directive with your existing setup.

### Customising the notification service

Automations A3, A4, and A6 send alerts via `notify.notify` (HA's built-in
generic notification service).  If you want push notifications to a specific
phone, replace `notify.notify` with your device-specific service name, e.g.
`notify.mobile_app_my_phone`.