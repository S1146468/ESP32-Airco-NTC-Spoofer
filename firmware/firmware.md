# Firmware

This document explains the ESPHome firmware for the **ESP32 Airco NTC Spoofer**, the available configuration variants, how the NTC control works, and how to tune the compressor protection timings.

## Firmware variants

Two example ESPHome configurations are provided.

### Two-room configuration

Use:

```text
esp32-airco-ntc-spoofer-two-room.yaml
```

This version supports two Home Assistant room-temperature sensors and an `Airco Area` selector.

The user can select:

```text
None
Room 1
Room 2
```

The selected room sensor is used by the local ESPHome thermostat.

This version keeps the behaviour of the original firmware. The installation-specific names and Home Assistant entity IDs are moved into `substitutions`, but the control flow itself is unchanged.

### Single-room configuration

Use:

```text
esp32-airco-ntc-spoofer-single-room.yaml
```

This version is intended for installations with one fixed room-temperature sensor.

It removes:

- the second room sensor;
- the `Airco Area` selector;
- the logic used to choose between two room sensors.

The NTC switching principle and thermostat behaviour remain the same.

## Hardware / GPIO mapping

The corrected PCB mapping is:

| XIAO ESP32-C3 | GPIO | Function |
|---|---:|---|
| D2 | GPIO4 | Original air-conditioner NTC path |
| D3 | GPIO5 | 5 °C spoof path |
| D4 | GPIO6 | 35 °C spoof path |

The resistor placement and firmware mapping must agree with this table.

The intended spoof states are:

```text
GPIO5 -> NTC_5C  -> cold simulated temperature
GPIO6 -> NTC_35C -> warm simulated temperature
```


## UML diagrams

The diagrams below document the firmware architecture and the control flow. They use a simple **draw.io / diagrams.net-style** layout so the relationships stay readable when viewed in GitHub.

The editable diagrams.net source is available at:

```text
docs/diagrams/firmware-diagrams.drawio
```

### System architecture

This component-style diagram shows how Home Assistant, the ESPHome thermostat, the GPIO outputs, the NTC switching hardware, and the portable air conditioner interact.

![Firmware architecture](docs/diagrams/firmware-architecture.svg)

The important boundary is that the ESP32 does not directly control the compressor. It selects the resistance seen by the air conditioner's NTC input. The air conditioner's own controller still decides when to operate its compressor.

### Two-room control state

The two-room firmware preserves the original control logic. Every control cycle checks the conditions required for external NTC control. If any required condition fails, the original NTC path is selected.

![Two-room control flow](docs/diagrams/firmware-control-flow.svg)

The `COOL` and `IDLE` states shown above correspond directly to the ESPHome thermostat actions:

```text
COOL
  -> current_state_ntc_35c = true
  -> current_state_ntc_5c  = false

IDLE
  -> current_state_ntc_5c  = true
  -> current_state_ntc_35c = false
```

### Periodic update sequence

The following sequence diagram shows the order used by the existing two-room `update_all` script.

![Firmware update sequence](docs/diagrams/firmware-update-sequence.svg)

The order is intentionally:

```text
airco_active_state
        ↓
airco_set_temp_mode
        ↓
output_set
```

This matters because the thermostat can update `current_state_ntc_5c` or `current_state_ntc_35c` before `output_set` applies the corresponding GPIO state during the same control cycle.


## Operating principle

The PCB does **not** directly switch the compressor or mains supply.

Instead, it changes the resistance seen by the air conditioner's NTC temperature-sensor input.

The air conditioner still controls its own compressor and internal protection logic.

The PCB can present one of three states:

```text
Original NTC
5 °C spoof
35 °C spoof
```

Conceptually:

```text
                           +-------------------+
Original NTC -------------|                   |
                           | NTC switching PCB |----> Air-conditioner NTC input
5 °C resistor ------------|                   |
                           |                   |
35 °C resistor -----------|                   |
                           +-------------------+
                                    ^
                                    |
                                 ESP32-C3
```

## Thermostat behaviour

The ESP32 runs an ESPHome thermostat using a Home Assistant room-temperature sensor.

Home Assistant supplies:

- the current room temperature;
- the requested air-conditioner mode;
- the requested target temperature.

