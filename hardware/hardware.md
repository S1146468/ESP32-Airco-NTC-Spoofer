# Hardware

This directory contains the editable PCB design, fabrication outputs, assembly data, mechanical export, and reference documentation for the **ESP32 Airco NTC Spoofer**.

The board is designed to sit between a portable air conditioner's original NTC temperature sensor and the air-conditioner controller. An ESP32-C3 can select the original NTC or one of two fixed-resistance spoof paths.

The PCB does **not** directly switch the compressor or mains power. It only changes what the air conditioner's NTC input sees.

For firmware setup and thermostat behaviour, see:

[../firmware/firmware.md](../firmware/firmware.md)

## Contents

### Editable EasyEDA source

- [Schematic source](easyeda/SCH_ESP32-Airco-NTC-Spoofer-Schematic.json)
- [PCB source](easyeda/PCB_ESP32-Airco-NTC-Spoofer-PCB.json)
- [EasyEDA import notes](easyeda/README.txt)

### Design documents

- [Schematic PDF](docs/Schematic_ESP32-Airco-NTC-Spoofer.pdf)
- [PCB PDF](docs/PCB_ESP32-Airco-NTC-Spoofer-PCB.pdf)

### Manufacturing files

- [Gerber ZIP](gerbers/Gerber_ESP32-Airco-NTC-Spoofer.zip)
- [Individual Gerber and drill files](gerbers/Gerber_Export/)
- [Bill of materials](bom/BOM_ESP32-Airco-NTC-Spoofer.csv)
- [Pick-and-place file](assembly/PickAndPlace_ESP32-Airco-NTC-Spoofer-PCB.csv)

### Mechanical files

- [OBJ 3D model](mechanical/OBJ_ESP32-Airco-NTC-Spoofer-PCB.obj)
- [OBJ material file](mechanical/OBJ_ESP32-Airco-NTC-Spoofer-PCB.mtl)

## Directory structure

```text
hardware/
├── assembly/
│   └── PickAndPlace_ESP32-Airco-NTC-Spoofer-PCB.csv
├── bom/
│   └── BOM_ESP32-Airco-NTC-Spoofer.csv
├── docs/
│   ├── PCB_ESP32-Airco-NTC-Spoofer-PCB.pdf
│   └── Schematic_ESP32-Airco-NTC-Spoofer.pdf
├── easyeda/
│   ├── PCB_ESP32-Airco-NTC-Spoofer-PCB.json
│   ├── SCH_ESP32-Airco-NTC-Spoofer-Schematic.json
│   └── README.txt
├── gerbers/
│   ├── Gerber_ESP32-Airco-NTC-Spoofer.zip
│   └── Gerber_Export/
├── mechanical/
│   ├── OBJ_ESP32-Airco-NTC-Spoofer-PCB.mtl
│   └── OBJ_ESP32-Airco-NTC-Spoofer-PCB.obj
└── hardware.md
```

## Hardware overview

The design contains three selectable NTC paths:

1. the original air-conditioner NTC;
2. a fixed resistance representing approximately **5 °C**;
3. a fixed resistance representing approximately **35 °C**.

The ESP32 does not measure the air conditioner's NTC input. It controls solid-state relays that determine which path is presented to the air-conditioner controller.

Conceptually:

```text
 Original NTC
     |
     +-------------------+
                         |
 39 kΩ / 5 °C spoof ----+----> NTC switching ----> Air-conditioner NTC input
                         |
 10 kΩ / 35 °C spoof ---+
                         ^
                         |
                    XIAO ESP32-C3
```

The exact relationship between resistance and simulated temperature depends on the NTC curve used by the air conditioner. The 39 kΩ and 10 kΩ values are the values used in this design.

## NTC switching architecture

### Original NTC path

The original sensor path uses:

```text
U4, U5 -> CPC1117NTR
```

These devices form the original NTC pass-through path.

The hardware is designed so that the original NTC path is the fallback state when the control electronics are not actively spoofing the temperature.

### 5 °C spoof path

The 5 °C spoof path uses:

```text
R5 = 39 kΩ
GPIO5 / XIAO D3
NTC_5C
```

The firmware activates this path when it wants the air conditioner to see a cold temperature.

### 35 °C spoof path

The 35 °C spoof path uses:

