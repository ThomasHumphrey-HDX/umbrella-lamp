# UmbrellaLamp

A pair of Home Assistant automations that turn an RGB lamp into a rain indicator. Walk up to the front door and the lamp pulses a colour that tells you whether to grab an umbrella.

## How it works

### `RainWarning.yaml` – the lamp pulse

Triggered when a presence/occupancy sensor near the door turns on.

1. Fetches the hourly forecast from your weather integration.
2. Looks at the **next 3 hours** and totals the precipitation (mm) and counts the hours with a rainy condition (`rainy`, `pouring`, `lightning-rainy`).
3. Picks a rain level:

   | Level   | Condition                                   | Pulse colour | Brightness |
   |---------|---------------------------------------------|--------------|------------|
   | `heavy` | ≥ 1.0 mm **or** ≥ 2 rainy hours             | Deep blue    | 100%       |
   | `light` | ≥ 0.1 mm **or** ≥ 1 rainy hour              | Light blue   | 75%        |
   | `none`  | anything less                               | Dim orange   | 45%        |

4. Sets the lamp to a warm orange baseline, then pulses between the baseline and the rain colour 3 times.

### `RainCheck.yaml` – background rain flag

Runs every 15 minutes (and on Home Assistant startup). It totals forecast precipitation over the **next 6 hours** and turns an `input_boolean` on if it exceeds 0.3 mm, off otherwise. Handy for dashboards, notifications or other automations.

## Requirements

- Home Assistant 2024.10 or newer (uses the `triggers:` / `actions:` syntax and `weather.get_forecasts`)
- A weather integration that provides **hourly** forecasts (e.g. Open-Meteo or Met.no)
- An RGB-capable light
- A presence, occupancy or motion `binary_sensor` near the door
- An `input_boolean` helper (for `RainCheck.yaml` only)

## Setup

1. Create an `input_boolean` helper: **Settings → Devices & services → Helpers → Create helper → Toggle**.
2. Replace the placeholder entity IDs in the YAML files with your own:

   | Placeholder                   | Replace with                      |
   |-------------------------------|-----------------------------------|
   | `light.examplelight`          | Your RGB lamp                     |
   | `binary_sensor.examplesensor` | Your door presence/motion sensor  |
   | `weather.examplehome`         | Your weather entity (appears twice per file – in `target` and in the `result[...]` template) |
   | `input_boolean.examplerain`   | The helper you created in step 1  |

3. In Home Assistant go to **Settings → Automations & scenes → Create automation → Edit in YAML**, paste in each file, and save.

## Customising

- **Thresholds** – edit the `rain_level` template in `RainWarning.yaml` or the `0.3` in `RainCheck.yaml`.
- **Colours** – `baseline_color` and `pulse_color` are RGB lists.
- **Forecast window** – change `forecast[:3]` / `forecast[:6]` to look further ahead or behind.
- **Number of pulses** – change `count: 3` in the `repeat` block.