When cooling is required:

```text
35 °C spoof selected
```

The air conditioner sees a warm simulated temperature and therefore continues cooling.

When the target temperature is satisfied:

```text
5 °C spoof selected
```

The air conditioner sees a cold simulated temperature and therefore stops requesting cooling.

When external control is inactive, the original NTC is reconnected.

## Two-room control flow

The two-room configuration works as follows:

```text
Room 1 sensor -----+
                   |
                   +----> Airco Area selector ----> room_temp
                   |
Room 2 sensor -----+

Home Assistant climate mode --------+
                                     |
Home Assistant target temperature ---+----> ESPHome thermostat
                                     |
room_temp ---------------------------+
```

The selected room is configured through:

```yaml
substitutions:
  room_1_name: "Room 1"
  room_2_name: "Room 2"

  room_1_temperature_entity: "sensor.room_1_temperature"
  room_2_temperature_entity: "sensor.room_2_temperature"

  airco_climate_entity: "climate.air_conditioner"
```

The available area options are:

```text
None
Room 1
Room 2
```

When `None` is selected, external NTC spoofing is disabled and the original NTC is used.

## Single-room control flow

The single-room version removes the room selector.

The configured room sensor is always used:

```text
Room sensor ------------------------+
                                    |
Home Assistant climate mode --------+----> ESPHome thermostat
                                    |
Home Assistant target temperature --+
```

Configure it with:

```yaml
substitutions:
  room_temperature_entity: "sensor.room_temperature"
  airco_climate_entity: "climate.air_conditioner"
```

## Two-room configuration behaviour

The generalized two-room file intentionally preserves the original firmware behaviour.

External control is considered active when:

- Wi-Fi is connected;
- the selected room-temperature sensor contains a valid value;
- an area other than `None` is selected;
- the Home Assistant climate entity reports `cool`;
- the Home Assistant target temperature contains a valid numeric value.

The firmware checks these conditions in:

```text
airco_active_state
```

The main control sequence is:

```text
update_all
    |
    +--> airco_active_state
    |
    +--> airco_set_temp_mode
    |
    +--> output_set
```

This sequence runs every:

```yaml
update_interval: "5s"
```

by default.

## NTC output logic

When external control is active:

```text
Original NTC -> disconnected

Then either:

5 °C spoof  -> active
35 °C spoof -> inactive

or:

5 °C spoof  -> inactive
35 °C spoof -> active
```

When external control is inactive:

```text
Original NTC -> connected
5 °C spoof   -> disconnected
35 °C spoof  -> disconnected
```

This lets the air conditioner return to its own original temperature sensor.

## Thermostat hysteresis

The two-room configuration currently uses:

```yaml
cool_deadband: 0.1 °C
cool_overrun: 0.35 °C
```

With a target of:

```text
22.0 °C
```

cooling starts at approximately:

```text
22.0 + 0.1 = 22.1 °C
```

and cooling stops at approximately:

```text
22.0 - 0.35 = 21.65 °C
```

This creates hysteresis around the target temperature.

A wider hysteresis generally produces longer cooling cycles and fewer compressor starts.

A narrower hysteresis generally produces tighter temperature regulation but may increase cycling frequency.

## Compressor protection settings

The firmware contains:

```yaml
substitutions:
  min_off_time: "4min"
  min_on_time: "6min"
  min_idle_time: "0s"
```

These are used by the ESPHome thermostat as:

```yaml
min_cooling_off_time: ${min_off_time}
min_cooling_run_time: ${min_on_time}
min_idle_time: ${min_idle_time}
```

### `min_off_time`

`min_off_time` controls how long cooling must remain OFF before the thermostat may request cooling again.

Its main purpose is to avoid rapid compressor restarts.

Example:

```yaml
min_off_time: "4min"
```

means that after cooling stops, the ESPHome thermostat will not request another cooling cycle for at least four minutes.

### `min_on_time`

`min_on_time` controls how long cooling must remain ON before the thermostat may stop cooling.

Example:

```yaml
min_on_time: "6min"
```

means that once the thermostat starts a cooling cycle, it will keep that state active for at least six minutes.

### `min_idle_time`

The default is:

```yaml
min_idle_time: "0s"
```

