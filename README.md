# Tesla Charge Scheduler for Home Assistant

A smart Tesla charging automation for Home Assistant that schedules charging based on departure time, battery target, and real-world conditions including battery protection, software updates, and location-aware voltage detection.

> **Note: `scylla` is my car's name** (a Tesla Model 3 Long Range, 75 kWh, NMC). Every entity ID in these files (`sensor.scylla_…`, `number.scylla_…`, `input_number.scylla_…`, …) and the automation names and notifications use it. You need to replace `scylla` / `Scylla` with your own car's name — see *Before you start* in the Setup section.

---

## Features

- **Departure-based scheduling** — calculates the exact time to start charging so the car is ready at your departure time
- **Weekday/weekend departure defaults** — automatically switches between weekday and weekend departure times at 18:00
- **Battery protection** — for NMC/NCA chemistry, caps charging at 42% (when limit = 50%) or 47% (when limit = 51%) to reduce degradation (NMC/NCA only — turned off for LFP, see *Battery chemistry*). The car won't accept limits below 50% so the charger is turned off at the target %
- **Immediate charge when below protection cap** — if the car drops below the cap minus a small hysteresis (42% − 2% = 40% by default) overnight, charging starts immediately regardless of departure time
- **Smart resume after protection cap** — if the protection charge will finish before the scheduled departure charge, it stops at 42% and resumes at the scheduled time; otherwise charges straight to the target
- **Location-aware voltage detection** — uses GPS + OpenStreetMap to detect the country when plugged in and sets the correct slow/fast voltages automatically
- **Slow/Fast charging speed selection** — UI dropdown with 15A breaker (12A), 20A breaker (16A), and Fast options
- **Target range input** — set a desired range in km or mi and the target % is calculated automatically; mutually exclusive with target % (last one touched wins)
- **Amperage handling** — arriving home resets the amperage to your default (27 A); when the car wakes up at home the UI amperage is pushed to the car (capped to the car's reported maximum); away from home the car's value is only mirrored to the UI and never written; implausible readings above a sanity ceiling are ignored
- **Software update handling** — if a software update starts mid-charge, pauses charging and resumes automatically when the update finishes (only if the pause was caused by the update)
- **UI dashboard card** — shows charge time, start time, current range, status badge, history graph, and NMC warning
- **Persistent helpers** — all settings survive HA restarts
- **Works while the car sleeps** — the last valid battery % is stored, so the schedule is still computed when the car is asleep or after an HA restart

---

## Files

| File | What it contains | Where it goes |
|---|---|---|
| `tesla-smart-charge-scheduler.yaml` | Main charge scheduler automation | New automation → Edit in YAML (paste) |
| `tesla-smart-charge-helper.yaml` | Helper automation: UI sync, amps handling, speed changes, update handling, battery level backup | New automation → Edit in YAML (paste) |
| `input_number.yaml` | Number helpers (battery capacity, voltages, amps, protection limits, last known battery…) | `/config/input_number.yaml` * |
| `input_select.yaml` | Charging speed dropdown | `/config/input_select.yaml` * |
| `input_boolean.yaml` | Flags (protection charge active, update in progress) | `/config/input_boolean.yaml` * |
| `input_datetimes.yaml` | Departure time | `/config/input_datetimes.yaml` * |
| `rest.yaml` | OpenStreetMap country-code sensor (and a weather sensor) | `/config/rest.yaml` * |
| `template.yaml` | Template sensors (safe battery level, charge target, charge time, start time, voltage map…) | `/config/template.yaml` * |
| `ui-km.yaml` | Dashboard card (km) | Dashboard → Add card → Manual (paste) |
| `ui-mi.yaml` | Dashboard card (miles) | Dashboard → Add card → Manual (paste) |

\* **Config files:** if you don't already have that file in `/config`, copy it there and add its include from Step 1. If you already have one (for example your own `template.yaml`), merge the entries from this file into yours instead of overwriting it.

---

## Requirements

- Home Assistant 2023.x or later
- Tesla integration (official or custom) and TeslaMate via MQTT, providing the following entities (replace `scylla` with your car name):
  - `sensor.scylla_usable_battery_level`
  - `number.scylla_charge_limit`
  - `sensor.scylla_charge_limit_soc`
  - `number.scylla_charging_amps`
  - `switch.scylla_charger`
  - `binary_sensor.scylla_charger` (cable connected)
  - `binary_sensor.scylla_charging` (actively charging)
  - `sensor.scylla_charger_voltage`
  - `sensor.scylla_charger_phases`
  - `sensor.scylla_time_to_full_charge`
  - `sensor.scylla_est_battery_range` (TeslaMate MQTT, always in km; `template.yaml` derives the miles version)
  - `sensor.scylla_charging_state` (TeslaMate)
  - `device_tracker.scylla_location` (home/away, plus the latitude/longitude used for voltage detection)
  - `update.scylla_software_update`
- OpenStreetMap Nominatim REST sensor (see `rest.yaml`)

---

## Setup

### Before you start — replace `scylla` with your car's name

These files were written for a car named **Scylla**, so the entity IDs are `sensor.scylla_…`, `input_number.scylla_…` and so on. Home Assistant builds the car's entity IDs from the name of your car, so yours will differ (for example `sensor.mycar_usable_battery_level`).

1. In Home Assistant open **Developer Tools → States**, search for your car's name and note the prefix it uses.
2. Check that your car's entities exist under that prefix (see the list in *Requirements*). Names can differ slightly between Tesla integrations and TeslaMate, so adjust any suffix that doesn't match.
3. Replace the name in **every** file before you copy or paste anything, for example from a terminal in the folder containing these files:

   ```bash
   sed -i 's/scylla/mycar/g; s/Scylla/MyCar/g' *.yaml
   ```

   On macOS, `sed` needs an empty backup suffix after `-i`:

   ```bash
   sed -i '' 's/scylla/mycar/g; s/Scylla/MyCar/g' *.yaml
   ```

   The lowercase form covers entity IDs and `unique_id`s; the capitalised form covers friendly names, the automation names and notification text.

The helpers and template sensors defined in these files are created with the new name, so they stay consistent as long as the replace is done everywhere. Throughout this README `scylla` stands for your car's name.

### Step 1 — Add to `configuration.yaml`

```yaml
input_number: !include input_number.yaml
input_select: !include input_select.yaml
input_boolean: !include input_boolean.yaml
input_datetime: !include input_datetimes.yaml
template: !include template.yaml
rest: !include rest.yaml
```

If you already have any of these includes, keep yours and merge the contents of the matching file from this repo into it.

### Step 2 — Add REST sensor

Add the contents of `rest.yaml` to `/config/rest.yaml` (merge if you already have one). This creates `sensor.scylla_location_country_code` using the car's GPS coordinates and OpenStreetMap.

### Step 3 — Add template sensors

Add the contents of `template.yaml` to `/config/template.yaml` (merge if you already have one). This creates:
- `sensor.scylla_usable_battery_level_safe` — always-available battery %; falls back to `input_number.scylla_last_known_battery` when the car sleeps
- `sensor.scylla_est_battery_range_mi` — estimated range in miles, converted from the km sensor `sensor.scylla_est_battery_range` (miles users: the scheduler, helper and card use it when `range_unit` is `mi`)
- `sensor.scylla_charge_limit` — always-available charge limit (fallback when car sleeps)
- `sensor.scylla_charging_amps` — always-available charging amps
- `sensor.scylla_charge_target_pct` — actual charge target after protection logic
- `sensor.scylla_charge_time` — estimated charge duration
- `sensor.scylla_charge_start_at` — scheduled charge start time
- `sensor.scylla_charger_1percent_increase_time` — minutes per 1% at current settings
- `sensor.ev_voltage_map` — world voltage map used by automations

The file also contains a few optional extras (range in miles, return-home battery %, climate suggestions). The return-home line on the dashboard card needs the Waze Travel Time integration (`sensor.waze_scylla_to_home`).

### Step 4 — Restart Home Assistant

Required to register all new helpers and template sensors.

### Step 5 — Create automations

In HA: **Settings → Automations → + Create automation → ⋮ → Edit in YAML**

Paste `tesla-smart-charge-scheduler.yaml`, save. Repeat for `tesla-smart-charge-helper.yaml`.

Keep time values quoted (e.g. `'00:01:00'`, `'06:00:00'`) — some YAML parsers read an unquoted `00:01:00` as the number 60, which Home Assistant rejects as a trigger time.

### Step 6 — Add dashboard card

In your dashboard, add a **Manual card** and paste either `ui-km.yaml` or `ui-mi.yaml` depending on your preferred range unit.

---

## Configuration

All configuration is in the `variables:` block of `tesla-smart-charge-scheduler.yaml`:

```yaml
variables:
  range_unit: "km"                    # "km" or "mi"
  default_target_pct: 50              # Target % to reset to after each session
  default_departure_time_weekday: "06:00:00"
  default_departure_time_weekend: "08:00:00"
  default_charging_amps: 27           # Amperage to reset to after each session
  enable_battery_protection: true     # true for NMC/NCA, false for LFP (see Battery chemistry)
  protection_limit_low_pct: 42        # Charge cap when limit = 50%
  protection_limit_mid_pct: 47        # Charge cap when limit = 51%
  notification_target: "notify.notify"  # Notify service for all notifications (also set in the helper)
  voltage_override_slow: 0            # Set > 0 to override auto-detected slow voltage
  voltage_override_fast: 0            # Set > 0 to override auto-detected fast voltage
```

`tesla-smart-charge-helper.yaml` has two variables of its own:

```yaml
variables:
  default_amps_fallback: 27           # Only used when an amps helper/sensor is unknown/unavailable
  notification_target: notify.notify  # Keep equal to notification_target in the scheduler
```

`default_amps_fallback` is not the default charging amps (that is `default_charging_amps` above, stored in `input_number.scylla_default_charging_amps`). It only keeps templates from erroring when a value is unavailable (car asleep, HA just restarted). Keep it equal to `default_charging_amps`.

`notification_target` appears in both automations and every notification goes through it, so keep the two values equal. See *How notifications are sent* below.

### Battery chemistry

Check your car's touchscreen: **Controls → Software → Additional Vehicle Information**

- **NMC/NCA** (most Model 3 LR/Performance, Model S, Model X) → `enable_battery_protection: true`
- **LFP** (Model 3 SR/Standard Range made after 2021) → `enable_battery_protection: false`

NMC/NCA cells last longer when they are not kept full, and the car won't accept a charge limit below 50%. With protection **on**, the limit slider's lowest settings (50% and 51%) mean "keep the battery low": the scheduler targets a lower cap and turns the charger off when the battery reaches it. With protection **off** the limit is used exactly as set.

| | Protection on (NMC/NCA) | Protection off (LFP) |
|---|---|---|
| Limit set to 50% | Charges to **42%** (`protection_limit_low_pct`) | Charges to 50% |
| Limit set to 51% | Charges to **47%** (`protection_limit_mid_pct`) | Charges to 51% |
| Limit 52% or more | Charges to the limit | Charges to the limit |
| Battery below the cap minus the hysteresis (40% by default) | Charges immediately to 42%, then resumes at the scheduled time | No immediate charge — normal departure-based schedule |
| Charger stopped at the cap | Yes, by the helper automation | No |
| "Charging for battery protection" badge and NMC/NCA warning (> 90%) on the card | Shown | Hidden |

The scheduler copies the setting to `input_boolean.scylla_battery_protection_enabled` every time it runs, so the helper automation, the template sensors and the dashboard card all follow it. The helper starts as `on` and only changes once the scheduler has run, so protection is never silently off after a fresh install or a restart.

For LFP, also change `default_target_pct` (50 by default): it is the limit the scheduler restores each time the car leaves home. LFP owners typically want a higher default, such as 100.

### Slow charger breaker

In the UI card, set **Charging speed** to match your home charger:
- `Slow 12A (15A breaker)` — standard North American 15A circuit
- `Slow 16A (20A breaker)` — North American 20A circuit
- `Fast` — Level 2 charger (240V North America, 230V Europe, etc.)

### Two-car households

Entity IDs are not parameterised (there is no `car_name` variable). Duplicate both automations, the dashboard card and the helper/template entries, and find-and-replace `scylla` with your second car's entity prefix everywhere.

### Battery capacity

Update `input_number.scylla_battery_capacity_kwh` in your helpers to match your car:
- Model 3 LR: 75 kWh
- Model 3 SR: 57.5 kWh
- Model Y LR: 75 kWh
- Model S: 100 kWh

As your battery degrades over time, adjust this value to keep charge time estimates accurate.

---

## How It Works

### Daily flow (normal night at home)

1. **18:00** — Departure time updates to tomorrow's default (weekday 06:00 or weekend 08:00), but only if no custom charge is planned
2. **00:01** — Main automation recalculates and schedules charge start time
3. **HH:MM** — Charger turns on at the calculated start time
4. **Leaving home** — settings reset to defaults (target %, departure time, target range, charging speed, amperage helper). Skipped while a battery-protection charge is active
5. **Arriving home** — amperage is reset to the default and pushed to the car (see *Arriving home and waking up* below)

### Battery protection flow

1. Car is plugged in and drops below the cap minus the hysteresis (`input_number.scylla_protection_hysteresis`, default 2 → below 40%), from vampire drain or driving. The hysteresis stops the car's small post-charge SOC drop from restarting charging
2. If the protection charge can finish before the scheduled departure charge → `input_boolean.scylla_protection_charge_active` is turned on and charging starts now
3. The helper automation stops the charger once the battery reaches the cap (42%, or 47% when the limit is 51%), turns the flag off, and the scheduler resumes at the scheduled time for the remainder
4. If there is not enough time to stop at the cap → charges straight to the target %
5. If you raise the target while a protection charge is running, the helper re-checks the timing and cancels the protection stop if needed

### Software update flow

1. Update starts mid-charge → helper detects the `in_progress` attribute changing from `false` to `true`
2. `input_boolean.scylla_update_in_progress` is turned on, the charger turns off, notification sent with update percentage
3. Update finishes → helper detects `in_progress` changing from `true` to `false`
4. Only if the flag is on: flag cleared, waits 3 minutes for the car to reboot
5. Charger turns back on, notification sent

An update that finishes while no charge was paused by the helper does nothing, so a restart or an unrelated update can't start charging outside the schedule.

### Arriving home and waking up

1. Car arrives home → amperage helper set to the default (`input_number.scylla_default_charging_amps`); if the car is awake and reports a different value, it is pushed to the car immediately
2. Car asleep on arrival → when it wakes at home and reports a different value, the UI amperage is pushed to the car
3. Car away from home → the car's value is only mirrored to the UI, never written (the car may store its charge current per location)
4. Amperage changed in the Tesla app while the car is awake at home → mirrored back to the UI
5. Edits of the amperage in the UI are pushed to the car immediately only when it is awake (`sensor.scylla_state` is `online` or `charging`), so a sleeping car is not woken up by the slider. You get a notification when an edit is deferred. The value is sent when the car comes online at home, or the scheduler pushes it right before a charge starts

### While the car sleeps

When the car sleeps, `sensor.scylla_usable_battery_level` becomes unavailable. `sensor.scylla_usable_battery_level_safe` returns the live value when valid, otherwise `input_number.scylla_last_known_battery`, which the helper updates on every valid reading and which survives restarts. Until the first valid reading it is 0 and the scheduler skips.

### Location-aware voltage

1. Car plugged in → REST API called to get country code from GPS
2. Country looked up in hardcoded world voltage map
3. `input_number.scylla_slow_voltage` and `input_number.scylla_fast_voltage` set automatically
4. Charge time estimates in UI update immediately
5. Override with `voltage_override_slow` / `voltage_override_fast` if needed

---

## Dashboard Card

The card shows (in order):
- **Status badge** — 🔴 Not connected / 🔋 Charging for battery protection / 🟢 Charging / 🟡 Connected not charging
- **Start at** — scheduled charge start time, or "Now" if below protection cap
- **Charge time** — estimated duration based on current speed/amps/voltage
- **Departure time** — editable
- **Current battery %** — live from car
- **Target battery %** — actual target after protection logic
- **Set target %** — editable (50-100%)
- **Current range** — live from car
- **Set target range** — enter a range and target % is calculated automatically
- **Amperage setting (car)** — the amps setting held by the car (or the helper value while the car sleeps); it is a setting, not a measured current, so it does not drop to 0 when idle
- **Set amperage** — editable
- **Charging speed** — dropdown (Slow 12A / Slow 16A / Fast)
- **Voltage/phase info** — slow and fast voltages with phase count, plus live voltage when charging
- **Range → % conversion** — shown when target range is set (e.g. 📍 350 km → 72%)
- **NMC/NCA warning** — 🔴 shown when target > 90% (only when battery protection is enabled)
- **Charge history** — 12-hour graph of charger state
- **Return home** — only when the car is away: distance, travel time and the battery % needed to get home (needs Waze Travel Time)

---

## Helpers Created

### `input_number.yaml`

| Helper | Purpose | Default |
|---|---|---|
| `scylla_battery_capacity_kwh` | Battery size for time calculations | 75 |
| `scylla_target_range_km` | Target range in km (0 = use % instead) | 0 |
| `scylla_target_range_mi` | Target range in mi (0 = use % instead) | 0 |
| `scylla_slow_voltage` | Auto-set slow voltage from GPS | 120 |
| `scylla_fast_voltage` | Auto-set fast voltage from GPS | 240 |
| `scylla_protection_low` | Protection cap at 50% limit | 42 |
| `scylla_protection_mid` | Protection cap at 51% limit | 47 |
| `scylla_protection_hysteresis` | Gap below the cap before an immediate protection charge starts | 2 |
| `scylla_set_charge_limit` | Persistent charge limit (restored after restart, minimum 50) | 50 |
| `scylla_set_charging_amps` | Persistent amperage (restored after restart) | 27 |
| `scylla_default_charging_amps` | Default amps after session reset and on arrival home | 27 |
| `scylla_amps_sanity_max` | Amps readings above this are ignored as glitches | 40 |
| `scylla_last_known_battery` | Last valid battery % (fallback while the car sleeps) | — |

### `input_select.yaml`

| Helper | Options |
|---|---|
| `scylla_charging_speed` | Slow 12A (15A breaker) / Slow 16A (20A breaker) / Fast |

### `input_boolean.yaml`

| Helper | Purpose |
|---|---|
| `scylla_protection_charge_active` | Flag: battery-protection charge running; stop at the cap and resume at the scheduled time |
| `scylla_update_in_progress` | Flag: software update paused charging session |
| `scylla_battery_protection_enabled` | Mirror of `enable_battery_protection`, written by the scheduler on every run; read by the helper automation, the template sensors and the dashboard card |

### `input_datetimes.yaml`

| Helper | Purpose | Default |
|---|---|---|
| `scylla_departure_time` | Departure time for scheduling | 06:00 |

---

## Notifications

### How notifications are sent

Every notification uses the service in `notification_target` (default `notify.notify`). When the Home Assistant Companion app is installed on a phone, the `mobile_app` integration provides the service automatically. `notify.notify` sends to **every** phone registered with the Companion app, and each phone also gets its own service named `notify.mobile_app_<device name>`.

To send to a single phone, set `notification_target: notify.mobile_app_<device name>` in **both** automations. To find the exact name, open **Developer Tools → Actions** and type `notify.`. If you also use another notification integration, check there that `notify.notify` points where you expect. `notify.yaml` is only for other notification platforms (email, Telegram, …).

### What is sent

The system sends phone notifications for:
- Charge scheduled (with start time and slow/fast duration estimates)
- Charge starting now (when start time already passed)
- Battery below protection cap — charging immediately
- Not enough time to stop at protection cap — charging straight to target
- Charging paused for software update (with update percentage)
- Charging resumed after software update
- Cable not connected at charge start time
- No voltage detected after 5 minutes (charge may have failed)
- Charging confirmed with actual voltage, amps, phases, and car's own time estimate
- Amperage capped to car maximum
- Implausible amps reading ignored (above the sanity ceiling)
- Battery protection cancelled, or target updated during a protection charge
- Charging paused at the protection cap

---

## Troubleshooting

**Charge time is wrong**
- Check `input_number.scylla_set_charging_amps` and `input_select.scylla_charging_speed` match your charger
- Verify `input_number.scylla_fast_voltage` / `scylla_slow_voltage` are correct for your location
- Make sure `input_number.scylla_target_range_km` is 0 if you want to use target % instead

**Car not starting charge**
- Check automation trace: Settings → Automations → Scylla - Smart Charge Scheduler → Traces
- Verify `switch.scylla_charger` exists and can be controlled
- The car must be plugged in — `binary_sensor.scylla_charger` must be `on`

**Settings lost after restart**
- All `input_number`, `input_select`, `input_boolean`, and `input_datetime` helpers persist automatically
- If values reset, check HA logs for errors loading helper files

**Unknown entities in card**
- Wait for car to wake up — template sensors fall back to helpers when car is asleep
- Check Developer Tools → States for the entity values

**Voltage not auto-detecting**
- Verify `sensor.scylla_location_country_code` exists and has a valid ISO country code
- Check the REST sensor is configured in `rest.yaml` with correct latitude/longitude entities
- Use `voltage_override_slow` / `voltage_override_fast` in `tesla-smart-charge-scheduler.yaml` to hardcode values

**Car charges past 42% during protection**
- Verify `input_boolean.scylla_protection_charge_active` was set to `on` by the main automation
- Check automation trace for the helper — `battery_level_changed` trigger should fire as battery rises
- The Tesla integration update interval affects how quickly the stop fires — typically within 1-2%

**Range shows unknown, or a range target is ignored**
- Check that `sensor.scylla_est_battery_range` exists and is in km (miles users: also `sensor.scylla_est_battery_range_mi`) in Developer Tools → States
- A template sensor keeps the entity ID it was first registered with, even if you later change its `name`. If an old entry such as `sensor.scylla_estimated_range_mi` exists, rename it in Settings → Entities (or delete it and restart)

**Schedule skipped while the car sleeps or after a restart**
- Check `sensor.scylla_usable_battery_level_safe` in Developer Tools → States — it should hold the last known battery %, not 0
- If it is 0, check that `input_number.scylla_last_known_battery` has a value; it is filled on the first valid battery reading after you install the helper automation

**Amperage is wrong after getting home**
- Check the helper automation trace for the `arrived_home` and `amps_changed` runs
- The helper's `Wake-up at home` step only fires when the car's amps entity goes from unknown/unavailable to a value; if your integration keeps the last value while asleep, the arrival push still applies when the car is awake

**Automation will not save: "Expected HH:MM, HH:MM:SS…"**
- Quote time values (`'00:01:00'`, not `00:01:00`)

---

## World Voltage Reference

Voltages are auto-detected from GPS. Common values:

| Region | Slow | Fast |
|---|---|---|
| North America (CA, US, MX) | 120V | 240V |
| Europe (most countries) | 230V | 400V |
| Japan | 100V | 200V |
| UK | 230V | 230V |
| Australia | 230V | 230V |

Override with `voltage_override_slow` and `voltage_override_fast` in `tesla-smart-charge-scheduler.yaml` if auto-detection is wrong for your location.
