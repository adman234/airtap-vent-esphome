# Adding the vents to Home Assistant

Short version: **they just work.** No helpers, no template sensors, no
automations. That is the point of putting the control loop on the device.

## 1. Flash and adopt

```bash
esphome run airtap-office.yaml
```

The device joins WiFi and announces itself over mDNS. Home Assistant's ESPHome
integration should discover it under **Settings → Devices & Services**. Adopt it
and paste the API encryption key from your `secrets.yaml` when prompted.

If it does not appear, add it manually by hostname (`<name>-<mac>.local`) or IP.

## 2. What you get, automatically

| Entity | Purpose |
|---|---|
| `fan.*_airtap_fan` | The fan. On/off + speed 1–10. |
| `switch.*_auto_mode` | AUTO vs MANUAL. Turning the fan on/off from HA drops this to MANUAL. |
| `binary_sensor.*_auto_fan_demand` | The decision itself — whether AUTO currently wants the fan running. |
| `sensor.*_vent_temperature` | NTC probe in the register, °F. |
| `button.*_reset_to_auto_default` | Back to AUTO @ 40% on demand. |
| `switch.*_require_hvac_active` | Only boost while the thermostat is heating/cooling. |
| `switch.*_disable_panel_buttons` | Lock out the physical buttons. |
| `number.*_auto_fan_speed` | Resting speed. Returns to 4 every boot. |
| `number.*_auto_buffer` | °F the vent must beat the room by to be worth running. |
| `number.*_room_deadband` | Hysteresis around the thermostat setpoints. |
| `number.*_min_run_time` / `*_min_off_time` | Anti short-cycling. |
| `number.*_oled_brightness` | Display brightness. |
| `sensor.*_wifi_signal`, `button.*_esp_reboot` | Diagnostics. |

## 3. What to delete

Once the vents are adopted, remove the old scaffolding — it will fight the
device if you leave it running:

- The **template helper** carrying the `room > cool_sp and vent < (room - buffer)`
  Jinja. That logic now lives in `binary_sensor.*_auto_fan_demand`, computed
  on-device from the same inputs.
- The **"set vents to AUTO mode after Home Assistant restart"** automation.
- The **"once per day, set AUTO mode and 40%"** automation.
- Any automation that turns the vent fan on or off. Anything toggling
  `fan.*_airtap_fan` from HA is treated as a deliberate human override and will
  drop the vent into MANUAL — exactly the behaviour those automations existed to
  paper over.

## 4. What must already exist

The device imports three things from HA. All of them have to resolve.

- **A room temperature sensor**, named in the device's `room_temp_entity`
  substitution, reporting **°F**.
- **A thermostat** named in `thermostat_entity`, exposing `target_temp_high` and
  `target_temp_low`.
- That thermostat reporting **`hvac_action`** (`heating` / `cooling` / `idle`).

### The gotcha worth knowing about

`target_temp_high` and `target_temp_low` **only exist while the thermostat is in
Heat/Cool (`heat_cool`) mode.** In Cool-only or Heat-only mode a thermostat
exposes a single `temperature` attribute instead, and those two attributes go
away entirely.

If that happens, the imported sensors go unavailable, `Auto Fan Demand` reads
false, and the vent never runs. It fails silent and closed.

This was equally true of the original template helper, so it is not a regression
— but it is the first thing to check if a vent stops calling for the fan. If you
run your system in single-setpoint modes, the demand logic in
`packages/airtap-vent.yaml` needs to read `temperature` instead, which is a small
change to the `auto_demand` lambda.

## 5. Verifying it works

After a flash, on the device's OLED you should see `AUTO`, the vent and room
temperatures, and the fan speed with `CALL` / `---` showing whether AUTO wants
the fan.

In HA, check in this order:

1. `switch.*_auto_mode` is **on**. If it is off right after a boot, the mode
   guard is not doing its job — check the ESPHome log for the fan restoring a
   state at startup.
2. `number.*_auto_fan_speed` reads **4**.
3. `sensor.*_vent_temperature` is plausible, not `nan` or wildly off.
4. `binary_sensor.*_auto_fan_demand` tracks what you expect as the room drifts
   off setpoint while the system is running.

If demand is stuck off while the system is actively heating or cooling, turn off
`switch.*_require_hvac_active` and see if it starts calling. That isolates
whether your thermostat's `hvac_action` is the thing blocking it.

## 6. Optional dashboard card

```yaml
type: entities
title: Office Vent
entities:
  - entity: fan.airtap_vent_airtap_fan
  - entity: switch.airtap_vent_auto_mode
  - entity: binary_sensor.airtap_vent_auto_fan_demand
  - entity: sensor.airtap_vent_vent_temperature
  - type: divider
  - entity: number.airtap_vent_auto_fan_speed
  - entity: number.airtap_vent_auto_buffer
  - entity: number.airtap_vent_room_deadband
  - entity: button.airtap_vent_reset_to_auto_default
```

Entity IDs depend on your device name and friendly name — copy the real ones from
the device page rather than trusting the above verbatim.