For this project, compressor restart protection is mainly handled by `min_off_time`.

`min_idle_time` normally does not need to duplicate the same delay.

# Tuning `min_off_time` and `min_on_time`

The timing values should be tuned for the portable air conditioner being controlled.

A practical method is to connect the portable air conditioner to a **power-monitoring smart-home plug** and observe the compressor's normal ON and OFF cycling.

> [!IMPORTANT]
> The air conditioner must be **actively regulating room temperature** during the measurement.
>
> If the room is much warmer than the target and the compressor runs continuously, the measurement does not provide useful minimum ON/OFF cycle information.

## Required equipment

You need:

- the portable air conditioner;
- a power-monitoring smart plug;
- access to the plug's live or historical power graph;
- a room where the air conditioner can repeatedly reach and move away from its target temperature.

The smart plug must be rated for the air conditioner's voltage, current, and startup load.

## Why power monitoring works

A portable air conditioner often has at least two clearly distinguishable power levels:

```text
High power -> compressor running
Lower power -> compressor stopped, fan may still be running
```

The fan may continue running when the compressor stops, so compressor OFF does **not** necessarily mean the power consumption drops to zero.

A simplified power trace looks like:

```text
Power
  ^
  |
  |        +---------------+              +---------------+
  |        |  Compressor   |              |  Compressor   |
  |        |      ON       |              |      ON       |
  |--------+               +--------------+               +------
  |
  |          <--- ON --->     <--- OFF --->   <--- ON --->
  |
  +------------------------------------------------------------> time
```

## Measurement procedure

### 1. Connect the smart plug

Connect the air conditioner through the power-monitoring smart plug.

Verify that the plug updates quickly enough to distinguish compressor starts and stops.

### 2. Let the air conditioner regulate normally

Set the air conditioner to cooling mode.

Choose a setpoint that causes the unit to repeatedly reach the target and then allow the temperature to rise again.

The desired behaviour is:

```text
compressor ON
      |
      v
room cools
      |
      v
compressor OFF
      |
      v
room warms
      |
      v
compressor ON again
```

If the compressor runs continuously, the room is not close enough to the setpoint for useful timing measurements.

## 3. Measure several cycles

Record multiple complete compressor cycles.

For example:

| Cycle | Compressor ON time | Compressor OFF time |
|---|---:|---:|
| 1 |  |  |
| 2 |  |  |
| 3 |  |  |
| 4 |  |  |
| 5 |  |  |

Do not use a single cycle as the basis for the final configuration.

## 4. Determine the observed OFF time

For each cycle:

```text
OFF time =
next compressor start
-
previous compressor stop
```

Find the shortest repeatable OFF period.

Example measurements:

```text
4m 20s
3m 55s
4m 05s
3m 47s
4m 11s
```

Shortest observed OFF time:

```text
3m 47s
```

A reasonable conservative setting would be:

```yaml
min_off_time: "4min"
```

## 5. Determine the observed ON time

For each cycle:

```text
ON time =
compressor stop
-
compressor start
```

Example measurements:

```text
6m 30s
5m 58s
5m 42s
6m 15s
5m 55s
```

Shortest observed ON time:

```text
5m 42s
```

A reasonable conservative setting would be:

```yaml
min_on_time: "6min"
```

## 6. Update the configuration

Change:

```yaml
substitutions:
  min_off_time: "4min"
  min_on_time: "6min"
```

to the values determined for the air conditioner.

Then validate, compile, and flash the ESPHome configuration.

## Important limitation of this method

The power-monitoring method tells you how the air conditioner behaves during normal operation.

It does **not** necessarily reveal the manufacturer's absolute compressor protection limits.

For example, an observed six-minute compressor ON period may simply mean that the room took six minutes to cool enough.

It does not prove that the compressor has an internal six-minute minimum-run timer.

Similarly:

```text
observed OFF time
```

may consist of:

```text
internal anti-short-cycle delay
+
time needed for the room to warm above the thermostat threshold
```

For this reason:

- use manufacturer or service-manual limits if they are available;
- measure multiple cycles;
- do not deliberately force rapid compressor cycling;
- prefer conservative values;
- round timings upward rather than downward.

The goal is not to discover the shortest possible compressor cycle.