```text
R8 = 10 kΩ
GPIO6 / XIAO D4
NTC_35C
```

The firmware activates this path when it wants the air conditioner to see a warm temperature.

### GPIO mapping

The PCB and firmware must use the same mapping:

| XIAO ESP32-C3 | GPIO | PCB net | Function |
|---|---:|---|---|
| D2 | GPIO4 | `NTC_AIRCO` | Original NTC path control |
| D3 | GPIO5 | `NTC_5C` | 5 °C spoof path |
| D4 | GPIO6 | `NTC_35C` | 35 °C spoof path |

For the matching firmware configuration, see:

[../firmware/firmware.md](../firmware/firmware.md)

## Fail-safe behaviour

The hardware uses different relay types for the original sensor path and the spoof paths.

The intended unpowered / fallback state is:

```text
Original NTC -> connected
5 °C spoof   -> disconnected
35 °C spoof  -> disconnected
```

This is important because a loss of ESP32 control should return temperature sensing to the original air-conditioner NTC rather than intentionally forcing either spoof resistance.

The firmware should preserve the same principle whenever external control is inactive.

Before installing the board, verify this behaviour with a multimeter.

## Main components

The current BOM contains the following major devices:

| Reference | Part | Function |
|---|---|---|
| U3 | XIAO ESP32-C3 | Main controller |
| U4, U5 | CPC1117NTR | Original NTC switching path |
| U6-U9 | CPC1017NTR | Spoof-path switching |
| R5 | 39 kΩ | 5 °C spoof resistor |
| R8 | 10 kΩ | 35 °C spoof resistor |
| U1 | AP63205WU-7 | Power conversion |
| U2 | LM66100DCKR | Power-path circuitry |
| NTC_IN | JST SM02B-GHS-TB(LF)(SN) | Original NTC connection |
| NTC_OUT | JST SM02B-GHS-TB(LF)(SN) | Air-conditioner NTC input connection |
| PWR | JST SM02B-GHS-TB(LF)(SN) | Board power connection |

The complete component list, manufacturer part numbers, and LCSC part numbers are available in:

[BOM_ESP32-Airco-NTC-Spoofer.csv](bom/BOM_ESP32-Airco-NTC-Spoofer.csv)

## Connectors

The board contains three main connectors:

```text
PWR
NTC_IN
NTC_OUT
```

### `NTC_IN`

Connect the original air-conditioner NTC sensor to `NTC_IN`.

### `NTC_OUT`

Connect `NTC_OUT` to the air-conditioner controller where the original NTC would normally connect.

The board then selects whether `NTC_OUT` sees:

- the real sensor;
- the 5 °C spoof resistance;
- the 35 °C spoof resistance.

### `PWR`

`PWR` supplies the PCB electronics.

Refer to the schematic before connecting power. Do not assume that the NTC connector and power connector use the same voltage or pinout.

## Editable design files

The board was designed with **EasyEDA Standard**.

The editable source files are:

- [`easyeda/SCH_ESP32-Airco-NTC-Spoofer-Schematic.json`](easyeda/SCH_ESP32-Airco-NTC-Spoofer-Schematic.json)
- [`easyeda/PCB_ESP32-Airco-NTC-Spoofer-PCB.json`](easyeda/PCB_ESP32-Airco-NTC-Spoofer-PCB.json)

No custom schematic symbols or PCB footprints are required by the current design.

### Opening the source in EasyEDA Standard

In EasyEDA Standard:

1. open EasyEDA;
2. select **File -> Open -> EasyEDA...**;
3. select the JSON file;
4. import/save it into a project if you want to edit it.

The schematic and PCB JSON files are the authoritative editable hardware source.

The PDFs, Gerbers, BOM, pick-and-place data, and OBJ model are generated outputs.

## Schematic and PCB PDFs

PDF exports are included so the design can be inspected without EasyEDA.

### Schematic

[Open the schematic PDF](docs/Schematic_ESP32-Airco-NTC-Spoofer.pdf)

### PCB

[Open the PCB PDF](docs/PCB_ESP32-Airco-NTC-Spoofer-PCB.pdf)

If the EasyEDA source is changed, regenerate these PDFs so the documentation remains synchronized with the editable design.

## PCB manufacturing specifications

