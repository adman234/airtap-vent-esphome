# AirTap Vent — ESPHome

ESPHome firmware for AC Infinity AirTap T-series register booster fans whose
controller has been replaced with a Seeed XIAO ESP32-C6.

The fan runs on PWM, the original four panel buttons and SSD1306 OLED are kept,
an NTC probe reads the register temperature, and the whole thing talks to Home
Assistant over WiFi using the native API.

The differentiator versus a plain ESPHome fan config is that **the vent decides
for itself when to run.** There are no Home Assistant helpers, template sensors,
or automations involved.

## Control model

Home Assistant supplies three inputs — room temperature, the thermostat's two
setpoints, and what the thermostat is currently doing. Everything else is
computed on-device, so the vent keeps working on its last known inputs if HA
restarts.

AUTO calls for the fan when the system is actively moving conditioned air **and**
boosting this register would help:

```
room > cool_setpoint  AND  vent < (room - buffer)     -> cooling assist
room < heat_setpoint  AND  vent > (room + buffer)     -> heating assist
```

with hysteresis around the setpoints and minimum run/off times so it does not
short-cycle.

### Resting state

Every boot lands in **AUTO at 40%** (speed 4 of 10). `auto_mode` and `auto_speed`
are deliberately not restored across reboots; tuning values like the buffer,
deadband and minimum times are, because only a person ever changes those.

Exactly three things drop the vent to MANUAL:

| Action | Result |
|---|---|
| Panel Power / Up / Down button | MANUAL, fan follows the button |
| Panel Mode button | toggles AUTO ↔ MANUAL |
| Anyone turning the fan on/off from Home Assistant | MANUAL |

Nothing else can. A reboot, or the **Reset To Auto Default** button, returns to
AUTO at 40%.

## Layout

```
airtap-office.yaml          # complete, self-contained config for one vent
airtap-guest.yaml           # same, with its own substitutions block
secrets.yaml.example
docs/home-assistant.md      # adopting the devices in HA, and what to delete
```

Each device file is standalone — drop it straight into your ESPHome directory,
no includes or packages needed. Everything below the `substitutions:` block is
byte-identical between the two files, so adding a vent means copying one and
editing the top eight lines.

The tradeoff is deliberate: a change to the shared logic has to be applied to
every device file. If you grow past three or four vents, move the body back into
`packages/airtap-vent.yaml` and have each device file `!include` it.

## Getting started

1. `cp secrets.yaml.example secrets.yaml` and fill it in. The real `secrets.yaml`
   is gitignored.
2. Set `device_address` in the per-device file to whatever the unit answers on
   **right now**. After a rename flash lands, update it to
   `<name>-<mac suffix>.local`.
3. `esphome run airtap-office.yaml`
4. Follow [`docs/home-assistant.md`](docs/home-assistant.md) to adopt it and to
   remove the helpers and automations it replaces.

## A bug worth knowing about

If you are running a similar config, check this one. It presents as *"the vents
keep going into manual mode and changing speed on their own."*

```yaml
restore_mode: RESTORE_DEFAULT_OFF
on_turn_on:
  - lambda: 'if (!id(auto_applying)) { id(auto_mode) = false; }'
```

Restoring the fan state happens inside `Fan::setup()`.
`FanRestoreState::apply()` calls `fan.publish_state()`, which calls
`state_callback_.call()` — and that is exactly what `FanTurnOnTrigger`
subscribes to. The trigger is edge-triggered with `last_on_` initialised to
`false`, so a fan restoring to **on** looks like a genuine off→on transition and
fires `on_turn_on`. The `auto_applying` guard is false during boot, so AUTO gets
switched off on every reboot that restored a running fan.

From there the device is stuck: `apply_auto` returns immediately because it is
gated on `auto_mode`, so the fan sits at its restored speed indefinitely. That
accounts for both halves of the symptom. A fan restoring to *off* does not fire,
which is why it looks intermittent.

Fixed here with `restore_mode: ALWAYS_OFF` plus a `booting` global guarding both
fan triggers, so no boot-time state publish can change the mode. The four panel
buttons are also debounced — contact bounce on **Mode** toggles AUTO more than
once per press, which looks like the same symptom from a different cause.

## Why not Zigbee

The ESP32-C6 has an 802.15.4 radio and ESPHome has a Zigbee component, so this
looks tempting. It does not work for this design, and the reason is not radio
quality.

ESPHome's Zigbee component supports only `light`, `switch`, `binary_sensor` and
`sensor`. There is **no Zigbee equivalent of `sensor: platform: homeassistant`** —
that is a native-API feature. The entire control model depends on HA pushing room
temperature and setpoints to the device, so the on-device logic could not work at
all. On top of that there is no `fan` platform (speed would have to be smuggled
through a dimmable `light` and re-wrapped on the HA side), no `number` platform
for any of the tuning values, and no OTA, explicitly not planned. Running WiFi
alongside for OTA is possible on the C6, but ESPHome warns that WiFi station plus
a Zigbee *router* destabilises the mesh — and a mains-powered vent controller is
exactly what you would want routing.

WiFi and the native API give real `fan` / `number` / `switch` entities, OTA,
logs, a local web UI, and HA-imported sensors. Zigbee's genuine advantage here is
mesh range; if a vent has weak WiFi, a u.FL external antenna is the cheaper fix.
On the XIAO C6, GPIO3 must be driven low to power the RF switch and GPIO14
selects internal versus external.

## Credits and license

Based on the AirTap T-Series Gen 2 Rev 1 (4-button) ESPHome configuration by
**[SiloCity Labs](https://github.com/SiloCityLabs/esp32-airtap)**, which also
covers the Gen 1 boards, 3D-printable housings and hardware documentation. They
sell assembled boards — that project is where to start if you have not done the
hardware modification yet.

Licensed **CC BY-SA 4.0**, inherited from upstream's ShareAlike term. See
[LICENSE](LICENSE).

The author's earlier Bluetooth LE approach — a HACS integration talking to the
stock AirTap controller — lives at
[adman234/ac-infinity-airtap-hacs](https://github.com/adman234/ac-infinity-airtap-hacs)
and is superseded by this.