The goal is to make sure the external thermostat does not request cycling faster than the air conditioner normally does.

## Recommended tuning workflow

A practical workflow is:

1. Run the portable air conditioner without changing the ESPHome protection values.
2. Connect it through a suitable power-monitoring smart plug.
3. Let it actively regulate the room temperature.
4. Measure several compressor ON periods.
5. Measure several compressor OFF periods.
6. Find the shortest repeatable ON and OFF times.
7. Compare them with the current firmware settings.
8. Round conservatively upward.
9. Update `min_on_time` and `min_off_time`.
10. Run the air conditioner again with the NTC spoofer controlling it.
11. Compare the new power trace with the original behaviour.

If the external thermostat causes significantly more frequent compressor starts than the air conditioner's own control system, increase the protection timings and/or thermostat hysteresis.

## Example tuning result

Suppose the measured cycles are:

| Cycle | ON time | OFF time |
|---|---:|---:|
| 1 | 7:10 | 4:35 |
| 2 | 6:25 | 4:08 |
| 3 | 5:42 | 3:47 |
| 4 | 6:05 | 4:12 |
| 5 | 5:51 | 3:55 |

The shortest observed values are:

```text
ON  = 5m 42s
OFF = 3m 47s
```

The resulting configuration could be:

```yaml
substitutions:
  min_off_time: "4min"
  min_on_time: "6min"
```

# Configuration examples

## Two-room example

```yaml
substitutions:
  device_name: "esp32-airco-ntc-spoofer"
  friendly_name: "ESP32 Airco NTC Spoofer"

  room_1_name: "Living Room"
  room_2_name: "Bedroom"

  room_1_temperature_entity: "sensor.living_room_temperature"
  room_2_temperature_entity: "sensor.bedroom_temperature"

  airco_climate_entity: "climate.portable_air_conditioner"

  update_interval: "5s"

  min_off_time: "4min"
  min_on_time: "6min"
  min_idle_time: "0s"
```

## Single-room example

```yaml
substitutions:
  device_name: "esp32-airco-ntc-spoofer"
  friendly_name: "ESP32 Airco NTC Spoofer"

  room_temperature_entity: "sensor.living_room_temperature"
  airco_climate_entity: "climate.portable_air_conditioner"

  update_interval: "5s"

  min_off_time: "4min"
  min_on_time: "6min"
  min_idle_time: "0s"

  cool_deadband: "0.1 °C"
  cool_overrun: "0.35 °C"
```

# Secrets

Do not commit actual Wi-Fi credentials, API encryption keys, OTA passwords, or other private installation data to the public repository.

Use ESPHome's `secrets.yaml`.

Example:

```yaml
wifi_ssid: "YOUR_WIFI_SSID"
wifi_password: "YOUR_WIFI_PASSWORD"

api_encryption_key: "YOUR_API_ENCRYPTION_KEY"
ota_password: "YOUR_OTA_PASSWORD"

device_static_ip: "192.168.1.50"
gateway_ip: "192.168.1.1"
subnet: "255.255.255.0"
```

For a public repository, provide a file such as:

```text
secrets.example.yaml
```

containing placeholders only.

# Commissioning checklist

Before relying on the controller unattended:

- [ ] Verify the correct resistor is fitted to the 5 °C path.
- [ ] Verify the correct resistor is fitted to the 35 °C path.
- [ ] Verify GPIO5 controls `NTC_5C`.
- [ ] Verify GPIO6 controls `NTC_35C`.
- [ ] Verify the original NTC works when external control is inactive.
- [ ] Verify the 5 °C spoof produces the expected air-conditioner response.
- [ ] Verify the 35 °C spoof produces the expected air-conditioner response.
- [ ] Verify the configured Home Assistant room sensor is correct.
- [ ] For the two-room version, verify both `Airco Area` selections.
- [ ] Verify the target temperature is imported correctly.
- [ ] Verify cooling mode is imported correctly.
- [ ] Measure several compressor ON/OFF cycles.
- [ ] Tune `min_off_time` and `min_on_time`.
- [ ] Observe the complete system over multiple normal cooling cycles.

# ESPHome reference

ESPHome thermostat documentation:

<https://esphome.io/components/climate/thermostat/>