The first boards were manufactured by JLCPCB using the following configuration:

| Parameter | Specification |
|---|---|
| Base material | FR-4 |
| Layer count | 4 |
| Board dimensions | 31.29 mm × 57.78 mm |
| PCB thickness | 1.6 mm |
| Material Tg | TG135 |
| Solder mask | Blue |
| Silkscreen | White |
| Surface finish | HASL with lead |
| Outer copper weight | 1 oz |
| Inner copper weight | 0.5 oz |
| Via covering | Plugged |
| Minimum via hole | 0.30 mm |
| Electrical test | Flying probe, fully tested |
| Quality standard | IPC Class 2 |
| Board outline tolerance | ±0.2 mm |
| Specified stackup | No |
| Gold fingers | No |
| Castellated holes | No |
| Press-fit holes | No |
| Edge plating | No |
| Blind slots | No |
| Backdrilling | No |
| Countersink holes | No |
| Deburring / edge rounding | No |
| UL marking | No |
| Silkscreen technology | Ink-jet printing |

The original JLCPCB order reported the minimum via setting as approximately:

```text
0.3 mm hole / 0.4-0.45 mm diameter
```

Treat the EasyEDA PCB design and the fabrication house's current design-rule check as authoritative when placing a new order.

## Layer structure

The board uses four copper layers:

```text
Layer 1 -> Top copper
Layer 2 -> Inner layer 1
Layer 3 -> Inner layer 2
Layer 4 -> Bottom copper
```

No specific manufacturer dielectric stackup was requested for the original build.

If the design is changed in a way that depends on controlled impedance, dielectric thickness, or a specific return-plane geometry, define a stackup explicitly rather than relying on the manufacturer's standard four-layer construction.

## Gerber files

The fabrication ZIP is:

[Gerber_ESP32-Airco-NTC-Spoofer.zip](gerbers/Gerber_ESP32-Airco-NTC-Spoofer.zip)

The individual exported files are available in:

[gerbers/Gerber_Export/](gerbers/Gerber_Export/)

The current Gerber export contains:

```text
Drill_PTH_Through.DRL
Drill_PTH_Through_Via.DRL
Gerber_BoardOutlineLayer.GKO
Gerber_BottomLayer.GBL
Gerber_BottomSolderMaskLayer.GBS
Gerber_DocumentLayer.GDL
Gerber_InnerLayer1.G1
Gerber_InnerLayer2.G2
Gerber_TopLayer.GTL
Gerber_TopPasteMaskLayer.GTP
Gerber_TopSilkscreenLayer.GTO
Gerber_TopSolderMaskLayer.GTS
```

There is no bottom paste or bottom silkscreen file in the current export.

Before sending the ZIP to a board manufacturer, inspect it in a Gerber viewer.

Check at minimum:

- board outline;
- all four copper layers;
- plated-through-hole drills;
- via drills;
- solder-mask openings;
- top silkscreen;
- top paste layer;
- connector orientation.

## BOM

The bill of materials is:

[BOM_ESP32-Airco-NTC-Spoofer.csv](bom/BOM_ESP32-Airco-NTC-Spoofer.csv)

The BOM includes manufacturer and LCSC part numbers for the current design.

Before ordering:

- verify that the required parts are still available;
- verify that substitutions have compatible electrical characteristics;
- verify package and footprint compatibility;
- pay particular attention to the relay type and default contact state;
- verify that R5 remains 39 kΩ and R8 remains 10 kΩ.

Do not silently substitute the normally-closed original-NTC relay with a normally-open equivalent.

## Pick-and-place file

The assembly position file is:

[PickAndPlace_ESP32-Airco-NTC-Spoofer-PCB.csv](assembly/PickAndPlace_ESP32-Airco-NTC-Spoofer-PCB.csv)

When ordering PCBA assembly, always inspect the manufacturer's placement preview.

Verify:

- connector orientation;
- XIAO ESP32-C3 orientation;
- relay orientation;
- resistor values;
- polarity-sensitive parts;
- rotation of small IC packages.

Do not assume that a successful BOM/CPL import means all rotations are correct.

## Mechanical model

EasyEDA Standard exports the PCB assembly as OBJ rather than native STEP.

The current mechanical files are:

