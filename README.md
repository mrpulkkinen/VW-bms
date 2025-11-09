# VW-bms
A Teensy 3.2 firmware and set of helper sketches for decoding, monitoring, and controlling VW e-Golf battery modules over CAN. The active firmware (`VWBMSV2`) polls the OEM CMUs, enforces pack safety limits, drives contactors/chargers, and exposes a serial console for diagnostics and configuration.

## Firmware highlights
- `VWBMSV2/VWBMSV2.ino` bootstraps the hardware (GPIO, ADC, PWM, CAN, EEPROM), tracks pack state (SOC, charge/discharge modes, charger coordination), and mediates Victron VE.Direct/VE.Can messaging.
- `BMSModule.*` models an individual CMU: it decodes cell voltages, temperatures, alert/fault bits, and balancing status directly from VW frames.
- `BMSModuleManager.*` owns a fleet of modules, providing helpers to sum pack voltage, aggregate temperature extremes, balance cells, and produce CSV/console summaries used by the logger and CLI.
- `SerialConsole.*` implements the USB menu (`h`, `p`, `d`, `B`, etc.) so you can wake boards, clear faults, trigger balancing, and stream pack details every few seconds while tuning parameters.
- `Logger.*` delivers timestamped log output with adjustable verbosity (`LOGLEVEL` in the console) and is the backbone for bench validation.
- `CONFIG.H` and the `EEPROMSettings` struct centralize hardware defaults: pack geometry, charger profiles, current-sensor selection, CAN IDs, trip timers, and hysteresis thresholds. Rebuild whenever you touch this file so the Teensy EEPROM stays synchronized.

## Repository layout
- `VWBMSV2/` – Active Teensy 3.2 sketch plus supporting modules, logger, CLI, config header, and generated HEX/build artifacts.
- `Teensy_can/` – Minimal FlexCAN sketch for sniffing bus traffic or validating charger/module IDs before flashing the full firmware.
- `VWbms/` – Quick prototype sketch used for experiments outside the main firmware loop.
- `VW GTE E-Golf Can/` – Reference DBC, INI, and spreadsheet captures of VW IDs (`0x0BA` poll, `0x1CC–0x1D4` replies) for verifying decoder changes.
- `e-golf-notes.md` – Wiring, contactor, and EEPROM migration details specific to the e-Golf integration checklist.
- `AGENTS.md` – Maintainer guidance on coding style, safety practices, and PR expectations (summarized here).

## Build and flash
Install the Teensy core once:

```bash
arduino-cli core install teensy:avr
```

Compile, upload, and monitor via USB:

```bash
arduino-cli compile --fqbn teensy:avr:teensy31 VWBMSV2
arduino-cli upload --fqbn teensy:avr:teensy31 -p /dev/ttyACM0 VWBMSV2
teensy_loader_cli --mcu=TEENSY32 -w VWBMSV2/build/VWBMSV2.ino.hex
arduino-cli monitor -p /dev/ttyACM0 -c baudrate=115200
```

Rebuild any time you modify `CONFIG.H`, `EEPROMSettings`, or module code so cached artifacts do not carry stale calibrations.

## Configuration workflow
1. **Pack geometry & thresholds** – Edit `VWBMSV2/CONFIG.H` to set `Scells`, `Pstrings`, voltage/temperature limits, charger types, current-sensor selection (`Analoguedual`, `Canbus`, etc.), and CAN IDs (`controlid` `0x0BA`, module response base `0x1CC` by default).
2. **EEPROM schema** – Keep the `EEPROMSettings` layout in sync with your changes and bump `EEPROM_VERSION` whenever fields move; this prevents mismatched flash/EEPROM data.
3. **Board profiles** – Map pins (inputs, contactor outputs, PWM current display) directly in `VWBMSV2.ino` or extend `CONFIG.H` if you bring up new hardware.
4. **Victron / charger integration** – Adjust VE.Direct strings, charger CAN IDs, and current limits inside `VWBMSV2.ino` to match your hardware, then document the measured voltage and temperature behavior in your PR.
5. **Balancing & ignore windows** – Tune `balanceVoltage`, `balanceHyst`, `IgnoreVolt`, and `DeltaVolt` via `CONFIG.H` so `BMSModuleManager::balanceCells` keeps the pack within safe deltas.

## Diagnostics and testing
- **Serial Console** – Connect at 115200 baud, toggle pack summaries (`p`) or detailed dumps (`d`), run `B` to pulse balancing, `F/R/W` to discover and renumber CMUs, and set `LOGLEVEL` interactively.
- **Logger** – For bench work, raise the log level in `Logger.cpp` or through the console so you capture charge/drive/balance transitions alongside CAN measurements.
- **CAN sniffing** – Use `Teensy_can/Teensy_can.ino` or an external tool with `VW GTE E-Golf Can/VWtest.dbc` to confirm decoder changes (`decodecan`, `decodebalVW`, `decodetemp`) before committing.
- **Bench validation** – Exercise charge, drive, and balance modes while capturing Victron/charger interactions; there is no automated unit test harness yet, so console logs and CAN captures prove behavior.

## Safety and integration notes
- Follow the e-Golf checklist in `e-golf-notes.md`: discharge the DC-link capacitor before service, wire TE 1-1670990-1 pins (1=GND, 3=Enable 12 V, 5=+12 V, 6/7=CAN) correctly, and set pack series count plus charger thresholds in `CONFIG.H`.
- Document EEPROM layout or migration changes in PRs so other boards can be upgraded safely.
- Never commit customer-specific CAN keys or VINs; keep any local overrides in ignored `*.local.h` files.

With this layout and tooling, you can iterate on VW e-Golf battery support, extend sensor/charger integrations, and capture the evidence needed for safe firmware updates.
