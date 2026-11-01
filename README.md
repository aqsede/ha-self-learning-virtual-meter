# Self-Learning Virtual Power Sensors for Unmetered Switches

A Home Assistant package that turns plain wall switches into virtual power and energy sensors. It watches what happens on an upstream meter when a switch toggles, learns the load's wattage from that, and keeps learning, so swapping a bulb corrects itself without any manual reset.

> **Status: untested.** The YAML parses and the median logic has been checked in isolation, but the package has not yet run on a live Home Assistant instance. Validate it with Developer Tools and watch the first few events with debug logging on (see [Verifying](#verifying-a-new-install)).

## How it works

```
 switch toggles ON or OFF
          │
          ▼
 automation.lw_dispatch ──► script.lw_learn
                                  │
        record "before" on the reference meter
                                  │
        wait `settle` seconds, read (a1), wait 3 s, read (a2)
                                  │
        sample = |after − before|   (after = mean of a1, a2)
                                  │
        reject if any of:
          • a1 and a2 differ by more than `tol`          (unstable)
          • switch changed again since the script began   (flicker)
          • a sibling switch changed within the last 10 s (crosstalk)
          • a guard sensor is ON or changed recently      (e.g. water heater)
          • sample outside [min, max]                     (noise / standby)
                                  │ accepted
                                  ▼
        append to rolling history of last 5 samples (input_text)
        learned watts = MEDIAN of history             (input_number)
                                  │
                                  ▼
 template sensor: learned watts while switch is ON, else 0
                                  │
                                  ▼
 Riemann sum (left) ──► kWh sensor for the Energy Dashboard
```

Both turn-on and turn-off events produce samples, so each on/off cycle contributes two.

**Why median of 5:** a single bad sample (a compressor starting mid-window, say) is outvoted. If you change a bulb, the new value takes over after about 3 new samples, with no reset needed.

## Requirements

- Home Assistant with the `triggers:` / `actions:` YAML syntax (2024.10 or newer).
- Packages enabled in `configuration.yaml`:

  ```yaml
  homeassistant:
    packages: !include_dir_named packages
  ```
- At least one **reference meter** that sees the load: a dedicated power meter behind the switch, or an upstream CT clamp covering that circuit.
- The switches must be Home Assistant `switch` entities whose state changes reliably when the relay changes.

## Installation

1. Copy `virtual_lights.yaml` to `<config>/packages/virtual_lights.yaml`.
2. Edit the [configuration](#configuration) section of the script to match your entities.
3. Developer Tools → YAML → **Check configuration**, then restart Home Assistant.
4. Turn on `input_boolean.lw_debug` and toggle a learned switch a few times.
5. Add the `Virtual … Energy` sensors to **Settings → Dashboards → Energy → Individual devices**.

If your Home Assistant version rejects the `platform: integration` YAML at the bottom of the file, create the same Riemann sum sensors as UI helpers instead (method *left*, metric prefix *k*, time unit *hours*, max sub-interval 5 min).

## What gets created

| Entity | Purpose |
|---|---|
| `script.lw_learn` | Does one learning attempt for one switch event. Holds all per-switch settings. |
| `automation.lw_dispatch` | Starts the script when any learned switch toggles. |
| `input_boolean.lw_debug` | When on, every attempt is written to the Logbook with the reason. |
| `input_text.lw_<key>_samples` | Rolling history, e.g. `12.1;12.3;11.9` (max 5 values). |
| `input_number.lw_<key>_watts` | Learned watts (the median of the history). |
| `sensor.lights_ctb_net_power` | Reference signal: CT B minus every metered load. Unclamped. |
| `sensor.virtual_<room>_<load>_power` | Virtual power sensor (W). Shows `learned_watts` and `samples` as attributes. |
| `sensor.virtual_<room>_<load>_energy` | Energy (kWh) via left Riemann sum. |

A virtual power sensor is `unavailable` until its first valid sample, unless the switch is off, in which case it reads 0. This keeps unlearned loads out of the energy statistics.

## This installation

| Room | Switches | Reference |
|---|---|---|
| Bedroom 2 | `switch.bedroom_2_wall_switch_light`, `switch.bedroom_2_wall_switch_fan` | `sensor.light_fan_light_fan_power` (Shelly PM mini behind both gangs) |
| Bedroom 1 | `switch.bedroom_wall_switch_main_light`, `…_secondary_light`, `…_fan` | `sensor.lights_ctb_net_power` (shared CT B) |

- **Bedroom 2 light** is learned from the PM mini. The tolerance is tight (1 W) because the meter is dedicated.
- **Bedroom 2 fan** is *not* learned. Its power is the PM mini reading minus the light, so remote speed changes are tracked live and light plus fan always equals the meter.
- **Bedroom 1** is learned from CT B net power. The net signal is CT B minus `hisense_tugs`, `light_fan_light_fan` (Bedroom 2), `midea_all_easy_pro`, `midea_celest` and `celest_bedroom_3`.
- **Bedroom 1 fan** uses a DC motor with remote speed control, which Home Assistant cannot see. See [Limitations](#limitations).

## Configuration

Everything configurable lives in two dictionaries under `script.lw_learn` → `variables:`.

### `meters`

```yaml
meters:
  ctb:
    ref: sensor.lights_ctb_net_power   # the reference power sensor
    tol: 3.0                           # max W difference between the two "after" readings
    switches:                          # EVERY switch behind this meter
      - switch.bedroom_wall_switch_main_light
      - ...
    guards:                            # reject samples while these are on / just changed
      - binary_sensor.water_heater_estatus
      - binary_sensor.microwave_estatus
```

- `switches` must list **every** switch behind that meter, whether or not it is learned and even if it has its own meter. A toggle on any of them near the sample window discards the sample. Missing switches here is the most common cause of bad learning.
- `guards` is for loads you cannot subtract and that would swamp a small delta.

### `lights`

```yaml
lights:
  switch.bedroom_wall_switch_main_light:
    key: b1_main      # helper names: input_text.lw_b1_main_samples, input_number.lw_b1_main_watts
    meter: ctb        # which entry in `meters`
    min: 3            # accepted delta range in W
    max: 80
    settle: 6         # seconds before the first "after" reading
    tol: 4.0          # optional, overrides the meter tolerance
```

### Tuning

| Setting | Raise it when | Lower it when |
|---|---|---|
| `min` | Standby draw or meter noise gets learned as a "light". Keep it above the meter's noise floor. | You need to learn very small loads (only works on a precise dedicated meter). |
| `max` | A real load is rejected as too big. | Other loads leak into samples. Keep every `max` below each `input_number` max (200 W). |
| `settle` | Samples vary because a driver or motor is still ramping (fans: 10 s or more). | Samples are stable and you want faster learning. |
| `tol` | Too many `stable=False` rejections on a noisy CT. | You want stricter agreement between readings. |

## Adding another switch

1. **Helpers.** Add an `input_text` (`max: 255`) named `lw_<key>_samples` and an `input_number` (0 to 200, step 0.1) named `lw_<key>_watts`.
2. **Script.** Under `lights:` add an entry for the switch. Add the switch to its meter's `switches` list. Add it to the CT B list even if it has its own meter.
3. **Automation.** Add the switch to the trigger `entity_id` list in `lw_dispatch`.
4. **Sensors.** Copy one of the existing virtual power templates and one `platform: integration` entry, and change the names and entities.

## Verifying a new install

1. Turn on `input_boolean.lw_debug`.
2. Toggle one learned switch on, wait about 15 seconds, toggle it off, wait again. Keep other loads quiet.
3. Open **Logbook** and look for entries from "Learned Watts". Each line shows:

   ```
   ACCEPTED (on): sample=11.8W before=412.3 a1=424.0 a2=424.3 stable=True still_same=True crosstalk=False range=3-80
   ```
4. After a few accepted samples, check `input_number.lw_<key>_watts` against the label on the bulb or driver.
5. Leave debug on for a day. If rejections dominate, use the reasons in the log to adjust `settle`, `tol`, `min` or the `switches` list.

### Seeding, resetting and overriding

- **Seed a starting value:** write something like `10;10;10` into `input_text.lw_<key>_samples`. Real samples then replace it.
- **Restart learning for a switch:** clear its `input_text.lw_<key>_samples`.
- **Overriding the watts directly does not stick.** `input_number.lw_*_watts` is overwritten by the next accepted sample. Edit the samples helper instead.

## Limitations

- **Dimmable lights and variable loads** have no single wattage. This is for on/off loads.
- **Measurement floor.** A load below the reference meter's noise floor (a few watts on a clamp) won't learn on that reference.
- **Remote-controlled fans.** Home Assistant can't see speed set by a remote. The virtual sensor shows the learned typical running power whenever the wall switch is on, even if you have stopped the fan with the remote, which overstates it. Turn-on events usually show only standby (below `min`) and are rejected. Turn-off events while running are the useful ones.
- **Scenes.** Several lights toggled within about 10 s of each other produce no samples (crosstalk guard). This is intentional.
- **Meter timing.** Separate meters update at slightly different moments. Subtracting one from another can glitch for a second or two when a big load changes. The stability check and the median absorb most of it, but not all.
- **Bedroom 2 fan after a bulb swap.** The fan reads off by the wattage difference until about 3 new light samples move the median. Light plus fan always matches the PM mini.
- **"Before" reading.** It is taken when the script starts. If the meter already published the change, the sample is low. The median makes this a minor accuracy cost, not a failure.

## Design notes

- The `still_same` check requires that the switch's `last_changed` is not newer than the script start. This catches on-off-on flicker and unavailable-and-back, which a plain state comparison misses.
- `max_sub_interval` on the energy sensors is deliberate. Without it, a long ON period is booked in one lump at switch-off, which skews hourly bars in the Energy Dashboard.
- The learned history is stored with `;` as the separator. A comma-separated string can be parsed by Home Assistant's template engine as a tuple.
- The `lights_watt_baseline` helper in the main configuration is a fixed guess and is not used here. Once enough lights are learned, the sum of the **unmetered** virtual lights (leave out Bedroom 2, whose PM mini is already subtracted) can replace it in the "Shelly CT B Remaining Power" template for better water-heater and microwave detection.

## Possible future improvement

If the 10 s crosstalk lookback ever rejects too many useful scene toggles, the structural fix is a reference-stability signal (for example a Statistics helper or a latching template binary sensor meaning "the reference was within tolerance for the last few seconds before this toggle") that the script gates on instead of the lookback. A shorter lookback (`t0 - 6`) is the simpler compromise to try first.