- [`OBJ_ESP32-Airco-NTC-Spoofer-PCB.obj`](mechanical/OBJ_ESP32-Airco-NTC-Spoofer-PCB.obj)
- [`OBJ_ESP32-Airco-NTC-Spoofer-PCB.mtl`](mechanical/OBJ_ESP32-Airco-NTC-Spoofer-PCB.mtl)

These files are suitable for visualization and approximate enclosure/layout work.

A native STEP file is not included.

If precise mechanical CAD integration is required, prefer a native STEP export from a CAD tool capable of producing proper solid geometry rather than simply converting the OBJ mesh and assuming it is equivalent.

## Regenerating manufacturing files

When the schematic or PCB is modified, use the editable EasyEDA source as the starting point.

A typical release workflow is:

1. modify the EasyEDA schematic;
2. update/synchronize the PCB if required;
3. run ERC on the schematic;
4. run DRC on the PCB;
5. regenerate the schematic PDF;
6. regenerate the PCB PDF;
7. regenerate the Gerber ZIP;
8. regenerate the BOM;
9. regenerate the pick-and-place file;
10. regenerate the OBJ model if the physical layout changed;
11. inspect the Gerbers;
12. compare the new outputs with the previous revision before publishing.

Generated manufacturing outputs should not be edited manually to create a design change.

## Hardware verification

Before ordering or installing a new revision, check the following.

### Design checks

- [ ] EasyEDA ERC completed.
- [ ] EasyEDA PCB DRC completed.
- [ ] Schematic and PCB source correspond to the same revision.
- [ ] Schematic PDF regenerated.
- [ ] PCB PDF regenerated.
- [ ] Gerbers regenerated from the current PCB.
- [ ] BOM regenerated from the current schematic/PCB.
- [ ] Pick-and-place file regenerated.
- [ ] Gerbers inspected in an independent viewer.

### NTC path checks

- [ ] GPIO4 controls the original NTC path.
- [ ] GPIO5 controls `NTC_5C`.
- [ ] GPIO6 controls `NTC_35C`.
- [ ] R5 is 39 kΩ.
- [ ] R8 is 10 kΩ.
- [ ] Original NTC passes through when the hardware is in its fallback state.
- [ ] 5 °C spoof path is isolated when inactive.
- [ ] 35 °C spoof path is isolated when inactive.
- [ ] Only the intended spoof resistance is connected when each spoof path is enabled.

### Assembly checks

- [ ] `NTC_IN` and `NTC_OUT` are not reversed during installation.
- [ ] `PWR` polarity/pinout is verified from the schematic.
- [ ] XIAO ESP32-C3 orientation is correct.
- [ ] Relay part numbers match the BOM.
- [ ] R5 and R8 values are physically verified.
- [ ] No solder bridges are present around the NTC switching circuitry.

## Bench testing before installation

Before connecting the PCB to an air conditioner, perform basic continuity and resistance checks.

With the board in its fallback state, verify that the original NTC path connects through from `NTC_IN` to `NTC_OUT`.

Then test the spoof states and confirm that the resistance visible at `NTC_OUT` changes to the expected value.

Do not connect the PCB to unknown high-voltage wiring. The design is intended for the air conditioner's low-voltage NTC sensing circuit and its intended board-power connection.

## Source vs generated files

The repository separates editable source from generated outputs.

### Editable source

```text
easyeda/*.json
```

### Generated documentation

```text
docs/*.pdf
```

### Generated fabrication data

```text
gerbers/*
bom/*
assembly/*
```

### Generated mechanical data

```text
mechanical/*.obj
mechanical/*.mtl
```

When there is a disagreement between generated files and the editable design source, stop and determine which revision is correct before manufacturing.

## Versioning

For released hardware revisions, record both a hardware revision and a repository release version.

For example:

```text
Hardware revision: REV 1.0
Repository release: v1.0.0
```

When the physical PCB changes, update the hardware revision and regenerate the manufacturing outputs.

Changes that affect only documentation do not necessarily require a new PCB revision.

## License

The hardware design files in this directory are licensed under the
**CERN Open Hardware Licence Version 2 - Strongly Reciprocal
(CERN-OHL-S-2.0)**.

See [CERN-OHL-S-2.0.txt](../LICENSES/CERN-OHL-S-2.0.txt).
