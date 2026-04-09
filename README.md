# Xiaomi Air Purifier - Reactive PM2.5 Control

[![Home Assistant Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue?logo=home-assistant)](https://www.home-assistant.io/docs/automation/using_blueprints/)

A Home Assistant automation blueprint that provides **reactive, real-time fan speed control** for Xiaomi Air Purifiers based on PM2.5 readings. Designed as a smarter, faster replacement for the built-in Auto mode.

## Why?

The stock Auto mode on Xiaomi Air Purifier Pro reacts slowly to pollution spikes. This blueprint:

- **Reacts instantly** to PM2.5 sensor changes (state trigger, not polling)
- **Detects rapid spikes** and temporarily boosts fan to maximum
- **Applies hysteresis** to prevent fan oscillation near thresholds
- **Enforces night mode** with configurable quiet hours and speed cap
- **Reacts to open windows** - boost to fight smog or go minimum to save the filter
- **Away mode** - reduce speed or turn off when nobody is home
- **Falls back gracefully** to a secondary sensor if the primary becomes unavailable

## Requirements

- **Home Assistant** 2024.6.0 or newer
- **Xiaomi Air Purifier** integrated via [Xiaomi Miot Auto](https://github.com/al-one/hass-xiaomi-miot) (HACS)
- A `fan` entity and a `number` entity (Favorite Level) exposed by the integration

## Installation

### Option 1: Import via URL

[![Open your Home Assistant instance and show the blueprint import dialog](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FITLCI%2Fha-air-purifier-blueprint%2Fblob%2Fmain%2Fblueprint.yaml)

### Option 2: Manual

1. Copy `blueprint.yaml` to your HA config directory:
   ```
   config/blueprints/automation/ITLCI/xiaomi_air_purifier_reactive.yaml
   ```
2. Reload automations or restart Home Assistant.

## Configuration

| Parameter | Default | Description |
|---|---|---|
| **Air Purifier** | *(required)* | The `fan` entity of your purifier |
| **Favorite Level** | *(required)* | The `number` entity for favorite level (range depends on model, e.g. 0-17 for Pro) |
| **PM2.5 Sensor** | *(required)* | Primary PM2.5 sensor (triggers instant reaction) |
| **PM2.5 Fallback** | *(empty)* | Secondary sensor, used when primary is unavailable |
| **Threshold: Moderate** | 25 ug/m3 | PM2.5 level to enter moderate zone (must be < High) |
| **Threshold: High** | 50 ug/m3 | PM2.5 level to enter high zone (must be < Very High) |
| **Threshold: Very High** | 75 ug/m3 | PM2.5 level to enter very high zone |
| **Hysteresis** | 7 ug/m3 | PM2.5 must drop this far below threshold to downshift |
| **Speed: Clean** | 1 | Fan level for clean air (below moderate) |
| **Speed: Moderate** | 5 | Fan level for moderate zone |
| **Speed: High** | 8 | Fan level for high zone |
| **Speed: Very High** | 12 | Fan level for very high zone |
| **Spike threshold** | 15 ug/m3 | Single-update PM2.5 jump that triggers boost (0 = disabled) |
| **Speed: Boost** | 17 | Fan speed during spike boost |
| **Window sensor** | *(empty)* | Binary sensor for window/door (optional) |
| **Window action** | boost | What to do when window is open: `boost` or `minimum` |
| **Presence sensor** | *(empty)* | Person or binary_sensor for away detection (optional) |
| **Away speed** | 1 | Fan level when nobody is home (0 = off) |
| **Night mode** | enabled | Whether to enforce speed cap at night |
| **Night start** | 22:30 | Quiet hours start |
| **Night end** | 07:00 | Quiet hours end |
| **Max speed at night** | 3 | Hard speed cap during night hours |

## How It Works

```
PM2.5 sensor changes value (or periodic tick)
        │
        ▼
  ┌──────────────┐ yes  ┌────────────────┐
  │ Away mode?   │─────▶│ Use away speed │──────┐
  └──────┬───────┘      └────────────────┘      │
         │ no                                    │
         ▼                                       │
  ┌──────────────┐ yes  ┌────────────────────┐  │
  │ Window open? │─────▶│ Boost or minimum   │  │
  └──────┬───────┘      │ (user configured)  │──┤
         │ no           └────────────────────┘  │
         ▼                                       │
  ┌─────────────┐     ┌──────────────┐          │
  │ Spike check │────▶│ Boost speed  │          │
  │ delta >= 15 │ yes └──────┬───────┘          │
  └──────┬──────┘            │                   │
         │ no                ▼                   │
         ▼           ┌──────────────┐           │
  ┌─────────────┐    │ Take higher  │           │
  │ Zone logic  │───▶│ of zone/boost│           │
  │ + hysteresis│    └──────┬───────┘           │
  └─────────────┘           │                   │
                            ▼                   │
                 ┌─────────────────────┐        │
                 │ Night cap (if not   │◀───────┘
                 │ window override)    │
                 └──────────┬──────────┘
                            ▼
                 ┌─────────────────────┐
                 │ Clamp to device     │
                 │ min/max range       │
                 └──────────┬──────────┘
                            ▼
                 ┌─────────────────────┐
                 │ Set level if changed│
                 └─────────────────────┘
```

### Trigger Strategy

1. **State trigger** on the primary PM2.5 sensor - reacts within seconds of a reading change
2. **Time pattern trigger** every minute - catches window, presence, and fallback sensor changes

### Hysteresis

When PM2.5 drops, the fan won't immediately downshift. It holds the current zone until PM2.5 drops below `threshold - hysteresis`. This prevents the fan from cycling up and down near zone boundaries.

### Spike Detection

If PM2.5 jumps by more than the spike delta in a single sensor update (e.g., someone starts cooking), the fan immediately goes to boost speed. On the next sensor update, normal zone logic resumes - the fan stays high if PM2.5 is still elevated, or ramps down if the spike was brief.

## License

MIT
